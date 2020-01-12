## 1. 使用

使用可以参考http://www.mybatis.org/mybatis-3/zh/getting-started.html

## 2. mapper接口与配置绑定

在使用mybatis时，我一直有个疑问，我们定义的mapper接口和xml文件是如何关联起来的的，为什么xml的namespace要和接口的全路径相同。

这个分析要从mybatis的配置文件加载开始，分析下面一行代码:

```java
String resource = "mybatis-config.xml";
InputStream inputStream = Resources.getResourceAsStream(resource);
SqlSessionFactory sqlSessionFactory = new SqlSessionFactoryBuilder().build(inputStream);
```

![](https://raw.githubusercontent.com/xiaoming-he/notes/master/img/20190817194523.png)

2.1 首先SqlSessionFactoryBuilder调用build会去加载配置文件mybatis.xml 

```java
public SqlSessionFactory build(InputStream inputStream, String environment, Properties properties) {
  try {
    XMLConfigBuilder parser = new XMLConfigBuilder(inputStream, environment, properties);
    return build(parser.parse());
  } catch (Exception e) {
    throw ExceptionFactory.wrapException("Error building SqlSession.", e);
  } finally {
    ErrorContext.instance().reset();
    try {
      inputStream.close();
    } catch (IOException e) {
      // Intentionally ignore. Prefer previous error.
    }
  }
}
```

2.2 加载配置文件交给XMLConfigBuilder去解析配置文件，XMLConfigBuilder按照标签一个一个解析。

```java
private void parseConfiguration(XNode root) {
  try {
    //issue #117 read properties first
    propertiesElement(root.evalNode("properties"));
    Properties settings = settingsAsProperties(root.evalNode("settings"));
    loadCustomVfs(settings);
    typeAliasesElement(root.evalNode("typeAliases"));
    pluginElement(root.evalNode("plugins"));
    objectFactoryElement(root.evalNode("objectFactory"));
    objectWrapperFactoryElement(root.evalNode("objectWrapperFactory"));
    reflectorFactoryElement(root.evalNode("reflectorFactory"));
    settingsElement(settings);
    // read it after objectFactory and objectWrapperFactory issue #631
    environmentsElement(root.evalNode("environments"));
    databaseIdProviderElement(root.evalNode("databaseIdProvider"));
    typeHandlerElement(root.evalNode("typeHandlers"));
    mapperElement(root.evalNode("mappers"));
  } catch (Exception e) {
    throw new BuilderException("Error parsing SQL Mapper Configuration. Cause: " + e, e);
  }
}
```

这节主要研究mapper的加载过程，所以我们主要看mapperElement(root.evalNode("mappers"));方法。

2.3 解析mappers标签，如果mappers的子标签是package，则加载包下所有的mapper，如果子标签是mapper，则根据mapper的属性值加载mapper。

```java
private void mapperElement(XNode parent) throws Exception {
  if (parent != null) {
    for (XNode child : parent.getChildren()) {
      if ("package".equals(child.getName())) {
        String mapperPackage = child.getStringAttribute("name");
        configuration.addMappers(mapperPackage);
      } else {
        String resource = child.getStringAttribute("resource");
        String url = child.getStringAttribute("url");
        String mapperClass = child.getStringAttribute("class");
        if (resource != null && url == null && mapperClass == null) {
          ErrorContext.instance().resource(resource);
          InputStream inputStream = Resources.getResourceAsStream(resource);
          XMLMapperBuilder mapperParser = new XMLMapperBuilder(inputStream, configuration, resource, configuration.getSqlFragments());
          //解析mapper的xml文件 
          mapperParser.parse();
        } else if (resource == null && url != null && mapperClass == null) {
          ErrorContext.instance().resource(url);
          InputStream inputStream = Resources.getUrlAsStream(url);
          XMLMapperBuilder mapperParser = new XMLMapperBuilder(inputStream, configuration, url, configuration.getSqlFragments());
          mapperParser.parse();
        } else if (resource == null && url == null && mapperClass != null) {
          Class<?> mapperInterface = Resources.classForName(mapperClass);
          configuration.addMapper(mapperInterface);
        } else {
          throw new BuilderException("A mapper element may only specify a url, resource or class, but not more than one.");
        }
      }
    }
  }
 }
```

判断属性后，都会交给XMLMapperBuilder去解析mapper的xml文件。

2.4 XMLMapperBuilder的parse方法会解析mapper的xml配置文件，并把解析的内容都存放到Configuration中。

```java
public void parse() {
  if (!configuration.isResourceLoaded(resource)) {
    configurationElement(parser.evalNode("/mapper"));
    configuration.addLoadedResource(resource);
    //主要是这行代码，让mapper接口与mapper配置文件关联起来
    bindMapperForNamespace();
  }

  parsePendingResultMaps();
  parsePendingCacheRefs();
  parsePendingStatements();
}
```

主要关注bindMapperForNamespace方法，这个方法将mapper接口与mapper配置文件关联起来

2.5 bindMapperForNamespace方法获得mapper配置文件的namespace，根据namespace路径去加载mapper接口类，如果加载成功，则添加到Configuration中.

```java
private void bindMapperForNamespace() {
  String namespace = builderAssistant.getCurrentNamespace();
  if (namespace != null) {
    Class<?> boundType = null;
    try {
      boundType = Resources.classForName(namespace);
    } catch (ClassNotFoundException e) {
      //ignore, bound type is not required
    }
    if (boundType != null) {
      if (!configuration.hasMapper(boundType)) {
        // Spring may not know the real resource name so we set a flag
        // to prevent loading again this resource from the mapper interface
        // look at MapperAnnotationBuilder#loadXmlResource
        configuration.addLoadedResource("namespace:" + namespace);
        configuration.addMapper(boundType);
      }
    }
  }
}
```

所以就是在这个地方，mapper的接口与mapper文件产生了关联.

2.6 当mapper接口与mapper配置文件绑定之后，就会注册到Configuration中，Configruation把这个操作交给了MapperRegistry处理。这点mybatis的代码写的还是挺好的，mybatis所有的操作基本都围绕Configuration，但是Configuration自己不处理业务，基本都是委托别人处理。

```java
public <T> void addMapper(Class<T> type) {
  if (type.isInterface()) {
    if (hasMapper(type)) {
      throw new BindingException("Type " + type + " is already known to the MapperRegistry.");
    }
    boolean loadCompleted = false;
    try {
      knownMappers.put(type, new MapperProxyFactory<T>(type));
      // It's important that the type is added before the parser is run
      // otherwise the binding may automatically be attempted by the
      // mapper parser. If the type is already known, it won't try.
      //注解解析
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

MapperRegistry会判断添加的mapper是不是一个接口，如果是一个接口，则创建一个MapperProxyFactory对象，并把它存到knownMappers中，从名字可以大概猜出MapperProxyFactory的作用，用于创建MapperProxy对象，MapperProxy的作用是什么在以后讨论。MapperAnnotationBuilder是解析mapper接口中的注解的，比如@Select，@insert。

根据注解的解析和xml的解析顺序，又可以解决一个疑问，如果同一个方法，注解和xml都配置了，会使用哪一个？由代码可以看出，注解解析在前，xml解析在后，xml的会覆盖注解的，所以最终使用的是xml配置的。

## 3. 生成mapper对象

我们的mapper接口与xml绑定后，又是怎样去调用的，因为我们并没有创建mapper接口的实现类，但是却可以调用方法。所以实现类肯定是由mybatis生成，mybatis又是怎样生成的呢？先看下流程图。

3.1 在上一节中，我们知道mapper接口与xml配置绑定后，会注册到MapperRegistry中，所以要Mapper的获取也是从MapperRegistry中。

```java
public <T> T getMapper(Class<T> type, SqlSession sqlSession) {
  final MapperProxyFactory<T> mapperProxyFactory = (MapperProxyFactory<T>) knownMappers.get(type);
  if (mapperProxyFactory == null) {
    throw new BindingException("Type " + type + " is not known to the MapperRegistry.");
  }
  try {
    return mapperProxyFactory.newInstance(sqlSession);
  } catch (Exception e) {
    throw new BindingException("Error getting mapper instance. Cause: " + e, e);
  }
}
```

会首先拿到绑定时生成的MapperProxyFactory对象，然后调用MapperProxyFactory的newInstace生成Mapper对象。

3.2 MapperProxyFactory的newInstace先生成一个mapperProxy对象，然后动态代理生成Mapper对象。

```java
protected T newInstance(MapperProxy<T> mapperProxy) {
  return (T) Proxy.newProxyInstance(mapperInterface.getClassLoader(), new Class[] { mapperInterface }, mapperProxy);
}

public T newInstance(SqlSession sqlSession) {
  final MapperProxy<T> mapperProxy = new MapperProxy<T>(sqlSession, mapperInterface, methodCache);
  return newInstance(mapperProxy);
}
```

3.3 上面的代码使用的是jdk动态代理，但由不完全是动态代理，真正的动态代理结构如下图所示:

![](https://raw.githubusercontent.com/xiaoming-he/notes/master/img/20190820144427.png)

但Mapper的动态代理并没有实现类，结构图如下

![](https://raw.githubusercontent.com/xiaoming-he/notes/master/img/20190820144530.png)

mybatis设计一个接口的目的只是方便使用者，编写时方便，不然可以直接通过xml配置，然后根据id执行sql。

## 4. 调用Mapper方法

Mapper对象生成后，Mapper接口中的方法是如何调用的？由第三小节知道，Mapper对象是动态代理生成的，真正的调用对象是mapperProxy的invoke方法。

4.1 invoke方法先解析调用的方法签名和方法对应的sql，封装成MapperMethod对象并缓存起来，然后调用MapperMethod的execute方法。

```java
public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
  try {
    if (Object.class.equals(method.getDeclaringClass())) {
      return method.invoke(this, args);
    } else if (isDefaultMethod(method)) {
      return invokeDefaultMethod(proxy, method, args);
    }
  } catch (Throwable t) {
    throw ExceptionUtil.unwrapThrowable(t);
  }
  //换成MapperMethod对象，主要是方法的签名和sql信息
  final MapperMethod mapperMethod = cachedMapperMethod(method);
  return mapperMethod.execute(sqlSession, args);
}
```

4.2 MapperMethod的execute方法根据sql的类型，调用SqlSession的方法，所以Mybatis的所有对sql的操作，最终多事交给了SqlSession去执行。

```java
public Object execute(SqlSession sqlSession, Object[] args) {
  Object result;
  switch (command.getType()) {
    case INSERT: {
      Object param = method.convertArgsToSqlCommandParam(args);
      result = rowCountResult(sqlSession.insert(command.getName(), param));
      break;
    }
    case UPDATE: {
      Object param = method.convertArgsToSqlCommandParam(args);
      result = rowCountResult(sqlSession.update(command.getName(), param));
      break;
    }
    case DELETE: {
      Object param = method.convertArgsToSqlCommandParam(args);
      result = rowCountResult(sqlSession.delete(command.getName(), param));
      break;
    }
    case SELECT:
      if (method.returnsVoid() && method.hasResultHandler()) {
        executeWithResultHandler(sqlSession, args);
        result = null;
      } else if (method.returnsMany()) {
        result = executeForMany(sqlSession, args);
      } else if (method.returnsMap()) {
        result = executeForMap(sqlSession, args);
      } else if (method.returnsCursor()) {
        result = executeForCursor(sqlSession, args);
      } else {
        Object param = method.convertArgsToSqlCommandParam(args);
        result = sqlSession.selectOne(command.getName(), param);
      }
      break;
    case FLUSH:
      result = sqlSession.flushStatements();
      break;
    default:
      throw new BindingException("Unknown execution method for: " + command.getName());
  }
  if (result == null && method.getReturnType().isPrimitive() && !method.returnsVoid()) {
    throw new BindingException("Mapper method '" + command.getName() 
                               + " attempted to return null from a method with a primitive return type (" + method.getReturnType() + ").");
  }
  return result;
}
```





