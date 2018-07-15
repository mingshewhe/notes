接上一节[十二、spring aop之aop实现原理(二)](https://www.jianshu.com/p/6156bcc2cd84), 上一节我们分析了spring是如何查找所有的Advisor(增强器),这节我们分析匹配当前bean的Advisor(增强器),只有有匹配的才需要创建代理。实现逻辑还是在AbstractAdvisorAutoProxyCreator#findEligibleAdvisors(Class<?> beanClass, String beanName)中。
findEligibleAdvisors方法的实现代码如下:
```java
protected List<Advisor> findEligibleAdvisors(Class<?> beanClass, String beanName) {
    //1. 查找所有的增强器
    List<Advisor> candidateAdvisors = findCandidateAdvisors();
    //2. 找到匹配当前bean的增强器
    List<Advisor> eligibleAdvisors = findAdvisorsThatCanApply(candidateAdvisors, beanClass, beanName);
    extendAdvisors(eligibleAdvisors);
    if (!eligibleAdvisors.isEmpty()) {
        eligibleAdvisors = sortAdvisors(eligibleAdvisors);
    }
    return eligibleAdvisors;
}
```
不过这节我们分析的是findAdvisorsThatCanApply(candidateAdvisors, beanClass, beanName)方法。
# findAdvisorsThatCanApply
```java
protected List<Advisor> findAdvisorsThatCanApply(
        List<Advisor> candidateAdvisors, Class<?> beanClass, String beanName) {

    ProxyCreationContext.setCurrentProxiedBeanName(beanName);
    try {
        //过滤已经得到的Advisors
        return AopUtils.findAdvisorsThatCanApply(candidateAdvisors, beanClass);
    }
    finally {
        ProxyCreationContext.setCurrentProxiedBeanName(null);
    }
}
```
AopUtils.findAdvisorsThatCanApply()实现逻辑：
```java
public static List<Advisor> findAdvisorsThatCanApply(List<Advisor> candidateAdvisors, Class<?> clazz) {
    if (candidateAdvisors.isEmpty()) {
        return candidateAdvisors;
    }
    List<Advisor> eligibleAdvisors = new LinkedList<Advisor>();
    for (Advisor candidate : candidateAdvisors) {
        if (candidate instanceof IntroductionAdvisor && canApply(candidate, clazz)) {
            eligibleAdvisors.add(candidate);
        }
    }
    boolean hasIntroductions = !eligibleAdvisors.isEmpty();
    for (Advisor candidate : candidateAdvisors) {
        if (candidate instanceof IntroductionAdvisor) {
            // already processed
            continue;
        }
        //调试代码执行这一行，IntroductionAdvisor增强器我也不知道在什么时候使用
        if (canApply(candidate, clazz, hasIntroductions)) {
            eligibleAdvisors.add(candidate);
        }
    }
    return eligibleAdvisors;
}
```
findAdvisorsThatCanApply方法的主要功能是寻找所有Advisor(增强器)中适用于当前class的Advisor,而对于真正的匹配在canApply方法中实现。
```java
public static boolean canApply(Advisor advisor, Class<?> targetClass, boolean hasIntroductions) {
    if (advisor instanceof IntroductionAdvisor) {
        return ((IntroductionAdvisor) advisor).getClassFilter().matches(targetClass);
    }
    //spring aop生成的增强器是InstantiationModelAwarePointcutAdvisorImpl对象，实现了PointcutAdvisor
    else if (advisor instanceof PointcutAdvisor) {
        PointcutAdvisor pca = (PointcutAdvisor) advisor;
        return canApply(pca.getPointcut(), targetClass, hasIntroductions);
    }
    else {
        // It doesn't have a pointcut so we assume it applies.
        return true;
    }
}

public static boolean canApply(Pointcut pc, Class<?> targetClass, boolean hasIntroductions) {
    Assert.notNull(pc, "Pointcut must not be null");
    //匹配类
    if (!pc.getClassFilter().matches(targetClass)) {
        return false;
    }

    //匹配方法
    MethodMatcher methodMatcher = pc.getMethodMatcher();
    if (methodMatcher == MethodMatcher.TRUE) {
        // No need to iterate the methods if we're matching any method anyway...
        return true;
    }

    IntroductionAwareMethodMatcher introductionAwareMethodMatcher = null;
    if (methodMatcher instanceof IntroductionAwareMethodMatcher) {
        introductionAwareMethodMatcher = (IntroductionAwareMethodMatcher) methodMatcher;
    }

    Set<Class<?>> classes = new LinkedHashSet<Class<?>>(ClassUtils.getAllInterfacesForClassAsSet(targetClass));
    classes.add(targetClass);
    for (Class<?> clazz : classes) {
        //遍历所有方法，找到匹配的方法就返回
        Method[] methods = ReflectionUtils.getAllDeclaredMethods(clazz);
        for (Method method : methods) {
            if ((introductionAwareMethodMatcher != null &&
                    introductionAwareMethodMatcher.matches(method, targetClass, hasIntroductions)) ||
                    methodMatcher.matches(method, targetClass)) {
                return true;
            }
        }
    }

    return false;
}
```
匹配规则是，先匹配类，再遍历方法是否有匹配的。canApply方法是一个短路操作，只要找到一个匹配就返回true，说明bean是需要被代理的。这里我不分析spring是如何执行匹配的，阅读源码得有个度，我们只需要知道spring给我们规定的规则就好，我们切点匹配使用最多的是execution。接下来分析execution规则。
# execution规则
execution格式如下
```java
//execution(<修饰符模式>?<返回类型模式><方法名模式>(<参数模式>)<异常模式>?)
execution(modifiers-pattern? ret-type-pattern declaring-type-pattern?name-pattern(param-pattern)
            throws-pattern?)
```
除了返回类型模式（上面代码片段中的ret-type模式）、方法名模式和参数模式之外，所有部分都是可选的，通配符*表示所有的。
下面有一些示例：
1. 匹配所有public修饰的方法
```java
execution(public * *(..))
```
2. 匹配所有set开头的方法
```java
execution(* set*(..))
```
3. 匹配AccountService接口所有的方法
```java
execution(* com.xyz.service.AccountService.*(..))
```
4. 匹配service包下所有的方法
```java
execution(* com.xyz.service.*.*(..))
```
5. 匹配service包以及子包所有方法
```java
execution(* com.xyz.service..*.*(..))
```