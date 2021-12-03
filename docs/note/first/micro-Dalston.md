# 面试题

**springcloud和dubbo的区别？**

答：dubbo:基于RPC远程过程调用，RPC基于Socket，工作在会话层。自定义数据格式。速度快，效率高；

cloud基于http的restful调用。基于TCP，工作在应用层，规定了数据传输的格式。缺点是消息封装臃肿，优势是对服务的 提供和调用方没有任何技术限定，自由灵活，更符合微服务理念。

springboot和<u>springcloud</u>关系？

答：Springboot专注于快速方便的开发单个个体微服务

Springcloud是关注全局的微服务协调整理治理架构，它将springboot开发的一个个单体微服务整合并管理起来。为各个微服务之间提供配置管理、服务发现、断路器、路由、微代理、事件总线、全局锁等继承服务。

<u>什么是服务熔断，什么是服务降级？</u>

<u>微服务的优缺点分别是什么</u>？

it<u>公司用的微服务架构有哪些</u>？

答：阿里dubbo/HSF

​		京东JSF

​		新浪微博Motan

​		当当网DubboX

<u>微服务技术栈又那些</u>？

答：

| 微服务条目       | 落地技术                               |      |
| ---------------- | -------------------------------------- | ---- |
| 服务开发         | springboot spring springmvc            |      |
| 服务配置与管理   | Netflix公司的Archaius、阿里的Diamond等 |      |
| 服务注册与发现   | Eureka、Zookeeper、Consul              |      |
| 服务调用         | Rest、RPC、gRPC                        |      |
| 服务熔断器       | Hystrix、Envoy                         |      |
| 负载均衡         | Ribbon、Nginx                          |      |
| 服务接口调用     | Feign                                  |      |
| 消息队列         | Kafka、RabbitMQ、ActiveMQ              |      |
| 服务配置中心管理 | springcloud config、chef               |      |
| 服务路由         | Zuul                                   |      |
| 服务监控         | Zabbix、Nagios、Metrics、Spectator     |      |
| 全链路追踪       | Zipkin、Brave、Dapper                  |      |
| 服务部署         | Docker、OpenStack、Kubernetes          |      |
| 数据流操作开发包 | SpringCloud Stream                     |      |
| 事件消息总线     | SpringCloud Bus                        |      |

eureka和zookeeper都可以提供服务注册于发现的功能，他们的区别是？

# springcloud

父工程

```
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-dependencies</artifactId>
    <version>Dalston.SR1</version>
    <type>pom</type>
    <scope>import</scope>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-dependencies</artifactId>
    <version>1.5.9.RELEASE</version>
</dependency>
```

## Eureka

是Netflix的一个子模块，类似于dubbo的服务注册Zookeeper，用来实现服务注册和发现。 系统中其他的微服务使用eureka client连接到eureka server并维持心跳连接。

**服务端**：

```
        <dependency>
<!--            eureka服务端-->
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-netflix-eureka-server</artifactId>
            <version>2.1.1.RELEASE</version>
        </dependency>
```

```
在启动类上添加组件注解
@EnableEurekaServer
其他组件类似
如@EnableZuulProxy
```

```
eureka:
  instance:
    hostname: localhost #eureka服务端实例名称
  client:
    register-with-eureka: false #不向注册中心注册自己
    fetch-registry: false  #自己本身为注册中心
    service-url:
      defaultZone: http://${eureka.instance.hostname}:${server.port}/eureka/
      #设置与eureka server交互的地址查询服务和注册服务需要的地址
```

**provider**：

```
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
    <version>2.1.1.RELEASE</version>
</dependency>
```

```
eureka:
  client:
    service-url:
      defaultZone: http://localhost:7001/eureka
```

**改善**

搭建完后 为了简化网页中对客户端的名字，在微服务客户端进行如下添加设置：

```
eureka:
	......
  instance:
    instance-id: dept8001
```

为了将鼠标放在网页客户端名字上提示ip端口，添加：

```
eureka:
	......
  instance:
	prefer-ip-address: true
```

微服务info信息页，在微服务客户端：

```
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```

总的父工程添加构建build信息

```
<build>                  <!--构建-->
    <finalName>microservicecloud</finalName> <!--父工程名-->
    <resources>
        <resource>
            <directory>src/main/resources</directory>  <!--允许访问所有工程的资源目录-->
            <filtering>true</filtering>          <!--过滤开启-->
        </resource>
    </resources>
    <plugins>
        <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-resources-plugin</artifactId>
            <configuration>
                <delimiters>
                    <delimit>@</delimit>       <!--替换@开头并结尾的占位符-->
                </delimiters>
            </configuration>
        </plugin>
    </plugins>
</build>
```

在微服务客户端自己的配置文件中添加

```
info:                 #自定义的键值对
  app.name: atguigu-microservicecloud           #写死
  company.name: www.atguigu.com
  build.artifactId: "@project.artifactId@"       #父目录的构建插件扫描的地方  返回pom文件module名字
  build.version: "@project.version@"
```

访问其info界面为上述内容的json数据

**服务发现**

在provider controller中

```
@Resource
private DiscoveryClient client; 
```

```
@GetMapping("discovery")  //服务发现
public Object discovery(){
   //所有服务
   List<String> list = client.getServices();
   System.out.println("******"+list);

   //某个服务的所有实例
   List<ServiceInstance> srvList=client.getInstances("MICROSERVICECLOUD-DEPT");
   for (ServiceInstance element : srvList) {
      System.out.println(element.getServiceId()+"\t"+element.getHost()+"\t"+element.getPort()+"\t"+element.getUri());
   }
   return client;
}
```

并在启动类添加注解

```
@EnableDiscoveryClient //服务发现
```

最后浏览器访问：

http://localhost:8001/dept/discovery

comsumer也可以访问服务发现

### 服务端集群

创建多个service模块

为了区分每个服务端名 模拟三台机器 在同一台机的情况下设置一下域名映射

127.0.0.1 eureka7001.com
127.0.0.1 eureka7002.com
127.0.0.1 eureka7003.com

交叉配置

在一台上配置另外两台

```
server:
  port: 7001
eureka:
  instance:
    hostname: eureka7001.com #eureka服务端实例名称
  client:
    register-with-eureka: false #不向注册中心注册自己
    fetch-registry: false  #自己本身为注册中心
    service-url:
#      defaultZone: http://${eureka.instance.hostname}:${server.port}/eureka/  单机情况
      #设置与eureka server交互的地址查询服务和注册服务
      defaultZone:  http://eureka7002.com:7002/eureka/,http://eureka7003.com:7003/eureka/
```

服务提供端修改注册地址为三台服务注册端：

```
eureka:
  client:   #客户端注册进eureka
    service-url:
      defaultZone: http://eureka7001.com:7001/eureka/,http://eureka7002.com:7002/eureka/,http://eureka7003.com:7003/eureka/
```

开启三个服务注册和服务提供端

访问一个http://eureka7001.com:7001

集群与服务分开显示

集群：DS Replicas  一个服务提供端连接另外两个

### eureka自我保护

当过一段时间 eureka网址显示红色内容：

**EMERGENCY! EUREKA MAY BE INCORRECTLY CLAIMING INSTANCES ARE UP WHEN THEY'RE NOT. RENEWALS ARE LESSER THAN THRESHOLD AND HENCE THE INSTANCES ARE NOT BEING EXPIRED JUST TO BE SAFE.**

Netflix在设计Eureka时遵循AP原则

某时刻一个微服务不可用了，eureka不会立即清理，依旧会对该微服务的信息进行保存。

在自我保护模式中，server会保护服务注册表中的信息，不再注销任何服务实例，当它收到的心跳数重新恢复到阈值以上时，该server节点就会自动退出保护模式。它的设计哲学就是宁可保留错误的服务注册信息，也不盲目注销任何可能健康的服务实例。是一种应对网路异常保护的安全措施。

enable-self-preservation: false # 关闭自我保护模式（缺省为打开）

### eureka和zookeeper

RDBMS关系型数据库---》 ACID

A:原子性

C:一致性

I:独立性

D:持久性

NOSQL 非关系型数据库  ---》CAP

C consisitency:一致性

A availability:可用性

P partition tolerance:分区容错性

一个分布式系统 不能全满足CAP，都要CAP三进二，即CP、AP、CA组合

CP：MongDB,redis,HBase

AP：CouchDB...

CA：RDBMS 

由于当前的网络硬件肯定会出现延迟丢包的问题，所以分布式结构 分区容错性P必须满足

淘宝、京东：双11当天，只能选AP。保证网站不崩溃

在向注册中心查询服务列表时候，我们可以容忍注册中心返回的是几分钟以前的注册信息，但不能接受服务直接不可用。也就是说，服务注册功能对可用性要求高于一致性。

Zookeeper保证CP：

但是zk会出现这样一种情况，当master节点因为网络故障与其他节点失去联系时，剩余节点会重新进行leader选举。问题在于选举leader的时间太长，为30~120s，且选举期间整个zk集群都是不可用的，这就导致在选取期间注册服务瘫痪。在云部署的环境下，因网络问题使得zk集群失去master节点是较大概率会发生的事情，虽然服务能够最终恢复，但是漫长的选举时间导致的注册长期不可用时不能容忍的。

Eureka保证AP：
eureka各个节点都是平等的，几个节点挂掉不会影响正常节点的工作，剩余的节点依然可以提供注册和查询服务。而eureka的客户端在向某个eureka注册或时如果发现连接失败，则会自动切换至其他节点，只要有一台eureka在，就能保证注册服务可用，只不过查到的信息可能不是最新的。除此之外，eureka还有一种自我保护功能，如果在15分组内超过85%的节点都没有正常的心跳，那么eureka就认为客户端与注册中心出现了网络故障，此时会出现以下几种情况：

1.eureka不在从注册表中移除因为长时间没有收到心跳而应该过期的服务；

2.eureka仍然能接受新服务的注册和查询请求，但是不会同步到其他节点上，即保证当前节点依然可用；

3.当网络稳定时，当前实例新的注册信息会被同步到其他节点中。

## ribbon负载均衡

Load Balance ，如nginx、LVS、硬件f5等。

springcloud负载均衡算法可以自定义。

集中式LB：在服务的消费方和提供方之间使用独立的LB设施（可以是硬件，如f5，也可以是软件，如nginx），由该设施负载把访问请求通过某种策略转发至服务的提供方。

进程内LB：将LB逻辑继承到消费方，消费方从服务注册中心获知由哪些地址可用，然后再从这些地址中选择出一个合适的服务器。Ribbon就属于进程内LB，它只是一个类库，集成于消费方进程。

provider修改：

```
<!--        ribbon相关-->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
    <version>2.1.1.RELEASE</version>
</dependency>
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-ribbon</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-config</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>
```

application.yml追加eureka服务注册地址

```
eureka:
  client:
    register-with-eureka: false
    service-url:
      defaultZone: http://eureka7001.com:7001/eureka/,http://eureka7002.com:7002/eureka/,http://eureka7003.com:7003/eureka/
```

给RestTemplate添加注解@LoadBalanced

```
@Bean
@LoadBalanced //负载均衡
public RestTemplate getRestTemplate(){
   return new RestTemplate();
```

启动类添加@EnableEurekaClient，真正使用微服务

在将访问类的url修改为

```
private static final String REST_URL_PREFIX="http://MICROSERVICECLOUD-DEPT";//服务提供者的应用名
```

ribbon工作步骤：

1.先选择eureka serverr 它优先选择在同一区域内负载较少的server;

2.再根据用户指定的策略（默认轮询），从server取到的服务注册列表中选择一个地址

### 核心组件IRule

irule 接口

根据特定算法从服务列表中选取一个要访问的服务。 

自带7种 包括轮询RoundRobinRule.

智能轮询RetryRule：当有一个服务的实例挂了以后，在一定时间内还是采用轮询访问挂了的实例，在时间过后如果挂了的实例还没有恢复则在轮询中去除该实例

更换其他算法，在消费者配置类中添加方法：

```
@Bean
   public IRule myRule(){
//    return  new RoundRobinRule();
//    return new RandomRule(); //指定负载均衡算法，不使用默认的轮询,使用随机
      return new RetryRule();
```

**更换轮询规则**

不再使用7种自带的策略，在<u>启动该微服务</u>的时候能去加载自定义的ribbon配置类，从而使配置生效，@RibbonClient，形如：

name :使用特定策略的微服务

configuration:策略类；IRule接口实现类

```
@RibbonClient(name = "MICROSERVICECLOUD-DEPT",configuration = MySelfRule.class)
```

注意：

这个自定义的配置类不能放在@ComponentScan所扫描的当前包以及子包下，否则我们自定义的这个配置类就会被所有的Ribbon客户端所共享，也就是说我们达不到<u>特殊化定制</u>的目的了。

```
@Configuration
public class MySelfRule {
	@Bean
   public IRule myRule(){
      return new RandomRule();
   }
}
```

**自定义规则**

新要求：同样是轮询 ，但每个服务器调用5次

获取ribbon的RandonRule源码给自己的类，修改其内部：

```
public class MyRandomRule extends AbstractLoadBalancerRule {
   //当total==5以后，指针才能往下走
   private int total=0; //总共被调用的次数，目前要求每台服务器被调用5次
   //当前对外提供服务的服务器地址
   private int index=0;//当前服务的机器号

   /**
    * Randomly choose from all living servers
    */
// @edu.umd.cs.findbugs.annotations.SuppressWarnings(value = "RCN_REDUNDANT_NULLCHECK_OF_NULL_VALUE")
   public Server choose(ILoadBalancer lb, Object key) {
      if (lb == null) {
         return null;
      }
      Server server = null;

      while (server == null) {
         if (Thread.interrupted()) {
            return null;
         }
         List<Server> upList = lb.getReachableServers();
         List<Server> allList = lb.getAllServers();

         int serverCount = allList.size();
         if (serverCount == 0) {
            /*
             * No servers. End regardless of pass, because subsequent passes
             * only get more restrictive.
             */
            return null;
         }

//       int index = chooseRandomInt(serverCount);
//       server = upList.get(index);
         if (total<5){
            server=upList.get(index);
            total++;
         }else {
            total=0;
            index++;
            if (index>=upList.size()){
               index=0;
            }
         }

         if (server == null) {
            /*
             * The only time this should happen is if the server list were
             * somehow trimmed. This is a transient condition. Retry after
             * yielding.
             */
            Thread.yield();
            continue;
         }

         if (server.isAlive()) {
            return (server);
         }

         // Shouldn't actually happen.. but must be transient or a bug.
         server = null;
         Thread.yield();
      }

      return server;

   }

   protected int chooseRandomInt(int serverCount) {
      return ThreadLocalRandom.current().nextInt(serverCount);
   }

   @Override
   public Server choose(Object key) {
      return choose(getLoadBalancer(), key);
   }

   @Override
   public void initWithNiwsConfig(IClientConfig iClientConfig) {

   }
}
```

修改启动包外的配置类：

```
@Configuration
public class MySelfRule {
	@Bean
   public IRule myRule(){
      return new MyRandomRule();
   }
}
```

## Feign负载均衡

声明式web service客户端，能让编写客户端更加简单，它的使用方法是<u>定义一个接口，然后在上面添加注解</u>，同时页支持JAX-RS标准的注解。也支持可拔插式的编码器和解码器。Spring cloud 对feign进行了封装，使其支持了Spring mvc标准注解和HttpMessageConverters。feign可以与eureka和ribbon组合使用以支持负载均衡。

保留面向接口编程，类似于dao层接口加Mapper注解

Feign集成了Ribbon利用Ribbon维护了MICROSERVICECLOUD-DEPT的服务列表信息，并通过轮询实现了客户端的负载均衡，与ribbon不同的是，通过feign只需要定义服务绑定接口且以声明式的方法，优雅而简单的实现了服务调用。

```
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-feign</artifactId>
</dependency>
```

```
@FeignClient(value = "MICROSERVICECLOUD-DEPT")
public interface DeptClientService {
   @RequestMapping(value = "/dept/get/{id}",method = RequestMethod.GET)
   public Dept get(@PathVariable("id") long id);
   @RequestMapping(value = "/dept/list",method = RequestMethod.GET)
   public List<Dept> list();
   @RequestMapping(value = "/dept/add",method = RequestMethod.POST)
   public Dept insert(Dept dept);
}
```

```
@RestController
public class DeptController_Consumer {
   @Autowired
   private DeptClientService service;

   @RequestMapping(value = "consumer/dept/add")
   public Dept add(Dept dept){
      Dept dept1 = service.insert(dept);
      return dept1;
   }
   @RequestMapping("consumer/dept/selectOne/{id}")
   public Dept selectOne(@PathVariable("id") Long id) {
      Dept dept = service.get(id);
      return dept;
   }

   @RequestMapping("consumer/dept/list")
   public List<Dept> list() {
      return service.list();
   }

   /**
    * 消费端访问服务发现
    * @return
    */
   @RequestMapping("consumer/dept/discovery")
   public Object discovery(){
      return restTemplate.getForObject(REST_URL_PREFIX+"/dept/discovery",Object.class);
   }
}
```

修改启动类，添加注解EnableFeignClients

## Hystrix断路器

复杂的分布式体系结构中的应用程序有数十个依赖关系，每个依赖关系在某些时候将不可避免地失败、异常，Hystrix能保证在一个依赖出问题的情况下，不会导致整体服务失败，避免级联故障，提供分布式系统的弹性。

断路器本身是一种开关装置，当某个服务单元发生故障之后，通过断路器的故障监控（类似熔断保险丝），向调用方返回一个符合预期的、可处理的备选响应FallBack，而不是长时间的等待或者抛出调用方无法处理的异常，这样就保证了服务调用方的线程不会被长时间不必要地占用，从而避免了故障在分布式系统中地蔓延，乃至雪崩。

服务熔断：是应对雪崩效应地一种微服务链路保护机制。

springcloud里地熔断机制通过hystrix实现，会监控微服务间地调用情况，当失败的调用达到一定阈值时，缺省是5s内20次调用失败就会启动熔断机制，熔断机制的注解是@HystrixCommand

```
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-hystrix</artifactId>
</dependency>
```

Controller

```
@HystrixCommand(fallbackMethod = "processHystrix_Get")
public Dept selectOne(@PathVariable("id") Long id) {
   Dept dept = this.deptService.queryById(id);
   if (null == dept){
      throw new RuntimeException("该ID:"+id+"没有相应信息");
   }
   return dept;
}
public Dept processHystrix_Get(@PathVariable("id") Long id){
   return new Dept().setDeptno(id).setDName("该ID:"+id+"没有相应信息,null--@HystrixCommand").setDb_source("no this database in MYSQL")
}
```

一旦调用服务方法失败并抛出了错误信息后，会自动调用@HystrixCommand标注好的fallbackMethod方法。

修改启动类

```
@EnableCircuitBreaker //开启对hystrix熔断机制的支持
```

### 服务降级

整体资源快不够了，忍痛将某些服务先关掉，待渡过难关再开启回来。

<u>服务降级处理是在客户端实现完成的，与服务端没有关系。</u>

前面的断路器实例一个controller方法需要一个fallback方法，方法膨胀；并且fallback方法与业务方法在一起，耦合度高。

```
@Component 
public class DeptClientServiceFallbackFactory implements FallbackFactory<DeptClientService>{

   @Override
   public DeptClientService create(Throwable throwable) {
      return new DeptClientService() {
         @Override
         public Dept get(long id) {
            return new Dept().setDeptno(id).setDName("该ID:"+id+"没有相应信息,Consumer客户端提供的降级信息,此刻服务Provider已经关闭").setDb_source("no this database in MYSQL");
         }

         @Override
         public List<Dept> list() {
            return null;
         }
         @Override
         public Dept insert(Dept dept) {
            return null;
         }
      };
   }
}
```

在DeptClientService接口的注解FeignClient中添加属性fallbackFactory。

```
@FeignClient(value = "MICROSERVICECLOUD-DEPT",fallbackFactory = DeptClientServiceFallbackFactory.class)
```

修改feign消费者工程的yml,添加

```
feign:
  hystrix:
    enabled: true
```

启动3个服务端，启动服务提供端，启动feign消费者，访问：http://localhost/consumer/dept/selectOne/3 （正常访问）

关闭8001后再访问.

此时服务端provider已经挂了，但是我们做了服务降级处理，让客户端在服务端不可用时也会获得提示信息而不会挂起耗死服务器（一直等到服务超时）。

开启8001还可以继续使用

### 服务熔断和服务降级

服务熔断：一般是某个服务故障或者异常引起，类似现实中的保险丝，当某个异常条件被触发，直接熔断整个服务，而不是一直等到此服务超时。

服务降级：一般是从整体符合考虑。就是当某个服务熔断之后，服务器将不再被调用。此时客户端自己准备一个本地的fallback回调，返回一个缺省值。这样做虽然服务水平下降，但好歹可用，比直接挂掉要强。

在访问量高的时候体现强。

## 服务监控hystrixDashboard

```
    <dependencies>
<!--        服务监控hystrixDashboard-->
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-hystrix</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-hystrix-dashboard</artifactId>
        </dependency>
<!--        自己的api-->
        <dependency>
            <groupId>com.atguigu</groupId>
            <artifactId>microservicecloud-api</artifactId>
            <version>1.0-SNAPSHOT</version>
        </dependency>
<!--        feign-->
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-feign</artifactId>
        </dependency>
<!--        ribbon-->
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-config</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-ribbon</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-eureka</artifactId>
        </dependency>
    </dependencies>
```

yml

```
server:
  port: 9001
```

主启动类，注解@EnableHystrixDashboard

给所有provider 8001/8002/8003监控依赖配置

```
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```

启动dashboard

访问http://localhost:9001/hystrix

豪猪提示：单个监控的访问url

*Single Hystrix App:* http://hystrix-app:port/hystrix.stream

将http://localhost:8001/hystrix.stream填到文本框中，点击monitor stream

delay为服务器上轮询监控信息的延迟时间，默认为2000ms，可以通过配置该属性来降低客户端的网络和cpu消耗。

title对应头部变体hystrix stream之后的内容，默认会使用具体监控实例的url。

查看

7色 1圆 1线

多次刷新访问服务看效果

圆：通过颜色的变化代表实例的健康程度；通过大小的变化达标实例的请求流量。

## zuul路由网关

路由：负责将外部请求转发到具体的微服务实例上，是实现外部访问同一入口的基础。

过滤：负责对请求的处理过程进行干预，是实现请求效验、服务聚合等功能的基础。

Zuul和Eureka进行整合，将Zuul自身注册为Eureka服务治理下的应用，同时从Eureka中获得其他微服务的消息，即以后的访问微服务都是通过zuul跳转后获得。

注意：zuul服务最终还是会注册进eureka。

```
<dependencies>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-eureka</artifactId>
        </dependency>
<!--        zuul-->
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-zuul</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-actuator</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-hystrix</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-config</artifactId>
        </dependency>
    </dependencies>
```

yml

```
server:
  port: 9527

spring:
  application:
    name: microservicecloud-zuul-gateway

eureka:
  client:
    service-url:
      defaultZone: http://eureka7001.com:7001/eureka/,http://eureka7002.com:7002/eureka/,http://eureka7003.com:7003/eureka/
  instance:
    instance-id: gateway-9527.com
    prefer-ip-address: true

info:                 #自定义的键值对
  app.name: atguigu-microservicecloud           #写死
  company.name: www.atguigu.com
  build.artifactId: "@project.artifactId@"       #父目录的构建插件扫描的地方  返回pom文件module名字
  build.version: "@project.version@"
```

hosts修改

127.0.0.1 myzuul.com

启动类

```
@SpringBootApplication
@EnableZuulProxy
public class Zuul_9527_StartSpringCloudApp {
   public static void main(String[] args) {
      SpringApplication.run(Zuul_9527_StartSpringCloudApp.class,args);
   }
}
```

启动server，rovider, zuul

在eureka系统可看dept和zuul应用实例

不用路由：http://localhost:8001/dept/selectOne/3

启用路由：http://myzuul.com:9527/microservicecloud-dept/dept/selectOne/3

### zuul路由访问映射规则

前面路由访问暴露了微服务名，这里使用代理

在9527工程的yml添加：

```
zuul:
  routes:
    mydept.serviceId: microservicecloud-dept
    mydept.path: /mydept/**
```

http://myzuul.com:9527/mydept/dept/selectOne/3

问题：

http://myzuul.com:9527/microservicecloud-dept/dept/selectOne/3还能够访问，违背了单入口原则，需要隐藏：

```
zuul:
  routes:
    mydept.serviceId: microservicecloud-dept
    mydept.path: /mydept/**
  ignored-services: microservicecloud-dept
```

如果需要将所有微服务都忽略掉，则

```
zuul:
  ignored-services: "*"
```

设置同一公共前缀

```
zuul:
  prefix: /hunnu
```

http://myzuul.com:9527/hunnu/mydept/dept/selectOne/3

## Config分布式配置中心

微服务意味着要将单体应用中的业务拆分成一个个子服务，每个服务的粒度相对较小，因此系统中会出现大连过的服务。由于每个服务都需要必要的配置信息才能运行，所以一套集中式的、动态的配置管理设施是必不可少的。springcloud提供了ConfigServer来解决这个问题，我们每个微服务自己带着一个application.yml，上百个配置文件的管理.......

Config为微服务架构中的微服务提供集中化的外部配置支持，配置服务器为各个不同微服务应用的所有环境提供了一个中心化的外部配置。

分为服务端和客户端

服务端也成为分布式配置中心，它是一个独立的微服务应用，用来连接配置服务器并为客户端提供获取配置信息，加密/解密信息等访问接口。

客户端是通过指定的配置中心来管理应用资源，以及与业务相关的配置内容，并在启动的时候从配置中心获取和加载配置信息配置服务器采用git来存储配置信息，这样就有助于对环境配置进行版本管理，并且可以通过git客户端工具来方便的管理和访问配置的内容。

功能：集中管理配置文件；不同环境不同配置，动态化的配置更新，分环境部署比如dev/test/prod/beta/release;运行期间动态调整配置，不再需要在每个服务部署的机器上编写配置文件，服务会向配置中心统一拉取配置自己的信息，当配置发生变动时，服务不需要重启即可感知到配置的变化并应用新的配置；将配置信息以REST接口的形式暴露。

**服务端配置**

在github/gitee上新建config仓库,由上一步获得http/ssh协议的git地址

在clone下来的目录下新建application.yml,多环境配置：

```yml
spring:
 profiles:
  active:
  - dev
---
spring:
 profiles: dev   #开发环境  
 application:
  name: microservicecloud-config-atguigu-dev
---  
spring:
 profiles: test    #测试环境  
 application:
  name: microservicecloud-config-atguigu-test
 #请保存为utf-8格式
```

注意：保存格式必须为utf-8

然后推送给远程仓库

新建模块，作为cloud的配置中心模块

```
<dependencies>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-config-server</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-actuator</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-eureka</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-hystrix</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-config</artifactId>
    </dependency>
</dependencies>
```

yml

```
server:
  port: 3344
spring:
  application:
    name: microservicecloud-config
  cloud:
    config:
      server:
        git:
          uri: https://gitee.com/li-zhuofan-350562/microservicecloud-config.git
```

启动类

```
@SpringBootApplication
@EnableConfigServer
public class Config_3344_StartSpringCloudApp {
   public static void main(String[] args) {
      SpringApplication.run(Config_3344_StartSpringCloudApp.class,args);
   }
}
```

hosts修改

127.0.0.1 config3344.com

测试

启动微服务3344

http://config3344.com:3344/application-dev.yml

http://config3344.com:3344/application-test.yml

http://config3344.com:3344/application-xxx.yml（不存在的配置）

配置读取规则(了解)

/{application}-{profile}.yml       

前面使用的方式

/{application}/{profile}[/{lable}]

如http://config3344.com:3344/application/dev/master

/{lable}/{application}-{profile}.yml

如http://config3344.com:3344/master/application-dev.yml

**客户端配置**

在clone下来的目录下新建microservicecloud-config-client.yml

```
spring:
 profiles:
  active:
  - dev
---
server:
 port: 8201
spring:
 profiles: dev   #开发环境  
 application:
  name: microservicecloud-config-client
eureka:
 client:
  service-url:
   defaultZone: http://eureka-dev.com:7001/eureka/
---  
server:
 port: 8202
spring:
 profiles: test   #测试环境  
 application:
  name: microservicecloud-config-client
eureka:
 client:
  service-url:
   defaultZone: http://eureka-test.com:7001/eureka/
 #请保存为utf-8格式
```

提交到git

新建模块3355

```
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-config</artifactId>
</dependency>
```

新建bootstrap.yml

解释：

application.yml是用户级的资源配置项

bootstrap.yml是系统级的，优先级更高，不会被本地配置覆盖

```yml
spring:
  cloud:
    config:
      name: microservicecloud-config-client #需要从git仓库里读取的资源名称，与前面的文件名对应，但没有yml后缀
      profile: dev #本此访问的配置项，决定从git仓库中读取什么，这里为8201开发环境
      label: master
      uri: http://config3344.com:3344  #本微服务启动后先去找3344号服务，通过springcloudConfig获取git的服务地址
```

新建application.yml  无关紧要

```
spring:
  application:
    name: microservicecloud-config-client
```

验证能否从git仓库中读取配置

```java
@RestController
public class ConfigClientRest {
	//读配置文件
	@Value("${spring.application.name}")
	private String applicationName;
	@Value("${eureka.client.service-url.defaultZone}")
	private String eurekaServers;
	@Value("${server.port}")
	private String port;
	@RequestMapping("config")
	private String getConfig(){
		String str = "applicationName:"+applicationName+"\t eurekaServers+"+eurekaServers+"\t port:"+port;
//		String str = "applicationName:"+applicationName;
		System.out.println("****str:"+str);
		return str;
	}
}
```

启动类

```
@SpringBootApplication
public class ConfigClient_3355_StartSpringCloudApp {
   public static void main(String[] args) {
      SpringApplication.run(ConfigClient_3355_StartSpringCloudApp.class,args);
   }
}
```

测试

启动config配置中心3344 自测:http://config3344.com:3344/application-dev.yml

启动3355作为client准备访问

bootstrap.yml里面的profile值是什么，决定从git仓库中读取什么

假如目前是profile:dev 则访问：http://client-config.com:8201/config

假如目前是profile:test 则访问：http://client-config.com:8202/config



### 应用：

做一个eureka服务+一个dept访问的微服务，将两个微服务的配置统一由git仓库获得，实现统一配置分布式管理，完成多环境的变更。

在clone的目录下新建文件

1.microservicecloud-config-eureka-client.yml

```
spring:
  profiles:
    active:
    - dev

---
server:
  port: 7001
spring:
  profiles: dev
  application:
    name: microservicecloud-config-eureka-client
eureka:
  instance:
    hostname: eureka7001.com
  client:
    register-with-eureka: false     #当前的eurekaserver自己不注册进服务列表
    fetch-registry: false  #不通过eureka获取注册信息
    service-url: 
      defaultZnoe: http://eureka7001.com:7001/eureka/
---
server:
  port: 7001
spring:
  profiles: test
  application:
    name: microservicecloud-config-eureka-client
eureka:
  instance:
    hostname: eureka7001.com
  client:
    register-with-eureka: false
    fetch-registry: false
    service-url: 
      defaultZnoe: http://eureka7001.com:7001/eureka/
```

2.microservicecloud-config-dept-client.yml

```
spring:
  profiles:
    active:
    - dev
---
server:
  port: 8001
spring:
  profiles: dev
  application:
    name: microservicecloud-config-dept-client
  datasource:
    type: com.alibaba.druid.pool.DruidDataSource
    driver-class-name: com.mysql.jdbc.Driver
    url: jdbc:mysql://localhost:3306/clouddb01?useUnicode=true&characterEncoding=utf8&serverTimezone=Asia/Shanghai
    username: root
    password: 350562
    dbcp2:
      min-idle: 5   #数据库连接池的最小持续连接数
      initial-size: 5     #初始化连接数
      max-total: 5      #最大~
      max-wait-millis: 200   #等待连接获取的最大超时时间
mybatis:
  config-location: classpath:mybatis/mybatis.cfg.xml
  type-aliases-package: com.atguigu.entity
  mapper-locations:
    - classpath:mybatis/mapper/**/*.xml
eureka:
  client:  
    service-url:
      defaultZone: http://eureka7001.com:7001/eureka/
  instance:
    instance-id: dept-8001.com
    prefer-ip-address: true
info:             
  app.name: atguigu-microservicecloud          
  company.name: www.atguigu.com
  build.artifactId: "@project.artifactId@"      
  build.version: "@project.version@"
---
server:
  port: 8001
spring:
  profiles: test
  application:
    name: microservicecloud-config-dept-client
  datasource:
    type: com.alibaba.druid.pool.DruidDataSource
    driver-class-name: com.mysql.jdbc.Driver
    url: jdbc:mysql://localhost:3306/clouddb02?useUnicode=true&characterEncoding=utf8&serverTimezone=Asia/Shanghai
    username: root
    password: 350562
    dbcp2:
      min-idle: 5   #数据库连接池的最小持续连接数
      initial-size: 5     #初始化连接数
      max-total: 5      #最大~
      max-wait-millis: 200   #等待连接获取的最大超时时间
mybatis:
  config-location: classpath:mybatis/mybatis.cfg.xml
  type-aliases-package: com.atguigu.entity
  mapper-locations:
    - classpath:mybatis/mapper/**/*.xml
eureka:
  client:  
    service-url:
      defaultZone: http://eureka7001.com:7001/eureka/
  instance:
    instance-id: dept-8001.com
    prefer-ip-address: true
info:             
  app.name: atguigu-microservicecloud          
  company.name: www.atguigu.com
  build.artifactId: "@project.artifactId@"      
  build.version: "@project.version@"
```

开发环境用db01，测试环境用db02

```
    <dependencies>
<!--        spring cloud config-->
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-config</artifactId>
        </dependency>
        <dependency>
            <!--            eureka服务端-->
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-eureka-server</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework</groupId>
            <artifactId>springloaded</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-devtools</artifactId>
        </dependency>
    </dependencies>
```

启动类

```
@SpringBootApplication
@EnableEurekaServer
public class Config_Git_EurekaServerApplication {
   public static void main(String[] args) {
      SpringApplication.run(Config_Git_EurekaServerApplication.class,args);
   }
}
```

application.yml

```
spring:
  application:
    name: microservicecloud-config-eureka-client
```

bootstrap.yml

```
spring:
  cloud:
    config:
      name: microservicecloud-config-eureka-client #需要从git仓库上读取的资源名称，注意没有yml后缀
      profile: dev
      label: master
      uri: http://config3344.com:3344 #本微服务启动后先去找3344号服务，通过springcloudConfig获取git仓库的地址
```

服务端测试：

启动config3344，保证config总配置；启动7001微服务

访问http://eureka7001.com:7001/  查看eureka系统

```
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-config</artifactId>
</dependency>
```

bootstrap.yml

```
spring:
  cloud:
    config:
      name: microservicecloud-config-dept-client
      profile: test  #测试环境 用db02
      label: master
      uri: http://config3344.com:3344
```

application.yml

```
spring:
  application:
    name: microservicecloud-config-dept-client
```

启动类不用变，可修改名字

测试

启动config3344，启动config-eureka-client7001，启动condif-dept-client-8001

如果当前的dept是test环境 则访问http://localhost:8001/dept/list 可以看到数据库为02

如果当前的dept是dev环境 则访问http://localhost:8001/dept/list 可以看到数据库为01

如果修改git仓库中的配置文件，比如修改操作开发环境数据库并push，也可以达到修改效果。

## 总结

1.开发技术栈以spring cloud 为主 ，单个微服务模块以ssm+springboot组合进行开发。

2.前端层，页面h5+thymeleaf/样式css3+bootstrap/前端框架jquery+node|vue等。

3.负载层，前端访问通过http或https协议到达服务端的LB，可以是f5等硬件做负载均衡，还可以自行部署LVS+Keepalived等，前期量小可以直接用nginx。

4.网关层，请求通过LB后，会到达整个微服务体系的网关层Zuul，内嵌Ribbon做客户端负载均衡，Hystrix做熔断降级等。

5.服务注册，采用Eureka来做服务治理，Zuul会从Eureka集群获取已发布的微服务访问地址，然后根据配置把请求代理到相应的微服务去。

6.docker容器，所有的微服务模块都部署在docker容器里面，而且前后端的服务完全分开，各自独立部署后前端微服务调用后端微服务，后端微服务之间会有相互调用。

7.服务调用，微服务模块之间调用都采用标准的http/https+Rest+json的方式，调用技术采用feign+httpclient+ribbon+hystrix。

8.统一配置，每个微服务模块跟eureka集群、配置中心等进行交互。

9.第三方框架，每个微服务根据实现的需要，通常还要使用一些三方框架，常见的有：缓存服务redis、图片服务FastDFS、搜索服务ElasticSearch、安全管理shiro等。

10.mysql数据库，可以按照微服务模块进行拆分，统一访问公共库或者单独自己库，可以单独构建mysql集群或者分库分表mycat等。