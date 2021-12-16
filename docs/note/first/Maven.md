# 标签

## Classfilier

```
<dependency>  
    <groupId>net.sf.json-lib</groupId>   
    <artifactId>json-lib</artifactId>   
    <version>2.2.2</version>  
    <classifier>jdk13</classifier>    
</dependency>
```



# maven私服

## 私服说明 

maven仓库分为本地仓库和远程仓库，而远程仓库又分为maven中央仓库、其他远程仓库和私服（私有服务器）。其中，中央仓库是由maven官方提供的，而私服就需要我们自己搭建了。

maven私服就是公司局域网内的maven远程仓库，每个员工的电脑上安装maven软件并且连接maven私服，程序员可以将自己开发的项目打成jar并发布到私服，其它项目组成员就可以从私服下载所依赖的jar。私服还充当一个代理服务器的角色，当私服上没有jar包时会从maven中央仓库自动下载。

nexus 是一个maven仓库管理器（其实就是一个软件），nexus可以充当maven私服，同时nexus还提供强大的仓库管理、构件搜索等功能。

## 搭建maven私服

①下载nexus

https://help.sonatype.com/repomanager2/download/download-archives---repository-manager-oss

②安装nexus

将下载的压缩包进行解压，进入bin目录

![1559551510928](.\img\图片17.png)

打开cmd窗口并进入上面bin目录下，执行nexus.bat install命令安装服务（注意需要以管理员身份运行cmd命令）

![1559551531544](.\img\图片18.png)

③启动nexus

经过前面命令已经完成nexus的安装，可以通过如下两种方式启动nexus服务：

在Windows系统服务中启动nexus

![1559551564441](.\img\图片19.png)

在命令行执行nexus.bat start命令启动nexus

![1559551591730](.\img\图片20.png)

④访问nexus

启动nexus服务后，访问http://localhost:8081/nexus

点击右上角LogIn按钮，进行登录。使用默认用户名admin和密码admin123登录系统

登录成功后点击左侧菜单Repositories可以看到nexus内置的仓库列表（如下图）

![1559551620133](.\img\图片.png)

nexus仓库类型

通过前面的仓库列表可以看到，nexus默认内置了很多仓库，这些仓库可以划分为4种类型，每种类型的仓库用于存放特定的jar包，具体说明如下：

①hosted，宿主仓库，部署自己的jar到这个类型的仓库，包括Releases和Snapshots两部分，Releases为公司内部发布版本仓库、 Snapshots为公司内部测试版本仓库 

②proxy，代理仓库，用于代理远程的公共仓库，如maven中央仓库，用户连接私服，私服自动去中央仓库下载jar包或者插件

③group，仓库组，用来合并多个hosted/proxy仓库，通常我们配置自己的maven连接仓库组

④virtual(虚拟)：兼容Maven1版本的jar或者插件

![1559551723693](.\img\图片21.png)

nexus仓库类型与安装目录对应关系

![1559551752012](.\img\图片22.png)

## 将项目发布到maven私服

maven私服是搭建在公司局域网内的maven仓库，公司内的所有开发团队都可以使用。例如技术研发团队开发了一个基础组件，就可以将这个基础组件打成jar包发布到私服，其他团队成员就可以从私服下载这个jar包到本地仓库并在项目中使用。

将项目发布到maven私服操作步骤如下：

1. 配置maven的settings.xml文件

```xml
<server>
<id>releases</id>
<username>admin</username>   
<password>admin123</password>
</server>
<server>
<id>snapshots</id>
<username>admin</username>
<password>admin123</password>
</server>
```

​      注意：一定要在idea工具中引入的maven的settings.xml文件中配置 

2. 配置项目的pom.xml文件

```xml
<distributionManagement>
<repository>
   <id>releases</id>
   <url>http://localhost:8081/nexus/content/repositories/releases/</url>
</repository>
<snapshotRepository>
   <id>snapshots</id>               <url>http://localhost:8081/nexus/content/repositories/snapshots/</url>    </snapshotRepository>
</distributionManagement>
```

3. 执行mvn deploy命令

![1559551977984](.\img\图片23.png)

## 从私服下载jar到本地仓库

前面我们已经完成了将本地项目打成jar包发布到maven私服，下面我们就需要从maven私服下载jar包到本地仓库。

具体操作步骤如下：

在maven的settings.xml文件中配置下载模板

```xml
<profile>
	<id>dev</id>
		<repositories>
		<repository>
			<id>nexus</id>
		<!--仓库地址，即nexus仓库组的地址-->
			<url>
			http://localhost:8081/nexus/content/groups/public/</url>
		<!--是否下载releases构件-->
			<releases>
			<enabled>true</enabled>
			</releases>
		<!--是否下载snapshots构件-->
			<snapshots>
				<enabled>true</enabled>
			</snapshots>
		</repository>
		</repositories>
<pluginRepositories>
	<!-- 插件仓库，maven的运行依赖插件，也需要从私服下载插件 -->
	<pluginRepository>
		<id>public</id>
		<name>Public Repositories</name>
		<url>
		http://localhost:8081/nexus/content/groups/public/</url>
		</pluginRepository>
		</pluginRepositories>
</profile>
```

在maven的settings.xml文件中配置激活下载模板

```xml
<activeProfiles>
	<activeProfile>dev</activeProfile>
</activeProfiles>
```

# 将第三方jar安装到本地仓库和maven私服

在maven工程的pom.xml文件中配置某个jar包的坐标后，如果本地的maven仓库不存在这个jar包，maven工具会自动到配置的maven私服下载，如果私服中也不存在，maven私服就会从maven中央仓库进行下载。

但是并不是所有的jar包都可以从中央仓库下载到，比如常用的Oracle数据库驱动的jar包在中央仓库就不存在。此时需要到Oracle的官网下载驱动jar包，然后将此jar包通过maven命令安装到我们本地的maven仓库或者maven私服中，这样在maven项目中就可以使用maven坐标引用到此jar包了。

## 将第三方jar安装到本地仓库

①下载Oracle的jar包（略）

②mvn install命令进行安装

​      mvn install:install-file -Dfile=ojdbc14-10.2.0.4.0.jar -DgroupId=com.oracle -DartifactId=ojdbc14 – 

​      Dversion=10.2.0.4.0 -Dpackaging=jar

③查看本地maven仓库，确认安装是否成功

![1559552325997](.\img\图片24.png)

## 将第三方jar安装到maven私服

①下载Oracle的jar包（略）

②在maven的settings.xml配置文件中配置第三方仓库的server信息

```xml
<server>
  <id>thirdparty</id>
  <username>admin</username>
  <password>admin123</password>
</server>
```

③执行mvn deploy命令进行安装

​      mvn deploy:deploy-file -Dfile=ojdbc14-10.2.0.4.0.jar -DgroupId=com.oracle -DartifactId=ojdbc14 –

​      Dversion=10.2.0.4.0 -Dpackaging=jar –

​      Durl=http://localhost:8081/nexus/content/repositories/thirdparty/ -DrepositoryId=thirdparty

# maven插件

Maven 实际上是一个依赖插件执行的框架，每个任务实际上是由插件完成。Maven 插件通常被用来： 

打包jar 文件 

创建 war 文件 

编译代码文件

 代码单元测试 

创建工程文档 

创建工程报告 

插件通常提供了一个目标的集合，并且可以使用下面的语法执行：

```
mvn [plugin-name]:[goal-name]
```

例如，一个 Java 工程可以使用 maven-compiler-plugin 的 compile-goal 编译，使用以下命令：

```
mvn compiler:compile
```

## maven-compiler-plugin

设置maven编译的jdk版本，maven3默认用jdk1.5，maven2默认用jdk1.3

```
<plugin>
	<groupId>org.apache.maven.plugins</groupId>
	<artifactId>maven-compiler-plugin</artifactId>
	<version>3.1</version>
	<configuration>
		<source>1.8</source> <!-- 源代码使用的JDK版本 -->
		<target>1.8</target> <!-- 需要生成的目标class文件的编译版本 -->
		<encoding>UTF-8</encoding><!-- 字符集编码 -->
	</configuration>
</plugin>
```

## tomcat插件

```
<plugin>
	<groupId>org.apache.tomcat.maven</groupId>
	<artifactId>tomcat7-maven-plugin</artifactId>
	<version>2.2</version>
	<configuration>
		<port>8080</port>
		<uriEncoding>UTF-8</uriEncoding>
		<path>/xinzhi</path>
		<finalName>xinzhi</finalName>
	</configuration>
</plugin>
```

点击idea右侧的maven我们可以方便的看到我们使用了什么插件，并可以点击执行相应的命令。

通过插件和命令我们都可以启动项目了，都不用部署到tomcat里了。

## war打包插件

```
<plugin>
	<groupId>org.apache.maven.plugins</groupId>
	<artifactId>maven-war-plugin</artifactId>
	<configuration>
		<warName>test</warName>
		<webResources>
			<resource>
				<directory>src/main/webapp/WEB-INF</directory>
				<filtering>true</filtering>
				<targetPath>WEB-INF</targetPath>
				<includes>
					<include>web.xml</include>
				</includes>
			</resource>
		</webResources>
	</configuration>
</plugin>
```

执行命令 mvn clean package打包

## jar插件（多）

maven-jar-plugin，默认的打包插件，用来打普通的project JAR包；

maven-shade-plugin，用来打可执行JAR包，也就是所谓的fat JAR包； 

maven-assembly-plugin，支持自定义的打包结构，也可以定制依赖项等。 

我们日常使用的以maven-assembly-plugin为最多，因为大数据项目中往往有很多shell脚本、SQL脚 本、.properties及.xml配置项等，采用assembly插件可以让输出的结构清晰而标准化。

```
<plugin>
	<groupId>org.apache.maven.plugins</groupId>
	<artifactId>maven-assembly-plugin</artifactId>
	<version>3.3.0</version>
	<executions>
		<execution>
			<id>make-assembly</id>
			<!-- 绑定到package生命周期 -->
			<phase>package</phase>
			<goals>
				<!-- 只运行一次 -->
				<goal>single</goal>
			</goals>
		</execution>
	</executions>
	<configuration>
		<archive>
			<manifest>
				<addClasspath>true</addClasspath>
				<mainClass>com.xinzi.Test</mainClass> <!-- 你的主类名 -->
			</manifest>
		</archive>
		<descriptorRefs>
			<descriptorRef>jar-with-dependencies</descriptorRef>
		</descriptorRefs>
	</configuration>
</plugin>
```

