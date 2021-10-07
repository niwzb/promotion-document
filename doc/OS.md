#  Page cache  
* IO分为缓存IO和直接IO（O_DIRECT）  
* 多个page组成page cache  
* page cache管理数据结构：radix tree（搜索树）和双向链表  

### radix tree  
* 一颗多叉数，从0层到6层  
* 叶子节点是页帧struct page，每个page存储4096个字节，从左到右按顺序依次类推  
* 中间节点为组织节点，指示某一地址上的数据所在的页；根节点为数据文件的空间地址(address_space)  
* 每颗数最多能存储16TB字节  

## 写入时机  
* 将数据写入cache，标记page为dirty，加入dirty list，内核周期的将dirty list写入磁盘  
* 数据写入  
> 1、数据在用户态的 buffer 中，调用 write 将数据传给内核；2、数据在 Page Cache 中，返回写入的字节数（成功返回）；3、内核将数据刷新到磁盘。  


# zero-copy 
* 传统文件访问需要两次上下文切换，三次数据拷贝  
> 用户态切换内核态，操作系统发起读请求，IO阻塞，数据从磁盘copy到驱动器缓冲区（DMA拷贝），数据从驱动器缓冲区copy到内核缓冲区，数据从内核缓冲区copy到用户内存，内核态切换用户态。  
* 数据网络传输需要四次上下文切换，四次数据拷贝  
> 读取数据，用户空间发送到socket  

## 实现zero-copy方式 
* mmap（内存映射文件），随机读写效率慢  
* sendfile（linux），linux2.1还是需要一次cpu copy（从内核态缓冲区到socket缓冲区），linux2.4使用文件描述符和DMA copy
> kafka利用sendfile实现zero-copy  
* splice（linux2.6），可以适用于任意两个文件描述符之间互相通信。通过管道实现，两个缓冲区中间连接一个管道  
> 并不是把第一个缓冲区的数据拷贝到pipe，而是将指针（引用）拷贝进去。 


# 内存屏障  
> volatile变量规则：对volatile变量的写入操作必须在对该变量的读操作之前执行。  
> volatile保证可见性、顺序性、传递性、禁用重排序（读操作禁止重排序之后的操作，写操作禁止重排序之前的操作）  
* Store：将处理器缓存的数据刷新到内存中。  
* Load：将内存存储的数据拷贝到处理器的缓存中。  
* MESI：缓存一致性协议。
* happens-before 和 as-if-serial。
* final、lock、cas。
