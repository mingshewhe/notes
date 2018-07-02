# 代码
```java
ClassPathXmlApplicationContext ac = new ClassPathXmlApplicationContext("spring-context.xml");
```
# 类图
继承的类图，忽略实现的接口
![image.png](https://upload-images.jianshu.io/upload_images/10236819-105d4250cd386257.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

# 时序图
![image.png](https://upload-images.jianshu.io/upload_images/10236819-6981e6e5078ff647.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
# 详细步骤
1. new ClassPathXmlApplicationContext(),初始化ClassPathXmlApplicationContext
    ```java
    public ClassPathXmlApplicationContext(String[] configLocations, boolean refresh, ApplicationContext parent)
            throws BeansException {
        super(parent);
        setConfigLocations(configLocations);
        if (refresh) {
            refresh();
        }
    }
    ```
2. 调用AbstractRefreshableConfigApplicationContext的setConfigLocations(configLocations),设置xml文件路径
    ```java
    public void setConfigLocations(String... locations) {
        if (locations != null) {
            Assert.noNullElements(locations, "Config locations must not be null");
            this.configLocations = new String[locations.length];
            for (int i = 0; i < locations.length; i++) {
                this.configLocations[i] = resolvePath(locations[i]).trim();
            }
        }
        else {
            this.configLocations = null;
        }
    }
    ```
3. 调用AbstractApplicationContext的refresh(),这个方法内容太多，对于ClassPathXmlApplicationContext加载bean，只需了解它的obtainFreshBeanFactory方法。
4. obtainFreshBeanFactory方法,获取BeanFactory。
5. 调用AbstractRefreshableApplicationContext的refreshBeanFactory方法
    ```java
    protected final void refreshBeanFactory() throws BeansException {
        if (hasBeanFactory()) {
            destroyBeans();
            closeBeanFactory();
        }
        try {
            //创建BeanFactory
            DefaultListableBeanFactory beanFactory = createBeanFactory();
            beanFactory.setSerializationId(getId());
            customizeBeanFactory(beanFactory);
            loadBeanDefinitions(beanFactory);
            synchronized (this.beanFactoryMonitor) {
                this.beanFactory = beanFactory;
            }
        }
        catch (IOException ex) {
            throw new ApplicationContextException("I/O error parsing bean definition source for " + getDisplayName(), ex);
        }
    }
    ```
    ```java
    protected DefaultListableBeanFactory createBeanFactory() {
        return new DefaultListableBeanFactory(getInternalParentBeanFactory());
    }
    ```
6. 调用AbstractXmlApplicationContext的loadBeanDefinitions方法，加载bean
    ```java
    protected void loadBeanDefinitions(DefaultListableBeanFactory beanFactory) throws BeansException, IOException {
        // Create a new XmlBeanDefinitionReader for the given BeanFactory.
        XmlBeanDefinitionReader beanDefinitionReader = new XmlBeanDefinitionReader(beanFactory);

        // Configure the bean definition reader with this context's
        // resource loading environment.
        beanDefinitionReader.setEnvironment(this.getEnvironment());
        beanDefinitionReader.setResourceLoader(this);
        beanDefinitionReader.setEntityResolver(new ResourceEntityResolver(this));

        // Allow a subclass to provide custom initialization of the reader,
        // then proceed with actually loading the bean definitions.
        initBeanDefinitionReader(beanDefinitionReader);
        loadBeanDefinitions(beanDefinitionReader);
    }
    ```
7. 加载bean主要是对配置文件的读取，并将配置中的bean读取到DefaultListableBeanFactory中