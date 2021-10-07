## kafka组件：  
* producer、broker、topic、partition、replication(默认10个，不能大于broker数量)、message、consumer、consumer-group、zookeeper  
* producer如何保证消息顺序  
> 1：指定partition；2：给消息设置key（hash）；3：轮询  
* producer如何保证消息不丢失  
> 使用ACK应答机制，0-只发送不管集群是否成功写入磁盘；1-保证集群写入leader副本；all-全部写入所有副本   
* producer只连接leader副本，follower主动链接leader同步数据  
* producer往不存在的topic写入数据时，集群默认创建topic，partition和replication数量都是1  

## kafka数据存储  
* partition > segment > log、index、timeindex ；利用segment + 有序offset + 稀疏索引 + 二分查找 + 顺序查找解决查询效率  
* message包含offset(8bit)、消息大小(1bit)、消息体......  
* 消息删除策略：基于时间（默认7天）；基于文件大小（默认1GB）  
* consumer消费消息同producer一样直接链接leader  
* consumer维护消费位置  
> 早起版本维护在zk里，consumer间隔一段时间上报一次，会导致重复消费；新版本维护在kafka集群topic为__consumer_offsets里  

## 选举机制  
* 从broker中选举一个做为controller，partition的leader由controller决定  
* controller从partition里的state获取ISR集合  
> 如果还有一个replica存活，则直接选举为leader，更新ISR集合；  
> 如果有多个replica存活，则随机选举一个为leader，可能有数据丢失；  
> 如果集合为空，则新leader被置为-1    
* 将新的leader、isr、leader_epoch、controller_epoch写入zk  

## 如何处理所有Replica都不工作  
* 1.等待ISR中的任一个Replica“活”过来，并且选它作为Leader  
* 2.选择第一个“活”过来的Replica（不一定是ISR中的）作为Leader

## high water（0.11版本之前[造成数据丢失、数据离散]，之后引入leader epoch）  
* 每个replica都有LEO(log end offset)和HW，leo保存下一条消息的offset，HW 小于等于 LEO，HW表示已经完成备份  
* follower更新leo：follower向leader发送FETCH（带有leo）请求，写入日志自动更新leo  
* follower更新HW：取自己的LEO和leader响应FETCH里的HW比较，取小的更新HW  
* leader更新leo：接受producer写入数据时自动更新；更新leader端follower的leo，接受FETCH请求，读取数据后先更新LEO（用FETCH里的LEO）再发送数据  
* leader更新HW，满足以下条件，尝试更新  
> 1：broker宕机副本被踢出ISR；2：副本被选举为leader；3：producer写入数据；4：受理follower的FETCH请求  
> 从ISR集合里或者副本LEO落后于leader LEO的时长不大于replica.lag.time.max.ms参数值 取最小的LEO更新为分区HW（leader HW）  

## 消息投递  
> 最多一次；最少一次；有且仅有一次

## 快？  
* 顺序写入  
* MMF（内存映射文件），利用操作系统Page做物理内存和文件之间的映射，操作内存会同步到文件；64位操作系统可用20GB  
* 将index文件全部映射到内存  
* 使用sendfile做网络传输zero-copy

## producer数据批量处理  
* buffer.memory 客户端将数据写入缓冲区（默认32MB）  
* batch.size 一次发送的batch消息大小（默认16KB）  
* linger.ms 批量消息等待时长  


