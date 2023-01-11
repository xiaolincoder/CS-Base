# 4.9 已建立连接的TCP，收到SYN会发生什么？

大家好，我是小林。

昨晚有位读者问了我这么个问题：

![](https://img-blog.csdnimg.cn/ea1c6e0165f04232ab02046132e63d0f.jpg)


大概意思是，一个已经建立的 TCP 连接，客户端中途宕机了，而服务端此时也没有数据要发送，一直处于 establish 状态，客户端恢复后，向服务端建立连接，此时服务端会怎么处理？

看过我的图解网络的读者都知道，TCP 连接是由「四元组」唯一确认的。

然后这个场景中，客户端的 IP、服务端 IP、目的端口并没有变化，所以这个问题关键要看客户端发送的 SYN 报文中的源端口是否和上一次连接的源端口相同。

**1. 客户端的 SYN 报文里的端口号与历史连接不相同**

如果客户端恢复后发送的 SYN 报文中的源端口号跟上一次连接的源端口号不一样，此时服务端会认为是新的连接要建立，于是就会通过三次握手来建立新的连接。

那旧连接里处于 Established 状态的服务端最后会怎么样呢？

如果服务端发送了数据包给客户端，由于客户端的连接已经被关闭了，此时客户的内核就会回 RST 报文，服务端收到后就会释放连接。

如果服务端一直没有发送数据包给客户端，在超过一段时间后，TCP 保活机制就会启动，检测到客户端没有存活后，接着服务端就会释放掉该连接。

**2. 客户端的 SYN 报文里的端口号与历史连接相同**

如果客户端恢复后，发送的 SYN 报文中的源端口号跟上一次连接的源端口号一样，也就是处于 Established 状态的服务端收到了这个 SYN 报文。

大家觉得服务端此时会做什么处理呢？
- 丢掉 SYN 报文？
- 回复 RST 报文？
- 回复 ACK 报文？

刚开始我看到这个问题的时候，也是没有思路的，因为之前没关注过，然后这个问题不能靠猜，所以我就看了 RFC 规范和看了 Linux 内核源码，最终知道了答案。

我不卖关子，先直接说答案。

![](https://cdn.xiaolincoding.com/gh/xiaolincoder/ImageHost4@main/网络/est_syn.png)


**处于 Established 状态的服务端如果收到了客户端的 SYN 报文（注意此时的 SYN 报文其实是乱序的，因为 SYN 报文的初始化序列号其实是一个随机数），会回复一个携带了正确序列号和确认号的 ACK 报文，这个 ACK 被称之为 Challenge ACK。**

**接着，客户端收到这个 Challenge ACK，发现序列号并不是自己期望收到的，于是就会回 RST 报文，服务端收到后，就会释放掉该连接。**

## RFC 文档解释

RFC 793 文档里的第 34 页里，有说到这个例子。

![](https://img-blog.csdnimg.cn/873ad18443c040708c415bab6592ae41.png)

原文的解释我也贴出来给大家看看。

- When the SYN arrives at line 3, TCP B, being in a synchronized state,
and the incoming segment outside the window, responds with an acknowledgment indicating what sequence it next expects to hear (ACK
100).
- TCP A sees that this segment does not acknowledge anything it
sent and, being unsynchronized, sends a reset (RST) because it has
detected a half-open connection.
- TCP B aborts at line 5.  
- TCP A willcontinue to try to establish the connection;

我就不瞎翻译了，意思和我在前面用中文说的解释差不多。

## 源码分析
处于 establish 状态的服务端如果收到了客户端的 SYN 报文时，内核会调用这些函数：

```csharp
tcp_v4_rcv
  -> tcp_v4_do_rcv
    -> tcp_rcv_established
      -> tcp_validate_incoming
        -> tcp_send_ack
```


我们只关注 tcp_validate_incoming 函数是怎么处理 SYN 报文的，精简后的代码如下：

![](https://img-blog.csdnimg.cn/780bc02c8fa940c0a320a5916b216c21.png)

从上面的代码实现可以看到，处于 establish 状态的服务端，在收到报文后，首先会判断序列号是否在窗口内，如果不在，则看看 RST 标记有没有被设置，如果有就会丢掉。然后如果没有 RST 标志，就会判断是否有 SYN 标记，如果有 SYN 标记就会跳转到 syn_challenge 标签，然后执行 tcp_send_challenge_ack 函数。

tcp_send_challenge_ack 函数里就会调用 tcp_send_ack 函数来回复一个携带了正确序列号和确认号的 ACK 报文。

## 如何关闭一个 TCP 连接？

这里问题大家这么一个问题，如何关闭一个 TCP 连接？

可能大家第一反应是「杀掉进程」不就行了吗？

是的，这个是最粗暴的方式，杀掉客户端进程和服务端进程影响的范围会有所不同：
- 在客户端杀掉进程的话，就会发送 FIN 报文，来断开这个客户端进程与服务端建立的所有 TCP 连接，这种方式影响范围只有这个客户端进程所建立的连接，而其他客户端或进程不会受影响。
- 而在服务端杀掉进程影响就大了，此时所有的 TCP 连接都会被关闭，服务端无法继续提供访问服务。

所以，关闭进程的方式并不可取，最好的方式要精细到关闭某一条 TCP 连接。

有的小伙伴可能会说，伪造一个四元组相同的 RST 报文不就行了？

这个思路很好，但是不要忘了还有个序列号的问题，你伪造的 RST 报文的序列号一定能被对方接受吗？

如果 RST 报文的序列号不能落在对方的滑动窗口内，这个 RST 报文会被对方丢弃的，就达不到关闭的连接的效果。

所以，**要伪造一个能关闭 TCP 连接的 RST 报文，必须同时满足「四元组相同」和「序列号正好落在对方的滑动窗口内」这两个条件。**

直接伪造符合预期的序列号是比较困难，因为如果一个正在传输数据的 TCP 连接，滑动窗口时刻都在变化，因此很难刚好伪造一个刚好落在对方滑动窗口内的序列号的 RST 报文。

办法还是有的，**我们可以伪造一个四元组相同的 SYN 报文，来拿到“合法”的序列号！**

正如我们最开始学到的，如果处于 establish 状态的服务端，收到四元组相同的 SYN 报文后，**会回复一个 Challenge ACK，这个 ACK 报文里的「确认号」，正好是服务端下一次想要接收的序列号，说白了，就是可以通过这一步拿到服务端下一次预期接收的序列号。**

**然后用这个确认号作为 RST 报文的序列号，发送给服务端，此时服务端会认为这个 RST 报文里的序列号是合法的，于是就会释放连接！**

在 Linux 上有个叫 killcx 的工具，就是基于上面这样的方式实现的，它会主动发送 SYN 包获取 SEQ/ACK 号，然后利用 SEQ/ACK 号伪造两个 RST 报文分别发给客户端和服务端，这样双方的 TCP 连接都会被释放，这种方式活跃和非活跃的 TCP 连接都可以杀掉。


使用方式也很简单，只需指明客户端的 IP 和端口号。

```csharp
./killcx <IP地址>:<端口号>
```
killcx 工具的工作原理，如下图。

![](https://img-blog.csdnimg.cn/95592346a9a747819cd27741a660213c.png)

它伪造客户端发送 SYN 报文，服务端收到后就会回复一个携带了正确「序列号和确认号」的 ACK 报文（Challenge ACK），然后就可以利用这个 ACK 报文里面的信息，伪造两个 RST 报文：
- 用 Challenge ACK 里的确认号伪造 RST 报文发送给服务端，服务端收到 RST 报文后就会释放连接。
- 用 Challenge ACK 里的序列号伪造 RST 报文发送给客户端，客户端收到 RST 也会释放连接。

正是通过这样的方式，成功将一个 TCP 连接关闭了！

这里给大家贴一个使用 killcx 工具关闭连接的抓包图，大家多看看序列号和确认号的变化。

![](https://img-blog.csdnimg.cn/71cbefee5ab741018386b6a37f492614.png?)

所以，以后抓包中，如果莫名奇妙出现一个 SYN 包，有可能对方接下来想要对你发起的 RST 攻击，直接将你的 TCP 连接断开！

怎么样，很巧妙吧！

---

最新的图解文章都在公众号首发，别忘记关注哦！！如果你想加入百人技术交流群，扫码下方二维码回复「加群」。

![img](https://cdn.xiaolincoding.com/gh/xiaolincoder/ImageHost3@main/%E5%85%B6%E4%BB%96/%E5%85%AC%E4%BC%97%E5%8F%B7%E4%BB%8B%E7%BB%8D.png)