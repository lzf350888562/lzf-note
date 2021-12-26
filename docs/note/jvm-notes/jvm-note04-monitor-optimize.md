[java性能问题定位](https://javaguide.cn/java/tips/locate-performance-problems/%E6%89%8B%E6%8A%8A%E6%89%8B%E6%95%99%E4%BD%A0%E5%AE%9A%E4%BD%8D%E5%B8%B8%E8%A7%81Java%E6%80%A7%E8%83%BD%E9%97%AE%E9%A2%98/)

# 概述

## 背景说明

**生产环境中的问题**

1.生产环境发生了内存溢出该如何处理？

2.生产环境应该给服务器分配多少内存合适？

3.如何对垃圾回收器的性能进行调优？

4.生产环境CPU负载飙高该如何处理？

5.生产环境应该给应用分配多少线程合适？

6.不加log，如何确定请求是否执行了某一行代码？

7.不加log，如何实时查看某个方法的入参与返回值？

**为什么要调优**

1.防止出现OOM

2.解决OOM

3.减少Full GC出现的频率

**不同阶段的考虑**

1.上线前

2.项目运行阶段

3.线上出现OOM

## 调优概述

**监控的依据**

1.运行日志

2.异常堆栈

3.GC日志

4.线程快照

5.堆转储快照

**调优的大方向**

1.合理地编写代码

2.充分并合理地使用硬件资源

3.合理地进行jvm调优

## 性能优化地步骤

**1发现问题：性能监控**

![](.\picture\m-001.png)

1.GC频繁

2.cpu load 过高

3.OOM

4.内存泄漏

5.死锁

6.程序响应时间过长

**2排查问题：性能分析**

![](.\picture\m-002.png)

1.打印GC日志，通过GCviewer或者GCeasy来分析日志信息

2.灵活运用命令行工具，jstack,jmap,jinfo等

3.dump出堆文件，使用内存分析工具分析文件

4.使用阿里Arthas，或jconsole,jvisualVM来实时查看jvm状态

5.jstack查看堆栈信息

**3解决问题：性能调优**

1.适当增加内存，根据业务背景选择垃圾回收器

2.优化代码，控制内存使用

3.增加机器，分散节点压力

4.合理设置线程池线程数量

5.使用中间件提高程序效率，比如缓存、消息队列等

其他

## 性能评价/测试指标

![](.\picture\m-003.png)

停顿时间/响应时间

![](.\picture\m-004.png)

吞吐量

并发数

内存占用

相互间的关系

# jvm监控及诊断工具

## 概述

体会1:

使用数据说明问题，使用知识分析问题，使用工具处理问题

体会2:

无监控不调优

### 简单命令行工具

![](.\picture\m-005.png)

![](.\picture\m-006.png)

## jps查看正在运行的java进程

java process status

显示指定系统中所有的hotspot虚拟机的进程(查看虚拟机进程信息)，可用于查看正在运行的虚拟机进程。

说明：对于本地虚拟机进程来说，进程的本地虚拟机id于操作系统的进程id是一致的，是唯一的。

### 基本格式

jps [options] [hostid]

![](.\picture\m-007.png)

![](.\picture\m-008.png)

## jstat查看jvm统计信息

![](.\picture\m-009.png)

### 基本语法

jstat -<option> [-t] [-h<lines>] <vmid> [<interval> [<count]]

**option参数**

jstat -gc pid

![](.\picture\m-010.png)

![](.\picture\m-011.png)

**interval参数**

用于指定输出统计数据的周期，单位为毫秒，即：查询间隔

jstat -class pid 1000

1秒输出一次

**count参数**

用于指定查询的总次数

jstat -class pid 1000 10

1秒输出一次 一共10次

**-t参数**

显示程序的运行时间(Timestamp) 单位为秒

我们可以比较java进程的启动时间以及总GC时间(GCT列)，或者两次测量的间隔时间以及总GC时间的增量，来得出GC时间占运行时间的比例。

如果该比例超过20%，则说明目前堆的压力较大，如果超过90%则说明堆里几乎没有可用的空间，随时都可能抛出OOM异常

**-h**

可在周期性数据输出时，输出多少行数据后输出一个表头信息

### 补充

![](.\picture\m-012.png)

## jinfo实时查看和修改jvm配置参数

![](.\picture\m-013.png)

### 基本语法

![](.\picture\m-014.png)

### 查看行为

jinfo -sysprops pid 查看由System.getProperties取得的参数

jinfo -flags pid 查看曾经赋值过的一些参数

jinfo -flags 具体参数 pid 查看具体参数

如jinfo -flag UseGlGC pid

​	jinfo -flag MaxHeapSize pid

### 修改行为

![](.\picture\m-015.png)

boolean类型：

jinfo -flag <u>+</u>PrintGCDetails pid

非boolean类型：

jinfo -flag +MaxHeapFreeRatio=90 pid

### 拓展

java -XX:+PrintFlagsInitial

查看所有jvm参数启动的初始值

java -XX:+PrintFlagsFinal

查看所有jvm参数的最终值(常用)

java -XX:+PrintCommandLineFlags

查看那些以及被用户或者jvm设置过的详细的XX参数的名称和值

## jmap到处内存映像文件&内存使用情况

![](.\picture\m-016.png)

### 基本语法

![](.\picture\m-022.png)

![](.\picture\m-023.png)

### 使用1导出内存印象文件

![](.\picture\m-024.png)

![025](.\picture\m-025.png)

**手动方式**

![](.\picture\m-026.png)



自动方式

OOM时

![](.\picture\m-028.png)

![](.\picture\m-027.png)

### 使用2显示堆内存信息

jmap -heap pid

jmap histo pid           对象实例个数和大小统计

使用3其他(linux)

jmap -permstat pid

查看系统过的ClassLoader信息

jmap -finalizerinfo

查看堆积在finalizer队列中的对象

### 小结

![](.\picture\m-029.png)

## jhatJDK自带堆分析工具

![](.\picture\m-30.png)

### 基本语法

jhat [option] [dumfile]

![](.\picture\m-31.png)

## jstack打印JVM线程快照

![](.\picture\m-032.png)

<img src=".\picture\m-033.png"  />

### 基本语法

![](.\picture\m-034.png)

## jcmd多功能命令行

![](.\picture\m-035.png)

jcmd -l 列出所有的jvm进程 相当于jps -m

jcmd pid help 针对指定的进程，列出支持的所有命令

jcmd pid 具体指令    显示指定进程的指令命令的数据

如 jcmd pid Thread.print 相当于 jstack pid 

​	jcmd pid  GC.class_histogram 相当于 jmap -histo

​	jcmd pid  VM.system_properties 相当于 jinfo -sysprops pid 

​	jcmd pid GC.heap_dump xx.hprof 生成dump文件

如

The following commands are available:
JFR.stop
JFR.start
JFR.dump
JFR.check
VM.native_memory
VM.check_commercial_features
VM.unlock_commercial_features
ManagementAgent.stop
ManagementAgent.start_local
ManagementAgent.start
VM.classloader_stats
GC.rotate_log
Thread.print
GC.class_stats
GC.class_histogram
GC.heap_dump
GC.finalizer_info
GC.heap_info
GC.run_finalization
GC.run
VM.uptime
VM.dynlibs
VM.flags
VM.system_properties
VM.command_line
VM.version
help

## jstatd远程主机信息收集

![](.\picture\m-036.png)

GUI工具

![](.\picture\m-038.png)

![037](.\picture\m-037.png)

![](.\picture\m-039.png)

## jconsole

![](.\picture\m-040.png)

![](.\picture\m-041 .png)

## visualvm

![](.\picture\m-042.png)

jdk9 visualvm不再内置 需要下载 字体很小

![](.\picture\m-043.png)

![](.\picture\m-044.png)

然后在idea配置插件 选择visualvm的位置

### 远程连接

![](.\picture\m-045.png)

### 主要功能

![](.\picture\m-047.png)

## eclipse MAT

![](.\picture\m-048.png)

![](.\picture\m-049.png)

### dump文件

![](.\picture\m-050.png)

### 说明

![](.\picture\m-051.png)

### 获取dump文件

![](.\picture\m-052.png)

### 分析dump文件

**histogram**

展示各个类的实例数目以及这些实例的shallow heap 或retained heap的总和

**thread overview**

查看系统中的java线程

查看局部变量的信息

**获得对象项目引用的关系**

with outgoing references

with incoming references

**深堆与浅堆**

![](.\picture\m-053.png)

![](.\picture\m-054.png)

**对象的实际大小**

![](.\picture\m-055.png)

案例

StudentTrace

![](.\picture\m-056.png)

**支配树**

![](.\picture\m-057.png)

![](.\picture\m-059.png)

## 再谈内存泄漏

### 理解与分类

![](.\picture\m-060.png)

![](.\picture\m-061.png)

![062](.\picture\m-062.png)

![](.\picture\m-063.png)

![](.\picture\m-064.png)

### java中的内存泄漏8种情况

静态集合类

![](.\picture\m-065.png)

单例模式

![](.\picture\m-066.png)

内部类持有外部类

![](.\picture\m-067.png)

各种连接

![](.\picture\m-068.png)

变量不合理的作用域

![](.\picture\m-069.png)

改变哈希值

![](.\picture\m-070.png)

缓存泄漏

![](.\picture\m-071.png)

监听器和回调

![](.\picture\m-072.png)

### 案例

1.Stack

2.安卓开发：在退出后因为线程没有关闭所以key内存泄漏了

![](.\picture\m-073.png)

![](.\picture\m-075.png)

![074](.\picture\m-074.png)

解决办法

![](.\picture\m-076.png)

## 支持使用OQL语言查询对象信息

MAT支持一种类似于SQL的查询语言OQL(Object Query Language)。OQL使用类SQL语法，可以在堆种进行对象的查找和筛选

### SELECT

![](.\picture\m-077.png)

### FROM

![](.\picture\m-078.png)

### WHERE

![](.\picture\m-079.png)

### 内置对象与方法

![](.\picture\m-080.png)

更详细的需要自行学习

## jprofiler

![](.\picture\m-081.png)

![](.\picture\m-082.png)

**主要功能**

![](.\picture\m-083.png)

### 安装

在jprofiler中配置idea

![](.\picture\m-084.png)

在idea中继承jprofiler

安装插件

在settings->Tools->jprofiler配置exe路径

### 数据采集方式

**Instrumentation重构模式**

**Sampling抽样模式**

![](.\picture\m-086.png)

![085](.\picture\m-085.png)

![](.\picture\m-087.png)

**遥感监测Telemetries**

**内存视图Live  Memory**

![](.\picture\m-088.png)

**堆遍历Heap Walker**

**cpu视图 cpu views**

**线程视图 threads**

![](.\picture\m-089.png)

**监视器&锁Monitors&locks**

![](.\picture\m-090.png)

## Arthas

visualvm与jprofiler

![](.\picture\m-091.png)

![](.\picture\m-092.png)

![](.\picture\m-093.png)

### 下载安装与使用

![](.\picture\m-095.png)

![094](.\picture\m-094.png)

工程目录

![](.\picture\m-096.png)

启动

![](.\picture\m-097.png)

查看日志

cat ~/logs/arthas/arthas.log

查看帮相

java -jar arthas -boot.jar -h

web console

![](.\picture\m-098.png)

退出

![](.\picture\m-099.png)

### 指令

具体在官方查看

**基础指令**

![](.\picture\m-100.png)

**jvm相关**

![](.\picture\m-101.png)

dashboard：

dashboard -h 帮助

默认隔一段时间打印 

-i 指定间隔

-n 指定打印次数

thread：

id 查看指定线程

-b 查看阻塞线程

-i 指定时间段

**class/classloader相关**

![](.\picture\m-102.png)

sc

![](.\picture\m-103.png)

sm

![](.\picture\m-104.png)

jad

![](.\picture\m-105.png)

也可以仅编译方法

mc、redefine

![](.\picture\m-106.png)

classloader

![](.\picture\m-107.png)

monitor/watch/trace相关

monitor

![](.\picture\m-108.png)

trace

![](.\picture\m-111.png)

watch

![](.\picture\m-109.png)

![110](.\picture\m-110.png)

stack

![](.\picture\m-112.png)

tt

![](.\picture\m-113.png)

**其他**

![](.\picture\m-114.png)

## java mission control

![](.\picture\m-115.png)

![116](.\picture\m-116.png)

![117](.\picture\m-117.png)

![](.\picture\m-118.png)

### java Flight Recorder

![](.\picture\m-121.png)

事件类型

![](.\picture\m-119.png)

启动方式

![](.\picture\m-122.png)

![123](.\picture\m-123.png)

![124](.\picture\m-124.png)

取样分析

![](.\picture\m-120.png)

![](.\picture\m-126.png)

![125](.\picture\m-125.png)

## 其他工具

Flame Graphs火焰图

![](.\picture\m-127.png)

Tprofiler

![](.\picture\m-128.png)

下载：https://github.com/alibaba/TProfiler

Btrace

![](.\picture\m-129.png)

# jvm运行时参数

## jvm参数选项类型

### 标志参数选项：

特点：比较稳定，后续版本基本不会变化，以-开头

运行java或java -help 可以看到所有的标准选项

-X参数选项：

特点：非标准化参数，功能比较稳定，以-X开头

运行java -X可以看到所有X选项

![](.\picture\m-130.png)

-XX参数选项：

特点：非标准化参数，使用的最多的参数类型，这类选项属于实验性，不稳定，以-XX开头。用于开发和调试jvm

分类：

boolean类型：-XX:+/-<option>

非boolean类型

数值型格式 -XX:<option>=<number>

number表示数值，可以带参数 不区分大小写

非数值型格式-XX:<name>=<string>

![](.\picture\m-131.png)

## 添加jvm参数选项

![](.\picture\m-131.png)

![](.\picture\m-132.png)

![133](.\picture\m-133.png)

## 常用jvm参数选项

### 打印设置的XX选项和值

![](.\picture\m-134.png)

### 堆、栈、方法区内存相关选项

```
* -XX:+PrintFlagsFinal -Xms600m -Xmx600m
*  -XX:SurvivorRatio=8  需要显式指定，为了让这个设置生效 还需关闭自适应策略↓
*  -XX:-UseAdaptiveSizePolicy
* 默认情况下，新生代占 1/3 ： 200m，老年代占2/3 : 400m
*   其中，Eden默认占新生代的8/10 : 160m ,Survivor0，Survivor1各占新生代的1/10 ： 20m
```

栈

![](.\picture\m-135.png)

堆

![136](.\picture\m-136.png)

![137](.\picture\m-137.png)

方法区

![](.\picture\m-138.png)

直接内存

![](.\picture\m-139.png)

### OOM相关选项

![](.\picture\m-139.png)

### 垃圾回收器相关

查看默认垃圾回收器

![](.\picture\m-140.png)

各种垃圾收集器具体

![](.\picture\m-141.png)

![142](.\picture\m-142.png)

Parallel回收器

![143](.\picture\m-143.png)

![](.\picture\m-144.png)

cms

![145](.\picture\m-145.png)

![146](.\picture\m-146.png)

![](.\picture\m-147.png)

g1

![](.\picture\m-148.png)

![149](.\picture\m-149.png)

![](.\picture\m-150.png)

### GC日志相关选项

常用参数

![](.\picture\m-151.png)

其他参数

![](.\picture\m-152.png)

### 其他参数

![](.\picture\m-153.png)

## 通过java代码获取jvm参数

![](.\picture\m-154.png)

![155](.\picture\m-155.png)

![156](.\picture\m-156.png)

# 分析GC日志

![](.\picture\m-157.png)

![](.\picture\m-158.png)

## GC日志分类

### MinorGC

![](.\picture\m-159.png)

### FullGC

![160](.\picture\m-160.png)

## GC日志结构剖析

### 垃圾回收器

![](.\picture\m-161.png)

### GC前后情况

![162](.\picture\m-162.png)

### GC时间

![163](.\picture\m-163.png)

## GC日志分析工具

![](.\picture\m-163.png)

![](.\picture\m-164.png)