# API文档

## Swagger

### Springfox

当我们在使用Spring MVC写接口的时候，为了生成API文档，为了方便整合Swagger，都是用这个SpringFox的这套封装。但是，自从2.9.2版本更新之后，就一直没有什么动静，也没有更上Spring Boot的大潮流，有一段时间还一直都是写个配置类来为项目添加文档配置的。

```xml
<dependency>
    <groupId>io.springfox</groupId>
    <artifactId>springfox-swagger2</artifactId>
    <version>2.2.2</version>
</dependency>
<dependency>
    <groupId>io.springfox</groupId>
    <artifactId>springfox-swagger-ui</artifactId>
    <version>2.2.2</version>
</dependency>
```

```java
@Configuration
@EnableSwagger2
public class Swagger2 {
    @Bean
    public Docket createRestApi() {
        return new Docket(DocumentationType.SWAGGER_2)
                .apiInfo(apiInfo())
                .select()
                 //为当前包下controller生成API文档
                .apis(RequestHandlerSelectors.basePackage("com.example.demo.controller")
                //为有@Api注解的Controller生成API文档
                //.apis(RequestHandlerSelectors.withClassAnnotation(Api.class))
                //为有@ApiOperation注解的方法生成API文档
                //.apis(RequestHandlerSelectors.withMethodAnnotation(ApiOperation.class))
                // 扫描所有 
                //.apis(RequestHandlerSelectors.any())
                .paths(PathSelectors.any())
                .build();
                // 统一前缀
                //.pathMapping("api");
    }
    private ApiInfo apiInfo() {
        return new ApiInfoBuilder()
                .title("xxx")
                .description("xxx")
                .termsOfServiceUrl("www.xxx.com")
                .contact("xxx")
                .version("1.0")
                .build();
    }

}
```

### 使用模板

```java
@ApiModel(value = "UserEntity", description = "用户实体")
class UserEntity{
    @ApiModelProperty("用户ID")
    private Integer userId;
    @ApiModelProperty("用户名称")
    private String username;
    @ApiModelProperty("用户密码")
    private String password;
    @ApiModelProperty("用户手机")
    private String mobile;
    // getter
}
```

```java
@ApiModel(value = "UserEntity", description = "用户实体")
class UserEntity{
    @ApiModelProperty("用户ID")
    private Integer userId;
    @ApiModelProperty("用户名称")
    private String username;
    @ApiModelProperty("用户密码")
    private String password;
    @ApiModelProperty("用户手机")
    private String mobile;
    // gette
}
```

### token传递

```java
     @Bean
    public Docket createRestApi() {
         return new Docket(DocumentationType.SWAGGER_2)
                .apiInfo(apiInfo())
                .select()
                .apis(RequestHandlerSelectors.withClassAnnotation(Api.class))
                .paths(PathSelectors.any())
                .build()
                .securitySchemes(securitySchemes())
                .securityContexts(securityContexts());
    }
    /**
     * 安全模式，这里指定token通过Authorization头请求头传递
     */
    private List<ApiKey> securitySchemes() {
        return newArrayList(
            // header字符串可替换为In.HEADER.toValue()
                new ApiKey("Authorization", "Authorization", "header"));
    }

    private List<SecurityContext> securityContexts() {
        return newArrayList(
                SecurityContext.builder()
                        .securityReferences(defaultAuth())
                        //指定需要认证的接口路径, 登录注册接口除外
                        .forPaths(PathSelectors.regex("^/(user|author)/(?!(login|register)).*$"))
                        .build()
        );
    }
     List<SecurityReference> defaultAuth() {
        AuthorizationScope authorizationScope = new AuthorizationScope("global", "accessEverything");
        AuthorizationScope[] authorizationScopes = new AuthorizationScope[1];
        authorizationScopes[0] = authorizationScope;
        return newArrayList(
                new SecurityReference("Authorization", authorizationScopes));
    }
```

页面右上角出现`Authorize` 按钮,点击按钮会出现一个弹窗，弹窗内可输入 `Authorization`

### Spring4all Starter

```xml
<dependency>
    <groupId>com.spring4all</groupId>
    <artifactId>swagger-spring-boot-starter</artifactId>
    <version>1.9.0.RELEASE</version>
</dependency>
```

```java
@EnableSwagger2Doc
@SpringBootApplication
public class Chapter22Application {
    public static void main(String[] args) {
        SpringApplication.run(Chapter22Application.class, args);
    }
}
```

```properties
swagger.title=spring-boot-starter-swagger
swagger.description=Starter for swagger 2.x
swagger.version=1.4.0.RELEASE
swagger.license=Apache License, Version 2.0
swagger.licenseUrl=https://www.apache.org/licenses/LICENSE-2.0.html
swagger.termsOfServiceUrl=https://github.com/dyc87112/spring-boot-starter-swagger
swagger.contact.name=didi
swagger.contact.url=http://blog.didispace.com
swagger.contact.email=dyc87112@qq.com
swagger.base-package=com.didispace
swagger.base-path=/**
```

- `swagger.title`：标题
- `swagger.description`：描述
- `swagger.version`：版本
- `swagger.license`：许可证
- `swagger.licenseUrl`：许可证URL
- `swagger.termsOfServiceUrl`：服务条款URL
- `swagger.contact.name`：维护人
- `swagger.contact.url`：维护人URL
- `swagger.contact.email`：维护人email
- `swagger.base-package`：swagger扫描的基础包，默认：全扫描
- `swagger.base-path`：需要处理的基础URL规则，默认：/**

https://github.com/SpringForAll/spring-boot-starter-swagger

### Springfox Starter

```xml
<dependency>
    <groupId>io.springfox</groupId>
    <artifactId>springfox-boot-starter</artifactId>
    <version>3.0.0</version>
<dependency>
```

应用主类增加注解`@EnableOpenApi`。

注意：

1. 这次更新，移除了原来默认的swagger页面路径：`http://host/context-path/swagger-ui.html`，新增了两个可访问路径：`http://host/context-path/swagger-ui/index.html`和`http://host/context-path/swagger-ui/`
2. 通过调整日志级别，还可以看到新版本的swagger文档接口也有新增，除了以前老版本的文档接口`/v2/api-docs`之外，还多了一个新版本的`/v3/api-docs`接口。

### 自定义配置

**自定义默认分组的名称**

```java
@Api(tags = "教师管理")
@Api(tags = "学生管理")
```

`@Api`的`tags`内容替代了默认产生的`teacher-controller`和`student-controller`。

**合并分组**

`tags`属性其实是个数组类型, 可以通过定义同名的`Tag`来汇总`Controller`中的接口.

```java
@Api(tags = {"教师管理", "教学管理"})
@Api(tags = {"学生管理", "教学管理"})
```

**更细粒度的接口分组**

精确到具体某个接口的合并, 可通过在接口方法上使用`@ApiOperation`注解中的`tags`属性做更细粒度的接口分类定义，比如上面的需求就可以这样子写：

```java
@ApiOperation(value = "获取学生清单", tags = "教学管理")
```

**tag排序**

方式1

```java
@Bean
public UiConfiguration uiConfiguration(SwaggerPropertes swaggerPropertes){
    return UiConfigurationBuilder.builder()
        .xxx(xxx)
        .tagsSorter(swaggerPropertes.getUiConfig().getTagsSorter())
        .xxx(xxx)
        .build();
}
```

方式2

```properties
swagger.ui-config.tags-sorter=alpha
```

但是,查看源码TagsSorter枚举发现,无可排序选项,Swagger只提供了一个选项,就是按**字母顺序**排列.

所以在不拓展源码的情况下只能仅依靠使用方式的定义来实现排序的建议：为Tag的命名做编号

```java
@Api(tags = {"1-教师管理","3-教学管理"})
@Api(tags = {"2-学生管理"})
```

**接口排序**

方式 1

```java
@Bean
public UiConfiguration uiConfiguration(SwaggerPropertes swaggerPropertes){
    return UiConfigurationBuilder.builder()
        .xxx(xxx)
        .operationSorter(swaggerPropertes.getUiConfig().getOperationSorter)
        .xxx(xxx)
        .build();
}
```

方式2

```pro
swagger.ui-config.operations-sorter=alpha
```

两个配置项：`alpha`和`method`, 分别代表了按字母表排序以及按方法定义顺序排序. 默认为`alpha`

**参数排序**

默认情况下，Swagger对Model参数内容的展现也是按字母顺序排列的.

可通过`@ApiModelProperty`注解的`position`参数来实现位置的设置.

```java
 @ApiModelProperty(value = "用户编号", position = 1)
 @ApiModelProperty(value = "用户姓名", position = 2)
```

## knife4j

对swagger增强, 中文文档:[https://doc.xiaominfo.com/](https://doc.xiaominfo.com/)

## 静态文档生成

### Swagger2Markup

Swagger2Markup是Github上的一个开源项目。该项目主要用来将Swagger自动生成的文档转换成几种流行的格式以便于静态部署和使用，比如：AsciiDoc、Markdown、Confluence。

#### 生成AsciiDoc

使用的相关依赖和仓库

```xml
<dependencies>
    ...
    <dependency>
        <groupId>io.github.swagger2markup</groupId>
        <artifactId>swagger2markup</artifactId>
        <version>1.3.3</version>
        <scope>test</scope>
    </dependency>
</dependencies>
<repositories>
    <repository>
        <snapshots>
            <enabled>false</enabled>
        </snapshots>
        <id>jcenter-releases</id>
        <name>jcenter</name>
        <url>http://jcenter.bintray.com</url>
    </repository>
</repositories>
```

把`scope`设置为test，这样这个依赖就不会打包到正常运行环境中去。

```java
@RunWith(SpringRunner.class)
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.DEFINED_PORT)
public class DemoApplicationTests {
    @Test
    public void generateAsciiDocs() throws Exception {
        URL remoteSwaggerFile = new URL("http://localhost:8080/v2/api-docs");
        Path outputDirectory = Paths.get("src/docs/asciidoc/generated");

        //    输出Ascii格式
        Swagger2MarkupConfig config = new Swagger2MarkupConfigBuilder()
                .withMarkupLanguage(MarkupLanguage.ASCIIDOC)
                .build();

        Swagger2MarkupConverter.from(remoteSwaggerFile)
                .withConfig(config)
                .build()
                .toFolder(outputDirectory);
    }
}
```

- `MarkupLanguage.ASCIIDOC`：指定了要输出的最终格式。除了`ASCIIDOC`之外，还有`MARKDOWN`和`CONFLUENCE_MARKUP`，分别定义了其他格式，后面会具体举例。
- `from(remoteSwaggerFile`：指定了生成静态部署文档的源头配置，可以是URL，也可以是符合Swagger规范的String类型或者从文件中读取的流。如果是对当前使用的Swagger项目，我们通过使用访问本地Swagger接口的方式，如果是从外部获取的Swagger文档配置文件，就可以通过字符串或读文件的方式
- `toFolder(outputDirectory)`：指定最终生成文件的具体*目录*位置

执行了测试用例之后，指定目录下生成数个adoc(AsciiDoc)文件：

如果要将生成输出到单个文件:可以通过替换`toFolder(Paths.get("src/docs/asciidoc/generated")`为`toFile(Paths.get("src/docs/asciidoc/generated/all"))`，将转换结果输出到一个单一的文件中，这样可以最终生成html的也是单一的。

#### Maven插件形式

```xml
<plugin>
    <groupId>io.github.swagger2markup</groupId>
    <artifactId>swagger2markup-maven-plugin</artifactId>
    <version>1.3.3</version>
    <configuration>
        <swaggerInput>http://localhost:8080/v2/api-docs</swaggerInput>
        <outputDir>src/docs/asciidoc/generated-by-plugin</outputDir>
        <config>
            <swagger2markup.markupLanguage>ASCIIDOC</swagger2markup.markupLanguage>
        </config>
    </configuration>
</plugin>
```

在使用插件生成前，需要先启动应用。然后执行插件，就可以在`src/docs/asciidoc/generated-by-plugin`目录下看到也生成了上面一样的adoc文件了

#### **转换为html**

在完成了从Swagger文档配置文件到AsciiDoc的源文件转换之后，就是如何将AsciiDoc转换成可部署的HTML内容了。这里继续在上面的工程基础上，引入一个Maven插件来完成。

```xml
<plugin>
    <groupId>org.asciidoctor</groupId>
    <artifactId>asciidoctor-maven-plugin</artifactId>
    <version>1.5.6</version>
    <configuration>
           <sourceDirectory>src/docs/asciidoc/generated</sourceDirectory>
           <outputDirectory>src/docs/asciidoc/html</outputDirectory>
           <backend>html</backend>
           <sourceHighlighter>coderay</sourceHighlighter>
           <attributes>
            <toc>left</toc>
          </attributes>
      </configuration>
</plugin>
```

过上面的配置，执行该插件的`asciidoctor:process-asciidoc`命令之后，就能在`src/docs/asciidoc/html`目录下生成最终可用的静态部署HTML了。在完成生成之后，可以直接通过浏览器来看查看

#### 其他类型文档

要生成Markdown和Confluence的方式非常简单

方法1:

过Java代码来生成：只需要修改`withMarkupLanguage`属性来指定不同的格式以及`toFolder`属性为结果指定不同的输出目录。

```
.withMarkupLanguage(MarkupLanguage.MARKDOWN)
或
.withMarkupLanguage(MarkupLanguage.CONFLUENCE_MARKUP)
```

方法2:

插件swagger2markup.markupLanguage修改
