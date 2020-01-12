> MyBatis 的强大特性之一便是它的动态 SQL。如果你有使用 JDBC 或其它类似框架的经验，你就能体会到根据不同条件拼接 SQL 语句的痛苦。例如拼接时要确保不能忘记添加必要的空格，还要注意去掉列表最后一个列名的逗号。利用动态 SQL 这一特性可以彻底摆脱这种痛苦。

## 1. sql语句加载

要分析sql语句的加载，还是得从下面代码着手，分析mybatis是如何解析xml中的sql语句。

```java
SqlSessionFactory sqlSessionFactory = new SqlSessionFactoryBuilder().build(inputStream);
```

![](https://raw.githubusercontent.com/xiaoming-he/notes/master/img/20190821091541.png)

1.1 前面部分和mapper的配置文件加载过程一样，毕竟sql是写在mapper的配置文件中。我们从第4个方法开始

```java
public void parse() {
  if (!configuration.isResourceLoaded(resource)) {
    //解析配置文件
    configurationElement(parser.evalNode("/mapper"));
    configuration.addLoadedResource(resource);
    bindMapperForNamespace();
  }

  parsePendingResultMaps();
  parsePendingCacheRefs();
  parsePendingStatements();
}
```

1.2 configurationElement解析mapper中所有的标签，这里我们分析sql语句的解析

```java
private void configurationElement(XNode context) {
  try {
    String namespace = context.getStringAttribute("namespace");
    if (namespace == null || namespace.equals("")) {
      throw new BuilderException("Mapper's namespace cannot be empty");
    }
    builderAssistant.setCurrentNamespace(namespace);
    cacheRefElement(context.evalNode("cache-ref"));
    cacheElement(context.evalNode("cache"));
    parameterMapElement(context.evalNodes("/mapper/parameterMap"));
    resultMapElements(context.evalNodes("/mapper/resultMap"));
    //<sql>标签在sql语句解析之前
    sqlElement(context.evalNodes("/mapper/sql"));
    buildStatementFromContext(context.evalNodes("select|insert|update|delete"));
  } catch (Exception e) {
    throw new BuilderException("Error parsing Mapper XML. The XML location is '" + resource + "'. Cause: " + e, e);
  }
}
```

1.3 buildStatementFromContext解析mapper中的sql语句，sql语句解析是一个列表，循环解析，解析的工作交给了XMLStatementBuilder处理。

```java
private void buildStatementFromContext(List<XNode> list, String requiredDatabaseId) {
  for (XNode context : list) {
    final XMLStatementBuilder statementParser = new XMLStatementBuilder(configuration, builderAssistant, context, requiredDatabaseId);
    try {
      statementParser.parseStatementNode();
    } catch (IncompleteElementException e) {
      configuration.addIncompleteStatement(statementParser);
    }
  }
}
```

1.4 XMLStatementBuilder的parseStatementNode解析sql标签的属性，代码比较长，我们主要关注sql语句是如何解析的。

```java
public void parseStatementNode() {
    ....

    //1. 解析include标签，include直接将sql复制过来
    // Include Fragments before parsing
    XMLIncludeTransformer includeParser = new XMLIncludeTransformer(configuration, builderAssistant);
    includeParser.applyIncludes(context.getNode());
    
  // 2. 解析sql语句
    // Parse the SQL (pre: <selectKey> and <include> were parsed and removed)
    SqlSource sqlSource = langDriver.createSqlSource(configuration, context, parameterTypeClass);
    ...

    builderAssistant.addMappedStatement(id, sqlSource, statementType, sqlCommandType,
        fetchSize, timeout, parameterMap, parameterTypeClass, resultMap, resultTypeClass,
        resultSetTypeEnum, flushCache, useCache, resultOrdered, 
        keyGenerator, keyProperty, keyColumn, databaseId, langDriver, resultSets);
  }
```

1.5 langDriver指的是XMLLanguageDriver对象，干活的是XMLScriptBuilder对象，XMLScriptBuilder会把sql语句解析生成SqlSource对象

```java
public SqlSource parseScriptNode() {
  //动态sql的核心解析
  MixedSqlNode rootSqlNode = parseDynamicTags(context);
  SqlSource sqlSource = null;
  if (isDynamic) {
    sqlSource = new DynamicSqlSource(configuration, rootSqlNode);
  } else {
    sqlSource = new RawSqlSource(configuration, rootSqlNode, parameterType);
  }
  return sqlSource;
}
```

如果语句是动态的，则是生成DynamicSqlSource对象，所以我们主要看DynamicSqlSource对象。

1.6 parseDynamicTags是解析sql的核心方法，把所有的sql标签封装成SqlNode对象。这里的设计思路挺好的，用命令模式把不同的标签解析成不同的SqlNode对象，用命令模式把不同标签中的内容组合成sqlNode。

1.7 sql解析出来后，就会将select|insert|update上标签的属性封装到MappedStatement中，并且添加到Configuration中。

```java
public void addMappedStatement(MappedStatement ms) {
  //key是mapper的namespace+id
  mappedStatements.put(ms.getId(), ms);
}
```

## 2. 动态sql生成

上面讲了动态sql的加载，我们在运行时的动态sql是如何生成的呢？由Mapper原理分析知道，mybatis对sql的执行最终是交给sqlSession中的各种方法，这里我们分析DefaultSqlSession中的selectList方法。

2.1 selectList方法先从Configuration中得到MappedStatement对象，MappedStatement对象是sql加载最终生成的对象。得到MappedStatement后，把活交给了Executor对象。

```java
@Override
public <E> List<E> selectList(String statement, Object parameter, RowBounds rowBounds) {
  try {
    MappedStatement ms = configuration.getMappedStatement(statement);
    return executor.query(ms, wrapCollection(parameter), rowBounds, Executor.NO_RESULT_HANDLER);
  } catch (Exception e) {
    throw ExceptionFactory.wrapException("Error querying database.  Cause: " + e, e);
  } finally {
    ErrorContext.instance().reset();
  }
}
```

2.2 Executor有多个实现类，query操作交给了父类BaseExecutor，BaseExecutor首先就是从MappedStatement获取sql。

```java
@Override
public <E> List<E> query(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler) throws SQLException {
  BoundSql boundSql = ms.getBoundSql(parameter);
  CacheKey key = createCacheKey(ms, parameter, rowBounds, boundSql);
  return query(ms, parameter, rowBounds, resultHandler, key, boundSql);
}
```

2.3 MappedStatement在sql加载的时候存了一个sqlSource对象，如果是动态sql，则是DynamicSqlSource对象。获取sql的任务又交给了DynamicSqlSource。

```java
public BoundSql getBoundSql(Object parameterObject) {
  BoundSql boundSql = sqlSource.getBoundSql(parameterObject);
  List<ParameterMapping> parameterMappings = boundSql.getParameterMappings();
  if (parameterMappings == null || parameterMappings.isEmpty()) {
    boundSql = new BoundSql(configuration, boundSql.getSql(), parameterMap.getParameterMappings(), parameterObject);
  }

  // check for nested result maps in parameter mappings (issue #30)
  for (ParameterMapping pm : boundSql.getParameterMappings()) {
    String rmId = pm.getResultMapId();
    if (rmId != null) {
      ResultMap rm = configuration.getResultMap(rmId);
      if (rm != null) {
        hasNestedResultMaps |= rm.hasNestedResultMaps();
      }
    }
  }
  return boundSql;
}
```

2.4 DynamicSqlSource是最终得到sql语句的地方。rootSqlNode是指的MixedSqlNode对象，上文说过SqlNode是用的组合模式。所以rootSqlNode.apply(context)是解析这个数。

```java
@Override
public BoundSql getBoundSql(Object parameterObject) {
  DynamicContext context = new DynamicContext(configuration, parameterObject);
  //组合模式，解析树结构组装sql语句，设计的很好
  rootSqlNode.apply(context);
  SqlSourceBuilder sqlSourceParser = new SqlSourceBuilder(configuration);
  Class<?> parameterType = parameterObject == null ? Object.class : parameterObject.getClass();
  SqlSource sqlSource = sqlSourceParser.parse(context.getSql(), parameterType, context.getBindings());
  BoundSql boundSql = sqlSource.getBoundSql(parameterObject);
  for (Map.Entry<String, Object> entry : context.getBindings().entrySet()) {
    boundSql.setAdditionalParameter(entry.getKey(), entry.getValue());
  }
  return boundSql;
}
```



