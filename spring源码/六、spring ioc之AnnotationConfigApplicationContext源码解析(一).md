AnnotationConfigApplicationContext是Spring用来加载注解配置的ApplicationContext，它是如何加载所有的bean，与ClassPathXmlApplicationContext有什么区别，让我们接下来揭开它的神秘面纱。
# 类图
![AnnotationConfigApplicationContext](https://upload-images.jianshu.io/upload_images/10236819-8498f9561958f7e1.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
这里我们拿AnnotationConfigApplicationContext与ClassPathXmlApplicationContext对比，由[一、spring ioc之ClassPathXmlApplicationContext源码解析](https://www.jianshu.com/p/d707057497af)我们知道，ClassPathXmlApplicationContext加载bean的逻辑是在AbstractRefreshableApplicationContext的refreshBeanFactory()方法中，但是我们从类图中可以看出，AnnotationConfigApplicationContext并没有实现这个类，那么它是如何加载bean的呢？
# 初始化
```java
public AnnotationConfigApplicationContext(Class<?>... annotatedClasses) {
    this();
    register(annotatedClasses);
    refresh();
}
```
1. this()
```java
private final AnnotatedBeanDefinitionReader reader;
private final ClassPathBeanDefinitionScanner scanner;
public AnnotationConfigApplicationContext() {
    this.reader = new AnnotatedBeanDefinitionReader(this);
    this.scanner = new ClassPathBeanDefinitionScanner(this);
}
```
无参构造函数中主要是初始化AnnotatedBeanDefinitionReader和ClassPathBeanDefinitionScanner两个类。这里主要关注AnnotatedBeanDefinitionReader，跟踪这个类的初始化发现它会注册一堆BeanFactoryPostProcessor处理器，我们只需关注ConfigurationClassPostProcessor。
```java
public static Set<BeanDefinitionHolder> registerAnnotationConfigProcessors(
			BeanDefinitionRegistry registry, Object source) {
   ...
    if (!registry.containsBeanDefinition(CONFIGURATION_ANNOTATION_PROCESSOR_BEAN_NAME)) {
            RootBeanDefinition def = new RootBeanDefinition(ConfigurationClassPostProcessor.class);
            def.setSource(source);
            beanDefs.add(registerPostProcessor(registry, def, CONFIGURATION_ANNOTATION_PROCESSOR_BEAN_NAME));
    }
    ...
}
```
2. register(annotatedClasses)
这个方法主要是把所有的配置类注册成bean。
3. refresh()
一个很熟悉的方法，因为ClassPathXmlApplicationContext加载也调用了它，但是ClassPathXmlApplicationContext在调用obtainFreshBeanFactory()的时候就把所有的bean加载完成，但是AnnotationConfigApplicationContext并没有继承自AbstractRefreshableApplicationContext，所以在obtainFreshBeanFactory()这步还是没有加载bean。真正加载bean的操作是在invokeBeanFactoryPostProcessors(beanFactory),这个方法调用所有实现BeanFactoryPostProcessor接口的bean。那么BeanFactoryPostProcessor又是干嘛的呢？
# BeanFactoryPostProcessor处理器
和BeanPostProcessor原理一致，Spring提供了对BeanFactory进行操作的处理器BeanFactoryProcessor，简单来说就是获取容器BeanFactory，这样就可以在真正初始化bean之前对bean做一些处理操作。BeanFactoryProcessor定义如下:
```java
public interface BeanFactoryPostProcessor {
	void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) throws BeansException;
}
```

## BeanDefinitionRegistryPostProcessor
BeanDefinitionRegistryPostProcessor是BeanFactoryPostProcessor的子类，也只要一个方法
```java
public interface BeanDefinitionRegistryPostProcessor extends BeanFactoryPostProcessor {
	void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry registry) throws BeansException;
}
```
# 调用逻辑
AbstractApplicationContext#refresh()中调用了invokeBeanFactoryPostProcessors(beanFactory);这个方法逻辑如下：
1. 遍历所有实现了BeanDefinitionRegistryPostProcessor接口的bean
2. 调用实现了PriorityOrdered接口的BeanDefinitionRegistryPostProcessors的postProcessBeanDefinitionRegistry
3. 调用实现了Ordered接口的BeanDefinitionRegistryPostProcessors的postProcessBeanDefinitionRegistry
4. 调用普通的BeanDefinitionRegistryPostProcessors(没有实现Ordered接口和PriorityOrdered接口)的postProcessBeanDefinitionRegistry
5. 遍历所有实现BeanFactoryPostProcessor接口的bean，剩下的操作和BeanDefinitionRegistryPostProcessors的处理逻辑是一样的。
