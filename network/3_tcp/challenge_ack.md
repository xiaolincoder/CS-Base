# 4.9 已建立连接的 TCP，收到 SYN 会发生什么？

大家好，我是小林。

昨晚有位读者问了我这么个问题：

![](https://img-blog.csdnimg.cn/ea1c6e0165f04232ab02046132e63d0f.jpg)


大概意思是，一个已经建立的 TCP 连接，客户端中途宕机了，而服务端此时也没有数据要发送，一直处于 Established 状态，客户端恢复后，向服务端建立连接，此时服务端会怎么处理？

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

**处于 Established 状态的服务端，如果收到了客户端的 SYN 报文（注意此时的 SYN 报文其实是乱序的，因为 SYN 报文的初始化序列号其实是一个随机数），会回复一个携带了正确序列号和确认号的 ACK 报文，这个 ACK 被称之为 Challenge ACK。**

**接着，客户端收到这个 Challenge ACK，发现确认号（ack num）并不是自己期望收到的，于是就会回 RST 报文，服务端收到后，就会释放掉该连接。**

## RFC 文档解释

RFC 793 文档里的第 34 页里，有说到这个例子。

![](https://img-blog.csdnimg.cn/873ad18443c040708c415bab6592ae41.png)

原文的解释我也贴出来给大家看看。

- When the SYN arrives at line 3, TCP B, being in a synchronized state,
and the incoming segment outside the window, responds with an
acknowledgment indicating what sequence it next expects to hear (ACK
100).
- TCP A sees that this segment does not acknowledge anything it
sent and, being unsynchronized, sends a reset (RST) because it has
detected a half-open connection.
- TCP B aborts at line 5.  
- TCP A willcontinue to try to Established the connection;

我就不瞎翻译了，意思和我在前面用中文说的解释差不多。

## 源码分析
处于 Established 状态的服务端如果收到了客户端的 SYN 报文时，内核会调用这些函数：

```csharp
tcp_v4_rcv
  -> tcp_v4_do_rcv
    -> tcp_rcv_Establisheded
      -> tcp_validate_incoming
        -> tcp_send_ack
```


我们只关注 tcp_validate_incoming 函数是怎么处理 SYN 报文的，精简后的代码如下：

![](https://img-blog.csdnimg.cn/780bc02c8fa940c0a320a5916b216c21.png)

从上面的代码实现可以看到，处于 Established 状态的服务端，在收到报文后，首先会判断序列号是否在窗口内，如果不在，则看看 RST 标记有没有被设置，如果有就会丢掉。然后如果没有 RST 标志，就会判断是否有 SYN 标记，如果有 SYN 标记就会跳转到 syn_challenge 标签，然后执行 tcp_send_challenge_ack 函数。

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

如果 RST 报文的序列号不是对方期望收到的序列号，这个 RST 报文会被对方丢弃的，就达不到关闭的连接的效果。

举个例子，下面这个场景，客户端发送了一个长度为 100 的 TCP 数据报文，服务端收到后响应了 ACK 报文，表示收到了这个 TCP 数据报文。**服务端响应的这个 ACK 报文中的确认号（ack = x + 100）就是表明服务端下一次期望收到的序列号是 x + 100**。

![](https://cdn.xiaolincoding.com/gh/xiaolincoder/ImageHost4@main/网络/rst合法.png)

所以，**要伪造一个能关闭 TCP 连接的 RST 报文，必须同时满足「四元组相同」和「序列号是对方期望的」这两个条件。**

直接伪造符合预期的序列号是比较困难，因为如果一个正在传输数据的 TCP 连接，序列号都是时刻都在变化，因此很难刚好伪造一个正确序列号的 RST 报文。

### killcx 的工具

办法还是有的，**我们可以伪造一个四元组相同的 SYN 报文，来拿到“合法”的序列号！**

正如我们最开始学到的，如果处于 Established 状态的服务端，收到四元组相同的 SYN 报文后，**会回复一个 Challenge ACK，这个 ACK 报文里的「确认号」，正好是服务端下一次想要接收的序列号，说白了，就是可以通过这一步拿到服务端下一次预期接收的序列号。**

**然后用这个确认号作为 RST 报文的序列号，发送给服务端，此时服务端会认为这个 RST 报文里的序列号是合法的，于是就会释放连接！**

在 Linux 上有个叫 killcx 的工具，就是基于上面这样的方式实现的，它会主动发送 SYN 包获取 SEQ/ACK 号，然后利用 SEQ/ACK 号伪造两个 RST 报文分别发给客户端和服务端，这样双方的 TCP 连接都会被释放，这种方式活跃和非活跃的 TCP 连接都可以杀掉。


killcx 的工具使用方式也很简单，如果在服务端执行 killcx 工具，只需指明客户端的 IP 和端口号，如果在客户端执行 killcx 工具，则就指明服务端的 IP 和端口号。

```csharp
./killcx <IP地址>:<端口号>
```
killcx 工具的工作原理，如下图，下图是在客户端执行 killcx 工具。

![](https://img-blog.csdnimg.cn/95592346a9a747819cd27741a660213c.png)

它伪造客户端发送 SYN 报文，服务端收到后就会回复一个携带了正确「序列号和确认号」的 ACK 报文（Challenge ACK），然后就可以利用这个 ACK 报文里面的信息，伪造两个 RST 报文：
- 用 Challenge ACK 里的确认号伪造 RST 报文发送给服务端，服务端收到 RST 报文后就会释放连接。
- 用 Challenge ACK 里的序列号伪造 RST 报文发送给客户端，客户端收到 RST 也会释放连接。

正是通过这样的方式，成功将一个 TCP 连接关闭了！

这里给大家贴一个使用 killcx 工具关闭连接的抓包图，大家多看看序列号和确认号的变化。

![](https://img-blog.csdnimg.cn/71cbefee5ab741018386b6a37f492614.png?)

所以，以后抓包中，如果莫名奇妙出现一个 SYN 包，有可能对方接下来想要对你发起的 RST 攻击，直接将你的 TCP 连接断开！

怎么样，很巧妙吧！

### tcpkill 的工具

除了 killcx 工具能关闭 TCP 连接，还有 tcpkill 工具也可以做到。

这两个工具都是通过伪造 RST 报文来关闭指定的 TCP 连接，但是它们拿到正确的序列号的实现方式是不同的。

- tcpkill 工具是在双方进行 TCP 通信时，拿到对方下一次期望收到的序列号，然后将序列号填充到伪造的 RST 报文，并将其发送给对方，达到关闭 TCP 连接的效果。
- killcx 工具是主动发送一个 SYN 报文，对方收到后会回复一个携带了正确序列号和确认号的 ACK 报文，这个 ACK 被称之为 Challenge ACK，这时就可以拿到对方下一次期望收到的序列号，然后将序列号填充到伪造的 RST 报文，并将其发送给对方，达到关闭 TCP 连接的效果。

可以看到，这两个工具在获取对方下一次期望收到的序列号的方式是不同的。

tcpkill 工具属于被动获取，就是在双方进行 TCP 通信的时候，才能获取到正确的序列号，很显然**这种方式无法关闭非活跃的 TCP 连接**，只能用于关闭活跃的 TCP 连接。因为如果这条 TCP 连接一直没有任何数据传输，则就永远获取不到正确的序列号。

killcx 工具则是属于主动获取，它是主动发送一个 SYN 报文，通过对方回复的 Challenge ACK 来获取正确的序列号，所以这种方式**无论 TCP 连接是否活跃，都可以关闭**。

接下来，我就用这 tcpkill 工具来做个实验。

在这里，我用 nc 工具来模拟一个 TCP 服务端，监听 8888 端口。

![](https://cdn.xiaolincoding.com/gh/xiaolincoder/ImageHost4@main/网络/tcpkill/tcpkill1.png)

接着，在客户端机子上，用 nc 工具模拟一个 TCP 客户端，连接我们刚才启动的服务端，并且指定了客户端的端口为 11111。

![](https://cdn.xiaolincoding.com/gh/xiaolincoder/ImageHost4@main/网络/tcpkill/tcpkill2.png)

这时候，服务端就可以看到这条 TCP 连接了。

![图片](https://cdn.xiaolincoding.com/gh/xiaolincoder/ImageHost4@main/网络/tcpkill/tcpkill3.png)

注意，我这台服务端的公网 IP 地址是 121.43.173.240，私网 IP 地址是 172.19.11.21，在服务端通过 netstat 命令查看 TCP 连接的时候，则会将服务端的地址显示成私网 IP 地址。至此，我们前期工作就做好了。

接下来，我们在服务端执行 tcpkill 工具，来关闭这条 TCP 连接，看看会发生什么？

在这里，我指定了要关闭的客户端 IP 为 114.132.166.90 和端口为 11111 的 TCP 连接。

![图片](https://cdn.xiaolincoding.com/gh/xiaolincoder/ImageHost4@main/网络/tcpkill/tcpkill4.png)

可以看到，tcpkill 工具阻塞中，没有任何输出，而且此时的 TCP 连接还是存在的，并没有被干掉。

![图片](https://cdn.xiaolincoding.com/gh/xiaolincoder/ImageHost4@main/网络/tcpkill/tcpkill5.png)

为什么 TCP 连接没用被干掉？

因为在执行 tcpkill 工具后，这条 TCP 连接并没有传输任何数据，而 tcpkill 工具是需要拦截双方的 TCP 通信，才能获取到正确的序列号，从而才能伪装出正确的序列号的 RST 报文。

所以，从这里也说明了，**tcpkill 工具不适合关闭非活跃的 TCP 连接**。

接下来，我们尝试在客户端发送一个数据。

![图片](https://cdn.xiaolincoding.com/gh/xiaolincoder/ImageHost4@main/网络/tcpkill/tcpkill8.png)

可以看到，在发送了「hi」数据后，客户端就断开了，并且错误提示连接被对方关闭了。

此时，服务端已经查看不到刚才那条 TCP 连接了。

![图片](https://cdn.xiaolincoding.com/gh/xiaolincoder/ImageHost4@main/网络/tcpkill/tcpkill7.png)

然后，我们在服务端看看 tcpkill 工具输出的信息。

![图片](https://cdn.xiaolincoding.com/gh/xiaolincoder/ImageHost4@main/网络/tcpkill/tcpkill8.png)

可以看到， **tcpkill 工具给服务端和客户端都发送了伪造的 RST 报文，从而达到关闭一条 TCP 连接的效果**。

到这里我们知道了，运行 tcpkill 工具后，只有目标连接有新 TCP 包发送/接收的时候，才能关闭一条 TCP 连接。因此，**tcpkill 只适合关闭活跃的 TCP 连接，不适合用来关闭非活跃的 TCP 连接**。

上面的实验过程，我也抓了数据包，流程如下：

![图片](https://cdn.xiaolincoding.com/gh/xiaolincoder/ImageHost4@main/网络/tcpkill/tcpkill9.png)

最后一个 RST 报文就是 tcpkill 工具伪造的 RST 报文。

## 总结

要伪造一个能关闭 TCP 连接的 RST 报文，必须同时满足「四元组相同」和「序列号是对方期望的」这两个条件。

今天给大家介绍了两种关闭 TCP 连接的工具：tcpkill 和 killcx 工具。

这两种工具都是通过伪造 RST 报文来关闭 TCP 连接的，但是它们获取「对方下一次期望收到的序列号的方式是不同的，也正因此，造就了这两个工具的应用场景有区别。

- tcpkill 工具只能用来关闭活跃的 TCP 连接，无法关闭非活跃的 TCP 连接，因为 tcpkill 工具是等双方进行 TCP 通信后，才去获取正确的序列号，如果这条 TCP 连接一直没有任何数据传输，则就永远获取不到正确的序列号。
- killcx 工具可以用来关闭活跃和非活跃的 TCP 连接，因为 killcx 工具是主动发送 SYN 报文，这时对方就会回复 Challenge ACK，然后 killcx 工具就能从这个 ACK 获取到正确的序列号。

完！

---

最新的图解文章都在公众号首发，别忘记关注哦！！如果你想加入百人技术交流群，扫码下方二维码回复「加群」。

![img](https://cdn.xiaolincoding.com/gh/xiaolincoder/ImageHost3@main/%E5%85%B6%E4%BB%96/%E5%85%AC%E4%BC%97%E5%8F%B7%E4%BB%8B%E7%BB%8D.png)