接上一节[十一、spring aop之aop实现原理(一)](https://www.jianshu.com/p/18194f639625),这节我们分析spring是如何查找Advisor(增强器),实现逻辑在AbstractAdvisorAutoProxyCreator#findEligibleAdvisors(Class<?> beanClass, String beanName)中。
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
这节我们主要分析findCandidateAdvisors方法。
# findCandidateAdvisors方法
AnnotationAwareAspectJAutoProxyCreator重写了findCandidateAdvisors方法，AnnotationAwareAspectJAutoProxyCreator的实现如下:
```java
@Override
protected List<Advisor> findCandidateAdvisors() {
    // Add all the Spring advisors found according to superclass rules.
    //1. 调用父类的findCandidateAdvisors，查找所有实现Advisor接口的bean
    List<Advisor> advisors = super.findCandidateAdvisors();
    // Build Advisors for all AspectJ aspects in the bean factory.
    //2. 查找所有标注了@Aspect注解的bean，并生成Advisor对象
    advisors.addAll(this.aspectJAdvisorsBuilder.buildAspectJAdvisors());
    return advisors;
}
```
(1) 调用父类的findCandidateAdvisors，查找所有实现Advisor接口的bean
(2) 查找所有标注了@Aspect注解的bean，解析方法上的注解，根据注解生成对应的Advisor对象
## super.findCandidateAdvisors方法
super指的是AbstractAdvisorAutoProxyCreator类，AbstractAdvisorAutoProxyCreator中的实现如下:
```java
protected List<Advisor> findCandidateAdvisors() {
    return this.advisorRetrievalHelper.findAdvisorBeans();
}
```
advisorRetrievalHelper指的是BeanFactoryAdvisorRetrievalHelper，实现如下:
```java
public List<Advisor> findAdvisorBeans() {
    // Determine list of advisor bean names, if not cached already.
    String[] advisorNames = null;
    synchronized (this) {
        advisorNames = this.cachedAdvisorBeanNames;
        if (advisorNames == null) {
            // Do not initialize FactoryBeans here: We need to leave all regular beans
            // uninitialized to let the auto-proxy creator apply to them!
            //1. 查找所有类型是Advisor接口的bean
            advisorNames = BeanFactoryUtils.beanNamesForTypeIncludingAncestors(
                    this.beanFactory, Advisor.class, true, false);
            this.cachedAdvisorBeanNames = advisorNames;
        }
    }
    if (advisorNames.length == 0) {
        return new LinkedList<Advisor>();
    }

    List<Advisor> advisors = new LinkedList<Advisor>();
    for (String name : advisorNames) {
        //isEligibleBean(name)返回true,最终是调用AbstractAdvisorAutoProxyCreator#isEligibleAdvisorBean(name)方法 
        if (isEligibleBean(name)) {
            //如果当前增强器还没创建完，则跳过
            if (this.beanFactory.isCurrentlyInCreation(name)) {
                if (logger.isDebugEnabled()) {
                    logger.debug("Skipping currently created advisor '" + name + "'");
                }
            }
            else {
                try {
                    //添加Advisor
                    advisors.add(this.beanFactory.getBean(name, Advisor.class));
                }
                catch (BeanCreationException ex) {
                    Throwable rootCause = ex.getMostSpecificCause();
                    if (rootCause instanceof BeanCurrentlyInCreationException) {
                        BeanCreationException bce = (BeanCreationException) rootCause;
                        if (this.beanFactory.isCurrentlyInCreation(bce.getBeanName())) {
                            if (logger.isDebugEnabled()) {
                                logger.debug("Skipping advisor '" + name +
                                        "' with dependency on currently created bean: " + ex.getMessage());
                            }
                            // Ignore: indicates a reference back to the bean we're trying to advise.
                            // We want to find advisors other than the currently created bean itself.
                            continue;
                        }
                    }
                    throw ex;
                }
            }
        }
    }
    return advisors;
}
```
实现的逻辑：
1. 查找所有类型是Advisor接口的bean
2. 判断当前Advisor是否正在创建，如果是则跳过，否则添加Advisor。
## buildAspectJAdvisors方法
接下来分析this.advisorRetrievalHelper.findAdvisorBeans()，这个方法是spring aop能够简便使用的核心所在。当然逻辑也比较复杂,主要的调用时序图如下：
![findAdvisorBeans()](https://upload-images.jianshu.io/upload_images/10236819-a093ac82e88950dc.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
1. buildAspectJAdvisors()
BeanFactoryAspectJAdvisorsBuilder#buildAspectJAdvisors实现如下:
```java
public List<Advisor> buildAspectJAdvisors() {
    List<String> aspectNames = this.aspectBeanNames;

    if (aspectNames == null) {
        synchronized (this) {
            aspectNames = this.aspectBeanNames;
            if (aspectNames == null) {
                List<Advisor> advisors = new LinkedList<Advisor>();
                aspectNames = new LinkedList<String>();
                //获取所有的beanName
                String[] beanNames = BeanFactoryUtils.beanNamesForTypeIncludingAncestors(
                        this.beanFactory, Object.class, true, false);
                for (String beanName : beanNames) {
                    //默认返回true，调用的是AnnotationAwareAspectJAutoProxyCreator#isEligibleAspectBean
                    if (!isEligibleBean(beanName)) {
                        continue;
                    }
                    // We must be careful not to instantiate beans eagerly as in this case they
                    // would be cached by the Spring container but would not have been weaved.
                    //获取对应bean的类型
                    Class<?> beanType = this.beanFactory.getType(beanName);
                    if (beanType == null) {
                        continue;
                    }
                    //是否有@Aspect注解
                    if (this.advisorFactory.isAspect(beanType)) {
                        aspectNames.add(beanName);
                        AspectMetadata amd = new AspectMetadata(beanType, beanName);
                        //是不是单例，默认是
                        if (amd.getAjType().getPerClause().getKind() == PerClauseKind.SINGLETON) {
                            MetadataAwareAspectInstanceFactory factory =
                                    new BeanFactoryAspectInstanceFactory(this.beanFactory, beanName);
                            //解析标记AspectJ注解中的增强方法
                            List<Advisor> classAdvisors = this.advisorFactory.getAdvisors(factory);
                            if (this.beanFactory.isSingleton(beanName)) {
                                this.advisorsCache.put(beanName, classAdvisors);
                            }
                            else {
                                this.aspectFactoryCache.put(beanName, factory);
                            }
                            advisors.addAll(classAdvisors);
                        }
                        else {
                            // Per target or per this.
                            if (this.beanFactory.isSingleton(beanName)) {
                                throw new IllegalArgumentException("Bean with name '" + beanName +
                                        "' is a singleton, but aspect instantiation model is not singleton");
                            }
                            MetadataAwareAspectInstanceFactory factory =
                                    new PrototypeAspectInstanceFactory(this.beanFactory, beanName);
                            this.aspectFactoryCache.put(beanName, factory);
                            advisors.addAll(this.advisorFactory.getAdvisors(factory));
                        }
                    }
                }
                this.aspectBeanNames = aspectNames;
                return advisors;
            }
        }
    }

    if (aspectNames.isEmpty()) {
        return Collections.emptyList();
    }
    //记录在缓存中
    List<Advisor> advisors = new LinkedList<Advisor>();
    for (String aspectName : aspectNames) {
        List<Advisor> cachedAdvisors = this.advisorsCache.get(aspectName);
        if (cachedAdvisors != null) {
            advisors.addAll(cachedAdvisors);
        }
        else {
            MetadataAwareAspectInstanceFactory factory = this.aspectFactoryCache.get(aspectName);
            advisors.addAll(this.advisorFactory.getAdvisors(factory));
        }
    }
    return advisors;
}
```
方法的大致逻辑是:
(1) 获取所有beanName,这一步骤中所有在beanFactory中注册的Bean都会被提取出来。
(2) 遍历所有的beanName，并找出声明AspectJ注解的类，进行下一步处理。
(3) 对标记为AspectJ注解的类进行增强器的提取，提取逻辑交给了advisorFactory.getAdvisors(factory)。
(4) 将提取结果加入缓存。
2. getAdvisors()
ReflectiveAspectJAdvisorFactory#getAdvisors实现如下:
```java
@Override
public List<Advisor> getAdvisors(MetadataAwareAspectInstanceFactory aspectInstanceFactory) {
    Class<?> aspectClass = aspectInstanceFactory.getAspectMetadata().getAspectClass();
    String aspectName = aspectInstanceFactory.getAspectMetadata().getAspectName();
    validate(aspectClass);

    // We need to wrap the MetadataAwareAspectInstanceFactory with a decorator
    // so that it will only instantiate once.
    MetadataAwareAspectInstanceFactory lazySingletonAspectInstanceFactory =
            new LazySingletonAspectInstanceFactoryDecorator(aspectInstanceFactory);

    List<Advisor> advisors = new LinkedList<Advisor>();
    //遍历所有的非@Pointcut注解的方法
    for (Method method : getAdvisorMethods(aspectClass)) {
        //生成Advisor对象
        Advisor advisor = getAdvisor(method, lazySingletonAspectInstanceFactory, advisors.size(), aspectName);
        if (advisor != null) {
            advisors.add(advisor);
        }
    }

    // If it's a per target aspect, emit the dummy instantiating aspect.
    if (!advisors.isEmpty() && lazySingletonAspectInstanceFactory.getAspectMetadata().isLazilyInstantiated()) {
        Advisor instantiationAdvisor = new SyntheticInstantiationAdvisor(lazySingletonAspectInstanceFactory);
        advisors.add(0, instantiationAdvisor);
    }

    // Find introduction fields.
    for (Field field : aspectClass.getDeclaredFields()) {
        Advisor advisor = getDeclareParentsAdvisor(field);
        if (advisor != null) {
            advisors.add(advisor);
        }
    }

    return advisors;
}
```
(1) 遍历所有的非@Pointcut注解的方法
(2) 生成Advisor对象
3. getAdvisorMethods()
ReflectiveAspectJAdvisorFactory#getAdvisorMethods实现如下:
```java
private List<Method> getAdvisorMethods(Class<?> aspectClass) {
    final List<Method> methods = new LinkedList<Method>();
    ReflectionUtils.doWithMethods(aspectClass, new ReflectionUtils.MethodCallback() {
        @Override
        public void doWith(Method method) throws IllegalArgumentException {
            // Exclude pointcuts
            //去掉pointcuts标记的方法
            if (AnnotationUtils.getAnnotation(method, Pointcut.class) == null) {
                methods.add(method);
            }
        }
    });
    Collections.sort(methods, METHOD_COMPARATOR);
    return methods;
}
```
getAdvisorMethods()方法主要是找出被注解标记的方法，但是注解不是@Pointcut注解。
4. getAdvisor()
ReflectiveAspectJAdvisorFactory#getAdvisor()实现如下:
```java
@Override
public Advisor getAdvisor(Method candidateAdviceMethod, MetadataAwareAspectInstanceFactory aspectInstanceFactory,
        int declarationOrderInAspect, String aspectName) {

    validate(aspectInstanceFactory.getAspectMetadata().getAspectClass());
    //切点信息的获取
    AspectJExpressionPointcut expressionPointcut = getPointcut(
            candidateAdviceMethod, aspectInstanceFactory.getAspectMetadata().getAspectClass());
    if (expressionPointcut == null) {
        return null;
    }
    //根据切点信息生成增强器
    return new InstantiationModelAwarePointcutAdvisorImpl(expressionPointcut, candidateAdviceMethod,
            this, aspectInstanceFactory, declarationOrderInAspect, aspectName);
}
```
5.  getPointcut()获取切点信息。所谓获取切点信息就是指定注解的表达式信息的获取，如@Before("perform()")
```java
private AspectJExpressionPointcut getPointcut(Method candidateAdviceMethod, Class<?> candidateAspectClass) {
    //1. 找到方法上的注解，只能找到一个、找到就停止查找
    AspectJAnnotation<?> aspectJAnnotation =
            AbstractAspectJAdvisorFactory.findAspectJAnnotationOnMethod(candidateAdviceMethod);
    if (aspectJAnnotation == null) {
        return null;
    }

    //根据注解生成切点信息，解析例如@Before("perform()")注解
    AspectJExpressionPointcut ajexp =
            new AspectJExpressionPointcut(candidateAspectClass, new String[0], new Class<?>[0]);
    ajexp.setExpression(aspectJAnnotation.getPointcutExpression());
    ajexp.setBeanFactory(this.beanFactory);
    return ajexp;
}

//AbstractAspectJAdvisorFactory
protected static AspectJAnnotation<?> findAspectJAnnotationOnMethod(Method method) {
    Class<?>[] classesToLookFor = new Class<?>[] {
            Before.class, Around.class, After.class, AfterReturning.class, AfterThrowing.class, Pointcut.class};
    for (Class<?> c : classesToLookFor) {
        AspectJAnnotation<?> foundAnnotation = findAnnotation(method, (Class<Annotation>) c);
        if (foundAnnotation != null) {
            return foundAnnotation;
        }
    }
    return null;
}
```
6.  new InstantiationModelAwarePointcutAdvisorImpl，根据切点信息生成Advisor，所有的增强都由Advisor的实现类InstantiationModelAwarePointcutAdvisorImpl封装。
![InstantiationModelAwarePointcutAdvisorImpl](https://upload-images.jianshu.io/upload_images/10236819-7df06feeb956058d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
```java
public InstantiationModelAwarePointcutAdvisorImpl(AspectJExpressionPointcut declaredPointcut,
        Method aspectJAdviceMethod, AspectJAdvisorFactory aspectJAdvisorFactory,
        MetadataAwareAspectInstanceFactory aspectInstanceFactory, int declarationOrder, String aspectName) {

    this.declaredPointcut = declaredPointcut;
    this.declaringClass = aspectJAdviceMethod.getDeclaringClass();
    this.methodName = aspectJAdviceMethod.getName();
    this.parameterTypes = aspectJAdviceMethod.getParameterTypes();
    this.aspectJAdviceMethod = aspectJAdviceMethod;
    this.aspectJAdvisorFactory = aspectJAdvisorFactory;
    this.aspectInstanceFactory = aspectInstanceFactory;
    this.declarationOrder = declarationOrder;
    this.aspectName = aspectName;

    if (aspectInstanceFactory.getAspectMetadata().isLazilyInstantiated()) {
        // Static part of the pointcut is a lazy type.
        Pointcut preInstantiationPointcut = Pointcuts.union(
                aspectInstanceFactory.getAspectMetadata().getPerClausePointcut(), this.declaredPointcut);

        // Make it dynamic: must mutate from pre-instantiation to post-instantiation state.
        // If it's not a dynamic pointcut, it may be optimized out
        // by the Spring AOP infrastructure after the first evaluation.
        this.pointcut = new PerTargetInstantiationModelPointcut(
                this.declaredPointcut, preInstantiationPointcut, aspectInstanceFactory);
        this.lazy = true;
    }
    else {
        // A singleton aspect.
        this.pointcut = this.declaredPointcut;
        this.lazy = false;
        this.instantiatedAdvice = instantiateAdvice(this.declaredPointcut);
    }
```
7. instantiateAdvice实例化通知
```java
private Advice instantiateAdvice(AspectJExpressionPointcut pcut) {
    return this.aspectJAdvisorFactory.getAdvice(this.aspectJAdviceMethod, pcut,
            this.aspectInstanceFactory, this.declarationOrder, this.aspectName);
}
```
8. getAdvice()又调回ReflectiveAspectJAdvisorFactory，调用它的getAdvice方法，根据注解类型获取不同的通知。
```java
@Override
public Advice getAdvice(Method candidateAdviceMethod, AspectJExpressionPointcut expressionPointcut,
        MetadataAwareAspectInstanceFactory aspectInstanceFactory, int declarationOrder, String aspectName) {

    Class<?> candidateAspectClass = aspectInstanceFactory.getAspectMetadata().getAspectClass();
    validate(candidateAspectClass);

    AspectJAnnotation<?> aspectJAnnotation =
            AbstractAspectJAdvisorFactory.findAspectJAnnotationOnMethod(candidateAdviceMethod);
    if (aspectJAnnotation == null) {
        return null;
    }

    // If we get here, we know we have an AspectJ method.
    // Check that it's an AspectJ-annotated class
    if (!isAspect(candidateAspectClass)) {
        throw new AopConfigException("Advice must be declared inside an aspect type: " +
                "Offending method '" + candidateAdviceMethod + "' in class [" +
                candidateAspectClass.getName() + "]");
    }

    if (logger.isDebugEnabled()) {
        logger.debug("Found AspectJ method: " + candidateAdviceMethod);
    }

    AbstractAspectJAdvice springAdvice;
    //根据不同注解类型、封装不同的通知
    switch (aspectJAnnotation.getAnnotationType()) {
        case AtBefore:
            springAdvice = new AspectJMethodBeforeAdvice(
                    candidateAdviceMethod, expressionPointcut, aspectInstanceFactory);
            break;
        case AtAfter:
            springAdvice = new AspectJAfterAdvice(
                    candidateAdviceMethod, expressionPointcut, aspectInstanceFactory);
            break;
        case AtAfterReturning:
            springAdvice = new AspectJAfterReturningAdvice(
                    candidateAdviceMethod, expressionPointcut, aspectInstanceFactory);
            AfterReturning afterReturningAnnotation = (AfterReturning) aspectJAnnotation.getAnnotation();
            if (StringUtils.hasText(afterReturningAnnotation.returning())) {
                springAdvice.setReturningName(afterReturningAnnotation.returning());
            }
            break;
        case AtAfterThrowing:
            springAdvice = new AspectJAfterThrowingAdvice(
                    candidateAdviceMethod, expressionPointcut, aspectInstanceFactory);
            AfterThrowing afterThrowingAnnotation = (AfterThrowing) aspectJAnnotation.getAnnotation();
            if (StringUtils.hasText(afterThrowingAnnotation.throwing())) {
                springAdvice.setThrowingName(afterThrowingAnnotation.throwing());
            }
            break;
        case AtAround:
            springAdvice = new AspectJAroundAdvice(
                    candidateAdviceMethod, expressionPointcut, aspectInstanceFactory);
            break;
        case AtPointcut:
            if (logger.isDebugEnabled()) {
                logger.debug("Processing pointcut '" + candidateAdviceMethod.getName() + "'");
            }
            return null;
        default:
            throw new UnsupportedOperationException(
                    "Unsupported advice type on method: " + candidateAdviceMethod);
    }

    // Now to configure the advice...
    springAdvice.setAspectName(aspectName);
    springAdvice.setDeclarationOrder(declarationOrder);
    String[] argNames = this.parameterNameDiscoverer.getParameterNames(candidateAdviceMethod);
    if (argNames != null) {
        springAdvice.setArgumentNamesFromStringArray(argNames);
    }
    springAdvice.calculateArgumentBindings();
    return springAdvice;
}
```
从函数中可以看出，spring会根据不同的注解生成不同的通知，所有的通知的根接口都是Advice。
![Advice](https://upload-images.jianshu.io/upload_images/10236819-d6ac564a627dbe5a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
# 总结
spring获取Advisor(增强器)分为两步：
1. 获取所有类型是Advisor的bean。
2. 查找所有标注了@Aspect注解的bean，解析bean方法上的注解，根据注解生成对应的InstantiationModelAwarePointcutAdvisorImpl对象。
# 疑惑
我在分析源码的时候，在想如果一个方法上使用两个注解，会不会生成两个通知。如下面的代码
```java
  @Before("perform()")
  @After("perform()")
    public void takeSeats() {
        System.out.println("perform before take seats");
    }
```
答案是不会，只会生成一个通知，而且通知的顺序是Before, Around, After, AfterReturning, AfterThrowing。原因在getAdvice()方法中,在getAdvice()中有下面一段代码，是获取方法上的注解。
```java
AspectJAnnotation<?> aspectJAnnotation =	AbstractAspectJAdvisorFactory.findAspectJAnnotationOnMethod(candidateAdviceMethod);
```
```java
protected static AspectJAnnotation<?> findAspectJAnnotationOnMethod(Method method) {
    Class<?>[] classesToLookFor = new Class<?>[] {
            Before.class, Around.class, After.class, AfterReturning.class, AfterThrowing.class, Pointcut.class};
    for (Class<?> c : classesToLookFor) {
        AspectJAnnotation<?> foundAnnotation = findAnnotation(method, (Class<Annotation>) c);
        //短路操作，获取到注解就返回
        if (foundAnnotation != null) {
            return foundAnnotation;
        }
    }
    return null;
}
```
findAspectJAnnotationOnMethod方法是一个短路操作，它获取到注解就返回，所以最多只能返回一个，而遍历注解的顺序是Before, Around, After, AfterReturning, AfterThrowing。