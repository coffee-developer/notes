# Spring @Transactional源码解析

版本
+ spring boot: 2.1.11.RELEASE
+ spring-tx:5.1.12.RELEASE  

## 为什么想读Spring Transactional的源码
Spring的事务管理机制是面试必问的,但一般面试官问我的关于Spring事务的问题我一般只能回答它是基于Aop的以及传播机制再深入下去我就回答不出来了.所以趁着过年这段时间闲赋在家花了好几天研究了Spring Transactional的源码,希望下次面试官再问我这个问题我可以吊打他,哈哈哈哈哈哈哈哈哈(意淫中).

## 如何读取@Transactional信息  
### 从JavaDoc中找线索
解析@Transactional源码当然是先从@Transactional开始的.先看一下它的JavaDoc,spring的文档真他妈的详细.在阅读源码的时候看一下JavaDoc会给你带来很多帮助.  
JavaDoc里面有这样一句话
```
 * This annotation type is generally directly comparable to Spring's
 * {@link org.springframework.transaction.interceptor.RuleBasedTransactionAttribute}
 * class, and in fact {@link AnnotationTransactionAttributeSource} will directly
 * convert the data to the latter class, so that Spring's transaction support code
 * does not have to know about annotations. If no rules are relevant to the exception,
 * it will be treated like
 * {@link org.springframework.transaction.interceptor.DefaultTransactionAttribute}
 * (rolling back on {@link RuntimeException} and {@link Error} but not on checked
 * exceptions).
```
这句话为我们提供的线索:@Transactional上的内容由AnnotationTransactionAttributeSource生成RuleBasedTransactionAttribute(下文的事务行为).  
AnnotationTransactionAttributeSource实现了TransactionAttributeSource接口,TransactionAttributeSource只有一个方法,我们看一下那个方法呗.
```
	public interface TransactionAttributeSource {
		/**
		* Return the transaction attribute for the given method,
		* or {@code null} if the method is non-transactional.
		* @param method the method to introspect
		* @param targetClass the target class (may be {@code null},
		* in which case the declaring class of the method must be used)
		* @return the matching transaction attribute, or {@code null} if none found
		*/
		@Nullable
		TransactionAttribute getTransactionAttribute(Method method, @Nullable Class<?> targetClass);
	}
```
这个方法的功能是将方法上的事务行为内容解析出来  

### AnnotationTransactionAttributeSource读取事务信息
AnnotationTransactionAttributeSource继承了AbstractFallbackTransactionAttributeSource.AbstractFallbackTransactionAttributeSource是TransactionAttributeSource的骨架类跟模板类
```
public abstract class AbstractFallbackTransactionAttributeSource implements TransactionAttributeSource {
	protected abstract TransactionAttribute findTransactionAttribute(Class<?> clazz);
	protected abstract TransactionAttribute findTransactionAttribute(Method method);
	protected boolean allowPublicMethodsOnly() {
		return false;
	}
}
```
AbstractFallbackTransactionAttributeSource定义了3个模板方法给子类实现,前面两个是分别从class跟method上获取事务,第三个是否只从public方法上获取事务行为
```
	public TransactionAttribute getTransactionAttribute(Method method, @Nullable Class<?> targetClass){
		// 如果如果方法是object的实现直接返回null
		if (method.getDeclaringClass() == Object.class) {
			return null;
		}

		// 从本地缓存中查找相关缓存
		Object cacheKey = getCacheKey(method, targetClass);
		TransactionAttribute cached = this.attributeCache.get(cacheKey);
		// 如果缓存中有的话直接返回缓存内容
		if (cached != null) {
			// 做缓存穿透处理直接返回null
			if (cached == NULL_TRANSACTION_ATTRIBUTE) {
				return null;
			}
			else {
				return cached;
			}
		}
		else {
			// 如果没有的话就通过computeTransactionAttribute方法获取事务行为  
			// ps:下一个代码块是computeTransactionAttribute的代码解析
			TransactionAttribute txAttr = computeTransactionAttribute(method, targetClass);
			// 如果在方法上没有定义事务行为的话在缓存上放一个空值,避免缓存穿透
			if (txAttr == null) {
				this.attributeCache.put(cacheKey, NULL_TRANSACTION_ATTRIBUTE);
			}
			else {
				// 获取方法全名 例如 fullClassName.methodClassName
				String methodIdentification = ClassUtils.getQualifiedMethodName(method, targetClass);
				if (txAttr instanceof DefaultTransactionAttribute) {
					((DefaultTransactionAttribute) txAttr).setDescriptor(methodIdentification);
				}
				if (logger.isTraceEnabled()) {
					logger.trace("Adding transactional method '" + methodIdentification + "' with attribute: " + txAttr);
				}
				// 将事务行为放入缓存中
				this.attributeCache.put(cacheKey, txAttr);
			}
			return txAttr;
		}
	}
```
computeTransactionAttribute代码块
```
	protected TransactionAttribute computeTransactionAttribute(Method method, @Nullable Class<?> targetClass) {
		// 只解析方法访问符号是public的方法,如果不是不进行解析直接返回null 
		// AnnotationTransactionAttributeSource对于allowPublicMethodsOnly()方法的实现默认是返回它的publicMethodsOnly类实例变量  
		// spring在实例化AnnotationTransactionAttributeSource的时候publicMethodsOnly是true
		// ps: 这是平时开发中很容易踩到的坑,如果将@Transactional标记在非public方法上的时候事务不会生效
		if (allowPublicMethodsOnly() && !Modifier.isPublic(method.getModifiers())) {
			return null;
		}

		// 从代理对象上的一个方法，找到真实对象上对应的方法
		Method specificMethod = AopUtils.getMostSpecificMethod(method, targetClass);

		// 先从method上获取事务行为
		// ps: AnnotationTransactionAttributeSource调用TransactionAnnotationParser的parseTransactionAnnotation方法上获取事务行为
		// TransactionAnnotationParser主要有三个实现
		// Ejb3TransactionAnnotationParser 读取@javax.ejb.TransactionAttribute
		// JtaTransactionAnnotationParser 读取@javax.transaction.Transactional
		// SpringTransactionAnnotationParser 读取@org.springframework.transaction.annotation.Transactional
		TransactionAttribute txAttr = findTransactionAttribute(specificMethod);
		if (txAttr != null) {
			return txAttr;
		}

		// 如果method上获取不到事务行为的话从class上的注解获取
		txAttr = findTransactionAttribute(specificMethod.getDeclaringClass());
		if (txAttr != null && ClassUtils.isUserLevelMethod(method)) {
			return txAttr;
		}

		// 还是找不到的话继续查找
		if (specificMethod != method) {
			// 在spring aop生成proxyClass上的method上查找事务行为
			txAttr = findTransactionAttribute(method);
			if (txAttr != null) {
				return txAttr;
			}
			// 在定义这个定义这个method的class上读取事务行为
			txAttr = findTransactionAttribute(method.getDeclaringClass());
			if (txAttr != null && ClassUtils.isUserLevelMethod(method)) {
				return txAttr;
			}
		}

		return null;
	}
```
#### SpringTransactionAnnotationParser将@Transactional信息转换为事务行为
```
public class SpringTransactionAnnotationParser implements TransactionAnnotationParser, Serializable {
	/**
	*	AnnotationAttributes存储了@Transactional的信息
	*/ 
	protected TransactionAttribute parseTransactionAnnotation(AnnotationAttributes attributes) {
		RuleBasedTransactionAttribute rbta = new RuleBasedTransactionAttribute();

		Propagation propagation = attributes.getEnum("propagation");
		rbta.setPropagationBehavior(propagation.value());
		Isolation isolation = attributes.getEnum("isolation");
		rbta.setIsolationLevel(isolation.value());
		rbta.setTimeout(attributes.getNumber("timeout").intValue());
		rbta.setReadOnly(attributes.getBoolean("readOnly"));
		rbta.setQualifier(attributes.getString("value"));

		List<RollbackRuleAttribute> rollbackRules = new ArrayList<>();
		for (Class<?> rbRule : attributes.getClassArray("rollbackFor")) {
			rollbackRules.add(new RollbackRuleAttribute(rbRule));
		}
		for (String rbRule : attributes.getStringArray("rollbackForClassName")) {
			rollbackRules.add(new RollbackRuleAttribute(rbRule));
		}
		for (Class<?> rbRule : attributes.getClassArray("noRollbackFor")) {
			rollbackRules.add(new NoRollbackRuleAttribute(rbRule));
		}
		for (String rbRule : attributes.getStringArray("noRollbackForClassName")) {
			rollbackRules.add(new NoRollbackRuleAttribute(rbRule));
		}
		rbta.setRollbackRules(rollbackRules);

		return rbta;
	}
}
```
#### RuleBasedTransactionAttribute事务行为  
RuleBasedTransactionAttribute中rollbackRules返回一个布尔值,这个布尔值十分重要,它是在执行事务的时候发生异常的时候十分要回滚事务的依据
```
public class RuleBasedTransactionAttribute extends DefaultTransactionAttribute implements Serializable {
	public boolean rollbackOn(Throwable ex) {
		if (logger.isTraceEnabled()) {
			logger.trace("Applying rules to determine whether transaction should rollback on " + ex);
		}

		RollbackRuleAttribute winner = null;
		int deepest = Integer.MAX_VALUE;

		// 选出一个继承层次跟抛出的异常ex最接近的RollbackRuleAttribute来判断是否要回滚
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

		// 如果我们在@Transactional上没有定义回滚信息的话调用父类的rollbackOn方法来判断是否要进行回滚,父类的实现是只回滚RuntimeException跟Error
		if (winner == null) {
			logger.trace("No relevant rollback rule found: applying default rules");
			return super.rollbackOn(ex);
		}

		return !(winner instanceof NoRollbackRuleAttribute);
	}
}
```
## 如何拦截方法执行事务
现在已经告一段落了,我们理清了spring是如何读取我们标记在方法上@Transactional的内容了.接下来我们要看一下spring是如何做事务处理的  
我们现在将画面转移到AnnotationTransactionAttributeSource的JavaDoc上继续查找线索.在JavaDoc上的@see上标记着好几个类,其中一些我们已经解析过了,我们看一下没有解析的吧
首先TransactionProxyFactoryBean是跟spring xml的<tx>配置相关的类这里就不论述了  
### TransactionAspectSupport拦截方法并执行事务
TransactionInterceptor顾名思义是事务拦截器的意思,从页面上来看它会拦截定义事务的方法.TransactionInterceptor实现了spring aop的MethodInterceptor接口它会对方法进行拦截(下文什么是MethodInterceptor). 
TransactionInterceptor继承了TransactionAspectSupport,TransactionAspectSupport类定义了spring对于切面事务的支持,相关代码主要在invokeWithinTransaction方法上,下面是invokeWithinTransaction方法的代码块   
```
	protected Object invokeWithinTransaction(Method method, @Nullable Class<?> targetClass,
			final InvocationCallback invocation) throws Throwable {
		TransactionAttributeSource tas = getTransactionAttributeSource();
		// 读取事务行为
		final TransactionAttribute txAttr = (tas != null ? tas.getTransactionAttribute(method, targetClass) : null);
		// PlatformTransactionManager顾名思义是平台事务管理器,这里使用策略设计模式,在不同的场景下有不同的实现
		// 确定该事务由哪个事务管理器管理,如果在@Transactional上不指定的话默认的是DataSourceTransactionManager
		final PlatformTransactionManager tm = determineTransactionManager(txAttr);
		// 获取事务行为标识,一般情况下这里会获取到方法的全名
		final String joinpointIdentification = methodIdentification(method, targetClass, txAttr);
 
		// 这里不对CallbackPreferringPlatformTransactionManager进行详细解析,因为它是跟编码式事务(@Transactional是声明式事务)相关的类
		// 如果拦截的方法没有定义事务行为的话不进行拦截
		if (txAttr == null || !(tm instanceof CallbackPreferringPlatformTransactionManager)) {
			// 顾名思义获取事务不存在时根据情况获取事务,下面会对createTransactionIfNecessary方法进行解析
			TransactionInfo txInfo = createTransactionIfNecessary(tm, txAttr, joinpointIdentification);

			Object retVal;
			try {
				// 执行被拦截的方法
				retVal = invocation.proceedWithInvocation();
			}
			catch (Throwable ex) {
				// 发生异常时根据方法行为判断是否要提交或者回滚事务,跟我们定义@Transactional上的rollbackFor跟noRollbackFor相关
				// 下面会对noRollbackFor方法进行讲解
				completeTransactionAfterThrowing(txInfo, ex);
				throw ex;
			}
			finally {
				// 清理ThreadLocal中的事务并设置成上一个事务
				cleanupTransactionInfo(txInfo);
			}
			// 顾名思义进行代码提交,下面也会进行解析
			commitTransactionAfterReturning(txInfo);
			return retVal;
		}
	}
```
解析"下面会进行解析"的方法
```
protected TransactionInfo createTransactionIfNecessary(@Nullable PlatformTransactionManager tm,
			@Nullable TransactionAttribute txAttr, final String joinpointIdentification) {

		// 委派设计模式,让事务行为类返回方法名称
		if (txAttr != null && txAttr.getName() == null) {
			txAttr = new DelegatingTransactionAttribute(txAttr) {
				@Override
				public String getName() {
					return joinpointIdentification;
				}
			};
		}

		// 获取事务
		TransactionStatus status = null;
		if (txAttr != null) {
			if (tm != null) {
				// 获取或者创建事务
				status = tm.getTransaction(txAttr);
			}
			else {
				if (logger.isDebugEnabled()) {
					logger.debug("Skipping transactional joinpoint [" + joinpointIdentification +
							"] because no transaction manager has been configured");
				}
			}
		}
		// 对事务的执行做准备
		return prepareTransactionInfo(tm, txAttr, joinpointIdentification, status);
	}

	protected void completeTransactionAfterThrowing(@Nullable TransactionInfo txInfo, Throwable ex) {
		if (txInfo != null && txInfo.getTransactionStatus() != null) {
			if (logger.isTraceEnabled()) {
				logger.trace("Completing transaction for [" + txInfo.getJoinpointIdentification() +
						"] after exception: " + ex);
			}
			// 判断事务行为是否要回滚
			// 参考前面对于RuleBasedTransactionAttribute的解析
			if (txInfo.transactionAttribute != null && txInfo.transactionAttribute.rollbackOn(ex)) {
				try {
					// 调用PlatformTransactionManager事务管理器回滚事务
					txInfo.getTransactionManager().rollback(txInfo.getTransactionStatus());
				}
				catch (TransactionSystemException ex2) {
					logger.error("Application exception overridden by rollback exception", ex);
					ex2.initApplicationException(ex);
					throw ex2;
				}
				catch (RuntimeException | Error ex2) {
					logger.error("Application exception overridden by rollback exception", ex);
					throw ex2;
				}
			}
			else {
				// 否则提交事务
				try {
					// 调用PlatformTransactionManager事务管理器提交事务
					txInfo.getTransactionManager().commit(txInfo.getTransactionStatus());
				}
				catch (TransactionSystemException ex2) {
					logger.error("Application exception overridden by commit exception", ex);
					ex2.initApplicationException(ex);
					throw ex2;
				}
				catch (RuntimeException | Error ex2) {
					logger.error("Application exception overridden by commit exception", ex);
					throw ex2;
				}
			}
		}
	}

	protected void commitTransactionAfterReturning(@Nullable TransactionInfo txInfo) {
		if (txInfo != null && txInfo.getTransactionStatus() != null) {
			if (logger.isTraceEnabled()) {
				logger.trace("Completing transaction for [" + txInfo.getJoinpointIdentification() + "]");
			}
			// 调用PlatformTransactionManager事务管理器提交事务
			txInfo.getTransactionManager().commit(txInfo.getTransactionStatus());
		}
	}
```
### PlatformTransactionManager事务管理器操作事务
```
public interface PlatformTransactionManager{
	TransactionStatus getTransaction(@Nullable TransactionDefinition definition) throws TransactionException;
	void commit(TransactionStatus status) throws TransactionException;
	void rollback(TransactionStatus status) throws TransactionException;
}
```
看commit(TransactionStatus status)跟rollback(TransactionStatus status)的名字就可以看出来它们是对事务的提交跟回滚.getTransaction(TransactionStatus status)的JavaDoc上有这么一句话"Return a currently active transaction or create a new one, according to the specified propagation behavior.",根据我们在事务行为的传播机制来判断是否创建事务.让我们来看一下spring定义的传播机制吧
- REQUIRED: 如果当前没有事务的话就创建事务,它是@Transactional事务传播机制的默认值
- SUPPORTS: 如果当前有事务的话就加入当前事务执行,如果当前没事务就以无事务的方式执行
- MANDATORY: 当前必须存在事务,如果不存在抛出异常
- REQUIRES_NEW: 挂起当前事务并创建一个新的事务执行
- NOT_SUPPORTED: 跟REQUIRES_NEW相反,挂起当前事务并以无事务的方式执行
- NEVER: 跟MANDATORY相反,当前存在事务抛出异常
- NESTED: 嵌入式事务,如果当前存在事务的话就创建一个数据库SavePoint发生异常的时候回滚到这个SavePoint,不存在事务的话就创建一个新的事务  
  
#### AbstractPlatformTransactionManager对于PlatformTransactionManager的实现
##### getTransaction获取或者创建事务
```
public final TransactionStatus getTransaction(@Nullable TransactionDefinition definition) throws TransactionException {
		// 通过子类实现的模板方法来获取当前事务信息
		Object transaction = doGetTransaction();

		boolean debugEnabled = logger.isDebugEnabled();

		if (definition == null) {
			definition = new DefaultTransactionDefinition();
		}

		// 通过子类实现的模板方法来判断是否存在事务
		if (isExistingTransaction(transaction)) {
			// 处理对当前事务的情况, 例如如果拦截方法定义的传播行为是NERVER的话会抛出异常
			return handleExistingTransaction(definition, transaction, debugEnabled);
		}

		// 核对事务行为超时参数
		if (definition.getTimeout() < TransactionDefinition.TIMEOUT_DEFAULT) {
			throw new InvalidTimeoutException("Invalid transaction timeout", definition.getTimeout());
		}

		// spring对MANDATORY传播机制的定义: 要求当前必须存在事务,不存在时抛出异常
		if (definition.getPropagationBehavior() == TransactionDefinition.PROPAGATION_MANDATORY) {
			throw new IllegalTransactionStateException(
					"No existing transaction found for transaction marked with propagation 'mandatory'");
		}
		// REQUIRED,REQUIRES_NEW,NESTED在当前没有事务的情况下会开启新事务
		else if (definition.getPropagationBehavior() == TransactionDefinition.PROPAGATION_REQUIRED ||
				definition.getPropagationBehavior() == TransactionDefinition.PROPAGATION_REQUIRES_NEW ||
				definition.getPropagationBehavior() == TransactionDefinition.PROPAGATION_NESTED) {
			// 将事务信息挂起,因为这里的方法传参是null所有不会挂起上一个事务
			SuspendedResourcesHolder suspendedResources = suspend(null);
			if (debugEnabled) {
				logger.debug("Creating new transaction with name [" + definition.getName() + "]: " + definition);
			}
			try {
				boolean newSynchronization = (getTransactionSynchronization() != SYNCHRONIZATION_NEVER);
				DefaultTransactionStatus status = newTransactionStatus(
						definition, transaction, true, newSynchronization, debugEnabled, suspendedResources);
				// 通过子类实现的模板方法来开启新事务
				doBegin(transaction, definition);
				// 将当前的事务信息通过放在ThreadLocal中
				prepareSynchronization(status, definition);
				return status;
			}
			catch (RuntimeException | Error ex) {
				// 发生异常恢复事务到上个状态
				resume(null, suspendedResources);
				throw ex;
			}
		}
		else {
			// 对于SUPPORTS,NOT_SUPPORTED,NEVER这些在不存在事务的情况下不需要创建事务的传播机制创建一个空的事务
			if (definition.getIsolationLevel() != TransactionDefinition.ISOLATION_DEFAULT && logger.isWarnEnabled()) {
				logger.warn("Custom isolation level specified but no actual transaction initiated; " +
						"isolation level will effectively be ignored: " + definition);
			}
			boolean newSynchronization = (getTransactionSynchronization() == SYNCHRONIZATION_ALWAYS);
			return prepareTransactionStatus(definition, null, true, newSynchronization, debugEnabled, null);
		}
	}
```
##### handleExistingTransaction获取事务时对于已存在事务的处理
```
private TransactionStatus handleExistingTransaction(
			TransactionDefinition definition, Object transaction, boolean debugEnabled)
			throws TransactionException {
		
		// spring对NEVER传播机制的定义: 不允许存在事务存在事务时抛出异常
		if (definition.getPropagationBehavior() == TransactionDefinition.PROPAGATION_NEVER) {
			throw new IllegalTransactionStateException(
					"Existing transaction found for transaction marked with propagation 'never'");
		}

		// spring对NOT_SUPPORT传播机制的定义: 如果当前存在事务的话就挂起当前事务
		if (definition.getPropagationBehavior() == TransactionDefinition.PROPAGATION_NOT_SUPPORTED) {
			if (debugEnabled) {
				logger.debug("Suspending current transaction");
			}
			// 挂起当前事务
			Object suspendedResources = suspend(transaction);
			boolean newSynchronization = (getTransactionSynchronization() == SYNCHRONIZATION_ALWAYS);
			// 将当前的事务信息通过存储在ThreadLocal中
			return prepareTransactionStatus(definition, null, false, newSynchronization, debugEnabled, suspendedResources);
		}

		// spring对REQUIRE_NEW传播机制的定义: 如果当前存在事务就挂起当前事务并创建新事务
		if (definition.getPropagationBehavior() == TransactionDefinition.PROPAGATION_REQUIRES_NEW) {
			if (debugEnabled) {
				logger.debug("Suspending current transaction, creating new transaction with name [" +
						definition.getName() + "]");
			}
			// 挂起当前存在的事务
			SuspendedResourcesHolder suspendedResources = suspend(transaction);
			try {
				boolean newSynchronization = (getTransactionSynchronization() != SYNCHRONIZATION_NEVER);
				DefaultTransactionStatus status = newTransactionStatus(
						definition, transaction, true, newSynchronization, debugEnabled, suspendedResources);
				// 通过子类实现的模板方法来开启新事务
				doBegin(transaction, definition);
				// 将当前的事务信息通过放在ThreadLocal中
				prepareSynchronization(status, definition);
				return status;
			}
			catch (RuntimeException | Error beginEx) {
				// 发生异常时恢复事务状态到上一个状态
				resumeAfterBeginException(transaction, suspendedResources, beginEx);
				throw beginEx;
			}
		}

		// spring对于NESTED传播机制的定义: 存在事务时创建一个数据库SavePoint
		if (definition.getPropagationBehavior() == TransactionDefinition.PROPAGATION_NESTED) {
			// AbstractPlatformTransactionManager的子类实现不允许创建SavePoint时抛出异常
			if (!isNestedTransactionAllowed()) {
				throw new NestedTransactionNotSupportedException(
						"Transaction manager does not allow nested transactions by default - " +
						"specify 'nestedTransactionAllowed' property with value 'true'");
			}
			if (debugEnabled) {
				logger.debug("Creating nested transaction with name [" + definition.getName() + "]");
			}
			if (useSavepointForNestedTransaction()) {
				// 如果AbstractPlatformTransactionManager的子类实现是可以创建SavePoint的话就创建save
				// DataSourceTransactionManager的实现是可以创建SavePoint的
				DefaultTransactionStatus status =
						prepareTransactionStatus(definition, transaction, false, false, debugEnabled, null);
				status.createAndHoldSavepoint();
				return status;
			}
			else {
				// 如果AbstractPlatformTransactionManager的子类实现是可以不创建SavePoint的话创建一个新的事务
				boolean newSynchronization = (getTransactionSynchronization() != SYNCHRONIZATION_NEVER);
				DefaultTransactionStatus status = newTransactionStatus(
						definition, transaction, true, newSynchronization, debugEnabled, null);
				// 通过子类实现的模板方法开始事务
				doBegin(transaction, definition);
				// 将当前的事务信息存储在ThreadLocal中
				prepareSynchronization(status, definition);
				return status;
			}
		}

		// 除了上面需要特殊处理的传播机制,其他传播机制当前存在事务时不需要创建新的事务,下面直接创建一个空事务
		if (debugEnabled) {
			logger.debug("Participating in existing transaction");
		}

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
		boolean newSynchronization = (getTransactionSynchronization() != SYNCHRONIZATION_NEVER);
		return prepareTransactionStatus(definition, transaction, false, newSynchronization, debugEnabled, null);
	}
```

前面说了AbstractPlatformTransactionManager是PlatformTransactionManager接口的骨架类跟模板类,让我们AbstractPlatformTransactionManager中定义了几个模板方法供子类实现吧
```
public abstract class AbstractPlatformTransactionManager implements PlatformTransactionManager, Serializable {
	/**
	*	获取当前事务信息
	*/
	protected abstract Object doGetTransaction() throws TransactionException;
	/**
	*	判断当前的事务信息是否已经存在事务
	*/
	protected boolean isExistingTransaction(Object transaction) throws TransactionException;
	/**
	*	事务是否运行使用数据库的SavePoint
	*/
	protected boolean useSavepointForNestedTransaction();
	/**
	*	开始事务
	*/
	protected abstract void doBegin(Object transaction, TransactionDefinition definition) throws TransactionException;
	/**
	*	挂起当前事务并返回被挂起的事务信息
	*/
	protected Object doSuspend(Object transaction) throws TransactionException;
	/**
	*	完成事务或者发生异常的时候恢复事务状态到上个状态
	*/
	protected void doResume(@Nullable Object transaction, Object suspendedResources) throws TransactionException;
	/**
	*	对提交事务做准备,AbstractPlatformTransactionManager的所有子类都没实现这个方法
	*/
	protected void prepareForCommit(DefaultTransactionStatus status);
	/**
	*	提交事务
	*/
	protected abstract void doCommit(DefaultTransactionStatus status) throws TransactionException;
	/**
	*	回滚事务
	*/
	protected abstract void doRollback(DefaultTransactionStatus status) throws TransactionException;
	/**
	*	设置当前事务为只读
	*/
	protected void doSetRollbackOnly(DefaultTransactionStatus status) throws TransactionException;
}
```
## 一个完整事务的AOP模型定义  
总所周知Spring的事务管理通过AOP来实现,但我一直来以来有个疑问,"为什么平时那些自定义注解的AOP都是需要通过@Aspect跟@Pointcut来定义一个切面的,为什么Spring的就不用".这也是我研究@Transactional注解的原因,带着这个疑问我接下去看吧.  

### TransactionInterceptor定义Spring Transaction AOP的执行代码
现在我们把眼光放在TransactionInterceptor的继承实现上,它除了实现上文说到的TransactionAspectSupport类之外还实现了MethodInterceptor跟Serializable接口,Serializable接口就不用看了它的作用大家都知道.MethodInterceptor是aop联盟定义的一个接口,spring有对aop联盟定义的接口类有做设配.它相当于我们平时使用@Around注解一样可以生成一个环绕通知.他需要跟org.aopalliance.aop.Advice跟org.springframework.aop.support.AbstractPointcutAdvisor一起使用来定义一个完整的切面,前者是判断一个方法是否要执行拦截后者是一个aop的完整定义.  
invoke方法是它环绕通知的实现,我们可以看到它获取了被代理的class之后就调用了TransactionAspectSupport的invokeWithinTransaction方法来进行事务管理.  
```
public Object invoke(MethodInvocation invocation) throws Throwable {
		// Work out the target class: may be {@code null}.
		// The TransactionAttributeSource should be passed the target class
		// as well as the method, which may be from an interface.
		Class<?> targetClass = (invocation.getThis() != null ? AopUtils.getTargetClass(invocation.getThis()) : null);

		// Adapt to TransactionAspectSupport's invokeWithinTransaction...
		return invokeWithinTransaction(invocation.getMethod(), targetClass, invocation::proceed);
	}
```


### TransactionAttributeSourcePointcut定义Spring Transaction的拦截规则
TransactionAttributeSourcePointcut继承了StaticMethodMatcherPointcut类,StaticMethodMatcherPointcut是Pointcut的抽象类,StaticMethodMatcherPointcut它可于静态方法的判断  
matches方法是它判断一个方法是否应该拦截的依据  
```
public boolean matches(Method method, Class<?> targetClass) {
		if (TransactionalProxy.class.isAssignableFrom(targetClass) ||
				PlatformTransactionManager.class.isAssignableFrom(targetClass) ||
				PersistenceExceptionTranslator.class.isAssignableFrom(targetClass)) {
			return false;
		}

		// 用TransactionAttributeSource来获取一个方法上是否有定义事务行为,如果有就进行拦截
		// AnnotationTransactionAttributeSource是TransactionAttributeSource的一个实现,它会读取@Transactional上的内容
		TransactionAttributeSource tas = getTransactionAttributeSource();
		return (tas == null || tas.getTransactionAttribute(method, targetClass) != null);
	}
```

### BeanFactoryTransactionAttributeSourceAdvisor定义一个完整的Aop模型
BeanFactoryTransactionAttributeSourceAdvisor继承了AbstractBeanFactoryPointcutAdvisor类,AbstractBeanFactoryPointcutAdvisor是AbstractPointcutAdvisor的子类.它定义了完整的AOP模型.
```
public abstract class AbstractPointcutAdvisor implements PointcutAdvisor, Ordered, Serializable  {
	/**
	* 定义一个AOP模型的拦截需要做什么
	*/
	Advice getAdvice();
	
	/**
	* 判断一个方法是否要拦截
	*/
	Pointcut getPointcut();
}
```

## 总结
@Transactional定义了一个事务行为,它可以定义在methond跟class上,如果定义在class上的话该class上的所有方法的默认行为就是class上定义的@Transactional.  
AnnotationTransactionAttributeSource读取我们在method跟class上定义的事务行为转换成RuleBasedTransactionAttribute保存起来.  
TransactionAspectSupport的invokeWithinTransaction方法具体实现  
BeanFactoryTransactionAttributeSourceAdvisor定义了一个完整的AOP模型拦截,TransactionInterceptor是AOP模型的具体实现,TransactionAttributeSourcePointcut是判断一个方法是否需要拦截的依据

## 笔者信息
- wechat:359518081
- github: https://github.com/wkx359518081