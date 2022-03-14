# Maven构建工具

maven可帮助自动化构建过程, 从清理、编译、测试到生成报告, 再到打包部署.

maven以插件形式实现了大部分构建任务.

maven还是一个依赖管理工具和项目信息管理工具, 通过坐标定位每个构建(artifact), 即jar包. 

## POM

POM(Project Object Model, 项目对象模型)定义了项目的基本信息, 用户描述项目如何构建, 声明项目依赖等.

每个构建步骤都可以绑定一个或多个插件行为, 且Maven为大多数构建步骤编写并绑定了默认插件(如编译插件maven-compiler0-plugin, 测试插件maven-surefire-plugin). 当有特殊需要的时候, 可以配置插件定制构建行为, 甚至自己编写插件.

## 生命周期

Maven拥有三套相互独立的声明周期, 每个声明周期包含一些阶段, 这些阶段是有序的, 并且**后面的阶段依赖于前面的阶段**:

1.clean生命周期

```
pre-clean 执行一些清理前需要完成的工作;
clean 清理上一次构建生成的文件;
post-clean 执行一些清理后需要完成的工作.
```

2.default声明周期

定义了真正构建时所需要执行的所有步骤, 它是所有声明周期中最核心的部分.

```
validate
initialize
generate-sources
process-sources 处理项目主资源文件:对src/main/resources中的内容进行变量替换等工作之后复制到项目输出的主classpath目录中. 
generate-resources
process-resources
compile 编译项目的主源码:编译src/main/java下的java文件至项目输出的主classpath目录中.
process-classes
generate-test-sources
process-test-sources 处理项目测试资源文件:对src/test/resources目录的内容进行变量替换等工作后, 复制到项目输出的测试classpath目录中.
generate-test-resources
process-test-resources
test-compile 编译项目的测试代码:编译src/test/java目录下的java文件至项目输出的测试classpath目录中.
process-test-classes
test 使用单元测试框架运行测试, 测试代码不会被打包或部署.
prepare-package
package 接受编译好的代码, 打包成可发布的格式,如jar
pre-integration-test
integration-test
post-integration-test
verify
install 将包安装到maven本地仓库.
deploy	将最终的包复制到远程仓库.
```

3.site生命周期

建立和发布项目站点, maven能基于pom所包含的信息, 自动生成一个友好的站点.

```
pre-site 执行一些在生成项目站点之前需要完成的工作.
site 生成项目站点文档.
post-site 执行一些在生成项目站点之后需要完成的工作.
site-deploy 将生成的项目站点发布到服务器上.
```

### 命令行

命令行执行Maven任务: 通过调用Maven的生命周期阶段. 常见的有:

```
$mvn clean:调用clean生命周期的clean阶段.即实际执行clean生命周期的pre-clean和clean阶段.
$mvm test:调用default生命周期的test阶段.
$mvn clean install:调用clean生命周期的clean阶段和default生命周期的install阶段.该命令结合了两个生命周期:在执行构建时清理项目是一个很好的实践.
$mvn clean deploy site-deploy:...
```

## 插件

maven的核心仅定义了抽象的生命周期, **具体任务是交由插件完成的, 插件以独立的构建形式存在**, 因此maven核心的分发包很小, 只会在需要的时候下载并使用插件.

而一个插件为了复用代码, 往往能完成多个任务, **对应多个目标**, 如maven-depdendency-plugin能基于项目依赖做很多事情:

dependency:analyze插件目标能分析项目依赖;

dependency:tree能列出项目的依赖树;

dependency:list能列出项目所有已解析的依赖;

...

**插件的目标与生命周期的阶段相互绑定**, 以完成某个具体的构建任务, 如maven-compile-plugin插件的compile目标能完成default生命周期的compile阶段, 完成项目编译任务.

为了在不进行任何maven配置下构建maven项目, maven核心为一些**主要生命周期阶段绑定了很多插件的目标**, 当用户通过命令行调用生命周期阶段时, 对应的插件目标就会执行相应的任务(反向绑定?). 如clean生命周期的clean阶段与maven-clean-plugin:clean绑定

如何自定义将某个插件目标绑定到生命周期的某个阶段?

可通过plugin->executions->execution添加任务, 其下的id指定任务名称(随意), phrase指定要绑定的生命周期阶段, goals->goal指定要执行的插件目标.

随后运行mvn+phrase标签指定的阶段即可执行该插件的goal标签指定的目标.

> 通过maven-help-plugin查看插件详细信息, 了解插件目标的默认绑定阶段, 如:
>
> mvn help:describe-Dplugin = org.apache.maven.pluguins:maven-source-plugin:2.1.1-Ddetail
>
> 每个目标下的Bound to phase指定了该目标默认绑定的生命周期阶段

### 插件配置

完成了插件与生命周期绑定后, 可配置插件目标参数

命令行可使用-D+参数键=参数值的形式(-D为java自带)

POM全局配置方式可一次性配置之后所有基于该插件目标的任务都会使用该配置, 如可参考通过maven-compile-plugin插件指定编译的java版本

# 测试

在JUnit4标准下, 所有需要执行的测试方法都以test开头, 并以@Test标注

# 问题

## 默认jar无法运行

因为带有main方法的类信息不会添加到manifest中, 可打开jar文件中的META-INF/MANIFEST.MF文件查看.

解决方案: 通过配置maven-shade-plugin插件, mainClass标签设置为包含main方法的类, 接下来打包完就可以直接执行了

```
<plugin>
	<groupId>org.apache.maven.plugins</groupId>
	<artifactId>maven-shade-plugin</artifactId>
	<version>1.2.1</version>
	<executions>
		<execution>
			<phase>package</phase>
			<goals>
				<goal>shade</goal>
			</goals>
			<configuration>
				<transformers>
					<transformer implementation="org.apache.maven.plugins.shade.resource.ManifestResourceTransformer">
						<mainClass>com.juvenxu.mvnbook.helloworld.Helloworld</mainClass>
					</transformer>
				</transformers >
			</configuration>
		</execution>
	</executions>
</plugin>
```



# maven私服

### 私服说明 

maven仓库分为本地仓库和远程仓库，而远程仓库又分为maven中央仓库、其他远程仓库和私服（私有服务器）。其中，中央仓库是由maven官方提供的，而私服就需要我们自己搭建了。

maven私服就是公司局域网内的maven远程仓库，每个员工的电脑上安装maven软件并且连接maven私服，程序员可以将自己开发的项目打成jar并发布到私服，其它项目组成员就可以从私服下载所依赖的jar。私服还充当一个代理服务器的角色，当私服上没有jar包时会从maven中央仓库自动下载。

nexus 是一个maven仓库管理器（其实就是一个软件），nexus可以充当maven私服，同时nexus还提供强大的仓库管理、构件搜索等功能。

### 搭建maven私服

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

### 将项目发布到maven私服

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

### 从私服下载jar到本地仓库

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

## 将第三方jar安装到本地仓库和maven私服

在maven工程的pom.xml文件中配置某个jar包的坐标后，如果本地的maven仓库不存在这个jar包，maven工具会自动到配置的maven私服下载，如果私服中也不存在，maven私服就会从maven中央仓库进行下载。

但是并不是所有的jar包都可以从中央仓库下载到，比如常用的Oracle数据库驱动的jar包在中央仓库就不存在。此时需要到Oracle的官网下载驱动jar包，然后将此jar包通过maven命令安装到我们本地的maven仓库或者maven私服中，这样在maven项目中就可以使用maven坐标引用到此jar包了。

### 将第三方jar安装到本地仓库

①下载Oracle的jar包（略）

②mvn install命令进行安装

​      mvn install:install-file -Dfile=ojdbc14-10.2.0.4.0.jar -DgroupId=com.oracle -DartifactId=ojdbc14 – 

​      Dversion=10.2.0.4.0 -Dpackaging=jar

③查看本地maven仓库，确认安装是否成功

![1559552325997](.\img\图片24.png)

### 将第三方jar安装到maven私服

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

## maven插件

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

### maven-compiler-plugin

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

### tomcat插件

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

### war打包插件

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

### jar插件（多）

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

