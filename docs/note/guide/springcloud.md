# 微服务

微服务演进过程:

1.单点架构存在的问题: 资

源利用率差(如计算密集型程序在cpu快占满的情况下但硬盘内存使用率低);

不具备分区容错性;

查询效率低.

改进方案: 部署应用集群, 分布式缓存, 实现数据库高可用/读写分离.

2.集群模式(每个应用节点都提供全部功能, 每个存储节点提供全量数据)存在的问题: 

技术驱动非业务驱动, 业务耦合严重(每一个改动都需要重新部署);

改进方案: 服务化改造, 以业务为中心; 垂直拆分业务子系统降低耦合.

3.服务化架构(SOA): 强调对业务垂直拆分形成多个服务模块独立管理独立维护

没有统一通信标准, ESB重量级, 高成本,中间产品, 跳过.

4.微服务架构模式:将单个应用程序开发为一组小型服务的方法, 每个小服务运行在自己的进程中, 并以轻量级机制(HTTP REST API)通信

## 设计原则

单一职责 : 高内聚    每个服务只关注自己的业务, 独立 , 有界限;

服务自治 : 低耦合    每个服务独立开发&测试&构建&部署&运行等

轻量级通信 : 轻量级调用 , 跨平台, 跨语言   如Restful, 消息队列等

粒度进化 : 根据把控每个服务的粒度

## 可靠性

如果代码使用微服务与其他模块进行通信, 为了保障可靠性, 可使用的方案:

1.重试+补偿

如使用spring-retry模块便捷开发:

```
@Retryable(value = Exception.class, maxAttempts = 6,
			backoff = @Backoff(value = 1000))
public boolean xmethod(int a){
	//业务代码 和 与其他微服务模块远程通信代码
}
@Recover
public boolean doRecover(Throwable e,int a)throws ArithmeticException{
	log.error(...)
	// 补偿
	return false;
}
```

该方法表示方法抛出Exception异常时, 启用重试方案: 最多6次, 每次间隔1秒.

当超过6次重试失败后, 调用doRecover方法启用补偿方案.

2.方法调用表

数据库等持久化方法的调用状态, 并通过定时任务定期检查数据库, 对未成功的任务进行重试

# cloud

## Ribbon与nginx的负载均衡

`Ribbon` 是运行在消费者端的负载均衡器,工作原理就是 `Consumer` 端获取到了所有的服务列表之后，在其**内部**使用**负载均衡算法**，进行对多个系统的调用。

![img](picture/nginx-vs-ribbon2.jpg)

`nginx`是一种**集中式**的负载均衡器,**将所有请求都集中起来，然后再进行负载均衡**

![img](picture/nginx-vs-ribbon1.jpg)

## feign

factoryBean实现

## Gateway

网关是所有微服务的门户，路由转发仅仅是最基本的功能，除此之外还有其他的一些功能，比如：**认证**、**鉴权**、**熔断**、**限流**、**日志监控**等

# alibaba

## nacos

https://blog.csdn.net/cold___play/article/details/108032204

## sentinel

> sentinel与微服务关系不大, 将其放在这是因为它属于springcloud子项目

实现原理:

Sentinel Core为服务限流, 熔断提供了核心拦截器SentinelWebInterceptor , 这个拦截器默认对所有请求 /** 进行拦截 , 然后开始请求的链式处理流程, 在对于每一个处理请求的节点被称为slot (槽), 通过多个槽的连接形成处理链, 在请求的流转过程中, 如何有任何一个Slot验证未通过 , 都会产生BlockException ,请求处理链便会中断, 并返回"Blocked by sentinel"异常消息.

![](picture/sentinel流程.png)

这些slot中, 前3个为前置处理 , 用于收集&统计&分析必要的数据; 后4个为规则校验, 从dashboard推送的新规则保存在"规则池"中 , 然后对于slot进行读取并校验当前请求是否允许放行, 允许放行则进入下一个slot直到最终被RestController进行业务处理, 不允许放行则直接抛出BlockException返回响应.

- NodeSelectorSlot负责收集资源的路径, 并将这些资源的调用路径以树状结构存储起来, 用于根据调用路径来限流降级.
- ClusterBuilderSlot用于存储资源的统计信息已经调用者信息, 例如该资源的RT( 运行时间 )&QPS&Thread Count(线程总数)等 ,这些信息用于多维度限流与降级的依据.
- StatisticSlot用于记录&统计不同维度的runtime信息.
- SystemSlot通过系统的状态, 如CPU或内存的情况 , 来控制总流量的入口.
- AuthoritySlot根据黑白名单做黑板名单控制.
- FlowSlot根据预设的限流规则, 以及前面slot统计的状态, 来进行限流.
- DegradeSlot通过统计信息, 以及限流的规则, 来做熔断降级.

## Seata

> sentinel与微服务关系不大, 将其放在这是因为它属于springcloud子项目

seata主要三个角色:

事务管理器（TM）：决定什么时候全局提交/回滚

事务协调者（TC）：负责通知命令的中间件Seata-Server

资源管理器（RM）：做具体事的工具人

**AC模式**

AT 模式是 Seata 主推的分布式事务解决方案，对业务无侵入，真正做到了业务与事务分离，用户只需关注自己的“业务 SQL语句”。

AT模式使用起来非常简单，与完全没有使用分布式事务方案相比，业务逻辑不需要修改，只需要增加一个事务注解@GlobalTransactional即可

### TCC

TCC 模式需要用户根据自己的业务场景实现 try()、confirm() 和 cancel()
