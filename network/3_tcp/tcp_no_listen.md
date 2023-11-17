# 4.19 服务端没有 listen，客户端发起连接建立，会发生什么？

大家好，我是小林。

早上看到一个读者说面字节三面的时候，问了这个问题：

![图片](https://img-blog.csdnimg.cn/img_convert/5f5b9c96c86580e3f14978d5c10c7721.jpeg)

这位读者的角度是以为服务端没有调用 listen，客户端会 ping 不通服务器，很明显，搞错了。

ping 使用的协议是 ICMP，属于网络层的事情，而面试官问的是传输层的问题。

针对这个问题，服务端如果只 bind 了 IP 地址和端口，而没有调用 listen 的话，然后客户端对服务端发起了 TCP 连接建立，此时那么会发生什么呢？

## 做个实验

这个问题，自己做个实验就知道了。

我用下面这个程序作为例子，绑定了 IP 地址 + 端口，而没有调用 listen。

```c
/*******服务器程序  TCPServer.c ************/
#include <stdlib.h>
#include <stdio.h>
#include <errno.h>
#include <string.h>
#include <netdb.h>
#include <sys/types.h>
#include <netinet/in.h>
#include <sys/socket.h>

int main(int argc, char *argv[])
{
    int sockfd, ret;
    struct sockaddr_in server_addr;

    /* 服务器端创建 tcp socket 描述符 */
    sockfd = socket(AF_INET, SOCK_STREAM, 0);
    if(sockfd < 0)
    {
        fprintf(stderr, "Socket error:%s\n\a", strerror(errno));
        exit(1);
    }

    /* 服务器端填充 sockaddr 结构 */
    bzero(&server_addr, sizeof(struct sockaddr_in));
    server_addr.sin_family = AF_INET;
    server_addr.sin_addr.s_addr = htonl(INADDR_ANY);
    server_addr.sin_port = htons(8888);
  
  /* 绑定 ip + 端口 */
    ret = bind(sockfd, (struct sockaddr *)(&server_addr), sizeof(struct sockaddr));
    if(ret < 0)
    {
        fprintf(stderr, "Bind error:%s\n\a", strerror(errno));
        exit(1);
    }
  
  //没有调用 listen
    
    sleep(1000);
    close(sockfd);
    return 0;
}
```

然后，我用浏览器访问这个地址：http://121.43.173.240:8888/

![图片](https://img-blog.csdnimg.cn/img_convert/5bdb5443db5b97ff724ab94e014af6a5.png)

报错连接服务器失败。

同时，我也用抓包工具，抓了这个过程。

![图片](https://img-blog.csdnimg.cn/img_convert/a77921ffafbbff86d07983ca0db3e6e0.png)

可以看到，客户端对服务端发起 SYN 报文后，服务端回了 RST 报文。

所以，这个问题就有了答案，**服务端如果只 bind 了 IP 地址和端口，而没有调用 listen 的话，然后客户端对服务端发起了连接建立，服务端会回 RST 报文。**

## 源码分析

接下来，带大家源码分析一下。

Linux 内核处理收到 TCP 报文的入口函数是  tcp_v4_rcv，在收到 TCP 报文后，会调用 __inet_lookup_skb 函数找到 TCP 报文所属 socket。

```plain
int tcp_v4_rcv(struct sk_buff *skb)
{
 ...
  
 sk = __inet_lookup_skb(&tcp_hashinfo, skb, th->source, th->dest);
 if (!sk)
  goto no_tcp_socket;
 ...
}
```

__inet_lookup_skb 函数首先查找连接建立状态的 socket（__inet_lookup_established），在没有命中的情况下，才会查找监听套接口（__inet_lookup_listener）。

![图片](https://img-blog.csdnimg.cn/img_convert/88416aa95d255495e07fb3a002b2167b.png)

查找监听套接口（__inet_lookup_listener）这个函数的实现是，根据目的地址和目的端口算出一个哈希值，然后在哈希表找到对应监听该端口的 socket。

本次的案例中，服务端是没有调用 listen 函数的，所以自然也是找不到监听该端口的 socket。

所以，__inet_lookup_skb 函数最终找不到对应的 socket，于是跳转到 no_tcp_socket。

![图片](https://img-blog.csdnimg.cn/img_convert/54ee363e149ee3dfba30efb1a542ef5c.png)

在这个错误处理中，只要收到的报文（skb）的「校验和」没问题的话，内核就会调用 tcp_v4_send_reset 发送 RST 中止这个连接。

至此，整个源码流程就解析完。

其实很多网络的问题，大家都可以自己做实验来找到答案的。

![图片](https://img-blog.csdnimg.cn/img_convert/8d04584bf7fa40f02229d611a569f370.jpeg)

## 没有 listen，能建立 TCP 连接吗？

标题的问题在前面已经解答，**现在我们看另外一个相似的问题**。

之前看群消息，看到有读者面试腾讯的时候，被问到这么一个问题。

> 不使用 listen，可以建立 TCP 连接吗？

答案，**是可以的，客户端是可以自己连自己的形成连接（TCP 自连接），也可以两个客户端同时向对方发出请求建立连接（TCP 同时打开），这两个情况都有个共同点，就是没有服务端参与，也就是没有 listen，就能建立连接**。

> 那没有 listen，为什么还能建立连接？

我们知道执行 listen 方法时，会创建半连接队列和全连接队列。

三次握手的过程中会在这两个队列中暂存连接信息。

所以形成连接，前提是你得有个地方存放着，方便握手的时候能根据 IP + 端口等信息找到对应的 socket。

> 那么客户端会有半连接队列吗？

显然没有，因为客户端没有执行 listen，因为半连接队列和全连接队列都是在执行 listen 方法时，内核自动创建的。

但内核还有个全局 hash 表，可以用于存放 sock 连接的信息。

这个全局 hash 表其实还细分为 ehash，bhash 和 listen_hash 等，但因为过于细节，大家理解成有一个全局 hash 就够了，

**在 TCP 自连接的情况中，客户端在 connect 方法时，最后会将自己的连接信息放入到这个全局 hash 表中，然后将信息发出，消息在经过回环地址重新回到 TCP 传输层的时候，就会根据 IP + 端口信息，再一次从这个全局 hash 中取出信息。于是握手包一来一回，最后成功建立连接**。

TCP 同时打开的情况也类似，只不过从一个客户端变成了两个客户端而已。

> 做个实验

客户端自连接的代码，TCP socket 可以 connect 它本身 bind 的地址和端口：


```c
#include <sys/types.h> 
#include <sys/socket.h>
#include <netinet/in.h>
#include <unistd.h>
#include <stdlib.h>
#include <stdio.h>
#include <string.h>
#include <errno.h>

#define LOCAL_IP_ADDR		(0x7F000001) // IP 127.0.0.1
#define LOCAL_TCP_PORT		(34567) // 端口

int main(void)
{
	struct sockaddr_in local, peer;
	int ret;
	char buf[128];
	int sock = socket(AF_INET, SOCK_STREAM, 0);

	memset(&local, 0, sizeof(local));
	memset(&peer, 0, sizeof(peer));

	local.sin_family = AF_INET;
	local.sin_port = htons(LOCAL_TCP_PORT);
	local.sin_addr.s_addr = htonl(LOCAL_IP_ADDR);

	peer = local;	

    int flag = 1;
    ret = setsockopt(sock, SOL_SOCKET, SO_REUSEADDR, &flag, sizeof(flag));
    if (ret == -1) {
        printf("Fail to setsocket SO_REUSEADDR: %s\n", strerror(errno));
        exit(1);
    }

	ret = bind(sock, (const struct sockaddr *)&local, sizeof(local));
	if (ret) {
		printf("Fail to bind: %s\n", strerror(errno));
		exit(1);
	}
	
	ret = connect(sock, (const struct sockaddr *)&peer, sizeof(peer));
	if (ret) {
		printf("Fail to connect myself: %s\n", strerror(errno));
		exit(1);
	}
	
	printf("Connect to myself successfully\n");

    //发送数据
	strcpy(buf, "Hello, myself~");
	send(sock, buf, strlen(buf), 0);

	memset(buf, 0, sizeof(buf));
	
	//接收数据
	recv(sock, buf, sizeof(buf), 0);
	printf("Recv the msg: %s\n", buf);

    sleep(1000);
	close(sock);
	return 0;
}
```

编译运行：

![](https://img-blog.csdnimg.cn/9db974179b9e4a279f7edb0649752c27.png)


通过 netstat 命令命令客户端自连接的 TCP 连接：

![在这里插入图片描述](https://img-blog.csdnimg.cn/e2b116e843c14e468eadf9d30e1b877c.png)

从截图中，可以看到 TCP socket 成功的“连接”了自己，并发送和接收了数据包，netstat 的输出更证明了 TCP 的两端地址和端口是完全相同的。

---

最新的图解文章都在公众号首发，别忘记关注哦！！如果你想加入百人技术交流群，扫码下方二维码回复「加群」。

![img](https://cdn.xiaolincoding.com/gh/xiaolincoder/ImageHost3@main/%E5%85%B6%E4%BB%96/%E5%85%AC%E4%BC%97%E5%8F%B7%E4%BB%8B%E7%BB%8D.png)