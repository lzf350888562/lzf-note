# 集合

## List

**ArrarList**

https://javaguide.cn/java/collection/arraylist-source-code/

```
toArray 返回的是新数组,通过Arrays.copyOf方法生成;
大量调用native方法System.arraycopy,如指定插入位置的add方法,remove方法,Arrays.copyOf方法其实也是调用了arraycopy方法.类似C语言操作数组;
ensureCapacity方法在ArrayList内部没有被调用过,是给用户使用的,最好在 add 大量元素之前用 ensureCapacity 方法，以减少增量重新分配的次数

```



## Map

**HashMap**

https://javaguide.cn/java/collection/hashmap-source-code/

 JDK1.8 以后的 `HashMap` 在解决哈希冲突时有了较大的变化，当链表长度大于阈值（默认为 8,这个阈值为表示链表或红黑树大小的阈值,是常量）（将链表转换成红黑树前会判断，如果当前数组的长度小于 64，那么会选择先进行数组扩容，而不是转换为红黑树）时，将链表转化为红黑树，以减少搜索时间.

| 名称            | 用途                                                         |
| --------------- | ------------------------------------------------------------ |
| initialCapacity | HashMap 初始容量                                             |
| loadFactor      | 负载因子,控制数组存放数据的疏密程度                          |
| threshold       | 当前 HashMap 所能容纳键值对数量的最大值，超过这个值，则需扩容 |

默认情况下，HashMap 初始容量是16，负载因子为 0.75.

`threshold = capacity * loadFactor`

> `HashMap` 可以存储 null 的 key 和 value，但 null 作为键只能有一个，null 作为值可以有多个



**LinkedHashMap**



**CucurrentHashMap**

https://javaguide.cn/java/collection/concurrent-hash-map-source-code/

1.7以前为分段锁,1.8采用CAS和synchronized,只锁定当前链表或红黑二叉树的首节点，这样只要 hash 不冲突，就不会产生并发，效率又提升 N 倍。



## BitSet(BitMap)

BitSet是JAVA中对BitMap的实现

**BitMap**，即位图，使用每个位表示某种状态，适合处理整型的海量数据。本质上是哈希表的一种应用实现，原理也很简单，给定一个int整型数据，将该int整数映射到对应的位上，并将该位由0改为1。

BitMap通常用来去重 & 取两个集合的交集或并集等.

比如, 对最大值43亿左右的所有QQ号去重, 即可采用BitMap:

一个char(1B)类型的数据可以标识0-7这8个整数是否存在与否;

一个int(4B)或四个char类型的数据可以标识0-31这32个整数的存在与否;

...

要对43亿QQ号进行去重, 需要总共至少43位的多个结构类型标识每个QQ号:
4300000000/8/1024/1024≈ 512MB .  即该大小足够标识所有QQ号存在与否.

# IO

## io模型

为了保证操作系统的稳定性和安全性，一个进程的地址空间划分为 **用户空间（User space）** 和 **内核空间（Kernel space ）** 。

**用户空间的程序不能直接访问内核空间**。

用户进程想要执行 IO 操作的话，必须通过 **系统调用** 来间接访问内核空间.(常有磁盘io和网络io)

从应用程序的视角来看的话，我们的应用程序对操作系统的内核发起 IO 调用（系统调用），操作系统负责的内核执行具体的 IO 操作。也就是说，我们的应用程序实际上只是发起了 IO 操作的调用而已，具体 IO 的执行是由操作系统的内核来完成的。

> 当应用程序发起 I/O 调用后，会经历两个步骤：
>
> 1. 内核等待 I/O 设备准备好数据
> 2. 内核将数据从内核空间拷贝到用户空间。

### bio

![](picture/6a9e704af49b4380bb686f0c96d33b81tplv-k3u1fbpfcp-watermark.image)

### nio

Java 中的 NIO 是 **I/O 多路复用模型** 还是 同步非阻塞 IO 模型。

**同步非阻塞 IO 模型**

![图源：《深入拆解Tomcat & Jetty》](picture/bb174e22dbe04bb79fe3fc126aed0c61tplv-k3u1fbpfcp-watermark.image)

同步非阻塞 IO 模型中，应用程序会一直发起 read 调用，等待数据从内核空间拷贝到用户空间的这段时间里，线程依然是阻塞的，直到在内核把数据拷贝到用户空间。

相比于同步阻塞 IO 模型，同步非阻塞 IO 模型确实有了很大改进。通过轮询操作，避免了一直阻塞。

但是，这种 IO 模型同样存在问题：**应用程序不断进行 I/O 系统调用轮询数据是否已经准备好的过程是十分消耗 CPU 资源的。**

这个时候，**I/O 多路复用模型** 就上场了。

![img](picture/88ff862764024c3b8567367df11df6abtplv-k3u1fbpfcp-watermark.image)

IO 多路复用模型中，线程首先发起 select 调用，询问内核数据是否准备就绪，等内核把数据准备好了，用户线程再发起 read 调用。read 调用的过程（数据从内核空间->用户空间）还是阻塞的。

**IO 多路复用模型，通过减少无效的系统调用，减少了对 CPU 资源的消耗。**

Java 中的 NIO ，有一个非常重要的**选择器 ( Selector )** 的概念，也可以被称为 **多路复用器**。通过它，只需要一个线程便可以管理多个客户端连接。当客户端数据到了之后，才会为其服务。

![img](picture/0f483f2437ce4ecdb180134270a00144tplv-k3u1fbpfcp-watermark.image)

### aio

AIO 也就是 NIO 2。Java 7 中引入了 NIO 的改进版 NIO 2,它是异步 IO 模型。

异步 IO 是基于事件和回调机制实现的，也就是应用操作之后会直接返回，不会堵塞在那里，当后台处理完成，操作系统会通知相应的线程进行后续的操作。

![img](picture/3077e72a1af049559e81d18205b56fd7tplv-k3u1fbpfcp-watermark.image)

# 设计模式

## 代理模式

**静态代理**

```
public class SmsServiceImpl implements SmsService {
    public String send(String message) {
        System.out.println("send message:" + message);
        return message;
    }
}

public class SmsProxy implements SmsService {
    private final SmsService smsService;

    public SmsProxy(SmsService smsService) {
        this.smsService = smsService;
    }

    @Override
    public String send(String message) {
        //调用方法之前，我们可以添加自己的操作
        System.out.println("before method send()");
        smsService.send(message);
        //调用方法之后，我们同样可以添加自己的操作
        System.out.println("after method send()");
        return null;
    }
}

public class Main {
    public static void main(String[] args) {
        SmsService smsService = new SmsServiceImpl();
        SmsProxy smsProxy = new SmsProxy(smsService);
        smsProxy.send("java");
    }
}
```

**Proxy**

**JDK 动态代理最致命的问题是其只能代理实现了接口的类。**

```
public class DebugInvocationHandler implements InvocationHandler {
    private final Object target;

    public DebugInvocationHandler(Object target) {
        this.target = target;
    }

    public Object invoke(Object proxy, Method method, Object[] args) throws InvocationTargetException, IllegalAccessException {
        //调用方法之前，我们可以添加自己的操作
        System.out.println("before method " + method.getName());
        Object result = method.invoke(target, args);
        //调用方法之后，我们同样可以添加自己的操作
        System.out.println("after method " + method.getName());
        return result;
    }
}
```

```
public class JdkProxyFactory {
    public static Object getProxy(Object target) {
        return Proxy.newProxyInstance(
                target.getClass().getClassLoader(), // 目标类的类加载
                target.getClass().getInterfaces(),  // 代理需要实现的接口，可指定多个
                new DebugInvocationHandler(target)   // 代理对象对应的自定义 InvocationHandler
        );
    }
}
```

```
SmsService smsService = (SmsService) JdkProxyFactory.getProxy(new SmsServiceImpl());
smsService.send("java");
```

jdk动态代理也可以不使用被代理对象, 直接对接口进行代理, 比如mybatis的dao接口

```
public class OnlyInterfaceProxyTest {
    public static void main(String[] args) {
        MyMapper mapper = (MyMapper)Proxy.newProxyInstance(MyMapper.class.getClassLoader(), new Class[]{MyMapper.class}, new InvocationHandler() {
            @Override
            public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
                if (method.getName().equals("getCount")) {
                    return 100;
                } else {
                    return Arrays.asList("100","200");
                }
            }
        });
        System.out.println(mapper.getCount());
        System.out.println(mapper.getNames());
    }
}
interface MyMapper{
    int getCount();
    List<String> getNames();
}
```

**CGLIB**

```
public class AliSmsService {
    public String send(String message) {
        System.out.println("send message:" + message);
        return message;
    }
}

public class DebugMethodInterceptor implements MethodInterceptor {
    /**
     * @param o           代理对象（增强的对象）
     * @param method      被拦截的方法（需要增强的方法）
     * @param args        方法入参
     * @param methodProxy 用于调用原始方法
     */
    @Override
    public Object intercept(Object o, Method method, Object[] args, MethodProxy methodProxy) throws Throwable {
        //调用方法之前，我们可以添加自己的操作
        System.out.println("before method " + method.getName());
        Object object = methodProxy.invokeSuper(o, args);
        //调用方法之后，我们同样可以添加自己的操作
        System.out.println("after method " + method.getName());
        return object;
    }
}

public class CglibProxyFactory {
    public static Object getProxy(Class<?> clazz) {
        // 创建动态代理增强类
        Enhancer enhancer = new Enhancer();
        // 设置类加载器
        enhancer.setClassLoader(clazz.getClassLoader());
        // 设置被代理类
        enhancer.setSuperclass(clazz);
        // 设置方法拦截器
        enhancer.setCallback(new DebugMethodInterceptor());
        // 创建代理类
        return enhancer.create();
    }
}

AliSmsService aliSmsService = (AliSmsService) CglibProxyFactory.getProxy(AliSmsService.class);
aliSmsService.send("java");
```

 **CGLIB 动态代理是通过生成一个被代理类的子类来拦截被代理类的方法调**用，因此不能代理声明为 final 类型的类和方法。

就二者的效率来说，大部分情况都是 JDK 动态代理更优秀，随着 JDK 版本的升级，这个优势更加明显。

## 单例模式

**懒汉式**

线程安全与线程不安全区别于synchironized锁 , 加锁导致很大的性能开销，并且加锁其实只需要在第一次初始化的时候用到，之后的调用都没必要再进行加锁。

```
public class Singleton {  
    private static Singleton instance;  
    private Singleton (){}  
    public static (synchronized) Singleton getInstance() {  
    if (instance == null) {  
        instance = new Singleton();  
    }  
    return instance;  
    }  
}
```

**饿汉式**

```
public class Singleton {  
    private static Singleton instance = new Singleton();  
    private Singleton (){}  
    public static Singleton getInstance() {  
    return instance;  
    }  
}
```

**双检索**

对饿汉式的优化:先判断对象是否已经被初始化，再决定要不要加锁。

```
public class Singleton {  
    private volatile static Singleton singleton;  
    private Singleton (){}  
    public static Singleton getSingleton() {  
    if (singleton == null) {  
        synchronized (Singleton.class) {  
        if (singleton == null) {  
            singleton = new Singleton();  
        }  
        }  
    }  
    return singleton;  
    }  
}
```

>  执行双重检查是因为，如果多个线程同时了通过了第一次判空检查，并且其中一个线程首先通过了第二次检查并实例化了对象，那么剩余通过了第一次检查的线程就不会再去实例化对象。

**静态内部类**

只有显式调用 getInstance 方法时，才会显式装载 SingletonHolder 类，从而实例化 instance

```
public class Singleton {  
    private static class SingletonHolder {  
    private static final Singleton INSTANCE = new Singleton();  
    }  
    private Singleton (){}  
    public static final Singleton getInstance() {  
    return SingletonHolder.INSTANCE;  
    }  
}
```

**枚举**

结合了以上所有方式的优点 ,并且防止反序列化生成对象.

```
public enum Singleton {  
    INSTANCE;  
    public void whateverMethod() {  
    }  
}
```

### 为什么双检锁需要volatile

问题出在` singleton = new Singleton(); `这一行, 其可以分为三个步骤:

1. 分配内存空间
2. 初始化对象
3. 将对象指向刚分配的内存空间

但是有些编译器为了性能的原因，可能会将第二步和第三步进行**重排序**，顺序就成了：

1. 分配内存空间
2. 将对象指向刚分配的内存空间
3. 初始化对象

重排序后，两个线程发生了以下调用：

| Time | Thread A                  | Thread B                                  |
| :--- | :------------------------ | :---------------------------------------- |
| T1   | 检查到`singleton`为空     |                                           |
| T2   | 获取锁                    |                                           |
| T3   | 再次检查到`singleton`为空 |                                           |
| T4   | 为`singleton`分配内存空间 |                                           |
| T5   | 将`singleton`指向内存空间 |                                           |
| T6   |                           | 检查到`singleton`不为空                   |
| T7   |                           | 访问`singleton`（此时对象还未完成初始化） |
| T8   | 初始化`singleton`         |                                           |

在这种情况下，T7时刻线程B对`uniqueSingleton`的访问，访问的是一个**初始化未完成**的对象。