# 设计模式

**工厂设计模式** : Spring 使用工厂模式通过 `BeanFactory`(懒注入)、`ApplicationContext`(一次性注入且功能更丰富) 创建 bean 对象。

**代理设计模式** : Spring AOP -> JDK/CGLIB

**单例设计模式** : 默认Bean

**模板方法模式** : Spring中应用广泛, 如AbstractEnviroment提供的customizePropertySources方法让子类自定义添加PropertySource, 类似有许多以customize开头的方法, 以及refresh中的onRefresh方法, 还有xxxTemplate类也使用了该设计模式

**装饰者模式** : 允许向一个现有的对象添加新的功能，同时又不改变其结构。比如 `InputStream`.

Spring 中用到的包装器模式在类名上含有 `Wrapper`或者 `Decorator`。如HttpServletRequestWrapper

**观察者模式:** Spring 事件驱动模型

**适配器模式** : Spring AOP 的增强(AdvisorAdapter适配通知)和spring MVC 中的HanderMapping(HandlerAdapter适配控制器)。



# ioc

## refresh

> 为什么命名refresh而不叫init, 因为 ApplicationContext 建立起来以后，其实我们是可以通过调用 refresh() 这个方法重建的，refresh() 会将原来的 ApplicationContext 销毁，然后再重新执行一次初始化操作。

ClassPathXmlApplicationContext.refresh

1.prepareRefresh()

前戏,做容器刷新前的准备工作

①设置容器的启动时间;

②设置活跃状态为true;

③设置关闭状态为false;

④获取Environment对象并加载当前系统的属性值到Environment对象中(处理配置文件中的占位符);

⑤准备监听器和事件的集合对象,默认为空的集合.

2.obtainFreshBeanFactory()

创建容器对象**DefaultListableBeanFactory**  完成配置文件加载和解析工作 ,**转换为beanDefine**,比如xml配置的bean( 通过XmlBeanDefinitionReader).

DefaultListableBeanFactory是ConfigurableListableBeanFactory的实现类, 是一个功能强大的真实BeanFactory,

不管是类的初始化, 还是用户在ioc运行时动态注册bean, 都可以使用此类(ApplicationContext接口可获取AutowireCapableBeanFactory, 然后向下转型可得到该类).

核心实际上为beanName -> beanDefinition的map.

> ApplicationContext 继承自 BeanFactory，但是它不应该被理解为 BeanFactory 的实现类，而是说其内部持有一个实例化的 BeanFactory（DefaultListableBeanFactory）。以后所有的 BeanFactory 相关的操作其实是委托给这个实例来处理的。

> BeanDefinition 中保存了Bean 信息，比如这个 Bean 指向的是哪个类、是否是单例的、是否懒加载、这个 Bean 依赖了哪些 Bean 等。

3.prepareBeanFactory(beanFactory)

beanFactory的准备工作, 对其各种属性进行填充.

设置 BeanFactory 的类加载器，添加几个 BeanPostProcessor,如用于回调实现了Aware接口的beans方法的ApplicationContextAwareProcessor，添加ApplicationListener, 手动注册几个特殊的 bean.

4.postProcesssBeanFactoryPostProcessors(beanFactory)

子类可以在这步的时候添加一些特殊的 BeanFactoryPostProcessor 的实现类或做点什么事

5.invokeBeanFactoryPostProcessors(beanFactory)

调用各种BeanFactoryPostProcessor的postProcessBeanFactory(factory)方法, 此时因为bean未初始化, 所以在BeanFactoryPostProcessor对象中获取bean会导致提前初始化。

6.registerBeanPostProcessors(beanFactory)

注册`BeanPostProcessor`,这里仅注册,真正执行是在getBean方法, 其两个方法分别在 Bean 初始化之前和初始化之后得到执行.

7.initMessageSource()

初始化上下文message源 , 即不同语言的消息体,国际化处理

比如mvc i18n

8.initApplicaitonEventMulticaster()

初始化事件监听多路广播器

9.onRefresh

子类实现 , 初始化特殊的bean（在初始化 singleton beans 之前）,  典型的**模板方法**

10.RegisterListeners

在所有注册的bean中查找listener bean ,注册到消息广播器中

-------------------- 目前为止所有准备工作已做完--------

11.finishBeanFactoryInitialization(beanFactory)

**初始化剩下的非懒加载的单实例**

在方法一开始, 初始化了ConversionService ,此接口用于类型之间的转换，在Spring里其实就是把配置文件中的String转为其它类型，从3.0开始出现，目的和jdk的PropertyEditor接口是一样的，具体参考ConfigurableBeanFactory.setConversionService注释;

然后设置一个StringValueResolver用于解析注解的值, 为函数式接口, 这里通过AbstractApplicationContext调用AbstractPropertyResolver的resolvePlaceholder实现(与refresh前解析配置文件名类似)

以下流程不同版本的Spring可能代码变化较大, 但核心逻辑几乎没变:

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

12.finishRefresh

最后广播事件, ApplicationContext初始化完成.

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
三级缓存 提前暴露半成品
private final Map<String, ObjectFactory<?>> singletonFactories = new HashMap<>(16);
二级缓存 存放半成品bean
private final Map<String, Object> earlySingletonObjects = new ConcurrentHashMap<>(16);
```

其中,泛型`ObjectFactory`为函数式接口,只提供了getObject方法

因为循环依赖问题存在于实例化初始化阶段,所以该问题需要在refresh方法的finishBeanFactoryInitialization(beanFactory)方法中的beanFactory.preInstantiateSingletons方法中解决



第三级缓存原因在于使用aop代理问题, 为保证注入的对象不是原始对象, 需要提前创建代理对象.



## 事件驱动

事件对象一般都是java.util.EventObject的子类; ApplicationEventPublisher(即ApplicationContext)将请求委托给ApplicationEventMulticaster来实现的; 监听器是EventListener(jdk)的子类, 在refresh在初始化多播器后注册监听器.

initApplicationEventMulticaster方法执行时, 如果ioc容器中存在多播器, 则赋值给成员变量; 否则向ioc容器中注册一个新的SimpleApplicationEventMulticaster, 该类发布事件的代码为:

```
@Override
public void multicastEvent(final ApplicationEvent event, ResolvableType eventType) {
    ResolvableType type = (eventType != null ? eventType : resolveDefaultEventType(event));
    for (final ApplicationListener<?> listener : getApplicationListeners(event, type)) {
        Executor executor = getTaskExecutor();
        if (executor != null) {
            executor.execute(new Runnable() {
                @Override
                public void run() {
                    invokeListener(listener, event);
                }
            });
        } else {
            invokeListener(listener, event);
        }
    }
}
```

在该方法中, 通过父类AbstractApplicationEventMulticaster的方法获取, 其通过ConcurrentHashMap缓存实现, 事件源为key, 事件所有监听器为value. 如果缓存存在, 则直接返回, 否则通过ioc容器获取.

在该方法中, 如果executor不为空，那么监听器的执行是异步的. 具体配置方式见#spring使用笔记

## getBean

> 下面bean工厂指DefaultListableBeanFactory

refresh的finishBeanFactoryInitialization方法中:

1-1.先初始化conversionService用于类型之间的转换，在Spring里其实就是把配置文件中的String转为其它类型，从3.0开始出现，目的和jdk的PropertyEditor接口是一样的，参考ConfigurableBeanFactory.setConversionService注释.

1-2.添加一个StringValueResolver函数式接口用于解析注解的值, 这里实际上通过Enviroment调用;

1-3.初始化LoadTimeWeaverAware;

1-4.最后调用bean工厂的preInstantiateSingletons方法构建其他bean, 如果bean是FactoryBean类型且还是SmartFactoryBean类型，那么需要判断是否需要eagerInit(isEagerInit是此接口定义的方法)

以上初始化通过AbstractBeanFactory.getBean, 内部调用doGetBean, 参数分别为bean的name,bean的class, 创建bean需要的参数, 是否需要类型检查.

AbstractBeanFactory.doGetBean:

2-1.beanName转换, 如果是FactoryBean去掉前缀;

2-2.尝试直接获取该bean, 如果存在表示是Spring手动注册的bean, 然后通过getObjectForBeanInstance返回, 如果bean是FactoryBean则调用工厂方法创建实例返回, 否则返回Bean本身;

2-3.否则查看父容器是否存在BeanDefine, 如果存在则给父容器初始化;

2-4.BeanDefine的depend-on属性指定了依赖的bean, 如果有则先检测是否存在循环依赖, 然后注册依赖关系(销毁bean时先销毁被依赖的bean, 通过ConcurrentHashMap<String,Set>保存), 初始化依赖的bean;

2-5.如果是singleton, 则通过getSingleton初始化.(方法后面也会初始化其他scope,省略)

DefaultSingletonBeanRegistry.getSingleton(bean工厂的父类, 三级缓存在此):

3-1.查看一级缓存singletonObjects中是否存在bean, 是则直接返回, 否则调用传入的ObjectFactory函数接口的方法, 间接调用AbstractAutowireCapableBeanFactory.createBean方法

AbstractAutowireCapableBeanFactory.createBean:

4-1.调用RootBeanDefinition.prepareMethodOverrides判断该bean的lookup-methods方法是否存在;

4-2.调用resolveBeforeInstantiation执行bean的postProcessBeforeInstantiation方法, 如果返回不为空, 则直接执行postProcessAfterInitialization方法(Bean初始化结束了);

4-3.如果上述方法返回的bean不为空, 则bean创建完成, 直接返回, 方法结束; 否则调用doCreateBean;

AbstractAutowireCapableBeanFactory.doCreateBean:

5-1.调用createBeanInstance方法

AbstractAutowireCapableBeanFactory.createBeanInstance:

6-1.如果beanDefinition存在工厂方法(配置了factory-bean/factory-method), 则调用instantiateUsingFactoryMethod返回bean:

```
调用 ConstructorResolver.instantiateUsingFactoryMethod:
该方法初始化一个BeanWrapper, 最后调用ConstructorResolver.instantiate赋给beanWrapper:
instantiate方法中以策略模式真正执行初始化, 这里使用的是SimpleInstantiationStrategy, 调用其instantiate方法, 就是通过反射调用工厂方法得到对象:factoryMethod.invoke(factoryBean, args);
```

6-2.否则通过构造器自动装配:

```
先调用determineConstructorsFromBeanPostProcessors方法通过SmartInstantiationAwareBeanPostProcessor(Spring内部配置)获取构造器数组, 
(注意:先判断了bean的自动装配模式)然后调用autowireConstructor,
方法里调用了ConstructorResolver.autowireConstructor:
该方法初始化一个BeanWrapper,先找到合适的构造器,
然后调用ConstructorResolver.resolveAutowiredArgument根据构造器的参数类型去bean工厂查找相应的bean注入,
最后调用ConstructorResolver.instantiate赋给beanWrapper:
instantiate方法中以策略模式真正执行初始化,这里使用的是CglibSubclassingInstantiationStrategy, 调用其instantiate方法, 如果bean配置了lookup-methods方法(bd.getMethodOverrides()不为空), 则通过调用instantiateWithMethodInjection使用cglib生成代理对象返回;否则直接通过反射创建返回.
```

5-2.调用populateBean进行属性填充

```
首先获取beanDefinition的所有属性PropertyValues,然后在判断自动注入模式;
如果是AUTOWIRE_BY_NAME, 则调用autowireByName;如果是AUTOWIRE_BY_TYPE,则调用autowireByType.
两方法都是先获取到应用的bean给传入的MutablePropertyValues变量, 真正的赋值在applyPropertyValues中.
注意:applyPropertyValues中填充的条件是bean的该属性具有对应的public的setter方法
```

5-3.调用initializeBean初始化(bean构造且复制完成, 此处指的是init方法)

6-1.调用invokeAwareMethods方法触发Aware方法(bean实现的xxxAware接口)调用:

包括BeanNameAware,BeanClassLoaderAware,BeanFactoryAware

6-2.调用applyBeanPostProcessorsBeforeInitialization方法触发postProcessBeforeInitialization调用

6-3.调用postProcessBeforeInitialization方法触发init调用:

在调用init前, 先会检查bean是否实现了InitializingBean接口, 该接口的afterPropertiesSet允许此bean的所有属性都被设置后，给bean一个利用现有属性重新组织或是检查属性的机会(应用较为广泛)

2-6.调用AbstractBeanFactory.getObjectForBeanInstance

如果bean是FactoryBean，那么返回其工厂方法创建的bean，而不是自身。

2-7.判断scope是否为prototype,如果是则创建该类型bean(每次获取重新创建), 仔细看prototype与singleton区别只是多了一个getSingleton方法方法包装来检查是否已经存在bean

```
beforePrototypeCreation(beanName)方法确保同一时刻只有一个此bean在创建
prototypeInstance = createBean(beanName, mbd, args)方法创建bean, 与单例一样
afterPrototypeCreation与beforePrototypeCreation对应
```

2-8.其他scope: session,request处理.

## BeanDefinitionParser

在refresh的obtainFreshBeanFactory方法中, 会对BeanDefine进行加载与定义, 经过层层调用最终到达BeanDefinitionParserDelegate.parseCustomElement方法, 其中参数Element为经过SAX解析得到的DOM标签元素;

在BeanDefinitionParserDelegate.parseCustomElement方法中, 根据Element的namespaceURI获取对应的NamespaceHandler, 然后调用该handler的parse方法(为各种NamespaceHandler的父类NamespaceHandlerSupport实现);

NamespaceHandlerSupport.parse方法中, 会继续调用findParserForElement通过获取该元素的localName(如context:annotation-config标签就是annotation-config)寻找适用于此元素的BeanDefinitionParser对象; 然后调用对应BeanDefinitionParser.parse方法:

> 注意:对于context开头的的特殊bean, 不再对应普通的BeanDefinition, 而是被抽象成ComponentDefinition组件以完成特定功能, 组件集合表示为CompositeComponentDefinition

1.AnnotationConfigBeanDefinitionParser(annotation-config).parse:

```
1)调用AnnotationConfigUtils.registerAnnotationConfigProcessors注册BeanPostProcessor, 注意,这里传进去的参数parserContext.getRegistry()实际上就是bean工厂, 因此方法一进去就将参数强转为bean工厂:
 1-1)设置AnnotationAwareOrderComparator比较优先级(根据@Order与@Priority);
 1-2)设置ContextAnnotationAutowireCandidateResolver决定bean是否可作为依赖候选者;
 1-3)注册ConfigurationClassPostProcessor处理@Configuration类:
  在refresh的invokeBeanFactoryPostProcessors方法中执行
 1-4)注册AutowiredAnnotationBeanPostProcessor注入@Autowired的属性和方法:
 	先在AbstractAutowireCapableBeanFactory.doCreateBean中调用applyMergedBeanDefinitionPostProcessors方法(实例化bean之后,填充bean之前),调用AutowiredAnnotationBeanPostProcessor.postProcessMergedBeanDefinition扫描Autowired属性和方法保存为InjectionMetadata对象进行缓存;
 	然后在AbstractAutowireCapableBeanFactory.populateBean中调用AutowiredAnnotationBeanPostProcessor.postProcessPropertyValues方法(在bean的xml属性已经根据计算完毕之后,但还没有执行applyPropertyValues设置到bean之前),获取InjectionMetadata缓存然后注入.
 1-5)注册RequiredAnnotationBeanPostProcessor必须注入@Required标注的setter的属性
 	同AutowiredAnnotationBeanPostProcessor类似,但RequiredAnnotationBeanPostProcessor.postProcessMergedBeanDefinition为空实现;
 	仅在RequiredAnnotationBeanPostProcessor.postProcessPropertyValues中对bean进行检查,对已检查过的bean缓存到名为validatedBeanNames的set中防止做无用功.
 1-6)注册CommonAnnotationBeanPostProcessor开启对JSR-250的支持(rt.jar),可使用@Resouce等
 	同上类似
 1-7)注册PersistenceAnnotationBeanPostProcessor提供JPA的支持,classpath不存在,默认不注册
 1-8)注册EventListenerMethodProcessor提供对@EventListener支持
 1-9)注册DefaultEventListenerFactory产生EventLisener对象
2)构建CompositeComponentDefinition并注册组件;
```

2.ComponentScanBeanDefinitionParser(component-scan).parse:

> 注意: 如果配置了components-scan等于配置了annotation-config, 不需要额外配置

```
1)先通过元素获取其basePackage属性并解析占位符,解析分隔符;
2)调用configureScanner方法初始化扫描器:
 2-1)获取use-default-filters属性,如果为false,不扫描@Components、@Service、@Controller、@Repositry注解.
 2-2)创建与初始化ClassPathBeanDefinitionScanner,其中默认的resourcePattern为**/*.class,即扫描路径.
 2-3)调用parseBeanNameGenerator方法设置NameGenerator,调用parseScope方法设置ScopeMetadataResolver(解析scope)
 2-4)调用parseTypeFilters获取子元素exclude-filter与include-filter标签设置给scanner.
3)调用ClassPathBeanDefinitionScanner.scan(basePaceage)进行扫描:
4)构建CompositeComponentDefinition并注册组件;
```

----------------------------------仅需要解析为单个的BeanDefinition(AbstractSingleBeanDefinitionParser子类)

3.PropertyOverrideBeanDefinitionParser(property-override).doParse:

> property-override允许使用属性文件对bean属性进行替换:beanName.属性名=属性值
>
> <context:property-override location="property.properties" />

调用其父类AbstractPropertyLoadingBeanDefinitionParser.doParse,同上面一样获取解析元素以及其子元素的属性并保存在一个GenericBeanDefinition中:

```
1)properties-ref可直接引用一个Properties类型的bean作为数据源;
2)order在配置了多个property-override且存在相同key时指定优先级;
3)ignore-resource-not-found当属性文件不存在时true忽略false报错;
4)ignore-unresolvable当key不存在时true忽略false报错;
5)local-override???
```

保存该BeanDefinition的beanClass为PropertyOverrideConfigurer, 在其父类PropertyResourceConfigurer.postProcessBeanFactory中完成设置.

4.PropertyPlaceholderBeanDefinitionParser.doParse:

beanDefinition方式与上类似

```
1)system-properties-mdoe系统和本地配置同名属性优先级
2)value-separator配置分隔符
3)null-value遇到哪些值当做空值处理
4)trim-values移除开头和结尾空格
```

在PropertySourcesPlaceholderConfigurer.postProcessBeanFactory中完成处理, 对所有BeanDefinition的占位符进行替换.



# aop

> 1. 定义在private方法上的切面不会被执行，因为子类不能覆盖父类的私有方法。
> 2. 同一个代理子类内部的方法相互调用不会再次执行切面。(如@Trasaction)

aop相关解析器由AopNamespaceHandler注册:

```
@Override
public void init() {
    registerBeanDefinitionParser("config", new ConfigBeanDefinitionParser());
    registerBeanDefinitionParser("aspectj-autoproxy", new AspectJAutoProxyBeanDefinitionParser());
    registerBeanDefinitionDecorator("scoped-proxy", new ScopedProxyBeanDefinitionDecorator());
}
```

## ConfigBeanDefinitionParser

aop:config:配合事务使用, 其中advice-ref的advice必须实现org.aopalliance.aop.Advice的子接口.

```
<tx:advice id="txAdvice" transaction-manager="transactionManager">
    <tx:attributes>
        <tx:method name="get*" read-only="true" propagation="NOT_SUPPORTED"/>
        <tx:method name="find*" read-only="true" propagation="NOT_SUPPORTED"/>
        <tx:method name="*" propagation="REQUIRED"/>
    </tx:attributes>
</tx:advice>

<aop:config>
    <aop:pointcut expression="execution(* exam.service..*.*(..))" id="transaction"/>
    <aop:advisor advice-ref="txAdvice" pointcut-ref="transaction"/>
</aop:config>
```

aop:pointcut对应类AspectJExpressionPointcut; 

aop:advisor对应类DefaultBeanFactoryPointcutAdvisor;

aop:aspect对应类AspectJPointcutAdvisor; 

...

> aop:config的proxy-target-class属性表示是否为被代理对象生成CGLIB子类, 即主动使用cglib;
>
> expose-proxy属性表示是否将代理bean暴露给用户, 暴露代理对象可通过AopContext获得.

ConfigBeanDefinitionParser.parse:

1.定义一个CompositeComponentDefinition对象;

2.调用ConfigBeanDefinitionParser.configureAutoProxyCreator注册AspectJAwareAdvisorAutoProxyCreator, 用于生成代理子类:

  该类是SmartInstantiationAwareBeanPostProcessor的子类, 因此其入口在BeanPostProcessor的相关方法中(回顾getBean查看调用处):

①AbstractAutoProxyCreator.postProcessBeforeInstantiation:

```
1.isInfrastructureClass过滤基础设施类(用于实现aop内置的), shouldSkip跳过aop:aspect配置适用于当前bean的Advisor的Advice/Aspect对象,因为postProcessBeforeInstantiation方法会在每个bean初始化之前被调用，没有必要每次都进行基础类检测和跳过类检测，所以使用advisedBeans作为缓存用以提高性能。;
2.获取自定义TargetSource并进行代理子类创建(createProxy).
```

> 寻找Advisor通过findCandidateAdvisors委托给BeanFactoryAdvisorRetrievalHelper.findAdvisorBeans,首先是从容器中获取到所有的Advisor示例，然后调用isEligibleBean方法逐一判断Advisor是否适用于当前bean

②AbstractAutoProxyCreator.postProcessAfterInitialization:

```
调用wrapIfNecessary方法:
1.如果已进行过对自动以TargetSource生成代理子类,则直接返回,否则再进行一次同上过滤;
2.调用AbstractAdvisorAutoProxyCreator.getAdvicesAndAdvisorsForBean,里面继续调用AbstractAdvisorAutoProxyCreator.findEligibleAdvisors:
 2-1.调用findCandidateAdvisors寻找适用的Advisor(同上);
 2-2.调用findAdvisorsThatCanApply委托给 AopUtils.findAdvisorsThatCanApply,将IntroductionAdvisor放在Advisor链非IntroductionAdvisor的前面,并判断当前类和所有方法是否匹配Advisor的pointcut,如果类匹配但方法不全匹配, 便会用反射的方法获取targetClass(被检测类)的全部方法逐一交由Pointcut的MethodMatcher进行检测。
 2-3.调用AbstractAdvisorAutoProxyCreator.extendAdvisors允许子类向Advisor链表中添加自己的Advisor;
 2-4.调用AbstractAdvisorAutoProxyCreator.sortAdvisors对实现了Ordered接口的Advisor进行排序;
3.调用AbstractAutoProxyCreator.createProxy创建:
 在方法在创建一个ProxyFactory,执行ProxyFactory.getProxy,在方法中通过DefaultAopProxyFactory.createAopProxy根据proxy-target-classs以及是否实现了接口来决定jdk还是cglib,需要注意的是,它们都会对所有相关的Advisor进行链式调用.
 

```

## ScopedProxyBeanDefinitionDecorator

aop:scoped-proxy(与@ScopedProxy注解相同):

在默认情况下，如果一个singleton的bean中引用了一个prototype/session/request的bean，单例会永远持有一开始构造所赋给它的值. 

为了使每次调用这个Bean的时候都能够得到当前scope最新的值, 可给该引用的Bean添加scoped-proxy:

```
<bean id="userPreferences" class="com.foo.UserPreferences" scope="session">
    <aop:scoped-proxy/>
</bean>
<bean id="userManager" class="com.foo.UserManager">
    <property name="userPreferences" ref="userPreferences"/>
</bean>
```

由ScopedProxyBeanDefinitionDecorator完成解析, 入口处为DefaultBeanDefinitionDocumentReader.processBeanDefinition --> BeanDefinitionHolder.decorateBeanDefinitionIfRequired --> BeanDefinitionParserDelegate.decorateIfRequired --> NamespaceHandlerSupport.decorate --> ScopedProxyBeanDefinitionDecorator.decorate:

核心便是createScopedProxy方法，也是偷天换日: 创建一个新的BeanDefinition对象，beanName为被代理的bean的名字，被代理的bean名字为scopedTarget.原名字。被代理的bean扔将被注册到容器中。

新的BeanDefintion的beanClass为ScopedProxyFactoryBean

## AspectJAutoProxyBeanDefinitionParser

aop:aspectj-autoproxy用以开启对于@AspectJ注解风格AOP的支持. ???

# task

task相关解析器由AopNamespaceHandler注册:

```
@Override
public void init() {
    this.registerBeanDefinitionParser("annotation-driven", new AnnotationDrivenBeanDefinitionParser());
    this.registerBeanDefinitionParser("executor", new ExecutorBeanDefinitionParser());
    this.registerBeanDefinitionParser("scheduled-tasks", new ScheduledTasksBeanDefinitionParser());
    this.registerBeanDefinitionParser("scheduler", new SchedulerBeanDefinitionParser());
}
```

task:scheduler和task:scheduled-tasks用于定时任务调度;

task:execotor用于异步执行.

task:annotation-driven异步任务注解支持

**1.scheduled**

```xml
<task:scheduler id="scheduler" pool-size="3" />
<bean id="task" class="task.Task"/>
<task:scheduled-tasks scheduler="scheduler">
    <task:scheduled ref="task" method="print" cron="0/5 * * * * ?"/>
</task:scheduled-tasks>
```

SchedulerBeanDefinitionParser将task:scheduler解析为一个beanClass为ThreadPoolTaskScheduler的BeanDefinition;

ScheduledTasksBeanDefinitionParser将task:scheduled-tasks解析为一个beanClass为ContextLifecycleScheduledTaskRegistrar的BeanDefinition;

然后将**每一个**task:scheduled子标签解析为Task的子类, 如果属性为trigger, 子为TriggerTask; 如果属性为cron, 则为CronTask(TriggerTask子类); 如果属性为fixed-delay/fixed-rate, 则为IntervalTask;

ContextLifecycleScheduledTaskRegistrar的beanDefiniton中包含记录任务的cronTasksList、triggerTasksList、fixedDelayList、fixedRateList.

同一个Task可存在于多个list, 即任务类型属性标签可以重复.

执行入口处为ContextLifecycleScheduledTaskRegistrar.afterSingletonsInstantiated -->

ScheduledTaskRegistrar.scheduleTasks:

```
1)先对scheduler进行初始化, 如果没有配置task:scheduler, 则创建SingleThreadScheduledExecutor包装成ConcurrentTaskScheduler;
2)分类调用scheduledXXXTask调度不同类型的任务:
 在方法中最后会调用ConcurrentTaskScheduler.schedule --> ReschedulingRunnable.schedule具体执行.
```

**2.async**

需要使用到注解@Async, 指定executor的id

```
<task:executor id="executor" pool-size="3"/>
<task:annotation-driven executor="executor"/>
```

# transaction

```
<bean id="dataSource" class="org.springframework.jdbc.datasource.DriverManagerDataSource">
    <property name="driverClassName" value="${jdbc.driverClassName}" />
    <property name="url" value="${jdbc.url}" />
    <property name="username" value="${jdbc.username}" />
    <property name="password" value="${jdbc.password}" />
</bean>
<bean id="transactionManager" class="org.springframework.jdbc.datasource.DataSourceTransactionManager">
    <property name="dataSource" ref="dataSource"/>
</bean>
<tx:annotation-driven transaction-manager="transactionManager"/>
```

task相关解析器由AopNamespaceHandler注册:

```
@Override
public void init() {
    registerBeanDefinitionParser("advice", new TxAdviceBeanDefinitionParser());
    registerBeanDefinitionParser("annotation-driven", 
        new AnnotationDrivenBeanDefinitionParser());
    registerBeanDefinitionParser("jta-transaction-manager", new JtaTransactionManagerBeanDefinitionParser());
}
```

AnnotationDrivenBeanDefinitionParser(注意时spring-tx包下的).parse:

1.注册TransactionalEventListenerFactory, 允许自定义监听器监听事务的提交或其它动作;

2.调用AnnotationDrivenBeanDefinitionParser.AopAutoProxyConfigurer.configureAutoProxyCreator, 应该是AnnotationTransactionAttributeSource用于解析@Transactional注解属性, 然后应该与aop类似: 寻找Advisor生成代理类.(看不懂了哈哈!)

# mvc

![img](picture/de6d2b213f112297298f3e223bf08f28.png)



被`DispatcherServlet`检测的bean:

**1.HanderMapping**

两个主要的 HandlerMapping 实现是 RequestMappingHandlerMapping（支持@RequestMapping带注释的方法）和 SimpleUrlHandlerMapping（它维护 URI 路径模式到处理程序的显式注册）。

编程方式注册handler方法

```
@Configuration
public class MyConfig {

    @Autowired
    public void setHandlerMapping(RequestMappingHandlerMapping mapping, UserHandler handler) 
            throws NoSuchMethodException {

        RequestMappingInfo info = RequestMappingInfo
                .paths("/user/{id}").methods(RequestMethod.GET).build(); 

        Method method = UserHandler.class.getMethod("getUser", Long.class); 

        mapping.registerMapping(info, handler, method); 
    }
}
```

**2.HanderAdapter**

帮助 DispatcherServlet 调用映射到请求的处理程序，而不管该处理程序的实际调用方式如何。例如，调用带注解的控制器需要解析注解。HandlerAdapter 的主要目的是保护 DispatcherServlet 免受此类细节的影响。

**3.HandlerExceptionResolver**

解决异常的策略，可能将其映射到处理程序、HTML 错误视图或其他目标。

**4.ViewResolver**

Resolve logical `String`-based view names returned from a handler to an actual `View` with which to render to the response. 

5.LocalResolver

6.ThemeResolver

**7.MultipartResolver**

Abstraction for parsing a multi-part request (for example, browser form file upload) with the help of some multipart parsing library

8.FlashMapManager

## AntPathMatcher和PathPattern

修改默认策略

```
spring.mvc.pathmatch.matching-strategy=path-pattern-parser
```

https://blog.csdn.net/f641385712/article/details/118031407

## URL中的Ant匹配

Ant风格在mvc和security常用, 是一种路径匹配表达式。主要用来对 uri 的匹配。其实跟正则表达式作用是一样的，只不过正则表达式适用面更加宽泛， Ant 仅仅用于路径匹配

**通配符**

```
Ant 中的通配符有三种：
? 匹配任何单字符
* 匹配0或者任意数量的 字符
** 匹配0或者更多的 目录
这里注意了单个 * 是在一个目录内进行匹配。 而 ** 是可以匹配多个目录，一定不要迷糊。
```

**最长匹配原则**

一旦一个 uri 同时符合两个 Ant 匹配那么走匹配规则字符最多的。为什么走最长？因为 字符越长信息越多就越具体。比如 `/ant/a/path `同时满足` /**/path` 和` /ant/*/path` 那么走` /ant/*/path`

**在mvc和security中的使用**

```
// mvc
@GetMapping("/?ant")

// security
//... 
.antMatchers("/index.html","/static/**").permitAll()
```

