---
title: 批量获取bibtex(一步到位）
date: 2024-04-16
authors: [刘耀文]
categories:
  - 源码阅读

---

# Mybatis[plus] 源码阅读笔记

## 调用流程概述：

mybatis的前身是ibatis，原生的ibatis执行curd操作是固定的操作，使用SQLSession接口中的方法进行增删查改，我们先阅读Ibatis部分的代码，后续mybatis[plus]都是在此基础上扩展了MapperProxy以及预设sql语句动态sql条件等封装，我们看一下ibatis的Session核心接口：

<!-- more -->

```java
public interface SqlSession extends Closeable {
    
  <E> List<E> selectList(String statement);

  <K, V> Map<K, V> selectMap(String statement, Object parameter, String mapKey);

  <T> Cursor<T> selectCursor(String statement);

  int insert(String statement);

  int update(String statement);

  int delete(String statement);

}

```

这个接口里面最终要的四类方法为（selectList、selectMap、selectCursor）、insert、delete、update。

但其实最终执行sql语句的为Executor中的方法

```java
public interface Executor {

  int update(MappedStatement ms, Object parameter) throws SQLException;

  <E> List<E> query(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler,
      CacheKey cacheKey, BoundSql boundSql) throws SQLException;

  <E> List<E> query(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler)
      throws SQLException;

  <E> Cursor<E> queryCursor(MappedStatement ms, Object parameter, RowBounds rowBounds) throws SQLException;
}
```

其中分为query和update方法，最后由实际子类的doQuery、doUpdate执行真正的sql语句，其中进一步交给了StatementHandler对象处理

```java
public interface StatementHandler {

  Statement prepare(Connection connection, Integer transactionTimeout) throws SQLException;

  void parameterize(Statement statement) throws SQLException;

  void batch(Statement statement) throws SQLException;

  int update(Statement statement) throws SQLException;

  <E> List<E> query(Statement statement, ResultHandler resultHandler) throws SQLException;

  <E> Cursor<E> queryCursor(Statement statement) throws SQLException;

  BoundSql getBoundSql();

  ParameterHandler getParameterHandler();

}
```

可以看到里面最终到达update或者query方法

![image-20240611225027427](https://raw.githubusercontent.com/liyown/pic-go/master/blog/%E6%BA%90%E7%A0%81%E9%98%85%E8%AF%BB_mybatis%5Bplus%5D.md)

所以调用逻辑为SqlSession->Executor->StatementHandler，每个SqlSession有一个Executor，每个Executor有一个StatementHandler（configuration中获得的，经过插件的包装，后面会说）；

SqlSession的创建过程：

![image-20240611231131172](https://raw.githubusercontent.com/liyown/pic-go/master/blog/image-20240611231131172.png)

其中的核心为Configuration，整个ibatis都围绕这这个Configuration类，非常重要负责Executor、ParameterHandler、ResultSetHandler的创建，MapperStatements的保存

## 一、二级缓存

**一级缓存**

存在与执行器中，每次创建都会携带，所以每次会话都会有一级缓存的存在：BaseExecutor.java

```java
  public <E> List<E> query(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, CacheKey key, BoundSql boundSql) throws SQLException {
    ErrorContext.instance().resource(ms.getResource()).activity("executing a query").object(ms.getId());
    if (closed) {
      throw new ExecutorException("Executor was closed.");
    }
    if (queryStack == 0 && ms.isFlushCacheRequired()) {
      clearLocalCache();
    }
    List<E> list;
    try {
      queryStack++;
      list = resultHandler == null ? (List<E>) localCache.getObject(key) : null;
      if (list != null) {
        handleLocallyCachedOutputParameters(ms, key, parameter, boundSql);
      } else {
        list = queryFromDatabase(ms, parameter, rowBounds, resultHandler, key, boundSql);
      }
    } finally {
      queryStack--;
    }
    if (queryStack == 0) {
      for (DeferredLoad deferredLoad : deferredLoads) {
        deferredLoad.load();
      }
      // issue #601
      deferredLoads.clear();
      if (configuration.getLocalCacheScope() == LocalCacheScope.STATEMENT) {
        // issue #482
        clearLocalCache();
      }
    }
    return list;
  }
```

在查询之前，会先去缓存里面查找，其中key值与四个方面有关

```java
CacheKey key = createCacheKey(ms, parameter, rowBounds, boundSql);
```

只有当sql语句、参数、分页、与绑定参数的sql一样才会匹配；

**二级缓存**

存在与ms中的cache属性当中，只有当configuration中的cacheEnable为true才会创建cacheExecutor

```java
// Configuration.java
  public Executor newExecutor(Transaction transaction, ExecutorType executorType) {
    executorType = executorType == null ? defaultExecutorType : executorType;
    Executor executor;
    if (ExecutorType.BATCH == executorType) {
      executor = new BatchExecutor(this, transaction);
    } else if (ExecutorType.REUSE == executorType) {
      executor = new ReuseExecutor(this, transaction);
    } else {
      executor = new SimpleExecutor(this, transaction);
    }
    if (cacheEnabled) {
      executor = new CachingExecutor(executor);
    }
    executor = (Executor) interceptorChain.pluginAll(executor);
    return executor;
  }
// 默认为true
```

同时ms中的useCache为true，才会实际操作ms中的cache

```java
// CachingExecutor.java
  public <E> List<E> query(MappedStatement ms, Object parameterObject, RowBounds rowBounds, ResultHandler resultHandler, CacheKey key, BoundSql boundSql)
      throws SQLException {
    Cache cache = ms.getCache();
    if (cache != null) {
      flushCacheIfRequired(ms);
      if (ms.isUseCache() && resultHandler == null) {
        ensureNoOutParams(ms, boundSql);
        @SuppressWarnings("unchecked")
        List<E> list = (List<E>) tcm.getObject(cache, key);
        if (list == null) {
          list = delegate.query(ms, parameterObject, rowBounds, resultHandler, key, boundSql);
          tcm.putObject(cache, key, list); // issue #578 and #116
        }
        return list;
      }
    }
    return delegate.query(ms, parameterObject, rowBounds, resultHandler, key, boundSql);
  }
// 默认也为true
```

可以看到，只有当cach不为null才会实际操作缓存，在解析mapper.xml的时候才会判断是否设置了cache类型

```java
// XMLMapperBuilder.java  
private void configurationElement(XNode context) {
    try {
      String namespace = context.getStringAttribute("namespace");
      if (namespace == null || namespace.isEmpty()) {
        throw new BuilderException("Mapper's namespace cannot be empty");
      }
      builderAssistant.setCurrentNamespace(namespace);
      cacheRefElement(context.evalNode("cache-ref"));
      cacheElement(context.evalNode("cache"));
      parameterMapElement(context.evalNodes("/mapper/parameterMap"));
      resultMapElements(context.evalNodes("/mapper/resultMap"));
      sqlElement(context.evalNodes("/mapper/sql"));
      buildStatementFromContext(context.evalNodes("select|insert|update|delete"));
    } catch (Exception e) {
      throw new BuilderException("Error parsing Mapper XML. The XML location is '" + resource + "'. Cause: " + e, e);
    }
  }
// 类型有很多种
```

![](https://raw.githubusercontent.com/liyown/pic-go/master/blog/image-20240616163813806.png)

