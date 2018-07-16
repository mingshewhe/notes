**注：要了解spring事务最好先去了解spring aop，可以参考[十一、spring aop之简单使用](https://www.jianshu.com/p/18194f639625)**
# 简单使用
```java
@Configuration
//1. 开启事务
@EnableTransactionManagement
public class RootConfig {

    //2. 定义数据源
    @Bean
    public DataSource dataSource() {
        DruidDataSource dataSource = new DruidDataSource();
        return dataSource;
    }

    //3. 定义事务管理器
    @Bean
    public PlatformTransactionManager txManager() {
        return new DataSourceTransactionManager(dataSource());
    }
}

@Service
public class UserService {

    @Autowired
    private UserMapper userMapper;

    //4.声明事务
    @Transactional
    public void addUser(User user) {
        userMapper.addUser(user);
    }
}
```
要使用Spring事务，通常会有以下步骤：
1. 在配置文件上声明@EnableTransactionManagement注解开启事务
2. 定义数据源
3. 定义事务管理器
4. @Transactional声明事务
# 原理解析
@EnableTransactionManagement是开启spring事务，@EnableTransactionManagement源码如下：
```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Import(TransactionManagementConfigurationSelector.class)
public @interface EnableTransactionManagement {
	boolean proxyTargetClass() default false;
	AdviceMode mode() default AdviceMode.PROXY;
	int order() default Ordered.LOWEST_PRECEDENCE;
}
```
又看到熟悉的Import注解，这次使用的是ImportSelector方法，会调用ImportSelector接口的selectImports方法。
```java
@Override
public final String[] selectImports(AnnotationMetadata importingClassMetadata) {
    Class<?> annoType = GenericTypeResolver.resolveTypeArgument(getClass(), AdviceModeImportSelector.class);
    AnnotationAttributes attributes = AnnotationConfigUtils.attributesFor(importingClassMetadata, annoType);
    if (attributes == null) {
        throw new IllegalArgumentException(String.format(
            "@%s is not present on importing class '%s' as expected",
            annoType.getSimpleName(), importingClassMetadata.getClassName()));
    }

    AdviceMode adviceMode = attributes.getEnum(this.getAdviceModeAttributeName());
    //根据AdviceMode返回不同的类型，默认是AdviceMode.PROXY。
    String[] imports = selectImports(adviceMode);
    if (imports == null) {
        throw new IllegalArgumentException(String.format("Unknown AdviceMode: '%s'", adviceMode));
    }
    return imports;
}

@Override
protected String[] selectImports(AdviceMode adviceMode) {
    switch (adviceMode) {
        case PROXY:
            return new String[] {AutoProxyRegistrar.class.getName(), ProxyTransactionManagementConfiguration.class.getName()};
        case ASPECTJ:
            return new String[] {TransactionManagementConfigUtils.TRANSACTION_ASPECT_CONFIGURATION_CLASS_NAME};
        default:
            return null;
    }
}
```
根据AdviceMode选择创建不同的bean，AdviceMode默认是PROXY，所以返回的是AutoProxyRegistrar和ProxyTransactionManagementConfiguration。我们先来分析AutoProxyRegistrar。
## AutoProxyRegistrar
```java
public class AutoProxyRegistrar implements ImportBeanDefinitionRegistrar
```
由类的声明，AutoProxyRegistrar实现了ImportBeanDefinitionRegistrar，那在创建bean的时候会调用registerBeanDefinitions方法。registerBeanDefinitions方法的实现：
```java
@Override
public void registerBeanDefinitions(AnnotationMetadata importingClassMetadata, BeanDefinitionRegistry registry) {
    boolean candidateFound = false;
    Set<String> annoTypes = importingClassMetadata.getAnnotationTypes();
    for (String annoType : annoTypes) {
        AnnotationAttributes candidate = AnnotationConfigUtils.attributesFor(importingClassMetadata, annoType);
        if (candidate == null) {
            continue;
        }
        Object mode = candidate.get("mode");
        Object proxyTargetClass = candidate.get("proxyTargetClass");
        if (mode != null && proxyTargetClass != null && AdviceMode.class == mode.getClass() &&
                Boolean.class == proxyTargetClass.getClass()) {
            candidateFound = true;
            //只有@EnableTransactionManagement注解才会走到这里
            if (mode == AdviceMode.PROXY) {
                AopConfigUtils.registerAutoProxyCreatorIfNecessary(registry);
                if ((Boolean) proxyTargetClass) {
                    AopConfigUtils.forceAutoProxyCreatorToUseClassProxying(registry);
                    return;
                }
            }
        }
    }
    //...
}

public static BeanDefinition registerAutoProxyCreatorIfNecessary(BeanDefinitionRegistry registry, Object source) {
    return registerOrEscalateApcAsRequired(InfrastructureAdvisorAutoProxyCreator.class, registry, source);
}
```
AutoProxyRegistrar这个类主要是注册InfrastructureAdvisorAutoProxyCreator，注册这个bean有什么用呢?先看它的类图
![InfrastructureAdvisorAutoProxyCreator](https://upload-images.jianshu.io/upload_images/10236819-082b70ffcfb62ef3.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
由类图可以看出InfrastructureAdvisorAutoProxyCreator和AnnotationAwareAspectJAutoProxyCreator使用的是相同的父类AbstractAdvisorAutoProxyCreator，那么就会执行和spring aop相同的逻辑。所以AutoProxyRegistrar目的已经很明显了，开启spring aop。接下来再看ProxyTransactionManagementConfiguration的目的
## ProxyTransactionManagementConfiguration
```java
@Configuration
public class ProxyTransactionManagementConfiguration extends AbstractTransactionManagementConfiguration {

	@Bean(name = TransactionManagementConfigUtils.TRANSACTION_ADVISOR_BEAN_NAME)
	@Role(BeanDefinition.ROLE_INFRASTRUCTURE)
	public BeanFactoryTransactionAttributeSourceAdvisor transactionAdvisor() {
		BeanFactoryTransactionAttributeSourceAdvisor advisor = new BeanFactoryTransactionAttributeSourceAdvisor();
		advisor.setTransactionAttributeSource(transactionAttributeSource());
		advisor.setAdvice(transactionInterceptor());
		advisor.setOrder(this.enableTx.<Integer>getNumber("order"));
		return advisor;
	}

	@Bean
	@Role(BeanDefinition.ROLE_INFRASTRUCTURE)
	public TransactionAttributeSource transactionAttributeSource() {
		return new AnnotationTransactionAttributeSource();
	}

	@Bean
	@Role(BeanDefinition.ROLE_INFRASTRUCTURE)
	public TransactionInterceptor transactionInterceptor() {
		TransactionInterceptor interceptor = new TransactionInterceptor();
		interceptor.setTransactionAttributeSource(transactionAttributeSource());
		if (this.txManager != null) {
			interceptor.setTransactionManager(this.txManager);
		}
		return interceptor;
	}

}
```
ProxyTransactionManagementConfiguration是一个配置文件，注册了三个bean，BeanFactoryTransactionAttributeSourceAdvisor、AnnotationTransactionAttributeSource、TransactionInterceptor，而且这三个bean还存在依赖。他们之间的类图如下：
![image.png](https://upload-images.jianshu.io/upload_images/10236819-7757321f98538fad.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
在类图中我们看到几个熟悉的类，Advisor、Advice和Pointcut。ProxyTransactionManagementConfiguration主要的逻辑就是创建一个Advisor对象(BeanFactoryTransactionAttributeSourceAdvisor)，并且注入通知(AnnotationTransactionAttributeSource)和切点(TransactionInterceptor)。