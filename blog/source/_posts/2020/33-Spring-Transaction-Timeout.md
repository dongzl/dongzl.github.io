---
title: Spring 事务管理 Timeout 的一点问题研究
date: 2020-08-04 20:23:45
cover: https://gitee.com/dongzl/article-images/raw/master/cover/reading_book.png
# author information, multiple authors are set to array
# single author
author:
  - nick: 董宗磊
    link: https://www.github.com/dongzl

# post subtitle in your index page
subtitle: 本文《多看电子书规范扩展开放计划》原文内容转载。

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