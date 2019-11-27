# TCP中backlog参数的作用

当应用程序通过使用系统调用[`listen`](http://linux.die.net/man/2/listen)函数将套接字置于LISTEN状态时，它需要为该套接字指定一个 backlog。 backlog通常被描述为用于限制传入的连接数的队列。

![TCP状态图](TCP的backlog参数在linux中的作用.assets/tcp-state-diagram.png)

由于TCP使用三次握手，因此传入连接在到达ESTABLISHED状态之前会经过SYN RECEIVED的中间状态，并且可以通过accept系统调用返回到应用程序（请参见上面的[TCP状态图](http://commons.wikimedia.org/wiki/File:Tcp_state_diagram_fixed.svg)）。这意味着TCP / IP堆栈可以有两种方法来为处于LISTEN状态的套接字实现backlog队列：

1. 该实现使用单个队列，队列的大小由系统调用`listen`函数的`backlog`参数确定。收到SYN数据包后，它将发回SYN / ACK数据包，并将连接添加到队列中。接收到相应的ACK后，连接会将其状态更改为ESTABLISHED，并有资格切换到应用程序。这意味着队列可以包含处于两种不同状态的连接：SYN RECEIVED和ESTABLISHED。只有后一种状态的连接可以通过系统调用accept函数返回到应用程序。
2. 该实现使用两个队列，一个SYN队列（或半连接队列）和一个接受队列（全连接队列）。状态为SYN RECEIVED的连接被添加到SYN队列中，然后在它们的状态变为ESTABLISHED时（即，在收到第三次握手中的ACK数据包时）移至接受队列。顾名思义，该 `accept`调用使用来自接受队列的连接。在这种情况下，系统调用`listen`的`backlog` 参数确定接受队列的大小。

从历史上看，BSD派生的TCP实现使用第一种方法。该选择意味着当达到最大backlog时，系统将不再发送回SYN / ACK数据包以响应SYN数据包。通常，TCP实现将简单地丢弃SYN数据包（而不是使用RST数据包进行响应），以便客户端重试。这是W. Richard Stevens的经典教科书*TCP / IP Illustrated，第3卷中的* 14.5节 listen *Backlog* 队列 中描述的内容。

请注意，Stevens实际上解释了BSD实现确实使用了两个单独的队列，但是它们表现为具有固定最大大小的单个队列，该最大大小由`backlog`参数确定（但不必完全相等），即BSD在逻辑上的行为如选项1中所述：

> 队列限制适用于[…]半连接队列[…]上的条目数和[…]全连接队列[…]上的条目数之和。

在Linux上，情况有所不同，如[手册页](http://linux.die.net/man/2/listen)系统调用函数`listen`的说明所述：

> 在Linux 2.2上，TCP套接字上的`backlog`参数行为已更改。现在，它是等待接受的全连接套接字的队列长度 ，而不是半连接的请求数。可以使用linux系统参数`/proc/sys/net/ipv4/tcp_max_syn_backlog`来设置半连接套接字队列的最大长度。

这意味着当前的Linux版本将第二个选项与两个不同的队列一起使用：一个SYN队列，其大小由系统范围的参数设置指定；一个接受队列，其大小由应用程序指定。

现在有趣的问题是，如果接受队列已满并且需要将连接从SYN队列移至接受队列，即当接收到三次握手的ACK数据包时，这种实现方式的处理方式会是怎样的。这种情况由`net/ipv4/tcp_minisocks.c`中的`tcp_check_req`函数来处理。相关代码为：

```c
        child = inet_csk(sk)->icsk_af_ops->syn_recv_sock(sk, skb, req, NULL);
        if (child == NULL)
                goto listen_overflow;
```

对于IPv4，`net/ipv4/tcp_ipv4.c`中的第一行代码tcp_v4_syn_recv_sock函数将被调用，其中包含以下代码：

```c
        if (sk_acceptq_is_full(sk))
                goto exit_overflow;
```

我们在这里看到接受队列的检查。`exit_overflow`标签后面的代码将执行一些清理工作，更新`ListenOverflows`和`/proc/net/netstat`中的`ListenDrops`统计信息，最后返回`NULL`。这将触发`tcp_check_req`中`listen_overflow`代码的执行：

```c
listen_overflow:
        if (!sysctl_tcp_abort_on_overflow) {
                inet_rsk(req)->acked = 1;
                return NULL;
        }
```

这意味着除非`/proc/sys/net/ipv4/tcp_abort_on_overflow`设置为1（在这种情况下，紧接着上面显示的代码之后的代码将发送RST数据包），否则该实现基本上不会执行任何操作！

总而言之，如果Linux中的TCP实现接收到3次握手的ACK数据包并且接受队列已满，则它将基本上忽略该数据包。最初，这听起来很奇怪，但是请记住，有一个与SYN RECEIVED状态关联的计时器：如果未收到ACK数据包（或者如此处所考虑的那样被忽略），则TCP实现将重新发送该SYN / ACK数据包（通过`/proc/sys/net/ipv4/tcp_synack_retries`配置指定一定重试的次数，并使用 [exponential backoff](http://en.wikipedia.org/wiki/Exponential_backoff)  算法）。

对尝试连接（和发送数据）到已达到最大 backlog 的套接字的客户端进行数据包的跟踪，可以看到如下内容：

```shell
  0.000  127.0.0.1 -> 127.0.0.1  TCP 74 53302 > 9999 [SYN] Seq=0 Len=0
  0.000  127.0.0.1 -> 127.0.0.1  TCP 74 9999 > 53302 [SYN, ACK] Seq=0 Ack=1 Len=0
  0.000  127.0.0.1 -> 127.0.0.1  TCP 66 53302 > 9999 [ACK] Seq=1 Ack=1 Len=0
  0.000  127.0.0.1 -> 127.0.0.1  TCP 71 53302 > 9999 [PSH, ACK] Seq=1 Ack=1 Len=5
  0.207  127.0.0.1 -> 127.0.0.1  TCP 71 [TCP Retransmission] 53302 > 9999 [PSH, ACK] Seq=1 Ack=1 Len=5
  0.623  127.0.0.1 -> 127.0.0.1  TCP 71 [TCP Retransmission] 53302 > 9999 [PSH, ACK] Seq=1 Ack=1 Len=5
  1.199  127.0.0.1 -> 127.0.0.1  TCP 74 9999 > 53302 [SYN, ACK] Seq=0 Ack=1 Len=0
  1.199  127.0.0.1 -> 127.0.0.1  TCP 66 [TCP Dup ACK 6#1] 53302 > 9999 [ACK] Seq=6 Ack=1 Len=0
  1.455  127.0.0.1 -> 127.0.0.1  TCP 71 [TCP Retransmission] 53302 > 9999 [PSH, ACK] Seq=1 Ack=1 Len=5
  3.123  127.0.0.1 -> 127.0.0.1  TCP 71 [TCP Retransmission] 53302 > 9999 [PSH, ACK] Seq=1 Ack=1 Len=5
  3.399  127.0.0.1 -> 127.0.0.1  TCP 74 9999 > 53302 [SYN, ACK] Seq=0 Ack=1 Len=0
  3.399  127.0.0.1 -> 127.0.0.1  TCP 66 [TCP Dup ACK 10#1] 53302 > 9999 [ACK] Seq=6 Ack=1 Len=0
  6.459  127.0.0.1 -> 127.0.0.1  TCP 71 [TCP Retransmission] 53302 > 9999 [PSH, ACK] Seq=1 Ack=1 Len=5
  7.599  127.0.0.1 -> 127.0.0.1  TCP 74 9999 > 53302 [SYN, ACK] Seq=0 Ack=1 Len=0
  7.599  127.0.0.1 -> 127.0.0.1  TCP 66 [TCP Dup ACK 13#1] 53302 > 9999 [ACK] Seq=6 Ack=1 Len=0
 13.131  127.0.0.1 -> 127.0.0.1  TCP 71 [TCP Retransmission] 53302 > 9999 [PSH, ACK] Seq=1 Ack=1 Len=5
 15.599  127.0.0.1 -> 127.0.0.1  TCP 74 9999 > 53302 [SYN, ACK] Seq=0 Ack=1 Len=0
 15.599  127.0.0.1 -> 127.0.0.1  TCP 66 [TCP Dup ACK 16#1] 53302 > 9999 [ACK] Seq=6 Ack=1 Len=0
 26.491  127.0.0.1 -> 127.0.0.1  TCP 71 [TCP Retransmission] 53302 > 9999 [PSH, ACK] Seq=1 Ack=1 Len=5
 31.599  127.0.0.1 -> 127.0.0.1  TCP 74 9999 > 53302 [SYN, ACK] Seq=0 Ack=1 Len=0
 31.599  127.0.0.1 -> 127.0.0.1  TCP 66 [TCP Dup ACK 19#1] 53302 > 9999 [ACK] Seq=6 Ack=1 Len=0
 53.179  127.0.0.1 -> 127.0.0.1  TCP 71 [TCP Retransmission] 53302 > 9999 [PSH, ACK] Seq=1 Ack=1 Len=5
106.491  127.0.0.1 -> 127.0.0.1  TCP 71 [TCP Retransmission] 53302 > 9999 [PSH, ACK] Seq=1 Ack=1 Len=5
106.491  127.0.0.1 -> 127.0.0.1  TCP 54 9999 > 53302 [RST] Seq=1 Len=0
```

由于TCP实现载客户端支持获取多个SYN / ACK数据包，因此它将假定ACK数据包丢失并重新发送（请参见`TCP Dup ACK`上面跟踪中的行）。如果在达到最大SYN / ACK重试次数之前，服务器端的应用程序的backlog释放出了空间 （即接受队列的数量变小了），则TCP实现最终将处理这些重复的ACK，从而将连接状态从SYN RECEIVED转到ESTABLISHED，并将其添加到接受队列。否则，客户端将最终获得一个RST数据包（如上面的示例所示）。

数据包跟踪还向我们展示了一个有意思的事情。从客户端的角度来看，在接收到第一个SYN / ACK之后，连接将处于ESTABLISHED状态。如果它发送数据（不先等待服务器发送数据），那么该数据也将重新发送。幸运的是，[TCP慢启动](http://en.wikipedia.org/wiki/Slow-start) 会限制此阶段中发送的段（ segments ）数。

另一方面，如果客户端等待来自服务器的数据，而服务器的backlog满了却一直不释放空间，则最终结果是客户端上的连接处于ESTABLISHED状态，而服务器端上的连接状态为CLOSED 。这意味着最终会出现[半开连接](http://en.wikipedia.org/wiki/Half-open_connection)！

我们还没有讨论过另一方面。手册页中`listen`方法的引号表明，每个SYN数据包都会导致向SYN队列添加一个连接（除非该队列已满）。事情并非如此。原因是`net/ipv4/tcp_ipv4.c`文件中`tcp_v4_conn_request`函数中的以下代码（用于处理SYN数据包） ：

```c
        /* Accept backlog is full. If we have already queued enough
         * of warm entries in syn queue, drop request. It is better than
         * clogging syn queue with openreqs with exponentially increasing
         * timeout.
         */
        if (sk_acceptq_is_full(sk) && inet_csk_reqsk_queue_young(sk) > 1) {
                NET_INC_STATS_BH(sock_net(sk), LINUX_MIB_LISTENOVERFLOWS);
                goto drop;
        }
```

这意味着如果接受队列已满，则内核将对SYN数据包的接受速率施加限制。如果收到太多SYN数据包，则其中的一些将被丢弃。在这种情况下，客户端会重试发送SYN数据包，并且最终得到与BSD派生的实现相同的行为。

 让我们尝试看看为什么Linux所做的设计选择会优于传统的BSD实现。史蒂文斯提出以下有趣的观点：

> 如果全连接的队列已满（即，服务器进程或服务器主机非常忙，以致该进程无法通过`accept`系统调用足够快地将已完成的条目从队列中移出），或者半连接的队列已满，则可以使得backlog队列容量用完 。HTTP服务器面临的问题是，与新连接请求的到达率相比，当客户端和服务器之间的往返时间较长时，因为新的SYN在该队列上的条目占用了一个往返时间。[…]
>
> 全连接队列几乎总是空的，因为当在该队列上放置一个条目时，服务器将通过`accept`调用返回 ，并且服务器将完成的连接从队列中移出。

史蒂文斯建议的解决方案只是调大 backlog。这样做的问题在于，它假定应用程序不仅希望考虑如何处理新建立的传入连接，而且还要考虑流量特性（例如往返时间）来调整backlog。Linux中的实现有效地分离了这两个方面：应用程序仅负责调整backlog，以便可以足够快地调用 `accept`以避免填满接受队列）；然后，系统管理员可以根据流量特征调整`/proc/sys/net/ipv4/tcp_max_syn_backlog` 。

原文链接： http://veithen.io/2014/01/01/how-tcp-backlog-works-in-linux.html 

