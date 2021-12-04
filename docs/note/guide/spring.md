# 设计模式

**工厂设计模式** : Spring 使用工厂模式通过 `BeanFactory`、`ApplicationContext` 创建 bean 对象。

两者对比：

- `BeanFactory` ：延迟注入(使用到某个 bean 的时候才会注入),相比于`ApplicationContext` 来说会占用更少的内存，程序启动速度更快。
- `ApplicationContext` ：容器启动的时候，不管你用没用到，一次性创建所有 bean 。`BeanFactory` 仅提供了最基本的依赖注入支持，`ApplicationContext` 扩展了 `BeanFactory` ,除了有`BeanFactory`的功能还有额外更多功能，所以一般开发人员使用`ApplicationContext`会更多。



**代理设计模式** : Spring AOP 功能的实现。如果要代理的对象，实现了某个接口，那么Spring AOP会使用**JDK Proxy**，去创建代理对象，而对于没有实现接口的对象，就无法使用 JDK Proxy 去进行代理了，这时候Spring AOP会使用**Cglib** ，这时候Spring AOP会使用 **Cglib** 生成一个被代理对象的子类来作为代理.



**单例设计模式** : Spring 中的 Bean 默认都是单例的。Spring 通过 `ConcurrentHashMap` 实现单例注册表的特殊方式实现单例模式。

```
// 通过 ConcurrentHashMap（线程安全） 实现单例注册表
private final Map<String, Object> singletonObjects = new ConcurrentHashMap<String, Object>(64);

public Object getSingleton(String beanName, ObjectFactory<?> singletonFactory) {
        Assert.notNull(beanName, "'beanName' must not be null");
        synchronized (this.singletonObjects) {
            // 检查缓存中是否存在实例  
            Object singletonObject = this.singletonObjects.get(beanName);
            if (singletonObject == null) {
                //...省略了很多代码
                try {
                    singletonObject = singletonFactory.getObject();
                }
                //...省略了很多代码
                // 如果实例对象在不存在，我们注册到单例注册表中。
                addSingleton(beanName, singletonObject);
            }
            return (singletonObject != NULL_OBJECT ? singletonObject : null);
        }
    }
    //将对象添加到单例注册表
    protected void addSingleton(String beanName, Object singletonObject) {
            synchronized (this.singletonObjects) {
                this.singletonObjects.put(beanName, (singletonObject != null ? singletonObject : NULL_OBJECT));

            }
        }
}
```

**模板方法模式** : Spring 中 `jdbcTemplate`、`hibernateTemplate` 等以 Template 结尾的对数据库操作的类，它们就使用到了模板模式。一个抽象类公开定义了执行它的方法的方式/模板。它的子类可以按需要重写方法实现，但调用将以抽象类中定义的方式进行。这种类型的设计模式属于行为型模式。

```
public abstract class Game {
   abstract void initialize();
   abstract void startPlay();
   abstract void endPlay();
   //模板
   public final void play(){
      //初始化游戏
      initialize();
      //开始游戏
      startPlay();
      //结束游戏
      endPlay();
   }
}
```

**装饰者模式** : 允许向一个现有的对象添加新的功能，同时又不改变其结构。比如 `InputStream`家族.

Spring 中用到的包装器模式在类名上含有 `Wrapper`或者 `Decorator`。这些类基本上都是动态地给一个对象添加一些额外的职责

**观察者模式:** Spring 事件驱动模型就是观察者模式很经典的一个应用。

Spring事件模型中三个角色:

事件ApplicationEvent,它继承了`java.util.EventObject`并实现了 `java.io.Serializable`接口

监听者ApplicationListener,里面只定义了一个 `onApplicationEvent（）`方法来处理`ApplicationEvent`

发布者ApplicationEventPublisher,接口的`publishEvent（）`这个方法在`AbstractApplicationContext`类中被实现,通过`ApplicationEventMulticaster`来广播出去

**适配器模式** : Spring AOP 的增强或通知(Advice)使用到了适配器模式、spring MVC 中也是用到了适配器模式适配`Controller`。

在AOP中:

与'Advice'之相关的接口是`AdvisorAdapter` 。Advice 常用的类型有：`BeforeAdvice`（目标方法调用前,前置通知）、`AfterAdvice`（目标方法调用后,后置通知）、`AfterReturningAdvice`(目标方法执行结束后，return之前)等等。每个类型Advice（通知）都有对应的拦截器:`MethodBeforeAdviceInterceptor`、`AfterReturningAdviceAdapter`、`AfterReturningAdviceInterceptor`。Spring预定义的通知要通过对应的适配器，适配成 `MethodInterceptor`接口(方法拦截器)类型的对象（如：`MethodBeforeAdviceInterceptor` 负责适配 `MethodBeforeAdvice`）。

在MVC中:

`DispatcherServlet` 根据请求信息调用 `HandlerMapping`，解析请求对应的 `Handler`。解析到对应的 `Handler`（也就是我们平常说的 `Controller` 控制器）后，开始由`HandlerAdapter` 适配器处理。`HandlerAdapter` 作为期望接口，具体的适配器实现类用于对目标类进行适配，`Controller` 作为需要适配的类。



# IOC

[源码深入](https://javadoop.com/post/spring-ioc)

## Bean

声明周期![Spring Bean 生命周期](picture/24bc2bad3ce28144d60d9e0a2edf6c7f.jpg)

bean定义信息   -->

xml  &  注解 ... (如可以自设计json)  -->

抽象定义规范接口  BeanDefinition

Reader  -->

BeanDefinition  bean定义信息   -->  ioc容器

多个BeanFactoryPostProcessor  后置处理器

```
<bean id="xxx" class="xxx">
	<property name="xxx" value=${xxx.xxx}></property>  在后置处理器中进行替换(PlaceholderConfigurerSupport实现类)
</bean>

又比如springboot componentScan注解扫描的实现类 ConfigurationClassProcessor (多层调用后在ConfigurationClssParser中解析带有注解@Component , @PropertySource , @ComponentScan , @Import , @ImpoertSource的类 ,  @Bean的方法 ...)

可以通过实现该接口 获取beanDefinition 来设置属性值达到扩展效果
```

  -- >BeanFactory  反射生成对象 该接口的类上注释了  所有aware和声明周期接口执行顺序

```
1.实例化 : 开辟堆空间
2.初始化 : 给属性赋值   (类似python#__init__) 
 1)填充属性    populateBean方法 通过setter
 2)执行aware接口需要实现的方法 :通过spring中的bean对象来获取相应容器中的相关属性
比如bean要支持通过bean获取beanName , 可以实现BeanNameAware接口,实现方法setBeanName(String name)
 3)BeanPostProcessor#postProcessBeforeInitialization
 4)init-method
 5)BeanPostProcessor#postProcesspostProcessAfterInitialization
比如 aop代理就是在实现类AbstractAutoProxyCreator的接口方法postProcessAfterInitialization中完成代理对象并返回
```

-->  完整对象



> 在容器运行前置时 Environment -> StandardEnvironment  调用System#getenv$getProperty将属性存入Environment,方便后续调用
>
> 如mvc init-param属性

### refresh

ClassPathXmlApplicationContext.refresh

1.prepareRefresh()

前戏,做容器刷新前的准备工作

①设置容器的启动时间;

②设置活跃状态为true;

③设置关闭状态为false;

④获取Environment对象并加载当前系统的属性值到Environment对象中;

⑤准备监听器和事件的集合对象,默认为空的集合.

2.obtainFreshBeanFactory()

创建容器对象**DefaultListableBeanFactory**  完成配置文件加载和解析工作 ,转换为beanDefine,比如xml配置的bean

3.prepareBeanFactory(beanFactory)

beanFactory的准备工作, 对其各种属性进行填充.

4.**post**ProcesssBeanFactoryPostProcessors(beanFactory)

子类覆盖方法做额外的处理

5.invokeBeanFactoryPostProcessors(beanFactory)

调用各种beanFactory处理器 如beanFactoryPostProcessor

6.registerBeanPostProcessors(beanFactory)

注册bean处理器,这里仅注册,真正执行是在getBean方法

7.initMessageSource()

初始化上下文message源 , 即不同语言的消息体,国际化处理

比如mvc i18n

8.initApplicaitonEventMulticaster()

初始化事件监听多路广播器

9.onRefresh

子类实现 , 初始化其他bean

10.RegisterListeners

在所有注册的bean中查找listener bean ,注册到消息广播器中

-------------------- 目前为止所有准备工作已做完--------

11.finishBeanFactoryInitialization(beanFactory)

初始化剩下的非懒加载的单实例

```
beanFactory.preInstantiateSingletons() --> beanFactory.getBean(name)
--> doGetBean(...)  -->createBean(...)  -->createBeanInstance(...) --> instantiateBean(...) -->instantiate(...) 获取构造器  							   ->BeanUtils.instantiateClass(ctor,args) ctor.newInstance() 实例化
```

```
populateBean(..)  bean属性填充,如果依赖于其他bean的属性,则会递归初始化依赖的bean
```

```
initializeBean(...)  --> invokeAwareMethods(beanName,bean) 对特殊的bean处理Aware接口 (BeanNameAware . BeanClssLoaderAware , BeanFactoryAware) ,注意 其他bean的Aware接口方法并没有执行  如EnvironmentAware
```

```
applyBeanPostProcessorsBeforeInitialization(existingBean ,beanName)   -->
processor.postProcessBeforeInitialization(xx) 中执行其他Aware接口方法
```

```
invokeInitMethods(xxx) 调用用户自定义init方法
```

```
applyBeanPostProcessorsBeforeInitialization(existingBean ,beanName)  -->
```

### 循环依赖

ioc中

构造方法无法解决循环依赖问题,而通过settor方法可以解决循环依赖问题.

即**将实例化和初始化分开处理,提前暴露对象** ,在中间过程给其他对象赋值的时候,并不是一个完整对象

假如有A依赖B B依赖A的场景:

```
<bean id="a" class="org.example.A">
	<property name="b" ref="b">
</bean>
<bean id="b" class="org.example.B">
	<property name="a" ref="a">
</bean>
```

![循环依赖](picture/循环依赖.png)

```
一级缓存 存放成品bean
private final Map<String, Object> singletonObjects = new ConcurrentHashMap<>(256);
三级缓存
private final Map<String, ObjectFactory<?>> singletonFactories = new HashMap<>(16);
二级缓存 存放半成品bean
private final Map<String, Object> earlySingletonObjects = new ConcurrentHashMap<>(16);
```

其中,泛型`ObjectFactory`为函数式接口,只提供了getObject方法

因为循环依赖问题存在于实例化初始化阶段,所以该问题需要在refresh方法的finishBeanFactoryInitialization(beanFactory)方法中的beanFactory.preInstantiateSingletons方法中解决



而使用第三级缓存原因在于使用aop代理问题



















# AOP

`createAopProxy()` 方法 决定了是使用 JDK 还是 Cglib 来做动态代理，源码如下：

```java
public class DefaultAopProxyFactory implements AopProxyFactory, Serializable {

	@Override
	public AopProxy createAopProxy(AdvisedSupport config) throws AopConfigException {
		if (config.isOptimize() || config.isProxyTargetClass() || hasNoUserSuppliedProxyInterfaces(config)) {
			Class<?> targetClass = config.getTargetClass();
			if (targetClass == null) {
				throw new AopConfigException("TargetSource cannot determine target class: " +
						"Either an interface or a target is required for proxy creation.");
			}
			if (targetClass.isInterface() || Proxy.isProxyClass(targetClass)) {
				return new JdkDynamicAopProxy(config);
			}
			return new ObjenesisCglibAopProxy(config);
		}
		else {
			return new JdkDynamicAopProxy(config);
		}
	}
  .......
}
```



# MVC

![img](picture/de6d2b213f112297298f3e223bf08f28.png)

# cloud



## Ribbon负载均衡与nginx

`Ribbon` 是运行在消费者端的负载均衡器,工作原理就是 `Consumer` 端获取到了所有的服务列表之后，在其**内部**使用**负载均衡算法**，进行对多个系统的调用。

![img](picture/nginx-vs-ribbon2.jpg)

`nginx`是一种**集中式**的负载均衡器,**将所有请求都集中起来，然后再进行负载均衡**

![img](picture/nginx-vs-ribbon1.jpg)

## feign

factoryBean