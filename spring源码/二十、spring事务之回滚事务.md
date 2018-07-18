Spring事务信息准备好后，如果我们的程序出现了异常，又会如何回滚事务呢？这节我们分析Spring事务回滚原理。
TransactionAspectSupport#invokeWithinTransaction方法中部分代码块
```java
    try {
    // This is an around advice: Invoke the next interceptor in the chain.
    // This will normally result in a target object being invoked.
    //执行业务逻辑
    retVal = invocation.proceedWithInvocation();
    }
    catch (Throwable ex) {
    // target invocation exception
    completeTransactionAfterThrowing(txInfo, ex);
    throw ex;
    }
```
当执行业务逻辑抛出异常后，调用completeTransactionAfterThrowing方法回滚事务，**这里得注意的是，如果程序不把异常抛出，而且捕获异常，Spring是不会回滚的**。
```java
protected void completeTransactionAfterThrowing(TransactionInfo txInfo, Throwable ex) {
    //判断是否存在事务
    if (txInfo != null && txInfo.hasTransaction()) {
        if (logger.isTraceEnabled()) {
            logger.trace("Completing transaction for [" + txInfo.getJoinpointIdentification() +
                    "] after exception: " + ex);
        }
        //根据异常类型判断是否回滚，可以通过Transactional注解中rollbackFor、rollbackForClassName、noRollbackForClassName配置
        if (txInfo.transactionAttribute.rollbackOn(ex)) {
            try {
                //处理回滚
                txInfo.getTransactionManager().rollback(txInfo.getTransactionStatus());
            }
            catch (TransactionSystemException ex2) {
                logger.error("Application exception overridden by rollback exception", ex);
                ex2.initApplicationException(ex);
                throw ex2;
            }
            catch (RuntimeException ex2) {
                logger.error("Application exception overridden by rollback exception", ex);
                throw ex2;
            }
            catch (Error err) {
                logger.error("Application exception overridden by rollback error", ex);
                throw err;
            }
        }
        else {
            // We don't roll back on this exception.
            // Will still roll back if TransactionStatus.isRollbackOnly() is true.
            //如果不满足符合的异常类型，也同样会提交
            try {
                txInfo.getTransactionManager().commit(txInfo.getTransactionStatus());
            }
            catch (TransactionSystemException ex2) {
                logger.error("Application exception overridden by commit exception", ex);
                ex2.initApplicationException(ex);
                throw ex2;
            }
            catch (RuntimeException ex2) {
                logger.error("Application exception overridden by commit exception", ex);
                throw ex2;
            }
            catch (Error err) {
                logger.error("Application exception overridden by commit error", ex);
                throw err;
            }
        }
    }
}
```
回滚逻辑如下:
1. 判断是否存在事务，只有存在事务才执行回滚
2. 根据异常类型判断是否回滚。如果异常类型不符合，仍然会提交事务
3. 回滚处理

详细分析每个步骤
## 回滚条件
```java
txInfo.transactionAttribute.rollbackOn(ex)
```
TransactionInfo中的TransactionAttribute属性值的是RuleBasedTransactionAttribute，在解析@Transactional注解时初始化，它的rollbackOn方法实现如下:
```java
@Override
public boolean rollbackOn(Throwable ex) {
    if (logger.isTraceEnabled()) {
        logger.trace("Applying rules to determine whether transaction should rollback on " + ex);
    }

    RollbackRuleAttribute winner = null;
    int deepest = Integer.MAX_VALUE;

    //rollbackRules保存@Transactional注解中rollbackFor、rollbackForClassName、noRollbackForClassName配置的值
    if (this.rollbackRules != null) {
        for (RollbackRuleAttribute rule : this.rollbackRules) {
            int depth = rule.getDepth(ex);
            if (depth >= 0 && depth < deepest) {
                deepest = depth;
                winner = rule;
            }
        }
    }

    if (logger.isTraceEnabled()) {
        logger.trace("Winning rollback rule is: " + winner);
    }

    // User superclass behavior (rollback on unchecked) if no rule matches.
    //若@Transactional没有配置，默认调用父类的
    if (winner == null) {
        logger.trace("No relevant rollback rule found: applying default rules");
        return super.rollbackOn(ex);
    }

    return !(winner instanceof NoRollbackRuleAttribute);
}

//super
@Override
public boolean rollbackOn(Throwable ex) {
    return (ex instanceof RuntimeException || ex instanceof Error);
}
```
判断是否能够回滚的逻辑如下：
(1) 根据@Transactional注解中rollbackFor、rollbackForClassName、noRollbackForClassName配置的值，找到最符合ex的异常类型，如果符合的异常类型不是NoRollbackRuleAttribute，则可以执行回滚。
(2) 如果@Transactional没有配置，则默认使用RuntimeException和Error异常。
## 回滚处理
```java
txInfo.getTransactionManager().rollback(txInfo.getTransactionStatus());
```
交给事务管理器回滚事务。
```java
private void processRollback(DefaultTransactionStatus status) {
    try {
        try {
            triggerBeforeCompletion(status);
            //如果有安全点，回滚至安全点
            if (status.hasSavepoint()) {
                if (status.isDebug()) {
                    logger.debug("Rolling back transaction to savepoint");
                }
                status.rollbackToHeldSavepoint();
            }
            //如果是新事务，回滚事务
            else if (status.isNewTransaction()) {
                if (status.isDebug()) {
                    logger.debug("Initiating transaction rollback");
                }
                doRollback(status);
            }
            //如果有事务但不是新事务，则把标记事务状态，等事务链执行完毕后统一回滚
            else if (status.hasTransaction()) {
                if (status.isLocalRollbackOnly() || isGlobalRollbackOnParticipationFailure()) {
                    if (status.isDebug()) {
                        logger.debug("Participating transaction failed - marking existing transaction as rollback-only");
                    }
                    doSetRollbackOnly(status);
                }
                else {
                    if (status.isDebug()) {
                        logger.debug("Participating transaction failed - letting transaction originator decide on rollback");
                    }
                }
            }
            else {
                logger.debug("Should roll back transaction but cannot - no transaction available");
            }
        }
        catch (RuntimeException ex) {
            triggerAfterCompletion(status, TransactionSynchronization.STATUS_UNKNOWN);
            throw ex;
        }
        catch (Error err) {
            triggerAfterCompletion(status, TransactionSynchronization.STATUS_UNKNOWN);
            throw err;
        }
        triggerAfterCompletion(status, TransactionSynchronization.STATUS_ROLLED_BACK);
    }
    finally {
        //清空记录的资源并将挂起的资源恢复
        cleanupAfterCompletion(status);
    }
}
```
回滚处理的逻辑如下，其实代码已经很清晰了。
1. 如果存在安全点，则回滚事务至安全点，这个主要是处理嵌套事务，回滚安全点的操作还是交给了数据库处理.
```java
public void rollbackToHeldSavepoint() throws TransactionException {
    if (!hasSavepoint()) {
        throw new TransactionUsageException(
                "Cannot roll back to savepoint - no savepoint associated with current transaction");
    }
    getSavepointManager().rollbackToSavepoint(getSavepoint());
    getSavepointManager().releaseSavepoint(getSavepoint());
    setSavepoint(null);
}
```
因为这里使用的是JDBC的方式进行数据库连接，所以getSavepointManager返回的是JdbcTransactionObjectSupport，看下JdbcTransactionObjectSupport#rollbackToSavepoint
```java
@Override
public void rollbackToSavepoint(Object savepoint) throws TransactionException {
    ConnectionHolder conHolder = getConnectionHolderForSavepoint();
    try {
        conHolder.getConnection().rollback((Savepoint) savepoint);
    }
    catch (Throwable ex) {
        throw new TransactionSystemException("Could not roll back to JDBC savepoint", ex);
    }
}
```
2. 当前事务是一个新事务时，那么直接回滚，使用的是DataSourceTransactionManager事务管理器，所以调用DataSourceTransactionManager#doRollback,直接调用数据库连接的回滚方法。
```java
@Override
protected void doRollback(DefaultTransactionStatus status) {
    DataSourceTransactionObject txObject = (DataSourceTransactionObject) status.getTransaction();
    Connection con = txObject.getConnectionHolder().getConnection();
    if (status.isDebug()) {
        logger.debug("Rolling back JDBC transaction on Connection [" + con + "]");
    }
    try {
        con.rollback();
    }
    catch (SQLException ex) {
        throw new TransactionSystemException("Could not roll back JDBC transaction", ex);
    }
}
```
3. 当前存在事务，但又不是一个新的事务，只把事务的状态标记为read-only，等到事务链执行完毕后，统一回滚,调用DataSourceTransactionManager#doSetRollbackOnly
```java
@Override
protected void doSetRollbackOnly(DefaultTransactionStatus status) {
    DataSourceTransactionObject txObject = (DataSourceTransactionObject) status.getTransaction();
    if (status.isDebug()) {
        logger.debug("Setting JDBC transaction [" + txObject.getConnectionHolder().getConnection() +
                "] rollback-only");
    }
    txObject.setRollbackOnly();
}
```
4. 清空记录的资源并将挂起的资源恢复
```java
private void cleanupAfterCompletion(DefaultTransactionStatus status) {
    //设置完成状态，避免重复调用
    status.setCompleted();
    //如果是新的同步状态，则需要将绑定到当前线程的事务信息清理，传播行为中挂起事务的都会清理
    if (status.isNewSynchronization()) {
        TransactionSynchronizationManager.clear();
    }
    //如果是新事务，清理数据库连接
    if (status.isNewTransaction()) {
        doCleanupAfterCompletion(status.getTransaction());
    }
    //将挂起的事务恢复
    if (status.getSuspendedResources() != null) {
        if (status.isDebug()) {
            logger.debug("Resuming suspended transaction after completion of inner transaction");
        }
        resume(status.getTransaction(), (SuspendedResourcesHolder) status.getSuspendedResources());
    }
}
```
