上节[二、SqlSessionFactoryBean和MapperFactoryBean作用](https://www.jianshu.com/p/3e39a3bf7ccb)我们分析MapperFactoryBean对象在初始化的时候会将Mapper接口添加到Configuration对象中，在getObject时又会从Configuration对象中获取一个Mapper对象。这背后到底发生了什么，传入一个接口是如何返回一个对象的，是我们这节分析的重点。
# Configuration#addMapper(Class<T> type)
```java
 public <T> void addMapper(Class<T> type) {
    mapperRegistry.addMapper(type);
  }
```
把工作交给了mapperRegistry去处理。
```java
public <T> void addMapper(Class<T> type) {
    //1. 验证传入的类型是不是接口
    if (type.isInterface()) {
      //2. 验证是否已经加载过
      if (hasMapper(type)) {
        throw new BindingException("Type " + type + " is already known to the MapperRegistry.");
      }
      boolean loadCompleted = false;
      try {
        //3. 添加mapper接口到knownMappers，表示已经注册
        knownMappers.put(type, new MapperProxyFactory<T>(type));
        //4. 解析Mapper对应的xml
        MapperAnnotationBuilder parser = new MapperAnnotationBuilder(config, type);
        parser.parse();
        loadCompleted = true;
      } finally {
        if (!loadCompleted) {
          knownMappers.remove(type);
        }
      }
    }
 }
```
addMapper的逻辑如下:
1. 验证传入的类型是不是接口
2. 验证是否已经加载过
```java
public <T> boolean hasMapper(Class<T> type) {
    return knownMappers.containsKey(type);
}
```
3. 添加mapper接口到knownMappers,创建MapperProxyFactory对象，这个对象的作用在下面会讲。
4. 解析Mapper对应的xml文件,详情看下一小节。
## MapperAnnotationBuilder
MapperAnnotationBuilder作用是用来解析Mapper接口的xml文件和Mapper接口上的注解。看下它的parse方法。
```java
public void parse() {
    String resource = type.toString();
    //1. 判断类是否加载过
    if (!configuration.isResourceLoaded(resource)) {
      //2. 加载对应的xml文件
      loadXmlResource();
      configuration.addLoadedResource(resource);
      assistant.setCurrentNamespace(type.getName());
      //3. 后面的都是解析接口中的注解
      parseCache();
      parseCacheRef();
      Method[] methods = type.getMethods();
      for (Method method : methods) {
        try {
          // issue #237
          if (!method.isBridge()) {
            parseStatement(method);
          }
        } catch (IncompleteElementException e) {
          configuration.addIncompleteMethod(new MethodResolver(this, method));
        }
      }
    }
    parsePendingMethods();
}
```
解析的过程如下:
1. 判断类是否加载过
2. 解析对应的xml文件,根据接口全名获取xml路径
```java
private void loadXmlResource() {
if (!configuration.isResourceLoaded("namespace:" + type.getName())) {
    //根据接口全名获取xml路径
    String xmlResource = type.getName().replace('.', '/') + ".xml";
    InputStream inputStream = null;
    try {
    inputStream = Resources.getResourceAsStream(type.getClassLoader(), xmlResource);
    } catch (IOException e) {
    // ignore, resource is not required
    }
    if (inputStream != null) {
    XMLMapperBuilder xmlParser = new XMLMapperBuilder(inputStream, assistant.getConfiguration(), xmlResource, configuration.getSqlFragments(), type.getName());
    xmlParser.parse();
    }
}
}
```
3. 解析类中注解
# Configuration#getMapper(Class<T> type, SqlSession sqlSession)
```java
public <T> T getMapper(Class<T> type, SqlSession sqlSession) {
    return mapperRegistry.getMapper(type, sqlSession);
}
```
getMapper方法还是交给了mapperRegistry处理。
```java
public <T> T getMapper(Class<T> type, SqlSession sqlSession) {
    //1. 获得接口对应的MapperProxyFactory对象,这个对象是在addMapper中创建的
    final MapperProxyFactory<T> mapperProxyFactory = (MapperProxyFactory<T>) knownMappers.get(type);
    if (mapperProxyFactory == null) {
      throw new BindingException("Type " + type + " is not known to the MapperRegistry.");
    }
    try {
      //2. MapperProxyFactory通过动态代理创建对象
      return mapperProxyFactory.newInstance(sqlSession);
    } catch (Exception e) {
      throw new BindingException("Error getting mapper instance. Cause: " + e, e);
    }
}
```
1. 获得接口对应的MapperProxyFactory对象,这个对象是在addMapper中创建的
2. MapperProxyFactory通过动态代理根据接口创建对象, 看下MapperProxyFactory对象的newInstance方法
```java
public T newInstance(SqlSession sqlSession) {
    final MapperProxy<T> mapperProxy = new MapperProxy<T>(sqlSession, mapperInterface, methodCache);
    return newInstance(mapperProxy);
}

protected T newInstance(MapperProxy<T> mapperProxy) {
    return (T) Proxy.newProxyInstance(mapperInterface.getClassLoader(), new Class[] { mapperInterface }, mapperProxy);
}
```
动态代理创建对象的业务逻辑交给了MapperProxy对象。

# 总结
MapperFactoryBean传入一个接口，是把接口注册到Configuration对象中，并且解析对应的xml。MapperFactoryBean获取的对象是根据传入的接口动态代理创建了一个对象，对象的逻辑交给了MapperProxy处理。 