---
title: mysql
date: 2019-01-01 20:06:35
categories:
tags:
---


锁：先提一下存储引擎

共享锁   排他锁 独占锁  自旋锁  间隙锁 意向锁

表锁  页锁  行锁

innodb 每次读取数据 16K

写日志  用户态 内核态   0 拷贝

innodb_flush_log_at_trx_commit 0-1-2 默认 1  最安全 工作中设置为2

show variables like 'innodb_flush_log_at_trx_commit';

锁：如果表没有主键，是表锁；有主键是行锁

for update; 

lock in shard mode;

mysql  处理死锁：释放 undo 量比较小的  超时等待


wait for graphy   图的深度遍历

行锁 间隙锁  next key  

唯一索引查询  间隙锁 退化为 行锁


count(*)  count(1)   count(id)  新版本没有区别

阿里巴巴 变形虫项目





关于 Obejct o = new Object();

1、请解释一下对象的创建过程？（版初始化）
2、加问DCL与valatile问题？（指令重排）
3、对象在内存中的存储布局？（对象与数组的存储不同）
4、对象头具体包括什么？（markword klasspointer）
   synchronized
5、对象怎么定位？（直接 间接）
6、对象怎么分配？（栈上-线程本地-Eden-Old）
7、Obejct o = new Object() 在内存中占用多少字节？


new -> 偏向锁 -> 自旋锁 无锁 lockfree 轻量级锁 -> 重量级锁

https://blog.csdn.net/itcats_cn/article/details/80951929

句柄方式 经历两次引用找对象，查找效率比较高，  对象在java 堆内存中一直移动，垃圾回收  方便垃圾回收

方法区   1.7 > 永久代       1.8 < metaspace

https://blog.csdn.net/sinolover/article/details/94283857

https://segmentfault.com/a/1190000004606059

https://www.cnblogs.com/BlueStarWei/p/9358757.html

https://blog.csdn.net/weixin_45127309/article/details/93601578

UseCompressedOops  UseCompressedClassPointer

锁信息  对象信息 hashcode   对象头包含内容


1、jvm虚拟机
2、多线程 高并发
3、设计模式
4、redis
5、zk
6、mysql调优
7、spring 源码
8、netty

1、网约车项目
2、亿级流量多级缓存设计

<img src="https://gitee.com/dongzl/article-images/raw/master/2020/红黑树.png">





