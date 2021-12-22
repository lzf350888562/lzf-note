# jwt

## jjwt

```
<dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt</artifactId>
    <version>${jwt.version}</version>
</dependency>
```

token生成,通常可以将用户信息(甚至可以是序列化之后的用户对象)存到claims中

```

Map<String, Object> claims = new HashMap<>();
claims.put(Constants.LOGIN_USER_KEY, token);
 String token = Jwts.builder()
                .setClaims(claims)
                .signWith(SignatureAlgorithm.HS512, "secret: abcdefghijklmnopqrstuvwxyz")
                .compact();
               

 //----
 return Jwts.builder()
                .setHeaderParam("typ", "JWT")
                .setSubject(userId+"")//相当于设置在claims key为sub
                .setIssuedAt(nowDate)
                .setExpiration(expireDate)//相当于设置在claims key为exp
                .signWith(SignatureAlgorithm.HS512, secret)
                .compact();
```

token解析

```
private token parseToken(String token)
{
    Claims claims = Jwts.parser()
            .setSigningKey(secret)
            .parseClaimsJws(token)
            .getBody();
    return claims.get(Constants.LOGIN_USER_KEY);
}
```

设置claim的过期时间还可以通过claim.set("exp",xxx),设置用户信息也可用过claim.set("sub",serializedUserObj),

通过Claims获取过期时间

```
claims.getExpiration()
```

判断是否过期

```
claims.getExpiration().before(new Date())
```

获取subject

```
claims.getSubject()
```



## java-jwt

与shiro有使用案例

```
<dependency>
            <groupId>com.auth0</groupId>
            <artifactId>java-jwt</artifactId>
            <version>${jwt.auth0.version}</version>
        </dependency>
```

创建jwt

```
//首先拿到算法实例，用于创建后的签入 sign
//可以使用其他工具创建salt字符串 salt就是随机字符串 如shiro 
Algorithm algorithm = Algorithm.HMAC256(salt); //salt只有服务器知道
String token = JWT.create()
            .withIssuer("auth0")    // token发布者
            .withIssuedAt(new Date())   // 生成签名的时间
            .withExpiresAt(DateUtils.addHours(new Date(),2)) // 生成签名的有效期,小时
            .withClaim("name","wuyuwei") // 插入数据
            .sign(algorithm);
//可以拿到base64解密获取json对象
```

shiro创建salt

```
SecureRandomNumberGenerator secureRandom = new SecureRandomNumberGenerator();
String salt = secureRandom.nextBytes(16).toHex();
```

验证jwt 如果验证失败将异常JWTVerificationException

```
Algorithm algorithm = Algorithm.HMAC256("secret");
JWTVerifier verifier = JWT.require(algorithm)
       .withIssuer("auth0") //匹配指定的token发布者 auth0
       .build();
DecodedJWT jwt = verifier.verify(token); //解码JWT ，verifier 可复用
System.out.println(jwt);
System.out.println(jwt.getIssuer()); // => auth0
System.out.println(jwt.getIssuedAt()); // =>Sat Jan 11 20:25:13 CST 2020
System.out.println(jwt.getExpiresAt());
String algorithm = jwt.getAlgorithm(); //获取算法类型 HS256
String type = jwt.getType();	//获取token类型  JWT
Map<String, Claim> claims = jwt.getClaims();
Claim claim = claims.get("username");
System.out.println(claim.asString());
//或者
Claim claim = jwt.getClaim("name");
System.out.println(claim.asString()); 
```

jwt解码 将密文进行 base64 解密 出错异常JWTDecodeException

```
DecodedJWT jwt = JWT.decode(token);
//使用方式与上面相同
```



# springsecurity

![](picture/spring-security01.png)

Spring Security包含了众多的过滤器，这些过滤器形成了一条链，所有请求都必须通过这些过滤器后才能成功访问到资源。其中`UsernamePasswordAuthenticationFilter`过滤器用于处理基于表单方式的登录认证，而`BasicAuthenticationFilter`用于处理基于HTTP Basic方式的登录验证，后面还可能包含一系列别的过滤器（可以通过相应配置开启）。在过滤器链的末尾是一个名为`FilterSecurityInterceptor`的拦截器，用于判断当前请求身份认证是否成功，是否有相应的权限，当身份认证失败或者权限不足的时候便会抛出相应的异常。`ExceptionTranslateFilter`捕获并处理，所以我们在`ExceptionTranslateFilter`过滤器用于处理了`FilterSecurityInterceptor`抛出的异常并进行处理，比如需要身份认证时将请求重定向到相应的认证页面，当认证失败或者权限不足时返回相应的提示信息。

开启security

```
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-security</artifactId>
</dependency>
```

当Spring项目中引入了Spring Security依赖的时候，项目会默认开启如下配置：

```
security:
  basic:
    enabled: true
```

这个配置开启了一个HTTP basic类型的认证(高版本已经默认是表单认证了)，所有服务的访问都必须先过这个认证(HTTP Basic认证框)，默认的用户名为user，密码由Sping Security自动生成，在IDE的控制台，

**基于表单认证**

配置将HTTP Basic认证修改为基于表单的认证方式。

创建一个配置类`BrowserSecurityConfig`继承`org.springframework.security.config.annotation.web.configuration.WebSecurityConfigurerAdapter`这个抽象类并重写`configure(HttpSecurity http)`方法。`WebSecurityConfigurerAdapter`是由Spring Security提供的Web应用安全配置的适配器：

```
@Configuration
public class BrowserSecurityConfig extends WebSecurityConfigurerAdapter {
    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http.formLogin() // 表单方式
                .and()
                .authorizeRequests() // 授权配置
                .anyRequest()  // 所有请求
                .authenticated(); // 都需要认证
    }
}
```

再访问项目现在是表单认证方式而不是对话框了.

如果需要换回HTTP Basic的认证方式，我们只需要简单修改`configure`方法中的配置：

```
@Override
protected void configure(HttpSecurity http) throws Exception {
    // http.formLogin() // 表单方式
    http.httpBasic() // HTTP Basic方式
            .and()
            .authorizeRequests() // 授权配置
            .anyRequest()  // 所有请求
            .authenticated(); // 都需要认证
}
```

```
     * anyRequest          |   匹配所有请求路径
     * access              |   SpringEl表达式结果为true时可以访问
     * anonymous           |   匿名可以访问
     * denyAll             |   用户不能访问
     * fullyAuthenticated  |   用户完全认证可以访问（非remember-me下自动登录）
     * hasAnyAuthority     |   如果有参数，参数表示权限，则其中任何一个权限可以访问
     * hasAnyRole          |   如果有参数，参数表示角色，则其中任何一个角色可以访问
     * hasAuthority        |   如果有参数，参数表示权限，则其权限可以访问
     * hasIpAddress        |   如果有参数，参数表示IP地址，如果用户IP和参数匹配，则可以访问
     * hasRole             |   如果有参数，参数表示角色，则其角色可以访问
     * permitAll           |   用户可以任意访问
     * rememberMe          |   允许通过remember-me登录的用户访问
     * authenticated       |   用户登录后可访问
```



## 自定义认证过程+加密

spring boot 1.x

自定义认证的过程需要实现Spring Security提供的`UserDetailService`接口，该接口只有一个抽象方法`loadUserByUsername`

```
public interface UserDetailsService {
    UserDetails loadUserByUsername(String username) throws UsernameNotFoundException;
}
```

`loadUserByUsername`方法返回一个`UserDetail`对象，该对象也是一个接口，包含一些用于描述用户信息的方法，源码如下：

```
public interface UserDetails extends Serializable {
	//获取用户包含的权限，返回权限集合，权限是一个继承了GrantedAuthority的对象
    Collection<? extends GrantedAuthority> getAuthorities();
    //获取密码和用户名
    String getPassword();
    String getUsername();
    //判断账户是否未过期，未过期返回true反之返回false
    boolean isAccountNonExpired();
    //判断账户是否未锁定
    boolean isAccountNonLocked();
    //判断用户凭证是否没过期，即密码是否未过期
    boolean isCredentialsNonExpired();
    //判断用户是否可用
    boolean isEnabled();
}
```

实际中我们可以自定义`UserDetails`接口的实现类，也可以直接使用Spring Security提供的`UserDetails`接口实现类`org.springframework.security.core.userdetails.User`

创建一个`MyUser`对象，用于存放模拟的用户数据（实际中一般从数据库获取）：

```
public class MyUser implements Serializable {
    private static final long serialVersionUID = 3497935890426858541L;
    private String userName;
    private String password;
    private boolean accountNonExpired = true;
    private boolean accountNonLocked= true;
    private boolean credentialsNonExpired= true;
    private boolean enabled= true;
    // get,set略
}
```

创建`MyUserDetailService`实现`UserDetailService`：

```
@Configuration
public class UserDetailService implements UserDetailsService {
    @Autowired
    private PasswordEncoder passwordEncoder;
    @Override
    public UserDetails loadUserByUsername(String username) throws UsernameNotFoundException {
        // 模拟一个用户，替代数据库获取逻辑
        MyUser user = new MyUser();
        user.setUserName(username);
        user.setPassword(this.passwordEncoder.encode("123456"));
        // 输出加密后的密码
        System.out.println(user.getPassword());

        return new User(username, user.getPassword(), user.isEnabled(),
                user.isAccountNonExpired(), user.isCredentialsNonExpired(),
                user.isAccountNonLocked(), AuthorityUtils.commaSeparatedStringToAuthorityList("admin"));
    }
}
```

这里我们使用了`org.springframework.security.core.userdetails.User`类包含7个参数的构造器，其还包含一个三个参数的构造器`User(String username, String password,Collection<? extends GrantedAuthority> authorities)`，由于权限参数不能为空，所以这里先使用`AuthorityUtils.commaSeparatedStringToAuthorityList`方法模拟一个admin的权限，该方法可以将逗号分隔的字符串转换为权限集合。

此外我们还注入了`PasswordEncoder`对象，该对象用于密码加密（**强散列哈希**），注入前需要手动配置。我们在`BrowserSecurityConfig`中配置它：

```
@Configuration
public class BrowserSecurityConfig extends WebSecurityConfigurerAdapter {
    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();
    }
}
```

`PasswordEncoder`是一个密码加密接口，而`BCryptPasswordEncoder`是Spring Security提供的一个实现方法，我们也可以自己实现`PasswordEncoder`。不过Spring Security实现的`BCryptPasswordEncoder`已经足够强大，它对相同的密码进行加密后可以生成不同的结果。

这时候重启项目，访问http://localhost:8080/login，便可以使用任意用户名以及123456作为密码登录系统。我们多次进行登录操作，可以看到控制台输出的加密后的密码每次都不相同.

**拓展**

以下方式能实现简单方式认证并限制角色访问(固定账号密码并给与ADMIN角色):

```
 @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();
    }
 @Override
    public void configure(AuthenticationManagerBuilder auth) throws Exception {

        User.UserBuilder builder = User.builder().passwordEncoder(passwordEncoder()::encode);
        auth.inMemoryAuthentication().withUser(builder.username(username).password(password).roles("ADMIN").build());
    }
    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http.csrf().disable()//禁用了 csrf 功能
                .authorizeRequests()//限定签名成功的请求
                .antMatchers("/**").hasRole("ADMIN")
                .anyRequest().permitAll()//其他没有限定的请求，允许访问
                .and().anonymous()//对于没有配置权限的其他请求允许匿名访问
                .and().formLogin()//使用 spring security 默认登录页面
                .and().httpBasic();//启用http 基础验证

    }
```



### 替换默认登录页

src/main/resources/resources目录下定义一个login.html（不需要Controller跳转）

```
<!DOCTYPE html>
<html>
<head>
    <meta charset="UTF-8">
    <title>登录</title>
    <link rel="stylesheet" href="css/login.css" type="text/css">
</head>
<body>
    <form class="login-page" action="/login" method="post">
        <div class="form">
            <h3>账户登录</h3>
            <input type="text" placeholder="用户名" name="username" required="required" />
            <input type="password" placeholder="密码" name="password" required="required" />
            <button type="submit">登录</button>
        </div>
    </form>
</body>
</html>
```

修改配置:

```
@Override
protected void configure(HttpSecurity http) throws Exception {
    http.formLogin() // 表单登录
            // http.httpBasic() // HTTP Basic
            .loginPage("/login.html") 
            .loginProcessingUrl("/login")
            .and()
            .authorizeRequests() // 授权配置
            .antMatchers("/login.html").permitAll()
            .anyRequest()  // 所有请求
            .authenticated(); // 都需要认证
}
```

其中,`.loginPage("/login.html")`指定了跳转到登录页面的请求URL，`.loginProcessingUrl("/login")`对应登录页面form表单的`action="/login"`，`.antMatchers("/login.html").permitAll()`表示跳转到登录页面的请求不被拦截，否则会进入无限循环。

现在再访问网页点击登录可能还是会报错,需要把CSRF攻击防御关了,在配置最后加上:

```
.and().csrf().disable();
```

### 未登录限制访问

需求:在未登录的情况下，当用户访问html资源的时候跳转到登录页，否则返回JSON格式数据，状态码为401。

要实现这个功能我们将`loginPage`的URL改为`/authentication/require`，并且在`antMatchers`方法中加入该URL，让其免拦截:

```
@Override
protected void configure(HttpSecurity http) throws Exception {
    http.formLogin() // 表单登录
            // http.httpBasic() // HTTP Basic
            .loginPage("/authentication/require") // 登录跳转 URL
            .loginProcessingUrl("/login") // 处理表单登录 URL
            .and()
            .authorizeRequests() // 授权配置
            .antMatchers("/authentication/require", "/login.html").permitAll() // 登录跳转 URL 无需认证
            .anyRequest().authenticated() // 所有请求都需要认证
            .and().csrf().disable();
}
```

#### 接口放行

**1、使用注解方式**

在Controller的方法上加入该注解即可不需要验证权限

```java
@AnonymousAccess
```

**2、修改配置文件方式**

```
protected void configure(HttpSecurity httpSecurity) throws Exception {
    httpSecurity
            // 支付宝回调
            .antMatchers("/api/aliPay/return").anonymous()
            // 所有请求都需要认证
            .anyRequest().authenticated();
    httpSecurity
            .addFilterBefore(authenticationTokenFilter, UsernamePasswordAuthenticationFilter.class);
}
```

> TIP

> 使用 `permitAll()` 方法所有人都能访问，包括带上 `token` 访问

> 使用 `anonymous()` 所有人都能访问，但是带上 `token` 访问后会报错

### HttpSessionRequestCache和DefaultRedirectStrategy

HttpSessionRequestCache和DefaultRedirectStrategy的使用

```
@RestController
public class BrowserSecurityController {
    private RequestCache requestCache = new HttpSessionRequestCache();
    private RedirectStrategy redirectStrategy = new DefaultRedirectStrategy();

    @GetMapping("/authentication/require")
    @ResponseStatus(HttpStatus.UNAUTHORIZED)
    public String requireAuthentication(HttpServletRequest request, HttpServletResponse response) throws IOException {
        SavedRequest savedRequest = requestCache.getRequest(request, response);
        if (savedRequest != null) {
            String targetUrl = savedRequest.getRedirectUrl();
            if (StringUtils.endsWithIgnoreCase(targetUrl, ".html"))
                redirectStrategy.sendRedirect(request, response, "/login.html");
        }
        return "访问的资源需要身份认证！";
    }
}
```

其中`HttpSessionRequestCache`为Spring Security提供的用于缓存请求的对象，通过调用它的`getRequest`方法可以获取到本次请求的HTTP信息。`DefaultRedirectStrategy`的`sendRedirect`为Spring Security提供的用于处理重定向的方法。(两者配合使用可以使登录后跳转到原来页面,在处理成功这一内容中有示例)

上面代码获取了引发跳转的请求，根据请求是否以`.html`为结尾来对应不同的处理方法。如果是以`.html`结尾，那么重定向到登录页面，否则返回”访问的资源需要身份认证！”信息，并且HTTP状态码为401（`HttpStatus.UNAUTHORIZED`）。

这样当我们访问http://localhost:8080/hello的时候页面便会跳转到http://localhost:8080/authentication/require，并且输出”访问的资源需要身份认证！”，当我们访问http://localhost:8080/hello.html的时候，页面将会跳转到登录页面。

### 处理成功和失败和登出+获取认证对象

Spring Security有一套默认的处理登录成功和失败的方法：当用户登录成功时，页面会跳转会引发登录的请求，比如在未登录的情况下访问http://localhost:8080/hello，页面会跳转到登录页，登录成功后再跳转回来；登录失败时则是跳转到Spring Security默认的错误提示页面。下面我们通过一些自定义配置来替换这套默认的处理机制。

**自定义登录成功逻辑**

要改变默认的处理成功逻辑很简单，只需要实现`org.springframework.security.web.authentication.AuthenticationSuccessHandler`接口的`onAuthenticationSuccess`方法即可：

```
@Component
public class MyAuthenticationSucessHandler implements AuthenticationSuccessHandler {
	@Autowired
    private ObjectMapper mapper;
    @Override
    public void onAuthenticationSuccess(HttpServletRequest request, HttpServletResponse response,Authentication authentication) throws IOException, ServletException {
        response.setContentType("application/json;charset=utf-8");
        response.getWriter().write(mapper.writeValueAsString(authentication));
    }
}
```

其中`Authentication`参数既包含了认证请求的一些信息，比如IP，请求的SessionId等，也包含了用户信息，即前面提到的`User`对象。通过上面这个配置，用户登录成功后页面将打印出`Authentication`对象的信息。

要使这个配置生效，我们还的在`BrowserSecurityConfig`的`configure`中配置它：

```
@Configuration
public class BrowserSecurityConfig extends WebSecurityConfigurerAdapter {
    @Autowired
    private MyAuthenticationSucessHandler authenticationSucessHandler;
    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();
    }
    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http.formLogin() // 表单登录
                // http.httpBasic() // HTTP Basic
                .loginPage("/authentication/require") // 登录跳转 URL
                .loginProcessingUrl("/login") // 处理表单登录 URL
                .successHandler(authenticationSucessHandler) // 处理登录成功
                .and()
                .authorizeRequests() // 授权配置
                .antMatchers("/authentication/require", "/login.html").permitAll() // 登录跳转 URL 无需认证
                .anyRequest()  // 所有请求
                .authenticated() // 都需要认证
                .and().csrf().disable();
    }
}
```

登录后显示的json为:

```
{
  "authorities": [
    {
      "authority": "admin"
    }
  ],
  "details": {
    "remoteAddress": "0:0:0:0:0:0:0:1",
    "sessionId": "8D50BAF811891F4397E21B4B537F0544"
  },
  "authenticated": true,
  "principal": {
    "password": null,
    "username": "mrbird",
    "authorities": [
      {
        "authority": "admin"
      }
    ],
    "accountNonExpired": true,
    "accountNonLocked": true,
    "credentialsNonExpired": true,
    "enabled": true
  },
  "credentials": null,
  "name": "mrbird"
}
```

像`password`，`credentials`这些敏感信息，Spring Security已经将其屏蔽。

除此之外，我们也可以在登录成功后做页面的跳转，修改`MyAuthenticationSucessHandler`：

```
@Component
public class MyAuthenticationSucessHandler implements AuthenticationSuccessHandler {
    private RequestCache requestCache = new HttpSessionRequestCache();
    private RedirectStrategy redirectStrategy = new DefaultRedirectStrategy();
    @Autowired
    private ObjectMapper mapper;
    @Override
    public void onAuthenticationSuccess(HttpServletRequest request, HttpServletResponse response,Authentication authentication) throws IOException {
        SavedRequest savedRequest = requestCache.getRequest(request, response);
        redirectStrategy.sendRedirect(request, response, savedRequest.getRedirectUrl());
    }
}
```

通过上面配置，登录成功后页面将跳转回引发跳转的页面。如果想指定跳转的页面，比如跳转到`/index`，可以将`savedRequest.getRedirectUrl()`修改为`/index`(只需要DefaultRedirectStrategy即可)：

```
@Component
public class MyAuthenticationSucessHandler implements AuthenticationSuccessHandler {
    private RedirectStrategy redirectStrategy = new DefaultRedirectStrategy();
    @Override
    public void onAuthenticationSuccess(HttpServletRequest request, HttpServletResponse response,Authentication authentication) throws IOException {
        redirectStrategy.sendRedirect(request, response, "/index");
    }
}
```

然后在TestController中定义一个处理该请求的方法：

```
@RestController
public class TestController {
    @GetMapping("index")
    public Object index(){
        return SecurityContextHolder.getContext().getAuthentication();
    }
}
```

**获取认证对象**

登录成功后，便可以使用`SecurityContextHolder.getContext().getAuthentication()`获取到`Authentication`对象信息。除了通过这种方式获取`Authentication`对象信息外，也可以使用下面这种方式:

```
@RestController
public class TestController {
    @GetMapping("index")
    public Object index(Authentication authentication) {
        return authentication;
    }
}
```

**自定义登录失败逻辑**

自定义登录失败处理逻辑需要实现`org.springframework.security.web.authentication.AuthenticationFailureHandler`的`onAuthenticationFailure`方法,假如我们需要在登录失败的时候返回失败信息，可以这样处理：

```
@Component
public class MyAuthenticationFailureHandler implements AuthenticationFailureHandler {
    @Autowired
    private ObjectMapper mapper;
    @Override
    public void onAuthenticationFailure(HttpServletRequest request, HttpServletResponse response, AuthenticationException exception) throws IOException {
    //状态码定义为500（HttpStatus.INTERNAL_SERVER_ERROR.value()），即系统内部异常。
        response.setStatus(HttpStatus.INTERNAL_SERVER_ERROR.value());
        response.setContentType("application/json;charset=utf-8");
        response.getWriter().write(mapper.writeValueAsString(exception.getMessage()));
    }
}
```

`onAuthenticationFailure`方法的`AuthenticationException`参数是一个抽象类，Spring Security根据登录失败的原因封装了许多对应的实现类，查看`AuthenticationException`的Hierarchy：

![](picture/spring-security02.png)

不同的失败原因对应不同的异常，比如用户名或密码错误对应的是`BadCredentialsException`，用户不存在对应的是`UsernameNotFoundException`，用户被锁定对应的是`LockedException`等。

最后也要在配置中指定

```
 @Autowired
    private MyAuthenticationFailureHandler authenticationFailureHandler;
//....
.failureHandler(authenticationFailureHandler) // 处理登录失败
```

**AuthenticationEntryPoint**

以同样方式实现该类的commence方法同样可以实现认证失败处理.

指定方式

```
.exceptionHandling().authenticationEntryPoint(unauthorizedHandler).and()
```

**处理登出**

类似的处理登出的接口为`LogoutSuccessHandler`

## 认证方式

`WebSecurityConfigurerAdapter`配置类下：

```
	@Bean
    @Override
    public AuthenticationManager authenticationManagerBean() throws Exception
    {
        return super.authenticationManagerBean();
    }
    
    @Override
    protected void configure(AuthenticationManagerBuilder auth) throws Exception
    {
        auth.userDetailsService(userDetailsService).passwordEncoder(bCryptPasswordEncoder());
    }
```

实现UserDetailService接口， 在`public UserDetails loadUserByUsername(String username)`方法返回当前认证用户（实现了UserDetail接口）

认证最终会调用上面的方法

```
Authentication authentication = null;
try{
    authentication = authenticationManager
           .authenticate(new UsernamePasswordAuthenticationToken(username, password));
}
```

认证过后，可通过

```
authentication.getPrincipal()
```

获取当前认证用户



## 图形验证码+自定义过滤器

添加验证码大致可以分为三个步骤：根据随机数生成验证码图片；将验证码图片显示到登录页面；认证流程中加入验证码校验。

Spring Security的认证校验是由`UsernamePasswordAuthenticationFilter`过滤器完成的，所以我们的验证码校验逻辑应该在这个过滤器之前。

验证码功能需要用到`spring-social-config`依赖：

```
 <dependency>
    <groupId>org.springframework.social</groupId>
    <artifactId>spring-social-config</artifactId>
</dependency>
```

首先定义一个验证码对象ImageCode：

```
public class ImageCode {
    private BufferedImage image;
    private String code;
    private LocalDateTime expireTime;
    public ImageCode(BufferedImage image, String code, int expireIn) {
        this.image = image;
        this.code = code;
        this.expireTime = LocalDateTime.now().plusSeconds(expireIn);
    }
    public ImageCode(BufferedImage image, String code, LocalDateTime expireTime) {
        this.image = image;
        this.code = code;
        this.expireTime = expireTime;
    }
    boolean isExpire() {
        return LocalDateTime.now().isAfter(expireTime);
    }
    // get,set 略
}
```

ImageCode对象包含了三个属性：`image`图片，`code`验证码和`expireTime`过期时间。`isExpire`方法用于判断验证码是否已过期。

定义一个ValidateCodeController，用于处理生成验证码请求：

```
@RestController
public class ValidateController {
    public final static String SESSION_KEY= "SESSION_KEY_IMAGE_CODE";
    private SessionStrategy sessionStrategy = new HttpSessionSessionStrategy();
    @GetMapping("/code/image")
    public void createCode(HttpServletRequest request, HttpServletResponse response) throws IOException {
        ImageCode imageCode = createImageCode();
        //将图片对象以SESSION_KEY=imageCode形式存入session
        sessionStrategy.setAttribute(new ServletWebRequest(request), SESSION_KEY, imageCode);
        ImageIO.write(imageCode.getImage(), "jpeg", response.getOutputStream());
    }
}
```

`createImageCode`方法用于生成验证码对象，`org.springframework.social.connect.web.HttpSessionSessionStrategy`对象封装了一些处理Session的方法，包含了`setAttribute`、`getAttribute`和`removeAttribute`方法，具体可以查看该类的源码。使用`sessionStrategy`将生成的验证码对象存储到Session中，并通过IO流将生成的图片输出到登录页面上。

其中`createImageCode`方法代码如下所示：

```
private ImageCode createImageCode() {
    int width = 100; // 验证码图片宽度
    int height = 36; // 验证码图片长度
    int length = 4; // 验证码位数
    int expireIn = 60; // 验证码有效时间 60s
    BufferedImage image = new BufferedImage(width, height, BufferedImage.TYPE_INT_RGB);
    Graphics g = image.getGraphics();
    Random random = new Random();
    g.setColor(getRandColor(200, 250));
    g.fillRect(0, 0, width, height);
    g.setFont(new Font("Times New Roman", Font.ITALIC, 20));
    g.setColor(getRandColor(160, 200));
    for (int i = 0; i < 155; i++) {
        int x = random.nextInt(width);
        int y = random.nextInt(height);
        int xl = random.nextInt(12);
        int yl = random.nextInt(12);
        g.drawLine(x, y, x + xl, y + yl);
    }
    StringBuilder sRand = new StringBuilder();
    for (int i = 0; i < length; i++) {
        String rand = String.valueOf(random.nextInt(10));
        sRand.append(rand);
        g.setColor(new Color(20 + random.nextInt(110), 20 + random.nextInt(110), 20 + random.nextInt(110)));
        g.drawString(rand, 13 * i + 6, 16);
    }
    g.dispose();
    return new ImageCode(image, sRand.toString(), expireIn);
}
private Color getRandColor(int fc, int bc) {
    Random random = new Random();
    if (fc > 255) {
        fc = 255;
    }
    if (bc > 255) {
        bc = 255;
    }
    int r = fc + random.nextInt(bc - fc);
    int g = fc + random.nextInt(bc - fc);
    int b = fc + random.nextInt(bc - fc);
    return new Color(r, g, b);
}
```

在登录页面加上如下代码：

```
<span style="display: inline">
    <input type="text" name="imageCode" placeholder="验证码" style="width: 50%;"/>
    <img src="/code/image"/>
</span>
```

要使生成验证码的请求不被拦截，需要在`BrowserSecurityConfig`的`configure`方法中配置免拦截：

```
@Override
protected void configure(HttpSecurity http) throws Exception {
    http.formLogin() // 表单登录
            // http.httpBasic() // HTTP Basic
            .loginPage("/authentication/require") // 登录跳转 URL
            .loginProcessingUrl("/login") // 处理表单登录 URL
            .successHandler(authenticationSucessHandler) // 处理登录成功
            .failureHandler(authenticationFailureHandler) // 处理登录失败
            .and()
            .authorizeRequests() // 授权配置
            .antMatchers("/authentication/require",
                    "/login.html",
                    "/code/image").permitAll() // 无需认证的请求路径
            .anyRequest()  // 所有请求
            .authenticated() // 都需要认证
            .and().csrf().disable();
}
```

**认证流程添加验证码校验**

在校验验证码的过程中，可能会抛出各种验证码类型的异常，比如“验证码错误”、“验证码已过期”等，所以我们定义一个验证码类型的异常类, 注意，这里继承的是`AuthenticationException`而不是`Exception`：

```
public class ValidateCodeException extends AuthenticationException {
    private static final long serialVersionUID = 5022575393500654458L;
    ValidateCodeException(String message) {
        super(message);
    }
}
```

Spring Security实际上是由许多过滤器组成的过滤器链，处理用户登录逻辑的过滤器为`UsernamePasswordAuthenticationFilter`，而验证码校验过程应该是在这个过滤器之前的，即只有验证码校验通过后采去校验用户名和密码。由于Spring Security并没有直接提供验证码校验相关的过滤器接口，所以我们需要自己定义一个验证码校验的过滤器`ValidateCodeFilter`：

```
@Component
public class ValidateCodeFilter extends OncePerRequestFilter {
    @Autowired
    private AuthenticationFailureHandler authenticationFailureHandler;
    //可用来操作session
    private SessionStrategy sessionStrategy = new HttpSessionSessionStrategy();
    @Override
    protected void doFilterInternal(HttpServletRequest httpServletRequest, HttpServletResponse httpServletResponse, 
    	FilterChain filterChain) throws ServletException, IOException {
    	//post类型的login请求才进行验证码校验过滤
        if (StringUtils.equalsIgnoreCase("/login", httpServletRequest.getRequestURI())
                && StringUtils.equalsIgnoreCase(httpServletRequest.getMethod(), "post")) {
            try {
                validateCode(new ServletWebRequest(httpServletRequest));
            } catch (ValidateCodeException e) {
            //使用AuthenticationFailureHandler来处理异常
              authenticationFailureHandler.onAuthenticationFailure(httpServletRequest, httpServletResponse, e);
                return;
            }
        }
        filterChain.doFilter(httpServletRequest, httpServletResponse);
    }
    private void validateCode(ServletWebRequest servletWebRequest) throws ServletRequestBindingException {
         ImageCode codeInSession = (ImageCode) sessionStrategy.getAttribute(servletWebRequest, ValidateController.SESSION_KEY);
         //获取请求参数  ServletRequestUtils可以用来操作request
    	String codeInRequest = ServletRequestUtils.getStringParameter(servletWebRequest.getRequest(), "imageCode");
    	if (StringUtils.isBlank(codeInRequest)) {
        	throw new ValidateCodeException("验证码不能为空！");
    	}
    	if (codeInSession == null) {
             throw new ValidateCodeException("验证码不存在！");
    	}
    	if (codeInSession.isExpire()) {
        	sessionStrategy.removeAttribute(servletWebRequest, ValidateController.SESSION_KEY);
        	throw new ValidateCodeException("验证码已过期！");
    	}
    	if (!StringUtils.equalsIgnoreCase(codeInSession.getCode(), codeInRequest)) {
        	throw new ValidateCodeException("验证码不正确！");
    	}
    	sessionStrategy.removeAttribute(servletWebRequest, ValidateController.SESSION_KEY);
	}
}
```

在`doFilterInternal`方法中我们判断了请求URL是否为`/login`，该路径对应登录`form`表单的`action`路径，请求的方法是否为**`POST`**，是的话进行验证码校验逻辑，否则直接执行`filterChain.doFilter`让代码往下走。当在验证码校验的过程中捕获到异常时，调用Spring Security的校验失败处理器`AuthenticationFailureHandler`进行处理。

我们分别从`Session`中获取了`ImageCode`对象和请求参数`imageCode`（对应登录页面的验证码`<input>`框`name`属性）,然后进行了各种判断并抛出相应的异常。当验证码过期或者验证码校验通过时，我们便可以删除`Session`中的`ImageCode`属性了。

最后需要添加过滤器配置,注入了`ValidateCodeFilter`，然后通过`addFilterBefore`方法将`ValidateCodeFilter`验证码校验过滤器添加到了`UsernamePasswordAuthenticationFilter`前面。:

```
@Autowired
private ValidateCodeFilter validateCodeFilter;
@Override
protected void configure(HttpSecurity http) throws Exception {
    http.addFilterBefore(validateCodeFilter, UsernamePasswordAuthenticationFilter.class) // 添加验证码校验过滤器
            .formLogin() // 表单登录
            // http.httpBasic() // HTTP Basic
            .loginPage("/authentication/require") // 登录跳转 URL
            .loginProcessingUrl("/login") // 处理表单登录 URL
            .successHandler(authenticationSucessHandler) // 处理登录成功
            .failureHandler(authenticationFailureHandler) // 处理登录失败
            .and()
            .authorizeRequests() // 授权配置
            .antMatchers("/authentication/require",
                    "/login.html",
                    "/code/image").permitAll() // 无需认证的请求路径
            .anyRequest()  // 所有请求
            .authenticated() // 都需要认证
            .and().csrf().disable();
}
```

## RememberMe

在Spring Security中添加记住我功能很简单，大致过程是：当用户勾选了记住我选项并登录成功后，Spring Security会生成一个token标识，然后将该token标识持久化到数据库，并且生成一个与该token相对应的cookie返回给浏览器。当用户过段时间再次访问系统时，如果该cookie没有过期，Spring Security便会根据cookie包含的信息从数据库中获取相应的token信息，然后帮用户自动完成登录操作。

在配置了datasource相关后,在BrowserSecurityConfig中配置个token持久化对象：

```
@Configuration
public class BrowserSecurityConfig extends WebSecurityConfigurerAdapter {
    @Autowired
    private UserDetailService userDetailService;
    @Autowired
    private DataSource dataSource;
    @Bean
    public PersistentTokenRepository persistentTokenRepository() {
        JdbcTokenRepositoryImpl jdbcTokenRepository = new JdbcTokenRepositoryImpl();
        jdbcTokenRepository.setDataSource(dataSource);
        jdbcTokenRepository.setCreateTableOnStartup(false);
        return jdbcTokenRepository;
    }
    ...
}
```

`PersistentTokenRepository`为一个接口类，这里我们用的是数据库持久化，所以实例用的是`PersistentTokenRepository`的实现类`JdbcTokenRepositoryImpl`。

`JdbcTokenRepositoryImpl`需要指定数据源，所以我们将配置好的数据源对象`DataSource`注入进来并配置到`JdbcTokenRepositoryImpl`的`dataSource`属性中。`createTableOnStartup`属性用于是否启动项目时创建保存token信息的数据表，这里设置为false，我们自己手动创建。

查看`JdbcTokenRepositoryImpl`的源码，可以看到其包含了一个`CREATE_TABLE_SQL`属性,这个其实就是用于保存token对象数据表的SQL语句，我们复制出来手动执行创建表：

```sql
CREATE TABLE persistent_logins (
    username VARCHAR (64) NOT NULL,
    series VARCHAR (64) PRIMARY KEY,
    token VARCHAR (64) NOT NULL,
    last_used TIMESTAMP NOT NULL
)
```

在登录页加入

```
<input type="checkbox" name="remember-me"/> 记住我
```

**其中`name`属性必须为`remember-me`。**

最后我们需要在Spring Security的认证流程中启用记住我的功能，在`BrowserSecurityConfig`的`configure`方法中开启记住我功能：

```
@Override
protected void configure(HttpSecurity http) throws Exception {
    http.addFilterBefore(validateCodeFilter, UsernamePasswordAuthenticationFilter.class) // 添加验证码校验过滤器
            .formLogin() // 表单登录
                // http.httpBasic() // HTTP Basic
                .loginPage("/authentication/require") // 登录跳转 URL
                .loginProcessingUrl("/login") // 处理表单登录 URL
                .successHandler(authenticationSucessHandler) // 处理登录成功
                .failureHandler(authenticationFailureHandler) // 处理登录失败
                .and()
            .rememberMe()
                .tokenRepository(persistentTokenRepository()) // 配置 token 持久化仓库
                .tokenValiditySeconds(3600) // remember 过期时间，单为秒
                .userDetailsService(userDetailService) // 处理自动登录逻辑
            .and()
                .authorizeRequests() // 授权配置
                .antMatchers("/authentication/require",
                        "/login.html",
                        "/image/code").permitAll() // 无需认证的请求路径
                .anyRequest()  // 所有请求
                .authenticated() // 都需要认证
            .and()
                .csrf().disable();
}
```

`rememberMe()`用于开启记住我功能；`tokenRepository(persistentTokenRepository())`用于指定token持久化方法；`tokenValiditySeconds`配置了token的有效时长，单为为秒；`userDetailsService(userDetailService)`用于处理通过token对象自动登录，这里为我们自定义的`UserDetailsService`接口实现。

测试

勾选并成功登录后，可以看到网页多了个remember-me的cookie对象. 查看数据库表persistent_logins,里面生成了username对应的token.

## 短信验证码登录+filter注册

Spring Security默认只提供了账号密码的登录认证逻辑，所以要实现手机短信验证码登录认证功能，我们需要模仿Spring Security账号密码登录逻辑代码来实现一套自己的认证逻辑。

**短信验证码生成**

我们在Spring Security添加图形验证码的基础上来集成短信验证码登录的功能。

和图形验证码类似，我们先定义一个短信验证码对象SmsCode：

```
public class SmsCode {
    private String code;
    private LocalDateTime expireTime;
    public SmsCode(String code, int expireIn) {
        this.code = code;
        this.expireTime = LocalDateTime.now().plusSeconds(expireIn);
    }
    public SmsCode(String code, LocalDateTime expireTime) {
        this.code = code;
        this.expireTime = expireTime;
    }
    boolean isExpire() {
        return LocalDateTime.now().isAfter(expireTime);
    }
    // get,set略
}
```

接着在ValidateCodeController中加入生成短信验证码相关请求对应的方法：

```
@RestController
public class ValidateController {
    public final static String SESSION_KEY_SMS_CODE = "SESSION_KEY_SMS_CODE";
    private SessionStrategy sessionStrategy = new HttpSessionSessionStrategy();
    @GetMapping("/code/sms")
    public void createSmsCode(HttpServletRequest request, HttpServletResponse response, String mobile) throws IOException {
        SmsCode smsCode = createSMSCode();
        sessionStrategy.setAttribute(new ServletWebRequest(request), SESSION_KEY_SMS_CODE + mobile, smsCode);
        // 输出验证码到控制台代替短信发送服务
        System.out.println("您的登录验证码为：" + smsCode.getCode() + "，有效时间为60秒");
    }
    private SmsCode createSMSCode() {
        String code = RandomStringUtils.randomNumeric(6);
        return new SmsCode(code, 60);
    }
}
```

这里我们使用`createSMSCode`方法生成了一个6位的纯数字随机数，有效时间为60秒。然后通过`SessionStrategy`对象的`setAttribute`方法将短信验证码保存到了Session中，对应的key为`SESSION_KEY_SMS_CODE`。

**改造登录页**

```
<form class="login-page" action="/login/mobile" method="post">
    <div class="form">
        <h3>短信验证码登录</h3>
        <input type="text" placeholder="手机号" name="mobile" value="17777777777" required="required"/>
        <span style="display: inline">
            <input type="text" name="smsCode" placeholder="短信验证码" style="width: 50%;"/>
            <a href="/code/sms?mobile=17777777777">发送验证码</a>
        </span>
        <button type="submit">登录</button>
    </div>
</form>
```

其中a标签的`href`属性值对应我们的短信验证码生成方法的请求URL。Form的action对应处理短信验证码登录方法的请求URL，这个方法下面在进行具体实现。同时，我们需要在Spring Security中配置`/code/sms`路径免验证：

```
@Override
protected void configure(HttpSecurity http) throws Exception {
    http.addFilterBefore(validateCodeFilter, UsernamePasswordAuthenticationFilter.class) // 添加验证码校验过滤器
            .formLogin() // 表单登录
                // http.httpBasic() // HTTP Basic
                .loginPage("/authentication/require") // 登录跳转 URL
                .loginProcessingUrl("/login") // 处理表单登录 URL
                .successHandler(authenticationSucessHandler) // 处理登录成功
                .failureHandler(authenticationFailureHandler) // 处理登录失败
            .and()
                .authorizeRequests() // 授权配置
                .antMatchers("/authentication/require",
                        "/login.html", "/code/image","/code/sms").permitAll() // 无需认证的请求路径
                .anyRequest()  // 所有请求
                .authenticated() // 都需要认证
            .and()
                .csrf().disable();
}
```

**短信验证码登录认证逻辑**

在Spring Security中，使用用户名密码认证的过程大致如下图所示：

![](picture/spring-security03.png)

Spring Security使用`UsernamePasswordAuthenticationFilter`过滤器来拦截用户名密码认证请求，将用户名和密码封装成一个`UsernamePasswordToken`对象交给`AuthenticationManager`处理。`AuthenticationManager`将挑出一个支持处理该类型Token的`AuthenticationProvider`（这里为`DaoAuthenticationProvider`，`AuthenticationProvider`的其中一个实现类）来进行认证，认证过程中`DaoAuthenticationProvider`将调用`UserDetailService`的`loadUserByUsername`方法来获取UserDetails对象，如果UserDetails不为空并且密码和用户输入的密码匹配一致的话，则将认证信息保存到Session中，认证后我们便可以通过`Authentication`对象获取到认证的信息了。

由于Spring Security并没用提供短信验证码认证的流程，所以我们需要仿照上面这个流程来实现,将`UsernamePasswordAuthenticationFilter`替换为自定义的拦截短信验证码登录请求的拦截器`SmsAuthenticationFitler`,并将手机号码封装到一个叫`SmsAuthenticationToken`的对象中.

在Spring Security中，认证处理都需要通过`AuthenticationManager`来代理，所以这里我们依旧将`SmsAuthenticationToken`交由`AuthenticationManager`处理.

接着我们需要定义一个支持处理`SmsAuthenticationToken`对象的`SmsAuthenticationProvider`来代替`DaoAuthenticationProvider`

`SmsAuthenticationProvider`调用`UserDetailService`的`loadUserByUsername`方法来处理认证。与用户名密码认证不一样的是，这里是通过`SmsAuthenticationToken`中的手机号去数据库中查询是否有与之对应的用户，如果有，则将该用户信息封装到`UserDetails`对象中返回并将认证后的信息保存到`Authentication`对象中。

综上,我们一共要定义`SmsAuthenticationFitler`、`SmsAuthenticationToken`和`SmsAuthenticationProvider`，并将这些组建组合起来添加到Spring Security中.

**SmsAuthenticationToken**

参照`UsernamePasswordAuthenticationToken`的源码

```
public class SmsAuthenticationToken extends AbstractAuthenticationToken {
    private static final long serialVersionUID = SpringSecurityCoreVersion.SERIAL_VERSION_UID;
    private final Object principal;
    public SmsAuthenticationToken(String mobile) {
        super(null);
        this.principal = mobile;
        setAuthenticated(false);
    }
    public SmsAuthenticationToken(Object principal, Collection<? extends GrantedAuthority> authorities) {
        super(authorities);
        this.principal = principal;
        super.setAuthenticated(true); // must use super, as we override
    }
    @Override
    public Object getCredentials() {
        return null;
    }
    public Object getPrincipal() {
        return this.principal;
    }
    public void setAuthenticated(boolean isAuthenticated) throws IllegalArgumentException {
        if (isAuthenticated) {
            throw new IllegalArgumentException(
                    "Cannot set this token to trusted - use constructor which takes a GrantedAuthority list instead");
        }
        super.setAuthenticated(false);
    }
    @Override
    public void eraseCredentials() {
        super.eraseCredentials();
    }
}
```

`SmsAuthenticationToken`包含一个`principal`属性，从它的两个构造函数可以看出，在认证之前`principal`存的是手机号，认证之后存的是用户信息。`UsernamePasswordAuthenticationToken`原来还包含一个`credentials`属性用于存放密码，这里不需要就去掉了。

**SmsAuthenticationFilter**

参照`UsernamePasswordAuthenticationFilter`的源码

```
public class SmsAuthenticationFilter extends AbstractAuthenticationProcessingFilter {
    public static final String MOBILE_KEY = "mobile";
    private String mobileParameter = MOBILE_KEY;
    private boolean postOnly = true;
    public SmsAuthenticationFilter() {
        super(new AntPathRequestMatcher("/login/mobile", "POST"));
    }
    public Authentication attemptAuthentication(HttpServletRequest request,
                                                HttpServletResponse response) throws AuthenticationException {
        if (postOnly && !request.getMethod().equals("POST")) {
            throw new AuthenticationServiceException(
                    "Authentication method not supported: " + request.getMethod());
        }
        String mobile = obtainMobile(request);
        if (mobile == null) {
            mobile = "";
        }
        mobile = mobile.trim();
        SmsAuthenticationToken authRequest = new SmsAuthenticationToken(mobile);
        setDetails(request, authRequest);
        return this.getAuthenticationManager().authenticate(authRequest);
    }
    protected String obtainMobile(HttpServletRequest request) {
        return request.getParameter(mobileParameter);
    }
    protected void setDetails(HttpServletRequest request,
                              SmsAuthenticationToken authRequest) {
        //authenticationDetailsSource为父类属性
        authRequest.setDetails(authenticationDetailsSource.buildDetails(request));
    }
    public void setMobileParameter(String mobileParameter) {
        Assert.hasText(mobileParameter, "mobile parameter must not be empty or null");
        this.mobileParameter = mobileParameter;
    }
    public void setPostOnly(boolean postOnly) {
        this.postOnly = postOnly;
    }
    public final String getMobileParameter() {
        return mobileParameter;
    }
}
```

构造函数中指定了当请求为`/login/mobile`，请求方法为**POST**的时候该过滤器生效。`mobileParameter`属性值为mobile，对应登录页面手机号输入框的name属性。

`attemptAuthentication`方法从请求中获取到mobile参数值，并调用`SmsAuthenticationToken`的`SmsAuthenticationToken(String mobile)`构造方法创建了一个`SmsAuthenticationToken`。下一步就如流程图中所示的那样，`SmsAuthenticationFilter`将`SmsAuthenticationToken`交给`AuthenticationManager`处理。

**SmsAuthenticationProvider**

在创建完`SmsAuthenticationFilter`后，我们需要创建一个支持处理该类型Token的类，即`SmsAuthenticationProvider`，该类需要实现`AuthenticationProvider`的两个抽象方法：

```
public class SmsAuthenticationProvider implements AuthenticationProvider {
    private UserDetailService userDetailService;
    @Override
    public Authentication authenticate(Authentication authentication) throws AuthenticationException {
        SmsAuthenticationToken authenticationToken = (SmsAuthenticationToken) authentication;
        UserDetails userDetails = userDetailService.loadUserByUsername((String) authenticationToken.getPrincipal());
        if (userDetails == null)
            throw new InternalAuthenticationServiceException("未找到与该手机号对应的用户");

        SmsAuthenticationToken authenticationResult = new SmsAuthenticationToken(userDetails, userDetails.getAuthorities());

        authenticationResult.setDetails(authenticationToken.getDetails());

        return authenticationResult;
    }
    @Override
    public boolean supports(Class<?> aClass) {
        return SmsAuthenticationToken.class.isAssignableFrom(aClass);
    }
    public UserDetailService getUserDetailService() {
        return userDetailService;
    }
    public void setUserDetailService(UserDetailService userDetailService) {
        this.userDetailService = userDetailService;
    }
}
```

其中`supports`方法指定了支持处理的Token类型为`SmsAuthenticationToken`，`authenticate`方法用于编写具体的身份认证逻辑。

在`authenticate`方法中，我们从`SmsAuthenticationToken`中取出了手机号信息，并调用了`UserDetailService`的`loadUserByUsername`方法。该方法在用户名密码类型的认证中，主要逻辑是通过用户名查询用户信息，如果存在该用户并且密码一致则认证成功；而在短信验证码认证的过程中，该方法需要通过手机号去查询用户，如果存在该用户则认证通过。

认证通过后接着调用`SmsAuthenticationToken`的`SmsAuthenticationToken(Object principal, Collection<? extends GrantedAuthority> authorities)`构造函数构造一个认证通过的Token，包含了用户信息和用户权限。

你可能会问，为什么这一步没有进行短信验证码的校验呢？实际上短信验证码的校验是在`SmsAuthenticationFilter`之前完成的，即只有当短信验证码正确以后才开始走认证的流程。所以接下来我们需要定一个过滤器来校验短信验证码的正确性。

**SmsCodeFilter**

短信验证码的校验逻辑其实和图形验证码的校验逻辑基本一致，所以我们在图形验证码过滤器的基础上稍作修改

```
@Component
public class SmsCodeFilter extends OncePerRequestFilter {
    @Autowired
    private AuthenticationFailureHandler authenticationFailureHandler;
    private SessionStrategy sessionStrategy = new HttpSessionSessionStrategy();

    @Override
    protected void doFilterInternal(HttpServletRequest httpServletRequest, HttpServletResponse httpServletResponse, 
    	FilterChain filterChain) throws ServletException, IOException {
        if (StringUtils.equalsIgnoreCase("/login/mobile", httpServletRequest.getRequestURI())&& StringUtils.equalsIgnoreCase(httpServletRequest.getMethod(), "post")) {
            try {
                validateCode(new ServletWebRequest(httpServletRequest));
            } catch (ValidateCodeException e) {
                authenticationFailureHandler.onAuthenticationFailure(httpServletRequest, httpServletResponse, e);
                return;
            }
        }
        filterChain.doFilter(httpServletRequest, httpServletResponse);
    }

    private void validateSmsCode(ServletWebRequest servletWebRequest) throws ServletRequestBindingException {
        String smsCodeInRequest = ServletRequestUtils.getStringParameter(servletWebRequest.getRequest(), "smsCode");
        String mobile = ServletRequestUtils.getStringParameter(servletWebRequest.getRequest(), "mobile");
        ValidateCode codeInSession = (ValidateCode) sessionStrategy.getAttribute(servletWebRequest, FebsConstant.SESSION_KEY_SMS_CODE + mobile);

        if (StringUtils.isBlank(smsCodeInRequest)) {
            throw new ValidateCodeException("验证码不能为空！");
        }
        if (codeInSession == null) {
            throw new ValidateCodeException("验证码不存在，请重新发送！");
        }
        if (codeInSession.isExpire()) {
            sessionStrategy.removeAttribute(servletWebRequest, FebsConstant.SESSION_KEY_SMS_CODE + mobile);
            throw new ValidateCodeException("验证码已过期，请重新发送！");
        }
        if (!StringUtils.equalsIgnoreCase(codeInSession.getCode(), smsCodeInRequest)) {
            throw new ValidateCodeException("验证码不正确！");
        }
        sessionStrategy.removeAttribute(servletWebRequest, FebsConstant.SESSION_KEY_SMS_CODE + mobile);

    }
}
```

**配置生效**

在定义完所需的组件后，我们需要进行一些配置，将这些组件组合起来,创建一个配置类`SmsAuthenticationConfig`:

```
@Component
public class SmsAuthenticationConfig extends SecurityConfigurerAdapter<DefaultSecurityFilterChain, HttpSecurity> {
    @Autowired
    private AuthenticationSuccessHandler authenticationSuccessHandler;
    @Autowired
    private AuthenticationFailureHandler authenticationFailureHandler;
    @Autowired
    private UserDetailService userDetailService;
    @Override
    public void configure(HttpSecurity http) throws Exception {
        SmsAuthenticationFilter smsAuthenticationFilter = new SmsAuthenticationFilter();
        smsAuthenticationFilter.setAuthenticationManager(http.getSharedObject(AuthenticationManager.class));
        smsAuthenticationFilter.setAuthenticationSuccessHandler(authenticationSuccessHandler);
        smsAuthenticationFilter.setAuthenticationFailureHandler(authenticationFailureHandler);
        
        SmsAuthenticationProvider smsAuthenticationProvider = new SmsAuthenticationProvider();
        smsAuthenticationProvider.setUserDetailService(userDetailService);

        http.authenticationProvider(smsAuthenticationProvider)
                .addFilterAfter(smsAuthenticationFilter, UsernamePasswordAuthenticationFilter.class);

    }
}
```

在流程中第一步需要配置`SmsAuthenticationFilter`，分别设置了`AuthenticationManager`、`AuthenticationSuccessHandler`和`AuthenticationFailureHandler`属性。这些属性都是来自`SmsAuthenticationFilter`继承的`AbstractAuthenticationProcessingFilter`类中。

第二步配置`SmsAuthenticationProvider`，这一步只需要将我们自个的`UserDetailService`注入进来即可。

最后调用`HttpSecurity`的`authenticationProvider`方法指定了`AuthenticationProvider`为`SmsAuthenticationProvider`，并将`SmsAuthenticationFilter`过滤器添加到了`UsernamePasswordAuthenticationFilter`后面

最后一步需要做的是配置短信验证码校验过滤器，并且将短信验证码认证流程加入到Spring Security中。在`BrowserSecurityConfig`的`configure`方法中添加如下配置：

```
@Configuration
public class BrowserSecurityConfig extends WebSecurityConfigurerAdapter {
    @Autowired
    private MyAuthenticationSucessHandler authenticationSucessHandler;
    @Autowired
    private MyAuthenticationFailureHandler authenticationFailureHandler;
    @Autowired 
    private ValidateCodeFilter validateCodeFilter;
    @Autowired	
    private SmsCodeFilter smsCodeFilter;
    @Autowired
    private SmsAuthenticationConfig smsAuthenticationConfig;
    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();
    }
    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http.addFilterBefore(validateCodeFilter, UsernamePasswordAuthenticationFilter.class) // 添加验证码校验过滤器
            .addFilterBefore(smsCodeFilter, UsernamePasswordAuthenticationFilter.class) // 添加短信验证码校验过滤器
                .formLogin() // 表单登录
                    // http.httpBasic() // HTTP Basic
                    .loginPage("/authentication/require") // 登录跳转 URL
                    .loginProcessingUrl("/login") // 处理表单登录 URL
                    .successHandler(authenticationSucessHandler) // 处理登录成功
                    .failureHandler(authenticationFailureHandler) // 处理登录失败
                .and()
                    .authorizeRequests() // 授权配置
                    .antMatchers("/authentication/require",
                            "/login.html",
                            "/code/image","/code/sms").permitAll() // 无需认证的请求路径
                    .anyRequest()  // 所有请求
                    .authenticated() // 都需要认证
                .and()
                    .csrf().disable()
                .apply(smsAuthenticationConfig); // 将短信验证码认证配置加到 Spring Security 中
    }
}
```

## Session管理

**Session超时设置**

Session超时时间也就是用户登录的有效时间。要设置Session超时时间很简单，只需要在配置文件中添加：

```
server:
  session:
    timeout: 3600
```

单位为秒，通过上面的配置，Session的有效期为一个小时。

值得注意的是，Session的最小有效期为60秒，也就是说即使你设置为小于60秒的值，其有效期还是为60秒。

查看`TomcatEmbeddedServletContainerFactory`的源码即可发现原因：

![](picture/TomcatEmbedded-Session.png)

Session失效后，刷新页面后将跳转到认证页面，我们可以再添加一些配置，自定义Session失效后的一些行为。

在Spring Security中配置Session管理器，并配置Session失效后要跳转的URL：

```
@Override
protected void configure(HttpSecurity http) throws Exception {
    http.addFilterBefore(validateCodeFilter, UsernamePasswordAuthenticationFilter.class) // 添加验证码校验过滤器
        .addFilterBefore(smsCodeFilter,UsernamePasswordAuthenticationFilter.class) // 添加短信验证码校验过滤器
            .formLogin() // 表单登录
                .loginPage("/authentication/require") // 登录跳转 URL
                .loginProcessingUrl("/login") // 处理表单登录 URL
                .successHandler(authenticationSucessHandler) // 处理登录成功
                .failureHandler(authenticationFailureHandler) // 处理登录失败
            .and()
                .authorizeRequests() // 授权配置
                .antMatchers("/authentication/require",
                        "/login.html", "/code/image","/code/sms","/session/invalid").permitAll() // 无需认证的请求路径
                .anyRequest()  // 所有请求
                .authenticated() // 都需要认证
            .and()
                .sessionManagement() // 添加 Session管理器
                .invalidSessionUrl("/session/invalid") // Session失效后跳转到这个链接
            ......
}
```

上面配置了Session失效后跳转到`/session/invalid`，并且将这个URL添加到了免认证路径中。

在Controller里添加一个方法，映射该请求：

```
@GetMapping("session/invalid")
@ResponseStatus(HttpStatus.UNAUTHORIZED)
public String sessionInvalid(){
    return "session已失效，请重新认证";
}
```

### Session并发控制

**后者将前者踢出的策略**

```
@Autowired
private MySessionExpiredStrategy sessionExpiredStrategy;

@Override
protected void configure(HttpSecurity http) throws Exception {
    http.addFilterBefore(validateCodeFilter, UsernamePasswordAuthenticationFilter.class) // 添加验证码校验过滤器
        .addFilterBefore(smsCodeFilter,UsernamePasswordAuthenticationFilter.class) // 添加短信验证码校验过滤器
            .formLogin() // 表单登录
                .loginPage("/authentication/require") // 登录跳转 URL
                .loginProcessingUrl("/login") // 处理表单登录 URL
                .successHandler(authenticationSucessHandler) // 处理登录成功
                .failureHandler(authenticationFailureHandler) // 处理登录失败
            .and()
                .authorizeRequests() // 授权配置
                .antMatchers("/authentication/require",
                        "/login.html", "/code/image","/code/sms","/session/invalid").permitAll() // 无需认证的请求路径
                .anyRequest()  // 所有请求
                .authenticated() // 都需要认证
            .and()
                .sessionManagement() // 添加 Session管理器
                .invalidSessionUrl("/session/invalid") // Session失效后跳转到这个链接
                .maximumSessions(1)
                .expiredSessionStrategy(sessionExpiredStrategy)
                .and()
            ......
```

`maximumSessions`配置了最大Session并发数量为1个，如果mrbird这个账户登录后，在另一个客户端也使用mrbird账户登录，那么第一个使用mrbird登录的账户将会失效，类似于一个先入先出队列。`expiredSessionStrategy`配置了Session在并发下失效后的处理策略，这里为我们自定义的策略`MySessionExpiredStrategy`。

`MySessionExpiredStrategy`实现`SessionInformationExpiredStrategy`：

```
@Component
public class MySessionExpiredStrategy implements SessionInformationExpiredStrategy {
    @Override
    public void onExpiredSessionDetected(SessionInformationExpiredEvent event) throws IOException, ServletException {
        HttpServletResponse response = event.getResponse();
        response.setStatus(HttpStatus.UNAUTHORIZED.value());
        response.setContentType("application/json;charset=utf-8");
        response.getWriter().write("您的账号已经在别的地方登录，当前登录已失效。如果密码遭到泄露，请立即修改密码！");
    }
}
```

若验证此效果, 可以将session超时时间设置的久一点. 然后在两个浏览器登录,在前一个刷新.

**当Session达到最大有效数的时候，不再允许相同的账户登录**

要实现这个功能只需要在上面的配置中添加：

```
.and()
    .sessionManagement() // 添加 Session管理器
    .invalidSessionUrl("/session/invalid") // Session失效后跳转到这个链接
    .maximumSessions(1)
    .maxSessionsPreventsLogin(true)
    .expiredSessionStrategy(sessionExpiredStrategy)
    .and()
```

此时多浏览器登录将受限.

**提示:在实际开发中，发现Session并发控制只对Spring Security默认的登录方式——账号密码登录有效，而像短信验证码登录，社交账号登录并不生效，解决方案可以参考开源项目https://github.com/wuyouzhuguli/FEBS-Security**

### Session集群处理

Session集群听着高大上，其实实现起来很简单。当我们登录成功后，用户认证的信息存储在Session中，而这些Session默认是存储在运行运用的服务器上的，比如Tomcat，netty等。当应用集群部署的时候，用户在A应用上登录认证了，后续通过负载均衡可能会把请求发送到B应用，而B应用服务器上并没有与该请求匹配的认证Session信息，所以用户就需要重新进行认证。要解决这个问题，我们可以把Session信息存储在第三方容器里（如Redis集群），而不是各自的服务器，这样应用集群就可以通过第三方容器来共享Session了。

引入Redis和Spring Session依赖：

```
<dependency>
    <groupId>org.springframework.session</groupId>
    <artifactId>spring-session</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis</artifactId>
</dependency>
```

新版可尝试依赖

```
<dependency> 
	<groupId>org.springframework.session</groupId> 
	<artifactId>spring-session-data-redis</artifactId>
</dependency>
```

然后在yml中配置Session存储方式为Redis：

```
spring:
  session:
    store-type: redis
```

为了方便，Redis配置采用默认配置即可。

开启Redis，并且启动两个应用实例，一个端口为8080，另一个端口为9090。

在8080端口应用上登录,然后访问9090端口应用的主页,登录也是生效的

### 操作session

`SessionRegistry`包含了一些使用的操作Session的方法，比如：

1. 踢出用户（让Session失效）：

   ```
   String currentSessionId = request.getRequestedSessionId();
   sessionRegistry.getSessionInformation(sessionId).expireNow();
   ```

2. 获取所有Session信息：

   ```
   List<Object> principals = sessionRegistry.getAllPrincipals();
   ```

## 退出登录

Spring Security默认的退出登录URL为`/logout`，退出登录后，Spring Security会做如下处理：

1. 使当前的Sesion失效；
2. 清除与当前用户关联的RememberMe记录；
3. 清空当前的SecurityContext；
4. 重定向到登录页。

Spring Security允许我们通过配置来更改上面这些默认行为。

**自定义退出登录行为**

```
.and()
    .logout()
    .logoutUrl("/signout")
    .logoutSuccessUrl("/signout/success")
    .deleteCookies("JSESSIONID")
.and()
```

面配置了退出登录的URL为`/signout`，退出成功后跳转的URL为`/signout/success`，退出成功后删除名称为`JSESSIONID`的cookie。

在Controller中添加和`/signout/success`对应的方法：

```
@GetMapping("/signout/success")
public String signout() {
    return "退出成功，请重新登录";
}
```

接着将`/signout/success`添加到免认证路径里。启动项目，登录后访问`/signout`/

可看到退出成功，并且请求重定向到了`/signout/success`

除了指定`logoutUrl`外，我们也可以通过`logoutSuccessHandler`指定退出成功处理器来处理退出成功后的逻辑：

```
@Autowired
private MyLogOutSuccessHandler logOutSuccessHandler;

.and()
    .logout()
    .logoutUrl("/signout")
    // .logoutSuccessUrl("/signout/success")
    .logoutSuccessHandler(logOutSuccessHandler)
    .deleteCookies("JSESSIONID")
.and()
```

`MyLogOutSuccessHandler`实现`LogoutSuccessHandler`：

```
@Component
public class MyLogOutSuccessHandler implements LogoutSuccessHandler {
    @Override
    public void onLogoutSuccess(HttpServletRequest httpServletRequest, HttpServletResponse httpServletResponse, Authentication authentication) throws IOException, ServletException {
        httpServletResponse.setStatus(HttpStatus.UNAUTHORIZED.value());
        httpServletResponse.setContentType("application/json;charset=utf-8");
        httpServletResponse.getWriter().write("退出成功，请重新登录");
    }
}
```

效果和前面一样

## 权限控制

Spring Security权限控制可以配合授权注解使用

要开启这些注解，只需要在Spring Security配置文件中添加如下注解：

```
@Configuration
@EnableGlobalMethodSecurity(prePostEnabled = true)
public class BrowserSecurityConfig extends WebSecurityConfigurerAdapter {
   ...
}
```

在`UserDetailService`中，我们给当前登录用户授予了”admin”的权限，我们将这块代码改造一下：当登录用户为mrbird的时候，其拥有”admin”权限，其他用户则只有”test”权限：

```
@Configuration
public class UserDetailService implements UserDetailsService {
    @Autowired
    private PasswordEncoder passwordEncoder;

    @Override
    public UserDetails loadUserByUsername(String username) throws UsernameNotFoundException {
        // 模拟一个用户，替代数据库获取逻辑
        MyUser user = new MyUser();
        user.setUserName(username);
        user.setPassword(this.passwordEncoder.encode("123456"));
        // 输出加密后的密码
        System.out.println(user.getPassword());
        List<GrantedAuthority> authorities = new ArrayList<>();
        if (StringUtils.equalsIgnoreCase("mrbird", username)) {
            authorities = AuthorityUtils.commaSeparatedStringToAuthorityList("admin");
        } else {
            authorities = AuthorityUtils.commaSeparatedStringToAuthorityList("test");
        }
        return new User(username, user.getPassword(), user.isEnabled(),
                user.isAccountNonExpired(), user.isCredentialsNonExpired(),
                user.isAccountNonLocked(), authorities);
    }
}
```

添加一个方法，并且使用权限注解标明只有拥有“admin”权限的人才能访问：

```
@GetMapping("/auth/admin")
@PreAuthorize("hasAuthority('admin')")
public String authenticationTest() {
    return "您拥有admin权限，可以查看";
}
```

启动系统，使用mrbird账号登录, 在返回的认证信息中可以看到权限信息. 然后再访问/auth/admin可看到输出信息;

使用scott账号登录,在返回的认证信息中可以看到权限信息. 然后再访问/auth/admin将出错403;

我们可以自定义权限不足处理器来处理权限不足时候的操作。

新增一个处理器`MyAuthenticationAccessDeniedHandler`，实现`AccessDeniedHandler`接口：

```
@Component
public class MyAuthenticationAccessDeniedHandler implements AccessDeniedHandler {
    @Override
    public void handle(HttpServletRequest request, HttpServletResponse response, AccessDeniedException accessDeniedException) throws IOException {
        response.setStatus(HttpStatus.INTERNAL_SERVER_ERROR.value());
        response.setContentType("application/json;charset=utf-8");
        response.getWriter().write("很抱歉，您没有该访问权限");
    }
}
```

然后将这个处理器添加到Spring Security配置链中: 

```
@Autowired 
private MyAuthenticationAccessDeniedHandler authenticationAccessDeniedHandler;

@Override
protected void configure(HttpSecurity http) throws Exception {
    http.exceptionHandling()
            .accessDeniedHandler(authenticationAccessDeniedHandler)
        .and()
    ......
}
```

### 权限注解

Spring Security提供了三种不同的安全注解：

1.Spring Security自带的@Secured注解；

2.JSR-250的@RolesAllowed注解；

3.表达式驱动的注解，包括@PreAuthorize、@PostAuthorize、@PreFilter和 @PostFilter。

@**Secured**

在Spring-Security.xml中启用@Secured注解：

```
<global-method-security secured-annotations="enabled"/>
```

或者

```
@Configuration
@EnableGlobalMethodSecurity(securedEnabled = true)
public class WebSecurityConfig extends WebSecurityConfigurerAdapter {
}
```

例如只有拥有权限“ROLE_ADMIN”的用户才能访问下面这个方法：

```
@Secured("ROLE_ADMIN")
public void test(){
    ...
}
```

权限不足时，方法抛出Access Denied异常。

@Secured注解会使用一个String数组作为参数。每个String值是一个权限，调用这个方法至少需要具备其中的一个权限。如：

```
@Secured({"ROLE_ADMIN","ROLE_USER"})
public void test(){
    ...
}
```

@**RolesAllowed**

@RolesAllowed注解和@Secured注解在各个方面基本上都是一致的。启用@RolesAllowed注解：

```
<global-method-security jsr250-annotations="enabled"/>
```

或者

```
@Configuration
@EnableGlobalMethodSecurity(jsr250Enabled = true)
public class WebSecurityConfigurer extends WebSecurityConfigurerAdapter {
}
```

栗子：

```
@RolesAllowed("ROLE_ADMIN")
public void test(){
    ...
}
```

**SpEL注解**:

启用该注解：

```
<global-method-security pre-post-annotations="enabled"/>
```

或者

```
@Configuration
@EnableGlobalMethodSecurity(prePostEnabled = true)
public class WebSecurityConfigurer extends WebSecurityConfigurerAdapter {
}
```

@**PreAuthorize**

该注解用于方法前验证权限，比如限制非VIP用户提交blog的note字段字数不得超过1000字：

```
@PreAuthorize("hasRole('ROLE_ADMIN') and #form.note.length() <= 1000 or hasRole('ROLE_VIP')")
public void writeBlog(Form form){
    ...
}
```

表达式中的#form部分直接引用了方法中的同名参数。这使得Spring Security能够检查传入方法的参数，并将这些参数用于认证决策的制定。

常用的el表达式

| 表达式                    | 描述                                                         |
| ------------------------- | ------------------------------------------------------------ |
| hasRole([role])           | 当前用户是否拥有指定角色。                                   |
| hasAnyRole([role1,role2]) | 多个角色是一个以逗号进行分隔的字符串。如果当前用户拥有指定角色中的任意一个则返回true。 |

@**PostAuthorize**

方法后调用权限验证，比如校验方法返回值：

```
@PreAuthorize("hasRole(ROLE_USER)")
@PostAuthorize("returnObject.user.userName == principal.username")
public User getUserById(long id){
    ...		
}
```

Spring Security在SpEL中提供了名为returnObject 的变量。在这里方法返回一个User对象，所以这个表达式可以直接访问user对象中的userName属性。

#### 自定义权限验证方式

>  使用自定义权限验证的项目:若依 , el

接口都需要给超级管理员放行，而使用 `hasAnyRole('admin','user:list')` 每次都需要重复的添加 admin 权限，因此在可自定义权限验证方式，在验证的时候默认给拥有admin权限的用户放行。

```
@Service(value = "el")
public class ElPermissionConfig {

    public Boolean check(String ...permissions){
        // 获取当前用户的所有权限
        List<String> elPermissions = SecurityUtils.getCurrentUser().getAuthorities().stream().map(GrantedAuthority::getAuthority).collect(Collectors.toList());
        // 判断当前用户的所有权限是否包含接口上定义的权限
        return elPermissions.contains("admin") || Arrays.stream(permissions).anyMatch(elPermissions::contains);
    }
}
```

```java
@PreAuthorize("@el.check('user:list','user:add')")
```

## CSRF

禁用,如果不使用session,可以禁用,如使用jwt

```
.and().csrf().disable();
```

开启配置

```
 .csrf()
//开启基于cookie的csrf保护
.csrfTokenRepository(CookieCsrfTokenRepository.withHttpOnlyFalse())
//忽略这些路径的csrf保护以便admin-client注册
.ignoringAntMatchers(
    adminContextPath + "/instances",
    adminContextPath + "/actuator/**"
                );
```

## SecurityContextHolder

获取当前登录对象,需要强转为实现了UserDetails的用户类

```
SecurityContextHolder.getContext().getAuthentication().getPrincipal()
```

## SessionCreationPolicy

rest客户端访问rest服务端默认状态的配置策略是：无状态的；

假如默认配置策略不是无状态配置，则需要配置为无状态；

若不配置无状态，则rest服务端会爆掉，堆积海量的sessionId；

Session状态策略

```
public enum SessionCreationPolicy {
	/** Always create an {@link HttpSession} */
	ALWAYS,
	/**
	 * Spring Security will never create an {@link HttpSession}, but will use the
	 * {@link HttpSession} if it already exists
	 */
	NEVER,
	/** Spring Security will only create an {@link HttpSession} if required */
	IF_REQUIRED,
	/**
	 * Spring Security will never create an {@link HttpSession} and it will never use it
	 * to obtain the {@link SecurityContext}
	 */
	STATELESS
}
```

如果使用jwt，可以如下配置

```
 // CSRF禁用，因为不使用session
    .csrf().disable()
    //...
// 基于token，所以不需要session
    .sessionManagement().sessionCreationPolicy(SessionCreationPolicy.STATELESS).and()
```

或者配置文件配置session策略

```html
security.sessions: stateless
```

## 单账号

```
@Configuration
@EnableWebSecurity
public class WebSecurityConfig extends WebSecurityConfigurerAdapter {

    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http
            .authorizeRequests()
                .antMatchers("/", "/home").permitAll()
                .anyRequest().authenticated()
                .and()
            .formLogin()
                .loginPage("/login")
                .permitAll()
                .and()
            .logout()
                .permitAll();
    }

    @Autowired
    public void configureGlobal(AuthenticationManagerBuilder auth) throws Exception {
        auth
            .inMemoryAuthentication()
                .withUser("user").password("password").roles("USER");
    }

}
```

- `configureGlobal(AuthenticationManagerBuilder auth)`方法，在内存中创建了一个用户，该用户的名称为user，密码为password，用户角色为USER。



## 若依添加自定义Filter校验JWT

```
/**
 * 认证失败处理类 返回未授权
 */
@Component
public class AuthenticationEntryPointImpl implements AuthenticationEntryPoint, Serializable
{
    private static final long serialVersionUID = -8970718410437077606L;

    @Override
    public void commence(HttpServletRequest request, HttpServletResponse response, AuthenticationException e)
            throws IOException
    {
        int code = HttpStatus.UNAUTHORIZED;  
        String msg = StringUtils.format("请求访问：{}，认证失败，无法访问系统资源", request.getRequestURI());
        ServletUtils.renderString(response, JSON.toJSONString(AjaxResult.error(code, msg)));
    }
}
```

```
/**
 * 自定义退出处理类 返回成功
 */
@Configuration
public class LogoutSuccessHandlerImpl implements LogoutSuccessHandler
{
    @Autowired
    private TokenService tokenService;

    @Override
    public void onLogoutSuccess(HttpServletRequest request, HttpServletResponse response, Authentication authentication)
            throws IOException, ServletException
    {
        LoginUser loginUser = tokenService.getLoginUser(request);
        if (StringUtils.isNotNull(loginUser))
        {
            String userName = loginUser.getUsername();
            // 删除用户缓存记录
            tokenService.delLoginUser(loginUser.getToken());
            // 记录用户退出日志
            AsyncManager.me().execute(AsyncFactory.recordLogininfor(userName, Constants.LOGOUT, "退出成功"));
        }
        ServletUtils.renderString(response, JSON.toJSONString(AjaxResult.error(HttpStatus.SUCCESS, "退出成功")));
    }
}
```

```
/**
 * token过滤器 验证token有效性
 * 使用filter 在没有通过dispatcher前
 * interceptor属于springmvc  通过了dispatcher
 * @author ruoyi
 */
@Component
public class JwtAuthenticationTokenFilter extends OncePerRequestFilter
{
    @Autowired
    private TokenService tokenService;

    @Override
    protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response, FilterChain chain)
            throws ServletException, IOException
    {
        LoginUser loginUser = tokenService.getLoginUser(request);
        //如果loginUser为null才去获取Authentication对象
        if (StringUtils.isNotNull(loginUser) && StringUtils.isNull(SecurityUtils.getAuthentication()))
        {
            //验证token有效期
            tokenService.verifyToken(loginUser);
            UsernamePasswordAuthenticationToken authenticationToken = new UsernamePasswordAuthenticationToken(loginUser, null, loginUser.getAuthorities());
            authenticationToken.setDetails(new WebAuthenticationDetailsSource().buildDetails(request));
            SecurityContextHolder.getContext().setAuthentication(authenticationToken);
        }
        chain.doFilter(request, response);
    }
}
```

```
/**
 * spring security配置
 * 
 * @author ruoyi
 */
@EnableGlobalMethodSecurity(prePostEnabled = true, securedEnabled = true)
public class SecurityConfig extends WebSecurityConfigurerAdapter
{
    /**
     * 自定义用户认证逻辑
     */
    @Autowired
    private UserDetailsService userDetailsService;
    
    /**
     * 认证失败处理类
     */
    @Autowired
    private AuthenticationEntryPointImpl unauthorizedHandler;

    /**
     * 退出处理类
     */
    @Autowired
    private LogoutSuccessHandlerImpl logoutSuccessHandler;

    /**
     * token认证过滤器
     */
    @Autowired
    private JwtAuthenticationTokenFilter authenticationTokenFilter;
    
    /**
     * 跨域过滤器
     */
    @Autowired
    private CorsFilter corsFilter;
    
    /**
     * 解决 无法直接注入 AuthenticationManager
     *
     * @return
     * @throws Exception
     */
    @Bean
    @Override
    public AuthenticationManager authenticationManagerBean() throws Exception
    {
        return super.authenticationManagerBean();
    }

    /**
     * anyRequest          |   匹配所有请求路径
     * access              |   SpringEl表达式结果为true时可以访问
     * anonymous           |   匿名可以访问
     * denyAll             |   用户不能访问
     * fullyAuthenticated  |   用户完全认证可以访问（非remember-me下自动登录）
     * hasAnyAuthority     |   如果有参数，参数表示权限，则其中任何一个权限可以访问
     * hasAnyRole          |   如果有参数，参数表示角色，则其中任何一个角色可以访问
     * hasAuthority        |   如果有参数，参数表示权限，则其权限可以访问
     * hasIpAddress        |   如果有参数，参数表示IP地址，如果用户IP和参数匹配，则可以访问
     * hasRole             |   如果有参数，参数表示角色，则其角色可以访问
     * permitAll           |   用户可以任意访问
     * rememberMe          |   允许通过remember-me登录的用户访问
     * authenticated       |   用户登录后可访问
     */
    @Override
    protected void configure(HttpSecurity httpSecurity) throws Exception
    {
        httpSecurity
                // CSRF禁用，因为不使用session
                .csrf().disable()
                // 认证失败处理类
                .exceptionHandling().authenticationEntryPoint(unauthorizedHandler).and()
                // 基于token，所以不需要session
                .sessionManagement().sessionCreationPolicy(SessionCreationPolicy.STATELESS).and()
                // 过滤请求
                .authorizeRequests()
                // 对于登录login 注册register 验证码captchaImage 允许匿名访问
                .antMatchers("/login", "/register", "/captchaImage").anonymous()
                .antMatchers(
                        HttpMethod.GET,
                        "/",
                        "/*.html",
                        "/**/*.html",
                        "/**/*.css",
                        "/**/*.js",
                        "/profile/**"
                ).permitAll()
                .antMatchers("/common/download**").anonymous()
                .antMatchers("/common/download/resource**").anonymous()
                .antMatchers("/swagger-ui.html").anonymous()
                .antMatchers("/swagger-resources/**").anonymous()
                .antMatchers("/webjars/**").anonymous()
                .antMatchers("/*/api-docs").anonymous()
                .antMatchers("/druid/**").anonymous()
                // 除上面外的所有请求全部需要鉴权认证
                .anyRequest().authenticated()
                .and()
                .headers().frameOptions().disable();
        httpSecurity.logout().logoutUrl("/logout").logoutSuccessHandler(logoutSuccessHandler);
        // 添加JWT filter
        httpSecurity.addFilterBefore(authenticationTokenFilter, UsernamePasswordAuthenticationFilter.class);
        // 添加CORS filter
        httpSecurity.addFilterBefore(corsFilter, JwtAuthenticationTokenFilter.class);
        httpSecurity.addFilterBefore(corsFilter, LogoutFilter.class);
    }

    /**
     * 强散列哈希加密实现
     */
    @Bean
    public BCryptPasswordEncoder bCryptPasswordEncoder()
    {
        return new BCryptPasswordEncoder();
    }

    /**
     * 身份认证接口
     */
    @Override
    protected void configure(AuthenticationManagerBuilder auth) throws Exception
    {
        auth.userDetailsService(userDetailsService).passwordEncoder(bCryptPasswordEncoder());
    }
}
```

> 注册自定义的jwt认证过滤器加在`UsernamePasswordAuthenticationFilter`前

```
httpSecurity.addFilterBefore(authenticationTokenFilter, UsernamePasswordAuthenticationFilter.class);
```

获取当前登录对象,需要强转为实现了UserDetails的用户类

```
SecurityContextHolder.getContext().getAuthentication().getPrincipal()
```

## 动态权限控制

通过`FilterInvocationSecurityMetadataSource`实现动态权限控制 , 该类的主要功能就是通过当前的请求地址，获取该地址需要的用户角色。

```
@Component
public class CustomFilterInvocationSecurityMetadataSource implements FilterInvocationSecurityMetadataSource {
    @Autowired
    MenuService menuService;
    // AntPathMatcher 是一个正则匹配工具
    AntPathMatcher antPathMatcher = new AntPathMatcher();
    //根据用户传来的请求地址，分析请求需要的角色，并将所需要的角色放在 Collection中
    @Override
    public Collection<ConfigAttribute> getAttributes(Object o) throws IllegalArgumentException {
        String requestUrl = ((FilterInvocation) o).getFullRequestUrl();
        List<Menu> menus = menuService.getAllMenusWithRole();
        for (Menu menu: menus){
            if (antPathMatcher.match(menu.getUrl(), requestUrl)){
                List<Role> roles = menu.getRoles();
                String[] str = new String[roles.size()];
                for (int i=0; i<roles.size(); i++){
                    str[i] = roles.get(i).getName();
                }
                return SecurityConfig.createList(str);
            }
        }
        // 没有匹配上的，只要登录之后就可以访问，这里“ROLE_LOGIN”只是一个标记.
        return SecurityConfig.createList("ROLE_LOGIN");
    }
   ...
}
```

还需要实现`AccessDecisionManager`判断当前用户是否当前url具备指定的角色 , 如果不具备，就抛出 AccessDeniedException 异常，否则不做任何事即可.

```
@Component
public class CustomUrlDecisionManager implements AccessDecisionManager {
    @Override
    public void decide(Authentication authentication, Object o, Collection<ConfigAttribute> collection) throws AccessDeniedException, InsufficientAuthenticationException {
        for (ConfigAttribute configAttribute : collection) {
            String needRole = configAttribute.getAttribute();
            //如果需要的角色是ROLE_LOGIN，说明当前请求的URL用户登陆后即可访问
            if ("ROLE_LOGIN".equals(needRole)){
                if (authentication instanceof AnonymousAuthenticationToken){
                 	throw new AccessDeniedException("尚未登录，请登录！");
                }else {
                    return;
                }
            }
            Collection<? extends GrantedAuthority> authorities = authentication.getAuthorities();
            for (GrantedAuthority authority : authorities) {
                if (authority.getAuthority().equals(needRole)){
                    return;
                }
            }
        }
        throw new AccessDeniedException("权限不足，请联系管理员！");
    }
}
```

### 基于SpEL方式

注册一个包含返回值为布尔的公共方法: 方法具有两个参数 `Authentication`和`HttpServletRequest` ,目的在于拿着当前请求去和当前认证信息中包含的角色进行访问控制判断.

然后按照`@bean名称.方法名(authentication,request)`的格式配置到`HttpSecurity`对象中.

```
httpSecurity.authorizeRequests()
	.anyRequest()
	.access("@roleChecker.check(authentication,request)");
```

### AuthorizationManager

5.6版本增加的泛型接口, 它用来检查当前认证信息是否可以访问特定对象T.

> 对比SpEL方式 ,`RoleChecker`不就是`AuthorizationManager<HttpServletRequest>`?`AuthorizationManager`就是将这种访问决策抽象的更加泛化。

```
@FunctionalInterface
public interface AuthorizationManager<T> {
 
 default void verify(Supplier<Authentication> authentication, T object) {
  AuthorizationDecision decision = check(authentication, object);
        // 授权决策没有经过允许就403
  if (decision != null && !decision.isGranted()) {
   throw new AccessDeniedException("Access Denied");
  }
        // todo 没有null 的情况
 }

    // 钩子方法。
 @Nullable
 AuthorizationDecision check(Supplier<Authentication> authentication, T object);

}
```

在5.6中, 就可以这样去实现了

```

        httpSecurity.authorizeHttpRequests()
                .anyRequest()
                .access((authenticationSupplier, requestAuthorizationContext) -> {
                    // 当前用户的权限信息 比如角色
                    Collection<? extends GrantedAuthority> authorities = authenticationSupplier.get().getAuthorities();
                    // 当前请求上下文
                    // 我们可以获取携带的参数
                    Map<String, String> variables = requestAuthorizationContext.getVariables();
                    // 我们可以获取原始request对象
                    HttpServletRequest request = requestAuthorizationContext.getRequest();
                    //todo 根据这些信息 和业务写逻辑即可 最终决定是否授权 isGranted
                    boolean isGranted = true;
                    return new AuthorizationDecision(isGranted);
                });
```



# OAuth2

[OAuth](https://oauth.net/2/)是一种用来规范令牌（Token）发放的授权机制，主要包含了四种授权模式：授权码模式、简化模式、密码模式和客户端模式。Spring Security OAuth2对这四种授权模式进行了实现。

http://www.ruanyifeng.com/blog/2014/05/oauth_2_0.html 阮一峰讲auth2

https://datatracker.ietf.org/doc/html/rfc6749#section-4.1 RFC6749 The OAuth 2.0 Authorization Framework书籍

**四种授权模式**

在了解这四种授权模式之前，我们需要先学习一些和OAuth相关的名词。举个社交登录的例子吧，比如在浏览器上使用QQ账号登录虎牙直播，这个过程可以提取出以下几个名词：

1. **Third-party application** 第三方应用程序，比如这里的虎牙直播；
2. **HTTP service** HTTP服务提供商，比如这里的QQ（腾讯）;
3. **Resource Owner** 资源所有者，就是QQ的所有人，你；
4. **User Agent** 用户代理，这里指浏览器；
5. **Authorization server** 认证服务器，这里指QQ提供的第三方登录服务；
6. **Resource server** 资源服务器，这里指虎牙直播提供的服务，比如高清直播，弹幕发送等（需要认证后才能使用）。

认证服务器和资源服务器可以在同一台服务器上，比如前后端分离的服务后台，它即供认证服务（认证服务器，提供令牌），客户端通过令牌来从后台获取服务（资源服务器）；它们也可以不在同一台服务器上，比如上面第三方登录的例子。

**授权模式一  授权码模式**

授权码模式是最能体现OAuth2协议，最严格，流程最完整的授权模式，授权码通过前端传送，令牌则是储存在后端，而且所有与资源服务器的通信都在后端完成。流程如下所示：

![](picture/auth01-01.png)

**授权模式二  密码模式** 

在密码模式中，用户像客户端提供用户名和密码，客户端通过用户名和密码到认证服务器获取令牌。流程如下所示：

![](picture/auth02-01.png)

A. 用户向客户端提供用户名和密码；

B. 客户端向认证服务器换取令牌；

C. 发放令牌。

B步骤中，客户端发出的HTTP请求，包含以下参数：

1. grant_type：表示授权类型，此处的值固定为”password”，必选项。
2. username：表示用户名，必选项。
3. password：表示用户的密码，必选项。
4. scope：表示权限范围，可选项。

**详细信息其他两种授权模式查看上面的链接**

## Spring Security OAuth2

Spring框架对OAuth2协议进行了实现，下面学习下上面两种模式在Spring Security OAuth2相关框架的使用。

Spring Security OAuth2主要包含认证服务器和资源服务器这两大块的实现：

![](picture/security-auth2-01.png)

认证服务器主要包含了四种授权模式的实现和Token的生成与存储，我们也可以在认证服务器中自定义获取Token的方式（后面会介绍到）；资源服务器主要是在Spring Security的过滤器链上加了OAuth2AuthenticationProcessingFilter过滤器，即使用OAuth2协议发放令牌认证的方式来保护我们的资源。

### 配置认证服务器

新建一个Spring Boot项目，版本为2.1.6.RELEASE，并引入相关依赖

```
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>2.1.6.RELEASE</version>
        <relativePath/> <!-- lookup parent from repository -->
    </parent>
    <groupId>cc.mrbird</groupId>
    <artifactId>security</artifactId>
    <version>0.0.1-SNAPSHOT</version>
    <name>security</name>
    <description>Demo project for Spring Boot</description>

    <properties>
        <java.version>1.8</java.version>
        <spring-cloud.version>Greenwich.SR1</spring-cloud.version>
    </properties>

    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-oauth2</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-security</artifactId>
        </dependency>
        <dependency>
            <groupId>org.apache.commons</groupId>
            <artifactId>commons-lang3</artifactId>
        </dependency>
    </dependencies>

    <dependencyManagement>
        <dependencies>
            <dependency>
                <groupId>org.springframework.cloud</groupId>
                <artifactId>spring-cloud-dependencies</artifactId>
                <version>${spring-cloud.version}</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>
        </dependencies>
    </dependencyManagement>

    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>
        </plugins>
    </build>
</project>
```

在创建认证服务器前，我们先定义一个`MyUser`对象：

```
public class MyUser implements Serializable {
    private static final long serialVersionUID = 3497935890426858541L;
    private String userName;
    private String password;
    private boolean accountNonExpired = true;
    private boolean accountNonLocked= true;
    private boolean credentialsNonExpired= true;
    private boolean enabled= true;
    // get set 略
}
```

接着定义`UserDetailService`实现`org.springframework.security.core.userdetails.UserDetailsService`接口：

```
@Service
public class UserDetailService implements UserDetailsService {
    @Autowired
    private PasswordEncoder passwordEncoder;

    @Override
    public UserDetails loadUserByUsername(String username) throws UsernameNotFoundException {
        MyUser user = new MyUser();
        user.setUserName(username);
        user.setPassword(this.passwordEncoder.encode("123456"));
        return new User(username, user.getPassword(), user.isEnabled(),
                user.isAccountNonExpired(), user.isCredentialsNonExpired(),
                user.isAccountNonLocked(), AuthorityUtils.commaSeparatedStringToAuthorityList("admin"));
    }
}
```

这里的逻辑是用什么账号登录都可以，但是密码必须为123456，并且拥有”admin”权限.(同前面security)

创建认证服务器很简单，只需要在Spring Security的配置类上使用`@EnableAuthorizationServer`注解标注即可。创建`AuthorizationServerConfig`，代码如下所示：

```
@Configuration
@EnableAuthorizationServer
public class AuthorizationServerConfig extends WebSecurityConfigurerAdapter {
    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();
    }
}
```

这时候启动项目，会发现控制台打印出了随机分配的client-id和client-secret.

为了方便后面的测试，我们可以手动指定这两个值。在Spring Boot配置文件application.yml中添加如下配置:

```
security:
  oauth2:
    client:
      client-id: test
      client-secret: test1234
```

再启动项目查看输出验证是否替换成功.

### 授权码模式获取令牌

接下来开始往认证服务器请求授权码。打开浏览器，访问

```
http://localhost:8080/oauth/authorize?response_type=code&client_id=test&redirect_uri=http://mrbird.cc&scope=all&state=hello
```

URL中的几个参数在上面的授权码模式的A步骤里都有详细说明。这里response_type必须为code，表示授权码模式，client_id就是刚刚在配置文件中手动指定的test，redirect_uri这里随便指定一个地址即可，主要是用来重定向获取授权码的，scope指定为all，表示所有权限。

访问这个链接后，页面如下所示(这是spring security模块的功能,需要登录)：

![](picture/security-login.png)

需要登录认证，根据我们前面定义的`UserDetailService`逻辑，这里用户名随便输，密码为123456即可。此时还不能完成认证,输入后，页面跳转如下所示：

![](picture/security-error.png)

原因是上面指定的redirect_uri必须同时在配置文件中指定，我们往application.yml添加配置：

```
security:
  oauth2:
    client:
      client-id: test
      client-secret: test1234
      registered-redirect-uri: http://mrbird.cc
```

重启项目，重新执行上面的步骤，登录成功后页面成功跳转到了授权页面：

![](picture/security-approval.png)



选择同意Approve，然后点击Authorize按钮后，页面跳转到了我们指定的redirect_uri，并且带上了授权码信息:

![](picture/security-redirect-ok.png)

到这里我们就可以用这个授权码从认证服务器获取令牌Token了。

使用postman发送如下请求POST请求[localhost:8080/oauth/token](localhost:8080/oauth/token)(注意参数)：

![](picture/security-postman.png)

这里要填的参数和上面介绍的授权码模式D步骤介绍的一致。grant_type固定填authorization_code，code为上一步获取到的授权码，client_id和redirect_uri必须和我们上面定义的一致。

除了这几个参数外，我们还需要在请求头中填写：

![](picture/security-postman1.png)

key为Authorization，value为`Basic`加上`client_id:client_secret`经过base64加密后的值（可以使用http://tool.chinaz.com/Tools/Base64.aspx）:

![](picture/security-client-base64.png)

参数填写无误后，点击发送便可以获取到令牌Token：

```
{
    "access_token": "950018df-0199-4936-aa80-a3a66183f634",
    "token_type": "bearer",
    "refresh_token": "cc22e8b2-e069-459d-8c24-cfda0bc72128",
    "expires_in": 42827,
    "scope": "all"
}
```

**一个授权码只能换一次令牌**，如果再次点击postman的发送按钮，将返回：

```
{
    "error": "invalid_grant",
    "error_description": "Invalid authorization code: xw8x55"
}
```

### 密码模式获取令牌

和授权码模式相比，使用密码模式获取令牌就显得简单多了。

同样使用postman发送POST请求[localhost:8080/oauth/token](localhost:8080/oauth/token)：

![](picture/security-pwd-postman.png)

grant_type填password，表示密码模式；然后填写用户名和密码，头部也需要填写Authorization信息，内容和授权码模式介绍的一致，这里就不贴图了

点击发送，也可以获得令牌：

```
{
    "access_token": "d612cf50-6499-4a0c-9cd4-9c756839aa12",
    "token_type": "bearer",
    "refresh_token": "fdc6c77f-b910-46dc-a349-835dc0587919",
    "expires_in": 43090,
    "scope": "all"
}
```

### 配置资源服务器

为什么需要资源服务器呢？我们先来看下在没有定义资源服务器的时候，使用Token去获取资源时会发生什么。

定义一个REST接口：

```
@RestController
public class UserController {

    @GetMapping("index")
    public Object index(Authentication authentication){
        return authentication;
    }
}
```

启动项目，为了方便我们使用密码模式获取令牌，然后使用该令牌获取`/index`这个资源：

![](picture/security-resource-no.png)

Authorization值为`token_type access_token`，发送请求后，返回：

```
{
    "timestamp": "2019-03-24T13:13:43.818+0000",
    "status": 401,
    "error": "Unauthorized",
    "message": "Unauthorized",
    "path": "/index"
}
```

虽然令牌是正确的，但是并无法访问`/index`，所以我们必须配置资源服务器，让客户端可以通过合法的令牌来获取资源。

资源服务器的配置也很简单，只需要在配置类上使用`@EnableResourceServer`注解标注即可

```
@Configuration
@EnableResourceServer
public class ResourceServerConfig  {
}
```

重启服务，重复上面的步骤，再次访问`/index`便可以成功获取到信息：

```
{
    "authorities": [
        {
            "authority": "admin"
        }
    ],
    "details": {
        "remoteAddress": "0:0:0:0:0:0:0:1",
        "sessionId": null,
        "tokenValue": "621f59ba-3161-4c9b-aff8-a8335ce6e3cc",
        "tokenType": "bearer",
        "decodedDetails": null
    },
    "authenticated": true,
    "userAuthentication": {
        "authorities": [
            {
                "authority": "admin"
            }
        ],
        "details": {
            "grant_type": "password",
            "username": "mrbird",
            "scope": "all"
        },
        "authenticated": true,
        "principal": {
            "password": null,
            "username": "mrbird",
            "authorities": [
                {
                    "authority": "admin"
                }
            ],
            "accountNonExpired": true,
            "accountNonLocked": true,
            "credentialsNonExpired": true,
            "enabled": true
        },
        "credentials": null,
        "name": "mrbird"
    },
    "credentials": "",
    "oauth2Request": {
        "clientId": "test",
        "scope": [
            "all"
        ],
        "requestParameters": {
            "grant_type": "password",
            "username": "mrbird",
            "scope": "all"
        },
        "resourceIds": [],
        "authorities": [
            {
                "authority": "ROLE_USER"
            }
        ],
        "approved": true,
        "refresh": false,
        "redirectUri": null,
        "responseTypes": [],
        "extensions": {},
        "refreshTokenRequest": null,
        "grantType": "password"
    },
    "clientOnly": false,
    "principal": {
        "password": null,
        "username": "mrbird",
        "authorities": [
            {
                "authority": "admin"
            }
        ],
        "accountNonExpired": true,
        "accountNonLocked": true,
        "credentialsNonExpired": true,
        "enabled": true
    },
    "name": "mrbird"
}
```

在同时定义了认证服务器和资源服务器后，再去使用授权码模式获取令牌可能会遇到 Full authentication is required to access this resource 的问题，这时候只要确保认证服务器先于资源服务器配置即可，比如在认证服务器的配置类上使用`@Order(1)`标注，在资源服务器的配置类上使用`@Order(2)`标注。

## 自定义Token获取方式

### 自定义用户名密码方式获取令牌

在前面的基础上，我们先在资源服务器上加入一些基本的Spring Security配置:

```
@Configuration
@EnableResourceServer
public class ResourceServerConfig extends ResourceServerConfigurerAdapter {
    @Autowired
    private MyAuthenticationSucessHandler authenticationSucessHandler;
    @Autowired
    private MyAuthenticationFailureHandler authenticationFailureHandler;
    @Override
    public void configure(HttpSecurity http) throws Exception {
        http.formLogin() // 表单登录
                .loginProcessingUrl("/login") // 处理表单登录 URL
                .successHandler(authenticationSucessHandler) // 处理登录成功
                .failureHandler(authenticationFailureHandler) // 处理登录失败
            .and()
                .authorizeRequests() // 授权配置
                .anyRequest()  // 所有请求
                .authenticated() // 都需要认证
            .and()
                .csrf().disable();
    }
}
```

其中, MyAuthenticationFailureHandler与security处理失败中一样.

如何在登录成功处理器里返回令牌。在研究Spring Security OAuth2自带的令牌获取方式后，会发现令牌的产生可以归纳为以下几个步骤：

![](picture/security-oauth2-route.png)

我们可以参考这个流程，来实现在登录成功处理器`MyAuthenticationSucessHandler`里生成令牌并返回：

```
@Component
public class MyAuthenticationSucessHandler implements AuthenticationSuccessHandler {
    private Logger log = LoggerFactory.getLogger(this.getClass());

    @Autowired
    private ClientDetailsService clientDetailsService;
    @Autowired
    private AuthorizationServerTokenServices authorizationServerTokenServices;

    @Override
    public void onAuthenticationSuccess(HttpServletRequest request, HttpServletResponse response, Authentication authentication) throws IOException {
        // 1. 从请求头中获取 ClientId
        String header = request.getHeader("Authorization");
        if (header == null || !header.startsWith("Basic ")) {
            throw new UnapprovedClientAuthenticationException("请求头中无client信息");
        }

        String[] tokens = this.extractAndDecodeHeader(header, request);
        String clientId = tokens[0];
        String clientSecret = tokens[1];

        TokenRequest tokenRequest = null;

        // 2. 通过 ClientDetailsService 获取 ClientDetails
        ClientDetails clientDetails = clientDetailsService.loadClientByClientId(clientId);

        // 3. 校验 ClientId和 ClientSecret的正确性
        if (clientDetails == null) {
            throw new UnapprovedClientAuthenticationException("clientId:" + clientId + "对应的信息不存在");
        } else if (!StringUtils.equals(clientDetails.getClientSecret(), clientSecret)) {
            throw new UnapprovedClientAuthenticationException("clientSecret不正确");
        } else {
            // 4. 通过 TokenRequest构造器生成 TokenRequest
            tokenRequest = new TokenRequest(new HashMap<>(), clientId, clientDetails.getScope(), "custom");
        }

        // 5. 通过 TokenRequest的 createOAuth2Request方法获取 OAuth2Request
        OAuth2Request oAuth2Request = tokenRequest.createOAuth2Request(clientDetails);
        // 6. 通过 Authentication和 OAuth2Request构造出 OAuth2Authentication
        OAuth2Authentication auth2Authentication = new OAuth2Authentication(oAuth2Request, authentication);

        // 7. 通过 AuthorizationServerTokenServices 生成 OAuth2AccessToken
        OAuth2AccessToken token = authorizationServerTokenServices.createAccessToken(auth2Authentication);

        // 8. 返回 Token
        log.info("登录成功");
        response.setContentType("application/json;charset=UTF-8");
        response.getWriter().write(new ObjectMapper().writeValueAsString(token));
    }

    private String[] extractAndDecodeHeader(String header, HttpServletRequest request) {
        byte[] base64Token = header.substring(6).getBytes(StandardCharsets.UTF_8);

        byte[] decoded;
        try {
            decoded = Base64.getDecoder().decode(base64Token);
        } catch (IllegalArgumentException var7) {
            throw new BadCredentialsException("Failed to decode basic authentication token");
        }

        String token = new String(decoded, StandardCharsets.UTF_8);
        int delim = token.indexOf(":");
        if (delim == -1) {
            throw new BadCredentialsException("Invalid basic authentication token");
        } else {
            return new String[]{token.substring(0, delim), token.substring(delim + 1)};
        }
    }
}
```

启动项目，使用postman发送登录请求[localhost:8080/login](localhost:8080/login)：

![](picture/security-oauth2-tologin.png)

![](picture/security-oauth2-login.png)

点击发送后便可以成功获取到令牌：

```
{
    "access_token": "88a3dd6c-ab27-41af-95ee-5cd406fe5ab1",
    "token_type": "bearer",
    "refresh_token": "b316177d-68e9-4fc9-9f4a-804a7367ebc9",
    "expires_in": 43199
}
```

使用这个令牌便可以成功访问`/index`接口.

### 短信验证码获取令牌+Redis

在Spring Security短信验证码登录一节中，我们实现了通过短信验证码登录系统的功能，通过短信验证码获取令牌和它唯一的区别就是验证码的存储策略。之前的例子验证码存储在Session中，现在使用令牌的方式和系统交互后Session已经不适用了，我们可以使用第三方存储来保存我们的验证码（无论是短信验证码还是图形验证码都是一个道理），比如Redis等。

```
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis</artifactId>
</dependency>
```

在Spring Security短信验证码登录和上面自定义获取令牌的基础上,定义一个`RedisCodeService`，用于验证码的增删改：

```
/**
 * Redis操作验证码服务
 */
@Service
public class RedisCodeService {
    private final static String SMS_CODE_PREFIX = "SMS_CODE:";
    private final static Integer TIME_OUT = 300;

    @Autowired
    private StringRedisTemplate redisTemplate;
    /**
     * 保存验证码到 redis
     * @param smsCode 短信验证码
     */
    public void save(SmsCode smsCode, ServletWebRequest request, String mobile) throws Exception {
        redisTemplate.opsForValue().set(key(request, mobile), smsCode.getCode(), TIME_OUT, TimeUnit.SECONDS);
    }
    /**
     * 获取验证码
     * @return 验证码
     */
    public String get(ServletWebRequest request, String mobile) throws Exception {
        return redisTemplate.opsForValue().get(key(request, mobile));
    }
    /**
     * 移除验证码
     */
    public void remove(ServletWebRequest request, String mobile) throws Exception {
        redisTemplate.delete(key(request, mobile));
    }

    private String key(ServletWebRequest request, String mobile) throws Exception {
        String deviceId = request.getHeader("deviceId");
        if (StringUtils.isBlank(deviceId)) {
            throw new Exception("请在请求头中设置deviceId");
        }
        return SMS_CODE_PREFIX + deviceId + ":" + mobile;
    }
}
```

在ValidateController的获取验证码处修改为存入redis

```
@GetMapping("/code/sms")
public void createSmsCode(HttpServletRequest request, HttpServletResponse response, String mobile) throws Exception {
    SmsCode smsCode = createSMSCode();
    redisCodeService.save(smsCode, new ServletWebRequest(request), mobile);
    // 输出验证码到控制台代替短信发送服务
    System.out.println("手机号" + mobile + "的登录验证码为：" + smsCode.getCode() + "，有效时间为120秒");
}
```

在SmsCodeFilter的校验验证码方法处修改为从redis处理:

```
private void validateCode(ServletWebRequest servletWebRequest) throws Exception {
    String smsCodeInRequest = ServletRequestUtils.getStringParameter(servletWebRequest.getRequest(), "smsCode");
    String mobileInRequest = ServletRequestUtils.getStringParameter(servletWebRequest.getRequest(), "mobile");
    String codeInRedis = redisCodeService.get(servletWebRequest, mobileInRequest);
    if (StringUtils.isBlank(smsCodeInRequest)) {
        throw new Exception("验证码不能为空！");
    }
    if (codeInRedis == null) {
        throw new Exception("验证码已过期！");
    }
    if (!StringUtils.equalsIgnoreCase(codeInRedis, smsCodeInRequest)) {
        throw new Exception("验证码不正确！");
    }
    redisCodeService.remove(servletWebRequest, mobileInRequest);
}
```

启动系统，使用postman发送验证码：

![](picture/security-oauth2-sms-1.png)

请求头中带上deviceId（这里为随便填写的模拟值）：

![](picture/security-oauth2-sms-2.png)

点击发送后，控制台输出：

```
手机号17720202020的登录验证码为：619963，有效时间为120秒
```

接着用这个验证码去换取令牌，使用postman发送如下请求：

![](picture/security-oauth2-sms-3.png)

同样请求头中要带上deviceId和经过base64加密的`client_id:client_secret`：

![](picture/security-oauth2-sms-4.png)

点击发送后，便可以成功获取到令牌：

```
{
    "access_token": "7fe22e67-1a11-4708-8707-0100555a9d1a",
    "token_type": "bearer",
    "refresh_token": "7c7a814f-2ace-4171-9748-56cb1994b04b",
    "expires_in": 41982
}
```

## 自定义令牌配置

在前面，我们获取到的令牌都是基于Spring Security OAuth2默认配置生成的，Spring Security允许我们自定义令牌配置，比如不同的client_id对应不同的令牌，令牌的有效时间，令牌的存储策略等；我们也可以使用JWT来替换默认的令牌。

我们让认证服务器`AuthorizationServerConfig`继承`AuthorizationServerConfigurerAdapter`，并重写它的`configure(ClientDetailsServiceConfigurer clients)`方法.

认证服务器在继承了AuthorizationServerConfigurerAdapter适配器后，需要重写`configure(AuthorizationServerEndpointsConfigurer endpoints)`方法，指定 `AuthenticationManager`和`UserDetailService`。

```
@Configuration
@EnableAuthorizationServer
public class AuthorizationServerConfig extends AuthorizationServerConfigurerAdapter {
   // ......
    @Autowired
    private AuthenticationManager authenticationManager;
    @Autowired
    private UserDetailService userDetailService;
    @Override
    public void configure(AuthorizationServerEndpointsConfigurer endpoints) {
        endpoints.authenticationManager(authenticationManager)
                .userDetailsService(userDetailService);
    }
    @Override
    public void configure(ClientDetailsServiceConfigurer clients) throws Exception {
        clients.inMemory()
                .withClient("test1")
                .secret("test1111")
                .accessTokenValiditySeconds(3600)
                .refreshTokenValiditySeconds(864000)
                .scopes("all", "a", "b", "c")
                .authorizedGrantTypes("password")
            .and()
                .withClient("test2")
                .secret("test2222")
                .accessTokenValiditySeconds(7200);
    }
}
```

创建一个新的配置类`SecurityConfig`，在里面注册我们需要的`AuthenticationManager`Bean：

```
@Configuration
public class SecurityConfig extends WebSecurityConfigurerAdapter {
    @Bean(name = BeanIds.AUTHENTICATION_MANAGER)
    @Override
    public AuthenticationManager authenticationManagerBean() throws Exception {
        return super.authenticationManagerBean();
    }
}
```

此外，重写`configure(ClientDetailsServiceConfigurer clients)`方法主要配置了：

1. 定义两个client_id，及客户端可以通过不同的client_id来获取不同的令牌；
2. client_id为test1的令牌有效时间为3600秒，client_id为test2的令牌有效时间为7200秒；
3. client_id为test1的refresh_token（下面会介绍到）有效时间为864000秒，即10天，也就是说在这10天内都可以通过refresh_token来换取新的令牌；
4. 在获取client_id为test1的令牌的时候，scope只能指定为all，a，b或c中的某个值，否则将获取失败；
5. 只能通过密码模式(password)来获取client_id为test1的令牌，而test2则无限制。

启动项目，演示几个效果。启动项目后使用密码模式获取test1的令牌：

![](picture/security-oauth2-config-token.png)

和前面介绍的那样，头部需要传入`test1:test1111`经过base64加密后的值：

![](picture/security-oauth2-config-header.png)

点击发送后，意外的返回了错误！

![](picture/security-oauth2-config-error.png)

控制台输出了 Encoded password does not look like BCrypt 的告警。

查阅资料后发现，在新版本的spring-cloud-starter-oauth2指定client_secret的时候需要进行加密处理：

```
 @Override
    public void configure(ClientDetailsServiceConfigurer clients) throws Exception {
        clients.inMemory()
                .withClient("test1")
                .secret(new BCryptPasswordEncoder().encode("test1111"))
                .accessTokenValiditySeconds(3600)
                .refreshTokenValiditySeconds(864000)
                .scopes("all", "a", "b", "c")
                .authorizedGrantTypes("password")
            .and()
                .withClient("test2")
                .secret(new BCryptPasswordEncoder().encode("test2222"))
                .accessTokenValiditySeconds(7200);
    }
```

在前面自定义登录认证获取令牌一节中，我们在`MyAuthenticationSucessHandler`判断了client_secret的值是否正确。由于我们这里client_secret加密了，所以判断逻辑需要调整为下面这样:

```
else if (!passwordEncoder.matches(clientSecret, clientDetails.getClientSecret())) {
    throw new UnapprovedClientAuthenticationException("clientSecret不正确");
} 
```

修改后重启项目，重新使用密码模式获取令牌：

```
{
    "access_token": "c23376b0-efa3-4905-8356-8c9583c2a2a0",
    "token_type": "bearer",
    "expires_in": 3599,
    "scope": "all"
}
```

可以看到`expires_in`的时间是我们定义的3600秒

如果将scope指定为d再获取令牌:(因为test1只指定了scope为all,a,b,c的允许)

![](picture/security-oauth2-config-error1.png)

默认令牌是存储在内存中的，我们可以将它保存到第三方存储中，比如Redis。

创建`TokenStoreConfig`：

```
@Configuration
public class TokenStoreConfig {
    @Autowired
    private RedisConnectionFactory redisConnectionFactory;
    @Bean
    public TokenStore redisTokenStore (){
        return new RedisTokenStore(redisConnectionFactory);
    }
}
```

然后在认证服务器里指定该令牌存储策略。重写`configure(AuthorizationServerEndpointsConfigurer endpoints)`方法：

```
@Configuration
@EnableAuthorizationServer
public class AuthorizationServerConfig extends AuthorizationServerConfigurerAdapter {
    @Autowired
    private TokenStore redisTokenStore;
    @Override
    public void configure(AuthorizationServerEndpointsConfigurer endpoints) {
        endpoints.authenticationManager(authenticationManager)
            .tokenStore(redisTokenStore);
    }
    ......
}
```

重启项目获取令牌后，可以查看Redis中是否存储了令牌相关信息

![](picture/security-oauth2-config-redis.png)

### 使用JWT替换默认令牌

使用JWT替换默认的令牌（默认令牌使用UUID生成）只需要指定TokenStore为JwtTokenStore即可。

创建一个`JWTokenConfig`配置类：

```
@Configuration
public class JWTokenConfig {
    @Bean
    public TokenStore jwtTokenStore() {
        return new JwtTokenStore(jwtAccessTokenConverter());
    }
    @Bean
    public JwtAccessTokenConverter jwtAccessTokenConverter() {
        JwtAccessTokenConverter accessTokenConverter = new JwtAccessTokenConverter();
        accessTokenConverter.setSigningKey("test_key"); // 签名密钥
        return accessTokenConverter;
    }
}
```

签名密钥为`test_key`。在配置类里配置好`JwtTokenStore`后，我们在认证服务器里指定它：

```
@Configuration
@EnableAuthorizationServer
public class AuthorizationServerConfig extends AuthorizationServerConfigurerAdapter {
    @Autowired
    private TokenStore jwtTokenStore;
    @Autowired
    private JwtAccessTokenConverter jwtAccessTokenConverter;

    @Override
    public void configure(AuthorizationServerEndpointsConfigurer endpoints) {
        endpoints.authenticationManager(authenticationManager)
                .tokenStore(jwtTokenStore)
                .accessTokenConverter(jwtAccessTokenConverter);
    }
    ......
}
```

重启服务获取令牌，系统将返回如下格式令牌：

```
{
    "access_token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJleHAiOjE1NjE1MzI1MDEsInVzZXJfbmFtZSI6Im1yYmlyZCIsImF1dGhvcml0aWVzIjpbImFkbWluIl0sImp0aSI6IjJkZjY4MGNhLWFmN2QtNGU4Ni05OTdhLWI1ZmVkYzQxZmYwZSIsImNsaWVudF9pZCI6InRlc3QxIiwic2NvcGUiOltdfQ.dZ4SeuU3VWnSJKy5vELGQ0YkVRddcEydUlJAVovlycg",
    "token_type": "bearer",
    "expires_in": 3599,
    "scope": "all",
    "jti": "2df680ca-af7d-4e86-997a-b5fedc41ff0e"
}
```

将`access_token`中的内容复制到https://jwt.io/网站解析下：

![](picture/security-oauth2-config-jwt-io.png)

使用这个token访问`/index`可以成功获取到信息

#### **增加额外信息**

如果想在JWT中添加一些额外的信息，我们需要实现`TokenEnhancer`（Token增强器）：

```
public class JWTokenEnhancer implements TokenEnhancer {
    @Override
    public OAuth2AccessToken enhance(OAuth2AccessToken oAuth2AccessToken, OAuth2Authentication oAuth2Authentication) {
        Map<String, Object> info = new HashMap<>();
        info.put("message", "hello world");
        ((DefaultOAuth2AccessToken) oAuth2AccessToken).setAdditionalInformation(info);
        return oAuth2AccessToken;
    }
}
```

然后在`JWTokenConfig`里注册该Bean：

```
@Configuration
public class JWTokenConfig {
    ......

    @Bean
    public TokenEnhancer tokenEnhancer() {
        return new JWTokenEnhancer();
    }
}
```

最后在认证服务器里配置该增强器：

```
@Configuration
@EnableAuthorizationServer
public class AuthorizationServerConfig extends AuthorizationServerConfigurerAdapter {
    @Autowired
    private TokenStore jwtTokenStore;
    @Autowired
    private JwtAccessTokenConverter jwtAccessTokenConverter;
    @Autowired
    private TokenEnhancer tokenEnhancer;

    @Override
    public void configure(AuthorizationServerEndpointsConfigurer endpoints) {
        TokenEnhancerChain enhancerChain = new TokenEnhancerChain();
        List<TokenEnhancer> enhancers = new ArrayList<>();
        enhancers.add(tokenEnhancer);
        enhancers.add(jwtAccessTokenConverter);
        enhancerChain.setTokenEnhancers(enhancers);

        endpoints.tokenStore(jwtTokenStore)
                .accessTokenConverter(jwtAccessTokenConverter)
                .tokenEnhancer(enhancerChain);
    }
    ......
}
```

重启项目，再次获取令牌，然后去解析, 可以看到JSON内容里已经多了我们添加的message信息.

#### **jwt解析**

```
<dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt</artifactId>
    <version>0.9.1</version>
</dependency>
```

```
@GetMapping("index")
public Object index(@AuthenticationPrincipal Authentication authentication, HttpServletRequest request) {
    String header = request.getHeader("Authorization");
    String token = StringUtils.substringAfter(header, "bearer ");

    return Jwts.parser().setSigningKey("test_key".getBytes(StandardCharsets.UTF_8)).parseClaimsJws(token).getBody();
}
```

signkey需要和`JwtAccessTokenConverter`中指定的签名密钥一致。

#### 刷新令牌

令牌过期后我们可以使用refresh_token来从系统中换取一个新的可用令牌。但是从前面的例子可以看到，在认证成功后返回的JSON信息里并没有包含refresh_token，要让系统返回refresh_token，需要在认证服务器自定义配置里添加如下配置：

```
@Configuration
@EnableAuthorizationServer
public class AuthorizationServerConfig extends AuthorizationServerConfigurerAdapter {
	......
    @Override
    public void configure(ClientDetailsServiceConfigurer clients) throws Exception {
        clients.inMemory()
                .withClient("test1")
                .secret(new BCryptPasswordEncoder().encode("test1111"))
                .authorizedGrantTypes("password", "refresh_token")   //授权方式需要添加refresh_token
                .accessTokenValiditySeconds(3600)
                .refreshTokenValiditySeconds(864000)
                .scopes("all", "a", "b", "c")
            .and()
                .withClient("test2")
                .secret(new BCryptPasswordEncoder().encode("test2222"))
                .accessTokenValiditySeconds(7200);
    }
}
```

授权方式需要加上`refresh_token`，除了四种标准的OAuth2获取令牌方式外，Spring Security OAuth2内部把`refresh_token`当作一种拓展的获取令牌方式。

通过上面的配置，使用test1这个client_id获取令牌时将返回refresh_token，refresh_token的有效期为10天，即10天之内都可以用它换取新的可用令牌。

重启项目，认证成功后，系统返回如：

```
{
    "access_token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJleHAiOjE1NjE1NTgwOTcsInVzZXJfbmFtZSI6Im1yYmlyZCIsImF1dGhvcml0aWVzIjpbImFkbWluIl0sImp0aSI6Ijg2NTdhMDBlLTFiM2MtNDA5NS1iMjNmLTJlMjUxOWExZmUwMiIsImNsaWVudF9pZCI6InRlc3QxIiwic2NvcGUiOlsiYWxsIl19.hrxKOz3NKY6Eq8k5QeOqKhXUQ4aAbicrb6J5y-LBRA0",
    "token_type": "bearer",
    "refresh_token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VyX25hbWUiOiJtcmJpcmQiLCJzY29wZSI6WyJhbGwiXSwiYXRpIjoiODY1N2EwMGUtMWIzYy00MDk1LWIyM2YtMmUyNTE5YTFmZTAyIiwiZXhwIjoxNTYyNDE4NDk3LCJhdXRob3JpdGllcyI6WyJhZG1pbiJdLCJqdGkiOiI2MTNjMDVlNS1hNzUzLTRmM2UtOWViOC1hZGE4MTJmY2IyYWQiLCJjbGllbnRfaWQiOiJ0ZXN0MSJ9.efw9OePFUN9X6UGMF3h9BF_KO3zqyIfpvfmE8XklBDs",
    "expires_in": 3599,
    "scope": "all",
    "jti": "8657a00e-1b3c-4095-b23f-2e2519a1fe02"
}
```

假设现在access_token过期了，我们用refresh_token去换取新的令牌。使用postman发送如下请求：

![](picture/security-oauth2-config-jwt-reflush1.png)

![](picture/security-oauth2-config-jwt-reflush2.png)

## 单点登录SSO

SSO（Single Sign On）即单点登录，效果是多个系统间，只要登录了其中一个系统，别的系统不用登录操作也能访问。比如在浏览器上同时打开天猫和淘宝页面，在天猫页面进行登录，然后回到淘宝页面刷新后会发现淘宝也已经是登录状态了。这节将介绍如何使用Spring Security OAuth2实现单点登录。

**框架搭建**

需要创建一个maven多模块项目，包含认证服务器和两个客户端。

父模块:

```
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <packaging>pom</packaging>
    <modules>
        <module>sso-application-one</module>
        <module>sso-application-two</module>
        <module>sso-server</module>
    </modules>
    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>2.1.6.RELEASE</version>
        <relativePath/> <!-- lookup parent from repository -->
    </parent>
    <groupId>cc.mrbird</groupId>
    <artifactId>sso</artifactId>
    <version>0.0.1-SNAPSHOT</version>
    <name>sso</name>
    <description>Demo project for Spring Boot</description>

    <properties>
        <java.version>1.8</java.version>
        <spring-cloud.version>Greenwich.SR1</spring-cloud.version>
    </properties>

    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-oauth2</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-security</artifactId>
        </dependency>
        <dependency>
            <groupId>org.apache.commons</groupId>
            <artifactId>commons-lang3</artifactId>
        </dependency>
    </dependencies>

    <dependencyManagement>
        <dependencies>
            <dependency>
                <groupId>org.springframework.cloud</groupId>
                <artifactId>spring-cloud-dependencies</artifactId>
                <version>${spring-cloud.version}</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>
        </dependencies>
    </dependencyManagement>

    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>
        </plugins>
    </build>
</project>
```

sso-server(认证服务器)

```
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <parent>
        <artifactId>sso</artifactId>
        <groupId>cc.mrbird</groupId>
        <version>0.0.1-SNAPSHOT</version>
    </parent>
    <modelVersion>4.0.0</modelVersion>

    <artifactId>sso-server</artifactId>
</project>
```

sso-application-one和sso-application-two(客户端)

### 认证服务器配置

认证服务器作用就是作为统一令牌发放并校验的地方，所以我们先要编写一些基本的Spring Security 安全配置的代码指定如何进行用户认证。

新建一个Spring Security配置类，继承`WebSecurityConfigurerAdapter`：

```
@Configuration
public class WebSecurityConfig extends WebSecurityConfigurerAdapter {

    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();
    }
    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http.formLogin()
            .and()
                .authorizeRequests()
                .anyRequest()
                .authenticated();
    }
}
```

上面简单配置了密码加密使用bcrypt方式，并且所有请求都需要认证，认证方式为Spring Security自带的登录页面认证（也可以根据前面教程来自定义登录页面，这里为了简单起见，就直接用自带的登录页了）。

接着需要定义一个自定义用户登录认证的服务：

```
@Configuration
public class UserDetailService implements UserDetailsService {
    @Autowired
    private PasswordEncoder passwordEncoder;
    @Override
    public UserDetails loadUserByUsername(String username) throws UsernameNotFoundException {
        MyUser user = new MyUser();
        user.setUserName(username);
        user.setPassword(this.passwordEncoder.encode("123456"));

        return new User(username, user.getPassword(), user.isEnabled(),
                user.isAccountNonExpired(), user.isCredentialsNonExpired(),
                user.isAccountNonLocked(), AuthorityUtils.commaSeparatedStringToAuthorityList("user:add"));
    }
}
```

基本逻辑是用户名随便写，密码为123456，并且拥有`user:add`权限。

```
public class MyUser implements Serializable {
    private static final long serialVersionUID = 3497935890426858541L;
    private String userName;
    private String password;
    private boolean accountNonExpired = true;
    private boolean accountNonLocked= true;
    private boolean credentialsNonExpired= true;
    private boolean enabled= true;
    // get，set略
}
```

接着开始编写认证服务器配置。

新建SsoAuthorizationServerConfig，继承AuthorizationServerConfigurerAdapter：

```
@Configuration
@EnableAuthorizationServer
public class SsoAuthorizationServerConfig extends AuthorizationServerConfigurerAdapter {
    @Autowired
    private PasswordEncoder passwordEncoder;
    @Autowired
    private UserDetailService userDetailService;

    @Bean
    public TokenStore jwtTokenStore() {
        return new JwtTokenStore(jwtAccessTokenConverter());
    }

    @Bean
    public JwtAccessTokenConverter jwtAccessTokenConverter() {
        JwtAccessTokenConverter accessTokenConverter = new JwtAccessTokenConverter();
        accessTokenConverter.setSigningKey("test_key");
        return accessTokenConverter;
    }

    @Override
    public void configure(ClientDetailsServiceConfigurer clients) throws Exception {
        // 见下方
    }

    @Override
    public void configure(AuthorizationServerEndpointsConfigurer endpoints) {
        endpoints.tokenStore(jwtTokenStore())
                .accessTokenConverter(jwtAccessTokenConverter())
                .userDetailsService(userDetailService);
    }

    @Override
    public void configure(AuthorizationServerSecurityConfigurer security) {
        security.tokenKeyAccess("isAuthenticated()"); // 获取密钥需要身份认证
    }
}
```

Token使用JWT.

这里详细说下`configure(ClientDetailsServiceConfigurer clients)`的配置：

```
@Override
public void configure(ClientDetailsServiceConfigurer clients) throws Exception {
    clients.inMemory()
            .withClient("app-a")
            .secret(passwordEncoder.encode("app-a-1234"))
            .authorizedGrantTypes("refresh_token","authorization_code")
            .accessTokenValiditySeconds(3600)
            .scopes("all");
        .and()
            .withClient("app-b")
            .secret(passwordEncoder.encode("app-b-1234"))
            .authorizedGrantTypes("refresh_token","authorization_code")
            .accessTokenValiditySeconds(7200)
            .scopes("all");
}
```

这里分配了两个客户端配置，分别为app-a和app-b，因为使用默认的Spring Security登录页面来进行认证，所以需要开启`authorization_code`类型认证支持。

认证服务器的application.yml配置如下：

```
server:
  port: 8080
  servlet:
    context-path: /server
```

### 客户端配置

两个客户端的代码基本一致

在客户端SpringBoot入口类上添加`@EnableOAuth2Sso`注解，开启SSO的支持：

```
@EnableOAuth2Sso
@SpringBootApplication
public class SsoApplicaitonOne {
    public static void main(String[] args) {
        new SpringApplicationBuilder(SsoApplicaitonOne.class).run(args);
    }
}
```

接下来的重点是配置文件application.yml的配置：

```
security:
  oauth2:
    client:
      client-id: app-a
      client-secret: app-a-1234
      user-authorization-uri: http://127.0.0.1:8080/server/oauth/authorize
      access-token-uri: http://127.0.0.1:8080/server/oauth/token
    resource:
      jwt:
        key-uri: http://127.0.0.1:8080/server/oauth/token_key
server:
  port: 9090
  servlet:
    context-path: /app1
```

`security.oauth2.client.client-id`和`security.oauth2.client.client-secret`指定了客户端id和密码，这里和认证服务器里配置的client一致（另外一个客户端为app-b）；`user-authorization-uri`指定为认证服务器的`/oauth/authorize`地址，`access-token-uri`指定为认证服务器的`/oauth/token`地址，`jwt.key-uri`指定为认证服务器的`/oauth/token_key`地址。

这里端口指定为9090，`context-path`为app1，另一个客户端端口指定为9091，`context-path`为app2。

接着在resources/static下新增一个index.html页面，用于跳转到另外一个客户端：

```
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>管理系统一</title>
</head>
<body>
    <h1>管理系统一</h1>
    <a href="http://127.0.0.1:9091/app2/index.html">跳转到管理系统二</a>
</body>
</html>
```

为了验证是否认证成功，我们新增一个控制器：

```
@RestController
public class UserController {

    @GetMapping("user")
    public Principal user(Principal principal) {
        return principal;
    }
}
```

先启动认证服务器，然后在启动两个客户端。启动后，访问http://127.0.0.1:9090/app1/index.html,  页面将被重定向到认证服务器的登录页面，根据我们定义的UserDetailService，用户名随便填，密码为123456。登录后页面跳转到：

![](picture/security-oauth2-sso-redirect-error.png)

这个URL为:http://127.0.0.1:8080/server/oauth/authorize?client_id=app-a&redirect_uri=http://127.0.0.1:9090/app1/login&response_type=code&state=7Dqcbr

页面提示的意思是：非法请求，至少需要一个重定向URL被注册到client。从URL中可以看出，redirect_uri为http://127.0.0.1:9090/app1/login，所以我们修改认证服务器client相关配置如下：

```
@Override
public void configure(ClientDetailsServiceConfigurer clients) throws Exception {
    clients.inMemory()
            .withClient("app-a")
            .secret(passwordEncoder.encode("app-a-1234"))
            .authorizedGrantTypes("refresh_token","authorization_code")
            .accessTokenValiditySeconds(3600)
            .scopes("all")
            .redirectUris("http://127.0.0.1:9090/app1/login")
        .and()
            .withClient("app-b")
            .secret(passwordEncoder.encode("app-b-1234"))
            .authorizedGrantTypes("refresh_token","authorization_code")
            .accessTokenValiditySeconds(7200)
            .scopes("all")
            .redirectUris("http://127.0.0.1:9091/app2/login");
}
```

重启认证服务器，重复上面的过程，这次登录后，页面跳转到了授权页面,  点击Authorize ,到了app1的index,   其中包含了可以跳转到app2的index链接 ,  点击跳转,   结果为页面直接来到授权页，而不需要重新输入用户名密码，继续点击Authorize.

app2也已经成功登录，访问http://127.0.0.1:9091/app2/user看是否能成功获取到用户信息

### 自动授权

在上面过程中需要用户点击Authorize授权，体验并不是很好，我们可以去掉它。修改认证服务器Client配置如下：

```
@Override
public void configure(ClientDetailsServiceConfigurer clients) throws Exception {
    clients.inMemory()
            .withClient("app-a")
            .secret(passwordEncoder.encode("app-a-1234"))
            .authorizedGrantTypes("refresh_token","authorization_code")
            .accessTokenValiditySeconds(3600)
            .scopes("all")
            .autoApprove(true)    //自动授权
            .redirectUris("http://127.0.0.1:9090/app1/login")
        .and()
            .withClient("app-b")
            .secret(passwordEncoder.encode("app-b-1234"))
            .authorizedGrantTypes("refresh_token","authorization_code")
            .accessTokenValiditySeconds(7200)
            .scopes("all")
            .autoApprove(true)
            .redirectUris("http://127.0.0.1:9091/app2/login");
}
```

### 权限校验

在单点登录模式下进行权限校验

在客户端控制器里加入如下代码:

```
@GetMapping("auth/test1")
@PreAuthorize("hasAuthority('user:add')")
public String authTest1(){
    return "您拥有'user:add'权限";
}
@GetMapping("auth/test2")
@PreAuthorize("hasAuthority('user:update')")
public String authTest2(){
    return "您拥有'user:update'权限";
}
```

在客户端新增Spring Security配置类:

```
@Configuration
@EnableGlobalMethodSecurity(prePostEnabled = true)
public class WebSecurityConfigurer extends WebSecurityConfigurerAdapter {
}
```

完后，先启动认证服务器，在启动客户端。

在启动客户端的时候出现异常：

```
Caused by: java.lang.IllegalStateException: @Order on WebSecurityConfigurers must be unique. Order of 100 was already used on cc.mrbird.sso.client.config.WebSecurityConfigurer$$EnhancerBySpringCGLIB$$aa470b71@34d45ec0, so it cannot be used on org.springframework.boot.autoconfigure.security.oauth2.client.OAuth2SsoDefaultConfiguration$$EnhancerBySpringCGLIB$$6f69df92@18137eab too.
	at org.springframework.security.config.annotation.web.configuration.WebSecurityConfiguration.setFilterChainProxySecurityConfigurer(WebSecurityConfiguration.java:148) ~[spring-security-config-5.1.5.RELEASE.jar:5.1.5.RELEASE]
	at sun.reflect.NativeMethodAccessorImpl.invoke0(Native Method) ~[na:1.8.0_171]
	at sun.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:62) ~[na:1.8.0_171]
	at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43) ~[na:1.8.0_171]
	at java.lang.reflect.Method.invoke(Method.java:498) ~[na:1.8.0_171]
	at org.springframework.beans.factory.annotation.AutowiredAnnotationBeanPostProcessor$AutowiredMethodElement.inject(AutowiredAnnotationBeanPostProcessor.java:708) ~[spring-beans-5.1.8.RELEASE.jar:5.1.8.RELEASE]
	at org.springframework.beans.factory.annotation.InjectionMetadata.inject(InjectionMetadata.java:90) ~[spring-beans-5.1.8.RELEASE.jar:5.1.8.RELEASE]
	at org.springframework.beans.factory.annotation.AutowiredAnnotationBeanPostProcessor.postProcessProperties(AutowiredAnnotationBeanPostProcessor.java:374) ~[spring-beans-5.1.8.RELEASE.jar:5.1.8.RELEASE]
	... 16 common frames omitted
```

大致意思是，认证服务器已经配置了Spring Security配置，并且顺序为100，和客户端的Spring Security配置冲突了。所以我们修改下客户端的Spring Security配置顺序：

```
@Order(101)
@Configuration
@EnableGlobalMethodSecurity(prePostEnabled = true)
public class WebSecurityConfigurer extends WebSecurityConfigurerAdapter {
}
```

让它的优先级小于认证服务器的Spring Security配置。

重新启动客户端，进行单点登录操作，登录成功后，访问http://127.0.0.1:9090/app1/auth/test1.

网页显示 您拥有'user:add'权限

访问http://127.0.0.1:9090/app1/auth/test2   -->403











# shiro

在Spring Boot中集成Shiro进行用户的认证过程主要可以归纳为以下三点：

1、定义一个ShiroConfig，然后配置SecurityManager Bean，SecurityManager为Shiro的安全管理器，管理着所有Subject；

2、在ShiroConfig中配置ShiroFilterFactoryBean，其为Shiro过滤器工厂类，依赖于SecurityManager；

3、自定义Realm实现，Realm包含`doGetAuthorizationInfo()`和`doGetAuthenticationInfo()`方法.

```
 <dependency>
            <groupId>org.apache.shiro</groupId>
            <artifactId>shiro-spring-boot-web-starter</artifactId>
            <version>${shiro.spring.version}</version>
 </dependency>
  
```

```
<dependency>
    <groupId>org.apache.shiro</groupId>
    <artifactId>shiro-spring</artifactId>
    <version>1.4.0</version>
</dependency>
```

需要注意的是filterChain基于短路机制，即最先匹配原则，如：

```
/user/**=anon
/user/aa=authc 永远不会执行
```

Shiro为我们实现的过滤器:

| Filter Name       | Class                                                        | Description                                                  |
| :---------------- | :----------------------------------------------------------- | :----------------------------------------------------------- |
| anon              | [org.apache.shiro.web.filter.authc.AnonymousFilter](http://shiro.apache.org/static/current/apidocs/org/apache/shiro/web/filter/authc/AnonymousFilter.html) | 匿名拦截器，即不需要登录即可访问；一般用于静态资源过滤；示例`/static/**=anon` |
| authc             | [org.apache.shiro.web.filter.authc.FormAuthenticationFilter](http://shiro.apache.org/static/current/apidocs/org/apache/shiro/web/filter/authc/FormAuthenticationFilter.html) | 基于表单的拦截器；如`/**=authc`，如果没有登录会跳到相应的登录页面登录 |
| authcBasic        | [org.apache.shiro.web.filter.authc.BasicHttpAuthenticationFilter](http://shiro.apache.org/static/current/apidocs/org/apache/shiro/web/filter/authc/BasicHttpAuthenticationFilter.html) | Basic HTTP身份验证拦截器                                     |
| logout            | [org.apache.shiro.web.filter.authc.LogoutFilter](http://shiro.apache.org/static/current/apidocs/org/apache/shiro/web/filter/authc/LogoutFilter.html) | 退出拦截器，主要属性：redirectUrl：退出成功后重定向的地址（/），示例`/logout=logout` |
| noSessionCreation | [org.apache.shiro.web.filter.session.NoSessionCreationFilter](http://shiro.apache.org/static/current/apidocs/org/apache/shiro/web/filter/session/NoSessionCreationFilter.html) | 不创建会话拦截器，调用`subject.getSession(false)`不会有什么问题，但是如果`subject.getSession(true)`将抛出`DisabledSessionException`异常 |
| perms             | [org.apache.shiro.web.filter.authz.PermissionsAuthorizationFilter](http://shiro.apache.org/static/current/apidocs/org/apache/shiro/web/filter/authz/PermissionsAuthorizationFilter.html) | 权限授权拦截器，验证用户是否拥有所有权限；属性和roles一样；示例`/user/**=perms["user:create"]` |
| port              | [org.apache.shiro.web.filter.authz.PortFilter](http://shiro.apache.org/static/current/apidocs/org/apache/shiro/web/filter/authz/PortFilter.html) | 端口拦截器，主要属性`port(80)`：可以通过的端口；示例`/test= port[80]`，如果用户访问该页面是非80，将自动将请求端口改为80并重定向到该80端口，其他路径/参数等都一样 |
| rest              | [org.apache.shiro.web.filter.authz.HttpMethodPermissionFilter](http://shiro.apache.org/static/current/apidocs/org/apache/shiro/web/filter/authz/HttpMethodPermissionFilter.html) | rest风格拦截器，自动根据请求方法构建权限字符串；示例`/users=rest[user]`，会自动拼出user:read,user:create,user:update,user:delete权限字符串进行权限匹配（所有都得匹配，isPermittedAll） |
| roles             | [org.apache.shiro.web.filter.authz.RolesAuthorizationFilter](http://shiro.apache.org/static/current/apidocs/org/apache/shiro/web/filter/authz/RolesAuthorizationFilter.html) | 角色授权拦截器，验证用户是否拥有所有角色；示例`/admin/**=roles[admin]` |
| ssl               | [org.apache.shiro.web.filter.authz.SslFilter](http://shiro.apache.org/static/current/apidocs/org/apache/shiro/web/filter/authz/SslFilter.html) | SSL拦截器，只有请求协议是https才能通过；否则自动跳转会https端口443；其他和port拦截器一样； |
| user              | [org.apache.shiro.web.filter.authc.UserFilter](http://shiro.apache.org/static/current/apidocs/org/apache/shiro/web/filter/authc/UserFilter.html) | 用户拦截器，用户已经身份验证/记住我登录的都可；示例`/**=user` |

注解

```
@RequiresAuthentication：表示当前Subject已经通过login进行了身份验证；即 Subject. isAuthenticated() 返回 true
@RequiresUser：表示当前 Subject 已经身份验证或者通过记住我登录的。
@RequiresGuest：表示当前Subject没有身份验证或通过记住我登录过，即是游客身份。
@RequiresRoles(value={“admin”, “user”}, logical=Logical.AND)：表示当前 Subject 需要角色 admin 和user
@RequiresPermissions (value={“user:a”, “user:b”},logical= Logical.OR)：表示当前 Subject 需要权限 user:a 或user:b。

注意:使用注解需要在配置类中加入
@Bean
public AuthorizationAttributeSourceAdvisor authorizationAttributeSourceAdvisor(SecurityManager securityManager) {
    AuthorizationAttributeSourceAdvisor authorizationAttributeSourceAdvisor = new AuthorizationAttributeSourceAdvisor();
    authorizationAttributeSourceAdvisor.setSecurityManager(securityManager);
    return authorizationAttributeSourceAdvisor;
}
```

## 认证

```
@Configuration
public class ShiroConfig {
   @Bean
   public ShiroFilterFactoryBean shiroFilterFactoryBean(SecurityManager securityManager) {
      ShiroFilterFactoryBean shiroFilterFactoryBean = new ShiroFilterFactoryBean();
      shiroFilterFactoryBean.setSecurityManager(securityManager);
      shiroFilterFactoryBean.setLoginUrl("/login");
      shiroFilterFactoryBean.setSuccessUrl("/index");
      shiroFilterFactoryBean.setUnauthorizedUrl("/403");
      
      LinkedHashMap<String, String> filterChainDefinitionMap = new LinkedHashMap<>();
      //静态资源不拦截
      filterChainDefinitionMap.put("/css/**", "anon");
      filterChainDefinitionMap.put("/js/**", "anon");
      filterChainDefinitionMap.put("/fonts/**", "anon");
      filterChainDefinitionMap.put("/img/**", "anon");
      // druid数据源监控页面不拦截
      filterChainDefinitionMap.put("/druid/**", "anon");
      // 配置退出过滤器，其中具体的退出代码Shiro已经替我们实现了 
      filterChainDefinitionMap.put("/logout", "logout");
      filterChainDefinitionMap.put("/", "anon");
      // 除上以外所有url都必须认证通过才可以访问，未通过认证自动访问LoginUrl
      filterChainDefinitionMap.put("/**", "authc");
      
      shiroFilterFactoryBean.setFilterChainDefinitionMap(filterChainDefinitionMap);
      
      return shiroFilterFactoryBean;
   }
 
   @Bean  
    public SecurityManager securityManager(){  
       DefaultWebSecurityManager securityManager =  new DefaultWebSecurityManager();
       securityManager.setRealm(shiroRealm());
       return securityManager;  
    }  
   
   @Bean  
    public ShiroRealm shiroRealm(){  
       ShiroRealm shiroRealm = new ShiroRealm();  
       return shiroRealm;  
    }    
}
```

```
public class ShiroRealm extends AuthorizingRealm {
	//标准的做法应该是Service层
   @Autowired
   private UserMapper userMapper;

   @Override
   protected AuthorizationInfo doGetAuthorizationInfo(PrincipalCollection principal) {
      return null;
   }

   @Override
   protected AuthenticationInfo doGetAuthenticationInfo(AuthenticationToken token) throws AuthenticationException {
      String userName = (String) token.getPrincipal();
      String password = new String((char[]) token.getCredentials());

      System.out.println("用户" + userName + "认证-----ShiroRealm.doGetAuthenticationInfo");
      User user = userMapper.findByUserName(userName);

      if (user == null) {
         throw new UnknownAccountException("用户名或密码错误！");
      }
      if (!password.equals(user.getPassword())) {
         throw new IncorrectCredentialsException("用户名或密码错误！");
      }
      if (user.getStatus().equals("0")) {
         throw new LockedAccountException("账号已被锁定,请联系管理员！");
      }
      SimpleAuthenticationInfo info = new SimpleAuthenticationInfo(user, password, getName());
      return info;
   }
}
```

```
@Controller
public class LoginController {

	@GetMapping("/login")
	public String login() {
		return "login";
	}

	@PostMapping("/login")
	@ResponseBody
	public ResponseBo login(String username, String password) {
		password = MD5Utils.encrypt(username, password);
		UsernamePasswordToken token = new UsernamePasswordToken(username, password);
		Subject subject = SecurityUtils.getSubject();
		try {
			subject.login(token);
			return ResponseBo.ok();
		} catch (UnknownAccountException e) {
			return ResponseBo.error(e.getMessage());
		} catch (IncorrectCredentialsException e) {
			return ResponseBo.error(e.getMessage());
		} catch (LockedAccountException e) {
			return ResponseBo.error(e.getMessage());
		} catch (AuthenticationException e) {
			return ResponseBo.error("认证失败！");
		}
	}

	@RequestMapping("/")
	public String redirectIndex() {
		return "redirect:/index";
	}

	@RequestMapping("/index")
	public String index(Model model) {
		User user = (User) SecurityUtils.getSubject().getPrincipal();
		model.addAttribute("user", user);
		return "index";
	}
}
```



## RemenberMe

Shiro为我们提供了Remember Me的功能，用户的登录状态不会因为浏览器的关闭而失效，直到Cookie过期。

在上面的基础上修改ShiroFilterFactoryBean,SecurityManager,加入 SimpleCookie ,CookieRememberMeManager

```
@Configuration
public class ShiroConfig {
   @Bean
   public ShiroFilterFactoryBean shiroFilterFactoryBean(SecurityManager securityManager) {
      ShiroFilterFactoryBean shiroFilterFactoryBean = new ShiroFilterFactoryBean();
      shiroFilterFactoryBean.setSecurityManager(securityManager);
      shiroFilterFactoryBean.setLoginUrl("/login");
      shiroFilterFactoryBean.setSuccessUrl("/index");
      shiroFilterFactoryBean.setUnauthorizedUrl("/403");
      
      LinkedHashMap<String, String> filterChainDefinitionMap = new LinkedHashMap<>();
      
      filterChainDefinitionMap.put("/css/**", "anon");
      filterChainDefinitionMap.put("/js/**", "anon");
      filterChainDefinitionMap.put("/fonts/**", "anon");
      filterChainDefinitionMap.put("/img/**", "anon");
      filterChainDefinitionMap.put("/druid/**", "anon");
      filterChainDefinitionMap.put("/logout", "logout");
      filterChainDefinitionMap.put("/", "anon");
      //修改权限配置，将ShiroFilterFactoryBean的filterChainDefinitionMap.put("/**", "authc");更改为filterChainDefinitionMap.put("/**", "user");。user指的是用户认证通过或者配置了Remember Me记住用户登录状态后可访问。
      filterChainDefinitionMap.put("/**", "user");
      
      shiroFilterFactoryBean.setFilterChainDefinitionMap(filterChainDefinitionMap);
      return shiroFilterFactoryBean;
   }
 
   @Bean  
    public SecurityManager securityManager(){  
       DefaultWebSecurityManager securityManager =  new DefaultWebSecurityManager();
       securityManager.setRealm(shiroRealm());
       securityManager.setRememberMeManager(rememberMeManager());
       return securityManager;  
    }  
    /**
     * 配置Shiro生命周期处理器
     */
   @Bean(name = "lifecycleBeanPostProcessor")
    public LifecycleBeanPostProcessor lifecycleBeanPostProcessor() {
        return new LifecycleBeanPostProcessor();
    }
   
   @Bean  
    public ShiroRealm shiroRealm(){  
       ShiroRealm shiroRealm = new ShiroRealm();  
       return shiroRealm;  
    }  
   
   /**
    * cookie对象
    * @return
    */
   public SimpleCookie rememberMeCookie() {
      // 设置cookie名称，对应login.html页面的<input type="checkbox" name="rememberMe"/>
      SimpleCookie cookie = new SimpleCookie("rememberMe");
      // 设置cookie的过期时间，单位为秒，这里为一天
      cookie.setMaxAge(86400);
      return cookie;
   }
   
   /**
    * cookie管理对象
    * @return
    */
   public CookieRememberMeManager rememberMeManager() {
      CookieRememberMeManager cookieRememberMeManager = new CookieRememberMeManager();
      cookieRememberMeManager.setCookie(rememberMeCookie());
      // rememberMe cookie加密的密钥 
     cookieRememberMeManager.setCipherKey(Base64.decode("3AvVhmFLUs0KTA3Kprsdag=="));
      return cookieRememberMeManager;
   }
}
```

```
@Controller
public class LoginController {

   @GetMapping("/login")
   public String login() {
      return "login";
   }

   @PostMapping("/login")
   @ResponseBody
   public ResponseBo login(String username, String password, Boolean rememberMe) {
      password = MD5Utils.encrypt(username, password);
      UsernamePasswordToken token = new UsernamePasswordToken(username, password, rememberMe);
      Subject subject = SecurityUtils.getSubject();
      try {
         subject.login(token);
         return ResponseBo.ok();
      } catch (UnknownAccountException e) {
         return ResponseBo.error(e.getMessage());
      } catch (IncorrectCredentialsException e) {
         return ResponseBo.error(e.getMessage());
      } catch (LockedAccountException e) {
         return ResponseBo.error(e.getMessage());
      } catch (AuthenticationException e) {
         return ResponseBo.error("认证失败！");
      }
   }

   @RequestMapping("/")
   public String redirectIndex() {
      return "redirect:/index";
   }

   @RequestMapping("/index")
   public String index(Model model) {
      User user = (User) SecurityUtils.getSubject().getPrincipal();
      model.addAttribute("user", user);
      return "index";
   }
}
```

## JdbcRealm

```
public class MyCustomRealm extends JdbcRealm {
//模拟数据库数据
    private Map<String, String> credentials = new HashMap<>();
    private Map<String, Set<String>> roles = new HashMap<>();
    private Map<String, Set<String>> perm = new HashMap<>();
    {
      credentials.put("user", "password");
      credentials.put("user2", "password2");
      credentials.put("user3", "password3");
      roles.put("user", new HashSet<>(Arrays.asList("admin")));
      roles.put("user2", new HashSet<>(Arrays.asList("editor")));
      roles.put("user3", new HashSet<>(Arrays.asList("author")));
      perm.put("admin", new HashSet<>(Arrays.asList("*")));
      perm.put("editor", new HashSet<>(Arrays.asList("articles:*")));
      perm.put("author",
        new HashSet<>(Arrays.asList("articles:compose",
          "articles:save")));
    }

    @Override
    protected AuthenticationInfo doGetAuthenticationInfo(AuthenticationToken token)
      throws AuthenticationException {
        UsernamePasswordToken uToken = (UsernamePasswordToken) token
        if(uToken.getUsername() == null
          || uToken.getUsername().isEmpty()
          || !credentials.containsKey(uToken.getUsername())
          ) {
            throw new UnknownAccountException("username not found!");
        }
        return new SimpleAuthenticationInfo(
          uToken.getUsername(), credentials.get(uToken.getUsername()),
          getName());
    }

    @Override
    protected AuthorizationInfo doGetAuthorizationInfo(PrincipalCollection principals) {
        Set<String> roleNames = new HashSet<>();
        Set<String> permissions = new HashSet<>();
        principals.forEach(p -> {
            try {
                Set<String> roles = getRoleNamesForUser(null, (String) p);
                roleNames.addAll(roles);
                permissions.addAll(getPermissions(null, null,roles));
            } catch (SQLException e) {
                e.printStackTrace();
            }
        });
        SimpleAuthorizationInfo info = new SimpleAuthorizationInfo(roleNames);
        info.setStringPermissions(permissions);
        return info;
    }

    @Override
    protected Set<String> getRoleNamesForUser(Connection conn, String username) throws SQLException {
        if(!roles.containsKey(username)) {
            throw new SQLException("username not found!");
        }

       return roles.get(username);
    }

    @Override
    protected Set<String> getPermissions(Connection conn, String username, Collection<String> roleNames) throws SQLException {
        for (String role : roleNames) {
            if (!perm.containsKey(role)) {
                throw new SQLException("role not found!");
            }
        }
        Set<String> finalSet = new HashSet<>();
        for (String role : roleNames) {
            finalSet.addAll(perm.get(role));
        }
        return finalSet;
    }
}
```

```
@Controller
public class ShiroSpringController {
    @GetMapping("/")
    public String index() {
       return "index";
    }

    @RequestMapping( value = "/login", method = {RequestMethod.GET, RequestMethod.POST})
    public String login(HttpServletRequest req, UserCredentials cred, RedirectAttributes attr) {
      if(req.getMethod().equals(RequestMethod.GET.toString())) {
        return "login";
      } else {
        Subject subject = SecurityUtils.getSubject();
        if(!subject.isAuthenticated()) {
          UsernamePasswordToken token = new UsernamePasswordToken(
            cred.getUsername(), cred.getPassword(), cred.isRememberMe());
          try {
            subject.login(token);
          } catch (AuthenticationException ae) {
              ae.printStackTrace();
              attr.addFlashAttribute("error", "Invalid Credentials");
              return "redirect:/login";
          }
        }
        return "redirect:/secure";
      }
    }

    @GetMapping("/secure")
    public String secure(ModelMap modelMap) {
        Subject currentUser = SecurityUtils.getSubject();
        String role = "", permission = "";
        if(currentUser.hasRole("admin")) {
            role = role  + "You are an Admin";
        }
        else if(currentUser.hasRole("editor")) {
            role = role + "You are an Editor";
        }
        else if(currentUser.hasRole("author")) {
            role = role + "You are an Author";
        }
        if(currentUser.isPermitted("articles:compose")) {
            permission = permission + "You can compose an article, ";
        } else {
            permission = permission + "You are not permitted to compose an article!, ";
        }
        if(currentUser.isPermitted("articles:save")) {
            permission = permission + "You can save articles, ";
        } else {
            permission = permission + "\nYou can not save articles, ";
        }
        if(currentUser.isPermitted("articles:publish")) {
            permission = permission  + "\nYou can publish articles";
        } else {
            permission = permission + "\nYou can not publish articles";
        }
        modelMap.addAttribute("username", currentUser.getPrincipal());
        modelMap.addAttribute("permission", permission);
        modelMap.addAttribute("role", role);
        return "secure";
    }

    @PostMapping("/logout")
    public String logout() {
       Subject subject = SecurityUtils.getSubject();
       subject.logout();
       return "redirect:/";
    }
}
```

```
@SpringBootApplication(exclude = SecurityAutoConfiguration.class)
public class ShiroSpringApplication {

    private static final transient Logger log = LoggerFactory.getLogger(ShiroSpringApplication.class);

    public static void main(String... args) {
        SpringApplication.run(ShiroSpringApplication.class, args);
    }

    @Bean
    public Realm realm() {
        return new MyCustomRealm();
    }

    @Bean
    public ShiroFilterChainDefinition filterChainDefinition() {
        DefaultShiroFilterChainDefinition filter
          = new DefaultShiroFilterChainDefinition();

        filter.addPathDefinition("/secure", "authc");
        filter.addPathDefinition("/**", "anon");

        return filter;
    }
}
```

## 授权+注解

RBAC（Role-Based Access Control，基于角色的访问控制）模型设计用户，角色和权限间的关系。简单地说，一个用户拥有若干角色，每一个角色拥有若干权限。这样，就构造成“用户-角色-权限”的授权模型。在这种模型中，用户与角色之间，角色与权限之间，一般者是多对多的关系。5张表,两张中间表

```
@Configuration
public class ShiroConfig {
   
   @Bean
   public ShiroFilterFactoryBean shiroFilterFactoryBean(SecurityManager securityManager) {
      ShiroFilterFactoryBean shiroFilterFactoryBean = new ShiroFilterFactoryBean();
      shiroFilterFactoryBean.setSecurityManager(securityManager);
      shiroFilterFactoryBean.setLoginUrl("/login");
      shiroFilterFactoryBean.setSuccessUrl("/index");
      shiroFilterFactoryBean.setUnauthorizedUrl("/403");
      LinkedHashMap<String, String> filterChainDefinitionMap = new LinkedHashMap<>();
      
      filterChainDefinitionMap.put("/css/**", "anon");
      filterChainDefinitionMap.put("/js/**", "anon");
      filterChainDefinitionMap.put("/fonts/**", "anon");
      filterChainDefinitionMap.put("/img/**", "anon");
      filterChainDefinitionMap.put("/druid/**", "anon");
      filterChainDefinitionMap.put("/logout", "logout");
      filterChainDefinitionMap.put("/", "anon");
      filterChainDefinitionMap.put("/**", "user");
      
      shiroFilterFactoryBean.setFilterChainDefinitionMap(filterChainDefinitionMap);
      
      return shiroFilterFactoryBean;
   }
 
   @Bean  
    public SecurityManager securityManager(){  
       DefaultWebSecurityManager securityManager =  new DefaultWebSecurityManager();
       securityManager.setRealm(shiroRealm());
       securityManager.setRememberMeManager(rememberMeManager());
       return securityManager;  
    }  
   
   @Bean  
    public ShiroRealm shiroRealm(){  
       ShiroRealm shiroRealm = new ShiroRealm();  
       return shiroRealm;  
    }  
   
   public SimpleCookie rememberMeCookie() {
      SimpleCookie cookie = new SimpleCookie("rememberMe");
      cookie.setMaxAge(86400);
      return cookie;
   }
   
   public CookieRememberMeManager rememberMeManager() {
      CookieRememberMeManager cookieRememberMeManager = new CookieRememberMeManager();
      cookieRememberMeManager.setCookie(rememberMeCookie());
      cookieRememberMeManager.setCipherKey(Base64.decode("4AvVhmFLUs0KTA3Kprsdag=="));
      return cookieRememberMeManager;
   }
	/**
     * 开启shiro 注解模式
     * 可以在controller中的方法前加上注解
     * 如 @RequiresPermissions("userInfo:add")
     */
    @Bean
    public AuthorizationAttributeSourceAdvisor authorizationAttributeSourceAdvisor(SecurityManager securityManager) {
        AuthorizationAttributeSourceAdvisor authorizationAttributeSourceAdvisor = new AuthorizationAttributeSourceAdvisor();
        authorizationAttributeSourceAdvisor.setSecurityManager(securityManager);
        return authorizationAttributeSourceAdvisor;
    }
}
```

```
public class ShiroRealm extends AuthorizingRealm {

	@Autowired
	private UserMapper userMapper;
	@Autowired
	private UserRoleMapper userRoleMapper;
	@Autowired
	private UserPermissionMapper userPermissionMapper;

	/**
	 * 获取用户角色和权限,用户每请求一次权限接口,都会进行一次数据库访问
	 */
	@Override
	protected AuthorizationInfo doGetAuthorizationInfo(PrincipalCollection principal) {
		User user = (User) SecurityUtils.getSubject().getPrincipal();
		String userName = user.getUserName();

		System.out.println("用户" + userName + "获取权限-----ShiroRealm.doGetAuthorizationInfo");
		SimpleAuthorizationInfo simpleAuthorizationInfo = new SimpleAuthorizationInfo();

		// 获取用户角色集
		List<Role> roleList = userRoleMapper.findByUserName(userName);
		Set<String> roleSet = new HashSet<String>();
		for (Role r : roleList) {
			roleSet.add(r.getName());
		}
		simpleAuthorizationInfo.setRoles(roleSet);

		// 获取用户权限集
		List<Permission> permissionList = userPermissionMapper.findByUserName(userName);
		Set<String> permissionSet = new HashSet<String>();
		for (Permission p : permissionList) {
			permissionSet.add(p.getName());
		}
		simpleAuthorizationInfo.setStringPermissions(permissionSet);
		return simpleAuthorizationInfo;
	}

	/**
	 * 登录认证
	 */
	@Override
	protected AuthenticationInfo doGetAuthenticationInfo(AuthenticationToken token) throws AuthenticationException {
		String userName = (String) token.getPrincipal();
		String password = new String((char[]) token.getCredentials());

		System.out.println("用户" + userName + "认证-----ShiroRealm.doGetAuthenticationInfo");
		User user = userMapper.findByUserName(userName);

		if (user == null) {
			throw new UnknownAccountException("用户名或密码错误！");
		}
		if (!password.equals(user.getPassword())) {
			throw new IncorrectCredentialsException("用户名或密码错误！");
		}
		if (user.getStatus().equals("0")) {
			throw new LockedAccountException("账号已被锁定,请联系管理员！");
		}
		SimpleAuthenticationInfo info = new SimpleAuthenticationInfo(user, password, getName());
		return info;
	}
}
```

```
@Controller
public class LoginController {

   @GetMapping("/login")
   public String login() {
      return "login";
   }

   @PostMapping("/login")
   @ResponseBody
   public ResponseBo login(String username, String password, Boolean rememberMe) {
      password = MD5Utils.encrypt(username, password);
      UsernamePasswordToken token = new UsernamePasswordToken(username, password, rememberMe);
      Subject subject = SecurityUtils.getSubject();
      try {
         subject.login(token);
         return ResponseBo.ok();
      } catch (UnknownAccountException e) {
         return ResponseBo.error(e.getMessage());
      } catch (IncorrectCredentialsException e) {
         return ResponseBo.error(e.getMessage());
      } catch (LockedAccountException e) {
         return ResponseBo.error(e.getMessage());
      } catch (AuthenticationException e) {
         return ResponseBo.error("认证失败！");
      }
   }

   @RequestMapping("/")
   public String redirectIndex() {
      return "redirect:/index";
   }
   
   @GetMapping("/403")
   public String forbid() {
      return "403";
   }

   @RequestMapping("/index")
   public String index(Model model) {
      User user = (User) SecurityUtils.getSubject().getPrincipal();
      model.addAttribute("user", user);
      return "index";
   }
}
```

权限需求controller

```
@Controller
@RequestMapping("/user")
public class UserController {
   
   @RequiresPermissions("user:user")
   @RequestMapping("list")
   public String userList(Model model) {
      model.addAttribute("value", "获取用户信息");
      return "user";
   }
   
   @RequiresPermissions("user:add")
   @RequestMapping("add")
   public String userAdd(Model model) {
      model.addAttribute("value", "新增用户");
      return "user";
   }
   
   @RequiresPermissions("user:delete")
   @RequestMapping("delete")
   public String userDelete(Model model) {
      model.addAttribute("value", "删除用户");
      return "user";
   }
}
```

还可以添加403全局异常捕获

```
@ControllerAdvice
@Order(value = Ordered.HIGHEST_PRECEDENCE)
public class GlobalExceptionHandler {
   
   @ExceptionHandler(value = AuthorizationException.class)
   public String handleAuthorizationException() {
      return "403";
   }
}
```

## cacheManager:Shiro-Redis

在Shiro中加入缓存可以使权限相关操作尽可能快，避免频繁访问数据库获取权限信息，因为对于一个用户来说，其权限在短时间内基本是不会变化的。Shiro提供了Cache的抽象，其并没有直接提供相应的实现，因为这已经超出了一个安全框架的范围。在Shiro中可以集成常用的缓存实现。

开源项目,实现了redis作为缓存 缓存用户的权限 和 session信息.https://github.com/alexxiyang/shiro-redis/

该框架给我们实现了RedisManager,RedisCache(实现了Cache接口)以及RedisCacheManager(实现了CacheManager接口)

拓展:我们自己也可以按照此方法实现(novel-plus:admin),https://blog.csdn.net/qq_34021712/article/details/80791219,

```
<!-- shiro-spring -->
<dependency>
    <groupId>org.apache.shiro</groupId>
    <artifactId>shiro-spring</artifactId>
    <version>1.4.0</version>
</dependency>

<!-- shiro-redis -->
<dependency>
    <groupId>org.crazycake</groupId>
    <artifactId>shiro-redis</artifactId>
    <version>2.4.2.1-RELEASE</version>
</dependency>
```

配置文件配置redis

在Authorization例子的基础上在配置类中添加

```
	//@Bean
    //@DependsOn({"lifecycleBeanPostProcessor"})
    //public DefaultAdvisorAutoProxyCreator advisorAutoProxyCreator() {
    //    DefaultAdvisorAutoProxyCreator advisorAutoProxyCreator = new DefaultAdvisorAutoProxyCreator();
    //    advisorAutoProxyCreator.setProxyTargetClass(true);
    //    return advisorAutoProxyCreator;
    //}
	public RedisManager redisManager() {
		//可以另指定ip端口用户名密码信息
		RedisManager redisManager = new RedisManager();
		return redisManager;
	}

	public RedisCacheManager cacheManager() {
		RedisCacheManager redisCacheManager = new RedisCacheManager();
		redisCacheManager.setRedisManager(redisManager());
		return redisCacheManager;
	}
```

并在securityManager中指定缓存

```
@Bean  
    public SecurityManager securityManager(){  
       DefaultWebSecurityManager securityManager =  new DefaultWebSecurityManager();
       securityManager.setRealm(shiroRealm());
       securityManager.setRememberMeManager(rememberMeManager());
       securityManager.setCacheManager(cacheManager());
       return securityManager;  
    }  
```

使用cache后用户访问多次权限接口只会进行一次数据库查询(时间内)

## cacheManager:Shiro+Ehcache

```
<!-- shiro ehcache -->
<dependency>
    <groupId>org.apache.shiro</groupId>
    <artifactId>shiro-ehcache</artifactId>
    <version>1.3.2</version>
</dependency>
<!-- ehchache -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-cache</artifactId>
</dependency>
<dependency>
    <groupId>net.sf.ehcache</groupId>
    <artifactId>ehcache</artifactId>
</dependency>
```

shiro-ehcache.xml

也可以通过配置文件指定xml

```
 spring:
   cache:
	 ehcache:
       config: classpath:ehcache.xml
```

```
<?xml version="1.0" encoding="UTF-8"?>
<ehcache xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
   xsi:noNamespaceSchemaLocation="http://ehcache.org/ehcache.xsd"
   updateCheck="false">
   <diskStore path="java.io.tmpdir/Tmp_EhCache" />
   <defaultCache
        maxElementsInMemory="10000"
        eternal="false"
        timeToIdleSeconds="120"
        timeToLiveSeconds="120"
        overflowToDisk="false"
        diskPersistent="false"
        diskExpiryThreadIntervalSeconds="120" />
        
   <!-- 登录记录缓存锁定1小时 -->
    <cache name="passwordRetryCache"
        maxEntriesLocalHeap="2000"
        eternal="false"
        timeToIdleSeconds="3600"
        timeToLiveSeconds="0"
        overflowToDisk="false"
        statistics="true" />
</ehcache>
```

在Authrization例修改如下

```
@Bean
public EhCacheManager getEhCacheManager() {
   EhCacheManager em = new EhCacheManager();
   em.setCacheManagerConfigFile("classpath:config/shiro-ehcache.xml");
   return em;
}
@Bean  
public SecurityManager securityManager(){  
       DefaultWebSecurityManager securityManager =  new DefaultWebSecurityManager();
       securityManager.setRealm(shiroRealm());
       securityManager.setRememberMeManager(rememberMeManager());
       securityManager.setCacheManager(getEhCacheManager());
       return securityManager;  
}  
//@Bean(name = "lifecycleBeanPostProcessor")
//    public LifecycleBeanPostProcessor lifecycleBeanPostProcessor() {
//        return new LifecycleBeanPostProcessor();
//}
```

## Thymeleaf集成shiro

实际上Thymeleaf官方并没有提供Shiro的标签，我们需要引入第三方实现，地址为https://github.com/theborakompanioni/thymeleaf-extras-shiro。

标签官方http://shiro.apache.org/web.html#Web-JSP%252FGSPTagLibrary

```
<dependency>
		    <groupId>com.github.theborakompanioni</groupId>
		    <artifactId>thymeleaf-extras-shiro</artifactId>
		    <version>2.0.0</version>
</dependency>
```

```
<!DOCTYPE html>
<!-- 使用Shiro标签需要给html标签添加xmlns:shiro="http://www.pollix.at/thymeleaf/shiro"-->
<html 
	xmlns:th="http://www.thymeleaf.org" 
	xmlns:shiro="http://www.pollix.at/thymeleaf/shiro" >
<head>
	<meta charset="UTF-8">
	<title>首页</title>
</head>
<style>
	div {
		border: 1px dashed #ddd;
		padding: 10px;
		margin: 10px 10px 10px 0px;
	}
</style>
<body>
	<p>你好！[[${user.userName}]]</p>
	<p shiro:hasRole="admin">你的角色为超级管理员</p>
	<p shiro:hasRole="test">你的角色为测试账户</p>
	<div>
		<a shiro:hasPermission="user:user" th:href="@{/user/list}">获取用户信息</a>
		<a shiro:hasPermission="user:add" th:href="@{/user/add}">新增用户</a>
		<a shiro:hasPermission="user:delete" th:href="@{/user/delete}">删除用户</a>
	</div>
	<a th:href="@{/logout}">注销</a>
</body>
</html>
```

还需要在配置类中配置该方言标签

```
@Bean
public ShiroDialect shiroDialect() {
    return new ShiroDialect();
}
```

## shiro+session实现在线人数功能

```
public class UserOnline implements Serializable{
	private static final long serialVersionUID = 3828664348416633856L;
	// session id
    private String id;
    // 用户id
    private String userId;
    // 用户名称
    private String username;
	// 用户主机地址
    private String host;
    // 用户登录时系统IP
    private String systemHost;
    // 状态
    private String status;
    // session创建时间
    private Date startTimestamp;
    // session最后访问时间
    private Date lastAccessTime;
    // 超时时间
    private Long timeout;
    //gettor settor ...
}
```

实现shiro提供的SessionListener

```
public class ShiroSessionListener implements SessionListener{
   private final AtomicInteger sessionCount = new AtomicInteger(0);
   @Override
   public void onStart(Session session) {
      sessionCount.incrementAndGet();
   }
   @Override
   public void onStop(Session session) {
      sessionCount.decrementAndGet();
      
   }
   @Override
   public void onExpiration(Session session) {
      sessionCount.decrementAndGet();
   }
   public int getSessionCount() {
		return sessionCount.get();
	}
}
```

在Shiro+Ehcache上修改整个配置类,

为了能够在Spring Boot中使用`SessionDao`，我们在ShiroConfig中配置该Bean(EhCache),

如果想自己实现sessionDao,可以通过实现` AbstractSessionDAO`.

如果使用的是Redis作为缓存实现，那么SessionDAO则为`RedisSessionDAO`:

```
@Bean
public RedisSessionDAO sessionDAO() {
    RedisSessionDAO redisSessionDAO = new RedisSessionDAO();
    redisSessionDAO.setRedisManager(redisManager());
    return redisSessionDAO;
}
@Bean
public SessionDAO sessionDAO() {
     return redisSessionDAO();
   	//return new MemorySessionDAO();
   }
  
```

```
@Configuration
public class ShiroConfig {
	
	@Bean
	public EhCacheManager getEhCacheManager() {
	    EhCacheManager em = new EhCacheManager();
	    em.setCacheManagerConfigFile("classpath:config/shiro-ehcache.xml");
	    return em;
	}
	
	@Bean
	public ShiroFilterFactoryBean shiroFilterFactoryBean(SecurityManager securityManager) {
		ShiroFilterFactoryBean shiroFilterFactoryBean = new ShiroFilterFactoryBean();
		shiroFilterFactoryBean.setSecurityManager(securityManager);
		shiroFilterFactoryBean.setLoginUrl("/login");
		shiroFilterFactoryBean.setSuccessUrl("/index");
		shiroFilterFactoryBean.setUnauthorizedUrl("/403");
		LinkedHashMap<String, String> filterChainDefinitionMap = new LinkedHashMap<>();
		
		filterChainDefinitionMap.put("/css/**", "anon");
		filterChainDefinitionMap.put("/js/**", "anon");
		filterChainDefinitionMap.put("/fonts/**", "anon");
		filterChainDefinitionMap.put("/img/**", "anon");
		filterChainDefinitionMap.put("/druid/**", "anon");
		filterChainDefinitionMap.put("/logout", "logout");
		filterChainDefinitionMap.put("/", "anon");
		filterChainDefinitionMap.put("/**", "user");
		
		shiroFilterFactoryBean.setFilterChainDefinitionMap(filterChainDefinitionMap);
		
		return shiroFilterFactoryBean;
	}
 
	@Bean  
    public SecurityManager securityManager(){  
       DefaultWebSecurityManager securityManager =  new DefaultWebSecurityManager();
       securityManager.setRealm(shiroRealm());
       securityManager.setRememberMeManager(rememberMeManager());
       securityManager.setCacheManager(getEhCacheManager());
       securityManager.setSessionManager(sessionManager());
       return securityManager;  
    }  
	
	@Bean(name = "lifecycleBeanPostProcessor")
    public LifecycleBeanPostProcessor lifecycleBeanPostProcessor() {
        return new LifecycleBeanPostProcessor();
    }
	
	@Bean  
    public ShiroRealm shiroRealm(){  
       ShiroRealm shiroRealm = new ShiroRealm();  
       return shiroRealm;  
    }  
	
	public SimpleCookie rememberMeCookie() {
		SimpleCookie cookie = new SimpleCookie("rememberMe");
		cookie.setMaxAge(86400);
		return cookie;
	}
	
	public CookieRememberMeManager rememberMeManager() {
		CookieRememberMeManager cookieRememberMeManager = new CookieRememberMeManager();
		cookieRememberMeManager.setCookie(rememberMeCookie());
		cookieRememberMeManager.setCipherKey(Base64.decode("4AvVhmFLUs0KTA3Kprsdag=="));
		return cookieRememberMeManager;
	}
	
	@Bean
    @DependsOn({"lifecycleBeanPostProcessor"})
    public DefaultAdvisorAutoProxyCreator advisorAutoProxyCreator() {
        DefaultAdvisorAutoProxyCreator advisorAutoProxyCreator = new DefaultAdvisorAutoProxyCreator();
        advisorAutoProxyCreator.setProxyTargetClass(true);
        return advisorAutoProxyCreator;
    }

    @Bean
    public AuthorizationAttributeSourceAdvisor authorizationAttributeSourceAdvisor(SecurityManager securityManager) {
        AuthorizationAttributeSourceAdvisor authorizationAttributeSourceAdvisor = new AuthorizationAttributeSourceAdvisor();
        authorizationAttributeSourceAdvisor.setSecurityManager(securityManager);
        return authorizationAttributeSourceAdvisor;
    }
    
    @Bean
	public ShiroDialect shiroDialect() {
		return new ShiroDialect();
	}
    
	@Bean
	public SessionDAO sessionDAO() {
		MemorySessionDAO sessionDAO = new MemorySessionDAO();
		return sessionDAO;
	}
	//在Shiro中，SessionDao通过org.apache.shiro.session.mgt.SessionManager进行管理，所以继续在ShiroConfig中配置SessionManager：
	@Bean
	public SessionManager sessionManager() {
		DefaultWebSessionManager sessionManager = new DefaultWebSessionManager();
		Collection<SessionListener> listeners = new ArrayList<SessionListener>();
		listeners.add(new ShiroSessionListener());
		sessionManager.setSessionListeners(listeners);
		sessionManager.setSessionDAO(sessionDAO());   //设置sessionDao
		return sessionManager;
	}
}
```

自定义service层

```
@Service("sessionService")
public class SessionServiceImpl implements SessionService {
	//shiro提供的session访问
	@Autowired
	private SessionDAO sessionDAO;

	//获取在线用户列表
	@Override
	public List<UserOnline> list() {
		List<UserOnline> list = new ArrayList<>();
		//获取所有session
		Collection<Session> sessions = sessionDAO.getActiveSessions();
		for (Session session : sessions) {
			UserOnline userOnline = new UserOnline();
			User user = new User();
			SimplePrincipalCollection principalCollection = new SimplePrincipalCollection();
			if (session.getAttribute(DefaultSubjectContext.PRINCIPALS_SESSION_KEY) == null) {
				continue;
			} else {
				//通过session获取用户
				principalCollection = (SimplePrincipalCollection) session
						.getAttribute(DefaultSubjectContext.PRINCIPALS_SESSION_KEY);
				user = (User) principalCollection.getPrimaryPrincipal();
				userOnline.setUsername(user.getUserName());
				userOnline.setUserId(user.getId().toString());
			}
			userOnline.setId((String) session.getId());
			userOnline.setHost(session.getHost());
			userOnline.setStartTimestamp(session.getStartTimestamp());
			userOnline.setLastAccessTime(session.getLastAccessTime());
			Long timeout = session.getTimeout();
			if (timeout == 0l) {
				userOnline.setStatus("离线");
			} else {
				userOnline.setStatus("在线");
			}
			userOnline.setTimeout(timeout);
			list.add(userOnline);
		}
		return list;
	}
	//踢出用户
	@Override
	public boolean forceLogout(String sessionId) {
		Session session = sessionDAO.readSession(sessionId);
		session.setTimeout(0);
		return true;
	}
}
```

通过SessionDao的`getActiveSessions()`方法，我们可以获取所有有效的Session，通过该Session，我们还可以获取到当前用户的Principal信息。

值得说明的是，当某个用户被踢出后（Session Time置为0），该Session并不会立刻从ActiveSessions中剔除，所以我们可以通过其timeout信息来判断该用户在线与否。

如果使用的Redis作为缓存实现，那么，`forceLogout()`方法需要稍作修改：

```
@Override
public boolean forceLogout(String sessionId) {
    Session session = sessionDAO.readSession(sessionId);
    sessionDAO.delete(session);
    return true;
}
```

接口测试

```
@Controller
@RequestMapping("/online")
public class SessionController {
	
	@Autowired
	SessionService sessionService;
	
	@RequestMapping("index")
	public String online() {
		return "online";
	}

	@ResponseBody
	@RequestMapping("list")
	public List<UserOnline> list() {
		return sessionService.list();
	}

	@ResponseBody
	@RequestMapping("forceLogout")
	public ResponseBo forceLogout(String id) {
		try {
			sessionService.forceLogout(id);
			return ResponseBo.ok();
		} catch (Exception e) {
			e.printStackTrace();
			return ResponseBo.error("踢出用户失败");
		}
	}
}
```

在线用户功能页面online.html

```
<!DOCTYPE html>
<html xmlns:th="http://www.thymeleaf.org">
<head>
    <meta charset="UTF-8">
    <title>在线用户管理</title>
    <script th:src="@{/js/jquery-1.11.1.min.js}"></script>
    <script th:src="@{/js/dateFormat.js}"></script>
</head>
<body>
    <h3>在线用户数：<span id="onlineCount"></span></h3>
    <table>
        <tr>
            <th>序号</th>
            <th>用户名称</th>
            <th>登录时间</th>
            <th>最后访问时间</th>
            <th>主机</th>
            <th>状态</th>
            <th>操作</th>
        </tr>
    </table>
    <a th:href="@{/index}">返回</a>
</body>
<script th:inline="javascript">
    var ctx = [[@{/}]];
    $.get(ctx + "online/list", {}, function(r){
        var length = r.length;
        $("#onlineCount").text(length);
        var html = "";
        for(var i = 0; i < length; i++){
            html += "<tr>"
                + "<td>" + (i+1) + "</td>"
                + "<td>" + r[i].username + "</td>"
                + "<td>" + new Date(r[i].startTimestamp).Format("yyyy-MM-dd hh:mm:ss") + "</td>"
                + "<td>" + new Date(r[i].lastAccessTime).Format("yyyy-MM-dd hh:mm:ss") + "</td>"
                + "<td>" + r[i].host + "</td>"
                + "<td>" + r[i].status + "</td>"
                + "<td><a href='#' onclick='offline(\"" + r[i].id + "\",\"" + r[i].status +"\")'>下线</a></td>"
                + "</tr>";
        }
        $("table").append(html);
    },"json");
	
    function offline(id,status){
        if(status == "离线"){
            alert("该用户已是离线状态！！");
            return;
        }
        $.get(ctx + "online/forceLogout", {"id": id}, function(r){
            if (r.code == 0) {
                alert('该用户已强制下线！');
                location.href = ctx + 'online/index';
            } else {
                alert(r.msg);
            }
        },"json");
    }
</script>
</html>
```

html.xml引入

```
<body>
    <p>你好！[[${user.userName}]]</p>
    <p shiro:hasRole="admin">你的角色为超级管理员</p>
    <p shiro:hasRole="test">你的角色为测试账户</p>
    <div>
        <a shiro:hasPermission="user:user" th:href="@{/user/list}">获取用户信息</a>
        <a shiro:hasPermission="user:add" th:href="@{/user/add}">新增用户</a>
        <a shiro:hasPermission="user:delete" th:href="@{/user/delete}">删除用户</a>
    </div>
    <a shiro:hasRole="admin" th:href="@{/online/index}">在线用户管理</a>
    <a th:href="@{/logout}">注销</a>
</body>
```

## 实现shiro-redisCache+sessionDao

注意redisCache和sessionDao 中使用的key不同

RedisManager

```
public class RedisManager {
    @Value("${spring.redis.host}")
    private String host = "127.0.0.1";

    @Value("${spring.redis.port}")
    private int port = 6379;

    // 0 - never expire
    private int expire = 0;

    //timeout for jedis try to connect to redis server, not expire time! In milliseconds
    @Value("${spring.redis.timeout}")
    private int timeout = 0;

    @Value("${spring.redis.password}")
    private String password = "";

    private static JedisPool jedisPool = null;

    public RedisManager() {

    }

    /**
     * 初始化方法
     */
    public void init() {
        if (jedisPool == null) {
            if (password != null && !"".equals(password)) {
                jedisPool = new JedisPool(new JedisPoolConfig(), host, port, timeout, password);
            } else if (timeout != 0) {
                jedisPool = new JedisPool(new JedisPoolConfig(), host, port, timeout);
            } else {
                jedisPool = new JedisPool(new JedisPoolConfig(), host, port);
            }

        }
    }

    /**
     * get value from redis
     *
     * @param key
     * @return
     */
    public byte[] get(byte[] key) {
        byte[] value = null;
        Jedis jedis = jedisPool.getResource();
        try {
            value = jedis.get(key);
        } finally {
            if (jedis != null) {
                jedis.close();
            }
        }
        return value;
    }

    /**
     * set
     *
     * @param key
     * @param value
     * @return
     */
    public byte[] set(byte[] key, byte[] value) {
        Jedis jedis = jedisPool.getResource();
        try {
            jedis.set(key, value);
            if (this.expire != 0) {
                jedis.expire(key, this.expire);
            }
        } finally {
            if (jedis != null) {
                jedis.close();
            }
        }
        return value;
    }

    /**
     * set
     *
     * @param key
     * @param value
     * @param expire
     * @return
     */
    public byte[] set(byte[] key, byte[] value, int expire) {
        Jedis jedis = jedisPool.getResource();
        try {
            jedis.set(key, value);
            if (expire != 0) {
                jedis.expire(key, expire);
            }
        } finally {
            if (jedis != null) {
                jedis.close();
            }
        }
        return value;
    }

    /**
     * del
     *
     * @param key
     */
    public void del(byte[] key) {
        Jedis jedis = jedisPool.getResource();
        try {
            jedis.del(key);
        } finally {
            if (jedis != null) {
                jedis.close();
            }
        }
    }

    /**
     * flush
     */
    public void flushDB() {
        Jedis jedis = jedisPool.getResource();
        try {
            jedis.flushDB();
        } finally {
            if (jedis != null) {
                jedis.close();
            }
        }
    }

    /**
     * size
     */
    public Long dbSize() {
        Long dbSize = 0L;
        Jedis jedis = jedisPool.getResource();
        try {
            dbSize = jedis.dbSize();
        } finally {
            if (jedis != null) {
                jedis.close();
            }
        }
        return dbSize;
    }

    /**
     * keys
     *
     * @param pattern regex
     * @return
     */
    public Set<byte[]> keys(String pattern) {
        Set<byte[]> keys = null;
        Jedis jedis = jedisPool.getResource();
        try {
            keys = jedis.keys(pattern.getBytes());
        } finally {
            if (jedis != null) {
                jedis.close();
            }
        }
        return keys;
    }

    public String getHost() {
        return host;
    }

    public void setHost(String host) {
        this.host = host;
    }

    public int getPort() {
        return port;
    }

    public void setPort(int port) {
        this.port = port;
    }

    public int getExpire() {
        return expire;
    }

    public void setExpire(int expire) {
        this.expire = expire;
    }

    public int getTimeout() {
        return timeout;
    }

    public void setTimeout(int timeout) {
        this.timeout = timeout;
    }

    public String getPassword() {
        return password;
    }

    public void setPassword(String password) {
        this.password = password;
    }
}
```

RedisCache

```
public class RedisCache<K, V> implements Cache<K, V> {

    private Logger logger = LoggerFactory.getLogger(this.getClass());

    /**
     * The wrapped Jedis instance.
     */
    private RedisManager cache;

    /**
     * The Redis key prefix for the sessions
     */
    private String keyPrefix = "shiro_redis_session:";

    /**
     * Returns the Redis session keys
     * prefix.
     * @return The prefix
     */
    public String getKeyPrefix() {
        return keyPrefix;
    }

    /**
     * Sets the Redis sessions key
     * prefix.
     * @param keyPrefix The prefix
     */
    public void setKeyPrefix(String keyPrefix) {
        this.keyPrefix = keyPrefix;
    }

    /**
     * 通过一个JedisManager实例构造RedisCache
     */
    public RedisCache(RedisManager cache){
        if (cache == null) {
            throw new IllegalArgumentException("Cache argument cannot be null.");
        }
        this.cache = cache;
    }

    /**
     * Constructs a cache instance with the specified
     * Redis manager and using a custom key prefix.
     * @param cache The cache manager instance
     * @param prefix The Redis key prefix
     */
    public RedisCache(RedisManager cache,
                      String prefix){

        this( cache );

        // set the prefix
        this.keyPrefix = prefix;
    }

    /**
     * 获得byte[]型的key
     * @param key
     * @return
     */
    private byte[] getByteKey(K key){
        if(key instanceof String){
            String preKey = this.keyPrefix + key;
            return preKey.getBytes();
        }else{
            return SerializeUtils.serialize(key);
        }
    }

    @Override
    public V get(K key) throws CacheException {
        logger.debug("根据key从Redis中获取对象 key [" + key + "]");
        try {
            if (key == null) {
                return null;
            }else{
                byte[] rawValue = cache.get(getByteKey(key));
                @SuppressWarnings("unchecked")
                V value = (V)SerializeUtils.deserialize(rawValue);
                return value;
            }
        } catch (Throwable t) {
            throw new CacheException(t);
        }

    }

    @Override
    public V put(K key, V value) throws CacheException {
        logger.debug("根据key从存储 key [" + key + "]");
        try {
            cache.set(getByteKey(key), SerializeUtils.serialize(value));
            return value;
        } catch (Throwable t) {
            throw new CacheException(t);
        }
    }

    @Override
    public V remove(K key) throws CacheException {
        logger.debug("从redis中删除 key [" + key + "]");
        try {
            V previous = get(key);
            cache.del(getByteKey(key));
            return previous;
        } catch (Throwable t) {
            throw new CacheException(t);
        }
    }

    @Override
    public void clear() throws CacheException {
        logger.debug("从redis中删除所有元素");
        try {
            cache.flushDB();
        } catch (Throwable t) {
            throw new CacheException(t);
        }
    }

    @Override
    public int size() {
        try {
            Long longSize = new Long(cache.dbSize());
            return longSize.intValue();
        } catch (Throwable t) {
            throw new CacheException(t);
        }
    }

    @SuppressWarnings("unchecked")
    @Override
    public Set<K> keys() {
        try {
            Set<byte[]> keys = cache.keys(this.keyPrefix + "*");
            if (CollectionUtils.isEmpty(keys)) {
                return Collections.emptySet();
            }else{
                Set<K> newKeys = new HashSet<K>();
                for(byte[] key:keys){
                    newKeys.add((K)key);
                }
                return newKeys;
            }
        } catch (Throwable t) {
            throw new CacheException(t);
        }
    }

    @Override
    public Collection<V> values() {
        try {
            Set<byte[]> keys = cache.keys(this.keyPrefix + "*");
            if (!CollectionUtils.isEmpty(keys)) {
                List<V> values = new ArrayList<V>(keys.size());
                for (byte[] key : keys) {
                    @SuppressWarnings("unchecked")
                    V value = get((K)key);
                    if (value != null) {
                        values.add(value);
                    }
                }
                return Collections.unmodifiableList(values);
            } else {
                return Collections.emptyList();
            }
        } catch (Throwable t) {
            throw new CacheException(t);
        }
    }
}
```

**RedisCacheManager**

```
public class RedisCacheManager implements CacheManager {

    private static final Logger logger = LoggerFactory
            .getLogger(RedisCacheManager.class);

    // fast lookup by name map
    private final ConcurrentMap<String, Cache> caches = new ConcurrentHashMap<String, Cache>();

    private RedisManager redisManager;

    /**
     * The Redis key prefix for caches
     */
    private String keyPrefix = "shiro_redis_cache:";

    /**
     * Returns the Redis session keys
     * prefix.
     * @return The prefix
     */
    public String getKeyPrefix() {
        return keyPrefix;
    }

    /**
     * Sets the Redis sessions key
     * prefix.
     * @param keyPrefix The prefix
     */
    public void setKeyPrefix(String keyPrefix) {
        this.keyPrefix = keyPrefix;
    }

    @Override
    public <K, V> Cache<K, V> getCache(String name) throws CacheException {
        logger.debug("获取名称为: " + name + " 的RedisCache实例");

        Cache c = caches.get(name);

        if (c == null) {

            // initialize the Redis manager instance
            redisManager.init();

            // create a new cache instance
            c = new RedisCache<K, V>(redisManager, keyPrefix);

            // add it to the cache collection
            caches.put(name, c);
        }
        return c;
    }

    public RedisManager getRedisManager() {
        return redisManager;
    }

    public void setRedisManager(RedisManager redisManager) {
        this.redisManager = redisManager;
    }
}
```

SessionDao

```
public class RedisSessionDAO extends AbstractSessionDAO {
    private static Logger logger = LoggerFactory.getLogger(RedisSessionDAO.class);
    
    private RedisManager redisManager;

    /**
     * The Redis key prefix for the sessions
     */
    private String keyPrefix = "shiro_redis_session:";

    @Override
    public void update(Session session) throws UnknownSessionException {
        this.saveSession(session);
    }

    /**
     * save session
     * @param session
     * @throws UnknownSessionException
     */
    private void saveSession(Session session) throws UnknownSessionException{
        if(session == null || session.getId() == null){
            logger.error("session or session id is null");
            return;
        }

        byte[] key = getByteKey(session.getId());
        byte[] value = SerializeUtils.serialize(session);
        session.setTimeout(redisManager.getExpire()*1000);
        this.redisManager.set(key, value, redisManager.getExpire());
    }

    @Override
    public void delete(Session session) {
        if(session == null || session.getId() == null){
            logger.error("session or session id is null");
            return;
        }
        redisManager.del(this.getByteKey(session.getId()));

    }

    @Override
    public Collection<Session> getActiveSessions() {
        Set<Session> sessions = new HashSet<Session>();

        Set<byte[]> keys = redisManager.keys(this.keyPrefix + "*");
        if(keys != null && keys.size()>0){
            for(byte[] key:keys){
                Session s = (Session)SerializeUtils.deserialize(redisManager.get(key));
                sessions.add(s);
            }
        }

        return sessions;
    }

    @Override
    protected Serializable doCreate(Session session) {
        Serializable sessionId = this.generateSessionId(session);
        this.assignSessionId(session, sessionId);
        this.saveSession(session);
        return sessionId;
    }

    @Override
    protected Session doReadSession(Serializable sessionId) {
        if(sessionId == null){
            logger.error("session id is null");
            return null;
        }

        Session s = (Session)SerializeUtils.deserialize(redisManager.get(this.getByteKey(sessionId)));
        return s;
    }

    /**
     * 获得byte[]型的key
     * @param  key
     * @return
     */
    private byte[] getByteKey(Serializable sessionId){
        String preKey = this.keyPrefix + sessionId;
        return preKey.getBytes();
    }

    public RedisManager getRedisManager() {
        return redisManager;
    }

    public void setRedisManager(RedisManager redisManager) {
        this.redisManager = redisManager;

        /**
         * 初始化redisManager
         */
        this.redisManager.init();
    }

    /**
     * Returns the Redis session keys
     * prefix.
     * @return The prefix
     */
    public String getKeyPrefix() {
        return keyPrefix;
    }

    /**
     * Sets the Redis sessions key
     * prefix.
     * @param keyPrefix The prefix
     */
    public void setKeyPrefix(String keyPrefix) {
        this.keyPrefix = keyPrefix;
    }
}
```

配置类

```
   @Bean
    public RedisManager redisManager() {
        RedisManager redisManager = new RedisManager();
        //如果采用redis默认设置可省略
        redisManager.setHost(host);
        redisManager.setPort(port);
        redisManager.setExpire(1800);// 配置缓存过期时间
        //redisManager.setTimeout(1800);
        //redisManager.setPassword(password);
        return redisManager;
    }

    public RedisCacheManager rediscacheManager() {
        RedisCacheManager redisCacheManager = new RedisCacheManager();
        redisCacheManager.setRedisManager(redisManager());
        return redisCacheManager;
    }
    
    @Bean
    public SecurityManager securityManager() {
        DefaultWebSecurityManager securityManager = new DefaultWebSecurityManager();
        //设置realm.
        securityManager.setRealm(userRealm());
        // 自定义缓存实现 使用redis
        securityManager.setCacheManager(rediscacheManager());
        securityManager.setSessionManager(sessionManager());
        return securityManager;
    }
    
//----------------------------------------------
	@Bean
    public SessionDAO sessionDAO() {
        RedisSessionDAO redisSessionDAO = new RedisSessionDAO();
        redisSessionDAO.setRedisManager(redisManager());
        return redisSessionDAO;
    }

    @Bean
    public DefaultWebSessionManager sessionManager() {
        DefaultWebSessionManager sessionManager = new DefaultWebSessionManager();
        sessionManager.setGlobalSessionTimeout(tomcatTimeout * 1000);
        //注册sesssionDao
        sessionManager.setSessionDAO(sessionDAO());
        Collection<SessionListener> listeners = new ArrayList<SessionListener>();
        listeners.add(new BDSessionListener());
        sessionManager.setSessionListeners(listeners);
        return sessionManager;
    }
```





## auth0+cache+jwt ??

```
public class JwtUtils {
   
   /**
     * 获得token中的信息无需secret解密也能获得
     * @return token中包含的签发时间
     */
    public static Date getIssuedAt(String token) {
        try {
            DecodedJWT jwt = JWT.decode(token);
            return jwt.getIssuedAt();
        } catch (JWTDecodeException e) {
            return null;
        }
    }
    /**
     * 获得token中的信息无需secret解密也能获得
     * @return token中包含的用户名
     */
    public static String getUsername(String token) {
        try {
            DecodedJWT jwt = JWT.decode(token);
            return jwt.getClaim("username").asString();
        } catch (JWTDecodeException e) {
            return null;
        }
    }
    /**
     * 生成签名,expireTime后过期
     * @param username 用户名
     * @param time 过期时间s
     * @return 加密的token
     */
    public static String sign(String username, String salt, long time) {
        try {
            Date date = new Date(System.currentTimeMillis()+time*1000);
            Algorithm algorithm = Algorithm.HMAC256(salt);
            // 附带username信息
            return JWT.create()
                    .withClaim("username", username)
                    .withExpiresAt(date)
                    .withIssuedAt(new Date())
                    .sign(algorithm);
        } catch (UnsupportedEncodingException e) {
            return null;
        }
    }
    /**
     * token是否过期
     * @return true：过期
     */
    public static boolean isTokenExpired(String token) {
        Date now = Calendar.getInstance().getTime();
        DecodedJWT jwt = JWT.decode(token);
        return jwt.getExpiresAt().before(now);
    }
    /**
     * 生成随机盐,长度32位
     * @return
     */
    public static String generateSalt(){
        SecureRandomNumberGenerator secureRandom = new SecureRandomNumberGenerator();
        String hex = secureRandom.nextBytes(16).toHex();
        return hex;
    }
}
```

```
@Service
public class UserService {
   
   private static final String encryptSalt = "F12839WhsnnEV$#23b";

   @Autowired
   private StringRedisTemplate redisTemplate;

    /**
     * 保存user登录信息，返回token
     */
    public String generateJwtToken(String username) {
       String salt = JwtUtils.generateSalt();
       //将salt保存到数据库或者缓存中
      redisTemplate.opsForValue().set("token:"+username, salt, 3600, TimeUnit.SECONDS);
       return JwtUtils.sign(username, salt, 3600); //生成jwt token，设置过期时间为1小时
    }
    
    /**
     * 获取上次token生成时的salt值和登录用户信息
     * @param username
     * @return
     */
    public UserDto getJwtTokenInfo(String username) {
       String salt = "";
       //从数据库或者缓存中取出jwt token生成时用的salt
      salt = redisTemplate.opsForValue().get("token:"+username);
       UserDto user = getUserInfo(username);
       user.setSalt(salt);
       return user;
    }

    /**
     * 清除token信息
     * @param username 登录用户名
     */
    public void deleteLoginInfo(String username) {
      redisTemplate.delete("token:"+username);
    }
    
    /**
     * 获取数据库中保存的用户信息，主要是加密后的密码
     * @param userName
     * @return
     */
    public UserDto getUserInfo(String userName) {
      UserDto user = new UserDto();
      String pwd=redisTemplate.opsForValue().get("user:"+userName);
      user.setUsername(userName);
      user.setEncryptPwd(new Sha256Hash(pwd,encryptSalt).toHex());

      return user;
    }
    
    public List<String> getUserRoles(Long userId){
      //模拟角色
      return Arrays.asList("admin");
    }
}
```

```
import org.apache.shiro.authc.HostAuthenticationToken;

public class JWTToken implements HostAuthenticationToken {
   private static final long serialVersionUID = 9217639903967592166L;
   private String token;
    private String host;
    public JWTToken(String token) {
        this(token, null);
    }
    public JWTToken(String token, String host) {
        this.token = token;
        this.host = host;
    }
    public String getToken(){
        return this.token;
    }
    public String getHost() {
        return host;
    }
    @Override
    public Object getPrincipal() {
        return token;
    }
    @Override
    public Object getCredentials() {
        return token;
    }
    @Override
    public String toString(){
        return token + ':' + host;
    }
}
```

```
import org.apache.shiro.authc.*;
import org.apache.shiro.authc.credential.HashedCredentialsMatcher;
import org.apache.shiro.authz.AuthorizationInfo;
import org.apache.shiro.authz.SimpleAuthorizationInfo;
import org.apache.shiro.crypto.hash.Sha256Hash;
import org.apache.shiro.realm.AuthorizingRealm;
import org.apache.shiro.subject.PrincipalCollection;
import org.apache.shiro.util.ByteSource;
import java.util.List;

public class DbShiroRealm extends AuthorizingRealm {
   private final Logger log = LoggerFactory.getLogger(DbShiroRealm.class);
   
   private static final String encryptSalt = "F12839WhsnnEV$#23b";
   private UserService userService;
   
   public DbShiroRealm(UserService userService) {
      this.userService = userService;
      this.setCredentialsMatcher(new HashedCredentialsMatcher(Sha256Hash.ALGORITHM_NAME));
   }
   
   @Override
    public boolean supports(AuthenticationToken token) {
        return token instanceof UsernamePasswordToken;
    }
   
   @Override
   protected AuthenticationInfo doGetAuthenticationInfo(AuthenticationToken token) throws AuthenticationException {
      UsernamePasswordToken userpasswordToken = (UsernamePasswordToken)token;
      String username = userpasswordToken.getUsername();
      UserDto user = userService.getUserInfo(username);
      if(user == null)
         throw new AuthenticationException("用户名或者密码错误");
      return new SimpleAuthenticationInfo(user, user.getEncryptPwd(), ByteSource.Util.bytes(encryptSalt), "dbRealm");
   }

   @Override
   protected AuthorizationInfo doGetAuthorizationInfo(PrincipalCollection principals) {      
        SimpleAuthorizationInfo simpleAuthorizationInfo = new SimpleAuthorizationInfo();
        UserDto user = (UserDto) principals.getPrimaryPrincipal();
        List<String> roles = user.getRoles();
        if(roles == null) {
            roles = userService.getUserRoles(user.getUserId());
            user.setRoles(roles);
        }
        if (roles != null)
            simpleAuthorizationInfo.addRoles(roles);

        return simpleAuthorizationInfo;
   } 
}
```

```
import org.apache.shiro.authc.AuthenticationException;
import org.apache.shiro.authc.AuthenticationInfo;
import org.apache.shiro.authc.AuthenticationToken;
import org.apache.shiro.authc.SimpleAuthenticationInfo;
import org.apache.shiro.authz.AuthorizationInfo;
import org.apache.shiro.authz.SimpleAuthorizationInfo;
import org.apache.shiro.realm.AuthorizingRealm;
import org.apache.shiro.subject.PrincipalCollection;

/**
 * 自定义身份认证
 * 基于HMAC（ 散列消息认证码）的控制域
 */
public class JWTShiroRealm extends AuthorizingRealm {
   private final Logger log = LoggerFactory.getLogger(JWTShiroRealm.class);

    protected UserService userService;

    public JWTShiroRealm(UserService userService){
        this.userService = userService;
        this.setCredentialsMatcher(new JWTCredentialsMatcher());
    }

    @Override
    public boolean supports(AuthenticationToken token) {
        return token instanceof JWTToken;
    }
    /**
     * 认证信息.(身份验证) : Authentication 是用来验证用户身份
     * 默认使用此方法进行用户名正确与否验证，错误抛出异常即可。
     */
    @Override
    protected AuthenticationInfo doGetAuthenticationInfo(AuthenticationToken authcToken) throws AuthenticationException {    
        JWTToken jwtToken = (JWTToken) authcToken;
        String token = jwtToken.getToken();
        UserDto user = userService.getJwtTokenInfo(JwtUtils.getUsername(token));
        if(user == null)
            throw new AuthenticationException("token过期，请重新登录");

        SimpleAuthenticationInfo authenticationInfo = new SimpleAuthenticationInfo(user, user.getSalt(), "jwtRealm");

        return authenticationInfo;
    }

    @Override
    protected AuthorizationInfo doGetAuthorizationInfo(PrincipalCollection principals) {
        return new SimpleAuthorizationInfo();
    }
}
```

```
import com.auth0.jwt.JWT;
import com.auth0.jwt.JWTVerifier;
import com.auth0.jwt.algorithms.Algorithm;
import com.auth0.jwt.exceptions.JWTVerificationException;
import org.apache.shiro.authc.AuthenticationInfo;
import org.apache.shiro.authc.AuthenticationToken;
import org.apache.shiro.authc.credential.CredentialsMatcher;

import java.io.UnsupportedEncodingException;
public class JWTCredentialsMatcher implements CredentialsMatcher {
   
   private final Logger log = LoggerFactory.getLogger(JWTCredentialsMatcher.class);

    @Override
    public boolean doCredentialsMatch(AuthenticationToken authenticationToken, AuthenticationInfo authenticationInfo) {
        String token = (String) authenticationToken.getCredentials();
        Object stored = authenticationInfo.getCredentials();
        String salt = stored.toString();
        
        UserDto user = (UserDto)authenticationInfo.getPrincipals().getPrimaryPrincipal();
        try {
            Algorithm algorithm = Algorithm.HMAC256(salt);
            JWTVerifier verifier = JWT.require(algorithm)
                    .withClaim("username", user.getUsername())
                    .build();
            verifier.verify(token);
            return true;
        } catch (UnsupportedEncodingException | JWTVerificationException e) {
            log.error("Token Error:{}", e.getMessage());
        }
        return false;
    }
}
```

配置类

```
import org.apache.shiro.authc.Authenticator;
import org.apache.shiro.authc.pam.FirstSuccessfulStrategy;
import org.apache.shiro.authc.pam.ModularRealmAuthenticator;
import org.apache.shiro.mgt.SecurityManager;
import org.apache.shiro.mgt.SessionStorageEvaluator;
import org.apache.shiro.realm.Realm;
import org.apache.shiro.spring.security.interceptor.AuthorizationAttributeSourceAdvisor;
import org.apache.shiro.spring.web.ShiroFilterFactoryBean;
import org.apache.shiro.spring.web.config.DefaultShiroFilterChainDefinition;
import org.apache.shiro.spring.web.config.ShiroFilterChainDefinition;
import org.apache.shiro.web.mgt.DefaultWebSessionStorageEvaluator;
import org.springframework.beans.factory.annotation.Qualifier;
import org.springframework.boot.web.servlet.FilterRegistrationBean;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
@Configuration
public class ShiroConfig {
    @Bean
    public AuthorizationAttributeSourceAdvisor authorizationAttributeSourceAdvisor(@Qualifier("securityManager") SecurityManager securityManager) {
        AuthorizationAttributeSourceAdvisor advisor = new AuthorizationAttributeSourceAdvisor();
        advisor.setSecurityManager(securityManager);
        return advisor;
    }

    @Bean
    public FilterRegistrationBean<Filter> filterRegistrationBean(SecurityManager securityManager,UserService userService) throws Exception{
        FilterRegistrationBean<Filter> filterRegistration = new FilterRegistrationBean<Filter>();
        filterRegistration.setFilter((Filter)shiroFilter(securityManager, userService).getObject());
        filterRegistration.addInitParameter("targetFilterLifecycle", "true");
        filterRegistration.setAsyncSupported(true);
        filterRegistration.setEnabled(true);
        filterRegistration.setDispatcherTypes(DispatcherType.REQUEST,DispatcherType.ASYNC);

        return filterRegistration;
    }

    @Bean
    public Authenticator authenticator(UserService userService) {
        ModularRealmAuthenticator authenticator = new ModularRealmAuthenticator();
        authenticator.setRealms(Arrays.asList(jwtShiroRealm(userService), dbShiroRealm(userService)));
        authenticator.setAuthenticationStrategy(new FirstSuccessfulStrategy());
        return authenticator;
    }

    @Bean
    protected SessionStorageEvaluator sessionStorageEvaluator(){
        DefaultWebSessionStorageEvaluator sessionStorageEvaluator = new DefaultWebSessionStorageEvaluator();
        sessionStorageEvaluator.setSessionStorageEnabled(false);
        return sessionStorageEvaluator;
    }

    @Bean("dbRealm")
    public Realm dbShiroRealm(UserService userService) {
        DbShiroRealm myShiroRealm = new DbShiroRealm(userService);
        return myShiroRealm;
    }

    @Bean("jwtRealm")
    public Realm jwtShiroRealm(UserService userService) {
        JWTShiroRealm myShiroRealm = new JWTShiroRealm(userService);
        return myShiroRealm;
    }

    /**
     * 设置过滤器
     */
    @Bean("shiroFilter")
    public ShiroFilterFactoryBean shiroFilter(SecurityManager securityManager, UserService userService) {
       ShiroFilterFactoryBean factoryBean = new ShiroFilterFactoryBean();
        factoryBean.setSecurityManager(securityManager);
        Map<String, Filter> filterMap = factoryBean.getFilters();
        filterMap.put("authcToken", createAuthFilter(userService));
        filterMap.put("anyRole", createRolesFilter());
        factoryBean.setFilters(filterMap);
        factoryBean.setFilterChainDefinitionMap(shiroFilterChainDefinition().getFilterChainMap());

        return factoryBean;
    }

    @Bean
    protected ShiroFilterChainDefinition shiroFilterChainDefinition() {
        DefaultShiroFilterChainDefinition chainDefinition = new DefaultShiroFilterChainDefinition();
        //swagger2免拦截
        chainDefinition.addPathDefinition("/swagger-ui.html**", "anon");
        chainDefinition.addPathDefinition("/v2/api-docs", "anon");
        chainDefinition.addPathDefinition("/swagger-resources/**", "anon");
        chainDefinition.addPathDefinition("/webjars/**", "anon");

        //接口拦截
        chainDefinition.addPathDefinition("/api/login/getToken", "noSessionCreation,anon");
        chainDefinition.addPathDefinition("/api/sql/register", "noSessionCreation,anon");
        chainDefinition.addPathDefinition("/api/test/hello", "noSessionCreation,anon");
        chainDefinition.addPathDefinition("/api/test/upload", "noSessionCreation,authcToken");
        chainDefinition.addPathDefinition("/**", "noSessionCreation,authcToken");
        return chainDefinition;
    }

    protected JwtAuthFilter createAuthFilter(UserService userService){
        return new JwtAuthFilter(userService);
    }

    protected AnyRolesAuthorizationFilter createRolesFilter(){
        return new AnyRolesAuthorizationFilter();
    }
}
```

过滤器

```
import org.apache.commons.lang3.BooleanUtils;
import org.apache.http.HttpStatus;
import org.apache.shiro.subject.Subject;
import org.apache.shiro.web.filter.authz.AuthorizationFilter;
import org.apache.shiro.web.util.WebUtils;

import javax.servlet.ServletRequest;
import javax.servlet.ServletResponse;
import javax.servlet.http.HttpServletResponse;
import java.io.IOException;
public class AnyRolesAuthorizationFilter  extends AuthorizationFilter {
   
   @Override
    protected void postHandle(ServletRequest request, ServletResponse response){
      request.setAttribute("anyRolesAuthFilter.FILTERED", true);
   }

    @Override
    protected boolean isAccessAllowed(ServletRequest servletRequest, ServletResponse servletResponse, Object mappedValue) throws Exception {
       Boolean afterFiltered = (Boolean)(servletRequest.getAttribute("anyRolesAuthFilter.FILTERED"));
        if( BooleanUtils.isTrue(afterFiltered))
           return true;
        
        Subject subject = getSubject(servletRequest, servletResponse);
        String[] rolesArray = (String[]) mappedValue;
        if (rolesArray == null || rolesArray.length == 0) { //没有角色限制，有权限访问
            return true;
        }
        for (String role : rolesArray) {
            if (subject.hasRole(role)) //若当前用户是rolesArray中的任何一个，则有权限访问
                return true;
        }
        return false;
    }

    @Override
    protected boolean onAccessDenied(ServletRequest request, ServletResponse response) throws IOException {
        HttpServletResponse httpResponse = WebUtils.toHttp(response);
        httpResponse.setCharacterEncoding("UTF-8");
        httpResponse.setContentType("application/json;charset=utf-8");
        httpResponse.setStatus(HttpStatus.SC_UNAUTHORIZED);
        return false;
    }
}
```

## SecurityUtils

```
public class ShiroUtils {
    @Autowired
    private static SessionDAO sessionDAO;

    public static Subject getSubjct() {
        return SecurityUtils.getSubject();
    }
    public static UserDO getUser() {
        Object object = getSubjct().getPrincipal();
        return (UserDO)object;
    }
    public static Long getUserId() {
        return getUser().getUserId();
    }
    public static void logout() {
        getSubjct().logout();
    }

    public static List<Principal> getPrinciples() {
        List<Principal> principals = null;
        Collection<Session> sessions = sessionDAO.getActiveSessions();
        return principals;
    }
}
```

