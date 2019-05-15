#### Spring事务机制及一种简单的主从数据源设置

##### 事务类型
1. REQUIRED: 支持事务，如果当前无事务则创建一个事务
2. SUPPORTS: 支持事务，如果当前无事务则在无事务环境运行
3. MANDATORY: 强制事务模式，如果当前无事务则抛出异常
4. REQUIRES_NEW：创建一个新事务，如果当前存在事务则挂起当前事务。
5. NOT_SUPPORTED: 不支持事务，如果当前存在事务则挂起当前事务
6. NEVER：不支持事务，如果当前存在事务则抛出异常
7. NESTED: 嵌套事务,当前无事务则创建一个
##### 事务实现原理
spring里面的事务是托管给```PlatformTransactionManager```，利用动态代理实现，所以重点就在```TransactionInterceptor.invoke```方法。
看看里面部分代码:
```
// If the transaction attribute is null, the method is non-transactional.
		TransactionAttributeSource tas = getTransactionAttributeSource();
		final TransactionAttribute txAttr = (tas != null ? tas.getTransactionAttribute(method, targetClass) : null);
		final PlatformTransactionManager tm = determineTransactionManager(txAttr);
		final String joinpointIdentification = methodIdentification(method, targetClass, txAttr);

		if (txAttr == null || !(tm instanceof CallbackPreferringPlatformTransactionManager)) {
			// Standard transaction demarcation with getTransaction and commit/rollback calls.
			TransactionInfo txInfo = createTransactionIfNecessary(tm, txAttr, joinpointIdentification);
			Object retVal = null;
			try {
				// This is an around advice: Invoke the next interceptor in the chain.
				// This will normally result in a target object being invoked.
				retVal = invocation.proceedWithInvocation();
			}
			catch (Throwable ex) {
				// target invocation exception
				completeTransactionAfterThrowing(txInfo, ex);
				throw ex;
			}
			finally {
				cleanupTransactionInfo(txInfo);
			}
			commitTransactionAfterReturning(txInfo);
			return retVal;
		}
```
源码里面已经加了详细的注释了，接下来分析几个关键方法：
1. ```tas.getTransactionAttribute```最终是调用的```computeTransactionAttribute```方法：
```
if (allowPublicMethodsOnly() && !Modifier.isPublic(method.getModifiers())) {
				return null;
			}
```
默认情况下限制了事务入口方法访问权限必须是public。
```
// First try is the method in the target class.
			txAtt = findTransactionAttribute(specificMethod);
			if (txAtt != null) {
				return txAtt;
			}

			// Second try is the transaction attribute on the target class.
			txAtt = findTransactionAttribute(specificMethod.getDeclaringClass());
			if (txAtt != null) {
				return txAtt;
			}
```
findTransactionAttribute方法实际是分析Transactional注解，而且根据代码顺序，方法上的注解优先于类上的全局注解限制。
2. ```createTransactionIfNecessary```的重点是```getTransaction```方法：
```
public final TransactionStatus getTransaction(@Nullable TransactionDefinition definition) throws TransactionException {
		Object transaction = doGetTransaction();

		// Cache debug flag to avoid repeated checks.
		boolean debugEnabled = logger.isDebugEnabled();

		if (definition == null) {
			// Use defaults if no transaction definition given.
			definition = new DefaultTransactionDefinition();
		}

		if (isExistingTransaction(transaction)) {
			// Existing transaction found -> check propagation behavior to find out how to behave.
			return handleExistingTransaction(definition, transaction, debugEnabled);
		}

		// Check definition settings for new transaction.
		if (definition.getTimeout() < TransactionDefinition.TIMEOUT_DEFAULT) {
			throw new InvalidTimeoutException("Invalid transaction timeout", definition.getTimeout());
		}

		// No existing transaction found -> check propagation behavior to find out how to proceed.
		if (definition.getPropagationBehavior() == TransactionDefinition.PROPAGATION_MANDATORY) {
			throw new IllegalTransactionStateException(
					"No existing transaction found for transaction marked with propagation 'mandatory'");
		}
		else if (definition.getPropagationBehavior() == TransactionDefinition.PROPAGATION_REQUIRED ||
				definition.getPropagationBehavior() == TransactionDefinition.PROPAGATION_REQUIRES_NEW ||
				definition.getPropagationBehavior() == TransactionDefinition.PROPAGATION_NESTED) {
			SuspendedResourcesHolder suspendedResources = suspend(null);
			if (debugEnabled) {
				logger.debug("Creating new transaction with name [" + definition.getName() + "]: " + definition);
			}
			try {
				boolean newSynchronization = (getTransactionSynchronization() != SYNCHRONIZATION_NEVER);
				DefaultTransactionStatus status = newTransactionStatus(
						definition, transaction, true, newSynchronization, debugEnabled, suspendedResources);
				doBegin(transaction, definition);
				prepareSynchronization(status, definition);
				return status;
			}
			catch (RuntimeException | Error ex) {
				resume(null, suspendedResources);
				throw ex;
			}
		}
		else {
			// Create "empty" transaction: no actual transaction, but potentially synchronization.
			if (definition.getIsolationLevel() != TransactionDefinition.ISOLATION_DEFAULT && logger.isWarnEnabled()) {
				logger.warn("Custom isolation level specified but no actual transaction initiated; " +
						"isolation level will effectively be ignored: " + definition);
			}
			boolean newSynchronization = (getTransactionSynchronization() == SYNCHRONIZATION_ALWAYS);
			return prepareTransactionStatus(definition, null, true, newSynchronization, debugEnabled, null);
		}
	}
```
这里主要是对不同的事务类型以及当前是否存在事务做相应的处理，但是不管是否存在真正的事务(```actualTransactionActive=true```),都是在SynchronizationActive环境运行。如果是在一个事务活动里面新起了一个事务活动会挂起当前事务活动并将相关信息（SuspendedResourcesHolder）保存留待后面恢复。
再看看```createTransactionIfNecessary```返回的类型```TransactionInfo```：
```
@Nullable
		private final PlatformTransactionManager transactionManager;

		@Nullable
		private final TransactionAttribute transactionAttribute;

		private final String joinpointIdentification;

		@Nullable
		private TransactionStatus transactionStatus;

		@Nullable
		private TransactionInfo oldTransactionInfo;
```
这里面保存了事务执行相关的信息以及上一个事务活动的信息(oldTransactionInfo)，并由ThreadLocal类型的transactionInfoHolder属性维护，所以在同一线程下构成了一个事务栈，**这也是事务属性无法跨线程传递的原因**。
3. ```completeTransactionAfterThrowing```主要是进行异常处理。
会根据```TransactionAttribute```实现类不同进行有不同的回滚限制，比如默认情况下要求是RuntimeException或者Error：
```
public boolean rollbackOn(Throwable ex) {
		return (ex instanceof RuntimeException || ex instanceof Error);
	}
```
否则会提交后抛出异常.
4. finally里面会重置当前事务信息为TransactionInfo里面保存的上一个事务。
5. ```commitTransactionAfterReturning```方法里面根据具体情况判断进行提交或者回滚。

#### 一种简单的主从设置
目前我们生产环境业务量不大，一主一从足够了。
##### 实现思路
1. 有一个管理器可以配置多个数据源并根据指定的key返回特定的数据源。
2. 通过编码或者拦截器的形式在获取DataSource的Connection前指定选择的key。
###### AbstractRoutingDataSource
```AbstractRoutingDataSource```是spring提供的一个类，属性resolvedDataSources的类型是Map<Object, DataSource>，
可以根据key保存不同的DataSource。方法```determineTargetDataSource```源码如下：
```
protected DataSource determineTargetDataSource() {
		Assert.notNull(this.resolvedDataSources, "DataSource router not initialized");
		Object lookupKey = determineCurrentLookupKey();
		DataSource dataSource = this.resolvedDataSources.get(lookupKey);
		if (dataSource == null && (this.lenientFallback || lookupKey == null)) {
			dataSource = this.resolvedDefaultDataSource;
		}
		if (dataSource == null) {
			throw new IllegalStateException("Cannot determine target DataSource for lookup key [" + lookupKey + "]");
		}
		return dataSource;
	}
```
就是通过方法```determineCurrentLookupKey```方法确定当前key，然后获取当前选择的DataSource。这个方法是个抽象方法，只需要继承类AbstractRoutingDataSource并实现该方法即可.
```
public class DynamicDataSource extends AbstractRoutingDataSource {
    @Override
    protected Object determineCurrentLookupKey() {
        return DatasourceHold.getDBType();
    }

    @Override
    public DataSource determineTargetDataSource() {
        return super.determineTargetDataSource();
    }
}
```
DatasourceHold是一个自定义的类，里面声明了三个ThreadLocal<String>类型的属性用以记录当前线程使用的数据库.
```
    //当前DataSource对应的Key
    private static final ThreadLocal<String> contextHolder = new ThreadLocal<>();

    // 是否强制主库
    private static final ThreadLocal<Boolean> force = new ThreadLocal<>();

    // 强制主库次数
    private static final ThreadLocal<Integer> forceTimes = new ThreadLocal<>();

```
###### Key的拦截切换
Mybatis执行sql的时候是通过Executor去获取Connection执行，所以需要依靠Mybatis提供的plugin实现：
```
@Intercepts({
        @Signature(type = Executor.class, method = "query", args = { MappedStatement.class, Object.class,
                RowBounds.class, ResultHandler.class }),
        @Signature(type = Executor.class, method = "update", args = { MappedStatement.class, Object.class}) })
@Slf4j
public class DynamicDataSourceInterceptor implements Interceptor {
    private Properties properties;

    @Override
    public Object intercept(Invocation invocation) throws Throwable {
        boolean isTransactional = TransactionSynchronizationManager.isSynchronizationActive();
        boolean isQuery = "query".equals(invocation.getMethod().getName());
        if (!isTransactional && !DatasourceHold.isForce()) {
            if (isQuery) {
                DatasourceHold.convertToSlave();
            } else {
                DatasourceHold.convertToMaster();
            }
        } else {
            DatasourceHold.convertToMaster();
        }
        
        return invocation.proceed();
    }

    @Override
    public Object plugin(Object arg0) {
        return Plugin.wrap(arg0, this);
    }

    @Override
    public void setProperties(Properties arg0) {
        properties = arg0;

    }
}
```
也就是在执行Executor的query方法和update前先指定数据源key。
这里需要注意方法```TransactionSynchronizationManager.isSynchronizationActive()```,有些会使用```isActualTransactionActive```，按照上面分析事务机制时描述的，有些时候没有真正事务也是会在SynchronizationActive环境下执行，而mybatis会在SynchronizationActive下缓存sqlSession(参考类SqlSessionUtils的getSqlSession方法)，这就导致executor里面获取Connection时获取到缓存好的连接，而不是根据当前key重新获取，会出现以下后果：比如在NOT_SUPPORTED类型下，并没有真正事务，但是会在SynchronizationActive下执行，如果第一条sql是query，sqlSession里面的连接就会是Slave的连接，这时候第二条sql是update，虽然DatasourceHold里面是维持的Master，但是并不会重新获取connection，而是复用之前缓存的Salve的连接，导致更新失败。
以上是无真正事务的情况下，如果存在真正事务，连接的获取会提前到DataSourceTransactionManager的doBegin方法，所以在这里也需要拦截切换数据源key，实现比较简单，直接继承DataSourceTransactionManager重写doBegin方法即可。
```
public class DynamicDataSourceTransactionManager extends DataSourceTransactionManager {

    private static final long serialVersionUID = 1L;

    public DynamicDataSourceTransactionManager(DataSource dataSource) {
        super(dataSource);
    }

    @Override
    protected void doBegin(Object transaction, TransactionDefinition definition) {
        boolean readOnly = definition.isReadOnly();
        if(readOnly){//只读
            DatasourceHold.convertToSlave();
        }else{//读写
            DatasourceHold.convertToMaster();
        }
        super.doBegin(transaction, definition);
    }

    @Override
    protected void doCleanupAfterCompletion(Object transaction) {
        super.doCleanupAfterCompletion(transaction);
        DatasourceHold.clearDBType();
    }

}
```