　　一般情况下，Spring通过反射机制利用bean的class属性指定实现类来实例化bean。在某些情况下，实例化bean过程比较复杂，如果按照传统的方式，则需要在<bean>中提供大量的配置信息，配置方式的灵活性是受限的，这是采用编码的方式可能得到一个简单的方案。Spring为此提供了一个FactoryBean的工厂接口，用户可以通过实现该接口定制实例化bean的逻辑。个人觉得在spring4.0之后可以用@Bean注解取代FactoryBean。
#　FactoryBean接口
该接口中定义了三个方法：
```java
//返回由FactoryBean创建的bean实例
T getObject() throws Exception;

//返回FactoryBean创建的bean类型
Class<?> getObjectType();

//返回由FactoryBean创建的bean实例作用域
boolean isSingleton();
```
当配置文件中的\<bean>的class属性配置的实现类是FactoryBean时，通过getBean()方法返回的不是FactoryBean本身，而是Factory#getObject()方法所返回的对象，相当于FactoryBean#getObject()代理了getBean()方法。
# 使用案例
Mybatis与spring整合，获取SqlSessionFactory就是通过FactoryBean，我们整合时的配置如下：
```xml
<bean id="sqlSessionFactory" class="org.mybatis.spring.SqlSessionFactoryBean">
  <property name="dataSource" ref="dataSource" />
</bean>
```
SqlSessionFactoryBean实现了FactoryBean，getObject()方法返回的就是SqlSessionFactory对象。
```java
public class SqlSessionFactoryBean implements FactoryBean<SqlSessionFactory>, InitializingBean, ApplicationListener<ApplicationEvent> {
    ...
    @Override
  public SqlSessionFactory getObject() throws Exception {
    if (this.sqlSessionFactory == null) {
      afterPropertiesSet();
    }
    return this.sqlSessionFactory;
  }
    ...
}
```
# 原理解析