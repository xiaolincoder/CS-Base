# 4.8 SYN 报文什么时候情况下会被丢弃？

大家好，我是小林。

之前有个读者在秋招面试的时候，被问了这么一个问题：SYN 报文什么时候情况下会被丢弃？

![](https://img-blog.csdnimg.cn/img_convert/d4df0c85e08f66f6a2aa2038af73adcc.png)

好家伙，现在面试都问那么细节了吗？

不过话说回来，这个问题跟工作上也是有关系的，因为我就在工作中碰到这么奇怪的时候，客户端向服务端发起了连接，但是连接并没有建立起来，通过抓包分析发现，服务端是收到 SYN 报文了，但是并没有回复 SYN+ACK（TCP 第二次握手），说明 SYN 报文被服务端忽略了，然后客户端就一直在超时重传 SYN 报文，直到达到最大的重传次数。

接下来，我就给出我遇到过 SYN 报文被丢弃的两种场景：

- 开启 tcp_tw_recycle 参数，并且在 NAT 环境下，造成 SYN 报文被丢弃

- TCP 两个队列满了（半连接队列和全连接队列），造成 SYN 报文被丢弃

## 坑爹的 tcp_tw_recycle

TCP 四次挥手过程中，主动断开连接方会有一个 TIME_WAIT 的状态，这个状态会持续 2 MSL 后才会转变为 CLOSED 状态。

![](https://img-blog.csdnimg.cn/img_convert/bee0c8e8d84047e7434803fb340f9e5d.png)

在 Linux  操作系统下，TIME_WAIT 状态的持续时间是 60 秒，这意味着这 60 秒内，客户端一直会占用着这个端口。要知道，端口资源也是有限的，一般可以开启的端口为 32768~61000，也可以通过如下参数设置指定范围：

```plain
 net.ipv4.ip_local_port_range
```

**如果客户端（发起连接方）的 TIME_WAIT 状态过多**，占满了所有端口资源，那么就无法对「目的 IP+ 目的 PORT」都一样的服务器发起连接了，但是被使用的端口，还是可以继续对另外一个服务器发起连接的。具体可以看我这篇文章：[客户端的端口可以重复使用吗？](https://xiaolincoding.com/network/3_tcp/port.html#%E5%AE%A2%E6%88%B7%E7%AB%AF%E7%9A%84%E7%AB%AF%E5%8F%A3%E5%8F%AF%E4%BB%A5%E9%87%8D%E5%A4%8D%E4%BD%BF%E7%94%A8%E5%90%97)

因此，客户端（发起连接方）都是和「目的 IP+ 目的 PORT」都一样的服务器建立连接的话，当客户端的 TIME_WAIT 状态连接过多的话，就会受端口资源限制，如果占满了所有端口资源，那么就无法再跟「目的 IP+ 目的 PORT」都一样的服务器建立连接了。

不过，即使是在这种场景下，只要连接的是不同的服务器，端口是可以重复使用的，所以客户端还是可以向其他服务器发起连接的，这是因为内核在定位一个连接的时候，是通过四元组（源 IP、源端口、目的 IP、目的端口）信息来定位的，并不会因为客户端的端口一样，而导致连接冲突。

但是 TIME_WAIT 状态也不是摆设作用，它的作用有两个：

- 防止具有相同四元组的旧数据包被收到，也就是防止历史连接中的数据，被后面的连接接受，否则就会导致后面的连接收到一个无效的数据，
- 保证「被动关闭连接」的一方能被正确的关闭，即保证最后的 ACK 能让被动关闭方接收，从而帮助其正常关闭;

不过，Linux 操作系统提供了两个可以系统参数来快速回收处于 TIME_WAIT 状态的连接，这两个参数都是默认关闭的：

- net.ipv4.tcp_tw_reuse，如果开启该选项的话，客户端（连接发起方）在调用 connect() 函数时，**如果内核选择到的端口，已经被相同四元组的连接占用的时候，就会判断该连接是否处于 TIME_WAIT 状态，如果该连接处于 TIME_WAIT 状态并且 TIME_WAIT 状态持续的时间超过了 1 秒，那么就会重用这个连接，然后就可以正常使用该端口了。**所以该选项只适用于连接发起方。
- net.ipv4.tcp_tw_recycle，如果开启该选项的话，允许处于 TIME_WAIT 状态的连接被快速回收；

要使得这两个选项生效，有一个前提条件，就是要打开 TCP 时间戳，即 net.ipv4.tcp_timestamps=1（默认即为 1)）。

**tcp_tw_recycle 在使用了 NAT 的网络下是不安全的！**

对于服务器来说，如果同时开启了 recycle 和 timestamps 选项，则会开启一种称之为「per-host 的 PAWS 机制」。

> 首先给大家说说什么是  PAWS 机制？

tcp_timestamps 选项开启之后，PAWS 机制会自动开启，它的作用是防止 TCP 包中的序列号发生绕回。

正常来说每个 TCP 包都会有自己唯一的 SEQ，出现 TCP 数据包重传的时候会复用 SEQ 号，这样接收方能通过 SEQ 号来判断数据包的唯一性，也能在重复收到某个数据包的时候判断数据是不是重传的。**但是 TCP 这个 SEQ 号是有限的，一共 32 bit，SEQ 开始是递增，溢出之后从 0 开始再次依次递增**。

所以当 SEQ 号出现溢出后单纯通过 SEQ 号无法标识数据包的唯一性，某个数据包延迟或因重发而延迟时可能导致连接传递的数据被破坏，比如：

![](https://img-blog.csdnimg.cn/img_convert/f5fbe947240026cc2f076267cb698496.png)

上图 A 数据包出现了重传，并在 SEQ 号耗尽再次从 A 递增时，第一次发的 A 数据包延迟到达了 Server，这种情况下如果没有别的机制来保证，Server 会认为延迟到达的 A 数据包是正确的而接收，反而是将正常的第三次发的 SEQ 为 A 的数据包丢弃，造成数据传输错误。

PAWS 就是为了避免这个问题而产生的，在开启 tcp_timestamps 选项情况下，一台机器发的所有 TCP 包都会带上发送时的时间戳，PAWS 要求连接双方维护最近一次收到的数据包的时间戳（Recent TSval），每收到一个新数据包都会读取数据包中的时间戳值跟 Recent TSval 值做比较，**如果发现收到的数据包中时间戳不是递增的，则表示该数据包是过期的，就会直接丢弃这个数据包**。

对于上面图中的例子有了 PAWS 机制就能做到在收到 Delay 到达的 A 号数据包时，识别出它是个过期的数据包而将其丢掉。

> 那什么是 per-host 的 PAWS 机制呢？

前面我提到，开启了 recycle 和 timestamps 选项，就会开启一种叫 per-host 的 PAWS 机制。**per-host 是对「对端 IP 做 PAWS 检查」**，而非对「IP + 端口」四元组做 PAWS 检查。

但是如果客户端网络环境是用了 NAT 网关，那么客户端环境的每一台机器通过 NAT 网关后，都会是相同的 IP 地址，在服务端看来，就好像只是在跟一个客户端打交道一样，无法区分出来。

Per-host PAWS 机制利用 TCP option 里的 timestamp 字段的增长来判断串扰数据，而 timestamp 是根据客户端各自的 CPU tick 得出的值。

当客户端 A 通过 NAT 网关和服务器建立 TCP 连接，然后服务器主动关闭并且快速回收 TIME-WAIT 状态的连接后，**客户端 B 也通过 NAT 网关和服务器建立 TCP 连接，注意客户端 A  和 客户端 B 因为经过相同的 NAT 网关，所以是用相同的 IP 地址与服务端建立 TCP 连接，如果客户端 B 的 timestamp 比 客户端 A 的 timestamp 小，那么由于服务端的 per-host 的 PAWS 机制的作用，服务端就会丢弃客户端主机 B 发来的 SYN 包**。

因此，tcp_tw_recycle 在使用了 NAT 的网络下是存在问题的，如果它是对 TCP 四元组做 PAWS 检查，而不是对「相同的 IP 做 PAWS 检查」，那么就不会存在这个问题了。

网上很多博客都说开启 tcp_tw_recycle 参数来优化 TCP，我信你个鬼，糟老头坏的很！

tcp_tw_recycle 在 Linux 4.12 版本后，直接取消了这一参数。

## accpet 队列满了

在 TCP 三次握手的时候，Linux 内核会维护两个队列，分别是：

- 半连接队列，也称 SYN 队列；
- 全连接队列，也称 accepet 队列；

服务端收到客户端发起的 SYN 请求后，**内核会把该连接存储到半连接队列**，并向客户端响应 SYN+ACK，接着客户端会返回 ACK，服务端收到第三次握手的 ACK 后，**内核会把连接从半连接队列移除，然后创建新的完全的连接，并将其添加到 accept 队列，等待进程调用 accept 函数时把连接取出来。**

![](https://img-blog.csdnimg.cn/img_convert/c9959166180b0e239bb48234ff7c2f5b.png)



### 半连接队列满了

当服务器造成 syn 攻击，就有可能导致 **TCP 半连接队列满了，这时后面来的 syn 包都会被丢弃**。

但是，**如果开启了 syncookies 功能，即使半连接队列满了，也不会丢弃 syn 包**。

syncookies 是这么做的：服务器根据当前状态计算出一个值，放在己方发出的 SYN+ACK 报文中发出，当客户端返回 ACK 报文时，取出该值验证，如果合法，就认为连接建立成功，如下图所示。

![](https://img-blog.csdnimg.cn/img_convert/58e01036d1febd0103dd0ec4d5acff05.png)

syncookies 参数主要有以下三个值：

- 0 值，表示关闭该功能；
- 1 值，表示仅当 SYN 半连接队列放不下时，再启用它；
- 2 值，表示无条件开启功能；

那么在应对 SYN 攻击时，只需要设置为 1 即可：


![](https://img-blog.csdnimg.cn/img_convert/e795b4ff5be76c85814ee190b4921f25.png)

这里给出几种防御 SYN 攻击的方法：

- 增大半连接队列；
- 开启 tcp_syncookies 功能
- 减少 SYN+ACK 重传次数

*方式一：增大半连接队列*

**要想增大半连接队列，我们得知不能只单纯增大 tcp_max_syn_backlog 的值，还需一同增大 somaxconn 和 backlog，也就是增大全连接队列**。否则，只单纯增大 tcp_max_syn_backlog 是无效的。

增大 tcp_max_syn_backlog 和 somaxconn 的方法是修改 Linux 内核参数：

![](https://img-blog.csdnimg.cn/img_convert/29f1fd2894162e15cbac938a2373b543.png)

增大 backlog 的方式，每个 Web 服务都不同，比如 Nginx 增大 backlog 的方法如下：

![](https://img-blog.csdnimg.cn/img_convert/a6b11fbd1fcb742cdcc87447fc23b73f.png)

最后，改变了如上这些参数后，要重启 Nginx 服务，因为半连接队列和全连接队列都是在 listen() 初始化的。

*方式二：开启 tcp_syncookies 功能*

开启 tcp_syncookies 功能的方式也很简单，修改 Linux 内核参数：

![](https://img-blog.csdnimg.cn/img_convert/54b7411607978cb9ff36d88cf47eb5c4.png)

*方式三：减少 SYN+ACK 重传次数*

当服务端受到 SYN 攻击时，就会有大量处于 SYN_RECV 状态的 TCP 连接，处于这个状态的 TCP 会重传 SYN+ACK，当重传超过次数达到上限后，就会断开连接。

那么针对 SYN 攻击的场景，我们可以减少 SYN+ACK 的重传次数，以加快处于 SYN_RECV 状态的 TCP 连接断开。

![](https://img-blog.csdnimg.cn/img_convert/19443a03430368b72c201113150471c5.png)

### 全连接队列满了

**在服务端并发处理大量请求时，如果 TCP accpet 队列过小，或者应用程序调用 accept() 不及时，就会造成 accpet 队列满了，这时后续的连接就会被丢弃，这样就会出现服务端请求数量上不去的现象。**

![](https://img-blog.csdnimg.cn/img_convert/d1538f8d3b50da26039bc6b171a13ad1.png)

我们可以通过 ss 命令来看 accpet 队列大小，在「LISTEN 状态」时，`Recv-Q/Send-Q` 表示的含义如下：

![](https://img-blog.csdnimg.cn/img_convert/d7e8fcbb4afa583687b76064b7f1afac.png)


- Recv-Q：当前 accpet 队列的大小，也就是当前已完成三次握手并等待服务端 `accept()` 的 TCP 连接个数；
- Send-Q：当前 accpet 最大队列长度，上面的输出结果说明监听 8088 端口的 TCP 服务进程，accpet 队列的最大长度为 128；

如果 Recv-Q 的大小超过 Send-Q，就说明发生了 accpet 队列满的情况。

要解决这个问题，我们可以：

- 调大 accpet 队列的最大长度，调大的方式是通过**调大 backlog 以及 somaxconn 参数。**
- 检查系统或者代码为什么调用 accept()  不及时；

关于 SYN 队列和 accpet 队列，我之前写过一篇很详细的文章：[TCP 半连接队列和全连接队列满了会发生什么？又该如何应对？](https://mp.weixin.qq.com/s/2qN0ulyBtO2I67NB_RnJbg)

---

好了，今天就分享到这里啦。

最新的图解文章都在公众号首发，别忘记关注哦！！如果你想加入百人技术交流群，扫码下方二维码回复「加群」。

![](https://cdn.xiaolincoding.com/gh/xiaolincoder/ImageHost3@main/%E5%85%B6%E4%BB%96/%E5%85%AC%E4%BC%97%E5%8F%B7%E4%BB%8B%E7%BB%8D.png)

