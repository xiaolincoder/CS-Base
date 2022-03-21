# 小林 x 图解计算机基础

**在线阅读网站**：[https://xiaolincoding.com/](https://xiaolincoding.com/)

本站所有文章都是我[公众号：小林coding](https://mp.weixin.qq.com/s/02036z-FMOCLpZ_otwMwBg)的原创文章，内容包含图解计算机网络、操作系统、计算机组成、数据库，共 1000 张图 + 30 万字，破除晦涩难懂的计算机基础知识，让天下没有难懂的八股文！🚀

曾经我也苦恼于那些晦涩难弄的计算机基础知识，但在我啃了一本又一本的书，看了一个又一个的视频后，终于对这些“家伙”有了认识。我想着，这世界上肯定有一些朋友也跟我有一样的苦恼，为此下决心，用图解 + 通熟易懂的讲解来帮助大家理解，利用工作之余，坚持输出图解文章两年之久，这才有了今天的网站!


##  :books:  图解系列 PDF 下载

- [图解网络 + 图解系统 PDF 下载](https://mp.weixin.qq.com/s/02036z-FMOCLpZ_otwMwBg)

## :open_book:《图解网络》
- **介绍**:point_down:：
  - [图解系统介绍](/network/README.md)
- **网络基础篇** :point_down:
  - [TCP/IP 网络模型有哪几层？](/network/1_base/tcp_ip_model.md) 
  - [键入网址到网页显示，期间发生了什么？](/network/1_base/what_happen_url.md) 
  - [Linux 系统是如何收发网络包的？](/network/1_base/how_os_deal_network_package.md) 
- **HTTP 篇** :point_down:
  - [HTTP 常见面试题](/network/2_http/http_interview.md) 
  - [HTTP/1.1如何优化？](/network/2_http/http_optimize.md) 
  - [HTTPS RSA 握手解析](/network/2_http/https_rsa.md) 
  - [HTTPS ECDHE 握手解析](/network/2_http/https_ecdhe.md) 
  - [HTTPS 如何优化？](/network/2_http/https_optimize.md) 
  - [HTTP/2 牛逼在哪？](/network/2_http/http2.md) 
  - [HTTP/3 强势来袭](/network/2_http/http3.md) 
- **TCP 篇** :point_down:
  - [TCP 三次握手与四次挥手面试题](/network/3_tcp/tcp_interview.md) 
  - [TCP 重传、滑动窗口、流量控制、拥塞控制](/network/3_tcp/tcp_feature.md) 
  - [TCP 实战抓包分析](/network/3_tcp/tcp_tcpdump.md) 
  - [TCP 半连接队列和全连接队列](/network/3_tcp/tcp_queue.md) 
  - [如何优化 TCP?](/network/3_tcp/tcp_optimize.md) 
  - [如何理解是 TCP 面向字节流协议？](/network/3_tcp/tcp_stream.md) 
  - [为什么 TCP 每次建立连接时，初始化序列号都要不一样呢？](/network/3_tcp/isn_deff.md) 
  - [SYN 报文什么时候情况下会被丢弃？](/network/3_tcp/syn_drop.md) 
  - [四次挥手中收到乱序的 FIN 包会如何处理？](/network/3_tcp/out_of_order_fin.md) 
  - [在 TIME_WAIT 状态的 TCP 连接，收到 SYN 后会发生什么？](/network/3_tcp/time_wait_recv_syn.md) 
  - [TCP 连接，一端断电和进程崩溃有什么区别？](/network/3_tcp/tcp_down_and_crash.md) 
  - [拔掉网线后， 原本的 TCP 连接还存在吗？](/network/3_tcp/tcp_unplug_the_network_cable.md) 
  - [tcp_tw_reuse 为什么默认是关闭的？](/network/3_tcp/tcp_tw_reuse_close.md) 
  - [HTTPS 中 TLS 和 TCP 能同时握手吗？](/network/3_tcp/tcp_tls.md) 
  - [TCP Keepalive 和 HTTP Keep-Alive 是一个东西吗？](/network/3_tcp/tcp_http_keepalive.md) 
- **IP 篇** :point_down:
  - [IP 基础知识全家桶](/network/4_ip/ip_base.md) 	
  - [ping 的工作原理](/network/4_ip/ping.md) 	
- **学习心得** :point_down:
  - [计算机网络怎么学？](/network/5_learn/learn_network.md) 	
  - [画图经验分享](/network/5_learn/draw.md) 	

## :open_book:《图解系统》
- **介绍**:point_down:：
  - [图解系统介绍](/os/README.md)
- **硬件结构** :point_down:
  - [CPU 是如何执行程序的？](/os/1_hardware/how_cpu_run.md) 
  - [磁盘比内存慢几万倍？](/os/1_hardware/storage.md) 
  - [如何写出让 CPU 跑得更快的代码？](/os/1_hardware/how_to_make_cpu_run_faster.md) 
  - [CPU 缓存一致性](/os/1_hardware/cpu_mesi.md) 
  - [CPU 是如何执行任务的？](/os/1_hardware/how_cpu_deal_task.md) 
  - [什么是软中断？](/os/1_hardware/soft_interrupt.md) 
  - [为什么 0.1 + 0.2 不等于 0.3 ？](/os/1_hardware/float.md) 
- **操作系统结构** :point_down:
  - [Linux 内核 vs Windows 内核](/os/2_os_structure/linux_vs_windows.md) 
- **内存管理** :point_down:
  - [为什么要有虚拟内存？](/os/3_memory/vmem.md) 
- **进程管理** :point_down:
  - [进程、线程基础知识](/os/4_process/process_base.md) 
  - [进程间有哪些通信方式？](/os/4_process/process_commu.md) 
  - [多线程冲突了怎么办？](/os/4_process/multithread_sync.md) 
  - [怎么避免死锁？](/os/4_process/deadlock.md) 
  - [什么是悲观锁、乐观锁？](/os/4_process/pessim_and_optimi_lock.md) 
  - [一个进程最多可以创建多少个线程？](/os/4_process/create_thread_max.md) 
- **调度算法** :point_down:
  - [进程调度/页面置换/磁盘调度算法](/os/5_schedule/schedule.md)
- **文件系统** :point_down:
  - [文件系统全家桶](/os/6_file_system/file_system.md) 	
- **设备管理** :point_down:
  - [键盘敲入 A 字母时，操作系统期间发生了什么？](/os/7_device/device.md) 
- **网络系统** :point_down:
  - [什么是零拷贝？](/os/8_network_system/zero_copy.md) 
  - [I/O 多路复用：select/poll/epoll](/os/8_network_system/selete_poll_epoll.md) 
  - [高性能网络模式：Reactor 和 Proactor](/os/8_network_system/reactor.md) 
  - [什么是一致性哈希？](/os/8_network_system/hash.md) 
- **学习心得** :point_down:
  - [如何查看网络的性能指标？](/os/9_linux_cmd/linux_network.md) 	
  - [画图经验分享](/os/9_linux_cmd/pv_uv.md) 	
- **学习心得** :point_down:
  - [计算机网络怎么学？](/os/10_learn/learn_os.md) 	
  - [画图经验分享](/os/10_learn/draw.md) 

## :open_book:《图解MySQL》
- **介绍**:point_down:：
  - [图解MySQL介绍](/mysql/README.md)
- **索引篇** :point_down:
  - [从数据页的角度看 B+ 树](/mysql/index/page.md)
  - [为什么 MySQL 采用 B+ 树作为索引？](/mysql/index/why_index_chose_bpuls_tree.md)
  - [索引失效有哪些？](/mysql/index/index_lose.md)
  - [MySQL 使用 like “%x“，索引一定会失效吗？](/mysql/index/index_issue.md)
  - [count(\*) 和 count(1) 有什么区别？哪个性能最好？](/mysql/index/count.md)
- **事务篇** :point_down:
  - [事务隔离级别是怎么实现的？](/mysql/transaction/mvcc.md) 	
  - [幻读是怎么被解决的？](/mysql/transaction/phantom.md) 	
- **锁篇** :point_down:
  - [MySQL 有哪些锁？](/mysql/lock/mysql_lock.md) 	
  - [MySQL 是怎么加锁的？](/mysql/lock/how_to_lock.md) 	
  - [update 没加索引会锁全表](/mysql/lock/update_index.md) 	
  - [MySQL 死锁了，怎么办？](/mysql/lock/deadlock.md) 	
- **日志篇** :point_down:
  - :joy:  正在赶稿的路上。。。。。
- **内存篇** :point_down:
  - :joy: 正在赶稿的路上。。。。。	

##  :open_book: 《图解Redis》

- **介绍**:point_down:：
  - [图解Redis介绍](/redis/README.md)
- **数据结构篇** :point_down:
  - [Redis 数据结构](/redis/data_struct/data_struct.md)
- **持久化篇** :point_down:
  - [事务隔离级别是怎么实现的？](/redis/storage/aof.md) 	
  - [幻读是怎么被解决的？](/redis/storage/rdb.md) 	
- **集群篇** :point_down:
  - [什么是缓存雪崩、击穿、穿透？](/redis/cluster/cache_problem.md) 	
  - [主从复制是怎么实现的？](/redis/cluster/master_slave_replication.md) 	
  - :joy:  正在赶稿的路上。。。。。

## :muscle: 学习心得

- [计算机基础学习路线](/cs_learn/) ：计算机基础学习书籍 + 视频推荐，全面且清晰。
- [互联网校招心得](/reader_nb/) ：小林神仙读者们的校招和学习心得，值得学习。


## :thought_balloon: 公众号


最新的图解文章都在公众号首发，强烈推荐关注！如果你想加入百人技术交流群，扫码下方二维码回复「加群」。

![](https://cdn.jsdelivr.net/gh/xiaolincoder/ImageHost3@main/其他/公众号介绍.png)

 
