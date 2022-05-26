cloud alibaba 2.1.0.RELEASE

# cloud变更

服务注册中心：（eureka停更不停用）Zookeeper  Consul <u>Nacos</u>    ；

服务调用1: Ribbon LoadBalancer(将取代ribbon)；

服务调用2：（Feign被取代了）  OpenFeign；

服务降级：(Hystrix被取代了)  resilience4j   <u>Sentienl</u>;

服务网关：(Zuul被取代了)  (Zuul2不稳定) gateway；

服务配置：(Config被取代了) Nacos；

服务总线：(Bus被取代了) Nacos;

# Spring Cloud Alibaba

https://spring-cloud-alibaba-group.github.io/github-pages/edgware/spring-cloud-alibaba.html 使用。本文本所借鉴处。

https://github.com/alibaba/spring-cloud-alibaba/blob/master/README-zh.md 文档

主要功能：

- **服务限流降级**：默认支持 WebServlet、WebFlux, OpenFeign、RestTemplate、Spring Cloud Gateway, Zuul, Dubbo 和 RocketMQ 限流降级功能的接入，可以在运行时通过控制台实时修改限流降级规则，还支持查看限流降级 Metrics 监控。
- **服务注册与发现**：适配 Spring Cloud 服务注册与发现标准，默认集成了 Ribbon 的支持。
- **分布式配置管理**：支持分布式系统中的外部化配置，配置更改时自动刷新。
- **消息驱动能力**：基于 Spring Cloud Stream 为微服务应用构建消息驱动能力。
- **分布式事务**：使用 @GlobalTransactional 注解， 高效并且对业务零侵入地解决分布式事务问题。
- **阿里云对象存储**：阿里云提供的海量、安全、低成本、高可靠的云存储服务。支持在任何应用、任何时间、任何地点存储和访问任意类型的数据。
- **分布式任务调度**：提供秒级、精准、高可靠、高可用的定时（基于 Cron 表达式）任务调度服务。同时提供分布式的任务执行模型，如网格任务。网格任务支持海量子任务均匀分配到所有 Worker（schedulerx-client）上执行。
- **阿里云短信服务**：覆盖全球的短信服务，友好、高效、智能的互联化通讯能力，帮助企业迅速搭建客户触达通道。

组件

**[Sentinel](https://github.com/alibaba/Sentinel)**：把流量作为切入点，从流量控制、熔断降级、系统负载保护等多个维度保护服务的稳定性。

**[Nacos](https://github.com/alibaba/Nacos)**：一个更易于构建云原生应用的动态服务发现、配置管理和服务管理平台。

**[RocketMQ](https://rocketmq.apache.org/)**：一款开源的分布式消息系统，基于高可用分布式集群技术，提供低延时的、高可靠的消息发布与订阅服务。

**[Dubbo](https://github.com/apache/dubbo)**：Apache Dubbo™ 是一款高性能 Java RPC 框架。

**[Seata](https://github.com/seata/seata)**：阿里巴巴开源产品，一个易于使用的高性能微服务分布式事务解决方案。

**[Alibaba Cloud OSS](https://www.aliyun.com/product/oss)**: 阿里云对象存储服务（Object Storage Service，简称 OSS），是阿里云提供的海量、安全、低成本、高可靠的云存储服务。您可以在任何应用、任何时间、任何地点存储和访问任意类型的数据。

**[Alibaba Cloud SchedulerX](https://help.aliyun.com/document_detail/43136.html)**: 阿里中间件团队开发的一款分布式任务调度产品，提供秒级、精准、高可靠、高可用的定时（基于 Cron 表达式）任务调度服务。

**[Alibaba Cloud SMS](https://www.aliyun.com/product/sms)**: 覆盖全球的短信服务，友好、高效、智能的互联化通讯能力，帮助企业迅速搭建客户触达通道。

更多组件请参考 [Roadmap](https://github.com/alibaba/spring-cloud-alibaba/blob/master/Roadmap-zh.md)。

这里使用的版本为2.1.0 

父工程依赖 cloud2020

```
<!--      spring cloud alibaba 2.1.0.RELEASE-->
<dependency>
    <groupId>com.alibaba.cloud</groupId>
    <artifactId>spring-cloud-alibaba-dependencies</artifactId>
    <version>2.1.0.RELEASE</version>
    <type>pom</type>
    <scope>import</scope>
</dependency>
```

# Nacos服务注册和配置中心

nacos=eureka+config+bus

**下载安装运行**

账号密码都是nacos

## 服务注册中心

**provider**

新增子模块cloudalibaba-provider-payment9001

```
<dependency>
    <groupId>com.alibaba.cloud</groupId>
    <artifactId>spring-cloud-starter-alibaba-nacos-discovery</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```

```
server:
  port: 9001
spring: 
  application:
   name: nacos-payment-provider
  cloud: 
    nacos:
      discovery: 
        server-addr: 127.0.0.1:8848  #配置nacos地址
management: 
  endpoints:
    web:
      exposure:
        include: '*'
```

```
@SpringBootApplication
@EnableDiscoveryClient
public class PaymentMain9001 {
   public static void main(String[] args) {
      SpringApplication.run(PaymentMain9001.class, args);
   }
}
```

```
@RestController
public class PaymentController {
   @Value("${server.port}")
   private String serverPort;
   @GetMapping("/payment/nacos/{id}")
   public String getPayment(@PathVariable("id") Integer id){
      return "nacos registry, server port :"+serverPort+"\t id : "+id;
   }
}
```

参照9001新建9002。

启动nacos 9001 9002,

进入nacos系统  服务管理->服务列表 可以看到服务和实例

访问http://localhost:9001/payment/nacos/22

**consumer**

```
<dependency>
    <groupId>com.alibaba.cloud</groupId>
    <artifactId>spring-cloud-starter-alibaba-nacos-discovery</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```

```
server:
  port: 83
spring:
  application:
    name: nacos-order-consumer
  cloud:
    nacos:
      discovery:
        server-addr: 127.0.0.1:8848  #配置nacos地址
#自定义 consumer要去访问的微服务名称（注册进nacos的provider） 用于在controller中获取，实现配置解耦,可以不写，直接在controller定义
service-url:
  nacos-user-service: http://nacos-payment-provider
```

```
@SpringBootApplication
@EnableDiscoveryClient
public class OrderNacosMain83 {
   public static void main(String[] args) {
      SpringApplication.run(OrderNacosMain83.class, args);
   }
}
```

config包下配置类

```
@Configuration
public class ApplicationContextConfig {
    @Bean
    @LoadBalanced
    public RestTemplate getRestTemplate(){
        return new RestTemplate();
    }
}
```

controller

```
@RestController
@Slf4j
public class OrderNacosController {
   @Resource
   private RestTemplate restTemplate;
   @Value("${service-url.nacos-user-service}")
   private String serverURL;
   @GetMapping("/consumer/payment/nacos/{id}")
   public String paymentInfo(@PathVariable("id") Long id){
      return restTemplate.getForObject(serverURL+"/payment/nacos/"+id,String.class);
   }
}
```

测试 启动nacos 9001 9002 83

访问http://localhost:83/consumer/payment/nacos/13

## 各注册中心对比

| 服务注册与发现框架 | CAP | 控制台管理 | 社区活跃度 |
| --------- | --- | ----- | ----- |
| Eureka    | AP  | 支持    | 低     |
| Zookeeper | CP  | 不支持   | 中     |
| Consul    | CP  | 支持    | 高     |
| Nacos     | AP  | 支持    | 高     |

Nacos还可以AP与CP切换。

临时实例：客户端上报健康状态；摘除不健康实例；非持久化

持久化实例：服务端探测健康状态；保留不健康实例；持久化

AP模式注册的为临时实例。CP模式下注册实例之前必须先注册服务，如果服务不存在，则会返回错误

## 配置中心

新建子模块cloudalibaba-config-nacos-client3377

```
<dependencies>
    <dependency>
        <groupId>com.alibaba.cloud</groupId>
        <artifactId>spring-cloud-starter-alibaba-nacos-config</artifactId>
    </dependency>
    <dependency>
        <groupId>com.alibaba.cloud</groupId>
        <artifactId>spring-cloud-starter-alibaba-nacos-discovery</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-actuator</artifactId>
    </dependency>
</dependencies>
```

```
@EnableDiscoveryClient
@SpringBootApplication
public class NacosConfigClientMain3377 {
   public static void main(String[] args) {
      SpringApplication.run(NacosConfigClientMain3377.class, args);
   }
}
```

bootstrap

```
server:
  port: 3377
spring:
  application:
    name: nacos-config-client
  cloud:
    nacos:
      discovery:
        server-addr: localhost:8848      #服务注册中心
      config:
        server-addr: localhost:8848      #配置中心
        file-extension: yaml     #指定yaml格式的配置
```

application

```
spring:
  profiles:
    active: dev  #开发环境配置文件
```

controller

```
@RestController
@RefreshScope     //client配置文件支持动态刷新
public class ConfigClientController {
   @Value("${config.info}")
   private String configInfo;

   @GetMapping("config/info")
   public String getConfigInfo(){
      return configInfo;
   }
}
```

**匹配规则**

nacos中的Data id的组成格式及与SpringBoot配置文件中的匹配规则

${prefix}-${spring.profiles.active}.${file-extension}

- `prefix` 默认为 `spring.application.name` 的值，也可以通过配置项 `spring.cloud.nacos.config.prefix`来配置。
- `spring.profiles.active` 即为当前环境对应的 profile，详情可以参考 [Spring Boot文档](https://docs.spring.io/spring-boot/docs/current/reference/html/boot-features-profiles.html#boot-features-profiles)。 **注意：当 `spring.profiles.active` 为空时，对应的连接符 `-` 也将不存在，dataId 的拼接格式变成 `${prefix}.${file-extension}`**
- `file-exetension` 为配置内容的数据格式，可以通过配置项 `spring.cloud.nacos.config.file-extension` 来配置。目前只支持 `properties` 和 `yaml` 类型。

在client文件中实际上等价于：${spring.application.name}-${spring.profiles.active}.${spring.cloud.nacos.config.file-extension}

因此，前面bootstrap.yml中表示匹配的配置文件 nacos-config-client-dev.yaml

在nacos系统中新建配置

Data ID:nacos-config-client-dev.yaml

文件格式选择yaml。内容

config:

  info: from nacos config center, nacos-config-client-dev.yaml, version=1

点击发布。查看配置列表

测试：启动nacos 3377

访问：http://localhost:3377/config/info

注意：这里配置中心的文件名必须为yaml。yml不能识别，这是nacos本身的问题，后面的版本已经修复。

**自带动态刷新**

修改Nacos中的yaml配置文件，再次访问，就会发现配置以及刷新。

## 分类配置

多环境配置

Namespace+Group+Data ID 

类似java里面的package名和类名

最外层的namespace是可以用于区分部署环境的，Group和DataID逻辑上区分两个目标对象。

默认：

Namespace=public 

Group=DEFAULT_GROUP

Cluster默认为DEFAULT 

假如有三个环境：开发测试生产，就可以创建3各Namespace,不同的namespace隔离。

Group可与把不同的微服务划分到同一个分组里面去。

一个微服务可以包含多个Cluster（集群），Cluster是对指定微服务的一个虚拟划分。

比如说为了容灾，将Service微服务分别部署在了杭州机房和广州机房，这是就可以给杭州机房的微服务起一个集群名称HZ，给广州机房的微服务起一个集群名称GZ，还可以尽量让同一个机房的微服务互相调用以提升性能。

### Data ID方案

新建另外一个Data ID为nacos-config-client-test.yaml的配置文件 

通过spring-profile.active属性进行多环境下配置文件读取

将application.yml修改为

```
spring:
  profiles:
    active: test  #测试环境配置文件
#    active: dev  #开发环境配置文件
```

结合bootstrap.yml，匹配的配置文件就应该是nacos-config-client-test.yaml

测试，启动nacos 3377

http://localhost:3377/config/info

### Group方案

新建配置文件 1

Data ID :nacos-config-client-dev.yaml

Group:DEV_GROUP

内容

config:
  info: from nacos config center  group dev-group, nacos-config-client-dev.yaml, version=1

新建配置文件 2

Data ID :nacos-config-client-info.yaml

Group:TEST_GROUP

内容

config:
  info: from nacos config center  group test-group, nacos-config-client-info.yaml, version=1

如何使用？

在config属性下增加一条group的配置即可。

使用DEV_GROUP组的dev环境下的配置文件：

```
config:
  server-addr: localhost:8848      #配置中心
  file-extension: yaml     #指定yaml格式的配置
  group: DEV_GROUP          #选择分组
```

```
spring:
  profiles:
#    active:  test  #测试环境配置文件
    active: dev  #开发环境配置文件
```

http://localhost:3377/config/info

使用TEST_GROUP组的info环境下的配置文件：

```
config:
  server-add r: localhost:8848      #配置中心
  file-extension: yaml     #指定yaml格式的配置
  group: TEST_GROUP          #选择分组
```

```
spring:
  profiles:
    active: info
```

http://localhost:3377/config/info

### Namespace方案

新建dev和test两个命名空间，可以看到左上角多了两个命名空间空间。前面所建的所有配置文件都在public中。

在dev命名空间创建配置文件：

Data ID :nacos-config-client-dev.yaml

Group:DEV_GROUP

内容

config:
  info: from nacos config center ,from group dev-group ,from dev namespace , nacos-config-client-dev.yaml, version=1

如何访问？

在nacos系统中 复制dev命名空间ID，在config属性下增加一条属性namespace配置即可。

```
config:
  server-addr: localhost:8848      #配置中心
  file-extension: yaml     #指定yaml格式的配置
  group: DEV_GROUP          #选择分组
  namespace: fa3aefda-4113-4ac8-8844-afcd9235be08   #选择命名空间
```

```
spring:
  profiles:
#    active:  test  #测试环境配置文件
    active: dev  #开发环境配置文件
```

http://localhost:3377/config/info

## 集群和持久化

https://nacos.io/zh-cn/docs/cluster-mode-quick-start.html 集群部署官方文档

vip:virtual ip    可以使用nginx作为负载均衡器

nacos默认使用嵌入式数据库derby实现数据的存储（电脑重启后数据还在）。所以如果启动多个默认配置下的nacos节点，数据存储是存在一致性问题的（每一个nacos节点一份derby存储）。为了解决这个问题nacos采用了集中式存储的方式来支持集群化部署，目前只支持mysql的存储。

nacos支持的三种部署模式：

1.单机模式-用于测试和单机试用。

2.集群模式-用于生产环境，确保高可用。

3.多集群模式-用于多数据中心场景。

如何持久化？

derby切换到mysql配置步骤：

https://nacos.io/zh-cn/docs/deployment.html

1.nacos-server-1.1.4\nacos\conf目录下找到sql脚本：nacos-mysql.sql，

新建mysql数据库 nacos_config执行该脚本。

2.nacos-server-1.1.4\nacos\conf目录下找到文件application.properties,添加内容

```properties
spring.datasource.platform=mysql

db.num=1
db.url.0=jdbc:mysql://127.0.0.1:3306/nacos_config?characterEncoding=utf8&connectTimeout=1000&socketTimeout=3000&autoReconnect=true
db.user=root
db.password=350562
```

从此以后nacos数据从derby改为mysql。

开启necos，并进入， 可以看到以前的配置清空了，说明迁移成功。

验证：

随机新建配置，在mysql的nacos数据库的config_info表中可以看到配置信息。

### Linux版生产环境配置

安装步骤

```
需要1个nginx，3个nacos注册中心，1个mysql
到上面的网址下载nacos-server-1.1.4.tar.gz,传到linux上后，解压后安装。
或者在有网的情况下
wget -P /opt https://github.com/alibaba/nacos/releases/download/1.1.4/nacos-server-1.1.4.tar.gz
tar -zxvf  nacos-server-1.1.4.tar.gz
```

部署

问题：一个nacos: startup 8848

3个nacos: 3333 4444 5555 如何startup（实际中一般还是部署在不同的机器上）?

1.修改application.properties,可复制一份application.properties.bk。

注意：因为windows上面前面已经对nacos的mysql脚本执行，所以linux上的nacos使用win上的mysql。

所以下面mysql地址为win的ip.

```
spring.datasource.platform=mysql

db.num=1
db.url.0=jdbc:mysql://192.168.153.1:3306/nacos_config?characterEncoding=utf8&connectTimeout=1000&socketTimeout=3000&autoReconnect=true
db.user=root
db.password=350562
```

2.修改nacos集群配置文件cluster.conf。

因为只有cluster.conf.example,所以复制一份:

cp cluster.conf cluster.conf.example

修改cluster.conf，注释掉其他，添加：

192.168.153.134:3333
192.168.153.134:4444
192.168.153.134:5555     

注意：这里面的ip不能写127.0.0.1，为ifconfig下ens33下的地址。

3.编辑nacos的启动脚本startup.sh，使他能接受不同的启动端口

复制一份startup.sh.bk,修改startup.sh,

```
中间
while getopts ":m:f:s:" opt    
修改为
while getopts ":m:f:s:p:" opt 
并在下面的case里添加
    p)
      PORT=$OPTARG
```

```
最下方
nohup $JAVA ${JAVA_OPT} nacos.nacos >> ${BASE_DIR}/logs/start.out 2>&1 &
修改为
hup $JAVA -Dserver.port=${PORT} ${JAVA_OPT} nacos.nacos >> ${BASE_DIR}/logs/start.out 2>&1 &
```

4.nginx配置，需要安装nginx和nginx知识。

在nginx/conf下复制nginx.conf.bk，修改nginx.conf

```
修改为
server{
    listen        1111;
    server_name   localhost;

    location /{
         #root     html;
         #index    index.html index.htm;
         proxy_pass http://cluster;
    }
}
在上面添加(#gzip on;的下方)
upstream cluster{
     server 127.0.0.1:3333;
     server 127.0.0.1:4444;
     server 127.0.0.1:5555;
}
```

5.最后的启动:

nacos/bin下：

./startup.sh - p 3333

./startup.sh - p 4444

./startup.sh - p 5555

以集群方式启动，再验证：ps -ef|grep nacos|grep -v grep|wc -l 是否返回3.

在nginx/sbin下

./nginx -c /usr/local/nginx/conf/nginx.conf

验证 ps -ef|grep nginx 查看是否启动

通过nginx访问nacos http://192.168.153.134:1111/nacos/#/login

新建一个配置，查看mysql nacos_config  config_info.

微服务cloudalibaba-provider-payment9002启动注册进nacos集群:

server-addr: 192.168.153.134:1111

# Sentinel

类似hystrix的alibaba版本,容错限流监控。

https://github.com/alibaba/Sentinel/wiki/%E4%BB%8B%E7%BB%8D

分为两部分：

1.核心库（java客户端），不依赖任何框架/库，能够运行于所有java运行时环境，用时对dubbo/cloud等框架有较好支持。

2.控制台（dashboard）基于spring boot开发，打包好可以直接运行，不需要额外的tomcat等应用容器。

**下载安装运行**

https://github.com/alibaba/Sentinel/releases/tag/1.7.0

[sentinel-dashboard-1.7.0.jar](https://github.com/alibaba/Sentinel/releases/download/1.7.0/sentinel-dashboard-1.7.0.jar)

运行前提 ,java8  8080端口可用（tomcat默认端口）

http://localhost:8080/

用户名密码  sentinel

**客户端**

新建子模块cloudalibaba-sentinel-service8401

```
<!--        后续作持久化-->
        <dependency>
            <groupId>com.alibaba.csp</groupId>
            <artifactId>sentinel-datasource-nacos</artifactId>
        </dependency>
<!--        sentinel-->
        <dependency>
            <groupId>com.alibaba.cloud</groupId>
            <artifactId>spring-cloud-starter-alibaba-sentinel</artifactId>
        </dependency>
<!--        nacos-->
        <dependency>
            <groupId>com.alibaba.cloud</groupId>
            <artifactId>spring-cloud-starter-alibaba-nacos-discovery</artifactId>
        </dependency>
        <dependency>
            <groupId>edu.hunnu.cloud</groupId>
            <artifactId>cloud-api-commons</artifactId>
            <version>1.0-SNAPSHOT</version>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-actuator</artifactId>
        </dependency>
```

```
server:
  port: 8401
spring:
  application:
    name: cloudalibaba-sentinel-service
  cloud:
    nacos:
      discovery:
        server-addr: 127.0.0.1:8848  #配置nacos地址
    sentinel:
      transport:
        dashboard: localhost:8080   #8080监控8401
        port: 8719 #默认端口 如果8719被占用,会往后依次寻找未被占用的端口
management:
  endpoints:
    web:
      exposure:
        include: '*'
```

```
@SpringBootApplication
@EnableDiscoveryClient
public class MainApp8401 {
   public static void main(String[] args) {
      SpringApplication.run(MainApp8401.class, args);
   }
}
```

```
@RestController
public class FlowLimitController {
   @GetMapping("/testA")
   public String testA(){
      return  "---------------testA";
   }
   @GetMapping("/testB")
   public String testB(){
      return  "---------------testB";
   }
}
```

启动sentinel8080 nacos8848   8401

查看sentinel网页发现什么也没有，因为其采用懒加载机制。

访问依次controller即可http://localhost:8401/testA

再次刷新sentinel。

多次访问http://localhost:8401/testA查看实时监控变化。

## 流量控制

介绍和理论见官方

- 资源名：资源名，即限流规则的作用对象，唯一名称，默认请求路径。

- 针对来源：sentinel可以针对调用者进行限流，填写微服务名，默认default不区分来源。

- 阈值类型/单击阈值:

QPS（每秒的请求数量）：当调用该api的QPS达到阈值的时候，进行限流。

线程数：当调用该api的线程数达到阈值的时候，进行限流。

- 是否集群：不需要

- 流控模式：

直接：api达到限流条件时直接限流。

关联：当关联的资源达到阈值时，就限流自己。

链路：只记录指定链路上的流量（指定资源从入口资源进来的流量，如果达到阈值，就进行限流），api级别的针对来源。

- 流控效果

快速失败：直接失败，抛异常。

Warm Up：根据codeFactor（冷加载因子，默认3）的值，从阈值/codeFactor，经过预热时长，才达到设置的QPS阈值。

排队等待：匀速排队，让请求以匀速的速度通过，阈值类型必须设置为QPS，否则无效

### 流控模式

设置马上生效！

1.**直接**（默认）

1.1 **QPS**

簇点链路（可用列表视图），+流控

或者 流量规则 + 

资源名 /testA

针对来源：default

阈值类型：QPS

单击阈值：1

流控模式：直接

流控效果：快速失败

表示访问该服务的QPS超过1就直接报错。

访问http://localhost:8401/testA 如果过快（1s超过1次）就报错Blocked by Sentinel (flow limiting)

问题：如何返回我们自己的处理？

1.2 **线程数**

簇点链路（可用列表视图），+流控

或者 流量规则 + 

资源名 /testA

针对来源：default

阈值类型：线程数

单击阈值：1

流控模式：直接

流控效果：无

为了演示效果，在testA的方法中添加延迟，

```
try {
   TimeUnit.MILLISECONDS.sleep(800);
} catch (InterruptedException e) {
   e.printStackTrace();
}
```

多窗口多次访问http://localhost:8401/testA 

总结,qps拦截门外；线程数关门打狗。

2.**关联**

当与A关联的资源B达到阈值后，就限流A自己。

比如：支付接口达到阈值的时候，就限流下订单接口。

资源名 /testA

针对来源：default

阈值类型：QPS

单击阈值：1

流控模式：关联

关联资源：/testB

流控效果：快速失败

postman模拟并发密集访问testB（将请求信息保存save as到collection中，Run Collection, 设置Iterations:20,Delay:300，表示20个请求每隔0.3s访问一次）,运行后发现testA挂了。等20个线程结束，可重新访问testA.

3.**链路**

多个请求调用了同一微服务。

他处查阅。

### 流控效果

1.快速开始

2.Warm up 

默认coldFacotor为3，即请求QFS从 threshold/3开始，经过预热时长逐渐升至设定的QPS threshold

资源名 /testA

针对来源：default

阈值类型：QPS

单击阈值：10

流控模式：直接

流控效果：Warm up

预热时长：5

表示一开始QFS单击阈值从10/3=3开始，慢慢地在5s后恢复到10。

不停访问http://localhost:8401/testA，开始会报错，后面慢慢不报错了。

3.排队等待

用于处理间隔性突发的流量。

资源名 /testA

针对来源：default

阈值类型：QPS

单击阈值：1

流控模式：直接

流控效果：排队等待

超时时间：20000

表示每秒一次请求，超过时间就排队等待，等待的超时时间为20000ms。

为了演示效果，在请求方法里添加

```
log.info(Thread.currentThread().getName()+"\t...testA");
```

postman模拟访问（collection中添加3个请求，run Iterations:5,Delay:1000）测试,一共15个请求 ，控制台是否在1s一个输出15个。

## 服务降级

降级策略

- 平均响应时间 (`DEGRADE_GRADE_RT`)：当 1s 内持续进入 5 个请求，对应时刻的平均响应时间（秒级）均超过阈值（`count`，以 ms 为单位），那么在接下的时间窗口（`DegradeRule` 中的 `timeWindow`，以 s 为单位）之内，对这个方法的调用都会自动地熔断（抛出 `DegradeException`）。注意 Sentinel 默认统计的 RT 上限是 4900 ms，**超出此阈值的都会算作 4900 ms**，若需要变更此上限可以通过启动配置项 `-Dcsp.sentinel.statistic.max.rt=xxx` 来配置。
- 异常比例 (`DEGRADE_GRADE_EXCEPTION_RATIO`)：当资源的每秒请求量 >= 5（可配置），并且每秒异常总数占通过量的比值超过阈值（`DegradeRule` 中的 `count`）之后，资源进入降级状态，即在接下的时间窗口（`DegradeRule` 中的 `timeWindow`，以 s 为单位）之内，对这个方法的调用都会自动地返回。异常比率的阈值范围是 `[0.0, 1.0]`，代表 0% - 100%。
- 异常数 (`DEGRADE_GRADE_EXCEPTION_COUNT`)：当资源近 1 分钟的异常数目超过阈值之后会进行熔断。注意由于统计时间窗口是分钟级别的，若 `timeWindow` 小于 60s，则结束熔断状态后仍可能再进入熔断状态。

sentinel的断路器没有半开状态。sentinel的服务降级就是hystrix的服务熔断。

### 降级策略

1.RT（平均响应时间，秒级）：平均响应时间 超出阈值 且 在时间窗口内通过的请求>=5，两个条件同时满足后触发降级。窗口期过后关闭断路器。

```
@GetMapping("/testD")
public String testD(){
   /**
    * 演示 测试服务降级 RT  1S
    */
   try {
            TimeUnit.SECONDS.sleep(1);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
   log.info("testD 测试RT");
   return  "---------------testD";
}
```

新增降级规则

资源名：/testD

降级策略：RT

RT：200

时间窗口：1

表示在1s内如果请求数为5个以上，要求平均响应时间不大于200ms，即0.2s响应一个请求。

jmeter压测：新建线程组：

名称：线程组RT

线程数 10

循环次数 永远

添加http请求取样器

表示1s 10个请求 。

因为方法中添加了延迟1s,再加上1s内请求数超过5个，所以开启线程组后浏览器访问testD，发现已经被降级不能访问。关闭线程组，恢复访问，但是如果刷新过快,也会服务降级。

2.异常比列（秒级）：QPS>=5且异常比例（秒级统计）超过阈值时，触发降级；时间窗口结束后，关闭降级。

```
    @GetMapping("/testD")
   public String testD(){
      /**
       * 演示 测试服务降级 异常比例
       */
      log.info("testD 测试 异常比例");
      int age = 10/0;
      return  "---------------testD";
   }
}
```

资源名：/testD

降级策略：异常比例

异常比例：0.2

时间窗口：3

同样开启上面配置的线程组。访问testD。发现本来报算术异常的访问被降级，关闭线程组，再访问又出现异常（因为QPS<5）。

3.异常数（分钟统计）：超过阈值(近1分钟内)时触发降级，时间窗口结束后关闭降级。注意由于统计时间窗口是分钟级别的，若timeWindow小于60s，则结束熔断状态后仍可能再进入熔断状态。

资源名：/testD

降级策略：异常数

异常数：5

时间窗口：70

访问testE，70s内访问5次以后，降级。

## 热点限流

何为热点？热点即经常访问的数据。很多时候我们希望统计某个热点数据中访问频次最高的 Top K 数据，并对其访问进行限制。比如：

- 商品 ID 为参数，统计一段时间内最常购买的商品 ID 并进行限制
- 用户 ID 为参数，针对一段时间内频繁访问的用户 ID 进行限制

热点参数限流会统计传入参数中的热点参数，并根据配置的限流阈值与模式，对包含热点参数的资源调用进行限流。热点参数限流可以看做是一种特殊的流量控制，仅对包含热点参数的资源调用生效。

Sentinel 利用 LRU 策略统计最近最常访问的热点参数，结合令牌桶算法来进行参数级别的流控。热点参数限流支持集群模式。

SentinelResource类似hystrix的HystrixCommand。

**使用**

```
//value随意，只要求唯一    
//blockHandler表示违背了热点规则 则执行指定兜底方法，使用热点限流一定要配，否则返回异常页面而不会像前面一样返回sentienl默认提示
    @SentinelResource(value = "testHotKey",blockHandler = "deal_testHotKey")
    @GetMapping("/testHotKey")
    public String testHotKey(@RequestParam(value="p1",required = false) String p1,
                             @RequestParam(value="p2",required = false) String p2){
        return "---------------testHotKey  ";
    }

    public String deal_testHotKey(String p1, String p2, BlockException exception){
        //用来替换sentinel默认提示
        return "---------------deal_testHotKey  热点限流";
    }
```

新增热点规则：

资源名：testHotKey

限流模式：QPS模式

参数索引：0

单机阈值：1

统计窗口时长：1

注意：这里资源名@SentinelResource指定的value，为参数索引0表示第一个参数(在这里即p1)。

这里表示对资源的第1个参数进行热点限流，1秒内只能访问一次。

多次访问

http://localhost:8401/testHotKey?p1=a  ×

http://localhost:8401/testHotKey?p1=aa&p2=b  ×

http://localhost:8401/testHotKey?p2=b   √

### 参数例外项

前面是对所有带有某个指定参数进行限流。

如果我们期望某个参数是某个特殊值时，对他进行特定的限流规则时，怎么办？

假如：当参数p1的值等于5时，它的阈值可以达到200

修改上述的热点规则：

点击参数例外项：

参数类型 ：String

参数值 ： 5

限流阈值:   200

注意，设置了参数例外项一定要点添加按钮，否则不生效。

多次访问：

http://localhost:8401/testHotKey?p1=1   ×

http://localhost:8401/testHotKey?p1=5   √

### 注意

@SentinelResource处理的是Sentinel控制台配置的违规情况，有blockHandler方法的兜底处理。

但是@SentinelResource不处理运行时的异常。

测试，在方法中添加  int a=10/0;

## 系统限流

系统保护规则是从应用级别的入口流量进行控制，从单台机器的 load、CPU 使用率、平均 RT、入口 QPS 和并发线程数等几个维度监控应用指标，让系统尽可能跑在最大吞吐量的同时保证系统整体的稳定性。

系统保护规则是应用整体维度的，而不是资源维度的，并且**仅对入口流量生效**。入口流量指的是进入应用的流量（`EntryType.IN`），比如 Web 服务或 Dubbo 服务端接收的请求，都属于入口流量。

系统规则支持以下的模式：

- **Load 自适应**（仅对 Linux/Unix-like 机器生效）：系统的 load1 作为启发指标，进行自适应系统保护。当系统 load1 超过设定的启发值，且系统当前的并发线程数超过估算的系统容量时才会触发系统保护（BBR 阶段）。系统容量由系统的 `maxQps * minRt` 估算得出。设定参考值一般是 `CPU cores * 2.5`。
- **CPU usage**（1.5.0+ 版本）：当系统 CPU 使用率超过阈值即触发系统保护（取值范围 0.0-1.0），比较灵敏。
- **平均 RT**：当单台机器上所有入口流量的平均 RT 达到阈值即触发系统保护，单位是毫秒。
- **并发线程数**：当单台机器上所有入口流量的并发线程数达到阈值即触发系统保护。
- **入口 QPS**：当单台机器上所有入口流量的 QPS 达到阈值即触发系统保护。

入口QPS

新增系统规则

阈值类型：入口QPS

阈值：1

访问

```
@GetMapping("/testB")
public String testB(){
   return  "---------------testB";
}
```

## SentinelResource

新建

```
@RestController
public class RateLimitController {
   @GetMapping("/byResource")
   @SentinelResource(value = "byResource",blockHandler = "handleException")
   public CommonResult byResource(){
      return new CommonResult(200,"按资源名称限流测试",new Payment(2020L,"serial001"));
   }

   public CommonResult handleException(BlockException exception){
      return new CommonResult(444,exception.getClass().getCanonicalName()+"\t 服务不可用");
   }
}
```

1.按资源名称限流

新增流控规则

资源名：byResource         **无  /**

针对来源：default

阈值类型：QPS

单击阈值：1

1s多次访问http://localhost:8401/byResource

com.alibaba.csp.sentinel.slots.block.flow.FlowException     服务不可用。

问题：在此时关闭8401，再查看sentinel控制台的流控规则，发现刚刚定义的规则消失了。

所以这里是临时规则。

2.按url地址限流

```
/**
 * 按资url限流测试  
 */
@GetMapping("/byUrl")
@SentinelResource(value = "byUrl")
public CommonResult byUrl(){
   return new CommonResult(200,"按url名称限流测试",new Payment(2020L,"serial002"));
}
```

新增流控规则

资源名：/byUrl     **有 /**

针对来源：default

阈值类型：QPS

单击阈值：1

1s多次访问http://localhost:8401/byUrl

Blocked by Sentinel (flow limiting)

区别：

1使用@SentinelResource的value属性通过资源名定义规则如果违反规则的话走的是blockHander兜底方法。

2使用@GetMapping的value属性通过url定义规则如果违反规则的话走的是自带的兜底方法，即返回

Blocked by Sentinel (flow limiting)。

 综合问题：（与Hystrix前期问题一样）

1.如果使用系统默认的兜底方法，没有体现业务要求；

2.自定义的兜底方法与业务代码耦合；

3，每个业务方法都添加一个兜底方法，代码膨胀；

4，全局统一的处理方法没有体现。

## 自定义限流处理

创建CustomerBlockHandler类用于自定义限流处理逻辑。

```
public class CustomerBlockHandler {
   public static CommonResult handlerException(BlockException exception){
      return new CommonResult(444,"按客户自定义,CoustomerBlockHandler全局限流处理---1号");
   }
   public static CommonResult handlerException2(BlockException exception){
      return new CommonResult(444,"按客户自定义,CoustomerBlockHandler全局限流处理---3号");
   }
}
```

controller方法@SentinelResource指定block handler 类名和方法名

```
@GetMapping("/customerBlockHanlder")
@SentinelResource(value = "customerBlockHanlder",
      blockHandlerClass = CustomerBlockHandler.class,
      blockHandler = "handlerException2")
public CommonResult customerBlockHanlder(){
   return new CommonResult(200,"按客户自定义",new Payment(2020L,"serial003"));
}
```

新增流控规则

资源名：customerBlockHanlder        无  /

针对来源：default

阈值类型：QPS

单击阈值：1

1s内多次访问http://localhost:8401/customerBlockHanlder

完成代码解耦！

## 服务熔断

sentinel结合ribbon或者feign

### Ribbon系列

参照cloudalibaba-provider-payment9001 新建 9003 9004 provider

```
@RestController
public class PaymentController {
   @Value("${server.port}")
   private String serverPort;
   public static HashMap<Long, Payment> hashMap = new HashMap<>();
   static {
      hashMap.put(1L,new Payment(1L, UUID.randomUUID().toString()));
      hashMap.put(2L,new Payment(2L, UUID.randomUUID().toString()));
      hashMap.put(3L,new Payment(3L, UUID.randomUUID().toString()));
   }
   @GetMapping("/paymentSQL/{id}")
   public CommonResult paymentSQL(@PathVariable("id") Long id){
      Payment payment = hashMap.get(id);
      return new CommonResult(200,"from mysql,serverPort: "+serverPort,payment);
   }
}
```

启动9003 9004

访问http://localhost:9003/paymentSQL/3

http://localhost:9004/paymentSQL/3

参照cloudalibaba-consumer-nacos-order83新建cloudalibaba-consumer-nacos-order84

```
<dependency>
    <groupId>com.alibaba.cloud</groupId>
    <artifactId>spring-cloud-starter-alibaba-sentinel</artifactId>
</dependency>
```

```
server:
  port: 84
spring:
  application:
    name: nacos-order-consumer
  cloud:
    nacos:
      discovery:
        server-addr: 127.0.0.1:8848  #配置nacos地址
    sentinel:
      transport:
        dashboard: 127.0.0.1:8080 #配置sentinel dashboard地址
        port: 8719   #客户端提供给dashboard访问或者查看sentinel的运行访问的参数：
#自定义 consumer要去访问的微服务名称（注册进nacos的provider） 用于在controller中获取，实现配置解耦,可以不写，直接在controller定义
service-url:
  nacos-user-service: http://nacos-payment-provider
```

```
@SpringBootApplication
@EnableDiscoveryClient
public class OrderNacosMain84 {
   public static void main(String[] args) {
      SpringApplication.run(OrderNacosMain84.class, args);
   }
}
```

config包下

```
@Configuration
public class ApplicationContextConfig {
   @Bean
   @LoadBalanced
   public RestTemplate getRestTemplate(){
      return new RestTemplate();
   }
}
```

controller

```
@RestController
@Slf4j
public class CircleBreakerController {
    public static final String SERVICE_URL = "http://nacos-payment-provider";
    @Resource
    private RestTemplate restTemplate;

    @RequestMapping("/consumer/fallback/{id}")
    @SentinelResource(value = "fallback")    //没有配置
//    @SentinelResource(value = "fallback", fallback = "handerFallback")   //fallback只负责业务异常
//    @SentinelResource(value = "fallback", blockHandler = "blockFallback")//block只负责sentinel控制台配置违规
//    @SentinelResource(value = "fallback", fallback = "handerFallback",blockHandler = "blockFallback")
    public CommonResult<Payment> fallback(@PathVariable Long id) {
        CommonResult<Payment> result = restTemplate.getForObject(SERVICE_URL + "/paymentSQL/" + id, CommonResult.class, id);
        if (id == 4) {
            throw new IllegalArgumentException(" IllegalArgumentException,非法参数异常————————————————");
        } else if (result.getData() == null) {
            throw new NullPointerException("NullPointerException,该id没有对应记录，空值异常——————————————————");
        }
        return result;
    }

    public CommonResult<Payment> handerFallback(@PathVariable Long id,Throwable throwable){
        return new CommonResult<>(4443,"fallback兜底负责业务异常处理------>"+throwable.getMessage(),new Payment(id,null));
    }
    public CommonResult<Payment> blockFallback(@PathVariable Long id, BlockException exception){
        return new CommonResult<>(4444,"blockHandler兜底负责sentinel控制台配置违规------>"+exception.getMessage(),new Payment(id,null));
    }
}
```

fallback管运行异常

block管配置违规

测试：启动9003 9004 84

访问http://localhost:84/consumer/fallback/1  

http://localhost:84/consumer/fallback/4   

http://localhost:84/consumer/fallback/5   

1.@SentinelResource(value = "fallback")    //没有配置。

查询 1  2  3 正常 

查询4 非法参数异常 error界面 不友好

查询 5 空指针异常 error界面 不友好

2.@SentinelResource(value = "fallback", fallback = "handerFallback")   //fallback只负责业务异常。

查询 1  2  3 正常 ;

查询4 返回fallback异常兜底处理后的信息 ;

查询 5 返回fallback异常兜底处理后的信息 ;

比较：在HystrixCommand里， fallback代表的是服务降级的兜底方法。

3.@SentinelResource(value = "fallback", blockHandler = "blockFallback")//block只负责sentinel控制台配置违规。

sentinel控制台添加流控或降级方法:(这里添加降级)

```
资源名：fallback
降级策略：异常数
异常数：2
时间窗口：5
```

查询 1  2  3 正常 ;

查询4 出现error界面 多次error违反规则后返回blockHandler配置兜底处理后的信息 ;

查询 5 出现error界面 多次error违反规则后返回blockHandler配置兜底处理后的信息 ;

4.@SentinelResource(value = "fallback", fallback = "handerFallback",blockHandler = "blockFallback")。

sentinel控制台添加流控或降级方法:(这里添加流控)

```
资源名：fallback
针对来源：default
阈值类型：QPS
单机阈值：1
```

查询 1  2  3 1s内1次正常，违反规则后返回配置兜底处理后的信息

查询4 在1s内1次时返回fallback异常兜底的处理信息，违反规则后返回blockHandler配置兜底处理后的信息

查询 5 在1s内1次时返回fallback异常兜底的处理信息，违反规则后返回blockHandler配置兜底处理后的信息

**总结：sentinel配置的流控或降级blockHandle处理优先。**

附加：异常忽略

```
    @RequestMapping("/consumer/fallback/{id}")
   @SentinelResource(value = "fallback", 
   fallback = "handerFallback",
   blockHandler = "blockFallback"，
   exceptionsToIgnore = {IllegalArgumentException.class})
```

访问http://localhost:84/consumer/fallback/4

返回不友好的error界面

### Feign系列

参考84新建cloudalibaba-consumer-nacos-order85

```
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-openfeign</artifactId>
</dependency>
```

```
server:
  port: 85
spring:
  application:
    name: nacos-order-consumer
  cloud:
    nacos:
      discovery:
        server-addr: 127.0.0.1:8848  #配置nacos地址
    sentinel:
      transport:
        dashboard: 127.0.0.1:8080 #配置sentinel dashboard地址
        port: 8719   #客户端提供给dashboard访问或者查看sentinel的运行访问的参数：
#自定义 consumer要去访问的微服务名称（注册进nacos的provider） 用于在controller中获取，实现配置解耦,可以不写，直接在controller定义
service-url:
  nacos-user-service: http://nacos-payment-provider
#激活sentinel对feign的支持
feign:
  sentinel:
    enabled: true
```

```
@EnableDiscoveryClient
@SpringBootApplication
@EnableFeignClients
public class OrderNacosMain85 {
   public static void main(String[] args) {
      SpringApplication.run(OrderNacosMain85.class, args);
   }
}
```

service包下

```
@FeignClient(value = "nacos-payment-provider",fallback = PaymentFallbackService.class)
public interface PaymentService {
    @GetMapping(value = "/paymentSQL/{id}")
    CommonResult<Payment> paymentSQL(@PathVariable("id") Long id);
}
```

```
@Component
public class PaymentFallbackService implements PaymentService{
   @Override
   public CommonResult<Payment> paymentSQL(Long id) {
      return new CommonResult<>(44444,"feign服务降级返回",new Payment(id,"error serial"));
   }
}
```

controller

```
@RestController
@Slf4j
public class CircleBreakerController {
   @Resource
   private PaymentService paymentService;
   @GetMapping("/consumer/paymentSQL/{id}")
   public CommonResult<Payment> paymentSQL(@PathVariable("id") Long id){
      return paymentService.paymentSQL(id);
   }
}
```

测试 启动9003 85

http://localhost:85/paymentSQL/1  可以正常访问

故意关闭9003，再访问，服务降级处理

## 规则持久化

前面提到过，如果重启配置了规则的服务器，则sentinel控制台会丢失此微服务的规则。

https://gitee.com/rmlb/Sentinel/wikis/%E5%9C%A8%E7%94%9F%E4%BA%A7%E7%8E%AF%E5%A2%83%E4%B8%AD%E4%BD%BF%E7%94%A8-Sentinel?sort_id=3419417

解决办法：

将限流配置规则持久化到nacos保存，只要刷新8401某个rest地址，sentinel控制台的流控规则就能看到，只要nacos里面的配置不删除，针对8401上sentinel上的流控规则持续有效。

```
<!--        后续作持久化-->
        <dependency>
            <groupId>com.alibaba.csp</groupId>
            <artifactId>sentinel-datasource-nacos</artifactId>
        </dependency>
```

```
server:
  port: 8401
spring:
  application:
    name: cloudalibaba-sentinel-service
  cloud:
    nacos:
      discovery:
        server-addr: 127.0.0.1:8848  #配置nacos地址
    sentinel:
      transport:
        dashboard: localhost:8080   #8080监控8401
        port: 8719 #默认端口 如果8719被占用,会往后依次寻找未被占用的端口
      datasource: 
        ds1:
          nacos:
            server-addr: localhost:8848
            dataId: cloudalibaba-sentinel-service
            groupId: DEFAULT_GROUP
            data-type: json
            rule-type: flow
management:
  endpoints:
    web:
      exposure:
        include: '*'
```

nacos控制台 新建配置：

Data ID:cloudalibaba-sentinel-service

配置格式:json

配置内容：

```json
[
​    {
​        "resource":"/byUrl",           //资源名称        
​        "limitApp":"default",            //来源应用
​        "grade":1,                        //阈值类型，0表示线程数，1表示QPS
​        "count":1,                        //单击阈值
​        "strategy":0,                    //流控模式，0表示直接，1表示关联，2表示链路
​        "controlBehavior":0,            //流控效果，0表示快速失败，1表示Warm Up,2表示排队等待
​        "clusterMode":false            //是否集群
​    }
]
```

启动8401，刷新sentinel流控规则（因为sentinel懒加载，所以需要先访问http://localhost:8401/byUrl）发现规则有了，在多次访问http://localhost:8401/byUrl后能展现nacos配置内容所表示的规则。

重点：故意重启8401，刷新sentinel流控规则,发现规则还存在，并能正确生效。

# Seata

Seata是一款开源的分布式事务解决方案，致力于在微服务架构下提供高性能和简单易用的分布式事务服务。

官网seata.io

典型的分布式事务过程（一ID+三组件模型）：

Transaction ID （XID）:全局唯一的事务ID；

3组件（seata常用术语）：

TC (Transaction Coordinator) - 事务协调者

维护全局和分支事务的状态，驱动全局事务提交或回滚。

TM (Transaction Manager) - 事务管理器

定义全局事务的范围：开始全局事务、提交或回滚全局事务。

RM (Resource Manager) - 资源管理器

管理分支事务处理的资源，与TC交谈以注册分支事务和报告分支事务的状态，并驱动分支事务提交或回滚。

处理过程：

https://p1-tt.byteimg.com/origin/pgc-image/86e1e7dfbc2b401792c2560bfd6ec7d1?from=pc

- TM 请求 TC，开始一个新的全局事务，TC 会为这个全局事务生成一个 XID。
- XID 通过微服务的调用链传递到其他微服务。
- RM 把本地事务作为这个XID的分支事务注册到TC。
- TM 请求 TC 对这个 XID 进行提交或回滚。
- TC 指挥这个 XID 下面的所有分支事务进行提交、回滚。

配置难，使用简单。spring有控制本地事务的注解@Transactional，seata只需要@GlobalTransactional注解控制全局事务。

**分布式事务**

微服务应用每个服务内部的数据一致性由本地事务来保证，但是全局的数据一致性问题没法保证

一次业务操作需要跨多个数据源或需要跨多个系统进行远程调用，就会产生分布式事务问题。

**下载安装**

http://seata.io/zh-cn/blog/download.html

下载seata-server-1.0.0.zip并解压；

修改conf目录下的file.conf配置文件（可以拷贝一份），修改自定义事务组名称+事务日志存储模式为db+数据库连接信息：

1.

service修改，将"default修改"，任意命名，后缀约定"_tx_group"

```conf
service {
  vgroup_mapping.my_test_tx_group = "fsp_tx_group"  #测试用的事务分组
  default.grouplist = "127.0.0.1:8091"
  enableDegrade = false
  disable = false
  max.commit.retry.timeout = "-1"
  max.rollback.retry.timeout = "-1"
}
```

2.store，默认存在文件里

```
store {
  mode = "db"           #修改存储模式
  file {
    dir = "sessionStore"
    max-branch-session-size = 16384
    max-global-session-size = 512
    file-write-buffer-cache-size = 16384
    session.reload.read_size = 100
    flush-disk-mode = async
  }
  db {
    datasource = "dbcp"
    db-type = "mysql"
    driver-class-name = "com.mysql.jdbc.Driver"
    url = "jdbc:mysql://127.0.0.1:3306/seata"      #
    user = "root"                                   #
    password = "350562"                            #
    min-conn = 1
    max-conn = 3
    global.table = "global_table"
    branch.table = "branch_table"
    lock-table = "lock_table"
    query-limit = 100
  }
}
```

mysql新建数据库seata，sql文件在conf下的db_store.sql。

修改conf下的registry.conf配置文件（备份）,指明注册中心为nacos，并修改连接信息。

registry type = "nacos"

nacos serverAddr = "localhost:8848"

完成！！！

启动nacos8848 再启动seata-server(bin下seata-server.bat)。

显示load RegistryProvider等表示成功.

注意：如果出现闪退，检查配置文件,jdk8环境

## 分布式交易解决方案

业务说明:

这里我们会创建三个服务,一个订单服务,一个库存服务,一个账户服务.

当用户下单时,会在订单服务中创建一个订单,然后通过远程调用库存服务来扣减下单商品的库存,再通过远程调用账户服务来扣减用户账户里面的余额,最后在订单服务中修改订单状态为已完成.

该操作跨越三个数据库,有两次远程调用,很明显会出现分布式事务的问题.

http://seata.io/img/architecture.png

### 数据库

业务表

1.seata_order

```sql
CREATE TABLE `t_order` (
  `id` bigint(11) NOT NULL AUTO_INCREMENT,
  `user_id` bigint(11) DEFAULT NULL COMMENT '用户id',       
  `product_id` bigint(11) DEFAULT NULL COMMENT '产品id',    
  `count` int(11) DEFAULT NULL COMMENT '数量',
  `money` decimal(11,0) DEFAULT NULL COMMENT '金额',        
  `status` int(1) DEFAULT NULL COMMENT '订单状态:0创建中;1已完结',
  PRIMARY KEY (`id`)
) ENGINE=InnoDB AUTO_INCREMENT=56 DEFAULT CHARSET=utf8
```

2.seata_storage

```sql
CREATE TABLE `t_storage` (
  `id` bigint(11) NOT NULL AUTO_INCREMENT,
  `product_id` bigint(11) DEFAULT NULL COMMENT '产品id',    
  `total` int(11) DEFAULT NULL COMMENT '总库存',
  `used` int(11) DEFAULT NULL COMMENT '已用库存',
  `residue` int(11) DEFAULT NULL COMMENT '剩余库存',
  PRIMARY KEY (`id`)
) ENGINE=InnoDB AUTO_INCREMENT=2 DEFAULT CHARSET=utf8 
```

3.seata_account

```sql
CREATE TABLE `t_account` (
  `id` bigint(11) NOT NULL AUTO_INCREMENT,
  `user_id` bigint(11) DEFAULT NULL COMMENT '用户id',       
  `total` decimal(10,0) DEFAULT NULL COMMENT '总额度',      
  `used` decimal(10,0) DEFAULT NULL COMMENT '已用余额',
  `residue` decimal(10,0) DEFAULT NULL COMMENT '剩余可用额度
',
  PRIMARY KEY (`id`)
) ENGINE=InnoDB AUTO_INCREMENT=2 DEFAULT CHARSET=utf8
```

回滚日志表

3个库下都需要建立各自的回滚日志表

\seata_server\0.9.0\seata\conf下的db_undo_log.sql.

在每个库都执行一次该sql脚本,每个库下的表名为undo_log

### Order

新建子模块seata-order-service2001

```
  <dependency>
            <groupId>com.alibaba.cloud</groupId>
            <artifactId>spring-cloud-starter-alibaba-seata</artifactId>
            <exclusions>
                <exclusion>
                    <groupId>seata-all</groupId>
                    <artifactId>io.seata</artifactId>
                </exclusion>
            </exclusions>
        </dependency>
        <dependency>
            <groupId>io.seata</groupId>
            <artifactId>seata-all</artifactId>
            <version>0.9.0</version>
        </dependency>
        <dependency>
            <groupId>com.alibaba.cloud</groupId>
            <artifactId>spring-cloud-starter-alibaba-nacos-discovery</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-openfeign</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-actuator</artifactId>
        </dependency>
        <dependency>
            <groupId>org.mybatis.spring.boot</groupId>
            <artifactId>mybatis-spring-boot-starter</artifactId>
        </dependency>
        <dependency>
            <groupId>com.alibaba</groupId>
            <artifactId>druid-spring-boot-starter</artifactId>
            <version>1.1.10</version>
        </dependency>
        <dependency>
            <groupId>mysql</groupId>
            <artifactId>mysql-connector-java</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-jdbc</artifactId>
        </dependency>
        <dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-devtools</artifactId>
        </dependency>
        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
        </dependency>
```

```
server:
  port: 2001
spring:
  application:
   name: seata-order-service
  cloud:
    nacos:
      discovery:
        server-addr: 127.0.0.1:8848  #配置nacos地址
    alibaba:
      seata:
        #自定义事务组名称,需要与自己修改的file.conf中的vgroup_mapping.my_test_tx_group 一样
        tx-service-group: fsp_tx_group
  datasource:
    driver-class-name: com.mysql.jdbc.Driver
    url: jdbc:mysql://localhost:3306/seata_order?useUnicode=true&characterEncoding=utf8&serverTimezone=Asia/Shanghai&useSSL=false
    username: root
    password: 350562
management:
  endpoints:
    web:
      exposure:
        include: '*'
feign:
  hystrix:
    enabled: false
logging:
  level: 
    io:
      seata: info
mybatis:
  mapperLocations: classpath:mapper/*.xml
```

file.conf

```conf
transport {
  # tcp udt unix-domain-socket
  type = "TCP"
  #NIO NATIVE
  server = "NIO"
  #enable heartbeat
  heartbeat = true
  #thread factory for netty
  thread-factory {
    boss-thread-prefix = "NettyBoss"
    worker-thread-prefix = "NettyServerNIOWorker"
    server-executor-thread-prefix = "NettyServerBizHandler"
    share-boss-worker = false
    client-selector-thread-prefix = "NettyClientSelector"
    client-selector-thread-size = 1
    client-worker-thread-prefix = "NettyClientWorkerThread"
    # netty boss thread size,will not be used for UDT
    boss-thread-size = 1
    #auto default pin or 8
    worker-thread-size = 8
  }
  shutdown {
    # when destroy server, wait seconds
    wait = 3
  }
  serialization = "seata"
  compressor = "none"
}

service {

  vgroup_mapping.fsp_tx_group = "default" #修改自定义事务组名称

  default.grouplist = "127.0.0.1:8091"
  enableDegrade = false
  disable = false
  max.commit.retry.timeout = "-1"
  max.rollback.retry.timeout = "-1"
  disableGlobalTransaction = false
}


client {
  async.commit.buffer.limit = 10000
  lock {
    retry.internal = 10
    retry.times = 30
  }
  report.retry.count = 5
  tm.commit.retry.count = 1
  tm.rollback.retry.count = 1
}

## transaction log store
store {
  ## store mode: file、db
  mode = "db"

  ## file store
  file {
    dir = "sessionStore"

    # branch session size , if exceeded first try compress lockkey, still exceeded throws exceptions
    max-branch-session-size = 16384
    # globe session size , if exceeded throws exceptions
    max-global-session-size = 512
    # file buffer size , if exceeded allocate new buffer
    file-write-buffer-cache-size = 16384
    # when recover batch read size
    session.reload.read_size = 100
    # async, sync
    flush-disk-mode = async
  }

  ## database store
  db {
    ## the implement of javax.sql.DataSource, such as DruidDataSource(druid)/BasicDataSource(dbcp) etc.
    datasource = "dbcp"
    ## mysql/oracle/h2/oceanbase etc.
    db-type = "mysql"
    driver-class-name = "com.mysql.jdbc.Driver"
    url = "jdbc:mysql://127.0.0.1:3306/seata"
    user = "root"
    password = "123456"
    min-conn = 1
    max-conn = 3
    global.table = "global_table"
    branch.table = "branch_table"
    lock-table = "lock_table"
    query-limit = 100
  }
}
lock {
  ## the lock store mode: local、remote
  mode = "remote"

  local {
    ## store locks in user's database
  }

  remote {
    ## store locks in the seata's server
  }
}
recovery {
  #schedule committing retry period in milliseconds
  committing-retry-period = 1000
  #schedule asyn committing retry period in milliseconds
  asyn-committing-retry-period = 1000
  #schedule rollbacking retry period in milliseconds
  rollbacking-retry-period = 1000
  #schedule timeout retry period in milliseconds
  timeout-retry-period = 1000
}

transaction {
  undo.data.validation = true
  undo.log.serialization = "jackson"
  undo.log.save.days = 7
  #schedule delete expired undo_log in milliseconds
  undo.log.delete.period = 86400000
  undo.log.table = "undo_log"
}

## metrics settings
metrics {
  enabled = false
  registry-type = "compact"
  # multi exporters use comma divided
  exporter-list = "prometheus"
  exporter-prometheus-port = 9898
}

support {
  ## spring
  spring {
    # auto proxy the DataSource bean
    datasource.autoproxy = false
  }
}
```

registry.conf

```
registry {
  # file 、nacos 、eureka、redis、zk、consul、etcd3、sofa
  type = "nacos"

  nacos {
    serverAddr = "localhost:8848"
    namespace = ""
    cluster = "default"
  }
  eureka {
    serviceUrl = "http://localhost:8761/eureka"
    application = "default"
    weight = "1"
  }
  redis {
    serverAddr = "localhost:6379"
    db = "0"
  }
  zk {
    cluster = "default"
    serverAddr = "127.0.0.1:2181"
    session.timeout = 6000
    connect.timeout = 2000
  }
  consul {
    cluster = "default"
    serverAddr = "127.0.0.1:8500"
  }
  etcd3 {
    cluster = "default"
    serverAddr = "http://localhost:2379"
  }
  sofa {
    serverAddr = "127.0.0.1:9603"
    application = "default"
    region = "DEFAULT_ZONE"
    datacenter = "DefaultDataCenter"
    cluster = "default"
    group = "SEATA_GROUP"
    addressWaitTime = "3000"
  }
  file {
    name = "file.conf"
  }
}

config {
  # file、nacos 、apollo、zk、consul、etcd3
  type = "file"

  nacos {
    serverAddr = "localhost"
    namespace = ""
  }
  consul {
    serverAddr = "127.0.0.1:8500"
  }
  apollo {
    app.id = "seata-server"
    apollo.meta = "http://192.168.1.204:8801"
  }
  zk {
    serverAddr = "127.0.0.1:2181"
    session.timeout = 6000
    connect.timeout = 2000
  }
  etcd3 {
    serverAddr = "http://localhost:2379"
  }
  file {
    name = "file.conf"
  }
}
```

domain(entity)下

```
@Data
@AllArgsConstructor
@NoArgsConstructor
public class Order {
   private Long id;
   private Long userId;
   private Long productId;
   private Integer count;
   private BigDecimal money;
   private Integer status; //订单状态：0：创建中；1：已完结
}
```

```
@Data
@AllArgsConstructor
@NoArgsConstructor
public class CommonResult<T>{
   private Integer code;
   private String message;
   private T      data;
    public CommonResult(Integer code, String message){
      this(code,message,null);
    }
}
```

mapper文件夹下OrderMapper.xml

```
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE mapper PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN" "http://mybatis.org/dtd/mybatis-3-mapper.dtd" >
<mapper namespace="edu.hunnu.cloud.dao.OrderDao">
    <resultMap id="BaseResultMap" type="edu.hunnu.cloud.domain.Order">
        <id column="id" property="id" jdbcType="BIGINT"/>
        <result column="user_id" property="userId" jdbcType="BIGINT"/>
        <result column="product_id" property="productId" jdbcType="BIGINT"/>
        <result column="count" property="count" jdbcType="INTEGER"/>
        <result column="money" property="money" jdbcType="DECIMAL"/>
        <result column="status" property="status" jdbcType="INTEGER"/>
    </resultMap>
    <insert id="create">
        insert into t_order (id,user_id,product_id,count,money,status)
        values (null,#{userId},#{productId},#{count},#{money},0);
    </insert>
    <update id="update">
        update t_order set status = 1
        where user_id=#{userId} and status = #{status};
    </update>
</mapper>
```

service下接口

```
public interface OrderService
{
   void create(Order order);
}
```

```
@FeignClient(value = "seata-storage-service")
public interface StorageService
{
   @PostMapping(value = "/storage/decrease")
   CommonResult decrease(@RequestParam("productId") Long productId, @RequestParam("count") Integer count);
}
```

```
@FeignClient(value = "seata-account-service")
public interface AccountService
{
   @PostMapping(value = "/account/decrease")
   CommonResult decrease(@RequestParam("userId") Long userId, @RequestParam("money") BigDecimal money);
}
```

service.impl下实现OrderService

```
@Service
@Slf4j
public class OrderServiceImpl implements OrderService {
   @Resource
   private OrderDao orderDao;
   @Resource
   private StorageService storageService;
   @Resource
   private AccountService accountService;
   /**
    * 创建订单->调用库存服务扣减库存->调用账户服务扣减账户余额->修改订单状态
    * 简单说：下订单->扣库存->减余额->改状态
    */
   @Override
   //@GlobalTransactional(name = "fsp-create-order",rollbackFor = Exception.class)
   public void create(Order order) {
      log.info("----->开始新建订单");
      //1 新建订单
      orderDao.create(order);

      //2 扣减库存
      log.info("----->订单微服务开始调用库存，做扣减Count");
      storageService.decrease(order.getProductId(),order.getCount());
      log.info("----->订单微服务开始调用库存，做扣减end");

      //3 扣减账户
      log.info("----->订单微服务开始调用账户，做扣减Money");
      accountService.decrease(order.getUserId(),order.getMoney());
      log.info("----->订单微服务开始调用账户，做扣减end");

      //4 修改订单状态，从零到1,1代表已经完成
      log.info("----->修改订单状态开始");
      orderDao.update(order.getUserId(),0);
      log.info("----->修改订单状态结束");

      log.info("----->下订单结束了，O(∩_∩)O哈哈~");
   }
}
```

controller

```
@RestController
public class OrderController {
   @Resource
   private OrderService orderService;
   @GetMapping("/order/create")
   public CommonResult create(Order order) {
      orderService.create(order);
      return new CommonResult(200,"订单创建成功");
   }
}
```

config包下

```
@Configuration
@MapperScan({"edu.hunnu.cloud.dap"})
public class MybatisConfig {
}
```

```
import com.alibaba.druid.pool.DruidDataSource;
import io.seata.rm.datasource.DataSourceProxy;
import org.apache.ibatis.session.SqlSessionFactory;
import org.mybatis.spring.SqlSessionFactoryBean;
import org.mybatis.spring.transaction.SpringManagedTransactionFactory;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.boot.context.properties.ConfigurationProperties;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.core.io.support.PathMatchingResourcePatternResolver;
import javax.sql.DataSource;

/**
 * 使用Seata对数据源进行代理
 */
@Configuration
public class DataSourceProxyConfig {
    @Value("${mybatis.mapperLocations}")
    private String mapperLocations;
    @Bean
    @ConfigurationProperties(prefix = "spring.datasource")
    public DataSource druidDataSource(){
        return new DruidDataSource();
    }
    @Bean
    public DataSourceProxy dataSourceProxy(DataSource dataSource) {
        return new DataSourceProxy(dataSource);
    }
    @Bean
    public SqlSessionFactory sqlSessionFactoryBean(DataSourceProxy dataSourceProxy) throws Exception {
        SqlSessionFactoryBean sqlSessionFactoryBean = new SqlSessionFactoryBean();
        sqlSessionFactoryBean.setDataSource(dataSourceProxy);
        sqlSessionFactoryBean.setMapperLocations(new PathMatchingResourcePatternResolver().getResources(mapperLocations));
        sqlSessionFactoryBean.setTransactionFactory(new SpringManagedTransactionFactory());
        return sqlSessionFactoryBean.getObject();
    }
}
```

启动类

```
@EnableDiscoveryClient
@EnableFeignClients
//取消数据源的自动创建，而是使用自己定义的
@SpringBootApplication(exclude = DataSourceAutoConfiguration.class)
public class SeataOrderMainApp2001 {
   public static void main(String[] args) {
      SpringApplication.run(SeataOrderMainApp2001.class, args);
   }
}
```

测试:启动nacos    seata    2001   控制台无Error

### Storage

新建子模块seata-storage-service2002

pom与2001一样

yml

```
server:
  port: 2002
spring:
  application:
    name: seata-storage-service
  cloud:
    alibaba:
      seata:
        tx-service-group: fsp_tx_group
    nacos:
      discovery:
        server-addr: localhost:8848
  datasource:
    driver-class-name: com.mysql.jdbc.Driver
    url: jdbc:mysql://localhost:3306/seata_storage?useUnicode=true&characterEncoding=utf8&serverTimezone=Asia/Shanghai&useSSL=false
    username: root
    password: 350562
logging:
  level:
    io:
      seata: info
mybatis:
  mapperLocations: classpath:mapper/*.xml
```

file.conf与registry.conf与2001一样.

domain中的CommonResult与2001一样

新建Storage

```
@Data
@NoArgsConstructor
@AllArgsConstructor
public class Storage {
    private Long id;
    //产品id
    private Long productId;
    //总库存
    private Integer total;
    //已用库存
    private Integer used;
    //剩余库存
    private Integer residue;
}
```

dao

```
@Mapper
public interface StorageDao {
   //扣减库存
   void decrease(@Param("productId") Long productId, @Param("count") Integer count);
}
```

mapper/StorageMapper.xml

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE mapper PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN" "http://mybatis.org/dtd/mybatis-3-mapper.dtd" >

<mapper namespace="edu.hunnu.cloud.dao.StorageDao">
    <resultMap id="BaseResultMap" type="edu.hunnu.cloud.domain.Storage">
        <id column="id" property="id" jdbcType="BIGINT"/>
        <result column="product_id" property="productId" jdbcType="BIGINT"/>
        <result column="total" property="total" jdbcType="INTEGER"/>
        <result column="used" property="used" jdbcType="INTEGER"/>
        <result column="residue" property="residue" jdbcType="INTEGER"/>
    </resultMap>
    <update id="decrease">
        UPDATE
            t_storage
        SET
            used = used + #{count},residue = residue - #{count}
        WHERE
            product_id = #{productId}
    </update>
</mapper>
```

service:只需要storage的相关

```
public interface StorageService {
   /**
    * 扣减库存
    */
   void decrease(Long productId, Integer count);
}
```

```
import edu.hunnu.cloud.dao.StorageDao;
import edu.hunnu.cloud.service.StorageService;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.stereotype.Service;
import javax.annotation.Resource;

/**
 * @author hunnu/lzf
 * @Date 2021/5/29
 */
@Service
public class StorageServiceImpl implements StorageService {
   private static final Logger LOGGER = LoggerFactory.getLogger(StorageServiceImpl.class);
   @Resource
   private StorageDao storageDao;
   /**
    * 扣减库存
    */
   @Override
   public void decrease(Long productId, Integer count) {
      LOGGER.info("------->storage-service中扣减库存开始");
      storageDao.decrease(productId,count);
      LOGGER.info("------->storage-service中扣减库存结束");
   }
}
```

controller

```
@RestController
public class StorageController {
   @Autowired
   private StorageService storageService;
   /**
    * 扣减库存
    */
   @RequestMapping("/storage/decrease")
   public CommonResult decrease(Long productId, Integer count) {
      storageService.decrease(productId, count);
      return new CommonResult(200,"扣减库存成功！");
   }
}
```

config与2001一样

启动类

```
@EnableDiscoveryClient
@EnableFeignClients
@SpringBootApplication(exclude = DataSourceAutoConfiguration.class)
public class SeataStorageServiceApp2002 {
   public static void main(String[] args) {
      SpringApplication.run(SeataStorageServiceApp2002.class, args);
   }
}
```

测试:启动nacos  seata 2002 .

### Account

新建子模块seata-account-service2003

pom与2001一样

yml

```
server:
  port: 2003
spring:
  application:
    name: seata-account-service
  cloud:
    alibaba:
      seata:
        tx-service-group: fsp_tx_group
    nacos:
      discovery:
        server-addr: localhost:8848
  datasource:
    driver-class-name: com.mysql.jdbc.Driver
    url: jdbc:mysql://localhost:3306/seata_account?useUnicode=true&characterEncoding=utf8&serverTimezone=Asia/Shanghai&useSSL=false
    username: root
    password: 350562
feign:
  hystrix:
    enabled: false
logging:
  level:
    io:
      seata: info
mybatis:
  mapperLocations: classpath:mapper/*.xml
```

file.conf与registry.conf与2001一样.

domain中的CommonResult与2001一样

新建Account

```
@Data
@AllArgsConstructor
@NoArgsConstructor
public class Account {
   private Long id;
   //用户id
   private Long userId;
   //总额度
   private BigDecimal total;
   //已用额度
   private BigDecimal used;
   //剩余额度
   private BigDecimal residue;
}
```

dao

```
@Mapper
public interface AccountDao {
   /**
    * 扣减账户余额
    */
   void decrease(@Param("userId") Long userId, @Param("money") BigDecimal money);
}
```

mapper/AcoountMapper.xml

```
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE mapper PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN" "http://mybatis.org/dtd/mybatis-3-mapper.dtd" >

<mapper namespace="edu.hunnu.cloud.dao.AccountDao">

    <resultMap id="BaseResultMap" type="edu.hunnu.cloud.domain.Account">
        <id column="id" property="id" jdbcType="BIGINT"/>
        <result column="user_id" property="userId" jdbcType="BIGINT"/>
        <result column="total" property="total" jdbcType="DECIMAL"/>
        <result column="used" property="used" jdbcType="DECIMAL"/>
        <result column="residue" property="residue" jdbcType="DECIMAL"/>
    </resultMap>
    <update id="decrease">
        UPDATE t_account
        SET
            residue = residue - #{money},used = used + #{money}
        WHERE
            user_id = #{userId};
    </update>
</mapper>
```

service接口

```
public interface AccountService {
   /**
    * 扣减账户余额
    * @param userId 用户id
    * @param money 金额
    */
   void decrease(@RequestParam("userId") Long userId, @RequestParam("money") BigDecimal money);
}
```

impl

```
@Service
public class AccountServiceImpl implements AccountService {
   private static final Logger LOGGER = LoggerFactory.getLogger(AccountServiceImpl.class);
   @Resource
   AccountDao accountDao;
   /**
    * 扣减账户余额
    */
   @Override
   public void decrease(Long userId, BigDecimal money) {
      LOGGER.info("------->account-service中扣减账户余额开始");
              //模拟超时异常,全局事务回滚
      accountDao.decrease(userId,money);
      LOGGER.info("------->account-service中扣减账户余额结束");
   }
}
```

controller

```
@RestController
public class AccountController {
   @Resource
   AccountService accountService;
   /**
    * 扣减账户余额
    */
   @RequestMapping("/account/decrease")
   public CommonResult decrease(@RequestParam("userId") Long userId, @RequestParam("money") BigDecimal money){
      accountService.decrease(userId,money);
      return new CommonResult(200,"扣减账户余额成功！");
   }
}
```

config与2001一样

启动类

```
@EnableDiscoveryClient
@EnableFeignClients
//取消数据源的自动创建，而是使用自己定义的
@SpringBootApplication(exclude = DataSourceAutoConfiguration.class)
public class SeataAccountMainApp2003 {
   public static void main(String[] args) {
      SpringApplication.run(SeataAccountMainApp2003.class, args);
   }
}
```

测试 启动nacos seata 2003

### Test

1.没有添加@GlobalTransactional的情况

下订单-减库存-扣余额-改状态

先查看数据库表初始状态

访问

http://localhost:2001/order/create?userId=1&productId=1&count=10&money=100

查看返回信息,与数据库表变化.

故意设置AccountServiceImpl超时

```
@Service
public class AccountServiceImpl implements AccountService {
   private static final Logger LOGGER = LoggerFactory.getLogger(AccountServiceImpl.class);
   @Resource
   AccountDao accountDao;
   /**
    * 扣减账户余额
    */
   @Override
   public void decrease(Long userId, BigDecimal money) {
      LOGGER.info("------->account-service中扣减账户余额开始");
      //模拟超时异常,全局事务回滚
      try {
         TimeUnit.SECONDS.sleep(20);
      } catch (InterruptedException e) {
         e.printStackTrace();
      }
      accountDao.decrease(userId,money);
      LOGGER.info("------->account-service中扣减账户余额结束");
   }
}
```

因为Feign的默认调用超时时间限定为1秒,所以这里会出现read timeout 异常(这里也可以用算术异常)

访问

http://localhost:2001/order/create?userId=1&productId=1&count=10&money=100

Read timed out executing POST http://seata-account-service/account/decrease?userId=1&money=100

检查数据表,发现order表产生新订单,并且storage表库存已经减少,account表余额也已经减少,但是order表产生的新订单status还是0.

解决方法:添加注解.

2.添加@GlobalTransactional的情况

<u>在OrderServiceImpl的create方法上添加注解</u>

```
@GlobalTransactional(name = "fsp-create-order",rollbackFor = Exception.class)
```

rollbackFor表示回滚的异常.

访问

http://localhost:2001/order/create?userId=1&productId=1&count=10&money=100

发现没有任何一条新记录插进来.

完成分布式事务控制!!!

## 附加

生产中使用1.0以后的GA版本,0.9不支持集群.

再谈TC/TM/RM

https://p1-tt.byteimg.com/origin/pgc-image/86e1e7dfbc2b401792c2560bfd6ec7d1?from=pc

TC:seata server,也就是启动的seata-server.bat,负责全局操控.

TM:被@GlobalTransactional注解所标注处,事务的发起方.

RM:与@GlobalTransactional注解所在方法相关联的DB操作分支,事务的参与方.

seata有四种模式:AT(当前默认使用),TCC, SAGA, XA.

AT模式整体机制:

两阶段提交协议的演变：

- 一阶段：业务数据和回滚日志记录在同一个本地事务中提交，释放本地锁和连接资源。
- 二阶段：
  - 提交异步化，非常快速地完成。
  - 回滚通过一阶段的回滚日志进行反向补偿。

可打断点查看seata库的表和业务库的undo_log表.

前置镜像 后置镜像 反向补偿(逆向SQL)