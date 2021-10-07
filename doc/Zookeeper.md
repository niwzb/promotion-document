
## zk保证分布式特性
* 顺序一致性、原子性、单一视图、可靠性、实时性（最终一致性）

## zk文件系统
* Zookeeper 为了保证高吞吐和低延迟，在内存中维护了这个树状的目录结构，这种特性使得 Zookeeper 不能用于存放大量的数据，每个节点的存放数据上限为1M。
## zab协议 保证了各个 server 之间的同步
* 恢复模式
> 当服务启动或者在领导者崩溃后，Zab就进入了恢复模式，当领导者被选举出来，且大多数 server 完成了和 leader 的状态同步以后，恢复模式就结束了。
> 状态同步保证了 leader 和 server 具有相同的系统状态。
* 广播模式
> 一旦 leader 已经和多数的 follower 进行了状态同步后，它就可以开始广播消息了，即进入广播状态。
* 新的server加入集群：在恢复模式下启动，发现 leader，并和 leader 进行状态同步。待到同步结束，它也参与消息广播。

## znode节点数据类型
* PERSISTENT-持久节点
* EPHEMERAL-临时节点
* PERSISTENT_SEQUENTIAL-持久顺序节点
* EPHEMERAL_SEQUENTIAL-临时顺序节点

## Watcher 机制（消息通知）
* 特性：一次性、客户端串行执行（watcher table）、轻量
> 通知事件从 server 发送到 client 是异步的。zk只能保证最终的一致性，而无法保证强一致性。
* 服务端接收到请求，判断是否需要注册 Watcher，需要的话将数据节点的节点路径和 ServerCnxn（ServerCnxn 代表一个客户端和服务端的连接，实现了 Watcher 的 process 接口，此时可以看成一个 Watcher 对象）存储在WatcherManager 的 WatchTable 和 watch2Paths 中去。
* Watcher 触发 将通知状态（SyncConnected）、事件类型（NodeDataChanged）以及节点路径封装成一个 WatchedEvent 对象。
* 查询 Watcher 从 WatchTable 中根据节点路径查找 Watcher。
* 提取并从 WatchTable 和 Watch2Paths 中删除对应 Watcher。
* 调用 process 方法来触发 Watcher（通过 ServerCnxn 对应的 TCP 连接发送 Watcher 事件通知）。
* 客户端 SendThread 线程接收事件通知，交由 EventThread 线程回调 Watcher。

## 选举
> 角色：Leader、Follower、Observer
> 状态：LOOKING、FOLLOWING、LEADING、OBSERVING
> 算法：TCP协议实现的FastLeaderElection
* 启动选举
> 1.选自己（myid,zxid）
> 2.优先比较ZXID（先比较epoch）,ZXID不一样的时候，较大的那个选票所在的服务实例作为Leader
> 3.如果两个选票的ZXID相同的话，那么就会比较myid，默认为myid较大的服务实例作为Leader
* 运行期间选举
> 1.更新自身状态
> 2.同启动选举

## 数据同步
* Follower 和 Observer 向 Leader注册；数据同步；同步确认。
* 分类：直接差异化同步（DIFF 同步）；先回滚再差异化同步（TRUNC+DIFF 同步）；仅回滚同步（TRUNC 同步）；全量同步（SNAP 同步）。

## 消息顺序性
* 采用全局递增的事务 Id 来标识，所有的 proposal（提议）都在被提出的时候加上了 zxid，zxid 实际上是一个 64 位的数字，高 32 位是 epoch（ 时期; 纪元; 世; 新时代）用来标识 leader 周期，如果有新的 leader 产生出来，epoch会自增，低 32 位用来递增计数。

## zookeeper 负载均衡和 nginx 负载均衡区别
* zk 的负载均衡是可以调控，nginx 只是能调权重，nginx 的吞吐量比 zk 大很多。
