# 容器化

物理机时代 --> 虚拟机时代  -->  容器化时代(轻量级VM, 灵活)

![image-20211207215323989](picture/image-20211207215323989.png)

阿里云腾讯云根据虚拟机+容器化构建云原生

镜像: 镜像是文件,是只读的,提供了运行程序完整的数据,是应用程序的"集装箱"

容器: 镜像的运行环境,迷你的"linux操作系统",由Docker负责创建,容器之间彼此隔离

## Docker一条龙

使用Centos7下Docker发布Nginx+Tomcat+MySQL项目

![image-20211208002233892](picture/image-20211208002233892.png)

1.安装Docker

```
#安装底层工具
sudo yum install ‐y yum‐utils device‐mapper‐persistent‐data lvm2
#加入阿里云yum仓库提速docker下载过程
sudo yum‐config‐manager ‐‐add‐repo http://mirrors.aliyun.com/docker‐ce/linux/centos/docker‐ce.repo
#更新仓库的源信息
sudo yum makecache fast
#下载docker安装
sudo yum ‐y install docker‐ce
#启动docker服务
sudo service docker start
#显示docker客户端和服务端信息(docker引擎) ,版本对应兼容性更好
docker version
```

2.aliyun加速镜像下载

https://cr.console.aliyun.com/cn-shanghai/instances/mirrors  --> 镜像加速器

```
sudo mkdir ‐p /etc/docker
sudo tee /etc/docker/daemon.json <<‐'EOF'
{
"registry‐mirrors": ["https://fskvstob.mirror.aliyuncs.com"]
}
EOF
sudo systemctl daemon‐reload
sudo systemctl restart docker
```

3.创建虚拟网段(重要)

```
#创建名称为my-bridge的网段
docker network create ‐d bridge my‐bridge
```

4.构建mysql应用实例

> 镜像仓库 [dockerhub ](hub.docker.com) 
>
> 搜索镜像 --> tags --> 选择版本 --> 查看使用方式 

将数据库初始化sql文件加入宿主机器的/usr/local/lzf/sql下, 用于初始化mysql;

新建文件夹 /usr/local/lzf/data , 用于存放mysql数据.

```
docker run \
‐p 3306:3306 \
‐‐network my‐bridge \
‐‐name db \
‐v /usr/local/lzf/sql:/docker‐entrypoint‐initdb.d \
‐v /usr/local/lzf/data:/var/lib/mysql \
‐e MYSQL_ROOT_PASSWORD=root \
‐d mysql:5.7
```

命令解析:

- docker run ...mysql 5.7 代表创建并自动运行mysql 5.7 容器，如果宿主机没有 mysql 5.7镜像，则自动会从dockerhub进行下载。
- --name db是容器的名字(**重要**)
- -p 3306:3306 代表将容器内部MySQL5.7 映射到宿主机的3306端口，这样才可 以从外界进行访问(左边为宿主机端口)
- --network my-bridge 代表将db容器加入到my-bridge虚拟网段，这样才可以 和其他容器通信(**非常关键**)
- -v /usr/local/lzf/sql ... 代表将宿主机的sql目录挂载到容器内的dockerentrypoint-initdb.d，根据dockerhub 的描述，放入docker-entrypoint-initdb.d 目录下的SQL文件会在MySQL容器创建后自动执行，完成数据初始化任务。(项目需要的数据)
- -v /usr/local/lzf/data ...同样是挂载，因为容器很容易创建或迁移，如果将 MySQL数据文件保存在容器内部，容器销毁数据就会丢失，因此同样使用-v命令将 容器内产生的数据文件挂载到宿主机的data目录下，这样即使容器销毁数据也不会丢失。(**重要**)
- -e MYSQL_ROOT_PASSWORD=root ,mysql容器要求的环境参数，说明创建 容器时默认数据库root密码为root.
- -d 代表采用后台模式运行，不加-d则采用前台独占方式运行

> 执行完后可以通过与宿主机ip+映射到宿主机的端口访问mysql , 账号密码为root , 并且存在sql文件初始化后的数据.
>
> 并且随着mysql启动运行, 可以查看宿主机的 /usr/local/lzf/data 目录下已经有了mysql的相关文件 

![image-20211208002307309](picture/image-20211208002307309.png)

5.构建tomcat应用 , 如springboot打包后的jar包(自带tomcat) : app.jar

在项目中, 配置了mysql数据源信息为:

```
spring:
  datasource:
    driver-class-name: com.mysql.jdbc.Driver
    url: jdbc:mysql://db:3306/app?useUnicode=true&useSSL=false
    username: root
    password: root
```

> 注意, 数据源url处的ip填的是mysql容器名

这里将app.jar 放在 /usr/local/lzf/app下 ,并且创建app目录下同级文件dockerfile文件(重要)

```
FROM openjdk:11
ADD ./app /usr/local/lzf
WORKDIR /usr/local/lzf
CMD ["java","‐jar", "bsbdj.jar"]
```

FROM: 指定该镜像基于哪个原始镜像进行扩展( 这里为jdk环境 ) ,  基准镜像

ADD: 将指定宿主机目录中的文件 复制到 镜像指定目录下 (不存在则自动创建)  ,这里为拷贝jar

WORKDIR:  在镜像内部切换工作目录 (相当于 CD)  ,这里为切换到jar所在的目录工作

CMD:  最后执行的命令.

```
#切换到dockerfile所在目录
cd /usr/local/lzf
#自动读取dockerfile,并在宿主机生成镜像 ,镜像名字为lzf/app:1.0 , 最后.代表当前目录(即dockerfile所在目录)
docker build ‐t lzf/app:1.0 .
#验证
docker images
```

**创建app容器**:

```
docker run ‐‐name app1 \
‐‐network my‐bridge \
‐p 8080:8080 \
‐d lzf/app:1.0
docker run ‐‐name app2 \
‐‐network my‐bridge \
‐p 8081:8080 \
‐d lzf/app:1.0
docker run ‐‐name app3 \
‐‐network my‐bridge \
‐p 8082:8080 \
‐d lzf/app:1.0
```

> 这里给3个app指定了网段和容器名 , 而在app的配置文件中配置的url主机为db ,且mysql镜像容器的容器名为db , 3个app和db在同一网段.  
>
> 这里不用关心ip地址, 只要填写容器名即可, docker 会自动帮我们分配管理ip , 这样做的好处为在测试开发的时候只要约定指定的容器名 , 运维发布只需要关心约定的容器名.

> 执行完成后可以通过宿主ip:8080访问应用了

6.构建nginx应用 , 实现反向代理服务器 ,配置负载均衡策略

```
docker run ‐‐name nginx \
‐v /usr/local/lzf/nginx/nginx.conf:/etc/nginx/nginx.conf \
‐‐network my‐bridge \
‐p 80:80 \
‐d nginx
```

> 挂载宿主机的nginx配置文件到容器 , 指定负载均衡

> 如果下载没有指定端口 , 默认添加lastest , 下载最新版

```
#后端服务器池
upstream lzf {
		server app1:8080 ;
		server app2:8080 ;
		server app3:8080 ;
	}

	server {
		#nginx通过80端口提供服务
		listen 80;
		#使用bsbdj服务器池进行后端处理
		location /{
			proxy_pass http://lzf; 
			proxy_set_header Host $host;
			proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
		}
	}
```

> 注意: nginx配置里 , 映射的也是容器名 , 且端口为容器内的端口 ,而不是宿主机映射的端口.
>
> docker中容器间通信一律指定容器名+容器内部端口的方式

现在访问宿主机ip , 就能以负载均衡的方式访问应用了.





> docker命令

```
#查看当前运行的容器
docker ps
#查看当前已下载的镜像
docker images
#查看网段内容器信息
docker network inspect my‐bridge删除所有容器与镜像
```

删除所有容器与镜像

```
docker rm ‐f nginx
docker rm ‐f app1
docker rm ‐f app2
docker rm ‐f app3
docker rm ‐f db
docker rmi ‐f itlaoqi/bsbdj:1.0
docker rmi ‐f mysql:5.7
docker rmi ‐f openjdk:11
docker network rm my‐bridge
```



> 为了解决多机部署 ,可加入k8s进行分发部署



# JVM

jvm选项规则

1. `java -version` 标准选项 ,任何版本JVM/任何平台都可使用
2. `java -Xms10m` 非标准选项, 部分版本(基本主流都)识别
3. `java -XX:+PrintGCDetails` 不稳定参数, 不同JVM有差异, 随时可能会被移除.

> PS: +代表开启/-代表关闭

## 调优

**调优建议**

- 大多数情况JVM生产环境考虑调整**①最大堆和最小堆大小②GC收集器③新生代大小** 三方面
- 在没有全面监控和收集性能数据之前, 调优都是扯淡
- 99%情况是代码出现了问题, 而不是JVM参数设置不对

### 优化选项

1.jdk1.8有限使用G1收集器, 摆脱各种选项烦恼(jdk9产生 , jdk1.8内置)

```
Java -jar -XX:+UseG1GC -Xms2G -Xmx2G -Xss256k
-XX:MaxGCPauseMillis=300 -Xloggc:/logs/gc.log 
-XX:+PrintGCTimeStamps -XX:+PrintGCDetails  xxx.jar
```

解析:

1) `-Xms`与`-Xmx`配置值相同 , 当程序运行时 ,自动把空间直接分配 , **减少内存交换**(动态内存调整);
2) 而具体评估`-Xmx`值得方法为: 第一次起始设置大一点, 跟踪监控日志 , 调整为堆峰值的*2 或 *3即可;
3) `-Xloggc`配置了监控日志地址, `-XX:+PrintGCTimeStamps`表示打印GC具体时间,  `-XX:+PrintGCDetails`表示打印GC详细信息;
4) `-XX:MaxGCPauseMillis`表示最多300毫秒STW时间 ( G1独有, 设置STW时间) , 200~500区间 ,增大可减少GC次数, 提高吞吐;
5) 虚拟机栈默认采用一个线程分配1M空间, 因为局部变量表不保存对象(指针) ,多余(不涉及复杂业务运算)且内存压力大, 采用`-Xss`设置为128k/256k即可, 不建议超过256k , 如超过考虑其他优化, 特别是代码;
6) G1一般不设置新生代大小 , G1的新生代是动态调整的.



# Zookeeper



## Apache Curator

Apache Curator(https://curator.apache.org/)是一个比较完善的ZooKeeper客户端框架，通 过封装的一套高级API 简化了ZooKeeper的操作。

通过查看官方文档，可以发现Curator主要解决 了三类问题： 

1.封装ZooKeeper client与ZooKeeper server之间的连接处理 

2.提供了一套Fluent风格的操作API

3.提供ZooKeeper各种应用场景(recipe， 比如：分布式锁服务、集群领导选举、共享 计数器、缓存机制、分布式队列等)的抽象封装



## 安装

**centos手动安装**

```
# Zookeeper依赖JVM运行，先安装JDK
2 yum ‐y install java‐1.8.0‐openjdk
3 cd /usr/local
4 # 下载Zookeeper Tar
5 wget http://dlcdn.apache.org/zookeeper/zookeeper‐3.7.0/apache‐zookeeper‐
3.7.0‐bin.tar.gz
6 tar ‐zxvf apache‐zookeeper‐3.7.0‐bin.tar.gz
7 cd ./apache‐zookeeper‐3.7.0‐bin/
8 cd ./conf/
9 # 按实例文件复制重命名到 zoo.cfg
10 cp zoo_sample.cfg zoo.cfg
11 cd ../bin
12 # 启动Zookeeper
13 ./zkServer.sh start
14 # 防火墙放行 2181 是Zookeeper服务端口 8080是Web命令端口（可选）
15 firewall‐cmd ‐‐zone=public ‐‐add‐port=2181/tcp ‐‐permanent
16 firewall‐cmd ‐‐zone=public ‐‐add‐port=8080/tcp ‐‐permanent
17 # 重载防火墙
18 firewall‐cmd ‐‐reload
```

访问 8080/commands/stat 成功即表示安装成功

**Docker安装**

```
docker run ‐‐privileged=true ‐d ‐‐name zookeeper ‐p 2181:2181 ‐p 8080:808
0 ‐d zookeeper:latest
2 firewall‐cmd ‐‐zone=public ‐‐add‐port=2181/tcp ‐‐permanent
3 firewall‐cmd ‐‐zone=public ‐‐add‐port=8080/tcp ‐‐permanent
4 firewall‐cmd ‐‐reload
```

**IDEA插件Zoolytic**

安装完成后在View->Tool Windows->zoolytic

输入ip:2181连接zk















































# 发布部署

应用发布与持续继承

![CI](picture/CI.png)

## 全量发布

全量发布分为蓝绿发布和红黑发布

假设应用程序需要在运行的过程中进行升级 ,  用户通过网关(nginx ,spring cloud gateway等)进行访问服务集群. 

**蓝绿发布** 指的是在部署两个相同的集群A和B的情况下 .

升级时先断开集群A , 所有请求落到集群B , 之后集群A开始进行升级, 待升级完成后 ,网关配置接入集群A 并同时断开集群B.

**红黑发布** 指的是在已有一个集群(红)的情况下.

升级时,开辟一个全新集群部署新版本(黑) , 当部署完成后 , 网关接入新集群 , 下掉旧集群(完全释放资源).



**红黑发布的优势**:充分利用了云计算的弹性伸缩优势 ( 适合容器化&虚拟化部署 ,资源需要的时候才进行分配 )，从而获得了两个收益：一是，简化了流程(不需要频繁设置网关)；二是，避免了在升级的过程中，由于只有一半的服务器提供服务，而可能导致的系统过载问题。



## 增量发布

灰度发布属于增量发布 , 服务升级的过程中，新旧版本会同时为用户提供服务。

比如通过网关先将部分请求接入新版本集群, 可视情况逐渐提升比例最后将旧版本集群替换掉.

> 如果新旧版本无法协同工作,
>
> 1.可采用红黑发布
>
> 2.可考虑独立部署新旧版本数据源, 但数据源之间的同步等问题考验架构师和DBA水平.

**灰度发布带来的挑战**

1.考虑数据库变更对旧版本的兼容性影响

例如：某数据表有ab两个字段, 在旧应用中存在SQL： insert into t values(‘a’,’b’);

但v1.1版本中在数据表增加了c字段，就版本运行就会报错

因此在考虑未来灰度发布的情况，要求团队成员写SQL必须明确字段( 实际上都是这么做的)

insert into t(a,b) values(‘a’,’b’);

> TIPS:任何删除、更新字段信息的操作都要格外谨慎

2.灰度发布用户群的选择

不能直接采用类似于Nginx的权重Weight

会导致一个用户不同请求在新旧版本间反复横跳，出现无法预期的Bug

可通过Nginx + Lua脚本化

基于IP或者UserID等用户稳定特性然后Hash取模来决定访问新旧版本

3.什么时候才可以提升分配比例

在发布过程中，我们应该注意监测用户请求失败率、用户请求处理时长和异常出现数量这几个信息，以保证快速发现问题并及时回滚。

在灰度发布的时候，可以部署相对较小的集群，让集群保持在高压力确认新版应用的性能情况，之后再酌情进行扩容。

> TIPS: 服务监控可使用简单粗暴的SkyWalking



# 硬件对性能的影响

**CPU**

> mysql不支持多cpu对同一SQL并发处理 , 5.6以后版本对多核cpu进行了优化

并发比较高的场景CPU的数量比频率重要;

CPU密集性场景和复杂SQL则频率越高越好



**内存**

> innonDB会把索引和数据都缓存到内存

理想选择时服务器内存大于数据总量(不现实)

内存频率越高处理速度越快

内存总量小要合理组织热点数据, 保证内存覆盖

内存对写操作也有重要的性能影响



**硬盘**

优先SSD(服务器专业SSD比机械硬盘贵好几倍)

## RAID

RAID是磁盘冗余队列的简称. 简单来说RAID的作用就是可以把多个容量较小的磁盘组成一组容量更大的磁盘, 并提供数据冗余来保证数据完整性的技术

**RAID 0** 是最早出现的RAID模式 ,也称为数据条带 . 是组建磁盘阵列中最简单的一种形式 ,  只需要2块及以上的硬盘即可,  成本低 , 可以提高整个磁盘的性能和吞吐量.

但RAIO 0没有提供冗余(备份) 或错误修复能力 .

**RAID 1** 又称磁盘镜像, 原理是把一个磁盘的数据镜像到另一个磁盘上 , 也就是说数据在写入一块磁盘的同时 , 会在另一块闲置的磁盘上生成镜像文件, 在不影响性能的情况下最大限度的保证系统的可靠性和可修复性.

**RAID 5** 又称为分布式奇偶校验磁盘阵列

通过分布式奇偶校验块把数据分散到多个磁盘上 , 这样如果任何一盘数据失效 , 都可以奇偶校验块中重建 . 但是如果两块磁盘失效 ,则整个卷的数据都无法恢复.

**RAID 10** 又称为分片的镜像

它是对磁盘先做RAID 1 之后对两组RAID 1的磁盘再做RAID 0 , 所以读写都有良好的性能, 相对于RAID 5重建起来更简单 , 速度也更快

| 等级    | 特点             | 冗余 | 盘数 | 读   | 写             |
| ------- | ---------------- | ---- | ---- | ---- | -------------- |
| RAID 0  | 便宜,快速,危险   | 否   | N    | 快   | 慢             |
| RAID 1  | 高速度,简单,安全 | 有   | 2    | 快   | 快             |
| RAID 5  | 安全,成本折中    | 有   | N+1  | 快   | 取决于最慢的盘 |
| RAID 10 | 贵,高速,安全     | 有   | 2N   | 快   | 快             |

