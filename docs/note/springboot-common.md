

# Spring

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

1. `AnnotatedTypeMetadata`：注解信息.如((StandardMethodMetadata)metadata).getMethodName()可获取方法名.

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

| **Conditions**               | **描述**                              |
| ---------------------------- | ------------------------------------- |
| @ConditionalOnBean           | 在存在某个bean的时候                  |
| @ConditionalOnMissingBean    | 不存在某个bean的时候                  |
| @ConditionalOnClass          | 当前classpath可以找到某个类型的类时   |
| @ConditionalOnMissingClass   | 当前classpath不可以找到某个类型的类时 |
| @ConditionalOnResource       | 当前classpath是否存在某个资源文件     |
| @ConditionalOnProperty       | 当前jvm是否包含某个系统属性为某个值   |
| @ConditionalOnWebApplication | 当前spring context是否是web应用程序   |
| @ConditionalOnExpression     | SpEL                                  |

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

## AOP

Spring 中的 AOP 模块中：如果目标对象实现了接口，则默认采用 JDK 动态代理，否则采用 CGLIB 动态代理。

- Pointcut（切点），指定在什么情况下才执行 AOP，例如方法被打上某个注解的时候
- JoinPoint（连接点），程序运行中的执行点，例如一个方法的执行或是一个异常的处理；并且在 Spring AOP 中，只有方法连接点
- Advice（增强），对连接点进行增强（代理）：在方法调用前、调用后 或者 抛出异常时，进行额外的处理
- Aspect（切面），由 Pointcut 和 Advice 组成，可理解为：要在什么情况下（Pointcut）对哪个目标（JoinPoint）做什么样的增强（Advice）

回顾代理模式:

1.java proxy

```java
Proxy.newProxyInstance(ClassLoader loader, Class<?>[] interfaces, InvocationHandler h)

loader :类加载器，用于加载代理对象。
interfaces : 被代理类实现的一些接口；
h : 实现了 InvocationHandler 接口的对象；
```

2.cglib(多种拦截器接口自查资料)

```java
Enhancer enhancer = new Enhancer();
enhancer.setClassLoader(SampleClass.class.getClassLoader())
enhancer.setSuperclass(SampleClass.class);
enhancer.setCallback(new MethodInterceptor() {
    @Override
    public Object intercept(Object obj, Method method, Object[] args, MethodProxy proxy) throws Throwable {
            System.out.println("before method run...");
            Object result = proxy.invokeSuper(obj, args);
            System.out.println("after method run...");
            return result;
        }
    });
SampleClass sample = (SampleClass) enhancer.create();
       sample.test();
```

切面包含了5个通知方法：

- 前置通知（@Before）：在目标方法被调用之前调用通知功能；
- 后置通知（@After）：在目标方法完成之后调用通知，此时不会关心方法的输出是什么；
- 返回通知（@AfterReturning）：在目标方法成功执行之后调用通知；
- 异常通知（@AfterThrowing）：在目标方法抛出异常后调用通知。
- 环绕通知  (@Aroud)  :  在目标方法执行之前和之后都可以执行额外代码的通知。

这几个通知的顺序在不同的Spring版本中有所不同：

1. Spring4.x

   - 正常情况：环绕前置  —->  @Before —-> 目标方法 —-> 环绕返回  -->环绕最终(finally)—-> @After —-> @AfterReturning
   - 异常情况：环绕前置  —-> @Before —-> 目标方法 —-> 环绕异常  -->环绕最终(finally)—-> @After —-> @AfterThrowing

2. Spring5.x

   - 正常情况：环绕前置  —-> @Before —-> 目标方法 —-> @AfterReturning —-> @After  —-> 环绕返回  -->环绕最终(finally)
   - 异常情况：环绕前置  —-> @Before —-> 目标方法 —-> @AfterThrowing —-> @After  —-> 环绕异常  -->环绕最终(finally)

   在有多个切面时,可以通过@Order或Ordered接口指定顺序.

   基于注解的方式实现AOP需要在配置类中添加注解@EnableAspectJAutoProxy

   ```java
   // exposeProxy = true表示通过aop框架暴露该代理对象到AOP上下文(Thr中,AopContext能够访问
   //(通过AopContext的ThreadLocal实现)
   @EnableAspectJAutoProxy(exposeProxy = true)
   //可以调用代理对象的方法
   //应用场景,当想在serivce没有@Transactional的方法中调用带有@Transactional注解的方法时,要使@Transactional生效时,使用this.xxx调用的不是代理之后的方法,而下面的方式可以调用代理方法
   ((T) AopContext.currentProxy()).xxx;
   ```
   
   因为spring采用动态代理机制来实现事务控制，而动态代理(jdk代理,因为service实现了接口)最终都是要调用原始对象的，而原始对象在去调用方法时，是不会再触发代理了.
   
   可通过注解的proxyTargetClass=true指定使用cglib代理方式.
   
   >注意: SpringBoot2.x开始为了避免使用JDK代理出现的各种问题, 如在非@Transaction方法中调用@Transaction方法事务不会生效、只能通过接口自动注入等问题, 默认使用的AOP实现为CGLIB!!!

> pointcut表达式支持更直观的操作, 如
>
> ```
> @Pointcut("com.xyz.myapp.CommonPointcuts.dataAccessOperation() && args(account,..)")
> private void accountDataAccessOperation(Account account) {}
> 
> @Before("accountDataAccessOperation(account)")
> public void validateAccount(Account account) {
>     // ...
> }
> ```
>
> args（account,..） 部分有两个用途。首先，它将匹配限制为仅那些方法执行，其中方法至少采用一个参数，并且传递给该参数的是 Account 的实例。其次，它通过 account 参数使实际的 Account 对象可用于advice。
>
> 又或者
>
> ```
> @Before("com.xyz.lib.Pointcuts.anyPublicMethod() && @annotation(auditable)")
> public void audit(Auditable auditable) {
>     AuditCode code = auditable.value();
> }
> ```
>

JointPoint使用

```java
//通过joinPoint获取方法
Signature signature = joinPoint.getSignature();
MethodSignature methodSignature = (MethodSignature) signature;
Method method = methodSignature.getMethod();
//获取目标对象
joinPoint.getTarget().getClass()
//通过joinPoint获取方法参数 s
joinPoint.getArgs()
//获取代理对象
joinPoint.getThis():
```

### 脚手架

> 摘自阿里技术微信公众号文章

**定义方法增强处理器**

我们先定义出 ”代理“ 的抽象：方法增强处理器 MethodAdviceHandler 。之后我们定义的每一个注解，都绑定一个对应的 MethodAdviceHandler 的实现类，当目标方法被代理时，由对应的 MethodAdviceHandler 的实现类来处理该方法的代理访问。

```

/**
 * 方法增强处理器
 *
 * @param <R> 目标方法返回值的类型
 */
public interface MethodAdviceHandler<R> {

    /**
     * 目标方法执行之前的判断，判断目标方法是否允许执行。默认返回 true，即 默认允许执行
     *
     * @param point 目标方法的连接点
     * @return 返回 true 则表示允许调用目标方法；返回 false 则表示禁止调用目标方法。
     * 当返回 false 时，此时会先调用 getOnForbid 方法获得被禁止执行时的返回值，然后
     * 调用 onComplete 方法结束切面
     */
    default boolean onBefore(ProceedingJoinPoint point) { return true; }

    /**
     * 禁止调用目标方法时（即 onBefore 返回 false），执行该方法获得返回值，默认返回 null
     *
     * @param point 目标方法的连接点
     * @return 禁止调用目标方法时的返回值
     */
    default R getOnForbid(ProceedingJoinPoint point) { return null; }

    /**
     * 目标方法抛出异常时，执行的动作
     *
     * @param point 目标方法的连接点
     * @param e     抛出的异常
     */
    void onThrow(ProceedingJoinPoint point, Throwable e);

    /**
     * 获得抛出异常时的返回值，默认返回 null
     *
     * @param point 目标方法的连接点
     * @param e     抛出的异常
     * @return 抛出异常时的返回值
     */
    default R getOnThrow(ProceedingJoinPoint point, Throwable e) { return null; }

    /**
     * 目标方法完成时，执行的动作
     *
     * @param point     目标方法的连接点
     * @param startTime 执行的开始时间
     * @param permitted 目标方法是否被允许执行
     * @param thrown    目标方法执行时是否抛出异常
     * @param result    执行获得的结果
     */
    default void onComplete(ProceedingJoinPoint point, long startTime, boolean permitted, boolean thrown, Object result) { }
}
```

为了方便 MethodAdviceHandler 的使用，我们定义一个抽象类，提供一些常用的方法。

```

public abstract class BaseMethodAdviceHandler<R> implements MethodAdviceHandler<R> {

    protected final Logger logger = LoggerFactory.getLogger(this.getClass());

    /**
     * 抛出异常时候的默认处理
     */
    @Override
    public void onThrow(ProceedingJoinPoint point, Throwable e) {
        String methodDesc = getMethodDesc(point);
        Object[] args = point.getArgs();
        logger.error("{} 执行时出错，入参={}", methodDesc, JSON.toJSONString(args, true), e);
    }

    /**
     * 获得被代理的方法
     *
     * @param point 连接点
     * @return 代理的方法
     */
    protected Method getTargetMethod(ProceedingJoinPoint point) {
        // 获得方法签名
        Signature signature = point.getSignature();
        // Spring AOP 只有方法连接点，所以 Signature 一定是 MethodSignature
        return ((MethodSignature) signature).getMethod();
    }

    /**
     * 获得方法描述，目标类名.方法名
     *
     * @param point 连接点
     * @return 目标类名.执行方法名
     */
    protected String getMethodDesc(ProceedingJoinPoint point) {
        // 获得被代理的类
        Object target = point.getTarget();
        String className = target.getClass().getSimpleName();

        Signature signature = point.getSignature();
        String methodName = signature.getName();

        return className + "." + methodName;
    }
}
```

**定义方法切面的抽象**

同理，将方法切面的公共逻辑抽取出来，定义出方法切面的抽象 —— 后续每定义一个注解，对应的方法切面继承自这个抽象类就好.

```

/**
 * 方法切面抽象类，由子类来指定切点和绑定的方法增强处理器的类型
 */
public abstract class BaseMethodAspect implements ApplicationContextAware {

    /**
     * 切点，通过 @Pointcut 指定相关的注解
     */
    protected abstract void pointcut();

    /**
     * 对目标方法进行环绕增强处理，子类需通过 pointcut() 方法指定切点
     *
     * @param point 连接点
     * @return 方法执行返回值
     */
    @Around("pointcut()")
    public Object advice(ProceedingJoinPoint point) {
        // 获得切面绑定的方法增强处理器的类型
        Class<? extends MethodAdviceHandler<?>> handlerType = getAdviceHandlerType();
        // 从 Spring 上下文中获得方法增强处理器的实现 Bean
        MethodAdviceHandler<?> adviceHandler = appContext.getBean(handlerType);
        // 使用方法增强处理器对目标方法进行增强处理
        return advice(point, adviceHandler);
    }

    /**
     * 获得切面绑定的方法增强处理器的类型
     */
    protected abstract Class<? extends MethodAdviceHandler<?>> getAdviceHandlerType();

    /**
     * 使用方法增强处理器增强被注解的方法
     *
     * @param point   连接点
     * @param handler 切面处理器
     * @return 方法执行返回值
     */
    private Object advice(ProceedingJoinPoint point, MethodAdviceHandler<?> handler) {
        // 执行之前，返回是否被允许执行
        boolean permitted = handler.onBefore(point);

        // 方法返回值
        Object result;
        // 是否抛出了异常
        boolean thrown = false;
        // 开始执行的时间
        long startTime = System.currentTimeMillis();

        // 目标方法被允许执行
        if (permitted) {
            try {
                // 执行目标方法
                result = point.proceed();
            } catch (Throwable e) {
                // 抛出异常
                thrown = true;
                // 处理异常
                handler.onThrow(point, e);
                // 抛出异常时的返回值
                result = handler.getOnThrow(point, e);
            }
        }
        // 目标方法被禁止执行
        else {
            // 禁止执行时的返回值
            result = handler.getOnForbid(point);
        }

        // 结束
        handler.onComplete(point, startTime, permitted, thrown, result);

        return result;
    }

    private ApplicationContext appContext;

    @Override
    public void setApplicationContext(ApplicationContext applicationContext) throws BeansException {
        appContext = applicationContext;
    }
}
```

此时，我们基于 AOP 的代理模式小架子就已经搭好了。之所以需要这个小架子，是为了后续新增注解时，能够进行横向的扩展：每次新增一个注解（XxxAnno），只需要实现一个新的方法增强处理器（XxxHandler）和新的方法切面 （XxxAspect），而不会修改现有代码，从而完美符合 **对修改关闭，对扩展开放** 设计模式理念。

**使用**:

**定义一个注解**

```
/**
 * 用于产生调用记录的注解，会记录下方法的出入参、调用时长
 */
@Target({ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
public @interface InvokeRecordAnno {
    /**
     * 调用说明
     */
    String value() default "";
}
```

**方法增强处理器的实现**

```

@Component
public class InvokeRecordHandler extends BaseMethodAdviceHandler<Object> {

    /**
     * 记录方法出入参和调用时长
     */
    @Override
    public void onComplete(ProceedingJoinPoint point, long startTime, boolean permitted, boolean thrown, Object result) {
        String methodDesc = getMethodDesc(point);
        Object[] args = point.getArgs();
        long costTime = System.currentTimeMillis() - startTime;

        logger.warn("\n{} 执行结束，耗时={}ms，入参={}, 出参={}",
                    methodDesc, costTime,
                    JSON.toJSONString(args, true),
                    JSON.toJSONString(result, true));
    }

    @Override
    protected String getMethodDesc(ProceedingJoinPoint point) {
        Method targetMethod = getTargetMethod(point);
        // 获得方法上的 InvokeRecordAnno
        InvokeRecordAnno anno = targetMethod.getAnnotation(InvokeRecordAnno.class);
        String description = anno.value();

        // 如果没有指定方法说明，那么使用默认的方法说明
        if (StringUtils.isBlank(description)) {
            description = super.getMethodDesc(point);
        }

        return description;
    }
}
```

**方法切面的实现**

```

@Aspect
@Order(1)
@Component
public class InvokeRecordAspect extends BaseMethodAspect {

    /**
     * 指定切点（处理打上 InvokeRecordAnno 的方法）
     */
    @Override
    @Pointcut("@annotation(xyz.mizhoux.experiment.proxy.anno.InvokeRecordAnno)")
    protected void pointcut() { }

    /**
     * 指定该切面绑定的方法切面处理器为 InvokeRecordHandler
     */
    @Override
    protected Class<? extends MethodAspectHandler<?>> getHandlerType() {
        return InvokeRecordHandler.class;
    }
}
```

**其他**

1.@Aspect 用来告诉 Spring 这是一个切面，然后 Spring 在启动会时扫描 @Pointcut 匹配的方法，然后对这些目标方法进行织入处理：即使用切面中打上 @Around 的方法来对目标方法进行增强处理。

2.@Order 是用来标记这个切面应该在哪一层，数字越小，则在越外层（越先进入，越后结束） —— 方法调用记录的切面很明显应该在大气层（小编：王者荣耀术语，即最外层），因为方法调用记录的切面应该最后结束，所以我们给一个小点的数字。

**测试**

```
@RestController
@RequestMapping("proxy")
public class ProxyTestController {

    @GetMapping("test")
    @InvokeRecordAnno("测试代理模式")
    public Map<String, Object> testProxy(@RequestParam String biz,
                                         @RequestParam String param) {
        Map<String, Object> result = new HashMap<>(4);
        result.put("id", 123);
        result.put("nick", "之叶");

        return result;
    }
}
```

**扩展**

假设我们要在目标方法抛出异常时进行处理：抛出异常时，把异常信息异步发送到邮箱或者钉钉，然后根据方法的返回值类型，返回相应的错误响应。

**定义相应的注解**

```
/**
 * 用于异常处理的注解
 */
@Target({ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
public @interface ExceptionHandleAnno { }
```

**实现方法增强处理器**

```

@Component
public class ExceptionHandleHandler extends BaseMethodAdviceHandler<Object> {

    /**
     * 抛出异常时的处理
     */
    @Override
    public void onThrow(ProceedingJoinPoint point, Throwable e) {
        super.onThrow(point, e);
        // 发送异常到邮箱或者钉钉的逻辑
    }

    /**
     * 抛出异常时的返回值
     */
    @Override
    public Object getOnThrow(ProceedingJoinPoint point, Throwable e) {
        // 获得返回值类型
        Class<?> returnType = getTargetMethod(point).getReturnType();

        // 如果返回值类型是 Map 或者其子类
        if (Map.class.isAssignableFrom(returnType)) {
            Map<String, Object> result = new HashMap<>(4);
            result.put("success", false);
            result.put("message", "调用出错");

            return result;
        }

        return null;
    }
}
```

如果返回值的类型是个 Map，那么我们就返回调用出错情况下的对应 Map 实例（真实情况一般是返回业务系统中的 Response）。

**实现方法切面**

```
@Aspect
@Order(10)
@Component
public class ExceptionHandleAspect extends BaseMethodAspect {

    /**
     * 指定切点（处理打上 ExceptionHandleAnno 的方法）
     */
    @Override
    @Pointcut("@annotation(xyz.mizhoux.experiment.proxy.anno.ExceptionHandleAnno)")
    protected void pointcut() { }

    /**
     * 指定该切面绑定的方法切面处理器为 ExceptionHandleHandler
     */
    @Override
    protected Class<? extends MethodAdviceHandler<?>> getAdviceHandlerType() {
        return ExceptionHandleHandler.class;
    }
}
```

异常处理一般是非常内层的切面，所以我们将@Order 设置为 10，让 ExceptionHandleAspect 在 InvokeRecordAspect 更内层（即之后进入、之前结束），从而外层的 InvokeRecordAspect 也可以记录到抛出异常时的返回值。修改测试用的方法，加上 @ExceptionHandleAnno：

```
@RestController
@RequestMapping("proxy")
public class ProxyTestController {

    @GetMapping("test")
    @ExceptionHandleAnno
    @InvokeRecordAnno("测试代理模式")
    public Map<String, Object> testProxy(@RequestParam String biz,
                                         @RequestParam String param) {
        if (biz.equals("abc")) {
            throw new IllegalArgumentException("非法的 biz=" + biz);
        }

        Map<String, Object> result = new HashMap<>(4);
        result.put("id", 123);
        result.put("nick", "之叶");

        return result;
    }
}
```

测试结果：异常处理的切面先结束，方法调用记录的切面后结束。

**其他问题**

问：抛出异常时， InvokeRecordHandler 的 onThrow 方法没有执行，为什么呢？

答：因为 InvokeRecordAspect 比 ExceptionHandleAspect 在更外层，外层的 InvokeRecordAspect 在执行时，执行的已经是内层的 ExceptionHandleAspect 代理过的方法，而对应的 ExceptionHandleHandler 已经把异常 “消化” 了，即 ExceptionHandleAspect 代理过的方法已经不会再抛出异常。

问:  如果我们要 限制单位时间内方法的调用次数，比如 3s 内用户只能提交表单 1 次，似乎也可以通过这个代理模式的套路来实现。

答： 首先定义好注解（注解可以包含单位时间、最大调用次数等参数），然后在方法切面处理器的 onBefore 方法里面，使用缓存记录下单位时间内用户的提交次数，如果超出最大调用次数，返回 false，那么目标方法就不被允许调用了；然后在 getOnForbid 的方法里面，返回这种情况下的响应。

## @Async

`@Async`注解就能将原来的同步函数变为异步函数,

为了让@Async注解能够生效，还需要配置@EnableAsync

```java
@Component
public class Task {
    public static Random random =new Random();
    @Async
    public void doTaskOne() throws Exception {
        System.out.println("开始做任务一");
        long start = System.currentTimeMillis();
        Thread.sleep(random.nextInt(10000));
        long end = System.currentTimeMillis();
        System.out.println("完成任务一，耗时：" + (end - start) + "毫秒");
    }
    @Async
    public void doTaskTwo() throws Exception {
        System.out.println("开始做任务二");
        long start = System.currentTimeMillis();
        Thread.sleep(random.nextInt(10000));
        long end = System.currentTimeMillis();
        System.out.println("完成任务二，耗时：" + (end - start) + "毫秒");
    }
}
```

```java
@SpringBootApplication
@EnableAsync
public class Application {
	public static void main(String[] args) {
		SpringApplication.run(Application.class, args);
	}
}

@RunWith(SpringJUnit4ClassRunner.class)
@SpringApplicationConfiguration(classes = Application.class)
public class ApplicationTests {
	@Autowired
	private Task task;
	@Test
	public void test() throws Exception {
		task.doTaskOne();
		task.doTaskTwo();
	}
}
```

此时反复执行单元测试会遇到各种不同的结果，比如：

- 没有任何任务相关的输出
- 有部分任务相关的输出
- 乱序的任务相关的输出

原因是主程序在异步调用三个函数之后并不会理会这三个函数是否执行完成了，由于没有其他需要执行的内容，所以程序就自动结束了，导致了不完整或是没有输出任务相关内容的情况。

> @Async所修饰的函数不要定义为static类型，这样异步调用不会生效

### 异步回调

为了让异步函数能正常结束，可使用`Future<T>`来返回异步调用的结果.

```java
@Component
public class Task {
    public static Random random =new Random();
    @Async
    public Future<String> doTaskOne() throws Exception {
        System.out.println("开始做任务一");
        long start = System.currentTimeMillis();
        Thread.sleep(random.nextInt(10000));
        long end = System.currentTimeMillis();
        System.out.println("完成任务一，耗时：" + (end - start) + "毫秒");
        return new AsyncResult<>("任务一完成");
    }
}
```

```java
@Test
public void test() throws Exception {
	long start = System.currentTimeMillis();

	Future<String> task1 = task.doTaskOne();
	Future<String> task2 = task.doTaskTwo();

	while(true) {
		if(task1.isDone() && task2.isDone()) {
			// 二个任务都调用完成，退出循环等待
			break;
		}
		Thread.sleep(1000);
	}
	long end = System.currentTimeMillis();
	System.out.println("任务全部完成，总耗时：" + (end - start) + "毫秒");
}
```

### 配置默认线程池

异步任务存在的问题

```java
@RestController
public class HelloController {
    @Autowired
    private AsyncTasks asyncTasks;
        
    @GetMapping("/hello")
    public String hello() {
        // 将可以并行的处理逻辑，拆分成三个异步任务同时执行
        CompletableFuture<String> task1 = asyncTasks.doTaskOne();
        CompletableFuture<String> task2 = asyncTasks.doTaskTwo();
        CompletableFuture<String> task3 = asyncTasks.doTaskThree();
        
        CompletableFuture.allOf(task1, task2, task3).join();
        return "Hello World";
    }
}
```

多次调用接口会频繁创建线程执行可能会出现内存溢出。

因为Spring Boot默认用于异步任务的线程池是这样配置的：

```java
public static class Pool{
	private int queueCapacity = Integer.MAX_VALUE; //缓冲队列的容量
	private int maxSize = Integer.MAX_VALUE; //允许的最大线程数
	//...
}
```

配置默认线程池:

```properties
spring.task.execution.pool.core-size=2
spring.task.execution.pool.max-size=5
spring.task.execution.pool.queue-capacity=10
spring.task.execution.pool.keep-alive=60s
spring.task.execution.pool.allow-core-thread-timeout=true
spring.task.execution.thread-name-prefix=task-
```

参数解释

- `spring.task.execution.pool.core-size`：线程池创建时的初始化线程数，默认为8
- `spring.task.execution.pool.max-size`：线程池的最大线程数，默认为int最大值
- `spring.task.execution.pool.queue-capacity`：用来缓冲执行任务的队列，默认为int最大值
- `spring.task.execution.pool.keep-alive`：线程终止前允许保持空闲的时间
- `spring.task.execution.pool.allow-core-thread-timeout`：是否允许核心线程超时
- `spring.task.execution.shutdown.await-termination`：是否等待剩余任务完成后才关闭应用
- `spring.task.execution.shutdown.await-termination-period`：等待剩余任务完成的最大时间
- `spring.task.execution.thread-name-prefix`：线程名的前缀，设置好了之后可以方便我们在日志中查看处理任务所在的线程池

### 自定义线程池

由于@Async默认每个任务创建一个线程, 造成资源浪费并可能OOM, 所以可通过自定义线程池的方式:

```java
@SpringBootApplication
public class Application {
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }

    @EnableAsync
    @Configuration
    class TaskPoolConfig {
        @Bean("taskExecutor")
        public Executor taskExecutor() {
            ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
            executor.setCorePoolSize(10);
            executor.setMaxPoolSize(20);
            executor.setQueueCapacity(200);
            executor.setKeepAliveSeconds(60);
            executor.setThreadNamePrefix("taskExecutor-");
            executor.setRejectedExecutionHandler(new ThreadPoolExecutor.CallerRunsPolicy());
            return executor;
        }
    }
}
```

参数:

- `corePoolSize`：线程池核心线程的数量，默认值为1（这就是默认情况下的异步线程池配置使得线程不能被重用的原因）。
- `maxPoolSize`：线程池维护的线程的最大数量，只有当核心线程都被用完并且缓冲队列满后，才会开始申超过请核心线程数的线程，默认值为`Integer.MAX_VALUE`。
- `queueCapacity`：缓冲队列。
- `keepAliveSeconds`：超出核心线程数外的线程在空闲时候的最大存活时间，默认为60秒。
- `threadNamePrefix`：线程名前缀。
- `waitForTasksToCompleteOnShutdown`：是否等待所有线程执行完毕才关闭线程池，默认值为false。
- `awaitTerminationSeconds`：`waitForTasksToCompleteOnShutdown`的等待的时长，默认值为0，即不等待。
- `rejectedExecutionHandler`：当没有线程可以被使用时的处理策略（拒绝任务），默认策略为`abortPolicy`，包含下面四种策略：

```
callerRunsPolicy：用于被拒绝任务的处理程序，它直接在 execute 方法的调用线程中运行被拒绝的任务；如果执行程序已关闭，则会丢弃该任务。
abortPolicy：直接抛出java.util.concurrent.RejectedExecutionException异常。
discardOldestPolicy：当线程池中的数量等于最大线程数时、抛弃线程池中最后一个要执行的任务，并执行新传入的任务。
discardPolicy：当线程池中的数量等于最大线程数时，不做任何动作。
```

在`@Async`注解中指定线程池名让异步调用的执行任务使用这个线程池中的资源来运行:

```java
@Slf4j
@Component
public class Task {
    public static Random random = new Random();
    @Async("taskExecutor")
    public void doTaskOne() throws Exception {
        log.info("开始做任务一");
        long start = System.currentTimeMillis();
        Thread.sleep(random.nextInt(10000));
        long end = System.currentTimeMillis();
        log.info("完成任务一，耗时：" + (end - start) + "毫秒");
    }
    @Async("taskExecutor")
    public void doTaskTwo() throws Exception {
        log.info("开始做任务二");
        long start = System.currentTimeMillis();
        Thread.sleep(random.nextInt(10000));
        long end = System.currentTimeMillis();
        log.info("完成任务二，耗时：" + (end - start) + "毫秒");
    }
}
```

测试: 这里主线程使用join方法等待子线程执行完毕

```java
@RunWith(SpringJUnit4ClassRunner.class)
@SpringBootTest
public class ApplicationTests {
    @Autowired
    private Task task;

    @Test
    public void test() throws Exception {
        task.doTaskOne();
        task.doTaskTwo();

        Thread.currentThread().join();
    }
}
```

### 线程池隔离

以创建多个线程池,通过@Async指定线程池名称使得不同的任务使用不同的线程池.

```java
@EnableAsync
@Configuration
public class TaskPoolConfig {

    @Bean
    public Executor taskExecutor1() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(2);
        executor.setMaxPoolSize(2);
        executor.setQueueCapacity(10);
        executor.setKeepAliveSeconds(60);
        executor.setThreadNamePrefix("executor-1-");
        executor.setRejectedExecutionHandler(new ThreadPoolExecutor.CallerRunsPolicy());
        return executor;
    }

    @Bean
    public Executor taskExecutor2() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(2);
        executor.setMaxPoolSize(2);
        executor.setQueueCapacity(10);
        executor.setKeepAliveSeconds(60);
        executor.setThreadNamePrefix("executor-2-");
        executor.setRejectedExecutionHandler(new ThreadPoolExecutor.CallerRunsPolicy());
        return executor;
    }
}
```

```java
@Slf4j
@Component
public class AsyncTasks {

    public static Random random = new Random();

    @Async("taskExecutor1")
    public CompletableFuture<String> doTaskOne(String taskNo) throws Exception {
        log.info("开始任务：{}", taskNo);
        long start = System.currentTimeMillis();
        Thread.sleep(random.nextInt(10000));
        long end = System.currentTimeMillis();
        log.info("完成任务：{}，耗时：{} 毫秒", taskNo, end - start);
        return CompletableFuture.completedFuture("任务完成");
    }

    @Async("taskExecutor2")
    public CompletableFuture<String> doTaskTwo(String taskNo) throws Exception {
        log.info("开始任务：{}", taskNo);
        long start = System.currentTimeMillis();
        Thread.sleep(random.nextInt(10000));
        long end = System.currentTimeMillis();
        log.info("完成任务：{}，耗时：{} 毫秒", taskNo, end - start);
        return CompletableFuture.completedFuture("任务完成");
    }
}
```

```java
@Slf4j
@SpringBootTest
public class Chapter77ApplicationTests {

    @Autowired
    private AsyncTasks asyncTasks;

    @Test
    public void test() throws Exception {
        long start = System.currentTimeMillis();

        // 线程池1
        CompletableFuture<String> task1 = asyncTasks.doTaskOne("1");
        CompletableFuture<String> task2 = asyncTasks.doTaskOne("2");
        CompletableFuture<String> task3 = asyncTasks.doTaskOne("3");

        // 线程池2
        CompletableFuture<String> task4 = asyncTasks.doTaskTwo("4");
        CompletableFuture<String> task5 = asyncTasks.doTaskTwo("5");
        CompletableFuture<String> task6 = asyncTasks.doTaskTwo("6");

        // 一起执行
        CompletableFuture.allOf(task1, task2, task3, task4, task5, task6).join();

        long end = System.currentTimeMillis();

        log.info("任务全部完成，总耗时：" + (end - start) + "毫秒");
    }
}
```

### 修改默认线程池

> Spring中使用的@Async注解，底层是基于SimpleAsyncTaskExecutor去执行任务，只不过它不是线程池，而是每次都新开一个线程。

实现AsyncConfigurer,  只需要加`@Async`注解就可以，不用在注解属性指定线程池。

```java
@Slf4j
@Configuration
public class NativeAsyncTaskExecutePool implements AsyncConfigurer{


    //注入配置类
    @Autowired
    TaskThreadPoolConfig config;

    @Override
    public Executor getAsyncExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        //核心线程池大小
        executor.setCorePoolSize(config.getCorePoolSize());
        //最大线程数
        executor.setMaxPoolSize(config.getMaxPoolSize());
        //队列容量
        executor.setQueueCapacity(config.getQueueCapacity());
        //活跃时间
        executor.setKeepAliveSeconds(config.getKeepAliveSeconds());
        //线程名字前缀
        executor.setThreadNamePrefix("MyExecutor-");

        // setRejectedExecutionHandler：当pool已经达到max size的时候，如何处理新任务
        // CallerRunsPolicy：不在新线程中执行任务，而是由调用者所在的线程来执行
        executor.setRejectedExecutionHandler(new ThreadPoolExecutor.CallerRunsPolicy());
        executor.initialize();
        return executor;
    }

    /**
     *  异步任务中异常处理
     * @return
     */
    @Override
    public AsyncUncaughtExceptionHandler getAsyncUncaughtExceptionHandler() {
        return new AsyncUncaughtExceptionHandler() {

            @Override
            public void handleUncaughtException(Throwable arg0, Method arg1, Object... arg2) {
                log.error("=========================="+arg0.getMessage()+"=======================", arg0);
                log.error("exception method:"+arg1.getName());
            }
        };
    }
}
```
