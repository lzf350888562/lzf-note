# Mybatis

**#{}与${}**

1）#{}是预编译处理，$ {}是字符串替换。

2）MyBatis在处理#{}时，会将SQL中的#{}替换为?号，使用PreparedStatement的set方法来赋值；MyBatis在处理 $ { } 时，就是把 ${ } 替换成变量的值。

3）使用 #{} 可以有效的防止SQL注入，提高系统安全性。

> like语句不能使用 %#{xxx}% ,应使用concat;
>
> in语句不能使用 in (${ids}), 应使用foreach标签

> order by 不能使用 order by #{} ,因为order by后接列名,属于代码语义的一部分. 如果在语义分析部分没有确定下来就相当于执行 order by 空列名, 有语法 错误

## Dao与xml原理

Dao 接口的全限名，就是映射文件中的 namespace 的值，接口的方法名，就是映射文件中 `MappedStatement` 的 id 值，接口方法内的参数，就是传递给 sql 的参数。

MyBatis 将所有 Xml 配置信息都封装到 All-In-One 重量级对象 Configuration 内部。在 Xml 映射文件中， `<parameterMap>` 标签会被解析为 `ParameterMap` 对象，其每个子元素会被解析为 `ParameterMapping` 对象。 `<resultMap>` 标签会被解析为 `ResultMap` 对象，其每个子元素会被解析为 `ResultMapping` 对象。每一个 `<select>、<insert>、<update>、<delete>` 标签均会被解析为 `MappedStatement` 对象，标签内的 sql 会被解析为 BoundSql 对象。 

## 执行器

MyBatis 有三种基本的 Executor 执行器， `SimpleExecutor` 、 `ReuseExecutor` 、 `BatchExecutor` 。

`SimpleExecutor` ：每执行一次 update 或 select，就开启一个 Statement 对象，用完立刻关闭 Statement 对象。

`ReuseExecutor` ：执行 update 或 select，以 sql 作为 key 查找 Statement 对象，存在就使用，不存在就创建，用完后，不关闭 Statement 对象，而是放置于 Map<String, Statement>内，供下一次使用。简言之，就是重复使用 Statement 对象。

`BatchExecutor` ：执行 update（没有 select，JDBC 批处理不支持 select），将所有 sql 都添加到批处理中（addBatch()），等待统一执行（executeBatch()），它缓存了多个 Statement 对象，每个 Statement 对象都是 addBatch()完毕后，等待逐一执行 executeBatch()批处理。与 JDBC 批处理相同。

作用范围：Executor 的这些特点，都严格限制在 SqlSession 生命周期范围内。

## 分页

实现分页的方式

**(1)** MyBatis 使用 RowBounds 对象进行分页，它是针对 ResultSet 结果集执行的**内存分页**(查询出来再分页)，而非物理分页；

在dao接口传入一个RowBounds(int  offset , int limit);就可进行分页

**(2)** 可以在 sql 内直接书写带有物理分页的参数来完成物理分页功能，

**(3)** 也可以使用分页插件来完成物理分页。 

分页插件的基本原理是使用 MyBatis 提供的插件接口，实现自定义插件，在插件的拦截方法内拦截待执行的 sql，然后重写 sql，根据 dialect 方言，添加对应的物理分页语句和物理分页参数。

### 插件

mybatis插件就是对ParameterHandler、ResultSetHandler、StatementHandler、Executor这四个接口上的方法进行拦截，利用JDK动态代理机制，为这些接口的实现类创建代理对象，在执行方法时，先去执行代理对象的方法，从而执行自己编写的拦截逻辑，所以真正要用好mybatis插件，主要还是要熟悉这四个接口的方法以及这些方法上的参数的含义；

另外，如果配置了多个拦截器的话，会出现层层代理的情况，即代理对象代理了另外一个代理对象，形成一个代理链条，执行的时候，也是层层执行；

关于mybatis插件涉及到的设计模式和软件思想如下：

设计模式：代理模式、责任链模式；
 软件思想：AOP编程思想，降低模块间的耦合度，使业务模块更加独立；
 一些注意事项：

不要定义过多的插件，代理嵌套过多，执行方法的时候，比较耗性能；
 拦截器实现类的intercept方法里最后不要忘了执行invocation.proceed()方法，否则多个拦截器情况下，执行链条会断掉；

```java
import org.apache.ibatis.executor.statement.StatementHandler;
import org.apache.ibatis.mapping.BoundSql;
import org.apache.ibatis.plugin.*;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import java.sql.Connection;
import java.util.Properties;

@Intercepts({
        @Signature(type = StatementHandler.class, method = "prepare", args = {Connection.class, Integer.class})
})
public class MyPlugin implements Interceptor {
    private Logger logger = LoggerFactory.getLogger(MyPlugin.class);
    private long time;


    //方法拦截
    @Override
    public Object intercept(Invocation invocation) throws Throwable {
        //通过StatementHandler获取执行的sql
        StatementHandler statementHandler = (StatementHandler) invocation.getTarget();
        BoundSql boundSql = statementHandler.getBoundSql();
        String sql = boundSql.getSql();

        long start = System.currentTimeMillis();
        Object proceed = invocation.proceed();
        long end = System.currentTimeMillis();
        if ((end - start) > time) {
            logger.info("本次数据库操作是慢查询，sql是:" + sql);
        }
        return proceed;
    }

    //获取到拦截的对象，底层也是通过代理实现的，实际上是拿到一个目标代理对象
    @Override
    public Object plugin(Object target) {
        return Plugin.wrap(target, this);
    }

    //获取设置的阈值等参数
    @Override
    public void setProperties(Properties properties) {
        this.time = Long.parseLong(properties.getProperty("time"));
    }
}
```

在 springboot 那配置一下（我用的是 mybatisplus）



```java
import com.baomidou.mybatisplus.autoconfigure.ConfigurationCustomizer;
import com.baomidou.mybatisplus.core.MybatisConfiguration;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

import java.util.Properties;

@Configuration
public class MapperConfig {
    //将插件加入到mybatis插件拦截链中
    @Bean
    public ConfigurationCustomizer configurationCustomizer() {
        return new ConfigurationCustomizer() {
            @Override
            public void customize(MybatisConfiguration configuration) {
                //插件拦截链采用了责任链模式，执行顺序和加入连接链的顺序有关
                MyPlugin myPlugin = new MyPlugin();
                //设置参数，比如阈值等，可以在配置文件中配置，这里直接写死便于测试
                Properties properties = new Properties();
                //这里设置慢查询阈值为1毫秒，便于测试
                properties.setProperty("time", "1");
                myPlugin.setProperties(properties);
                configuration.addInterceptor(myPlugin);
            }
        };
    }
}
```

## 延迟加载

1. 延迟加载的条件：resultMap可以实现高级映射（使用association、collection实现一对一及一对多映射），association、collection具备延迟加载功能。
2. 延迟加载的好处：
   先从单表查询、需要时再从关联表去关联查询，大大提高 数据库性能，因为查询单表要比关联查询多张表速度要快。
3. 延迟加载的实例：
   如果查询订单并且关联查询用户信息。如果先查询订单信息即可满足要求，当我们需要查询用户信息时再查询用户信息。把对用户信息的按需去查询就是延迟加载。

在 MyBatis 配置文件中，可以配置是否启用延迟加载 `lazyLoadingEnabled=true|false。`

它的原理是，使用 `CGLIB` 创建目标对象的代理对象，当调用目标方法时，进入拦截器方法，比如调用 `a.getB().getName()` ，拦截器 `invoke()` 方法发现 `a.getB()` 是 null 值，那么就会单独发送事先保存好的查询关联 B 对象的 sql，把 B 查询上来，然后调用 a.setB(b)，于是 a 的对象 b 属性就有值了，接着完成 `a.getB().getName()` 方法的调用。这就是延迟加载的基本原理。

不光是 MyBatis，几乎所有的包括 Hibernate，支持延迟加载的原理都是一样的。

## 缓存

**一级缓存**：它指的是Mybatis中sqlSession对象的缓存，当我们执行查询以后，查询的结果会同时存入到SqlSession为我们提供的一块区域中，该区域的结构是一个Map，当我们再次查询同样的数据，mybatis会先去sqlsession中查询是否有，有的话直接拿出来用，当SqlSession对象消失时，mybatis的一级缓存也就消失了，同时一级缓存是SqlSession范围的缓存，**当调用SqlSession的修改、添加、删除、commit(),close()等方法时，就会清空一级缓存。**

**二级缓存**：它指得是Mybatis中SqlSessionFactory对象的缓存，**由同一个SqlSessionFactory对象创建的SqlSession共享其缓存**，但是其中**缓存的是数据而不是对象**，通过存储的数据构造对象返回, 所以从二级缓存再次查询出得结果的对象与第一次存入的对象是不一样的。

> 因为mybatis缓存在本地, 所以在分布式情况下, 这种缓存策略存在不一致问题.

一级缓存命中情况:

```
session = factory.openSession();
userDao = session.getMapper(UserDao.class);
//第一次获取该用户
User user1 = userDao.findById(45);
System.out.println(user1);
//第二次获取该用户  从一级缓存中查询
User user2 = userDao.findById(45);
System.out.println(user2);
//true 因为一级缓存的是对象
System.out.println(user1 == user2); 
session.close();
```

清空一级缓存情况:

```
session = factory.openSession();
userDao = session.getMapper(UserDao.class);
//第一次获取该用户
User user1 = userDao.findById(45);
System.out.println(user1);
//  session.commit();        //调用SqlSession的commit方法清空缓存
user1.setUsername("更新用户");
userDao.updateUser(user1);   //通过更新SqlSession清空缓存
User user2 = userDao.findById(45);
System.out.println(user2);
//false 因为情况过缓存, 且对象已更新
System.out.println(user1 == user2); 
session.close();
```

二级缓存命中情况:

```
//第一次查询 并更新二级缓存
SqlSession session1 = factory.openSession();
UserDao userDao1 = session1.getMapper(UserDao.class);
User user1 = userDao1.findById(45);
System.out.println(user1);
session1.commit(); 			//commit()方法提交二级缓存 同时清空一级缓存
session1.close();			
//第二次查找命中二级缓存
SqlSession session2 = factory.openSession();
UserDao userDao2 = session2.getMapper(UserDao.class);
User user2 = userDao2.findById(45);
session2.commit(); 			//commit()方法提交二级缓存 同时清空一级缓存
session2.close();
System.out.println(user2); 
System.out.println(user1 == user2);	//false 因为二级缓存的是数据, 而不是对象
```



# Safe

## CSRF

**CSRF（Cross Site Request Forgery）**一般被翻译为 **跨站请求伪造** 。

https://tech.meituan.com/2018/10/11/fe-security-csrf.html

流程:

```
受害者登录a.com，并保留了登录凭证（Cookie）。
攻击者引诱受害者访问了b.com。
b.com 向 a.com 发送了一个请求：a.com/act=xx。浏览器会默认携带a.com的Cookie。
a.com接收到请求后，对请求进行验证，并确认是受害者的凭证，误以为是受害者自己发送的请求。
a.com以受害者的名义执行了act=xx。
攻击完成，攻击者在受害者不知情的情况下，冒充受害者，让a.com执行了自己定义的操作。
```

举个简单的例子：

这一天，小明同学百无聊赖地刷着Gmail邮件。大部分都是没营养的通知、验证码、聊天记录之类。但有一封邮件引起了小明的注意：[甩卖比特币，一个只要998！！], 小明当然知道这种肯定是骗子，但还是抱着好奇的态度点了进去。果然，这只是一个什么都没有的空白页面，小明失望的关闭了页面。一切似乎什么都没有发生……但小明的Gmail中，被偷偷设置了一个过滤规则，这个规则使得所有的邮件都会被自动转发到黑客的邮箱。

```
<form method="POST" action="https://mail.google.com/mail/h/ewt1jmuj4ddv/?v=prf" enctype="multipart/form-data"> 
    <input type="hidden" name="cf2_emc" value="true"/> 
    <input type="hidden" name="cf2_email" value="hacker@hakermail.com"/> 
    .....
    <input type="hidden" name="irf" value="on"/> 
    <input type="hidden" name="nvp_bu_cftb" value="Create Filter"/> 
</form> 
<script> 
    document.forms[0].submit();
</script>
```

这个页面只要打开，就会向Gmail发送一个post请求。请求中，执行了“Create Filter”命令，将所有的邮件，转发到action指定的邮箱.  请求发送时，携带着小明的登录凭证（Cookie），Gmail的后台接收到请求，验证了确实有小明的登录凭证，于是成功给小明配置了过滤器。

小明由于刚刚就登陆了Gmail，所以这个请求发送时，携带着小明的登录凭证（Cookie），Gmail的后台接收到请求，验证了确实有小明的登录凭证，于是成功给小明配置了过滤器。

黑客可以查看小明的所有邮件，包括邮件里的域名验证码等隐私信息。拿到验证码之后，黑客就可以要求域名服务商把域名重置给自己。

CSRF只能通过浏览器自己带上Cookie，不能操作Cookie来获取到Token并加到http请求的参数中。

## XSS

XSS攻击通常指的是利用网页开发时留下的漏洞, 通过巧妙的方法注入恶意指令代码到网页, 使用户加载并执行攻击者恶意制造的网页程序.

> token和cookie都不能避免**跨站脚本攻击（Cross Site Scripting）XSS** ,XSS 中攻击者会用各种方式将恶意代码注入到其他用户的页面中

比如攻击者在提交的表单中输入

```html
<script>
	window.location=http://假网站.com/login.html;
</script>
```

假网站的登录页面UI与主站完全相同, 而安全警惕性不高的用户通常不会考虑原因, 就再登录一次, 最后造成大量敏感数据失窃.

可通过对特殊字符进行转义解决.

```
&lt;script&gt;
	window.location=http://假网站.com/login.html;
&lt;/script&gt;
```

Q:什么时候进行XSS过滤呢?

A:

1.一定不要相信客户端数据, 表单欺诈

2.一定要在服务端做转义符转换和有效性校验(验证码)

org.springframework.web.util.HtmlUtils 提供了转义符转换功能

```
//转义符转换
String str = HtmlUtils.htmlEscape(str);
//反向转化
String str = HtmlUtils.htmlescape(str);
```



## JWT

**JWT(Json Web Token)**是一个经过加密的，包含用户信息的且具有时效性的固定格式字符串 

![image-20211206205223334](picture/image-20211206205223334.png)

**Header (标头)  ** : 描述 JWT 的元数据，定义了生成签名的算法以及 `Token` 的类型。

```
标头Header   : eyJhbGciOiJIUzI1NiJ9
{
  "alg": "HS256",
  "typ": "JWT"
}
```

**Payload  (载荷)**  : 用来存放实际需要传递的数据 , 它的 **iss**（发布者）， **exp**（到期时间）， **sub**（主题）等是非强制性的但建议使用的预定义要求.

```
载荷Payload    :  eyJzdWIiOiJ7XCJ1c.....
{
  "sub": "1234567890",
  "name": "John Doe",
  "admin": true,
  "exp": "...",
  ...
}
```

**Signature（签名）** ：服务器通过`Payload`、`Header`(先base64编码)和一个后端提供的密钥(`secret`)使用 `Header` 里面指定的签名算法（默认是 HMAC SHA256）生成。

```
签名Sign    :  NT8QBdoK4S-....
HMACSHA256(base64UrlEncode(header) + "." +  base64UrlEncode(payload),  secret)
```

> 只有在传递的原始数据和签名相同的情况,才认为这个JWT是合规的

> jwt 只有最后签名部是加密的, 因此前面两部分不能存放敏感信息(base64)

此外 jwt还支持设置过期时间.

[具体使用见](note/safe.md)

可以将 `Token` 保存在 Cookie 或者 localStorage 里面，以后客户端发出的所有请求都会携带这个令牌.

但通常更好的做法是放在 HTTP Header 的 Authorization 字段中：`Authorization: Bearer Token`。

### JWT认证

jwt根据验证签名的时机划分 , 认证方式有:

网关统一校验 :JWT校验无感知，验签过程无侵入，执行效率低，适用于低并发企业级应用(网关为瓶颈).

![image-20211206211024490](picture/image-20211206211024490.png)

应用认证方案 :控制更加灵活，有一定代码侵入，代码可以灵活控制，适用于追求性能互联网应用

![image-20211206211041243](picture/image-20211206211041243.png)

在应用认证中 , 如果后端请求只有部分需要认证 , 可通过AOP实现.

> 在两种方案中, 高并发环境下如何构建高可用的认证中心也是需要思考的问题

### 续签

JWT不设置过期时间行不行？

不行，会留下”太空垃圾”，后患无穷 , JWT不建议设置长时有效期  ,续签JWT必须有退出机制

**在不允许改变Token令牌下实现续签**

![image-20211206213233959](picture/image-20211206213233959.png)

> 该方案中, redis的value没有作用

> 为什么不直接将jwt存入redis key ,因为jwt占用空间大,  而加入客户端环境特征 , 可以避免认为盗取

> 缺陷: 将jwt相关信息存入redis 并设置过期时间 , 意味着jwt变为有状态的了



**允许前端改变Token令牌下实现续签**

登录后认证中心返回两个jwt , 后续请求时两个token都未过期才校验成功

1. 登录后30分钟内请求

![image-20211206213957833](picture/image-20211206213957833.png)

2.30-60分钟中间访问,access_token过期, 后端服务将refresh_token传入认证中心刷新接口 , 认证中心根据payload重新生成两个token,前端接收到后进行替换

![image-20211206214320228](picture/image-20211206214320228.png)

3. 60分钟之后访问直接认证失败.

>  此方案并没有在后台保存jwt相关信息 , 所以jwt保持了无状态特征

> 缺陷 : 客户端需要大量针对jwt续签的改造工作

存在重复生成JWT的问题, 两个access_token都过期且refresh_token都没过期的请求同时访问,会生成两组不同的JWT, 存在逻辑问题 , 但不影响使用.

![image-20211206220120405](picture/image-20211206220120405.png)





## SSO

SSO(Single Sign On)即单点登录说的是用户登陆多个子系统的其中一个就有权访问与其相关的其他系统。举个例子我们在登陆了京东金融之后，我们同时也成功登陆京东的京东超市、京东国际、京东生鲜等子系统。

> SSO与OAuth2的区别是:SSO是一种思想，或者说是一种解决方案，是抽象的; OAuth2是用来允许用户授权第三方应用访问他在另一个服务器上的资源的一种协议，它不是用来做单点登录的，但我们可以利用它来实现单点登录。

CAS(Central Authentication Service), 	旨在为 Web 应用系统提供一种可靠的单点登录方法. 允许用户访问多个应用程序只提供一次用户凭据（如用户名和密码）。它还允许Web应用程序对用户进行身份验证，而不必获取用户的安全凭证，比如密码。包含CAS client 和 CAS server

SSO单点登录访问流程主要有以下步骤：

1. 访问服务：SSO客户端发送请求访问应用系统提供的服务资源。
2. 定向认证：SSO客户端会重定向用户请求到SSO服务器。
3. 用户认证：用户身份认证。
4. 发放票据：SSO服务器会产生一个随机的Service Ticket。
5. 验证票据：SSO服务器验证票据Service Ticket的合法性，验证通过后，允许客户端访问服务。
   1. 传输用户信息：SSO服务器验证票据通过后，传输用户认证结果信息给客户端。

## OAuth2

OAuth 是一个行业的标准授权协议，主要用来授权第三方应用获取有限的权限。

实际上它就是一种授权机制，它的最终目的是**为第三方应用颁发一个有时效性的令牌 Token**，使得第三方应用能够通过该令牌获取相关的资源。

```
resource owner，资源所有者，能够允许访问受保护资源的实体。如果是个人，被称为 end-user。
resource server，资源服务器，托管受保护资源的服务器。
client，客户端，使用资源所有者的授权代表资源所有者发起对受保护资源的请求的应用程序。如：web网站，移动应用等。
authorization server，授权服务器，能够向客户端颁发令牌。
user-agent，用户代理，帮助资源所有者与客户端沟通的工具，一般为 web 浏览器，移动 APP 等。
```

协议流程:

![oauth2-roles](picture/oauth2-roles.jpg)

- (A) Client 请求 Resource Owner 的授权。授权请求可以直接向 Resource Owner 请求，也可以通过 Authorization Server 间接的进行。
- (B) Client 获得授权许可。
- (C) Client 向 Authorization Server 请求访问令牌。
- (D) Authorization Server 验证授权许可，如果有效则颁发访问令牌。
- (E) Client 通过访问令牌从 Resource Server 请求受保护资源。
- (F) Resource Server 验证访问令牌，有效则响应请求。

**授权**

一个客户端想要获得授权，就需要先到服务商那注册你的应用。一般需要你提供下面这些信息：

- 应用名称
- 应用网站
- 重定向 URI 或回调 URL（redirect_uri）

重定向 URI 是服务商在用户授权（或拒绝）应用程序之后重定向用户的地址，因此也是用于处理授权代码或访问令牌的应用程序的一部分。在你注册成功之后，你会从服务商那获取到你的应用相关的信息：

- 客户端标识 client_id
- 客户端密钥 client_secret

`client_id` 用来表识客户端（公开），`client_secret` 用来验证客户端身份（保密）。

[**OAuth 2.0 **](https://oauth.net/2/)规定了四种获得令牌(授权)的流程:

1.**授权码** : 指的是第三方应用先申请一个授权码，然后再用该码获取令牌。

[GitHub OAuth 第三方登录示例](http://www.ruanyifeng.com/blog/2019/04/github-oauth.html)

具体流程:

1. 资源拥有者（用户）需要登录客户端（APP），他选择了第三方登录。
2. 客户端（APP）重定向到第三方授权服务器。此时客户端携带了客户端标识（client_id），那么第三方就知道这是哪个客户端，资源拥有者确定（拒绝）授权后需要重定向到哪里。
3. 用户确认授权，客户端（APP）被重定向到注册时给定的 URI，并携带了第三方给定的 code。
4. 在重定向的过程中**，客户端拿到 code 与 `client_id`、`client_secret` 去授权服务器请求令牌，如果成功，直接请求资源服务器获取资源，整个过程，用户代理是不会拿到令牌 token 的**。
5. 客户端（APP）拿到令牌 token 后就可以向第三方的资源服务器请求资源了。

当我想在 coding 上通过 github 账号登录时：

①`GET 请求` 点击登录，重定向到 github 的授权端点：

```
https://github.com/login/oauth/authorize?
  response_type=code&
  client_id=a5ce5a6c7e8c39567ca0&
  redirect_uri=https://coding.net/api/oauth/github/callback&
  scope=user:email
```

| 字段          | 描述                                           |
| :------------ | :--------------------------------------------- |
| response_type | 必须，固定为 code，表示这是一个授权码请求。    |
| client_id     | 必须，在 github 注册获得的客户端 ID。          |
| redirect_uri  | 可选，接受或拒绝注册后的跳转网址的重定向 URI。 |
| scope         | 可选，请求资源范围，多个空格隔开。             |
| state         | 可选（推荐），如果存在，原样返回给客户端。     |

用户确认授权后, 返回code

```
https://coding.net/api/oauth/github/callback?code=fb6a88dc09e843b33f&scope=user:email
```

②`POST 请求` ，当获取到授权码 code 后，客户端需要用它获取令牌 token：

```
https://github.com/login/oauth/access_token?
  client_id=a5ce5a6c7e8c39567ca0&
  client_secret=xxxx&
  grant_type=authorization_code&
  code=fb6a88dc09e843b33f&
  redirect_uri=https://coding.net/api/oauth/github/callback
```

| 字段          | 描述                                                         |
| :------------ | :----------------------------------------------------------- |
| grant_type    | 必须，固定为 authorization_code (刷新的时候为refresh_token)。 |
| code          | 必须，上一步获取到的授权码。                                 |
| redirect_uri  | 必须（如果请求/authorize接口有），完成授权后的回调地址，与注册时一致。 |
| client_id     | 必须，客户端标识。                                           |
| client_secret | 必须，客户端密钥。                                           |

返回值：

```
{
  "access_token":"a14afef0f66fcffce3e0fcd2e34f6ff4",
  "token_type":"bearer",
  "expires_in":3920,
  "refresh_token":"5d633d136b6d56a41829b73a424803ec"
}
```

| 字段          | 描述                                            |
| :------------ | :---------------------------------------------- |
| access_token  | 这个就是最终获取到的令牌。                      |
| token_type    | 令牌类型，常见有 bearer/mac/token（可自定义）。 |
| expires_in    | 失效时间。                                      |
| refresh_token | 刷新令牌，用来刷新 access_token。               |

③获取资源服务器资源，拿着 access_token 就可以获取账号的相关信息了：

```
curl -H "Authorization: token a14afef0f66fcffce3e0fcd2e34f6ff4" https://api.github.com/user
```

④`POST 请求` 刷新令牌

我们的 access_token 是有时效性的，当在获取 github 用户信息时，如果返回 token 过期：

```
https://github.com/login/oauth/access_token?
  client_id=a5ce5a6c7e8c39567ca0&
  client_secret=xxxx&
  redirect_uri=https://coding.net/api/oauth/github/callback&
  grant_type=refresh_token&
  refresh_token=5d633d136b6d56a41829b73a424803ec
```

| 字段          | 描述                             |
| :------------ | :------------------------------- |
| redirect_uri  | 必须                             |
| grant_type    | 必须，固定为 refresh_token       |
| refresh_token | 必须，上面获取到的 refresh_token |

返回值：

```
{
  "access_token":"a14afef0f66fcffce3e0fcd2e34f6ee4",
  "token_type":"bearer",
  "expires_in":3920,
  "refresh_token":"4a633d136b6d56a41829b73a424803vd"
}
```

> refresh_token 只有在 access_token 过期时才能使用，并且只能使用一次。当换取到的 access_token 再次过期时，使用新的 refresh_token 来换取 access_token



2.**隐藏模式**: 要求用户授权应用程序，然后授权服务器将访问令牌传回给用户代理，用户代理将其传递给客户端。

①同样以 coding 和 github 为例 ( response_type变为了token 表示要求直接返回token)：

```
https://github.com/login/oauth/authorize?
  response_type=token&
  client_id=a5ce5a6c7e8c39567ca0&
  redirect_uri=https://coding.net/api/oauth/github/callback&
  scope=user:email
```

| 字段          | 描述                                                      |
| :------------ | :-------------------------------------------------------- |
| response_type | 必须，固定为 token。                                      |
| client_id     | 必须。当客户端被注册时，有授权服务器分配的客户端标识。    |
| redirect_uri  | 可选。由客户端注册的重定向URI。                           |
| scope         | 可选。请求可能的作用域。                                  |
| state         | 可选(推荐)。任何需要被传递到客户端请求的URI客户端的状态。 |

返回值：

```
https://coding.net/api/oauth/github/callback#
  access_token=a14afef0f66fcffce3e0fcd2e34f6ff4&
  token_type=token&
  expires_in=3600
  scope=user:email
```

| 字段         | 描述                                                      |
| :----------- | :-------------------------------------------------------- |
| access_token | 必须。授权服务器分配的访问令牌。                          |
| token_type   | 必须。令牌类型。                                          |
| expires_in   | 推荐，访问令牌过期的秒数。                                |
| scope        | 可选，访问令牌的作用域。                                  |
| state        | 必须，如果出现在授权请求期间，和请求中的 state 参数一样。 |

②用户代理提取令牌 token 并提交给 coding

③coding 拿到 token 就可以获取用户信息了

```
curl -H "Authorization: token a14afef0f66fcffce3e0fcd2e34f6ff4" https://api.github.com/user
```



3.**资源所有者密码模式**: 用户将其服务凭证（用户名和密码）直接提供给客户端 (**高度信任下**) ，该客户端使用凭据从服务获取访问令牌。

`POST 请求` 密码凭证流程

```
https://oauth.example.com/token?grant_type=password&username=USERNAME&password=PASSWORD&client_id=CLIENT_ID
```

| 字段       | 描述                                 |
| :--------- | :----------------------------------- |
| grant_type | 必须，固定为 password。              |
| username   | 必须，UTF-8 编码的资源拥有者用户名。 |
| password   | 必须，UTF-8 编码的资源拥有者密码。   |
| scope      | 可选，授权范围。                     |

返回值：

```
{ 
  "access_token"  : "...",
  "token_type"    : "...",
  "expires_in"    : "...",
  "refresh_token" : "...",
}
```

如果授权服务器验证成功，那么将直接返回令牌 token，该客户端已被授权。



4.**客户端模式** : 这种模式只需要提供 `client_id` 和 `client_secret` 即可获取授权。一般用于后端 API 的相关操作。

`POST 请求` 客户端凭证流程：

```
https://oauth.example.com/token?grant_type=client_credentials&client_id=CLIENT_ID&client_secret=CLIENT_SECRET
```

| 字段       | 描述                          |
| :--------- | :---------------------------- |
| grant_type | 必须。固定client_credentials. |
| scope      | 可选。授权的作用域。          |

返回值

```
{ 
  "access_token"  : "...",
  "token_type"    : "...",
  "expires_in"    : "...",
}
```

如果授权服务器验证成功，那么将直接返回令牌 token，改客户端已被授权。

# shiro

## Authentication

Subject.login -->  SecurityManager.login(token) -->	Authenticator.doAuthenticate(token)   --> Realm.getAuthenticationInfo(token)

**AuthenticationToken**

`UsernamePasswordToken`

**RemenberMe**

- **Remembered**: A remembered `Subject` is not anonymous and has a known identity (i.e. `subject.`[`getPrincipals()`](https://shiro.apache.org/static/current/apidocs/org/apache/shiro/subject/Subject.html#getPrincipals--) is non-empty). But this identity is remembered from a previous authentication during a **previous** session. A subject is considered remembered if `subject.`[`isRemembered()`](https://shiro.apache.org/static/current/apidocs/org/apache/shiro/subject/Subject.html#isRemembered--) returns `true`.
- **Authenticated**: An authenticated `Subject` is one that has been successfully authenticated (i.e. the `login` method was invoked without throwing an exception) *during the Subject’s current session*. A subject is considered authenticated if `subject.`[`isAuthenticated()`](https://shiro.apache.org/static/current/apidocs/org/apache/shiro/subject/Subject.html#isAuthenticated--) returns `true`.

Remenbered的标识使系统知道该用户可能是谁，但实际上，无法绝对保证记住的主题是否代表预期的用户。一旦Subject通过身份验证，他们就不再被视为仅被记住，因为他们的身份将在**当前会话期间**得到验证。

因此，尽管应用程序的许多部分仍然可以基于Remenbered的主体（如自定义视图）执行特定于用户的逻辑，但在用户通过执行成功的身份验证尝试合法地验证其身份之前，它通常不应执行高度敏感的操作。

> Authenticated和Remenbered经过身份验证的状态是互斥的 - 一个为 true 的值表示另一个状态为 false 值，反之亦然。

由于 Web 应用程序中 remenbered 的身份通常与 Cookie 一起保留，并且 Cookie 只能在提交响应正文之前删除，因此强烈建议在调用 subject.logout（） 后立即将最终用户重定向到新的视图或页面。这保证了任何与安全相关的 Cookie 都会按预期删除。这是对HTTP cookie功能的限制，而不是Shiro的限制。



**Authenticator**

the Shiro `SecurityManager` implementations default to using a [`ModularRealmAuthenticator`](https://shiro.apache.org/static/current/apidocs/org/apache/shiro/authc/pam/ModularRealmAuthenticator.html) instance.

如果只有单个Realm,  ModularRealmAuthenticator 进行直接调用;  如果有多个Realm , ModularRealmAuthenticator 实例将利用其配置的 AuthenticationStrategy 启动多Realm认证尝试



**AuthenticationStrategy**

| `AuthenticationStrategy` class                               | Description                                                  |
| :----------------------------------------------------------- | :----------------------------------------------------------- |
| [`AtLeastOneSuccessfulStrategy`](https://shiro.apache.org/static/current/apidocs/org/apache/shiro/authc/pam/AtLeastOneSuccessfulStrategy.html) | 如果一个（或多个）Realm 成功通过身份验证，则整体尝试将被视为成功。如果没有身份验证成功，则尝试将失败。 |
| [`FirstSuccessfulStrategy`](https://shiro.apache.org/static/current/apidocs/org/apache/shiro/authc/pam/FirstSuccessfulStrategy.html) | 只有从第一个成功通过身份验证的 Realm 返回的信息才会被使用。所有进一步的 Realm 都将被忽略。如果没有身份验证成功，则尝试将失败。 |
| [`AllSuccessfulStrategy`](https://shiro.apache.org/static/current/apidocs/org/apache/shiro/authc/pam/AllSuccessfulStrategy.html) | 所有配置的Realm必须都认证成功时, 整体尝试才被视为成功, 如果任一验证失败, 则尝试将失败 |

如果你想自己创建自己的 AuthenticationStrategy 实现，你可以使用` org.apache.shiro.authc.pam.AbstractAuthenticationStrategy` 作为起点。其会自动实现"捆绑"/聚合行为，即将每个 Realm 的结果合并到一个 AuthenticationInfo 实例中。



**Realm Order**

需要指出的是，ModularRealmAuthenticator 将按迭代顺序与 SecurityManager上的Realm 实例进行交互，这一点非常重要。

realm的顺序可以通过定义的形式隐式声明 或 通过 securityManager.realms属性显式声明.



## Authorization

授权核心3元素: 权限, 角色, 用户.

执行授权的三种方式:

1.**以编程方式** (Subject方法)

角色检查:

```
hasRole(String roleName) 和 hasAllRoles(Collection<String> roleNames), 如果subject分配了指定参数的一个或多个角色, 返回true;
hasRole(List<String> roleNames) 返回方法参数中满足hasRole(String roleName)的结果数组
```

角色断言(比角色检查简洁,不需要构建自己的异常):

If the `Subject` does not have the expected role, an `AuthorizationException` will be thrown.

```
checkRole(String roleName), checkRoles(Collection<String> roleNames) 和 checkRoles(String... roleNames),
如果subject分配了指定参数中的角色,则静默返回, 否则异常.
```

权限检查(基于权限的授权与应用程序的原始功能密切相关,且代码有改动时修改粒度更细):

执行权限检查可通过`org.apache.shiro.authz.Permission `接口的实例，并将其传递给接受权限实例的 isPersmited 方法。如:

```
currentUser.isPermitted(new PrinterPermission("laserjet4400n", "print"))
```

```
isPermitted(Permission p)和isPermittedAll(Collection<Permission> perms),如果subject包含指定的permission实例的权限汇总,则返回true;
isPermittedAll(List<Permission> perms)返回方法参数中满足isPermitted(Permission p)的结果数组
```

基于对象的权限过于"沉重", 可使用通过字符串的方式表示权限实例,下面方式与上述基于对象的权限方式效果相同

```
currentUser.isPermitted("printer:print:laserjet4400n")
```

该方式其实是由`org.apache.shiro.authz.permission.WildcardPermission`实现,相当于

```
currentUser.isPermitted(new WildcardPermission("printer:print:laserjet4400n"))
```

> Q: 为什么要对字符串权限转换为WildcardPermission对象?
>
> 因为权限检查是通过隐含逻辑, 而不是简单的字符串相等性检查来评估的。比如支持*通配符检查
>
> 关于通配符权限字符串的使用, 具体见官方https://shiro.apache.org/permissions.html

权限断言:

```
checkPermission(Permission p)
checkPermission(String perm)
checkPermissions(Collection<Permission> perms)
checkPermissions(String... perms)
如果subject分配了指定参数中的权限,则静默返回, 否则异常
```

2.**java注解方式**

`@RequiresAuthentication`：表示当前Subject已经通过login进行了身份验证；即 Subject. isAuthenticated() 返回 true

```
//注解相当于以下方法内代码
if (!SecurityUtils.getSubject().isAuthenticated()) {
        throw new AuthorizationException(...);
}
```

`@RequiresGuest`：表示当前Subject没有**身份验证**或通过**记住我**登录过, 即游客身份。

```
PrincipalCollection principals = SecurityUtils.getSubject().getPrincipals();
if (principals != null && !principals.isEmpty()) {
    throw new AuthorizationException(...);
}
```

`@RequiresPermissions`:  表示要求当前subject具有一个或多个权限才能执行带批注的方法, 可通过logical属性指定与还是或

```
@RequiresPermissions("account:create")
//相当于方法内代码
 Subject currentUser = SecurityUtils.getSubject();
if (!subject.isPermitted("account:create")) {
    throw new AuthorizationException(...);
}
```

```
@RequiresPermissions (value={“user:a”, “user:b”},logical= Logical.OR)：表示当前 Subject 需要权限 user:a 或user:b。
```

`@RequiresRoles`与`@RequiresPermissions`类似

`@RequiresUser`：表示当前 Subject 已经身份验证或者通过记住我登录的。

```
PrincipalCollection principals = SecurityUtils.getSubject().getPrincipals();
if (principals == null || principals.isEmpty()) {
    throw new AuthorizationException(...);
}
```

3.**JSP/GSP 标签**

见https://shiro.apache.org/web.html#Web-taglibrary的JSP/GSP Tag Library章节



流程:

代码调用hasRole,isPermitted...变体(如果有, 如注解 )  -->  Subject.hasRole,isPermitted.....   -->

SecurityManager.hasRole,isPermitted... (实现了Authorizer接口)-->

SecurityManager方法内部委托给Authorizer实例(默认为ModularRealmAuthorizer)调用, 它支持在任何授权操作期间协调一个或多个 Realm 实例。 -->

ModularRealmAuthorizer再委托给其内部的实现了Authorizer的Realm实例集合调用



**ModularRealmAuthorizer** 

SecurityManager 实现默认使用 ModularRealmAuthorizer 实例。ModularRealmAuthorizer 同样支持具有单个 Realm 和具有多个 Realm 的应用程序。

对于任何授权操作，ModularRealmAuthorizer 将循环访问其内部 Realm 集合，并按迭代顺序与每个 Realm 进行交互(如果没实现Authorizer接口则跳过)。

在Realm的处理过程中:

1.如果 Realm 的 hasRole* 或 isPersmited* 变体方法返回true，则Realm方法立即返回true，并且所有剩余的 Realm 都将短路。这意味着只要一个Realm允许即可。

2.如果 Realm方法导致异常，则该异常将作为授权异常传播到Subject调用方。

> Q: 我们通常自定义的Realm实例与Authorizer接口实例有啥关系?
>
> A: 如果要让Realm支持授权操作, 需要实现Realm的抽象子类AuthorizingRealm, 该抽象类实现了Authorizer接口的方法, 并以模板设计模式方式提供了子类实现的getAuthorizationInfo(token)方法用于获取授权信息, 在实现的Authorizer接口的方法中进行调用.



**PermissionResolver**

在使用字符串权限方法的时候, 需要PermissionResolver来将字符串转换为Permission实例.  所有Realm的实现内部默认为用来处理`WildcardPermission`字符串格式的`WildcardPermissionResolver`

如果想要自定义权限语法并创建对应的`PermissionResolver`实现, 为了让所有已配置的Realm实例都支持该语法, 可配置全局`PermissionResolver` 到`securityManager.authorizer.permissionResolver`上, 为了让Realm实例中使用此`PermissionResolver`, 还必须要给Realm实现`PermissionResolverAware`接口.

>  也可以单独为某个Realm指定`PermissionResolver` .



> 最后, 如果你的应用程序使用多个Realm来执行授权，并且 ModularRealmAuthorizer 的默认简单基于迭代的短路授权行为不适合您的需要，则您可能希望创建一个自定义Authorizer并相应地配置 SecurityManager。

## Realm

Realm 用于将访问特定于应用程序的安全数据(例如用户、角色和权限)数据转换为 Shiro 理解的格式. Realm通常与数据源(jdbc,文件io等数据访问api)具有一对一的相关性. 所以Realm本质上是一个特定于安全性的 DAO

**Realm Authentication**

在 Realm 执行认证之前，会调用其 support 方法。如果返回值为 true，则只有这样才会调用其 getAuthenticationInfo（token） 方法。

support方法检查是否支持处理所提交令牌的类型.()

如果支持, Authenticator 将调用Realm的 getAuthenticationInfo（token） 方法 , 对token中的principal 与 数据源中查找到的账户数据(凭据)进行匹配, 如果匹配, 则返回一个`AuthenticationInfo`实例 , 该实例即**按Shiro理解的格式封装账户数据**.   如果不匹配, 可抛出shiro认证异常.

> 通常开发者习惯继承Realm子类AuthorizingRealm实现通用的身份验证和授权工作流

**Credentials Matching**

在Realm的认证中, 需要对账户凭证数据进行匹配 . 为了提供可扩展性,  `AuthenticatingRealm`和他的子类提供了`CredentialsMatcher`用于比较凭证.

Shiro提供了`SimpleCredentialsMatcher`和子类`HashedCredentialsMatcher`开箱即用.

Shiro 的所有开箱即用的 Realm 实现都默认使用 SimpleCredentialsMatcher。

`SimpleCredentialsMatcher `对存储的帐户凭据与身份验证令牌中提交的内容执行简单的直接相等性检查。  

`HashedCredentialsMatcher`在`SimpleCredentialsMatcher`上增加了加密哈希策略(前提需要对凭据进行数据存储前进行了单向哈希处理), shiro提供了支持多种哈希算法的子类供使用, 如使用SHA-256方式

```
//入库
String hashedPasswordBase64 = new Sha256Hash(plainTextPassword, salt, 1024).toBase64();
User user = new User(username, hashedPasswordBase64);
user.setPasswordSalt(salt);
userDAO.create(user);
//对应的凭证匹配器为
org.apache.shiro.authc.credential.Sha256CredentialsMatcher
```



也可以自定义实现.

通过下列方式进行设置

```
AuthenticatingRealm.setCredentialsMatcher(customMatcher)
```



**Realm Authentication**

SecurityManager 将授权检查任务委派给Authorizer，默认为 ModularRealmAuthorizer。

## Session Manager

**Session**

shiro Session支持自定义存储,异构客户端访问,事件监听,主机和IP地址保留,回话过期延长,无侵入加入web应用,可用于SSO.

```
Subject currentUser = SecurityUtils.getSubject();
Session session = currentUser.getSession();
//上面相当于
Session session = currentUser.getSession(true);
```

`currentUser.getSession(boolean create)`与`HttpServletRequest.getSession(boolean create)`功能相同, 但shiro的能应用于非Web应用:

- 如果session已存在,则忽略boolean返回session; 
- 如果session不存在,boolean为true则创建session返回;
- 如果session不存在,boolean为false则返回null(通常用于阻止创建没必要的session);

**Session Manager**

SessionManager 是由 SecurityManager 维护的顶级组件。   默认DefaultSessionManager提供了应用程序所需的会话管理功能，如会话验证、孤立清理等。可通过setter修改.

**Session Timeout**

SessionManager实现默认为30分组session timeout, 可通过globalSessionTimeout属性修改. 也可以设置单个session timeout.

**Session Listener**

实现 SessionListener 接口（或继承SessionListenerAdapter）实现对session事件(任一session)进行响应。   

默认的 SessionManager sessionListeners 属性是一个集合,  因此可以使用一个或多个侦听器实现来配置 SessionManager

**Session Storage (SessionDao)**

SessionManager 实现将session的CRUD 操作委托给内部组件 SessionDAO.

实现该接口可以将session驻留在特定数据源中.

> 默认情况下，Web 应用程序不使用本机会话管理器，而是保留 Servlet 容器的默认会话管理器，该管理器不支持 SessionDAO。如果要在基于 Web 的应用程序中为自定义会话存储或会话群集启用 SessionDAO，则必须首先配置本机 Web 会话管理器`rg.apache.shiro.web.session.mgt.DefaultWebSessionManager`

Shiro 可以启用EHCache SessionDAO 将会话存储在内存中，并支持在内存受限时溢出到磁盘.(shiro官方建议开启).

这是需要通过`EnterpriseCacheSessionDAO`使用`EhCacheManager`来实现存储session.

**Session ID**

SessionDAO 使用 `SessionIdGenerator `组件在每次创建新会话时生成新的会话 ID.默认实现为`JavaUuidSessionIdGenerator`，它基于 Java UUID 生成字符串 ID.

**Session Validation&Scheduling**

验证session以便删除无效的session

> 出于性能原因, session验证仅发生在session访问的时候, 如Subject.getSession(), 这意味着如果没有额外的定期验证, 孤立session项将开始填满session存储.

为了防止孤立session项堆积，SessionManager 实现支持` SessionValidationScheduler `负责定期验证会话，以确保在必要时清理它们.

默认 SessionValidationScheduler 为 `ExecutorServiceSessionValidationScheduler`，它使用 JDK `ScheduledExecutorService` 来控制验证的频率。   默认情况下，此实现将每小时执行一次验证。可以通过指定新的`ExecutorServiceSessionValidationScheduler`实例并指定不同的间隔 (interval) 来更改验证发生的速率.

> 如果想设置一个不受 Shiro 控制的进程来执行验证。例如，使用redis缓存并依靠缓存的"生存时间"设置来自动清除旧session; 或者，设置了一个 cron 作业来自动清除自定义数据存储。在这些情况下，您可以关闭session验证计划: 通过设置`securityManager.sessionManager.sessionValidationSchedulerEnabled`为false, 但这不会禁用session访问时的验证.

**Invalid Session Delete**

默认情况下，每当 Shiro 检测到无效session时，都会尝试通过 SessionDAO.delete（session） 方法将其从session存储中删除。

如果某些应用程序可能不希望 Shiro 自动删除session。例如，如果应用程序提供了支持可查询的 SessionDAO，并希望旧的或无效的session在一段时间内可用。这将允许团队对数据存储运行查询，例如，查看用户在上周创建了多少个session，或者用户session的平均持续时间或类似的报告类型查询。

可通过`securityManager.sessionManager.deleteInvalidSessions`禁用shiro自动删除无效session.

# Spring Security

## UserDetailsManager

`UserDetailsManager` 负责对安全用户实体抽象 UserDetails 的增删查改操作

默认情况下, 在`org.springframework.boot.autoconfigure.security.servlet.UserDetailsServiceAutoConfiguration`

中会注入一个 `InMemoryUserDetailsManager `,     并指定一个User对象( 用户信息主题UserDetails的实现类) , 这个User用户名为user,  密码在控制台可以看到.



## PasswordEncoder

 `PasswordEncoder` 就是我们对密码进行编码 的工具接口。该接口只有两个功能： 一个是匹配验证。另一个是密码编码。

在`WebSecurityConfigurerAdapter`中对`PasswordEncoder`进行了懒加载: 

```
// 获取最终干活的PasswordEncoder
private PasswordEncoder getPasswordEncoder() {
	if (this.passwordEncoder != null) {
		return this.passwordEncoder;
	}
	PasswordEncoder passwordEncoder = getBeanOrNull(PasswordEncoder.class);
	if (passwordEncoder == null) {
		passwordEncoder = PasswordEncoderFactories.createDelegatingPasswordEncoder();
	}
	this.passwordEncoder = passwordEncoder;
	return passwordEncoder;
}
// 从Spring IoC容器中获取Bean 有可能获取不到
private <T> T getBeanOrNull(Class<T> type) {
	try {
		return this.applicationContext.getBean(type);
	} catch(NoSuchBeanDefinitionException notFound) {
	return null;
	}
}
```

其中, 如果能从从Spring IoC容器中获取 PasswordEncoder 的Bean就用该Bean作为编码器，没有就通过`PasswordEncoderFactories.createDelegatingPasswordEncoder();` 获取

```
public static PasswordEncoder createDelegatingPasswordEncoder() {
	String encodingId = "bcrypt";
	Map<String, PasswordEncoder> encoders = new HashMap<>();
	encoders.put(encodingId, new BCryptPasswordEncoder());
	encoders.put("ldap", new org.springframework.security.crypto.password.LdapShaPasswordEncoder());
	encoders.put("MD4", new org.springframework.security.crypto.password.Md4PasswordEncoder());
	encoders.put("MD5", new org.springframework.security.crypto.password.MessageDigestPasswordEncoder("MD5"));
	encoders.put("noop",org.springframework.security.crypto.password.NoOpPasswordEncoder.getInstance());
	encoders.put("pbkdf2", new Pbkdf2PasswordEncoder());
	encoders.put("scrypt", new SCryptPasswordEncoder());
	encoders.put("SHA-1", new org.springframework.security.crypto.password.MessageDigestPasswordEncoder("SHA1"));
	encoders.put("SHA-256", new org.springframework.security.crypto.password.MessageDigestPasswordEncoder("SHA256"));	
	encoders.put("sha256", new org.springframework.security.crypto.password.StandardPasswordEncoder());
	return new DelegatingPasswordEncoder(encodingId, encoders);

```

返回的是一个`DelegatingPasswordEncoder`委托密码编码器对象, 继承了多种编码方式, 默认采用bcrypt进行编码(通过传入encodingId). 

> 因为先会从ioc中获取编码器, 所以我们也可以直接注入一个PasswordEncoder类型的bean, 来替换掉默认的PasswordEncoder

## 自动配置入口

在spring boot中, spring security是通过`org.springframework.boot.autoconfigure.security.servlet.SecurityAutoConfiguration`来完成自动配置的.

在这个配置类中, 除了使用@Import导入了其他三个配置类`SpringBootWebSecurityConfiguration 、 WebSecurityEnablerConfiguration 和 SecurityDataConfiguration` 以外, 还将 `DefaultAuthenticationEventPublisher` 作为默认的` AuthenticationEventPublisher` 注入 Spring IoC 容器.  熟悉spring的我一看就知道是事件发布器.

该类内置 了一个 HashMap> 维护了认证异常处理和对应异常事件处理逻辑的映射关系，比如账户过期异常 AccountExpiredException 对 应认证过期事件 AuthenticationFailureExpiredEvent ，也就是说发生不同认证的异常使用不同处理策略。

 1.**SpringBootWebSecurityConfiguration**

```
@Configuration
@ConditionalOnClass(WebSecurityConfigurerAdapter.class)
@ConditionalOnMissingBean(WebSecurityConfigurerAdapter.class)
@ConditionalOnWebApplication(type = Type.SERVLET)
public class SpringBootWebSecurityConfiguration {
	@Configuration
	@Order(SecurityProperties.BASIC_AUTH_ORDER)
	static class DefaultConfigurerAdapter extends WebSecurityConfigurerAdapter {
	}
}
```

默认情况下 `DefaultConfigurerAdapter` 将以 `SecurityProperties.BASIC_AUTH_ORDER` （ -5 ） 的顺序注入 Spring IoC 容器，这是个空实现。 **如果我们需要个性化可以通过继承 `WebSecurityConfigurerAdapter` 来实现**。

2.**WebSecurityEnablerConfiguration**

```
@Configuration
@ConditionalOnBean(WebSecurityConfigurerAdapter.class)
@ConditionalOnMissingBean(name = BeanIds.SPRING_SECURITY_FILTER_CHAIN)
@ConditionalOnWebApplication(type = ConditionalOnWebApplication.Type.SERVLET)
@EnableWebSecurity
public class WebSecurityEnablerConfiguration {
}
```

这个配置类的作用仅仅是为了启动**@EnableWebSecurity**:

```
@Retention(value = java.lang.annotation.RetentionPolicy.RUNTIME)
@Target(value = { java.lang.annotation.ElementType.TYPE })
@Documented
@Import({ WebSecurityConfiguration.class,
	SpringWebMvcImportSelector.class,
	OAuth2ImportSelector.class })
@EnableGlobalAuthentication
@Configuration
public @interface EnableWebSecurity {
	boolean debug() default false;
}
```

又导入了`WebSecurityConfiguration, SpringWebMvcImportSelector, OAuth2ImportSelector` 和启动了`@EnableGlobalAuthentication`

① **WebSecurityConfiguration**

该配置类使用一个 `WebSecurity` 对象基于用户指定的或者默认的安全配置，你可以通过继承 `WebSecurityConfigurerAdapter` 或者实现 `WebSecurityConfigurer `来定制 `WebSecurity` 创建一个 `FilterChainProxy` Bean来对用户请求进行安全过滤。这个 `FilterChainProxy` 的名称就是` WebSecurityEnablerConfiguration` 上的 `BeanIds.SPRING_SECURITY_FILTER_CHAIN `也就是 springSecurityFilterChain ,它是一个 Filter，**最终会被作为Servlet过滤器链中的一个Filter应用到Servlet容器中**。安全处理的策略主要是过滤器的调用顺序。

```java
@Configuration
public class WebSecurityConfiguration implements ImportAware,BeanClassLoaderAware {
	private WebSecurity webSecurity;
	// 是否启用了调试模式，来自注解 @EnableWebSecurity 的属性 debug，缺省值 false
	private Boolean debugEnabled;
	private List<SecurityConfigurer<Filter, WebSecurity>> webSecurityConfigurers;
	private ClassLoader beanClassLoader;
	@Autowired(required = false)
	private ObjectPostProcessor<Object> objectObjectPostProcessor;
	/**
	* 代理监听器 监听 DefaultAuthenticationEventPublisher 的一些处理策略
	*/
	@Bean
	public static DelegatingApplicationListener delegatingApplicationListener(){
		return new DelegatingApplicationListener();
	}
	/**
	* 安全SpEL表达式处理器 SecurityExpressionHandler 缺省为一个DefaultWebSecurityExpressionHandler
	*/
	@Bean
	@DependsOn(AbstractSecurityWebApplicationInitializer.DEFAULT_FILTER_NAME)
	public SecurityExpressionHandler<FilterInvocation> webSecurityExpressionHandler() {
		return webSecurity.getExpressionHandler();
	}
	/**
	* Spring Security核心过滤器链
	*/
	@Bean(name = AbstractSecurityWebApplicationInitializer.DEFAULT_FILTER_NAME)
	public Filter springSecurityFilterChain() throws Exception {
		boolean hasConfigurers = webSecurityConfigurers != null && !webSecurityConfigurers.isEmpty();
		if (!hasConfigurers) {
			WebSecurityConfigurerAdapter adapter = objectObjectPostProcessor
				.postProcess(new WebSecurityConfigurerAdapter() {
				});
			webSecurity.apply(adapter);
		}
		return webSecurity.build();
	}
	/**
	* 用于模板 如JSP Freemarker 的一些页面标签按钮控制支持
	*/
	@Bean
	@DependsOn(AbstractSecurityWebApplicationInitializer.DEFAULT_FILTER_NAME)
	public WebInvocationPrivilegeEvaluator privilegeEvaluator() throws Exception{
		return webSecurity.getPrivilegeEvaluator();
	}
	/**
	* webSecurity应用SecurityConfigurer实例，
	* 注入参数的依赖对应的bean autowiredWebSecurityConfigurersIgnoreParents也在该类中定义
	*/
	@Autowired(required = false)
	public void setFilterChainProxySecurityConfigurer(
			ObjectPostProcessor<Object> objectPostProcessor,
			@Value("#
				{@autowiredWebSecurityConfigurersIgnoreParents.getWebSecurityConfigurers()}")
				List<SecurityConfigurer<Filter, WebSecurity>> webSecurityConfigurers)throws Exception {
		webSecurity = objectPostProcessor.postProcess(new WebSecurity(objectPostProcessor));
		if (debugEnabled != null) {
		webSecurity.debug(debugEnabled);
		}
		Collections.sort(webSecurityConfigurers,AnnotationAwareOrderComparator.INSTANCE);
		Integer previousOrder = null;
		Object previousConfig = null;
		for (SecurityConfigurer<Filter, WebSecurity> config : webSecurityConfigurers) {
			Integer order = AnnotationAwareOrderComparator.lookupOrder(config);
			if (previousOrder != null && previousOrder.equals(order)) {
				throw new IllegalStateException(
					"@Order on WebSecurityConfigurers must be unique. Order of "
					+ order + " was already used on " +
					previousConfig + ", so it cannot be used on "
					+ config + " too.");
			}
			previousOrder = order;
			previousConfig = config;
		}
		for (SecurityConfigurer<Filter, WebSecurity> webSecurityConfigurer : webSecurityConfigurers){
			webSecurity.apply(webSecurityConfigurer);
		}
		this.webSecurityConfigurers = webSecurityConfigurers;
	}
	/**
	* 从当前bean容器中获取所有的WebSecurityConfigurer bean。
	* 这些WebSecurityConfigurer通常是由开发人员实现的配置类，并且继承自WebSecurityConfigurerAdapter
	*/
	@Bean
	public static AutowiredWebSecurityConfigurersIgnoreParents
		autowiredWebSecurityConfigurersIgnoreParents(
			ConfigurableListableBeanFactory beanFactory) {
		return new AutowiredWebSecurityConfigurersIgnoreParents(beanFactory);
	}
	//......
}
```

② **SpringWebMvcImportSelector**

该类是为了对 Spring Mvc 进行支持的。一旦发现应用使用 Spring Mvc 的核心前置控制器 `DispatcherServlet` 就会引入 `WebMvcSecurityConfiguration `。主要是为了适配 Spring Mvc 。

③ **OAuth2ImportSelector**

该类是为了对 OAuth2.0 开放授权协议进行支持。 ClientRegistration 如果被引用，具体点也就是 spring-security-oauth2 模块被启用（引入依赖jar）时。会启用 OAuth2 客户端配置 `OAuth2ClientConfiguration `。

④**@EnableGlobalAuthentication**

这个类主要引入了` AuthenticationConfiguration` 目的主要为了构造 认证管理器` AuthenticationManager `。 



3.**SecurityFilterAutoConfiguration**

用于向Servlet容器注册一个名称为 `securityFilterChainRegistration` 的bean, 实现类是 `DelegatingFilterProxyRegistrationBean` 。该 bean 的目的是注册另外一个 Servlet Filter Bean 到 Servlet 容器,实现类为 `DelegatingFilterProxy` 。 DelegatingFilterProxy 其实是一个代理过滤器，它被 Servlet 容器用于处理请求时，会将任务委托给指定给自己另外一个Filter bean。 对于 SecurityFilterAutoConfiguration ,来讲，这个被代理的Filter bean的名字为 **springSecurityFilterChain** , 也就是我们上面提到过的 Spring Security Web提供的用于请求安全处理的Filter bean，其实现类是 `FilterChainProxy` 。

```java
@Configuration
// 仅在 Servlet 环境下生效
@ConditionalOnWebApplication(type = Type.SERVLET)
// 确保安全属性配置信息被加载并以bean形式被注册到容器
@EnableConfigurationProperties(SecurityProperties.class)
// 仅在特定类存在于 classpath 上时才生效
@ConditionalOnClass({ AbstractSecurityWebApplicationInitializer.class,
	SessionCreationPolicy.class })
// 指定该配置类在 SecurityAutoConfiguration 配置类应用之后应用
@AutoConfigureAfter(SecurityAutoConfiguration.class)
public class SecurityFilterAutoConfiguration {
	// 要注册到 Servlet 容器的 DelegatingFilterProxy Filter的目标代理Filter bean的名称springSecurityFilterChain
	private static final String DEFAULT_FILTER_NAME =
		AbstractSecurityWebApplicationInitializer.DEFAULT_FILTER_NAME;
	// 安装上面方式注入securityFilterChainRegistration,
	@Bean
	@ConditionalOnBean(name = DEFAULT_FILTER_NAME)
	public DelegatingFilterProxyRegistrationBean securityFilterChainRegistration(
			SecurityProperties securityProperties) {
		DelegatingFilterProxyRegistrationBean registration = new
		DelegatingFilterProxyRegistrationBean(DEFAULT_FILTER_NAME);
		registration.setOrder(securityProperties.getFilter().getOrder());
		registration.setDispatcherTypes(getDispatcherTypes(securityProperties));
		return registration;
	}
	private EnumSet<DispatcherType> getDispatcherTypes(SecurityProperties securityProperties) {
		if (securityProperties.getFilter().getDispatcherTypes() == null) {
			return null;
		}
		return securityProperties.getFilter().getDispatcherTypes().stream()
			.map((type) -> DispatcherType.valueOf(type.name()))
			.collect(Collectors.collectingAndThen(Collectors.toSet(), EnumSet::copyOf));
	}
}
```

## WebSecurityConfigurerAdapter 

安全配置类,  通过该类可以自定义配置, 通常直接实现该类重写其方法即可, 也可以模仿`SpringBootWebSecurityConfiguration`的方式:来重写:

```
@Configuration
@ConditionalOnClass(WebSecurityConfigurerAdapter.class)
@ConditionalOnWebApplication(type = Type.SERVLET)
public class  CustomSpringBootWebSecurityConfiguration {
	@Configuration(proxyBeanMethods = false)
	@Order(SecurityProperties.BASIC_AUTH_ORDER)
	static class DefaultConfigurerAdapter extends WebSecurityConfigurerAdapter {
		@Override
		protected void configure(AuthenticationManagerBuilder auth) throws Exception {
			super.configure(auth);
		}
		@Override
		public void configure(WebSecurity web) throws Exception {
			super.configure(web);
		}
		@Override
		protected void configure(HttpSecurity http) throws Exception {
			super.configure(http);
		}
	}

}

```

**认证管理器配置方法: **`void configure(AuthenticationManagerBuilder auth)` : 用来配置认证管理器` AuthenticationManager`, (管理所有的`UserDetails`等)

**核心过滤器配置方法:**` void configure(WebSecurity web)`: 用来配置 `WebSecurity` 。WebSecurity 是基于 Servlet Filter 用来配置 springSecurityFilterChain ,  而 springSecurityFilterChain 又被委托给了 Spring Security 核心过滤器 Bean `DelegatingFilterProxy` 。 相关逻辑可以在 `WebSecurityConfiguration` 中找到。我们一般不会过多来自定义 WebSecurity , 使用较多的使其 ignoring() 方法用来忽略 Spring Security 对静态资源的控制。

**安全过滤器配置方法:**`void configure(HttpSecurity http)`:用来配置 `HttpSecurity` 。 HttpSecurity 用于构建一个安全过滤器链 SecurityFilterChain 。 SecurityFilterChain 最终 被注入核心过滤器 。 HttpSecurity 有许多我们需要的配置。我们可以通过它来进行自定义安全访问策略。这也是大家最经常重写的方法.

## 登录

自定义访问控制主要是通过 HttpSecurity 来构建的。默认它提供了三种登录方式： 

formLogin() 普通表单登录

oauth2Login() 基于 OAuth2.0 认证/授权协议 

openidLogin() 基于 OpenID 身份认证规范

以上三种方式统统是 AbstractAuthenticationFilterConfigurer 实现的，



**form 表单登录**

启用表单登录通过两种方式一种是通过 **HttpSecurity 的 apply(C configurer) 方法自己构造一 个 AbstractAuthenticationFilterConfigurer** 的实现，这种是比较高级的玩法。 另一种是我们常见的使用 **HttpSecurity 的 formLogin() 方法来自定义 FormLoginConfigurer** 。

1. **FormLoginConfigurer**

该类是 form 表单登录的配置类。

它提供了一些我们常用的配置方法：

- loginPage(String loginPage) : 登录页面而并不是接口，对于前后分离模式需要我们进行改造 默认为 /login 。
- loginProcessingUrl(String loginProcessingUrl) 实际表单向后台提交用户信息的 Action ，再由过滤器 UsernamePasswordAuthenticationFilter 拦截处理，该 Action 其实 不会处理任何逻辑。 
- usernameParameter(String usernameParameter) 用来自定义用户参数名，默认 username 。
- passwordParameter(String passwordParameter) 用来自定义用户密码名，默认 password 
- failureUrl(String authenticationFailureUrl) 登录失败后会重定向到此路径， 一般前后 分离不会使用它。 
- failureForwardUrl(String forwardUrl) 登录失败会转发到此， 一般前后分离用到它。 可定 义一个 Controller （控制器）来处理返回值,但是要注意 RequestMethod 。 
- defaultSuccessUrl(String defaultSuccessUrl, boolean alwaysUse) 默认登陆成功后 跳转到此 ，如果 alwaysUse 为 true 只要进行认证流程而且成功，会一直跳转到此。一般推荐 默认值 false
- successForwardUrl(String forwardUrl) 效果等同于上面 defaultSuccessUrl 的 alwaysUse 为 true 但是要注意 RequestMethod 。 
- successHandler(AuthenticationSuccessHandler successHandler) 自定义认证成功处理 器，可替代上面所有的 success 方式 
- failureHandler(AuthenticationFailureHandler authenticationFailureHandler) 自定 义失败处理器，可替代上面所有的 failure 方式 
- permitAll(boolean permitAll) form 表单登录是否放开

例如: 对于controller

```
@RestController
@RequestMapping("/login")
public class LoginController {
	@Resource
	private SysUserService sysUserService;
	@PostMapping("/failure")
	public Rest loginFailure() {
		return RestBody.failure(HttpStatus.UNAUTHORIZED.value(), "登录失败了，老哥");
	}
	@PostMapping("/success")
	public Rest loginSuccess() {
		User principal = (User) SecurityContextHolder.getContext().getAuthentication().getPrincipal();
		String username = principal.getUsername();
		SysUser sysUser = sysUserService.queryByUsername(username);
		sysUser.setEncodePassword("[PROTECT]");
		return RestBody.okData(sysUser,"登录成功");
	}
}
```

配置登录:

```
@Override
protected void configure(HttpSecurity http) throws Exception {
	http.csrf().disable()
		.cors()
		.and()
		.authorizeRequests().anyRequest().authenticated()
		.and()
		.formLogin()
		.loginProcessingUrl("/process")			//登录的Url
		.successForwardUrl("/login/success")  	//登录成功后的Url, 登录成功后的转发?
		.failureForwardUrl("/login/failure");	//登录失败后的Url
}
```

**UsernamePasswordAuthenticationFilter** 

Http登录认证由过滤器` UsernamePasswordAuthenticationFilter` 进行处理,  它的作用是拦截登录请求并获取账号和 密码，然后把账号密码封装到`UsernamePasswordAuthenticationToken` 中交给`AuthenticationManager` 去作认证。

我们可以实现我们自己的登录方式(比如验证码 , json), 通过 HttpSecurity 的 `addFilterBefore(preLoginFilter, UsernamePasswordAuthenticationFilter.class) `方法进行前来实现。

##  AuthenticationManager

`AuthenticationManager`在`void configure(AuthenticationManagerBuilder auth)` 中配置.

`AuthenticationManager` 的实现 `ProviderManager` 管理了众多的 `AuthenticationProvider` 。每一个` AuthenticationProvider `都只支持特定类型的 `Authentication` ，然后是对适配到的 Authentication 进行认证，只要有一个 AuthenticationProvider 认证成功，那么就认为认证成功，所有的都没有通过才认为是认证失败。认证成功后的 Authentication 就变成授信凭据，并触发认证成功的事件。认证失败的就抛出异常触发认证失败的事件。我们也可以借此来实现多种认证并存。

 `AuthenticationManager`通过`AuthenticationConfiguration`初始化: 

```
public AuthenticationManager getAuthenticationManager() throws Exception {
	// 先判断 AuthenticationManager 是否初始化
	if (this.authenticationManagerInitialized) {
		// 如果已经初始化 那么直接返回初始化的
		return this.authenticationManager;
	}
	// 否则就去 Spring IoC 中获取其构建类
	AuthenticationManagerBuilder authBuilder = this.applicationContext.getBean(AuthenticationManagerBuilder.class);
	// 如果不是第一次构建 好像是每次总要通过Builder来进行构建
	if (this.buildingAuthenticationManager.getAndSet(true)) {
	// 返回 一个委托的AuthenticationManager
		return new AuthenticationManagerDelegator(authBuilder);
	}
	// 如果是第一次通过Builder构建 将全局的认证配置整合到Builder中 那么以后就不用再整合全局的配置了
	for (GlobalAuthenticationConfigurerAdapter config : globalAuthConfigurers) {
		authBuilder.apply(config);
	}
	// 构建AuthenticationManager
	authenticationManager = authBuilder.build();
	// 如果构建结果为null
	if (authenticationManager == null) {
	// 再次尝试去Spring IoC 获取懒加载的 AuthenticationManager Bean
authenticationManager = getAuthenticationManagerBean();
}
// 修改初始化状态
this.authenticationManagerInitialized = true;
return authenticationManager;
}
```

