# 4.22 TCP 四次挥手，可以变成三次吗？

大家好，我是小林。

有位读者面美团时，被问到：**TCP 四次挥手中，能不能把第二次的 ACK 报文，放到第三次 FIN 报文一起发送？**

![](https://img-blog.csdnimg.cn/6e02477ccea24facbf7eada108158bc2.png)


虽然我们在学习 TCP 挥手时，学到的是需要四次来完成 TCP 挥手，但是**在一些情况下，TCP 四次挥手是可以变成 TCP 三次挥手的**。

![在这里插入图片描述](https://img-blog.csdnimg.cn/52f35dcbe24a4ca7abb23f292837c707.png)


而且在用 wireshark 工具抓包的时候，我们也会常看到 TCP 挥手过程是三次，而不是四次，如下图：

![在这里插入图片描述](https://img-blog.csdnimg.cn/361207c2e5c34bec8708b79990ba7e99.png)

先来回答为什么 RFC 文档里定义 TCP 挥手过程是要四次？

再来回答什么情况下，什么情况会出现三次挥手？

## TCP 四次挥手

TCP 四次挥手的过程如下：

![在这里插入图片描述](https://img-blog.csdnimg.cn/18635e15653a4affbdab2c9bf72d599e.png)


具体过程：

- 客户端主动调用关闭连接的函数，于是就会发送 FIN 报文，这个  FIN 报文代表客户端不会再发送数据了，进入 FIN_WAIT_1 状态；
- 服务端收到了 FIN 报文，然后马上回复一个 ACK 确认报文，此时服务端进入 CLOSE_WAIT 状态。在收到 FIN 报文的时候，TCP 协议栈会为 FIN 包插入一个文件结束符 EOF 到接收缓冲区中，服务端应用程序可以通过 read 调用来感知这个 FIN 包，这个 EOF 会被**放在已排队等候的其他已接收的数据之后**，所以必须要得继续 read 接收缓冲区已接收的数据；
- 接着，当服务端在 read 数据的时候，最后自然就会读到 EOF，接着 **read() 就会返回 0，这时服务端应用程序如果有数据要发送的话，就发完数据后才调用关闭连接的函数，如果服务端应用程序没有数据要发送的话，可以直接调用关闭连接的函数**，这时服务端就会发一个 FIN 包，这个  FIN 报文代表服务端不会再发送数据了，之后处于 LAST_ACK 状态；
- 客户端接收到服务端的 FIN 包，并发送 ACK 确认包给服务端，此时客户端将进入 TIME_WAIT 状态；
- 服务端收到 ACK 确认包后，就进入了最后的 CLOSE 状态；
- 客户端经过 2MSL 时间之后，也进入 CLOSE 状态；

你可以看到，每个方向都需要**一个 FIN 和一个 ACK**，因此通常被称为**四次挥手**。

### 为什么 TCP 挥手需要四次呢？

服务器收到客户端的 FIN 报文时，内核会马上回一个 ACK 应答报文，**但是服务端应用程序可能还有数据要发送，所以并不能马上发送 FIN 报文，而是将发送 FIN 报文的控制权交给服务端应用程序**：

   - 如果服务端应用程序有数据要发送的话，就发完数据后，才调用关闭连接的函数；
   - 如果服务端应用程序没有数据要发送的话，可以直接调用关闭连接的函数，

从上面过程可知，**是否要发送第三次挥手的控制权不在内核，而是在被动关闭方（上图的服务端）的应用程序，因为应用程序可能还有数据要发送，由应用程序决定什么时候调用关闭连接的函数，当调用了关闭连接的函数，内核就会发送 FIN 报文了，** 所以服务端的 ACK 和 FIN 一般都会分开发送。

> FIN 报文一定得调用关闭连接的函数，才会发送吗？


不一定。

如果进程退出了，不管是不是正常退出，还是异常退出（如进程崩溃），内核都会发送 FIN 报文，与对方完成四次挥手。

### 粗暴关闭 vs 优雅关闭

前面介绍 TCP 四次挥手的时候，并没有详细介绍关闭连接的函数，其实关闭的连接的函数有两种函数：

- close 函数，同时 socket 关闭发送方向和读取方向，也就是 socket 不再有发送和接收数据的能力。如果有多进程/多线程共享同一个 socket，如果有一个进程调用了 close 关闭只是让 socket 引用计数 -1，并不会导致 socket 不可用，同时也不会发出 FIN 报文，其他进程还是可以正常读写该 socket，直到引用计数变为 0，才会发出 FIN 报文。
- shutdown 函数，可以指定 socket 只关闭发送方向而不关闭读取方向，也就是 socket 不再有发送数据的能力，但是还是具有接收数据的能力。如果有多进程/多线程共享同一个 socket，shutdown 则不管引用计数，直接使得该 socket 不可用，然后发出 FIN 报文，如果有别的进程企图使用该 socket，将会受到影响。

如果客户端是用 close 函数来关闭连接，那么在 TCP 	四次挥手过程中，如果收到了服务端发送的数据，由于客户端已经不再具有发送和接收数据的能力，所以客户端的内核会回 RST 报文给服务端，然后内核会释放连接，这时就不会经历完成的 TCP 四次挥手，所以我们常说，调用 close 是粗暴的关闭。

![在这里插入图片描述](https://img-blog.csdnimg.cn/3b5f1897d2d74028aaf4d552fbce1a74.png)


当服务端收到 RST 后，内核就会释放连接，当服务端应用程序再次发起读操作或者写操作时，就能感知到连接已经被释放了：

- 如果是读操作，则会返回 RST 的报错，也就是我们常见的 Connection reset by peer。
- 如果是写操作，那么程序会产生 SIGPIPE 信号，应用层代码可以捕获并处理信号，如果不处理，则默认情况下进程会终止，异常退出。

相对的，shutdown 函数因为可以指定只关闭发送方向而不关闭读取方向，所以即使在 TCP 四次挥手过程中，如果收到了服务端发送的数据，客户端也是可以正常读取到该数据的，然后就会经历完整的 TCP 四次挥手，所以我们常说，调用 shutdown 是优雅的关闭。

![优雅关闭.drawio.png](https://img-blog.csdnimg.cn/71f5646ec58849e5921adc08bb6789d4.png)


但是注意，shutdown 函数也可以指定「只关闭读取方向，而不关闭发送方向」，但是这时候内核是不会发送 FIN 报文的，因为发送 FIN 报文是意味着我方将不再发送任何数据，而 shutdown 如果指定「不关闭发送方向」，就意味着 socket 还有发送数据的能力，所以内核就不会发送 FIN。

## 什么情况会出现三次挥手？

当被动关闭方（上图的服务端）在 TCP 挥手过程中，「**没有数据要发送」并且「开启了 TCP 延迟确认机制」，那么第二和第三次挥手就会合并传输，这样就出现了三次挥手。**

![在这里插入图片描述](https://img-blog.csdnimg.cn/d7b349efa4f94453943b433b704a4ca8.png)


然后因为 TCP 延迟确认机制是默认开启的，所以导致我们抓包时，看见三次挥手的次数比四次挥手还多。

> 什么是  TCP 延迟确认机制？

当发送没有携带数据的 ACK，它的网络效率也是很低的，因为它也有 40 个字节的 IP 头 和 TCP 头，但却没有携带数据报文。
为了解决 ACK 传输效率低问题，所以就衍生出了 **TCP 延迟确认**。
TCP 延迟确认的策略：

- 当有响应数据要发送时，ACK 会随着响应数据一起立刻发送给对方
- 当没有响应数据要发送时，ACK 将会延迟一段时间，以等待是否有响应数据可以一起发送
- 如果在延迟等待发送 ACK 期间，对方的第二个数据报文又到达了，这时就会立刻发送 ACK

![](https://img-blog.csdnimg.cn/33f3d2d54a924b0a80f565038327e0e4.png)


延迟等待的时间是在 Linux 内核中定义的，如下图：

![](https://img-blog.csdnimg.cn/ae241915337a4d2c9cb2f7ab91e6661d.png)

关键就需要 HZ 这个数值大小，HZ 是跟系统的时钟频率有关，每个操作系统都不一样，在我的 Linux 系统中 HZ 大小是 1000，如下图：

![](https://img-blog.csdnimg.cn/7a67bd4dc2894335b974e38674ba90b4.png)


知道了 HZ 的大小，那么就可以算出：

- 最大延迟确认时间是 200 ms（1000/5）
- 最短延迟确认时间是 40 ms（1000/25）

> 怎么关闭 TCP 延迟确认机制？

如果要关闭 TCP 延迟确认机制，可以在 Socket 设置里启用 TCP_QUICKACK。

```cpp
// 1 表示开启 TCP_QUICKACK，即关闭 TCP 延迟确认机制
int value = 1;
setsockopt(socketfd, IPPROTO_TCP, TCP_QUICKACK, (char*)& value, sizeof(int));
```

### 实验验证

#### 实验一

接下来，来给大家做个实验，验证这个结论：

> 当被动关闭方（上图的服务端）在 TCP 挥手过程中，「**没有数据要发送」并且「开启了 TCP 延迟确认机制」，那么第二和第三次挥手就会合并传输，这样就出现了三次挥手。**


服务端的代码如下，做的事情很简单，就读取数据，然后当 read 返回 0 的时候，就马上调用 close 关闭连接。因为 TCP 延迟确认机制是默认开启的，所以不需要特殊设置。

```cpp
#include <stdlib.h>
#include <stdio.h>
#include <errno.h>
#include <string.h>
#include <netdb.h>
#include <sys/types.h>
#include <netinet/in.h>
#include <sys/socket.h>
#include <netinet/tcp.h>

#define MAXLINE 1024

int main(int argc, char *argv[])
{

    // 1. 创建一个监听 socket
    int listenfd = socket(AF_INET, SOCK_STREAM, 0);
    if(listenfd < 0)
    {
        fprintf(stderr, "socket error : %s\n", strerror(errno));
        return -1;
    }

    // 2. 初始化服务器地址和端口
    struct sockaddr_in server_addr;
    bzero(&server_addr, sizeof(struct sockaddr_in));
    server_addr.sin_family = AF_INET;
    server_addr.sin_addr.s_addr = htonl(INADDR_ANY);
    server_addr.sin_port = htons(8888);

    // 3. 绑定地址 + 端口
    if(bind(listenfd, (struct sockaddr *)(&server_addr), sizeof(struct sockaddr)) < 0)
    {
        fprintf(stderr,"bind error:%s\n", strerror(errno));
        return -1;
    }

    printf("begin listen....\n");

    // 4. 开始监听
    if(listen(listenfd, 128))
    {
        fprintf(stderr, "listen error:%s\n\a", strerror(errno));
        exit(1);
    }


    // 5. 获取已连接的 socket
    struct sockaddr_in client_addr;
    socklen_t client_addrlen = sizeof(client_addr);
    int clientfd = accept(listenfd, (struct sockaddr *)&client_addr, &client_addrlen);
    if(clientfd < 0) {
        fprintf(stderr, "accept error:%s\n\a", strerror(errno));
        exit(1);
    }

    printf("accept success\n");

    char message[MAXLINE] = {0};
    
    while(1) {
        //6. 读取客户端发送的数据
        int n = read(clientfd, message, MAXLINE);
        if(n < 0) { // 读取错误
            fprintf(stderr, "read error:%s\n\a", strerror(errno));
            break;
        } else if(n == 0) {  // 返回 0，代表读到 FIN 报文
            fprintf(stderr, "client closed \n");
            close(clientfd); // 没有数据要发送，立马关闭连接
            break;
        }

        message[n] = 0; 
        printf("received %d bytes: %s\n", n, message);
    }
	
    close(listenfd);
    return 0;
}
```

客户端代码如下，做的事情也很简单，与服务端连接成功后，就发送数据给服务端，然后睡眠一秒后，就调用 close 关闭连接，所以客户端是主动关闭方：

```cpp
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

    // 1. 创建一个监听 socket
    int connectfd = socket(AF_INET, SOCK_STREAM, 0);
    if(connectfd < 0)
    {
        fprintf(stderr, "socket error : %s\n", strerror(errno));
        return -1;
    }

    // 2. 初始化服务器地址和端口
    struct sockaddr_in server_addr;
    bzero(&server_addr, sizeof(struct sockaddr_in));
    server_addr.sin_family = AF_INET;
    server_addr.sin_addr.s_addr = inet_addr("127.0.0.1");
    server_addr.sin_port = htons(8888);
    
    // 3. 连接服务器
    if(connect(connectfd, (struct sockaddr *)(&server_addr), sizeof(server_addr)) < 0)
    {
        fprintf(stderr,"connect error:%s\n", strerror(errno));
        return -1;
    }

    printf("connect success\n");


    char sendline[64] = "hello, i am xiaolin";

    //4. 发送数据
    int ret = send(connectfd, sendline, strlen(sendline), 0);
    if(ret != strlen(sendline)) {
        fprintf(stderr,"send data error:%s\n", strerror(errno));
        return -1;
    }

    printf("already send %d bytes\n", ret);

    sleep(1);

    //5. 关闭连接
    close(connectfd);
    return 0;
}
```

编译服务端和客户端的代码：

![在这里插入图片描述](https://img-blog.csdnimg.cn/291c6bdf93fa4e04b1606eef57d76836.png)




先启用服务端：

![在这里插入图片描述](https://img-blog.csdnimg.cn/a975aa542caf41b2a1f303563df697b7.png)


然后用 tcpdump 工具开始抓包，命令如下：

```bash
tcpdump -i lo tcp and port 8888 -s0 -w /home/tcp_close.pcap
```

然后启用客户端，可以看到，与服务端连接成功后，发完数据就退出了。

![在这里插入图片描述](https://img-blog.csdnimg.cn/8ea9f527a68a4c0184edc8842aaf55d6.png)


此时，服务端的输出：

![在这里插入图片描述](https://img-blog.csdnimg.cn/ff5f9ae91a3a4576b59e6e9c4716464d.png)


接下来，我们来看看抓包的结果。

![在这里插入图片描述](https://img-blog.csdnimg.cn/b542a2777aca4419b47205484b52cc03.png)


可以看到，TCP 挥手次数是 3 次。

所以，下面这个结论是没问题的。

> 结论：当被动关闭方（上图的服务端）在 TCP 挥手过程中，「**没有数据要发送」并且「开启了 TCP 延迟确认机制（默认会开启）」，那么第二和第三次挥手就会合并传输，这样就出现了三次挥手。**

#### 实验二

我们再做一次实验，来看看**关闭 TCP 延迟确认机制，会出现四次挥手吗？**

客户端代码保持不变，服务端代码需要增加一点东西。

在上面服务端代码中，增加了打开了 TCP_QUICKACK（快速应答）机制的代码，如下：

![在这里插入图片描述](https://img-blog.csdnimg.cn/fbbe19e6b1cc4a21b024588950b88eee.png)


编译好服务端代码后，就开始运行服务端和客户端的代码，同时用 tcpdump 进行抓包。

抓包的结果如下，可以看到是四次挥手。
![在这里插入图片描述](https://img-blog.csdnimg.cn/b6327b1057d64f54997c0eb322b28a55.png)


所以，当被动关闭方（上图的服务端）在 TCP 挥手过程中，「**没有数据要发送」，同时「关闭了 TCP 延迟确认机制」，那么就会是四次挥手。**

> 设置 TCP_QUICKACK 的代码，为什么要放在 read 返回 0 之后？


我也是多次实验才发现，在 bind 之前设置 TCP_QUICKACK 是不生效的，只有在 read 返回 0 的时候，设置 TCP_QUICKACK 才会出现四次挥手。

网上查了下资料说，设置 TCP_QUICKACK 并不是永久的，所以每次读取数据的时候，如果想要立刻回 ACK，那就得在每次读取数据之后，重新设置 TCP_QUICKACK。

而我这里的实验，目的是为了当收到客户端的 FIN 报文（第一次挥手）后，立马回 ACK 报文。所以就在 read 返回 0 的时候，设置 TCP_QUICKACK。当然，实际应用中，没人会在这个位置设置 TCP_QUICKACK，因为操作系统都通过 TCP 延迟确认机制帮我们把四次挥手优化成了三次挥手了。

## 总结

当被动关闭方在 TCP 挥手过程中，如果「没有数据要发送」，同时「没有开启 TCP_QUICKACK（默认情况就是没有开启，没有开启 TCP_QUICKACK，等于就是在使用 TCP 延迟确认机制）」，那么第二和第三次挥手就会合并传输，这样就出现了三次挥手。

**所以，出现三次挥手现象，是因为 TCP 延迟确认机制导致的。**

----

***哈喽，我是小林，就爱图解计算机基础，如果觉得文章对你有帮助，欢迎微信搜索「小林 coding」***

![img](https://cdn.xiaolincoding.com/gh/xiaolincoder/ImageHost3@main/%E5%85%B6%E4%BB%96/%E5%85%AC%E4%BC%97%E5%8F%B7%E4%BB%8B%E7%BB%8D.png)

