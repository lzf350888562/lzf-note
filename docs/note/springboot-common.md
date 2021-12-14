

# 配置文件

默认参数

```
host: ${REDIS_HOST:127.0.0.1}
```

在Spring Boot 2.0中对配置属性加载的时候会除了像1.x版本时候那样**移除特殊字符**外，还会将配置均以**全小写**的方式进行匹配和加载。  等价例

```
spring.jpa.databaseplatform=mysql
spring.jpa.database-platform=mysql
spring.jpa.databasePlatform=mysql
spring.JPA.database_platform=mysql
```

```
spring:
  jpa:
    databaseplatform: mysql
    database-platform: mysql
    databasePlatform: mysql
    database_platform: mysql
```

**Tips：推荐使用全小写配合`-`分隔符的方式来配置，比如：`spring.jpa.database-platform=mysql`**

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

**注意：在Spring Boot 2.0中对于List类型的配置必须是连续的，不然会抛出`UnboundConfigurationPropertiesException`异常，所以如下配置是不允许的：**

```
foo[0]=a
foo[2]=b
```

**在Spring Boot 1.x中上述配置是可以的，`foo[1]`由于没有配置，它的值会是`null`**

Map**类型**

Map类型在properties和yaml中的标准配置方式如下：

- properties格式：

```
spring.my-example.foo=bar
spring.my-example.hello=world
```

- yaml格式：

```
spring:
  my-example:
    foo: bar
    hello: world
```

**注意：如果Map类型的key包含非字母数字和`-`的字符，需要用`[]`括起来，比如：**

```
spring:
  my-example:
    '[foo.baz]': bar
```

在Spring应用程序的environment中读取属性的时候，每个属性的唯一名称符合如下规则：

- 通过`.`分离各个元素
- 最后一个`.`将前缀与属性名称分开
- 必须是字母（a-z）和数字(0-9)
- 必须是小写字母
- 用连字符`-`来分隔单词
- 唯一允许的其他字符是`[`和`]`，用于List的索引
- 不能以数字开头

所以，如果我们要读取配置文件中`spring.jpa.database-platform`的配置，可以这样写：

```
context.getEnvironment().containsProperty("spring.jpa.database-platform")
```

而下面的方式是无法获取到`spring.jpa.database-platform`配置内容的：

```
context.getEnvironment().containsProperty("spring.jpa.databasePlatform")
```

**注意：使用`@Value`获取配置内容的时候也需要这样的特点**

**全新绑定api**

在Spring Boot 2.0中增加了新的绑定API来帮助我们更容易的获取配置信息

```
//配置文件
com.didispace.foo=bar
//指定前缀
@Data
@ConfigurationProperties(prefix = "com.didispace")
public class FooProperties {
    private String foo;
}
//通过Binder获取配置信息
@SpringBootApplication
public class Application {
    public static void main(String[] args) {
        ApplicationContext context = SpringApplication.run(Application.class, args);
        Binder binder = Binder.get(context.getEnvironment());
        // 绑定简单配置
        FooProperties foo = binder.bind("com.didispace", Bindable.of(FooProperties.class)).get();
        System.out.println(foo.getFoo());
    }
}
//List类
com.didispace.post[0]=Why Spring Boot
com.didispace.post[1]=Why Spring Cloud

com.didispace.posts[0].title=Why Spring Boot
com.didispace.posts[0].content=It is perfect!
com.didispace.posts[1].title=Why Spring Cloud
com.didispace.posts[1].content=It is perfect too!
//获取
ApplicationContext context = SpringApplication.run(Application.class, args);

Binder binder = Binder.get(context.getEnvironment());

// 绑定List配置
List<String> post = binder.bind("com.didispace.post",Bindable.listOf(String.class)).get();
System.out.println(post);

List<PostInfo> posts = binder.bind("com.didispace.posts",Bindable.listOf(PostInfo.class)).get();
System.out.println(posts);
```

## 自定义属性警告问题

下面内容都与该依赖有关

```
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-configuration-processor</artifactId>
</dependency>
```

除了再IDEA中手动取消警告,更合适的方式是配置元数据.

在spring boot 项目autoconfigure依赖包下 ,可以考到一个名为additional-spring-configuration-metadata.json文件.

里面指定了属性配置的元数据信息,它可以帮助IDE来完成配置联想和配置提示的展示。

**配置元数据的自动生成**

```
@Data
@Configuration
@ConfigurationProperties(prefix = "com.didispace")
public class DidiProperties {
    private String from;
}
```

mvn install

查看target.classes.META-INF下的spring-configuration-metadata.json文件,

并且再使用属性配置时,已经没有高亮警告了.

## include

一个springboot会有db、ftp、redis等的不同配置信息。我们可以使用spring.profiles.include来实现三种不同环境的一键切换。

比如有application-redis.yml

可以使用方式导入(多个以逗号分开):

```
spring.profiles.include: redis
```

## 获取pom文件属性

```
info: 
  app:  
    name: "@project.name@"
    description: "@project.description@"  
    version: "@project.version@"  
    spring-boot-version: "@project.parent.version@" 
```

## @Value

如果要初始化static成员,可加在settor方法上设置.

```
@Data
@Component
public class RsaProperties {
    public static String privateKey;

    @Value("${rsa.private_key}")
    public void setPrivateKey(String privateKey) {
        RsaProperties.privateKey = privateKey;
    }
}
```

## ConfigurationProperties

**模块装配技术**

如果一个配置类只配置@ConfigurationProperties注解，而没有使用@Component，那么在IOC容器中是获取不到properties 配置文件转化的bean。说白了 @EnableConfigurationProperties 相当于把使用  @ConfigurationProperties 的类进行了一次注入。

`@EnableConfigurationProperties` 文档中解释：
 当`@EnableConfigurationProperties`注解应用到你的`@Configuration`时， 任何被`@ConfigurationProperties`注解的beans将自动被Environment属性配置。 这种风格的配置特别适合与SpringApplication的外部YAML配置进行配合使用。

bean的属性配置加载

1.(使用(Component)).

```
@Data
@Component
@ConfigurationProperties(prefix = "yyzx.properties")
public class AnnotationDesc {
}
```

2.使用EnableConfigurationProperties

```
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

```
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

3.或者在每个字段上通过@Value获取(冗余)



## @PropertySource

springboot引入其他配置文件方式(与EnvironmentPostProcessor的区别)

```
@PropertySource("classpath:your.properties")
```

@PropertySource 中的属性解释 
1.value：指明加载配置文件的路径。 
2.ignoreResourceNotFound：指定的配置文件不存在是否报错，默认是false。当设置为 true 时，若该文件不存在，程序不会报错。实际项目开发中，最好设置 ignoreResourceNotFound 为 false。 
3.encoding：指定读取属性文件所使用的编码，我们通常使用的是UTF-8。

该注解可与配合上面的@Configuration等一起使用

# 事件模型

在Spring Boot 2.0中对事件模型做了一些增强，主要就是增加了`ApplicationStartedEvent`事件，所以在2.0版本中所有的事件按执行的先后顺序如下：

- `ApplicationStartingEvent`
- `ApplicationEnvironmentPreparedEvent`
- `ApplicationPreparedEvent`
- `ApplicationStartedEvent` <= 新增的事件
- `ApplicationReadyEvent`
- `ApplicationFailedEvent`

配置事件监听器

```
@Slf4j
public class ApplicationPreparedEventListener implements ApplicationListener<ApplicationPreparedEvent> {
    @Override
    public void onApplicationEvent(ApplicationPreparedEvent event) {
        log.info("......ApplicationPreparedEvent......");
    }

}
@Slf4j
public class ApplicationStartedEventListener implements ApplicationListener<ApplicationStartedEvent> {
    @Override
    public void onApplicationEvent(ApplicationStartedEvent event) {
        log.info("......ApplicationStartedEvent......");
    }

}
@Slf4j
public class ApplicationReadyEventListener implements ApplicationListener<ApplicationReadyEvent> {

    @Override
    public void onApplicationEvent(ApplicationReadyEvent event) {
        log.info("......ApplicationReadyEvent......");
    }
}
```

在`/src/main/resources/`目录下新建：`META-INF/spring.factories`配置文件，通过配置`org.springframework.context.ApplicationListener`来加载上面我们编写的监听器。

```
org.springframework.context.ApplicationListener=  com.didispace.ApplicationPreparedEventListener,\  com.didispace.ApplicationReadyEventListener,\  com.didispace.ApplicationStartedEventListener
```

## 发布与监听

在使用Spring构建的应用程序中，适当使用事件发布与监听的机制可以使我们的代码灵活度更高，降低耦合度。Spring提供了完整的事件发布与监听模型，在该模型中，事件发布方只需将事件发布出去，无需关心有多少个对应的事件监听器；监听器无需关心是谁发布了事件，并且可以同时监听来自多个事件发布方发布的事件，通过这种机制，事件发布与监听是解耦的。

自定义事件

```
public class MyEvent extends ApplicationEvent {
//构造器source参数表示当前事件的事件源，一般传入Spring的context上下文对象即可。
    public MyEvent(Object source) {
        super(source);
    }
}
```

事件发布器,需要通过事件发布器ApplicationEventPublisher完成,而可以通过实现ApplicationEventPublisherAware接口获取,  实现ApplicationContextAware接口获取上下文对象.

```
@Component
public class MyEventPublisher implements ApplicationEventPublisherAware, ApplicationContextAware {

    private ApplicationContext applicationContext;
    private ApplicationEventPublisher applicationEventPublisher;

    private final Logger logger = LoggerFactory.getLogger(this.getClass());

    @Override
    public void setApplicationContext(ApplicationContext applicationContext) throws BeansException {
        this.applicationContext = applicationContext;
    }

    @Override
    public void setApplicationEventPublisher(ApplicationEventPublisher applicationEventPublisher) {
        this.applicationEventPublisher = applicationEventPublisher;
    }

    public void publishEvent() {
        logger.info("开始发布自定义事件MyEvent");
        MyEvent myEvent = new MyEvent(applicationContext);
        applicationEventPublisher.publishEvent(myEvent);
        logger.info("发布自定义事件MyEvent结束");
    }
}
```

**监听**

方式1:注解监听

```
@Component
public class MyAnnotationEventListener {
    private final Logger logger = LoggerFactory.getLogger(this.getClass());
    @EventListener
    public void onMyEventPublished(MyEvent myEvent) {
        logger.info("收到自定义事件MyEvent -- MyAnnotationEventListener");
    }
}
```

被@EventListener注解标注的方法入参为MyEvent类型，所以只要MyEvent事件被发布了，该监听器就会起作用，即该方法会被回调。

方式2:编程实现监听

```
@Component
public class MyEventListener implements ApplicationListener<MyEvent> {
    private final Logger logger = LoggerFactory.getLogger(this.getClass());
    @Override
    public void onApplicationEvent(MyEvent event) {
        logger.info("收到自定义事件MyEvent");
    }
}
```

## 深入

源码解析

https://mrbird.cc/%E6%B7%B1%E5%85%A5%E7%90%86%E8%A7%A3Spring%E4%BA%8B%E4%BB%B6%E5%8F%91%E5%B8%83%E4%B8%8E%E7%9B%91%E5%90%AC.html

### 异步监听

单个异步:

首先需要在springboot入口类上通过`@EnableAsync`注解开启异步，然后在需要异步执行的监听器方法上使用`@Async`注解标注.(注解监听方式)

整体异步:

通过前面源码分析，我们知道多播器在广播事件时，会先判断是否有指定executor，有的话通过executor执行监听器逻辑。所以我们可以通过指定executor的方式来让所有的监听方法都异步执行,

```
@Configuration
public class AsyncEventConfigure {

    @Bean(name = AbstractApplicationContext.APPLICATION_EVENT_MULTICASTER_BEAN_NAME)
    public ApplicationEventMulticaster simpleApplicationEventMulticaster() {
        SimpleApplicationEventMulticaster eventMulticaster = new SimpleApplicationEventMulticaster();
        eventMulticaster.setTaskExecutor(new SimpleAsyncTaskExecutor());
        return eventMulticaster;
    }
}
```

在配置类中，我们注册了一个名称为AbstractApplicationContext.APPLICATION_EVENT_MULTICASTER_BEAN_NAME（即applicationEventMulticaster）的Bean，用于覆盖默认的事件多播器，然后指定了TaskExecutor，SimpleAsyncTaskExecutor为Spring提供的异步任务executor。

异步化测试可通过查看输出日志的线程来测试.

### **多事件监听器**

@EventListener方式单个事件监听器除了可以监听多个事件.

```
@Component
public class MyAnnotationEventListener {
    private final Logger logger = LoggerFactory.getLogger(this.getClass());
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
}
```

ApplicationListener接口方式单个类型事件也可以有多个监听器同时监听,还可以**通过实现Ordered接口实现排序（或者@Order注解标注）。**

```
@Component
public class MyEventListener implements ApplicationListener<MyEvent>, Ordered {
    private final Logger logger = LoggerFactory.getLogger(this.getClass());
    @Override
    public void onApplicationEvent(MyEvent event) {
        logger.info("收到自定义事件MyEvent，我的优先级较高");
    }
    @Override
    public int getOrder() {return 0;}
}

```

```
@Component
public class MyAnnotationEventListener implements Ordered {
    private final Logger logger = LoggerFactory.getLogger(this.getClass());
    @EventListener(classes = {MyEvent.class})
    public void onMyEventPublished(ApplicationEvent event) {
        if (event instanceof MyEvent) {
            logger.info("监听到MyEvent事件，我的优先级较低");
        }
    }
    @Override
    public int getOrder() {return 1;}
}
```

### 条件监听+SpEL

@EventListener注解还包含一个condition属性，可以配合SpEL表达式来条件化触发监听方法。修改MyEvent，添加一个boolean类型属性：

```
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

```
public void publishEvent() {
        logger.info("开始发布自定义事件MyEvent");
        MyEvent myEvent = new MyEvent(applicationContext);
        myEvent.setFlag(false); // 设置为false
        applicationEventPublisher.publishEvent(myEvent);
        logger.info("发布自定义事件MyEvent结束");
    }
```

条件监听实现:

```
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

因此MyEvent发布的时候,上面监听器没有触发

### 事务事件监听器

Spring 4.2开始提供了一个@TransactionalEventListener注解用于监听数据库事务的各个阶段：

1. AFTER_COMMIT - 事务成功提交；
2. AFTER_ROLLBACK – 事务回滚后；
3. AFTER_COMPLETION – 事务完成后（无论是提交还是回滚）；
4. BEFORE_COMMIT - 事务提交前；

例子：

```
@TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
public void onTransactionChange(ApplicationEvent event){
    logger.info("监听到事务提交事件");
}
```



# Bean生命周期

指定生命周期方法

```
@Bean(initMethod = "init", destroyMethod = "destory")
```

除了上面两种指定初始化和销毁方法的方式外，我们还可以使用`@PostConstruct`和`@PreDestroy`注解修饰方法来指定相应的初始化和销毁方法.

直接加到Bean所在类的方法上即可.

除了上面两种方式以外,Spring还为我们提供了和初始化，销毁相对应的接口：

- `InitializingBean`接口包含一个`afterPropertiesSet`方法，我们可以通过实现该接口，然后在这个方法中编写初始化逻辑。
- `DisposableBean`接口包含一个`destory`方法，我们可以通过实现该接口，然后再这个方法中编写销毁逻辑。

## BeanPostProcessor

深入前,先了解实例化和初始化:

Initialization为初始化的意思，Instantiation为实例化的意思。在Spring Bean生命周期中，实例化指的是创建Bean的过程，初始化指的是Bean创建后，对其属性进行赋值（populate bean）、后置处理等操作的过程，所以Instantiation执行时机先于Initialization。

Spring提供了一个`BeanPostProcessor`接口，俗称**Bean后置通知处理器**，它提供了两个方法`postProcessBeforeInitialization`和`postProcessAfterInitialization`。其中`postProcessBeforeInitialization`在组件的初始化方法调用之前执行，`postProcessAfterInitialization`在组件的初始化方法调用之后执行。它们都包含两个入参：

1. bean：当前组件对象；

2. beanName：当前组件在容器中的名称(按名称注入的为名称,按类注入的为全类名).

   两个方法都返回一个Object类型，我们可以直接返回当前组件对象，或者包装(**增强,代理**等)后返回。

   注意:这个接口实现类默认是针对ioc容器中所有的bean.

```
public class MyBeanPostProcessor implements BeanPostProcessor {
    @Override
    public Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {
        System.out.println(beanName + " 初始化之前调用");
        return bean;
    }
    @Override
    public Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {
        System.out.println(beanName + " 初始化之后调用");
        return bean;
    }
}
```

在配置类中注册该组件:

```
@Bean
public MyBeanPostProcessor myBeanPostProcessor () {
    return new MyBeanPostProcessor();
}
```

**另外,如果由多个需要注入ioc容器的该Processor或者类似的Processor,可以使用@Order,Ordered接口来自定义执行顺序**

**InstantiationAwareBeanPostProcessor**

InstantiationAwareBeanPostProcessor是BeanPostProcessor子接口,增加了3个方法:

- `postProcessBeforeInstantiation(Class<?> beanClass, String beanName)`：Bean实例化前执行该方法.	

  beanClass：待实例化的Bean类型；

  beanName：待实例化的Bean名称。

  方法作用为：在Bean实例化前调用该方法，返回值可以为代理后的Bean，以此代替Bean默认的实例化过程。返回值不为null时，后续只会调用BeanPostProcessor的 postProcessAfterInitialization方法，而不会调用别的后续后置处理方法（如postProcessBeforeInitialization、postProcessBeforeInstantiation等方法）；返回值也可以为null，这时候Bean将按默认方式初始化。

- `postProcessAfterInstantiation(Object bean, String beanName)`：Bean实例化之后执行该方法

  bean：实例化后的Bean，此时属性还没有被赋值；

  beanName：Bean名称。

  方法作用为：当Bean通过构造器或者工厂方法被实例化后，当属性还未被赋值前，该方法会被调用，一般用于自定义属性赋值。方法返回值为布尔类型，返回true时，表示Bean属性需要被赋值；返回false表示跳过Bean属性赋值，并且InstantiationAwareBeanPostProcessor的postProcessProperties方法不会被调用。

- `postPorcessProperties(PropertiesValues values,Object bean,String beanName)`: Bean属性赋值后调用该方法,并且postProcessorAfterInstantiation方法返回值必须为true

  values: bean属性的值;

  bean:  赋值后的bean;

  beanName: Bean名称.



综上,bean在整个生命周期中上面内容的执行顺序为:

postProcessBeforeInstantiationfan方法 --> 调用无参构造函数 -->  postProcessAfterInstantiation方法  --> Bean属性赋值  -->  postProcessProperties  -->  postProcessBeforeInitialization方法  --> init方法  --> postProcessAfterInitialization方法 -->destory方法

**源码**

postProcessAfterInitialization和InstantiationAwareBeanPostProcessor的方法都和Bean生命周期有关，要分析它们的实现原理自然要从Bean的创建过程入手。Bean创建的入口为`AbstractAutowireCapableBeanFactory`的createBean方法，查看其源码解析：https://mrbird.cc/%E6%B7%B1%E5%85%A5%E7%90%86%E8%A7%A3Spring-BeanPostProcessor-InstantiationAwareBeanPostProcessor.html

注意:

1.如果在postProcessBeforeInstantiation方法中如果返回非null(如**使用CGLIB动态代理生成了一个对象并返**回),则接下来会跳过postProcessAfterInstantiation，postProcessPropertyValues以及自定义的初始化方法(start方法), 直接执行postProcessAfterInitialization方法,并且postProcessAfterInitialization如果返回不为null,则mbd.beforeInstantiationResolved被标注为true而直接返回给ioc容器了;否则才会按照原本的流程继续执行doCreateBean方法.

2.如果postProcessAfterInstantiation方法返回值如果为false,则postProcessPropertyValues不会被执行.

3.postProcessPropertyValues的执行条件是postProcessAfterInstantiation返回true且postProcessBeforeInstantiation返回null. 在该方法中可以修改属性的赋值:

```
@Override
public PropertyValues postProcessPropertyValues(PropertyValues pvs, PropertyDescriptor[] pds, Object bean,
		String beanName) throws BeansException {
	System.out.println("<---postProcessPropertyValues--->");
	if(bean instanceof User){
		PropertyValue value = pvs.getPropertyValue("name");
		System.out.println("修改前name的值是:"+value.getValue());
		value.setConvertedValue("bobo");
	}
	return pvs;
}
```

介绍和源码解析见https://blog.csdn.net/qq_38526573/article/details/88091702

## BeanFactoryPostProcessor

该接口包含一个方法:postProcessBeanFactory.

方法注释说明了其执行时机:BeanFactory标准初始化之后，所有的Bean定义已经被加载，但Bean的实例还没被创建（不包括BeanFactoryPostProcessor类型）。**该方法通常用于修改bean的定义，Bean的属性值等，甚至可以在此快速初始化Bean。**

**BeanDefinitionRegistryPostProcessor**

BeanDefinitionRegistryPostProcessor继承自BeanFactoryPostProcessor，新增了一个postProcessBeanDefinitionRegistry方法：

postProcessBeanDefinitionRegistry方法的执行时机为：所有的Bean定义即将被加载，但Bean的实例还没被创建时。也就是说，BeanDefinitionRegistryPostProcessor的postProcessBeanDefinitionRegistry方法执行时机先于BeanFactoryPostProcessor的postProcessBeanFactory方法。这个方法通常用于给IOC容器添加额外的组件。

```
@Component
public class MyBeanFactoryPostProcessor implements BeanDefinitionRegistryPostProcessor {
    private static final Logger logger = LoggerFactory.getLogger(MyBeanFactoryPostProcessor.class);
    public MyBeanFactoryPostProcessor() {
        logger.info("实例化MyBeanFactoryPostProcessor Bean");
    }
     @Override
    public void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry registry) throws BeansException {
        int beanDefinitionCount = registry.getBeanDefinitionCount();
        logger.info("Bean定义个数: " + beanDefinitionCount);
        // 添加一个新的Bean定义
        RootBeanDefinition definition = new RootBeanDefinition(Object.class);
        registry.registerBeanDefinition("hello", definition);
    }
    @Override
    public void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) throws BeansException {
        int beanDefinitionCount = beanFactory.getBeanDefinitionCount();
        logger.info("Bean定义个数: " + beanDefinitionCount);
    }
    @Component
    static class TestBean {
        public TestBean() {
            logger.info("实例化TestBean");
        }
    }
}
```

执行顺序为:

postProcessBeanDefinitionRegistry方法 -->

实现了BeanFactoryPostProcessor的类的构造函数  -->

postProcessBeanFactory 方法 -->

其他的bean的构造函数

源码解析https://mrbird.cc/%E6%B7%B1%E5%85%A5%E7%90%86%E8%A7%A3BeanFactoryPostProcessor-BeanDefinitionRegistryPostProcessor.html

# 启动后执行

实现ApplicationRunner或CommandLineRunner接口.

源码在`SpringApplication`的callRunners方法中执行;

# Spring组件注册

在传统spring的xml配置中,指定组件扫描的方式为

```
<context:component-scan base-package=""></context:component-scan>
```

路径下所有被`@Controller`、`@Service`、`@Repository`和`@Component`注解标注的类都会被纳入IOC容器中。



## @ComponentScan

在springboot中 ,该注解默认扫描启动类所在的包下的类和子包下的类.

允许我们指定扫描策略,即指定哪些被扫描，哪些不被扫描，查看其源码可发现这两个属性：

```
Filter[] includeFilters() default {};
Filter[] excludeFilters() default {};
```

其中Filter也是注解

```
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

例

```
@Configuration
@ComponentScan(value = "cc.mrbird.demo",
        excludeFilters = {
                @Filter(type = FilterType.ANNOTATION,
                        classes = {Controller.class, Repository.class}),
                @Filter(type = FilterType.ASSIGNABLE_TYPE, classes = User.class)
        })
public class WebConfig {

}
```

上面我们指定了两种排除扫描的规则：

1. 根据注解来排除（`type = FilterType.ANNOTATION`）,这些注解的类型为`classes = {Controller.class, Repository.class}`。即`Controller`和`Repository`注解标注的类不再被纳入到IOC容器中。
2. 根据指定类型类排除（`type = FilterType.ASSIGNABLE_TYPE`），排除类型为`User.class`，其子类，实现类都会被排除。

可通过ApplicationContext实例的getBeanDefinitionNames方法获取所有bean来验证.

### @FilterType

上面两种为常用的规则,还有的规则如下

```
public enum FilterType {
    /**
     * Filter candidates marked with a given annotation.
     * @see org.springframework.core.type.filter.AnnotationTypeFilter
     */
    ANNOTATION,
    /**
     * Filter candidates assignable to a given type.
     *
     * @see org.springframework.core.type.filter.AssignableTypeFilter
     */
    ASSIGNABLE_TYPE,
    /**
     * Filter candidates matching a given AspectJ type pattern expression.
     * @see org.springframework.core.type.filter.AspectJTypeFilter
     */
    ASPECTJ,
    /**
     * Filter candidates matching a given regex pattern.
     * @see org.springframework.core.type.filter.RegexPatternTypeFilter
     */
    REGEX,
    /**
     * Filter candidates using a given custom
     * {@link org.springframework.core.type.filter.TypeFilter} implementation.
     */
    CUSTOM
}
```

我们还可以通过`ASPECTJ`表达式，`REGEX`正则表达式和`CUSTOM`自定义规则（下面详细介绍）来指定扫描策略。

```
@Configuration
@ComponentScan(value = "cc.mrbird.demo",
        includeFilters = {
                @Filter(type = FilterType.ANNOTATION, classes = Service.class)
        }, useDefaultFilters = false)
public class WebConfig {

}
```

上面配置了只将`Service`纳入IOC容器，并且需要用`useDefaultFilters = false`来关闭Spring默认的扫描策略才能让我们的配置生效（,扫描带有@Component  @Repository  @Service  @Controller 的组件）。

注意:有一些Processor和一个Factory是ioc容器启动必须需要的监听和处理类.

## 多扫描策略

在Java 8之前，我们可以使用`@ComponentScans`来配置多个`@ComponentScan`以实现多扫描规则配置：

```
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
@Documented
public @interface ComponentScans {    
	ComponentScan[] values();
}
```

而在Java 8中，新增了`@Repeatable`注解，使用该注解修饰的注解可以重复使用，查看`@ComponentScan`源码会发现其已经被该注解标注：

```
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
@Documented
@Repeatable(ComponentScans.class)
public @interface ComponentScan {    
	//...
}
```

所以除了使用`@ComponentScans`来配置多扫描规则外，我们还可以通过多次使用`@ComponentScan`来指定多个不同的扫描规则。

## 自定义扫描策略

自定义扫描策略需要我们实现`org.springframework.core.type.filter.TypeFilter`接口

```
public class MyTypeFilter implements TypeFilter {
    @Override
    public boolean match(MetadataReader metadataReader, MetadataReaderFactory metadataReaderFactory) throws IOException {
        return false;
    }
}
```

1. `MetadataReader`：当前正在扫描的类的信息；
2. `MetadataReaderFactory`：可以通过它来获取其他类的信息。

```
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

```
@Configuration
@ComponentScan(value = "cc.mrbird.demo",
        excludeFilters = {
            @Filter(type = FilterType.CUSTOM, classes = MyTypeFilter.class)
        })
public class WebConfig {
}
```

配合`excludeFilters`使用意指当被扫描的类名包含`er`时，该类不被纳入IOC容器中。

因为`User`，`UserMapper`，`UserService`和`UserController`等类的类名都包含`er`，所以它们都没有被纳入到IOC容器中。

## 组件作用域

默认情况下，在Spring的IOC容器中每个组件都是单例的，即无论在任何地方注入多少次，这些对象都是同一个.

通过@Scope的value或者scopeName指定

1. `singleton`：单实例（默认）,在Spring IOC容器启动的时候会调用方法创建对象然后纳入到IOC容器中，以后每次获取都是直接从IOC容器中获取（`map.get()`）；
2. `prototype`：多实例，IOC容器启动的时候并不会去创建对象，而是在每次获取的时候才会去调用方法创建对象；
3. `request`：一个请求对应一个实例；
4. `session`：同一个session对应一个实例。

**懒加载**

懒加载@Lazy是针对单例模式而言的，正如前面所说，IOC容器中的组件默认是单例的，容器启动的时候会调用方法创建对象然后纳入到IOC容器中。

要测试可以在 Bean注册的地方加入一句话以观察：

```
@Configuration
public class WebConfig {
    @Bean
    @Lazy
    public User user() {
        System.out.println("往IOC容器中注册user bean");
        return new User("mrbird", 18);
    }
}
```

## 条件注册组件

@**Conditional**

使用`@Conditional`注解我们可以指定组件注册的条件，即满足特定条件才将组件纳入到IOC容器中。

在使用该注解之前，我们需要创建一个类，实现`Condition`接口.

该接口包含一个`matches`方法，包含两个入参:

1. `ConditionContext`：上下文信息.如conditionContext.getBeanFactory(), conditionContext.getClassLoader(), conditionContext.getEnvironment() conditionContext.getRegistry()；

1. `AnnotatedTypeMetadata`：注解信息.如((StandardMethodMetadata)metadata).getMethodName()可获取方法名,下面为user方法。

```
public class MyCondition implements Condition {
    @Override
    public boolean matches(ConditionContext context, AnnotatedTypeMetadata metadata) {
        String osName = context.getEnvironment().getProperty("os.name");
        return osName != null && osName.contains("Windows");
    }
}
```

```
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

context.getRegistry()获取的是BeanDefinitionRegistry,包含一些和Bean有关的方法.在导入组件模块有更详细介绍

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



@**Profile**

可以根据不同的环境变量来注册不同的组件

```
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
```

```
ConfigurableApplicationContext context1 = new SpringApplicationBuilder(DemoApplication.class)
                .web(WebApplicationType.NONE)
                .profiles("java8")
                //.profiles("java7")
                .run(args);

CalculateService service = context1.getBean(CalculateService.class);
System.out.println("求合结果： " + service.sum(1, 2, 3, 4, 5, 6, 7, 8, 9, 10));
```

### 配置文件切换缓存

用的多的有@ConditionalOnProperty, 比如通过配置文件实现缓存切换.

其中CacheService为统一接口

```
@ConditionalOnProperty(prefix = "cache", name = "type", havingValue = "redis")
@RequiredArgsConstructor
@Service
public class RedisServiceImpl implements CacheService {
	private final StringRedisTemplate stringRedisTemplate;
	private final RedisTemplate<Object,  Object>  redisTemplate;
	@Override
	public String get(String key) {
		return stringRedisTemplate.opsForValue().get(key);
	}
	@Override
	public void set(String key, String value) {
		stringRedisTemplate.opsForValue().set(key,value);
	}
	@Override
	public void set(String key, String value, long timeout) {		stringRedisTemplate.opsForValue().set(key,value,timeout, TimeUnit.SECONDS);

	}
	@Override
	public Object getObject(String key) {
		return redisTemplate.opsForValue().get(key);
	}
	@Override
	public void setObject(String key, Object value) {
		redisTemplate.opsForValue().set(key,value);
	}
	@Override
	public void setObject(String key, Object value, long timeout) {		redisTemplate.opsForValue().set(key,value,timeout, TimeUnit.SECONDS);
	}
	@Override
	public void del(String key) {
		redisTemplate.delete(key);
		stringRedisTemplate.delete(key);
	}
	@Override
	public boolean contains(String key) {
		return redisTemplate.hasKey(key) || stringRedisTemplate.hasKey(key);
	}
	@Override
	public void expire(String key, long timeout) {
		redisTemplate.expire(key,timeout, TimeUnit.SECONDS);
		stringRedisTemplate.expire(key,timeout, TimeUnit.SECONDS);
	}
}
```

```
@ConditionalOnProperty(prefix = "cache", name = "type", havingValue = "ehcache")
@RequiredArgsConstructor
@Service
public class EhCacheServiceImpl implements CacheService {
    private final CacheManager cacheManager;
    /**
     * 获得一个Cache，没有则创建一个。
     *
     * @return
     */
    private Cache getCache() {

        Cache cache = cacheManager.getCache("util_cache");
        return cache;
    }
    @Override
    public String get(String key) {
        Element element = getCache().get(key);
        return element == null ? null : (String) element.getObjectValue();
    }
    @Override
    public void set(String key, String value) {
        Element element = new Element(key, value);
        Cache cache = getCache();
        //不过期
        cache.getCacheConfiguration().setEternal(true);
        cache.put(element);

    }
    @Override
    public void set(String key, String value, long timeout) {
        Element element = new Element(key, value);
        element.setTimeToLive((int) timeout);
        Cache cache = getCache();
        cache.put(element);

    }
    @Override
    public void del(String key) {
        getCache().remove(key);


    }
    @Override
    public boolean contains(String key) {
        return getCache().isKeyInCache(key);
    }
    @Override
    public void expire(String key, long timeout) {
        Element element = getCache().get(key);
        if (element != null) {
            Object value = element.getValue();
            element = new Element(key, value);
            element.setTimeToLive((int) timeout);
            Cache cache = getCache();
            cache.put(element);
        }
    }
    /**
     * 根据key获取缓存的Object类型数据
     */
    @Override
    public Object getObject(String key) {
        Element element = getCache().get(key);
        return element == null ? null : element.getObjectValue();
    }
    /**
     * 设置Object类型的缓存
     */
    @Override
    public void setObject(String key, Object value) {
        Element element = new Element(key, value);
        Cache cache = getCache();
        //不过期
        cache.getCacheConfiguration().setEternal(true);
        cache.put(element);

    }
    /**
     * 设置一个有过期时间的Object类型的缓存,单位秒
     */
    @Override
    public void setObject(String key, Object value, long timeout) {
        Element element = new Element(key, value);
        element.setTimeToLive((int) timeout);
        Cache cache = getCache();
        cache.put(element);

    }
}
```



## 导入组件

@import

示例:在配置类上添加:

```
@Import({Hello.class})
```

该导入方式默认以全类名方式导入

如果需要一次性导入较多组件，我们可以使用`ImportSelector`来实现。

### **ImportSelector**

`ImportSelector`是一个接口，包含一个`selectImports`方法，方法返回类的全类名数组（即需要导入到IOC容器中组件的全类名数组），包含一个`AnnotationMetadata`类型入参，通过这个参数我们可以获取到使用`ImportSelector`的类的全部注解信息。

我们新建一个`ImportSelector`实现类`MyImportSelector`：

```
public class MyImportSelector implements ImportSelector {
    @Override
    public String[] selectImports(AnnotationMetadata importingClassMetadata) {
        return new String[]{
                "cc.mrbird.demo.domain.Apple",
                "cc.mrbird.demo.domain.Banana",
                "cc.mrbird.demo.domain.Watermelon"
        };
    }
}
```

```
//在配置类的@Import注解上使用MyImportSelector来把这三个组件快速地导入到IOC容器中：
@Import({MyImportSelector.class})
```

### ImportBeanDefinitionRegistrar

手动往IOC容器导入组件.

```
public interface ImportBeanDefinitionRegistrar {
    public void registerBeanDefinitions(AnnotationMetadata importingClassMetadata, BeanDefinitionRegistry registry);
}
```

1. `AnnotationMetadata`：可以通过它获取到类的注解信息；
2. `BeanDefinitionRegistry`：Bean定义注册器，包含了一些和Bean有关的方法：

```
 public interface BeanDefinitionRegistry extends AliasRegistry {
    void registerBeanDefinition(String var1, BeanDefinition var2) throws BeanDefinitionStoreException;
    void removeBeanDefinition(String var1) throws NoSuchBeanDefinitionException;
    BeanDefinition getBeanDefinition(String var1) throws NoSuchBeanDefinitionException;
    boolean containsBeanDefinition(String var1);
    String[] getBeanDefinitionNames();
    int getBeanDefinitionCount();
    boolean isBeanNameInUse(String var1);
}
```

**借助`BeanDefinitionRegistry`的`registerBeanDefinition`方法来往IOC容器中注册Bean。**

该方法包含两个入参，第一个为需要注册的Bean名称（Id）,第二个参数为Bean的定义信息，它是一个接口，我们可以使用其实现类`RootBeanDefinition`来完成.

为了演示`ImportBeanDefinitionRegistrar`的使用，我们先新增一个类，名称为`Strawberry`，代码略。

然后新增一个`ImportBeanDefinitionRegistrar`实现类`MyImportBeanDefinitionRegistrar`：

```
public class MyImportBeanDefinitionRegistrar implements ImportBeanDefinitionRegistrar {
    @Override
    public void registerBeanDefinitions(AnnotationMetadata importingClassMetadata, BeanDefinitionRegistry registry) {
        final String beanName = "strawberry";
        boolean contain = registry.containsBeanDefinition(beanName);
        if (!contain) {
            RootBeanDefinition rootBeanDefinition = new RootBeanDefinition(Strawberry.class);
            registry.registerBeanDefinition(beanName, rootBeanDefinition);
        }
    }
}
```

在上面的实现类中，我们先通过`BeanDefinitionRegistry`的`containsBeanDefinition`方法判断IOC容器中是否包含了名称为`strawberry`的组件，如果没有，则手动通过`BeanDefinitionRegistry`的`registerBeanDefinition`方法注册一个。

定义好`MyImportBeanDefinitionRegistrar`后，我们同样地在配置类的`@Import`中使用它：

```
@Import({MyImportBeanDefinitionRegistrar.class})
```

其他BeanDefinition的实现类需要自查资料.

## FactoryBean注册组件

Spring还提供了一个`FactoryBean`接口，我们可以通过实现该接口来注册组件，该接口包含了两个抽象方法和一个默认方法,创建一个类Cherry(随意)和FactoryBean实现类:

```
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

定义好`CherryFactoryBean`后，我们在配置类中注册这个类：

```
@Bean
public CherryFactoryBean cherryFactoryBean() {
    return new CherryFactoryBean();
}
```

测试从容器中获取：

```
ApplicationContext context = new AnnotationConfigApplicationContext(WebConfig.class);
Object cherry = context.getBean("cherryFactoryBean");
System.out.println(cherry.getClass());
```

输出内容为Cherry对象的全类名.

可看到，虽然我们获取的是Id为`cherryFactoryBean`的组件，但其获取到的实际是`getObject`方法里返回的对象。

如果我们要获取`cherryFactoryBean`本身，则可以这样做：

```
Object cherryFactoryBean = context.getBean("&cherryFactoryBean");
System.out.println(cherryFactoryBean.getClass());
```

为什么加上`&`前缀就可以获取到相应的工厂类了呢，查看`BeanFactory`的源码会发现原因:

里面定义了字符串 FACTORY_BEAN_PREFIX = "&"

## BeanDefinitionRegistry

手动注册

```
@Data
public class Person {
    private String name;
    private String age;
}
```

方式一:

实现`BeanFactoryAware`接口, 可获取到Spring工厂类，因`BeanFactory`是一个接口，通过调试可知，Spring窗口注入的工厂实现类是`DefaultListableBeanFactory`，通过其源码可以看到，这个Bean工厂类实现了`BeanDefinitionRegistry`接口，通过这个接口的`registerBeanDefinition`方法，就可以将bean注册到Spring容器了

```
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
        //重点
        BeanDefinitionRegistry registry = (BeanDefinitionRegistry) this.beanFactory;
        registry.registerBeanDefinition("person", builder.getBeanDefinition());
    }   
}
```

方式二:

```
@Component
public class PersonBeanRegiser implements ImportBeanDefinitionRegistrar {
    @Override
    public void registerBeanDefinitions(AnnotationMetadata importingClassMetadata, BeanDefinitionRegistry ){
        BeanDefinitionBuilder builder = BeanDefinitionBuilder.genericBeanDefinition(Person.class);
        builder.addPropertyValue("name", "张三");
        builder.addPropertyValue("age", 20);
        //重点
        BeanDefinitionRegistry registry = (BeanDefinitionRegistry) this.beanFactory;
        registry.registerBeanDefinition("person", builder.getBeanDefinition());
    }   
}
```



# 自动装配

## 模式注解

Stereotype Annotation俗称为模式注解，Spring中常见的模式注解有`@Service`，`@Repository`，`@Controller`等，它们都“派生”自`@Component`注解。

我们都知道，凡是被`@Component`标注的类都会被Spring扫描并纳入到IOC容器中，那么由`@Component`派生的注解所标注的类也会被扫描到IOC容器中.

@component具有派生性和层次性:即如果自定义注解上标注了@Component或标注了Spring自带的模式注解,则标注了该自定义注解的类也会被注入到IOC容器中/

## @Enable模块驱动

`@Enable`模块驱动在Spring Framework 3.1后开始支持。这里的模块通俗的来说就是一些为了实现某个功能的组件的集合。通过`@Enable`模块驱动，我们可以开启相应的模块功能。

`@Enable`模块驱动可以分为“注解驱动”和“接口编程”两种实现方式.

### 注解驱动

基于注解驱动的示例可以查看`@EnableWebMvc`源码

```
@Retention(RetentionPolicy.RUNTIME)
@Target({ElementType.TYPE})
@Documented
@Import({DelegatingWebMvcConfiguration.class})
public @interface EnableWebMvc {
}
```

该注解通过`@Import`导入一个配置类`DelegatingWebMvcConfiguration`,该配置类又继承自`WebMvcConfigurationSupport`，里面定义了一些Bean的声明(包括requestMappingHandlerMapping、mvcPathMatcher、mvcUrlPathHelper、mvcResourceUrlProvider等等)。

所以，基于注解驱动的`@Enable`模块驱动其实就是通过`@Import`来导入一个配置类，以此实现相应模块的组件注册，当这些组件注册到IOC容器中，这个模块对应的功能也就可以使用了。

**自定义基于注解驱动的`@Enable`模块驱动**

配置类

```
@Configuration
public class HelloWorldConfiguration {

    @Bean
    public String hello() {
        return "hello world";
    }
}
```

@Enable注解,在该注解类上通过`@Import`导入了刚刚创建的配置类。

```
@Target({ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Import(HelloWorldConfiguration.class)
public @interface EnableHelloWorld {
}
```

测试

```
@EnableHelloWorld
public class TestEnableBootstap {
    public static void main(String[] args) {
        ConfigurableApplicationContext context = new SpringApplicationBuilder(TestEnableBootstap.class)
                .web(WebApplicationType.NONE)
                .run(args);
        String hello = context.getBean("hello", String.class);
        System.out.println("hello Bean: " + hello);
        context.close();
    }
}
```

### 接口编程

通过接口编程的方式来实现`@Enable`模块驱动。Spring中，基于接口编程方式的有`@EnableCaching`注解，查看其源码：

```
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

`EnableCaching`注解通过`@Import`导入了`CachingConfigurationSelector`类，该类间接实现了`ImportSelector`接口，在Spring组件注册中，介绍了可以通过`ImportSelector`来实现组件注册。

**所以通过接口编程实现`@Enable`模块驱动的本质是：通过`@Import`来导入接口`ImportSelector`实现类，该实现类里可以定义需要注册到IOC容器中的组件，以此实现相应模块对应组件的注册。**

**自定义基于接口编程实现@Enable模块驱动**

```
public class HelloWorldImportSelector implements ImportSelector {
    @Override
    public String[] selectImports(AnnotationMetadata importingClassMetadata) {
    	//返回所有要导入的配置类的全类名
        return new String[]{HelloWorldConfiguration.class.getName()};
    }
}
```

```
@Target({ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Import(HelloWorldImportSelector.class)
public @interface EnableHelloWorld {
}
```

测试结果与注解驱动一致.

## 工厂加载

Spring 工厂加载机制的实现类为`AutoConfigurationImportSelector`+`SpringFactoriesLoader`,该类指定了

**FACTORIES_RESOURCE_LOCATION = "META-INF/spring.factories"** 

该类的方法会读取META-INFO目录下的spring.factories配置文件.

查看sprinb-boot-autoconfigure-x.x.x.RELEASE.jar下的该文件可看到内容为:

```
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
...很多配置类...
```

当启动类被`@EnableAutoConfiguration`标注后(springboot 主启动类注解中有),看是否可以纳入到IOC容器中进行管理。

可随便点击几个熟悉的配置类查看, 通常上面的配置类里面都包含了条件注册(ConditionOnXXX), 比如只有在包含某个类的时候才注册,这也是为什么ioc实际注册的配置类并不是全部spring.factories下的内容.

具体可以debug查看`AutoConfigurationImportSelector&getAutoConfigurationEntry`方法

**仿照上面方式自定义实现工厂加载模式**

```
@Configuration
@EnableHelloWorld
@ConditionalOnProperty(name = "helloworld", havingValue = "true")
public class HelloWorldAutoConfiguration {
}
```

在resources目录下新建META-INF目录，并创建spring.factories文件：

```
# Auto Configure
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
com.example.demo.configuration.HelloWorldAutoConfiguration
```

在配置文件application.properties中添加`helloworld=true`配置

```
helloworld=true
```

测试

```
@EnableAutoConfiguration
public class EnableAutoConfigurationBootstrap {
    public static void main(String[] args) {
        ConfigurableApplicationContext context = new SpringApplicationBuilder(EnableAutoConfigurationBootstrap.class)
                .web(WebApplicationType.NONE)
                .run(args);
        String hello = context.getBean("hello", String.class);
        System.out.println("hello Bean: " + hello);
        context.close();
    }
}
```

# 依赖注入

多个同接口组件，除了使用ApplicationContext注入，还可以使用更便捷的Map方式，其中string代表bean的名称，如果没有指定名称，则为类名的驼峰命名

```
Map<String, [接口]>
```

如

```
@Service(value = "txt")
public class FileBookContentServiceImpl implements BookContentService{
//...
}

@Service(value = "db")
public class DbBookContentServiceImpl implements BookContentService {
//...
}

//注入
@Autowire
private final Map<String, BookContentService> bookContentServiceMap;

//使用
bookContentServiceMap.get("txt")
```

在实际中，map的String可以从配置文件获取来达到修改使用的service

# aware回调接口

**ApplicationContextAware**

当一个`ApplicationContext`创建实现该`org.springframework.context.ApplicationContextAware`接口的对象实例时 ，会为该实例提供对该 的引用`ApplicationContext`.

**BeanNameAware**

当 `ApplicationContext` 创建一个实现`BeanNameAware` 接口的类时，将为该类提供对其关联对象定义中定义的名称的引用

**其他aware接口**

https://docs.spring.io/spring-framework/docs/current/reference/html/core.html#aware-list

## SpringUtils

```
@Component
public final class SpringUtils implements BeanFactoryPostProcessor, ApplicationContextAware 
{
    /** Spring应用上下文环境 */
    private static ConfigurableListableBeanFactory beanFactory;

    private static ApplicationContext applicationContext;

    @Override
    public void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) throws BeansException 
    {
        SpringUtils.beanFactory = beanFactory;
    }

    @Override
    public void setApplicationContext(ApplicationContext applicationContext) throws BeansException 
    {
        SpringUtils.applicationContext = applicationContext;
    }

    /**
     * 获取对象
     *
     * @param name
     * @return Object 一个以所给名字注册的bean的实例
     * @throws org.springframework.beans.BeansException
     *
     */
    @SuppressWarnings("unchecked")
    public static <T> T getBean(String name) throws BeansException
    {
        return (T) beanFactory.getBean(name);
    }

    /**
     * 获取类型为requiredType的对象
     *
     * @param clz
     * @return
     * @throws org.springframework.beans.BeansException
     *
     */
    public static <T> T getBean(Class<T> clz) throws BeansException
    {
        T result = (T) beanFactory.getBean(clz);
        return result;
    }

    /**
     * 如果BeanFactory包含一个与所给名称匹配的bean定义，则返回true
     *
     * @param name
     * @return boolean
     */
    public static boolean containsBean(String name)
    {
        return beanFactory.containsBean(name);
    }

    /**
     * 判断以给定名字注册的bean定义是一个singleton还是一个prototype。 如果与给定名字相应的bean定义没有被找到，将会抛出一个异常（NoSuchBeanDefinitionException）
     *
     * @param name
     * @return boolean
     * @throws org.springframework.beans.factory.NoSuchBeanDefinitionException
     *
     */
    public static boolean isSingleton(String name) throws NoSuchBeanDefinitionException
    {
        return beanFactory.isSingleton(name);
    }

    /**
     * @param name
     * @return Class 注册对象的类型
     * @throws org.springframework.beans.factory.NoSuchBeanDefinitionException
     *
     */
    public static Class<?> getType(String name) throws NoSuchBeanDefinitionException
    {
        return beanFactory.getType(name);
    }

    /**
     * 如果给定的bean名字在bean定义中有别名，则返回这些别名
     *
     * @param name
     * @return
     * @throws org.springframework.beans.factory.NoSuchBeanDefinitionException
     *
     */
    public static String[] getAliases(String name) throws NoSuchBeanDefinitionException
    {
        return beanFactory.getAliases(name);
    }

    /**
     * 获取aop代理对象
     * 
     * @param invoker
     * @return
     */
    @SuppressWarnings("unchecked")
    public static <T> T getAopProxy(T invoker)
    {
        return (T) AopContext.currentProxy();
    }

    /**
     * 获取当前的环境配置，无配置返回null
     *
     * @return 当前的环境配置
     */
    public static String[] getActiveProfiles()
    {
        return applicationContext.getEnvironment().getActiveProfiles();
    }

    /**
     * 获取当前的环境配置，当有多个环境配置时，只获取第一个
     *
     * @return 当前的环境配置
     */
    public static String getActiveProfile()
    {
        final String[] activeProfiles = getActiveProfiles();
        return StringUtils.isNotEmpty(activeProfiles) ? activeProfiles[0] : null;
    }
}
```









# AOP

Spring 中的 AOP 模块中：如果目标对象实现了接口，则默认采用 JDK 动态代理，否则采用 CGLIB 动态代理。

- Pointcut（切点），指定在什么情况下才执行 AOP，例如方法被打上某个注解的时候
- JoinPoint（连接点），程序运行中的执行点，例如一个方法的执行或是一个异常的处理；并且在 Spring AOP 中，只有方法连接点
- Advice（增强），对连接点进行增强（代理）：在方法调用前、调用后 或者 抛出异常时，进行额外的处理
- Aspect（切面），由 Pointcut 和 Advice 组成，可理解为：要在什么情况下（Pointcut）对哪个目标（JoinPoint）做什么样的增强（Advice）

回顾代理模式:

1.java proxy

```
Proxy.newProxyInstance(ClassLoader loader, Class<?>[] interfaces, InvocationHandler h)

loader :类加载器，用于加载代理对象。
interfaces : 被代理类实现的一些接口；
h : 实现了 InvocationHandler 接口的对象；
```

2.cglib(多种拦截器接口自查资料)

```
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

   spring aop深入理解 https://mrbird.cc/%E6%B7%B1%E5%85%A5%E7%90%86%E8%A7%A3Spring-AOP%E5%8E%9F%E7%90%86.html

   ```
   // exposeProxy = true表示通过aop框架暴露该代理对象到AOP上下文(Thr中,AopContext能够访问
   //(通过AopContext的ThreadLocal实现)
   @EnableAspectJAutoProxy(exposeProxy = true)
   //可以调用代理对象的方法
   //应用场景,当想在serivce没有@Transactional的方法中调用带有@Transactional注解的方法时,要使@Transactional生效时,使用this.xxx调用的不是代理之后的方法,而下面的方式可以调用代理方法
   ((T) AopContext.currentProxy()).xxx;
   ```
   
   因为spring采用动态代理机制来实现事务控制，而动态代理(jdk代理,因为service实现了接口)最终都是要调用原始对象的，而原始对象在去调用方法时，是不会再触发代理了.
   
   **问题:** 为什么自测结果不符?
   
   ```
   public class Parent {
   	public void f1(){
   		this.f2();
   	}
   	public void f2(){
   		System.out.println("f2...");
   	}
   
   	public static void main(String[] args) {
   		Enhancer enhancer = new Enhancer();
   		enhancer.setSuperclass(Parent.class);
   		enhancer.setCallback(new MethodInterceptor() {
   			@Override
   			public Object intercept(Object o, Method method, Object[] args, MethodProxy methodProxy) throws Throwable {
   				if(method.getName().equals("f2")){
   					System.out.println("------>");
   					Object o1 = methodProxy.invokeSuper(o, args);
   					System.out.println("<------");
   					return o1;
   				}
   				return methodProxy.invokeSuper(o,args);
   			}
   		});
   		Parent o = (Parent)enhancer.create();
   		o.f1();
   	}
   }  f
   ------>
   f2...
   <------
   ```
   
   答案:因为cglib生成的代理对象实际上为被代理对象的子类对象(这点可以从使用jdk代理增强时需要new一个被代理对象可以看出来), 所以cglib代理对象内部使用this调用的都是增强后的方法. 而jdk生成的代理对象与被代理对象是兄弟关系,代理对象中需要传入被代理对象来调用.
   
   使用jdk代理结果截然不同
   
   ```
   public class ProxyTest {
       public static void main(String[] args) {
           Parent2 parent2 = new Parent2();
           P proxyObj = (P)Proxy.newProxyInstance(Parent2.class.getClassLoader(), Parent2.class.getInterfaces(), new InvocationHandler() {
               @Override
               public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
                   Object res = null;
                   if(method.getName().equals("f2")){
                       System.out.println("---");
                       res = method.invoke(parent2, args);
                       System.out.println("---");
                   }else{
                       res = method.invoke(parent2, args);
                   }
                   return res;
               }
           });
           proxyObj.f1();
   
       }
   
   }
   class Parent2 implements P{
       public void f1(){
           //this 为被代理对象
           this.f2();
       }
       public void f2(){
           System.out.println("f2...");
       }
   }
   interface P {
       void f1();
       void f2();
   }
   只输出 f2...
   ```
   
   
   
   **切入点表达式**
   
   切入点指示符用来指示切入点表达式目的，，在Spring AOP中目前只有执行方法这一个连接点，Spring AOP支持的AspectJ切入点指示符如下：
   
            execution：用于匹配方法执行的连接点；
       
            within：用于匹配指定类型内的方法执行；
       
            this：用于匹配当前AOP代理对象类型的执行方法；注意是AOP代理对象的类型匹配，这样就可能包括引入接口也类型匹配；
       
            target：用于匹配当前目标对象类型的执行方法；注意是目标对象的类型匹配，这样就不包括引入接口也类型匹配；
       
            args：用于匹配当前执行的方法传入的参数为指定类型的执行方法；
       
            @within：用于匹配所以持有指定注解类型内的方法；
       
            @target：用于匹配当前目标对象类型的执行方法，其中目标对象持有指定的注解；
       
            @args：用于匹配当前执行的方法传入的参数持有指定注解的执行；
       
            @annotation：用于匹配当前执行方法持有指定注解的方法；
   
   1.execution：使用“execution(方法表达式)”匹配方法执行；
   
   | 模式                                                         | 描述                                                         |
   | ------------------------------------------------------------ | ------------------------------------------------------------ |
   | public * *(..)                                               | 任何公共方法的执行                                           |
   | \* cn.javass..IPointcutService.*()                           | cn.javass包及所有子包下IPointcutService接口中的任何无参方法  |
   | \* cn.javass..*.*(..)                                        | cn.javass包及所有子包下任何类的任何方法                      |
   | \* cn.javass..IPointcutService.*( *)                         | cn.javass包及所有子包下IPointcutService接口的任何只有一个参数方法 |
   | \* (!cn.javass..IPointcutService+).*(..)                     | 非“cn.javass包及所有子包下IPointcutService接口及子类型”的任何方法 |
   | \* cn.javass..IPointcutService+.*()                          | cn.javass包及所有子包下IPointcutService接口及子类型的的任何无参方法 |
   | \* cn.javass..IPointcut*.test*(java.util.Date)               | cn.javass包及所有子包下IPointcut前缀类型的的以test开头的只有一个参数类型为java.util.Date的方法，注意该匹配是根据方法签名的参数类型进行匹配的，而不是根据执行时传入的参数类型决定的如定义方法：public void test(Object obj);即使执行时传入java.util.Date，也不会匹配的； |
   | \* cn.javass..IPointcut*.test*(..) throwsIllegalArgumentException, ArrayIndexOutOfBoundsException | cn.javass包及所有子包下IPointcut前缀类型的的任何方法，且抛出IllegalArgumentException和ArrayIndexOutOfBoundsException异常 |
   | \* (cn.javass..IPointcutService+&& java.io.Serializable+).*(..) | 任何实现了cn.javass包及所有子包下IPointcutService接口和java.io.Serializable接口的类型的任何方法 |
   | @java.lang.Deprecated * *(..)                                | 任何持有@java.lang.Deprecated注解的方法                      |
   | @java.lang.Deprecated @cn.javass..Secure  * *(..)            | 任何持有@java.lang.Deprecated和@cn.javass..Secure注解的方法  |
   | @(java.lang.Deprecated \|\|cn.javass..Secure) * *(..)        | 任何持有@java.lang.Deprecated或@ cn.javass..Secure注解的方法 |
   | (@cn.javass..Secure  *) *(..)                                | 任何返回值类型持有@cn.javass..Secure的方法                   |
   | \*  (@cn.javass..Secure *).*(..)                             | 任何定义方法的类型持有@cn.javass..Secure的方法               |
   | \* *(@cn.javass..Secure (*) , @cn.javass..Secure (*))        | 任何签名带有两个参数的方法，且这个两个参数都被@ Secure标记了，如public void test(@Secure String str1, @Secure String str1); |
   | \* *((@ cn.javass..Secure *))或\* *(@ cn.javass..Secure *)   | 任何带有一个参数的方法，且该参数类型持有@ cn.javass..Secure；如public void test(Model model);且Model类上持有@Secure注解 |
   | \* *(@cn.javass..Secure (@cn.javass..Secure *) ,@ cn.javass..Secure (@cn.javass..Secure *)) | 任何带有两个参数的方法，且这两个参数都被@ cn.javass..Secure标记了；且这两个参数的类型上都持有@ cn.javass..Secure； |
   | \* *(java.util.Map<cn.javass..Model, cn.javass..Model>,..)   | 任何带有一个java.util.Map参数的方法，且该参数类型是以< cn.javass..Model, cn.javass..Model >为泛型参数；注意只匹配第一个参数为java.util.Map,不包括子类型；如public void test(HashMap<Model, Model> map, String str);将不匹配，必须使用“* *(java.util.HashMap<cn.javass..Model,cn.javass..Model>, ..)”进行匹配;而public void test(Map map, int i);也将不匹配，因为泛型参数不匹配 |
   | \* *(java.util.Collection<@cn.javass..Secure *>)             | 任何带有一个参数（类型为java.util.Collection）的方法，且该参数类型是有一个泛型参数，该泛型参数类型上持有@cn.javass..Secure注解；如public void test(Collection<Model> collection);Model类型上持有@cn.javass..Secure |
   
   2.within：使用“within(类型表达式)”匹配指定类型内的方法执行；
   
   | 模式                                 | 描述                                                         |
   | ------------------------------------ | ------------------------------------------------------------ |
   | within(cn.javass..*)                 | cn.javass包及子包下的任何方法执行                            |
   | within(cn.javass..IPointcutService+) | cn.javass包或所有子包下IPointcutService类型及子类型的任何方法 |
   | within(@cn.javass..Secure *)         | 持有cn.javass..Secure注解的任何类型的任何方法必须是在目标对象上声明这个注解，在接口上声明的对它不起作用 |
   
   3.this：使用“this(类型全限定名)”匹配当前AOP代理对象类型的执行方法；注意是AOP代理对象的类型匹配，这样就可能包括引入接口方法也可以匹配；注意this中使用的表达式必须是类型全限定名，不支持通配符；
   
   4.target：使用“target(类型全限定名)”匹配当前目标对象类型的执行方法；注意是目标对象的类型匹配，这样就不包括引入接口也类型匹配；注意target中使用的表达式必须是类型全限定名，不支持通配符；
   
   5.args：使用“args(参数类型列表)”匹配当前执行的方法传入的参数为指定类型的执行方法；注意是匹配传入的参数类型，不是匹配方法签名的参数类型；参数类型列表中的参数必须是类型全限定名，通配符不支持；args属于动态切入点，这种切入点开销非常大，非特殊情况最好不要使用；
   
   6.@within：使用“@within(注解类型)”匹配所以持有指定注解类型内的方法；注解类型也必须是全限定类型名；
   
   | @within (cn.javass.spring.chapter6.Secure) | 任何目标对象对应的类型持有Secure注解的类方法；必须是在目标对象上声明这个注解，在接口上声明的对它不起作用 |
   | ------------------------------------------ | ------------------------------------------------------------ |
   
   7.@target：使用“@target(注解类型)”匹配当前目标对象类型的执行方法，其中目标对象持有指定的注解；注解类型也必须是全限定类型名；
   
   | @target (cn.javass.spring.chapter6.Secure) | 任何目标对象持有Secure注解的类方法；必须是在目标对象上声明这个注解，在接口上声明的对它不起作用 |
   | ------------------------------------------ | ------------------------------------------------------------ |
   
   8.@args：使用“@args(注解列表)”匹配当前执行的方法传入的参数持有指定注解的执行；注解类型也必须是全限定类型名；
   
   | 模式                                     | 描述                                                         |
   | ---------------------------------------- | ------------------------------------------------------------ |
   | @args (cn.javass.spring.chapter6.Secure) | 任何一个只接受一个参数的方法，且方法运行时传入的参数持有注解 cn.javass.spring.chapter6.Secure；动态切入点，类似于arg指示符； |
   
   9.@annotation：使用“@annotation(注解类型)”匹配当前执行方法持有指定注解的方法；注解类型也必须是全限定类型名；
   
   | @annotation(cn.javass.spring.chapter6.Secure ) | 当前执行方法上持有注解 cn.javass.spring.chapter6.Secure将被匹配 |
   | ---------------------------------------------- | ------------------------------------------------------------ |
   
   组合
   
   ```
   @Pointcut("@annotation(com.ruoyi.common.annotation.DataSource)"   
           + "|| @within(com.ruoyi.common.annotation.DataSource)")
   ```
   

JointPoint使用

```
//通过joinPoint获取方法
Signature signature = joinPoint.getSignature();
MethodSignature methodSignature = (MethodSignature) signature;
Method method = methodSignature.getMethod();
//通过joinPoint获取类
joinPoint.getTarget().getClass()
//通过joinPoint获取方法参数 s
joinPoint.getArgs()
```

## 日志AOP

```
@Aspect
@Component
public class LogAspect
{
    private static final Logger log = LoggerFactory.getLogger(LogAspect.class);
    // 配置织入点
    @Pointcut("@annotation(com.ruoyi.common.annotation.Log)")
    public void logPointCut()
    {
    }
    @AfterReturning(pointcut = "logPointCut()", returning = "jsonResult")
    public void doAfterReturning(JoinPoint joinPoint, Object jsonResult)
    {
        handleLog(joinPoint, null, jsonResult);
    }

    @AfterThrowing(value = "logPointCut()", throwing = "e")
    public void doAfterThrowing(JoinPoint joinPoint, Exception e)
    {
        handleLog(joinPoint, e, null);
    }
    //或者一次性使用环绕通知
    @Around("logPointCut()")
	public void around(ProceedingJoinPoint point) {
		long beginTime = System.currentTimeMillis();
		try {
			// 执行方法
			point.proceed();
		} catch (Throwable e) {
			e.printStackTrace();
		}
		// 执行时长(毫秒)
		long time = System.currentTimeMillis() - beginTime;
		// 保存日志
		saveLog(point, time);
	}
	private void saveLog(ProceedingJoinPoint joinPoint, long time) {
		MethodSignature signature = (MethodSignature) joinPoint.getSignature();
		Method method = signature.getMethod();
		SysLog sysLog = new SysLog();
		Log logAnnotation = method.getAnnotation(Log.class);
		if (logAnnotation != null) {
			// 注解上的描述
			sysLog.setOperation(logAnnotation.value());
		}
		// 请求的方法名
		String className = joinPoint.getTarget().getClass().getName();
		String methodName = signature.getName();
		sysLog.setMethod(className + "." + methodName + "()");
		// 请求的方法参数值
		Object[] args = joinPoint.getArgs();
		// 请求的方法参数名称
		LocalVariableTableParameterNameDiscoverer u = new LocalVariableTableParameterNameDiscoverer();
		String[] paramNames = u.getParameterNames(method);
		if (args != null && paramNames != null) {
			String params = "";
			for (int i = 0; i < args.length; i++) {
				params += "  " + paramNames[i] + ": " + args[i];
			}
			sysLog.setParams(params);
		}
		// 获取request
		HttpServletRequest request = HttpContextUtils.getHttpServletRequest();
		// 设置IP地址
		sysLog.setIp(IPUtils.getIpAddr(request));
		// 模拟一个用户名
		sysLog.setUsername("mrbird");
		sysLog.setTime((int) time);
		Date date = new Date();
		sysLog.setCreateTime(date);
		// 保存系统日志
		sysLogDao.saveSysLog(sysLog);
	}
}
```

## 脚手架

注解+增强处理器

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

![](D:\Study notes\LZFNotes\utils\note\picture\aop-order.jpg)

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

# @Async异步调用

再Spring Boot中，提供了`@Async`注解就能将原来的同步函数变为异步函数,

为了让@Async注解能够生效，还需要在Spring Boot的主程序中配置@EnableAsync

测试

```
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
    @Async
    public void doTaskThree() throws Exception {
        System.out.println("开始做任务三");
        long start = System.currentTimeMillis();
        Thread.sleep(random.nextInt(10000));
        long end = System.currentTimeMillis();
        System.out.println("完成任务三，耗时：" + (end - start) + "毫秒");
    }
}
```

```
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
		task.doTaskThree();
	}

}
```

此时可以反复执行单元测试，您可能会遇到各种不同的结果，比如：

- 没有任何任务相关的输出
- 有部分任务相关的输出
- 乱序的任务相关的输出

原因是目前`doTaskOne`、`doTaskTwo`、`doTaskThree`三个函数的时候已经是异步执行了。主程序在异步调用之后，主程序并不会理会这三个函数是否执行完成了，由于没有其他需要执行的内容，所以程序就自动结束了，导致了不完整或是没有输出任务相关内容的情况。

**注： @Async所修饰的函数不要定义为static类型，这样异步调用不会生效**

## 异步回调

为了让`doTaskOne`、`doTaskTwo`、`doTaskThree`能正常结束，假设我们需要统计一下三个任务并发执行共耗时多少，这就需要等到上述三个函数都完成调动之后记录时间，并计算结果。

我们需要使用`Future<T>`来返回异步调用的结果.

修改:

```
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
    @Async
    public Future<String> doTaskTwo() throws Exception {
        System.out.println("开始做任务二");
        long start = System.currentTimeMillis();
        Thread.sleep(random.nextInt(10000));
        long end = System.currentTimeMillis();
        System.out.println("完成任务二，耗时：" + (end - start) + "毫秒");
        return new AsyncResult<>("任务二完成");
    }
    @Async
    public Future<String> doTaskThree() throws Exception {
        System.out.println("开始做任务三");
        long start = System.currentTimeMillis();
        Thread.sleep(random.nextInt(10000));
        long end = System.currentTimeMillis();
        System.out.println("完成任务三，耗时：" + (end - start) + "毫秒");
        return new AsyncResult<>("任务三完成");
    }
}
```

```
@Test
public void test() throws Exception {

	long start = System.currentTimeMillis();

	Future<String> task1 = task.doTaskOne();
	Future<String> task2 = task.doTaskTwo();
	Future<String> task3 = task.doTaskThree();

	while(true) {
		if(task1.isDone() && task2.isDone() && task3.isDone()) {
			// 三个任务都调用完成，退出循环等待
			break;
		}
		Thread.sleep(1000);
	}
	long end = System.currentTimeMillis();
	System.out.println("任务全部完成，总耗时：" + (end - start) + "毫秒");
}
```

### Future简单示例

`Future`是对于具体的`Runnable`或者`Callable`任务的执行结果进行取消、查询是否完成、获取结果的接口。必要时可以通过get方法获取执行结果，该方法会阻塞直到任务返回结果。

它声明这样的五个方法：

- cancel方法用来取消任务，如果取消任务成功则返回true，如果取消任务失败则返回false。参数mayInterruptIfRunning表示是否允许取消正在执行却没有执行完毕的任务，如果设置true，则表示可以取消正在执行过程中的任务。如果任务已经完成，则无论mayInterruptIfRunning为true还是false，此方法肯定返回false，即如果取消已经完成的任务会返回false；如果任务正在执行，若mayInterruptIfRunning设置为true，则返回true，若mayInterruptIfRunning设置为false，则返回false；如果任务还没有执行，则无论mayInterruptIfRunning为true还是false，肯定返回true。
- isCancelled方法表示任务是否被取消成功，如果在任务正常完成前被取消成功，则返回 true。
- isDone方法表示任务是否已经完成，若任务完成，则返回true；
- get()方法用来获取执行结果，这个方法会产生阻塞，会一直等到任务执行完毕才返回；
- get(long timeout, TimeUnit unit)用来获取执行结果，如果在指定时间内，还没获取到结果，就直接返回null。

```
@Slf4j
@RunWith(SpringJUnit4ClassRunner.class)
@SpringBootTest
public class ApplicationTests {
    @Autowired
    private Task task;

    @Test
    public void test() throws Exception {
        Future<String> futureResult = task.run();
        String result = futureResult.get(5, TimeUnit.SECONDS);
        log.info(result);
    }
}
```

在get方法中还定义了该线程执行的超时时间，通过执行这个测试我们可以观察到执行时间超过5秒的时候，这里会抛出超时异常，该执行线程就能够因执行超时而释放回线程池，不至于一直阻塞而占用资源。

### 内存溢出(配置默认线程池)

通过异步任务加速执行的实现，是否存在问题或风险呢？

```
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

虽然，从单次接口调用来说，是没有问题的。但当接口被客户端频繁调用的时候，异步任务的数量就会大量增长：3 x n（n为请求数量），如果任务处理不够快，就很可能会出现内存溢出的情况。那么为什么会内存溢出呢？根本原因是由于Spring Boot默认用于异步任务的线程池是这样配置的：

```
public static class Pool{
	private int queueCapacity = Integer.MAX_VALUE; //缓冲队列的容量
	private int maxSize = Integer.MAX_VALUE; //允许的最大线程数
	//...
}
```

所以，默认情况下，一般任务队列就可能把内存给堆满了。所以，我们真正使用的时候，还需要对异步任务的执行线程池做一些基础配置，以防止出现内存溢出导致服务不可用的问题。

配置默认线程池:

```
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

```
@Test
public void test1() throws Exception {
    long start = System.currentTimeMillis();

    CompletableFuture<String> task1 = asyncTasks.doTaskOne();
    CompletableFuture<String> task2 = asyncTasks.doTaskTwo();
    CompletableFuture<String> task3 = asyncTasks.doTaskThree();

    CompletableFuture.allOf(task1, task2, task3).join();

    long end = System.currentTimeMillis();

    log.info("任务全部完成，总耗时：" + (end - start) + "毫秒");
}
```

```
2021-09-15 00:31:50.013  INFO 77985 --- [         task-1] com.didispace.chapter76.AsyncTasks       : 开始做任务一
2021-09-15 00:31:50.013  INFO 77985 --- [         task-2] com.didispace.chapter76.AsyncTasks       : 开始做任务二
2021-09-15 00:31:52.452  INFO 77985 --- [         task-1] com.didispace.chapter76.AsyncTasks       : 完成任务一，耗时：2439毫秒
2021-09-15 00:31:52.452  INFO 77985 --- [         task-1] com.didispace.chapter76.AsyncTasks       : 开始做任务三
2021-09-15 00:31:55.880  INFO 77985 --- [         task-2] com.didispace.chapter76.AsyncTasks       : 完成任务二，耗时：5867毫秒
2021-09-15 00:32:00.346  INFO 77985 --- [         task-1] com.didispace.chapter76.AsyncTasks       : 完成任务三，耗时：7894毫秒
2021-09-15 00:32:00.347  INFO 77985 --- [           main] c.d.chapter76.Chapter76ApplicationTests  : 任务全部完成，总耗时：10363毫秒
```

- 任务一和任务二会马上占用核心线程，任务三进入队列等待
- 任务一完成，释放出一个核心线程，任务三从队列中移出，并占用核心线程开始处理

**注意**：这里可能有的小伙伴会问，最大线程不是5么，为什么任务三是进缓冲队列，不是创建新线程来处理吗？这里要理解缓冲队列与最大线程间的关系：只有在缓冲队列满了之后才会申请超过核心线程数的线程来进行处理。所以，这里只有缓冲队列中10个任务满了，再来第11个任务的时候，才会在线程池中创建第三个线程来处理。这个这里就不具体写列子了，读者可以自己调整下参数，或者调整下单元测试来验证这个逻辑。

## 自定义线程池

```
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

```
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
    @Async("taskExecutor")
    public void doTaskThree() throws Exception {
        log.info("开始做任务三");
        long start = System.currentTimeMillis();
        Thread.sleep(random.nextInt(10000));
        long end = System.currentTimeMillis();
        log.info("完成任务三，耗时：" + (end - start) + "毫秒");
    }
}
```

测试: 这里主线程使用join方法等待子线程执行完毕

```
@RunWith(SpringJUnit4ClassRunner.class)
@SpringBootTest
public class ApplicationTests {

    @Autowired
    private Task task;

    @Test
    public void test() throws Exception {

        task.doTaskOne();
        task.doTaskTwo();
        task.doTaskThree();

        Thread.currentThread().join();
    }
}
```

### 线程池隔离

知道了如何自定义线程池后,我们可以采用创建多个线程池,通过@Async指定线程池名称使得不同的任务使用不同的线程池.

```
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

```
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

```
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

执行情况:

1. 线程池1的三个任务，task1和task2会先获得执行线程，然后task3因为没有可分配线程进入缓冲队列
2. 线程池2的三个任务，task4和task5会先获得执行线程，然后task6因为没有可分配线程进入缓冲队列
3. 任务task3会在task1或task2完成之后，开始执行
4. 任务task6会在task4或task5完成之后，开始执行

### 线程池的拒绝策略

假设，线程池配置为核心线程数2、最大线程数2、缓冲队列长度2。此时，有5个异步任务同时开始，会发生什么？

```
 @Bean
 public Executor taskExecutor1() {
    ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
    executor.setCorePoolSize(2);
    executor.setMaxPoolSize(2);
    executor.setQueueCapacity(2);
    executor.setKeepAliveSeconds(60);
    executor.setThreadNamePrefix("executor-1-");
    return executor;
}

 		// 线程池1
        CompletableFuture<String> task1 = asyncTasks.doTaskOne("1");
        CompletableFuture<String> task2 = asyncTasks.doTaskOne("2");
        CompletableFuture<String> task3 = asyncTasks.doTaskOne("3");
        CompletableFuture<String> task4 = asyncTasks.doTaskOne("4");
        CompletableFuture<String> task5 = asyncTasks.doTaskOne("5");

        // 一起执行
        CompletableFuture.allOf(task1, task2, task3, task4, task5).join();
```

会出现异常 `org.springframework.core.task.TaskRejectedException: Executor [java.util.concurrent.ThreadPoolExecutor@3e1a3801[Running, pool size = 2, active threads = 2, queued tasks = 2, completed tasks = 0]] did not accept task:`中，可以很明确的知道，第5个任务因为超过了执行线程+缓冲队列长度，而被拒绝了。

所有，**默认情况下，线程池的拒绝策略是：当线程池队列满了，会丢弃这个任务，并抛出异常**。

**配置拒绝策略**

虽然线程池有默认的拒绝策略，但实际开发过程中，有些业务场景，直接拒绝的策略往往并不适用，有时候我们可能会选择舍弃最早开始执行而未完成的任务、也可能会选择舍弃刚开始执行而未完成的任务等更贴近业务需要的策略。所以，为线程池配置其他拒绝策略或自定义拒绝策略是很常见的需求.

```
ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
//...其他线程池配置
executor.setRejectedExecutionHandler(new ThreadPoolExecutor.AbortPolicy());
```

在`ThreadPoolExecutor`中提供了4种线程的策略可以供开发者直接使用，你只需要像下面这样设置即可：

```
// AbortPolicy策略:默认策略，如果线程池队列满了丢掉这个任务并且抛出RejectedExecutionException异常。
executor.setRejectedExecutionHandler(new ThreadPoolExecutor.AbortPolicy());

// DiscardPolicy策略:如果线程池队列满了，会直接丢掉这个任务并且不会有任何异常。
executor.setRejectedExecutionHandler(new ThreadPoolExecutor.DiscardPolicy());

// DiscardOldestPolicy策略:如果队列满了，会将最早进入队列的任务删掉腾出空间，再尝试加入队列。
executor.setRejectedExecutionHandler(new ThreadPoolExecutor.DiscardOldestPolicy());

// CallerRunsPolicy策略:如果添加到线程池失败，那么主线程会自己去执行该任务，不会等待线程池中的线程去执行。
executor.setRejectedExecutionHandler(new ThreadPoolExecutor.CallerRunsPolicy());
```

而如果你要自定义一个拒绝策略，那么可以这样写：

```
executor.setRejectedExecutionHandler(new RejectedExecutionHandler() {
    @Override
    public void rejectedExecution(Runnable r, ThreadPoolExecutor executor) {
        // 拒绝策略的逻辑
    }
});
```

当然如果你喜欢用Lambda表达式，也可以这样写：

```
executor.setRejectedExecutionHandler((r, executor1) -> {
    // 拒绝策略的逻辑
});
```

## 修改默认线程池

实现AsyncConfigurer,  只需要加`@Async`注解就可以，不用在注解属性指定线程池。

```
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



## 线程池优雅关闭

模拟问题

```
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
            ThreadPoolTaskScheduler executor = new ThreadPoolTaskScheduler();
            executor.setPoolSize(20);
            executor.setThreadNamePrefix("taskExecutor-");
            return executor;
        }
    }
}
```

这里使用了外部资源redis

```
@Slf4j
@Component
public class Task {
    @Autowired
    private StringRedisTemplate stringRedisTemplate;
    @Async("taskExecutor")
    public void doTaskOne() throws Exception {
        log.info("开始做任务一");
        long start = System.currentTimeMillis();
        log.info(stringRedisTemplate.randomKey());
        long end = System.currentTimeMillis();
        log.info("完成任务一，耗时：" + (end - start) + "毫秒");
    }
    @Async("taskExecutor")
    public void doTaskTwo() throws Exception {
        log.info("开始做任务二");
        long start = System.currentTimeMillis();
        log.info(stringRedisTemplate.randomKey());
        long end = System.currentTimeMillis();
        log.info("完成任务二，耗时：" + (end - start) + "毫秒");
    }
    @Async("taskExecutor")
    public void doTaskThree() throws Exception {
        log.info("开始做任务三");
        long start = System.currentTimeMillis();
        log.info(stringRedisTemplate.randomKey());
        long end = System.currentTimeMillis();
        log.info("完成任务三，耗时：" + (end - start) + "毫秒");
    }
}
```

测试方法 (模拟高并发下的ShutDown情况)

```
@RunWith(SpringJUnit4ClassRunner.class)
@SpringBootTest
public class ApplicationTests {
    @Autowired
    private Task task;
    @Test
    @SneakyThrows
    public void test() {
        for (int i = 0; i < 10000; i++) {
            task.doTaskOne();
            task.doTaskTwo();
            task.doTaskThree();

            if (i == 9999) {
                System.exit(0);
            }
        }
    }
}
```

说明：通过for循环往上面定义的线程池中提交任务，由于是异步执行，在执行过程中，利用`System.exit(0)`来关闭程序，此时由于有任务在执行，就可以观察这些异步任务的销毁与Spring容器中其他资源的顺序是否安全。

运行上面的单元测试，我们将碰到下面的异常内容。

```
JedisConnectionException: Could not get a resource from the pool
```

应用关闭的时候异步任务还在执行，由于Redis连接池先销毁了，导致异步任务中要访问Redis的操作就报了上面的错.

**解决办法**

```
@Bean("taskExecutor")
public Executor taskExecutor() {
    ThreadPoolTaskScheduler executor = new ThreadPoolTaskScheduler();
    executor.setPoolSize(20);
    executor.setThreadNamePrefix("taskExecutor-");
    executor.setWaitForTasksToCompleteOnShutdown(true);    //关键
    executor.setAwaitTerminationSeconds(60);  
    return executor;
}
```

说明：`setWaitForTasksToCompleteOnShutdown（true）`该方法就是这里的关键，用来设置线程池关闭的时候等待所有任务都完成再继续销毁其他的Bean，这样这些异步任务的销毁就会先于Redis线程池的销毁。同时，这里还设置了`setAwaitTerminationSeconds(60)`，该方法用来设置线程池中任务的等待时间，如果超过这个时候还没有销毁就强制销毁，以确保应用最后能够被关闭，而不是阻塞住









# 获取配置文件内容

```
applicationContext.getBean(Environment.class).getProperty(key,defaultValue)
```

