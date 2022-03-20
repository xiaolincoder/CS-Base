# 幻读是怎么被解决的？

大家好，我是小林。

我之前写过一篇数据库事务的文章「 [事务、事务隔离级别和MVCC](https://mp.weixin.qq.com/s/sCgIWj0HjMgUqVIHwLXduQ)」，这篇我说过什么是幻读。

![在这里插入图片描述](https://img-blog.csdnimg.cn/d1da346e962046609ade36637d9feb03.png?x-oss-process=image/watermark,type_ZHJvaWRzYW5zZmFsbGJhY2s,shadow_50,text_Q1NETiBA5bCP5p6XY29kaW5n,size_20,color_FFFFFF,t_70,g_se,x_16)



然后前几天有位读者跟我说，我这个幻读例子不是已经被「可重复读」隔离级别解决了吗？为什么还要有 next-key 呢？

他有这个质疑，是因为他做了这个实验。

实验的数据库表 t_stu 如下，其中 id 为主键。

![在这里插入图片描述](https://img-blog.csdnimg.cn/7f9df142b3594daeaaca495abb7133f5.png)
然后在可重复读隔离级别下，有两个事务的执行顺序如下：

![在这里插入图片描述](https://img-blog.csdnimg.cn/e576e047dccc47d5a59636ea342750b8.png?x-oss-process=image/watermark,type_ZHJvaWRzYW5zZmFsbGJhY2s,shadow_50,text_Q1NETiBA5bCP5p6XY29kaW5n,size_20,color_FFFFFF,t_70,g_se,x_16)



从这个实验结果可以看到，即使事务 B 中途插入了一条记录，事务 A 前后两次查询的结果集都是一样的，并没有出现所谓的幻读现象。

读者做的实验之所以看不到幻读现象，是因为在可重复读隔离级别下，**普通的查询是快照读，是不会看到别的事务插入的数据的**。

可重复读隔离级是由 MVCC（多版本并发控制）实现的，实现的方式是启动事务后，在执行第一个查询语句后，会创建一个视图，然后后续的查询语句都用这个视图，「快照读」读的就是这个视图的数据，视图你可以理解为版本数据，这样就使得每次查询的数据都是一样的。


MySQL 里除了普通查询是快照度，其他都是**当前读**，比如update、insert、delete，这些语句执行前都会查询最新版本的数据，然后再做进一步的操作。

这很好理解，假设你要 update 一个记录，另一个事务已经 delete 这条记录并且 提交事务了，这样不是会产生冲突吗，所以 update 的时候肯定要知道最新的数据。

另外，`select ... for update` 这种查询语句是当前读，每次执行的时候都是读取最新的数据。 

**因此，要讨论「可重复读」隔离级别的幻读现象，是要建立在「当前读」的情况下。**

接下来，我们假设`select ... for update`当前读是不会加锁的（实际上是会加锁的），在做一遍读者那个实验。

![](https://img-blog.csdnimg.cn/1f872ff92b644b5f81cee2dd9188b199.png?x-oss-process=image/watermark,type_ZHJvaWRzYW5zZmFsbGJhY2s,shadow_50,text_Q1NETiBA5bCP5p6XY29kaW5n,size_20,color_FFFFFF,t_70,g_se,x_16)


这时候，事务 B 插入的记录，就会被事务 A 的第二条查询语句查询到（因为是当前读），这样就会出现前后两次查询的结果集合不一样，这就出现了幻读。

所以，**Innodb 引擎为了解决「可重复读」隔离级别使用「当前读」而造成的幻读问题，就引出了 next-key 锁**，就是记录锁和间隙锁的组合。
- 记录锁，锁的是记录本身；
- 间隙锁，锁的就是两个值之间的空隙，以防止其他事务在这个空隙间插入新的数据，从而避免幻读现象。

比如，执行这条语句的时候，会锁住，然后期间如果有其他事务在这个锁住的范围插入数据就会被阻塞。

![](https://img-blog.csdnimg.cn/3af285a8e70f4d4198318057eb955520.png?x-oss-process=image/watermark,type_ZHJvaWRzYW5zZmFsbGJhY2s,shadow_50,text_Q1NETiBA5bCP5p6XY29kaW5n,size_20,color_FFFFFF,t_70,g_se,x_16)



next-key 锁的加锁规则其实挺复杂的，在一些场景下会退化成记录锁或间隙锁，我之前也写一篇加锁规则，详细可以看这篇「[我做了一天的实验！](https://mp.weixin.qq.com/s/i5QWx3QPZNkV51ghFbtXCw)」


需要注意的是，next-key lock 锁的是索引，而不是数据本身，所以如果 update 语句的 where 条件没有用到索引列，那么就会全表扫描，在一行行扫描的过程中，不仅给行加上了行锁，还给行两边的空隙也加上了间隙锁，相当于锁住整个表，然后直到事务结束才会释放锁。

所以在线上千万不要执行没有带索引的 update 语句，不然会造成业务停滞，我有个读者就因为干了这个事情，然后被老板教育了一波，详细可以看这篇「[完蛋，公司被一条 update 语句干趴了！](https://mp.weixin.qq.com/s/9R8ChusahrJvLGmUvHWBgA)」



---

好了，这次就聊到这啦，学到的点个赞呀！