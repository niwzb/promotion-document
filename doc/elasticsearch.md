## 架构

## 倒排索引

## 数据结构
* Index  ES将数据存储于一个或多个索引中，索引是具有类似特性的文档的集合。
* Type 类型是索引内部的逻辑分区(category/partition)，其意义完全取决于用户需求。
> 在 5.X 版本中，一个 index 下可以创建多个 type；  
> 在 6.X 版本中，一个 index 下只能存在一个 type；  
> 在 7.X 版本中，直接去除了 type 的概念，就是说 index 不再会有 type。 
* Document 文档是索引和搜索的原子单位，它是包含了一个或多个域（Field）的容器，基于JSON格式进行表示。
> document路由原理   
①路由算法：shard = hash(routing) % number_of_primary_shards  
②决定一个document在哪个shard上，最重要的一个值就是routing值，默认是_id，也可手动指定，相同的routing值，每次过来，从hash函数中，产出的hash值一定是相同的  
例：手动指定一个routing value，比如 put /index/type/id?routing=user_id  
③这就是primary shard数量不可变的原因。

* Inverted Index 每一个文档都对应一个ID。倒排索引会按照指定语法对每一个文档进行分词，然后维护一张表，列举所有文档中出现的terms以及它们出现的文档ID和出现频率。搜索时同样会对关键词进行同样的分词分析，然后查表得到结果。
* Term Dictionary 将所有的term排序，二分法查找term，logN的查找效率
* Term Index（字典索引）是一颗 (trie) 前缀树
> B-Tree通过减少磁盘寻道次数来提高查询性能

## master
* 节点（Node）ES集群中的节点有三种不同的类型：
> 主节点：负责管理集群范围内的所有变更，例如增加、删除索引，或者增加、删除节点等。 主节点并不需要涉及到文档级别的变更和搜索等操作。可以通过属性node.master进行设置。
> 数据节点：存储数据和其对应的倒排索引。默认每一个节点都是数据节点（包括主节点），可以通过node.data属性进行设置。
> 协调节点：如果node.master和node.data属性均为false，则此节点称为协调节点，用来响应客户请求，均衡每个节点的负载。
* 分片（Shard）
> 一个索引中的数据保存在多个分片中，相当于水平分表。
> 在索引建立的时候就已经确定了主分片数，但是副本分片数可以随时修改。默认情况下，一个索引会有5个主分片，而其副本可以有任意数量。

## 并发问题
### 写数据
> commit point：列出所有已知 segment 的文件。
* 当用户向一个节点提交了一个索引新文档的请求，节点会计算新文档应该加入到哪个分片（shard）中。
每个节点都存储有每个分片存储在哪个节点的信息，因此协调节点会将请求发送给对应的节点。
注意这个请求会发送给主分片，等主分片完成索引，会并行将请求发送到其所有副本分片，
保证每个分片都持有最新数据。
> 每次写入新文档时，都会先写入内存 buffer（这时数据是搜索不到的），同时将数据写入 translog 日志文件。
* es每隔1秒（可配置）或者 buffer 快满时执行一次刷新（refresh）操作，将 buffer 数据 refresh 到 os cache 即操作系统缓存。这时数据就可以被搜索到了：
* buffer 的文档被写入到一个新的 segment 中（这1秒时间内写入内存的新文档都会被写入一个文件系统缓存（filesystem cache）中，并构成一个分段（segment））；
* segment 被打开以供搜索（此时这个segment里的文档可以被搜索到，但是尚未写入硬盘，即如果此时发生断电，则这些文档可能会丢失）；
* 内存 buffer 清空。
* 不断有新的文档写入，则这一过程将不断重复执行。每隔一秒将生成一个新的segment，当 translog 越来越大达到一定长度的时候，就会触发 flush 操作（flush 完成了 Lucene 的 commit 操作）
* 第一步将 buffer 中现有数据 refresh 到 os cache 中去，清空 buffer；
* 然后，将一个 commit point 写入磁盘文件，同时强行将 os cache 中目前所有的数据都 fsync 到磁盘文件中去；
* 最后清空现有 translog 日志文件并生成新的translog。
> 当然，translog本身也是文件，存在于内存当中，如果发生断电一样会丢失。因此，ES会在每隔5秒时间或是一次写入请求完成后将translog写入磁盘。

### segment 合并
* buffer 每 refresh 一次，就会产生一个 segment（默认情况下是 1 秒钟产生一个），这样 segment 会越来越多，此时会定期执行 merge。
> 将多个 segment 合并成一个，并将新的 segment 写入磁盘；
> 新增一个 commit point，标识所有新的 segment；
> 新的 segment 被打开供搜索使用；
> 删除旧的 segment。

### 删除和更新
* 由于 segment 是不可变的，索引删除的时候既不能把文档从 segment 删除，也不能修改 segment 反映文档的更新。
> 删除操作，会生成一个 .del 文件，commit point 会包含这个 .del 文件。.del 文件将文档标识为 deleted 状态，在结果返回前从结果集中删除。  
> 更新操作，会将原来的文档标识为 deleted 状态，然后新写入一条数据。查询时两个文档有可能都被索引到，但是被标记为删除的文档会被从结果集删除。

### 查询 
* 查询的过程大体上分为查询（query）和取回（fetch）两个阶段。
这个节点的任务是广播查询请求到所有相关分片，并将它们的响应整合成全局排序后的结果集合，
这个结果集合会返回给客户端。
> 查询阶段  
> 取回阶段
* 相关性计算
在搜索过程中对文档进行排序，需要对每一个文档进行打分，判别文档与搜索条件的相关程度。

### filter执行原理
* 在倒排索引中查找搜索串，获取document list
* 为每个在倒排索引中搜索到的结果，构建一个bitset
* 遍历每个过滤条件对应的bitset，优先从最稀疏的开始搜索，查找满足所有条件的document
* caching bitset
> caching bitset会跟踪query，在最近256个query中超过一定次数的过滤条件，缓存其bitset。对于小segment（<1000 或<3%），不缓存bitset。这样下次如果在有这个条件过来的时候，就不用重新扫描倒排索引，反复生成bitset，可以大幅度提升性能。
* filter大部分的情况下，是在query之前执行的，可以尽可能过滤掉多的数据
* 如果document有新增和修改，那么caching bitset会被自动更新
* 以后只要有相同的filter条件的查询请求打过来，就会直接使用这个过滤条件对应的bitset
> 这样查询性能就会很高，一些热的filter查询，就会被cache住。