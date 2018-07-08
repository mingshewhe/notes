@Import注解给Spring bean创建带来很大的灵活性，因其对配置的封装，极大简化了Spring的使用。Spring中的Enable*基本上都是通过Import注解来实现的。
# 定义
```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface Import {

	/**
	 * {@link Configuration}, {@link ImportSelector}, {@link ImportBeanDefinitionRegistrar}
	 * or regular component classes to import.
	 */
	Class<?>[] value();

}
```
从注释可以看出，Import支持导入Configuration、ImportSelector和ImportBeanDefinitionRegistrar,那么它们是怎么使用的呢？
# 使用
## 直接导入配置类(Configuration)
查看@EnableWebMvc源码，就是直接导入配置类，加载Web需要的各种bean。
```java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
@Documented
@Import(DelegatingWebMvcConfiguration.class)
public @interface EnableWebMvc {
}
```
DelegatingWebMvcConfiguration的父类中使用了@Bean声明bean。
## 依据条件选择配置类(ImportSelector)
查看@EnableAsync源码，就是导入ImportSelector接口。
```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Import(AsyncConfigurationSelector.class)
public @interface EnableAsync {
	Class<? extends Annotation> annotation() default Annotation.class;
	boolean proxyTargetClass() default false;
	AdviceMode mode() default AdviceMode.PROXY;
	int order() default Ordered.LOWEST_PRECEDENCE;
}
```
AsyncConfigurationSelector根接口为ImportSelector,这个接口需重写selectImports方法，在方法内进行事先条件判断。返回的对象中定义了bean。
```java
public class AsyncConfigurationSelector extends AdviceModeImportSelector<EnableAsync> {

	private static final String ASYNC_EXECUTION_ASPECT_CONFIGURATION_CLASS_NAME =
			"org.springframework.scheduling.aspectj.AspectJAsyncConfiguration";
	@Override
	public String[] selectImports(AdviceMode adviceMode) {
		switch (adviceMode) {
			case PROXY:
				return new String[] { ProxyAsyncConfiguration.class.getName() };
			case ASPECTJ:
				return new String[] { ASYNC_EXECUTION_ASPECT_CONFIGURATION_CLASS_NAME };
			default:
				return null;
		}
	}

}
```
## 动态注册Bean(ImportBeanDefinitionRegistrar)
查看@EnableAspectJAutoProxy
```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Import(AspectJAutoProxyRegistrar.class)
public @interface EnableAspectJAutoProxy {
	boolean proxyTargetClass() default false;
	boolean exposeProxy() default false;
}
```
AspectJAutoProxyRegistrar实现了ImportBeanDefinitionRegistrar接口，ImportBeanDefinitionRegistrar的作用是在运行时自动添加Bean到已有配置类，通过重写方法:
```java
public void registerBeanDefinitions(
			AnnotationMetadata importingClassMetadata, BeanDefinitionRegistry registry);

```
其中AnnotationMetadata参数用来获取当前配置类上的注解，BeanDefinitionRegistry参数用来注册Bean。
```java
class AspectJAutoProxyRegistrar implements ImportBeanDefinitionRegistrar {
	@Override
	public void registerBeanDefinitions(
			AnnotationMetadata importingClassMetadata, BeanDefinitionRegistry registry) {

		AopConfigUtils.registerAspectJAnnotationAutoProxyCreatorIfNecessary(registry);

		AnnotationAttributes enableAspectJAutoProxy =
				AnnotationConfigUtils.attributesFor(importingClassMetadata, EnableAspectJAutoProxy.class);
		if (enableAspectJAutoProxy.getBoolean("proxyTargetClass")) {
			AopConfigUtils.forceAutoProxyCreatorToUseClassProxying(registry);
		}
		if (enableAspectJAutoProxy.getBoolean("exposeProxy")) {
			AopConfigUtils.forceAutoProxyCreatorToExposeProxy(registry);
		}
	}

}
```
# 源码解析
在上节中[七、spring ioc之AnnotationConfigApplicationContext源码解析(二)](https://www.jianshu.com/p/1c7a46d4fcb9)讲了注解配置的bean如何加载，其中有解析@Import注解，我们看下它的源码，代码在ConfigurationClassParser#processImports.
```java
private void processImports(ConfigurationClass configClass, SourceClass currentSourceClass,
        Collection<SourceClass> importCandidates, boolean checkForCircularImports) {

    if (importCandidates.isEmpty()) {
        return;
    }

    if (checkForCircularImports && isChainedImportOnStack(configClass)) {
        this.problemReporter.error(new CircularImportProblem(configClass, this.importStack));
    }
    else {
        this.importStack.push(configClass);
        try {
            for (SourceClass candidate : importCandidates) {
                if (candidate.isAssignable(ImportSelector.class)) {
                    // Candidate class is an ImportSelector -> delegate to it to determine imports
                    Class<?> candidateClass = candidate.loadClass();
                    ImportSelector selector = BeanUtils.instantiateClass(candidateClass, ImportSelector.class);
                    ParserStrategyUtils.invokeAwareMethods(
                            selector, this.environment, this.resourceLoader, this.registry);
                    if (this.deferredImportSelectors != null && selector instanceof DeferredImportSelector) {
                        this.deferredImportSelectors.add(
                                new DeferredImportSelectorHolder(configClass, (DeferredImportSelector) selector));
                    }
                    else {
                        //调用selectImports方法，解析返回对象中的bean
                        String[] importClassNames = selector.selectImports(currentSourceClass.getMetadata());
                        Collection<SourceClass> importSourceClasses = asSourceClasses(importClassNames);
                        processImports(configClass, currentSourceClass, importSourceClasses, false);
                    }
                }
                else if (candidate.isAssignable(ImportBeanDefinitionRegistrar.class)) {
                    // Candidate class is an ImportBeanDefinitionRegistrar ->
                    // delegate to it to register additional bean definitions
                    Class<?> candidateClass = candidate.loadClass();
                    ImportBeanDefinitionRegistrar registrar =
                            BeanUtils.instantiateClass(candidateClass, ImportBeanDefinitionRegistrar.class);
                    ParserStrategyUtils.invokeAwareMethods(
                            registrar, this.environment, this.resourceLoader, this.registry);
                    //把ImportBeanDefinitionRegistrar添加到ConfigurationClass中的importBeanDefinitionRegistrars属性中，importBeanDefinitionRegistrars是一个Map对象
                    configClass.addImportBeanDefinitionRegistrar(registrar, currentSourceClass.getMetadata());
                }
                else {
                    // Candidate class not an ImportSelector or ImportBeanDefinitionRegistrar ->
                    // process it as an @Configuration class
                    this.importStack.registerImport(
                            currentSourceClass.getMetadata(), candidate.getMetadata().getClassName());
                    processConfigurationClass(candidate.asConfigClass(configClass));
                }
            }
        }
        catch (BeanDefinitionStoreException ex) {
            throw ex;
        }
        catch (Throwable ex) {
            throw new BeanDefinitionStoreException(
                    "Failed to process import candidates for configuration class [" +
                    configClass.getMetadata().getClassName() + "]", ex);
        }
        finally {
            this.importStack.pop();
        }
    }
}
```
执行逻辑如下:
1. @Import配置的类是否是ImportSelector的子类，如果是，调用selectImports()方法，并再次解析selectImports()方法的返回值。
2. @Import配置的类是否是ImportBeanDefinitionRegistrar的子类，如果是，把对象添加到ConfigurationClass中的importBeanDefinitionRegistrars属性中
3. 如果是其他的类(配置了@Configuration)，调用processConfigurationClass方法解析
## ImportBeanDefinitionRegistrar中的bean注册
上一步中的ImportBeanDefinitionRegistrar只是添加到ConfigurationClass，并没有解析其中的bean。真正的解析是在ConfigurationClassBeanDefinitionReader#loadBeanDefinitionsFromRegistrars。
```java
private void loadBeanDefinitionsFromRegistrars(Map<ImportBeanDefinitionRegistrar, AnnotationMetadata> registrars) {
    for (Map.Entry<ImportBeanDefinitionRegistrar, AnnotationMetadata> entry : registrars.entrySet()) {
        entry.getKey().registerBeanDefinitions(entry.getValue(), this.registry);
    }
}
```
遍历ConfigurationClass中的importBeanDefinitionRegistrars，并且调用ImportBeanDefinitionRegistrar的registerBeanDefinitions方法加载bean。
