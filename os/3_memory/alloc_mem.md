# 4.4 在 4GB 物理内存的机器上，申请 8G 内存会怎么样？

大家好，我是小林。

看到读者在群里讨论这些面试题：

![](https://cdn.xiaolincoding.com/gh/xiaolincoder/ImageHost/操作系统/内存管理/读者提问.png)

其中，第一个问题「**在 4GB 物理内存的机器上，申请 8G 内存会怎么样？**」存在比较大的争议，有人说会申请失败，有的人说可以申请成功。

这个问题在没有前置条件下，就说出答案就是耍流氓。这个问题要考虑三个前置条件：

- 操作系统是 32 位的，还是 64 位的？
- 申请完 8G 内存后会不会被使用？
- 操作系统有没有使用 Swap 机制？

所以，我们要分场景讨论。

## 操作系统虚拟内存大小

应用程序通过 malloc 函数申请内存的时候，实际上申请的是虚拟内存，此时并不会分配物理内存。

当应用程序读写了这块虚拟内存，CPU 就会去访问这个虚拟内存，这时会发现这个虚拟内存没有映射到物理内存，CPU 就会产生**缺页中断**，进程会从用户态切换到内核态，并将缺页中断交给内核的 Page Fault Handler（缺页中断函数）处理。

缺页中断处理函数会看是否有空闲的物理内存：

- 如果有，就直接分配物理内存，并建立虚拟内存与物理内存之间的映射关系。
- 如果没有空闲的物理内存，那么内核就会开始进行[回收内存](https://xiaolincoding.com/os/3_memory/mem_reclaim.html)的工作，如果回收内存工作结束后，空闲的物理内存仍然无法满足此次物理内存的申请，那么内核就会放最后的大招了触发 OOM（Out of Memory）机制。

32 位操作系统和 64 位操作系统的虚拟地址空间大小是不同的，在 Linux 操作系统中，虚拟地址空间的内部又被分为**内核空间和用户空间**两部分，如下所示：

![](https://img-blog.csdnimg.cn/3a6cb4e3f27241d3b09b4766bb0b1124.png)

通过这里可以看出：

- `32` 位系统的内核空间占用 `1G`，位于最高处，剩下的 `3G` 是用户空间；
- `64` 位系统的内核空间和用户空间都是 `128T`，分别占据整个内存空间的最高和最低处，剩下的中间部分是未定义的。

### 32 位系统的场景

> 现在可以回答这个问题了：在 32 位操作系统、4GB 物理内存的机器上，申请 8GB 内存，会怎么样？

因为 32 位操作系统，进程最多只能申请 3 GB 大小的虚拟内存空间，所以进程申请 8GB 内存的话，在申请虚拟内存阶段就会失败（我手上没有 32 位操作系统测试，我估计失败的错误是 cannot allocate memory，也就是无法申请内存失败）。

### 64 位系统的场景

> 在 64 位操作系统、4GB 物理内存的机器上，申请 8G 内存，会怎么样？

64 位操作系统，进程可以使用 128 TB 大小的虚拟内存空间，所以进程申请 8GB 内存是没问题的，因为进程申请内存是申请虚拟内存，只要不读写这个虚拟内存，操作系统就不会分配物理内存。

我们可以简单做个测试，我的服务器是 64 位操作系统，但是物理内存只有 2 GB：

![](https://cdn.xiaolincoding.com/gh/xiaolincoder/ImageHost/操作系统/内存管理/2gb.png)

现在，我在机器上，连续申请 4 次 1 GB 内存，也就是一共申请了 4 GB 内存，注意下面代码只是单纯分配了虚拟内存，并没有使用该虚拟内存：

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <errno.h>

#define MEM_SIZE 1024 * 1024 * 1024

int main() {
    char* addr[4];
    int i = 0;
    for(i = 0; i < 4; ++i) {
        addr[i] = (char*) malloc(MEM_SIZE);
        if(!addr[i]) {
            printf("执行 malloc 失败，错误：%s\n",strerror(errno));
		        return -1;
        }
        printf("主线程调用 malloc 后，申请 1gb 大小得内存，此内存起始地址：0X%p\n", addr[i]);
    }
    
    //输入任意字符后，才结束
    getchar();
    return 0;
}
```

然后运行这个代码，可以看到，我的物理内存虽然只有 2GB，但是程序正常分配了 4GB 大小的虚拟内存：

![](https://cdn.xiaolincoding.com/gh/xiaolincoder/ImageHost/操作系统/内存管理/虚拟内存4g.png)

我们可以通过下面这条命令查看进程（test）的虚拟内存大小：

```shell
# ps aux | grep test
USER       PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
root      7797  0.0  0.0 4198540  352 pts/1    S+   16:58   0:00 ./test
```

其中，VSZ 就代表进程使用的虚拟内存大小，RSS 代表进程使用的物理内存大小。可以看到，VSZ 大小为 4198540，也就是 4GB 的虚拟内存。

> 之前有读者跟我反馈，说他自己也做了这个实验，然后发现 64 位操作系统，在申请 4GB 虚拟内存的时候失败了，这是为什么呢？

失败的错误：

![](https://cdn.xiaolincoding.com/gh/xiaolincoder/ImageHost/操作系统/内存管理/033读者-1.png)

我当时帮他排查了下，发现跟 Linux 中的 [overcommit_memory](http://linuxperf.com/?p=102) 参数有关，可以使用 `cat /proc/sys/vm/overcommit_memory` 来查看这个参数，这个参数接受三个值：

- 如果值为 0（默认值），代表：Heuristic overcommit handling，它允许 overcommit，但过于明目张胆的 overcommit 会被拒绝，比如 malloc 一次性申请的内存大小就超过了系统总内存。Heuristic 的意思是“试探式的”，内核利用某种算法猜测你的内存申请是否合理，大概可以理解为单次申请不能超过 free memory + free swap + pagecache 的大小 + SLAB 中可回收的部分，超过了就会拒绝 overcommit。
- 如果值为 1，代表：Always overcommit. 允许 overcommit，对内存申请来者不拒。
- 如果值为 2，代表：Don’t overcommit. 禁止 overcommit。

当时那位读者的 overcommit_memory 参数是默认值 0，所以申请失败的原因可能是内核认为我们申请的内存太大了，它认为不合理，所以 malloc() 返回了 Cannot allocate memory 错误，这里申请 4GB 虚拟内存失败的同学可以将这个 overcommit_memory 设置为 1，就可以 overcommit 了。

```shell
echo 1 > /proc/sys/vm/overcommit_memory 
```

设置完为 1 后，读者的机子就可以正常申请 4GB 虚拟内存了。

![](https://cdn.xiaolincoding.com/gh/xiaolincoder/ImageHost/操作系统/内存管理/033读者-2.png)

**不过我的环境 overcommit_memory 是 0，在 64 系统、2 G 物理内存场景下，也是可以成功申请 4 G 内存的，我怀疑可能是不同版本的内核在 overcommit_memory 为 0 时，检测内存申请是否合理的算法可能是不同的。**

**总之，如果你申请大内存的时候，不想被内核检测内存申请是否合理的算法干扰的话，将 overcommit_memory 设置为 1 就行。**

> 那么将这个 overcommit_memory 设置为 1 之后，64 位的主机就可以申请接近 128T 虚拟内存了吗？

不一定，还得看你服务器的物理内存大小。

读者的服务器物理内存是 2 GB，实验后发现，进程还没有申请到 128T 虚拟内存的时候就被杀死了。

![](https://cdn.xiaolincoding.com/gh/xiaolincoder/ImageHost/操作系统/内存管理/033读者-3.png)

注意，这次是 killed，而不是 Cannot Allocate Memory，说明并不是内存申请有问题，而是触发 OOM 了。

但是为什么会触发 OOM 呢？

那得看你的主机的「物理内存」够不够大了，即使 malloc 申请的是虚拟内存，只要不去访问就不会映射到物理内存，但是申请虚拟内存的过程中，还是使用到了物理内存（比如内核保存虚拟内存的数据结构，也是占用物理内存的），如果你的主机是只有 2GB 的物理内存的话，大概率会触发 OOM。

可以使用 top 命令，点击两下 m，通过进度条观察物理内存使用情况。

![](https://cdn.xiaolincoding.com/gh/xiaolincoder/ImageHost/操作系统/内存管理/033读者-4.png)

可以看到申请虚拟内存的过程中**物理内存使用量一直在增长**。

![](https://cdn.xiaolincoding.com/gh/xiaolincoder/ImageHost/操作系统/内存管理/033读者-5.png)

![](https://cdn.xiaolincoding.com/gh/xiaolincoder/ImageHost/操作系统/内存管理/033读者-6.png)

![](https://cdn.xiaolincoding.com/gh/xiaolincoder/ImageHost/操作系统/内存管理/033读者-7.png)

直到直接内存回收之后，也无法回收出一块空间供这个进程使用，这个时候就会触发 OOM，给所有能杀死的进程打分，分数越高的进程越容易被杀死。

在这里当然是这个进程得分最高，那么操作系统就会将这个进程杀死，所以最后会出现 killed，而不是 Cannot allocate memory。

> 那么 2GB 的物理内存的 64 位操作系统，就不能申请 128T 的虚拟内存了吗？

其实可以，上面的情况是还没开启 swap 的情况。

使用 swapfile 的方式开启了 1GB 的 swap 空间之后再做实验：

![](https://cdn.xiaolincoding.com/gh/xiaolincoder/ImageHost/操作系统/内存管理/033读者-8.png)

![](https://cdn.xiaolincoding.com/gh/xiaolincoder/ImageHost/操作系统/内存管理/033读者-9.png)

发现出现了 Cannot allocate memory，但是其实到这里已经成功了，

打开计算器计算一下，发现已经申请了 127.998T 虚拟内存了。

![](https://cdn.xiaolincoding.com/gh/xiaolincoder/ImageHost/操作系统/内存管理/033读者-10.png)

实际上我们是不可能申请完整个 128T 的用户空间的，因为程序运行本身也需要申请虚拟空间

申请 127T 虚拟内存试试：

![](https://cdn.xiaolincoding.com/gh/xiaolincoder/ImageHost/操作系统/内存管理/033读者-11.png)

发现进程没有被杀死，也没有 Cannot allocate memory，也正好是 127T 虚拟内存空间。

![](https://cdn.xiaolincoding.com/gh/xiaolincoder/ImageHost/操作系统/内存管理/033读者-12.png)

在 top 中我们可以看到这个申请了 127T 虚拟内存的进程。

![](https://cdn.xiaolincoding.com/gh/xiaolincoder/ImageHost/操作系统/内存管理/033读者-13.png)

## Swap 机制的作用

前面讨论在 32 位/64 位操作系统环境下，申请的虚拟内存超过物理内存后会怎么样？

- 在 32 位操作系统，因为进程最大只能申请 3 GB 大小的虚拟内存，所以直接申请 8G 内存，会申请失败。
- 在 64 位操作系统，因为进程最大只能申请 128 TB 大小的虚拟内存，即使物理内存只有 4GB，申请 8G 内存也是没问题，因为申请的内存是虚拟内存。

程序申请的虚拟内存，如果没有被使用，它是不会占用物理空间的。当访问这块虚拟内存后，操作系统才会进行物理内存分配。

如果申请物理内存大小超过了空闲物理内存大小，就要看操作系统有没有开启 Swap 机制：

- 如果没有开启 Swap 机制，程序就会直接 OOM；
- 如果有开启 Swap 机制，程序可以正常运行。

> 什么是 Swap 机制？

当系统的物理内存不够用的时候，就需要将物理内存中的一部分空间释放出来，以供当前运行的程序使用。那些被释放的空间可能来自一些很长时间没有什么操作的程序，这些被释放的空间会被临时保存到磁盘，等到那些程序要运行时，再从磁盘中恢复保存的数据到内存中。

另外，当内存使用存在压力的时候，会开始触发内存回收行为，会把这些不常访问的内存先写到磁盘中，然后释放这些内存，给其他更需要的进程使用。再次访问这些内存时，重新从磁盘读入内存就可以了。

这种，将内存数据换出磁盘，又从磁盘中恢复数据到内存的过程，就是 Swap 机制负责的。

Swap 就是把一块磁盘空间或者本地文件，当成内存来使用，它包含换出和换入两个过程：

- **换出（Swap Out）** ，是把进程暂时不用的内存数据存储到磁盘中，并释放这些数据占用的内存；
- **换入（Swap In）**，是在进程再次访问这些内存的时候，把它们从磁盘读到内存中来；

Swap 换入换出的过程如下图：

![](https://img-blog.csdnimg.cn/388a29f45fe947e5a49240e4eff13538.png)

使用 Swap 机制优点是，应用程序实际可以使用的内存空间将远远超过系统的物理内存。由于硬盘空间的价格远比内存要低，因此这种方式无疑是经济实惠的。当然，频繁地读写硬盘，会显著降低操作系统的运行速率，这也是 Swap 的弊端。

Linux 中的 Swap 机制会在内存不足和内存闲置的场景下触发：

- **内存不足**：当系统需要的内存超过了可用的物理内存时，内核会将内存中不常使用的内存页交换到磁盘上为当前进程让出内存，保证正在执行的进程的可用性，这个内存回收的过程是强制的直接内存回收（Direct Page Reclaim）。直接内存回收是同步的过程，会阻塞当前申请内存的进程。
- **内存闲置**：应用程序在启动阶段使用的大量内存在启动后往往都不会使用，通过后台运行的守护进程（kSwapd），我们可以将这部分只使用一次的内存交换到磁盘上为其他内存的申请预留空间。kSwapd 是 Linux 负责页面置换（Page replacement）的守护进程，它也是负责交换闲置内存的主要进程，它会在[空闲内存低于一定水位](https://xiaolincoding.com/os/3_memory/mem_reclaim.html#%E5%B0%BD%E6%97%A9%E8%A7%A6%E5%8F%91-kSwapd-%E5%86%85%E6%A0%B8%E7%BA%BF%E7%A8%8B%E5%BC%82%E6%AD%A5%E5%9B%9E%E6%94%B6%E5%86%85%E5%AD%98)时，回收内存页中的空闲内存保证系统中的其他进程可以尽快获得申请的内存。kSwapd 是后台进程，所以回收内存的过程是异步的，不会阻塞当前申请内存的进程。

Linux 提供了两种不同的方法启用 Swap，分别是 Swap 分区（Swap Partition）和 Swap 文件（Swapfile），开启方法可以看[这个资料](https://support.huaweicloud.com/trouble-ecs/ecs_trouble_0322.html)：

- Swap 分区是硬盘上的独立区域，该区域只会用于交换分区，其他的文件不能存储在该区域上，我们可以使用 `swapon -s` 命令查看当前系统上的交换分区；
- Swap 文件是文件系统中的特殊文件，它与文件系统中的其他文件也没有太多的区别；

> Swap 换入换出的是什么类型的内存？

内核缓存的文件数据，因为都有对应的磁盘文件，所以在回收文件数据的时候，直接写回到对应的文件就可以了。

但是像进程的堆、栈数据等，它们是没有实际载体，这部分内存被称为匿名页。而且这部分内存很可能还要再次被访问，所以不能直接释放内存，于是就需要有一个能保存匿名页的磁盘载体，这个载体就是 Swap 分区。

匿名页回收的方式是通过 Linux 的 Swap 机制，Swap 会把不常访问的内存先写到磁盘中，然后释放这些内存，给其他更需要的进程使用。再次访问这些内存时，重新从磁盘读入内存就可以了。

接下来，通过两个实验，看看申请的物理内存超过物理内存会怎样？

- 实验一：没有开启 Swap 机制
- 实验二：有开启 Swap 机制

### 实验一：没有开启 Swap 机制

我的服务器是 64 位操作系统，但是物理内存只有 2 GB，而且没有 Swap 分区：

![](https://cdn.xiaolincoding.com/gh/xiaolincoder/ImageHost/操作系统/内存管理/2gb.png)

我们改一下前面的代码，使得在申请完 4GB 虚拟内存后，通过 memset 函数访问这个虚拟内存，看看在没有 Swap 分区的情况下，会发生什么？

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <errno.h>

#define MEM_SIZE 1024 * 1024 * 1024

int main() {
    char* addr[4];
    int i = 0;
    for(i = 0; i < 4; ++i) {
        addr[i] = (char*) malloc(MEM_SIZE);
        if(!addr[i]) {
            printf("执行 malloc 失败，错误：%s\n",strerror(errno));
            return -1;
        }
        printf("主线程调用 malloc 后，申请 1gb 大小得内存，此内存起始地址：0X%p\n", addr[i]);
    }

    for(i = 0; i < 4; ++i) {
        printf("开始访问第 %d 块虚拟内存 (每一块虚拟内存为 1 GB)\n", i + 1);
        memset(addr[i], 0, MEM_SIZE);
    }
    
    //输入任意字符后，才结束
    getchar();
    return 0;
}
```

运行结果：

![](https://cdn.xiaolincoding.com/gh/xiaolincoder/ImageHost/操作系统/内存管理/发生oom.png)

可以看到，在访问第 2 块虚拟内存（每一块虚拟内存是 1 GB）的时候，因为超过了机器的物理内存（2GB），进程（test）被操作系统杀掉了。

通过查看 message 系统日志，可以发现该进程是被操作系统 OOM killer 机制杀掉了，日志里报错了 Out of memory，也就是发生 OOM（内存溢出错误）。

![](https://cdn.xiaolincoding.com/gh/xiaolincoder/ImageHost/操作系统/内存管理/oom日志.png)

> 什么是 OOM?

内存溢出 (Out Of Memory，简称 OOM) 是指应用系统中存在无法回收的内存或使用的内存过多，最终使得程序运行要用到的内存大于能提供的最大内存。此时程序就运行不了，系统会提示内存溢出。

### 实验二：有开启 Swap 机制

我用我的 mac book pro 笔记本做测试，我的笔记本是 64 位操作系统，物理内存是 8 GB，目前 Swap 分区大小为 1 GB（注意这个大小不是固定不变的，Swap 分区总大小是会动态变化的，当没有使用 Swap 分区时，Swap 分区总大小是 0；当使用了 Swap 分区，Swap 分区总大小会增加至 1 GB；当 Swap 分区已使用的大小超过 1 GB 时；Swap 分区总大小就会增加到至 2 GB；当 Swap 分区已使用的大小超过 2 GB 时；Swap 分区总大小就增加至 3GB，如此往复。这个估计是 macos 自己实现的，Linux 的分区则是固定大小的，Swap 分区不会根据使用情况而自动增长）。

![](https://cdn.xiaolincoding.com/gh/xiaolincoder/ImageHost/操作系统/内存管理/swap分区大小.png)

为了方便观察磁盘 I/O 情况，我们改进一下前面的代码，分配完 32 GB 虚拟内存后（笔记本物理内存是 8 GB），通过一个 while 循环频繁访问虚拟内存，代码如下：

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

#define MEM_SIZE 32 * 1024 * 1024 * 1024

int main() {
    char* addr = (char*) malloc((long)MEM_SIZE);
    printf("主线程调用 malloc 后，目前共申请了 32gb 的虚拟内存\n");
    
    //循环频繁访问虚拟内存
    while(1) {
          printf("开始访问 32gb 大小的虚拟内存...\n");
          memset(addr, 0, (long)MEM_SIZE);
    }
    return 0;
}
```

运行结果如下：

![](https://cdn.xiaolincoding.com/gh/xiaolincoder/ImageHost/操作系统/内存管理/代码3运行结果.png)

可以看到，在有 Swap 分区的情况下，即使笔记本物理内存是 8 GB，申请并使用 32 GB 内存是没问题，程序正常运行了，并没有发生 OOM。

从下图可以看到，进程的内存显示 32 GB（这个不要理解为占用的物理内存，理解为已被访问的虚拟内存大小，也就是在物理内存呆过的内存大小），系统已使用的 Swap 分区达到 2.3 GB。

![](https://cdn.xiaolincoding.com/gh/xiaolincoder/ImageHost/操作系统/内存管理/test进程内存情况.png)

此时我的笔记本电脑的磁盘开始出现“沙沙”的声音，通过查看磁盘的 I/O 情况，可以看到磁盘 I/O 达到了一个峰值，非常高：

![](https://cdn.xiaolincoding.com/gh/xiaolincoder/ImageHost/操作系统/内存管理/磁盘io.png)

> 有了 Swap 分区，是不是意味着进程可以使用的内存是无上限的？

当然不是，我把上面的代码改成了申请 64GB 内存后，当进程申请完 64GB 虚拟内存后，使用到 56 GB（这个不要理解为占用的物理内存，理解为已被访问的虚拟内存大小，也就是在物理内存呆过的内存大小）的时候，进程就被系统 kill 掉了，如下图：

![](https://cdn.xiaolincoding.com/gh/xiaolincoder/ImageHost/操作系统/内存管理/被kill掉.png)

当系统多次尝试回收内存，还是无法满足所需使用的内存大小，进程就会被系统 kill 掉了，意味着发生了 OOM（*PS：我没有在 macos 系统找到像 linux 系统里的 /var/log/message 系统日志文件，所以无法通过查看日志确认是否发生了 OOM*）。

## 总结

至此，验证完成了。简单总结下：

- 在 32 位操作系统，因为进程理论上最大能申请 3 GB 大小的虚拟内存，所以直接申请 8G 内存，会申请失败。
- 在 64 位 位操作系统，因为进程理论上最大能申请 128 TB 大小的虚拟内存，即使物理内存只有 4GB，申请 8G 内存也是没问题，因为申请的内存是虚拟内存。如果这块虚拟内存被访问了，要看系统有没有 Swap 分区：
  - 如果没有 Swap 分区，因为物理空间不够，进程会被操作系统杀掉，原因是 OOM（内存溢出）；
  - 如果有 Swap 分区，即使物理内存只有 4GB，程序也能正常使用 8GB 的内存，进程可以正常运行；

---

***哈喽，我是小林，就爱图解计算机基础，如果觉得文章对你有帮助，欢迎微信搜索「小林 coding」，关注后，回复「网络」再送你图解网络 PDF***

![](https://cdn.xiaolincoding.com/gh/xiaolincoder/ImageHost3@main/%E5%85%B6%E4%BB%96/%E5%85%AC%E4%BC%97%E5%8F%B7%E4%BB%8B%E7%BB%8D.png)



