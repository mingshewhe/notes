前文[BeanPostProcessor解析](https://www.jianshu.com/p/efa98910fb97)、[FactoryBean的使用](https://www.jianshu.com/p/df8120c5c924)和[bean循环依赖](https://www.jianshu.com/p/16287a26be25)是对这节内容的细化，这节相当于对bean做个总结。
![image.png](https://upload-images.jianshu.io/upload_images/10236819-facc729071867346.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
# InstantiationAwareBeanPostProcessor前置处理
在目标对象实例化之前调用，该方法的返回值类型是Object，我们可以返回任何类型的值。由于这个时候目标对象还未实例化，所以这个返回值可以用来代替原本该生成的目标对象的实例(比如代理对象)。如果该方法的返回值代替原本该生成的目标对象，后续只有postProcessAfterInitialization方法会调用，其它方法不再调用；否则按照正常的流程走
```java
protected Object resolveBeforeInstantiation(String beanName, RootBeanDefinition mbd) {
    Object bean = null;
    if (!Boolean.FALSE.equals(mbd.beforeInstantiationResolved)) {
        // Make sure bean class is actually resolved at this point.
        if (!mbd.isSynthetic() && hasInstantiationAwareBeanPostProcessors()) {
            Class<?> targetType = determineTargetType(beanName, mbd);
            if (targetType != null) {
                bean = applyBeanPostProcessorsBeforeInstantiation(targetType, beanName);
                if (bean != null) {
                    bean = applyBeanPostProcessorsAfterInitialization(bean, beanName);
                }
            }
        }
        mbd.beforeInstantiationResolved = (bean != null);
    }
    return bean;
}
```
# 实例化bean
实例化bean有三种方式：
1. 无参构造，这种是最简单的,获得beanClass，然后反射创建。
2. 有参构造，比较麻烦，需要匹配参数，调用合适的构造方法。
3. 工厂方法调用，直接调用方法创建bean，比如@bean声明的方法。

实例化对象被包装在BeanWrapper对象中，BeanWrapper提供了设置对象属性的接口，从而避免了使用反射机制设置属性。
具体代码实现在AbstractAutowireCapableBeanFactory#createBeanInstance
# 属性填充
具体实现代码在AbstractAutowireCapableBeanFactory#populateBean中。
populateBean函数处理的流程如下:
1. InstantiationAwareBeanPostProcessor处理器的postProcessAfterInstantiation函数的应用，此函数可以控制程序是否继续进行属性填充。
2. 根据注入类型(byName/byType)，提取依赖的bean，并统一存入PropertyValues。
3. 应用InstantiationAwareBeanPostProcessor处理器的postProcessPropertyValues，对属性获取完毕填充前对属性的再次处理。
4. 将所有的PropertyValues中的属性填充至BeanWrapper中。
# 注入Aware接口
Spring会检测该对象是否实现了xxxAware接口，并将相关的xxxAware实例注入给bean。
```java
private void invokeAwareMethods(final String beanName, final Object bean) {
    if (bean instanceof Aware) {
        if (bean instanceof BeanNameAware) {
            ((BeanNameAware) bean).setBeanName(beanName);
        }
        if (bean instanceof BeanClassLoaderAware) {
            ((BeanClassLoaderAware) bean).setBeanClassLoader(getBeanClassLoader());
        }
        if (bean instanceof BeanFactoryAware) {
            ((BeanFactoryAware) bean).setBeanFactory(AbstractAutowireCapableBeanFactory.this);
        }
    }
    }
```
# BeanPostProcessorc处理
这步可以对bean进行处理改造，详情可以参考[BeanPostProcessor解析](https://www.jianshu.com/p/efa98910fb97)
# InitializingBean与init-method
检查bean是否实现InitializingBean，如果有就调用afterPropertiesSet()方法，若自定义了init-method方法，也是在这步调用的。
```java
protected void invokeInitMethods(String beanName, final Object bean, RootBeanDefinition mbd)
			throws Throwable {

    boolean isInitializingBean = (bean instanceof InitializingBean);
    if (isInitializingBean && (mbd == null || !mbd.isExternallyManagedInitMethod("afterPropertiesSet"))) {
        if (logger.isDebugEnabled()) {
            logger.debug("Invoking afterPropertiesSet() on bean with name '" + beanName + "'");
        }
        if (System.getSecurityManager() != null) {
            try {
                AccessController.doPrivileged(new PrivilegedExceptionAction<Object>() {
                    @Override
                    public Object run() throws Exception {
                        ((InitializingBean) bean).afterPropertiesSet();
                        return null;
                    }
                }, getAccessControlContext());
            }
            catch (PrivilegedActionException pae) {
                throw pae.getException();
            }
        }
        else {
            ((InitializingBean) bean).afterPropertiesSet();
        }
    }

    if (mbd != null) {
        String initMethodName = mbd.getInitMethodName();
        if (initMethodName != null && !(isInitializingBean && "afterPropertiesSet".equals(initMethodName)) &&
                !mbd.isExternallyManagedInitMethod(initMethodName)) {
            invokeCustomInitMethod(beanName, bean, mbd);
        }
    }
}
```
# 注册DisposableBean
Spring中不但提供了对于初始化方法的扩展入口，同样也提供了销毁方法的扩展入口，对于销毁方法的扩展，除了我们熟知的配置属性destory-method方法外，用户还可以注册后处理器DestructionAwareBeanPostProcessor来统一处理bean的销毁方法。
# 检查FactoryBean接口
查看[FactoryBean的使](https://www.jianshu.com/p/efa98910fb97)用
# 参考文章
1. [https://www.zhihu.com/question/38597960](https://www.zhihu.com/question/38597960)
2. 《Spring源码深度解析》