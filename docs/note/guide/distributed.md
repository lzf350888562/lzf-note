# CAP和BASE理论

**CAP**

当发生网络分区的时候，如果我们要继续服务，那么强一致性和可用性只能 2 选 1。也就是说当网络分区之后 P 是前提，决定了 P 之后才有 C 和 A 的选择。也就是说分区容错性（Partition tolerance）我们是必须要实现的。

**为啥不可能选择 CA 架构呢？** 举个例子：若系统出现“分区”，系统中的某个节点在进行写操作。为了保证 C， 必须要禁止其他节点的读写操作，这就和 A 发生冲突了。如果为了保证 A，其他节点的读写操作正常的话，那就和 C 发生冲突了。

**如果网络分区正常的话（系统在绝大部分时候所处的状态），也就说不需要保证 P 的时候，C 和 A 能够同时保证。**

比如 ZooKeeper、HBase 就是 CP 架构，Cassandra、Eureka 就是 AP 架构，Nacos 不仅支持 CP 架构也支持 AP 架构。



**BASE**

**BASE** 是 **Basically Available（基本可用）** 、**Soft-state（软状态）** 和 **Eventually Consistent（最终一致性）** 三个短语的缩写。BASE 理论是对 CAP 中一致性 C 和可用性 A 权衡的结果，其来源于对大规模互联网系统分布式实践的总结，是基于 CAP 定理逐步演化而来的，它大大降低了我们对系统的要求。

**BASE 理论的核心思想**

即使无法做到强一致性，但每个应用都可以根据自身业务特点，采用适当的方式来使系统达到最终一致性。

> 也就是牺牲数据的一致性来满足系统的高可用性，系统中一部分数据不可用或者不一致时，仍需要保持系统整体“主要可用”。

**BASE 理论本质上是对 CAP 的延伸和补充，更具体地说，是对 CAP 中 AP 方案的一个补充。**

AP 方案只是在系统发生分区故障的时候放弃一致性，而不是永远放弃一致性。在分区故障恢复后，系统应该达到最终一致性。这一点其实就是 BASE 理论延伸的地方。

1.基本可用

基本可用是指分布式系统在出现不可预知故障的时候，允许损失部分可用性。但是，这绝不等价于系统不可用。

什么叫允许损失部分可用性呢？

- 响应时间上的损失: 正常情况下，处理用户请求需要 0.5s 返回结果，但是由于系统出现故障，处理用户请求的时间变为 3 s。
- 系统功能上的损失：正常情况下，用户可以使用系统的全部功能，但是由于系统访问量突然剧增，系统的部分非核心功能无法使用。

2.软状态

软状态指允许系统中的数据存在中间状态（**CAP 理论中的数据不一致**），并认为该中间状态的存在不会影响系统的整体可用性，即允许系统在不同节点的数据副本之间进行数据同步的过程存在延时。

3.最终一致性

最终一致性强调的是系统中所有的数据副本，**在经过一段时间的同步后，最终能够达到一个一致的状态**。因此，最终一致性的本质是需要系统保证最终数据能够达到一致，而不需要实时保证系统数据的强一致性。

> 分布式一致性的 3 种级别：
>
> 1. **强一致性** ：系统写入了什么，读出来的就是什么。
> 2. **弱一致性** ：不一定可以读取到最新写入的值，也不保证多少时间之后读取到的数据是最新的，只是会尽量保证某个时刻达到数据一致的状态。
> 3. **最终一致性** ：弱一致性的升级版，系统会保证在一定时间内达到数据一致的状态。
>

Q:如何保证最终一致性?

A:重试(MQ) , 数据校对程序(可通过定时任务实现 , 本质还是重发, 金融业务使用较多) , APM链路监控系统人工介入

# 高可用

高可用描述的是一个系统在大部分时间都是可用的，可以为我们提供服务的。高可用代表系统即使在发生硬件故障或者系统升级的时候，服务仍然是可用的。

原因:

- 黑客攻击；
- 硬件故障，比如服务器坏掉。
- 并发量/用户请求量激增导致整个服务宕掉或者部分服务不可用。
- 代码中的坏味道导致内存泄漏或者其他问题导致程序挂掉。
- 网站架构某个重要的角色比如 Nginx 或者数据库突然不可用。
- 自然灾害或者人为破坏
- ...

提高办法:

- 注重代码质量,测试严格把关

比较常见的内存泄漏、循环依赖都是对系统可用性极大的损害

- 使用集群,减少单点故障
- 限流
- 超时和重试机制设置

一旦用户请求超过某个时间的得不到响应，就抛出异常。在读取第三方服务的时候，尤其适合设置超时和重试机制。一些 RPC 框架都自带的超时重试的配置。如果不进行超时设置可能会导致请求响应速度慢，甚至导致请求堆积进而让系统无法再处理请求。重试的次数一般设为 3 次，再多次的重试没有好处，反而会加重服务器压力（部分场景使用失败重试机制会不太适合）。 

- 熔断机制

超时和重试机制设置之外，熔断机制也是很重要的。 熔断机制说的是系统自动收集所依赖服务的资源使用情况和性能指标，当所依赖的服务恶化或者调用失败次数达到某个阈值的时候就迅速失败，让当前系统立即切换依赖其他备用服务。 比较常用的流量控制和熔断降级框架是 Netflix 的 Hystrix 和 alibaba 的 Sentinel。

- 异步调用

异步调用的话我们不需要关心最后的结果，这样我们就可以用户请求完成之后就立即返回结果.

- 使用缓存
- 容灾备份

容灾:将系统所产生的的所有重要数据多备份几份。

备份:在异地建立两个完全相同的系统。当某个地方的系统突然挂掉，整个应用系统可以切换到另一个，这样系统就可以正常提供服务了。



规避单点是高可用架构设计最基本的考量.(实际上总是无法避免)

## Keepalived+VIP

**Keepalive**是Linux轻量级别的高可用解决方案.

其主要通过虚拟路由冗余(VRRP) 来实现高可用功能, 其部署和使用非常简单, 所有配置只需要一个配置文件即可完成.

虚拟路由冗余协议(VRRP)是由IETF提出的解决局域网中配置静态网关出现单点失效现象的路由协议.

> Keepalived是基于VRRP（Virtual Router Redundancy Protocol，虚拟路由器冗余协议）协议的一款高可用软件。Keepailived有一台主服务器（master）和多台备份服务器（backup），在主服务器和备份服务器上面部署相同的服务配置，使用一个虚拟IP地址对外提供服务，当主服务器出现故障时，虚拟IP地址会自动漂移到备份服务器。

**VIP(虚拟IP)**与实际网卡绑定的ip地址不同, VIP在内网中被动态的映射到不同的MAC地址上, 也就是映射到不同的机器设备上 ,keepalive通过"心跳机制"检测服务器状态, Master主节点宕机则自动将"IP漂移"到Backup备用机上实现高可用.



在两台linux上配置keepalived.conf , 主备配置只有state和priority不同.

```
vrrp_instance VI_1 {
	state MASTER			#主服务器
	interface ens33			#绑定的网卡
	virtual_router_id 51	#虚拟路由编号 (用于分组)
	priority 100			#优先级，高优先级竞选为master
	advert_int 2			#每2秒发送一次心跳包
	authentication {		#设置认证
		auth_type PASS		#认证方式，类型主要有PASS、AH 两种
		auth_pass 123456	#认证密码
	}
	virtual_ipaddress {		#设置vip
		192.168.237.5/24
	}
}
# ---------------------
vrrp_instance VI_1 {
	state BACKUP			#备服务器
	interface ens33			#绑定的网卡
	virtual_router_id 51	#虚拟路由编号 (用于分组)
	priority 99			    #优先级，高优先级竞选为master
	advert_int 2			#每2秒发送一次心跳包
	authentication {		#设置认证
		auth_type PASS		#认证方式，类型主要有PASS、AH 两种
		auth_pass 123456	#认证密码
	}
	virtual_ipaddress {		#设置vip
		192.168.237.5/24
	}
}
```

设置完后master设置vip为237.5,  master的keepalived每两秒发送一次VRRP心跳包给backup, 让backup知道其还在工作 ,如果超时未接收到, 则需要keepalived进行vip漂移.

> 因为设置了优先级 , 所以即使master中途宕机,vip漂移到backup后,master恢复时, vip会自动漂移回原master, 新master自动降级回backup

### Nginx故障切换

在通常的一nginx多web服务器的架构下(反向代理+负载均衡) ,

为了实现nginx高可用, 可以再配置一台nginx , 并通过配置keepalived+VIP实现高可用.

当主服务器宕机, vip即可漂移到备服务器.

Q:如果当主服务器未宕机, 但其运行的nginx挂了,此时keepalived还是能发送心跳包所以不会发送漂移.

所以**keepalived如何观测nginx是否有效是故障切换的重点**.

通过给keepalived指定脚本观测nginx.

```conf
vrrp_script check_nginx{
	script "/etc/keepalived/nginx_check.sh"		#nginx服务器检查脚本
	interval 2 									#触发间隔
	weight 1									#权重
}
```

nginx_check.sh (手写脚本 , 有更好的方案 , pid文件不一定会消失)

```sh
#!/bin/bash
#检查nginx的pid文件是否存在 不存在时 killall keepalived VIP漂移
NGINXPID="/usr/install/nginx/logs/nginx.pid"
if [ ! -f $NGINXPID ];then
	killall keepalived
fi
```

效果就是如果发现nginx挂掉, 就kill keepalived进程, 自然不会发送心跳包了

### 互为主备+DNS轮询

上述情况下 , 在第一台nginx没挂之前, 第二台nginx都是闲置状态.

为了让另外一台nginx也投入使用,  可对两台nginx设置互为主备: 通过再加入一组VIP配置, vip为237.6

```
vrrp_instance VI_2 {  		#修改处1
	state BACKUP			#修改处2
	interface ens33			
	virtual_router_id 52	#修改处3
	priority 199			#修改处4
	advert_int 2			
	authentication {		
		auth_type PASS		
		auth_pass 123456	
	}
	virtual_ipaddress {		
		192.168.237.6/24	#修改处5
	}
}
//-------------------
vrrp_instance VI_2 {  		#修改处1
	state MASTER			#修改处2
	interface ens33			
	virtual_router_id 52	#修改处3
	priority 200			#修改处4
	advert_int 2			
	authentication {		
		auth_type PASS		
		auth_pass 123456	
	}
	virtual_ipaddress {		
		192.168.237.6/24	#修改处5
	}
}
```

Q:如何让两个vip都能被访问且负载均衡呢?

A:利用DNS轮询机制给一个域名绑定这两个VIP , 而任一nginx挂掉, 另一个nginx服务器都同时持有237.5和237.6的VIP , 都不影响使用.

DNS轮询缺点:

1.只负责IP轮询获取,不保证节点可用;

2.DNS IP列表更新有延迟;

3.外网IP占用严重(所以中间通过nginx转发);

4.安全性降低

### 其他问题

1. 每次在原master挂掉又恢复后, 都要进行来回两次VIP漂移, 节点会出现短暂不可提供.

Q:如何让原backup称为主以后干脆一直使用原backup作为主?

A:设置nopreempt非抢占模式 , 只需要加入一个配置即可

```
vrrp_instance VI_1 {
	state BACKUP			
	interface ens33			
	virtual_router_id 51	
	nopreempt			#非抢占模式
	priority 99			    
	advert_int 2			
	authentication {		
		auth_type PASS		
		auth_pass 123456	
	}
	virtual_ipaddress {		
		192.168.237.5/24
	}
}
```

2. 如果两台服务器中发送心跳的网络出现问题, 这种情况下backup因为接收不到master的心跳而提升为主, 从而出现两个相同的IP(脑裂)?

因为问题发生在网络, 所以首先需要提高局域网可用性.

禁止pkill -9 keepalived (强制关闭, keepalive不会回收VIP) ,使用pkill keepalived正常结束.

解决网络问题后, 在备机上需要 systemctl restart network 重启网络对ip重置 去掉VIP.

## 限流

### 限流算法

1. 固定窗口计数器算法;

 固定窗口计数器算法规定了我们单位时间处理的请求数量

![固定窗口计数器算法](picture/8ded7a2b90e1482093f92fff555b3615.png)

**这种限流算法无法保证限流速率(临界问题, 比如前一个时间窗格的最后和后一个时间窗格的开始都处理了大量请求,虽然都没有超过阈值, 但在这两个时间点内请求数量却超过了阈值)，因而无法保证突然激增的流量。**

> 可通过redis的incr原子增加简单实现

2.滑动窗口计数器算法;

为了解决固定窗口计数器算法的临界值问题, TCP网络通信协议中就是采用该算法来解决网络拥堵问题。

滑动窗口计数器算法相比于固定窗口计数器算法的优化在于：**它把时间以一定比例分片** 。

滑动时间窗口将计数器中的实际周期切分为多个小的时间窗口, 分别在每个小时间窗口中记录访问次数, 然后根据时间将窗口往前滑动并删除过期的小时间窗口, 最终只需要统计滑动窗口范围内的小时间窗口的总的请求次数即可.

**当滑动窗口的小窗口划分的越多，滑动窗口的滚动就越平滑，限流的统计就会越精确。**

> 可通过前后指针算法实现
>
> ```c++
> int left = 0, right = 0;
> while (right < s.size()) {`
> 	// 增⼤窗⼝
> 	window.add(s[right]);
> 	right++;
> 	while (window needs shrink) {
> 		// 缩⼩窗⼝
> 		window.remove(s[left]);
> 		left++;
> 	}
> }
> ```
> 

> redis的zset也可以实现滑动窗口, 通过scope配合圈出时间窗口, value为请求的唯一标识,  窗口外的记录全部删除.

3.漏桶算法

我们可以把发请求的动作比作成注水到桶中，我们处理请求的过程可以比喻为漏桶漏水。我们往桶中以任意速率流入水，以一定速率流出水。当水超过桶流量则丢弃，因为桶容量是不变的，保证了整体的速率。

如果想要实现这个算法的话也很简单，准备一个队列用来保存请求，然后我们定期从队列中拿请求来执行就好了（和消息队列削峰/限流的思想是一样的）。![漏桶算法](picture/75938d1010138ce66e38c6ed0392f103.png)

漏桶算法能强行限制数据的传输速率(漏水速度) , 所以**当系统在短时间内有突发的大流量时, 漏桶算法处理不了**

> 可通过redis-cell模块插件实现, 仅一条命令: 
>
> ```
> > cl.throttle key 15 30 60  #表示漏斗初始容量为15, 每60s频率为30
> 1) (integer) 0 # 0 表示允许，1 表示拒绝
> 2) (integer) 15 # 漏斗容量 capacity
> 3) (integer) 14 # 漏斗剩余空间 left_quota
> 4) (integer) -1 # 如果拒绝了，需要多长时间后再试(漏斗有空间了，单位秒)
> 5) (integer) 2 # 多长时间后，漏斗完全空出来(left_quota==capacity，单位秒)
> ```

4.令牌桶算法

和漏桶算法一样，主角还是桶。不过现在桶里装的是令牌了，请求在被处理之前需要拿到一个令牌，请求处理完毕之后将这个令牌丢弃（删除）。我们**根据限流大小，按照一定的速率往桶里添加令牌**。如果桶装满了，就不能继续往里面继续添加令牌了(丢弃)。 

![令牌桶算法](picture/eca0e5eaa35dac938c673fecf2ec9a93.png)

由于桶的作用, 该算法**可以处理短时间大流量的场景, 但不能超过桶的大小**.



### 限流工具

**单机限流工具**

单机限流可以直接使用 Google Guava 自带的限流工具类 `RateLimiter` 。 `RateLimiter` 基于令牌桶算法，可以应对突发流量。

除了最基本的令牌桶算法(平滑突发限流)实现之外，Guava 的`RateLimiter`还提供了 **平滑预热限流** 的算法实现。

平滑突发限流就是按照指定的速率放令牌到桶里，而平滑预热限流会有一段预热时间，预热时间之内，速率会逐渐提升到配置的速率。

```
<dependency>
    <groupId>com.google.guava</groupId>
    <artifactId>guava</artifactId>
    <version>31.0.1-jre</version>
</dependency>
```

```
 // 1s 放 5 个令牌到桶里也就是 0.2s 放 1个令牌到桶里
        RateLimiter rateLimiter = RateLimiter.create(5);
        for (int i = 0; i < 10; i++) {
            double sleepingTime = rateLimiter.acquire(1);
            System.out.printf("get 1 tokens: %ss%n", sleepingTime);
        }
//输出
get 1 tokens: 0.0s
get 1 tokens: 0.188413s
get 1 tokens: 0.197811s
get 1 tokens: 0.198316s
get 1 tokens: 0.19864s
get 1 tokens: 0.199363s
get 1 tokens: 0.193997s
get 1 tokens: 0.199623s
get 1 tokens: 0.199357s
get 1 tokens: 0.195676s
```

平滑预热限流

```
// 1s 放 5 个令牌到桶里也就是 0.2s 放 1个令牌到桶里
// 预热时间为3s,也就说刚开始的 3s 内发牌速率会逐渐提升到 0.2s 放 1 个令牌到桶里
        RateLimiter rateLimiter = RateLimiter.create(5, 3, TimeUnit.SECONDS);
        for (int i = 0; i < 20; i++) {
            double sleepingTime = rateLimiter.acquire(1);
            System.out.printf("get 1 tokens: %sds%n", sleepingTime);
        }
//输出
get 1 tokens: 0.0s
get 1 tokens: 0.561919s
get 1 tokens: 0.516931s
get 1 tokens: 0.463798s
get 1 tokens: 0.41286s
get 1 tokens: 0.356172s
get 1 tokens: 0.300489s
get 1 tokens: 0.252545s
get 1 tokens: 0.203996s
get 1 tokens: 0.198359s
```

单机限流工具还有Bucket4j和Resilience4j

Spring Cloud Gateway 中自带的单机限流的早期版本就是基于 Bucket4j 实现的。后来，替换成了Resilience4j .

Resilience4j 不仅提供限流，还提供了熔断、负载保护、自动重试等保障系统高可用开箱即用的功能。并且，Resilience4j 的生态也更好，很多网关都使用 Resilience4j 来做限流熔断的。



**分布式限流工具**

- 借助中间件限流: 如sentinel, 或使用redis手动实现(可配合lua脚本)限流逻辑
- 网关限流

netflix zuul

spring cloud gateway

kong

apisix

shenyu



## 降级&熔断

降级是从系统功能优先级的角度考虑如何应对系统故障。

服务降级指的是当服务器压力剧增的情况下，根据当前业务情况及流量对一些服务和页面有策略的降级，以此释放服务器资源以保证核心任务的正常运行。

熔断和降级是两个比较容易混淆的概念，两者的含义并不相同。

降级的目的在于应对系统自身的故障，而熔断的目的在于应对当前系统依赖的外部系统或者第三方系统的故障。

## Raft选举算法

Raft 算法是分布式系统开发首选的共识算法。 主要在分布式集群架构下进行领导者（主节点）的确认。

现在流行的组件 Etcd、Consul、Nacos、RocketMQ、Redis Sentinel 底层都是采用Raft算法来确 认集群中的主节点，再通过主节点向其他节点下发指令。

**Raft角色**

跟随者（Follower）：普通群众，默默接收和来自领导者的消息，当领导者心跳 信息超时的时候，就主动站出来，推荐自己当候选人。 

候选人（Candidate）：候选人将向其他节点请求投票 RPC 消息，通知其他节点来投票，如果赢得了大多数投票选票，就晋升当领导者。 

领导者（Leader）：霸道总裁，一切以我为准。处理写请求、管理日志复制和 不断地发送心跳信息，通知其他节点“我是领导者，我还活着，你们不要发起新的选举，不用找新领导来替代我”。

**Raft选举过程**

1.初始状态

初始状态下，集群中所有节点都是跟随者的状态 , 任期（Term）都为 0

Raft 算法实现了随机超时时间的特性，每个节点等待领导者节点心跳信息的超时时间间隔是随机的(防止所有节点同一时间向其他节点发起投票)。

比如 A 节点等待超时的时间间隔 150 ms，B 节点 200 ms，C 节点 175 ms。 那么 a 先超时，最先因为没有等到领导者的心跳信息，发生超时。

2.发起投票

当 A 节点的超时时间到了后，A 节点成为候选者，并增加自己的任期编号，Term 值从 0 更新为 1，并给自己投了一票( , Vote Count = 1 ).

3.成为领导者的简化过程

```
1. 节点 A 成为候选者后，向其他节点发送请求投票 RPC 信息，请它们选举自己为领导者。
2. 节点 B 和 节点 C 接收到节点 A 发送的请求投票信息后，在编号为 1 的这届任期内，还没有进行过投票，就把选票投给节点 A，并增加自己的任期编号(更新为1)。
3. 节点 A 收到 3 次投票，得到了大多数节点（n/2+1)的投票，从候选者成为本届任期内的新的领导者。
4. 节点 A 作为领导者，固定的时间间隔给 节点 B 和节点 C 发送心跳信息，告诉节点 B 和 C，我是领导者。
5. 节点 B 和节点 C 发送响应信息给节点 A，告诉节点 A 我是正常的。
```

4.领导者的任期解释

英文单词是 term，领导者是有任期的。

- 自动增加：跟随者在等待领导者心跳信息超时后，推荐自己为候选人，会增加自 己的任期号，如第二个过程时，节点 A 任期为 0，推举自己为候选人时，任期编号增加为 1。
- 更新为较大值：当节点发现自己的任期编号比其他节点小时，会更新到较大的编号值。比如节点 A 的任期为 1，请求投票，投票消息中包含了节点 A 的任期编号， 且编号为 1，节点 B 收到消息后，会将自己的任期编号更新为 1。 
- 恢复为跟随者：如果一个候选人或者领导者，发现自己的任期编号比其他节点 小，那么它会立即恢复成跟随者状态。这种场景出现在**分区错误恢复**后，任期为 3 的领导者受到任期编号为 4 的心跳消息，那么前者将立即恢复成跟随者状态。 
- 拒绝消息：如果一个节点接收到较小的任期编号值的请求，那么它会直接拒绝这 个请求，比如任期编号为 6 的节点 A，收到任期编号为 5 的节点 B 的请求投票 RPC 消息，那么节点 A 会拒绝这个消息。
- 一个任期内，领导者一直都会领导者，直到自身出现问题（如宕机），或者网络问题（延迟），其他节点发起一轮新的选举。

5.防止多个节点同时发起投票

为了防止多个节点同时发起投票，会给每个节点分配一个随机的选举超时时间。这个时间内，节点不能成为候选者，只能等到超时。比如上述例子，节点 A 先超时，先成为了候选者。这种巧妙的设计，在大多数情况下只有一个服务器节点先发起选举，而不是同时发起选举，减少了因选票瓜分导致选举失败的情况。

6.触发新的一轮选举 如果领导者节点出现故障，则会触发新的一轮选举。当领导者节点 A 发生故 障，节点 B 和 节点 C 就会重新选举 Leader。

```
1. 节点 A 发生故障，节点 B 和节点 C 没有收到领导者节点 A 的心跳信息，等待超时。
2. 节点 C (175ms) 先发生超时，节点 C 成为候选人。
3. 节点 C 向节点 A 和 节点 B 发起请求投票信息。
4. 节点 C 响应投票，将票投给了 C，而节点 A 因为发生故障了，无法响应 C 的投票请求。
5. 节点 C 收到两票（大多数票数），成为领导者。
6. 节点 C 向节点 A 和 B 发送心跳信息，节点 B 响应心跳信息，节点 A 不响应心跳信息。
7. 节点 A 恢复后，收到节点 C 的高任期消息，自身将为跟随者，接收节点 C 的消息。
```





# 数据一致性

分布式锁是解决并发时资源争抢的问题，分布式事务和本地事务是解决流程化提交问题。

## 分布式事务

分布式事务就是指事务的资源分别位于不同的分布式系统的不同节点之上的事务

解决方案有:2PC  TCC 消息事务

### 二阶段提交 2PC

二阶段提交指为了使基于分布式系统架构下的所有节点在进行事务提交时保持一致性而设计的算法. 具体思路为: 参与者将操作成败通知协调者,再由协调者根据所有参与者的反馈情况决定各参与者是否要提交操作还是中止操作.

实现了XA规范(定义了TM和RM)

**准备阶段**: 事务协调者(事务管理器)给每个参与者(资源管理器)发送Prepare消息, 每个参与者要么直接返回失败,要么在本地执行事务并写本地redo和undo日志,但不提交,达到"万事俱备只欠东风"的状态.

![image-20211205164829308](picture/image-20211205164829308.png)

**提交(回滚)阶段**:如果协调者收到了参与者的失败消息或者超时, 直接给每个参与者发送回滚消息; 否则发送提交消息. 参与者再根据协调者的指令执行提交或回滚操作, 释放所有事务处理过程中使用的锁资源 

![image-20211205165027109](picture/image-20211205165027109.png)

缺点:

1.同步阻塞: 执行过程中,所有参与者节点都是事务阻塞型的.  当参与者占有公共资源时, 其他第三方节点访问公共资源不得不处于阻塞状态(本地事务排他锁),且占用数据库连接资源, 因为没有提交, 严重会导致崩溃.

2.单点故障: 因为协调者的重要性, 一旦其发送故障. 参与者会一直阻塞下去. 特别是在第二阶段的时候(选举新协调者通常也无法解决该问题).

3.数据不一致: 在二阶段提交的时候, 协调者向参与者发送commit请求后发生了局部网络异常或协调者中途出现故障, 会导致只有一部分参与者接收到commit请求执行操作, 整个系统出现数据不一致现象.

解决方案:  手动补偿(TCC)或脚本补偿.

### 三阶段提交 3PC

**CanCommit阶段**: 

1.事务询问: 协调者向参与者发送CanCommit请求, 询问是否可以执行事务提交操作, 然后等待参与者的响应.

2.响应反馈: 参与者接收到CanCommit请求后, 正常情况下如果其**认为**可以顺利执行事务, 则返回Yes响应, 并进入预备状态, 否则返回No.

**PreCommit阶段** : 

如果协调者从所有参与者获得的返回都是Yes响应, 那么就开始事务的预执行:

1.发送预提交请求: 协调者向参与者发送PreCommit请求, 并进入Prepared状态.

2.事务预提交:参与者接收到PreCommit请求后, 会执行事务操作, 并将undo和redo信息记录到事务日志中.

3.响应反馈: 如果参与者成功的执行了事务操作, 则返回ACK响应, 同时开始等待最终指令.

如果任何一个参与者向协调者发送了No响应, 或等待超时之后, 协调者都没有收到参与者的响应, 那么就执行事务的中断.

1.发送中断请求: 协调者向所有参与者发送abort请求.

2.中断事务: 参与者收到来自协调者的abort请求之后(或超时) , 执行事务的中断

**DoCommit阶段**

如果协调者从所有参与者获得的返回都是ACK响应, 执行**事务提交**:

1.发送提交请求: 协调者收到参与者发送的ACK响应后将从预提交状态进入提交状态. 并向所有参与者发送DoCommit请求.

2.事务提交: 参与者接收到doCommit请求之后, 执行正式的事务提交. 并在完成事务提交之后释放所有事务资源.

3.响应反馈: 事务提交完之后, 向协调者发送ACK响应.

4.完成事务: 协调者接收到所有参与者的ACK响应后, 完成事务.

如果协调者接收到参与者发送的响应不是ACK,或响应超时, 执行**中断事务**

1.发送中断请求:协调者向所有参与者发送abort请求.

2.事务回滚: 参与者收到abort请求(或超时)之后, 利用其在阶段二记录的undo信息执行事务的回滚操作, 并在完成回滚之后释放掉所有的事务资源.

3.反馈结果: 参与者完成事务回滚之后, 向协调者发送ACK消息.

4.中断事务: 协调者接收到参与者反馈的ACK响应后, 执行事务的中断.



**2PC与3PC的区别**:

1.3PC比2PC多了CanCommit阶段, 不占用资源只校验一下sql, 如果不能执行可直接中断, 进而减少了不必要的资源浪费.

2.3PC在PreCommit和DoCommit阶段对参与者增加了超时机制(防止阻塞), 而2PC只有协调者有超时机制.

> 虽然引入了参与者超时机制, 防止数据库长时间阻塞, 但会破坏一致性. 通常做法为异步补偿, 日终补偿, 完整性校验, 人工补录.

### TCC

**Try-Confirm-Cancel**: 又称为二阶段补偿事务, 2PC的-prepare-commit-rollback对应TCC的try-confirm-cancel, 区别在于2PC对于开发者层面是无法感知的, 给数据库做资源操作; 而TCC是站在开发者层面的, 三个阶段对资源的操作需要开发者自己实现, 通常通过修改表结构增加字段实现. 

Try: 业务检查阶段, 主要进行业务校验和检查或者资源预留, 也可能是直接进行业务操作.

Confirm: 业务确认阶段, 对Try阶段校验过的业务或预留资源进行确认.

Cancel: 业务回滚阶段, 和Confirm是互斥的, 用于释放Try阶段预留资源业务.

**TCC如何解决一致性问题**?

比如在A给B转账100元的场景下的简单流程:

**Try阶段**
A: 余额减100, 冻结字段加100;
B: 冻结字段加100
**confirm阶段**
A: 冻结字段减100;
B:冻结字段减100, 余额加100
**cancel阶段**
A: 余额加100, 冻结字段减100;
B:冻结字段减100



在销售与库存业务中, 也可以升级为:

**Try阶段**
订单服务:修改订单的状态为**支付中**
账户服务:账户余额不变，可用余额减1，然后将1这个数字冻结在一个单独的字段里
库存服务:库存数量不变，可销售库存减1，然后将1这个数字冻结在一个单独的字段里
**confirm阶段**
订单服务:修改订单的状态为**支付完成**
账户服务:账户余额变为(当前值减冻结字段的值)，可用余额不变(Try阶段减过了),冻结字段清0。
库存服务:库存变为(当前值减冻结字段的值)，可销售库存不变(Try阶段减过了)，冻结字段清0。
**cancel阶段**
订单服务:修改订单的状态为**未支付**
账户服务:账户余额不变，可用余额变为(当前值加冻结字段的值)，冻结字段清0。
库存服务:库存不变，可销售库存变为(当前值加冻结字段的值)，冻结字段清0。



**TCC存在的问题**

**TCC空回滚**: 指的是没有调用TCC资源Try方法的情况下, 调用了二阶段的Cancel方法. 

比如当Try请求由于网络延迟或故障等原因, 没有对数据执行修改而返回异常, 这时如果Cancel进行了对数据的修改会导致数据不一致.

解决思路的关键是要识别出这个空回滚: 需要知道Try阶段是否执行, 如果执行了就正常回滚, 否则空回滚.

可通过分支**事务记录表**实现, 在第一阶段Try中会插入一条记录表示Try阶段执行了. Cancel中读取该记录, 如果存在, 则正常回滚, 否则空回滚.

**TCC悬挂问题**: 指的是二阶段Cancel比Try先执行.

出现的原因为调用分支事务Try时, 由于网络拥堵造成超时, TM就会通知RM回滚该事务, 在回滚完成后, Try请求才到达参与者执行完成.

对于这种情况下Try阶段预留的业务资源就再也没有人能处理了 , 所以称为空悬挂.

可通过**分支事务表**实现, 在执行Try前, 判断当前全局事务下是否有二阶段事务记录, 如果有则不执行Try.

**幂等性问题**: 当Try方法之后的Confirm或Cancel失败, 会触发Retry对其他们进行重试, 因此需要保证TCC下的二阶段Confirm和Cancel接口幂等性.

可通过**分支事务表**记录执行状态, 每次二阶段执行前查看该状态. 

> TCC设计之初就认为C/C一定会成功, 但事实并不如意. 可以将大部分工作放到try阶段, 尽量少的工作放到C/C阶段.

### 消息事务

通过可靠消息服务实现最终一致性: 事务发起方执行完本地事务后, 发出一条消息, 事务参与方一定能够接收消息并可以成功处理自己的事务.

需要通过[本地消息表](./middleware#本地消息表保障最终一致性)保证事务一致性

### Seata

seata的设计为分布式事务设计理念中的二阶段提交.



事务管理器（TM）：决定什么时候全局提交/回滚  --> @GlobalTransactionl

事务协调者（TC）：负责通知命令的中间件            --> Seata-Server

资源管理器（RM）：做具体事的工具人				  --> @Transactional

1.通过添加seata核心注解@GlobalTransactional注解开启全局事务 , TM通知TC向下通达给RM开启本地事务

![image-20211205170142581](picture/image-20211205170142581.png)

![image-20211205170117506](picture/image-20211205170117506.png)

2.待本地事务都**提交**完成后,TM通过TC向RM下达全局事务处理结果.

![image-20211205170519113](picture/image-20211205170519113.png)

![image-20211205170857133](picture/image-20211205170857133.png)



Q:如果事务中间阶段出了问题, 而在RM处理本地子事务时,处理完成后是直接写表提交, 在TC下达分支结果时,是如何实现回滚的?

以主流的AT模式为例 , Seata AT模式下如何实现数据自动提交、回滚?

seata AT通过在所有数据库增加一张UNDO_LOG表.

> seata AT通过sql parser第三方jar包生成逆向sql , 存储在UNDO_LOG表中.  如:
>
> insert into 订单表 values(1,...);   -->  delete from 订单表 where id = 1; 
>
> update 会员积分表 set point = 50 where pid=1   --> update 会员积分表 set point = 40 where pid=1 

如果收到TC下达的分支提交, 则删掉UNDO_LOG中对于的记录即可;

如果收到TC下达的分支回滚, 执行UNDO_LOG中的**逆向SQL**,还原年数据.

Q: Seata如何避免并发场景的脏读与脏写?

利用**TC**自带的**分布式锁**完成:

![image-20211205172341582](picture/image-20211205172341582.png)

Q: 怎么使用Seata框架，来保证事务的隔离性？

因seata一阶段本地事务已提交，为防止其他事务脏读脏写需要加强隔离。

1.脏读select语句加for update，代理方法增加@GlobalLock+@Transactional或@GlobalTransaction

2.脏写 必须使用@GlobalTransaction

注：如果你查询的业务的接口没有GlobalTransactional包裹，也就是这个方法上压根没有分布式事务的
需求，这时你可以在方法上标注@GlobalLock+@Transactional 注解，并且在查询语句上加 for update。
如果你查询的接口在事务链路上外层有GlobalTransactional注解，那么你查询的语句只要加for update就
行。设计这个注解的原因是在没有这个注解之前，需要查询分布式事务读已提交的数据，但业务本身不
需要分布式事务。若使用GlobalTransactional注解就会增加一些没用的额外的rpc开销比如begin返回
xid，提交事务等。GlobalLock简化了rpc过程，使其做到更高的性能。

## 分布式锁

适用于金融交易等场合

在分布式系统下 ,由于每个服务部署在独立的机器, 运行的应用更是独立的进程, 传统的上锁因为只针对一个jvm中多个线程起到同步作用, 因此是无效的.

分布式锁的主要实现方案:

| 分类               | 方案                                         | 实现原理                                                     | 优点                                                         | 缺点                                                         |
| ------------------ | -------------------------------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ | ------------------------------------------------------------ |
| 基于数据库         | 基于mysql 表唯一索引                         | 1.表增加唯一索引 2.加锁：执行insert语句，若报错，则表明加锁失败(多次插入--自旋) 3.解锁：执行delete语句 | 完全利用DB现有能力，实现简单                                 | 1.锁无超时自动失效机制，有死锁风险 2.不支持锁重入，不支持阻塞等待 3.操作数据库开销大，性能不高 |
|                    | 基于MongoDB findAndModify原子操作            | 1.加锁：执行findAndModify原子命令查找document，若不存在则新增 2.解锁：删除document | 实现也很容易，较基于MySQL唯一索引的方案，性能要好很多        | 1.大部分公司数据库用MySQL，可能缺乏相应的MongoDB运维、开发人员 2.锁无超时自动失效机制 |
| 基于分布式协调系统 | 基于ZooKeeper                                | 1.加锁：在/lock目录下创建临时有序节点，判断创建的节点序号是否最小。若是，则表示获取到锁；否则watch /lock目录下序号比自身小的前一个节点 2.解锁：删除节点 | 1.由zk保障系统高可用 2.Curator框架已原生支持系列分布式锁命令，使用简单 | 1.需单独维护一套zk集群，维保成本高 2.需要时间创建和释放节点保证强一致性, 浪费时间 3.如果客户端与zk之间存在网络问题, session超时同样会断开导致锁释放而无法完全保证一致 |
| 基于缓存           | 基于redis命令(worker单线程的串行)            | 1. 加锁：执行setnx(若以存在,多次执行--自旋)，若成功再执行expire添加过期时间 (如果key到期但事务还未处理完需要考虑延期问题)2. 解锁：执行delete命令 | 实现简单性能好,通常单独一台用于锁的redis(可用性可以通过冗余设备:电源/网卡等)与业务redis分开 | 1.setnx和expire分2步执行，非原子操作；若setnx执行成功，但expire执行失败，就可能出现死锁 (现在有setnxex可避免该情况)2.delete命令存在误删除非当前线程持有的锁的可能(延迟删除以后再进行了delete,可通过唯一value解决, 但是匹配与删除两个操作不为原子操作) 3.不支持阻塞等待、不可重入 4.主从同步(AP)可能出现锁丢失问题 |
|                    | 基于redis Lua脚本能力(如jedis.eval(lua文本)) | 1. 加锁：执行SET lock_name random_value EX seconds NX 命令  2. 解锁：执行Lua脚本，释放锁时验证random_value防止误解锁 (ARGV[1]为random_value, KEYS[1]为lock_name) if redis.call("get", KEYS[1]) == ARGV[1] then  return redis.call("del",KEYS[1]) else  return 0 end | 配合Lua脚本保证连续多个指令原子性, 加入随机value, 释放锁时进行验证, 防止误解锁.  这个随机值也可以使用userid等唯一值. | 同上；实现逻辑上也更严谨，除了单点问题，生产环境采用用这种方案，问题也不大。存在锁自动释放问题, 不支持锁重入，不支持阻塞等待 |

> redis+ThreadLocal封装可实现重入锁: 仅第一次set成功, 后续重入通过ThreadLocal计数 

适用分布式锁有以下几个场景： 

数据价值大，必须要保证一致性的。例如：金融业务系统间的转账汇款等。 

总结：重要的但对并发要求不高的系统可以使用分布式锁，对于并发量高、数据价值小、 对一致性要求没那么高的系统可以进行最终一致性(BASE)处理，保证并发的前提下通过重试、程序矫正、人工补录的方式进行处理。

> 为了减少分布式锁中间件的IO性能, 可以配合jvm锁同时使用, 减少落到分布式锁上的线程数.

> 分布式锁有两大类:
>
> 1. 类似CAS自旋式分布式锁: mysql, redis
> 2. event事件通知: zk, etcd
>
> 结合分布式情况下的IO性能问题, 第2种锁性能更高

> 实际中也要考虑网络通信对分布式锁的影响, 如多个云服务器的锁调用肯定比同一局域网中调用慢很多(上千倍).

### Redission

对redis实现锁进行了封装

单机模式:

```
// 构造redisson实现分布式锁必要的Config
Config config = new Config();
config.useSingleServer().setAddress("redis://172.29.1.180:5379").setPassword("a123456").setDatabase(0);
// 构造RedissonClient
RedissonClient redissonClient = Redisson.create(config);
// 设置锁定资源名称
RLock disLock = redissonClient.getLock("DISLOCK");
boolean isLock;
try {
    //尝试获取分布式锁
    isLock = disLock.tryLock(500, 15000, TimeUnit.MILLISECONDS);
    if (isLock) {
        //TODO if get lock success, do something;
        Thread.sleep(15000);
    }
} catch (Exception e) {
} finally {
    // 无论如何, 最后都要解锁
    disLock.unlock();
}
```

实现为Lua脚本:

KEYS[1] : 锁名

KEYS[2] : 解锁消息PubSub频道

ARGV[1] : 重入数量 , 0标识解锁消息.	

ARGV[2] : 持有锁的有效时间,默认30s

ARGV[3] : 唯一标识,获取锁时set的唯一值 , 客户端ID

1.加锁

```
-- 若锁不存在：则新增锁，并设置锁重入计数为1、设置锁过期时间
if (redis.call('exists', KEYS[1]) == 0) then
    redis.call('hset', KEYS[1], ARGV[2], 1);
    redis.call('pexpire', KEYS[1], ARGV[1]);
    return nil;
end;

-- 若锁存在，且唯一标识也匹配：则表明当前加锁请求为锁重入请求，故锁重入计数+1，并再次设置锁过期时间
if (redis.call('hexists', KEYS[1], ARGV[2]) == 1) then
    redis.call('hincrby', KEYS[1], ARGV[2], 1);
    redis.call('pexpire', KEYS[1], ARGV[1]);
    return nil;
end;

-- 若锁存在，但唯一标识不匹配：表明锁是被其他线程占用，当前线程无权解他人的锁，直接返回锁剩余过期时间
return redis.call('pttl', KEYS[1]);
```

2.解锁

```
-- 若锁不存在：则直接广播解锁消息，并返回1
if (redis.call('exists', KEYS[1]) == 0) then
    redis.call('publish', KEYS[2], ARGV[1]);
    return 1; 
end;
 
-- 若锁存在，但唯一标识不匹配：则表明锁被其他线程占用，当前线程不允许解锁其他线程持有的锁
if (redis.call('hexists', KEYS[1], ARGV[3]) == 0) then
    return nil;
end; 
 
-- 若锁存在，且唯一标识匹配：则先将锁重入计数减1
local counter = redis.call('hincrby', KEYS[1], ARGV[3], -1); 
if (counter > 0) then 
    -- 锁重入计数减1后还大于0：表明当前线程持有的锁还有重入，不能进行锁删除操作，但可以友好地帮忙设置下过期时期
    redis.call('pexpire', KEYS[1], ARGV[2]); 
    return 0; 
else 
    -- 锁重入计数已为0：间接表明锁已释放了。直接删除掉锁，并广播解锁消息，去唤醒那些争抢过锁但还处于阻塞中的线程
    redis.call('del', KEYS[1]); 
    redis.call('publish', KEYS[2], ARGV[1]); 
    return 1;
end;
 
return nil;
```



### Redlock

在不管是redis命令、redis+lua还是redission, 这类琐最大的缺点就是它加锁时只作用在一个Redis节点上，即使Redis通过sentinel保证高可用，如果这个master节点由于某些原因发生了主从切换，那么就会出现锁丢失的情况

比如, 应用在Redis的master节点上拿到了锁, 但是这个加锁的key还没有同步到slave节点, 这时master故障，发生故障转移，slave节点升级为master节点, 从而导致锁丢失。

Redis作者antirez基于分布式环境下提出了一种更高级的分布式锁的实现方式：**Redlock**。

该锁由client实现而非redis实现.

具体实现为在redis分布式环境中,对于N个**完全独立**的节点:

1.依次尝试从N个节点使用相同的key和**具有唯一性的value**获取锁, 并**设置超时时间**;

2.当且仅当从**大多数的Redis节点**都取到锁，并且使用的时间小于锁失效时间时，锁才算获取成功。

>  redisson对redlock进行了实现:`RedissonRedLock

```
Config config1 = new Config();
config1.useSingleServer().setAddress("redis://172.29.1.180:5378")
        .setPassword("a123456").setDatabase(0);
RedissonClient redissonClient1 = Redisson.create(config1);

Config config2 = new Config();
config2.useSingleServer().setAddress("redis://172.29.1.180:5379")
        .setPassword("a123456").setDatabase(0);
RedissonClient redissonClient2 = Redisson.create(config2);

Config config3 = new Config();
config3.useSingleServer().setAddress("redis://172.29.1.180:5380")
        .setPassword("a123456").setDatabase(0);
RedissonClient redissonClient3 = Redisson.create(config3);

String resourceName = "REDLOCK";
RLock lock1 = redissonClient1.getLock(resourceName);
RLock lock2 = redissonClient2.getLock(resourceName);
RLock lock3 = redissonClient3.getLock(resourceName);

RedissonRedLock redLock = new RedissonRedLock(lock1, lock2, lock3);
boolean isLock;
try {
    isLock = redLock.tryLock(500, 30000, TimeUnit.MILLISECONDS);
    System.out.println("isLock = "+isLock);
    if (isLock) {
        //TODO if get lock success, do something;
        Thread.sleep(30000);
    }
} catch (Exception e) {
} finally {
    // 无论如何, 最后都要解锁
    System.out.println("");
    redLock.unlock();
}
```

**Watch Dog自动续租机制**

在上锁成功后，Redisson会调用renewExpiration()方法开启一个Watch Dog线程，为锁自动续期。每过1/3时间续一次，成功则继续下一次续期，失败取消续期操作:

```java
private void renewExpiration() {
    ExpirationEntry ee = EXPIRATION_RENEWAL_MAP.get(getEntryName());
    if (ee == null) {
        return;
    }

    Timeout task = commandExecutor.getConnectionManager().newTimeout(timeout -> {
        ExpirationEntry ent = EXPIRATION_RENEWAL_MAP.get(getEntryName());
        if (ent == null) {
            return;
        }
        Long threadId = ent.getFirstThreadId();
        if (threadId == null) {
            return;
        }

        RFuture<Boolean> future = renewExpirationAsync(threadId);
        future.onComplete((res, e) -> {
            if (e != null) {
                log.error("Can't update lock " + getRawName() + " expiration", e);
                EXPIRATION_RENEWAL_MAP.remove(getEntryName());
                return;
            }

            if (res) {
                // reschedule itself
                renewExpiration();
            } else {
                cancelExpirationRenewal(null);
            }
        });
    }, internalLockLeaseTime / 3, TimeUnit.MILLISECONDS);

    ee.setTimeout(task);
}
```

具体续租(lua脚本):

```java
protected RFuture<Boolean> renewExpirationAsync(long threadId) {
    return evalWriteAsync(getRawName(), LongCodec.INSTANCE, RedisCommands.EVAL_BOOLEAN,
                          "if (redis.call('hexists', KEYS[1], ARGV[2]) == 1) then " +
                          "redis.call('pexpire', KEYS[1], ARGV[1]); " +
                          "return 1; " +
                          "end; " +
                          "return 0;",
                          Collections.singletonList(getRawName()),
                          internalLockLeaseTime, getLockName(threadId));
}
```

>  

### zookeeper

zk是一种提供配置管理、分布式协同以及命名的中心化服务.

zk本身并没有提供分布式锁的概念, 而是基于其节点特性提供的.

Zookeeper的数据存储结构就像一棵树，这棵树由节点组成，这种节点叫做Znode。至于这些节点具体干什么用, 是由开发人员自己定义的.

**Znode**分为四种类型 

持久节点 （PERSISTENT） 默认的节点类型。创建节点的客户端与zookeeper断开连接后，该节点依旧存在 ;

持久节点顺序节点（PERSISTENT_SEQUENTIAL） 在持久节点特性上, 按照访问时间的先后顺序进行顺序存储;

临时节点（EPHEMERAL） 和持久节点相反，当创建节点的客户端与zookeeper断开连接后，临时节点会被删除

**临时顺序节点**（EPHEMERAL_SEQUENTIAL） 结合和临时节点和顺序节点的特点：在创建节点时，Zookeeper 根据创建的时间顺序给该节点名称进行编号；**当创建节点的客户端与zookeeper断开连接(session)后，临时节点会被删除**。



Q:为什么是使用临时顺序节点？

A:利用临时顺序节点可以避免“惊群效应”，当某一个客户端释放锁以后，其他客户端不会一 窝蜂的涌入争抢锁资源，而是按时间顺序一个一个来获取锁进行处理。

![image-20211209225641605](picture/image-20211209225641605.png)

- 每一个客户端在尝试获取锁时都自动由ZK创建该顺序节点的“子节点”，按0001、0002这样的编号标识访问顺序
- 从第二个客户端开始，ZK不但创建0002子节点，还会监听前一个0001节点。当客户端1处理完毕或者其他原因释放0001节点后，ZK会通过Watch监听机制通知客户端2进行后续处理，以此保证处理的有序性，避免“惊群效应”产生。



**通过`curator‐recipes`(Apache Curator:zk客户端框架)代码实现**

```
<dependency>
	<groupId>org.apache.curator</groupId>
	<artifactId>curator‐recipes</artifactId>
	<version>5.2.0</version>
</dependency>
```

模拟简单防止商品超卖, 强制将并行变为串行...(

```java
import org.apache.curator.RetryPolicy;
import org.apache.curator.framework.CuratorFramework;
import org.apache.curator.framework.CuratorFrameworkFactory;
import org.apache.curator.framework.recipes.locks.InterProcessMutex;
import org.apache.curator.retry.ExponentialBackoffRetry;
import org.springframework.stereotype.Service;

@Service
public class WarehouseService {
    public static int shoe = 10;

    public int outOfWarehouseWithLock() throws Exception {
        //设置ZK客户端重试策略, 如果无法获取锁资源, 则每隔5秒重试一次，最多重试10次
        RetryPolicy policy = new ExponentialBackoffRetry(5000, 10);

        //创建ZK客户端，连接到Zookeeper服务器
        CuratorFramework client = CuratorFrameworkFactory.builder().connectString("192.168.31.103:2181").retryPolicy(policy).build();
        //创建与Zookeeper服务器的连接
        client.start();

        //声明锁对象，本质就是ZK顺序临时节点
        final InterProcessMutex mutex = new InterProcessMutex(client, "/locks/wh-shoe");
        try {
            //请求锁，创建锁
            mutex.acquire();
            //处理业务逻辑
            if (WarehouseService.shoe > 0) {
                Thread.sleep(1000);
                //扣减库存
                return --WarehouseService.shoe;
            } else {
                //库存不足，
                throw new RuntimeException("运动鞋库存不足");
            }
        } finally {
            //释放锁
            mutex.release();
        }
    }

    public int outOfWarehouse() throws Exception {
        //处理业务逻辑
        if (WarehouseService.shoe > 0) {
            Thread.sleep(1000);
            //扣减库存
            return --WarehouseService.shoe;
        } else {
            throw new RuntimeException("运动鞋库存不足");
        }
    }
}
```

> 可通过jmeter+zoolytic加入断点调试查看zk变化, 验证流程

## **Paxos**选举算法

Paxos算法是Lamport宗师提出的一种基于消息传递的分布式一致性算法，使其获得2013年图灵奖。

在Zookeeper中，通过Paxos算法选举出主节点，同时保证集群数据的强一致性（CP)。



# RPC

**RPC（Remote Procedure Call）** 即远程过程调用. 通过 RPC 可以帮助我们调用远程计算机上某个服务的方法，这个过程就像调用本地方法一样简单。并且！我们不需要了解底层网络编程的具体细节。

步骤:

- 服务消费端（client）以本地调用的方式调用远程服务；
- 客户端 Stub（client stub） 接收到调用后负责将方法、参数等组装成能够进行网络传输的消息体（序列化）：`RpcRequest`；
- 客户端 Stub（client stub） 找到远程服务的地址，并将消息发送到服务提供端；
- 服务端 Stub（桩）收到消息将消息反序列化为Java对象: `RpcRequest`；
- 服务端 Stub（桩）根据`RpcRequest`中的类、方法、方法参数等信息调用本地的方法；
- 服务端 Stub（桩）得到方法执行结果并将组装成能够进行网络传输的消息体：`RpcResponse`（序列化）发送至消费方；
- 客户端 Stub（client stub）接收到消息并将消息反序列化为Java对象:`RpcResponse`



## dubbo

https://dubbo.apache.org/zh/docs/v2.7/dev/design/

是一款高性能、轻量级的开源 Java RPC 框架。

Dubbo 提供了六大核心能力

1. 面向接口代理的高性能RPC调用。
2. 智能容错和负载均衡。
3. 服务自动注册和发现。
4. 高度可扩展能力。
5. 运行期流量调度。
6. 可视化的服务治理与运维。

另外，Dubbo 除了能够应用在分布式系统中，也可以应用在现在比较火的微服务系统中。不过，由于 Spring Cloud 在微服务中应用更加广泛，所以，我觉得一般我们提 Dubbo 的话，大部分是分布式系统的情况。

![dubbo-relation](picture/dubbo-relation.jpg)

**Container：** 服务运行容器，负责加载、运行服务提供者。必须。

**Provider：** 暴露服务的服务提供方，会向注册中心注册自己提供的服务。必须。

**Consumer：** 调用远程服务的服务消费方，会向注册中心订阅自己所需的服务。必须。

**Registry：** 服务注册与发现的注册中心。注册中心会返回服务提供者地址列表给消费者。非必须。

**Monitor：** 统计服务的调用次数和调用时间的监控中心。服务消费者和提供者会定时发送统计数据到监控中心。



# 缓存

> 此类场景不能使用缓存: 如果用户通过身份验证查看敏感信息，然后注销，我们不希望恶意用户能够单击后退按钮查看敏感信息。

![image-20211204201728899](picture/image-20211204201728899.png)

## 客户端缓存

html中图片,css,js,字体等静态资源进行缓存.

以百度logo图为例,通过请求头Expires属性指定过期实现

![image-20211204194947172](picture/image-20211204194947172.png)

status code后面可能为  

from memory cache  (不关闭浏览器多次访问)

或者 

from disk cache  (在访问过的前提下重启浏览器访问)

> 前端页面可以手动缓存到LocalStorage(h5) , 不过期, 可通过js封装有效期

## 应用层缓存

客户端缓存的Expires响应头的属性, 是在CDN内容分发网络中设置.

CDN是互联网静态资源分发的主要技术手段.

比如将北京的静态资源缓存到上海的服务器,上海的用户就近访问静态资源.

CDN的核心技术为**智能DNS**, 根据用户id自动访问就近CDN节点,如果就近节点不存在资源,则由就近CDN节点访问源节点抽取数据再返回(**回源**)

>  比如租用阿里云CDN服务,除了 缓存文件外,还可以在后台为其设置缓存的属性,如之前的Expires,Cache-Control.

>  Expires是指定具体某个时间点缓存到期, 而Cache-Control则代表缓存的有效期是多长时间.



另外Nginx是跨平台高性能web服务器, 比如后端的tomcat服务器可通过nginx做前置的软负载均衡,为应用提供高可用性. 

但通常来自全国各地的访问用户对资源响应速度和带宽要求较高,所以通常选择CDN内容分发.

而在一些企业级应用中,用户分布在固定场所且并发度不高, 不需要部署CDN这样重量级解决方案,通常只需要单独部署一台nginx服务器,利用其自带的资源缓存能力以及压缩功能即可.

nginx设置静态资源缓存配置如下:

![image-20211204201752658](picture/image-20211204201752658.png)

设置缓存之后,nginx在本地用一个个目录进行缓存.



## 服务器缓存

客户端缓存和应用层缓存都是对静态资源进行缓存

服务器缓存可分为

进程内(本地)缓存:本地开辟一块内容空间作为缓存, 如 hibernate,mybatis一二级缓存,springmvc页面缓存,ehcache;

进程外缓存:独立部署的分布式集中缓存,比如redis.

缓存对读写性能的提升:

- 读优化：当请求命中缓存后，可直接返回，从而略过IO读取，减小读的成本。
- 写优化：将写操作在缓冲中合并，让IO设备可以批量处理，减小写的成本。



加缓存并不是简单的加一个分布式缓存即可 ,这是不严谨的. 分布式缓存也成为了瓶颈，高频的QPS是一笔负担;另外缓存驱逐以及网络抖动会影响系统的稳定性，此时引入本地缓存, 可以减轻分布式缓存的压力，并减少网络以及序列化开销。

缓存策略需要按照 **先近到远，先快后慢逐级访问**

如: 商品秒杀，若无本地缓存，都保存在redis 每完成一笔交易，局域网会进行若干网络通信，可能存在网络异常不稳定因素且redis会承担所有节点的压力，当突发流量若超过容载上限redis会崩溃 .所以java的应用端也需要设计多级缓存.



通过进程内缓存和进程外缓存组合分担压力:

![image-20211204203639318](picture/image-20211204203639318.png)

ehcache（进程内缓存）可以在缓存不存在时去redis进程外缓存进行读取，redis没有读取数据库 数据库再对ehcache，redis进行更新. 

但该设计如何保证一致性问题? 如果修改商品的价格,如何保证缓存同步更新?

可引入消息队列（MQ）的主动推送功能，对服务实例推送变更实例,

即在修改商品价格后，向MQ发送变更消息，MQ将消息推送到服务实例服务实例将原缓存数据先删除，再创建缓存. 

> 因为需要MQ, 成本较高, 维护困难, 目前可使用redis6监听实现缓存更新

什么情况适用多级缓存架构
1、缓存数据稳定
2、可能产生高并发场景（12306）应用启动时进行预热处理，访问前将热点数据先缓存，减少后端压力
3、一定程度上允许数据不一致不重要的信息更新处理方式：T+1,ETL日终处理把数据进行补全.

> 服务端可通过设置Cache-control请求头(还有条件请求头如Last-Modified and Etag) 来配置http缓存.
>
> 通常还可以对静态资源进行设置:
>
> ```
> @Configuration
> @EnableWebMvc
> public class WebConfig implements WebMvcConfigurer {
> 
>     @Override
>     public void addResourceHandlers(ResourceHandlerRegistry registry) {
>         registry.addResourceHandler("/resources/**")
>                 .addResourceLocations("/public", "classpath:/static/")
>                 .setCacheControl(CacheControl.maxAge(Duration.ofDays(365)));
>     }
> }
> ```

## 缓存读写策略

1.**Cache Aside Pattern** 旁观缓存模式  适合读多写少场景

写: 

- 先更新 DB
- 然后直接删除 cache 。

读:

- 从 cache 中读取数据，读取到就直接返回
- cache中读取不到的话，就从 DB 中读取数据返回
- 再把数据放到 cache 中。

缺陷:

1.首次请求数据一定不在cache;

解决办法:将热点数据可以提前放入cache .

2.写操作频繁会导致cache中数据频繁删除, 存在不一致问题:

解决办法:

数据库和缓存数据强一致场景 ：更新DB的时候同样更新cache，不过我们需要加一个锁/分布式锁来保证更新cache的时候不存在线程安全问题。

可以短暂地允许数据库和缓存数据不一致的场景 ：更新DB的时候同样更新cache，但是给缓存加一个比较短的过期时间，这样的话就可以保证即使数据不一致的话影响也比较小。

**写数据的过程为什么不能先删除cache后更新db?**

会导致db与cache数据不一致.

比如线程1写A请求先删除cache, 线程2读A请求发现cache中不存在,就读db,并将读到的A缓存到cache, 然后线程2在db中写A. 之后所有访问A的请求都落到cache中,并且值与db不一致.

**先更新db,后删除cache也会出现不一致问题**,几率较小

比如在cache没有缓存A情况下,线程1写A请求走db,线程2读A请求走db(快照读),线程1先删除cache,线程2再写入cache.之后所有访问A的请求走的都是cache,缓存的是db中不存在的数据.  (缺陷2)





2.**Read/Write Through Pattern**(读写穿透)

Read/Write Through Pattern 中服务端把 cache 视为主要数据存储，从中读取数据并将数据写入其中。cache 服务负责将此数据读取和写入 DB，从而减轻了应用程序的职责。

这种缓存读写策略小伙伴们应该也发现了在平时在开发过程中非常少见。抛去性能方面的影响，大概率是因为我们经常使用的分布式缓存 Redis 并没有提供 cache 将数据写入DB的功能。

**写（Write Through）：**

- 先查 cache，cache 中不存在，直接更新 DB。
- cache 中存在，则先更新 cache，然后 cache 服务自己更新 DB（**同步更新 cache 和 DB**）。

**读(Read Through)：**

- 从 cache 中读取数据，读取到就直接返回 。
- 读取不到的话，先从 DB 加载，写入到 cache 后返回响应。

Read-Through Pattern 实际只是在 Cache-Aside Pattern 之上进行了封装。在 Cache-Aside Pattern 下，发生读请求的时候，如果 cache 中不存在对应的数据，是由客户端自己负责把数据写入 cache，而 Read Through Pattern 则是 cache 服务自己来写入缓存的，这对客户端是透明的。

和 Cache Aside Pattern 一样， Read-Through Pattern 也有首次请求数据一定不再 cache 的问题，对于热点数据可以提前放入缓存中。



3.**Write Behind Pattern**（异步缓存写入）

Write Behind Pattern 和 Read/Write Through Pattern 很相似，两者都是由 cache 服务来负责 cache 和 DB 的读写。

但是，两个又有很大的不同：**Read/Write Through 是同步更新 cache 和 DB，而 Write Behind Caching 则是只更新缓存，不直接更新 DB，而是改为异步批量的方式来更新 DB。**

很明显，这种方式对数据一致性带来了更大的挑战，比如cache数据可能还没异步更新DB的话，cache服务可能就就挂掉了。

这种策略在我们平时开发过程中也非常非常少见，但是不代表它的应用场景少，比如消息队列中消息的异步写入磁盘、MySQL 的 InnoDB Buffer Pool 机制都用到了这种策略。

Write Behind Pattern 下 DB 的写性能非常高，非常适合一些数据经常变化又对数据一致性要求没那么高的场景，比如浏览量、点赞量。

## 缓存一致性

![](picture/cache-consistent.jpg)

### Cache Aside Pattern

> **不要考虑去update更新缓存** :无论是先更新缓存再写数据库还是先写数据库再更新缓存, 只要是更新, 都会引发一致性问题.

`Cache Aside Pattern`是经典的缓存一致性处理模式  ,本质是“先写库，再删缓存”

如果采用"先删缓存, 再写库" , 在更新线程1删除id=1的cache后, 如果线程2来读取id=1,因为cache不存在, 线程2直接请求db然后写cache, 然后线程1最后写db , 这样就导致了cache与db数据不一致问题. 这种方式下的**不一致时间窗口为缓存过期或下一次删缓存**.

如果采用“先写库，再删缓存”, 也存在不一致的情况, 在读取线程1查询id=1的数据, cache不存在直接查询db, 此时更新线程2先更新id=1的db数据,  这时线程1将cache设置为查询后的值, 然后线程2删除cache.  **整个过程存在极短的不一致时间窗口: 查询线程写cache后到更新线程删cache这段时间存在不一致**.

"先写库，再删缓存" , 也存在一种极端情况为 : 在上面的最后两个操作顺序调换, 即线程2先删除cache, 线程1再设置cache为查询后的值 这种情况的不一致时间窗口与"先删缓存, 再写库"一样.

**解决方案**: **延迟双删**

在更新db操作完成后,对cache删除后通过定时任务或MQ延迟消息, 在一定时间(1s-2s)之后再删除一次cache.  这样**不一致时间窗口限定在两次删除之间的时间间隔**

> `Cache Aside Pattern + 延迟双删` 无锁方案只能在保证并发的前提下尽可能减少不一致的可能, 这也是AP模式下**BASE**的一种技术实现.

> 如果是分布式缓存, 会由于中间件或网络的问题导致缓存删除失败

# 幂等性

**指的是发一次接口调用与发多次相同的接口消息都能得到与预期相符的结果**

在实际应用中 , 为了提高通信的可靠性,通信框架/MQ可能会向数据服务推送多条相同的信息 , 如

```
PUT https://xxxxx.com/employee/salary
{"id" : "1:,"incr_salary":500}
```

后台逻辑为

```
//典型的ORM做法
Employee employee = employeeService.selectById(1);
employee.setSalary(employee.getSalary() + incrSalary);
employeeService.update(employee)
```

问题在于每重复发送一次请求工资就会多500

解决办法:

1.代码前置判断

```
if(!员工已调薪){进行调薪}
```

缺点: 项目中需要前置判断的地方太多了，一不留神就漏了 , 这种技术问题不应该成为干扰程序员写业务代码的因素

所以大家需要一种无侵入的幂等解决方案.

2.**幂等表**

应用系统发送请求,

![image-20211206000227897](picture/image-20211206000227897.png)

应用系统再次请求, redis中已经存在requestid

![image-20211206000238849](picture/image-20211206000238849.png)

数据服务处理完更新,修改redis中value

![image-20211206001345466](picture/image-20211206001345466.png)

> 也可以后台接口可通过AOP修改redis.
>
> requestId可通过后端生成, 在每次访问应用中可能应发幂等性问题的页面时返回.
>
> 如果提交表单中存在唯一约束,可以将该唯一约束缓存来替代requestId, 比如用户名唯一. 操作完对缓存进行及时删除.

## 幂等失效

如果请求并发访问，在首次请求完成修改服务端状态前，并发的其他请求和首次请求都通过了幂等判断，对数据库状态进行了多次修改。会导致幂等失效.

如果是新增数据, 可以通过数据库的唯一索引解决问题.

对于修改, 通过只能使用分布式锁来解决了.

# APM链路跟踪

指运行时通过某种方式记录下服务之间的调用过程, 在通过可视化的UI界面快速定位到出错点.

- 基于日志收集调用链路:Sleuth & Zipkin

Sleuth额外生成链路跟踪日志, 如

```
2021-01-01 15:00:31.441 INFO [a-serivce, 4f34v3c5545646,x234v543534,true]
```

该格式为OpenTracing规范, OpenTracing为一致的链路追踪日志规范与API接口, 格式为

```
[微服务ID,Trace ID,Span ID,导出标识]
```

比如下单业务需要经过订单服务->库存服务->账务服务, 这三个步骤的Trace ID相同, 每个步骤的Span ID不同; 

导出标识表示日志是否被发送给了日志收集组件(如Zipkin).

> Sleuth只需要导入springboot依赖即可产生该日志

侵入代码, Zipkin为推特开发的分布式链路追踪系统, Zipkin客户端收集Sleuth产生的日志发送到Zipkin服务端, Zipkin服务端采用可视化方式展示.

- 基于Agent(运行时二进制)收集调用链路:SkyWalking

```
java -javaagent:skywalking-agent.jar
-Dskywalking.agent.service_name=a-service
-Dskywalking.collector.backend_serivce=192.168.31.10:11800
-Dskywalking.logging.file_name=a-serivce-api.log
-jar a-service.jar 
```

不侵入, 但侵入程序运行过程, 监听代码执行过程进行, 对调用关系进行梳理. 其收集的指标比日志方式要多很多.

> 链路跟踪产品还有pinpoint,istio,cat,jeager等等

