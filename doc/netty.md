## Netty 的特点
* 高并发（NIO）、传输快（ZERO-COPY）、封装好（JDK NIO）
* 可以通过 ChannelHandler 对通信框架进行灵活地扩展。

## Netty 高性能
* IO 线程模型：Reactor线程模型-同步非阻塞，用最少的资源做更多的事。
* 内存零拷贝：尽量减少不必要的内存拷贝，实现了更高效率的传输。
* 内存池设计：申请的内存可以重用，主要指直接内存（DirectByteBuffer）。内部实现是用一颗二叉查找树管理内存分配情况。
* 串形化处理读写：避免使用锁带来的性能开销。
* 高性能序列化协议：支持 protobuf 等高性能序列化协议。

## Netty的线程模型
* Netty通过Reactor模型基于多路复用器接收并处理用户请求，内部实现了两个线程池，boss线程池和work线程池，  
其中boss线程池的线程负责处理请求的accept事件，当接收到accept事件的请求时，把对应的socket封装到一个NioSocketChannel中，  
并交给work线程池，其中work线程池负责请求的read和write事件，由对应的Handler处理。
> 单线程模型、多线程模型、主从多线程模型

* Netty主要基于主从Reactors多线程模型（如下图）做了一定的修改，其中主从Reactor多线程模型有多个Reactor：MainReactor和SubReactor：  
MainReactor负责客户端的连接请求，并将请求转交给SubReactor  
SubReactor负责相应通道的IO读写请求  
非IO请求（具体逻辑处理）的任务则会直接写入队列，等待worker threads进行处理  
![线程模型](image/threadmodel.png)


## TCP 粘包/拆包
* 应用程序写入的字节大小大于套接字发送缓冲区的大小，会发生拆包现象，而应用程序写入数据小于套接字缓冲区大小，  
网卡将应用多次写入的数据发送到网络上，这将会发生粘包现象；
> 进行MSS（最大报文段长度）大小的TCP分段，当TCP报文长度-TCP头部长度>MSS的时候将发生拆包  
以太网帧的payload（净荷）大于MTU（最大传输单元）（1500字节）进行ip分片。

### 解决方法
* 消息定长：FixedLengthFrameDecoder
* 行分隔符：LineBasedFrameDecoder
* 自定义分隔符 ：DelimiterBasedFrameDecoder
* 将消息分为消息头和消息体：LengthFieldBasedFrameDecoder

##  Netty 的零拷贝
* Netty 的接收和发送 ByteBuffer 采用 DIRECT BUFFERS，使用堆外直接内存进行 Socket 读写，不需要进行字节缓冲区的二次拷贝。
* Netty 提供了组合 Buffer 对象，可以聚合多个 ByteBuffer 对象，用户可以像操作一个 Buffer 那样方便的对组合 Buffer 进行操作，避免了传统通过内存拷贝的方式将几个小 Buffer 合并成一个大的 Buffer。
* Netty 的文件传输采用了 transferTo 方法，它可以直接将文件缓冲区的数据发送到目标 Channel，避免了传统通过循环 write 方式导致的内存拷贝问题。

## Netty 有两种发送消息的方式
* 直接写入 Channel 中，消息从 ChannelPipeline 当中尾部开始移动；
* 写入和 ChannelHandler 绑定的 ChannelHandlerContext 中，消息从 ChannelPipeline 中的下一个 ChannelHandler 中移动。  

## 序列化协议
* 影响序列化性能的关键因素：序列化后的码流大小（网络带宽的占用）、序列化的性能（CPU资源占用）；是否支持跨语言（异构系统的对接和开发语言切换）。
### 常见序列化协议
* Java默认提供的序列化：码流大
* XML：大文件
* Json：不适合性能要求为ms级别的情况
* Fastjson
* Thrift
* Avro
* Protobuf
* Hessian 

## NioEventLoop
* NioEventLoop中维护了一个线程和任务队列，支持异步提交执行任务，线程启动时会调用NioEventLoop的run方法，执行I/O任务和非I/O任务：  
I/O任务 即selectionKey中ready的事件，如accept、connect、read、write等，由processSelectedKeys方法触发。  
非IO任务 添加到taskQueue中的任务，如register0、bind0等任务，由runAllTasks方法触发。
 

  
