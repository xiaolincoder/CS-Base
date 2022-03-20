# 图解系统介绍

大家好，我是小林，是《图解系统》的作者，本站的内容都是整理于我[公众号](https://mp.weixin.qq.com/s/02036z-FMOCLpZ_otwMwBg)里的图解文章。

还没关注的朋友，可以微信搜索「**小林coding**」，关注我的公众号，**后续最新版本的 PDF 会在我的公众号第一时间发布**，而且会有更多其他系列的图解文章，比如操作系统、计算机组成、数据库、算法等等。

简单介绍下这个《图解系统》，整个内容共有 **`15W` 字 + `400` 张图**，文字都是小林一个字一个字敲出来的，图片都是小林一个点一条线画出来的，非常的不容易。

## 适合什么群体？

《图解系统》主要是**面向程序员**的，因为小林本身也是个程序员，所以涉及到的知识主要是关于程序员日常工作或者面试的操作系统知识。

非常适合有一点操作系统，但是又不怎么扎实，或者知识点串不起来的同学，说白**这本图解系统就是为了拯救半桶水的同学而出来**。因为小林写的图解系统就四个字，**通俗易懂**！

《图解系统》不仅仅涉及了操作系统的内容，还涉及一些计算机组成和 Linux 命令的内容。

当然还是操作系统的内容占比较高，基本把操作系统进程管理、内存管理、文件系统、设备管理、网络系统这五大结构图解了。

计算机组成主要涉及是 CPU 方面的知识，我们不关注 CPU 是怎么设计与实现的，只关注跟我们开发者有关系的 CPU 知识。

当然，也适合面试突击操作系统知识时拿来看，不敢说 100 % 涵盖了面试的网络问题，但是至少 90% 是有的，而且内容的深度应对大厂也是搓搓有余的，有非常多的读者跑来感激小林的图解系统，**帮助他们拿到了国内很多一线大厂的 offer**。

## 要怎么阅读？

很诚恳的告诉你，《图解系统》不是教科书。而是我在公众号里写的图解系统文章的整合，所以肯定是没有教科书那么细致和全面，当然也不就不会有很多废话，都是直击重点，不绕弯，而且有的知识点书上看不到。

阅读的顺序可以不用从头读到尾，你可以根据你想要了解的知识点，通过本站的搜索功能，去看哪个章节的文章就好，可以随意阅读任何章节。

本站的左侧边拦就是《图解系统》的目录结构（别看篇章不多，每一章都是很长很长的文章哦 :laughing:）：

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
  - [进程、线程基础知识](/os/4_process/tcp_feature.md) 
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



## 有错误怎么办？

小林是个**手残党**，时常写出错别字，如果你在学习的过程中，**如果你发现有任何错误或者疑惑的地方，欢迎你通过下方的邮箱或者底部留言给小林**。

勘误邮箱：xiaolincoding@163.com

小林抽时间会逐个修正，然后发布新版本的图解系统 PDF，一起迭代出更好的图解系统！
