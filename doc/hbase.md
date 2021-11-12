## HBase 中实现了两种 compaction 的方式：
* minor and major. 这两种 compaction 方式的
* 区别是：  
1、Minor 操作只用来做部分文件的合并操作以及包括 minVersion=0 并且设置 ttl 的过
期版本清理，不做任何删除数据、多版本数据的清理工作。  
2、Major 操作是对 Region 下的 HStore 下的所有 StoreFile 执行合并操作，最终的结果
是整理合并出一个文件。 

## 11.HBase 读写流程？（☆☆☆☆☆） 
   
   读：  
   ① HRegionServer 保存着 meta 表以及表数据，要访问表数据，首先 Client 先去访问zookeeper，从 zookeeper 里面获取 meta 表所在的位置信息，即找到这个 meta 表在哪个HRegionServer 上保存着。  
   ② 接着 Client 通过刚才获取到的 HRegionServer 的 IP 来访问 Meta 表所在的HRegionServer，从而读取到 Meta，进而获取到 Meta 表中存放的元数据。  
   ③ Client 通过元数据中存储的信息，访问对应的 HRegionServer，然后扫描所在HRegionServer 的 Memstore 和 Storefile 来查询数据。  
   ④ 最后 HRegionServer 把查询到的数据响应给 Client。  
   
   
   写：  
   ① Client 先访问 zookeeper，找到 Meta 表，并获取 Meta 表元数据。  
   ② 确定当前将要写入的数据所对应的 HRegion 和 HRegionServer 服务器。  
   ③ Client 向该 HRegionServer 服务器发起写入数据请求，然后 HRegionServer 收到请求  
   并响应。  
   ④ Client 先把数据写入到 HLog，以防止数据丢失。  
   ⑤ 然后将数据写入到 Memstore。  
   ⑥ 如果 HLog 和 Memstore 均写入成功，则这条数据写入成功  
   ⑦ 如果 Memstore 达到阈值，会把 Memstore 中的数据 flush 到 Storefile 中。  
   ⑧ 当 Storefile 越来越多，会触发 Compact 合并操作，把过多的 Storefile 合并成一个大  
   的 Storefile。  
   ⑨ 当 Storefile 越来越大，Region 也会越来越大，达到阈值后，会触发 Split 操作，将Region 一分为二。
