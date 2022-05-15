

@[toc]
## AT
![在这里插入图片描述](https://img-blog.csdnimg.cn/d023764264cd41f4899b2ebe8fce3077.png)
- 模块：
	- Transaction Coordinator (TC)：事务协调器，维护全局事务的运行状态，负责协调并驱动全局事务的提交或回滚。
	- Transaction Manager(TM) ：控制全局事务的边界，负责开启一个全局事务，并最终发起全局提交或全局回滚的决议。
	- Resource Manager (RM)：控制分支事务，负责分支注册、状态汇报，并接收事务协调器的指令，驱动分支（本地）事务的提交和回滚。
- 流程，两阶段提交协议的演变：
	- 一阶段：业务数据和回滚日志记录在同一个本地事务中提交，释放本地锁和连接资源。
	- 二阶段：提交异步化，非常快速地完成。回滚通过一阶段的回滚日志进行反向补偿。

### TM
> 一句话实现：@GlobalTransactional注解的类通过动态代理，封装开启、提交、回滚事务的逻辑

#### 全局事务入口
- 带注解的业务方法
```java
    @Override
    @GlobalTransactional(timeoutMills = 30000000, name = "dubbo-demo-tx")
    public void purchase(String userId, String commodityCode, int orderCount) {
        LOGGER.info("purchase begin ... xid: " + RootContext.getXID());
        stockService.deduct(commodityCode, orderCount);
        orderService.create(userId, commodityCode, orderCount);
        if (mockException()) {
            throw new RuntimeException("random exception mock!");
        }
        LOGGER.info("purchase end ... xid: " + RootContext.getXID());
    }
```

- 增强业务类的拦截器为io.seata.spring.annotation.GlobalTransactionalInterceptor
```java
public class GlobalTransactionalInterceptor implements ConfigurationChangeListener, MethodInterceptor {
    @Override
    public Object invoke(final MethodInvocation methodInvocation) throws Throwable {
        Class<?> targetClass =
            methodInvocation.getThis() != null ? AopUtils.getTargetClass(methodInvocation.getThis()) : null;
        Method specificMethod = ClassUtils.getMostSpecificMethod(methodInvocation.getMethod(), targetClass);
        if (specificMethod != null && !specificMethod.getDeclaringClass().equals(Object.class)) {
            final Method method = BridgeMethodResolver.findBridgedMethod(specificMethod);
            final GlobalTransactional globalTransactionalAnnotation =
                getAnnotation(method, targetClass, GlobalTransactional.class);
            final GlobalLock globalLockAnnotation = getAnnotation(method, targetClass, GlobalLock.class);
            boolean localDisable = disable || (degradeCheck && degradeNum >= degradeCheckAllowTimes);
            if (!localDisable) {
                if (globalTransactionalAnnotation != null) {
					//增强全局事务
                    return handleGlobalTransaction(methodInvocation, globalTransactionalAnnotation);
                } else if (globalLockAnnotation != null) {
                    return handleGlobalLock(methodInvocation, globalLockAnnotation);
                }
            }
        }
        return methodInvocation.proceed();
    }
}
```

#### 协调全局事务
- 核心逻辑封装在transactionalTemplate中

```java
    Object handleGlobalTransaction(final MethodInvocation methodInvocation,
        final GlobalTransactional globalTrxAnno) throws Throwable {
        boolean succeed = true;
        try {
			//事务模板
            return transactionalTemplate.execute(new TransactionalExecutor() {
                @Override
                public Object execute() throws Throwable {
                    return methodInvocation.proceed();
                }

	     		......
            });
        } catch (TransactionalExecutor.ExecutionException e) {
            TransactionalExecutor.Code code = e.getCode();
            switch (code) {
                case RollbackDone:
                    throw e.getOriginalException();
	              ......
                default:
                    throw new ShouldNeverHappenException(String.format("Unknown TransactionalExecutor.Code: %s", code));
            }
        } finally {
            if (degradeCheck) {
                EVENT_BUS.post(new DegradeCheckEvent(succeed));
            }
        }
    }
```

- 事务模板的细节
```java
public class TransactionalTemplate {

    public Object execute(TransactionalExecutor business) throws Throwable {
        // 1. Get transactionInfo
        TransactionInfo txInfo = business.getTransactionInfo();
        if (txInfo == null) {
            throw new ShouldNeverHappenException("transactionInfo does not exist");
        }
        // 1.1 Get current transaction, if not null, the tx role is 'GlobalTransactionRole.Participant'.
        GlobalTransaction tx = GlobalTransactionContext.getCurrent();

        // 1.2 Handle the transaction propagation.
        Propagation propagation = txInfo.getPropagation();
        SuspendedResourcesHolder suspendedResourcesHolder = null;
        try {
            switch (propagation) {
	             ......
                case REQUIRED:
                    // If current transaction is existing, execute with current transaction,
                    // else continue and execute with new transaction.
					//默认为加入当前事务，如果没有事务则创建
                    break;
                ......
                default:
                    throw new TransactionException("Not Supported Propagation:" + propagation);
            }

            // 1.3 If null, create new transaction with role 'GlobalTransactionRole.Launcher'.
            if (tx == null) {
                tx = GlobalTransactionContext.createNew();
            }

            // set current tx config to holder
            GlobalLockConfig previousConfig = replaceGlobalLockConfig(txInfo);

            try {
                // 2. If the tx role is 'GlobalTransactionRole.Launcher', send the request of beginTransaction to TC,
                //    else do nothing. Of course, the hooks will still be triggered.
				//开启事务
                beginTransaction(txInfo, tx);

                Object rs;
                try {
                    // 业务逻辑
                    rs = business.execute();
                } catch (Throwable ex) {
                    // 3. 回滚事务
                    completeTransactionAfterThrowing(txInfo, tx, ex);
                    throw ex;
                }

                // 4. 提交事务
                commitTransaction(tx);

                return rs;
            } finally {
                //5. clear
                resumeGlobalLockConfig(previousConfig);
                triggerAfterCompletion();
                cleanUp();
            }
        } finally {
            // If the transaction is suspended, resume it.
            if (suspendedResourcesHolder != null) {
                tx.resume(suspendedResourcesHolder);
            }
        }
    }
}
```

- 开启事务
```java
public class TransactionalTemplate {
    private void beginTransaction(TransactionInfo txInfo, GlobalTransaction tx) throws TransactionalExecutor.ExecutionException {
        try {
			//前置增强
            triggerBeforeBegin();
			//开启事务
            tx.begin(txInfo.getTimeOut(), txInfo.getName());
			//后置增强
            triggerAfterBegin();
        } catch (TransactionException txe) {
            throw new TransactionalExecutor.ExecutionException(tx, txe,
                TransactionalExecutor.Code.BeginFailure);

        }
    }
}

public class DefaultGlobalTransaction implements GlobalTransaction {
   @Override
    public void begin(int timeout, String name) throws TransactionException {
        ......
        String currentXid = RootContext.getXID();
        if (currentXid != null) {
            throw new IllegalStateException("Global transaction already exists," +
                " can't begin a new global transaction, currentXid = " + currentXid);
        }
		//开启事务在TransactionManager的实现类中，默认为DefaultTransactionManager
        xid = transactionManager.begin(null, null, name, timeout);
        status = GlobalStatus.Begin;
        RootContext.bind(xid);
        if (LOGGER.isInfoEnabled()) {
            LOGGER.info("Begin new global transaction [{}]", xid);
        }
    }
}

public class DefaultTransactionManager implements TransactionManager {
    @Override
    public String begin(String applicationId, String transactionServiceGroup, String name, int timeout)
        throws TransactionException {
        GlobalBeginRequest request = new GlobalBeginRequest();
        request.setTransactionName(name);
        request.setTimeout(timeout);
		//同步通知tc事务开启，netty实现类为AbstractNettyRemotingClient
        GlobalBeginResponse response = (GlobalBeginResponse) syncCall(request);
        if (response.getResultCode() == ResultCode.Failed) {
            throw new TmTransactionException(TransactionExceptionCode.BeginFailed, response.getMsg());
        }
        return response.getXid();
    }
    private AbstractTransactionResponse syncCall(AbstractTransactionRequest request) throws TransactionException {
        try {
            return (AbstractTransactionResponse) TmNettyRemotingClient.getInstance().sendSyncRequest(request);
        } catch (TimeoutException toe) {
            throw new TmTransactionException(TransactionExceptionCode.IO, "RPC timeout", toe);
        }
    }
}

```

- 提交事务，实现类同开启事务
```java
public class TransactionalTemplate {
    private void commitTransaction(GlobalTransaction tx) throws TransactionalExecutor.ExecutionException {
        try {
            triggerBeforeCommit();
			//提交事务
            tx.commit();
            triggerAfterCommit();
        } catch (TransactionException txe) {
            // 4.1 Failed to commit
            throw new TransactionalExecutor.ExecutionException(tx, txe,
                TransactionalExecutor.Code.CommitFailure);
        }
    }
}

public class DefaultGlobalTransaction implements GlobalTransaction {
    @Override
    public void commit() throws TransactionException {
        ......
        int retry = COMMIT_RETRY_COUNT <= 0 ? DEFAULT_TM_COMMIT_RETRY_COUNT : COMMIT_RETRY_COUNT;
        try {
            while (retry > 0) {
                try {
					//实现在TransactionManager中
                    status = transactionManager.commit(xid);
                    break;
                } catch (Throwable ex) {
                    LOGGER.error("Failed to report global commit [{}],Retry Countdown: {}, reason: {}", this.getXid(), retry, ex.getMessage());
                    retry--;
                    if (retry == 0) {
                        throw new TransactionException("Failed to report global commit", ex);
                    }
                }
            }
        } finally {
            if (xid.equals(RootContext.getXID())) {
                suspend();
            }
        }
        if (LOGGER.isInfoEnabled()) {
            LOGGER.info("[{}] commit status: {}", xid, status);
        }
    }
}

public class DefaultTransactionManager implements TransactionManager {
    @Override
    public GlobalStatus commit(String xid) throws TransactionException {
        GlobalCommitRequest globalCommit = new GlobalCommitRequest();
        globalCommit.setXid(xid);
		//同步通知TC
        GlobalCommitResponse response = (GlobalCommitResponse) syncCall(globalCommit);
        return response.getGlobalStatus();
    }
}

```

- 回滚事务，实现类同开启事务
```java
public class TransactionalTemplate {
    private void completeTransactionAfterThrowing(TransactionInfo txInfo, GlobalTransaction tx, Throwable originalException) throws TransactionalExecutor.ExecutionException {
        //根据异常和回滚规则，判断是否需要回滚，如果无须回滚，直接提交
		//默认情况无回滚规则，直接回滚
		if (txInfo != null && txInfo.rollbackOn(originalException)) {
            try {
                rollbackTransaction(tx, originalException);
            } catch (TransactionException txe) {
                // Failed to rollback
                throw new TransactionalExecutor.ExecutionException(tx, txe,
                        TransactionalExecutor.Code.RollbackFailure, originalException);
            }
        } else {
            // not roll back on this exception, so commit
            commitTransaction(tx);
        }
    }
    private void rollbackTransaction(GlobalTransaction tx, Throwable originalException) throws TransactionException, TransactionalExecutor.ExecutionException {
        triggerBeforeRollback();
		//回滚
        tx.rollback();
        triggerAfterRollback();
        // 3.1 Successfully rolled back
        throw new TransactionalExecutor.ExecutionException(tx, GlobalStatus.RollbackRetrying.equals(tx.getLocalStatus())
            ? TransactionalExecutor.Code.RollbackRetrying : TransactionalExecutor.Code.RollbackDone, originalException);
    }

}

public class DefaultGlobalTransaction implements GlobalTransaction {
   @Override
    public void rollback() throws TransactionException {
		 ......

        int retry = ROLLBACK_RETRY_COUNT <= 0 ? DEFAULT_TM_ROLLBACK_RETRY_COUNT : ROLLBACK_RETRY_COUNT;
        try {
            while (retry > 0) {
                try {
					//实现在TransactionManager中
                    status = transactionManager.rollback(xid);
                    break;
                } catch (Throwable ex) {
                    LOGGER.error("Failed to report global rollback [{}],Retry Countdown: {}, reason: {}", this.getXid(), retry, ex.getMessage());
                    retry--;
                    if (retry == 0) {
                        throw new TransactionException("Failed to report global rollback", ex);
                    }
                }
            }
        } finally {
            if (xid.equals(RootContext.getXID())) {
                suspend();
            }
        }
    }
}

public class DefaultTransactionManager implements TransactionManager {
    @Override
    public GlobalStatus rollback(String xid) throws TransactionException {
        GlobalRollbackRequest globalRollback = new GlobalRollbackRequest();
        globalRollback.setXid(xid);
		//通知TC回滚事务
        GlobalRollbackResponse response = (GlobalRollbackResponse) syncCall(globalRollback);
        return response.getGlobalStatus();
    }
}

```

### RM
> 一句话实现：DataSourceProxy 对 JDBC 标准接口操作代理增强，封装undo和回滚，二阶段提交异步化。

#### 一阶段提交本地事务
- 执行业务sql
```java
    @Override
    public void deduct(String commodityCode, int count) {
        LOGGER.info("Stock Service Begin ... xid: " + RootContext.getXID());
        LOGGER.info("Deducting inventory SQL: update stock_tbl set count = count - {} where commodity_code = {}", count,
            commodityCode);

        jdbcTemplate.update("update stock_tbl set count = count - ? where commodity_code = ?",
            new Object[] {count, commodityCode});
        LOGGER.info("Stock Service End ... ");

    }
```

- 执行入口为spring的jdbcTemplate
```java
public class JdbcTemplate extends JdbcAccessor implements JdbcOperations {
	@Override
	public int update(String sql, @Nullable Object... args) throws DataAccessException {
		return update(sql, newArgPreparedStatementSetter(args));
	}
	@Override
	public int update(String sql, @Nullable PreparedStatementSetter pss) throws DataAccessException {
		return update(new SimplePreparedStatementCreator(sql), pss);
	}
	protected int update(final PreparedStatementCreator psc, @Nullable final PreparedStatementSetter pss)
			throws DataAccessException {
		return updateCount(execute(psc, ps -> {
			try {
				if (pss != null) {
					pss.setValues(ps);
				}
				//执行update语句
				int rows = ps.executeUpdate();
				if (logger.isTraceEnabled()) {
					logger.trace("SQL update affected " + rows + " rows");
				}
				return rows;
			}
			finally {
				if (pss instanceof ParameterDisposer) {
					((ParameterDisposer) pss).cleanupParameters();
				}
			}
		}));
	}
```

- 执行时走seata实现的preparestatement代理
```java
public class PreparedStatementProxy extends AbstractPreparedStatementProxy
    implements PreparedStatement, ParametersHolder {
    @Override
    public int executeUpdate() throws SQLException {
        return ExecuteTemplate.execute(this, (statement, args) -> statement.executeUpdate());
    }
}

public class ExecuteTemplate {
    public static <T, S extends Statement> T execute(StatementProxy<S> statementProxy,
                                                     StatementCallback<T, S> statementCallback,
                                                     Object... args) throws SQLException {
        return execute(null, statementProxy, statementCallback, args);
    }

   public static <T, S extends Statement> T execute(List<SQLRecognizer> sqlRecognizers,
                                                     StatementProxy<S> statementProxy,
                                                     StatementCallback<T, S> statementCallback,
                                                     Object... args) throws SQLException {
        if (!RootContext.requireGlobalLock() && BranchType.AT != RootContext.getBranchType()) {
            // Just work as original statement
            return statementCallback.execute(statementProxy.getTargetStatement(), args);
        }

        String dbType = statementProxy.getConnectionProxy().getDbType();
		//获取sql识别器，用于解析sql
        if (CollectionUtils.isEmpty(sqlRecognizers)) {
            sqlRecognizers = SQLVisitorFactory.get(
                    statementProxy.getTargetSQL(),
                    dbType);
        }
        Executor<T> executor;
        if (CollectionUtils.isEmpty(sqlRecognizers)) {
            executor = new PlainExecutor<>(statementProxy, statementCallback);
        } else {
            if (sqlRecognizers.size() == 1) {
                SQLRecognizer sqlRecognizer = sqlRecognizers.get(0);
                switch (sqlRecognizer.getSQLType()) {
                    case INSERT:
                        executor = EnhancedServiceLoader.load(InsertExecutor.class, dbType,
                                new Class[]{StatementProxy.class, StatementCallback.class, SQLRecognizer.class},
                                new Object[]{statementProxy, statementCallback, sqlRecognizer});
                        break;
                    case UPDATE:
						//获取update语句的执行器
                        executor = new UpdateExecutor<>(statementProxy, statementCallback, sqlRecognizer);
                        break;
                    case DELETE:
                        executor = new DeleteExecutor<>(statementProxy, statementCallback, sqlRecognizer);
                        break;
                    case SELECT_FOR_UPDATE:
                        executor = new SelectForUpdateExecutor<>(statementProxy, statementCallback, sqlRecognizer);
                        break;
                    default:
                        executor = new PlainExecutor<>(statementProxy, statementCallback);
                        break;
                }
            } else {
                executor = new MultiExecutor<>(statementProxy, statementCallback, sqlRecognizers);
            }
        }
        T rs;
        try {
			//执行语句
            rs = executor.execute(args);
        } catch (Throwable ex) {
            if (!(ex instanceof SQLException)) {
                // Turn other exception into SQLException
                ex = new SQLException(ex);
            }
            throw (SQLException) ex;
        }
        return rs;
    }
}
```

- update执行器流程：
```java
public abstract class BaseTransactionalExecutor<T, S extends Statement> implements Executor<T> {
    @Override
    public T execute(Object... args) throws Throwable {
        String xid = RootContext.getXID();
        if (xid != null) {
            statementProxy.getConnectionProxy().bind(xid);
        }

        statementProxy.getConnectionProxy().setGlobalLockRequire(RootContext.requireGlobalLock());
		//执行dml        
		return doExecute(args);
    }
}

public abstract class AbstractDMLBaseExecutor<T, S extends Statement> extends BaseTransactionalExecutor<T, S> {
    @Override
    public T doExecute(Object... args) throws Throwable {
        AbstractConnectionProxy connectionProxy = statementProxy.getConnectionProxy();
        if (connectionProxy.getAutoCommit()) {
			//默认自动提交
            return executeAutoCommitTrue(args);
        } else {
            return executeAutoCommitFalse(args);
        }
    }
    protected T executeAutoCommitTrue(Object[] args) throws Throwable {
        ConnectionProxy connectionProxy = statementProxy.getConnectionProxy();
        try {
			//改为手动提交事务
            connectionProxy.changeAutoCommit();
            return new LockRetryPolicy(connectionProxy).execute(() -> {
				//执行手动提交的事务
                T result = executeAutoCommitFalse(args);
				//手动提交
                connectionProxy.commit();
                return result;
            });
        } catch (Exception e) {
            // when exception occur in finally,this exception will lost, so just print it here
            LOGGER.error("execute executeAutoCommitTrue error:{}", e.getMessage(), e);
            if (!LockRetryPolicy.isLockRetryPolicyBranchRollbackOnConflict()) {
                connectionProxy.getTargetConnection().rollback();
            }
            throw e;
        } finally {
            connectionProxy.getContext().reset();
            connectionProxy.setAutoCommit(true);
        }
    }
}
```

- 执行手动提交的事务
```java
public abstract class AbstractDMLBaseExecutor<T, S extends Statement> extends BaseTransactionalExecutor<T, S> {
  
    protected T executeAutoCommitFalse(Object[] args) throws Exception {
        if (!JdbcConstants.MYSQL.equalsIgnoreCase(getDbType()) && isMultiPk()) {
            throw new NotSupportYetException("multi pk only support mysql!");
        }
		//获取update执行前的记录
        TableRecords beforeImage = beforeImage();
		//执行业务Update
        T result = statementCallback.execute(statementProxy.getTargetStatement(), args);
		//获取update执行后的记录
        TableRecords afterImage = afterImage(beforeImage);
		//通过before,after生成undolog
        prepareUndoLog(beforeImage, afterImage);
        return result;
    }
}
public class UpdateExecutor<T, S extends Statement> extends AbstractDMLBaseExecutor<T, S> {
    @Override
    protected TableRecords beforeImage() throws SQLException {
        ArrayList<List<Object>> paramAppenderList = new ArrayList<>();
        TableMeta tmeta = getTableMeta();
		//生成select for update语句，准备加全局锁和旧值
        String selectSQL = buildBeforeImageSQL(tmeta, paramAppenderList);
		//执行select for update
        return buildTableRecords(tmeta, selectSQL, paramAppenderList);
    }
    @Override
    protected TableRecords afterImage(TableRecords beforeImage) throws SQLException {
        TableMeta tmeta = getTableMeta();
        if (beforeImage == null || beforeImage.size() == 0) {
            return TableRecords.empty(getTableMeta());
        }
		//生成select语句，获取记录新值
        String selectSQL = buildAfterImageSQL(tmeta, beforeImage);
        ResultSet rs = null;
        try (PreparedStatement pst = statementProxy.getConnection().prepareStatement(selectSQL)) {
            SqlGenerateUtils.setParamForPk(beforeImage.pkRows(), getTableMeta().getPrimaryKeyOnlyName(), pst);
            rs = pst.executeQuery();
            return TableRecords.buildRecords(tmeta, rs);
        } finally {
            IOUtil.close(rs);
        }
    }
}

```

- 手动提交
```java
public class ConnectionProxy extends AbstractConnectionProxy {
   @Override
    public void commit() throws SQLException {
        try {
            LOCK_RETRY_POLICY.execute(() -> {
                doCommit();
                return null;
            });
        } catch (SQLException e) {
            if (targetConnection != null && !getAutoCommit() && !getContext().isAutoCommitChanged()) {
                rollback();
            }
            throw e;
        } catch (Exception e) {
            throw new SQLException(e);
        }
    }
    private void doCommit() throws SQLException {
        if (context.inGlobalTransaction()) {
			//提交全局事务的分支事务
            processGlobalTransactionCommit();
        } else if (context.isGlobalLockRequire()) {
            processLocalCommitWithGlobalLocks();
        } else {
            targetConnection.commit();
        }
    }
   private void processGlobalTransactionCommit() throws SQLException {
        try {
			//注册分支事务
            register();
        } catch (TransactionException e) {
            recognizeLockKeyConflictException(e, context.buildLockKeys());
        }
        try {
			//持久化undo log
            UndoLogManagerFactory.getUndoLogManager(this.getDbType()).flushUndoLogs(this);
            //手动commit
			targetConnection.commit();
        } catch (Throwable ex) {
            LOGGER.error("process connectionProxy commit error: {}", ex.getMessage(), ex);
            report(false);
            throw new SQLException(ex);
        }
        if (IS_REPORT_SUCCESS_ENABLE) {
            report(true);
        }
        context.reset();
    }
   private void register() throws TransactionException {
        if (!context.hasUndoLog() || !context.hasLockKey()) {
            return;
        }
		//在DefaultResourceManager中实现注册分支事务
        Long branchId = DefaultResourceManager.get().branchRegister(BranchType.AT, getDataSourceProxy().getResourceId(),
            null, context.getXid(), null, context.buildLockKeys());
        context.setBranchId(branchId);
    }
}

public class DefaultResourceManager implements ResourceManager {
    @Override
    public Long branchRegister(BranchType branchType, String resourceId,
                               String clientId, String xid, String applicationData, String lockKeys)
        throws TransactionException {
        return getResourceManager(branchType).branchRegister(branchType, resourceId, clientId, xid, applicationData,
            lockKeys);
    }
}

public abstract class AbstractResourceManager implements ResourceManager {
    @Override
    public Long branchRegister(BranchType branchType, String resourceId, String clientId, String xid, String applicationData, String lockKeys) throws TransactionException {
        try {
            BranchRegisterRequest request = new BranchRegisterRequest();
            request.setXid(xid);
            request.setLockKey(lockKeys);
            request.setResourceId(resourceId);
            request.setBranchType(branchType);
            request.setApplicationData(applicationData);
			
			//netty通知TC注册分支事务
            BranchRegisterResponse response = (BranchRegisterResponse) RmNettyRemotingClient.getInstance().sendSyncRequest(request);
            if (response.getResultCode() == ResultCode.Failed) {
                throw new RmTransactionException(response.getTransactionExceptionCode(), String.format("Response[ %s ]", response.getMsg()));
            }
            return response.getBranchId();
        } catch (TimeoutException toe) {
            throw new RmTransactionException(TransactionExceptionCode.IO, "RPC Timeout", toe);
        } catch (RuntimeException rex) {
            throw new RmTransactionException(TransactionExceptionCode.BranchRegisterFailed, "Runtime", rex);
        }
    }
}

```
#### 二阶段提交全局事务
- TC的netty调用触发，入口为AbstractNettyRemoting
```java
public abstract class AbstractNettyRemoting implements Disposable {
   protected void processMessage(ChannelHandlerContext ctx, RpcMessage rpcMessage) throws Exception {
        Object body = rpcMessage.getBody();
        if (body instanceof MessageTypeAware) {
            MessageTypeAware messageTypeAware = (MessageTypeAware) body;
			//根据msg类型获取processor
            final Pair<RemotingProcessor, ExecutorService> pair = this.processorTable.get((int) messageTypeAware.getTypeCode());
            if (pair != null) {
                if (pair.getSecond() != null) {
                    try {
                        pair.getSecond().execute(() -> {
                            try {
								//processor处理入口
                                pair.getFirst().process(ctx, rpcMessage);
                            } catch (Throwable th) {
                                LOGGER.error(FrameworkErrorCode.NetDispatch.getErrCode(), th.getMessage(), th);
                            } finally {
                                MDC.clear();
                            }
                        });
                    } catch (RejectedExecutionException e) {
                       ......
                    }
                } else {
                    try {
                        pair.getFirst().process(ctx, rpcMessage);
                    } catch (Throwable th) {
                        LOGGER.error(FrameworkErrorCode.NetDispatch.getErrCode(), th.getMessage(), th);
                    }
                }
            } else {
                LOGGER.error("This message type [{}] has no processor.", messageTypeAware.getTypeCode());
            }
        } else {
            LOGGER.error("This rpcMessage body[{}] is not MessageTypeAware type.", body);
        }
    }
}

public class RmBranchCommitProcessor implements RemotingProcessor {
  @Override
    public void process(ChannelHandlerContext ctx, RpcMessage rpcMessage) throws Exception {
        String remoteAddress = NetUtil.toStringAddress(ctx.channel().remoteAddress());
        Object msg = rpcMessage.getBody();
        if (LOGGER.isInfoEnabled()) {
            LOGGER.info("rm client handle branch commit process:" + msg);
        }
        handleBranchCommit(rpcMessage, remoteAddress, (BranchCommitRequest) msg);
    }

    private void handleBranchCommit(RpcMessage request, String serverAddress, BranchCommitRequest branchCommitRequest) {
        BranchCommitResponse resultMessage;
        resultMessage = (BranchCommitResponse) handler.onRequest(branchCommitRequest, null);
        if (LOGGER.isDebugEnabled()) {
            LOGGER.debug("branch commit result:" + resultMessage);
        }
        try {
            this.remotingClient.sendAsyncResponse(serverAddress, request, resultMessage);
        } catch (Throwable throwable) {
            LOGGER.error("branch commit error: {}", throwable.getMessage(), throwable);
        }
    }
}

public abstract class AbstractRMHandler extends AbstractExceptionHandler
    implements RMInboundHandler, TransactionMessageHandler {
    @Override
    public AbstractResultMessage onRequest(AbstractMessage request, RpcContext context) {
        if (!(request instanceof AbstractTransactionRequestToRM)) {
            throw new IllegalArgumentException();
        }
        AbstractTransactionRequestToRM transactionRequest = (AbstractTransactionRequestToRM)request;
        transactionRequest.setRMInboundMessageHandler(this);

		//处理netty请求
        return transactionRequest.handle(context);
    }
    @Override
    public BranchCommitResponse handle(BranchCommitRequest request) {
        BranchCommitResponse response = new BranchCommitResponse();
        exceptionHandleTemplate(new AbstractCallback<BranchCommitRequest, BranchCommitResponse>() {
            @Override
            public void execute(BranchCommitRequest request, BranchCommitResponse response)
                throws TransactionException {
				//处理netty请求
                doBranchCommit(request, response);
            }
        }, request, response);
        return response;
    }
    protected void doBranchCommit(BranchCommitRequest request, BranchCommitResponse response)
        throws TransactionException {
        String xid = request.getXid();
        long branchId = request.getBranchId();
        String resourceId = request.getResourceId();
        String applicationData = request.getApplicationData();
        if (LOGGER.isInfoEnabled()) {
            LOGGER.info("Branch committing: " + xid + " " + branchId + " " + resourceId + " " + applicationData);
        }
		//处理netty请求
        BranchStatus status = getResourceManager().branchCommit(request.getBranchType(), xid, branchId, resourceId,
            applicationData);
        response.setXid(xid);
        response.setBranchId(branchId);
        response.setBranchStatus(status);
        if (LOGGER.isInfoEnabled()) {
            LOGGER.info("Branch commit result: " + status);
        }

    }
}

public class DataSourceManager extends AbstractResourceManager {
    public BranchStatus branchCommit(BranchType branchType, String xid, long branchId, String resourceId,
                                     String applicationData) throws TransactionException {
		//全局提交异步处理
        return asyncWorker.branchCommit(xid, branchId, resourceId);
    }
}

public class AsyncWorker {
    public BranchStatus branchCommit(String xid, long branchId, String resourceId) {
        Phase2Context context = new Phase2Context(xid, branchId, resourceId);
        addToCommitQueue(context);
        return BranchStatus.PhaseTwo_Committed;
    }
    private void addToCommitQueue(Phase2Context context) {
		//阻塞队列异步执行
        if (commitQueue.offer(context)) {
            return;
        }
        CompletableFuture.runAsync(this::doBranchCommitSafely, scheduledExecutor)
                .thenRun(() -> addToCommitQueue(context));
    }
    void doBranchCommitSafely() {
        try {
            doBranchCommit();
        } catch (Throwable e) {
            LOGGER.error("Exception occur when doing branch commit", e);
        }
    }

    private void doBranchCommit() {
        if (commitQueue.isEmpty()) {
            return;
        }

        // transfer all context currently received to this list
        List<Phase2Context> allContexts = new LinkedList<>();
		//阻塞队列获取任务
        commitQueue.drainTo(allContexts);

        // group context by their resourceId
        Map<String, List<Phase2Context>> groupedContexts = groupedByResourceId(allContexts);
		//执行COMMIT
        groupedContexts.forEach(this::dealWithGroupedContexts);
    }
   private void dealWithGroupedContexts(String resourceId, List<Phase2Context> contexts) {
        DataSourceProxy dataSourceProxy = dataSourceManager.get(resourceId);
        if (dataSourceProxy == null) {
            LOGGER.warn("Failed to find resource for {}", resourceId);
            return;
        }

        Connection conn;
        try {
            conn = dataSourceProxy.getPlainConnection();
        } catch (SQLException sqle) {
            LOGGER.error("Failed to get connection for async committing on {}", resourceId, sqle);
            return;
        }

        UndoLogManager undoLogManager = UndoLogManagerFactory.getUndoLogManager(dataSourceProxy.getDbType());

        // split contexts into several lists, with each list contain no more element than limit size
        List<List<Phase2Context>> splitByLimit = Lists.partition(contexts, UNDOLOG_DELETE_LIMIT_SIZE);
		//删除undolog
        splitByLimit.forEach(partition -> deleteUndoLog(conn, undoLogManager, partition));
    }
}
```

#### 二阶段回滚全局事务
- TC的netty调用触发，入口为AbstractNettyRemoting
```java
public abstract class AbstractNettyRemoting implements Disposable {
   protected void processMessage(ChannelHandlerContext ctx, RpcMessage rpcMessage) throws Exception {
        Object body = rpcMessage.getBody();
        if (body instanceof MessageTypeAware) {
            MessageTypeAware messageTypeAware = (MessageTypeAware) body;
			//根据msg类型获取processor
            final Pair<RemotingProcessor, ExecutorService> pair = this.processorTable.get((int) messageTypeAware.getTypeCode());
            if (pair != null) {
                if (pair.getSecond() != null) {
                    try {
                        pair.getSecond().execute(() -> {
                            try {
								//processor处理入口
                                pair.getFirst().process(ctx, rpcMessage);
                            } catch (Throwable th) {
                                LOGGER.error(FrameworkErrorCode.NetDispatch.getErrCode(), th.getMessage(), th);
                            } finally {
                                MDC.clear();
                            }
                        });
                    } catch (RejectedExecutionException e) {
                       ......
                    }
                } else {
                    try {
                        pair.getFirst().process(ctx, rpcMessage);
                    } catch (Throwable th) {
                        LOGGER.error(FrameworkErrorCode.NetDispatch.getErrCode(), th.getMessage(), th);
                    }
                }
            } else {
                LOGGER.error("This message type [{}] has no processor.", messageTypeAware.getTypeCode());
            }
        } else {
            LOGGER.error("This rpcMessage body[{}] is not MessageTypeAware type.", body);
        }
    }
}

public class RmBranchRollbackProcessor implements RemotingProcessor {
    @Override
    public void process(ChannelHandlerContext ctx, RpcMessage rpcMessage) throws Exception {
        String remoteAddress = NetUtil.toStringAddress(ctx.channel().remoteAddress());
        Object msg = rpcMessage.getBody();
        if (LOGGER.isInfoEnabled()) {
            LOGGER.info("rm handle branch rollback process:" + msg);
        }
        handleBranchRollback(rpcMessage, remoteAddress, (BranchRollbackRequest) msg);
    }
    private void handleBranchRollback(RpcMessage request, String serverAddress, BranchRollbackRequest branchRollbackRequest) {
        BranchRollbackResponse resultMessage;
		//处理netty请求
        resultMessage = (BranchRollbackResponse) handler.onRequest(branchRollbackRequest, null);
        if (LOGGER.isDebugEnabled()) {
            LOGGER.debug("branch rollback result:" + resultMessage);
        }
        try {
            this.remotingClient.sendAsyncResponse(serverAddress, request, resultMessage);
        } catch (Throwable throwable) {
            LOGGER.error("send response error: {}", throwable.getMessage(), throwable);
        }
    }
}

public abstract class AbstractRMHandler extends AbstractExceptionHandler
    implements RMInboundHandler, TransactionMessageHandler {
    @Override
    public AbstractResultMessage onRequest(AbstractMessage request, RpcContext context) {
        if (!(request instanceof AbstractTransactionRequestToRM)) {
            throw new IllegalArgumentException();
        }
        AbstractTransactionRequestToRM transactionRequest = (AbstractTransactionRequestToRM)request;
        transactionRequest.setRMInboundMessageHandler(this);

		//处理netty请求
        return transactionRequest.handle(context);
    }
    @Override
    public BranchRollbackResponse handle(BranchRollbackRequest request) {
        BranchRollbackResponse response = new BranchRollbackResponse();
        exceptionHandleTemplate(new AbstractCallback<BranchRollbackRequest, BranchRollbackResponse>() {
            @Override
            public void execute(BranchRollbackRequest request, BranchRollbackResponse response)
                throws TransactionException {
				//处理netty请求
                doBranchRollback(request, response);
            }
        }, request, response);
        return response;
    }
    protected void doBranchRollback(BranchRollbackRequest request, BranchRollbackResponse response)
        throws TransactionException {
        String xid = request.getXid();
        long branchId = request.getBranchId();
        String resourceId = request.getResourceId();
        String applicationData = request.getApplicationData();
        if (LOGGER.isInfoEnabled()) {
            LOGGER.info("Branch Rollbacking: " + xid + " " + branchId + " " + resourceId);
        }
		//在DataSourceManager中执行回滚
        BranchStatus status = getResourceManager().branchRollback(request.getBranchType(), xid, branchId, resourceId,
            applicationData);
        response.setXid(xid);
        response.setBranchId(branchId);
        response.setBranchStatus(status);
        if (LOGGER.isInfoEnabled()) {
            LOGGER.info("Branch Rollbacked result: " + status);
        }
    }
}

public class DataSourceManager extends AbstractResourceManager {
    @Override
    public BranchStatus branchRollback(BranchType branchType, String xid, long branchId, String resourceId,
                                       String applicationData) throws TransactionException {
        DataSourceProxy dataSourceProxy = get(resourceId);
        if (dataSourceProxy == null) {
            throw new ShouldNeverHappenException();
        }
        try {
			//undologManager中执行回滚
            UndoLogManagerFactory.getUndoLogManager(dataSourceProxy.getDbType()).undo(dataSourceProxy, xid, branchId);
        } catch (TransactionException te) {
          ......
        }
        return BranchStatus.PhaseTwo_Rollbacked;

    }
}

public abstract class AbstractUndoLogManager implements UndoLogManager {
    @Override
    public void undo(DataSourceProxy dataSourceProxy, String xid, long branchId) throws TransactionException {
        Connection conn = null;
        ResultSet rs = null;
        PreparedStatement selectPST = null;
        boolean originalAutoCommit = true;

        for (; ; ) {
            try {
                conn = dataSourceProxy.getPlainConnection();

                // 手动提交
                if (originalAutoCommit = conn.getAutoCommit()) {
                    conn.setAutoCommit(false);
                }

                selectPST = conn.prepareStatement(SELECT_UNDO_LOG_SQL);
                selectPST.setLong(1, branchId);
                selectPST.setString(2, xid);
				//根据xid和branchid，用select for update查undolog
                rs = selectPST.executeQuery();

                boolean exists = false;
                while (rs.next()) {
	                ......
                    String contextString = rs.getString(ClientTableColumnsName.UNDO_LOG_CONTEXT);
                    Map<String, String> context = parseContext(contextString);

                    byte[] rollbackInfo = getRollbackInfo(rs);

                    String serializer = context == null ? null : context.get(UndoLogConstants.SERIALIZER_KEY);
                    UndoLogParser parser = serializer == null ? UndoLogParserFactory.getInstance()
                        : UndoLogParserFactory.getInstance(serializer);
                    BranchUndoLog branchUndoLog = parser.decode(rollbackInfo);

                    try {
                        // put serializer name to local
                        setCurrentSerializer(parser.getName());
                        List<SQLUndoLog> sqlUndoLogs = branchUndoLog.getSqlUndoLogs();
                        if (sqlUndoLogs.size() > 1) {
                            Collections.reverse(sqlUndoLogs);
                        }
                        for (SQLUndoLog sqlUndoLog : sqlUndoLogs) {
                            TableMeta tableMeta = TableMetaCacheFactory.getTableMetaCache(dataSourceProxy.getDbType()).getTableMeta(
                                conn, sqlUndoLog.getTableName(), dataSourceProxy.getResourceId());
                            sqlUndoLog.setTableMeta(tableMeta);
                            AbstractUndoExecutor undoExecutor = UndoExecutorFactory.getUndoExecutor(
                                dataSourceProxy.getDbType(), sqlUndoLog);
							//解析undo并恢复记录
                            undoExecutor.executeOn(conn);
                        }
                    } finally {
                        // remove serializer name
                        removeCurrentSerializer();
                    }
                }

                if (exists) {
					//删除undo并提交
                    deleteUndoLog(xid, branchId, conn);
                    conn.commit();
                    if (LOGGER.isInfoEnabled()) {
                        LOGGER.info("xid {} branch {}, undo_log deleted with {}", xid, branchId,
                            State.GlobalFinished.name());
                    }
                } else {
					//无undolog，说明一阶段未执行成功，回滚为空回滚
					//insert undolog，通过uk防止延迟的一阶段请求提交，造成数据不一致
                    insertUndoLogWithGlobalFinished(xid, branchId, UndoLogParserFactory.getInstance(), conn);
                    conn.commit();
                    if (LOGGER.isInfoEnabled()) {
                        LOGGER.info("xid {} branch {}, undo_log added with {}", xid, branchId,
                            State.GlobalFinished.name());
                    }
                }

                return;
            } catch (SQLIntegrityConstraintViolationException e) {
                // Possible undo_log has been inserted into the database by other processes, retrying rollback undo_log
                if (LOGGER.isInfoEnabled()) {
                    LOGGER.info("xid {} branch {}, undo_log inserted, retry rollback", xid, branchId);
                }
            } catch (Throwable e) {
               ......

            } finally {
                ......
            }
        }
    }
}

```

### TC
> 一句话实现：netty server接收TM、RM的请求，修改事务状态，并通知TM、RM操作。

#### 开启全局事务
- 和RM公用netty底层封装，入口同样为AbstractNettyRemoting#processMessage，只是执行的processor是ServerOnRequestProcessor
```java
public class ServerOnRequestProcessor implements RemotingProcessor {
   @Override
    public void process(ChannelHandlerContext ctx, RpcMessage rpcMessage) throws Exception {
        if (ChannelManager.isRegistered(ctx.channel())) {
            onRequestMessage(ctx, rpcMessage);
        } else {
            ......
        }
    }
   private void onRequestMessage(ChannelHandlerContext ctx, RpcMessage rpcMessage) {
        Object message = rpcMessage.getBody();
	    ......
        if (message instanceof MergedWarpMessage) {
            AbstractResultMessage[] results = new AbstractResultMessage[((MergedWarpMessage) message).msgs.size()];
            for (int i = 0; i < results.length; i++) {
                final AbstractMessage subMessage = ((MergedWarpMessage) message).msgs.get(i);
				//TransactionMessageHandler的实现类中处理请求                
				results[i] = transactionMessageHandler.onRequest(subMessage, rpcContext);
            }
            MergeResultMessage resultMessage = new MergeResultMessage();
            resultMessage.setMsgs(results);
            remotingServer.sendAsyncResponse(rpcMessage, ctx.channel(), resultMessage);
        } else {
            // the single send request message
            final AbstractMessage msg = (AbstractMessage) message;
            AbstractResultMessage result = transactionMessageHandler.onRequest(msg, rpcContext);
            remotingServer.sendAsyncResponse(rpcMessage, ctx.channel(), result);
        }
    }
}

public class DefaultCoordinator extends AbstractTCInboundHandler implements TransactionMessageHandler, Disposable {
    @Override
    public AbstractResultMessage onRequest(AbstractMessage request, RpcContext context) {
        if (!(request instanceof AbstractTransactionRequestToTC)) {
            throw new IllegalArgumentException();
        }
        AbstractTransactionRequestToTC transactionRequest = (AbstractTransactionRequestToTC) request;
        transactionRequest.setTCInboundHandler(this);

		//多层引用后在回到DefaultCoordinator中的doGlobalBegin处理
        return transactionRequest.handle(context);
    }

   @Override
    protected void doGlobalBegin(GlobalBeginRequest request, GlobalBeginResponse response, RpcContext rpcContext)
        throws TransactionException {
		//调用core开启事务
        response.setXid(core.begin(rpcContext.getApplicationId(), rpcContext.getTransactionServiceGroup(),
            request.getTransactionName(), request.getTimeout()));
        if (LOGGER.isInfoEnabled()) {
            LOGGER.info("Begin new global transaction applicationId: {},transactionServiceGroup: {}, transactionName: {},timeout:{},xid:{}",
                rpcContext.getApplicationId(), rpcContext.getTransactionServiceGroup(), request.getTransactionName(), request.getTimeout(), response.getXid());
        }
    }
}

public class DefaultCore implements Core {
    @Override
    public String begin(String applicationId, String transactionServiceGroup, String name, int timeout)
        throws TransactionException {
        GlobalSession session = GlobalSession.createGlobalSession(applicationId, transactionServiceGroup, name,
            timeout);
        MDC.put(RootContext.MDC_KEY_XID, session.getXid());
        session.addSessionLifecycleListener(SessionHolder.getRootSessionManager());
		//开启事务，触发listener
        session.begin();

        // transaction start event
        eventBus.post(new GlobalTransactionEvent(session.getTransactionId(), GlobalTransactionEvent.ROLE_TC,
            session.getTransactionName(), applicationId, transactionServiceGroup, session.getBeginTime(), null, session.getStatus()));

        return session.getXid();
    }
}
public class GlobalSession implements SessionLifecycle, SessionStorable {
    @Override
    public void begin() throws TransactionException {
        this.status = GlobalStatus.Begin;
        this.beginTime = System.currentTimeMillis();
        this.active = true;
        for (SessionLifecycleListener lifecycleListener : lifecycleListeners) {
			//持久化事务，默认使用file持久化
            lifecycleListener.onBegin(this);
        }
    }
}
public abstract class AbstractSessionManager implements SessionManager, SessionLifecycleListener {
  private void writeSession(LogOperation logOperation, SessionStorable sessionStorable) throws TransactionException {
        if (!transactionStoreManager.writeSession(logOperation, sessionStorable)) {
          ......
        }
    }
}

public class FileTransactionStoreManager extends AbstractTransactionStoreManager
    implements TransactionStoreManager, ReloadableStore {
   @Override
    public boolean writeSession(LogOperation logOperation, SessionStorable session) {
        writeSessionLock.lock();
        long curFileTrxNum;
        try {
			//写缓存
            if (!writeDataFile(new TransactionWriteStore(session, logOperation).encode())) {
                return false;
            }
            lastModifiedTime = System.currentTimeMillis();
            curFileTrxNum = FILE_TRX_NUM.incrementAndGet();
            if (curFileTrxNum % PER_FILE_BLOCK_SIZE == 0
                && (System.currentTimeMillis() - trxStartTimeMills) > MAX_TRX_TIMEOUT_MILLS) {
                return saveHistory();
            }
        } catch (Exception exx) {
            LOGGER.error("writeSession error, {}", exx.getMessage(), exx);
            return false;
        } finally {
            writeSessionLock.unlock();
        }
		//刷盘，默认地址 sessionStore\root.data
        flushDisk(curFileTrxNum, currFileChannel);
        return true;
    }

}
```

#### 提交全局事务
- 入口路径一致，只是DefaultCoordinator中调用doGlobalCommit方法
```java
public class DefaultCoordinator extends AbstractTCInboundHandler implements TransactionMessageHandler, Disposable {
    @Override
    protected void doGlobalCommit(GlobalCommitRequest request, GlobalCommitResponse response, RpcContext rpcContext)
        throws TransactionException {
        MDC.put(RootContext.MDC_KEY_XID, request.getXid());
        response.setGlobalStatus(core.commit(request.getXid()));
    }
}

public class DefaultCore implements Core {
    @Override
    public GlobalStatus commit(String xid) throws TransactionException {
        GlobalSession globalSession = SessionHolder.findGlobalSession(xid);
        if (globalSession == null) {
            return GlobalStatus.Finished;
        }
        globalSession.addSessionLifecycleListener(SessionHolder.getRootSessionManager());
        // just lock changeStatus

        boolean shouldCommit = SessionHolder.lockAndExecute(globalSession, () -> {
            // Highlight: Firstly, close the session, then no more branch can be registered.
            globalSession.closeAndClean();
            if (globalSession.getStatus() == GlobalStatus.Begin) {
                if (globalSession.canBeCommittedAsync()) {
					//默认异步提交
                    globalSession.asyncCommit();
                    return false;
                } else {
                    globalSession.changeStatus(GlobalStatus.Committing);
                    return true;
                }
            }
            return false;
        });

        if (shouldCommit) {
            boolean success = doGlobalCommit(globalSession, false);
            //If successful and all remaining branches can be committed asynchronously, do async commit.
            if (success && globalSession.hasBranch() && globalSession.canBeCommittedAsync()) {
                globalSession.asyncCommit();
                return GlobalStatus.Committed;
            } else {
                return globalSession.getStatus();
            }
        } else {
            return globalSession.getStatus() == GlobalStatus.AsyncCommitting ? GlobalStatus.Committed : globalSession.getStatus();
        }
    }
```

- 异步提交由DefaultCoordinator中的asyncCommitting这一线程池完成
```java
public class DefaultCoordinator extends AbstractTCInboundHandler implements TransactionMessageHandler, Disposable {
    protected void handleAsyncCommitting() {
        Collection<GlobalSession> asyncCommittingSessions = SessionHolder.getAsyncCommittingSessionManager()
            .allSessions();
        if (CollectionUtils.isEmpty(asyncCommittingSessions)) {
            return;
        }
        SessionHelper.forEach(asyncCommittingSessions, asyncCommittingSession -> {
            try {
                // Instruction reordering in DefaultCore#asyncCommit may cause this situation
                if (GlobalStatus.AsyncCommitting != asyncCommittingSession.getStatus()) {
                    //The function of this 'return' is 'continue'.
                    return;
                }
                asyncCommittingSession.addSessionLifecycleListener(SessionHolder.getRootSessionManager());
				//发起全局提交                
				core.doGlobalCommit(asyncCommittingSession, true);
            } catch (TransactionException ex) {
                LOGGER.error("Failed to async committing [{}] {} {}", asyncCommittingSession.getXid(), ex.getCode(), ex.getMessage(), ex);
            }
        });
    }
}
public class DefaultCore implements Core {

    @Override
    public boolean doGlobalCommit(GlobalSession globalSession, boolean retrying) throws TransactionException {
        boolean success = true;
        // start committing event
        eventBus.post(new GlobalTransactionEvent(globalSession.getTransactionId(), GlobalTransactionEvent.ROLE_TC,
            globalSession.getTransactionName(), globalSession.getApplicationId(), globalSession.getTransactionServiceGroup(),
            globalSession.getBeginTime(), null, globalSession.getStatus()));

        if (globalSession.isSaga()) {
            success = getCore(BranchType.SAGA).doGlobalCommit(globalSession, retrying);
        } else {
            Boolean result = SessionHelper.forEach(globalSession.getSortedBranches(), branchSession -> {
                // if not retrying, skip the canBeCommittedAsync branches
                if (!retrying && branchSession.canBeCommittedAsync()) {
                    return CONTINUE;
                }

                BranchStatus currentStatus = branchSession.getStatus();
                if (currentStatus == BranchStatus.PhaseOne_Failed) {
                    globalSession.removeBranch(branchSession);
                    return CONTINUE;
                }
                try {
					//core中提交全局事务
                    BranchStatus branchStatus = getCore(branchSession.getBranchType()).branchCommit(globalSession, branchSession);

                    switch (branchStatus) {
                        case PhaseTwo_Committed:
                            globalSession.removeBranch(branchSession);
                            return CONTINUE;
                        ......
                    }
                } catch (Exception ex) {
                    StackTraceLogger.error(LOGGER, ex, "Committing branch transaction exception: {}",
                        new String[] {branchSession.toString()});
                    if (!retrying) {
                        globalSession.queueToRetryCommit();
                        throw new TransactionException(ex);
                    }
                }
                return CONTINUE;
            });
            ......
        }
        //If success and there is no branch, end the global transaction.
        if (success && globalSession.getBranchSessions().isEmpty()) {
            SessionHelper.endCommitted(globalSession);

            // committed event
            eventBus.post(new GlobalTransactionEvent(globalSession.getTransactionId(), GlobalTransactionEvent.ROLE_TC,
                globalSession.getTransactionName(), globalSession.getApplicationId(), globalSession.getTransactionServiceGroup(),
                globalSession.getBeginTime(), System.currentTimeMillis(), globalSession.getStatus()));

            LOGGER.info("Committing global transaction is successfully done, xid = {}.", globalSession.getXid());
        }
        return success;
    }
}

public abstract class AbstractCore implements Core {
   @Override
    public BranchStatus branchCommit(GlobalSession globalSession, BranchSession branchSession) throws TransactionException {
        try {
            BranchCommitRequest request = new BranchCommitRequest();
            request.setXid(branchSession.getXid());
            request.setBranchId(branchSession.getBranchId());
            request.setResourceId(branchSession.getResourceId());
            request.setApplicationData(branchSession.getApplicationData());
            request.setBranchType(branchSession.getBranchType());
			//底层通过netty通知rm提交全局事务
            return branchCommitSend(request, globalSession, branchSession);
        } catch (IOException | TimeoutException e) {
            throw new BranchTransactionException(FailedToSendBranchCommitRequest,
                    String.format("Send branch commit failed, xid = %s branchId = %s", branchSession.getXid(),
                            branchSession.getBranchId()), e);
        }
    }
    protected BranchStatus branchCommitSend(BranchCommitRequest request, GlobalSession globalSession,
                                            BranchSession branchSession) throws IOException, TimeoutException {
        //底层netty实现
		BranchCommitResponse response = (BranchCommitResponse) remotingServer.sendSyncRequest(
                branchSession.getResourceId(), branchSession.getClientId(), request);
        return response.getBranchStatus();
    }
}
```

#### 回滚全局事务
- 入口路径一致，只是DefaultCoordinator中调用doGlobalRollback方法
```java
public class DefaultCoordinator extends AbstractTCInboundHandler implements TransactionMessageHandler, Disposable {
    @Override
    protected void doGlobalRollback(GlobalRollbackRequest request, GlobalRollbackResponse response,
                                    RpcContext rpcContext) throws TransactionException {
        MDC.put(RootContext.MDC_KEY_XID, request.getXid());
        response.setGlobalStatus(core.rollback(request.getXid()));
    }

}

public class DefaultCore implements Core {

   @Override
    public GlobalStatus rollback(String xid) throws TransactionException {
        GlobalSession globalSession = SessionHolder.findGlobalSession(xid);
        if (globalSession == null) {
            return GlobalStatus.Finished;
        }
        globalSession.addSessionLifecycleListener(SessionHolder.getRootSessionManager());
        // just lock changeStatus
        boolean shouldRollBack = SessionHolder.lockAndExecute(globalSession, () -> {
            globalSession.close(); // Highlight: Firstly, close the session, then no more branch can be registered.
            if (globalSession.getStatus() == GlobalStatus.Begin) {
                globalSession.changeStatus(GlobalStatus.Rollbacking);
                return true;
            }
            return false;
        });
        if (!shouldRollBack) {
            return globalSession.getStatus();
        }
		//实际回滚事务
        doGlobalRollback(globalSession, false);
        return globalSession.getStatus();
    }
   @Override
    public boolean doGlobalRollback(GlobalSession globalSession, boolean retrying) throws TransactionException {
        boolean success = true;
        // start rollback event
        eventBus.post(new GlobalTransactionEvent(globalSession.getTransactionId(),
                GlobalTransactionEvent.ROLE_TC, globalSession.getTransactionName(),
                globalSession.getApplicationId(),
                globalSession.getTransactionServiceGroup(), globalSession.getBeginTime(),
                null, globalSession.getStatus()));

        if (globalSession.isSaga()) {
            success = getCore(BranchType.SAGA).doGlobalRollback(globalSession, retrying);
        } else {
            Boolean result = SessionHelper.forEach(globalSession.getReverseSortedBranches(), branchSession -> {
                BranchStatus currentBranchStatus = branchSession.getStatus();
                if (currentBranchStatus == BranchStatus.PhaseOne_Failed) {
                    globalSession.removeBranch(branchSession);
                    return CONTINUE;
                }
                try {
					//回滚分支事务，类似全局提交，底层NETTY实现
                    BranchStatus branchStatus = branchRollback(globalSession, branchSession);
                    switch (branchStatus) {
                        case PhaseTwo_Rollbacked:
                            globalSession.removeBranch(branchSession);
                            LOGGER.info("Rollback branch transaction successfully, xid = {} branchId = {}", globalSession.getXid(), branchSession.getBranchId());
                            return CONTINUE;
                        case PhaseTwo_RollbackFailed_Unretryable:
                            SessionHelper.endRollbackFailed(globalSession);
                            LOGGER.info("Rollback branch transaction fail and stop retry, xid = {} branchId = {}", globalSession.getXid(), branchSession.getBranchId());
                            return false;
                        default:
                            LOGGER.info("Rollback branch transaction fail and will retry, xid = {} branchId = {}", globalSession.getXid(), branchSession.getBranchId());
                            if (!retrying) {
                                globalSession.queueToRetryRollback();
                            }
                            return false;
                    }
                } catch (Exception ex) {
                   ......
                }
            });
          ......
        return success;
    }
}
```

#### 注册分支事务
- 入口路径一致，只是DefaultCoordinator中调用doBranchRegister方法
```java
public class DefaultCoordinator extends AbstractTCInboundHandler implements TransactionMessageHandler, Disposable {
    @Override
    protected void doBranchRegister(BranchRegisterRequest request, BranchRegisterResponse response,
                                    RpcContext rpcContext) throws TransactionException {
        MDC.put(RootContext.MDC_KEY_XID, request.getXid());
        response.setBranchId(
            core.branchRegister(request.getBranchType(), request.getResourceId(), rpcContext.getClientId(),
                request.getXid(), request.getApplicationData(), request.getLockKey()));
    }

}

public abstract class AbstractCore implements Core {
   @Override
    public Long branchRegister(BranchType branchType, String resourceId, String clientId, String xid,
                               String applicationData, String lockKeys) throws TransactionException {
        GlobalSession globalSession = assertGlobalSessionNotNull(xid, false);
        return SessionHolder.lockAndExecute(globalSession, () -> {
            globalSessionStatusCheck(globalSession);
            globalSession.addSessionLifecycleListener(SessionHolder.getRootSessionManager());
            BranchSession branchSession = SessionHelper.newBranchByGlobal(globalSession, branchType, resourceId,
                    applicationData, lockKeys, clientId);
            MDC.put(RootContext.MDC_KEY_BRANCH_ID, String.valueOf(branchSession.getBranchId()));
            branchSessionLock(globalSession, branchSession);
            try {
				//类似开启事务，在session中的listener注册分支事务，并持久化
                globalSession.addBranch(branchSession);
            } catch (RuntimeException ex) {
                branchSessionUnlock(branchSession);
                throw new BranchTransactionException(FailedToAddBranch, String
                        .format("Failed to store branch xid = %s branchId = %s", globalSession.getXid(),
                                branchSession.getBranchId()), ex);
            }
            if (LOGGER.isInfoEnabled()) {
                LOGGER.info("Register branch successfully, xid = {}, branchId = {}, resourceId = {} ,lockKeys = {}",
                    globalSession.getXid(), branchSession.getBranchId(), resourceId, lockKeys);
            }
            return branchSession.getBranchId();
        });
    }
}
```

#### 补偿机制
- 依赖session持久化和定时任务重试，定时任务如下。
```java
public class DefaultCoordinator extends AbstractTCInboundHandler implements TransactionMessageHandler, Disposable {

    private ScheduledThreadPoolExecutor retryRollbacking = new ScheduledThreadPoolExecutor(1,
        new NamedThreadFactory("RetryRollbacking", 1));

    private ScheduledThreadPoolExecutor retryCommitting = new ScheduledThreadPoolExecutor(1,
        new NamedThreadFactory("RetryCommitting", 1));
}
```

## 参考
- [官网](http://seata.io/zh-cn/docs/overview/what-is-seata.html)
- [seata源码分析](https://objcoding.com/category/#Seata)
- [seata是什么](https://mp.weixin.qq.com/s?__biz=MzIwNTI2ODY5OA==&mid=2649939246&idx=1&sn=0668730aa133078972a24d7cab5d4f8b&chksm=8f350e9bb842878d96d71bf909c9e41b06e29e492464fc3eedbb552feea4e01ece3c45d81a1d&scene=178&cur_album_id=1689090473073639428#rd)
- [tc源码分析](https://www.freesion.com/article/6634911870/)