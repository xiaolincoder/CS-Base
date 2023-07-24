# 4.11 在 TIME_WAIT 状态的 TCP 连接，收到 SYN 后会发生什么？

大家好，我是小林。

周末跟朋友讨论了一些 TCP 的问题，在查阅《Linux 服务器高性能编程》这本书的时候，发现书上写了这么一句话：

![图片](https://img-blog.csdnimg.cn/img_convert/65739ee668999bda02aa9236aad6437f.png)

书上说，处于 TIME_WAIT 状态的连接，在收到相同四元组的 SYN 后，会回 RST 报文，对方收到后就会断开连接。

书中作者只是提了这么一句话，没有给予源码或者抓包图的证据。

起初，我看到也觉得这个逻辑也挺符合常理的，但是当我自己去啃了 TCP 源码后，发现并不是这样的。

所以，今天就来讨论下这个问题，「**在 TCP 正常挥手过程中，处于 TIME_WAIT 状态的连接，收到相同四元组的 SYN 后会发生什么？**」

问题现象如下图，左边是服务端，右边是客户端：

![图片](https://img-blog.csdnimg.cn/img_convert/74b53919396dcda634cfd5b5795cbf16.png)

## 先说结论

在跟大家分析 TCP 源码前，我先跟大家直接说下结论。

针对这个问题，**关键是要看 SYN 的「序列号和时间戳」是否合法**，因为处于 TIME_WAIT 状态的连接收到 SYN 后，会判断 SYN 的「序列号和时间戳」是否合法，然后根据判断结果的不同做不同的处理。

先跟大家说明下，什么是「合法」的 SYN？

- **合法 SYN**：客户端的  SYN 的「序列号」比服务端「期望下一个收到的序列号」要**大**，**并且** SYN 的「时间戳」比服务端「最后收到的报文的时间戳」要**大**。
- **非法 SYN**：客户端的  SYN 的「序列号」比服务端「期望下一个收到的序列号」要**小**，**或者** SYN 的「时间戳」比服务端「最后收到的报文的时间戳」要**小**。

上面 SYN 合法判断是基于双方都开启了 TCP 时间戳机制的场景，如果双方都没有开启 TCP 时间戳机制，则 SYN 合法判断如下：

- **合法 SYN**：客户端的  SYN 的「序列号」比服务端「期望下一个收到的序列号」要**大**。
- **非法 SYN**：客户端的  SYN 的「序列号」比服务端「期望下一个收到的序列号」要**小**。

### 收到合法 SYN

如果处于 TIME_WAIT 状态的连接收到「合法的 SYN」后，**就会重用此四元组连接，跳过 2MSL 而转变为 SYN_RECV 状态，接着就能进行建立连接过程**。

用下图作为例子，双方都启用了 TCP 时间戳机制，TSval 是发送报文时的时间戳：

![图片](https://img-blog.csdnimg.cn/img_convert/39d0d04adf72fe3d37623acff9ae2507.png)

上图中，在收到第三次挥手的 FIN 报文时，会记录该报文的 TSval（21），用 ts_recent 变量保存。然后会计算下一次期望收到的序列号，本次例子下一次期望收到的序列号就是 301，用 rcv_nxt 变量保存。

处于 TIME_WAIT 状态的连接收到 SYN 后，**因为 SYN 的 seq（400）大于 rcv_nxt（301），并且 SYN 的 TSval（30）大于 ts_recent（21），所以是一个「合法的 SYN」，于是就会重用此四元组连接，跳过 2MSL 而转变为 SYN_RECV 状态，接着就能进行建立连接过程。**

### 收到非法的 SYN

如果处于 TIME_WAIT 状态的连接收到「非法的 SYN」后，就会**再回复一个第四次挥手的 ACK 报文，客户端收到后，发现并不是自己期望收到确认号（ack num），就回 RST 报文给服务端**。

用下图作为例子，双方都启用了 TCP 时间戳机制，TSval 是发送报文时的时间戳：

![](https://cdn.xiaolincoding.com/gh/xiaolincoder/network/tcp/tw收到不合法.png)

上图中，在收到第三次挥手的 FIN 报文时，会记录该报文的 TSval（21），用 ts_recent 变量保存。然后会计算下一次期望收到的序列号，本次例子下一次期望收到的序列号就是 301，用 rcv_nxt 变量保存。

处于 TIME_WAIT 状态的连接收到 SYN 后，**因为 SYN 的 seq（200）小于 rcv_nxt（301），所以是一个「非法的 SYN」，就会再回复一个与第四次挥手一样的 ACK 报文，客户端收到后，发现并不是自己期望收到确认号，就回 RST 报文给服务端**。

> PS：这里先埋一个疑问，处于 TIME_WAIT 状态的连接，收到 RST 会断开连接吗？

## 源码分析

下面源码分析是基于 Linux 4.2 版本的内核代码。

Linux 内核在收到 TCP 报文后，会执行 `tcp_v4_rcv` 函数，在该函数和 TIME_WAIT 状态相关的主要代码如下：

```c
int tcp_v4_rcv(struct sk_buff *skb)
{
  struct sock *sk;
 ...
  //收到报文后，会调用此函数，查找对应的 sock
 sk = __inet_lookup_skb(&tcp_hashinfo, skb, __tcp_hdrlen(th), th->source,
          th->dest, sdif, &refcounted);
 if (!sk)
  goto no_tcp_socket;

process:
  //如果连接的状态为 time_wait，会跳转到 do_time_wait
 if (sk->sk_state == TCP_TIME_WAIT)
  goto do_time_wait;

...

do_time_wait:
  ...
  //由 tcp_timewait_state_process 函数处理在 time_wait 状态收到的报文
 switch (tcp_timewait_state_process(inet_twsk(sk), skb, th)) {
    // 如果是 TCP_TW_SYN，那么允许此 SYN 重建连接
    // 即允许 TIM_WAIT 状态跃迁到 SYN_RECV
    case TCP_TW_SYN: {
      struct sock *sk2 = inet_lookup_listener(....);
      if (sk2) {
          ....
          goto process;
      }
    }
    // 如果是 TCP_TW_ACK，那么，返回记忆中的 ACK
    case TCP_TW_ACK:
      tcp_v4_timewait_ack(sk, skb);
      break;
    // 如果是 TCP_TW_RST 直接发送 RESET 包
    case TCP_TW_RST:
      tcp_v4_send_reset(sk, skb);
      inet_twsk_deschedule_put(inet_twsk(sk));
      goto discard_it;
     // 如果是 TCP_TW_SUCCESS 则直接丢弃此包，不做任何响应
    case TCP_TW_SUCCESS:;
 }
 goto discard_it;
}
```

该代码的过程：

1. 接收到报文后，会调用 `__inet_lookup_skb()` 函数查找对应的 sock 结构；
2. 如果连接的状态是 `TIME_WAIT`，会跳转到 do_time_wait 处理；
3. 由 `tcp_timewait_state_process()` 函数来处理收到的报文，处理后根据返回值来做相应的处理。

先跟大家说下，如果收到的 SYN 是合法的，`tcp_timewait_state_process()` 函数就会返回 `TCP_TW_SYN`，然后重用此连接。如果收到的 SYN 是非法的，`tcp_timewait_state_process()` 函数就会返回 `TCP_TW_ACK`，然后会回上次发过的 ACK。

接下来，看 `tcp_timewait_state_process()` 函数是如何判断 SYN 包的。

```c
enum tcp_tw_status
tcp_timewait_state_process(struct inet_timewait_sock *tw, struct sk_buff *skb,
      const struct tcphdr *th)
{
 ...
  //paws_reject 为 false，表示没有发生时间戳回绕
  //paws_reject 为 true，表示发生了时间戳回绕
 bool paws_reject = false;

 tmp_opt.saw_tstamp = 0;
  //TCP 头中有选项且旧连接开启了时间戳选项
 if (th->doff > (sizeof(*th) >> 2) && tcptw->tw_ts_recent_stamp) { 
  //解析选项
    tcp_parse_options(twsk_net(tw), skb, &tmp_opt, 0, NULL);

  if (tmp_opt.saw_tstamp) {
   ...
      //检查收到的报文的时间戳是否发生了时间戳回绕
   paws_reject = tcp_paws_reject(&tmp_opt, th->rst);
  }
 }

....

  //是 SYN 包、没有 RST、没有 ACK、时间戳没有回绕，并且序列号也没有回绕，
 if (th->syn && !th->rst && !th->ack && !paws_reject &&
     (after(TCP_SKB_CB(skb)->seq, tcptw->tw_rcv_nxt) ||
      (tmp_opt.saw_tstamp && //新连接开启了时间戳
       (s32)(tcptw->tw_ts_recent - tmp_opt.rcv_tsval) < 0))) { //时间戳没有回绕
    // 初始化序列号
    u32 isn = tcptw->tw_snd_nxt + 65535 + 2; 
    if (isn == 0)
      isn++;
    TCP_SKB_CB(skb)->tcp_tw_isn = isn;
    return TCP_TW_SYN; //允许重用 TIME_WAIT 四元组重新建立连接
 }


 if (!th->rst) {
    // 如果时间戳回绕，或者报文里包含 ack，则将 TIMEWAIT 状态的持续时间重新延长
  if (paws_reject || th->ack)
    inet_twsk_schedule(tw, &tcp_death_row, TCP_TIMEWAIT_LEN,
        TCP_TIMEWAIT_LEN);

     // 返回 TCP_TW_ACK, 发送上一次的 ACK
    return TCP_TW_ACK;
 }
 inet_twsk_put(tw);
 return TCP_TW_SUCCESS;
}
```

如果双方启用了 TCP 时间戳机制，就会通过 `tcp_paws_reject()` 函数来判断时间戳是否发生了回绕，也就是「当前收到的报文的时间戳」是否大于「上一次收到的报文的时间戳」：

- 如果大于，就说明没有发生时间戳绕回，函数返回 false。
- 如果小于，就说明发生了时间戳回绕，函数返回 true。

从源码可以看到，当收到 SYN 包后，如果该 SYN 包的时间戳没有发生回绕，也就是时间戳是递增的，并且 SYN 包的序列号也没有发生回绕，也就是 SYN 的序列号「大于」下一次期望收到的序列号。就会初始化一个序列号，然后返回 TCP_TW_SYN，接着就重用该连接，也就跳过 2MSL 而转变为 SYN_RECV 状态，接着就能进行建立连接过程。

如果双方都没有启用 TCP 时间戳机制，就只需要判断 SYN 包的序列号有没有发生回绕，如果 SYN 的序列号大于下一次期望收到的序列号，就可以跳过 2MSL，重用该连接。

如果 SYN 包是非法的，就会返回 TCP_TW_ACK，接着就会发送与上一次一样的 ACK 给对方。

## 在 TIME_WAIT 状态，收到 RST 会断开连接吗？

在前面我留了一个疑问，处于 TIME_WAIT 状态的连接，收到 RST 会断开连接吗？

会不会断开，关键看 `net.ipv4.tcp_rfc1337` 这个内核参数（默认情况是为 0）：

- 如果这个参数设置为 0，收到 RST 报文会提前结束 TIME_WAIT 状态，释放连接。
- 如果这个参数设置为 1，就会丢掉 RST 报文。

源码处理如下：

```c
enum tcp_tw_status
tcp_timewait_state_process(struct inet_timewait_sock *tw, struct sk_buff *skb,
      const struct tcphdr *th)
{
....
  //rst 报文的时间戳没有发生回绕
 if (!paws_reject &&
     (TCP_SKB_CB(skb)->seq == tcptw->tw_rcv_nxt &&
      (TCP_SKB_CB(skb)->seq == TCP_SKB_CB(skb)->end_seq || th->rst))) {

      //处理 rst 报文
      if (th->rst) {
        //不开启这个选项，当收到 RST 时会立即回收 tw，但这样做是有风险的
        if (twsk_net(tw)->ipv4.sysctl_tcp_rfc1337 == 0) {
          kill:
          //删除 tw 定时器，并释放 tw
          inet_twsk_deschedule_put(tw);
          return TCP_TW_SUCCESS;
        }
      } else {
        //将 TIMEWAIT 状态的持续时间重新延长
        inet_twsk_reschedule(tw, TCP_TIMEWAIT_LEN);
      }

      ...
      return TCP_TW_SUCCESS;
    }
}
```

TIME_WAIT 状态收到 RST 报文而释放连接，这样等于跳过 2MSL 时间，这么做还是有风险。

sysctl_tcp_rfc1337 这个参数是在 rfc 1337 文档提出来的，目的是避免因为 TIME_WAIT 状态收到 RST 报文而跳过  2MSL 的时间，文档里也给出跳过  2MSL 时间会有什么潜在问题。

TIME_WAIT 状态之所以要持续 2MSL 时间，主要有两个目的：

- 防止历史连接中的数据，被后面相同四元组的连接错误的接收；
- 保证「被动关闭连接」的一方，能被正确的关闭；

详细的为什么要设计 TIME_WAIT 状态，我在这篇有详细说明：[如果 TIME_WAIT 状态持续时间过短或者没有，会有什么问题？](https://mp.weixin.qq.com/s?__biz=MzUxODAzNDg4NQ==&mid=2247502380&idx=1&sn=7b82818a5fb6f1127d17f0ded550c4bd&scene=21#wechat_redirect)

虽然 TIME_WAIT 状态持续的时间是有一点长，显得很不友好，但是它被设计来就是用来避免发生乱七八糟的事情。

《UNIX 网络编程》一书中却说道：**TIME_WAIT 是我们的朋友，它是有助于我们的，不要试图避免这个状态，而是应该弄清楚它**。

所以，我个人觉得将 `net.ipv4.tcp_rfc1337` 设置为 1 会比较安全。

## 总结

在 TCP 正常挥手过程中，处于 TIME_WAIT 状态的连接，收到相同四元组的 SYN 后会发生什么？

如果双方开启了时间戳机制：

- 如果客户端的  SYN 的「序列号」比服务端「期望下一个收到的序列号」要**大**，**并且**SYN 的「时间戳」比服务端「最后收到的报文的时间戳」要**大**。那么就会重用该四元组连接，跳过 2MSL 而转变为 SYN_RECV 状态，接着就能进行建立连接过程。
- 如果客户端的  SYN 的「序列号」比服务端「期望下一个收到的序列号」要**小**，**或者**SYN 的「时间戳」比服务端「最后收到的报文的时间戳」要**小**。那么就会**再回复一个第四次挥手的 ACK 报文，客户端收到后，发现并不是自己期望收到确认号，就回 RST 报文给服务端**。

在 TIME_WAIT 状态，收到 RST 会断开连接吗？

- 如果 `net.ipv4.tcp_rfc1337` 参数为 0，则提前结束 TIME_WAIT 状态，释放连接。
- 如果 `net.ipv4.tcp_rfc1337` 参数为 1，则会丢掉该 RST 报文。

完！

---

最新的图解文章都在公众号首发，别忘记关注哦！！如果你想加入百人技术交流群，扫码下方二维码回复「加群」。

![img](https://cdn.xiaolincoding.com/gh/xiaolincoder/ImageHost3@main/%E5%85%B6%E4%BB%96/%E5%85%AC%E4%BC%97%E5%8F%B7%E4%BB%8B%E7%BB%8D.png)