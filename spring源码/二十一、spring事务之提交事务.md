之前我们分析了Spring的事务异常处理机制，那么当我们的业务处理正常，Spring又是如何帮我们提交事务的呢？
```java
protected void commitTransactionAfterReturning(TransactionInfo txInfo) {
    if (txInfo != null && txInfo.hasTransaction()) {
        if (logger.isTraceEnabled()) {
            logger.trace("Completing transaction for [" + txInfo.getJoinpointIdentification() + "]");
        }
        txInfo.getTransactionManager().commit(txInfo.getTransactionStatus());
    }
}
```
# 事务提交
```java
@Override
public final void commit(TransactionStatus status) throws TransactionException {
    //1. 判断事务是不是已经完成
    if (status.isCompleted()) {
        throw new IllegalTransactionStateException(
                "Transaction is already completed - do not call commit or rollback more than once per transaction");
    }

    DefaultTransactionStatus defStatus = (DefaultTransactionStatus) status;
    //2. 如果在事务链中已经被标记回滚，那么不会尝试提交事务，直接回滚，不过我没找到在哪设置这个值
    if (defStatus.isLocalRollbackOnly()) {
        if (defStatus.isDebug()) {
            logger.debug("Transactional code has requested rollback");
        }
        processRollback(defStatus);
        return;
    }
    //3. shouldCommitOnGlobalRollbackOnly()默认返回false，isGlobalRollbackOnly是在嵌入事务回滚的时候赋值的
    if (!shouldCommitOnGlobalRollbackOnly() && defStatus.isGlobalRollbackOnly()) {
        if (defStatus.isDebug()) {
            logger.debug("Global transaction is marked as rollback-only but transactional code requested commit");
        }
        processRollback(defStatus);
        // Throw UnexpectedRollbackException only at outermost transaction boundary
        // or if explicitly asked to.
        if (status.isNewTransaction() || isFailEarlyOnGlobalRollbackOnly()) {
            throw new UnexpectedRollbackException(
                    "Transaction rolled back because it has been marked as rollback-only");
        }
        return;
    }
    //4. 提交事务
    processCommit(defStatus);
}
```
事务提交的逻辑如下:
1. 判断事务是否已经完成，如果完成抛出异常
2. 判断事务是否已经被标记成回滚，则执行回滚操作，不过这步我没找到在哪设置值。
3. 嵌入事务标记回滚，如果嵌入事务抛出了异常执行了回滚，但是在调用方把嵌入事务的异常个捕获没有抛出，就会执行这一步。
```java
@Override
public boolean isGlobalRollbackOnly() {
    return ((this.transaction instanceof SmartTransactionObject) &&
            ((SmartTransactionObject) this.transaction).isRollbackOnly());
}
```
4. 提交事务
```java
private void processCommit(DefaultTransactionStatus status) throws TransactionException {
    try {
        boolean beforeCompletionInvoked = false;
        try {
            prepareForCommit(status);
            triggerBeforeCommit(status);
            triggerBeforeCompletion(status);
            beforeCompletionInvoked = true;
            boolean globalRollbackOnly = false;
            if (status.isNewTransaction() || isFailEarlyOnGlobalRollbackOnly()) {
                globalRollbackOnly = status.isGlobalRollbackOnly();
            }
            //1. 如果有保存点则清除保存点信息
            if (status.hasSavepoint()) {
                if (status.isDebug()) {
                    logger.debug("Releasing transaction savepoint");
                }
                status.releaseHeldSavepoint();
            }
            //2. 如果是新事务，提交事务
            else if (status.isNewTransaction()) {
                if (status.isDebug()) {
                    logger.debug("Initiating transaction commit");
                }
                doCommit(status);
            }
            // Throw UnexpectedRollbackException if we have a global rollback-only
            // marker but still didn't get a corresponding exception from commit.
            if (globalRollbackOnly) {
                throw new UnexpectedRollbackException(
                        "Transaction silently rolled back because it has been marked as rollback-only");
            }
        }
        catch (UnexpectedRollbackException ex) {
            // can only be caused by doCommit
            triggerAfterCompletion(status, TransactionSynchronization.STATUS_ROLLED_BACK);
            throw ex;
        }
        catch (TransactionException ex) {
            // can only be caused by doCommit
            if (isRollbackOnCommitFailure()) {
                doRollbackOnCommitException(status, ex);
            }
            else {
                triggerAfterCompletion(status, TransactionSynchronization.STATUS_UNKNOWN);
            }
            throw ex;
        }
        catch (RuntimeException ex) {
            if (!beforeCompletionInvoked) {
                triggerBeforeCompletion(status);
            }
            doRollbackOnCommitException(status, ex);
            throw ex;
        }
        catch (Error err) {
            if (!beforeCompletionInvoked) {
                triggerBeforeCompletion(status);
            }
            doRollbackOnCommitException(status, err);
            throw err;
        }

        // Trigger afterCommit callbacks, with an exception thrown there
        // propagated to callers but the transaction still considered as committed.
        try {
            triggerAfterCommit(status);
        }
        finally {
            triggerAfterCompletion(status, TransactionSynchronization.STATUS_COMMITTED);
        }

    }
    finally {
        //清空记录的资源并将挂起的资源恢复
        cleanupAfterCompletion(status);
    }
}
```
这一步的执行流程是：
(1) 触发提交事务前的处理
(2) 判断是否有保存点，如果有清除保存点
(3) 如果是新事务，则提交事务，提交事务的操作还是交给数据库处理.
```java
@Override
protected void doCommit(DefaultTransactionStatus status) {
    DataSourceTransactionObject txObject = (DataSourceTransactionObject) status.getTransaction();
    Connection con = txObject.getConnectionHolder().getConnection();
    if (status.isDebug()) {
        logger.debug("Committing JDBC transaction on Connection [" + con + "]");
    }
    try {
        con.commit();
    }
    catch (SQLException ex) {
        throw new TransactionSystemException("Could not commit JDBC transaction", ex);
    }
}
```
(4) 和回滚事务一样，清空记录的资源并将挂起的资源恢复
