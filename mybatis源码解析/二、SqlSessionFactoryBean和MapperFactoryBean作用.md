# SqlSessionFactoryBean
SqlSessionFactoryBean实现了FactoryBean接口，在创建bean时会调用getObject方法。
```java
@Override
public SqlSessionFactory getObject() throws Exception {
if (this.sqlSessionFactory == null) {
    afterPropertiesSet();
}

return this.sqlSessionFactory;
}
```
如果sqlSessionFactory对象为空，则调用 afterPropertiesSet()方法，因为SqlSessionFactoryBean还实现了InitializingBean接口，在bean创建的时候就会调用afterPropertiesSet()方法。所以这步this.sqlSessionFactory == null只是增加个校验。
```java
@Override
public void afterPropertiesSet() throws Exception {
notNull(dataSource, "Property 'dataSource' is required");
notNull(sqlSessionFactoryBuilder, "Property 'sqlSessionFactoryBuilder' is required");
state((configuration == null && configLocation == null) || !(configuration != null && configLocation != null),
            "Property 'configuration' and 'configLocation' can not specified with together");

this.sqlSessionFactory = buildSqlSessionFactory();
}
```
afterPropertiesSet方法调用buildSqlSessionFactory方法创建sqlSessionFactory对象。buildSqlSessionFactory方法很长，不过主要是给Configuration对象设置属性值，这里就不贴代码了。
# MapperFactoryBean
MapperFactoryBean是用来创建MyBatis Mapper对象的，MapperFactoryBean也实现了FactoryBean接口，间接实现InitializingBean接口，我们先看下InitializingBean接口的afterPropertiesSet实现。
```java
@Override
public final void afterPropertiesSet() throws IllegalArgumentException, BeanInitializationException {
    // Let abstract subclasses check their configuration.
    checkDaoConfig();

    // Let concrete implementations initialize themselves.
    try {
        //空实现
        initDao();
    }
    catch (Exception ex) {
        throw new BeanInitializationException("Initialization of DAO failed", ex);
    }
}
```
checkDaoConfig方法由MapperFactoryBean实现。主要是把Mapper接口添加到Configuration对象中。
```java
@Override
protected void checkDaoConfig() {
    super.checkDaoConfig();

    notNull(this.mapperInterface, "Property 'mapperInterface' is required");
    //如果Configuration中不存在Mapper，则添加Mapper
    Configuration configuration = getSqlSession().getConfiguration();
    if (this.addToConfig && !configuration.hasMapper(this.mapperInterface)) {
        try {
        configuration.addMapper(this.mapperInterface);
        } catch (Exception e) {
        logger.error("Error while adding the mapper '" + this.mapperInterface + "' to configuration.", e);
        throw new IllegalArgumentException(e);
        } finally {
        ErrorContext.instance().reset();
        }
    }
}
```
接下来我们再看MapperFactoryBean中的getObject方法。
```java
@Override
public T getObject() throws Exception {
    return getSqlSession().getMapper(this.mapperInterface);
}
```
getObject方法返回Mapper对象，Mapper对象是从Configuration中获取。
