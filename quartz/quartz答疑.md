这篇笔记记录我在阅读quartz源码的时候是如何分析的，如何去查找问题的.

## 1. 任务的状态

可以参考https://segmentfault.com/a/1190000015492260 写的很好，分析很详细，这里我盗张图，quartz状态转化如下:

![](https://raw.githubusercontent.com/xiaoming-he/notes/master/img/20190803160044.png)

## 2. 如何查询任务

上节写过，生产者线程会查所有要执行的触发器，在QuartzSchedulerThread的run方法中

```java
 triggers = qsRsrcs.getJobStore().acquireNextTriggers(
                                now + idleWaitTime, Math.min(availThreadCount, qsRsrcs.getMaxBatchSize()), qsRsrcs.getBatchTimeWindow());
```

- idleWaitTime:默认30s，可通过配置属性`org.quartz.scheduler.idleWaitTime`设置。
- availThreadCount`：获取可用（空闲）的工作线程数量，总会大于1，因为该方法会一直阻塞，直到有工作线程空闲下来。
- `maxBatchSize`：一次拉取trigger的最大数量，默认是1，可通过`org.quartz.scheduler.batchTriggerAcquisitionMaxCount`改写
- `batchTimeWindow`：时间窗口调节参数，默认是0，可通过`org.quartz.scheduler.batchTriggerAcquisitionFireAheadTimeWindow`改写
- `misfireThreshold`： 超过这个时间还未触发的trigger,被认为发生了misfire,默认60s，可通过`org.quartz.jobStore.misfireThreshold`设置。

我们使用的是数据库存储，所以acquireNextTriggers调用的是org.quartz.impl.jdbcjobstore.JobStoreSupport#acquireNextTriggers。

```java
public List<OperableTrigger> acquireNextTriggers(final long noLaterThan, final int maxCount, final long timeWindow)
        throws JobPersistenceException {
        //1. 判断锁，获取锁
        String lockName;
        if(isAcquireTriggersWithinLock() || maxCount > 1) { 
            lockName = LOCK_TRIGGER_ACCESS;
        } else {
            lockName = null;
        }
        return executeInNonManagedTXLock(lockName, 
                new TransactionCallback<List<OperableTrigger>>() {
                    public List<OperableTrigger> execute(Connection conn) throws JobPersistenceException {
                      //这个方法是获取触发器的
                        return acquireNextTrigger(conn, noLaterThan, maxCount, timeWindow);
                    }
                },
                new TransactionValidator<List<OperableTrigger>>() {
                    public Boolean validate(Connection conn, List<OperableTrigger> result) throws JobPersistenceException {
                        try {
                            List<FiredTriggerRecord> acquired = getDelegate().selectInstancesFiredTriggerRecords(conn, getInstanceId());
                            Set<String> fireInstanceIds = new HashSet<String>();
                            for (FiredTriggerRecord ft : acquired) {
                                fireInstanceIds.add(ft.getFireInstanceId());
                            }
                            for (OperableTrigger tr : result) {
                                if (fireInstanceIds.contains(tr.getFireInstanceId())) {
                                    return true;
                                }
                            }
                            return false;
                        } catch (SQLException e) {
                            throw new JobPersistenceException("error validating trigger acquisition", e);
                        }
                    }
                });
    }
```

acquireNextTriggers方法先获取锁，然后回调调用acquireNextTrigger获取触发器，查询触发器的sql。

```sql
SELECT TRIGGER_NAME, TRIGGER_GROUP, NEXT_FIRE_TIME, PRIORITY FROM {0}TRIGGERS WHERE SCHED_NAME = {1} AND TRIGGER_STATE = ? AND 
NEXT_FIRE_TIME <= ? AND (MISFIRE_INSTR = -1 OR (MISFIRE_INSTR != -1 AND NEXT_FIRE_TIME >= ?)) ORDER BY NEXT_FIRE_TIME ASC, PRIORITY DESC
```

由sql可以得知，quartz可以查询过去60s将来30s的触发器。查询出来后会把触发器保存到QRTZ_FIRED_TRIGGERS表中，作用在第三节会讲。

## 3. 如何保证任务不丢失

任务在什么情况下会丢失。没有多余的消费者线程可以消费，服务器重启导致任务丢失。针对这两种情况，quartz是如何做的

### 3.1 线程堵塞导致任务丢失

如何模拟线程堵塞，这里我把消费者线程池大小设置为1

```yaml
spring:
  quartz:
    job-store-type: jdbc
    overwriteExistingJobs: true
    properties:
      org.quartz.threadPool.threadCount: 1
```

任务每5秒执行一次，但是执行的时候睡眠10s中，这样就会导致任务丢失。

```java
public class TestTaskJob1 extends QuartzJobBean {
    private static final Logger logger = LoggerFactory.getLogger(TestTaskJob1.class);

    @Override
    protected void executeInternal(JobExecutionContext jobExecutionContext) throws JobExecutionException {
        String localDateTime = LocalDateTime.now().format(DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm:ss"));
        logger.info("TestTaskJob1-->" + localDateTime);

        try {
            Thread.sleep(10000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}
```

在第二节中，在查询触发器的sql中有一句(MISFIRE_INSTR = -1 OR (MISFIRE_INSTR != -1 AND NEXT_FIRE_TIME >= ?)这样的条件，MISFIRE_INSTR的值如果是-1，则会一直查询出来，那么这个值代表什么意思呢？可以查看https://www.jianshu.com/p/634d2a6fae7b， 通过设置misfire的值，可以设置保证任务不丢失。不过执行时间有延迟而已。

### 3.2 服务器重启导致

misfire可以保证阻塞状态下，任务不丢失，但是如果正在执行过程中，服务器挂了，如何保证不丢失？
在第二节中讲到，查询触发器的时候，会把查询的触发器插入到QRTZ_FIRED_TRIGGERS表中，当服务器重启的时候，会去读取这张表，将任务恢复执行。

## 4. 如何保证分布式一致性

为了保证任务的可靠用，我们基本都会部署多台服务器，但是部署多台服务器就会出现任务在多台服务器中被执行，这种情况该如何处理。

在第二节中获取触发器的时候，获取触发器是通过executeInNonManagedTXLock方法回调的，看下executeInNonManagedTXLock的实现逻辑.

```java
protected <T> T executeInNonManagedTXLock(
            String lockName, 
            TransactionCallback<T> txCallback, final TransactionValidator<T> txValidator) throws JobPersistenceException {
        boolean transOwner = false;
        Connection conn = null;
        try {
           
           if (lockName != null) {
                // If we aren't using db locks, then delay getting DB connection 
                // until after acquiring the lock since it isn't needed.
                //1. 获取数据库连接
             		if (getLockHandler().requiresConnection()) {
                    conn = getNonManagedTXConnection();
                }
                //2.获取锁
                transOwner = getLockHandler().obtainLock(conn, lockName);
            }
            
            if (conn == null) {
                conn = getNonManagedTXConnection();
            }
            //3. 回调查询结果
            final T result = txCallback.execute(conn);
            try {
                commitConnection(conn);
            } catch (JobPersistenceException e) {
                rollbackConnection(conn);
                if (txValidator == null || !retryExecuteInNonManagedTXLock(lockName, new TransactionCallback<Boolean>() {
                    @Override
                    public Boolean execute(Connection conn) throws JobPersistenceException {
                        return txValidator.validate(conn, result);
                    }
                })) {
                    throw e;
                }
            }

            Long sigTime = clearAndGetSignalSchedulingChangeOnTxCompletion();
            if(sigTime != null && sigTime >= 0) {
                signalSchedulingChangeImmediately(sigTime);
            }
            
            return result;
        } catch (JobPersistenceException e) {
            rollbackConnection(conn);
            throw e;
        } catch (RuntimeException e) {
            rollbackConnection(conn);
            throw new JobPersistenceException("Unexpected runtime exception: "
                    + e.getMessage(), e);
        } finally {
            try {
                releaseLock(lockName, transOwner);
            } finally {
                cleanupConnection(conn);
            }
        }
    }
```

主要分析getLockHandler().obtainLock(conn, lockName);这段逻辑，getLockHandler是获取锁处理对象，因为使用的是数据库模式，所以是DBSemaphore#obtainLock。

```java
public boolean obtainLock(Connection conn, String lockName)
        throws LockException {

        if(log.isDebugEnabled()) {
            log.debug(
                "Lock '" + lockName + "' is desired by: "
                        + Thread.currentThread().getName());
        }
  			//1. 判断是不是自己获取了锁，锁可重入
        if (!isLockOwner(lockName)) {

          //2. 执行sql获取锁
            executeSQL(conn, lockName, expandedSQL, expandedInsertSQL);
            
            if(log.isDebugEnabled()) {
                log.debug(
                    "Lock '" + lockName + "' given to: "
                            + Thread.currentThread().getName());
            }
           //3. 如果取到锁，把锁放到threadLocal中
            getThreadLocks().add(lockName);
            //getThreadLocksObtainer().put(lockName, new
            // Exception("Obtainer..."));
        } else if(log.isDebugEnabled()) {
            log.debug(
                "Lock '" + lockName + "' Is already owned by: "
                        + Thread.currentThread().getName());
        }

        return true;
    }
```

获取锁的逻辑是：

1. 判断锁是不是已经被自己获取了，判断的逻辑是threadLocal中是不是有值，这个目的是锁可以重入。

2. 执行sql获取锁，

   ```java
   SELECT * FROM QRTZ_LOCKS WHERE SCHED_NAME = 'quartzScheduler' AND LOCK_NAME = ? FOR UPDATE
   ```

   根据锁的名字，使用for update获取行锁。