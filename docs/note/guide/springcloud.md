## Ribbon与nginx的负载均衡

`Ribbon` 是运行在消费者端的负载均衡器,工作原理就是 `Consumer` 端获取到了所有的服务列表之后，在其**内部**使用**负载均衡算法**，进行对多个系统的调用。

![img](picture/nginx-vs-ribbon2.jpg)

`nginx`是一种**集中式**的负载均衡器,**将所有请求都集中起来，然后再进行负载均衡**

![img](picture/nginx-vs-ribbon1.jpg)

## feign

factoryBean实现

# alibaba

## sentinel

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