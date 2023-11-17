# 图解 MySQL 介绍

《图解 MySQL》目前还在连载更新中，大家不要催啦:joy: ，更新完会第一时间整理 PDF 的。

目前已经更新好的文章：

- **基础篇**:point_down:
  
   - [执行一条 SQL 查询语句，期间发生了什么？](/mysql/base/how_select.md)
   - [MySQL 一行记录是怎么存储的？](/mysql/base/row_format.md)
   
- **索引篇** :point_down:
  
   - [索引常见面试题](/mysql/index/index_interview.md)
   - [从数据页的角度看 B+ 树](/mysql/index/page.md)
   - [为什么 MySQL 采用 B+ 树作为索引？](/mysql/index/why_index_chose_bpuls_tree.md)
   - [MySQL 单表不要超过 2000W 行，靠谱吗？](/mysql/index/2000w.md)
   - [索引失效有哪些？](/mysql/index/index_lose.md)
   - [MySQL 使用 like“%x“，索引一定会失效吗？](/mysql/index/index_issue.md)
   - [count(\*) 和 count(1) 有什么区别？哪个性能最好？](/mysql/index/count.md)
   
- **事务篇** :point_down:
	- [事务隔离级别是怎么实现的？](/mysql/transaction/mvcc.md) 	
	- [MySQL 可重复读隔离级别，完全解决幻读了吗？](/mysql/transaction/phantom.md) 	
	
- **锁篇** :point_down:
	- [MySQL 有哪些锁？](/mysql/lock/mysql_lock.md) 	
	- [MySQL 是怎么加锁的？](/mysql/lock/how_to_lock.md) 	
	- [update 没加索引会锁全表？](/mysql/lock/update_index.md) 	
	- [MySQL 记录锁 + 间隙锁可以防止删除操作而导致的幻读吗？](/mysql/lock/lock_phantom.md) 	
	- [MySQL 死锁了，怎么办？](/mysql/lock/deadlock.md) 	
	- [字节面试：加了什么锁，导致死锁的？](/mysql/lock/show_lock.md)
	
- **日志篇** :point_down:
	
	- [undo log、redo log、binlog 有什么用？](/mysql/log/how_update.md) 	
	
- **内存篇** :point_down:
	
	- [揭开 Buffer_Pool 的面纱](/mysql/buffer_pool/buffer_pool.md) 	
	
	----
	
	最新的图解文章都在公众号首发，别忘记关注哦！！如果你想加入百人技术交流群，扫码下方二维码回复「加群」。
	
	![img](https://cdn.xiaolincoding.com/gh/xiaolincoder/ImageHost3@main/%E5%85%B6%E4%BB%96/%E5%85%AC%E4%BC%97%E5%8F%B7%E4%BB%8B%E7%BB%8D.png)