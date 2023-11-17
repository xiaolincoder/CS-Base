# 10.2  如何从日志分析 PV、UV？



很多时候，我们观察程序是否如期运行，或者是否有错误，最直接的方式就是看运行**日志**，当然要想从日志快速查到我们想要的信息，前提是程序打印的日志要精炼、精准。

但日志涵盖的信息远不止于此，比如对于 nginx 的 access.log 日志，我们可以根据日志信息**分析用户行为**。

什么用户行为呢？比如分析出哪个页面访问次数（*PV*）最多，访问人数（*UV*）最多，以及哪天访问量最多，哪个请求访问最多等等。

这次，将用一个大概几万条记录的 nginx 日志文件作为案例，一起来看看如何分析出「用户信息」。

![](https://cdn.jsdelivr.net/gh/xiaolincoder/ImageHost4@main/网络/log/提纲日志.png)



---


## 别急着开始

当我们要分析日志的时候，先用 `ls -lh` 命令查看日志文件的大小，如果日志文件大小非常大，最好不要在线上环境做。

比如我下面这个日志就 6.5M，不算大，在线上环境分析问题不大。

![](https://cdn.jsdelivr.net/gh/xiaolincoder/ImageHost4@main/网络/log/ls.png)

如果日志文件数据量太大，你直接一个 `cat` 命令一执行，是会影响线上环境，加重服务器的负载，严重的话，可能导致服务器无响应。

当发现日志很大的时候，我们可以使用 `scp` 命令将文件传输到闲置的服务器再分析，scp 命令使用方式如下图：

![](https://cdn.jsdelivr.net/gh/xiaolincoder/ImageHost4@main/网络/log/scp.png)


---

## 慎用 cat

大家都知道 `cat` 命令是用来查看文件内容的，但是日志文件数据量有多少，它就读多少，很显然不适用大文件。

对于大文件，我们应该养成好习惯，用 `less` 命令去读文件里的内容，因为 less 并不会加载整个文件，而是按需加载，先是输出一小页的内容，当你要往下看的时候，才会继续加载。



![](https://cdn.jsdelivr.net/gh/xiaolincoder/ImageHost4@main/网络/log/less.png)



可以发现，nginx 的 access.log 日志每一行是一次用户访问的记录，从左到右分别包含如下信息：

- 客户端的 IP 地址；
- 访问时间；
- HTTP 请求的方法、路径、协议版本、协议版本、返回的状态码；
- User Agent，一般是客户端使用的操作系统以及版本、浏览器及版本等；


不过，有时候我们想看日志最新部分的内容，可以使用 `tail` 命令，比如当你想查看倒数 5 行的内容，你可以使用这样的命令：


![](https://cdn.jsdelivr.net/gh/xiaolincoder/ImageHost4@main/网络/log/tail-n.png)



如果你想实时看日志打印的内容，你可以使用 `tail -f` 命令，这样你看日志的时候，就会是阻塞状态，有新日志输出的时候，就会实时显示出来。


---

## PV  分析


PV 的全称叫 *Page View*，用户访问一个页面就是一次 PV，比如大多数博客平台，点击一次页面，阅读量就加 1，所以说 PV 的数量并不代表真实的用户数量，只是个点击量。

对于 nginx 的 `acess.log` 日志文件来说，分析 PV 还是比较容易的，既然日志里的内容是访问记录，那有多少条日志记录就有多少 PV。

我们直接使用 `wc -l` 命令，就可以查看整体的 PV 了，如下图一共有 49903 条 PV。

![](https://cdn.jsdelivr.net/gh/xiaolincoder/ImageHost4@main/网络/log/wc.png)




---

## PV 分组

nginx 的 `acess.log` 日志文件有访问时间的信息，因此我们可以根据访问时间进行分组，比如按天分组，查看每天的总 PV，这样可以得到更加直观的数据。

要按时间分组，首先我们先「访问时间」过滤出来，这里可以使用 awk 命令来处理，awk 是一个处理文本的利器。

awk 命令默认是以「空格」为分隔符，由于访问时间在日志里的第 4 列，因此可以使用 `awk '{print $4}' access.log` 命令把访问时间的信息过滤出来，结果如下：

![](https://cdn.jsdelivr.net/gh/xiaolincoder/ImageHost4@main/网络/log/awk日期.png)





上面的信息还包含了时分秒，如果只想显示年月日的信息，可以使用 `awk` 的 `substr` 函数，从第 2 个字符开始，截取 11 个字符。

![](https://cdn.jsdelivr.net/gh/xiaolincoder/ImageHost4@main/网络/log/awk日期2.png)




接着，我们可以使用 `sort` 对日期进行排序，然后使用 `uniq -c` 进行统计，于是按天分组的 PV 就出来了。

可以看到，每天的 PV 量大概在 2000-2800：


![](https://cdn.jsdelivr.net/gh/xiaolincoder/ImageHost4@main/网络/log/awkpv.png)


注意，**使用 `uniq -c` 命令前，先要进行 `sort` 排序**，因为 uniq 去重的原理是比较相邻的行，然后除去第二行和该行的后续副本，因此在使用 uniq 命令之前，请使用 sort 命令使所有重复行相邻。

---

## UV 分析

UV 的全称是 *Uniq Visitor*，它代表访问人数，比如公众号的阅读量就是以 UV 统计的，不管单个用户点击了多少次，最终只算 1 次阅读量。

access.log 日志里虽然没有用户的身份信息，但是我们可以用「客户端 IP 地址」来**近似统计** UV。


![](https://cdn.jsdelivr.net/gh/xiaolincoder/ImageHost4@main/网络/log/uv.png)

该命令的输出结果是 2589，也就说明 UV 的量为 2589。上图中，从左到右的命令意思如下：

- `awk '{print $1}' access.log`，取日志的第 1 列内容，客户端的 IP 地址正是第 1 列；
- `sort`，对信息排序；
- `uniq`，去除重复的记录；
- `wc -l`，查看记录条数；

---

## UV 分组

假设我们按天来分组分析每天的 UV 数量，这种情况就稍微比较复杂，需要比较多的命令来实现。

既然要按天统计 UV，那就得把「日期 + IP 地址」过滤出来，并去重，命令如下：


![](https://cdn.jsdelivr.net/gh/xiaolincoder/ImageHost4@main/网络/log/uv分组.png)


具体分析如下：

- 第一次 `ack` 是将第 4 列的日期和第 1 列的客户端 IP 地址过滤出来，并用空格拼接起来；
- 然后 `sort` 对第一次 ack 输出的内容进行排序；
- 接着用 `uniq` 去除重复的记录，也就说日期 +IP 相同的行就只保留一个；

上面只是把 UV 的数据列了出来，但是并没有统计出次数。

如果需要对当天的 UV 统计，在上面的命令再拼接 `awk '{uv[$1]++;next}END{for (ip in uv) print ip, uv[ip]}'` 命令就可以了，结果如下图：

![](https://cdn.jsdelivr.net/gh/xiaolincoder/ImageHost4@main/网络/log/awknext.png)

awk 本身是「逐行」进行处理的，当执行完一行后，我们可以用 `next` 关键字来告诉 awk 跳转到下一行，把下一行作为输入。

对每一行输入，awk 会根据第 1 列的字符串（也就是日期）进行累加，这样相同日期的 ip 地址，就会累加起来，作为当天的 uv 数量。

之后的 `END` 关键字代表一个触发器，就是当前面的输入全部完成后，才会执行 END {} 中的语句，END 的语句是通过 foreach 遍历 uv 中所有的 key，打印出按天分组的 uv 数量。

---

## 终端分析

nginx 的 access.log 日志最末尾关于 User Agent 的信息，主要是客户端访问服务器使用的工具，可能是手机、浏览器等。

因此，我们可以利用这一信息来分析有哪些终端访问了服务器。

User Agent 的信息在日志里的第 12 列，因此我们先使用 `awk` 过滤出第 12 列的内容后，进行 `sort` 排序，再用 `uniq -c` 去重并统计，最后再使用 `sort -rn`（*r 表示逆向排序，n 表示按数值排序*）对统计的结果排序，结果如下图：


![](https://cdn.jsdelivr.net/gh/xiaolincoder/ImageHost4@main/网络/log/terminal.png)

---

## 分析 TOP3 的请求

access.log 日志中，第 7 列是客户端请求的路径，先使用 `awk` 过滤出第 7 列的内容后，进行 `sort` 排序，再用 `uniq -c` 去重并统计，然后再使用 `sort -rn` 对统计的结果排序，最后使用 `head -n 3` 分析 TOP3 的请求，结果如下图：


![](https://cdn.jsdelivr.net/gh/xiaolincoder/ImageHost4@main/网络/log/TOP3.png)

---

## 关注作者

***哈喽，我是小林，就爱图解计算机基础，如果觉得文章对你有帮助，欢迎微信搜索「小林 coding」，关注后，回复「网络」再送你图解网络 PDF***

![](https://cdn.jsdelivr.net/gh/xiaolincoder/ImageHost3@main/其他/公众号介绍.png)