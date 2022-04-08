# Log

## SpringBoot

默认情况下，Spring Boot会用Logback来记录日志，并用INFO级别输出到控制台。

```
xxxxxxxxxx 2016-04-13 08:23:50.120  INFO 37397 --- [           main] org.hibernate.Version                    : HHH000412: Hibernate Core {4.3.11.Final}
```

输出内容元素具体如下：

- 时间日期 — 精确到毫秒
- 日志级别 — ERROR, WARN, INFO, DEBUG or TRACE
- 进程ID
- 分隔符 — `---` 标识实际日志的开始
- 线程名 — 方括号括起来（可能会截断控制台输出）
- Logger名 — 通常使用源代码的类名
- 日志内容

**TRACE** < **DEBUG** < **INFO** < **WARN** < **ERROR** < **FATAL**

配置日志级别,默认全局为info级别,特定包下为debug级别

```
logging:
  level:
    root:info
    com.example.xxx1:debug
    com.example.xxx2:debug
```

**多彩输出**

如果你的终端支持ANSI，设置彩色输出会让日志更具可读性。通过在`application.properties`中设置`spring.output.ansi.enabled`参数来支持。

- NEVER：禁用ANSI-colored输出（默认项）
- DETECT：会检查终端是否支持ANSI，是的话就采用彩色输出（推荐项）
- ALWAYS：总是使用ANSI-colored格式输出，若终端不支持的时候，会有很多干扰信息，不推荐使用

**文件输出**

Spring Boot默认配置只会输出到控制台，并不会记录到文件中，但是我们通常生产环境使用时都需要以文件方式记录。

若要增加文件输出，需要在`application.properties`中配置`logging.file`或`logging.path`属性。

- logging.file，设置文件，可以是绝对路径，也可以是相对路径。如：`logging.file=my.log`
- logging.path，设置目录，会在该目录下创建spring.log文件，并写入日志内容，如：`logging.path=/var/log`

> 日志文件会在10Mb大小的时候被截断，产生新的日志文件，默认级别为：ERROR、WARN、INFO

> 日志文件以天为单位, 到第二天时前一天的日志自动打包成gz

**自定义输出格式**

在Spring Boot中可以通过在`application.properties`配置如下参数控制输出格式：

- logging.pattern.console：定义输出到控制台的样式（不支持JDK Logger）
- logging.pattern.file：定义输出到文件的样式（不支持JDK Logger）

## 自定义日志配置

由于日志服务一般都在ApplicationContext创建前就初始化了，它并不是必须通过Spring的配置文件控制。因此通过系统属性和传统的Spring Boot外部配置文件依然可以很好的支持日志控制和管理。

根据不同的日志系统，你可以按如下规则组织配置文件名，就能被正确加载：

- Logback：`logback-spring.xml`, `logback-spring.groovy`, `logback.xml`, `logback.groovy`
- Log4j：`log4j-spring.properties`, `log4j-spring.xml`, `log4j.properties`, `log4j.xml`
- Log4j2：`log4j2-spring.xml`, `log4j2.xml`
- JDK (Java Util Logging)：`logging.properties`

**Spring Boot官方推荐优先使用带有`-spring`的文件名作为你的日志配置（如使用`logback-spring.xml`，而不是`logback.xml`）**

## Logback

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE xml>
<configuration  scan="true" scanPeriod="60 seconds" debug="false">
    <contextName>logback</contextName>
    <property name="log.path" value="log" />
    <!--输出到控制台-->
    <appender name="console" class="ch.qos.logback.core.ConsoleAppender">
       <!-- <filter class="ch.qos.logback.classic.filter.ThresholdFilter">
            <level>ERROR</level>
        </filter>-->
        <encoder>
            <pattern>%d{HH:mm:ss.SSS} %contextName [%thread] %-5level %logger{36} - %msg%n</pattern>
        </encoder>
    </appender>

    <!--输出到文件-->
    <appender name="file" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
            <fileNamePattern>${log.path}/logback.%d{yyyy-MM-dd}.log</fileNamePattern>
            <maxHistory>30</maxHistory>
        	<totalSizeCap>1GB</totalSizeCap>
        </rollingPolicy>
        <encoder>
            <pattern>%d{HH:mm:ss.SSS} %contextName [%thread] %-5level %logger{36} - %msg%n</pattern>
        </encoder>
    </appender>

    <root level="info">
        <appender-ref ref="console" />
        <appender-ref ref="file" />
    </root>

    <!-- 包 -->
    <!-- 控制controller包下的所有类的日志的打印，没有指定打印级别，继承root的日志级别"info" -->
    <!-- 没有指定appender, 继承root的"console"和file"-->
    <logger name="com.mrbird.controller"/>
    <!-- 类的全路径 -->
    <!-- 指定了处理"warn"级别日志,不传递到上级(root)-->
    <logger name="com.mrbird.controller.LoginController" level="WARN" additivity="false">
        <!-- 指定了addpender -->
        <appender-ref ref="console"/>
    </logger>
</configuration>
```

**根节点`<configuration>`包含的属性**

- scan：当此属性设置为true时，配置文件如果发生改变，将会被重新加载，默认值为true。
- scanPeriod：设置监测配置文件是否有修改的时间间隔，如果没有给出时间单位，默认单位是毫秒。当scan为true时，此属性生效。默认的时间间隔为1分钟。
- debug：当此属性设置为true时，将打印出logback内部日志信息，实时查看logback运行状态。默认值为false。

**`<contextName>`设置上下文名称**

每个logger都关联到logger上下文，默认上下文名称为“default”。但可以使用设置成其他名字，用于区分不同应用程序的记录。一旦设置，不能修改,可以通过%contextName来打印日志上下文名称

**`<property>`设置变量值** 有name和value属性；其中name的值是变量的名称，value的值时变量定义的值。定义的值会被插入到logger上下文中。定义变量后，可以使“${}”来使用变量。

**`<appender>`格式化日志输出节点**，有name和class属性，class用来指定哪种输出策略，常用就是控制台输出策略和文件输出策略。

**`<encoder>`对日志进行编码**：

- `%d{HH: mm:ss.SSS}`——日志输出时间。
- `%thread`——输出日志的进程名字，这在Web应用以及异步任务处理中很有用。
- `%-5level`——日志级别，并且使用5个字符靠左对齐。
- `%logger{36}`——日志输出者的名字。
- `%msg`——日志消息。
- `%n`——平台的换行符

**`rollingPolicy`标签**：

- `<fileNamePattern>${log.path}/logback.%d{yyyy-MM-dd}.log</fileNamePattern>`定义了日志的切分方式——把每一天的日志归档到一个文件中；
- `<maxHistory>30</maxHistory>`表示只保留最近30天的日志，以防止日志填满整个磁盘空间。同理，可以使用`%d{yyyy-MM-dd_HH-mm}`来定义精确到分的日志切分方式；
- `<totalSizeCap>1GB</totalSizeCap>`用来指定日志文件的上限大小，例如设置为1GB的话，那么到了这个值，就会删除旧的日志。

**`<root>`指定最基础的日志输出级别**

只有一个level属性，用来设置打印级别，大小写无关：TRACE, DEBUG, INFO, WARN, ERROR, ALL 和 OFF，不能设置为INHERITED或者同义词NULL。

默认是DEBUG。可以包含零个或多个元素，标识这个appender将会添加到这个logger。

```xml
<root level="debug">
    <appender-ref ref="console" />
    <appender-ref ref="file" />
</root>
```

**`<logger>`设置某一个包或者具体的某一个类的日志打印级别、以及指定`<appender>`**。`<logger>`有一个name属性，一个可选的level和一个可选的addtivity属性。

- `name`：用来指定受此logger约束的某一个包或者具体的某一个类。
- `level`：用来设置打印级别，大小写无关：TRACE, DEBUG, INFO, WARN, ERROR, ALL 和 OFF，还有一个特俗值INHERITED或者同义词NULL，代表强制执行上级的级别。如果未设置此属性，那么当前logger将会继承上级的级别。
- `addtivity`：是否向上级logger传递打印信息。默认是true。

# log4j

## springboot

在创建Spring Boot工程时，我们引入了`spring-boot-starter`，其中包含了`spring-boot-starter-logging`，该依赖内容就是Spring Boot默认的日志框架Logback，所以我们在引入log4j之前，需要先排除该包的依赖，再引入log4j的依赖，就像下面这样：

```xml

<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter</artifactId>
    <exclusions>
        <exclusion> 
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-logging</artifactId>
        </exclusion>
    </exclusions>
</dependency>

<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-log4j</artifactId>
</dependency>
```

```xml
<dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
            <!-- 排除默认的logback日志，使用log4j-->
            <exclusions>
                <exclusion>
                    <groupId>org.springframework.boot</groupId>
                    <artifactId>spring-boot-starter-logging</artifactId>
                </exclusion>
                <exclusion>
                    <groupId>org.slf4j</groupId>
                    <artifactId>slf4j-log4j12</artifactId>
                </exclusion>
                <exclusion>
                    <groupId>ch.qos.logback</groupId>
                    <artifactId>logback-access</artifactId>
                </exclusion>
            </exclusions>
        </dependency>
```

#### 控制台输出

通过如下配置，设定root日志的输出级别为INFO，appender为控制台输出stdout

```
# LOG4J配置
log4j.rootCategory=INFO, stdout

# 控制台输出
log4j.appender.stdout=org.apache.log4j.ConsoleAppender
log4j.appender.stdout.layout=org.apache.log4j.PatternLayout
log4j.appender.stdout.layout.ConversionPattern=%d{yyyy-MM-dd HH:mm:ss,SSS} %5p %c{1}:%L - %m%n
```

#### 输出到文件

在开发环境，我们只是输出到控制台没有问题，但是到了生产或测试环境，或许持久化日志内容，方便追溯问题原因。可以通过添加如下的appender内容，按天输出到不同的文件中去，同时还需要为`log4j.rootCategory`添加名为file的appender，这样root日志就可以输出到`logs/all.log`文件中了。

```
#
log4j.rootCategory=INFO, stdout, file

# root日志输出
log4j.appender.file=org.apache.log4j.DailyRollingFileAppender
log4j.appender.file.file=logs/all.log
log4j.appender.file.DatePattern='.'yyyy-MM-dd
log4j.appender.file.layout=org.apache.log4j.PatternLayout
log4j.appender.file.layout.ConversionPattern=%d{yyyy-MM-dd HH:mm:ss,SSS} %5p %c{1}:%L - %m%n
```

#### 分类输出

当我们日志量较多的时候，查找问题会非常困难，常用的手段就是对日志进行分类，比如：

- 可以按不同package进行输出。通过定义输出到`logs/my.log`的appender，并对`com.didispace`包下的日志级别设定为DEBUG级别、appender设置为输出到`logs/my.log`的名为`didifile`的appender。

```
# com.didispace包下的日志配置
log4j.category.com.didispace=DEBUG, didifile

# com.didispace下的日志输出
log4j.appender.didifile=org.apache.log4j.DailyRollingFileAppender
log4j.appender.didifile.file=logs/my.log
log4j.appender.didifile.DatePattern='.'yyyy-MM-dd
log4j.appender.didifile.layout=org.apache.log4j.PatternLayout
log4j.appender.didifile.layout.ConversionPattern=%d{yyyy-MM-dd HH:mm:ss,SSS} %5p %c{1}:%L ---- %m%n
```

- 可以对不同级别进行分类，比如对ERROR级别输出到特定的日志文件中，具体配置可以如下。

```
log4j.logger.error=errorfile
# error日志输出
log4j.appender.errorfile=org.apache.log4j.DailyRollingFileAppender
log4j.appender.errorfile.file=logs/error.log
log4j.appender.errorfile.DatePattern='.'yyyy-MM-dd
log4j.appender.errorfile.Threshold = ERROR
log4j.appender.errorfile.layout=org.apache.log4j.PatternLayout
log4j.appender.errorfile.layout.ConversionPattern=%d{yyyy-MM-dd HH:mm:ss,SSS} %5p %c{1}:%L - %m%n
```

## 多环境日志级别控制

- 创建多环境配置文件
  - `application-dev.properties`：开发环境
  - `application-test.properties`：测试环境
  - `application-prod.properties`：生产环境
- `application.properties`中添加属性：`spring.profiles.active=dev`（默认激活`application-dev.properties`配置）
- `application-dev.properties`和`application-test.properties`配置文件中添加日志级别定义：`logging.level.com.didispace=DEBUG`
- `application-prod.properties`配置文件中添加日志级别定义：`logging.level.com.didispace=INFO`

```

# LOG4J配置
log4j.category.com.didispace=${logging.level.com.didispace}, didifile

# com.didispace下的日志输出
log4j.appender.didifile=org.apache.log4j.DailyRollingFileAppender
log4j.appender.didifile.file=logs/my.log
log4j.appender.didifile.DatePattern='.'yyyy-MM-dd
log4j.appender.didifile.layout=org.apache.log4j.PatternLayout
log4j.appender.didifile.layout.ConversionPattern=%d{yyyy-MM-dd HH:mm:ss,SSS} %5p %c{1}:%L ---- %m%n
```

```
#
java -jar xxx.jar --spring.profiles.active=prod
```

## aop统一配置日志案例

web环境前提准备

```
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>

@RestController
public class HelloController {

    @RequestMapping(value = "/hello", method = RequestMethod.GET)
    @ResponseBody
    public String hello(@RequestParam String name) {
        return "Hello " + name;
    }

}
```

```
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-aop</artifactId>
</dependency>
```

在完成了引入AOP依赖包后，一般来说并不需要去做其他配置。也许在Spring中使用过注解配置方式的人会问是否需要在程序主类中增加`@EnableAspectJAutoProxy`来启用，实际并不需要。

可以看下面关于AOP的默认配置属性，其中`spring.aop.auto`属性默认是开启的，也就是说只要引入了AOP依赖后，默认已经增加了`@EnableAspectJAutoProxy`。

```
# AOP
spring.aop.auto=true # Add @EnableAspectJAutoProxy.
spring.aop.proxy-target-class=false # Whether subclass-based (CGLIB) proxies are to be created (true) as
 opposed to standard Java interface-based proxies (false)
```

**注意:(spring5 ,springboot2以前的标准)**

我们需要使用CGLIB来实现AOP的时候，需要配置`spring.aop.proxy-target-class=true`，不然默认使用的是标准Java的实现(jdk代理只能只针对实现了接口的类).

在spring5,如果类实现了接口,则使用jdk代理实现,否则使用CGLIB

在springboot2开始 默认使用CGLIB实现aop

如果使用springboot1,对没有实现接口的controller aop的话默认会报错

### 实现Web层的日志切面

实现AOP的切面主要有以下几个要素：

- 使用`@Aspect`注解将一个java类定义为切面类

- 使用`@Pointcut`定义一个切入点，可以是一个规则表达式，比如下例中某个package下的所有函数，也可以是一个注解等。

- 根据需要在切入点不同位置的切入内容

  ```
  使用`@Before`在切入点开始处切入内容
  
  使用`@After`在切入点结尾处切入内容
  
  使用`@AfterReturning`在切入点return内容之后切入内容（可以用来对处理返回值做一些加工处理）
  
  使用`@Around`在切入点前后切入内容，并自己控制何时执行切入点自身的内容
  
  使用`@AfterThrowing`用来处理当切入内容部分抛出异常之后的处理逻辑
  ```

  

  ```
  @Aspect
  @Component
  public class WebLogAspect {
  
      private Logger logger = Logger.getLogger(getClass());
  
      @Pointcut("execution(public * com.didispace.web..*.*(..))")
      public void webLog(){}
  
      @Before("webLog()")
      public void doBefore(JoinPoint joinPoint) throws Throwable {
          // 接收到请求，记录请求内容
          ServletRequestAttributes attributes = (ServletRequestAttributes) RequestContextHolder.getRequestAttributes();
          HttpServletRequest request = attributes.getRequest();
  
          // 记录下请求内容
          logger.info("URL : " + request.getRequestURL().toString());
          logger.info("HTTP_METHOD : " + request.getMethod());
          logger.info("IP : " + request.getRemoteAddr());
          logger.info("CLASS_METHOD : " + joinPoint.getSignature().getDeclaringTypeName() + "." + joinPoint.getSignature().getName());
          logger.info("ARGS : " + Arrays.toString(joinPoint.getArgs()));
  
      }
  
      @AfterReturning(returning = "ret", pointcut = "webLog()")
      public void doAfterReturning(Object ret) throws Throwable {
          // 处理完请求，返回内容
          logger.info("RESPONSE : " + ret);
      }
  
  }
  ```

### 拓展

#### aop切面同步问题(统计处理时间)

在WebLogAspect切面中，分别通过doBefore和doAfterReturning两个独立函数实现了切点头部和切点返回后执行的内容，若我们想统计请求的处理时间，就需要在doBefore处记录时间，并在doAfterReturning处通过当前时间与开始处记录的时间计算得到请求处理的消耗时间。

那么我们是否可以在WebLogAspect切面中定义一个成员变量来给doBefore和doAfterReturning一起访问呢？是否会有同步问题呢？

的确，直接在这里定义基本类型会有同步问题，所以我们可以引入ThreadLocal对象，像下面这样进行记录

```

@Aspect
@Component
public class WebLogAspect {

    private Logger logger = Logger.getLogger(getClass());

    ThreadLocal<Long> startTime = new ThreadLocal<>();

    @Pointcut("execution(public * com.didispace.web..*.*(..))")
    public void webLog(){}

    @Before("webLog()")
    public void doBefore(JoinPoint joinPoint) throws Throwable {
        startTime.set(System.currentTimeMillis());
        // 省略日志记录内容
    }

    @AfterReturning(returning = "ret", pointcut = "webLog()")
    public void doAfterReturning(Object ret) throws Throwable {
        // 处理完请求，返回内容
        logger.info("RESPONSE : " + ret);
        logger.info("SPEND TIME : " + (System.currentTimeMillis() - startTime.get()));
    }
}
```

#### AOP切面的优先级

由于通过AOP实现，程序得到了很好的解耦，但是也会带来一些问题，比如：我们可能会对Web层做多个切面，校验用户，校验头信息等等，这个时候经常会碰到切面的处理顺序问题。

所以，我们需要定义每个切面的优先级，我们需要`@Order(i)`注解来标识切面的优先级。**i的值越小，优先级越高**。

所以我们可以这样子总结：

- 在切入点前的操作，按order的值由小到大执行
- 在切入点后的操作，按order的值由大到小执行

## 日志输出到mongodb

思路:log4j提供的输出器实现自Appender接口，要自定义appender输出到MongoDB，只需要继承AppenderSkeleton类，并实现几个方法即可完成。

```
<dependency>
    <groupId>org.mongodb</groupId>
    <artifactId>mongodb-driver</artifactId>
    <version>3.2.2</version>
</dependency>
```

实现appender

```
public class MongoAppender  extends AppenderSkeleton {
    private MongoClient mongoClient;
    private MongoDatabase mongoDatabase;
    private MongoCollection<BasicDBObject> logsCollection;
    private String connectionUrl;
    private String databaseName;
    private String collectionName;

    @Override
    protected void append(LoggingEvent loggingEvent) {
        if(mongoDatabase == null) {
            MongoClientURI connectionString = new MongoClientURI(connectionUrl);
            mongoClient = new MongoClient(connectionString);
            mongoDatabase = mongoClient.getDatabase(databaseName);
            logsCollection = mongoDatabase.getCollection(collectionName, BasicDBObject.class);
        }
        logsCollection.insertOne((BasicDBObject) loggingEvent.getMessage());

    }
    @Override
    public void close() {
        if(mongoClient != null) {
            mongoClient.close();
        }
    }
    @Override
    public boolean requiresLayout() {
        return false;
    }
    // 省略getter和setter
}
```

配置文件

```
log4j.logger.mongodb=INFO, mongodb
# mongodb输出
log4j.appender.mongodb=com.didispace.log.MongoAppender
log4j.appender.mongodb.connectionUrl=mongodb://localhost:27017
log4j.appender.mongodb.databaseName=logs
log4j.appender.mongodb.collectionName=logs_request
```

**使用**

注意: logger取名为mongodb

```
@Aspect
@Order(1)
@Component
public class WebLogAspect {
    private Logger logger = Logger.getLogger("mongodb");

    @Pointcut("execution(public * com.didispace.web..*.*(..))")
    public void webLog(){}

    @Before("webLog()")
    public void doBefore(JoinPoint joinPoint) throws Throwable {
        // 获取HttpServletRequest
        ServletRequestAttributes attributes = (ServletRequestAttributes) RequestContextHolder.getRequestAttributes();
        HttpServletRequest request = attributes.getRequest();
        // 获取要记录的日志内容
        BasicDBObject logInfo = getBasicDBObject(request, joinPoint);
        logger.info(logInfo);
    }

    private BasicDBObject getBasicDBObject(HttpServletRequest request, JoinPoint joinPoint) {
        // 基本信息
        BasicDBObject r = new BasicDBObject();
        r.append("requestURL", request.getRequestURL().toString());
        r.append("requestURI", request.getRequestURI());
        r.append("queryString", request.getQueryString());
        r.append("remoteAddr", request.getRemoteAddr());
        r.append("remoteHost", request.getRemoteHost());
        r.append("remotePort", request.getRemotePort());
        r.append("localAddr", request.getLocalAddr());
        r.append("localName", request.getLocalName());
        r.append("method", request.getMethod());
        r.append("headers", getHeadersInfo(request));
        r.append("parameters", request.getParameterMap());
        r.append("classMethod", joinPoint.getSignature().getDeclaringTypeName() + "." + joinPoint.getSignature().getName());
        r.append("args", Arrays.toString(joinPoint.getArgs()));
        return r;
    }

    private Map<String, String> getHeadersInfo(HttpServletRequest request) {
        Map<String, String> map = new HashMap<>();
        Enumeration headerNames = request.getHeaderNames();
        while (headerNames.hasMoreElements()) {
            String key = (String) headerNames.nextElement();
            String value = request.getHeader(key);
            map.put(key, value);
        }
        return map;
    }
}
```

## actuator loggers端点动态修改日志级别

```
	<!--  spring boot actuator 监控端点-->
	<dependency>
		<groupId>org.springframework.boot</groupId>
		<artifactId>spring-boot-starter-actuator</artifactId>
	</dependency>
	<dependency>
		<groupId>org.springframework.boot</groupId>
		<artifactId>spring-boot-starter-web</artifactId>
	</dependency>
```

- 在应用主类中添加一个接口用来测试日志级别的变化，比如下面的实现：

```
@RestController
@SpringBootApplication
public class DemoApplication {
	private Logger logger = LoggerFactory.getLogger(getClass());

	@RequestMapping(value = "/test", method = RequestMethod.GET)
	public String testLogLevel() {
		logger.debug("Logger Level ：DEBUG");
		logger.info("Logger Level ：INFO");
		logger.error("Logger Level ：ERROR");
		return "";
	}

	public static void main(String[] args) {
		SpringApplication.run(DemoApplication.class, args);
	}
}
```

- 为了后续的试验顺利，在`application.properties`中增加一个配置，来关闭安全认证校验。

```
management.security.enabled=false
```

不然在访问`/loggers`端点的时候，会报如下错误：

```
{
  "timestamp": 1485873161065,
  "status": 401,
  "error": "Unauthorized",
  "message": "Full authentication is required to access this resource.",
  "path": "/loggers/com.didispace"
}
```

访问`/test`端点,可以看到输出的是INFO 和 ERROR级别日志 (INFO默认级别)

尝试通过`/logger`端点来将日志级别调整为`DEBUG`，比如，发送POST请求到`/loggers/com.didispace`端点，其中请求体Body内容为：

```
{
    "configuredLevel": "DEBUG"
}
```

重新访问`/test`端点,可以看到输出了包括DEBUG在内的所有日志.达到了动态修改日志级别的效果.

其他:查看当前日志级别

发送GET请求到`/loggers/com.didispace`

```
{
  "configuredLevel": "DEBUG",
  "effectiveLevel": "DEBUG"
}
```

不限定条件，直接通过GET请求访问`/loggers`来获取所有的日志级别设置

## 配置文件案例

```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration>
    <properties>
        <property name="CONSOLE_PATTERN">xxx</property>
        <property name="FILE_NAME">xxx</property>
    </properties>
    <!--先定义所有的appender-->
    <appenders>
        <!--这个输出控制台的配置-->
        <console name="Console" target="SYSTEM_OUT">
            <!--输出日志的格式-->
            <PatternLayout pattern="${CONSOLE_PATTERN}"/>
        </console>
        <!-- 这个会打印出所有的info及以下级别的信息，每次大小超过size，则这size大小的日志会自动存入按年份-月份建立的文件夹下面并进行压缩，作为存档-->
        <RollingFile name="RollingFileInfo" fileName="D:/log/toutiao/${FILE_NAME}.log"
                     filePattern="D:/log/toutiao/$${date:yyyy-MM}/${FILE_NAME}-%d{yyyy-MM-dd}-%i.log">
            <PatternLayout pattern="${CONSOLE_PATTERN}"/>
            <Policies>
                <TimeBasedTriggeringPolicy/>
                <SizeBasedTriggeringPolicy size="100 MB"/>
            </Policies>
        </RollingFile>
    </appenders>
    <loggers>
        <!--过滤掉spring和mybatis的一些无用的DEBUG信息-->
        <logger name="org.springframework" level="INFO"></logger>
        <logger name="org.mybatis" level="INFO"></logger>
        <logger name="org.apache.http" level="INFO"></logger>
        <logger name="org.apache.kafka" level="INFO"></logger>
        <logger name="com.netflix.discovery" level="INFO"></logger>
        <logger name="org.hibernate" level="INFO"></logger>
        <root level="${log.level}">
            <appender-ref ref="Console"/>
            <appender-ref ref="RollingFileInfo"/>
        </root>
    </loggers>
</configuration>
```

properties

```
 ### 设置###
log4j.rootLogger = debug,stdout,D,E
 
### 输出信息到控制抬 ###
log4j.appender.stdout = org.apache.log4j.ConsoleAppender
log4j.appender.stdout.Target = System.out
log4j.appender.stdout.layout = org.apache.log4j.PatternLayout
log4j.appender.stdout.layout.ConversionPattern = [%-5p] %d{yyyy-MM-dd HH:mm:ss,SSS} method:%l%n%m%n
 
### 输出DEBUG 级别以上的日志到=E://logs/error.log ###
log4j.appender.D = org.apache.log4j.DailyRollingFileAppender
log4j.appender.D.File = E://logs/log.log
log4j.appender.D.Append = true
log4j.appender.D.Threshold = DEBUG 
log4j.appender.D.layout = org.apache.log4j.PatternLayout
log4j.appender.D.layout.ConversionPattern = %-d{yyyy-MM-dd HH:mm:ss}  [ %t:%r ] - [ %p ]  %m%n
 
### 输出ERROR 级别以上的日志到=E://logs/error.log ###
log4j.appender.E = org.apache.log4j.DailyRollingFileAppender
log4j.appender.E.File =E://logs/error.log 
log4j.appender.E.Append = true
log4j.appender.E.Threshold = ERROR 
log4j.appender.E.layout = org.apache.log4j.PatternLayout
log4j.appender.E.layout.ConversionPattern = %-d{yyyy-MM-dd HH:mm:ss}  [ %t:%r ] - [ %p ]  %m%n

## 指定包下日志等级
log4j.logger.org.apache.shiro=INFO

log4j.logger.org.apache.shiro.util.ThreadContext=WARN
log4j.logger.org.apache.shiro.cache.ehcache.EhCache=WARN
```

