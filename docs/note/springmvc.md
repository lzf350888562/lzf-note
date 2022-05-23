# SpringMVC

## WebMvcConfigurer通用配置

或者继承WebMvcConfigurerAdapter抽象类

**跨域**

```java
@Configuration
public class CorsConfig implements WebMvcConfigurer {
    @Override
    public void addCorsMappings(CorsRegistry registry) {
        registry.addMapping("/**")
                .allowedOrigins("*")
                .allowedMethods("GET", "HEAD", "POST", "PUT", "DELETE", "OPTIONS")
                .allowCredentials(true)
                .maxAge(3600)
                .allowedHeaders("*");
    }
}
```

**资源处理器**

```java
public void addResourceHandlers(ResourceHandlerRegistry registry) {
        registry.addResourceHandler("/upload/**").addResourceLocations("file:" + Constants.FILE_UPLOAD_DIC).setCachePeriod(31556926);;
        registry.addResourceHandler("/goods-img/**").addResourceLocations("file:" + Constants.FILE_UPLOAD_DIC);
    }
```

**HttpMessageConverter**

```java
@Configuration
public class WebConfigurer implements WebMvcConfigurer {
    public void extendMessageConverters(List<HttpMessageConverter<?>> converters) {
        converters.add(new PropertiesHttpMessageConverter());
    }
}
```

## WebServerFactoryCustomizer

通过创建 WebServerFactoryCustomizer子类的实例并注册为Bean, 可对SpringBoot内置嵌入Web容器进行设置.

WebServerFactory 对象创建完毕后， WebServerFactoryCustomizerBeanPostProcessor 会从 BeanFactory 中查询所有 WebServerFactoryCustomizer 的Bean生成列表、排序，然后逐一调用 WebServerFactoryCustomizer 的 customize 方法。

```java
//泛型也可以为TomcatServletWebServerFactory
//改变配置文件的端口然后重启app(close方法),加载配置文件获取端口达到动态修改端口效果
@Component
public class WebServerFactoryCustomizationBean implements WebServerFactoryCustomizer<ConfigurableServletWebServerFactory> {
    @Override
    public void customize(ConfigurableServletWebServerFactory server) {
        server.setPort(8081);
    }
}
```

## RequestContextHolder

该类用来获取请求属性, 如:

```java
ServletRequestAttributes attributes = (ServletRequestAttributes)RequestContextHolder.getRequestAttributes();
HttpServletRequest request = attributes.getRequest();
HttpServletResponse response = attributes.getResponse();
```

> 当对HttpServletRequest进行自动注入的时候, 不能像普通Bean一样注入,  而是注入了一个代理request, 当调用其方法时实际调用的是RequestObjectFactory工厂生成的对象的方法, 底层一样同RequestContextHolder实现, 生成的RequestAttributes为线程ThreadLocal变量.

## @ServletComponentScan

在SpringBootApplication上使用@ServletComponentScan注解后，Servlet、Filter、Listener可以直接通过@WebServlet、@WebFilter、@WebListener注解自动注册，无需其他代码。

## 类型转换

默认情况下，支持简单类型（整数、长整型、日期等）。可以通过 WebDataBinder或通过注册Formatters(Converter)来自定义类型转换。

### PropertyEditorSupport

1.通过`PropertyEditorRegistrar`全局注册自定义属性编辑器(PropertyEditorSupport), 这里注册了对String和Date类型的转换器.

```java
public class StringEditor extends PropertyEditorSupport {
    @Override
    public void setAsText(String text) throws IllegalArgumentException {
        if (text == null) {
            setValue(null);
        }  else {
            setValue(HtmlUtils.htmlEscape(text));
        }
    }
}
```

```java
public class DateEditor extends PropertyEditorSupport {
    private final String[] datePatterns;

    public DateEditor() {
        this(new String[]{"yyyy", "yyyy-MM", "yyyyMM", "yyyy/MM", "yyyy-MM-dd", "yyyyMMdd", "yyyy/MM/dd", "yyyy-MM-dd HH:mm:ss", "yyyyMMddHHmmss", "yyyy/MM/dd HH:mm:ss"});
    }

    public DateEditor(String[] datePatterns) {
        this.datePatterns = datePatterns;
    }
    @Override
    public void setAsText(String text) {
        if (text == null) {
            setValue(null);
        } else {
            try {
                setValue(DateUtils.parseDate(text.trim(), datePatterns));
            } catch (ParseException e) {
                setValue(null);
            }
        }
    }
}
```

注册

```java
@Configuration
public class WebBindingInitializerConfiguration {
    @Bean
    public ConfigurableWebBindingInitializer configurableWebBindingInitializer(FormattingConversionService mvcConversionService, Validator mvcValidator) {
        ConfigurableWebBindingInitializer initializer = new ConfigurableWebBindingInitializer();
        initializer.setConversionService(mvcConversionService);
        initializer.setValidator(mvcValidator);
        initializer.setPropertyEditorRegistrar(propertyEditorRegistry -> {
            propertyEditorRegistry.registerCustomEditor(String.class, new StringEditor());
            propertyEditorRegistry.registerCustomEditor(Date.class, new DateEditor());
        });
        return initializer;
    }
}
```

2.如果想要局部注册, 可通过@InitBinder方法的WebDataBinder参数进行注册, 该转换器只针对当前Controller:

```java
@InitBinder
public void initBinder(WebDataBinder binder){
    // Date 类型转换
    binder.registerCustomEditor(Date.class, new PropertyEditorSupport(){
        @Override
        public void setAsText(String text){
            setValue(DateUtils.parseDate(text,parsePatterns));
        }
    });
}
```

WebDataBinder可实现对参数的更多控制, 如:

```java
@InitBinder
public void initBinder(WebDataBinder binder){
    binder.setDisallowedFields("id");
}
```

### Converter

类型转换器, Spring自带了很多消息转换器, 也可以自定义, 将转换器注册到ioc容器中即可生效,不需要WebMvcConfigurer.

比如时间类型转换器,声明的该转换器之后前端传来的指定格式内的时间字符串会自动转换为Date对象:

```java
public class DateConverter implements Converter<String, Date> {
    private static final List<String> formats = new ArrayList<>(4);
    static {
        formats.add("yyyy-MM");
        formats.add("yyyy-MM-dd");
        formats.add("yyyy-MM-dd HH:mm");
        formats.add("yyyy-MM-dd HH:mm:ss");
    }

    @Override
    public Date convert(String source) {
        String value = source.trim();
        if ("".equals(value)) {
            return null;
        }
        if (source.matches("^\\d{4}-\\d{1,2}$")) {
            return parseDate(source, formats.get(0));
        } else if (source.matches("^\\d{4}-\\d{1,2}-\\d{1,2}$")) {
            return parseDate(source, formats.get(1));
        } else if (source.matches("^\\d{4}-\\d{1,2}-\\d{1,2} {1}\\d{1,2}:\\d{1,2}$")) {
            return parseDate(source, formats.get(2));
        } else if (source.matches("^\\d{4}-\\d{1,2}-\\d{1,2} {1}\\d{1,2}:\\d{1,2}:\\d{1,2}$")) {
            return parseDate(source, formats.get(3));
        } else {
            throw new IllegalArgumentException("Invalid boolean value '" + source + "'");
        }
    }

    private Date parseDate(String dateStr, String format) {
        Date date = null;
        try {
            DateFormat dateFormat = new SimpleDateFormat(format);
            date = dateFormat.parse(dateStr);
        } catch (Exception e) {
            e.printStackTrace();
        }
        return date;
    }
}
```

> 也可以实现ConverterFactory接口

## 内容协商

**内容协商**机制是指客户端和服务器端就响应的资源内容进行交涉，然后提供给客户端最为适合的资源。内容协商会以响应资源的语言、字符集、编码方式等作为判断的基准。HTTP请求头中Content-Type，Accept等内容就是内容协商判断的标准。在Spring Boot中，一个完整的内容协商过程如下图所示：

![](picture/contentNegotiation.png)

> @RequestBody和@ResponseBody都是通过HttpMessageConverter对请求体转换(序列化)和转换为响应体(反序列化)的.

核心组件:

| 组件                              | 名称          | 说明                                |
|:------------------------------- |:----------- |:--------------------------------- |
| ContentNegotiationManager       | 内容协商管理器     | ContentNegotiationStrategy 控制策略   |
| MediaType                       | 媒体类型        | HTTP 消息媒体类型，如 text/html           |
| @RequestMapping#consumes        | 消费媒体类型      | 请求头 Content-Type 媒体类型映射           |
| @RequestMapping#produces        | 生产媒体类型      | 响应头 Content-Type 媒体类型映射           |
| HttpMessageConverter            | HTTP消息转换器接口 | HTTP 消息转换器，用于反序列化 HTTP 请求或序列化响应   |
| WebMvcConfigurer                | Web MVC 配置器 | 配置 REST 相关的组件                     |
| HandlerMethod                   | 处理方法        | @RequestMapping 标注的方法             |
| HandlerMethodArgumentResolver   | 处理方法参数解析器   | 用于 HTTP 请求中解析 HandlerMethod 参数内容  |
| HandlerMethodReturnValueHandler | 处理方法返回值解析器  | 用于 HandlerMethod 返回值解析为 HTTP 响应内容 |

### HttpMessageConverter

`HttpMessageConverter`为HTTP消息转换接口，Spring根据不同的媒体类型进行了相应的实现。比如上图中Accept为application/json，所以在第7步中，会选择使用`HttpMessageConverter`的实现类`MappingJackson2HttpMessageConverter`来处理返回值。

使用Fastjson

```java
@Override
public void configureMessageConverters(List<HttpMessageConverter<?>> converters) {
    // 使用 fastjson 序列化，会导致 @JsonIgnore 失效，可以使用 @JSONField(serialize = false) 替换
    FastJsonHttpMessageConverter converter = new FastJsonHttpMessageConverter();
    List<MediaType> supportMediaTypeList = new ArrayList<>();
    supportMediaTypeList.add(MediaType.APPLICATION_JSON_UTF8);
    FastJsonConfig config = new FastJsonConfig();
    config.setDateFormat("yyyy-MM-dd HH:mm:ss");
    config.setSerializerFeatures(SerializerFeature.DisableCircularReferenceDetect);
    converter.setFastJsonConfig(config);
    converter.setSupportedMediaTypes(supportMediaTypeList);
    converter.setDefaultCharset(StandardCharsets.UTF_8);
    converters.add(converter);
}
```

官方案例:

```java
@Configuration
@EnableWebMvc
public class WebConfiguration implements WebMvcConfigurer {
    @Override
    public void configureMessageConverters(List<HttpMessageConverter<?>> converters) {
        Jackson2ObjectMapperBuilder builder = new Jackson2ObjectMapperBuilder()
                .indentOutput(true)
                .dateFormat(new SimpleDateFormat("yyyy-MM-dd"))
                .modulesToInstall(new ParameterNamesModule());
        converters.add(new MappingJackson2HttpMessageConverter(builder.build()));
        converters.add(new MappingJackson2XmlHttpMessageConverter(builder.createXmlMapper(true).build()));
    }
}
```

> 如果需要覆盖默认的converter,  可直接注入HttpMessageConverters

**xml内容协商**

```xml
<dependency>
    <groupId>com.fasterxml.jackson.dataformat</groupId>
    <artifactId>jackson-dataformat-xml</artifactId>
</dependency>
```

以传统spring应用需要引入`MappingJackson2XmlHttpMessageConverter`为例

```java
@Configuration
public class MessageConverterConfig1 extends WebMvcConfigurerAdapter {
    @Override
    public void configureMessageConverters(List<HttpMessageConverter<?>> converters) {
        Jackson2ObjectMapperBuilder builder = Jackson2ObjectMapperBuilder.xml();
        builder.indentOutput(true);
        converters.add(new MappingJackson2XmlHttpMessageConverter(builder.build()));
    }
}
```

在Spring Boot应用不用像上面这么麻烦，只需要加入`jackson-dataformat-xml`依赖，Spring Boot就会自动引入`MappingJackson2XmlHttpMessageConverter`的实现：

案例 xml消息转换器

```java
//定义对象与xml的关系
@Data
@NoArgsConstructor
@AllArgsConstructor
@JacksonXmlRootElement(localName = "User")
public class User {
    @JacksonXmlProperty(localName = "name")
    private String name;
    @JacksonXmlProperty(localName = "age")
    private Integer age;

}
```

```
其对应的xml的样例为
<User>
    <name>aaaa</name>
    <age>10</age>
</User>
```

使用(发送xml,返回xml)

```java
@RestController 
public class UserController {
    @PostMapping(value = "/user", 
        consumes = MediaType.APPLICATION_XML_VALUE, 
        produces = MediaType.APPLICATION_XML_VALUE)
    public User create(@RequestBody User user) {
        user.setName("didispace.com : " + user.getName());
        user.setAge(user.getAge() + 100);
        return user;
    }
}
```

**自定义内容协商**

假如现在要实现一个用于处理 Content-Type 为 text/properties 媒体类型的 HttpMessageConverter 实现类PropertiesHttpMessageConverter，当我们在请求体中传输下面内容时：

```
name:mrbrid
age:18
```

能够自动转换为Properties对象。

参照*MappingJackson2HttpMessageConverter的*实现方式继承`AbstractGenericHttpMessageConverter`的方式来实现`HttpMessageConverter`接口,

其中`readInternal`为反序列化过程，即将HTTP请求反序列化为参数的过程；`writeInternal`为序列化过程，将响应序列化:

```java
public class PropertiesHttpMessageConverter extends AbstractGenericHttpMessageConverter<Properties> {
    //在构造函数中指定它能处理的媒体类型(content-type)
    public PropertiesHttpMessageConverter() {
        super(new MediaType("text", "properties"));
    }
    @Override
    protected void writeInternal(Properties properties, Type type, HttpOutputMessage outputMessage) throws IOException, HttpMessageNotWritableException {
        // 获取请求头
        HttpHeaders headers = outputMessage.getHeaders();
        // 获取 content-type
        MediaType contentType = headers.getContentType();
        // 获取编码
        Charset charset = null;
        if (contentType != null) {
            charset = contentType.getCharset();
        }
        charset = charset == null ? Charset.forName("UTF-8") : charset;
        // 获取请求体
        OutputStream body = outputMessage.getBody();
        OutputStreamWriter outputStreamWriter = new OutputStreamWriter(body, charset);

        properties.store(outputStreamWriter, "Serialized by PropertiesHttpMessageConverter#writeInternal");
    }
     @Override
    protected Properties readInternal(Class<? extends Properties> clazz, HttpInputMessage inputMessage) throws IOException, HttpMessageNotReadableException {
        Properties properties = new Properties();
        // 获取请求头
        HttpHeaders headers = inputMessage.getHeaders();
        // 获取 content-type
        MediaType contentType = headers.getContentType();
        // 获取编码
        Charset charset = null;
        if (contentType != null) {
            charset = contentType.getCharset();
        }
        charset = charset == null ? Charset.forName("UTF-8") : charset;
        // 获取请求体输入流
        InputStream body = inputMessage.getBody();
        InputStreamReader inputStreamReader = new InputStreamReader(body, charset);
        // 加载
        properties.load(inputStreamReader);
        return properties;
    }

    @Override
    public Properties read(Type type, Class<?> contextClass, HttpInputMessage inputMessage) throws IOException, HttpMessageNotReadableException {
        return readInternal(null, inputMessage);
    }
}
```

```Java
@Configuration
public class WebConfigurer implements WebMvcConfigurer {
    @Override
    public void extendMessageConverters(List<HttpMessageConverter<?>> converters) {
        // converters.add(new PropertiesHttpMessageConverter());
        // 指定顺序，这里为第一个,否则会被排在前面的MappingJackson2HttpMessageConverter优先处理
        // 因为需要@定义的是REST接口，所以响应默认会被序列化为JSON格式
        converters.add(0, new PropertiesHttpMessageConverter());
    }
}
```

controller

```java
@RestController
public class TestController {
    //指定接收的Content-Type为text/properties
    @GetMapping(value = "test", consumes = "text/properties")
    public Properties test(@RequestBody Properties properties) {
        return properties;
    }
}
```

上面这种方式必须依赖于`@RequestBody`和`@ResponseBody`注解，除此之外我们还可以通过自定义`HandlerMethodArgumentResolver`和`HandlerMethodReturnValueHandler`实现类的方式来处理内容协商。

### HandlerMethodArgumentResolver

`HandlerMethodArgumentResolver`俗称方法参数解析器，用于解析由`@RequestMapping`注解（或其派生的注解）所标注的方法的参数。

测试通过实现`HandlerMethodArgumentResolver`的方式来将HTTP请求体的内容自动解析为Properties对象。

```java
public class PropertiesHandlerMethodArgumentResolver implements HandlerMethodArgumentResolver {
    //用于指定支持解析的参数类型，这里为Properties类型
    @Override
    public boolean supportsParameter(MethodParameter parameter) {
        return Properties.class.equals(parameter.getParameterType());
    }
    //用于实现解析逻辑，解析过程和PropertiesHttpMessageConverter的readInternal方法类似。
    @Override
    public Object resolveArgument(MethodParameter parameter, ModelAndViewContainer mavContainer, NativeWebRequest webRequest, WebDataBinderFactory binderFactory) throws Exception {
        ServletWebRequest servletWebRequest = (ServletWebRequest) webRequest;
        HttpServletRequest request = servletWebRequest.getRequest();
        String contentType = request.getHeader("Content-Type");

        MediaType mediaType = MediaType.parseMediaType(contentType);
        // 获取编码
        Charset charset = mediaType.getCharset() == null ? Charset.forName("UTF-8") : mediaType.getCharset();
        // 获取输入流
        InputStream inputStream = request.getInputStream();
        InputStreamReader inputStreamReader = new InputStreamReader(inputStream, charset);

        // 输入流转换为 Properties
        Properties properties = new Properties();
        properties.load(inputStreamReader);
        return properties;
    }
}
```

接着，我们还需将`PropertiesHandlerMethodArgumentResolver`添加到Spring自带的`HandlerMethodArgumentResolver`实现类集合中.

但是我们不能在配置类`WebMvcConfigurer`中通过重写`addArgumentResolvers`的方式来添加，查看该方法源码上注释着:   通过这个方法来添加的方法参数解析器不会覆盖Spring内置的方法参数解析器(**添加到最后**)，如果需要这么做的话，可以直接通过修改RequestMappingHandlerAdapter来实现。

```Java
@Configuration
public class WebConfigurer implements WebMvcConfigurer {
    @Autowired
    private RequestMappingHandlerAdapter requestMappingHandlerAdapter;

    @PostConstruct  //WebConfigurer配置类装配完毕的时候执行
    public void init() {
        // 获取当前 RequestMappingHandlerAdapter 所有的 ArgumentResolver对象
        List<HandlerMethodArgumentResolver> argumentResolvers = requestMappingHandlerAdapter.getArgumentResolvers();
        List<HandlerMethodArgumentResolver> newArgumentResolvers = new ArrayList<>(argumentResolvers.size() + 1);
        // 添加 PropertiesHandlerMethodArgumentResolver 到集合第一个位置
        newArgumentResolvers.add(0, new PropertiesHandlerMethodArgumentResolver());
        // 将原 ArgumentResolver 添加到集合中
        newArgumentResolvers.addAll(argumentResolvers);
        // 重新设置 ArgumentResolver对象集合
        requestMappingHandlerAdapter.setArgumentResolvers(newArgumentResolvers);
    }
}
```

通过`requestMappingHandlerAdapter`对象的`setArgumentResolvers`方法来重新设置方法解析器集合，将`PropertiesHandlerMethodArgumentResolver`添加到集合的第一个位置。

之所以要将`PropertiesHandlerMethodArgumentResolver`添加到第一个位置是因为Properties本质也是一个Map对象，而Spring内置的`MapMethodProcessor`就是用于处理Map参数类型的，如果不将`PropertiesHandlerMethodArgumentResolver`优先级提高，那么Properties类型参数会被`MapMethodProcessor`解析，从而出错.

测试:

```java
@RestController
public class TestController {
    @GetMapping(value = "test1", consumes = "text/properties")
    public Properties test1(Properties properties) {  //参数没有被@RequestBody标注
        return properties;
    }
}
```

成功执行，并且返回了正确的内容.

但是返回的序列化工作还是通过PropertiesHttpMessageConverter`的`writeInternal+@ResponseBody实现的.

接着我们开始实现自定义方法返回值解析器，并且不依赖于`@ResponseBody`注解。

> newbee mall 通过该接口 ,  对带指定注解的接口参数使用请求头里的token从dao获取登录用户对象解析赋值    :
> 
> ```java
> @Component
> public class TokenToAdminUserMethodArgumentResolver implements HandlerMethodArgumentResolver {
>         @Autowired
>         private NewBeeAdminUserTokenMapper newBeeAdminUserTokenMapper;
>         public TokenToAdminUserMethodArgumentResolver() {
>         }
>     public boolean supportsParameter(MethodParameter parameter) {
>             if (parameter.hasParameterAnnotation(TokenToAdminUser.class)) {
>                 return true;
>            }
>             return false;
>        }
> 
>      public Object resolveArgument(MethodParameter parameter, ModelAndViewContainer mavContainer, NativeWebRequest webRequest, WebDataBinderFactory binderFactory) {
>             if (parameter.getParameterAnnotation(TokenToAdminUser.class) instanceof TokenToAdminUser) {
>                 String token = webRequest.getHeader("token");
>                 if (null != token && !"".equals(token) && token.length() == Constants.TOKEN_LENGTH) {
>                     AdminUserToken adminUserToken = newBeeAdminUserTokenMapper.selectByToken(token);
>                     if (adminUserToken == null) {
>                         NewBeeMallException.fail(ServiceResultEnum.ADMIN_NOT_LOGIN_ERROR.getResult());
>                     } else if (adminUserToken.getExpireTime().getTime() <= System.currentTimeMillis()) {
>                         NewBeeMallException.fail(ServiceResultEnum.ADMIN_TOKEN_EXPIRE_ERROR.getResult());
>                     }
>                     return adminUserToken;
>                 } else {
>                     NewBeeMallException.fail(ServiceResultEnum.ADMIN_NOT_LOGIN_ERROR.getResult());
>                 }
>             }
>         return null;
>         }
>    }
> ```
> 
> 并且该项目是通过WebMvcConfigurer接口的addArgumentResolver方法添加该解析器的,即不覆盖spring自带的解析器链路
> 
> ```java
> public void addArgumentResolvers(List<HandlerMethodArgumentResolver> argumentResolvers) {
>     argumentResolvers.add(tokenToMallUserMethodArgumentResolver);
>     argumentResolvers.add(tokenToAdminUserMethodArgumentResolver);
>    }
> ```

### HandlerMethodReturnValueHandler

`HandlerMethodArgumentResolver`俗称方法返回值解析器，用于解析由`@RequestMapping`注解（或其派生的注解）所标注的方法的返回值。

测试通过实现`HandlerMethodReturnValueHandler`的方式来自定义一个用于处理返回值类型为Properties类型的解析器。

```java
public class PropertiesHandlerMethodReturnValueHandler implements HandlerMethodReturnValueHandler {
    @Override
    public boolean supportsReturnType(MethodParameter returnType) {
        return Properties.class.equals(returnType.getMethod().getReturnType());
    }
    @Override
    public void handleReturnValue(Object returnValue, MethodParameter returnType, ModelAndViewContainer mavContainer, NativeWebRequest webRequest) throws Exception {
        Properties properties = (Properties) returnValue;

        ServletWebRequest servletWebRequest = (ServletWebRequest) webRequest;

        HttpServletResponse response = servletWebRequest.getResponse();
        ServletServerHttpResponse servletServerHttpResponse = new ServletServerHttpResponse(response);

        // 获取请求头
        HttpHeaders headers = servletServerHttpResponse.getHeaders();

        MediaType contentType = headers.getContentType();
        // 获取编码
        Charset charset = null;
        if (contentType != null) {
            charset = contentType.getCharset();
        }

        charset = charset == null ? Charset.forName("UTF-8") : charset;

        // 获取请求体
        OutputStream body = servletServerHttpResponse.getBody();
        OutputStreamWriter outputStreamWriter = new OutputStreamWriter(body, charset);

        properties.store(outputStreamWriter, "Serialized by PropertiesHandlerMethodReturnValueHandler#handleReturnValue");

        // 告诉 Spring MVC 请求已经处理完毕
        //    mavContainer.setRequestHandled(true);
    }
}
```

`supportsReturnType`方法指定了处理返回值的类型，`handleReturnValue`方法用于处理返回值

接着将`PropertiesHandlerMethodReturnValueHandler`添加到到Spring自带的`HandlerMethodReturnValueHandler`实现类集合中，添加方式和自定义`HandlerMethodArgumentResolver`一致：

```java
@Configuration
public class WebConfigurer implements WebMvcConfigurer {
    @Autowired
    private RequestMappingHandlerAdapter requestMappingHandlerAdapter;
    @PostConstruct
    public void init() {
        // 获取当前 RequestMappingHandlerAdapter 所有的 ArgumentResolver对象
        List<HandlerMethodArgumentResolver> argumentResolvers = requestMappingHandlerAdapter.getArgumentResolvers();
        List<HandlerMethodArgumentResolver> newArgumentResolvers = new ArrayList<>(argumentResolvers.size() + 1);
        // 添加 PropertiesHandlerMethodArgumentResolver 到集合第一个位置
        newArgumentResolvers.add(0, new PropertiesHandlerMethodArgumentResolver());
        // 将原 ArgumentResolver 添加到集合中
        newArgumentResolvers.addAll(argumentResolvers);
        // 重新设置 ArgumentResolver对象集合
        requestMappingHandlerAdapter.setArgumentResolvers(newArgumentResolvers);

        // 获取当前 RequestMappingHandlerAdapter 所有的 returnValueHandlers对象
        List<HandlerMethodReturnValueHandler> returnValueHandlers = requestMappingHandlerAdapter.getReturnValueHandlers();
        List<HandlerMethodReturnValueHandler> newReturnValueHandlers = new ArrayList<>(returnValueHandlers.size() + 1);
        // 添加 PropertiesHandlerMethodReturnValueHandler 到集合第一个位置
        newReturnValueHandlers.add(0, new PropertiesHandlerMethodReturnValueHandler());
        // 将原 returnValueHandlers 添加到集合中
        newReturnValueHandlers.addAll(returnValueHandlers);
        // 重新设置 ReturnValueHandlers对象集合
        requestMappingHandlerAdapter.setReturnValueHandlers(newReturnValueHandlers);
    }
}
```

现在,在controller中就可以完全去掉@RequestBody和@ResponseBody了.

```java
@Controller
public class TestController {
    @GetMapping(value = "test1", consumes = "text/properties")
    public Properties test1(Properties properties) {  
        return properties;
    }
}
```

注意:测试虽然返回结果没有问题,但是控制台还是显示了异常信息:

```
javax.servlet.ServletException: Circular view path [test1]: would dispatch back to the current handler URL [/test1] again. Check your ViewResolver setup! (Hint: This may be the result of an unspecified view, due to default view name generation.)
```

在Spring中如果Controller中的方法没有被`@ResponseBody`标注的话，默认会把返回值当成视图的名称.而这里我们并不希望解析的Properties值被当成视图名称，所以我们需要在`PropertiesHandlerMethodReturnValueHandler`的`handleReturnValue`方法最后一行添加如下代码：

```
// 告诉 Spring MVC 请求已经处理完毕
mavContainer.setRequestHandled(true);
```

## ServletContextListener

监听 ServletContext 对象的生命周期，实际上就是监听 Web 应用的生命周期。

当Servlet 容器启动或终止Web 应用时，会触发ServletContextEvent 事件，该事件由ServletContextListener 来处理。在 ServletContextListener 接口中定义了处理ServletContextEvent 事件的两个方法。

通常用来在context中设置参数(可以从配置文件中读取).

使用时可以通过ServletContext的getAttribute获取,

也可以在模板中直接使用,如thymeleaf:  ${application.website.xxx} ,可以用来设置网站的固定页尾内容.

```java
@WebListener
public class StarterListener implements ServletContextListener {
    @Override
    public void contextInitialized(ServletContextEvent sce) {
        sce.getServletContext().setAttribute("website",xxx);
    }
    @Override
    public void contextDestroyed(ServletContextEvent sce) {
    //...
    }
}
```

## Filter与Intercepter

### Filter

1.@WebFilter+@Component

```java
@Component
@WebFilter(urlPatterns = {"/*"})
public class TimeFilter implements Filter{
    @Override
    public void init(FilterConfig filterConfig) throws ServletException {
        System.out.println("过滤器初始化");
    }

    @Override
    public void doFilter(ServletRequest servletRequest, ServletResponse servletResponse, FilterChain filterChain) throws IOException, ServletException {
        System.out.println("开始执行过滤器");
        Long start = new Date().getTime();
        filterChain.doFilter(servletRequest, servletResponse);
        System.out.println("【过滤器】耗时 " + (new Date().getTime() - start));
        System.out.println("结束执行过滤器");
    }

    @Override
    public void destroy() {
        System.out.println("过滤器销毁");
    }
}
```

`@Component`注解让`TimeFilter`成为Spring上下文中的一个Bean，`@WebFilter`注解的`urlPatterns`属性配置了哪些请求可以进入该过滤器，`/*`表示所有请求

2.通过`FilterRegistrationBean`来注册过滤器。`FilterRegistrationBean`也可以指定过滤器类为泛型,

多个过滤器需要多个FilterRegistrationBean

```java
@Configuration
public class WebConfig {
    @Bean
    public FilterRegistrationBean timeFilter() {
        FilterRegistrationBean filterRegistrationBean = new FilterRegistrationBean();
        TimeFilter timeFilter = new TimeFilter();
        filterRegistrationBean.setFilter(timeFilter);

        List<String> urlList = new ArrayList<>();
        urlList.add("/*");
        filterRegistrationBean.setUrlPatterns(urlList);
        return filterRegistrationBean;
    }
}
```

**其他配置**

initParameter: 在Filter的init方法中FilterConfig类型的参数方法getInitParameter获取.

> 配置类方式注入可通过FilterRegistrationBean的addInitParameter添加; 注解方式注入可通过@WebInitParam注解加入

order: 指定过滤器顺序.

dispatcherType: 处理请求类型, 包括5种:

1.REQUEST:所有类型的请求;

2.FOWARD:只有当当前页面是通过请求转发转发过来的情形时，才会走指定的过滤器;

3.INCLUDE:只要是通过<jsp:include page="xxx.jsp" />，嵌入进来的页面，每嵌入的一个页面，都会走一次指定的过滤器;

4.ASYNC;

5.ERROR.

> `OncePerRequestFilter` 为Filter实现类, 确保在一次请求中只通过一次filter

### Intercepter

实现`HandlerInterceptor`或继承`HandlerInterceptorAdapter`.

```java
public class TimeInterceptor implements HandlerInterceptor {
    //方法在处理拦截之前执行
    @Override
    public boolean preHandle(HttpServletRequest httpServletRequest, HttpServletResponse httpServletResponse, Object o) throws Exception {
        System.out.println("处理拦截之前");
        httpServletRequest.setAttribute("startTime", new Date().getTime());
        //获取处理该请求的controller
        System.out.println(((HandlerMethod) o).getBean().getClass().getName());
        //获取处理该请求的方法
        System.out.println(((HandlerMethod) o).getMethod().getName());
        return true;
    }
    //当被拦截的方法没有抛出异常成功时才会处理
    @Override
    public void postHandle(HttpServletRequest httpServletRequest, HttpServletResponse httpServletResponse, Object o, ModelAndView modelAndView) throws Exception {
        System.out.println("开始处理拦截");
        Long start = (Long) httpServletRequest.getAttribute("startTime");
        System.out.println("【拦截器】耗时 " + (new Date().getTime() - start));
    }
    //方法无论被拦截的方法抛出异常与否都会执行
    @Override
    public void afterCompletion(HttpServletRequest httpServletRequest, HttpServletResponse httpServletResponse, Object o, Exception e) throws Exception {
        System.out.println("处理拦截之后");
        Long start = (Long) httpServletRequest.getAttribute("startTime");
        System.out.println("【拦截器】耗时 " + (new Date().getTime() - start));
        System.out.println("异常信息 " + e);
    }
}
```

要使拦截器在Spring Boot中生效，还需要如下两步配置：

1.在拦截器类上加入`@Component`注解；

2.在`WebConfig`中通过`InterceptorRegistry`注册过滤器:

```java
@Configuration
public class WebConfig extends WebMvcConfigurerAdapter {
    @Autowired
    private TimeInterceptor timeInterceptor;
    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(timeInterceptor).addPathPatterns("/**").excludePathPatterns("/admin/**");
    }
```

相较于过滤器，拦截器多了Object和Exception对象，所以可以获取的信息比过滤器要多的多。

**若依防重复请求**:

若依的防重复请求的依据为请求的参数或请求体在指定时间内唯一.

```java
@Inherited
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface RepeatSubmit{
   /**
    * 间隔时间(ms)，小于此时间视为重复提交
    */
   public int interval() default 5000;
   /**
    * 提示消息
    */
   public String message() default "不允许重复提交，请稍后再试";
}
```

```java
/**
 * 防止重复提交拦截器, 模板设计模式
 */
@Component
public abstract class RepeatSubmitInterceptor extends HandlerInterceptorAdapter{
    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception{
        if (handler instanceof HandlerMethod){
            //即controller方法
            HandlerMethod handlerMethod = (HandlerMethod) handler;
            Method method = handlerMethod.getMethod();
            RepeatSubmit annotation = method.getAnnotation(RepeatSubmit.class);
            if (annotation != null){
                //判断是否重复提交, 如果是, 直接返回错误json结果
                if (this.isRepeatSubmit(request, annotation)){
                    AjaxResult ajaxResult = AjaxResult.error(annotation.message());
                    ServletUtils.renderString(response, JSONObject.toJSONString(ajaxResult));
                    return false;
                }
            }
            return true;
        }
        else{
            return super.preHandle(request, response, handler);
        }
    }

    /**
     * 验证是否重复提交由子类实现具体的防重复提交的规则
     */
    public abstract boolean isRepeatSubmit(HttpServletRequest request, RepeatSubmit annotation);
}
```

```java
/**
 * 判断请求url和数据是否和上一次相同，
 * 如果和上次相同，则是重复提交表单。 有效时间为5秒内。
 */
@Component
public class SameUrlDataInterceptor extends RepeatSubmitInterceptor{
    public final String REPEAT_PARAMS = "repeatParams";
    public final String REPEAT_TIME = "repeatTime";

    // 令牌自定义标识
    @Value("${token.header}")
    private String header;

    @Autowired
    private RedisCache redisCache;

    @SuppressWarnings("unchecked")
    @Override
    public boolean isRepeatSubmit(HttpServletRequest request, RepeatSubmit annotation{
        //将请求体或者请求参数和时间戳以map的形式作为防止重复提交检验的value
        String nowParams = "";
        if (request instanceof RepeatedlyRequestWrapper){
            RepeatedlyRequestWrapper repeatedlyRequest = (RepeatedlyRequestWrapper) request;
            nowParams = HttpHelper.getBodyString(repeatedlyRequest);
        }
        // body参数为空，获取Parameter的数据
        if (StringUtils.isEmpty(nowParams)){
            nowParams = JSONObject.toJSONString(request.getParameterMap());
        }
        Map<String, Object> nowDataMap = new HashMap<String, Object>();
        nowDataMap.put(REPEAT_PARAMS, nowParams);
        nowDataMap.put(REPEAT_TIME, System.currentTimeMillis());

        // 请求地址
        String url = request.getRequestURI();

        // 唯一值(没有消息头则使用请求地址) 这里为token
        String submitKey = request.getHeader(header);
        if (StringUtils.isEmpty(submitKey)){
            submitKey = url;
        }

        // 唯一标识作为cache的key(指定前缀 + 消息头/请求地址)
        String cacheRepeatKey = Constants.REPEAT_SUBMIT_KEY + submitKey;

        Object sessionObj = redisCache.getCacheObject(cacheRepeatKey);
        if (sessionObj != null) { 
            Map<String, Object> sessionMap = (Map<String, Object>) sessionObj;
            if (sessionMap.containsKey(url)){
                Map<String, Object> preDataMap = (Map<String, Object>) sessionMap.get(url);
                //是否小于指定重复提交间隔时间
                if (compareParams(nowDataMap, preDataMap) && compareTime(nowDataMap, preDataMap, annotation.interval())){
                    return true;
                }
            }
        }
        //首次提交 或者 重复 --> 重新生成
        Map<String, Object> cacheMap = new HashMap<String, Object>();
        //再包装一层
        cacheMap.put(url, nowDataMap);
        redisCache.setCacheObject(cacheRepeatKey, cacheMap, annotation.interval(), TimeUnit.MILLISECONDS);
        return false;
    }

    /**
     * 判断参数是否相同
     */
    private boolean compareParams(Map<String, Object> nowMap, Map<String, Object> preMap){
        String nowParams = (String) nowMap.get(REPEAT_PARAMS);
        String preParams = (String) preMap.get(REPEAT_PARAMS);
        return nowParams.equals(preParams);
    }

    /**
     * 判断两次间隔时间
     */
    private boolean compareTime(Map<String, Object> nowMap, Map<String, Object> preMap, int interval){
        long time1 = (Long) nowMap.get(REPEAT_TIME);
        long time2 = (Long) preMap.get(REPEAT_TIME);
        if ((time1 - time2) < interval){
            return true;
        }
        return false;
    }
}
```

```java
@Configuration
public class ResourcesConfig implements WebMvcConfigurer{
    @Autowired
    private RepeatSubmitInterceptor repeatSubmitInterceptor;

    @Override
    public void addInterceptors(InterceptorRegistry registry){
        registry.addInterceptor(repeatSubmitInterceptor).addPathPattens("/**");
    }
}
```

## HttpServletRequestWrapper

该类继承于HttpServletRequest,可用于对request重构, 如参数过滤、方法重写等

如构建可重复读的request, 将请求体缓存:

```java
public class RepeatedlyRequestWrapper extends HttpServletRequestWrapper {
    private final byte[] body;

    public RepeatedlyRequestWrapper(HttpServletRequest request, ServletResponse response) 
throws IOException{
        super(request);
        request.setCharacterEncoding("UTF-8");
        response.setCharacterEncoding("UTF-8");
        body = HttpHelper.getBodyString(request).getBytes("UTF-8");
    }

    @Override
    public BufferedReader getReader() throws IOException {
        return new BufferedReader(new InputStreamReader(getInputStream()));
    }

    @Override
    public ServletInputStream getInputStream() throws IOException {
        final ByteArrayInputStream bais = new ByteArrayInputStream(body);
        return new ServletInputStream() {
            @Override
            public int read() throws IOException {
                return bais.read();
            }
            @Override
            public int available() throws IOException {
                return body.length;
            }
            @Override
            public boolean isFinished() {
                return false;
            }
            @Override
            public boolean isReady() {
                return false;
            }
            @Override
            public void setReadListener(ReadListener readListener) {
            }
        };
    }
}
```

并在Filter中进行对请求进行构建

```java
 @Override
    public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain)
            throws IOException, ServletException {
        ServletRequest requestWrapper = null;
        if (request instanceof HttpServletRequest
                && StringUtils.startsWithIgnoreCase(request.getContentType(), MediaType.APPLICATION_JSON_VALUE)) {
            requestWrapper = new RepeatedlyRequestWrapper((HttpServletRequest) request, response);
        }
        if (null == requestWrapper) {
            chain.doFilter(request, response);
        } else {
            chain.doFilter(requestWrapper, response);
        }
    }
```

### spring-session-data-redis

https://docs.spring.io/spring-session/docs/current/reference/html5/guides/boot-redis.html

    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-redis</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.session</groupId>
        <artifactId>spring-session-data-redis</artifactId>
    </dependency>

分布式session共享整合框架

在项目启动类上添加如下注解,新版本可以不加了

```
@EnableRedisHttpSession
```

注解的主要作用是注册一个SessionRepositoryFilter，这个Filter会拦截到所有的请求，对Session进行操作，注入SessionRepositoryFilter的代码在RedisHttpSessionConfiguration类的父类SpringHttpSessionConfiguration中。注册SessionRepositoryFilter时需要一个SessionRepository参数，这个参数是在RedisHttpSessionConfiguration中被注入进入的。

请求进来的时候拦截器会先将request和response拦截住，然后将这两个对象转换成Spring内部的包装类SessionRepositoryRequestWrapper和SessionRepositoryResponseWrapper对象。SessionRepositoryRequestWrapper类重写了原生的getSession方法。

查看getSession方法可以看到,当调用SessionRepositoryRequestWrapper对象的getSession方法拿Session的时候，会先从当前请求的属性中查找.CURRENT_SESSION属性，如果能拿到直接返回，这样操作能减少Redis操作，提升性能。

最后SessionRepositoryFilter的doFilterInternal方法最后有一个finally代码块，这个代码块 `wrappedRequest.commitSession();`的功能就是将Session同步到Redis。

总结: 当请求进来的时候，SessionRepositoryFilter会先拦截到请求，将request和Response对象转换成SessionRepositoryRequestWrapper和SessionRepositoryResponseWrapper。后续当第一次调用request的getSession方法时，会调用到SessionRepositoryRequestWrapper的getSession方法。这个方法的逻辑是先从request的属性中查找，如果找不到；再查找一个key值是"SESSION"的cookie，通过这个cookie拿到sessionId去redis中查找，如果查不到，就直接创建一个RedisSession对象，同步到Redis中（同步的时机根据配置来）。

## 下载与显示

Content-Disposition 属性是作为对下载文件的一个标识字段

Content-Disposition属性有两种类型：

inline 和 attachment inline ：将文件内容直接显示在页面

 attachment：弹出对话框让用户下载具体例子：

```
response.addHeader("Content-Disposition","inline;filename=" + new String(filename.getBytes(),"utf-8"));
```

所以,在页面打开:

```
File file = new File("rfc1806.txt");  
String filename = file.getName();  
response.setHeader("Content-Type","text/plain");  
response.addHeader("Content-Disposition","inline;filename=" + new String(filename.getBytes(),"utf-8"));  
response.addHeader("Content-Length","" + file.length());  
```

> 前端可以使用< embed :src="xxx+'#toolbar=0&scrollbar=0&navpanes=0'" type="xxx" />

而弹出下载保存框:

```
File file = new File("rfc1806.txt");  
String filename = file.getName();  
response.setHeader("Content-Type","text/plain");  
response.addHeader("Content-Disposition","attachment;filename=" + new String(filename.getBytes(),"utf-8"));  
response.addHeader("Content-Length","" + file.length());  
```
