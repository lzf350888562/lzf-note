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
