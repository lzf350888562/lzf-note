# 集合

## ArrarList

```
toArray 返回的是新数组,通过Arrays.copyOf方法生成;
大量调用native方法System.arraycopy,如指定插入位置的add方法,remove方法,Arrays.copyOf方法其实也是调用了arraycopy方法.类似C语言操作数组;
ensureCapacity方法在ArrayList内部没有被调用过,是给用户使用的,最好在 add 大量元素之前用 ensureCapacity 方法，以减少增量重新分配的次数
```

## HashMap

 JDK1.8 以后的 `HashMap` 在解决哈希冲突时有了较大的变化，当链表长度大于阈值（默认为 8,这个阈值为表示链表或红黑树大小的阈值,是常量）（将链表转换成红黑树前会判断，如果当前数组的长度小于 64，那么会选择先进行数组扩容，而不是转换为红黑树）时，将链表转化为红黑树，以减少搜索时间.

| 名称            | 用途                                                         |
| --------------- | ------------------------------------------------------------ |
| initialCapacity | HashMap 初始容量                                             |
| loadFactor      | 负载因子,控制数组存放数据的疏密程度                          |
| threshold       | 当前 HashMap 所能容纳键值对数量的最大值，超过这个值，则需扩容 |

默认情况下，HashMap 初始容量是16，负载因子为 0.75.

`threshold = capacity * loadFactor`

> `HashMap` 可以存储 null 的 key 和 value，但 null 作为键只能有一个，null 作为值可以有多个

**自动扩容**

扩容发生在putVal方法的最后，即写入元素之后才会判断是否需要扩容操作，当自增后的size大于之前所计算好的阈值threshold，即执行resize操作。

在resize中, 通过位运算<<1进行容量扩充，即扩容1倍，同时新的阈值newThr也扩容为旧阈值的1倍。

扩容时，总共存在三种情况：

- 哈希桶数组中某个位置只有1个元素，即不存在哈希冲突时，则直接将该元素copy至新哈希桶数组的对应位置即可。

- 哈希桶数组中某个位置的节点为树节点时，则执行红黑树的扩容操作。

- 哈希桶数组中某个位置的节点为普通节点时，则执行链表扩容操作，在JDK1.8中，为了避免之前版本中并发扩容所导致的死链问题，引入了高低位链表辅助进行扩容操作。

**初始化与懒加载**

初始化的时候只会设置默认的负载因子，并不会进行其他初始化的操作，在首次使用的时候才会进行初始化。

当new一个新的HashMap的时候，不会立即对哈希数组进行初始化，而是在首次put元素的时候，通过resize()方法进行初始化。

resize()中会设置默认的初始化容量DEFAULT_INITIAL_CAPACITY为16，扩容的阈值为0.75*16 = 12，即哈希桶数组中元素达到12个便进行扩容操作。

最后创建容量为16的Node数组，并赋值给成员变量哈希桶table，即完成了HashMap的初始化操作。

**哈希计算**

哈希表以哈希命名，足以说明哈希计算在该数据结构中的重要程度。而在实现中，JDK并没有直接使用Object的native方法返回的hashCode作为最终的哈希值，而是进行了二次加工。

该二次加工hash与ConcurrentHashMap有一点不同,  核心的计算逻辑相同，都是使用key对应的hashCode与其hashCode右移16位的结果进行异或操作。此处，将高16位与低16位进行异或的操作称之为扰动函数，目的是将高位的特征融入到低位之中，降低哈希冲突的概率。

举个例子来理解下扰动函数的作用：

```
hashCode(key1) = 0000 0000 0000 1111 0000 0000 0000 0010
hashCode(key2) = 0000 0000 0000 0000 0000 0000 0000 0010
```

若HashMap容量为4，在不使用扰动函数的情况下，key1与key2的hashCode注定会冲突（后两位相同，均为01）。

经过扰动函数处理后，可见key1与key2 hashcode的后两位不同，上述的哈希冲突也就避免了。

```
hashCode(key1) ^ (hashCode(key1) >>> 16)
0000 0000 0000 1111 0000 0000 0000 1101

hashCode(key2) ^ (hashCode(key2) >>> 16)
0000 0000 0000 0000 0000 0000 0000 0010
```

这种增益会随着HashMap容量的减少而增加。《An introduction to optimising a hashing strategy》文章中随机选取了哈希值不同的352个字符串，当HashMap的容量为2^9时，使用扰动函数可以减少10%的碰撞，可见扰动函数的必要性。

此外，ConcurrentHashMap中经过扰乱函数处理之后，需要与HASH_BITS做与运算，HASH_BITS为0x7ffffff，即只有最高位为0，这样运算的结果使hashCode永远为正数。在ConcurrentHashMap中，预定义了几个特殊节点的hashCode，如：MOVED、TREEBIN、RESERVED，它们的hashCode均定义为负值。因此，将普通节点的hashCode限定为正数，也就是为了防止与这些特殊节点的hashCode产生冲突。

> 通过哈希运算，可以将不同的输入值映射到指定的区间范围内，随之而来的是**哈希冲突**问题。考虑一个极端的case，假设所有的输入元素经过哈希运算之后，都映射到同一个哈希桶中，那么查询的复杂度将不再是O(1)，而是O(n)，相当于线性表的顺序遍历。因此，哈希冲突是影响哈希计算性能的重要因素之一。哈希冲突如何解决呢？主要从两个方面考虑，一方面是避免冲突，另一方面是在冲突时合理地解决冲突，尽可能提高查询效率。前者在上面的章节中已经进行介绍，即通过扰动函数来增加hashCode的随机性，避免冲突。针对后者，HashMap中给出了两种方案：拉链表与红黑树。

①**拉链表**

在JDK1.8之前，HashMap中是采用拉链表的方法来解决冲突，即当计算出的hashCode对应的桶上已经存在元素，但两者key不同时，会基于桶中已存在的元素拉出一条链表，将新元素链到已存在元素的前面。当查询存在冲突的哈希桶时，会顺序遍历冲突链上的元素。同一key的判断逻辑如下，先判断hash值是否相同，再比较key的地址或值是否相同。

```
if(p.hash == hash &&
	((k = p.key) == key || (key != null && key.equals(k))))
```

死链条:在JDK1.8之前，HashMap在并发场景下扩容时存在一个bug，形成死链，导致get该位置元素的时候，会死循环，使CPU利用率高居不下。这也说明了HashMap不适于用在高并发的场景，高并发应该优先考虑JUC中的ConcurrentHashMap。然而，精益求精的JDK开发者们并没有选择绕过问题，而是选择直面问题并解决它。在JDK1.8之中，引入了高低位链表（双端链表）。

什么是高低位链表呢？在扩容时，哈希桶数组buckets会扩容一倍，以容量为8的HashMap为例，原有容量8扩容至16，将[0, 7]称为低位，[8, 15]称为高位，低位对应loHead、loTail，高位对应hiHead、hiTail。

扩容时会依次遍历旧buckets数组的每一个位置上面的元素：

- 若不存在冲突，则重新进行hash取模，并copy到新buckets数组中的对应位置。
- 若存在冲突元素，则采用高低位链表进行处理。通过e.hash & oldCap来判断取模后是落在高位还是低位。举个例子：假设当前元素hashCode为0001（忽略高位），其运算结果等于0，说明扩容后结果不变，取模后还是落在低位[0, 7]，即0001 & 1000 = 0000，还是原位置，再用低位链表将这类的元素链接起来。假设当前元素的hashCode为1001， 其运算结果不为0，即1001 & 1000 = 1000 ，扩容后会落在高位，新的位置刚好是旧数组索引（1） + 旧数据长度（8） = 9，再用高位链表将这些元素链接起来。最后，将高低位链表的头节点分别放在扩容后数组newTab的指定位置上，即完成了扩容操作。这种实现降低了对共享资源newTab的访问频次，先组织冲突节点，最后再放入newTab的指定位置。避免了JDK1.8之前每遍历一个元素就放入newTab中，从而导致并发扩容下的死链问题。

![](picture/高低链表.jpg)

②**红黑树**

在JDK1.8之中，HashMap引入了红黑树来处理哈希冲突问题，而不再是拉链表。那么为什么要引入红黑树来替代链表呢？虽然链表的插入性能是O(1)，但查询性能确是O(n)，当哈希冲突元素非常多时，这种查询性能是难以接受的。因此，在JDK1.8中，如果冲突链上的元素数量大于8，并且哈希桶数组的长度大于64时，会使用红黑树代替链表来解决哈希冲突，此时的节点会被封装成TreeNode而不再是Node（TreeNode其实继承了Node，以利用多态特性），使查询具备O(logn)的性能。

这里简单地回顾一下红黑树，它是一种平衡的二叉树搜索树，类似地还有AVL树。两者核心的区别是AVL树追求“绝对平衡”，在插入、删除节点时，成本要高于红黑树，但也因此拥有了更好的查询性能，适用于读多写少的场景。然而，对于HashMap而言，读写操作其实难分伯仲，因此选择红黑树也算是在读写性能上的一种折中。

**位运算相关**

①**为什么哈希桶数组的大小为大于等于给定值的最小2的整数次幂?**

tableSizeFor根据输入容量大小cap来计算最终哈希桶数组的容量大小，找到大于等于给定值cap的最小2的整数次幂。

```
static final int tableSizeFor(int cap){
	int n = cap -1;
	n |= n >>> 1;
	n |= n >>> 2;
	n |= n >>> 4;
	n |= n >>> 8;
	n |= n >>> 16;
	return (n < 0) ? 1 : (n >= MAXIMUM_CAPACITY) ? MAXIMUM_CAPACITY : n + 1;
}
```

> 整个过程是找到cap对应二进制中最高位的1，然后每次以2倍的步长（依次移位1、2、4、8、16）复制最高位1到后面的所有低位，把最高位1后面的所有位全部置为1，最后进行+1，即完成了进位。

为了提高计算与存储效率，使每个元素对应hash值能够准确落入哈希桶数组给定的范围区间内。确定数组下标采用的算法是 hash & (n - 1)，n即为哈希桶数组的大小。由于其总是2的整数次幂，这意味着n-1的二进制形式永远都是0000111111的形式，即从最低位开始，连续出现多个1，该二进制与任何值进行&运算都会使该值映射到指定区间[0, n-1]。比如：当n=8时，n-1对应的二进制为0111，任何与0111进行&运算都会落入[0,7]的范围内，即落入给定的8个哈希桶中，存储空间利用率100%。举个反例，当n=7，n-1对应的二进制为0110，任何与0110进行&运算会落入到第0、6、4、2个哈希桶，而不是[0,6]的区间范围内，少了1、3、5三个哈希桶，这导致存储空间利用率只有不到60%，同时也增加了哈希碰撞的几率。

②**ASHIFT偏移量计算**

ASHIFT用于获取给定值的最高有效位数（移位除了能够进行乘除运算，还能用于保留高、低位操作，右移保留高位，左移保留低位）。

ConcurrentHashMap中的ABASE+ASHIFT是用来计算哈希数组中某个元素在实际内存中的初始位置，ASHIFT采取的计算方式是31与scale前导0的数量做差，也就是scale的实际位数-1。scale就是哈希桶数组Node[]中每个元素的大小，通过((long)i << ASHIFT) + ABASE)进行计算，便可得到数组中第i个元素的起始内存地址。

前导0的数量是通过Integer的静态方法numberOfLeadingZeros计算出来的.

假设 i = 0000 0000 0000 0100 0000 0000 0000 0000，n = 1

```
i >>> 16  0000 0000 0000 0000 0000 0000 0000 0100   不为0
i >>> 24  0000 0000 0000 0000 0000 0000 0000 0000   等于0
```

右移了24位等于0，说明24位到31位之间肯定全为0，即n = 1 + 8 = 9，由于高8位全为0，并且已经将信息记录至n中，因此可以舍弃高8位，即 i <<= 8。此时，

```
i = 0000 0100 0000 0000 0000 0000 0000 0000
```

类似地，i >>> 28 也等于0，说明28位到31位全为0，n = 9 + 4 = 13，舍弃高4位。此时，

```
i = 0100 0000 0000 0000 0000 0000 0000 0000
```

继续运算，

```
i >>> 30  0000 0000 0000 0000 0000 0000 0000 0001   不为0
i >>> 31  0000 0000 0000 0000 0000 0000 0000 0000   等于0
```

最终可得出n = 13，即有13个前导0。n -= i  >>> 31是检查最高位31位是否是1，因为n初始化为1，如果最高位是1，则不存在前置0，即n = n - 1 = 0。

总结一下，以上的操作其实是基于二分法的思想来定位二进制中1的最高位，先看高16位，若为0，说明1存在于低16位；反之存在高16位。由此将搜索范围由32位（确切的说是31位）减少至16位，进而再一分为二，校验高8位与低8位，以此类推。

计算过程中校验的位数依次为16、8、4、2、1，加起来刚好为31。为什么是31不是32呢？因为前置0的数量为32的情况下i只能为0，在前面的if条件中已经进行过滤。这样一来，非0值的情况下，前置0只能出现在高31位，因此只需要校验高31位即可。最终，用总位数减去计算出来的前导0的数量，即可得出二进制的最高有效位数。代码中使用的是31 - Integer.numberOfLeadingZeros(scale)，而不是总位数32，这是为了能够得到哈希桶数组中第i个元素的起始内存地址，方便进行CAS等操作。

## LinkedHashMap

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

**io模型**

为了保证操作系统的稳定性和安全性，一个进程的地址空间划分为 **用户空间（User space）** 和 **内核空间（Kernel space ）** 。

**用户空间的程序不能直接访问内核空间**。

用户进程想要执行 IO 操作的话，必须通过 **系统调用** 来间接访问内核空间.(常有磁盘io和网络io)

我们的应用程序实际上只是发起了 IO 操作的调用而已，具体 IO 的执行是由操作系统的内核来完成的。

> 当应用程序发起 I/O 调用后，会经历两个步骤：
>
> 1. 内核等待 I/O 设备准备好数据
> 2. 内核将数据从内核空间拷贝到用户空间。

**bio**

BIO是同步阻塞模型，一个客户端连接对应一个处理线程。在BIO中，accept和read方法都是阻塞操作，如果没有连接请求，accept方法阻塞；如果无数据可读取，read方法阻塞。

![](picture/6a9e704af49b4380bb686f0c96d33b81tplv-k3u1fbpfcp-watermark.image)

**nio**

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

客户端发送的连接请求注册在多路复用器Selector上，服务端线程通过轮询多路复用器查看是否有IO请求，有则进行处理。

![img](picture/0f483f2437ce4ecdb180134270a00144tplv-k3u1fbpfcp-watermark.image)

> Epoll是Linux下多路复用IO接口select/poll的增强版本，它能显著提高程序在大量并发连接中只有少量活跃的情况下的系统CPU利用率，获取事件的时候，它无须遍历整个被侦听的描述符集，只要遍历那些被内核IO事件异步唤醒而加入Ready队列的描述符集合就行了。

**aio**

AIO 也就是 NIO 2。Java 7 中引入了 NIO 的改进版 NIO 2,它是异步 IO 模型。

异步 IO 是基于事件和回调机制实现的，也就是应用操作之后会直接返回，不会堵塞在那里，当后台处理完成，操作系统会通知相应的线程进行后续的操作。

![img](picture/3077e72a1af049559e81d18205b56fd7tplv-k3u1fbpfcp-watermark.image)

## 使用缓冲流减少IO

如果使用普通的FileInputStream/FileOutputStream实现文件读写:

```
long begin = System.currentTimeMillis();
        try (FileInputStream input = new FileInputStream("C:/456.png");
             FileOutputStream output = new FileOutputStream("C:/789.png")) {
            byte[] bytes = new byte[1024];
            int i;
            while ((i = input.read(bytes)) != -1) {
                output.write(bytes,0,i);
            }
        } catch (IOException e) {
            log.error("复制文件发生异常",e);
        }
        log.info("常规流读写，总共耗时ms："+(System.currentTimeMillis() - begin));
```

如果是不带缓冲的流，读取到一个字节或者字符的，就会直接输出数据了。而带缓冲的流，读取到一个字节或者字符时，先不输出，而是等达到缓冲区的最大容量，才一次性输出: 

```
long begin = System.currentTimeMillis();
        try (BufferedInputStream bufferedInputStream = new BufferedInputStream(new FileInputStream("C:/456.png"));
        BufferedOutputStream  bufferedOutputStream = new BufferedOutputStream(new FileOutputStream("C:/789.png"))) {
            byte[] bytes = new byte[1024];
            int i;
            while ((i = input.read(bytes)) != -1) {
                output.write(bytes,0,i);
            }
        } catch (IOException e) {
            log.error("复制文件发生异常",e);
        }
        log.info("总共耗时ms"+(System.currentTimeMillis() - begin));
```



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

## 工厂模式

角色

- 抽象产品角色(都有)
- 具体产品角色(都有)
- 抽象工厂角色(工厂方法模式和抽象工厂模式有)
- 具体工厂角色(都有)
- 上下文角色(都有)——调用工厂获取对象

**简单工厂模式**

抽象产品

```
public interface Cpu {
    void calculate();
}
```

具体产品

```
public class ACpu implements Cpu {
    @Override
    public void calculate() {
        System.out.println("this is A cpu");
    }
}

public class BCpu implements Cpu {
    @Override
    public void calculate() {
        System.out.println("this is B cpu");
    }
}
```

具体工厂

> 创建哪种具体产品类型代码在工厂内部逻辑，如果需要新增具体产品，需要修改工厂类方法, 不符合开-闭原则    

```
public class CpuFactory {
    public static Cpu createCpu(Class classType) {
        if (classType.getName().equals(ACpu.class.getName())) {
            return new ACpu();
        } else if (classType.getName().equals(BCpu.class.getName())) {
            return new BCpu();
        }
        return null;
    }
}
```

**工厂方法模式**

> 具体的工厂负责创建具体的产品，工厂内部的逻辑只负责创建对应对象，决定生成什么产品的逻辑在外部客户端，客户端通过选择使用具体工厂，间接决定了生成什么产品。

抽象产品

```
public interface Cpu {
    void calculate();
}
```

具体产品

```
public class ACpu implements Cpu {
    @Override
    public void calculate() {
        System.out.println("this is A cpu");
    }
}

public class BCpu implements Cpu {
    @Override
    public void calculate() {
        System.out.println("this is B cpu");
    }
}
```

抽象工厂

```
public interface CpuFactory {
    Cpu createCpu();
}
```

具体工厂

```
public class ACpuFactory implements CpuFactory{
    @Override
    public  Cpu createCpu() {
        return new ACpu();
    }
}

public class BCpuFactory implements CpuFactory {
    @Override
    public Cpu createCpu() {
        return new BCpu();
    }
}
```

常见的数据库连接工厂，SqlSessionFactory，抽象产品是一个数据库连接，具体产品至于是oracle提供的，还是mysql提供的，我并不需要关心，因为都能让我通过sql来操作数据。

### 抽象工厂模式

抽象工厂模式下，以产品族维度来建厂，一个厂里有多个创建方法，每个创建方法负责创建一个产品线(产品族)

抽象产品

```
public interface Cpu {
    void calculate();
}

public interface Mainboard {
    void installCpu();
}
```

具体产品

```
public class ACpu implements Cpu {
    @Override
    public void calculate() {
        System.out.println("this is A cpu");
    }
}

public class BCpu implements Cpu {
    @Override
    public void calculate() {
        System.out.println("this is B cpu");
    }
}

public class AMainboard implements Mainboard {
    @Override
    public void installCpu() {
        System.out.println("this is A mainboard");
    }
}

public class BMainboard implements Mainboard {
    @Override
    public void installCpu() {
        System.out.println("this is B mainboard");
    }
}
```

抽象工厂

```
public interface AbatractFactory {
    Cpu createCpu();
    Mainboard createMainboard();
}
```

具体工厂

```
public class AFactory implements AbatractFactory {
    @Override
    public Cpu createCpu() {
        return new ACpu();
    }
    @Override
    public Mainboard createMainboard() {
        return new AMainboard();
    }
}

public class BFactory implements AbatractFactory {
    @Override
    public Cpu createCpu() {
        return new BCpu();
    }
    @Override
    public Mainboard createMainboard() {
        return new BMainboard();
    }
}
```

> 抽象工厂模式分离了接口和实现 , 并使得使切换产品族变得容易.
>
> 但缺点是不太容易扩展新的产品:  每给产品族添加新产品时，就要在抽象工厂中添加新产品创建方法，同时要给所有的具体工厂增加接口。

## 装饰模式

![](picture/decorator01.jpg)

角色：

- Component：抽象构件
- ConcreteComponent：具体构件
- Decorator：抽象装饰类
- ConcreteDecorator：具体装饰类

> - 优点：比继承更加灵活（继承是耦合度很大的静态关系），可以动态的为对象增加职责，可以通过使用不同的装饰器组合为对象扩展N个新功能，而不会影响到对象本身。
>
> - 缺点：当一个对象的装饰器过多时，会产生很多的装饰类小对象和装饰组合策略，增加系统复杂度，增加代码的阅读理解成本。

> 适用场景:
>
> - 适合需要通过配置（如：diamond）来动态增减对象功能的场景。
>
> - 适合一个对象需要N种功能排列组合的场景（如果用继承，会使子类数量爆炸式增长）, 如InputStream.

> 注意: 一个装饰类的接口必须与被装饰类的接口保持相同，对于客户端来说无论是装饰之前的对象还是装饰之后的对象都可以一致对待。

抽象构件:

```
interface  Component{
    public void operation();
}
```

具体构件:

```
class ConcreteComponent implements Component{
    public ConcreteComponent(){
        System.out.println("创建具体构件角色");       
    }   
    public void operation(){
        System.out.println("调用具体构件角色的方法operation()");           
    }
}
```

抽象装饰(为抽象类, 需要通过构造函数传入被装饰类对象)

```
class Decorator implements Component{
    private Component component;   
    public Decorator(Component component){
        this.component=component;
    }   
    public void operation(){
        component.operation();
    }
}
```

具体装饰

```
class ConcreteDecorator extends Decorator{
    public ConcreteDecorator(Component component){
        super(component);
    }   
    public void operation(){
        super.operation();
        addBehavior();
    }
    public void addBehavior(){
        System.out.println("为具体构件角色增加额外的功能addBehavior()");           
    }
}
```

使用

```
public static void main(String[] args){
    Component component = new ConcreteComponent();
    component.operation();
    System.out.println("---------------------------------");
    Component decorator = new ConcreteDecorator(component);
    decorator.operation();
}
```

> 个人觉得与JDK动态代理类似.
>
> 区别: 代理模式的访问控制主要在于对目标可的透明访问, 而装饰模式由客户端对目标类对象进行增强.

## 策略模式

![](picture/stragegy01.jpg)

角色:

- Context: 环境类
- Strategy: 抽象策略类
- ConcreteStrategy: 具体策略类

> - 优点：策略模式提供了对“开闭原则”的完美支持，用户可以在不修改原有系统的基础上选择算法或行为。干掉复杂难看的if-else。
>
> - 缺点：调用时，必须提前知道都有哪些策略模式类，才能自行决定当前场景该使用何种策略。

> 适用场景: 一个系统需要动态地在几种可替换算法中选择一种。不希望使用者关心算法细节，将具体算法封装进策略类中。

抽象策略

```
interface Strategy{   
    public void algorithm();    //策略方法
}
```

具体策略

```
class ConcreteStrategyA implements Strategy{
    public void algorithm(){
        System.out.println("具体策略A的策略方法被访问！");
    }
}

class ConcreteStrategyB implements Strategy{
  public void algorithm(){
      System.out.println("具体策略B的策略方法被访问！");
  }
}
```

环境类

```
class Context{
    private Strategy strategy;
    public Strategy getStrategy(){
        return strategy;
    }
    public void setStrategy(Strategy strategy){
        this.strategy=strategy;
    }
    public void algorithm(){
        strategy.algorithm();
    }
}
```

使用

```
public static void main(String[] args){
    Context context = new Context();
    Strategy strategyA = new ConcreteStrategyA();
    context.setStrategy(strategyA);
    context.algorithm();
    System.out.println("-----------------");
    Strategy strategyB = new ConcreteStrategyB();
    context.setStrategy(strategyB);
    context.algorithm();
}
```

## 观察者模式

![](picture/observe.jpg)

角色： 

- Subject：抽象目标
- ConcreteSubject：具体目标
- Observer：抽象观察者
- ConcreteObserver：具体观察者

> - 优点：将复杂的串行处理逻辑变为单元化的独立处理逻辑，被观察者只是按照自己的逻辑发出消息，不用关心谁来消费消息，每个观察者只处理自己关心的内容。逻辑相互隔离带来简单清爽的代码结构。
>
> - 缺点：观察者较多时，可能会花费一定的开销来发消息，但这个消息可能仅一个观察者消费。

> 适用场景: 适用于一对多的的业务场景，一个对象发生变更，会触发N个对象做相应处理的场景。例如：订单调度通知，任务状态变化等。

抽象目标

```
abstract class Subject{
    protected List<Observer> observerList = new ArrayList<Observer>();   
    public void add(Observer observer){  		//增加观察者方法
        observers.add(observer);
    }    
    public void remove(Observer observer){   	//删除观察者方法
        observers.remove(observer);
    }   
    public abstract void notify(); 				//通知观察者方法
}
```

具体目标

```
class ConcreteSubject extends Subject{
   private Integer state;
   public void setState(Integer state){
        this.state = state;  
        notify();				// 状态改变通知观察者
    }
    public void notify(){
        System.out.println("具体目标状态发生改变...");
        System.out.println("--------------");       
        for(Observer obs:observers){
            obs.process();
        }
    }          
}
```

抽象观察者

```
interface Observer{
    void process(); //具体的处理
}
```

具体观察者

```
class ConcreteObserverA implements Observer{
    public void process(){
        System.out.println("具体观察者A处理！");
    }
}

class ConcreteObserverB implements Observer{
    public void process(){
        System.out.println("具体观察者B处理！");
    }
}
```

使用

```
public static void main(String[] args){
    Subject subject = new ConcreteSubject();
    Observer obsA = new ConcreteObserverA();
    Observer obsb = new ConcreteObserverB();
    subject.add(obsA);
    subject.add(obsB);
    subject.setState(0);
}
```

