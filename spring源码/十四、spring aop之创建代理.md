接上一节[十三、spring aop之查找匹配的增强器](https://www.jianshu.com/p/daed8d3d50c0),在获取所有对应bean的增强器后，便可以进行代理的创建。

# 初始化ProxyFactory
AbstractAutoProxyCreator#createProxy
```java
protected Object createProxy(
        Class<?> beanClass, String beanName, Object[] specificInterceptors, TargetSource targetSource) {

    if (this.beanFactory instanceof ConfigurableListableBeanFactory) {
        AutoProxyUtils.exposeTargetClass((ConfigurableListableBeanFactory) this.beanFactory, beanName, beanClass);
    }

    ProxyFactory proxyFactory = new ProxyFactory();
    //获取当前类中相关属性
    proxyFactory.copyFrom(this);

    //决定对于给定的bean是否应该使用targetClass而不是它的接口代理
    if (!proxyFactory.isProxyTargetClass()) {
        if (shouldProxyTargetClass(beanClass, beanName)) {
            proxyFactory.setProxyTargetClass(true);
        }
        else {
            evaluateProxyInterfaces(beanClass, proxyFactory);
        }
    }

    //拦截器和增强器结合
    Advisor[] advisors = buildAdvisors(beanName, specificInterceptors);
    proxyFactory.addAdvisors(advisors);
    //设置目标类
    proxyFactory.setTargetSource(targetSource);
    customizeProxyFactory(proxyFactory);

    proxyFactory.setFrozen(this.freezeProxy);
    if (advisorsPreFiltered()) {
        proxyFactory.setPreFiltered(true);
    }
    //创建代理
    return proxyFactory.getProxy(getProxyClassLoader());
}
```
对于代理类的创建及处理，Spring委拖给了ProxyFacotry去处理，而在createProxy方法中主要是对ProxyFactory的初始化操作，进而对真正的创建代理做准备，这些初始化操作包括如下内容：
（1）获取当前类的属性
```java
public void copyFrom(ProxyConfig other) {
    Assert.notNull(other, "Other ProxyConfig object must not be null");
    this.proxyTargetClass = other.proxyTargetClass;
    this.optimize = other.optimize;
    this.exposeProxy = other.exposeProxy;
    this.frozen = other.frozen;
    this.opaque = other.opaque;
}
```
(2) 决定对于给定的bean是否应该使用targetClass而不是它的接口代理,这个主要是在@EnableAspectJAutoProxy(proxyTargetClass = true)配置，如果proxyTargetClass=true，表示使用cglib代理，否则就得看代理类是否实现了接口。
(3) 结合拦截器和增强器
```java
protected Advisor[] buildAdvisors(String beanName, Object[] specificInterceptors) {
    // Handle prototypes correctly...
    //解析拦截器，这个拦截器的值我现在还不知道怎么设置
    Advisor[] commonInterceptors = resolveInterceptorNames();

    List<Object> allInterceptors = new ArrayList<Object>();
    if (specificInterceptors != null) {
        //增强器加入到拦截器中
        allInterceptors.addAll(Arrays.asList(specificInterceptors));
        if (commonInterceptors.length > 0) {
            if (this.applyCommonInterceptorsFirst) {
                allInterceptors.addAll(0, Arrays.asList(commonInterceptors));
            }
            else {
                allInterceptors.addAll(Arrays.asList(commonInterceptors));
            }
        }
    }
    if (logger.isDebugEnabled()) {
        int nrOfCommonInterceptors = commonInterceptors.length;
        int nrOfSpecificInterceptors = (specificInterceptors != null ? specificInterceptors.length : 0);
        logger.debug("Creating implicit proxy for bean '" + beanName + "' with " + nrOfCommonInterceptors +
                " common interceptors and " + nrOfSpecificInterceptors + " specific interceptors");
    }

    Advisor[] advisors = new Advisor[allInterceptors.size()];
    //遍历所有的拦截器，并将拦截器转换成Advisor
    for (int i = 0; i < allInterceptors.size(); i++) {
        advisors[i] = this.advisorAdapterRegistry.wrap(allInterceptors.get(i));
    }
    return advisors;
}

@Override
public Advisor wrap(Object adviceObject) throws UnknownAdviceTypeException {
    //如果要封装的对象本身就是Advisor类型的那么无需再包装
    if (adviceObject instanceof Advisor) {
        return (Advisor) adviceObject;
    }
    //只能封装Advice和Advisor类型
    if (!(adviceObject instanceof Advice)) {
        throw new UnknownAdviceTypeException(adviceObject);
    }
    Advice advice = (Advice) adviceObject;
    if (advice instanceof MethodInterceptor) {
        // So well-known it doesn't even need an adapter.
        return new DefaultPointcutAdvisor(advice);
    }
    for (AdvisorAdapter adapter : this.adapters) {
        // Check that it is supported.
        if (adapter.supportsAdvice(advice)) {
            return new DefaultPointcutAdvisor(advice);
        }
    }
    throw new UnknownAdviceTypeException(advice);
}
```
(4) 设置目标类
(5) 进行获取代理操作
# 创建代理
ProxyCreatorSupport#createAopProxy
```java
protected final synchronized AopProxy createAopProxy() {
    if (!this.active) {
        activate();
    }
    //创建代理
    return getAopProxyFactory().createAopProxy(this);
}

@Override
public AopProxy createAopProxy(AdvisedSupport config) throws AopConfigException {
    if (config.isOptimize() || config.isProxyTargetClass() || hasNoUserSuppliedProxyInterfaces(config)) {
        Class<?> targetClass = config.getTargetClass();
        if (targetClass == null) {
            throw new AopConfigException("TargetSource cannot determine target class: " +
                    "Either an interface or a target is required for proxy creation.");
        }
        if (targetClass.isInterface() || Proxy.isProxyClass(targetClass)) {
            return new JdkDynamicAopProxy(config);
        }
        //cglib动态代理
        return new ObjenesisCglibAopProxy(config);
    }
    else {
        //jdk动态代理
        return new JdkDynamicAopProxy(config);
    }
}
```
从if中的判断条件可以看到使用cglib需要满足三个条件中的一个：
> Optimize：用来控制通过CGLIB创建的代理是否使用激化的优化策略。除非完全了解AOP代理如何处理优化，否则不推荐用户使用这个设置。目前这个属性仅用于CGLIB代理，对于JDK动态代理无效。(我也没找到在哪设置)。
ProxyTargetClass：这个属性为true时，目标类本身被代理而不是目标类的接口(直接继承目标类而不是实现目标类的接口)，如果这个属性值设置为true，CGLIB代理将被创建。
hasNoUserSuppliedProxyInterfaces：是否存在代理接口.

下面对JDK与CGLIB方式的总结：
- 如果目标类实现了接口，默认情况下会采用JDK的动态代理实现AOP。
- 如果目标对象实现了接口，可以通过ProxyTargetClass强制使用CGLIB实现AOP。
- 如果目标对象没有实现了接口，必须采用CGLIB，Spring会自动在JDK动态代理和CGLIB之间切换。

## spring JDK动态代理实现
我们在[九、spring aop之jdk动态代理](https://www.jianshu.com/p/4a8c50030865)讲了实现JDK动态代理要做两件事：
1. 实现InvocationHandler，重写invoke方法实现代理逻辑
2. 使用Proxy.newProxyInstance生成代理对象

那么spring中的jdk代理是如何实现的呢？我们查看JdkDynamicAopProxy类。
JdkDynamicAopProxy实现了InvocationHandler接口。
```java
final class JdkDynamicAopProxy implements AopProxy, InvocationHandler, Serializable
```
spring的代理对象通过调用AopPorxy#getProxy方法获取,而JdkDynamicAopProxy实现了AopPorxy，看下getProxy方法的实现逻辑，调用了Proxy.newProxyInstance创建代理对象。
```java
@Override
public Object getProxy(ClassLoader classLoader) {
    if (logger.isDebugEnabled()) {
        logger.debug("Creating JDK dynamic proxy: target source is " + this.advised.getTargetSource());
    }
    Class<?>[] proxiedInterfaces = AopProxyUtils.completeProxiedInterfaces(this.advised, true);
    findDefinedEqualsAndHashCodeMethods(proxiedInterfaces);
    //生成代理对象
    return Proxy.newProxyInstance(classLoader, proxiedInterfaces, this);
}
```
再看invoke方法。
```java
@Override
public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
    MethodInvocation invocation;
    Object oldProxy = null;
    boolean setProxyContext = false;

    TargetSource targetSource = this.advised.targetSource;
    Class<?> targetClass = null;
    Object target = null;

    try {
        //equals方法的处理
        if (!this.equalsDefined && AopUtils.isEqualsMethod(method)) {
            // The target does not implement the equals(Object) method itself.
            return equals(args[0]);
        }
        //hashCode方法处理
        else if (!this.hashCodeDefined && AopUtils.isHashCodeMethod(method)) {
            // The target does not implement the hashCode() method itself.
            return hashCode();
        }
        else if (method.getDeclaringClass() == DecoratingProxy.class) {
            // There is only getDecoratedClass() declared -> dispatch to proxy config.
            return AopProxyUtils.ultimateTargetClass(this.advised);
        }
        else if (!this.advised.opaque && method.getDeclaringClass().isInterface() &&
                method.getDeclaringClass().isAssignableFrom(Advised.class)) {
            // Service invocations on ProxyConfig with the proxy config...
            return AopUtils.invokeJoinpointUsingReflection(this.advised, method, args);
        }

        Object retVal;

        //有时候目标对象内部的自我调用将无法实施切面中的增强则需要通过此属性暴露代理,这个再讲事物不生效时会讲用法
        if (this.advised.exposeProxy) {
            // Make invocation available if necessary.
            oldProxy = AopContext.setCurrentProxy(proxy);
            setProxyContext = true;
        }

        // May be null. Get as late as possible to minimize the time we "own" the target,
        // in case it comes from a pool.
        target = targetSource.getTarget();
        if (target != null) {
            targetClass = target.getClass();
        }

        // Get the interception chain for this method.
        //获取当前方法的拦截器链
        List<Object> chain = this.advised.getInterceptorsAndDynamicInterceptionAdvice(method, targetClass);

        // Check whether we have any advice. If we don't, we can fallback on direct
        // reflective invocation of the target, and avoid creating a MethodInvocation.
        if (chain.isEmpty()) {
            // We can skip creating a MethodInvocation: just invoke the target directly
            // Note that the final invoker must be an InvokerInterceptor so we know it does
            // nothing but a reflective operation on the target, and no hot swapping or fancy proxying.
            Object[] argsToUse = AopProxyUtils.adaptArgumentsIfNecessary(method, args);
            retVal = AopUtils.invokeJoinpointUsingReflection(target, method, argsToUse);
        }
        else {
            // We need to create a method invocation...
            //将拦截器封装在ReflectiveMethodInvocation，调用proceed方法执行拦截器链 
            invocation = new ReflectiveMethodInvocation(proxy, target, method, args, targetClass, chain);
            // Proceed to the joinpoint through the interceptor chain.
            retVal = invocation.proceed();
        }

        // Massage return value if necessary.
        Class<?> returnType = method.getReturnType();
        if (retVal != null && retVal == target &&
                returnType != Object.class && returnType.isInstance(proxy) &&
                !RawTargetAccess.class.isAssignableFrom(method.getDeclaringClass())) {
            // Special case: it returned "this" and the return type of the method
            // is type-compatible. Note that we can't help if the target sets
            // a reference to itself in another returned object.
            retVal = proxy;
        }
        else if (retVal == null && returnType != Void.TYPE && returnType.isPrimitive()) {
            throw new AopInvocationException(
                    "Null return value from advice does not match primitive return type for: " + method);
        }
        return retVal;
    }
    finally {
        if (target != null && !targetSource.isStatic()) {
            // Must have come from TargetSource.
            targetSource.releaseTarget(target);
        }
        if (setProxyContext) {
            // Restore old proxy.
            AopContext.setCurrentProxy(oldProxy);
        }
    }
}
```
上面的方法中最主要的工作是创建一个拦截器链，并使用ReflectiveMethodInvocation类进行了链的封装，

## spring CGLIB动态代理实现原理
在[九、spring aop之jdk动态代理](https://www.jianshu.com/p/4a8c50030865)我们讲了实现cglib动态代理也需要做两件事：
1. 实现MethodInterceptor接口，重写intercept方法。
2. 使用Enhancer创建代理对象。

开始我以为cglib会和jdk一样，ObjenesisCglibAopProxy会实现MethodInterceptor接口，但是并没有，而且父类也没有实现。所以我先看到getProxy()方法，在创建Enhancer中enhancer.setCallbacks(callbacks)中callbacks对象是什么?
```java
@Override
public Object getProxy(ClassLoader classLoader) {
    if (logger.isDebugEnabled()) {
        logger.debug("Creating CGLIB proxy: target source is " + this.advised.getTargetSource());
    }

    try {
        Class<?> rootClass = this.advised.getTargetClass();
        Assert.state(rootClass != null, "Target class must be available for creating a CGLIB proxy");

        Class<?> proxySuperClass = rootClass;
        if (ClassUtils.isCglibProxyClass(rootClass)) {
            proxySuperClass = rootClass.getSuperclass();
            Class<?>[] additionalInterfaces = rootClass.getInterfaces();
            for (Class<?> additionalInterface : additionalInterfaces) {
                this.advised.addInterface(additionalInterface);
            }
        }

        // Validate the class, writing log messages as necessary.
        validateClassIfNecessary(proxySuperClass, classLoader);

        // Configure CGLIB Enhancer...
        Enhancer enhancer = createEnhancer();
        if (classLoader != null) {
            enhancer.setClassLoader(classLoader);
            if (classLoader instanceof SmartClassLoader &&
                    ((SmartClassLoader) classLoader).isClassReloadable(proxySuperClass)) {
                enhancer.setUseCache(false);
            }
        }
        enhancer.setSuperclass(proxySuperClass);
        enhancer.setInterfaces(AopProxyUtils.completeProxiedInterfaces(this.advised));
        enhancer.setNamingPolicy(SpringNamingPolicy.INSTANCE);
        enhancer.setStrategy(new ClassLoaderAwareUndeclaredThrowableStrategy(classLoader));

        Callback[] callbacks = getCallbacks(rootClass);
        Class<?>[] types = new Class<?>[callbacks.length];
        for (int x = 0; x < types.length; x++) {
            types[x] = callbacks[x].getClass();
        }
        // fixedInterceptorMap only populated at this point, after getCallbacks call above
        enhancer.setCallbackFilter(new ProxyCallbackFilter(
                this.advised.getConfigurationOnlyCopy(), this.fixedInterceptorMap, this.fixedInterceptorOffset));
        enhancer.setCallbackTypes(types);

        // Generate the proxy class and create a proxy instance.
        return createProxyClassAndInstance(enhancer, callbacks);
    }
    catch (CodeGenerationException ex) {
        throw new AopConfigException("Could not generate CGLIB subclass of " + this.advised.getTargetClass() +
                ": Common causes of this problem include using a final class or a non-visible class",
                ex);
    }
    catch (IllegalArgumentException ex) {
        throw new AopConfigException("Could not generate CGLIB subclass of " + this.advised.getTargetClass() +
                ": Common causes of this problem include using a final class or a non-visible class",
                ex);
    }
    catch (Throwable ex) {
        // TargetSource.getTarget() failed
        throw new AopConfigException("Unexpected AOP exception", ex);
    }
}
```
上面代理逻辑不难，和我们自定义cglib逻辑一样，主要是getCallbacks(rootClass)不一样，我们看下它的实现：
```java
private Callback[] getCallbacks(Class<?> rootClass) throws Exception {
    // Parameters used for optimization choices...
    //@EnableAspectJAutoProxy(exposeProxy = true)中exposeProxy的值
    boolean exposeProxy = this.advised.isExposeProxy();
    //没看到哪里设置，默认false
    boolean isFrozen = this.advised.isFrozen();
    //目标类是否是静态，aop创建的是SingletonTargetSource对象，返回true
    boolean isStatic = this.advised.getTargetSource().isStatic();

    // Choose an "aop" interceptor (used for AOP calls).
    //我们主要研究这个对象
    Callback aopInterceptor = new DynamicAdvisedInterceptor(this.advised);

    // Choose a "straight to target" interceptor. (used for calls that are
    // unadvised but can return this). May be required to expose the proxy.
    Callback targetInterceptor;
    if (exposeProxy) {
        targetInterceptor = isStatic ?
                new StaticUnadvisedExposedInterceptor(this.advised.getTargetSource().getTarget()) :
                new DynamicUnadvisedExposedInterceptor(this.advised.getTargetSource());
    }
    else {
        targetInterceptor = isStatic ?
                new StaticUnadvisedInterceptor(this.advised.getTargetSource().getTarget()) :
                new DynamicUnadvisedInterceptor(this.advised.getTargetSource());
    }

    // Choose a "direct to target" dispatcher (used for
    // unadvised calls to static targets that cannot return this).
    Callback targetDispatcher = isStatic ?
            new StaticDispatcher(this.advised.getTargetSource().getTarget()) : new SerializableNoOp();

    Callback[] mainCallbacks = new Callback[] {
            aopInterceptor,  // for normal advice
            targetInterceptor,  // invoke target without considering advice, if optimized
            new SerializableNoOp(),  // no override for methods mapped to this
            targetDispatcher, this.advisedDispatcher,
            new EqualsInterceptor(this.advised),
            new HashCodeInterceptor(this.advised)
    };

    Callback[] callbacks;

    // If the target is a static one and the advice chain is frozen,
    // then we can make some optimizations by sending the AOP calls
    // direct to the target using the fixed chain for that method.
    if (isStatic && isFrozen) {
        Method[] methods = rootClass.getMethods();
        Callback[] fixedCallbacks = new Callback[methods.length];
        this.fixedInterceptorMap = new HashMap<String, Integer>(methods.length);

        // TODO: small memory optimization here (can skip creation for methods with no advice)
        for (int x = 0; x < methods.length; x++) {
            List<Object> chain = this.advised.getInterceptorsAndDynamicInterceptionAdvice(methods[x], rootClass);
            fixedCallbacks[x] = new FixedChainStaticTargetInterceptor(
                    chain, this.advised.getTargetSource().getTarget(), this.advised.getTargetClass());
            this.fixedInterceptorMap.put(methods[x].toString(), x);
        }

        // Now copy both the callbacks from mainCallbacks
        // and fixedCallbacks into the callbacks array.
        callbacks = new Callback[mainCallbacks.length + fixedCallbacks.length];
        System.arraycopy(mainCallbacks, 0, callbacks, 0, mainCallbacks.length);
        System.arraycopy(fixedCallbacks, 0, callbacks, mainCallbacks.length, fixedCallbacks.length);
        this.fixedInterceptorOffset = mainCallbacks.length;
    }
    else {
        callbacks = mainCallbacks;
    }
    return callbacks;
}
```
上面这个方法主要是根据各种条件创建不同的Callback对象，对于Aop我们主要研究DynamicAdvisedInterceptor，它实现了MethodInterceptor接口，我们看下它的intercept实现逻辑。
```java
@Override
public Object intercept(Object proxy, Method method, Object[] args, MethodProxy methodProxy) throws Throwable {
    Object oldProxy = null;
    boolean setProxyContext = false;
    Class<?> targetClass = null;
    Object target = null;
    try {
        if (this.advised.exposeProxy) {
            // Make invocation available if necessary.
            oldProxy = AopContext.setCurrentProxy(proxy);
            setProxyContext = true;
        }
        // May be null. Get as late as possible to minimize the time we
        // "own" the target, in case it comes from a pool...
        target = getTarget();
        if (target != null) {
            targetClass = target.getClass();
        }
        //封装调用链
        List<Object> chain = this.advised.getInterceptorsAndDynamicInterceptionAdvice(method, targetClass);
        Object retVal;
        // Check whether we only have one InvokerInterceptor: that is,
        // no real advice, but just reflective invocation of the target.
        if (chain.isEmpty() && Modifier.isPublic(method.getModifiers())) {
            // We can skip creating a MethodInvocation: just invoke the target directly.
            // Note that the final invoker must be an InvokerInterceptor, so we know
            // it does nothing but a reflective operation on the target, and no hot
            // swapping or fancy proxying.
            Object[] argsToUse = AopProxyUtils.adaptArgumentsIfNecessary(method, args);
            retVal = methodProxy.invoke(target, argsToUse);
        }
        else {
            // We need to create a method invocation...
            //创建CglibMethodInvocation对象，逐一调用调用链
            retVal = new CglibMethodInvocation(proxy, target, method, args, targetClass, chain, methodProxy).proceed();
        }
        retVal = processReturnType(proxy, target, method, retVal);
        return retVal;
    }
    finally {
        if (target != null) {
            releaseTarget(target);
        }
        if (setProxyContext) {
            // Restore old proxy.
            AopContext.setCurrentProxy(oldProxy);
        }
    }
}
```
intercept方法和jdk动态代理的invoke方法实现逻辑是一模一样，也是先封装调用链，然后逐一调用调用链，不过jdk动态代理使用的是ReflectiveMethodInvocation对象，而cglib使用的是CglibMethodInvocation。不过ReflectiveMethodInvocation和CglibMethodInvocation中的proceed方法实现逻辑是一样的。

# 参考资料
- 《Spring源码深度解析》