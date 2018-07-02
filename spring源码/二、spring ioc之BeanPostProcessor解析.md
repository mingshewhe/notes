Spring的bean能够高度扩展，BeanPostProcessor功不可没，这个接口可以对bean实例做一些自定义修改，比如spring aop就是利用这个接口实现对bean的动态代理。

# 如何用
使用方法很简单，实现BeanPostProcessor，然后将自定义的BeanPostProcessor声明为一个bean就可以了。spring会自动扫描所有实现BeanPostProcessor的bean。
```java
@Component
public class MyBeanPostProcessor implements BeanPostProcessor {
    @Override
    public Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {
        System.out.println(beanName + " MyBeanPostProcessor#MyBeanPostProcessor");
        return bean;
    }

    @Override
    public Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {
        System.out.println(beanName + " MyBeanPostProcessor#postProcessAfterInitialization");
        return bean;
    }
}
```

# BeanPostProcessor子接口
> 子接口内容摘抄自 https://fangjian0423.github.io/2017/06/20/spring-bean-post-processor/

BeanPostProcessor还有一些子接口的定义，分别实现扩展不同的功能，类图如下所示:
![image.png](https://upload-images.jianshu.io/upload_images/10236819-603f43a506131c11.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


## InstantiationAwareBeanPostProcessor
InstantiationAwareBeanPostProcessor接口继承自BeanPostProcessor接口。多出了3个方法：
```java
//实例化之前调用
Object postProcessBeforeInstantiation(Class<?> beanClass, String beanName) throws BeansException;

//实例化之后调用
boolean postProcessAfterInstantiation(Object bean, String beanName) throws BeansException;

//在这还可以改变属性
PropertyValues postProcessPropertyValues(
    PropertyValues pvs, PropertyDescriptor[] pds, Object bean, String beanName)
    throws BeansException;
```
1.  InstantiationAwareBeanPostProcessor接口继承BeanPostProcessor接口，它内部提供了3个方法，再加上BeanPostProcessor接口内部的2个方法，所以实现这个接口需要实现5个方法。InstantiationAwareBeanPostProcessor接口的主要作用在于目标对象的实例化过程中需要处理的事情，包括实例化对象的前后过程以及实例的属性设置
2.  postProcessBeforeInstantiation方法是最先执行的方法，它在目标对象实例化之前调用，该方法的返回值类型是Object，我们可以返回任何类型的值。由于这个时候目标对象还未实例化，所以这个返回值可以用来代替原本该生成的目标对象的实例(比如代理对象)。如果该方法的返回值代替原本该生成的目标对象，后续只有postProcessAfterInitialization方法会调用，其它方法不再调用；否则按照正常的流程走
3.  postProcessAfterInstantiation方法在目标对象实例化之后调用，这个时候对象已经被实例化，但是该实例的属性还未被设置，都是null。如果该方法返回false，会忽略属性值的设置；如果返回true，会按照正常流程设置属性值
4.  postProcessPropertyValues方法对属性值进行修改(这个时候属性值还未被设置，但是我们可以修改原本该设置进去的属性值)。如果postProcessAfterInstantiation方法返回false，该方法不会被调用。可以在该方法内对属性值进行修改
5.  父接口BeanPostProcessor的2个方法postProcessBeforeInitialization和postProcessAfterInitialization都是在目标对象被实例化之后，并且属性也被设置之后调用的
6.  Instantiation表示实例化，Initialization表示初始化。实例化的意思在对象还未生成，初始化的意思在对象已经生成
## SmartInstantiationAwareBeanPostProcessor
SmartInstantiationAwareBeanPostProcessor接口继承InstantiationAwareBeanPostProcessor接口。多出了3个方法：
```java
// 预测Bean的类型，返回第一个预测成功的Class类型，如果不能预测返回null
Class<?> predictBeanType(Class<?> beanClass, String beanName) throws BeansException;
// 选择合适的构造器，比如目标对象有多个构造器，在这里可以进行一些定制化，选择合适的构造器
// beanClass参数表示目标实例的类型，beanName是目标实例在Spring容器中的name
// 返回值是个构造器数组，如果返回null，会执行下一个PostProcessor的determineCandidateConstructors方法；否则选取该PostProcessor选择的构造器
Constructor<?>[] determineCandidateConstructors(Class<?> beanClass, String beanName) throws BeansException;
// 获得提前暴露的bean引用。主要用于解决循环引用的问题
// 只有单例对象才会调用此方法
Object getEarlyBeanReference(Object bean, String beanName) throws BeansException;
```
1.  SmartInstantiationAwareBeanPostProcessor接口继承InstantiationAwareBeanPostProcessor接口，它内部提供了3个方法，再加上父接口的5个方法，所以实现这个接口需要实现8个方法。SmartInstantiationAwareBeanPostProcessor接口的主要作用也是在于目标对象的实例化过程中需要处理的事情。它是InstantiationAwareBeanPostProcessor接口的一个扩展。主要在Spring框架内部使用
2.  predictBeanType方法用于预测Bean的类型，返回第一个预测成功的Class类型，如果不能预测返回null。主要在于BeanDefinition无法确定Bean类型的时候调用该方法来确定类型
3.  determineCandidateConstructors方法用于选择合适的构造器，比如类有多个构造器，可以实现这个方法选择合适的构造器并用于实例化对象。该方法在postProcessBeforeInstantiation方法和postProcessAfterInstantiation方法之间调用，如果postProcessBeforeInstantiation方法返回了一个新的实例代替了原本该生成的实例，那么该方法会被忽略
4.  getEarlyBeanReference主要用于解决循环引用问题。比如ReferenceA实例内部有ReferenceB的引用，ReferenceB实例内部有ReferenceA的引用。首先先实例化ReferenceA，实例化完成之后提前把这个bean暴露在ObjectFactory中，然后populate属性，这个时候发现需要ReferenceB。然后去实例化ReferenceB，在实例化ReferenceB的时候它需要ReferenceA的实例才能继续，这个时候就会去ObjectFactory中找出了ReferenceA实例，ReferenceB顺利实例化。ReferenceB实例化之后，ReferenceA的populate属性过程也成功完成，注入了ReferenceB实例。提前把这个bean暴露在ObjectFactory中，这个ObjectFactory获取的实例就是通过getEarlyBeanReference方法得到的
## BeanPostProcessor
BeanPostProcessor接口是最顶层的接口，接口定义：
```java
public interface BeanPostProcessor {
    // 初始化之前的操作
    Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException;
    // 初始化之后的操作
    Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException;
}
```
1.  postProcessBeforeInitialization是指bean在初始化之前需要调用的方法
2.  postProcessAfterInitialization是指bean在初始化之后需要调用的方法
3.  postProcessBeforeInitialization和postProcessAfterInitialization方法被调用的时候。这个时候bean已经被实例化，并且所有该注入的属性都已经被注入，是一个完整的bean
4.  这2个方法的返回值可以是原先生成的实例bean，或者使用wrapper包装这个实例
# 何时调用
在我们调用getBean()方法时(也可能是注入时)，会调用AbstractAutowireCapableBeanFactory的creatBean()方法。在创建bean的过程中，会调用BeanPostProcessor。
![image.png](https://upload-images.jianshu.io/upload_images/10236819-dceb24db7d79010a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
在AbstractAutowireCapableBeanFactory类中，有三个方法分别是:applyBeanPostProcessorsBeforeInitialization、applyBeanPostProcessorsAfterInitialization、applyBeanPostProcessorsBeforeInstantiation,从名字就可以看出来是调用BeanPostProcessors的各种方法，我们分析一下applyBeanPostProcessorsBeforeInitialization。
```java
@Override
public Object applyBeanPostProcessorsBeforeInitialization(Object existingBean, String beanName)
        throws BeansException {

    Object result = existingBean;
    for (BeanPostProcessor beanProcessor : getBeanPostProcessors()) {
        result = beanProcessor.postProcessBeforeInitialization(result, beanName);
        if (result == null) {
            return result;
        }
    }
    return result;
}
```
```java
public List<BeanPostProcessor> getBeanPostProcessors() {
    return this.beanPostProcessors;
}
```
代码逻辑并不复杂，遍历所有的beanPostProcessors，然后执行postProcessBeforeInitialization方法，其他方法也是类似的执行方式。beanPostProcessors是类的成员变量，那么这个变量什么时候添加值进去的呢？
# 何时注册
这个可以追溯到容器创建的时候，在AbstractApplicationContext的refresh()方法中，调用了registerBeanPostProcessors(beanFactory)，这个方法实现逻辑是扫描所有实现了BeanPostProcessor的类型，并把它们注册到beanPostProcessors中，代码逻辑不难，可以自己去源码中查看这个方法的实现逻辑。