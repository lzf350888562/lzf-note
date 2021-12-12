#  JVM与Java体系结构

默认虚拟机（java -version ）：Java HotSpot(TM) 64-Bit Server VM (build 9.0.4+11, mixed mode)

**java:跨平台语言**

java程序 -->

字节码文件 -->

win版jvm/linux版jvm/Mac版jvm

write once,run anywhere

**jvm:跨语言的平台**

kotlin/clojure/groovy/scala/jython/jruby/javascript -->

各自的编译器 -->

字节码文件 -->

java虚拟机  只关心字节码文件

(java7发布:通过JSR-292规范基本实现在java虚拟机平台上允许非java语言编写的和程序)

> **字节码**:平常说的java字节码指的是用java语言编译成的字节码，准确的说任何能在jvm平台上直执行的字节码格式都一样，所以应该统称为jvm字节码

java**虚拟机解释**

java虚拟机就是二进制字节码的运行环境，负责装在字节码到其内部，解释、编译为对应平台上的机器指令执行。每一条java指令，java虚拟机规范中都由详细定义，如怎么取操作数，怎么处理操作数，处理结果放在哪。

特点

1.一次编译，到处运行 (javac)

2.自动内存管理

3.自动垃圾回收功能

**jvm整体结构**

![jvm整体结构](.\jvm-img\jvm整体结构.jpg)

多线程共享堆和方法区

每个线程独享虚拟机栈、本地方法栈、程序计数器

执行引擎：翻译字节码（解释执行）+JIT编译器（编译执行）

二次编译：将字节码编译成机器指令.

电脑只能执行机器指令，jvm由执行引擎完成。

![jvm架构模型](.\jvm-img\jvm架构模型.png)

两种架构执行同样功能需要的指令

![栈式架构与寄存器架构同样操作需要的指令](.\jvm-img\栈式架构与寄存器架构同样操作需要的指令.png)

指令集分别为8位 16位

由于跨平台性的设计 java的指令都是根据栈来设计。

不同平台cpu架构不同所以不能设计位基于寄存器的架构

## 反编译

实际上为解析字节码文件

1.javap -v xxxxx.class

2.jclasslib

## jvm生命周期

1.虚拟器的启动

是通过引导类加载器创建一个初始类来完成的，这个类是由虚拟机的具体实现指定的。

2.虚拟机的执行

一个运行中的java虚拟机有着一个清晰的任务：执行java程序。

程序开始执行他才运行，程序结束时他就停止。

执行一个所谓的java程序的时候，真正在执行的是一个叫做java虚拟机的进程

jps 输出显示当前进程

3.虚拟机的退出

以下几种情况：

程序正常执行结束

程序遇到异常或错误而终止

操作系统错误导致java虚拟机进程终止

某线程调用Runtime类或System类的exit方法，或Runtime类的halt方法，并且java安全管理器也允许这次exit或halt操作。 

# 类加载子系统

![类加载子系统图1](.\jvm-img\类加载子系统图1.png)    ![](.\jvm-img\类加载子系统图1中文版.png)

## 类加载的过程和类加载器分类

![](.\jvm-img\类加载子系统的作用.png)

CAFEBABE 开头字节码文件

![![](.\jvm-img\类加载器代码1.png](.\jvm-img\类加载器示例代码1.png)



## Classloader

![](.\jvm-img\类加载子系统说明.png)

### java9类加载器模块化

java9之前的classloader：

- bootstrap classloader加载rt.jar，jre/lib/endorsed
- ext classloader加载jre/lib/ext
- application classloader加载-cp指定的类

java9以及之后的classloader：

- bootstrap classloader加载lib/modules 

  java.base                  			 java.security.sasl

  java.datatransfer           java.xml

  java.desktop                jdk.httpserver

  java.instrument             jdk.internal.vm.ci

  java.logging                jdk.management

  java.management             jdk.management.agent

  java.management.rmi         jdk.naming.rmi

  java.naming                 jdk.net

  java.prefs                  jdk.sctp

  java.rmi                    jdk.unsupported

- ext classloader更名为platform classloader，加载lib/modules

  java.activation*            jdk.accessibility

  java.compiler*              jdk.charsets

  java.corba*                 jdk.crypto.cryptoki

  java.scripting              jdk.crypto.ec

  java.se                     jdk.dynalink

  java.se.ee                  jdk.incubator.httpclient

  java.security.jgss          jdk.internal.vm.compiler*

  java.smartcardio            jdk.jsobject

  java.sql                    jdk.localedata

  java.sql.rowset             jdk.naming.dns

  java.transaction*           jdk.scripting.nashorn

  java.xml.bind*              jdk.security.auth

  java.xml.crypto             jdk.security.jgss

  java.xml.ws*                jdk.xml.dom

  java.xml.ws.annotation*     jdk.zipfs

- application classloader加载-cp，-mp指定的类 

  jdk.aot                     jdk.jdeps
  jdk.attach                  jdk.jdi
  jdk.compiler                jdk.jdwp.agent
  jdk.editpad                 jdk.jlink
  jdk.hotspot.agent           jdk.jshell
  jdk.internal.ed             jdk.jstatd
  jdk.internal.jvmstat        jdk.pack
  jdk.internal.le             jdk.policytool
  jdk.internal.opt            jdk.rmic
  jdk.ja rtool                 jdk.scripting.nashorn.shell
  jdk.javadoc                 jdk.xml.bind*
  jdk.jcmd                    jdk.xml.ws*
  jdk.jconsole

小结

java9模块化之后，对classloader有所改造，其中一点就是将ext classloader改为platform classloader，另外模块化之后，对应的classloader加载各自对应的模块。

## 运行时数据区

![](.\jvm-img\运行时数据区和线程.png)

## 虚拟机栈

![](.\jvm-img\虚拟机栈2.png)



![](.\jvm-img\虚拟机栈3.png)

![](.\jvm-img\虚拟机栈4.png)

  

返回指令areturn 表示引用类型

返回指令return 表示void类型

### 虚拟机栈总结

栈帧的内部结构  

重要：局部变量表和操作数栈

还有：方法返回地址和动态链接

可忽略：附加信息

面试题：

1.内存溢出OOM ：OutOfMemoryError

2.需要考虑多种情况，不能

3.不是，只能减少溢出几率。固定内存给其他需要内存的余地留得少了

4.不会 

程序计数器 运算速度最快 空间小 无Error 无GC

虚拟机栈 只有进栈出栈 不需要显式回收  有Error 无GC

本地方法栈  有Error 无GC

堆 有Error <u>有GC</u>

方法区 生命周期长 有Error 有GC

## 本地方法栈

![](.\jvm-img\本地方法接口1.png)

在内存溢出方面与虚拟机栈相同

Hotspot jvm中 直接将本地方法栈与虚拟机栈合二为一

![](.\jvm-img\本地方法栈1.png)

## 堆的内存分配与GC

![](.\jvm-img\堆1.png)

```java
 * 1. 设置堆空间大小的参数
 * -Xms 用来设置堆空间（年轻代+老年代）的初始内存大小
 *      -X 是jvm的运行参数
 *      ms 是memory start
 * -Xmx 用来设置堆空间（年轻代+老年代）的最大内存大小
 *
 * 2. 默认堆空间的大小
 *    初始内存大小：物理电脑内存大小 / 64
 *             最大内存大小：物理电脑内存大小 / 4
 *    物理电脑内存大小实际上是小于原物理内存大小的
 * 3. 手动设置：-Xms600m -Xmx600m
 *     开发中建议将初始堆内存和最大的堆内存设置成相同的值。
 *       最大堆内存如果大于初始堆内存在所用内存不足时进行扩容，会影响系统性能
 * 4. 查看设置的参数：方式一：命令行（环境变量） jps ：查看当前进程 /  jstat -gc 进程id   查看进程内存使用情况
 *                   老年代 OC：总量 OU：已使用
 *                   新生代 EC: 伊甸园区总量 EU：伊甸园区已使用
 *                         S0C: S0区总量  S0U：S0区已使用
 *                         S1C:           S1U:
 *                         S0C（或S1C）+EC+OC=Runtime.getRuntime().totalMemory()
 *                         S0与S1  同时只有一个与伊甸园区配合使用
 *                  方式二：-XX:+PrintGCDetails
 *                  jdk文件下bin目录下的jvisualvm.exe 查看
 *                   jdk8以前自带 jdk9以及以后需自行下载 且需要在解压文件的etc目录下的visualvm.conf文件下加入jdk路径
 *                   在bin下启动  且安装visual gc插件
 *                  在visual GC 的graphs项中能看到
 *                   Eden Space、Survivor 0、Survivor 1、Old Gen 的第二个 参数加起来和为 10M
```

```java
* -XX:NewRatio ： 设置新生代与老年代的比例。默认值是2. 即新生代1 老年代2
* 可命令行 jinfo -flag NewRatio 进程id 查看    
* -XX:SurvivorRatio ：设置新生代中Eden区与Survivor区的比例。默认值是8 即8 1 1 自适应为6 即6 1 1
* 、可命令行 jinfo -flag SurvivorRatio 进程id 查看  
* -XX:-UseAdaptiveSizePolicy ：关闭自适应的内存分配策略  （暂时用不到）  -：关闭 
* -Xmn:设置新生代的空间的大小。 （一般不设置：因为会设置堆的总大小与新老年代比例后间接设置了此参数）
```

先看下图的一部分后 存在的遗留问题 ：

伊甸园区的回收会促使s0或s1的回收（YGC），并不意味着幸存者区会在满时进行自动gc。

1.当幸存者区满足一定条件时，有特殊的规则(动态对象年龄判断)让其晋升到老年区：

在允许担保分配下, 当幸存者区中低于某个年龄的对象容量大于幸存者区容量的一半时, 高于该年龄的对象全部担保到老年区

2.如果对象的大小比伊甸园区的容量大，直接考虑放入老年区。

老年区垃圾回收算法：majorgc，fullgc

如果老年区再放不下 ->OOM

这里的前提 不允许动态扩建堆。

![](.\jvm-img\堆2.png)

OOM总是伴随着full gc

![](.\jvm-img\堆3.png)

![为什么有TLAB](.\jvm-img\为什么有TLAB.png)

![](.\jvm-img\什么是TLAB.png)



![](.\jvm-img\堆图1.png)

![ ](.\jvm-img\TLAB的再说明.png)

```
优先在tlab空间分配，否则加锁分配
jinfo -flag UseTLAB 进程id   查看是否开启TLAB   
```

![](.\jvm-img\堆图2.png)





### 堆空间参数设置

![image243](.\jvm-img\media\image243.png)

具体查看某个参数  命令行 jps jinfo -flag 参数名称 进程id![image244](.\jvm-img\media\image244.png)

![image245](.\jvm-img\media\image245.png)

现在HandlePromotionFailure始终为TRUE 即只要检查红字部分 。  

![image246](.\jvm-img\media\image246.png)

### 逃逸分析

![image247](.\jvm-img\media\image247.png)

![image248](.\jvm-img\media\image248.png)

![image249](.\jvm-img\media\image249.png)

![image250](.\jvm-img\media\image250.png)

![image251](.\jvm-img\media\image251.png)

![image252](.\jvm-img\media\image252.png)

![image253](.\jvm-img\media\image253.png)

![image254](.\jvm-img\media\image254.png)

![image255](.\jvm-img\media\image255.png)

![image256](.\jvm-img\media\image256.png)

 

![image257](.\jvm-img\media\image257.png)

偏向锁![image258](.\jvm-img\media\image258.png)

![image259](.\jvm-img\media\image259.png)

![image260](.\jvm-img\media\image260.png)

局部变量表中

![image261](.\jvm-img\media\image261.png)

![image262](.\jvm-img\media\image262.png)

server在64位电脑上默认开启 java -version 可查看![image263](.\jvm-img\media\image263.png)

**对象实例目前都是分配在堆上**

![image264](.\jvm-img\media\image264.png)

## 方法区

![image269](.\jvm-img\media\image269.png)

![image270](.\jvm-img\media\image270.png)

![image271](.\jvm-img\media\image271.png)

**方法区的解释**

大体：共享 存储运行时类的结构、常量池

![image273](.\jvm-img\media\image273.png)

![image274](.\jvm-img\media\image274.png)

![image275](.\jvm-img\media\image275.png)

jdk8中，类元数据存储在本地内存中，这个空间叫元空间。

以前永久代为方法区属于jvm内存一部分所以容易oom，现在元空间为方法区，属于本地内存了，在jvm外。

本地内存不足时oom 

![image276](.\jvm-img\media\image276.png)

![image277](.\jvm-img\media\image277.png)

![image278](.\jvm-img\media\image278.png)



![image280](.\jvm-img\media\image280.png)

![image281](.\jvm-img\media\image281.png)

![image282](.\jvm-img\media\image282.png)

![image283](.\jvm-img\media\image283.png)

![image284](.\jvm-img\media\image284.png)

![image286](.\jvm-img\media\image286.png)

![image287](.\jvm-img\media\image287.png)

![image288](.\jvm-img\media\image288.png)

![image289](.\jvm-img\media\image289.png)

![image290](.\jvm-img\media\image290.png)

![image291](.\jvm-img\media\image291.png)



![image292](.\jvm-img\media\image292.png)

即在字节码文件中可以看到 其值，而非final的类变量在初始化clinit时候赋值  

### 常量池与运行时常量池

![image293](.\jvm-img\media\image293.png)

![image294](.\jvm-img\media\image294.png)

![image295](.\jvm-img\media\image295.png)

![image296](.\jvm-img\media\image296.png)

![image297](.\jvm-img\media\image297.png)

![image298](.\jvm-img\media\image298.png)

![image299](.\jvm-img\media\image299.png)

![image300](.\jvm-img\media\image300.png)

![image301](.\jvm-img\media\image301.png)

![image302](.\jvm-img\media\image302.png)

![image303](.\jvm-img\media\image303.png)

![image304](.\jvm-img\media\image304.png)

![image305](.\jvm-img\media\image305.png)

![image306](.\jvm-img\media\image306.png)

![image307](.\jvm-img\media\image307.png)

![image308](.\jvm-img\media\image308.png)

![image309](.\jvm-img\media\image309.png)

![image310](.\jvm-img\media\image310.png)

![image311](.\jvm-img\media\image311.png)

![image312](.\jvm-img\media\image312.png)

![image313](.\jvm-img\media\image313.png)

![image314](.\jvm-img\media\image314.png)

![image315](.\jvm-img\media\image315.png)

![image316](.\jvm-img\media\image316.png)

![image317](.\jvm-img\media\image317.png)

### 方法区的演进

![image319](.\jvm-img\media\image319.png)

![image320](.\jvm-img\media\image320.png)

![image321](.\jvm-img\media\image321.png)

![image322](.\jvm-img\media\image322.png)

![image324](.\jvm-img\media\image324.png)

![image325](.\jvm-img\media\image325.png)

![image326](.\jvm-img\media\image326.png)

![image327](.\jvm-img\media\image327.png)

存在于堆的老年代中

再次印证了new的对象都在堆中

![image328](.\jvm-img\media\image328.png)

jdk9自带的工具

![image329](.\jvm-img\media\image329.png)

![image330](.\jvm-img\media\image330.png)

![image331](.\jvm-img\media\image331.png)

### 方法区的垃圾回收

![image333](.\jvm-img\media\image333.png)

![image334](.\jvm-img\media\image334.png)

![image335](.\jvm-img\media\image335.png)

![image336](.\jvm-img\media\image336.png)



![image337](.\jvm-img\media\image337.png)

![image338](.\jvm-img\media\image338.png)

![image342](.\jvm-img\media\image342.png)

![image343](.\jvm-img\media\image343.png)

Class的newInstance：反射的方式创建对象，只能调用空参的构造器，权限必须时public

Constructor的newInstance：反射的方式，可调用空参、带参的构造器，没有权限要求。

clone：不需要任何构造器，需要当前类实现Cloneable接口，实现clone方法。

反序列化：从文件、网络中获取一个对象的二进制流，然后还原成内存中的对象。

### 对象创建与内存布局图

![image344](.\jvm-img\media\image344.png)

 1.虚拟机遇到一条new指令，首先区检查这个指令的参数能否在Metaspace的常量池中定位到一个类的符号引用，并且检查这个符号引用代表的类是否已经加载、解析和初始化。（即判断类元信息是否存在）。如果没有，那么在双亲委派模式下，使用当前类加载器以ClassLoader+包名+类名为Key进行查找对应的.class文件，如果没有找到文件,则抛出ClassNotFoundException异常，如果找到，则进行类加载，并生成对应的Class类对象。

2.为对象分配内存：首先计算对象占用空间大小，接着在堆中划分一块内存给对象。如果实列成员变量时引用变量，仅分配引用变量空间即可，即4字节大小。

如果内存规时规整的，那么虚拟机将采用的是指针碰撞法来为对象分配内存。指针碰撞法意思是所有用过的内存在一边，空闲的内存在另一边，中间放着一个指针作为分界点的指示器，分配内存就仅仅是把指针指向空闲的那边挪动一段与对象大小相等的距离罢了。如果垃圾收集器选择的是Serial、ParNew这种基于压缩算法的，虚拟机采用这种分配方式，一般使用带有compact（整理）过程的收集器时，使用指针碰撞。

如果内存不规整：虚拟机需要维护一个列表：空间列表分配法，意思是虚拟机维护了一个列表，记录上那些内存块是可用的，再分配的时候从列表中找到一块足够大的空间划分给对象实例并更新列表上的内容。

说明：选择哪种分配方式由java堆是否规整决定，而java堆是否规整又由所采用的垃圾收集器是否带有压缩整理功能决定。

5.设置对象的对象头：  将对象的所属类（即类的元数据信息）、对象的hashcode和对象的gc信息、锁信息等数据存储在对象头中。这个过程的具体设置方式取决于jvm的实现

6.init：从java程序的视角来看，初始化才正式开始，初始化成员变量，执行实例化代码块，调用类的构造方法，并把堆内对象的首地址赋给引用变量。因此一般来说（由字节码中是否跟随由invokespecial指令所决定），new指令之后会接着就是执行方法，把对象按照程序员的意愿进行初始化，这样一个真正的可用的对象才算完全创建出来。

![image345](.\jvm-img\media\image345.png)

![s](.\jvm-img\media\image346.png)

对象头（运行时元数据(mark word)和类型指针）+实例数据+对齐填充

![image347](.\jvm-img\media\image347.png)

### 对象的访问定位

![image350](.\jvm-img\media\image350.png)

![image349](.\jvm-img\media\image349.png)

句柄访问：引用中存储稳定句柄地址，对象被移动时（在垃圾回收时移动对象很普遍）只会改变句柄中实例数据指针即可，引用本身不需要被修改。

![image351](.\jvm-img\media\image351.png)

直接指针（Hotspot使用）：引用直接指向实例数据，效率高，速度快

![image352](.\jvm-img\media\image352.png)

## 直接内存

![image354](.\jvm-img\media\image354.png)

![image355](.\jvm-img\media\image355.png)

![image356](.\jvm-img\media\image356.png)

![image357](.\jvm-img\media\image357.png)

![image358](.\jvm-img\media\image358.png)

![image359](.\jvm-img\media\image359.png)

## 执行引擎

![image363](.\jvm-img\media\image363.png)

![image364](.\jvm-img\media\image364.png)

![image365](.\jvm-img\media\image365.png)

![image366](.\jvm-img\media\image366.png)

![image367](.\jvm-img\media\image367.png)

![image368](.\jvm-img\media\image368.png)

![image370](.\jvm-img\media\image370.png)

橙色：前端编译器 javac 与jvm无关

绿色：解释

蓝色：编译



![image371](.\jvm-img\media\image371.png)

![image372](.\jvm-img\media\image372.png)

![image373](.\jvm-img\media\image373.png)

![image374](.\jvm-img\media\image374.png)

![image375](.\jvm-img\media\image375.png)

![image376](.\jvm-img\media\image376.png)

![image377](.\jvm-img\media\image377.png)

![image378](.\jvm-img\media\image378.png)

![image379](.\jvm-img\media\image379.png)

![image380](.\jvm-img\media\image380.png)

![image381](.\jvm-img\media\image381.png)

![image382](.\jvm-img\media\image382.png)

![image383](.\jvm-img\media\image383.png)

![image385](.\jvm-img\media\image385.png)

![image386](.\jvm-img\media\image386.png)

![image387](.\jvm-img\media\image387.png)

![image388](.\jvm-img\media\image388.png)

![image389](.\jvm-img\media\image389.png)

![image390](.\jvm-img\media\image390.png)

![image391](.\jvm-img\media\image391.png)

![image392](.\jvm-img\media\image392.png)

![image393](.\jvm-img\media\image393.png)

![image394](.\jvm-img\media\image394.png)

![image395](.\jvm-img\media\image395.png)

![image396](.\jvm-img\media\image396.png)

![image397](.\jvm-img\media\image397.png)



![image398](.\jvm-img\media\image398.png)

![image399](.\jvm-img\media\image399.png)

![image400](.\jvm-img\media\image400.png)

![image401](.\jvm-img\media\image401.png)

![image402](.\jvm-img\media\image402.png)

HotSpot VM执行方式设置

![image403](.\jvm-img\media\image403.png)

cmd: 

java -Xint -version

java -Xcomp -version

java -Xmixed -version

或者作为 VM options参数

![image404](.\jvm-img\media\image404.png)

64位JDK仅支持-server

![image405](.\jvm-img\media\image405.png)

![image406](.\jvm-img\media\image406.png)

![image407](.\jvm-img\media\image407.png)

![image408](.\jvm-img\media\image408.png)

![image409](.\jvm-img\media\image409.png)

![image410](.\jvm-img\media\image410.png)