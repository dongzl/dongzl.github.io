---
title: Spring 事务管理 Timeout 的一点问题研究
date: 2020-08-04 20:23:45
cover: https://gitee.com/dongzl/article-images/raw/master/cover/spring_study.png
# author information, multiple authors are set to array
# single author
author:
  - nick: 董宗磊
    link: https://www.github.com/dongzl

# post subtitle in your index page
subtitle: 本文是工作中使用 Spring 提供的事务管理机制时遇到的一点小问题的研究。

categories: 
  - web开发

tags: 
  - Spring
---

### 背景描述

最近排查一个老系统的问题，比较诡异，一个导入 `Excel` 表格大批量创建商品的功能，由于创建商品需要操作好多个不同的表，因此使用了数据库事务保证数据的完整性，最近经常出现大量创建商品时系统运行一段时间卡住不动了，数据库上就会出现几个已经运行很长时间无法提交的事务，找 DBA 排查也只能杀掉事务，无法准确定位到底是哪个 `SQL` 导致事务无法提交。

推测感觉是在事务当中有耗时的操作，导致事务卡住，程序无法继续执行，事务无法提交，也无法回滚，类似于在事务中进行了非常耗时的 `RPC` 调用，这是明令禁止的，怀疑可能是类似的问题。

现在运行的老系统，都是用的 Sping 编程式事务管理方式，没有设置事务的超时时间，使用默认值，想着通过事务的超时时间设置尝试解决这个问题，就引出了这一篇的内容。

**结论：线上结论吧，其实通过设置超时时间解决这个问题有点本末倒置，如果程序在事务当中执行了一个非常耗时的操作，上不来也下不去了，卡在中间，不能提交也无法回滚，设置了事务的超时时间也是没什么作用的。**

### Spring 事务管理超时时间设置

```xml
<bean name="productTransactionManager" class="org.springframework.jdbc.datasource.DataSourceTransactionManager">
    <property name="dataSource" ref="productDataSource"/>
</bean>
```

```java
private PlatformTransactionManager productTransactionManager;

public TransactionTemplate getProductTransactionManager() {
    return new TransactionTemplate(productTransactionManager);
}

public void updateProduct(Product product) {
    TransactionTemplate template = getProductTransactionManager();
    template.setTimeout(3);
    template.execute(new TransactionCallback<Boolean>() {
        @Override
        public Boolean doInTransaction(final TransactionStatus status) {
            try {
                insertA(product);
                updateB(product);
                updateC(product);
                return true;
            } catch (Exception e) {
                status.setRollbackOnly();
                return false;
            }
        }
    });
}
```

通过设置 `TransactionTemplate` 对象的 `timeout` 属性，我们将超时时间设置为 3s，默认值是 -1。

下面是模式的测试代码，测试场景一：

```java
public void updateProduct(Product product) {
    TransactionTemplate template = getProductTransactionManager();
    template.setTimeout(3);
    template.execute(new TransactionCallback<Boolean>() {
        @Override
        public Boolean doInTransaction(final TransactionStatus status) {
            try {
                insertA(product);
                updateB(product);
                updateC(product);
                // sleep 5s，模拟事务超时
                Thread.sleep(5000L);
                return true;
            } catch (Exception e) {
                status.setRollbackOnly();
                return false;
            }
        }
    });
}
```

上来就想直接这样通过 `Thread.sleep(5000L);` 等待一段时间，模拟一下事务超时，想的结果一定是抛出了异常，直接进入 `catch` 语句块整个事务回滚了，没想到模拟执行了两次，居然没有抛出超时异常，事务正常提交了，有点怀疑了，感觉是自己设置错误了，或者是事务超时设置没有生效，但是查了半天依然没有什么头绪。

接着想调整一下代码试试，也许是代码编译有问题，就看到了如下代码：

```java
public void updateProduct(Product product) {
    TransactionTemplate template = getProductTransactionManager();
    template.setTimeout(3);
    template.execute(new TransactionCallback<Boolean>() {
        @Override
        public Boolean doInTransaction(final TransactionStatus status) {
            try {
                // sleep 5s，模拟事务超时
                Thread.sleep(5000L);
                insertA(product);
                updateB(product);
                updateC(product);
                return true;
            } catch (Exception e) {
                status.setRollbackOnly();
                return false;
            }
        }
    });
}
```

这次再一执行，不负众望，这次成功了，事务出现了超时，抛出了 `org.springframework.transaction.TransactionTimedOutException` 异常，这就有点不明白了，为什么代码只是调整了一下前后顺序，结果就完全不一样了。本来事务生效了就可以前面的问题继续查了，但是这个问题不明白心理还是一直痒痒，真是好奇心害死猪。

网上搜了一下，相关的知识内容不太多，但是很幸运很快搜到了一篇很有价值的文章，遇到的问题与前面情况非常类似：

[Spring事务超时时间可能存在的错误认识](https://blog.csdn.net/AlbertFly/article/details/51462081)

自己遇到情况可能和上面也不完全一样，所以又仔细分析了一下。

### Spring 源码阅读分析

通过代码分析我们可以知道 `TransactionTemplate` 类继承自 `DefaultTransactionDefinition`，`timeout` 是该类的一个属性，我们在配置的事务管理器是 `org.springframework.jdbc.datasource.DataSourceTransactionManager` 这个类，这个类是控制我们事务执行的核心类内容，在这个类的 doBegin 方法中，我们可以查到事务开始时对超时时间的设置：

```java
protected void doBegin(Object transaction, TransactionDefinition definition) {
    DataSourceTransactionObject txObject = (DataSourceTransactionObject) transaction;
    Connection con = null;
    if (txObject.getConnectionHolder() == null || txObject.getConnectionHolder().isSynchronizedWithTransaction()) {
        Connection newCon = this.dataSource.getConnection();
        if (logger.isDebugEnabled()) {
            logger.debug("Acquired Connection [" + newCon + "] for JDBC transaction");
        }
        txObject.setConnectionHolder(new ConnectionHolder(newCon), true);
    }

    txObject.getConnectionHolder().setTransactionActive(true);
    int timeout = determineTimeout(definition);
    if (timeout != TransactionDefinition.TIMEOUT_DEFAULT) {
        txObject.getConnectionHolder().setTimeoutInSeconds(timeout);
    }
}
```

其中 `determineTimeout` 用来获取我们设置的事务超时时间；然后设置到 `ConnectionHolder` 对象上（其是 `ResourceHolder`子类），接着看 `ResourceHolderSupport` 的 `setTimeoutInSeconds` 实现：

```java
/**
 * Set the timeout for this object in seconds.
 * @param seconds number of seconds until expiration
 */
public void setTimeoutInSeconds(int seconds) {
    setTimeoutInMillis(seconds * 1000);
}

/**
 * Set the timeout for this object in milliseconds.
 * @param millis number of milliseconds until expiration
 */
public void setTimeoutInMillis(long millis) {
    this.deadline = new Date(System.currentTimeMillis() + millis);
}
```

最终会在 `ResourceHolderSupport` 类的 `deadline` 属性上设置一个过期时间，而且是一个绝对时间参数。

```java
/**
 * Return the time to live for this object in seconds.
 * Rounds up eagerly, e.g. 9.00001 still to 10.
 * @return number of seconds until expiration
 * @throws TransactionTimedOutException if the deadline has already been reached
 */
public int getTimeToLiveInSeconds() {
    double diff = ((double) getTimeToLiveInMillis()) / 1000;
    int secs = (int) Math.ceil(diff);
    checkTransactionTimeout(secs <= 0);
    return secs;
}

/**
 * Return the time to live for this object in milliseconds.
 * @return number of millseconds until expiration
 * @throws TransactionTimedOutException if the deadline has already been reached
 */
public long getTimeToLiveInMillis() throws TransactionTimedOutException{
    if (this.deadline == null) {
        throw new IllegalStateException("No timeout specified for this resource holder");
    }
    long timeToLive = this.deadline.getTime() - System.currentTimeMillis();
    checkTransactionTimeout(timeToLive <= 0);
    return timeToLive;
}

/**
 * Set the transaction rollback-only if the deadline has been reached,
 * and throw a TransactionTimedOutException.
 */
private void checkTransactionTimeout(boolean deadlineReached) throws TransactionTimedOutException {
    if (deadlineReached) {
        setRollbackOnly();
        throw new TransactionTimedOutException("Transaction timed out: deadline was " + this.deadline);
    }
}
```

在 `ResourceHolderSupport` 类中的 `getTimeToLiveInSeconds` 和 `getTimeToLiveInMillis` 方法中，都会通过 `deadline` 参数检查事务是否已过期，如果过期直接抛出 `TransactionTimedOutException` 异常内容。

```java
/**
 * Apply the specified timeout - overridden by the current transaction timeout,
 * if any - to the given JDBC Statement object.
 * @param stmt the JDBC Statement object
 * @param dataSource the DataSource that the Connection was obtained from
 * @param timeout the timeout to apply (or 0 for no timeout outside of a transaction)
 * @throws SQLException if thrown by JDBC methods
 * @see java.sql.Statement#setQueryTimeout
 */
public static void applyTimeout(Statement stmt, DataSource dataSource, int timeout) throws SQLException {
    Assert.notNull(stmt, "No Statement specified");
    Assert.notNull(dataSource, "No DataSource specified");
    ConnectionHolder holder = (ConnectionHolder) TransactionSynchronizationManager.getResource(dataSource);
    if (holder != null && holder.hasTimeout()) {
        // Remaining transaction timeout overrides specified value.
        stmt.setQueryTimeout(holder.getTimeToLiveInSeconds());
    } else if (timeout >= 0) {
        // No current transaction timeout -> apply specified value.
        stmt.setQueryTimeout(timeout);
    }
}
```

`getTimeToLiveInSeconds` 方法被 `DataSourceUtils` 类的 `applyTransactionTimeout` 方法调用，其中这一行代码尤为重要：

```java
stmt.setQueryTimeout(holder.getTimeToLiveInSeconds());
```

在 `Statement` 的 `setQueryTimeout` 方法中就会调用 `getTimeToLiveInSeconds` 方法检查事务是否超时，最后再看 `applyTimeout` 方法的调用时机：

```java
/**
 * Prepare the given JDBC Statement (or PreparedStatement or CallableStatement),
 * applying statement settings such as fetch size, max rows, and query timeout.
 * @param stmt the JDBC Statement to prepare
 * @throws SQLException if thrown by JDBC API
 * @see #setFetchSize
 * @see #setMaxRows
 * @see #setQueryTimeout
 * @see org.springframework.jdbc.datasource.DataSourceUtils#applyTransactionTimeout
 */
protected void applyStatementSettings(Statement stmt) throws SQLException {
    int fetchSize = getFetchSize();
    if (fetchSize != -1) {
        stmt.setFetchSize(fetchSize);
    }
    int maxRows = getMaxRows();
    if (maxRows != -1) {
        stmt.setMaxRows(maxRows);
    }
    DataSourceUtils.applyTimeout(stmt, getDataSource(), getQueryTimeout());
}
```

在 `JdbcTemplate` 类中的 `applyStatementSettings` 方法中会调用 `DataSourceUtils#applyTimeout()` 方法，至此我们知道了在 `JdbcTemplate` 拿到 `Statement` 之后，执行之前会设置其 `queryTimeout`。

以上这些分析都是顺理成章的，但是这里还有一点疑问是，我们目前系统中使用的并不是 `JdbcTemplate` 在执行 `SQL` 语句，底层使用的 `ORM` 框架是 `MyBatis`。那么整个思路应该还是略有不同的。

### MyBatis 源码阅读分析

现在的系统中使用的 `MyBatis` 相关 jar 包的版本是：

> mybatis: 3.4.2
> mybatis-spring: 1.3.1

系统启动之后，我们将断点设置到了 `ResourceHolderSupport#getTimeToLiveInSeconds()` 方法上面，通过查看断点处上下文执行环境，分析代码执行情况：

`getTimeToLiveInSeconds` 方法被 `SpringManagedTransaction` 类中的 `getTimeout` 方法调用：

```java
@Override
public Integer getTimeout() throws SQLException {
  ConnectionHolder holder = (ConnectionHolder) TransactionSynchronizationManager.getResource(dataSource);
  if (holder != null && holder.hasTimeout()) {
    return holder.getTimeToLiveInSeconds();
  } 
  return null;
}
```

这里的 `getTimeout` 方法会在 `ReuseExecutor` 类中的 `prepareStatement` 方法中使用：

```java
private Statement prepareStatement(StatementHandler handler, Log statementLog) throws SQLException {
  Statement stmt;
  BoundSql boundSql = handler.getBoundSql();
  String sql = boundSql.getSql();
  if (hasStatementFor(sql)) {
    stmt = getStatement(sql);
    applyTransactionTimeout(stmt);
  } else {
    Connection connection = getConnection(statementLog);
    stmt = handler.prepare(connection, transaction.getTimeout());
    putStatement(sql, stmt);
  }
  handler.parameterize(stmt);
  return stmt;
}
```

其实代码跟到这里，基本上已经差不多了，`prepareStatement` 方法就是在为每条 `SQL` 语句的执行创建 `Statement` 对象，那么在每一次 `SQL` 执行的时候都会调用这个方法，同时也会判断事务的是否已经超时，如果已经超时，直接抛出异常，如果没有超时，则继续执行 `SQL` 语句。

最后再分析一下 `stmt = handler.prepare(connection, transaction.getTimeout());` 这行代码的执行逻辑，我们在 `BaseStatementHandler` 类中找到了具体的代码逻辑：

```java
@Override
public Statement prepare(Connection connection, Integer transactionTimeout) throws SQLException {
  ErrorContext.instance().sql(boundSql.getSql());
  Statement statement = null;
  try {
    statement = instantiateStatement(connection);
    setStatementTimeout(statement, transactionTimeout);
    setFetchSize(statement);
    return statement;
  } catch (SQLException e) {
    closeStatement(statement);
    throw e;
  } catch (Exception e) {
    closeStatement(statement);
    throw new ExecutorException("Error preparing statement.  Cause: " + e, e);
  }
}

protected void setStatementTimeout(Statement stmt, Integer transactionTimeout) throws SQLException {
  Integer queryTimeout = null;
  if (mappedStatement.getTimeout() != null) {
    queryTimeout = mappedStatement.getTimeout();
  } else if (configuration.getDefaultStatementTimeout() != null) {
    queryTimeout = configuration.getDefaultStatementTimeout();
  }
  if (queryTimeout != null) {
    stmt.setQueryTimeout(queryTimeout);
  }
  StatementUtil.applyTransactionTimeout(stmt, queryTimeout, transactionTimeout);
}
```

StatementUtil#applyTransactionTimeout：

```java
/**
 * Apply a transaction timeout.
 * <p>
 * Update a query timeout to apply a transaction timeout.
 * </p>
 * @param statement a target statement
 * @param queryTimeout a query timeout
 * @param transactionTimeout a transaction timeout
 * @throws SQLException if a database access error occurs, this method is called on a closed <code>Statement</code>
 */
public static void applyTransactionTimeout(Statement statement, Integer queryTimeout, Integer transactionTimeout) throws SQLException {
  if (transactionTimeout == null){
    return;
  }
  Integer timeToLiveOfQuery = null;
  if (queryTimeout == null || queryTimeout == 0) {
    timeToLiveOfQuery = transactionTimeout;
  } else if (transactionTimeout < queryTimeout) {
    timeToLiveOfQuery = transactionTimeout;
  }
  if (timeToLiveOfQuery != null) {
    statement.setQueryTimeout(timeToLiveOfQuery);
  }
}
```

在这里我们也看到了也在为 `Statement` 对象设置 `queryTimeout` 属性，当然会将我们设置的 `queryTimeout` 值和 `transactionTimeout` 的值进行比较，将比较后较小的值设置为最终结果值。

所以说事务超时的判断是在每一条待执行的 `SQL` 语句开启 `Statement` 时进行的，也是与该条 `SQL` 语句的 `queryTimeout` 属性值有直接关系的。当然如果最后一条 `SQL` 顺利执行完毕，没有任何超时，后面如果不再执行任何 `SQL` 语句执行，即使使用 `Thread.sleep(5000L);` 停顿一段时间，整个事务也是不会出现超时的。

### 分析总结

分析了使用 `JdbcTemplate` 和 `MyBatis` 框架的不同执行过程，虽然代码实现不同，但是最终的效果应该说是一致的。

在一个事务当中如果执行多条 `SQL`，每次执行创建 `Statement` 对象时都会检查是否已经出现超时，如果未超时，则会设置 `Statement` 的 `queryTimeout` 属性，继续执行 `SQL`，如果本次执行一切 OK，则在执行下一条 `SQL` 语句时会重复上面逻辑，如果所有 `SQL` 语句全部执行成功，即使在最后设置一个耗时操作，也不会出现事务超时的，耗时操作并不会计入事务的超时时间判断；当然，如果将耗时操作执行后，还有 `SQL` 语句需要执行，那么这个耗时操作的时间是会计入到事务的超时时间当中的。

在上面的博客文章中，作者总结了一个公式：

> Spring 事务超时 = 事务开始时到最后一个 Statement 创建时时间 + 最后一个 Statement 的执行时超时时间（即其queryTimeout）

不是特别理解作者要表达的意思，但是我认为这里表述不太准确，如果执行了多条 `SQL` 语句，很有可能在中间某条 `SQL` 就已经超时了，根本不会执行到所谓的最后一个 `Statement`，当然可能作者表述的另外一番意思。

### 参考资料

- [Spring事务超时时间可能存在的错误认识](https://blog.csdn.net/AlbertFly/article/details/51462081)