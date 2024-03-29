# Socket 编程和 IO 复用

## TCP 与 UDP 编程流程与接口

> socket编程 ----使用 TCP 协议的流程图 - 李点点的文章 - 知乎 https://zhuanlan.zhihu.com/p/466675391
> 
> https://www.mkssoftware.com/docs/man3/recvfrom.3.asp

+ 基于 TCP 的：
  
  TCP 是传送字节流的服务，所以通常使用 read, **recv**, write, **send**。不过使用 sendto, recvfrom 也不是不可以（但是也不能通过 recvfrom 获取地址，因为 connect 的时候已经绑过了，在实现上，就没有给出）
  
  > listen() 函数：主要作用就是**告知内核**将套接字 (sockfd) 变成被动的连接监听套接字（被动等待客户端的连接），可以通过参数设置内核中连接队列的长度，**TCP 三次握手也不是由这个函数完成，listen() 的作用仅仅告诉内核一些信息。**listen()函数不会阻塞，当有一个客户端主动连接（connect），Linux **内核就自动完成 TCP 三次握手**，将建立好的链接自动存储到**队列**中，如此重复。
  > 
  > accept() 函数功能是，从处于 established 状态的连接队列头部取出一个已经完成的连接，如果这个队列没有已经完成的连接，accept() 函数就会阻塞，直到取出队列中已完成的用户连接为止。
  
  > https://blog.csdn.net/tennysonsky/article/details/45621341/
  > 
  > https://stackoverflow.com/questions/26909982/using-write-read-on-udp-socket
  
  ![img](70.jpeg)

+ 基于 UDP 的
  
  ![img](v2-421882dccd0fdd11372647311fbb37d6_b.jpg)

## Unix 五大网络 IO 模型

一个输入操作通常包括两个阶段：

- 等待数据准备好：对于一个 Socket 上的输入操作，第一步通常涉及等待数据从网络中到达，当所等待数据到达时，它被复制到内核中的某个缓冲区。
- 从内核向进程复制数据：把数据从内核缓冲区复制到应用进程缓冲区。

<img src="image-20220215100951012.png" alt="image-20220215100951012" style="zoom: 50%;" />

### 阻塞式 IO

**应用进程被阻塞**，直到数据从内核缓冲区复制到应用进程缓冲区中才返回。在阻塞的过程中，**其它应用进程还可以执行**，因此阻塞**不意味着整个操作系统都被阻塞**。(因为其它应用进程还可以执行，阻塞的进程不消耗 CPU 时间，这种模型的 CPU 利用率会比较高。)

例如，阻塞式 socket 的数据接收函数

```c
ssize_t recvfrom(int sockfd, void *buf, size_t len, int flags, struct sockaddr *src_addr, socklen_t *addrlen);
```

<img src="image-20220214211331834.png" alt="image-20220214211331834" style="zoom:50%;" />

### 非阻塞 IO

应用进程执行系统调用之后，内核返回一个错误码 EWOULDBLOCK error 。应用进程可以继续执行，

用户进程需要不断的主动询问 kernel 数据好了没有，这种方式称为轮询（polling）。

由于 CPU 要处理更多的系统调用，因此这种模型的 CPU 利用率比较低。

<img src="image-20220214213019747.png" alt="image-20220214213019747" style="zoom:50%;" />

> socket() 函数创建的套接字默认是阻塞的，可通过设置参数 SOCK_NONBLOCK 使其变为非阻塞：
> 
> https://man7.org/linux/man-pages/man2/socket.2.html
> 
> ```text
> int s = socket(AF_INET, SOCK_STREAM | SOCK_NONBLOCK, IPPROTO_TCP);  
> ```

这玩意干吗用的？

1. read 的时候不等到指定的字节全部读完才返回，而是把缓冲区里的数据立即返回。
   
   如果缓冲区为空，也立刻返回，值为 -1，不会阻塞（比如你轮询键盘鼠标的输入比较有用？）

2. write 时，可能缓冲区不够放下所有数据，那就能写多少写多少，返回你实际写的字节数

![](https://img-blog.csdnimg.cn/img_convert/ade7cc7e016d25b699ffeaa77e918eef.png)

### IO 复用

> IO多路复用到底是不是异步的？ - 闪客sun的回答 - 知乎 https://www.zhihu.com/question/59975081/answer/1932776593

IO 复用方式下，单个 process (thread) 可以同时处理多个网络连接的 IO。使用 select，poll，epoll 等调用，不断的轮询相关的的所有 socket，当某个 socket 有数据到达了，就通知用户进程。

如果一个 Web 服务器没有 I/O 复用，那么每一个 Socket 连接都需要创建一个线程去处理。如果同时有几万个连接，那么就需要创建相同数量的线程。相比于多进程和多线程技术，I/O 多路复用技术的最大优势是系统开销小，系统不必创建进程/线程，也不必维护这些进程/线程。IO 复用的优势并不是单个连接能处理得更快，而是在于能处理更多的连接。

在IO multiplexing Model中，实际中，对于每一个socket，一般都设置成为 non-blocking，但是，整个用户的 process (thread) 其实是一直被阻塞的。只不过是被 select 阻塞，而不是被 socket IO 阻塞。

![image-20220215095847364](image-20220215095847364.png)

### 信号驱动方式

为 IO 准备好这样一个信号 SIGIO，注册一个处理函数。（比如通过  sigaction 系统调用）。注册后内核立即返回，应用进程可以继续执行，也就是说等待数据阶段应用进程是非阻塞的。

内核在数据到达时向应用进程发送 SIGIO 信号，应用进程收到之后在信号处理程序中再读数据，比如用 recvfrom。

相比于非阻塞式 I/O 的轮询方式，信号驱动 I/O 的 CPU 利用率更高。

<img src="image-20220214213919420.png" alt="image-20220214213919420" style="zoom:50%;" />

> https://www.itzhai.com/articles/it-seems-not-so-perfect-signal-driven-io.html
> 
> 信号驱动式 IO 对于 TCP 套接字产生的作用不大。因为该信号在 TCP 套接字中产生的过于频繁。进程对该信号一头雾水，没法确定套接字具体发生了什么事情。不过**可以对 TCP 监听套接字可以使用 SIGIO**，因为对于监听套接字，产生SIGIO信号的唯一条件是某个新连接完成了。这样就可以在SIGIO信号处理函数中获取新连接了。
> 
> 在UDP套接字中，只有 数据报到达套接字 和 套接字上发生异步错误 会产生SIGIO信号，只需要相应处理即可

### 异步 IO

应用进程执行 aio_read 系统调用会立即返回，应用进程可以继续执行，不会被阻塞，内核会在**所有操作完成之后**向应用进程发送信号。

异步 I/O 与信号驱动 I/O 的区别在于，**异步 I/O 的信号是通知应用进程 I/O 完成**，而**信号驱动 I/O 的信号是通知应用进程可以开始 I/O**。

> https://man7.org/linux/man-pages/man3/aio_read.3.html
> 
> https://man7.org/linux/man-pages/man7/aio.7.html

```c
int aio_read(struct aiocb *aiocbp);
struct aiocb {
    /* The order of these fields is implementation-dependent */

    int             aio_fildes;     /* File descriptor */
    off_t           aio_offset;     /* File offset */
    volatile void  *aio_buf;        /* Location of buffer */
    size_t          aio_nbytes;     /* Length of transfer */
    int             aio_reqprio;    /* Request priority */
    struct sigevent aio_sigevent;   /* Notification method */
    int             aio_lio_opcode; /* Operation to be performed;
                                                  lio_listio() only */

    /* Various implementation-internal fields not shown */
};
```

有一点像访问外存的 DMA 方式，把这件事委托出去，存好了再告诉我。

之前的信号驱动有点像访问外存的中断方式，准备好了告诉我，然后我读出来处理。

<img src="image-20220214214846236.png" alt="image-20220214214846236" style="zoom:50%;" />

## IO 复用的几个接口

select/poll/epoll

> IO 多路复用是什么意思？ - 小林coding的回答 - 知乎 https://www.zhihu.com/question/32163005/answer/1802684879
> 
> 下面所提到的 sockfd 是针对网络 IO 的，其实对于普通文件的文件描述符 file descriptor，也是一样的。
> 
> 一些代码示例：彻底理解 IO多路复用 - caspar的文章 - 知乎 https://zhuanlan.zhihu.com/p/150972878

### select

select 实现多路复用的方式是，将已连接的 Socket 都放到一个**sockfd集合**，然后调用 select 函数将 sockfd 集合**拷贝**到内核里，让内核来检查是否有网络事件产生，检查的方式很粗暴，就是通过**遍历** sockfd 集合的方式，当检查到有事件产生后，将此 Socket 标记为可读或可写， 接着再把整个 sockfd 集合**拷贝**回用户态里，然后用户态还需要再通过**遍历**的方法找到可读或可写的 Socket，然后再对其处理。

所以，对于 select 这种方式，需要**多次「遍历」sockfd 集合**，一个是在内核态里，一个次是在用户态里 ，而且还会发生 **2 次「拷贝」sockfd集合**，先从用户空间传入内核空间，由内核修改后，再传出到用户空间中。select 使用固定长度的 BitsMap 表示 sockfd 集合，而且所支持的文件描述符的个数是有限制的，**在 Linux 系统中， 默认最大值为 `1024`**。

```c
int select(int n, fd_set *readfds, fd_set *writefds, fd_set *exceptfds, struct timeval *timeout);
```

### poll

相比于 select，poll 不再用 BitsMap 来存储所关注的 sockfd，取而代之用动态数组，**以链表形式来组织，突破了 select 的 sockfd 个数限制**。

但是 poll 和 select 并没有太大的本质区别，**都是使用「线性结构」存储进程关注的 Socket 集合，因此都需要遍历文件描述符集合来找到可读或可写的 Socket，时间复杂度为 O(n)，而且也需要在用户态与内核态之间拷贝 sockfd 集合**，这种方式随着并发数上来，性能的损耗会呈指数级增长。

```c
int poll(struct pollfd *fds, unsigned int nfds, int timeout);
struct pollfd {
    int   fd;         /* file descriptor */
    short events;     /* requested events */
    short revents;    /* returned events */
};

// The structure for two events
struct pollfd fds[2];

// Monitor sock1 for input
fds[0].fd = sock1;
fds[0].events = POLLIN;

// Monitor sock2 for output
fds[1].fd = sock2;
fds[1].events = POLLOUT;

// Wait 10 seconds
int ret = poll( &fds, 2, 10000 );
// Check if poll actually succeed
if ( ret == -1 )
    // report error and abort
else if ( ret == 0 )
    // timeout; no event detected
else
{
    // If we detect the event, zero it out so we can reuse the structure
    if ( fds[0].revents & POLLIN )
        fds[0].revents = 0;
        // input event on sock1

    if ( fds[1].revents & POLLOUT )
        fds[1].revents = 0;
        // output event on sock2
}
```

### epoll

> 注:epoll 没有共享内存 https://www.zhihu.com/question/39792257

epoll 通过两个方面，很好解决了 select/poll 的问题。

+ epoll 在**内核里**使用**红黑树来跟踪进程所有待检测的 sockfd**，把需要监控的 socket 通过 `epoll_ctl()` 函数加入内核中的红黑树里。红黑树是个高效的数据结构，增删查一般时间复杂度是 `O(logn)`，通过对这棵黑红树进行操作，这样就不需要像 select/poll 每次操作时都传入整个 socket 集合，只需要传入一个待检测的 socket，减少了内核和用户空间大量的数据拷贝和内存分配。
+ epoll 使用事件驱动的机制，内核里**维护了一个链表来记录就绪事件**，当某个 socket 有事件发生时，内核会将其加入到这个就绪事件列表中。内核不再通过轮询的方式找到就绪的文件描述符，而是通过异步 IO 事件唤醒。
+ 当用户调用 `epoll_wait()` 函数时，只会从就绪列表中取出有事件发生的 sockfd，不需要像 select/poll 那样轮询扫描整个 socket 集合，大大提高了检测的效率。

![image-20220215102741802](image-20220215102741802.png)

epoll 的两种工作模式

+ 边缘触发（edge-triggered，ET）：当被监控的 Socket 描述符上有可读事件发生时，只会触发一次事件。一旦调用了 epoll_wait ，该 socket 就不在就绪事件列表里了。即使进程没有调用 read 函数从内核读取数据，也依然不会再次触发事件，除非有新的数据到达，因此我们程序要保证一次性将内核缓冲区的数据读取完；
+ 水平触发（level-triggered，LT）：当被监控的 Socket 上有可读事件发生时，用户程序总是可以从 epoll_wait 中获取到这个 socket，直到内核缓冲区数据被 read 函数读完才结束，目的是告诉我们有数据需要读取。

```c
int epoll_create(int size);
int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event)；
int epoll_wait(int epfd, struct epoll_event * events, int maxevents, int timeout);

// Create the epoll descriptor. Only one is needed per app, and is used to monitor all sockets.
// The function argument is ignored (it was not before, but now it is), so put your favorite number here
int pollingfd = epoll_create( 0xCAFE );

if ( pollingfd < 0 )
 // report error

// Initialize the epoll structure in case more members are added in future
struct epoll_event ev = { 0 };

// Associate the connection class instance with the event. You can associate anything
// you want, epoll does not use this information. We store a connection class pointer, pConnection1
ev.data.ptr = pConnection1;

// Monitor for input, and do not automatically rearm the descriptor after the event
ev.events = EPOLLIN | EPOLLONESHOT;
// Add the descriptor into the monitoring list. We can do it even if another thread is
// waiting in epoll_wait - the descriptor will be properly added
if ( epoll_ctl( epollfd, EPOLL_CTL_ADD, pConnection1->getSocket(), &ev ) != 0 )
    // report error

// Wait for up to 20 events (assuming we have added maybe 200 sockets before that it may happen)
struct epoll_event pevents[ 20 ];

// Wait for 10 seconds, and retrieve less than 20 epoll_event and store them into epoll_event array
int ready = epoll_wait( pollingfd, pevents, 20, 10000 );
// Check if epoll actually succeed
if ( ret == -1 )
    // report error and abort
else if ( ret == 0 )
    // timeout; no event detected
else
{
    // Check if any events detected
    for ( int i = 0; i < ready; i++ )
    {
        if ( pevents[i].events & EPOLLIN )
        {
            // Get back our connection pointer
            Connection * c = (Connection*) pevents[i].data.ptr;
            c->handleReadEvent();
         }
    }
}
```

### 他们的应用场景

很容易产生一种错觉认为只要用 epoll 就可以了，select 和 poll 都已经过时了，其实它们都有各自的使用场景。

#### 1. select 应用场景

select 的 timeout 参数精度为微秒，而 poll 和 epoll 为毫秒，**因此 select 更加适用于实时性要求比较高的场景**，比如核反应堆的控制。

select **可移植性更好**，几乎被所有主流平台所支持。

#### 2. poll 应用场景

poll **没有最大描述符数量的限制**，如果平台支持并且对实时性要求不高，应该使用 poll 而不是 select。

#### 3. epoll 应用场景

只需要运行在 Linux 平台上，**有大量的描述符需要同时轮询，并且这些连接最好是长连接**。

如果需要同时监控小于 1000 个描述符，就没有必要使用 epoll，因为这个应用场景下并不能体现 epoll 的优势。

如果需要监控的描述符状态变化多，而且都是非常短暂的，也没有必要使用 epoll。因为 epoll 中的所有描述符都存储在内核中，造成每次需要对描述符的状态改变都需要通过 epoll_ctl() 进行系统调用，频繁系统调用降低效率。并且 epoll 的描述符存储在内核，不容易调试。

### epoll 的简要工作流程

> 深入揭秘 epoll 是如何实现 IO 多路复用的 - 腾讯技术工程的文章 - 知乎 https://zhuanlan.zhihu.com/p/487497556

![img](v2-cf92cbbce0b44eab29291fc3702201a4_1440w.jpg)

#### 开始监听一系列 socket

若要监听一个 socket，会创建一个关联 socket 和 event_poll rb_tree 的中间结构 `epitem`。socket 和 eventpoll rb_tree 可以通过 `epitem` 互相找到对方。之所以使用红黑树，是因为添加删除相对较快、也方便通过 fd 的值快速找到相关节点。

![img](v2-8c2ae92d21fce01ec2f639fe7d3759ab_1440w.jpg)

具体来说分为三步：

1. 生成关联的 epitem
2. 在 socket 的等待队列中注册回调函数。（socket 需要在数据就绪的时候唤醒等待该 socket 的进程，此处我们选择唤醒 epoll 的回调函数，告知 epoll 有 socket 能用了）
3. 将 epitem 插入到红黑树中

```c
//file: fs/eventpoll.c
static int ep_insert(struct eventpoll *ep, struct epoll_event *event, struct file *tfile, int fd) {
    //3.1 分配并初始化 epitem
    //分配一个epi对象
    struct epitem *epi;
    if (!(epi = kmem_cache_alloc(epi_cache, GFP_KERNEL)))
        return -ENOMEM;

    //对分配的epi进行初始化
    //epi->ffd中存了句柄号和struct file对象地址
    INIT_LIST_HEAD(&epi->pwqlist);
    epi->ep = ep;
    ep_set_ffd(&epi->ffd, tfile, fd);

    //3.2 设置 socket 等待队列
    //定义并初始化 ep_pqueue 对象
    struct ep_pqueue epq;
    epq.epi = epi;
    init_poll_funcptr(&epq.pt, ep_ptable_queue_proc);

    //调用 ep_ptable_queue_proc 注册回调函数
    //实际注入的函数为 ep_poll_callback
    revents = ep_item_poll(epi, &epq.pt);

    ......
    //3.3 将epi插入到 eventpoll 对象中的红黑树中
    ep_rbtree_insert(ep, epi);
    ......
}
```

#### epoll_wait 等待能用的 socket

![img](v2-df1b90593fbe8e3ef77280333d762223_1440w.jpg)

- **wq：** 等待队列链表。软中断数据就绪的时候会通过 wq 来找到阻塞在 epoll 对象上的用户进程。
- **rbr：** 一棵红黑树。为了支持对海量连接的高效查找、插入和删除，eventpoll 内部使用了一棵红黑树。通过这棵树来管理用户进程下添加进来的所有 socket 连接。
- **rdllist：** 就绪的描述符的链表。当有的连接就绪的时候，内核会把就绪的连接放到 rdllist 链表里。这样应用进程只需要判断链表就能找出就绪进程，而不用去遍历整棵树。

epoll_wait 做的事情不复杂，当它被调用时它观察 eventpoll->rdllist 链表里有没有数据即可。有数据就返回，没有数据就创建一个等待队列项，将其添加到 eventpoll 的等待队列上，然后把自己阻塞掉就完事。

![img](v2-85e9e9f7d042469ca4a3e71aeaee867a_1440w.jpg)

> 注意：epoll_ctl 添加 socket 时也创建了等待队列项。不同的是这里的等待队列项是挂在 epoll 对象上的，而前者是挂在 socket 对象上的。

#### 接收到数据的处理

关于软中断是怎么处理网络帧的暂且不谈。总之，socket 的数据接受完之后，会唤醒等待队列中阻塞的进程。

```c
//file: net/ipv4/tcp_input.c
int tcp_rcv_established(struct sock *sk, struct sk_buff *skb,
            const struct tcphdr *th, unsigned int len)
{
    ......

    //接收数据到队列中
    eaten = tcp_queue_rcv(sk, skb, tcp_header_len,
                                    &fragstolen);

    //数据 ready，唤醒 socket 上阻塞掉的进程
    sk->sk_data_ready(sk, 0);
```

实际上，唤醒的就是 epoll_ctl 添加 socket 时在其上设置的回调函数 ep_poll_callback

![img](v2-10689ba69144589748abb5bd82cd949b_1440w.jpg)

执行 ep_poll callback 回调函数具体会：

1. 把自己的 epitem 添加到 epoll 的就绪队列中。

2. 查看 eventpoll 对象上的等待队列里是否有等待项（epoll_wait 执行的时候会设置）。如果有等待项，那就查找到等待项里设置的回调函数。总之会唤醒阻塞的用户进程。
   
   ![img](v2-4a4abcce05720f330679103150d26f43_1440w.jpg)

```c
//file: fs/eventpoll.c
static int ep_poll_callback(wait_queue_t *wait, unsigned mode, int sync, void *key)
{
    //获取 wait 对应的 epitem
    struct epitem *epi = ep_item_from_wait(wait);

    //获取 epitem 对应的 eventpoll 结构体
    struct eventpoll *ep = epi->ep;

    //1. 将当前epitem 添加到 eventpoll 的就绪队列中
    list_add_tail(&epi->rdllink, &ep->rdllist);

    //2. 查看 eventpoll 的等待队列上是否有在等待
    if (waitqueue_active(&ep->wq))
        wake_up_locked(&ep->wq);
```

## 网络编程模型（不仅考虑了 IO）

### Question：fork 之后，socket 会怎样？

> 一个进程绑定了端口号后，创建子进程（fork），子进程是不是和父进程绑定了同一个端口号？ - 知乎
> https://www.zhihu.com/question/360939266

**子进程因为具有父进程的文件描述符副本，可以处理绑定到同样的端口上的连接。** 所以可以认为子进程和父进程绑定了同一个端口号

**我们首先看 accept的问题**

绑定到同一个端口这件事，上面的情况指的是在 bind() 之后的 fork 调用，在 bind 之前实际上你也可以，并且不限于父子级关系的进程。Linux 内核有一个 PORTREUSE 机制。

假设你启动了一个进程，绑定 x 端口，并使用了 SO_REUSEPORT 参数。那么所有后续进程，只要同样使用 SO_REUSEPORT 绑定 x 端口，都可以成功启动，**内核会为这些进程公平的分配连接**，可以看成是一个内核级的负载均衡？需要注意的是，**这时候是有多个 socket 的，只是绑定了同一个端口而已**。

> 个人猜测，按理说不会带来惊群效应？

![preview](https://pic1.zhimg.com/v2-9f924c88e27b1b55b516d8e93f6e83d8_r.jpg?source=1940ef5c)

在 bind() 之后 fork 出来的所有子进程，也可以处理传入的连接，**不过要通过加锁或互斥来实现连接分配（也就是自行实现负载均衡）**。

每个子进程都可以去 accept 这个套接字，当有连接进来的时候，所有进程的 accept 都会返回，但是只有一个能够成功，其它都会失败，这个叫惊群效应。Nginx 对此有一定的优化，似乎是通过信号量之类的方式，只有争夺到的进程才能 accept。

![preview](https://pic2.zhimg.com/v2-0212d132a16bb60c67bd57bf91efb964_r.jpg?source=1940ef5c)

**关于已经 accept 的 socket 的 write read 操作**

> socket套接字在多线程发送数据时要加锁吗？ - 陈硕的回答 - 知乎
> https://www.zhihu.com/question/56899596/answer/150926723
> 
> socket套接字在多线程发送数据时要加锁吗？ - 笔默纸言的回答 - 知乎
> https://www.zhihu.com/question/56899596/answer/1161802619

所以一个好的tcp socket处理框架，应该是只有一个线程来负责数据的收发，从而避免那些无穷无尽的同步问题。

- 对于 UDP，多线程读写同一个 socket 不用加锁，不过更好的做法是每个线程有自己的 socket，避免 contention，可以用 SO_REUSEPORT 来实现这一点。

- 对于 TCP，通常多线程读写同一个 socket 是错误的设计，因为有 short write 的可能（对于非阻塞 IO，当发送缓冲区的剩余空间大小不足以容纳发送数据大小的时候，此时只会发送部分数据，并且生成错误码`EAGAIN`，此时就产生了short write现象）。
  
  假如你加锁，而又发生 short write，你是不是要一直等到整条消息发送完才解锁（无论阻塞IO还是非阻塞IO）？如果这样，你的临界区长度由对方什么时候接收数据来决定，一个慢的 peer 就把你的程序搞死了。
  
  对于不加锁的情况，可能会出现消息交叉的情况，比如一个进程发 aaaa，另一个发 bbbb，接受的可能是 abab
  
  > 有大佬表示：
  > 
  > 我们团队以前踩过这个坑。原因如下：tcp_sendmsg 的确会持有 lock_sock 锁，但在申请 sendbuf 时如果遇上内存不足的情况，会进入 sk_wait_event，在此期间会 release_sock，别的线程可能正好能够进入关键路径。

### 阻塞IO+多线程

client连接服务器，服务器有一个线程阻塞的调用 accept，accept 接收到连接后，创建一个线程来读写读写，并且处理业务逻辑

<img src="v2-e8b54da0e44d7163f44c49dc97e4fb24_b.jpg" alt="img" style="zoom:50%;" />

### 单 reactor + 单线程

典型的模型应用代表: redis。

服务器只有一个线程，一个epoll，epoll 接收到连接事件后就调用 accept，接收到客户端的fd后，将 fd 加入到 epoll，监听客户端的读写以及断开连接事件，读取到客户端的消息后，对消息进行解码，然后业务逻辑处理，处理完后编码需要回复的数据包，然后发送消息，交互完成后客户端断开连接，监听到断开连接的消息后调用close.

有可能业务逻辑过重，IO 不是瓶颈，反而是业务逻辑成了瓶颈。

![preview](v2-d9d08263bd06ae7d23d1c297b8b4719e_r.jpg)

### 单reacotor +消息队列+多线程模型（线程池）

跟 reactor + 单线程 的区别就是在业务逻辑处理的地方加了消息队列，接收到的消息放入接收消息队列，然后业务线程从该队列取消息进行业务处理 **（业务线程数量最好是跟cpu核心数，业务线程中基本没有了IO操作）** ，业务线程处理完后，将需要回复的消息放入发送队列中，当有可发送事件触发时，epoll 从发送队列取消息发送。

相比 单 reactor + 单线程，**解决了业务逻辑重的问题**。

<img src="image-20220215165510326.png" alt="image-20220215165510326" style="zoom:50%;" />

<img src="v2-b446dcdb8f75d1d70cd46ff9dce0f7fb_r.jpg" alt="preview" style="zoom:67%;" />

### 多 Reactor

在单 Reactor中，其要同时响应所有的事件，如果 IO 连接，读写都很多，可能会出现处理不过来的情况。（毕竟涉及到状态维护之类的？）

> 像 TCP 这样的需要经过 3 次握手才能建立连接，这个建立连接的过程也是要耗时间和资源的，单独分一个 Reactor 来处理，可以提高性能。

因此可以采用一个 main reactor + **n 个 sub reactor** 。

main reactor 一直在监听连接事件，如果有连接建立，main reactor 通过 accept 方法获取已连接的 SocketChannel，然后按照一定的算法选取一个 sub reactor，并把 SocketChannel 交给选定的 sub reactor 处理。

> 为了防止单个 sub-reactor 线程出错，还是考虑选用多进程的多 reactor。nginx 就是多进程 reactor 

![img](webp.webp)

## Go 的网络模型

> **Go netpoller 的价值**
> 
> https://strikefreedom.top/archives/go-netpoll-io-multiplexing-reactor#toc-head-18https://strikefreedom.top/archives/go-netpoll-io-multiplexing-reactor#toc-head-18
> 
> 通过前面对源码的分析，我们现在知道 Go netpoller 依托于 runtime scheduler，为开发者提供了一种强大的同步网络编程模式；然而，Go netpoller 存在的意义却远不止于此，Go netpoller I/O 多路复用搭配 Non-blocking I/O 而打造出来的这个原生网络模型，它最大的价值是把网络 I/O 的控制权牢牢掌握在 Go 自己的 runtime 里，关于这一点我们需要从 Go 的 runtime scheduler 说起
> 
> Go netpoller 是为了避免让操作网络 I/O 的 goroutine 陷入到系统调用从而进入内核态，因为一旦进入内核态，整个程序的控制权就会发生转移(到内核)，不再属于用户进程了，那么也就无法借助于 Go 强大的 runtime scheduler 来调度业务程序的并发了；而有了 netpoll 之后，借助于非阻塞 I/O ，G 就再也不会因为系统调用的读写而 (长时间) 陷入内核态，当 G 被阻塞在某个 network I/O 操作上时，实际上它不是因为陷入内核态被阻塞住了，而是被 Go runtime 调用 gopark 给 park 住了，此时 G 会被放置到某个 wait queue 中，而 M 会尝试运行下一个 _Grunnable 的 G，如果此时没有 _Grunnable 的 G 供 M 运行，那么 M 将解绑 P，并进入 sleep 状态。当 I/O available，在 epoll 的 eventpoll.rdr 中等待的 G 会被放到 eventpoll.rdllist 链表里并通过 netpoll 中的 epoll_wait 系统调用返回放置到全局调度队列或者 P 的本地调度队列，标记为 _Grunnable ，等待 P 绑定 M 恢复执行。

Go 基于 I/O multiplexing 和 goroutine scheduler 构建了一个简洁而高性能的原生网络模型(基于 Go 的 I/O 多路复用 `netpoller` )，提供了 `goroutine-per-connection` 这样简单的网络编程模式。

在这种模式下，开发者使用的是同步的模式去编写异步的逻辑，极大地降低了开发者编写网络应用时的心智负担，且借助于 Go runtime scheduler 对 goroutines 的高效调度，这个原生网络模型不论从适用性还是性能上都足以满足绝大部分的应用场景。

**一句话概括：每个 socket 用一个 goroutine 去处理，read 的时候是阻塞的，但是不是系统线程级别的阻塞，而是不调度该 goroutine，底层多个 socket 用 IO 多路复用（+回调？）的形式实现**。

这里有一个问题：为什么 golang 的网络 io 要阻塞用户层面 io，而在底层用多路复用。直接用阻塞的 io 效果不一样吗？

有一个猜想：

1. 从以前线程的角度来看，一个 IO 一个线程肯定开销太大了，依靠**系统级别进程的阻塞+唤醒**，代价比较高。

2. 那让用户自己去写 IO 多路复用吧。那有点麻烦啊。。。你要自己处理这些逻辑，epoll 注册之类的。能不能帮用户搞一下？
   
   诶！我来给用户包装一下。用户以同步的方式，以 `goroutine-per-connection` 的体验去处理各个 socket。底层依靠 IO 多路复用，我只需要将这个 Goroutine park 住就好了，不会导致 P 和 M 的绑定关系变化，不会进入内核态进行线程切换，也不影响同一个 P 中的其他 Goroutine。
   
   相比较底层不采用 IO 多路复用的形式，将用户层面的同步阻塞下放到系统线程层面的阻塞，为了不阻塞其他的 Goroutine，会导致 P 绑定到新的 M 上，且发生了系统线程切换，带来了开销。

### gnet

`goroutine-per-connection` 这种模式虽然简单高效，但是在某些极端的场景下也会暴露出问题：goroutine 虽然非常轻量，它的自定义栈内存初始值仅为 2KB，后面按需扩容；海量连接的业务场景下， goroutine-per-connection ，此时 goroutine 数量以及消耗的资源就会呈线性趋势暴涨。

在绝大部分应用场景下，推荐大家还是遵循 Go 的 best practices，使用原生的 Go 网络库来构建自己的网络应用。然而，在某些极度追求性能、压榨系统资源的业务场景下，我们可以考虑自己构建 Reactor 网络模型。这里介绍一个 **gnet**

gnet 是一个基于事件驱动的高性能和轻量级网络框架。它直接使用 epoll 和 kqueue 系统调用而非标准 Go 网络包 net 来构建网络应用，这也使得在某些极端的网络业务场景（比如海量连接、高频短连接、网络小包等等场景）gnet 达到了一个远超 Go net 的性能表现。

gnet 已经实现了 Multi-Reactors 和 Multi-Reactors + Goroutine Pool 两种网络模型，避免了一个 connection 就创建一个 goroutine。

![](https://raw.githubusercontent.com/panjf2000/illustrations/master/go/multi-reactors.png)

![](https://raw.githubusercontent.com/panjf2000/illustrations/master/go/multi-reactors%2Bthread-pool.png)