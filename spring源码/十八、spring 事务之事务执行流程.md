接上一节[十六、spring事物之@EnableTransactionManagement](https://www.jianshu.com/p/c53e169e9e86)。在spring aop中我们讲到spring会把Adivsor中的Advice转换成拦截器链，然后去调用。在上节中spring事务创建了一个BeanFactoryTransactionAttributeSourceAdvisor，并把TransactionInterceptor注入进去，而TransactionInterceptor实现了Advice接口。所以这节分析TransactionInterceptor是如何管理事务的。
![TransactionInterceptor](https://upload-images.jianshu.io/upload_images/10236819-dcba101d81f3b0db.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
由类图看出，TransactionInterceptor实现了MethodInterceptor接口，那么逻辑处理就会放在invoke方法中。
```java
@Override
public Object invoke(final MethodInvocation invocation) throws Throwable {
    // Work out the target class: may be {@code null}.
    // The TransactionAttributeSource should be passed the target class
    // as well as the method, which may be from an interface.
    Class<?> targetClass = (invocation.getThis() != null ? AopUtils.getTargetClass(invocation.getThis()) : null);

    // Adapt to TransactionAspectSupport's invokeWithinTransaction...
    //调用父类的invokeWithinTransaction方法
    return invokeWithinTransaction(invocation.getMethod(), targetClass, new InvocationCallback() {
        @Override
        public Object proceedWithInvocation() throws Throwable {
            //执行下一个调用链
            return invocation.proceed(); 
        }
    });
}
```
invoke方法把实现逻辑交给父类的invokeWithinTransaction，并利用回调的方式执行下一个调用链。invokeWithinTransaction的实现逻辑如下，代码粘贴最常用的部分:
```java
protected Object invokeWithinTransaction(Method method, Class<?> targetClass, final InvocationCallback invocation)
			throws Throwable {

    // If the transaction attribute is null, the method is non-transactional.
    //1. 获取对应事务属性
    final TransactionAttribute txAttr = getTransactionAttributeSource().getTransactionAttribute(method, targetClass);
    //2. 获取TransactionManager
    final PlatformTransactionManager tm = determineTransactionManager(txAttr);
    final String joinpointIdentification = methodIdentification(method, targetClass, txAttr);

    if (txAttr == null || !(tm instanceof CallbackPreferringPlatformTransactionManager)) {
        // Standard transaction demarcation with getTransaction and commit/rollback calls.
        //3. 创建TransactionInfo
        TransactionInfo txInfo = createTransactionIfNecessary(tm, txAttr, joinpointIdentification);
        Object retVal = null;
        try {
            // This is an around advice: Invoke the next interceptor in the chain.
            // This will normally result in a target object being invoked.
            //回调执行下一个调用链
            retVal = invocation.proceedWithInvocation();
        }
        catch (Throwable ex) {
            // target invocation exception
            //异常回滚
            completeTransactionAfterThrowing(txInfo, ex);
            throw ex;
        }
        finally {
            //清除事务信息
            cleanupTransactionInfo(txInfo);
        }
        //提交事务
        commitTransactionAfterReturning(txInfo);
        return retVal;
    }
}
```
invokeWithinTransaction代码逻辑非常清晰，这里不得不夸赞下spring的代码写的真好，见名知意、逻辑清晰。上面方法的逻辑如下：
1. 获取对应事务属性，也就是获取@Transactional注解上的属性
2. 获取TransactionManager，常用的如DataSourceTransactionManager事务管理
3. 在目标方法执行前获取事务并收集事务信息
4. 回调执行下一个调用链。
5. 一旦出现异常，尝试异常处理
6. 提交事务前的事务信息清理。
7. 提交事务。

我们按照流程分析:
##　获取对应事务属性
```java
final TransactionAttribute txAttr = getTransactionAttributeSource().getTransactionAttribute(method, targetClass);
```
getTransactionAttributeSource()获得的对象是在ProxyTransactionManagementConfiguration创建bean时注入的AnnotationTransactionAttributeSource对象。 AnnotationTransactionAttributeSource中getTransactionAttributeSource方法主要逻辑交给了computeTransactionAttribute方法，所以我们直接看computeTransactionAttribute代码实现。
```java
protected TransactionAttribute computeTransactionAttribute(Method method, Class<?> targetClass) {
    // Don't allow no-public methods as required.
    //1. allowPublicMethodsOnly()返回true，只能是公共方法
    if (allowPublicMethodsOnly() && !Modifier.isPublic(method.getModifiers())) {
        return null;
    }

    // Ignore CGLIB subclasses - introspect the actual user class.
    Class<?> userClass = ClassUtils.getUserClass(targetClass);
    // The method may be on an interface, but we need attributes from the target class.
    // If the target class is null, the method will be unchanged.
    //method代表接口中的方法、specificMethod代表实现类的方法
    Method specificMethod = ClassUtils.getMostSpecificMethod(method, userClass);
    // If we are dealing with method with generic parameters, find the original method.
    //处理泛型
    specificMethod = BridgeMethodResolver.findBridgedMethod(specificMethod);

    // First try is the method in the target class.
    //查看方法中是否存在事务
    TransactionAttribute txAttr = findTransactionAttribute(specificMethod);
    if (txAttr != null) {
        return txAttr;
    }

    // Second try is the transaction attribute on the target class.
    //查看方法所在类是否存在事务声明
    txAttr = findTransactionAttribute(specificMethod.getDeclaringClass());
    if (txAttr != null && ClassUtils.isUserLevelMethod(method)) {
        return txAttr;
    }

    //如果存在接口，则在接口中查找
    if (specificMethod != method) {
        // Fallback is to look at the original method.
        //查找接口方法
        txAttr = findTransactionAttribute(method);
        if (txAttr != null) {
            return txAttr;
        }
        // Last fallback is the class of the original method.
        //到接口类中寻找
        txAttr = findTransactionAttribute(method.getDeclaringClass());
        if (txAttr != null && ClassUtils.isUserLevelMethod(method)) {
            return txAttr;
        }
    }

    return null;
}
```
computeTransactionAttribute方法执行的逻辑是：
1. 判断是不是只运行公共方法，在AnnotationTransactionAttributeSource构造方法中传入true。若方法不是公共方法，则返回null。
2. 得到具体的方法，method方法可能是接口方法或者泛型方法。
3. 查看方法上是否存在事务
4. 查看方法所在类上是否存在事务
5. 查看接口的方法是否存在事务，查看接口上是否存在事务。

所以如果一个方法上用了@Transactional，类上和接口上也用了，以方法上的为主，其次才是类，最后才到接口。findTransactionAttribute方法的细节这里就不再描述了。
##  获取TransactionManager
determineTransactionManager方法逻辑:
```java
protected PlatformTransactionManager determineTransactionManager(TransactionAttribute txAttr) {
    // Do not attempt to lookup tx manager if no tx attributes are set
    if (txAttr == null || this.beanFactory == null) {
        return getTransactionManager();
    }
    String qualifier = txAttr.getQualifier();
    if (StringUtils.hasText(qualifier)) {
        return determineQualifiedTransactionManager(qualifier);
    }
    else if (StringUtils.hasText(this.transactionManagerBeanName)) {
        return determineQualifiedTransactionManager(this.transactionManagerBeanName);
    }
    else {
        //常用的会走到这里
        PlatformTransactionManager defaultTransactionManager = getTransactionManager();
        if (defaultTransactionManager == null) {
            defaultTransactionManager = this.transactionManagerCache.get(DEFAULT_TRANSACTION_MANAGER_KEY);
            if (defaultTransactionManager == null) {
                //从beanFactory获取PlatformTransactionManager类型的bean
                defaultTransactionManager = this.beanFactory.getBean(PlatformTransactionManager.class);
                this.transactionManagerCache.putIfAbsent(
                        DEFAULT_TRANSACTION_MANAGER_KEY, defaultTransactionManager);
            }
        }
        return defaultTransactionManager;
    }
}
```
determineTransactionManager方法常用的都是从beanFactory中获取，数据源的方式通过下面方式注册：
```java
@Bean
public PlatformTransactionManager txManager() {
    return new DataSourceTransactionManager(dataSource());
}
```