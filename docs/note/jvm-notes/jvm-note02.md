







### StringTable

![](.\jvm-img\media1\image1.png)

![image2](.\jvm-img\media1\image2.png)

![image3](.\jvm-img\media1\image3.png)

##### String的特性

![image4](.\jvm-img\media1\image4.png)

![image5](.\jvm-img\media1\image5.png)

![image6](.\jvm-img\media1\image6.png)

![image7](.\jvm-img\media1\image7.png)

![image8](.\jvm-img\media1\image8.png)

![image9](.\jvm-img\media1\image9.png)

```
intern(); //如果字符串常量池中没有对应的字符串的话，则在常量池中生成，如果不存在 则返回其引用。
为什么要有这个方法？因为非双引号声明的字符串或字符串拼接中存在变量的字符串不存在常量池中，可通过此方法加入常量池。
```

##### String的内存分配

![image10](.\jvm-img\media1\image10.png)

![image11](.\jvm-img\media1\image11.png)

![image12](.\jvm-img\media1\image12.png)

![image13](.\jvm-img\media1\image13.png)

![image14](.\jvm-img\media1\image14.png)

![image15](.\jvm-img\media1\image15.png)

永久代垃圾回收频率低且PermSize默认较小

##### String的基本操作

![image16](.\jvm-img\media1\image16.png)

![image17](.\jvm-img\media1\image17.png)

![image18](.\jvm-img\media1\image18.png)

![image19](.\jvm-img\media1\image19.png)

```java
因为toString方法的字符串也是通过双引号字符串返回的，所以其也在常量池里
```

##### 字符串的拼接操作

![image20](.\jvm-img\media1\image20.png)

![image21](.\jvm-img\media1\image21.png)

![image22](.\jvm-img\media1\image22.png)

![image23](.\jvm-img\media1\image23.png)

```
//如果拼接符号的前后出现了变量，则相当于在堆空间中new String()，具体的内容为拼接的结果：javaEEhadoop
//被final修饰的String在拼接时是作为常量来拼接的

    字符串拼接操作不一定使用的是StringBuilder!
       如果拼接符号左右两边都是字符串常量或常量引用，则仍然使用编译期优化，即非StringBuilder的方式。
       
    /*
    体会执行效率：通过StringBuilder的append()的方式添加字符串的效率要远高于使用String的字符串拼接方式！
    详情：① StringBuilder的append()的方式：自始至终中只创建过一个StringBuilder的对象
          使用String的字符串拼接方式：创建过多个StringBuilder和String的对象
         ② 使用String的字符串拼接方式：内存中由于创建了较多的StringBuilder和String的对象，内存占用更大；如果进行GC，需要花费额外的时间。

     改进的空间：在实际开发中，如果基本确定要前前后后添加的字符串长度不高于某个限定值highLevel的情况下,建议使用构造器实例化：
               StringBuilder s = new StringBuilder(highLevel);//new char[highLevel]
     */
```

![image24](.\jvm-img\media1\image24.png)

![image25](.\jvm-img\media1\image25.png)

##### intern()使用

![image26](.\jvm-img\media1\image26.png)

![image27](.\jvm-img\media1\image27.png)

![image28](.\jvm-img\media1\image28.png)

###### new String问题（重要）

```
 * 题目：
 * new String("ab")会创建几个对象？看字节码，就知道是两个。
 *     一个对象是：new关键字在堆空间创建的          new #2<java/lang/String>
 *     另一个对象是：字符串常量池中的对象"ab"。 字节码指令：ldc #3 <ab>
 *
 *
 * 思考：
 * new String("a") + new String("b")呢？   6个
 *  对象1：new StringBuilder()           new #2 <java/lang/StringBuilder>
 *  对象2： new String("a")              new #4 <java/lang/String>
 *  对象3： 常量池中的"a"                 ldc #5 <a>
 *  对象4： new String("b")              new #4 <java/lang/String>
 *  对象5： 常量池中的"b"                 ldc #8 <b>
 *
 *  深入剖析： StringBuilder的toString():
 *  对象6 ：new String("ab")
 *  强调一下，toString()的调用，在字符串常量池中，没有生成"ab"  因为toString的参数为变量
 字符串常量池中，没有生成"ab
 字符串常量池中，没有生成"ab
 字符串常量池中，没有生成"ab
 字符串常量池中，没有生成"ab
```

![image29](.\jvm-img\media1\image29.png)

![image30](.\jvm-img\media1\image30.png)

![image31](.\jvm-img\media1\image31.png)

![image32](.\jvm-img\media1\image32.png)

jdk6与jdk7的intern区别

![image33](.\jvm-img\media1\image33.png)

![image34](.\jvm-img\media1\image34.png)

![image35](.\jvm-img\media1\image35.png)

![image36](.\jvm-img\media1\image36.png)

![image37](.\jvm-img\media1\image37.png)

![image38](.\jvm-img\media1\image38.png)

![image39](.\jvm-img\media1\image39.png)

![image40](.\jvm-img\media1\image40.png)

![image41](.\jvm-img\media1\image41.png)

![image42](.\jvm-img\media1\image42.png)

##### G1中的String去重操作

##### ![image43](.\jvm-img\media1\image43.png)

常量池本身就是不重复的

String str1 = new String("hello");

String str2 = new String("hello");

针对该类去重

![image44](.\jvm-img\media1\image44.png)

![image45](.\jvm-img\media1\image45.png)

![image46](.\jvm-img\media1\image46.png)

![image47](.\jvm-img\media1\image47.png)

### 垃圾回收概述

![image48](.\jvm-img\media1\image48.png)

![image49](.\jvm-img\media1\image49.png)

##### 什么是垃圾？面试题

![image50](.\jvm-img\media1\image50.png)

![image51](.\jvm-img\media1\image51.png)

![image52](.\jvm-img\media1\image52.png)

![image53](.\jvm-img\media1\image53.png)

![image54](.\jvm-img\media1\image54.png)

![image55](.\jvm-img\media1\image55.png)

![image56](.\jvm-img\media1\image56.png)

![image57](.\jvm-img\media1\image57.png)

![image58](.\jvm-img\media1\image58.png)

![image59](.\jvm-img\media1\image59.png)

![image60](.\jvm-img\media1\image60.png)

![image61](.\jvm-img\media1\image61.png)

##### java垃圾回收机制

![image62](.\jvm-img\media1\image62.png)

![image63](.\jvm-img\media1\image63.png)

![image64](.\jvm-img\media1\image64.png)

![image65](.\jvm-img\media1\image65.png)

![image66](.\jvm-img\media1\image66.png)

### 垃圾回收相关算法

![image67](.\jvm-img\media1\image67.png)

![image68](.\jvm-img\media1\image68.png)

![image69](.\jvm-img\media1\image69.png)

##### 标记阶段：引用计数算法

致命缺陷 无法处理循环引用 所以jvm没有采用此算法

![image70](.\jvm-img\media1\image70.png)

![image71](.\jvm-img\media1\image71.png)

![image72](.\jvm-img\media1\image72.png)

![image73](.\jvm-img\media1\image73.png)

![image74](.\jvm-img\media1\image74.png)

![image75](.\jvm-img\media1\image75.png)

![image76](.\jvm-img\media1\image76.png)

##### 标记阶段：可达性分析算法

![image77](.\jvm-img\media1\image77.png)

![image78](.\jvm-img\media1\image78.png)

![image79](.\jvm-img\media1\image79.png)

![image80](.\jvm-img\media1\image80.png)

###### GC Roots（重要）

判断时需要STW：stop the world

简单理解：

如果一个指针保存了堆内存里面的对象，但自己又不在堆中，它就是个root

![image81](.\jvm-img\media1\image81.png)

![image82](.\jvm-img\media1\image82.png)

![image83](.\jvm-img\media1\image83.png)

![image84](.\jvm-img\media1\image84.png)

##### 对象终止（finalization）机制

![image85](.\jvm-img\media1\image85.png)

![image86](.\jvm-img\media1\image86.png)

![image87](.\jvm-img\media1\image87.png)

###### 对象的三种状态

![image88](.\jvm-img\media1\image88.png)

![image89](.\jvm-img\media1\image89.png)

##### MAT与JProfiler的GC Roots溯源

![image90](.\jvm-img\media1\image90.png)

![image91](.\jvm-img\media1\image91.png)

![image92](.\jvm-img\media1\image92.png)

![image93](.\jvm-img\media1\image93.png)

##### 清除阶段：标记-清除算法

![image94](.\jvm-img\media1\image94.png)

![image95](.\jvm-img\media1\image95.png)

标记：利用GC Roots标记可达对象在对象头中

清除：遍历对象头中没有标记可达的对象。

![image96](.\jvm-img\media1\image96.png)

![image97](.\jvm-img\media1\image97.png)

![image98](.\jvm-img\media1\image98.png)

##### 清除阶段：复制算法

![image99](.\jvm-img\media1\image99.png)

![image100](.\jvm-img\media1\image100.png)

![image101](.\jvm-img\media1\image101.png)

适合于存活对象少，垃圾对象多的场景（新生代）

![image102](.\jvm-img\media1\image102.png)

缺点需要维护对象引用；如果存活的对象很多甚至全存活，复制算法效率差。

前面讲过访问对象的两种方式：

1.句柄访问：局部变量表的引用指向句柄池里的到对象实例数据的指针，复制算法中要改变句柄池

2.直接访问：局部变量表的引用直接指向堆中对象实例数据，复制算法要改变引用



![image103](.\jvm-img\media1\image103.png)

##### 清除阶段：标记-压缩（整理）算法

![image104](.\jvm-img\media1\image104.png)

![image105](.\jvm-img\media1\image105.png)

![image106](.\jvm-img\media1\image106.png)

![image107](.\jvm-img\media1\image107.png)

![image108](.\jvm-img\media1\image108.png)

![image109](.\jvm-img\media1\image109.png)

![image110](.\jvm-img\media1\image110.png)

![image111](.\jvm-img\media1\image111.png)

##### 分代收集算法

![image112](.\jvm-img\media1\image112.png)

![image113](.\jvm-img\media1\image113.png)

![image114](.\jvm-img\media1\image114.png)

###### CMS

![image115](.\jvm-img\media1\image115.png)

##### 增量收集算法、分区算法

降低STW时间的算法

![image116](.\jvm-img\media1\image116.png)

![image117](.\jvm-img\media1\image117.png)

![image118](.\jvm-img\media1\image118.png)

![image119](.\jvm-img\media1\image119.png)

![image120](.\jvm-img\media1\image120.png)

![image121](.\jvm-img\media1\image121.png)

### 垃圾回收相关概念

![image122](.\jvm-img\media1\image122.png)

![image123](.\jvm-img\media1\image123.png)

![image124](.\jvm-img\media1\image124.png)

##### System.gc()

![image125](.\jvm-img\media1\image125.png)

![image126](.\jvm-img\media1\image126.png)

```
System.runFinalization();//强制调用失去引用的对象的finalize()方法
```

![image127](.\jvm-img\media1\image127.png)

![image128](.\jvm-img\media1\image128.png)

![image129](.\jvm-img\media1\image129.png)

##### 内存溢出

![image130](.\jvm-img\media1\image130.png)

![image131](.\jvm-img\media1\image131.png)

![image132](.\jvm-img\media1\image132.png)

##### 内存泄漏

![image133](.\jvm-img\media1\image133.png)

![image134](.\jvm-img\media1\image134.png)

###### 内存泄漏举例

![image135](.\jvm-img\media1\image135.png)



##### Stop The World

![image136](.\jvm-img\media1\image136.png)

###### 可达性分析

![](.\jvm-img\media1\image137.png)

![image138](.\jvm-img\media1\image138.png)

##### 垃圾回收的并行与并发

![image139](.\jvm-img\media1\image139.png)

![image140](.\jvm-img\media1\image140.png)

![image141](.\jvm-img\media1\image141.png)

![image142](.\jvm-img\media1\image142.png)

![image143](.\jvm-img\media1\image143.png)

![image144](.\jvm-img\media1\image144.png)

##### 安全点和安全区域

![image145](.\jvm-img\media1\image145.png)

![image146](.\jvm-img\media1\image146.png)

![image147](.\jvm-img\media1\image147.png)

马 驿站 安全点

![image148](.\jvm-img\media1\image148.png)

![image149](.\jvm-img\media1\image149.png)



![image150](.\jvm-img\media1\image150.png)

![image151](.\jvm-img\media1\image151.png)

引用关系存在的情况下

![image152](.\jvm-img\media1\image152.png)

##### 强引用-不回收

![image153](.\jvm-img\media1\image153.png)

![image154](.\jvm-img\media1\image154.png)

![image155](.\jvm-img\media1\image155.png)

![image156](.\jvm-img\media1\image156.png)

##### 软引用-不足即回收

![image157](.\jvm-img\media1\image157.png)

![image158](.\jvm-img\media1\image158.png)

![image159](.\jvm-img\media1\image159.png)

##### 弱引用-下次GC即回收

![image160](.\jvm-img\media1\image160.png)

![image161](.\jvm-img\media1\image161.png)

![image162](.\jvm-img\media1\image162.png)

```java
WeakHashMap:Entry继承自WeakReference，内存不足即回收，可用于缓存
```



##### 虚引用

![image163](.\jvm-img\media1\image163.png)

![image164](.\jvm-img\media1\image164.png)

![image165](.\jvm-img\media1\image165.png)

##### 终结器引用

![image166](.\jvm-img\media1\image166.png)

![image167](.\jvm-img\media1\image167.png)

### 垃圾回收器

![image168](.\jvm-img\media1\image168.png)

![image169](.\jvm-img\media1\image169.png)

![image170](.\jvm-img\media1\image170.png)

##### GC分类与性能指标

![image171](.\jvm-img\media1\image171.png)

![image172](.\jvm-img\media1\image172.png)

![image173](.\jvm-img\media1\image173.png)

![image174](.\jvm-img\media1\image174.png)

![image175](.\jvm-img\media1\image175.png)

![image176](.\jvm-img\media1\image176.png)



压缩式垃圾回收：再分配对象空间使用指针碰撞

非压缩式垃圾回收：再分配对象空间使用空闲列表

![image177](.\jvm-img\media1\image177.png)

###### 吞吐量和暂停时间

![image178](.\jvm-img\media1\image178.png)

目前内存越来越大 吞吐量越来越大 暂停时间却越来越大  现在要优化的主要就是暂停时间和吞吐量

![image179](.\jvm-img\media1\image179.png)

![image180](.\jvm-img\media1\image180.png)

![image181](.\jvm-img\media1\image181.png)

![image182](.\jvm-img\media1\image182.png)

##### 不同垃圾收集器概述

![image183](.\jvm-img\media1\image183.png)

![image184](.\jvm-img\media1\image184.png)

![image185](.\jvm-img\media1\image185.png)

CMS第一款并发GC 目前jdk已经移除

ZGC未来可期

ParallelGC jdk8默认

G1 jdk9默认

![image186](.\jvm-img\media1\image186.png)

![image187](.\jvm-img\media1\image187.png)

![image188](.\jvm-img\media1\image188.png)

![image189](.\jvm-img\media1\image189.png)

###### 组合关系

![image190](.\jvm-img\media1\image190.png)

![image191](.\jvm-img\media1\image191.png)

![image192](.\jvm-img\media1\image192.png)

###### 查看默认的GC

```java
*  -XX:+PrintCommandLineFlags 查看命令行相关参数，包含使用的垃圾收集器
*
*  -XX:+UseSerialGC:表明新生代使用Serial GC ，同时老年代使用Serial Old GC
*
*  -XX:+UseParNewGC：标明新生代使用ParNew GC
*
*  jdk8默认
*  -XX:+UseParallelGC:表明新生代使用Parallel GC
*  -XX:+UseParallelOldGC : 表明老年代使用 Parallel Old GC
*  说明：二者可以相互激活
*
*  -XX:+UseConcMarkSweepGC：表明老年代使用CMS GC。同时，年轻代会触发对ParNew 的使用
*
*  jdk9默认
*  -XX:+UseG1GC
```

![image193](.\jvm-img\media1\image193.png)

##### Serial回收器：串行回收

##### ![image194](.\jvm-img\media1\image194.png)

###### 新生代：复制算法

###### 老年代：标记-压缩

![image195](.\jvm-img\media1\image195.png)

![image196](.\jvm-img\media1\image196.png)

![image197](.\jvm-img\media1\image197.png)

![image198](.\jvm-img\media1\image198.png)

##### ParNew回收器：并行回收

##### ![image199](.\jvm-img\media1\image199.png)

###### 新生代：复制算法

![image200](.\jvm-img\media1\image200.png)

![image201](.\jvm-img\media1\image201.png)

多cpu（多核）

![image202](.\jvm-img\media1\image202.png)

![image203](.\jvm-img\media1\image203.png)

##### **Parallel回收器：吞吐量优先**

自成一派

![image204](.\jvm-img\media1\image204.png)

![image205](.\jvm-img\media1\image205.png)

###### 高吞吐量适合再后台运算而且不需要太多交互的任务

###### 新生代：复制算法

###### 老年代：标记-压缩

![image206](.\jvm-img\media1\image206.png)

服务器如果还使用serial old将拖累高性能服务器配置

![image207](.\jvm-img\media1\image207.png)

![image208](.\jvm-img\media1\image208.png)

###### 参数设置

![image209](.\jvm-img\media1\image209.png)

![image210](.\jvm-img\media1\image210.png)

默认开启

![image211](.\jvm-img\media1\image211.png)

##### CMS回收器：低延迟

![image212](.\jvm-img\media1\image212.png)

###### 低延迟适合互联网站或B/S系统的服务端上

![image213](.\jvm-img\media1\image213.png)

###### 老年代：标记-清除

![image214](.\jvm-img\media1\image214.png)

![image215](.\jvm-img\media1\image215.png)

初始标记和重新标记都有STW

###### 工作原理(重要)

![image216](.\jvm-img\media1\image216.png)

![image217](.\jvm-img\media1\image217.png)

![image218](.\jvm-img\media1\image218.png)

![image219](.\jvm-img\media1\image219.png)

###### 为什么不进行标记-压缩？

![image220](.\jvm-img\media1\image220.png)

###### 优缺点(致命缺点)

![image221](.\jvm-img\media1\image221.png)

![image222](.\jvm-img\media1\image222.png)

![image223](.\jvm-img\media1\image223.png)

ParallelGCThread默认值为CPU个数(核)

![image224](.\jvm-img\media1\image224.png)

![image225](.\jvm-img\media1\image225.png)

##### G1回收器：区域化分代式

![image226](.\jvm-img\media1\image226.png)

###### 区域分代化概述

![image227](.\jvm-img\media1\image227.png)

![image228](.\jvm-img\media1\image228.png)

![image229](.\jvm-img\media1\image229.png)

###### 优势

![image230](.\jvm-img\media1\image230.png)

![image231](.\jvm-img\media1\image231.png)

![image232](.\jvm-img\media1\image232.png)

![image233](.\jvm-img\media1\image233.png)

![image234](.\jvm-img\media1\image234.png)

吞吐量 （m-n）/m

###### 缺点

![image235](.\jvm-img\media1\image235.png)

###### 参数设置          

![image236](.\jvm-img\media1\image236.png)

2.  1-32M  ->  2-64G    2048个区域

如果指定堆大小 和 region大小 则区域个数也会变化

  3.95%达到  200ms到300ms都属于正常

如果设置的时间很短，如20ms，则gc从优先列表中挑选的region个数就非常少。如果应用程序占用内存速度快，这里回收的又少，将触发失败保护机制：Full GC。

所以不要一昧的少

4.并行STW工作线程

5.并发标记的线程数，数量需求没4多

6.触发GC阈值

###### G1回收器常用操作步骤

![image237](.\jvm-img\media1\image237.png)

###### 使用场景

![image238](.\jvm-img\media1\image238.png)

###### Region使用介绍

![image239](.\jvm-img\media1\image239.png)

![image240](.\jvm-img\media1\image240.png)

当对象大于0.5个region时 放到H

![image241](.\jvm-img\media1\image241.png)

![image242](.\jvm-img\media1\image242.png)

###### 回收过程

![image243](.\jvm-img\media1\image243.png)

混合回收=年轻代gc+老年代gc

所以 三个阶段都有年轻代gc

![image244](.\jvm-img\media1\image244.png)



![image245](.\jvm-img\media1\image245.png)

独占式 = STW

前面讲参数设置有触发并发GC的周期阈值设置 默认45% 这里指的就是并发标记+混合回收

###### Remembered Set

G1的额外开销之一

问题如：老年代中的对象指向新生代中的对象？

![image246](.\jvm-img\media1\image246.png)

对一个引用类型数据(GC Roots)写与引用类型数据存在不同region区域的对象时，通过卡表把该对象的相关引用信息记录到其所在region的记忆集中

卡表就是记忆集的实现。

![image247](.\jvm-img\media1\image247.png)

###### 回收过程1:年轻代GC

![image248](.\jvm-img\media1\image248.png)

![image249](.\jvm-img\media1\image249.png)

复制到to区，满足条件的复制到old区

不用担心年轻代对象指向老年代对象，因为混合回收需要进行年轻代GC，有GC Roots判断

需要担心老年代对象指向新生代对象。

![image250](.\jvm-img\media1\image250.png)

假设左object在老年代 右object在新生代 

![image251](.\jvm-img\media1\image251.png)

###### 回收过程2:并发标记

![image252](.\jvm-img\media1\image252.png)

###### 回收过程3:混合回收

![image253](.\jvm-img\media1\image253.png)

复制算法 无碎片

![image254](.\jvm-img\media1\image254.png)

###### Full GC、补充与优化

![image255](.\jvm-img\media1\image255.png)

![image256](.\jvm-img\media1\image256.png)

![image257](.\jvm-img\media1\image257.png)

##### 垃圾回收总结

![image258](.\jvm-img\media1\image258.png)

![image259](.\jvm-img\media1\image259.png)

![image260](.\jvm-img\media1\image260.png)

![image261](.\jvm-img\media1\image261.png)

![image262](.\jvm-img\media1\image262.png)

![image263](.\jvm-img\media1\image263.png)

![image264](.\jvm-img\media1\image264.png)

![image265](.\jvm-img\media1\image265.png)

#### 日志分析

![image266](.\jvm-img\media1\image266.png)

###### 参数

![image267](.\jvm-img\media1\image267.png)

###### 演示分析

![image268](.\jvm-img\media1\image268.png)

![image269](.\jvm-img\media1\image269.png)

![image270](.\jvm-img\media1\image270.png)

![image271](.\jvm-img\media1\image271.png)

![image272](.\jvm-img\media1\image272.png)

![image273](.\jvm-img\media1\image273.png)

![image274](.\jvm-img\media1\image274.png)

![image275](.\jvm-img\media1\image275.png)

![image276](.\jvm-img\media1\image276.png)

这里用的是jdk7为PSPermGen,

jdk8为MetaSpace

![image277](.\jvm-img\media1\image277.png)

![image278](.\jvm-img\media1\image278.png)

```
     * jdk7
     * 新生代有8m  一开始 前面三个数组在新生代 在第四个数组创建时，新生代空间不够 进行GC
     * 将前面三个数组移到老年代 再把第四个数组创建到新生代
     * jdk8
     * 新生代有8m  一开始 前面三个数组在新生代 在第四个数组创建时，属于大对象，直接放入老年区
```

![image279](.\jvm-img\media1\image279.png)

![image280](.\jvm-img\media1\image280.png)

##### 垃圾回收器的新发展

![image281](.\jvm-img\media1\image281.png)

![image282](.\jvm-img\media1\image282.png)

![image283](.\jvm-img\media1\image283.png)

![image284](.\jvm-img\media1\image284.png)

![image285](.\jvm-img\media1\image285.png)

![image286](.\jvm-img\media1\image286.png)

![image287](.\jvm-img\media1\image287.png)

![image288](.\jvm-img\media1\image288.png)

![image289](.\jvm-img\media1\image289.png)

![image290](.\jvm-img\media1\image290.png)

![image291](.\jvm-img\media1\image291.png)

![image292](.\jvm-img\media1\image292.png)

![image293](.\jvm-img\media1\image293.png)

![image294](.\jvm-img\media1\image294.png)

![image295](.\jvm-img\media1\image295.png)

![image296](.\jvm-img\media1\image296.png)

![image297](.\jvm-img\media1\image297.png)

![image298](.\jvm-img\media1\image298.png)

