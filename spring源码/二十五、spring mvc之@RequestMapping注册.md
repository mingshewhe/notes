不知道大家和我有没有相同的疑惑，就是Spring什么时候把Controller加载的，在类上声明@Controller或@RestController注解，只是声明注册一个bean，那么@RequestMapping又是在什么时候加载的呢？
@RequestMapping加载依赖的是RequestMappingHandlerMapping，所以这节主要分析RequestMappingHandlerMapping。
# RequestMappingHandlerMapping创建
不知道大家是否还有印象，我们在[二十三、spring mvc之简单使用](https://www.jianshu.com/p/970a457e7633)中的WebConfig上使用了@EnableWebMvc注解，这个注解导入了DelegatingWebMvcConfiguration配置，在DelegatingWebMvcConfiguration的父类WebMvcConfigurationSupport创建了RequestMappingHandlerMapping。
```java
@Bean
public RequestMappingHandlerMapping requestMappingHandlerMapping() {
    RequestMappingHandlerMapping mapping = createRequestMappingHandlerMapping();
    //...省略属性设置
    return mapping;
}
```
# RequestMappingHandlerMapping加载@RequestMapping
我一开始也不知道创建RequestMappingHandlerMapping作用，后来看到它的类图就明白加载过程。
![RequestMappingHandlerMapping](https://upload-images.jianshu.io/upload_images/10236819-cb2a5a486df9a01c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
RequestMappingHandlerMapping的父类实现了InitializingBean，那么在初始化的时候就会调用afterPropertiesSet方法。看下父类afterPropertiesSet的实现:
```java
@Override
public void afterPropertiesSet() {
    initHandlerMethods();
}

protected void initHandlerMethods() {
    if (logger.isDebugEnabled()) {
        logger.debug("Looking for request mappings in application context: " + getApplicationContext());
    }
    //detectHandlerMethodsInAncestorContexts默认false，所以是得到所有的beanNames
    String[] beanNames = (this.detectHandlerMethodsInAncestorContexts ?
            BeanFactoryUtils.beanNamesForTypeIncludingAncestors(getApplicationContext(), Object.class) :
            getApplicationContext().getBeanNamesForType(Object.class));

    for (String beanName : beanNames) {
        if (!beanName.startsWith(SCOPED_TARGET_NAME_PREFIX)) {
            Class<?> beanType = null;
            try {
                beanType = getApplicationContext().getType(beanName);
            }
            catch (Throwable ex) {
                // An unresolvable bean type, probably from a lazy bean - let's ignore it.
                if (logger.isDebugEnabled()) {
                    logger.debug("Could not resolve target class for bean with name '" + beanName + "'", ex);
                }
            }
            //2. isHandler由子类实现
            if (beanType != null && isHandler(beanType)) {
                //3. 查找controller中有@RequestMapping注解的方法,并注册到请求
                detectHandlerMethods(beanName);
            }
        }
    }
    //空方法
    handlerMethodsInitialized(getHandlerMethods());
}
```
afterPropertiesSet实现交给了initHandlerMethods，initHandlerMethods的执行流程如下:
1. 得到所有的beanName
2. 遍历beanNames,调用isHandler判断bean是不是一个controller，isHandler由子类实现，RequestMappingHandlerMapping的实现如下，判断bean中有没有Controller或RequestMapping
```java
@Override
protected boolean isHandler(Class<?> beanType) {
    return (AnnotatedElementUtils.hasAnnotation(beanType, Controller.class) ||
            AnnotatedElementUtils.hasAnnotation(beanType, RequestMapping.class));
}
```
3. 查找controller中有@RequestMapping注解的方法,并注册到请求容器中
```java
protected void detectHandlerMethods(final Object handler) {
    //1. 得到controller真实类型
    Class<?> handlerType = (handler instanceof String ?
            getApplicationContext().getType((String) handler) : handler.getClass());
    final Class<?> userType = ClassUtils.getUserClass(handlerType);

    //3. 封装所有的Method和RequestMappingInfo到Map中
    Map<Method, T> methods = MethodIntrospector.selectMethods(userType,
            new MethodIntrospector.MetadataLookup<T>() {
                @Override
                public T inspect(Method method) {
                    try {
                        //2. 根据方法上的@RequestMapping信息构建RequestMappingInfo
                        return getMappingForMethod(method, userType);
                    }
                    catch (Throwable ex) {
                        throw new IllegalStateException("Invalid mapping on handler class [" +
                                userType.getName() + "]: " + method, ex);
                    }
                }
            });

    if (logger.isDebugEnabled()) {
        logger.debug(methods.size() + " request handler methods found on " + userType + ": " + methods);
    }
    for (Map.Entry<Method, T> entry : methods.entrySet()) {
        Method invocableMethod = AopUtils.selectInvocableMethod(entry.getKey(), userType);
        T mapping = entry.getValue();
        //4. 将RequestMappingInfo注册到请求容器中
        registerHandlerMethod(handler, invocableMethod, mapping);
    }
}
```
detectHandlerMethods是加载请求核心方法，执行流程如下:
(1) 得到controller真实类型,controller可能被代理
(2) 根据方法上的@RequestMapping信息构建RequestMappingInfo,由RequestMappingHandlerMapping实现。从代码上可以看出，必须方法上声明@RequestMapping，类上的@RequestMapping才会生效。
```java
@Override
protected RequestMappingInfo getMappingForMethod(Method method, Class<?> handlerType) {
    //1. 根据方法上的@RequestMapping创建RequestMappingInfo
    RequestMappingInfo info = createRequestMappingInfo(method);
    if (info != null) {
        //2. 查找类上是否有@RequestMapping，如果有则和方法上的组合
        RequestMappingInfo typeInfo = createRequestMappingInfo(handlerType);
        if (typeInfo != null) {
            info = typeInfo.combine(info);
        }
    }
    return info;
}
```
3. 封装所有的Method和RequestMappingInfo到Map中
4. 将RequestMappingInfo注册到请求容器中 
```java
protected void registerHandlerMethod(Object handler, Method method, T mapping) {
    this.mappingRegistry.register(mapping, handler, method);
}
```
这里就不再分析register方法的实现过程，主要是根据handler，method封装成HandlerMethod，再请求的时候会得到HandlerMethod，然后反射调用。