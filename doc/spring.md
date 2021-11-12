# 循环依赖
| 缓存 | 说明 |
| ------- | ------- |  
| singletonObjects | 第一级缓存，存放可用的完全初始化，成品的Bean。 |  
| earlySingletonObjects | 第二级缓存，存放半成品的Bean，半成品的Bean是已创建对象，但是未注入属性和初始化。用以解决循环依赖。 |  
| singletonFactories | 第三级缓存，存的是Bean工厂对象，用来生成半成品的Bean并放入到二级缓存中。用以解决循环依赖。如果Bean存在AOP的话，返回的是AOP的代理对象。 |  
	
## 
   	
   	