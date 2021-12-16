# 前端编译器

1.javac  bin目录下的javac.exe

javac HelloWorld.java

IDEA使用javac

HotSpot VM没有强制使用javac编译器，只要编译结果符合jvm规范都可以被jvm识别即可

2.Eclipse内置的ECJ(Eclipse Compiler for Java)编译器，为增量编译器

在Eclipse中，当开发人员编写完代码后，使用ctrl+S快捷键时，ECJ编译器所采取的编译方案是把未编译部分的源码逐行进行编译，而并非每次都全量编译，因此ECJ的效率比javac更加迅速与高效

Tomcat中同样也是使用ECJ编译器来编译jsp文件

<u>前端编译器并不会直接涉及编译优化等方面的技术，而是将这些具体优化细节移交给HotSpot的JIT编译器负责</u>

# class文件

经过前端编译器编译之后生成的字节码文件，是二进制的类文件，它的内容是jvm指令，而不像c/c++经由编译器直接生成机器码

## 字节码指令

java虚拟机的指令由一个字节长度的代表着某种特定操作含义的操作码(opcode)以及跟随其后的零至多个代表此操作所需参数的操作数(operand)所构成，虚拟机中许多指令并不包含操作数，只有一个操作码

操作码 （操作数）

## class文件格式

任何一个class文件都对应着唯一一个类或接口的定义信息，但实际上不一定以磁盘文件的形式存在，也可能通过网络传输反序列化到内存作为一个类

class文件中的数据项，无论是字节顺序还是数量，都被严格限定的。

class文件采用一种类似于C语言结构体的方式进行数据存储，这种结构中只有两种数据类型：无符号数和表。

1.无符号数属于基本的数据类型，以u1 u2 u4 u8来分别代表 1 2 4 8个字节的无符号数，无符号数可以用来描述数字、索引引用、数量值或按照UTF-8编码构成的字符串值

2.表是由多个无符号数或者其他表作为数据项构成的符合数据类型。所有的表的习惯以"_info"结尾，表用于描述由层次关系的复合结构的数据，整个class文件的本质就是一张表。由于表没有固定长度，所以通常会在其前面加上个数说明。

## class文件结构

见  Class字节码文件结构.md

```java
ClassFile {
    u4             magic;
    u2             minor_version;
    u2             major_version;
    u2             constant_pool_count;
    cp_info        constant_pool[constant_pool_count-1];
    u2             access_flags;
    u2             this_class;
    u2             super_class;
    u2             interfaces_count;
    u2             interfaces[interfaces_count];
    u2             fields_count;
    field_info     fields[fields_count];
    u2             methods_count;
    method_info    methods[methods_count];
    u2             attributes_count;
    attribute_info attributes[attributes_count];
}
```

二进制字节码解析：

源文件：中篇 chapter2.java1.Demo

解析文件：Demo字节码的解析.xls

**魔数**

4字节

唯一作用：确定这个文件是否为一个能被虚拟机接受的有效合法的class文件 即class文件的标识符

魔数值固定为0xCAFEFABE

使用魔数而不是扩展名进行识别主要是基于安全方面的考虑，因为扩展名可以被随意的改动

如果class文件不是以其固定值开头,jvm在进行文件校验的时候就会直接抛错误

Error:A JNI error has occurred, please check your installation and try again

Exception in thread "main" java.lang.ClassFormatError:Incompatible magic value 1885430635 in class file ...

**class文件的版本号**

4字节

前2个字节代表编译的副版本号(minor_version)

后2个字节代表编译的主版本号(major_version)

它们共同构成了class文件的格式版本号，具体见 class字节码文件结构.md

45开始  每个jdk大版本对应主版本号加一。

注意：不同版本的java编译器编译的class文件对应的版本不一样，目前，高版本的java虚拟机可以指向由低版本编译器生成的class文件，但是低版本的java虚拟机不能指向由高版本编译器生成的class文件。(向下兼容)

否则会抛出java.lang.UnsupportedClassVersionError  :........:Unsupported major.minor.version 

### 常量池

是class文件内容最丰富的区域之一，对class文件的字段和方法的解析有着至关重要的作用，是class文件的基石。

包括常量池的数量（2字节的常量池计数器）和若干常量池表项，容量技术从1开始

常量池计数器：由于常量池的数量不固定，所以需要两个字节来表示常量池容量计数值。从1开始，constant_pool_count=1表示常量池中由0个常量项

<u>常量池表项</u>中，用于存放编译期生成的各种<u>字面量</u>和<u>符号引用</u>，这部分内容将在类加载后进入方法区的运行时常量池（jdk8堆）中存放。常见的常量项见 class字节码文件结构.md

字面量的具体常量为：<u>文本字符串</u>和<u>声明为final的常量值</u>

符号引用：<u>类和接口的全限定名、字段的名称和描述符、方法的名称和描述符</u>

```java
* 全类名：com.atguigu.java1.Demo
* 全限定名：com/atguigu/java1/Demo      可加分号表示全限定名结束
```

描述符的作用是用来描述字段的数据类型、方法的参数列表和返回值，具体见 class字节码文件结构.md，方法toString 就会输出标志+权限定名

**符号引用与直接引用**

![](.\jvm-img\middle\符号引用与直接引用.jpg)

**常量类型与结构**

1.这14种表(或常量项结构)的共同的是：表的开始第一位是一个u1类型的标志位tag，代表当前常量类型。

2.CONSTANT_Utf8_info常量项时一种使用改进过的utf-8编码格式来存储诸如文字字符串、类或者接口的全限定名、字段或者方法的简单名称以及描述符等常量字符串信息。

3.这14种常量项还有一个特点是，其中13个常量项占用的字节固定，只有CONSTANT_Utf8_info占用字节不固定，其大小由length决定。因为从常量池放的内容可知，其存放的是字面量和符号引用，最终这些内容都会是一个字符串，这些字符串的大小是在编写程序时才确定。比如定义一个类,类名可以取长取短，所以在没编译前，大小不固定 ，编译后，通过utf-8编码，就可以知道其长度。

![](.\jvm-img\assets\1598773300484.png)

![1598773308492](.\jvm-img\assets\1598773308492.png)

### 访问标识

2字节 

具体的访问标记见 class字节码文件结构.md

多个访问标记 使用 | 组合

![](.\jvm-img\middle\访问标识.png)

### 类索引、父类索引、接口索引

这三项数据来确定这个类的继承关系，包括

this_class u2  存储对应常量项里的index

super_class u2 存储对应常量项里的index

interfaces_count u2  存

interfaces[interfaces_count] 每个u2  索引从0开始

### 字段表集合

包括字段计数器 u2  和 字段表

描述接口或类种声明的变量

字段的名字和字段的类型(描述符)都是无法固定的，只能引用常量池种的常量项来描述

字段表集合中不会列出从父类或者实现的接口中继承而来的字段，但有可能列出原本java代码中不存在的字段(如在内部类中为了保持对外部类的访问，会自动添加指向外部类实例的字段)。

<u>字段无法重载</u>

但是对于字节码的角度来讲，如果两个字段的描述符不一样，那字段重名就是合法的。但java语法不允许。

字段表中每个成员都必须时一个fields_info结构的数据项

![](.\jvm-img\middle\字段表.png)

字段访问标识见 class字节码文件结构.md

属性表集合，一个字段还可能拥有一些属性，用于存储更多的额外信息。比如初始化值（final）、一些注解信息等。属性个数存放在attribute_count中，属性具体内容存放在attributes数组中。

以常量属性为例，结构为

ConstantValue_attribute{

​	u2 attribute_name_index;

​    u4 attribute_length;

​    u2 constantValue_index;

}

对于常量属性而言， u4 attribute_length恒为2

### 方法表集合

methods：指向常量池索引的集合，它完整描述了每个方法的签名。

每一个method_info项都对应着一个类或接口中的方法信息，如修饰符，返回值，参数等。

如果方法不是抽象的或者native的，在字节码中就会体现出来。不包括从父类或接口继承的方法。并且methods可能出现由编译器自动添加的方法，如<clinit>

重载：与原方法具有相同的名称之外，必须拥有不同的方法中各个参数在常量池中的字段符号引用的集合，返回值不包含在特征签名之中。字节码文件允许在特征签名相同的情况下返回值不同，但java语法不允许。

![](.\jvm-img\middle\方法表.png)

描述符：返回值和参数类型

### 属性表集合

class文件所携带的辅助信息，比如该class文件的源文件的名称，以及任何带有RetentionPolicy.CLASS或者RetentionPolicy.RUNTIME的注解，这类信息通常被用于java虚拟机的验证和运行，以及java程序的调试，一般无需深入了解

此外，字段表、方法表都可以有自己的属性表，用于描述某些场景专有的信息。

**方法中的CODE属性**

```java
Code_attribute {
    u2 attribute_name_index;                 	 	属性名索引
    u4 attribute_length;							属性长度（从下一个位置到结束位置）
    u2 max_stack;									操作数栈深度的最大值
    u2 max_locals;									局部变量表所需的长度
    u4 code_length;									字节码指令的长度
    u1 code[code_length];							存储字节码指令（code_length个字节），操作数占2字节，所以带一个操作数的指令总共占3个字节，这就是为什么指令长度不协调的问题
    u2 exception_table_length;						异常表长度
    {   u2 start_pc;			
        u2 end_pc;
        u2 handler_pc;
        u2 catch_type;
    } exception_table[exception_table_length];
    u2 attributes_count;							属性集合计数器（Code属性中的属性,如LineNumberTable和LocalVariableTable）
    attribute_info attributes[attributes_count];	属性集合
}
```

**LineNumberTable**

```
LineNumberTable_attribute {
    u2 attribute_name_index;
    u4 attribute_length;
    u2 line_number_table_length;
    {   u2 start_pc;
        u2 line_number;	
    } line_number_table[line_number_table_length];
}
```

**LocalVariableTable**

```
LocalVariableTable_attribute {
    u2 attribute_name_index;
    u4 attribute_length;
    u2 local_variable_table_length;
    {   u2 start_pc;
        u2 length;
        u2 name_index;
        u2 descriptor_index;
        u2 index;
    } local_variable_table[local_variable_table_length];
}
```

**附加属性SourceFile**

额外属性表通用格式，指定源文件名

| 类型 | 名称                 | 数量             | 含义       |
| ---- | -------------------- | ---------------- | ---------- |
| u2   | attribute_name_index | 1                | 属性名索引 |
| u4   | attribute_length     | 1                | 属性长度   |
| u1   | info                 | attribute_length | 属性表     |

attribute_length为2

# javap

jdk自带的反解析工具，反解析出当前类对应的code区 javap -v xxx.class

javap -help  选项帮助

```shell
  -help  --help  -?                输出此用法消息
  -version                         版本信息,其实时当前javap所在jdk的版本信息，不是class在哪个jdk下生成的
  -public                          仅显示公共类和成员
  -protected                       显示受保护的/公共类和成员
  -package                         显示程序包/受保护的/公共类和成员 (默认)
  -p  -private                     显示所有类和成员
  -constants                       显示静态最终常量
```

```shell
	 -s                               输出内部类型签名
 	 -l                               输出行号和本地变量表
     -c                               对代码进行反汇编
     -v  -verbose                     输出附加信息(最详细，包括常量池，但不包括私有 可再加-p)
```

<u>最全  javap -v -p xxxx.class</u>

![](.\jvm-img\middle\javap总结.png)

# javac -g

解析字节码文件得到的信息中，有些信息需要使用javac编译成class文件时指定参数才能输出

比如 javac xx.java不会生成 对应的局部变量表等信息，如果你使用javac -g xx.java就可以生成所有相关信息了。

如果使用eclipse或idea，默认情况下会在编译时帮你生成局部变量表、指令和代码行偏移量映射表等信息的。

# 字节码指令集

如果不考虑异常处理，jvm的解释器可以使用下列伪代码当作最基本的执行模型来理解：

do{

自动计算pc寄存器的值加1；

根据pc寄存器的指定位置，从字节码流中取出操作码；

if(字节码存在操作数)从字节码流中取出操作数；

执行操作码所定义的操作；

​	}

## 数据类型

![](.\jvm-img\middle\字节码与数据类型.png)

![](.\jvm-img\middle\字节码与数据类型2.png)

## 指令分类

![](.\jvm-img\middle\指令分类.png)

### 加载与存储指令

![](.\jvm-img\middle\存储与加载指令.png)

iload_0和iload 0功能相同 将局部变量表索引0的数据压操作数栈

但前者1字节 后者3字节

### 常量入栈指令

![](.\jvm-img\middle\常量入栈指令.png)

ldc_w能接受16位参数，能支持的索引范围大于ldc，用于long和double

注意：

1.int i=3;      指令为iconst_3;

​    int j=6;       指令为bipush 6而不是iconst_6

​	int k=32768  指令为 ldc xxx

```java
int i = -1;  //iconst_m1
int a = 5;  //iconst_5
int b = 6;  //bipush 6
int c = 127;    //bipush 127
int d = 128;    // sipush 128
int e = 32767;  //sipush 32767
int f = 32768;  // ldc #7 <32768>
```

![](.\jvm-img\middle\常量入栈指令详细.png)

### 出栈装入局部变量表指令

![](.\jvm-img\middle\出栈装入局部变量表指令.png)

### 算术指令

![](.\jvm-img\middle\算术1.png)

![](.\jvm-img\middle\算术2.png)

![](.\jvm-img\middle\算术3.png)

自增直接在局部变量表里面操作 iinc 1 by 1

![](.\jvm-img\middle\算术4.png)

比较指令详细在条件跳转指令讲     

### 类型转换指令

**宽化类型转换 自动**

![](.\jvm-img\middle\宽化类型转换.png)

![](.\jvm-img\middle\宽化类型转换2.png)

**窄化类型转换 强制**

![](.\jvm-img\middle\窄化类型转换.png)

```
*  在多指令转换中  一般用i作中转站
```

![](.\jvm-img\middle\窄化类型转换2.png)

### 对象的创建与访问指令

**创建指令**

![](.\jvm-img\middle\创建指令.png)

**字段访问指令**

![](.\jvm-img\middle\字段访问指令1.png)

![](.\jvm-img\middle\字段访问指令2.png)

**数组创建指令**

![](.\jvm-img\middle\数组操作指令.png)

![](.\jvm-img\middle\数组访问指令2.png)

**类型检查指令**

![](.\jvm-img\middle\检查类型指令.png)

### 方法调用与返回指令

**方法调用**

![](.\jvm-img\middle\方法调用指令.png)

**方法返回**

![](.\jvm-img\middle\方法返回指令.png)

### 操作数栈管理指令

![](.\jvm-img\middle\操作数栈管理指令1.png)

![](.\jvm-img\middle\操作数栈管理指令2.png)

### 控制转移指令

**比较指令（前）**

![](.\jvm-img\middle\算术4.png)

**条件跳转指令**

![](.\jvm-img\middle\条件跳转指令.png)

**比较条件跳转指令**

![](.\jvm-img\middle\比较条件跳转指令.png)

**多条件分支跳转指令**

![](.\jvm-img\middle\多条件分支跳转指令.png)

**无条件跳转指令**

![](.\jvm-img\middle\无条件跳转.png)

### 异常处理指令

**抛出异常指令**

![](.\jvm-img\middle\抛出异常.png)

**处理异常与异常表**

![](.\jvm-img\middle\处理异常与异常表.png)

### 同步控制指令

monitor

**方法级的同步**

![](.\jvm-img\middle\方法级的同步1.png)

![](.\jvm-img\middle\方法级的同步2.png)

**方法内部一段指令序列的同步**

![](.\jvm-img\middle\指令同步1.png)

![](.\jvm-img\middle\指令同步2.png)

# 类的加载过程详解

![](.\jvm-img\middle\类的加载01.png)

## Loading加载阶段

### 加载完成的操作

![](.\jvm-img\middle\类的加载02.png)

### 二进制流的获取方式

![](.\jvm-img\middle\类的加载03.png)

### 类模型与Class实例的位置

![类的加载05](.\jvm-img\middle\类的加载05.png)

<img src=".\jvm-img\middle\类的加载04.png" style="zoom: 67%;" />

![](.\jvm-img\middle\类的加载06.png)

### 数组类的加载

![](.\jvm-img\middle\类的加载07.png)

## Linking链接阶段

### Verification验证

当类加载到系统后，就开始链接操作，验证时链接操作的第一步

它的目的时保证加载的字节码时合法，合理并且规范的

验证的步骤比较复杂，实际要验证的项目也很繁多，大体上虚拟机需要作以下检查，如图

![](.\jvm-img\middle\类的加载08.png)

![类的加载09](.\jvm-img\middle\类的加载09.png)

![](.\jvm-img\middle\类的加载10.png)

### Preparation准备

![](.\jvm-img\middle\类的加载11.png)

![](.\jvm-img\middle\类的加载12.png)

### Resolution解析

![](.\jvm-img\middle\类的加载13.png)

![](.\jvm-img\middle\类的加载14.png)

![类的加载15](.\jvm-img\middle\类的加载15.png)

## Initialization初始化阶段

![](.\jvm-img\middle\类的加载16.png)

由父及子，静态先行

![](.\jvm-img\middle\类的加载17.png)

什么情况下不生成clinit方法

![](.\jvm-img\middle\类的加载18.png)

### static与final的搭配问题

```java
* 说明：使用static + final修饰的字段的显式赋值的操作，到底是在哪个阶段进行的赋值？
* 情况1：在链接阶段的准备环节赋值
* 情况2：在初始化阶段<clinit>()中赋值
*
* 结论：
* 在链接阶段的准备环节赋值的情况：
* 1. 对于基本数据类型的字段来说，如果使用static final修饰，则显式赋值(直接赋值常量，而非调用方法）通常是在链接阶段的准备环节进行
* 2. 对于String来说，如果使用字面量的方式赋值，使用static final修饰的话，则显式赋值通常是在链接阶段的准备环节进行
*
* 在初始化阶段<clinit>()中赋值的情况：
* 排除上述的在准备环节赋值的情况之外的情况。
*
* 最终结论：使用static + final修饰，且显示赋值中不涉及到方法或构造器调用的基本数据类型或String类型字面量的显式赋值，是在链接阶段的准备环节进行。
```

### < clinit>()的线程安全性

![](.\jvm-img\middle\类的加载19.png)

### 类的初始化情况：主动使用与被动使用

如果针对代码，设置参数-XX:+TraceClassLoading，可以追踪类的加载信息并打印出来

![](.\jvm-img\middle\类的加载20.png)

![](.\jvm-img\middle\类的加载21.png)

![](.\jvm-img\middle\类的加载22.png)

## 类的Using使用

任何一个类型在使用之前都必须经历过完整的加载、链接和初始化三个类加载步骤，一旦一个类型成功经历过这3个步骤之后，便可使用了。

开发人员可以在程序中访问和调用它的静态类成员信息，比如静态字段和静态方法，或者使用new关键字为其创建对象实例

## 类的Unloading卸载

![](.\jvm-img\middle\类的加载23.png)

![](.\jvm-img\middle\类的加载24.png)

![](.\jvm-img\middle\类的加载25.png)

![](.\jvm-img\middle\类的加载26.png)

![](.\jvm-img\middle\类的加载27.png)

# 再谈类的加载器

![](.\jvm-img\middle\001.png)

## 类加载的分类

![](.\jvm-img\middle\002.png)

## 类加载的必要性

![](.\jvm-img\middle\003.png)

## 命名空间

![](.\jvm-img\middle\004.png)

## 类加载机制的基本特征

![](.\jvm-img\middle\005.png)

## 分类

![](.\jvm-img\middle\06.png)

非继承 而是包含关系

见上篇

![](.\jvm-img\middle\07.png)

## 测试不同类加载器

![](.\jvm-img\middle\08.png)

```
//关于数组类型的加载:使用的类的加载器与数组元素的类的加载器相同
```

数组类的class对象不是由类加载器去创建的，而是在java运行期jvm根据需要自动创建的，对于数组类的类加载器来说，是通过Class.getClassLoader()返回的，与数组中元素类型的类加载器是一样的；如果数组中元素类型是基本数据类型，数组类是没有类加载器的。

## ClassLoader源码解析

![](.\jvm-img\middle\09.png)

![](.\jvm-img\middle\10.png)

![](.\jvm-img\middle\12.png)

![](.\jvm-img\middle\13.png)

### loadClass()测试

![](.\jvm-img\middle\11.png)

```
//name="com.atguig.java.User" resolve=true-加载class的同时进行解析操作 这里为false
protected Class<?> loadClass(String name, boolean resolve)
    throws ClassNotFoundException
{
    synchronized (getClassLoadingLock(name)) { //同步操作，保证只能加载一次
        // First, check if the class has already been loaded
        //首先 在缓存中判断是否已经加载同名的类
        Class<?> c = findLoadedClass(name);
        if (c == null) {
            long t0 = System.nanoTime();
            try {
            	//当前类(这里为App)的父类加载器
                if (parent != null) {
                	//如果存在父类加载器，则调用父类加载器进行加载
                    c = parent.loadClass(name, false);
                } else {  //父类加载器为Bootstrap
                    c = findBootstrapClassOrNull(name);
                }
            } catch (ClassNotFoundException e) {
                // ClassNotFoundException thrown if class not found
                // from the non-null parent class loader
            }

			//当前类的父类加载器未加载此类 当前类的引导加载器未加载此类
            if (c == null) { 
                // If still not found, then invoke findClass in order
                // to find the class.
                //调用当前ClassLoader的findClass()
                long t1 = System.nanoTime();
                c = findClass(name);

                // this is the defining class loader; record the stats
                sun.misc.PerfCounter.getParentDelegationTime().addTime(t1 - t0);
                sun.misc.PerfCounter.getFindClassTime().addElapsedTimeFrom(t1);
                sun.misc.PerfCounter.getFindClasses().increment();
            }
        }
        if (resolve) {  //是否进行解析操作
            resolveClass(c);
        }
        return c;
    }
}
```

### SecureClassLoader与URLClassLoader

![](.\jvm-img\middle\15.png)

![](.\jvm-img\middle\16.png)

### ExtClassLoader与AppClassLoader

![](.\jvm-img\middle\17.png)

![18](.\jvm-img\middle\18.png)

### Class.forName()与ClassLoader.loadClass()

![](.\jvm-img\middle\19.png)

Class.forName为主动使用 ，会调用<clinit>

ClassLoader.loadClass为被动使用 ，不会调用<clinit>

## 双亲委派模型

### 定义与本质

![](.\jvm-img\middle\20.png)

### 优势与劣势

![](.\jvm-img\middle\21.png)

![](.\jvm-img\middle\22.png)

![](.\jvm-img\middle\23.png)

### 破坏双亲委派机制

**一  jdk1.2以前**

![](.\jvm-img\middle\24.png)

**二 线程上下文类加载器**

![](.\jvm-img\middle\25.png)

![](.\jvm-img\middle\26.png)

![](.\jvm-img\middle\27.png)

**三 用户追求**

![](.\jvm-img\middle\28.png)

![](.\jvm-img\middle\29.png)

### 热替换的实现

![](.\jvm-img\middle\30.png)

![31](.\jvm-img\middle\31.png)

## 沙箱安全机制

![](.\jvm-img\middle\32.png)

**jdk1.0**

![](.\jvm-img\middle\34.png)

**jdk1.1**

![](.\jvm-img\middle\35.png)

**jdk1.2**

![](.\jvm-img\middle\37.png)

![](.\jvm-img\middle\36.png)

**jdk1.6**

![](.\jvm-img\middle\33.png)

## 自定义类加载器

注意：在一般情况下，使用不同的类加载器去加载不同的功能模块，会提高应用程序的安全性。但是，如果涉及java类型转换，则加载器反而容易产生不美好的事情。在作java类型转换时，只有两个类型都是由同一个加载器所加载，才能进行类型转换，否则转换时会发生异常。

**实现方式**

![](.\jvm-img\middle\14.png)

说明：

1.其父类加载器是系统类加载器

2.jvm中的所有类加载都会使用java.lang.ClassLoader.loadClass(String)接口(自定义类加载器并重写java.lang.ClassLoader.loadClass(String)接口的除外)，连jdk的核心类库也不能例外

## java9新特性

![](.\jvm-img\middle\39.png)

![](.\jvm-img\middle\40.png)

![](.\jvm-img\middle\41.png)

![](.\jvm-img\middle\42.png)

![](.\jvm-img\middle\43.png)

java9之前的classloader：

- **bootstrap classloader加载rt.jar，jre/lib/endorsed**
- **ext classloader加载jre/lib/ext**
- **application classloader加载-cp指定的类**

java9以及之后的classloader：

在java模块化系统明确规定了三个类加载器各自加载的模块。

- **bootstrap classloader(启动类加载器)负责加载的lib/modules(模块)**

  java.base                         java.security.sasl

  java.datatransfer           java.xml

  java.desktop                   jdk.httpserver

  java.instrument             jdk.internal.vm.ci

  java.logging                     jdk.management

  java.management          jdk.management.agent

  java.management.rmi    jdk.naming.rmi

  java.naming                      jdk.net

  java.prefs                           jdk.sctp

  java.rmi                              jdk.unsupported

- **ext classloader更名为platform classloader(平台类加载器)负责加载lib/modules**

  java.activation*            jdk.accessibility

  java.compiler*              jdk.charsets

  java.corba*                    jdk.crypto.cryptoki

  java.scripting                 jdk.crypto.ec

  java.se                             jdk.dynalink

  java.se.ee                        jdk.incubator.httpclient

  java.security.jgss            jdk.internal.vm.compiler*

  java.smartcardio            jdk.jsobject

  java.sql                            jdk.localedata

  java.sql.rowset               jdk.naming.dns

  java.transaction*           jdk.scripting.nashorn

  java.xml.bind*                jdk.security.auth

  java.xml.crypto               jdk.security.jgss

  java.xml.ws*                    jdk.xml.dom

  java.xml.ws.annotation*     jdk.zipfs

- **application classloader(应用程序类加载器)负责加载-cp，-mp指定的类** 

  jdk.aot                         jdk.jdeps
  jdk.attach                    jdk.jdi
  jdk.compiler                jdk.jdwp.agent
  jdk.editpad                   jdk.jlink
  jdk.hotspot.agent       jdk.jshell
  jdk.internal.ed             jdk.jstatd
  jdk.internal.jvmstat     jdk.pack
  jdk.internal.le               jdk.policytool
  jdk.internal.opt            jdk.rmic
  jdk.ja rtool                    jdk.scripting.nashorn.shell
  jdk.javadoc                    jdk.xml.bind*
  jdk.jcmd                         jdk.xml.ws*
  jdk.jconsole  