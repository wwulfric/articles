socket 是如何连接的

## 使用套接字描述连接

要让两台遥远的计算机连接，无论如何，我们都要抽象出一根管子，使得两台计算机可以通信，这就是网络连接。

![网络连接[^2]](http://static.wulfric.me/net/client-and-server.png)

TCP/IP 协议族就是这一抽象思想的一个实现，它们将计算机的物理连接（网线，无线网络，电缆等）抽象成两个 IP 地址的连接，再通过 TCP 协议控制传输。如下图所示。

![TCP/IP 连接[^1]](http://static.wulfric.me/net/client-socket-server.png)

一个套接字是连接的一个端点。每个套接字都有相应的套接字地址，是由一个因特网地址和一个16位的整数端口组成的，用「地址：端口」来表示。我们用套接字对表示一个连接。所谓连接，就是 ip:port <=> ip:port。

套接字本质上是一个文件描述符，它是用来与另一个进程进行跨网络通信的抽象文件。在 Linux 中，所有的 I/O 设备（例如网络、磁盘和终端）都被模型化为一个文件，而所有的输入和输出都当作对相应文件的读和写来执行。这种将设备优雅地映射为文件的方式，允许 Linux 内核引出一个简单、低级的应用接口，称为 Unix I/O，这使得所有的输入和输出都能以一种统一且一致的方式来执行。[^1]

套接字的名称来源于套接管[^3]，如图所示，是将两个同构但没有接口的管道套接在一起的抽象。

![套接管](http://static.wulfric.me/net/socket-analog.jpg)

因此，客户端连接服务端，相当于客户端的 ip:port 透过网线伸出来长长的管道一直延伸到服务器，然后服务器拿一个套接字啪：把客户端的 ip:port 接到了服务器监听的 ip:port 上。

![连接](http://static.wulfric.me/net/xmoji-client-connect-server.png)

## 套接字的连接

TCP/IP 协议实现了标准模型 OSI，并简化为应用层、传输层、网络层、数据链路层和物理层。套接字编程接口是从应用层进入传输层的接口。传输层及以下通常作为操作系统内核的一部分，我们主要关心的是处在用户进程中的应用层，关于 TCP/IP 的通信细节可参考相关书籍（如 图解TCP/IP 等）。

![OSI, TCP/IP[^2]](http://static.wulfric.me/net/OSI-TCP_IP.png)

操作系统在内核层完成 TCP 的 3 次握手和各种流量控制，所提供的套接字的操作都是应用层细节。

基础的连接配图：1. clientfd -> listenfd -> connfd; 2. 带上 ip:port

`*:80,*:*` 表示监听 socket (listenfd)，listenfd 并没有实际的连接

### 套接字发起连接过程

socket 连接和接受连接过程描述

socket: 拿出套接管，并接上自己的一头

connect: 连上另一头

### 套接字接受连接过程

socket 创建套接字

bind 绑定端口

listen：告知别人这里这个端口可以连接，从主动套接字变为被动套接字，指示内核应该接受指向该套接字的连接请求。

在 gnu 的 manual 中介绍到[^4]，

> A socket that has been established as a server can accept connection requests from multiple clients. The server’s original socket *does not become part of the connection*; instead, `accept` makes a new socket which participates in the connection. `accept` returns the descriptor for this socket. The server’s original socket remains available for listening for further connection requests.
>
> 已建立为服务端的 socket 可以接受来自多个客户端的连接请求。服务端的原始 socket 并不会成为连接的一部分，而是由 accept 创建出一个新的 socket 来参与连接。accept 方法返回这个 socket 的文件描述符。服务端原始 socket 仍可用来监听后续连接请求。

accept：实际完成连接，并返回连接的套接字描述符

unix 网络编程，client 连接进入内核，内核完成 tcp 连接放入队列，accept 取出实际的连接并设置指向连接的内存区，返回文件描述符



https://juejin.im/post/5b29d2c4e51d4558b80b1d8c 动图非常好

https://juejin.im/post/5b344ad6e51d4558892eeb46 图



系列文章：nginx 各种超时演示；

java nioserver，socket，演示 java nio bug

java nio 实现 reactor 模型

netty ：https://juejin.im/post/5b38e41cf265da597c773abf 启动过程

netty 百万连接

netty 实现 service mesh

[^1]: 深入理解计算机系统
[^2]: UNIX 网络编程---卷 1：套接字联网 API
[^3]: [高性能Server---Reactor模型](http://www.ivaneye.com/2016/07/23/iomodel.html) 
[^4]: [gnu accept](https://www.gnu.org/software/libc/manual/html_node/Accepting-Connections.html)

