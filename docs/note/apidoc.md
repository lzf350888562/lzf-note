swagger实体类搭配javax.validation.constraints的校验注解能在api界面展示校验限制



# swagger2

## springfox

当我们在使用Spring MVC写接口的时候，为了生成API文档，为了方便整合Swagger，都是用这个SpringFox的这套封装。但是，自从2.9.2版本更新之后，就一直没有什么动静，也没有更上Spring Boot的大潮流，有一段时间还一直都是写个配置类来为项目添加文档配置的。

```
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

```
@Configuration
@EnableSwagger2
public class Swagger2 {

    @Bean
    public Docket createRestApi() {
        return new Docket(DocumentationType.SWAGGER_2)
                .apiInfo(apiInfo())
                .select()
                .apis(RequestHandlerSelectors.basePackage("com.example.demo.controller"))
                .paths(PathSelectors.any())
                .build();
    }

    private ApiInfo apiInfo() {
        return new ApiInfoBuilder()
                .title("Spring Boot中使用Swagger2构建RESTful APIs")
                .description("更多Spring Boot相关文章请关注：http://blog.didispace.com/")
                .termsOfServiceUrl("http://blog.didispace.com/")
                .contact("程序猿DD")
                .version("1.0")
                .build();
    }

}
```

### api文档生成限定

```
//为当前包下controller生成API文档
.apis(RequestHandlerSelectors.basePackage("com.java2nb.novel"))
//为有@Api注解的Controller生成API文档
//.apis(RequestHandlerSelectors.withClassAnnotation(Api.class))
//为有@ApiOperation注解的方法生成API文档
//.apis(RequestHandlerSelectors.withMethodAnnotation(ApiOperation.class))
```

### token传递

```
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
    private ApiInfo apiInfo() {
        return new ApiInfoBuilder()
                .title("novel-cloud")
                .description("小说精品屋-cloud接口文档")
                .contact("xxy")
                .version("1.0.0")
                .build();
    }

    private List<ApiKey> securitySchemes() {
        return newArrayList(
                new ApiKey("Authorization", "Authorization", "header"));
    }

    private List<SecurityContext> securityContexts() {
        return newArrayList(
                SecurityContext.builder()
                        .securityReferences(defaultAuth())
                        //用户微服务和作家微服务（不包括登陆/注册）的接口需要认证
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



## spring4all starter

为解决上面的问题,就造了这么个轮子： DD自造

```
<dependency>
    <groupId>com.spring4all</groupId>
    <artifactId>swagger-spring-boot-starter</artifactId>
    <version>1.9.0.RELEASE</version>
</dependency>
```

```
@EnableSwagger2Doc
@SpringBootApplication
public class Chapter22Application {
    public static void main(String[] args) {
        SpringApplication.run(Chapter22Application.class, args);
    }
}
```

```
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

## springfox starter

现在SpringFox出了一个starter，看了一下功能，虽然还不完美，但相较于之前我们自己的轮子来说还是好蛮多的。来看看这个版本有些什么亮点：

- Spring 5，Webflux 支持（仅请求映射支持，尚不支持功能端点）
- Spring Integration 支持
- Spring Boot 支持 springfox-boot-starter 依赖性（零配置，自动配置支持）
- 具有自动完成功能的文档化配置属性
- 更好的规范兼容性
- 支持 OpenApi 3.0.3
- 几乎零依赖性（唯一需要的库是 spring-plugin、pswagger-core）
- 现有的 swagger2 注释将继续有效，并丰富 open API 3.0 规范

```
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



# swagger3

```
<dependency>
    <groupId>io.springfox</groupId>
    <artifactId>springfox-boot-starter</artifactId>
    <version>3.0.0</version>
    <exclusions>
        <exclusion>
            <groupId>io.swagger</groupId>
            <artifactId>swagger-models</artifactId>
        </exclusion>
    </exclusions>
</dependency>
```

```
package com.ruoyi.web.core.config;

import java.util.ArrayList;
import java.util.List;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import com.ruoyi.common.config.RuoYiConfig;
import io.swagger.annotations.ApiOperation;
import io.swagger.models.auth.In;
import springfox.documentation.builders.ApiInfoBuilder;
import springfox.documentation.builders.PathSelectors;
import springfox.documentation.builders.RequestHandlerSelectors;
import springfox.documentation.service.ApiInfo;
import springfox.documentation.service.ApiKey;
import springfox.documentation.service.AuthorizationScope;
import springfox.documentation.service.Contact;
import springfox.documentation.service.SecurityReference;
import springfox.documentation.service.SecurityScheme;
import springfox.documentation.spi.DocumentationType;
import springfox.documentation.spi.service.contexts.SecurityContext;
import springfox.documentation.spring.web.plugins.Docket;

/**
 * Swagger2的接口配置
 * 
 * @author ruoyi
 */
@Configuration
public class SwaggerConfig
{
    /** 系统基础配置 */
    @Autowired
    private RuoYiConfig ruoyiConfig;

    /** 是否开启swagger */
    @Value("${swagger.enabled}")
    private boolean enabled;

    /** 设置请求的统一前缀 */
    @Value("${swagger.pathMapping}")
    private String pathMapping;

    /**
     * 创建API
     */
    @Bean
    public Docket createRestApi()
    {
        return new Docket(DocumentationType.OAS_30)
                // 是否启用Swagger
                .enable(enabled)
                // 用来创建该API的基本信息，展示在文档的页面中（自定义展示的信息）
                .apiInfo(apiInfo())
                // 设置哪些接口暴露给Swagger展示
                .select()
                // 扫描所有有注解的api，用这种方式更灵活
                .apis(RequestHandlerSelectors.withMethodAnnotation(ApiOperation.class))
                // 扫描指定包中的swagger注解
                // .apis(RequestHandlerSelectors.basePackage("com.ruoyi.project.tool.swagger"))
                // 扫描所有 .apis(RequestHandlerSelectors.any())
                .paths(PathSelectors.any())
                .build()
                /* 设置安全模式，swagger可以设置访问token */
                .securitySchemes(securitySchemes())
                .securityContexts(securityContexts())
                .pathMapping(pathMapping);
    }

    /**
     * 安全模式，这里指定token通过Authorization头请求头传递
     */
    private List<SecurityScheme> securitySchemes()
    {
        List<SecurityScheme> apiKeyList = new ArrayList<SecurityScheme>();
        apiKeyList.add(new ApiKey("Authorization", "Authorization", In.HEADER.toValue()));
        return apiKeyList;
    }

    /**
     * 安全上下文
     */
    private List<SecurityContext> securityContexts()
    {
        List<SecurityContext> securityContexts = new ArrayList<>();
        securityContexts.add(
                SecurityContext.builder()
                        .securityReferences(defaultAuth())
                        .operationSelector(o -> o.requestMappingPattern().matches("/.*"))
                        .build());
        return securityContexts;
    }

    /**
     * 默认的安全上引用
     */
    private List<SecurityReference> defaultAuth()
    {
        AuthorizationScope authorizationScope = new AuthorizationScope("global", "accessEverything");
        AuthorizationScope[] authorizationScopes = new AuthorizationScope[1];
        authorizationScopes[0] = authorizationScope;
        List<SecurityReference> securityReferences = new ArrayList<>();
        securityReferences.add(new SecurityReference("Authorization", authorizationScopes));
        return securityReferences;
    }

    /**
     * 添加摘要信息
     */
    private ApiInfo apiInfo()
    {
        // 用ApiInfoBuilder进行定制
        return new ApiInfoBuilder()
                // 设置标题
                .title("标题：若依管理系统_接口文档")
                // 描述
                .description("描述：用于管理集团旗下公司的人员信息,具体包括XXX,XXX模块...")
                // 作者信息
                .contact(new Contact(ruoyiConfig.getName(), null, null))
                // 版本
                .version("版本号:" + ruoyiConfig.getVersion())
                .build();
    }
}
```

```
//类上
@Api("用户信息管理")
//方法上
@ApiOperation(value = "创建用户", notes = "根据User对象创建用户")

@ApiOperation("获取用户详细")
@ApiImplicitParam(name = "userId", value = "用户ID", required = true, dataType = "int", paramType = "path")
@GetMapping("/{userId}")
public AjaxResult getUser(@PathVariable Integer userId)
{
    if (!users.isEmpty() && users.containsKey(userId))
    {
        return AjaxResult.success(users.get(userId));
    }
    else
    {
        return error("用户不存在");
    }
}

@ApiOperation("新增用户")
@ApiImplicitParams({
        @ApiImplicitParam(name = "userId", value = "用户id", dataType = "Integer"),
        @ApiImplicitParam(name = "username", value = "用户名称", dataType = "String"),
        @ApiImplicitParam(name = "password", value = "用户密码", dataType = "String"),
        @ApiImplicitParam(name = "mobile", value = "用户手机", dataType = "String")
    })
@PostMapping("/save")
public AjaxResult save(UserEntity user)
{
    if (StringUtils.isNull(user) || StringUtils.isNull(user.getUserId()))
    {
        return error("用户ID不能为空");
    }
    return AjaxResult.success(users.put(user.getUserId(), user));
}
```

实体类

```
@ApiModel(value = "UserEntity", description = "用户实体")
class UserEntity
{
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

访问 xxxx/swagger-ui.html,  捷径访问

```
@Controller
@RequestMapping("/tool/swagger")
public class SwaggerController extends BaseController
{
    @GetMapping()
    public String index()
    {
        return "redirect:/swagger-ui.html";
    }
}
```

# swagger配置详解

我们在Spring Boot中定义各个接口是以`Controller`作为第一级维度来进行组织的，`Controller`与具体接口之间的关系是一对多的关系。我们可以将同属一个模块的接口定义在一个`Controller`里。默认情况下，Swagger是以`Controller`为单位，对接口进行分组管理的。这个分组的元素在Swagger中称为`Tag`，但是这里的`Tag`与接口的关系并不是一对多的，它支持更丰富的多对多关系。

## **自定义默认分组的名称**

```
@Api(tags = "教师管理")
@Api(tags = "学生管理")
```

代码中`@Api`定义的`tags`内容替代了默认产生的`teacher-controller`和`student-controller`。

## **合并Controller分组**

`Tag`与`Controller`一一对应的情况，Swagger中还支持更灵活的分组！从`@Api`注解的属性中，可发现`tags`属性其实是个数组类型.

我们可以通过定义同名的`Tag`来汇总`Controller`中的接口，比如我们可以定义一个`Tag`为“教学管理”，让这个分组同时包含教师管理和学生管理的所有接口，可以这样来实现：

```
@Api(tags = {"教师管理", "教学管理"})
@Api(tags = {"学生管理", "教学管理"})
```

## 更细粒度的接口分组

通过`@Api`可以实现将`Controller`中的接口合并到一个`Tag`中，如果要精确到某个接口的合并呢？

那么上面的实现方式就无法满足了。这时候发，我们可以通过使用`@ApiOperation`注解中的`tags`属性做更细粒度的接口分类定义，比如上面的需求就可以这样子写：

```
@Api(tags = {"教师管理","教学管理"})
@RestController
@RequestMapping(value = "/teacher")
static class TeacherController {
    @ApiOperation(value = "xxx")
    @GetMapping("/xxx")
    public String xxx() {
        return "xxx";
    }
}

@Api(tags = {"学生管理"})
@RestController
@RequestMapping(value = "/student")
static class StudentController {
    @ApiOperation(value = "获取学生清单", tags = "教学管理")
    @GetMapping("/list")
    public String xxx() {
        return "xxx";
    }
}
```

## 内容顺序

修改方式1

```
@Bean
public UiConfiguration uiConfiguration(SwaggerPropertes swaggerPropertes){
	return UiConfigurationBuilder.builder()
		.xxx(xxx)
		.tagsSorter(swaggerPropertes.getUiConfig().getTagsSorter())
		.xxx(xxx)
		.build();
}
```

修改方式2

```
swagger.ui-config.tags-sorter=alpha
```

但是,查看源码TagsSorter枚举发现,无可排序选项,Swagger只提供了一个选项，就是按字母顺序排列。

所以在不拓展源码的情况下只能仅依靠使用方式的定义来实现排序的建议：为Tag的命名做编号

```
@Api(tags = {"1-教师管理","3-教学管理"})
@Api(tags = {"2-学生管理"})
```

## 接口排序

方式 1

```
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

```
swagger.ui-config.operations-sorter=alpha
```

很庆幸，这个配置不像Tag的排序配置没有可选项。它提供了两个配置项：`alpha`和`method`，分别代表了按字母表排序以及按方法定义顺序排序。当我们不配置的时候，改配置默认为`alpha`

## 参数排序

默认情况下，Swagger对Model参数内容的展现也是按字母顺序排列的.

如果我们希望可以按照Model中定义的成员变量顺序来展现，那么需要我们通过`@ApiModelProperty`注解的`position`参数来实现位置的设置，比如：

```
 @ApiModelProperty(value = "用户编号", position = 1)
 @ApiModelProperty(value = "用户姓名", position = 2)
```

# knife4j

[官方](https://doc.xiaominfo.com/)

# 静态文档生成Swagger2Markup

Swagger2Markup是Github上的一个开源项目。该项目主要用来将Swagger自动生成的文档转换成几种流行的格式以便于静态部署和使用，比如：AsciiDoc、Markdown、Confluence。

## 代码生成

使用的相关依赖和仓库

```
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

```
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
- `from(remoteSwaggerFile`：指定了生成静态部署文档的源头配置，可以是这样的URL形式，也可以是符合Swagger规范的String类型或者从文件中读取的流。如果是对当前使用的Swagger项目，我们通过使用访问本地Swagger接口的方式，如果是从外部获取的Swagger文档配置文件，就可以通过字符串或读文件的方式
- `toFolder(outputDirectory)`：指定最终生成文件的具体*目录*位置

执行了测试用例之后，指定目录下生成数个adoc(AsciiDoc)文件：

如果向将生成输出到单个文件:可以通过替换`toFolder(Paths.get("src/docs/asciidoc/generated")`为`toFile(Paths.get("src/docs/asciidoc/generated/all"))`，将转换结果输出到一个单一的文件中，这样可以最终生成html的也是单一的。

## Maven 插件来生成

```
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

**生成html**

在完成了从Swagger文档配置文件到AsciiDoc的源文件转换之后，就是如何将AsciiDoc转换成可部署的HTML内容了。这里继续在上面的工程基础上，引入一个Maven插件来完成。

```
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

## 其他类型文档

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
