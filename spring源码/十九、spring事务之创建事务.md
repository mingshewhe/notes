接上面一节[十八、spring 事务之事务执行流程](https://www.jianshu.com/p/c56bdc7f1d1d)，Spring获取事务管理器后，就开始创建事务信息，这里面的逻辑就比较复杂了，spring的嵌套事务都是在这里处理的。
```java
protected TransactionInfo createTransactionIfNecessary(
			PlatformTransactionManager tm, TransactionAttribute txAttr, final String joinpointIdentification) {

    // If no name specified, apply method identification as transaction name.
    if (txAttr != null && txAttr.getName() == null) {
        txAttr = new DelegatingTransactionAttribute(txAttr) {
            @Override
            public String getName() {
                return joinpointIdentification;
            }
        };
    }

    TransactionStatus status = null;
    if (txAttr != null) {
        if (tm != null) {
            //构建事务状态
            status = tm.getTransaction(txAttr);
        }
        else {
            if (logger.isDebugEnabled()) {
                logger.debug("Skipping transactional joinpoint [" + joinpointIdentification +
                        "] because no transaction manager has been configured");
            }
        }
    }
    //封装事务信息
    return prepareTransactionInfo(tm, txAttr, joinpointIdentification, status);
}
```
createTransactionIfNecessary逻辑挺简单的，
1. 获取事务状态
2. 构建事务信息

# 获取事务状态
AbstractPlatformTransactionManager#getTransaction
```java
@Override
	public final TransactionStatus getTransaction(TransactionDefinition definition) throws TransactionException {
    //1.获取事务
    Object transaction = doGetTransaction();

    // Cache debug flag to avoid repeated checks.
    boolean debugEnabled = logger.isDebugEnabled();

    if (definition == null) {
        // Use defaults if no transaction definition given.
        definition = new DefaultTransactionDefinition();
    }

    //判断当前线程是否存在事务，判断依据为当前线程记录连接不为空且连接中的(connectionHolder)中的transactionActive属性不为空
    if (isExistingTransaction(transaction)) {
        // Existing transaction found -> check propagation behavior to find out how to behave.
        return handleExistingTransaction(definition, transaction, debugEnabled);
    }

    // Check definition settings for new transaction.
    //事务超时设置验证
    if (definition.getTimeout() < TransactionDefinition.TIMEOUT_DEFAULT) {
        throw new InvalidTimeoutException("Invalid transaction timeout", definition.getTimeout());
    }

    // No existing transaction found -> check propagation behavior to find out how to proceed.
    //如果当前线程不存在事务，但是@Transactional却声明事务为PROPAGATION_MANDATORY抛出异常
    if (definition.getPropagationBehavior() == TransactionDefinition.PROPAGATION_MANDATORY) {
        throw new IllegalTransactionStateException(
                "No existing transaction found for transaction marked with propagation 'mandatory'");
    }
    //如果当前线程不存在事务，PROPAGATION_REQUIRED、PROPAGATION_REQUIRES_NEW、PROPAGATION_NESTED都得创建事务
    else if (definition.getPropagationBehavior() == TransactionDefinition.PROPAGATION_REQUIRED ||
            definition.getPropagationBehavior() == TransactionDefinition.PROPAGATION_REQUIRES_NEW ||
            definition.getPropagationBehavior() == TransactionDefinition.PROPAGATION_NESTED) {
        //空挂起
        SuspendedResourcesHolder suspendedResources = suspend(null);
        if (debugEnabled) {
            logger.debug("Creating new transaction with name [" + definition.getName() + "]: " + definition);
        }
        try {
            //默认返回true
            boolean newSynchronization = (getTransactionSynchronization() != SYNCHRONIZATION_NEVER);
            //构建事务状态
            DefaultTransactionStatus status = newTransactionStatus(
                    definition, transaction, true, newSynchronization, debugEnabled, suspendedResources);
            //构造transaction、包括设置connectionHolder、隔离级别、timeout
            //如果是新事务，绑定到当前线程
            doBegin(transaction, definition);
            //新事务同步设置，针对当前线程
            prepareSynchronization(status, definition);
            return status;
        }
        catch (RuntimeException ex) {
            resume(null, suspendedResources);
            throw ex;
        }
        catch (Error err) {
            resume(null, suspendedResources);
            throw err;
        }
    }
    else {
        // Create "empty" transaction: no actual transaction, but potentially synchronization.
        if (definition.getIsolationLevel() != TransactionDefinition.ISOLATION_DEFAULT && logger.isWarnEnabled()) {
            logger.warn("Custom isolation level specified but no actual transaction initiated; " +
                    "isolation level will effectively be ignored: " + definition);
        }
        //声明事务是PROPAGATION_SUPPORTS
        boolean newSynchronization = (getTransactionSynchronization() == SYNCHRONIZATION_ALWAYS);
        return prepareTransactionStatus(definition, null, true, newSynchronization, debugEnabled, null);
    }
}
```
## 创建新事务流程
1. 获取事务
创建对应的事务实例，这里使用的是DataSourceTransactionManager中的doGetTransaction方法，创建基于JDBC的事务实例，如果当前线程中存在关于dataSoruce的连接，那么直接使用。这里有一个对保存点的设置，是否开启允许保存点取决于是否设置了允许嵌入式事务。DataSourceTransactionManager默认是开启的。
```java
@Override
protected Object doGetTransaction() {
    DataSourceTransactionObject txObject = new DataSourceTransactionObject();
    txObject.setSavepointAllowed(isNestedTransactionAllowed());
    //从ThreadLocal中获取当前ConnectionHolder
    ConnectionHolder conHolder =
            (ConnectionHolder) TransactionSynchronizationManager.getResource(this.dataSource);
    txObject.setConnectionHolder(conHolder, false);
    return txObject;
}
```
2. 如果当先线程存在事务，则转向嵌套的事务处理。是否存在事务在DataSourceTransactionManager的isExistingTransaction方法中
```java
@Override
protected boolean isExistingTransaction(Object transaction) {
    DataSourceTransactionObject txObject = (DataSourceTransactionObject) transaction;
    return (txObject.hasConnectionHolder() && txObject.getConnectionHolder().isTransactionActive());
}
```
3. 事务超时设置验证
4. 事务PropagationBehavior属性的设置验证
5. 构建DefaultTransactionStatus。
6. 完善transaction，包括设置connectionHolder、隔离级别、timeout，如果是新事务，绑定到当前线程
对于一些隔离级别、timeout等功能的设置并不是由Spring来完成，而是委拖给底层的数据库连接去做的，而对于数据库连接的设置就是在doBegin函数中处理。
```java
@Override
protected void doBegin(Object transaction, TransactionDefinition definition) {
    DataSourceTransactionObject txObject = (DataSourceTransactionObject) transaction;
    Connection con = null;

    try {
        if (!txObject.hasConnectionHolder() ||
                txObject.getConnectionHolder().isSynchronizedWithTransaction()) {
            //不存在数据库连接，从连接池中获取连接
            Connection newCon = this.dataSource.getConnection();
            if (logger.isDebugEnabled()) {
                logger.debug("Acquired Connection [" + newCon + "] for JDBC transaction");
            }
            txObject.setConnectionHolder(new ConnectionHolder(newCon), true);
        }

        txObject.getConnectionHolder().setSynchronizedWithTransaction(true);
        con = txObject.getConnectionHolder().getConnection();

        //设置隔离级别
        Integer previousIsolationLevel = DataSourceUtils.prepareConnectionForTransaction(con, definition);
        txObject.setPreviousIsolationLevel(previousIsolationLevel);

        // Switch to manual commit if necessary. This is very expensive in some JDBC drivers,
        // so we don't want to do it unnecessarily (for example if we've explicitly
        // configured the connection pool to set it already).
        //更新自动提交设置，由spring控制提交
        if (con.getAutoCommit()) {
            txObject.setMustRestoreAutoCommit(true);
            if (logger.isDebugEnabled()) {
                logger.debug("Switching JDBC Connection [" + con + "] to manual commit");
            }
            con.setAutoCommit(false);
        }

        prepareTransactionalConnection(con, definition);
        //判断是否存在事务的依据
        txObject.getConnectionHolder().setTransactionActive(true);

        //设置超时时间
        int timeout = determineTimeout(definition);
        if (timeout != TransactionDefinition.TIMEOUT_DEFAULT) {
            txObject.getConnectionHolder().setTimeoutInSeconds(timeout);
        }

        // Bind the connection holder to the thread.
        if (txObject.isNewConnectionHolder()) {
            TransactionSynchronizationManager.bindResource(getDataSource(), txObject.getConnectionHolder());
        }
    }

    catch (Throwable ex) {
        if (txObject.isNewConnectionHolder()) {
            DataSourceUtils.releaseConnection(con, this.dataSource);
            txObject.setConnectionHolder(null, false);
        }
        throw new CannotCreateTransactionException("Could not open JDBC Connection for transaction", ex);
    }
}
```
可以说事务是从这个函数开始的，因为这个函数已经开始尝试了对数据库连接的获取，当然，在获取数据库连接的同时，一些必要的设置也是需要同步设置的。
- 尝试获取连接
当然并不是每次都会获取新的连接，如果当前线程中的ConnectionHolder已经存在，则没有必要再获取，或者，对于事务同步表示设置为true的需要重新获取连接。
- 设置隔离级别和只读标识
隔离级别和只读标识是交给数据库的connection去处理，spring并没有进行处理
- 更改默认的提交设置
- 设置标志位，标识当前连接已经被事务激活。
- 设置超时时间
- 将connectionHolder绑定到当前线程
7. 将事务信息记录在当前线程中
```java
protected void prepareSynchronization(DefaultTransactionStatus status, TransactionDefinition definition) {
    //新的事务
    if (status.isNewSynchronization()) {
        TransactionSynchronizationManager.setActualTransactionActive(status.hasTransaction());
        TransactionSynchronizationManager.setCurrentTransactionIsolationLevel(
                definition.getIsolationLevel() != TransactionDefinition.ISOLATION_DEFAULT ?
                        definition.getIsolationLevel() : null);
        TransactionSynchronizationManager.setCurrentTransactionReadOnly(definition.isReadOnly());
        TransactionSynchronizationManager.setCurrentTransactionName(definition.getName());
        TransactionSynchronizationManager.initSynchronization();
    }
}
```
## 嵌套事务处理
在[十六、spring事务之事务传播机制和隔离级别](https://www.jianshu.com/p/aa76625d3715)讲了Spring事务传播行为，那么对于已存在的事务，Spring是如何处理的呢？
```java
if (isExistingTransaction(transaction)) {
    // Existing transaction found -> check propagation behavior to find out how to behave.
    return handleExistingTransaction(definition, transaction, debugEnabled);
}

/**
* DataSourceTransactionManager#isExistingTransaction
*/
@Override
protected boolean isExistingTransaction(Object transaction) {
    //存在连接并且连接有事务
    DataSourceTransactionObject txObject = (DataSourceTransactionObject) transaction;
    return (txObject.hasConnectionHolder() && txObject.getConnectionHolder().isTransactionActive());
}
```
上面的代码是判断当前线程是否存在事务，如果存在，则执行handleExistingTransaction方法
```java
private TransactionStatus handleExistingTransaction(
        TransactionDefinition definition, Object transaction, boolean debugEnabled)
        throws TransactionException {
    //PROPAGATION_NEVER，直接抛异常
    if (definition.getPropagationBehavior() == TransactionDefinition.PROPAGATION_NEVER) {
        throw new IllegalTransactionStateException(
                "Existing transaction found for transaction marked with propagation 'never'");
    }

    //PROPAGATION_NOT_SUPPORTED, 挂起当前事务，返回空的事务
    if (definition.getPropagationBehavior() == TransactionDefinition.PROPAGATION_NOT_SUPPORTED) {
        if (debugEnabled) {
            logger.debug("Suspending current transaction");
        }
        Object suspendedResources = suspend(transaction);
        boolean newSynchronization = (getTransactionSynchronization() == SYNCHRONIZATION_ALWAYS);
        return prepareTransactionStatus(
                definition, null, false, newSynchronization, debugEnabled, suspendedResources);
    }

    //PROPAGATION_REQUIRES_NEW,挂起当前事务，重新创建一个新的事务
    if (definition.getPropagationBehavior() == TransactionDefinition.PROPAGATION_REQUIRES_NEW) {
        if (debugEnabled) {
            logger.debug("Suspending current transaction, creating new transaction with name [" +
                    definition.getName() + "]");
        }
        SuspendedResourcesHolder suspendedResources = suspend(transaction);
        try {
            boolean newSynchronization = (getTransactionSynchronization() != SYNCHRONIZATION_NEVER);
            DefaultTransactionStatus status = newTransactionStatus(
                    definition, transaction, true, newSynchronization, debugEnabled, suspendedResources);
            doBegin(transaction, definition);
            prepareSynchronization(status, definition);
            return status;
        }
        catch (RuntimeException beginEx) {
            resumeAfterBeginException(transaction, suspendedResources, beginEx);
            throw beginEx;
        }
        catch (Error beginErr) {
            resumeAfterBeginException(transaction, suspendedResources, beginErr);
            throw beginErr;
        }
    }

    //PROPAGATION_NESTED， 嵌套事务
    if (definition.getPropagationBehavior() == TransactionDefinition.PROPAGATION_NESTED) {
        if (!isNestedTransactionAllowed()) {
            throw new NestedTransactionNotSupportedException(
                    "Transaction manager does not allow nested transactions by default - " +
                    "specify 'nestedTransactionAllowed' property with value 'true'");
        }
        if (debugEnabled) {
            logger.debug("Creating nested transaction with name [" + definition.getName() + "]");
        }
        if (useSavepointForNestedTransaction()) {
            // Create savepoint within existing Spring-managed transaction,
            // through the SavepointManager API implemented by TransactionStatus.
            // Usually uses JDBC 3.0 savepoints. Never activates Spring synchronization.
            DefaultTransactionStatus status =
                    prepareTransactionStatus(definition, transaction, false, false, debugEnabled, null);
            status.createAndHoldSavepoint();
            return status;
        }
        else {
            // Nested transaction through nested begin and commit/rollback calls.
            // Usually only for JTA: Spring synchronization might get activated here
            // in case of a pre-existing JTA transaction.
            boolean newSynchronization = (getTransactionSynchronization() != SYNCHRONIZATION_NEVER);
            DefaultTransactionStatus status = newTransactionStatus(
                    definition, transaction, true, newSynchronization, debugEnabled, null);
            doBegin(transaction, definition);
            prepareSynchronization(status, definition);
            return status;
        }
    }

    // Assumably PROPAGATION_SUPPORTS or PROPAGATION_REQUIRED.
    if (debugEnabled) {
        logger.debug("Participating in existing transaction");
    }
    //isValidateExistingTransaction()返回false
    if (isValidateExistingTransaction()) {
        if (definition.getIsolationLevel() != TransactionDefinition.ISOLATION_DEFAULT) {
            Integer currentIsolationLevel = TransactionSynchronizationManager.getCurrentTransactionIsolationLevel();
            if (currentIsolationLevel == null || currentIsolationLevel != definition.getIsolationLevel()) {
                Constants isoConstants = DefaultTransactionDefinition.constants;
                throw new IllegalTransactionStateException("Participating transaction with definition [" +
                        definition + "] specifies isolation level which is incompatible with existing transaction: " +
                        (currentIsolationLevel != null ?
                                isoConstants.toCode(currentIsolationLevel, DefaultTransactionDefinition.PREFIX_ISOLATION) :
                                "(unknown)"));
            }
        }
        if (!definition.isReadOnly()) {
            if (TransactionSynchronizationManager.isCurrentTransactionReadOnly()) {
                throw new IllegalTransactionStateException("Participating transaction with definition [" +
                        definition + "] is not marked as read-only but existing transaction is");
            }
        }
    }
    //处理PROPAGATION_SUPPORTS or PROPAGATION_REQUIRED，直接使用已有事务
    boolean newSynchronization = (getTransactionSynchronization() != SYNCHRONIZATION_NEVER);
    return prepareTransactionStatus(definition, transaction, false, newSynchronization, debugEnabled, null);
}
```
对于已存在事务的处理，Spring处理流程如下:
1. 若传播行为是PROPAGATION_NEVER，则直接抛异常
2. 若传播行为是PROPAGATION_NOT_SUPPORTED, 挂起当前事务，返回空的事务
3. 若传播行为是PROPAGATION_REQUIRES_NEW,挂起当前事务，重新创建一个新的事务
4. 若传播行为是PROPAGATION_NESTED， 嵌套事务会先判断事务管理器是否支持嵌套，不支持直接抛出异常，然后判断是否能够使用安全点，如果不能使用处理逻辑和PROPAGATION_REQUIRES_NEW一样，不过不会挂起当前事务。
5. 若传播行为是PROPAGATION_SUPPORTS 或 PROPAGATION_REQUIRED，直接使用已有事务， 把事务标记成非新创建事务。
## 挂起事务
事务的处理还有个重要的点就是挂起当前事务，我们看下挂起逻辑:
```java
protected final SuspendedResourcesHolder suspend(Object transaction) throws TransactionException {
    if (TransactionSynchronizationManager.isSynchronizationActive()) {
        List<TransactionSynchronization> suspendedSynchronizations = doSuspendSynchronization();
        try {
            Object suspendedResources = null;
            if (transaction != null) {
                suspendedResources = doSuspend(transaction);
            }
            String name = TransactionSynchronizationManager.getCurrentTransactionName();
            TransactionSynchronizationManager.setCurrentTransactionName(null);
            boolean readOnly = TransactionSynchronizationManager.isCurrentTransactionReadOnly();
            TransactionSynchronizationManager.setCurrentTransactionReadOnly(false);
            Integer isolationLevel = TransactionSynchronizationManager.getCurrentTransactionIsolationLevel();
            TransactionSynchronizationManager.setCurrentTransactionIsolationLevel(null);
            boolean wasActive = TransactionSynchronizationManager.isActualTransactionActive();
            TransactionSynchronizationManager.setActualTransactionActive(false);
            return new SuspendedResourcesHolder(
                    suspendedResources, suspendedSynchronizations, name, readOnly, isolationLevel, wasActive);
        }
        catch (RuntimeException ex) {
            // doSuspend failed - original transaction is still active...
            doResumeSynchronization(suspendedSynchronizations);
            throw ex;
        }
        catch (Error err) {
            // doSuspend failed - original transaction is still active...
            doResumeSynchronization(suspendedSynchronizations);
            throw err;
        }
    }
    else if (transaction != null) {
        // Transaction active but no synchronization active.
        Object suspendedResources = doSuspend(transaction);
        return new SuspendedResourcesHolder(suspendedResources);
    }
    else {
        // Neither transaction nor synchronization active.
        return null;
    }
}
```
挂起的逻辑是把当前线程中的值读取出来然后存到SuspendedResourcesHolder对象中，把当前线程中的信息还原，并把SuspendedResourcesHolder封装到TransactionStatus中。
# 保存事务信息
到此为止，所有的事务都已经创建完成，我们需要将所有的事务信息统一记录在TransactionInfo类型的实例中，这个实例包含了目标方法开始前的所有状态信息，一旦事务执行失败，Spring可以通过TransactionInfo类型的实例中的信息来进行回滚等后续工作。
```java
protected TransactionInfo prepareTransactionInfo(PlatformTransactionManager tm,
        TransactionAttribute txAttr, String joinpointIdentification, TransactionStatus status) {

    TransactionInfo txInfo = new TransactionInfo(tm, txAttr, joinpointIdentification);
    if (txAttr != null) {
        // We need a transaction for this method...
        if (logger.isTraceEnabled()) {
            logger.trace("Getting transaction for [" + txInfo.getJoinpointIdentification() + "]");
        }
        // The transaction manager will flag an error if an incompatible tx already exists.
        txInfo.newTransactionStatus(status);
    }
    else {
        // The TransactionInfo.hasTransaction() method will return false. We created it only
        // to preserve the integrity of the ThreadLocal stack maintained in this class.
        if (logger.isTraceEnabled())
            logger.trace("Don't need to create transaction for [" + joinpointIdentification +
                    "]: This method isn't transactional.");
    }

    // We always bind the TransactionInfo to the thread, even if we didn't create
    // a new transaction here. This guarantees that the TransactionInfo stack
    // will be managed correctly even if no transaction was created by this aspect.
    //记录老的事务信息，保存新的
    txInfo.bindToThread();
    return txInfo;
}
```