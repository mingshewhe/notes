> 本文是直接摘抄《Spring源码深度解析》5.6节循环依赖，首先是加深自己的理解，其次是方便查阅。
# 什么是循环依赖
循环依赖就是循环引用，就是两个或多个bean相互之间持有对方，比如CircleA引用CircleB，circleB引用CircleC，CircleC引用CircleA，则它们最终反映为一个环。此处不是循环调用，循环调用是方法之间的调用，如下图所示。
![image.png](https://upload-images.jianshu.io/upload_images/10236819-900d0f82707e08c7.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
循环调用是无法解决的，除非有终结条件，否则就是死循环，最终导致内存溢出错误。
# Spring如何解决循环依赖
Spring容器循环依赖包括构造器循环依赖和setter循环依赖，那么Spring容器如何解决循环依赖呢？首先让我们来定义循环引用类：
```java
class TestA {
    private TestB testB;

    public void setTestB(TestB testB) {
        this.testB = testB;
    }
}

class TestB {
    private TestC testC;

    public void setTestC(TestC testC) {
        this.testC = testC;
    }
}

class TestC {
    private TestA testA;

    public void setTestA(TestA testA) {
        this.testA = testA;
    }
}
```
如何是我们自己硬编码，会怎么处理这种循环依赖，我们首先会把所有的对象都创建出来，然后再设置值。
```java
TestA testA = new TestA();
TestB testB = new TestB();
TestC testC = new TestC();
testA.setTestB(testB);
testB.setTestC(testC);
testC.setTestA(testA);
```
在Spring中将循环依赖的处理分为3中情况。
1. 构造器循环依赖
表示通过构造器注入构成的循环依赖，此依赖是无法解决的，只能抛出BeanCurrentlyInCreationException异常表示循环依赖。
如在创建TestA类时，构造器需要TestB类，那将去创建TestB，在创建TestB类时又发现需要TestC类，则又去创建TestC，最终在创建TestC时又需要TestA，从而形成一个环，没办法创建。
2. setter循环依赖
表示通过setter注入方式构成的循环依赖。对于setter注入造成的依赖是通过Spring容器提前暴露刚初始化完但未完成其他步骤(如setter注入)的bean来完成，而且只能解决单例作用域的bean循环依赖。从而使其他bean能引用到该bean，如下代码所示：(具体逻辑见第三节)
```java
addSingletonFactory(beanName, new ObjectFactory<Object>() {
    @Override
    public Object getObject() throws BeansException {
        return getEarlyBeanReference(beanName, mbd, bean);
    }
});
```
3. prototype范围的依赖处理
对于"prototype"作用域bean，Spring容器无法完成依赖注入，因为Spring容器不进行换成"prototype"作用域的bean，因为无法提前暴露一个创建中的bean。
# setter循环依赖实现原理
Spring容器对单例bean创建定义了几种不同的状态，并缓存中不同的Map中，Map的key值都是beanName。
```java
/** Cache of singleton objects: bean name --> bean instance */
//缓存已经创建好的bean
private final Map<String, Object> singletonObjects = new ConcurrentHashMap<String, Object>(256);

/** Cache of singleton factories: bean name --> ObjectFactory */
//有循环依赖时，在bean还没完全创建时，提前暴露ObjectFcatory，ObjectFactory能够获取bean。
private final Map<String, ObjectFactory<?>> singletonFactories = new HashMap<String, ObjectFactory<?>>(16);

/** Cache of early singleton objects: bean name --> bean instance */
//bean还没完全创建成功，缓存从ObjectFactory中获取的bean，earlySingletonObjects和singletonFactories只能其中一个缓存bean。
private final Map<String, Object> earlySingletonObjects = new HashMap<String, Object>(16);
```
或许现在还是没有明白singletonFactories和earlySingletonObjects是干嘛的，我们继续往下看，DefaultSingletonBeanRegistry#getSingleton(String beanName, boolean allowEarlyReference)方法，这方法会在创建bean的时候调用。
```java
protected Object getSingleton(String beanName, boolean allowEarlyReference) {
//bean实例是否已经创建完成
Object singletonObject = this.singletonObjects.get(beanName);
//若没有创建完成，是否正在创建
if (singletonObject == null && isSingletonCurrentlyInCreation(beanName)) {
    synchronized (this.singletonObjects) {
        //early singleton是否存在
        singletonObject = this.earlySingletonObjects.get(beanName);
        //若不存在，allowEarlyReference传进来true
        if (singletonObject == null && allowEarlyReference) {
            //从singleton factories获取ObjectFactory，并调用getObject()
            ObjectFactory<?> singletonFactory = this.singletonFactories.get(beanName);
            if (singletonFactory != null) {
                singletonObject = singletonFactory.getObject();
                //把bean添加到earlySingletonObjects中，并从singletonFactories移除
                this.earlySingletonObjects.put(beanName, singletonObject);
                this.singletonFactories.remove(beanName);
            }
        }
    }
}
return (singletonObject != NULL_OBJECT ? singletonObject : null);
}
```
isSingletonCurrentlyInCreation(beanName):这个方法表示该bean是否在创建中。在Spring中，会有个专门的属性默认为DefaultSingletonBeanRegistry的singletonsCurrentlyInCreation来记录bean的加载状态，在bean创建前会将beanName记录在属性中，在bean创建结束后会将beanName从属性中移除。在singleton下记录属性的函数是在DefaultSingletonBeanRegistry类的public Object getSingleton(String beanName, ObjectFactory<?> singletonFactory)函数的beforeSingletonCreation(beanName)和afterSingletonCreation(beanName)中。
最后我们再来关注singletonFactories是在什么时候添加进去的？
在AbstractAutowireCapableBeanFactory#doCreateBean(final String beanName, final RootBeanDefinition mbd, final Object[] args)有下面一段代码:
```java
// Eagerly cache singletons to be able to resolve circular references
// even when triggered by lifecycle interfaces like BeanFactoryAware.
//如果bean是单例的，并且允许循环依赖，并且正在创建，则调用addSingletonFactory
boolean earlySingletonExposure = (mbd.isSingleton() && this.allowCircularReferences &&
    isSingletonCurrentlyInCreation(beanName));
if (earlySingletonExposure) {
    if (logger.isDebugEnabled()) {
        logger.debug("Eagerly caching bean '" + beanName +
                "' to allow for resolving potential circular references");
    }
    //创建匿名ObjectFactory，并把ObjectFactory缓存到singletonFactories中
    addSingletonFactory(beanName, new ObjectFactory<Object>() {
        @Override
        public Object getObject() throws BeansException {
            //获取bean实例
            return getEarlyBeanReference(beanName, mbd, bean);
        }
    });
}
```
```java
protected void addSingletonFactory(String beanName, ObjectFactory<?> singletonFactory) {
    Assert.notNull(singletonFactory, "Singleton factory must not be null");
    synchronized (this.singletonObjects) {
        if (!this.singletonObjects.containsKey(beanName)) {
            this.singletonFactories.put(beanName, singletonFactory);
            this.earlySingletonObjects.remove(beanName);
            this.registeredSingletons.add(beanName);
        }
    }
}
```
我在看getEarlyBeanReference(beanName, mbd, bean)方法时，在想匿名ObjectFactory类是如何把beanName, mbd, bean这几个值给保存起来的，以前使用一般都会是在addSingletonFactory()方法中直接调用ObjectFactory的getObject方法，但是这里是缓存到Map中给以后调用，具体原理反编译之后就明白了，具体参考我另外一篇文章(明天补上).