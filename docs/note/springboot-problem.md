# Spring

## MVC

### Request路径

```java
request.getRequestURL() http://localhost:8080/jqueryLearn/resources/request.jsp 
request.getRequestURI() /jqueryLearn/resources/request.jsp
request.getContextPath()/jqueryLearn 
request.getServletPath()/resources/request.jsp 
```

**获取请求完整路径**

```java
//包括：域名，端口，上下文访问路径
ServletRequestAttributes attributes = (ServletRequestAttributes)RequestContextHolder.getRequestAttributes();
HttpServletRequest request = attributes.getRequest();
StringBuffer url = request.getRequestURL();
String contextPath = request.getServletContext().getContextPath();
url.delete(url.length() - request.getRequestURI().length(), url.length()).append(contextPath).toString();
```

### 获取IP

```java
/**
 * 使用Nginx等反向代理软件， 则不能通过request.getRemoteAddr()获取IP地址
 * 如果使用了多级反向代理的话，X-Forwarded-For的值并不止一个，而是一串IP地址，X-Forwarded-For中第一个非unknown的有效IP字符串，则为真实IP地址
 */
public static String getIpAddr(HttpServletRequest request) {
		String ip = request.getHeader("x-forwarded-for");
		if (ip == null || ip.length() == 0 || "unknown".equalsIgnoreCase(ip)) {
			ip = request.getHeader("Proxy-Client-IP");
		}
		if (ip == null || ip.length() == 0 || "unknown".equalsIgnoreCase(ip)) {
			ip = request.getHeader("WL-Proxy-Client-IP");
		}
		if (ip == null || ip.length() == 0 || "unknown".equalsIgnoreCase(ip)) {
			ip = request.getRemoteAddr();
		}
		return "0:0:0:0:0:0:0:1".equals(ip) ? "127.0.0.1" : ip;
	}
```

### 是否为Ajax请求

```java
public static boolean isAjaxRequest(HttpServletRequest request){
    String accept = request.getHeader("accept");
    if (accept != null && accept.indexOf("application/json") != -1){
        return true;
    }
    String xRequestedWith = request.getHeader("X-Requested-With");
    if (xRequestedWith != null && xRequestedWith.indexOf("XMLHttpRequest") != -1){
        return true;
    }
    String uri = request.getRequestURI();
    if (StringUtils.inStringIgnoreCase(uri, ".json", ".xml")){
        return true;
    }
    String ajax = request.getParameter("__ajax");
    if (StringUtils.inStringIgnoreCase(ajax, "json", "xml")){
        return true;
    }
    return false;
} 
```

### 静态资源目录位置

在我们开发Web应用的时候，需要引用大量的js、css、图片等静态资源。Spring Boot默认提供静态资源目录位置需置于classpath下，目录名需符合如下规则：

```
classpath:/META-INF/resources/
classpath:/resources/ 
classpath:/static/
classpath:/public/
```

举例：我们可以在`src/main/resources/`目录下创建`static`，在该位置放置一个图片文件。启动程序后，尝试访问`http://localhost:8080/D.jpg`。如能显示图片，配置成功。

> 默认映射/**, 可修改为: `spring.mvc.static-path-pattern=/resources/`

通过`WebMvcConfigurer` 的 `addResourceHandlers` 方法可以修改此行为.

模板文件在

```
classpath:/resources/template/
```

### Request header is too large

**方向一： 配置应用服务器使其允许的最大值 > 你实用实用的请求头数据大小**

如果用Spring Boot的话，只需要在配置文件里配置这个参数即可：

```
server.max-http-header-size=
```

**方向二：规避请求头过大的情况**

虽然上面的配置可以在解决，但是如果无节制的使用header部分，那么这个参数就会变得不可控。

对于请求头部分的数据其实本身并不建议放太大的数据，所以，还是建议把这些数据放到body里更为合理。

### 错误页面ErrorPageRegistrar

```java
public class ErrorPageConfig implements ErrorPageRegistrar {
    @Override
    public void registerErrorPages(ErrorPageRegistry registry) {
        // 1.错误类型为404，默认显示404.html网页
        ErrorPage e404 = new ErrorPage(HttpStatus.NOT_FOUND, "/404.html");
        // 2.错误类型为500，表示服务器响应错误，默认显示/500.html网页
        registry.addErrorPages(e404);
    }
}
```



## 模板引擎

### Thymeleaf

```xml
<dependency>
	<groupId>org.springframework.boot</groupId>
	<artifactId>spring-boot-starter-thymeleaf</artifactId>
</dependency>
```

模板:src/main/resources/templates`下新建模板文件`index.html

```html
<!DOCTYPE html>
<html>
<head lang="en">
    <meta charset="UTF-8" />
    <title></title>
</head>
<body>
<h1 th:text="${name}">Hello World</h1>
</body>
</html>
```

```java
@Controller
public class HelloController {
    @GetMapping("/")
    public String index(ModelMap map) {
        map.addAttribute("name", "xxx");
        return "index";
    }
}
```

**配置参数**

```
# Enable template caching.  false可使每次修改页面不用重启
spring.thymeleaf.cache=true 
# Check that the templates location exists.
spring.thymeleaf.check-template-location=true 
# Content-Type value.
spring.thymeleaf.content-type=text/html 
# Enable MVC Thymeleaf view resolution.
spring.thymeleaf.enabled=true 
# Template encoding.
spring.thymeleaf.encoding=UTF-8 
# Comma-separated list of view names that should be excluded from resolution.
spring.thymeleaf.excluded-view-names= 
# Template mode to be applied to templates. See also StandardTemplateModeHandlers.
spring.thymeleaf.mode=HTML5 
# Prefix that gets prepended to view names when building a URL.
spring.thymeleaf.prefix=classpath:/templates/ 
# Suffix that gets appended to view names when building a URL.
spring.thymeleaf.suffix=.html  spring.thymeleaf.template-resolver-order= # Order of the template resolver in the chain. spring.thymeleaf.view-names= # Comma-separated list of view names that can be resolved.
```

## 多环境配置

> 在命令行`--`可对`application.properties`属性值进行赋值的标识。

多环境配置文件名需要满足`application-{profile}.properties`的格式，其中`{profile}`对应环境标识，比如：

- `application-dev.properties`：开发环境
- `application-test.properties`：测试环境
- `application-prod.properties`：生产环境

至于哪个具体的配置文件会被加载，需要在`application.properties`文件中通过`spring.profiles.active`属性来设置，其值对应配置文件中的`{profile}`值

加入在单个文件中配置多环境:

2.4之前:

```
spring:
  profiles: "dev"
---
spring:
  profiles: "test"
---
spring:
  profiles: "prod"
```

2.4之后:`spring.profiles`配置用`spring.config.activate.on-profile`替代

```
spring:
  config:
    activate:
      on-profile: "dev"
---
spring:
  config:
    activate:
      on-profile: "test"
spring:
  config:
    activate:
      on-profile: "prod"
```

**分组配置:**  适合分别配置不同中间件

2.4之前:

```
spring:
  profiles:
    active: "dev"
---
spring.profiles: "dev"
spring.profiles.include: "dev-db,dev-mq"
---
spring.profiles: "dev-db"
db: dev-db.didispace.com
---
spring.profiles: "dev-mq"
mq: dev-mq.didispace.com
```

2.4之后

```
spring:
  profiles:
    active: "dev"
    group:
      "dev": "dev-db,dev-mq"
      "prod": "prod-db,prod-mq"
---
spring:
  config:
    activate:
      on-profile: "dev-db"

db: dev-db.didispace.com
---
spring:
  config:
    activate:
      on-profile: "dev-mq"

mq: dev-mq.didispace.com
---
spring:
  config:
    activate:
      on-profile: "prod-db"

db: prod-db.didispace.com
---
spring:
  config:
    activate:
      on-profile: "prod-mq"
mq: prod-mq.didispace.com
```

**激活当前配置的其他方式**(允许多个同时激活)

1.

```
AnnotationConfigApplicationContext ctx = new AnnotationConfigApplicationContext();
ctx.getEnvironment().setActiveProfiles("development");
```

2.

```
-Dspring.profiles.active="profile1,profile2"
```

## 异常处理机制

默认情况下, 当应用中产生异常时, Spring Boot根据发送请求头中的`accept`是否包含`text/html`来分别返回不同的响应信息.  当从浏览器地址栏中访问应用接口时，请求头中的`accept`便会包含`text/html`信息,  产生异常时,  通过`org.springframework.web.servlet.ModelAndView`对象来装载异常信息,  并以HTML的格式返回;  而当从客户端访问应用接口产生异常时  (客户端访问时，请求头中的`accept`不包含`text/html`),  则以JSON的格式返回异常信息: 

```java
/**
 *errorHtml和error方法的请求地址是一样的，但errorHtml通过produces = {"text/html"}判断请求头来执行
 */
@Controller
@RequestMapping({"${server.error.path:${error.path:/error}}"})
public class BasicErrorController extends AbstractErrorController {
   //...
    @RequestMapping(produces = {"text/html"})
    public ModelAndView errorHtml(HttpServletRequest request, HttpServletResponse response) {
        HttpStatus status = this.getStatus(request);
        Map<String, Object> model = Collections.unmodifiableMap(this.getErrorAttributes(request, this.isIncludeStackTrace(request, MediaType.TEXT_HTML)));
        response.setStatus(status.value());
        ModelAndView modelAndView = this.resolveErrorView(request, response, status, model);
        return modelAndView != null ? modelAndView : new ModelAndView("error", model);
    }

    @RequestMapping
    @ResponseBody
    public ResponseEntity<Map<String, Object>> error(HttpServletRequest request) {
        Map<String, Object> body = this.getErrorAttributes(request, this.isIncludeStackTrace(request, MediaType.ALL));
        HttpStatus status = this.getStatus(request);
        return new ResponseEntity(body, status);
    }
     //...
}
```

通过自定义错误静态页面可改变浏览器访问异常响应内容, 如:

添加resource/template/500.html、resource/public/error/404.html文件、resource/public/error/5xx.html文件.

或者通过创建advice进行全局异常捕获处理:

```java
@ControllerAdvice
public class ControllerExceptionHandler {
    @ExceptionHandler(UserNotExistException.class)
    @ResponseBody
    @ResponseStatus(HttpStatus.INTERNAL_SERVER_ERROR)
    public Map<String, Object> handleUserNotExistsException(UserNotExistException e) {
        Map<String, Object> map = new HashMap<>();
        map.put("id", e.getId());
        map.put("message", e.getMessage());
        return map;
    }
}
```

## war打包

```
<packaging>war</packaging>
```

然后添加如下的Tomcat依赖配置，覆盖Spring Boot自带的Tomcat依赖：

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-tomcat</artifactId>
    <scope>provided</scope>
</dependency>
```

在`<build></build>`标签内配置项目名：

```
<build>
    ...
    <finalName>demo</finalName>
</build>
```

添加启动类ServletInitializer：

```java
public class ServletInitializer extends SpringBootServletInitializer {
    @Override
    protected SpringApplicationBuilder configure(SpringApplicationBuilder application) {
        return application.sources(Application.class);
    }
}
```

其中Application为Spring Boot的启动类。

准备完毕后，运行`mvn clean package`命令即可在target目录下生产war包

## 跨域

**1.注解驱动**

Spring 4.2提供`@CrossOrigin`注解, 包含了以下属性:

| 属性             | 含义                                                         |
| :--------------- | :----------------------------------------------------------- |
| value            | 指定所支持域的集合，`*`表示所有域都支持，默认值为`*`。这些值对应HTTP请求头中的`Access-Control-Allow-Origin` |
| origins          | 同value                                                      |
| allowedHeaders   | 允许请求头中的header，默认都支持                             |
| exposedHeaders   | 响应头中允许访问的header，默认为空                           |
| methods          | 支持请求的方法，比如`GET`，`POST`，`PUT`等，默认和Controller中的方法上标注的一致。 |
| allowCredentials | 是否允许cookie随请求发送，使用时必须指定具体的域             |
| maxAge           | 预请求的结果的有效期，默认30分钟                             |

**2.接口编程**

```java
@Configuration
public class WebConfigurer implements WebMvcConfigurer {
    @Override
    public void addCorsMappings(CorsRegistry registry) {
        registry.addMapping("/**")
                .allowedOrigins("*")
                .allowedMethods("GET"); 
    }
}
```

**3.过滤器实现**

```java
@Bean
public FilterRegistrationBean corsFilter() {    
	UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();    
	CorsConfiguration config = new CorsConfiguration();    
	config.setAllowCredentials(true);    
	config.addAllowedOrigin("*");    
	source.registerCorsConfiguration("/**", config);    
	FilterRegistrationBean bean = new FilterRegistrationBean(new CorsFilter(source));   
    bean.setOrder(0);    
    return bean;
}
//或
@Bean
public CorsFilter corsFilter(){
    UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
    CorsConfiguration config = new CorsConfiguration();
    config.setAllowCredentials(true);
    config.addAllowedOrigin("*");
    config.addAllowedHeader("*");
    config.addAllowedMethod("*");
    source.registerCorsConfiguration("/**", config);
    return new CorsFilter(source);
    }
```

> 若CorsFilter与SpringSecurity的Fillter冲突, 在springSecurity中将corsFilter注册到所有security的功能Filter之前

## Devtool热部署

原理: 使用两个`ClassLoader`，一个`Classloader`加载那些不会改变的类（第三方Jar包），另一个`ClassLoader`加载会更改的类，称为`restart ClassLoader`，这样在有代码更改的时候，原来的`restart ClassLoader` 被丢弃，重新创建一个`restart ClassLoader`，由于需要加载的类相比较少，所以实现了较快的重启时间。

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-devtools</artifactId>
    <optional>true</optional>
</dependency>
```

devtools会监听classpath下的文件变动，并且会立即重启应用（发生在保存时机），因为其采用的虚拟机机制，该项重启是很快的。

所有可选配置

```properties
# Whether to enable a livereload.com-compatible server.
spring.devtools.livereload.enabled=true 

# Server port.
spring.devtools.livereload.port=35729 

# Additional patterns that should be excluded from triggering a full restart.
spring.devtools.restart.additional-exclude= 

# Additional paths to watch for changes.
spring.devtools.restart.additional-paths= 

# Whether to enable automatic restart.
spring.devtools.restart.enabled=true

# Patterns that should be excluded from triggering a full restart.
spring.devtools.restart.exclude=META-INF/maven/**,META-INF/resources/**,resources/**,static/**,public/**,templates/**,**/*Test.class,**/*Tests.class,git.properties,META-INF/build-info.properties

# Whether to log the condition evaluation delta upon restart.
spring.devtools.restart.log-condition-evaluation-delta=true 

# Amount of time to wait between polling for classpath changes.
spring.devtools.restart.poll-interval=1s 

# Amount of quiet time required without any classpath changes before a restart is triggered.
spring.devtools.restart.quiet-period=400ms 

# Name of a specific file that, when changed, triggers the restart check. If not specified, any classpath file change triggers the restart.
spring.devtools.restart.trigger-file=
```

## 系统公共属性

```java
Properties props = System.getProperties();
//系统名称 如Windows 10
props.getProperty("os.name");
//系统架构 如amd64
props.getProperty("os.arch");
//项目根路径
props.getProperty("user.dir");
//javahome
props.getProperty("java.home");
//java版本
props.getProperty("java.version");

//jvm名称
ManagementFactory.getRuntimeMXBean().getVmName();
//启动
ManagementFactory.getRuntimeMXBean().getStartTime()

//jvm运行内存情况  替代品oshi?
//jvm内存总量
Runtime.getRuntime().totalMemory();
//jvm可用内存
Runtime.getRuntime().freeMemory();
Runtime.getRuntime().maxMemory()
```

## 自定义属性警告问题

下面内容都与该依赖有关

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-configuration-processor</artifactId>
</dependency>
```

除了在IDEA中手动取消警告,更合适的方式是配置元数据.

在spring boot 项目autoconfigure依赖包下 ,可以看到一个名为additional-spring-configuration-metadata.json文件.

里面指定了属性配置的元数据信息,它可以帮助IDE来完成配置联想和配置提示的展示。

**配置元数据的自动生成**

```java
@Data
@Configuration
@ConfigurationProperties(prefix = "com.xxx")
public class DidiProperties {
    private String from;
}
```

mvn install

查看target.classes.META-INF下的spring-configuration-metadata.json文件,

并且再使用属性配置时,已经没有高亮警告了.

## Spring Boot CLI

Spring Boot CLI（Command Line Interface）是一个命令行工具，您可以用它来快速构建Spring原型应用。通过Spring Boot CLI，我们可以通过编写Groovy脚本来快速的构建出Spring Boot应用，并通过命令行的方式将其运行起来

**安装Spring Boot CLI**

**通用安装**:所有平台都可以使用的安装方法。

第一步：下载Spring Boot CLI的工具包：

- [spring-boot-cli-2.0.1.RELEASE-bin.zip](https://repo.spring.io/release/org/springframework/boot/spring-boot-cli/2.0.1.RELEASE/spring-boot-cli-2.0.1.RELEASE-bin.zip)
- [spring-boot-cli-2.0.1.RELEASE-bin.tar.gz](https://repo.spring.io/release/org/springframework/boot/spring-boot-cli/2.0.1.RELEASE/spring-boot-cli-2.0.1.RELEASE-bin.tar.gz)

第二步：解压下载内容，可以看到bin目录下已经有适用于windows和linux平台的两个可执行文件了，我们已经可以直接使用它；为了更方便的使用Spring Boot CLI的命令，我们可以将上面bin目录中对应的可执行文件加入到当前系统的环境变量即可。

**Mac OSX Brew安装**

在Mac OSX系统下面就非常方便了，我们可以通过Brew来进行安装，只需要分别执行下面的两条的命令即可：

```
$ brew tap pivotal/tap
$ brew install springboot
```

**验证安装**

不论使用哪种方法安装，在安装好之后，我们可以通过下面的命令来验证一下当前的安装结果：

```
$ spring --version
Spring CLI v2.0.0.RELEASE
```

**使用**

新建Groovy脚本 hello,groovy

```java
@RestController
class ThisWillActuallyRun {
    @RequestMapping("/")
    String home() {
        "Hello World!"
    }
}
```

使用`spring`命令运行该Groovy脚本

```
spring run hello.groovy
```

localhost:8080访问

## 不占用端口启动

```java
@SpringBootApplication
public class SpringBootDisableWebEnvironmentApplication {
    public static void main(String[] args) {
        new SpringApplicationBuilder(SpringBootDisableWebEnvironmentApplication .class)
            .web(WebApplicationType.NONE) // .REACTIVE, .SERVLET
            .run(args);
   }
}
```

## LocalateTime序列化

`LocalDate`、`LocalTime`、`LocalDateTime`是Java 8开始提供的时间日期API，主要用来优化Java 8以前对于时间日期的处理操作。在使用Spring Boot或使用Spring Cloud Feign的时候，如果请求参数或返回结果中有`LocalDate`、`LocalTime`、`LocalDateTime`会发生各种问题:

```java
@SpringBootApplication
public class Application {
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
    @RestController
    class HelloController {
        @PostMapping("/user")
        public UserDto user(@RequestBody UserDto userDto) throws Exception {
            return userDto;
        }
    }
    @Data
    @NoArgsConstructor
    @AllArgsConstructor
    static class UserDto {
        private String userName;
        private LocalDate birthday;
    }
}
```

调用这个接口的时候,会警告:

```
2018-03-13 09:22:58,445 WARN  [http-nio-9988-exec-3] org.springframework.web.servlet.mvc.support.DefaultHandlerExceptionResolver - Failed to read HTTP message: org.springframework.http.converter.HttpMessageNotReadableException: JSON parse error: Can not construct instance of java.time.LocalDate: no suitable constructor found, can not deserialize from Object value (missing default constructor or creator, or perhaps need to add/enable type information?); nested exception is com.fasterxml.jackson.databind.JsonMappingException: Can not construct instance of java.time.LocalDate: no suitable constructor found, can not deserialize from Object value (missing default constructor or creator, or perhaps need to add/enable type information?)
 at [Source: java.io.PushbackInputStream@67064c65; line: 1, column: 63] (through reference chain: java.util.ArrayList[0]->com.xxx.UserDto["birthday"])
```

发送请求体`{"userName":"Tom","birthday":"2000-01-20"}`, 查看返回内容发现birthday属性已经变化为:`{"userName":"Tom","birthday":[2000,1,20]}`.

即默认情况下Spring MVC对于`LocalDate`序列化成了一个数组类型.

**解决办法**

jackson为此提供了一整套的序列化方案

```xml
<dependency>
    <groupId>com.fasterxml.jackson.datatype</groupId>
    <artifactId>jackson-datatype-jsr310</artifactId>
</dependency>
```

在该模块中封装对Java 8的时间日期API序列化的实现.  其具体实现在这个类中: `com.fasterxml.jackson.datatype.jsr310.JavaTimeModule`（注意：一些较早版本封装在这个类中“`com.fasterxml.jackson.datatype.jsr310.JSR310Module`）. 在配置了依赖之后，我们只需要在上面的应用主类中增加这个序列化模块，并禁用对日期以时间戳方式输出的特性：

```java
@Bean
public ObjectMapper serializingObjectMapper() {
    ObjectMapper objectMapper = new ObjectMapper();
    objectMapper.disable(SerializationFeature.WRITE_DATES_AS_TIMESTAMPS);
    objectMapper.registerModule(new JavaTimeModule());
    return objectMapper;
}
```

## 找回请求路径列表日志

在Spring Boot 1.x中,对于Spring构建的Web应用在启动的时候，都会输出当前应用创建的HTTP接口列表.

这些日志接口信息是由`org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerMapping`类在启动的时候，通过扫描Spring MVC的`@Controller`、`@RequestMapping`等注解去发现应用提供的所有接口信息。然后在日志中打印，以方便开发者排查关于接口相关的启动是否正确。

从Spring Boot 2.1.0版本开始，就不再打印这些信息了，完整的启动日志变的非常少.

主要是由于从该版本开始，将这些日志的打印级别做了调整：从原来的`TRACE`调整为`INFO`。

增加对`RequestMappingHandlerMapping`类的打印级别设置

```
logging.level.org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerMapping=trace
```

在2.1.x版本之后，除了调整了日志级别之外，对于打印内容也做了调整。现在的打印内容根据接口创建的Controller类做了分类打印，这样更有助于开发者根据自己编写的Controller来查找初始化了那些HTTP接口

## 懒初始化

关于Spring Boot的性能问题是我们经常在内容平台上看到吐槽的关键词。这次在Spring Boot 2.2中，针对性能这一点，做了大幅的优化。应用程序的启动速度将变得更快，内存占用也会变得更少。

同时，为了加快应用的启动，还增加一个全局延迟初始化的配置参数`spring.main.lazy-initialization`，这可以让我们的应用更快的完成启动动作，但是值得注意的是，延迟启动也会有下面这些副作用：

- 应用在进行延迟初始化的时候，HTTP请求的处理会需要更长的时间
- 原本可能在启动期出现的错误，将延迟到启动的运行期间出现

> 还可以使用 SpringApplicationBuilder 上的 lazyInitialization 方法或 SpringApplication 上的 setLazyInitialization 方法以编程方式启用惰性初始化。

> 如果要禁用某些 Bean 的延迟初始化，同时对应用程序的其余部分使用延迟初始化，则可以使用 @Lazy（false） 注释将其 lazy 属性显式设置为 false。

## 冷门

>  @Inject 可代替 @Autowired,  `@Named` or `@ManagedBean`可代替@Component , 这三个注解均来自于JSR-330的**javax.inject**包

>依赖注入配置时,通过required=false属性 , @Nullable 和 java8的Optional方式, 可以达到不必须依赖的效果.
>
>同样,  这三种方式也可与MVC, required=false可与 @RequestParam、@RequestHeader等结合使用。

> JSR-330的@Singleton 相当于 @Scope("singleton") , 因为是默认, 所以无用

> 在spring配置类中, 可以用@ImportResource注解导入xml文件配置

>通过@ComponentScan的nameGenerator指定BeanNameGenerator可实现自定义 Bean 命名策略

> HttpEntity< T > 与@RequestBody功能相同:
>
> ```
> @PostMapping("/accounts")
> public void handle(HttpEntity<Account> entity) 
> 
> 等价于
> @PostMapping("/accounts")
> public void handle(@RequestBody Account account)
> ```
>
> 
