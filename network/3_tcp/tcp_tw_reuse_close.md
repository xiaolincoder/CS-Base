# 4.14 tcp_tw_reuse 为什么默认是关闭的？

大家好，我是小林。

上周有个读者在面试微信的时候，**被问到既然打开 net.ipv4.tcp_tw_reuse 参数可以快速复用处于 TIME_WAIT 状态的 TCP 连接，那为什么 Linux 默认是关闭状态呢？**

![图片](https://img-blog.csdnimg.cn/img_convert/23f1aea82a0b7c37f1031524600626f1.png)

![图片](https://img-blog.csdnimg.cn/img_convert/076e60b984028bf3ad762eb2bd7ed0f3.png)

好家伙，真的问好细节！

当时看到读者这个问题的时候，我也是一脸懵逼的，经过我的一番思考后，终于知道怎么回答这题了。

其实这题在变相问「**如果 TIME_WAIT 状态持续时间过短或者没有，会有什么问题？**」

因为开启 tcp_tw_reuse 参数可以快速复用处于 TIME_WAIT 状态的 TCP 连接时，相当于缩短了 TIME_WAIT 状态的持续时间。

可能有的同学会问说，使用 tcp_tw_reuse 快速复用处于 TIME_WAIT 状态的 TCP 连接时，是需要保证  net.ipv4.tcp_timestamps 参数是开启的（默认是开启的），而 tcp_timestamps 参数可以避免旧连接的延迟报文，这不是解决了没有 TIME_WAIT 状态时的问题了吗？

是解决部分问题，但是不能完全解决，接下来，我跟大家聊聊这个问题。

![图片](https://img-blog.csdnimg.cn/img_convert/d17df1a39a750c33948062ecfc9a8d32.png)

## 什么是 TIME_WAIT 状态？

TCP 四次挥手过程，如下图：

![图片](https://img-blog.csdnimg.cn/img_convert/e973a17cb5b1092085ca1bbcd7083559.png)图片

- 客户端打算关闭连接，此时会发送一个 TCP 首部 `FIN` 标志位被置为 `1`的报文，也即 `FIN` 报文，之后客户端进入 `FIN_WAIT_1` 状态。
- 服务端收到该报文后，就向客户端发送 `ACK` 应答报文，接着服务端进入 `CLOSED_WAIT` 状态。
- 客户端收到服务端的 `ACK` 应答报文后，之后进入 `FIN_WAIT_2` 状态。
- 等待服务端处理完数据后，也向客户端发送 `FIN` 报文，之后服务端进入 `LAST_ACK` 状态。
- 客户端收到服务端的 `FIN` 报文后，回一个 `ACK` 应答报文，之后进入 `TIME_WAIT` 状态
- 服务器收到了 `ACK` 应答报文后，就进入了 `CLOSE` 状态，至此服务端已经完成连接的关闭。
- 客户端在经过 `2MSL` 一段时间后，自动进入 `CLOSE` 状态，至此客户端也完成连接的关闭。

你可以看到，两个方向都需要**一个 FIN 和一个 ACK**，因此通常被称为**四次挥手**。

这里一点需要注意是：**主动关闭连接的，才有 TIME_WAIT 状态。**

可以看到，TIME_WAIT 是「主动关闭方」断开连接时的最后一个状态，该状态会持续 ***2MSL(Maximum Segment Lifetime)\*** 时长，之后进入 CLOSED 状态。

MSL 指的是 TCP 协议中任何报文在网络上最大的生存时间，任何超过这个时间的数据都将被丢弃。虽然 RFC 793 规定 MSL 为 2 分钟，但是在实际实现的时候会有所不同，比如 Linux 默认为 30 秒，那么 2MSL 就是 60 秒。

MSL 是由网络层的 IP 包中的 TTL 来保证的，TTL 是 IP 头部的一个字段，用于设置一个数据报可经过的路由器的数量上限。报文每经过一次路由器的转发，IP 头部的 TTL 字段就会减 1，减到 0 时报文就被丢弃。

MSL 与 TTL 的区别：MSL 的单位是时间，而 TTL 是经过路由跳数。所以 **MSL 应该要大于等于 TTL 消耗为 0 的时间**，以确保报文已被自然消亡。

**TTL 的值一般是 64，Linux 将 MSL 设置为 30 秒，意味着 Linux 认为数据报文经过 64 个路由器的时间不会超过 30 秒，如果超过了，就认为报文已经消失在网络中了**。

## 为什么要设计 TIME_WAIT 状态？

设计 TIME_WAIT 状态，主要有两个原因：

- 防止历史连接中的数据，被后面相同四元组的连接错误的接收；
- 保证「被动关闭连接」的一方，能被正确的关闭；

#### 原因一：防止历史连接中的数据，被后面相同四元组的连接错误的接收

为了能更好的理解这个原因，我们先来了解序列号（SEQ）和初始序列号（ISN）。

- **序列号**，是 TCP 一个头部字段，标识了 TCP 发送端到 TCP 接收端的数据流的一个字节，因为 TCP 是面向字节流的可靠协议，为了保证消息的顺序性和可靠性，TCP 为每个传输方向上的每个字节都赋予了一个编号，以便于传输成功后确认、丢失后重传以及在接收端保证不会乱序。**序列号是一个 32 位的无符号数，因此在到达 4G 之后再循环回到 0**。
- **初始序列号**，在 TCP 建立连接的时候，客户端和服务端都会各自生成一个初始序列号，它是基于时钟生成的一个随机数，来保证每个连接都拥有不同的初始序列号。**初始化序列号可被视为一个 32 位的计数器，该计数器的数值每 4 微秒加 1，循环一次需要 4.55 小时**。

给大家抓了一个包，下图中的 Seq 就是序列号，其中红色框住的分别是客户端和服务端各自生成的初始序列号。

![图片](https://img-blog.csdnimg.cn/img_convert/b70ee2f17636deeb3930010b6dcdabb7.png)

通过前面我们知道，**序列号和初始化序列号并不是无限递增的，会发生回绕为初始值的情况，这意味着无法根据序列号来判断新老数据**。

假设 TIME-WAIT 没有等待时间或时间过短，被延迟的数据包抵达后会发生什么呢？

![图片](https://img-blog.csdnimg.cn/img_convert/f1ba45cdb7d772ccd12dc604dee26c91.png)



- 服务端在关闭连接之前发送的 `SEQ = 301` 报文，被网络延迟了。
- 接着，服务端以相同的四元组重新打开了新连接，前面被延迟的 `SEQ = 301` 这时抵达了客户端，而且该数据报文的序列号刚好在客户端接收窗口内，因此客户端会正常接收这个数据报文，但是这个数据报文是上一个连接残留下来的，这样就产生数据错乱等严重的问题。

为了防止历史连接中的数据，被后面相同四元组的连接错误的接收，因此 TCP 设计了 TIME_WAIT 状态，状态会持续 `2MSL` 时长，这个时间**足以让两个方向上的数据包都被丢弃，使得原来连接的数据包在网络中都自然消失，再出现的数据包一定都是新建立连接所产生的。**

#### 原因二：保证「被动关闭连接」的一方，能被正确的关闭

如果客户端（主动关闭方）最后一次 ACK 报文（第四次挥手）在网络中丢失了，那么按照 TCP 可靠性原则，服务端（被动关闭方）会重发 FIN 报文。

假设客户端没有 TIME_WAIT 状态，而是在发完最后一次回 ACK 报文就直接进入 CLOSED 状态，如果该  ACK 报文丢失了，服务端则重传的 FIN 报文，而这时客户端已经进入到关闭状态了，在收到服务端重传的 FIN 报文后，就会回 RST 报文。

![图片](https://img-blog.csdnimg.cn/img_convert/8016c9f9b875649a5ab8bdd245c34729.png)

服务端收到这个 RST 并将其解释为一个错误（Connection reset by peer），这对于一个可靠的协议来说不是一个优雅的终止方式。

为了防止这种情况出现，客户端必须等待足够长的时间，确保服务端能够收到 ACK，如果服务端没有收到 ACK，那么就会触发 TCP 重传机制，服务端会重新发送一个 FIN，这样一去一来刚好两个 MSL 的时间。

![TIME-WAIT 时间正常，确保了连接正常关闭](https://cdn.xiaolincoding.com/gh/xiaolincoder/ImageHost4/网络/TIME-WAIT连接正常关闭.drawio.png)

客户端在收到服务端重传的 FIN 报文时，TIME_WAIT 状态的等待时间，会重置回 2MSL。

## tcp_tw_reuse 是什么？

在 Linux  操作系统下，TIME_WAIT 状态的持续时间是 60 秒，这意味着这 60 秒内，客户端一直会占用着这个端口。要知道，端口资源也是有限的，一般可以开启的端口为 32768~61000，也可以通过如下参数设置指定范围：

```plain
 net.ipv4.ip_local_port_range
```

**如果客户端（主动关闭连接方）的 TIME_WAIT 状态过多**，占满了所有端口资源，那么就无法对「目的 IP+ 目的 PORT」都一样的服务器发起连接了，但是被使用的端口，还是可以继续对另外一个服务器发起连接的。具体可以看我这篇文章：[客户端的端口可以重复使用吗？](https://xiaolincoding.com/network/3_tcp/port.html#%E5%AE%A2%E6%88%B7%E7%AB%AF%E7%9A%84%E7%AB%AF%E5%8F%A3%E5%8F%AF%E4%BB%A5%E9%87%8D%E5%A4%8D%E4%BD%BF%E7%94%A8%E5%90%97)

因此，客户端（主动关闭连接方）都是和「目的 IP+ 目的 PORT」都一样的服务器建立连接的话，当客户端的 TIME_WAIT 状态连接过多的话，就会受端口资源限制，如果占满了所有端口资源，那么就无法再跟「目的 IP+ 目的 PORT」都一样的服务器建立连接了。

不过，即使是在这种场景下，只要连接的是不同的服务器，端口是可以重复使用的，所以客户端还是可以向其他服务器发起连接的，这是因为内核在定位一个连接的时候，是通过四元组（源 IP、源端口、目的 IP、目的端口）信息来定位的，并不会因为客户端的端口一样，而导致连接冲突。

好在，Linux 操作系统提供了两个可以系统参数来快速回收处于 TIME_WAIT 状态的连接，这两个参数都是默认关闭的：

- net.ipv4.tcp_tw_reuse，如果开启该选项的话，客户端（连接发起方）在调用 connect() 函数时，**如果内核选择到的端口，已经被相同四元组的连接占用的时候，就会判断该连接是否处于 TIME_WAIT 状态，如果该连接处于 TIME_WAIT 状态并且 TIME_WAIT 状态持续的时间超过了 1 秒，那么就会重用这个连接，然后就可以正常使用该端口了**。所以该选项只适用于连接发起方。
- net.ipv4.tcp_tw_recycle，如果开启该选项的话，允许处于 TIME_WAIT 状态的连接被快速回收，该参数在 **NAT 的网络下是不安全的**！详细见这篇文章介绍：[SYN 报文什么时候情况下会被丢弃？](https://xiaolincoding.com/network/3_tcp/syn_drop.html)

要使得上面这两个参数生效，有一个前提条件，就是要打开 TCP 时间戳，即 net.ipv4.tcp_timestamps=1（默认即为 1）。

开启了 tcp_timestamps 参数，TCP 头部就会使用时间戳选项，它有两个好处，**一个是便于精确计算 RTT，另一个是能防止序列号回绕（PAWS）**，我们先来介绍这个功能。

序列号是一个 32 位的无符号整型，上限值是 4GB，超过 4GB 后就需要将序列号回绕进行重用。这在以前网速慢的年代不会造成什么问题，但在一个速度足够快的网络中传输大量数据时，序列号的回绕时间就会变短。如果序列号回绕的时间极短，我们就会再次面临之前延迟的报文抵达后序列号依然有效的问题。

为了解决这个问题，就需要有 TCP 时间戳。

试看下面的示例，假设 TCP 的发送窗口是 1 GB，并且使用了时间戳选项，发送方会为每个 TCP 报文分配时间戳数值，我们假设每个报文时间加 1，然后使用这个连接传输一个 6GB 大小的数据流。

![图片](https://img-blog.csdnimg.cn/img_convert/bf004909d9e44c3bc740737ced6731a0.png)

32 位的序列号在时刻 D 和 E 之间回绕。假设在时刻 B 有一个报文丢失并被重传，又假设这个报文段在网络上绕了远路并在时刻 F 重新出现。如果 TCP 无法识别这个绕回的报文，那么数据完整性就会遭到破坏。

使用时间戳选项能够有效的防止上述问题，如果丢失的报文会在时刻 F 重新出现，由于它的时间戳为 2，小于最近的有效时间戳（5 或 6），因此防回绕序列号算法（PAWS）会将其丢弃。

防回绕序列号算法要求连接双方维护最近一次收到的数据包的时间戳（Recent TSval），每收到一个新数据包都会读取数据包中的时间戳值跟 Recent TSval 值做比较，**如果发现收到的数据包中时间戳不是递增的，则表示该数据包是过期的，就会直接丢弃这个数据包**。

## 为什么 tcp_tw_reuse  默认是关闭的？

通过前面这么多铺垫，终于可以说这个问题了。

开启 tcp_tw_reuse 会有什么风险呢？我觉得会有 2 个问题。

### 第一个问题

我们知道开启 tcp_tw_reuse 的同时，也需要开启 tcp_timestamps，意味着可以用时间戳的方式有效的判断回绕序列号的历史报文。

但是，在看我看了防回绕序列号函数的源码后，发现对于 **RST 报文的时间戳即使过期了，只要 RST 报文的序列号在对方的接收窗口内，也是能被接受的**。

下面 tcp_validate_incoming 函数就是验证接收到的 TCP 报文是否合格的函数，其中第一步就会进行 PAWS 检查，由 tcp_paws_discard 函数负责。

```c
static bool tcp_validate_incoming(struct sock *sk, struct sk_buff *skb, const struct tcphdr *th, int syn_inerr)
{
    struct tcp_sock *tp = tcp_sk(sk);

    /* RFC1323: H1. Apply PAWS check first. */
    if (tcp_fast_parse_options(sock_net(sk), skb, th, tp) &&
        tp->rx_opt.saw_tstamp &&
        tcp_paws_discard(sk, skb)) {
        if (!th->rst) {
            ....
            goto discard;
        }
        /* Reset is accepted even if it did not pass PAWS. */
    }
```

当 tcp_paws_discard 返回 true，就代表报文是一个历史报文，于是就要丢弃这个报文。但是在丢掉这个报文的时候，会先判断是不是 RST 报文，如果不是 RST 报文，才会将报文丢掉。也就是说，即使 RST 报文是一个历史报文，并不会被丢弃。

假设有这样的场景，如下图：

![](https://img-blog.csdnimg.cn/img_convert/0df2003d41ec0ef23844975a85cfb722.png)

过程如下：

- 客户端向一个还没有被服务端监听的端口发起了 HTTP 请求，接着服务端就会回 RST 报文给对方，很可惜的是 **RST 报文被网络阻塞了**。
- 由于客户端迟迟没有收到 TCP 第二次握手，于是重发了 SYN 包，与此同时服务端已经开启了服务，监听了对应的端口。于是接下来，客户端和服务端就进行了 TCP 三次握手、数据传输（HTTP 应答 - 响应）、四次挥手。
- 因为**客户端开启了 tcp_tw_reuse，于是快速复用 TIME_WAIT 状态的端口，又与服务端建立了一个与刚才相同的四元组的连接**。
- 接着，**前面被网络延迟 RST 报文这时抵达了客户端，而且 RST 报文的序列号在客户端的接收窗口内，由于防回绕序列号算法不会防止过期的 RST，所以 RST 报文会被客户端接受了，于是客户端的连接就断开了**。

上面这个场景就是开启 tcp_tw_reuse 风险，**因为快速复用 TIME_WAIT 状态的端口，导致新连接可能被回绕序列号的 RST 报文断开了，而如果不跳过 TIME_WAIT 状态，而是停留 2MSL 时长，那么这个 RST 报文就不会出现下一个新的连接**。

可能大家会有这样的疑问，为什么 PAWS 检查要放过过期的 RST 报文。我翻了 RFC 1323，里面有一句提到：

*It is recommended that RST segments NOT carry timestamps, and that RST segments be acceptable regardless of their timestamp.  Old duplicate RST segments should be exceedingly unlikely, and their cleanup function should take precedence over timestamps.*

大概的意思：*建议 RST 段不携带时间戳，并且无论其时间戳如何，RST 段都是可接受的。老的重复的 RST 段应该是极不可能的，并且它们的清除功能应优先于时间戳。*

RFC 1323 提到说收历史的 RST 报文是极不可能，之所以有这样的想法是因为 TIME_WAIT 状态持续的 2MSL 时间，足以让连接中的报文在网络中自然消失，所以认为按正常操作来说是不会发生的，因此认为清除连接优先于时间戳。

而我前面提到的案例，是因为开启了 tcp_tw_reuse 状态，跳过了 TIME_WAIT 状态，才发生的事情。

有同学会说，都经过一个 HTTP 请求了，延迟的 RST 报文竟然还会存活？

一个 HTTP 请求其实很快的，比如我下面这个抓包，只需要 0.2 秒就完成了，远小于 MSL，所以延迟的 RST 报文存活是有可能的。

![图片](https://img-blog.csdnimg.cn/img_convert/2ac40ca1757888b7154a2baa8bbb9885.png)

### 第二个问题

开启 tcp_tw_reuse 来快速复用 TIME_WAIT 状态的连接，如果第四次挥手的 ACK 报文丢失了，服务端会触发超时重传，重传第三次挥手报文，处于 syn_sent 状态的客户端收到服务端重传第三次挥手报文，则会回 RST 给服务端。如下图：

![](https://cdn.xiaolincoding.com/gh/xiaolincoder/network/tcp/tcp_tw_reuse第二个问题.drawio.png)

这时候有同学就问了，如果 TIME_WAIT 状态被快速复用后，刚好第四次挥手的 ACK 报文丢失了，那客户端复用 TIME_WAIT 状态后发送的 SYN 报文被处于 last_ack 状态的服务端收到了会发生什么呢？

处于 last_ack 状态的服务端收到了 SYN 报文后，会回复确认号与服务端上一次发送 ACK 报文一样的 ACK 报文，这个 ACK 报文称为 [Challenge ACK](https://xiaolincoding.com/network/3_tcp/challenge_ack.html)，并不是确认收到 SYN 报文。

处于 syn_sent 状态的客户端收到服务端的  [Challenge ACK](https://xiaolincoding.com/network/3_tcp/challenge_ack.html) 后，发现不是自己期望收到的确认号，于是就会回复 RST 报文，服务端收到后，就会断开连接。

## 总结

tcp_tw_reuse 的作用是让客户端快速复用处于 TIME_WAIT 状态的端口，相当于跳过了 TIME_WAIT 状态，这可能会出现这样的两个问题：

- 历史 RST 报文可能会终止后面相同四元组的连接，因为 PAWS 检查到即使 RST 是过期的，也不会丢弃。
- 如果第四次挥手的 ACK 报文丢失了，有可能被动关闭连接的一方不能被正常的关闭;

虽然 TIME_WAIT 状态持续的时间是有一点长，显得很不友好，但是它被设计来就是用来避免发生乱七八糟的事情。

《UNIX 网络编程》一书中却说道：**TIME_WAIT 是我们的朋友，它是有助于我们的，不要试图避免这个状态，而是应该弄清楚它**。

---

最新的图解文章都在公众号首发，别忘记关注哦！！如果你想加入百人技术交流群，扫码下方二维码回复「加群」。

![](https://cdn.xiaolincoding.com/gh/xiaolincoder/ImageHost3@main/%E5%85%B6%E4%BB%96/%E5%85%AC%E4%BC%97%E5%8F%B7%E4%BB%8B%E7%BB%8D.png)