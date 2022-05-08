# Spring定制

## 配置文件

> 通过spring.profiles.include可指定导入application-xxx.yml属性配置文件

在Spring应用程序的environment中读取属性的时候，每个属性的唯一名称符合如下规则：

- 通过`.`分离各个元素
- 最后一个`.`将前缀与属性名称分开
- 必须是字母（a-z）和数字(0-9)
- 必须是小写字母
- 用连字符`-`来分隔单词
- 唯一允许的其他字符是`[`和`]`，用于List的索引
- 不能以数字开头

移除特殊字符并以**全小写**的方式进行**匹配**和加载, 以下等价

```
spring.jpa.databaseplatform=mysql
spring.jpa.database-platform=mysql
spring.jpa.databasePlatform=mysql
spring.JPA.database_platform=mysql
```

> 推荐使用全小写配合`-`分隔符的方式来配置, 并且通过Enviroment和@Value只能以该方式获取属性

List**类型**

在properties文件中使用`[]`来定位列表类型，比如：

```
spring.my-example.url[0]=http://example.com
spring.my-example.url[1]=http://spring.io
```

也支持使用**逗号**分割的配置方式，上面与下面的配置是等价的：

```
spring.my-example.url=http://example.com,http://spring.io
```

而在yaml文件中使用可以使用如下配置：

```
spring:
  my-example:
    url:
      - http://example.com
      - http://spring.io
```

也支持**逗号**分割的方式：

```
spring:
  my-example:
    url: http://example.com, http://spring.io
```

> 在Spring Boot 2.0中对于List类型的配置必须是连续的，不然会抛出`UnboundConfigurationPropertiesException`异常

Map**类型**

如果Map类型的key包含非字母数字和`-`的字符，需要用`[]`括起来，比如：

```
spring:
  my-example:
    '[foo.baz]': bar
```

> 另外, 属性文件中可通过`"@点分多级标签@"`获取pom文件内容

### @ConfigurationProperties

模块属性配置:

1.使用@Component注入.

```java
@Data
@Component
@ConfigurationProperties(prefix = "yyzx.properties")
public class AnnotationDesc {
}
```

2.使用@EnableConfigurationProperties注入.

```java
@Data
@ConfigurationProperties(prefix = "yyzx.properties")
public class AnnotationDesc {
  // 该书写方式，属性值注入成功
}
@Configuration
@EnableConfigurationProperties({
        AnnotationDesc.class,
        YyzxProperties2.class
})
public class SpringBootPlusConfig {

}
```

或者

```java
@Data
public class AnnotationDesc {
  // 该书写方式，属性值注入成功
}

@Configuration
@EnableConfigurationProperties
public class SpringBootPlusConfig {
    @Bean
    @ConfigurationProperties(prefix = "yyzx.properties")
    public AnnotationDesc annotationDesc() {
        return new AnnotationDesc();
    }
}
```

@ConfigurationProperties 也可以加在AnnotationDesc类定义上

3.或者在@Component类的每个字段上通过@Value获取(冗余)

### PropertySource

引入配置文件方式:

**1.@PropertySource**

```
@PropertySource("classpath:your.properties")
```

属性: 
1.value：配置文件的路径 
2.ignoreResourceNotFound：配置文件不存时在是否报错，默认是false
3.encoding：读取属性文件所使用的编码，通常使用UTF-8。

**2.EnvironmentPostProcessor**

在创建应用程序上下文之前，添加或者修改环境配置:

```java
public class CustomEnvironmentPostProcessor implements EnvironmentPostProcessor {
    private final Properties properties = new Properties();

    private String[] profiles = {
            "custom.properties",
    };

    @Override
    public void postProcessEnvironment(ConfigurableEnvironment environment, SpringApplication application) {
        for (String profile : profiles) {
            Resource resource = new ClassPathResource(profile);
            //将自定义配置文件属性加入到环境
            environment.getPropertySources().addLast(loadProfiles(resource));
        }
    }
    private PropertySource<?> loadProfiles(Resource resource) {
        if (!resource.exists()) {
            throw new IllegalArgumentException("file" + resource + "not exist");
        }
        try {
            properties.load(resource.getInputStream());
            return new PropertiesPropertySource(resource.getFilename(), properties);
        } catch (IOException ex) {
            throw new IllegalStateException("load resource exception" + resource, ex);
        }
    }
}
```

在META-INF下创建spring.factories，并且引入CustomEnvironmentPostProcessor 类

```
org.springframework.boot.env.EnvironmentPostProcessor=\
com.xxx.lzf.CustomEnvironmentPostProcessor
```

**3.PropertySourceLocator**

PropertySourceLocator接口支持扩展自定义配置加载到Environment中。

假设需要读取classpath下的my.json文件配置加载到spring环境变量中

```
public class JsonPropertySourceLocator implements PropertySourceLocator {
    private final static String DEFAULT_LOCATION = "classpath:my.json";

    @Override
    public PropertySource<?> locate(Environment environment) {
        // TODO 微服务配置中心远程RPC加载配置到spring环境变量中
        // 读取classpath下的my.json解析
        ResourceLoader resourceLoader = new DefaultResourceLoader(getClass().getClassLoader());
        Resource resource = resourceLoader.getResource(DEFAULT_LOCATION);
        if (resource == null) {
            return null;
        }
        return new MapPropertySource("myJson", mapPropertySource(resource));
    }

    //读取resource转换为map
    private Map<String, Object> mapPropertySource(Resource resource) {

        Map<String, Object> result = new HashMap<>();
        // 获取json格式的Map
        Map<String, Object> fileMap = JSONObject.parseObject(readFile(resource), Map.class);
        // 组装嵌套json
        processorMap("", result, fileMap);
    }
}
```

在classpath下META-INF/spring.factories文件定义:

```
org.springframework.cloud.bootstrap.BootstrapConfiguration=
\com.xxx.lzf.JsonPropertySourceLocator
```

使用这种方式非常灵活，只要在locate方法最后返回一个MapPropertySource对象即可，至于我们如何获取属性，这些都可以自己控制，例如我们实现从数据库读取配置来组装MapPropertySource，或者可以实现远程配置中心功能

**工作实践:数据库获取多个属性源**

如果要加载数据库中单个配置文件, 可在locate方法中创建一个实现自EnumerablePropertySource< JdbcTemplate>的类作为范围值, 通过传入jdbcTemplate给EnumerablePropertySource并通过其加载数据库中的配置文件.

如果要加载数据库中多个配置文件, 可在locate方法中创建一个CompositePropertySource对象作为返回值, 该对象表示多个PropertySource的组合, 提供了addPropertySource方法, 可创建多个实现自EnumerablePropertySource< JdbcTemplate>的类加入CompositePropertySource.

### Binder

springboot 1.x 获取属性必须通过Environment提供的接口:

```java
//判断是否包含键值
boolean containsProperty(String key);  
//获取属性值，如果获取不到返回null
String getProperty(String key);  
//获取属性值，如果获取不到返回缺省值
String getProperty(String key, String defaultValue); 
//获取属性对象；其转换和Converter有关，会根据sourceType和targetType查找转换器
<T> T getProperty(String key, Class<T> targetType);
```

springboot 2.x引入Binder用于对象与多个属性的绑定:

```java
//绑定对象
MailPropertiesC propertiesC = Binder.get(environment) //首先要绑定配置器
    //再将属性绑定到对象上
    .bind( "kaka.cream.mail-c", Bindable.of(MailPropertiesC.class) ).get(); //再获取实例

//绑定Map
Map<String,Object> propMap = Binder.get(environment)
    .bind( "fish.jdbc.datasource",Bindable.mapOf(String.class, Object.class) ).get();

//绑定List
List<String> list = Binder.get(environment)
    .bind( "kaka.cream.list",Bindable.listOf(String.class) ).get();    
```

## 事件模型

配置事件监听器或注解方式

```java
public class ApplicationPreparedEventListener implements ApplicationListener<ApplicationPreparedEvent> {
    @Override
    public void onApplicationEvent(ApplicationPreparedEvent event) {
        log.info("......ApplicationPreparedEvent......");
    }
}
```

在`/src/main/resources/META-INF/spring.factories`中添加:

```
org.springframework.context.ApplicationListener=  com.xxx.ApplicationPreparedEventListener
```

**将一个事件的结果作为另一个事件发布**

如需发布多个事件 , 可以将返回格式改为集合或数组

```java
@EventListener
public ListUpdateEvent handleBlockedListEvent(BlockedListEvent event) {
    // notify appropriate parties via notificationAddress and
    // then publish a ListUpdateEvent...
}
```

注解方式配置监听多个事件:

```java
@EventListener(classes = {MyEvent.class, ContextRefreshedEvent.class, ContextClosedEvent.class})
    public void onMyEventPublished(ApplicationEvent event) {
        if (event instanceof MyEvent) {
            logger.info("监听到MyEvent事件");
        }
        if (event instanceof ContextRefreshedEvent) {
            logger.info("监听到ContextRefreshedEvent事件");
        }
        if (event instanceof ContextClosedEvent) {
            logger.info("监听到ContextClosedEvent事件");
        }
    }
```

ApplicationListener接口方式也可以监听多个事件, 并可通过Order接口或注解指定顺序.

### 异步监听

可以根据需要注册任意数量的事件侦听器，但默认情况下，事件侦听器是同步接收事件的。即着 publishEvent方法会阻塞，直到所有侦听器都完成对事件的处理。

如果需要其他事件发布策略，`ApplicationEventMulticaster` 接口和 `SimpleApplicationEventMulticaster `实现。

1.单个异步:

首先需要在springboot入口类上通过`@EnableAsync`注解开启异步，然后在需要异步执行的监听器方法上使用`@Async`注解标注.

2.整体异步:

多播器在广播事件时，会先判断是否有指定executor，有的话通过executor执行监听器逻辑。所以可以通过指定executor的方式来让所有的监听方法都异步执行,

```java
@Configuration
public class AsyncEventConfigure { 
    //beanName即applicationEventMulticaster 用于覆盖默认的事件多播器
    @Bean(name = AbstractApplicationContext.APPLICATION_EVENT_MULTICASTER_BEAN_NAME)
    public ApplicationEventMulticaster simpleApplicationEventMulticaster() {
        SimpleApplicationEventMulticaster eventMulticaster = new SimpleApplicationEventMulticaster();
        eventMulticaster.setTaskExecutor(new SimpleAsyncTaskExecutor());
        return eventMulticaster;
    }
}
```

### SpEL条件监听

@EventListener注解还包含一个condition属性，可以配合SpEL表达式来条件化触发监听方法。修改MyEvent，添加一个boolean类型属性：

```java
public class MyEvent extends ApplicationEvent {
    private boolean flag;
    public boolean isFlag() {
        return flag;
    }
    public void setFlag(boolean flag) {
        this.flag = flag;
    }
    public MyEvent(Object source) {
        super(source);
    }
}
```

在发布事件时将该属性设置为false

```java
public void publishEvent() {
        logger.info("开始发布自定义事件MyEvent");
        MyEvent myEvent = new MyEvent(applicationContext);
        myEvent.setFlag(false); // 设置为false
        applicationEventPublisher.publishEvent(myEvent);
        logger.info("发布自定义事件MyEvent结束");
    }
```

条件监听实现:

```java
@Component
public class MyAnnotationEventListener implements Ordered {
    private final Logger logger = LoggerFactory.getLogger(this.getClass());
    //condition = "#event.flag"的含义为，当前event事件（这里为MyEvent）的flag属性为true的时候执行。
    @EventListener(classes = {MyEvent.class}, condition = "#event.flag")
    public void onMyEventPublished(ApplicationEvent event) {
        if (event instanceof MyEvent) {
            logger.info("监听到MyEvent事件");
        }
    }
    @Override
    public int getOrder() {
        return 1;
    }
}
```

## Bean生命周期

指定生命周期方法

1.`@Bean(initMethod = "init", destroyMethod = "destory")`

2.`@PostConstruct`和`@PreDestroy`

3.`InitializingBean`和`DisposableBean`接口

### BeanPostProcessor

默认是针对ioc容器中所有的Bean

`postProcessBeforeInitialization(Object bean, String beanName)`在Bean的初始化方法调用之前执行;

`postProcessAfterInitialization(Object bean, String beanName)`在Bean的初始化方法调用之后执行.

**InstantiationAwareBeanPostProcessor** 是BeanPostProcessor子接口,增加了3个方法:

`postProcessBeforeInstantiation(Class<?> beanClass, String beanName)`：在Bean实例化之前执行;

> 注意: 该方法返回null表示按默认方式进行实例化初始化Bean, 否则表示提前创建了Bean, 接着只会执行postProcessAfterInitialization方法

`postProcessAfterInstantiation(Object bean, String beanName)`：Bean实例化之后执行;

> 注意: 该方法在实例化之后初始化之前调用, 此时属性还没有被赋值,  一般用于自定义属性赋值.  方法返回false表示跳过Bean属性赋值, 并且InstantiationAwareBeanPostProcessor的postProcessProperties方法不会被调用

`postPorcessProperties(PropertiesValues values,Object bean,String beanName)`: Bean属性赋值后调用该方法;

> 注意: 该方法前提为postProcessAfterInstantiation返回true

综上,Bean在整个生命周期中上面内容的执行顺序为:

postProcessBeforeInstantiationfan方法 --> 调用无参构造函数 -->  postProcessAfterInstantiation方法  --> Bean属性赋值  -->  postProcessProperties  -->  postProcessBeforeInitialization方法  --> init方法  --> postProcessAfterInitialization方法 -->destory方法

### BeanFactoryPostProcessor

该接口包含一个方法:postProcessBeanFactory. 在所有的Bean定义已经被加载，但Bean的实例还没被创建（不包括BeanFactoryPostProcessor之类的特殊Bean）时执行。该方法通常用于修改bean的定义，Bean的属性值等，甚至可以在此快速初始化Bean。

**BeanDefinitionRegistryPostProcessor**继承自BeanFactoryPostProcessor，新增了postProcessBeanDefinitionRegistry方法：

执行在所有的Bean定义即将被加载，但Bean的实例还没被创建时。即postProcessBeanDefinitionRegistry方法先于postProcessBeanFactory方法。这个方法通常用于给IOC容器添加额外的组件。

执行顺序为:

postProcessBeanDefinitionRegistry方法 -->

实现了BeanFactoryPostProcessor的类的构造函数  -->

postProcessBeanFactory 方法 -->

其他的bean的构造函数

## Spring组件注册

在传统spring的xml配置中,指定组件扫描的方式为

```
<context:component-scan base-package=""></context:component-scan>
```

路径下所有被`@Controller`、`@Service`、`@Repository`和`@Component`注解标注的类都会被纳入IOC容器中。

### @ComponentScan

在springboot中 ,该注解默认扫描启动类所在的包下的类和子包下的类.

允许我们指定扫描策略,即指定哪些被扫描，哪些不被扫描，查看其源码可发现这两个属性：

```java
Filter[] includeFilters() default {};
Filter[] excludeFilters() default {};
```

其中Filter也是注解

```java
@Retention(RetentionPolicy.RUNTIME)
@Target({})
@interface Filter {
    FilterType type() default FilterType.ANNOTATION;

    @AliasFor("classes")
    Class<?>[] value() default {};

    @AliasFor("value")
    Class<?>[] classes() default {};
    String[] pattern() default {};
}
```

例如:

```java
@Configuration
@ComponentScan(value = "com.xxx.demo",
        excludeFilters = {
                @Filter(type = FilterType.ANNOTATION,
                        classes = {Controller.class, Repository.class}),
                @Filter(type = FilterType.ASSIGNABLE_TYPE, classes = User.class)
        })
public class WebConfig {

}
```

上面指定了两种排除扫描的规则：

1. 根据注解来排除（`type = FilterType.ANNOTATION`）,这些注解的类型为`classes = {Controller.class, Repository.class}`。即`Controller`和`Repository`注解标注的类不再被纳入到IOC容器中。
2. 根据指定类型类排除（`type = FilterType.ASSIGNABLE_TYPE`），排除类型为`User.class`，其子类，实现类都会被排除。

还有其他规则可查看FilterType源码, 其中

> 在Java 8之前,  可以使用`@ComponentScans`来配置多个`@ComponentScan`以实现多扫描规则配置; 
> 
> 在Java 8开始后,  `@ComponentScan`被新增的`@Repeatable(ComponentScans.class)`注解标注, 表示`@ComponentScan`可重复利用.

#### 自定义扫描策略

自定义扫描策略需要实现`org.springframework.core.type.filter.TypeFilter`接口, 包含两个参数:

1. `MetadataReader`：当前正在扫描的类的信息；
2. `MetadataReaderFactory`：可以通过它来获取其他类的信息。

```java
public class MyTypeFilter implements TypeFilter {
    @Override
    public boolean match(MetadataReader metadataReader, MetadataReaderFactory metadataReaderFactory) {
        // 获取当前正在扫描的类的注解信息
        AnnotationMetadata annotationMetadata = metadataReader.getAnnotationMetadata();
        // 获取当前正在扫描的类的类信息
        ClassMetadata classMetadata = metadataReader.getClassMetadata();
        // 获取当前正在扫描的类的路径等信息
        Resource resource = metadataReader.getResource();

        String className = classMetadata.getClassName();
        return StringUtils.hasText("er");
    }
}
```

上面指定了当被扫描的类名包含`er`时候，匹配成功.

```java
@Configuration
@ComponentScan(value = "com.xxx.demo",
        excludeFilters = {
            @Filter(type = FilterType.CUSTOM, classes = MyTypeFilter.class)
        })
public class WebConfig {
}
```

配合`excludeFilters`使用意指当被扫描的类名包含`er`时，该类不被纳入IOC容器中。

因为`User`，`UserMapper`，`UserService`和`UserController`等类的类名都包含`er`，所以它们都没有被纳入到IOC容器中。

### 条件注册组件

`@Conditional`指定组件注册的条件，即满足特定条件才将组件纳入到IOC容器中。

创建一个类，实现`Condition`接口,  包含一个`matches`方法,  两个入参:

1. `ConditionContext`：上下文信息. 如conditionContext.getBeanFactory(), conditionContext.getClassLoader(), conditionContext.getEnvironment() conditionContext.getRegistry() ;

2. `AnnotatedTypeMetadata`：注解信息.如((StandardMethodMetadata)metadata).getMethodName()可获取方法名.

```java
public class MyCondition implements Condition {
    @Override
    public boolean matches(ConditionContext context, AnnotatedTypeMetadata metadata) {
        String osName = context.getEnvironment().getProperty("os.name");
        return osName != null && osName.contains("Windows");
    }
}

@Bean
@Conditional(MyCondition.class)
public User user() {
    return new User("mrbird", 18);
}
```

在Windows环境下，User这个组件将被成功注册，如果是别的操作系统，这个组件将不会被注册到IOC容器中。

判断是否存在指定的bean:

```
context.getRegistry().containsBeanDefinition("xxx")
```

其他由@Conditional衍生的条件注册注解:

| **Conditions**               | **描述**                     |
| ---------------------------- | -------------------------- |
| @ConditionalOnBean           | 在存在某个bean的时候               |
| @ConditionalOnMissingBean    | 不存在某个bean的时候               |
| @ConditionalOnClass          | 当前classpath可以找到某个类型的类时     |
| @ConditionalOnMissingClass   | 当前classpath不可以找到某个类型的类时    |
| @ConditionalOnResource       | 当前classpath是否存在某个资源文件      |
| @ConditionalOnProperty       | 当前jvm是否包含某个系统属性为某个值        |
| @ConditionalOnWebApplication | 当前spring context是否是web应用程序 |
| @ConditionalOnExpression     | SpEL                       |

当@ConditionalOnBean和@ConditionalOnMissingBean放置在@Bean方法上时，目标类型默认为该方法的返回类型，如下面的示例所示：

```java
@Configuration(proxyBeanMethods = false)
public class MyAutoConfiguration {

    @Bean
    @ConditionalOnMissingBean
    public SomeService someService() {
        return new SomeService();
    }
}
```

`@Profile`可以根据不同的环境变量来注册不同的组件:

```java
public interface CalculateService {
    Integer sum(Integer... value);
}

@Service
@Profile("java7")
public class Java7CalculateServiceImpl implements CalculateService {
    @Override
    public Integer sum(Integer... value) {
        System.out.println("Java 7环境下执行");
        int result = 0;
        for (int i = 0; i <= value.length; i++) {
            result += i;
        }
        return result;
    }
}
@Service
@Profile("java8")
public class Java8CalculateServiceImpl implements CalculateService {
    @Override
    public Integer sum(Integer... value) {
        System.out.println("Java 8环境下执行");
        return Arrays.stream(value).reduce(0, Integer::sum);
    }
}

// 验证-------------
ConfigurableApplicationContext context1 = new SpringApplicationBuilder(DemoApplication.class)
                .web(WebApplicationType.NONE)
                .profiles("java8")
                //.profiles("java7")
                .run(args);

CalculateService service = context1.getBean(CalculateService.class);
System.out.println("求合结果： " + service.sum(1, 2, 3, 4, 5, 6, 7, 8, 9, 10));
```

> 其中当前活动的profile还可以通过注解配置, 如@ActiveProfile("java8")

### 导入组件(*)

1.@import, 该导入方式默认以全类名方式导入

```
@Import({Hello.class})
```

2.`ImportSelector`一次性导入多个组件, 包含`selectImports`方法，方法返回类的全类名数组（即需要导入到IOC容器中组件的全类名数组）:

```java
public class MyImportSelector implements ImportSelector {
    @Override
    public String[] selectImports(AnnotationMetadata importingClassMetadata) {
        return new String[]{
                "com.xxx.demo.domain.Apple",
                "com.xxx.demo.domain.Banana",
                "com.xxx.demo.domain.Watermelon"
        };
    }
}
```

然后在配置类上加入注解

```
@Import({MyImportSelector.class})
```

3.`ImportBeanDefinitionRegistrar`用于手动往IOC容器导入组件, 包含registerBeanDefinitions方法, 具有两个参数:

1. `AnnotationMetadata`：可以通过它获取到类的注解信息；
2. `BeanDefinitionRegistry`：Bean定义注册器.

```java
public class MyImportBeanDefinitionRegistrar implements ImportBeanDefinitionRegistrar {
    @Override
    public void registerBeanDefinitions(AnnotationMetadata importingClassMetadata, BeanDefinitionRegistry registry) {
        final String beanName = "strawberry";
        // 判断是否已存在
        boolean contain = registry.containsBeanDefinition(beanName);
        if (!contain) {  //不存在则创建
            RootBeanDefinition rootBeanDefinition = new RootBeanDefinition(Strawberry.class);
            registry.registerBeanDefinition(beanName, rootBeanDefinition);
        }
    }
}
```

然后在配置类上加入注解

```
@Import({MyImportBeanDefinitionRegistrar.class})
```

4.因为`BeanDefinitionRegistry`实际上就是ioc容器对象`DefaultListableBeanFactory`, 所以也可以直接通过其进行注册:

```java
@Component
public class PersonBeanRegiser implements BeanFactoryAware {
    private BeanFactory beanFactory;

    @Override
    public void setBeanFactory(BeanFactory beanFactory) throws BeansException {
        this.beanFactory = beanFactory;
    }

    @PostConstruct
    public void register(){
        BeanDefinitionBuilder builder = BeanDefinitionBuilder.genericBeanDefinition(Person.class);
        builder.addPropertyValue("name", "张三");
        builder.addPropertyValue("age", 20);
        // ioc容器DefaultListableBeanFactory实际上就是BeanDefinitionRegistry
        BeanDefinitionRegistry registry = (BeanDefinitionRegistry) this.beanFactory;
        registry.registerBeanDefinition("person", builder.getBeanDefinition());
    }   
}
```

#### BeanDefinitionBuilder

以ElasticSearch通过一个工厂类RestClientBeanBuilder(自定义)来提供RestClient和RestHighLevelClient工厂方法为例.

```java
//依赖非bean: 假如要注册的RestClientBeanBuilder构造函数需要传入一个Properties对象
Properties properties = ....;
BeanDefinitionBuilder builder = BeanDefinitionBuilder.genericBeanDefinition(RestClientBeanBuilder.class,()->{new RestClientBeanBuilder(properties)});
BeanDefinition beanDefinition = builder.getBeanDefinition();
// 设置为主优先级
beanDefinition.setPrimary(true);
// 注册
registry.registryBeanBefinition("restClientBeanBuilder",beanDefinition);

//依赖bean: 假如要注册的RestClientBuilder需要依赖RestClientBeanBuilder这个bean的工厂方法
BeanDefinitionBuilder builder1 = BeanDefinitionBuilder.genericBeanDefinition(RestClientBuilder.class);
// 注册该beanDefinition依赖的bean名称
builder1.addDependsOn("restClientBeanBuilder");
// 注册该beanDefinition在依赖的bean中的方法,即RestClientBeanBuilder的restClientBuilder方法. 和依赖的bean名称
builder1.setFactoryMethodOnBean("restClientBuilder","restClientBeanBuilder");
BeanDefinition beanDefinition1 = builder1.getBeanDefinition();
registry.registryBeanBefinition("restClientBuilder",beanDefinition1);

//依赖bean: 假如要注册的RestClient需要依赖RestClientBeanBuilder这个bean的工厂方法 且方法中使用到了restClientBuilder这个bean
BeanDefinitionBuilder builder2 = BeanDefinitionBuilder.genericBeanDefinition(RestClient.class);
// 注册该beanDefinition依赖的bean名称
builder2.addDependsOn("restClientBeanBuilder");
builder2.addDependsOn("restClientBuilder");
// 注册该beanDefinition在依赖的bean中的方法,即RestClientBeanBuilder的restClient方法. 和依赖的bean名称
builder2.setFactoryMethodOnBean("restClient","restClientBeanBuilder");
BeanDefinition beanDefinition2 = builder2.getBeanDefinition();
registry.registryBeanBefinition("restClient",beanDefinition2);

//依赖bean: 假如要注册的RestHighLevelClient需要依赖RestClientBeanBuilder这个bean的工厂方法 且方法中使用到了restClientBuilder这个bean
BeanDefinitionBuilder builder3 = BeanDefinitionBuilder.genericBeanDefinition(RestHighLevelClient.class);
// 注册该beanDefinition依赖的bean名称
builder3.addDependsOn("restClientBeanBuilder");
builder2.addDependsOn("restClientBuilder");
// 注册该beanDefinition在依赖的bean中的方法,即RestClientBeanBuilder的restClientBeanBuilder方法. 和依赖的bean名称
builder3.setFactoryMethodOnBean("restHighLevelClient","restClientBeanBuilder" )
BeanDefinition beanDefinition3 = builder3.getBeanDefinition();
registry.registryBeanBefinition("builder",beanDefinition3);
```

### FactoryBean

创建FactoryBean:

```java
public class CherryFactoryBean implements FactoryBean<Cherry> {
    @Override
    public Cherry getObject() {
        return new Cherry();
    }
    @Override
    public Class<?> getObjectType() {
        return Cherry.class;
    }
    @Override
    public boolean isSingleton() {
        return false;
    }
}
```

`getObject`返回需要注册的组件对象，`getObjectType`返回需要注册的组件类型，`isSingleton`指明该组件是否为单例。如果为多例的话，每次从容器中获取该组件都会调用其`getObject`方法。

在配置类中注册这个类：

```java
@Bean
public CherryFactoryBean cherryFactoryBean() {
    return new CherryFactoryBean();
}
```

从容器中获取：

```Java
ApplicationContext context = new AnnotationConfigApplicationContext(WebConfig.class);
Object cherry = context.getBean("cherryFactoryBean");
// 输出Cherry对象全类名
System.out.println(cherry.getClass());
Object cherryFactoryBean = context.getBean("&cherryFactoryBean");
// 输出CherryFactoryBean全类名
System.out.println(cherryFactoryBean.getClass());
```

## 自动装配

### 模块驱动

`@Enable`模块驱动在Spring Framework 3.1后开始支持。这里的模块通俗的来说就是一些为了实现某个功能的组件的集合。通过`@Enable`模块驱动，我们可以开启相应的模块功能。

`@Enable`模块驱动可以分为“注解驱动”和“接口编程”两种实现方式.

**1.注解驱动**, 该方式实际就是就是利用@Import导入@Configuration配置类:

配置类

```java
@Configuration
public class HelloWorldConfiguration {
    @Bean
    public String hello() {
        return "hello world";
    }
}
```

@Enable注解,在该注解类上通过`@Import`导入了刚刚创建的配置类。

```java
@Target({ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Import(HelloWorldConfiguration.class)
public @interface EnableHelloWorld {
}
```

**2.接口编程**, 该方式实际就是利用@Import导入ImportSelecter注册组件([导入组件(*)](中介绍过了)), 可参考@EnableCaching 注解:

```java
@Target({ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Import({CachingConfigurationSelector.class})
public @interface EnableCaching {
    boolean proxyTargetClass() default false;

    AdviceMode mode() default AdviceMode.PROXY;

    int order() default 2147483647;
}
```

### 工厂加载

Spring 工厂加载机制的实现类为`AutoConfigurationImportSelector`+`SpringFactoriesLoader`:

`SpringFactoriesLoader`类指定了`FACTORIES_RESOURCE_LOCATION = "META-INF/spring.factories"`.  该类的方法会读取META-INFO目录下的spring.factories配置文件.

查看sprinb-boot-autoconfigure-x.x.x.RELEASE.jar下的该文件可看到内容为:

```
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
...很多配置类
```

当启动类被`@EnableAutoConfiguration`标注后(springboot主启动类注解自带),`AutoConfigurationImportSelector`会去尝试注册这些指定配置类的Bean, 这些Bean注册方法通常带有条件注解`@ComditionOnXXX`, 比如限制在包含某个类的时候才进行注册, 这也是为什么ioc实际注册的配置类并不是全部spring.factories下的内容, 这是SpringBoot实现自动装配的关键.
