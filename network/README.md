
# 图解网络介绍

大家好，我是小林，是《图解网络》的作者，本站的内容都是整理于我[公众号](https://mp.weixin.qq.com/s/FYH1I8CRsuXDSybSGY_AFA)里的图解文章。

还没关注的朋友，可以微信搜索「**小林 coding**」，关注我的公众号，**后续最新版本的 PDF 会在我的公众号第一时间发布**，而且会有更多其他系列的图解文章，比如操作系统、计算机组成、数据库、算法等等。

简单介绍下《图解网络》，整个内容共有 **`20W` 字 + `500` 张图**，每一篇都自己手绘了很多图，目的也很简单，击破大家对于「八股文」的恐惧。

## 适合什么群体？

《图解网络》写的网络知识主要是**面向程序员**的，因为小林本身也是个程序员，所以涉及到的知识主要是关于程序员日常工作或者面试的网络知识。

非常适合有一点网络基础，但是又不怎么扎实，或者知识点串不起来的同学，说白**这本图解网络就是为了拯救半桶水的同学而出来的**。

因为小林写的图解网络就四个字，**通俗易懂**！

相信你在看这本图解网络的时候，你心里的感受会是：

- 「卧槽，原来是这样，大学老师教知识原来是这么理解」
- 「卧槽，我的网络知识串起来了」
- 「卧槽，我感觉面试稳了」
- 「卧槽，相见恨晚」

当然，也适合面试突击网络知识时拿来看。图解网络里的内容基本是面试常见的协议，比如 HTTP、HTTPS、TCP、UDP、IP 等等，也有很多面试常问的问题，比如：

- TCP 为什么三次握手？四次挥手？
- TCP 为什么要有 TIME_WAIT 状态？
- TCP 为什么是可靠传输协议，而 UDP 不是？
- 键入网址到网页显示，期间发生了什么？
- HTTPS 握手过程是怎样的？
- …….

不敢说 100 % 涵盖了面试的网络问题，但是至少 90% 是有的，而且内容的深度应对大厂也是绰绰有余，有非常多的读者跑来感激小林的图解网络，帮助他们拿到了国内很多一线大厂的 offer。

## 要怎么阅读？

很诚恳的告诉你，《图解网络》不是教科书，而是我写的图解网络文章的整合，所以肯定是没有教科书那么细致和全面，当然也就不会有很多废话，都是直击重点，不绕弯，而且有的知识点书上看不到。

阅读的顺序可以不用从头读到尾，你可以根据你想要了解的知识点，通过本站的搜索功能，去看哪个章节的内容就好，可以随意阅读任何章节。

本站的左侧边拦就是《图解网络》的目录结构（别看篇章不多，每一章都是很长很长的文章哦 :laughing:）：

- **网络基础篇** :point_down:
  - [TCP/IP 网络模型有哪几层？](/network/1_base/tcp_ip_model.md) 
  - [键入网址到网页显示，期间发生了什么？](/network/1_base/what_happen_url.md) 
  - [Linux 系统是如何收发网络包的？](/network/1_base/how_os_deal_network_package.md) 
- **HTTP 篇** :point_down:
	- [HTTP 常见面试题](/network/2_http/http_interview.md) 
	- [HTTP/1.1 如何优化？](/network/2_http/http_optimize.md) 
	- [HTTPS RSA 握手解析](/network/2_http/https_rsa.md) 
	- [HTTPS ECDHE 握手解析](/network/2_http/https_ecdhe.md) 
	- [HTTPS 如何优化？](/network/2_http/https_optimize.md) 
	- [HTTP/2 牛逼在哪？](/network/2_http/http2.md) 
	- [HTTP/3 强势来袭](/network/2_http/http3.md) 
	- [既然有 HTTP 协议，为什么还要有 RPC？](/network/2_http/http_rpc.md) 
	- [既然有 HTTP 协议，为什么还要有 websocket？](/network/2_http/http_websocket.md) 
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
	- [拔掉网线后，原本的 TCP 连接还存在吗？](/network/3_tcp/tcp_unplug_the_network_cable.md) 
	- [tcp_tw_reuse 为什么默认是关闭的？](/network/3_tcp/tcp_tw_reuse_close.md) 
	- [HTTPS 中 TLS 和 TCP 能同时握手吗？](/network/3_tcp/tcp_tls.md) 
	- [TCP Keepalive 和 HTTP Keep-Alive 是一个东西吗？](/network/3_tcp/tcp_http_keepalive.md) 
	- [TCP 有什么缺陷？](/network/3_tcp/tcp_problem.md)
	- [如何基于 UDP 协议实现可靠传输？](/network/3_tcp/quic.md)
	- [TCP 和 UDP 可以使用同一个端口吗？](/network/3_tcp/port.md)
	- [服务端没有 listen，客户端发起连接建立，会发生什么？](/network/3_tcp/tcp_no_listen.md)
	- [没有 accept，可以建立 TCP 连接吗？](/network/3_tcp/tcp_no_accpet.md)
	- [用了 TCP 协议，数据一定不会丢吗？](/network/3_tcp/tcp_drop.md)
	- [TCP 四次挥手，可以变成三次吗？](/network/3_tcp/tcp_three_fin.md)
- **IP 篇** :point_down:
	- [IP 基础知识全家桶](/network/4_ip/ip_base.md) 	
	- [ping 的工作原理](/network/4_ip/ping.md) 	
	- [断网了，还能 ping 通 127.0.0.1 吗？](/network/4_ip/ping_lo.md)
- **学习心得** :point_down:
	- [计算机网络怎么学？](/network/5_learn/learn_network.md) 	
  - [画图经验分享](/network/5_learn/draw.md) 	

## 质量如何？

图解网络的质量小林说的不算，读者说的算！

图解网络的第一个版本自去年发布以来，每隔一段时间，就会有不少的读者跑来感激小林。

他们说看了我的图解网络，轻松应对大厂的网络面试题，而且每次面试时问到网络问题，他们一点都不慌，甚至暗暗窃喜。

![在这里插入图片描述](https://img-blog.csdnimg.cn/160f55b965cf4c42ba160e327178a783.png)

## 有错误怎么办？

小林是个手残党，时常写出错别字。

如果你在学习的过程中，**如果你发现有任何错误或者疑惑的地方，欢迎你通过邮箱或者底部留言给小林**，勘误邮箱：xiaolincoding@163.com

小林抽时间会逐个修正，然后发布新版本的图解网络 PDF，一起迭代出更好的图解网络！

新的图解文章都在公众号首发，别忘记关注了哦！如果你想加入百人技术交流群，扫码下方二维码回复「加群」。

![](https://cdn.xiaolincoding.com/gh/xiaolincoder/ImageHost3@main/其他/公众号介绍.png)

