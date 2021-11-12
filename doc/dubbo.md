## 架构
* 第一层：service层，接口层，给服务提供者和消费者来实现的  
  第二层：config层，配置层，主要是对dubbo进行各种配置的  
  第三层：proxy层，服务代理层，透明生成客户端的stub和服务单的skeleton  
  第四层：registry层，服务注册层，负责服务的注册与发现  
  第五层：cluster层，集群层，封装多个服务提供者的路由以及负载均衡，将多个实例组合成一个服务  
  第六层：monitor层，监控层，对rpc接口的调用次数和调用时间进行监控  
  第七层：protocol层，远程调用层，封装rpc调用  
  第八层：exchange层，信息交换层，封装请求响应模式，同步转异步  
  第九层：transport层，网络传输层，抽象mina和netty为统一接口  
  第十层：serialize层，数据序列化层  

## dubbo spi 和 java spi区别

## 自定义扩展点

## 包装