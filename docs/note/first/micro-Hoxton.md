# 版本

为了使用springboot2.0以及以上版本，cloud使用G版和H版

springboot版本  2.2.2.RELEASE

springcloud版本  Hoxton.SR1

```java
<!--      springboot 2.2.2-->
      <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-dependencies</artifactId>
        <version>2.2.2.RELEASE</version>
        <type>pom</type>
        <scope>import</scope>
      </dependency>
<!--      springcloud hoxton.SR1-->
      <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-dependencies</artifactId>
        <version>Hoxton.SR1</version>
        <type>pom</type>
        <scope>import</scope>
      </dependency>
<!--      spring cloud alibaba 2.1.0.RELEASE-->
    
```



# 准备工作

聚合父工程

new Project --> maven --> jdk1.8 Create from archetype -->maven-archetype-site->name:cloud2020 groud:edu.hunnu.cloud

约定>配置>编码

editor -->file encoding -->uft8   Transparent native-to-ascii conversion

build --> compiler --> Annotation Processors --> Enable annotataion processing  支持注解

editor --> file type --> ignored files and folders -->  添加 *.idea和 *.iml  忽略idea自动生成的文件  按喜好设置。

pom packaging;

删除src；

pom：

添加属性

```
<maven.compiler.source>1.8</maven.compiler.source>
<maven.compiler.targer>1.8</maven.compiler.targer>
<junit.version>4.12</junit.version>
<log4j.version>1.2.17</log4j.version>
<lombok.version>1.16.18</lombok.version>
<mysql.version>5.1.47</mysql.version>
<druid.version>1.1.16</druid.version>
<mybatis.spring.boot.version>1.3.0</mybatis.spring.boot.version>
```

依赖和插件

```
  <dependencyManagement>
    <dependencies>
<!--      springboot 2.2.2-->
      <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-dependencies</artifactId>
        <version>2.2.2.RELEASE</version>
        <type>pom</type>
        <scope>import</scope>
      </dependency>
<!--      springcloud hoxton.SR1-->
      <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-dependencies</artifactId>
        <version>Hoxton.SR1</version>
        <type>pom</type>
        <scope>import</scope>
      </dependency>
    </dependencies>
  </dependencyManagement>
```

父工程搭建完成后mvn:install发布到仓库方便子工程继承。

**Provider**

新建子工程cloud-provider-payment8001

web与actuator通常绑定一块

```
<dependencies>
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
server:
  port: 8001

mybatis:
  type-aliases-package: edu.hunnu.cloud.entities
  mapper-locations: classpath:mapper/*.xml

spring:
  application:
    name: cloud-payment-service
  datasource:
    type: com.alibaba.druid.pool.DruidDataSource
    driver-class-name: com.mysql.jdbc.Driver
    url: jdbc:mysql://localhost:3306/db2019?useUnicode=true&characterEncoding=utf8&serverTimezone=Asia/Shanghai&useSSL=false
    username: root
    password: 350562
```

```
@SpringBootApplication
public class PaymentMain8001 {
   public static void main(String[] args) {
      SpringApplication.run(PaymentMain8001.class,args);
   }
}
```

```
@Data
@AllArgsConstructor
@NoArgsConstructor
public class Payment implements Serializable {
   private Long id;  //bigint 自增 主键
   private String serial; //varchar 默认''
}
```

```
@Data
@AllArgsConstructor
@NoArgsConstructor
public class CommonResult<T> implements Serializable {
   private Integer code;
   private String  message;
   private T       data;

   public CommonResult(Integer code,String message){
      this(code,message,null);
   }
}
```

```
@Mapper
public interface PaymentDao {
   public int add(Payment payment);
   public Payment getPaymentById(@Param("id") long id);

}
```

```
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE mapper PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN" "http://mybatis.org/dtd/mybatis-3-mapper.dtd">
<mapper namespace="edu.hunnu.cloud.dao.PaymentDao">
    <insert id="create" parameterType="Payment" useGeneratedKeys="true" keyProperty="id">
        insert into payment(serial) values(#{serial});
    </insert>

    <resultMap id="BaseResultMap" type="edu.hunnu.cloud.entities.Payment">
        <id column="id" property="id" jdbcType="BIGINT"/>
        <result column="serial" property="serial" jdbcType="VARCHAR"/>
    </resultMap>

    <select id="getPaymentById" parameterType="long" resultMap="BaseResultMap">
        select * from payment where id=#{id};
    </select>
</mapper>
```

注意 因为yml文件中用了实体类包扫描，所以此处直接使用Payment, 但是idea会报红，不影响使用。

service层省略

controller：

```
@RestController
@RequestMapping(value = "payment")
@Slf4j
public class PaymentController {
   @Resource
   private PaymentService paymentService;

   @PostMapping(value = "create")
   public CommonResult create(@RequestBody Payment payment){
      int result = paymentService.create(payment);
      log.info("*****插入结果："+result);
      if (result>0){
         return new CommonResult(200,"插入数据库成功",result);
      }else {
         return new CommonResult(444,"插入数据库失败",null);
      }
   }
   @GetMapping(value = "get/{id}")
   public CommonResult getPaymentById(@PathVariable("id")Long id){
      Payment result = paymentService.getPaymentById(id);
      log.info("*****查询结果："+result);
      if (result!=null){
         return new CommonResult(200,"查询成功",result);
      }else {
         return new CommonResult(444,"查询失败",null);
      }
   }
}
```

**consumer**

创建子模块cloud-consumer-order80

```
<dependencies>
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
server:
  port: 80
```

```
@SpringBootApplication
public class OrderMain80 {
   public static void main(String[] args) {
      SpringApplication.run(OrderMain80.class,args);
   }
}
```

```
@Configuration
public class ApplicationContextConfig {
   @Bean
   public RestTemplate getRestTemplate(){
      return new RestTemplate();
   }
}
```

```
@RestController
@Slf4j
public class OrderController {
   public static final String PAYMENT_URL="http://localhost:8001";

   @Resource
   private RestTemplate restTemplate ;

   @GetMapping("consumer/payment/create")
   public CommonResult<Payment> create(Payment payment){
      return restTemplate.postForObject(PAYMENT_URL+"/payment/create",payment,CommonResult.class);
   }
   @GetMapping("consumer/payment/get/{id}")
   public CommonResult<Payment> getPayment(@PathVariable("id") Long id){
      return restTemplate.getForObject(PAYMENT_URL+"/payment/get/"+id,CommonResult.class)
   }
}
```

# 服务注册中心

## Eureka

**Eureka Server**

新建子模块cloud-eureka-server7001

说明：老版本 1.X

```
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starte-eureka-server</artifactId>
</dependency>
```

**新版本 2.X**

```
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-eureka-server</artifactId>
</dependency>
```

```
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
  port: 7001

eureka:
  instance:
    hostname: localhost 
  client:
    register-with-eureka: false      #不注册自己
    fetch-registry: false          #表示自己端就是注册中心
    service-url: 
      #与 eureka server交互（查询服务和注册服务）地址
      defaultZone: http://${eureka.instance.hostname}:${server.port}/eureka/
```

```
@SpringBootApplication
@EnableEurekaServer
public class EurekaMain7001 {
   public static void main(String[] args) {
      SpringApplication.run(EurekaMain7001.class,args);
   }
}
```

http://localhost:7001/ 

**Provider**

将cloud-provider-payment8001注册进7001：

说明：老版本 1.X

```
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starte-eureka</artifactId>
</dependency>
```

**新版本 2.X**

```
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
</dependency>
```

```
server:
  port: 8001

mybatis:
  type-aliases-package: edu.hunnu.cloud.entities
  mapper-locations: classpath:mapper/*.xml

spring:
  application:
    name: cloud-payment-service  #微服务名称
  datasource:
    type: com.alibaba.druid.pool.DruidDataSource
    driver-class-name: com.mysql.jdbc.Driver
    url: jdbc:mysql://localhost:3306/db2019?useUnicode=true&characterEncoding=utf8&serverTimezone=Asia/Shanghai&useSSL=false
    username: root
    password: 350562
eureka:
  client:
    register-with-eureka: true
    #是否从EurekaServer抓取已有的注册信息，默认为true。单节点无所谓，集群必须设置为ture才能配合ribbon使用高负载均衡
    fetch-registry: true
    service-url:
      defaultZone: http://localhost:7001/eureka
```

主启动类添加  @EnableEurekaClient

**Consumer**

将cloud-consumer-order80注册进Eureka

```
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
</dependency>
```

```
server:
  port: 80

spring:
  application:
    name: cloud-order-service

eureka:
  client:
    register-with-eureka: true
    fetch-registry: true
    service-url:
      defaultZone: http://localhost:7001/eureka
```

也可以不将consumer注册进eureka server

### 高可用

#### Service

**互相注册，相互守望**

参考eureka-server7001创建7002

主类名修改

修改hosts：

127.0.0.1 eureka7001.com
127.0.0.1 eureka7002.com

```
server.port:7001    ~7002
eureka.instance.hostname:eureka7001.com    ~eureka7002.com
eureka.client.service-url.defaultZnoe:http://eureka7002.com:7002/eureka/                                                             ~http://eureka7001.com:7001/eureka/  相互注册
```

测试：启动7001 7002

http://eureka7001.com:7001 http://eureka7002.com:7002 相互DS Replicas

服务中心集群搭建完后，微服务注册地址：

修改defaultZone: http://eureka7001.com:7001/eureka/,http://eureka7002.com:7002/eureka/

#### provider

参考provider8001新建8002

主类名修改

server.port: 8001 - 8002

为了区分调用的是哪一个微服务服务器，对controller进行简单修改：

```
@RestController
@RequestMapping(value = "payment")
@Slf4j
public class PaymentController {
   @Resource
   private PaymentService paymentService;

   @Value("${server.port}")
   private String serverPort;

   @PostMapping(value = "create")
   public CommonResult create(@RequestBody Payment payment){
      int result = paymentService.create(payment);
      log.info("*****插入结果："+result);
      if (result>0){
         return new CommonResult(200,"插入数据库成功,server.port:"+serverPort,result);
      }else {
         return new CommonResult(444,"插入数据库失败,server.port:"+serverPort,null);
      }
   }

   @GetMapping(value = "get/{id}")
   public CommonResult getPaymentById(@PathVariable("id")Long id){
      Payment result = paymentService.getPaymentById(id);
      log.info("*******************查询结果："+result);
      if (result!=null){
         return new CommonResult(200,"查询成功,server.port:"+serverPort,result);
      }else {
         return new CommonResult(444,"查询失败,server.port:"+serverPort,null);
      }
   }
}
```

测试http://localhost:7002/或http://localhost:7001/

可以看到两个微服务,provider有两个实例

http://localhost/consumer/payment/get/32

通过返回的信息可以看到访问的总是8001，因为consumer80里的controller已经写死

PAYMENT_URL="http://localhost:8001";

修改**comsumer**

修改为PAYMENT_URL="http://CLOUD-PAYMENT-SERVICE"  其中填写微服务名称来进行集群版本访问，但还不知道具体访问哪一个实例，所以还需要开启负载均衡，在配置类中给RestTemplate添加注解@LoadBalanced。

### 信息完善

provider

**主机实例名称**

eureka.instance.instance-id: payment8001 - payment8002

访问eureka server界面可以看到 微服务实例名称改变

**访问信息**

鼠标停留实例名时左下角可以显示id地址

eureka.instance.prefer-ip-address: true

### DiscoveryClient

对于注册进eureka里面的微服务，可以通过服务发现来获取所有微服务，或指定微服务的所有实例的信息。

```
@Resource
private DiscoveryClient discoveryClient;

@GetMapping(value = "discovery")
	public Object discovery(){
		//获取所有微服务
		List<String> services = discoveryClient.getServices();
		for (String service : services) {
			log.info("*****"+service);
		}
		//根据微服务获取实例信息
		List<ServiceInstance> instances = discoveryClient.getInstances("CLOUD-PAYMENT-SERVICE");
		for (ServiceInstance instance : instances) {
			log.info("***"+instance.getServiceId()+"\t"+instance.getHost()+"\t"+instance.getPort()+"\t"+ instance.getUri());
		}
		return discoveryClient;
	}
```

启动类添加注解

```
@EnableDiscoveryClient
```

访问http://localhost:8001/payment/discovery 查看控制台和返回的json数据

### Eureka自我保护

eureka server系统的红字

某时刻某一微服务不可用了，Eureka不会立即清理，依旧会对该微服务的信息进行保存。属于CAP里面的AP分支。

见D版。

eureka自我保护默认开启

enreka.server.enable-self-preservation: false 关闭

eviction-interval-timer-in-ms: 2000  时间间隔

client provider设置：

instance.lease-renewal-interval-in-seconds:1 client向server发送心跳的时间间隔 默认30s。

instance.lease-expiration-duraion-in-seconds:2   server在收到最后一次心跳等在时间上限，默认90s，超时将剔除服务。

测试，在该设置下如果对一个服务关闭，server将马上对齐进行移除

## Zookeeper 

http://archive.apache.org/dist/zookeeper/下载zookeeper-3.4.9

```
#上传到centos7
tar -zxvf zookeeper-3.4.9.tar.gz  解压
#进入zookeeper-3.4.6目录，创建data目录
mkdir data
#进入conf目录 ，把zoo_sample.cfg 改名为zoo.cfg
mv zoo_sample.cfg zoo.cfg
# 打开zoo.cfg文件,  修改：
#  dataDir=<刚刚创建的data目录的位置>

#关闭防火墙
systemctl stop firewalld
systemctl status firewalld  灰色点
```

测试与微服务主机的连通

**provider**

新建子模块cloud-provider-payment8004

```
<dependencies>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-zookeeper-discovery</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
    </dependencies>
```

```
server:
  port: 8004

spring:
  application:
    name: cloud-provider-payment
  cloud:
    zookeeper:
      connect-string: 192.168.153.134:2181 #zookeeperip地址，2181为其默认端口
```

```
@SpringBootApplication
@EnableDiscoveryClient  //用于向使用consul或者zookeeper作为注册中心时注册服务
public class PaymentMain8004 {
   public static void main(String[] args) {
      SpringApplication.run(PaymentMain8004.class, args);
   }
}
```

```
@RestController
@Slf4j
public class PaymentController {
   @Value("${server.port}")
   private String serverPort;
   
   public String paymentzk(){
      return "springcloud with zookeeper: "+serverPort+"\t"+ UUID.randomUUID().toString();
   }
}
```

查看是否注册和注册信息

```
#centos7 切换到zookeeper-3.4.9/bin  启动zookeeper
./zkServer.sh start           
./zkCli.sh     
# 启动8004 访问http://localhost:8004/payment/zk
# 查看服务。  
ls /   可以看到多了一个services
ls /services   可以看到微服务名称
ls /services/cloud-provider-payment 可以看到一串流水号
get /services/cloud-provider-payment/流水号  可以看到微服务注册信息
```

**注意 默认注册的服务为临时节点：如果微服务断开在心跳过后将移除信息，如果同一个微服务再注册进来流水号将改变。**



**consumer**

新建子模块cloud-consumerzk-order80

配置与provider一样

只需要改名字和端口

```
server:
  port: 80

spring:
  application:
    name: cloud-consumer-order
  cloud:
    zookeeper:
      connect-string: 192.168.153.134:2181
```

```
@SpringBootApplication
@EnableDiscoveryClient
public class OrderZkMain80 {
   public static void main(String[] args) {
      SpringApplication.run(OrderZkMain80.class, args);
   }
}
```

```
@Configuration
public class ApplicationContextConfig {
   @Bean
   @LoadBalanced
   public RestTemplate getRestTemplate(){
      return  new RestTemplate();
   }
}
```

```
@RestController
@Slf4j
public class OrderZkController {
   //zk上暴露的服务名
   public static final String INVOKE_URL="http://cloud-provider-payment";
   @Resource
   private RestTemplate restTemplate;
   @GetMapping("/consumer/payment/zk")
   public String paymentInfo(){
      String result = restTemplate.getForObject(INVOKE_URL+"/payment/zk",String.class);
      return result;
   }
}
```

启动8004 80

ls /services 可以看到两个微服务

### **解决冲突**

如果zk版本冲突，不想跟换zk版本 ，可以修改pom版本

```
<dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-zookeeper-discovery</artifactId>
<!--            先排除自带的zookeeper3.5.3-->
            <exclusions>
                <exclusion>
                    <groupId>org.apache.zookeeper</groupId>
                    <artifactId>zookeeper</artifactId>
                </exclusion>
            </exclusions>
        </dependency>
<!--        添加zookeeper3.4.9-->
        <dependency>
            <groupId>org.apache.zookeeper</groupId>
            <artifactId>zookeeper</artifactId>
            <version>3.4.9</version>
        </dependency>
```

但是在导入新的zk之后有日志冲突，需要排除slf4j-log4j12

```
<dependency>
    <groupId>org.apache.zookeeper</groupId>
    <artifactId>zookeeper</artifactId>
    <version>3.4.9</version>
    <exclusions>
        <exclusion>
            <groupId>org.slf4j</groupId>
            <artifactId>slf4j-log4j12</artifactId>
        </exclusion>
    </exclusions>
</dependency>
```

## Consul

https://www.consul.io/intro/index.html

主要功能：

服务发现：http和dns两种发现方式；

健康检测：支持http,tcp,docker,shell脚本定制化；

KV存储：key/value的存储方式；

多数据中心：Consul支持多数据中心；

可视化web界面。

下载：有windows版本和liux版本

中文文档

https://www.springcloud.cc/spring-cloud-consul.html

安装与运行：

安装视频（官方）：http://learn.hashicorp.com/consul/getting-started/install.html

解压window64版本 运行exe；

在目录下

consul -version   查看版本

开发模式运行：

consul agent -dev

http://localhost:8500  访问Consul首页

**provider**

新建子模块cloud-providerconsul-payment8005

```
<dependencies>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-consul-discovery</artifactId>
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

```yml
server:
  port: 8005

spring:
  application:
    name: consul-provider-payment

  cloud:
    consul:
      host: localhost
      port: 8500             #可以不配置 默认8500
      discovery:
        service-name: ${spring.application.name}
```

```
@SpringBootApplication
@EnableDiscoveryClient
public class PaymentMain8005 {

   public static void main(String[] args) {
      SpringApplication.run(PaymentMain8005.class, args);
   }
}
```

```
@RestController
@Slf4j
public class PaymentController {
	@Value("${server.port}")
	private String serverPort;

	@RequestMapping("payment/consul")
	public String paymentConsul(){
		return "springcloud with consul: "+serverPort+"\t"+ UUID.randomUUID().toString();
	}
}
```

**consumer**

pom与provider一样

```
server:
  port: 80
spring:
  application:
    name: consul-consumer-order

  cloud:
    consul:
      host: localhost
      port: 8500             #可以不配置 默认8500
      discovery:
        service-name: ${spring.application.name}
```

```
@SpringBootApplication
@EnableDiscoveryClient
public class OrderConsulMain80 {
   public static void main(String[] args) {
      SpringApplication.run(OrderConsulMain80.class, args);
   }
}
```

```
@Configuration
public class ApplicationContextConfig {
   @Bean
   @LoadBalanced
   public RestTemplate getRestTemplate(){
      return  new RestTemplate();
   }
}
```

```
@RestController
@Slf4j
public class OrderConsulController {
   //zk上暴露的服务名
   public static final String INVOKE_URL="http://consul-provider-paymen";
   @Resource
   private RestTemplate restTemplate;
   @GetMapping("/consumer/payment/consul")
   public String paymentInfo(){
      String result = restTemplate.getForObject(INVOKE_URL+"/payment/consul",String.class);
      return result;
   }
}
```

## 三个注册中心的异同

| 组件名    | 语言 | CAP  | 健康检查 | 暴露接口 | CLOUD集成 |
| --------- | ---- | ---- | -------- | -------- | --------- |
| Eureka    | java | AP   | 可配支持 | HTTP     | √         |
| Consul    | go   | CP   | 支持     | HTTP/DNS | √         |
| Zookeeper | java | CP   | 支持     | 客户端   | √         |

# 负载均衡

## Ribbon

提供客服端的负载均衡和服务调用

LB （Load Balance） 见D版

在前面的eureka版本的注册中心时，搭建了provider集群，在consumer访问时，轮流访问8001和8002。

即ribbon+RestTemplate

而没有显示的看到pom文件引入ribbon是因为微服务客户端依赖对ribbon进行了集成。

### RestTemplate

api文档见spring.io的spring-framework下

常用方法

getForObject/getForEntity

petForObject/petForEntity

Object:返回对象为响应体种数据转化成的对象，基本可以理解为json

Entity:返回对象为ResponseEntity对象，包含了响应种的一些重要信息，如响应头，状态码，响应体等

测试

对cloud-consumer-order80的controller添加方法：

```
/**
 * getForEntity测试
 */
@GetMapping("consumer/payment/getForEntity/{id}")
public CommonResult<Payment> getPayment2(@PathVariable("id") Long id){
   ResponseEntity<CommonResult> forEntity = restTemplate.getForEntity(PAYMENT_URL + "/payment/get/" + id, CommonResult.class);
   if (forEntity.getStatusCode().is2xxSuccessful()){
      return forEntity.getBody();
   }else {
      return new CommonResult<>(444,"操作失败");
   }
}
```

访问http://localhost/consumer/payment/getForEntity/31

同样的查询结果，相比getForObject，getForEntity只是多了一层封装。

### 负载规则

ribbon负载规则问百度。

**官方警告：负载规则替换的自定义配置类不能放在@ComponentScan所扫描的包下，否则我们自定义的这个配置类就会被所有ribbon客户端所共享，达不到特殊化定制的目的。**

```
@Configuration
public class MySelfRule {
   @Bean
   public IRule myRule(){
      return new RandomRule();
   }
   
}
```

为实现对指定微服务特殊定制负载规则，需要指定微服务名称和规则类：

在启动类添加注解

```
@RibbonClient(name="CLOUD-PAYMENT-SERVICE",configuration = MySelfRule.class)
```

重启运行eureka server 2个provider 1个 consumer 访问http://localhost/consumer/payment/get/31  可以看到8001 8002随机访问。

也可以通过配置文件全局修改：

```
{服务名称}.ribbon.NFLoadBalancerRuleClassName
如：
user-service:
	ribbon:
		NFLoadBalancerRuleClassName: com.netflix.loadbalancer.RandomRule
```

**自定义规则**

先了解ribbon轮询负载均衡算法：

rest接口第几次请求数%服务器集群总数量= 实际调用服务器的位置下标，每次服务重启后rest接口计数从1开始。

可查看源码RoundRobinRule.

这里涉及CAS和自旋锁的知识。

参照源码自己编写自定义规则：

去掉@LoadBalanced注解.

创建LoadBalancer接口，和实现类MyLB：

```
public interface LoadBalancer {
   ServiceInstance instance(List<ServiceInstance> serviceInstanceList);
}
```

```
@Component
public class MyLB implements LoadBalancer{

   private AtomicInteger atomicInteger = new AtomicInteger(0);
   public final int getAndIncrement(){
      int current;
      int next;
      do{
         current=this.atomicInteger.get();
         next=current>=Integer.MAX_VALUE?0:current+1;
      }while (!this.atomicInteger.compareAndSet(current,next));
      System.out.println("*****next:"+next);
      return next;
   }
   @Override
   public ServiceInstance instance(List<ServiceInstance> serviceInstanceList) {
      int index = getAndIncrement() % serviceInstanceList.size();//%2
      return serviceInstanceList.get(index);
   }
}
```

修改provider8001 8002的controller

```
@Value("${server.port}")
	private String serverPort;
@GetMapping("lb")
public String lb(){
   return serverPort;
}
```

修改consumer80的 controller

```
@Resource
private RestTemplate restTemplate ;
@Resource
private LoadBalancer loadBalancer;
@Resource
private DiscoveryClient discoveryClient;
@GetMapping("consumer/payment/lb")
public String getPaymentLB(){
   List<ServiceInstance> instances = discoveryClient.getInstances("CLOUD-PAYMENT-SERVICE");
   if (instances==null||instances.size()<=0){
      return null;
   }
   ServiceInstance serviceInstance = loadBalancer.instance(instances);
   URI uri = serviceInstance.getUri();
   return restTemplate.getForObject(uri+"/payment/lb",String.class);
}
```

http://localhost/consumer/payment/lb查看端口 可以看到8001 8002切换

这是自己手写的轮询算法。并在80端可以看到next输出次数。

如果重启consumer80 ，重新从1开始输出。

## OpenFeign

Feign见D版，简化ribbon服务调用，自动封装服务调用客户端的开发量。以接口+注解的形式配置。**Feign集成了ribbon来实现负载均衡（轮询）**。

Feign的使用方式：使用Feign的注解定义接口，调用跟这个接口，就可以调用服务注册中心的服务。

OpenFeign在Feign的基础上支持了SpringMVC的注解，如RequestMapping等。OpenFeign可以解析SpringMVC的@RequestMapping注解下的接口，并通过动态代理的方式产生实现类，实现类中作负载均衡并调用其他服务。

使用方式：微服务调用接口+@FeignClient

新建子模块cloud-consumer-feign-order80，还是使用eureka注册中心

```
<dependencies>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-openfeign</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
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
server:
  port: 80

eureka:
  client:
    register-with-eureka: false
    service-url: 
      defaultZone: http://eureka7001.com:7001/eureka/,http://eureka7002.com:7002/eureka/
```

```
@SpringBootApplication
@EnableFeignClients
public class OrderFeignMain80 {
   public static void main(String[] args) {
      SpringApplication.run(OrderFeignMain80.class, args);
   }
}
```

```
@Component
@FeignClient(value="CLOUD-PAYMENT-SERVICE")
public interface PaymentFeignService {
	@GetMapping("/payment/get/{id}")
	CommonResult<Payment> getPaymentById(@PathVariable("id") long id);
}
```

```
@RestController
@Slf4j
public class OrderFeignController {
   @Resource
   private PaymentFeignService paymentFeignService;

   @GetMapping("consumer/payment/get/{id}")
   public CommonResult<Payment> getPaymentById(@PathVariable("id") Long id){
      return paymentFeignService.getPaymentById(id);
   }
}
```

启动 eureka集群， provider 8001 8002  opentFeign80

多次 http://localhost/consumer/payment/get/31  轮询

### 超时控制

provider模拟超时

```
@GetMapping("timeout")
public String paymentFeignTimeout(){
   //暂停几秒钟线程
   try {
      TimeUnit.SECONDS.sleep(3);
   }catch (InterruptedException e){
      e.printStackTrace();
   }
   return serverPort;
}
```

```
@Component
@FeignClient(value="CLOUD-PAYMENT-SERVICE")
public interface PaymentFeignService {
   @GetMapping("/payment/get/{id}")
   CommonResult<Payment> getPaymentById(@PathVariable("id") long id);

   @GetMapping("/payment/timeout")
   String paymentFeignTimeout();
}
```

```
@GetMapping("consumer/payment/timeout")
public String paymentFeignTimeout(){
   //客户端一般默认等待1s
   return paymentFeignService.paymentFeignTimeout();
}
```

测试 

启动2个server  provider8001  consumer80

先测http://localhost:8001/payment/timeout，等待3s

再测http://localhost/consumer/payment/timeout，错误页面

Read timed out executing GET http://CLOUD-PAYMENT-SERVICE/payment/timeout

**因为openFeign客户端一般默认等待1s。**

解决办法：设置feign客户端超时控制

feign consumer 80 :   yml :

```
server:
  port: 80

eureka:
  client:
    register-with-eureka: false
    service-url:
      defaultZone: http://eureka7001.com:7001/eureka/,http://eureka7002.com:7002/eureka/
#设置feign客户端超时时间（openFeign默认支持ribbon）
ribbon:
  #建立连接所用的时间，适用于网络状况正常的情况下，两端连接所用的时间
  ReadTimeout: 5000
  #建立连接后从服务器读取到可用资源所用的时间
  ConnectTimeout: 5000
```

### 日志增强

feign提供了日志打印功能，我们可以通过配置来调整日志级别，从而了解feign中http请求的细节，说白了就是feign接口的调用情况进行监控和输出。

日志级别：

NONE:默认，不显示任何日志；

BASIC:仅记录请求方法、URL、响应状态码及执行时间；

HEADERS:除了BASIC中的信息之外，还有请求和响应的头信息；

FULL:除了HEADERS中的信息之外，还有请求和响应的正文及元数据。

测试FULL：

在feign consumer 80 

```
@Configuration
public class FeignConfig {
	@Bean
	Logger.Level feignLoggerLevel(){
		return Logger.Level.FULL;
	}
}

```

```
server:
  port: 80

eureka:
  client:
    register-with-eureka: false
    service-url:
      defaultZone: http://eureka7001.com:7001/eureka/,http://eureka7002.com:7002/eureka/
#设置feign客户端超时时间（openFeign默认支持ribbon）
ribbon:
  #建立连接所用的时间，适用于网络状况正常的情况下，两端连接所用的时间
  ReadTimeout: 5000
  #建立连接后从服务器读取到可用资源所用的时间
  ConnectTimeout: 5000
logging:
  level:
    #feign日志以什么级别监控哪个接口
    edu.hunnu.cloud.service.PaymentFeignService: debug
```

测试 访问http://localhost/consumer/payment/get/31

可以看到DEBUG输出日志 

# 服务降级

## Hystrix+JMeter

介绍见D版

重要概念：

服务降级fallback：在某些情况（程序运行异常，超时，服务熔断触发服务降级，线程池打满）下，不让客户端等待并立刻返回一个友好的提示；

服务熔断circuitbreak：类似保险丝达到最大服务访问后，直接拒绝访问，拉闸限电，然后调用服务降级的方法并返回友好提示；

服务限流flowlimit：秒杀高并发等操作，严禁一窝蜂的过来拥挤，大家排队，一秒钟N个，有序进行；

**provider**

新建子模块cloud-provider-hystrix-payment8001

在service中模拟超时

```
 <dependencies>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-netflix-hystrix</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-actuator</artifactId> 
    </dependencies>
```

```
server:
  port: 8001

spring:
  application:
    name: cloud-provider-hystrix-payment  #微服务名称

eureka:
  client:
    register-with-eureka: true
    fetch-registry: true
    service-url:
      defaultZone: http://eureka7001.com:7001/eureka/
```

```
@SpringBootApplication
@EnableEurekaClient
public class PaymentHystrixMain8001 {
   public static void main(String[] args) {
      SpringApplication.run(PaymentHystrixMain8001.class, args);
   }
}
```

```
@Service
public class PaymentService {
	/**
	 * 正常访问
	 * @param id
	 * @return
	 */
	public String paymentInfo_OK(Integer id){
		return "线程池："+Thread.currentThread().getName()+"  paymentInfo_OK, id: "+id+"*********";
	}

	/**
	 * 超时方法 验证降级
	 * @param id
	 * @return
	 */
	public String paymentInfo_TimeOut(Integer id){
		int timeNumber=5;
		try {
			TimeUnit.SECONDS.sleep(timeNumber);
		} catch (InterruptedException e) {
			e.printStackTrace();
		}
		return "线程池："+Thread.currentThread().getName()+"  paymentInfo_OK, id: "+id+"*****耗时"+timeNumber+"秒";
	}
}
```

```
@RestController
@Slf4j
public class PaymentController {
   @Resource
   private PaymentService paymentService;
   @Value("${server.port}")
   private String serverPort;
   @GetMapping("payment/hystrix/ok/{id}")
   public String paymentInfo_OK(@PathVariable("id")Integer id){
      String result = paymentService.paymentInfo_OK(id);
      log.info("**OK**result:"+result);
      return result;
   }
   @GetMapping("payment/hystrix/timeout/{id}")
   public String paymentInfo_TimeOut(@PathVariable("id")Integer id){
      String result = paymentService.paymentInfo_OK(id);
      log.info("**TimeOut**result:"+result);
      return result;
   }
}
```

http://localhost:8001/payment/hystrix/ok/31

http://localhost:8001/payment/hystrix/timeout/31

都能正常访问。 非高并发

**Jmeter**

开启jmeter，开启20000个请求去访问CLOUD-PROVIDER-HYSTRIX-PAYMENT服务

运行解压缩包/bin下的ApacheJMeter.jar

测试计划-> 添加 ->线程 -> 线程组

```
名称 线程组202002
线程数 200
循环次数 100
```

保存

右击线程组-> 添加 -> 取样器Sampler -> HTTP请求

填写 服务器名称 （localhost） 端口  http请求路径http://localhost:8001/payment/hystrix/timeout/31

启动线程组

浏览器访问

http://localhost:8001/payment/hystrix/ok/31

http://localhost:8001/payment/hystrix/timeout/31

发现本来可以秒回的ok服务变慢了，timeout更不用说。

因为tomcat（springboot默认集成tomcat）的默认工作线程数被打满了，没有多余的线程来分解压力和处理。所以这里也可以变相减少tomcat工作线程数来测试。可以从终端输出中看到对每个请求的处理线程名字。

**consumer**

**hystrix可以部署在消费端也可以部署在提供端，一般部署在消费端**。所以这里新建consumer，注意，手动延迟是在provider的service设置的。

新建子模块cloud-consumer-feign-hystrix-order80

这里使用的是feign

```
<dependencies>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-openfeign</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-netflix-hystrix</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
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
server:
  port: 80

eureka:
  client:
    register-with-eureka: false
    service-url: 
     defaultZone: http://eureka7001.com:7001/eureka
```

```
@SpringBootApplication
@EnableFeignClients
public class OrderHystrixMain80 {
   public static void main(String[] args) {
      SpringApplication.run(OrderHystrixMain80.class, args);
   }
}
```

```
@Component
@FeignClient(value = "CLOUD-PROVIDER-HYSTRIX-PAYMENT")
public interface PaymentHystrixService {
   @GetMapping("payment/hystrix/ok/{id}")
   public String paymentInfo_OK(@PathVariable("id")Integer id);
   @GetMapping("payment/hystrix/timeout/{id}")
   public String paymentInfo_TimeOut(@PathVariable("id")Integer id);
}
```

```
@RestController
@Slf4j
public class OrderHystrixController {
   @Resource
   private PaymentHystrixService paymentHystrixService;
   @GetMapping("consumer/payment/hystrix/ok/{id}")
   public String paymentInfo_OK(@PathVariable("id")Integer id){
      String result = paymentHystrixService.paymentInfo_OK(id);
      return result;
   }
   @GetMapping("consumer/payment/hystrix/timeout/{id}")
   public String paymentInfo_TimeOut(@PathVariable("id")Integer id){
      String result = paymentHystrixService.paymentInfo_TimeOut(id);
      return result;
   }
}
```

consumer配了openFeign自带超时控制

http://localhost/consumer/payment/hystrix/ok/31

都能正常访问。

再开启jmeter对consumer8001 timeout服务的增压，再访问

http://localhost/consumer/payment/hystrix/ok/31

要么转圈圈 要么openFeign的超时错误。

**原因：在provider8001自测增压时，consumer80干等。**

解决办法：

超时导致服务器变慢->超时不再等待；

出错（服务器宕机或程序运行出错）->出错要有兜底。

即服务降级。

### 服务降级

@HystrixCommand

**fallback**：

**provider处理服务降级（次要）**

provider先从自身找问题，设置自身调用超时时间的峰值，超时需要降级，兜底处理的方法。

修改provider 

```
@HystrixCommand(fallbackMethod = "paymentInfo_TimeOutHandler",commandProperties = {
			@HystrixProperty(name="execution.isolation.thread.timeoutInMilliseconds",value = "3000")
	})//3秒峰值超时
public String paymentInfo_TimeOut(Integer id){
   int timeNumber=5;
   try {
      TimeUnit.SECONDS.sleep(timeNumber);
   } catch (InterruptedException e) {
      e.printStackTrace();
   }
   return "线程池："+Thread.currentThread().getName()+"  paymentInfo__TimeOut, id: "+id+"*****耗时"+timeNumber+"秒";
}
/**
 * 兜底方法
 */
public String paymentInfo_TimeOutHandler(Integer id){
   return "线程池："+Thread.currentThread().getName()+"  paymentInfo_TimeOutHandler, id: "+id+"*********系统繁忙";
}
```

当服务方法失败并抛出错误信息后，自动调用@HystrixCommand指定的

fallbackMethod的类中的方法。并设置超时时间峰值。

主启动类 添加注解

```
@EnableCircuitBreaker
```

http://localhost:8001/payment/hystrix/timeout/31

被兜底方法处理，该处为超时异常。

制造算术异常，在方法中添加错误计算；

被兜底方法处理，该处为算术异常。

**consumer处理服务降级（主要）**

consumer虽然不直接处理业务，但是也要有自己的服务降级处理。而且服务降级一般都在consumer端。

```
feign:
  hystrix:
    enabled: true
```

启动类添加注解

```
@EnableHystrix
```

```
@GetMapping("consumer/payment/hystrix/timeout/{id}")
@HystrixCommand(fallbackMethod = "paymentInfo_TimeOutHandler",commandProperties = {
      @HystrixProperty(name="execution.isolation.thread.timeoutInMilliseconds",value = "1500")
})
public String paymentInfo_TimeOut(@PathVariable("id")Integer id){
   String result = paymentHystrixService.paymentInfo_TimeOut(id);
   return result;
}
public String paymentInfo_TimeOutHandler(@PathVariable("id")Integer id){
   return "消费者80：支付系统繁忙，请稍后";
}
```

#### 全局服务降级

目前问题：每个业务方法对应一个兜底方法，代码膨胀。

解决办法：对于需要特殊处理的服务，依旧使用上面方法降级。对于其他的，添加全局服务降级统一处理。

@DefaultProperties(defaultFallback="")

```
@RestController
@Slf4j
@DefaultProperties(defaultFallback = "payment_Global_FallbackMethod")
public class OrderHystrixController {
   @Resource
   private PaymentHystrixService paymentHystrixService;
   @GetMapping("consumer/payment/hystrix/ok/{id}")
   public String paymentInfo_OK(@PathVariable("id")Integer id){
      String result = paymentHystrixService.paymentInfo_OK(id);
      return result;
   }
   @GetMapping("consumer/payment/hystrix/timeout/{id}")
// @HystrixCommand(fallbackMethod = "paymentInfo_TimeOutHandler",commandProperties = {
//       @HystrixProperty(name="execution.isolation.thread.timeoutInMilliseconds",value = "1500")
// })
   @HystrixCommand
   public String paymentInfo_TimeOut(@PathVariable("id")Integer id){
      int a=10/0;
      String result = paymentHystrixService.paymentInfo_TimeOut(id);
      return result;
   }
   public String paymentInfo_TimeOutHandler(@PathVariable("id")Integer id){
      return "消费者80：支付系统繁忙，请稍后";
   }

   public String payment_Global_FallbackMethod(){
      return "**********全局服务降级统一处理************";
   }
}
```

**说明：@HystrixCommand表示该服务方法要使用服务降级处理，如果HystrixCommand中指定了fallbackMethod则特殊处理，否则按照@DefaultProperties(defaultFallback="")指定的全局降级处理。达到通用和独享分开处理的效果。**

#### FeignFallback

目前问题：如果provider宕机了，需要处理；并且所有服务方法和兜底方法都在一个类里面，耦合度高。

解决办法：因为这里的服务降级在consumer80实现的，与provider8001没关系，而consumer80又是使用feign接口+注解的形式来调用服务，只需要为feign接口添加一个服务降级处理的实现类即可实现解耦。并且让客户机在服务端不可用时也会获得提示信息而不会挂机耗死服务器。

修改consumer hystrix 80 ，新建类PaymentFallbackService实现接口PaymentHystrixService，统一为接口里面的方法异常处理。

```
@Component
public class PaymentFallbackService implements PaymentHystrixService{
   @Override
   public String paymentInfo_OK(Integer id) {
      return "---------PaymentFallbackService 解耦形式降级处理";
   }

   @Override
   public String paymentInfo_TimeOut(Integer id) {
      return "---------PaymentFallbackService 解耦形式降级处理";
   }
}
```

并修改PaymentHystrixService接口，指定降级类

```
@Component
@FeignClient(value = "CLOUD-PROVIDER-HYSTRIX-PAYMENT",fallback = PaymentFallbackService.class)
public interface PaymentHystrixService {
   @GetMapping("payment/hystrix/ok/{id}")
   public String paymentInfo_OK(@PathVariable("id")Integer id);
   @GetMapping("payment/hystrix/timeout/{id}")
   public String paymentInfo_TimeOut(@PathVariable("id")Integer id);
}
```

### 服务熔断

熔断可以理解为降级的一种，将降级逻辑切换为主逻辑。当扇出链路的某个微服务出错不可用或者响应时间太长时，会进行服务的降级，进而熔断该节点微服务的调用，快速返回错误的响应信息。当检测到该节点的微服务调用响应正常后，恢复调用链路。

当失败的调用到一定阈值，默认为5秒内20次调用失败，就会启动熔断机制。注解同样为HystrixCommand

官网说明熔断器的过程，有关闭、半打开、打开状态。

provider端

**HystrixCommand**

要了解HystrixCommand注解的commandProperties属性的设置，类HystrixCommandProperties中说明了所有参数的赋值字符串和默认值。

```
@HystrixCommand(fallbackMethod = "paymentCircuitBreaker_fallback",commandProperties = {
			@HystrixProperty(name="circuitBreaker.enabled",value = "true"),
			@HystrixProperty(name="circuitBreaker.requestVolumeThreshold",value = "10"),
			@HystrixProperty(name="circuitBreaker.sleepWindowInMilliseconds",value = "10000"),
			@HystrixProperty(name="circuitBreaker.errorThresholdPercentage",value = "60"),
	})
	public String paymentCircuitBreaker(@PathVariable("id") Integer id){
		if (id<0){
			throw new RuntimeException("************id  不能为负数");
		}
		String serialNumber = IdUtil.simpleUUID();
		return Thread.currentThread().getName()+"\t调用成功，流水号"+serialNumber;
	}

	public String paymentCircuitBreaker_fallback(@PathVariable("id") Integer id){
		return "id 不能为负数，请重试    id:"+id;
	}
```

circuitBreaker.enabled：是否开启断路器

circuitBreaker.sleepWindowInMilliseconds：时间窗口期：断路器确定是否打开需要统计一些请求和错误数据，而统计的时间范围就是快照时间窗，默认为最近的10s。

circuitBreaker.requestVolumeThreshold：请求总数阈值：在快照时间窗口内，必须满足请求总数阈值才有资格熔断。默认为20， 意味着在10s以内，如果该hystrix服务的调用次数不足20次，即时所有的请求都超时或其他原因失败，断路器都不会打开。

circuitBreaker.errorThresholdPercentage：错误百分比阈值 ，当请求总数在快照窗口内超过了阈值，比如发生了30次调用，有15次发生了超时异常，超过了默认的50%阈值，这时候断路器将打开

属性有很多，这几个通常搭配使用，所以上述代码配置的意思是，在10s内请求达到10次或以上的条件下，如果失败率达到60%就跳闸。一段时间后（默认5s），这时候断路器半开状态，会让其中一个请求进行转发，如果成功，断路器关闭，如果失败，继续开启。重复检测服务访问。

```
@GetMapping("payment/circuit/{id}")
public String paymentCircuitBreaker(@PathVariable("id")Integer id){
   String result = paymentService.paymentCircuitBreaker(id);
   log.info("****Circuit***result:"+result);
   return result;
}
```

测试：启动单击7001 hystrix8001 hystrix80。

正常访问各一次，验证是否有问题：

http://localhost:8001/payment/circuit/31

http://localhost:8001/payment/circuit/-31

上述没问题后，测试跳闸情况：

访问多次（10S内10次以上且60%）：http://localhost:8001/payment/circuit/-31

再访问：http://localhost:8001/payment/circuit/31

可以看到本来正常访问的正数在一段时间内也走的是服务降级，这就是服务熔断，在熔断恢复后，又可正常访问

### 服务监控

**hystrixDashboard**

新增子模块cloud-consumer-hystrix-dashboard9001

```
<dependencies>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-netflix-hystrix-dashboard</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-actuator</artifactId>
    </dependency>
</dependencies>
```

注意 ，那些需要被监控的微服务都必须提供actuator依赖

```
server:
  port: 9001
```

```
@SpringBootApplication
@EnableHystrixDashboard
public class HystrixDashboardMain9001 {
   public static void main(String[] args) {
      SpringApplication.run(HystrixDashboardMain9001.class, args);
   }
}
```

```
//修改需要被监控的微服务端
@SpringBootApplication
@EnableEurekaClient
@EnableCircuitBreaker
public class PaymentHystrixMain8001 {
   public static void main(String[] args) {
      SpringApplication.run(PaymentHystrixMain8001.class, args);
   }
   /**
    * 此配置是为了服务监控而配置，与服务容错本身无关，springcloud升级后的坑
    * ServletRegistrationBean因为springboot的默认路径不是"/hystrix.stream"，只要 在自己的项目里配置上下面的servlet就可以了
    */
   @Bean
   public ServletRegistrationBean getServlet(){
      HystrixMetricsStreamServlet streamServlet = new HystrixMetricsStreamServlet();
      ServletRegistrationBean registrationBean = new ServletRegistrationBean(streamServlet);
      registrationBean.setLoadOnStartup(1);
      registrationBean.addUrlMappings("/hystrix.stream");
      registrationBean.setName("HystrixMetridcsStreamServlet");
      return registrationBean;
   }
}
```

启动7001，启动9001，启动 8001

用9001监控8001：

在http://localhost:9001/hystrix 中填写监控url http://localhost:8001/hystrix.stream Delay 2000 title随意

访问  http://localhost:8001/payment/circuit/31  多次刷新  ：

监控区 圆圈变大。 如何观看 ，见D版。

# 路由

Zuul1.0基于阻塞I/O的网关，基于Servlet2.5使用阻塞架构（在高并发情况下因为线程资源代价昂贵），不支持任何长连接如WebSocket，Zuul的涉及模式与Nginx很像，每次I/O都是从工作线程中选择一个执行，请求线程被阻塞到工作线程完成，差别是nginx用c++，zuul用java实现，而jvm本身会有一次加载较慢的情况，使得zuul性能相对较差。已进入维护阶段；

Zuul2.0更新跳票，且Springcloud没集成；

Gateway由Springcloud团队研发，基于WebFlux+Netty的（异步非阻塞模型）框架。

## Gateway

核心概念：

路由Route：构建网关的基本模块，由ID、目标URI、一系列的断言和过滤器组成，如果断言为true则匹配路由。

断言Predicate：参考java8的java.util.function.Predicate,开发人员可以匹配http请求中的所有内容。

过滤Filter：可以在请求被路由前或者之后对请求进行修改。

**Route**

新增子模块cloud-gateway-gateway9527

```
<dependencies>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-gateway</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
    </dependency>
</dependencies>
```

yml 关键，配置映射规则 uri+Path的kv对应8001 controller的访问地址

```
server:
  port: 9527

spring:
  application:
    name: cloud-gateway
  cloud:
    gateway:
      routes:
        - id: payment_routh               #路由的ID，没有固定规则但要求唯一，建议配合服务名
          uri: http://localhost:8001      #匹配后提供服务的路由地址
          predicates:
            - Path=/payment/get/**          #断言，路径相匹配的路由

        - id: payment_routh2               #路由的ID，没有固定规则但要求唯一，建议配合服务名
          uri: http://localhost:8001     #匹配后提供服务的路由地址
          predicates:
            - Path=/payment/lb/**          #断言，路径相匹配的路由

eureka:
  instance:
    hostname: cloud-gateway-service
  client:
    register-with-eureka: true
    fetch-registry: true
    service-url:
      defaultZone: http://eureka7001.com:7001/eureka
```

```
@SpringBootApplication
@EnableEurekaClient
public class GatewayMain9527 {
	public static void main(String[] args) {
		SpringApplication.run(GatewayMain9527.class, args);
	}
}
```

### 路由配置

gateway配置网关路由有yml和bean的方式。

上面的网关配置使用的是yml的方式。

下面介绍bean的方式（更繁琐）：在代码中注入RouteLocator的bean。

```
@Configuration
public class GatewayConfig {
	/**
	 * 配置规则：当访问http://localhost:9527/guonei时候会自动转发到地址：http://news.baidu.com/guonei
	 */
	@Bean
	public RouteLocator customRouteLocator(RouteLocatorBuilder routeLocatorBuilder){
		RouteLocatorBuilder.Builder routes = routeLocatorBuilder.routes();
		routes.route("path_route_baidu",r->r.path("/guonei").uri("http://news.baidu.com/guonei"));
		return routes.build();
	}
}
```

**动态路由（LB）**

不同于使用ribbon或feign访问微服务，来实现负载均衡。这里使用的是转发的方式来访问，为达到负载均衡的效果这里使用服务名来实现动态路由。

默认情况下Gateway会根据注册中心的服务列表，以注册中心上的微服务名为路径创建动态路由进行转发， 从而实现动态路由功能。

```
server:
  port: 9527

spring:
  application:
    name: cloud-gateway
  cloud:
    gateway:
      discovery:
        locator:
          enabled: true   #开启从注册中心动态创建路由功能，利用微服务名进行路由
      routes:
        - id: payment_routh               #路由的ID，没有固定规则但要求唯一，建议配合服务名
          #lb为uri的协议，表示Gateway的负载均衡功能
          uri: lb://CLOUD-PAYMENT-SERVICE   #利用微服务名进行路由        provider集群
          predicates:
            - Path=/payment/get/**          #断言，路径相匹配的路由

        - id: payment_routh2               #路由的ID，没有固定规则但要求唯一，建议配合服务名
          uri: lb://CLOUD-PAYMENT-SERVICE   #利用微服务名进行路由
          predicates:
            - Path=/payment/lb/**          #断言，路径相匹配的路由

eureka:
  instance:
    hostname: cloud-gateway-service
  client:
    register-with-eureka: true
    fetch-registry: true
    service-url:
      defaultZone: http://eureka7001.com:7001/eureka
```

测试，启动7001 普通版8001 8002，9527

多次访问

http://localhost:9527/payment/get/32

http://localhost:9527/payment/lb

#### Predicate

在9527的终端可以看到

o.s.c.g.r.RouteDefinitionRouteLocator    : Loaded RoutePredicateFactory [After]
o.s.c.g.r.RouteDefinitionRouteLocator    : Loaded RoutePredicateFactory [Before]
o.s.c.g.r.RouteDefinitionRouteLocator    : Loaded RoutePredicateFactory [Between]
o.s.c.g.r.RouteDefinitionRouteLocator    : Loaded RoutePredicateFactory [Cookie]
o.s.c.g.r.RouteDefinitionRouteLocator    : Loaded RoutePredicateFactory [Header]
o.s.c.g.r.RouteDefinitionRouteLocator    : Loaded RoutePredicateFactory [Host]
o.s.c.g.r.RouteDefinitionRouteLocator    : Loaded RoutePredicateFactory [Method]
o.s.c.g.r.RouteDefinitionRouteLocator    : Loaded RoutePredicateFactory [Path]
o.s.c.g.r.RouteDefinitionRouteLocator    : Loaded RoutePredicateFactory [Query]
o.s.c.g.r.RouteDefinitionRouteLocator    : Loaded RoutePredicateFactory [ReadBodyPredicateFactory]
o.s.c.g.r.RouteDefinitionRouteLocator    : Loaded RoutePredicateFactory [RemoteAddr]
o.s.c.g.r.RouteDefinitionRouteLocator    : Loaded RoutePredicateFactory [Weight]
o.s.c.g.r.RouteDefinitionRouteLocator    : Loaded RoutePredicateFactory [CloudFoundryRouteService]

就是yml文件中predicates属性的kv，我们前面只使用了Path。

Gateway创建Route对象时，使用RoutePredicateFactory创建Predicate对象，Predicate对象可以赋值给Route。Gateway包含许多内置的Route Predicate Factories。所有的Predicate都匹配HTTP的不同属性，多种Route Predicate Factories可以组合，并通过逻辑and。

详细见官方docs的Route Predicate Factories目录，有举例。

比如 \- After=2017-01-20T17:42:47.789-07:00[America/Denver] 表示

这条路由匹配2017年1月20日1日17:42山时间（丹佛）之后的任何请求。

问题在于 时间串怎么获取和使用？重点为时区问题。

提示：以下例子的启动均为 7001 8001（或加上8002） 9527.

方式如下：

```
public class T2 {
   public static void main(String[] args) {
      ZonedDateTime zonedDateTime = ZonedDateTime.now(); //获取当前时间+时区
      System.out.println(zonedDateTime);
   }
}
```

```
2021-05-24T19:50:32.417+08:00[GMT+08:00]
```

给lb路由设置After Route Predicate，时间需要设置到当前延后

```
- id: payment_routh2               #路由的ID，没有固定规则但要求唯一，建议配合服务名
#          uri: http://localhost:8001     #匹配后提供服务的路由地址
          uri: lb://CLOUD-PAYMENT-SERVICE   #利用微服务名进行路由
          predicates:
            - Path=/payment/lb/**          #断言，路径相匹配的路由
            - After=2021-05-24T19:50:32.417+08:00[GMT+08:00]
```

http://localhost:9527/payment/lb

可以看到指定时间前不能访问，可用于秒杀，服务开放现实。其他类似的有Before和Between.



Cookie Route Predicate:

比如 \- Cookie=chocolate, ch.p 表示 此路由匹配具有名为chocolate的cookie的请求，其值与ch.p正则表达式匹配。

给lb路由设置：- Cookie=username,lzf

1.不带cookie访问 

http://localhost:9527/payment/lb

bingo：这里可以使用类似黑窗口版本的postman的curl访问：

curl http://localhost:9527/payment/lb

不可访问

2.带上cookie访问

curl http://localhost:9527/payment/lb --cookie "username=lzf"

可以正常访问



Header Route Predicate:

比如 - Header=X-Request-Id, \d+  表示如果请求具有名为x-request-id的标题，则此路由匹配，其值与\d+正则表达式（即，它具有一个或多个数字的值）。

添加到lb：

1.不带Header访问

curl http://localhost:9527/payment/lb

不可访问

2.带上Header访问

curl http://localhost:9527/payment/lb -H "X-Request-Id:-123"

curl http://localhost:9527/payment/lb -H "X-Request-Id:123"



其他查阅资料。

总结，Predicate就是为了实现一组匹配规则让请求过来找到对应的Route进行处理。

#### Filter

生命周期：pre和post

种类：GatewayFilter和GlobalFilter

使用方式与Predicates类似 见官网。

如：filters:        - AddRequestHeader=X-Request-red, blue

对所有匹配请求添加X-Request-Red：Blue Header到Downstream Request的标题。

**自定义过滤**

通常不适宜自带的过滤器，实际中可以在此处token验证。

两个主要接口 GlobalFilter和 Ordered

用来全局日志记录、统一网关鉴权等。

在9527的filter包新建类

```
@Component
@Slf4j
public class MyLogGatewayFilter implements GlobalFilter, Ordered {
   @Override
   public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
      log.info("*****************come in MyLogGatewayFilter:"+new Date());
      String uname = exchange.getRequest().getQueryParams().getFirst("uname");
      if (uname==null){
         log.info("***********用户名为null");
         exchange.getResponse().setStatusCode(HttpStatus.NOT_ACCEPTABLE);
         return exchange.getResponse().setComplete();
      }
      return chain.filter(exchange);
   }

   @Override
   public int getOrder() {
      return 0;
   }
}
```

测试：去掉yml predicates 除了Path之外的内容，启动7001，8001，8002，9527.

http://localhost:9527/payment/lb  不能访问

http://localhost:9527/payment/lb?uname=123 可以访问

# 服务配置

## Config

介绍和比较见D版

新建springcloud-config仓库并clone到E:\git下

新建子模块cloud-config-center-3344

```
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-config-server</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
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

yml  后面的一些配置无关紧要 可省略 像01版

```
server:
  port: 3344

spring:
  application:
    name: cloud-config-center
  cloud:
    config:
      server:
        git:
          uri: https://gitee.com/li-zhuofan-350562/springcloud-config.git
          search-paths: 
            - springcloud-config
      label: master
eureka:
  client:
    service-url: 
      defaultZone: http://eureka7001.com:7001/eureka
```

```
@SpringBootApplication
@EnableConfigServer
public class ConfigCenterMain3344 {
   public static void main(String[] args) {
      SpringApplication.run(ConfigCenterMain3344.class, args);
   }
}
```

hosts:127.0.0.1 config3344.com

添加配置文件config-dev、config-test、config-prod

内容为

config:
  info: "master branch,springcloud-config/config-dev.yml version=1"

git add .

git commit -m ""

git push origin master

测试：启动7001，3344

访问：http://config3344.com:3344/master/config-dev.yml

http://config3344.com:3344/master/config-test.yml

显示配置文件内容。

**配置文件读取规则**

lable 分支

1./{label}/{application}-{profile}.yml

master分支:

http://config3344.com:3344/master/config-dev.yml

http://config3344.com:3344/master/config-test.yml

http://config3344.com:3344/master/config-prod.yml

dev分支：

http://config3344.com:3344/dev/config-dev.yml

2./{application}-{profile}.yml 没有指定分支，使用yml文件中指定的分支。

http://config3344.com:3344/config-dev.yml

http://config3344.com:3344/master/config-xxx.yml（不存在的配置） 返回空json

3./{application}/{profile}[/{label}].yml

### client

新建子模块cloud-config-client3355，搭建客户端读取远程仓库的配置文件信息

```
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-config</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
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

bootstrap.yml

注意：client模块下的application.yml文件改为bootstrap.yml文件很关键，因为bootstrap.yml文件比application.yml先加载，优先级更高。

```
server:
  port: 3355

spring:
  application:
    name: config-client
  cloud:
    #config客户端配置
    config:
      label: master   #分支名称
      name: config    #配置文件名称
      profile: dev    #读取后缀名称
      uri: http://config3344.com:3344
eureka:
  client:
    service-url: 
      defaultZone: http://eureka7001.com:7001/eureka
```

综上 master分支上的config-dev.yml被读取，即访问http://config3344.com:3344/master/config-dev.yml

```
@EnableEurekaClient
@SpringBootApplication
public class ConfigClientMain3355 {
   public static void main(String[] args) {
      SpringApplication.run(ConfigClientMain3355.class, args);
   }
}
```

```
@RestController
public class ConfigClientController {
   @Value("${config.info}")
   private String configInfo;
   
   @GetMapping("info")
   public String getConfigInfo(){
      return configInfo;
   }
}
```

测试：启动7001 3344 3355

http://localhost:3355/info

小提示：看区别 3344要转圈等待  3355秒访问

### 配置"动态"刷新

问题：在7001 3344 3355运行时对git远程仓库的配置文件修改，如将dev版本改为2，

则不重启微服务访问http://config3344.com:3344/master/config-dev.yml

可以看到实时变更，但是访问http://localhost:3355/info

却发现没有实时变更。难道每次更新配置都要重启3355 client？

解决办法：

查看3355pom文件 是否有actuator 和web 没则加上。

修改yml，暴露监控端口，用来刷新，添加内容如下

```
management:
  endpoints:
    web:
      exposure:
        include: "*"
```

在controller上添加注解@RefreshScope，具备刷新能力。

再测：启动，将dev版本改为3，访问http://localhost:3355/info，还是没有刷新。

注意，此时3344client还不能刷新，需要（运维人员）发送Post请求刷新3355。

curl -X POST "http://localhost:3355/actuator/refresh"

重新访问http://localhost:3355/info

问题：加入有多个微服务client，每个微服务都要执行一次post请求刷新？可否广播，一次通知，处处生效？

Bus引入！！！

# 消息总线

## Bus

bus支持两种消息代理：RabbitMQ和Kafka。

Bus是用来将分布式系统过的节点与轻量级消息系统链接起来的框架，它整合了java的事件处理机制和消息中间件的功能，Bus配合Config使用可以实现配置的动态刷新。

什么是总线：在微服务架构的系统中，通常会使用轻量级的消息代理来构建一个共用的消息主题，并让系统中所有的微服务实例都连接上来，由于该主题中产生的消息会被所有实例监听和消费，所以称它为消息总线。在总线上的各个实例，都可以方便地广播一些需要让其他连接在该主题上地实例idou指导地消息。

基本原理：ConfigClient实例都监听MQ中同一个topic（默认是springcloud bus)。当一个服务刷新数据的时候，它会把这个信息放入到topic中，这样其他监听同一topic的服务就嫩刚得到通知，然后去更新自身的配置。

这里使用的是rabbitmq

新建子模块cloud-config-client-3366

```
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-config</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
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
  port: 3366

spring:
  application:
    name: config-client
  cloud:
    #config客户端配置
    config:
      label: master   #分支名称
      name: config    #配置文件名称
      profile: dev    #读取后缀名称
      uri: http://config3344.com:3344
      #综上 master分支上的config-dev.yml被读取，即访问http://config3344.com:3344/master/config-dev.yml
eureka:
  client:
    service-url:
      defaultZone: http://eureka7001.com:7001/eureka

management:
  endpoints:
    web:
      exposure:
        include: "*"
```

```
@EnableEurekaClient
@SpringBootApplication
public class ConfigClientMain3366 {
   public static void main(String[] args) {
      SpringApplication.run(ConfigClientMain3366.class, args);
   }
}
```

controller

```
@RestController
@RefreshScope
public class ConfigClientController {
   @Value("${server.port}")
   private String serverPort;
   @Value("${config.info}")
   private String configInfo;
   @GetMapping("/info")
   public String info(){
      return "serverPort:"+serverPort+"\n configInfo:"+configInfo;
   }
}
```

**动态刷新全局广播方式**

设计思想

1.利用消息总线触发一个客户端/bus/refresh，从而刷新所有客户端的配置；

2.利用消息总线触发一个服务端ConfigServer的/bus/refresh端点，从而刷新所有客户端的配置。

第二种方式更加合适，合理。第一种方式打破了微服务的职责单一性，因为微服务本身是业务模块，它本不应该承担刷新配置的职责；破坏了微服务各节点的对等性；有一定局限性。

**配置**

1.给cloud-config-center-3344配置中心服务端添加消息总线支持

添加依赖，注意是否少了actuator，如果没有要补上，下同

```
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-bus-amqp</artifactId>
</dependency>
```

```
server:
  port: 3344

spring:
  application:
    name: cloud-config-center
  cloud:
    config:
      server:
        git:
          uri: https://gitee.com/li-zhuofan-350562/springcloud-config.git
          search-paths:
            - springcloud-config
      label: master
#rabbitmq
 rabbitmq:
   host: localhost
   port: 5672
   username: guest
   password: guest
eureka:
  client:
    service-url:
      defaultZone: http://eureka7001.com:7001/eureka
#rabbitmq相关,暴露bus刷新配置的端点
management:
  endpoints:
    web:
      exposure:
        include: 'bus-refresh'
```

2.给cloud-config-client-3355客户端添加消息总线支持

```
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-bus-amqp</artifactId>
</dependency>
```

```
server:
  port: 3355
spring:
  application:
    name: config-client
  cloud:
    #config客户端配置
    config:
      label: master   #分支名称
      name: config    #配置文件名称
      profile: dev    #读取后缀名称
      uri: http://config3344.com:3344
      #综上 master分支上的config-dev.yml被读取，即访问http://config3344.com:3344/master/config-dev.yml
  rabbitmq:
    host: localhost
    port: 5672
    username: guest
    password: guest
eureka:
  client:
    service-url:
      defaultZone: http://eureka7001.com:7001/eureka
management:
  endpoints:
    web:
      exposure:
        include: "*"
```

3.给cloud-config-client-3366客户端添加消息总线支持

与3355除了端口都一样

测试：顺序启动7001 3344  3355 3366

访问没修改的：

先从配置中心 3344 :http://config3344.com:3344/config-dev.yml

再从client 3355 3366：http://localhost:3355/info

http://localhost:3366/info

如果能正常访问后，对远程git仓库上面的配置文件版本+1

此时还没自动刷新，运维发送：

curl -X POST "http://localhost:3344/actuator/bus-refresh"

再访问

http://localhost:3355/info

http://localhost:3366/info

看看是否都进行了变更。

在rabbitmq监控系统的exchanges表格中可以看到类型为topic的springCloudBus,与前面所讲的理论相符。

### 动态刷新定点通知

问题：我不想通知所有微服务，我只想通知某一微服务如3355并不通知3366

公式：

```
http://localhost:配置中心端口号/actuator/bus-refresh/{destination}
{destination}表示 微服务名：端口
```

如 curl -X POST "http://localhost:3344/actuator/bus-refresh/config-client:3355"

测试：修改版本+1

先从配置中心 3344 :http://config3344.com:3344/config-dev.yml

再从client 3355 3366：http://localhost:3355/info

http://localhost:3366/info

可以看到3366没更新 3355更新了。

这也是每个微服务写应用名称的好处。

# 消息驱动

为什么引入cloud stream？

消息中间件有ActiveMQ,RabbitMQ,RocketMQ,Kafka，学习困难。需要一种新的技术，让人们不再关注mq的细节，只需要一种适配绑定的方式，自动的给我们在各种MQ内切换。

## Stream

是一个构建消息驱动微服务的架构。屏蔽底层消息中间件的差异，降低切换成本，统一消息的编程模型。

应用程序通过inputs或者outputs来与spring cloud stream中的binder对象交互。通过我们配置来binding绑定，而springcloudstream的binder对象负责与消息中间件交互，所以我们只需要搞清楚如何与stream交互就可以方便的使用消息驱动的方式。input对应消费者output对应生产者。

通过定义绑定器作为中间件，完美地实现了应用程序与消息中间件细节之间的解耦隔离。

通过使用spring integration来连接消息代理中间件以实现消息事件驱动。

stream为一些供应商的消息中间件产品提供了个性化的自动化配置实现，引用了发布-订阅、消费组、分区的三个核心概念。

目前仅支持RabbitMQ、Kafka

stream中的消息通信方式遵循了发布-订阅模式，topic主题进行广播，在rabbitmq就是exchange，在kafka中就是topic。

常用注解：

@Input:注解标识输入通道，通过该输入通道接收到的消息进入应用程序。

@Output:注解标识输出通道，发布的消息将通过该通道离开应用程序。

@StreamListener:监听队列，用于消费者的队列和消息接收。

@EnableBinding:指信道channel和exchange绑定在一起。

**provider**

新建子模块cloud-stream-rabbitmq-provider8801作为生产者进行发消息模块

```
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-stream-rabbit</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
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
  port: 8801

spring:
  application:
    name: cloud-stream-provider
  cloud:
    stream:
      binders:   #配置要绑定的rabbitmq的服务信息
        defaultRabbit: #标识定义的名称，用于binding整合
         type: rabbit #消息组件类型
         environment: #设置rabbitmq的相关环境配置
           spring:
             rabbitmq:
               host: localhost
               port: 5672
               username: guest
               password: guest

      bindings:  #服务的整合处理
        output:  #这个名字是一个通道的名字
          destination: studyExchange #表示要使用的exchange名称定义
          content-type: application/json #设置消息类型，本次为json，如果文本则为 text/plain
          binder: defaultRabbit #设置要绑定的消息服务的具体设置   会爆红 不用管

eureka:
  client:
    service-url: 
      defaultZone: http://eureka7001.com:7001/eureka
  instance:
    lease-renewal-interval-in-seconds: 2       #心跳的时间间隔 默认30
    lease-expiration-duration-in-seconds: 5  #如果现在超过了5秒的间隔 默认90
    instance-id: send-8801.com
    prefer-ip-address: true
```

```
@SpringBootApplication
public class StreamMQMain8801 {
   public static void main(String[] args) {
      SpringApplication.run(StreamMQMain8801.class, args);
   }
}
```

```
public interface IMessageProvider {
   String send();
}
```

```
@EnableBinding(Source.class)   //定义消息的推送管道  源
public class MessageProviderImpl implements IMessageProvider {
   @Resource
   private MessageChannel output;  //消息发送管道
   @Override
   public String send() {
      String serial = UUID.randomUUID().toString();
      output.send(MessageBuilder.withPayload(serial).build());
      System.out.println("*********serial:"+serial);
      return null;
   }
}
```

```
@RestController
public class SendMessageController {
   @Resource
   private IMessageProvider messageProvider;
   @GetMapping(value = "/sendMessage")
   public String sendMessage(){
      return messageProvider.send();
   }
}
```

测试：启动7001、rabbitmq、8801 

访问http://localhost:15672/ rabbit监控 exchange中可以看到名称studyExchange，

来源于yml的destination: studyExchange #表示要使用的exchange名称定义

多次访问http://localhost:8801/sendMessage

查看8801终端输出的uuid，再看rabbit监控中的overview

**consumer**

新建子模块cloud-stream-rabbitmq-consumer8802作为消息接收模块

```
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-stream-rabbit</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
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
  port: 8802

spring:
  application:
    name: cloud-stream-consumer
  cloud:
    stream:
      binders:   #配置要绑定的rabbitmq的服务信息
        defaultRabbit: #标识定义的名称，用于binding整合
          type: rabbit #消息组件类型
          environment: #设置rabbitmq的相关环境配置
            spring:
              rabbitmq:
                host: localhost
                port: 5672
                username: guest
                password: guest

      bindings:  #服务的整合处理
        input:  #这个名字是一个通道的名字
          destination: studyExchange #表示要使用的exchange名称定义 关键点
          content-type: application/json #设置消息类型，本次为json，如果文本则为 text/plain
          binder: defaultRabbit #设置要绑定的消息服务的具体设置

eureka:
  client:
    service-url:
      defaultZone: http://eureka7001.com:7001/eureka
  instance:
    lease-renewal-interval-in-seconds: 2       #心跳的时间间隔 默认30
    lease-expiration-duration-in-seconds: 5  #如果现在超过了5秒的间隔 默认90
    instance-id: receive-8802.com
    prefer-ip-address: true
```

```
@SpringBootApplication
public class StreamMQMain8802 {
   public static void main(String[] args) {
      SpringApplication.run(StreamMQMain8802.class, args);
   }
}
```

```
@Component
@EnableBinding(Sink.class)
public class ReceiveMessageListener {
   @Value("${server.port}")
   private String serverPort;

   @StreamListener(Sink.INPUT)
   public void input(Message<String> message){
      System.out.println("消费者1号 ------->接受到的消息："+message.getPayload()+"\t port:"+serverPort);
   }
}
```

测试8801发送8802接收

启动7001 8801 8802

http://localhost:8801/sendMessage

查看8801和8802的终端输出并对比

**如果再加入一个consumer**

新建cloud-stream-rabbitmq-consumer8803作为消息接收模块

与8802，修改端口和主启动类名字和微服务实例名以及监听输出即可

```
@Component
@EnableBinding(Sink.class)
public class ReceiveMessageListener {
   @Value("${server.port}")
   private String serverPort;
   @StreamListener(Sink.INPUT)
   public void input(Message<String> message){
      System.out.println("消费者2号 ------->接受到的消息："+message.getPayload()+"\t port:"+serverPort);
   }
}
```

测试8801发送8802 8803接收

启动7001 8801 8802 8803

http://localhost:8801/sendMessage

查看8801和8802 8803的终端输出并对比.

存在问题：**重复消费**

### 分组消费

上述2个消费者存在 重复消费问题，即一个消息被两个消费者处理了。

在实际生产中如果同一订单被两个服务获取到，那么就会造成数据错误，我们得避免这种情况。

这时我们可以使用stream中的消息分组来解决，在stream中处于同一个group中的多个消费者是竞争关系，就能够保证消息只会被其中一个应用消费一次。不同组可以重复消费。

rabbitmq默认会为一个exchange里的不同组分配不同的组流水号（可以指定组名）。点击exchange名字可以看到。在上述的情况下可以看到8802 8803在不同的组，随机的不同组流水号。

实现分组：

不同组的情况：

在8802和8803的input下分别添加group: group01和group: group02。

如

```
bindings:  #服务的整合处理
  input:  #这个名字是一个通道的名字
    destination: studyExchange #表示要使用的exchange名称定义 关键点
    content-type: application/json #设置消息类型，本次为json，如果文本则为 text/plain
    binder: defaultRabbit #设置要绑定的消息服务的具体设置
    group: group01
```

测试 启动7001 8801 8802 8803 

在rabbitmq中可以看到 组名已变成了自定义的，不再是流水号。

并且如果访问http://localhost:8801/sendMessage ，因为是不同组 ，所以8802和8803都会输出消费。

同组的情况（实现分组消费）：

在8802和8803的input下分别添加group: group01和group: group01。

在rabbitmq中点击组名group01可以看到consumer数量是2.

多次访问http://localhost:8801/sendMessage

8802和8803 轮询消费消息。

**持久化**

上述后面解决了重复消费问题。

问题，如果上述最后的情况停止8802和8803并去掉8802的yml中的group配置，多次访问http://localhost:8801/sendMessage.

再启动8802，因为无分组属性配置，后台没有打印信息；

再启动8803，因为有分组属性配置，后台打印出来前面mq上的信息。

在rabbitmq中可以看到，rabbitmq为8802自动生成了一个分组。

# 分布式请求链路跟踪

## Sleuth+zipkin

在微服务框架中，一个由客户端发起的请求在后端系统中会经过多个不同的服务节点调用来协同产生最后的请求结果，每一个前端请求都会形成一条复杂的分布式服务调用链路，链路中的任何一环出现高延迟或错误都会引起整个请求最后的失败。

Spring Cloud Sleuth提供了一套完整的服务跟踪解决方案，并且兼容支持了zipkin。

一条链路通过Trace Id唯一标识， Span标识发起的请求信息，各Span通过parent Id 关联起来。

Trace类似于树结构的span集合，标识一条调用链路，存在唯一标识；

span标识调用链路来源，通俗的理解span就是一次请求信息。

**zipkin**

cloud从F版开始已经不需要自己构建zipkin server了 只需调用jar包即可

https://search.maven.org/remote_content?g=io.zipkin.java&a=zipkin-server&v=LATEST&c=exec

下载zipkin-server-2.12.9-exec.jar，

java -jar zipkin-server-2.12.9-exec.jar 运行。 可能报错，不影响。也可以使用docker。

访问http://localhost:9411/zipkin/

**provider**

修改cloud-provider-payment8001

```
<!--        包含了sleuth+zipkin-->
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-zipkin</artifactId>
        </dependency>
```

```
spring:
  application:
    name: cloud-payment-service  #微服务名称
  zipkin:
    base-url: http://localhost:9411
  sleuth:
    sampler:
      #采样值介于0-1之间，1表示全部采集
      probability: 1
```

controller

```
@GetMapping("zipkin")
public String paymentZipkin(){
   return "hi,I'am paymentZipkin server fall back*************";
}
```

**consumer**

修改cloud-consumer-order80

```
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-zipkin</artifactId>
</dependency>
```

```
spring:
  application:
    name: cloud-order-service
  zipkin:
    base-url: http://localhost:9411
  sleuth:
    sampler:
      #采样值介于0-1之间，1表示全部采集
      probability: 1
```

```
@GetMapping("consumer/payment/zipkin")
public String paymentZipkin(){
   String result = restTemplate.getForObject("http://localhost:8001/payment/zipkin", String.class);
   return result;
}
```

测试：启动7001 8001 80   

访问多次http://localhost/consumer/payment/zipkin

进入http://localhost:9411/zipkin/

选择consumer80的服务名 span名称all 时间1小时 查找

选择provider8001的服务名 span名称all 时间1小时 查找

点击查找到的内容可以看到调用层次链路。

