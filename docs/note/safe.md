# 安全相关

## Jwt

### jjwt

```xml
<dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt</artifactId>
    <version>${jwt.version}</version>
</dependency>
```

jwt生成, 两种方式.

1.通过自定义claims:

```java
Map<String, Object> claims = new HashMap<>();
claims.put(Constants.LOGIN_USER_KEY, token);
 String token = Jwts.builder()
                .setClaims(claims)
                .signWith(SignatureAlgorithm.HS512, "abcdefghijklmnopqrstuvwxyz")
                .compact();
```

2.通过方法设置:

```java
String token = Jwts.builder()
                .setHeaderParam("typ", "JWT")
                .setSubject(userId+"")//相当于设置在claims key为sub
                .setIssuedAt(nowDate)
                .setExpiration(expireDate)//相当于设置在claims key为exp
                .signWith(SignatureAlgorithm.HS512, secret)
                .compact();
```

jwt解析

```java
private token parseToken(String token)
{
    Claims claims = Jwts.parser()
            .setSigningKey(secret)
            .parseClaimsJws(token)
            .getBody();
    return claims.get(Constants.LOGIN_USER_KEY);
}
```

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

### java-jwt

```xml
<dependency>
    <groupId>com.auth0</groupId>
    <artifactId>java-jwt</artifactId>
    <version>${jwt.auth0.version}</version>
</dependency>
```

创建jwt

```java
//首先拿到算法实例，用于创建jwt的sign
//可以使用其他工具创建salt字符串 salt就是随机字符串 
Algorithm algorithm = Algorithm.HMAC256(salt); //salt只有服务器知道
String token = JWT.create()
            .withIssuer("auth0")    // token发布者
            .withIssuedAt(new Date())   // 生成签名的时间
            .withExpiresAt(DateUtils.addHours(new Date(),2)) // 生成签名的有效期,小时
            .withClaim("name","xxx") // 插入数据
            .sign(algorithm);
//可以拿到base64解密获取json对象
```

shiro创建salt

```
SecureRandomNumberGenerator secureRandom = new SecureRandomNumberGenerator();
String salt = secureRandom.nextBytes(16).toHex();
```

验证jwt 如果验证失败将异常JWTVerificationException

```java
Algorithm algorithm = Algorithm.HMAC256(salt);
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

## Spring Security

Spring Security包含由众多的过滤器组成的一条链, 如:

`UsernamePasswordAuthenticationFilter`过滤器用于处理基于表单方式的登录认证;

> Spring Security使用`UsernamePasswordAuthenticationFilter`过滤器来拦截用户名密码认证请求，将用户名和密码封装成一个`UsernamePasswordToken`对象交给`AuthenticationManager`处理。`AuthenticationManager`将挑出一个支持处理该类型Token的`AuthenticationProvider`（这里为`DaoAuthenticationProvider`，`AuthenticationProvider`的其中一个实现类）来进行认证，认证过程中`DaoAuthenticationProvider`将调用`UserDetailService`的`loadUserByUsername`方法来获取UserDetails对象，如果UserDetails不为空并且密码和用户输入的密码匹配一致的话，则将认证信息保存到Session中，认证后我们便可以通过`Authentication`对象获取到认证的信息了。

`BasicAuthenticationFilter`用于处理基于HTTP Basic方式的登录验证等.

过滤器链的末尾为`FilterSecurityInterceptor`, 认证授权, 如不存在对应的权限将抛出异常.

异常由`ExceptionTranslateFilter`捕获并处理，如需要身份认证时将请求重定向到相应的认证页面，当认证失败或者权限不足时返回相应的提示信息。

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-security</artifactId>
</dependency>
```

当Spring项目中引入了Spring Security依赖的时候，项目会默认开启如下配置：

```xml
security:
  basic:
    enabled: true
```

这个配置开启了一个HTTP basic类型的认证(高版本已经默认是表单认证了)，所有服务的访问都必须先过这个认证(HTTP Basic认证框)，默认的用户名为user，密码由Sping Security自动生成，在IDE的控制台，

**基于表单认证**

配置将HTTP Basic认证修改为基于表单的认证方式。

创建一个配置类`BrowserSecurityConfig`继承`org.springframework.security.config.annotation.web.configuration.WebSecurityConfigurerAdapter`这个抽象类并重写`configure(HttpSecurity http)`方法。`WebSecurityConfigurerAdapter`是由Spring Security提供的Web应用安全配置的适配器：

```java
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

```
     * anyRequest          |   匹配所有请求路径
     * access              |   SpringEl表达式结果为true时可以访问
     * anonymous           |   匿名可以访问, 相当于对接口加@AnonymousAccess
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

### 自定义认证过程

自定义认证的过程需要实现Spring Security提供的`UserDetailService`接口.

>还可以通过实现AuthenticationProvider接口自定义认证

`loadUserByUsername`方法返回一个`UserDetail`接口，包含一些用于描述用户信息的方法，可以自定义`UserDetails`接口的实现类，也可以直接使用Spring Security提供的`UserDetails`接口实现类`org.springframework.security.core.userdetails.User`

创建一个`MyUser`对象，用于存放模拟的用户数据（实际中一般从数据库获取）：

```java
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

```java
@Configuration
public class UserDetailService implements UserDetailsService {
    @Autowired
    // 注入的BCryptPasswordEncoder对象,用于对密码强散列哈希加密, 相同密码每次加密后不同.
    // 此处使用该对象仅为了模拟数据库保存的加密后的密码,实际使用只需要配置由框架自动加密
    private PasswordEncoder passwordEncoder;
    @Override
    public UserDetails loadUserByUsername(String username) throws UsernameNotFoundException {
        // 模拟一个用户，替代数据库获取逻辑
        MyUser user = new MyUser();
        user.setUserName(username);
        user.setPassword(this.passwordEncoder.encode("123456"));
        // 输出加密后的密码
        System.out.println(user.getPassword());

        // 模拟一个admin的权限,该方法可以将逗号分隔的字符串转换为权限集合
        return new User(username, user.getPassword(), user.isEnabled(),
                user.isAccountNonExpired(), user.isCredentialsNonExpired(),
                user.isAccountNonLocked(), AuthorityUtils.commaSeparatedStringToAuthorityList("admin"));
    }
}
```

`org.springframework.security.core.userdetails.User`类包含一个7个参数的构造器和一个三个参数的构造器`User(String username, String password,Collection<? extends GrantedAuthority> authorities)`.

启动项目, 访问/login, 可使用任意username以及123456作为密码登录系统.

**拓展**

在测试或刚引入Spring Security时, 可通过`AuthenticationManagerBuilder`设置限制单一用户访问.

```java
@Bean
public PasswordEncoder passwordEncoder() {
    return new BCryptPasswordEncoder();
}
@Override
public void configure(AuthenticationManagerBuilder auth) throws Exception {
    User.UserBuilder builder = User.builder().passwordEncoder(passwordEncoder()::encode);     		       	     auth.inMemoryAuthentication()
    	.withUser(builder.username(username).password(password).roles("ADMIN").build());
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

### 自定义登录

src/main/resources/resources目录下定义一个login.html（不需要Controller跳转）

```html
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

```java
@Override
protected void configure(HttpSecurity http) throws Exception {
    http.formLogin() 
            .loginPage("/login.html") //登录页面请求URL
            .loginProcessingUrl("/login")	//登录表单action
            .and()
            .authorizeRequests() // 授权配置
            .antMatchers("/login.html").permitAll()	//登录不拦截, 否则会进入无限循环
            .anyRequest()  // 所有请求
            .authenticated(); // 都需要认证
    		.and().csrf().disable();//禁用csrf保护
}
```

### 登录成功和失败逻辑

Spring Security有一套默认的处理登录成功和失败的方法：当用户登录成功时，页面会跳转回引发登录的请求；登录失败时则是跳转到Spring Security默认的错误提示页面。可自定义配置来替换这套默认的处理机制。

#### 自定义登录成功逻辑

实现`org.springframework.security.web.authentication.AuthenticationSuccessHandler`接口的`onAuthenticationSuccess`方法

```java
@Component
public class MyAuthenticationSucessHandler implements AuthenticationSuccessHandler {
	@Autowired
    private ObjectMapper mapper;
    /**
	 *	Authentication对象参数包含了登录用户的UserDetail信息以及ip等信息
	 */
    @Override
    public void onAuthenticationSuccess(HttpServletRequest request, HttpServletResponse response,Authentication authentication) throws IOException, ServletException {
        response.setContentType("application/json;charset=utf-8");
        response.getWriter().write(mapper.writeValueAsString(authentication));
    }
}
```

配置

```java
@Configuration
public class BrowserSecurityConfig extends WebSecurityConfigurerAdapter {
    @Autowired
    private MyAuthenticationSucessHandler authenticationSucessHandler;

    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http.formLogin() 
                .loginPage("/authentication/require") 
                .loginProcessingUrl("/login")
                .successHandler(authenticationSucessHandler) // 处理登录成功
                .and()
                .authorizeRequests() 
                .antMatchers("/authentication/require", "/login.html").permitAll() 
                .anyRequest()
                .authenticated()
                .and().csrf().disable();
    }
}
```

如果想在登录成功后跳回到引发登录跳转的原页面, 可通过RequestCache和RedirectStrategy实现:

```java
@Component
public class MyAuthenticationSucessHandler implements AuthenticationSuccessHandler {
    private RequestCache requestCache = new HttpSessionRequestCache();
    private RedirectStrategy redirectStrategy = new DefaultRedirectStrategy();
    @Autowired
    private ObjectMapper mapper;
    @Override
    public void onAuthenticationSuccess(HttpServletRequest request,  response,Authentication authentication) throws IOException {
        SavedRequest savedRequest = requestCache.getRequest(request, response);
        redirectStrategy.sendRedirect(request, response, savedRequest.getRedirectUrl());
    }
}
```

如果想在登录后指定跳转的页面，比如跳转到`/index`，将上面`savedRequest.getRedirectUrl()`替换为字符串常量`/index`即可:

```java
@Component
public class MyAuthenticationSucessHandler implements AuthenticationSuccessHandler {
    private RedirectStrategy redirectStrategy = new DefaultRedirectStrategy();
    @Override
    public void onAuthenticationSuccess(HttpServletRequest request, HttpServletResponse response,Authentication authentication) throws IOException {
        redirectStrategy.sendRedirect(request, response, "/index");
    }
}
```

#### 自定义登录失败逻辑

实现`org.springframework.security.web.authentication.AuthenticationFailureHandler`的`onAuthenticationFailure`方法.

```java
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

`onAuthenticationFailure`方法的`AuthenticationException`参数是一个抽象类，Spring Security根据登录失败的原因封装了许多对应的实现类，如`BadCredentialsException`用户名或密码错误、`UsernameNotFoundException`用户名不存在等.

别忘了进行配置:

```
@Autowired
private MyAuthenticationFailureHandler authenticationFailureHandler;
//....
//... .failureHandler(authenticationFailureHandler) // 处理登录失败
```

**AuthenticationEntryPoint**:实现该类的commence方法同样可以实现认证失败处理.

配置方式略有不同

```
.exceptionHandling().authenticationEntryPoint(unauthorizedHandler).and()
```

### 验证码校验

认证前加入验证码校验:

Spring Security的认证校验是由`UsernamePasswordAuthenticationFilter`过滤器完成的，所以验证码校验逻辑需要在这个过滤器之前

验证码功能需要用到`spring-social-config`依赖：

```xml
 <dependency>
    <groupId>org.springframework.social</groupId>
    <artifactId>spring-social-config</artifactId>
</dependency>
```

首先定义一个验证码对象ImageCode：

```java
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

定义一个ValidateCodeController，用于处理生成验证码请求,  使用`sessionStrategy`将生成的验证码对象存储到Session中，并通过IO流将生成的图片输出到登录页面上：

```java
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

> `org.springframework.social.connect.web.HttpSessionSessionStrategy`对象封装了一些处理Session的方法，如setAttribute`、`getAttribute`和`removeAttribute`

创建验证码对象的方法如下, 直接使用即可:

```java
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

```html
<span style="display: inline">
    <input type="text" name="imageCode" placeholder="验证码" style="width: 50%;"/>
    <img src="/code/image"/>
</span>
```

要使生成验证码的请求不被拦截，需要在`BrowserSecurityConfig`的`configure`方法中配置免拦截：

```java
@Override
protected void configure(HttpSecurity http) throws Exception {
    http.formLogin() 
            .loginPage("/authentication/require")
            .loginProcessingUrl("/login")
            .successHandler(authenticationSucessHandler)
            .failureHandler(authenticationFailureHandler)
            .and()
            .authorizeRequests()
            .antMatchers("/authentication/require",
                    "/login.html",
                    "/code/image").permitAll() // 获取验证码接口放行
            .anyRequest()  
            .authenticated()
            .and().csrf().disable();
}
```

在校验验证码的过程中，可能会抛出各种验证码类型的异常，比如“验证码错误”、“验证码已过期”等，所以定义一个验证码类型的异常类, 继承Spring Security的认证异常抽象类`AuthenticationException`:

```java
public class ValidateCodeException extends AuthenticationException {
    private static final long serialVersionUID = 5022575393500654458L;
    ValidateCodeException(String message) {
        super(message);
    }
}
```

自定义验证码校验的过滤器`ValidateCodeFilter`：

```java
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

最后需要添加过滤器配置,  注入了`ValidateCodeFilter`，然后通过`addFilterBefore`方法将`ValidateCodeFilter`验证码校验过滤器添加到`UsernamePasswordAuthenticationFilter`之前:

```java
@Autowired
private ValidateCodeFilter validateCodeFilter;
@Override
protected void configure(HttpSecurity http) throws Exception {
	// 添加验证码校验过滤器
    http.addFilterBefore(validateCodeFilter, UsernamePasswordAuthenticationFilter.class) 
            //...
}
```

### RememberMe

在Spring Security中, RememberMe的逻辑为：当用户勾选了记住我选项并登录成功后，生成一个token并持久化到数据库，然后生成一个与该token相对应的cookie返回给浏览器。当用户过段时间再次访问系统时，如果该cookie没有过期，Spring Security便会根据cookie包含的信息从数据库中获取相应的token信息，帮用户自动完成登录操作。

在BrowserSecurityConfig中配置token持久化对象：

```java
@Configuration
public class BrowserSecurityConfig extends WebSecurityConfigurerAdapter {
    @Autowired
    private UserDetailService userDetailService;
    @Autowired
    private DataSource dataSource;
    @Bean
    public PersistentTokenRepository persistentTokenRepository() {
        // 数据库持久化, 使用JdbcTokenRepositoryImpl, 需要指定dataSource
        JdbcTokenRepositoryImpl jdbcTokenRepository = new JdbcTokenRepositoryImpl();
        jdbcTokenRepository.setDataSource(dataSource);
        // 是否启动项目时创建保存token信息的数据表
        jdbcTokenRepository.setCreateTableOnStartup(false);
        return jdbcTokenRepository;
    }
    ...
}
```

>  查看`JdbcTokenRepositoryImpl`的源码，可看到CREATE_TABLE_SQL`属性即存储token对象数据表的SQL语句:
>
> ```sql
> CREATE TABLE persistent_logins (
>     username VARCHAR (64) NOT NULL,
>     series VARCHAR (64) PRIMARY KEY,
>     token VARCHAR (64) NOT NULL,
>     last_used TIMESTAMP NOT NULL
> )
> ```

登录网页加入rememberMe选项, **其中`name`属性必须为`remember-me`**

```
<input type="checkbox" name="remember-me"/> 记住我
```

最后在认证流程中启用RemenberMe的功能, 并指定token持久化对象:

```java
@Override
protected void configure(HttpSecurity http) throws Exception {
    http.formLogin() 
       .and()
       .rememberMe()
       .tokenRepository(persistentTokenRepository()) // 配置 token 持久化仓库
       .tokenValiditySeconds(3600) // remember 过期时间，单为秒
       .userDetailsService(userDetailService) // 处理自动登录逻辑
       //...
}
```

> 设置userDetailService是为了处理根据token从数据库获取的用户名信息.

使用RememberMe登录后，网页新增了remember-me的cookie对象,  并且后台数据库表persistent_logins,  里面生成了username对应的token.

### 短信验证码登录

Spring Security默认提供的是账号密码的登录认证逻辑，如果要实现手机短信验证码登录认证功能,  需要自定义认证逻辑对自带的登录认证逻辑进行替换

**短信验证码生成**

定义短信验证码对象SmsCode：

```java
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

```java
@RestController
public class ValidateController {
    public final static String SESSION_KEY_SMS_CODE = "SESSION_KEY_SMS_CODE";
    private SessionStrategy sessionStrategy = new HttpSessionSessionStrategy();
    @GetMapping("/code/sms")
    public void createSmsCode(HttpServletRequest request, HttpServletResponse response, String mobile) throws IOException {
        // 生成了一个6位的纯数字随机数的验证码对象
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

**登录页**

```html
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

对获取短信URL`/code/sms`路径放行：

```java
@Override
protected void configure(HttpSecurity http) throws Exception {
    http.formLogin() 
		.authorizeRequests() 
    	.antMatchers("/authentication/require","/login.html","/code/sms").permitAll() 
        .anyRequest().authenticated() 
        .and().csrf().disable();
}
```

**短信验证码登录认证逻辑**

将`UsernamePasswordAuthenticationFilter`替换为自定义的拦截短信验证码登录请求的拦截器`SmsAuthenticationFitler`,并将手机号码封装到一个叫`SmsAuthenticationToken`的对象中.

在Spring Security中，认证处理都需要通过`AuthenticationManager`来代理，所以这里我们依旧将`SmsAuthenticationToken`交由`AuthenticationManager`处理.

接着我们需要定义一个支持处理`SmsAuthenticationToken`对象的`SmsAuthenticationProvider`来代替`DaoAuthenticationProvider`

`SmsAuthenticationProvider`调用`UserDetailService`的`loadUserByUsername`方法来处理认证。与用户名密码认证不一样的是，这里是通过`SmsAuthenticationToken`中的手机号去数据库中查询是否有与之对应的用户，如果有，则将该用户信息封装到`UserDetails`对象中返回并将认证后的信息保存到`Authentication`对象中。

综上,我们一共要定义`SmsAuthenticationFitler`、`SmsAuthenticationToken`和`SmsAuthenticationProvider`，并将这些组建组合起来添加到Spring Security中.

**SmsAuthenticationToken**

参照`UsernamePasswordAuthenticationToken`的源码

```java
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

```java
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

```java
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

```java
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

```java
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

```java
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
        http.addFilterBefore(smsCodeFilter, UsernamePasswordAuthenticationFilter.class) // 添加短信校验filter
                .formLogin() 
                //...
                .apply(smsAuthenticationConfig); // 将短信验证码认证配置加到 Spring Security 中
    }
}
```

### Session管理

通过`server.session.timeout`可以给session设置超时时间, 单位为秒, 最小值为60.

在Spring Security中指定session失效跳转url并配置Session管理器：

```java
@Override
protected void configure(HttpSecurity http) throws Exception {
    http.formLogin() 
       .authorizeRequests()
       .antMatchers("/authentication/require","/login.html", "/session/invalid").permitAll() 
       .anyRequest().authenticated() 
       .and()
       .sessionManagement() // 添加 Session管理器
       .invalidSessionUrl("/session/invalid") // Session失效后跳转到这个链接
       //......
}
```

**`SessionRegistry`**包含了一些使用的操作Session的方法，比如：

1. 踢出用户（让Session失效）：

   ```
   String currentSessionId = request.getRequestedSessionId();
   sessionRegistry.getSessionInformation(sessionId).expireNow();
   ```

2. 获取所有Session信息：

   ```
   List<Object> principals = sessionRegistry.getAllPrincipals();
   ```

**Session共享**实现:

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

或直接引入整合

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

之后启动多个应用即可.

#### 并发控制

**后者将前者踢出**

自定义session过期策略:

```java
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

配置最大session并发数量为1, 对于统一账号, 如果存在多重登录, 前面登录的session将自动失效(FIFO), 触发自定义session过期策略:

```java
@Autowired
private MySessionExpiredStrategy sessionExpiredStrategy;

@Override
protected void configure(HttpSecurity http) throws Exception {
     http.formLogin() 
       //...
       .and()
       .sessionManagement()
       .invalidSessionUrl("/session/invalid")
       .maximumSessions(1)	// 最大session并发数
       .expiredSessionStrategy(sessionExpiredStrategy)	// session过期策略
       //.....
}
```

**不再允许相同的账户登录**

要实现这个功能只需要在上面的配置中添加：

```java
.and()
.sessionManagement()
.invalidSessionUrl("/session/invalid") 
.maximumSessions(1)
.maxSessionsPreventsLogin(true)		// added
.expiredSessionStrategy(sessionExpiredStrategy)
.and()
```

此时多浏览器登录将受限.

### 退出登录

Spring Security默认的退出登录URL为`/logout`，退出登录后，Spring Security会做如下处理：

1. 使当前的Sesion失效；
2. 清除与当前用户关联的RememberMe记录；
3. 清空当前的SecurityContext；
4. 重定向到登录页。

自定义退出登录行为:

```java
.and()
    .logout()
    .logoutUrl("/signout")
    .logoutSuccessUrl("/signout/success")	//需放行
    .deleteCookies("JSESSIONID")
.and()
```

可以通过`logoutSuccessHandler`指定退出成功处理器来处理退出成功后的逻辑：

```java
@Autowired
private MyLogOutSuccessHandler logOutSuccessHandler;

.and()
    .logout()
    .logoutUrl("/signout")
    // .logoutSuccessUrl("/signout/success")		//与logoutSuccessHandler互斥
    .logoutSuccessHandler(logOutSuccessHandler)
    .deleteCookies("JSESSIONID")
.and()
```

`MyLogOutSuccessHandler`实现`LogoutSuccessHandler`：

```java
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

### 权限控制

Spring Security权限控制可以配合授权注解使用, 只需要加入@EnableGlobalMethodSecurity注解：

```java
@Configuration
@EnableGlobalMethodSecurity(prePostEnabled = true)
public class BrowserSecurityConfig extends WebSecurityConfigurerAdapter {
   ...
}
```

使用权限注解:

```java
@GetMapping("/auth/admin")
@PreAuthorize("hasAuthority('admin')")
public String authenticationTest() {
    return "您拥有admin权限，可以查看";
}
```

如果用户没有admin权限(UserDetail对象中), 则默认将403, 可以自定义权限认证失败处理器,  实现`AccessDeniedHandler`接口即可：

```java
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

添加配置: 

```java
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

#### 权限注解

Spring Security提供了三种不同的安全注解：

1.Spring Security自带的`@Secured`注解

在Spring-Security.xml中启用@Secured注解：

```xml
<global-method-security secured-annotations="enabled"/>
```

或者

```java
@Configuration
@EnableGlobalMethodSecurity(securedEnabled = true)
public class WebSecurityConfig extends WebSecurityConfigurerAdapter {
}
```

使用:

```java
@Secured("ROLE_ADMIN") //只有拥有权限“ROLE_ADMIN”的用户才能访问
public void test(){
    ...
}
```

权限不足时，方法抛出Access Denied异常。

@Secured注解会使用一个String数组作为参数。每个String值是一个权限，调用这个方法至少需要具备其中的一个权限。如：

```java
@Secured({"ROLE_ADMIN","ROLE_USER"})
public void test(){
    ...
}
```

2.JSR-250的`@RolesAllowed`注解

@RolesAllowed注解和@Secured注解在各个方面基本上都是一致的。启用@RolesAllowed注解：

```xml
<global-method-security jsr250-annotations="enabled"/>
```

或者

```java
@Configuration
@EnableGlobalMethodSecurity(jsr250Enabled = true)
public class WebSecurityConfigurer extends WebSecurityConfigurerAdapter {
}
```

使用:

```java
@RolesAllowed("ROLE_ADMIN")
public void test(){
    ...
}
```

3.表达式(SpEL)驱动的注解，包括@PreAuthorize、@PostAuthorize、@PreFilter和 @PostFilter:

启用该注解：

```xml
<global-method-security pre-post-annotations="enabled"/>
```

或者

```java
@Configuration
@EnableGlobalMethodSecurity(prePostEnabled = true)
public class WebSecurityConfigurer extends WebSecurityConfigurerAdapter {
}
```

@**PreAuthorize**

该注解用于方法前验证权限，比如限制非VIP用户提交blog的note字段字数不得超过1000字：

```java
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

接口都需要给超级管理员放行，而使用 `hasAnyRole('admin','user:list')` 每次都需要重复的添加 admin 权限，因此在可自定义权限验证方式，在验证的时候默认给拥有admin权限的用户放行。

```java
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

### 获取认证对象

获取当前认证Authentication对象,可通过SecurityContextHolder.

通过认证对象, 可获取登录用户信息Principal, 需要强转为实现了UserDetails的用户类

```
SecurityContextHolder.getContext().getAuthentication().getPrincipal();
```

还可以直接在需要获取认证对象的controller方法添加Authentication类型参数即可获取.

### 使用JWT

思路为, 在`UsernamePasswordAuthenticationFilter`前添加一个自定义过滤器, 对每次请求尝试获取jwt解析成用户对象, 如果解析成功, 则可以根据用户对象创建一个`UsernamePasswordAuthenticationToken`对象,  然后传给`SecurityContextHolder.getContext().setAuthentication(authentication)`的参数即可完成认证.

在首次登录时因为没有jwt, 会经`UsernamePasswordAuthenticationFilter`创建一个jwt, 并保存到前端header中.

另外, 如果使用jwt，因为不需要session, 可以将session创建策略修改为无状态, 并且可以禁用CSRF保护.

```
.csrf().disable()
.sessionManagement().sessionCreationPolicy(SessionCreationPolicy.STATELESS).and()
```

或者配置文件配置session策略:

```html
security.sessions: stateless
```

### 动态权限控制

通过`FilterInvocationSecurityMetadataSource`实现动态权限控制 , 该类的主要功能就是通过当前的请求地址，获取该地址需要的用户权限。

```java
@Component
public class CustomFilterInvocationSecurityMetadataSource implements FilterInvocationSecurityMetadataSource {
    @Autowired
    MenuService menuService;
    // AntPathMatcher 是一个正则匹配工具
    AntPathMatcher antPathMatcher = new AntPathMatcher();
    //根据用户传来的请求地址获取该地址需要的权限, 封装成集合返回
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

还需要实现`AccessDecisionManager`判断当前用户是否对当前url具备指定的权限 , 如果不具备，就抛出 AccessDeniedException 异常，否则不做任何事即可.

```java
@Component
public class CustomUrlDecisionManager implements AccessDecisionManager {
    @Override
    public void decide(Authentication authentication, Object o, Collection<ConfigAttribute> collection) throws AccessDeniedException, InsufficientAuthenticationException {
        for (ConfigAttribute configAttribute : collection) {
            String needRole = configAttribute.getAttribute();
            //如果需要的权限是ROLE_LOGIN，说明当前请求的URL用户登陆后即可访问
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

#### 基于SpEL方式

类似自定义注解鉴权, 注册一个包含返回值为布尔的公共方法: 方法具有两个参数 `Authentication`和`HttpServletRequest` ,目的在于拿着当前请求去和当前认证信息中包含的权限进行访问控制判断.

然后按照`@bean名称.方法名(authentication,request)`的格式配置到`HttpSecurity`对象中.

```
httpSecurity.authorizeRequests()
	.anyRequest()
	.access("@roleChecker.check(authentication,request)");
```

#### AuthorizationManager

5.6版本增加的泛型接口, 它用来检查当前认证信息是否可以访问特定对象T.

> 对比SpEL方式 ,`RoleChecker`不就是`AuthorizationManager<HttpServletRequest>`?`AuthorizationManager`就是将这种访问决策抽象的更加泛化。

```java
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

```java
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

## Spring Security OAuth2

Spring Security OAuth2主要包含认证服务器和资源服务器这两大块的实现：

认证服务器主要包含了四种授权模式的实现和Token的生成与存储，可以在认证服务器中自定义获取Token的方式

资源服务器主要是在Spring Security的过滤器链上加了OAuth2AuthenticationProcessingFilter过滤器，即使用OAuth2协议发放令牌认证的方式来保护我们的资源。

**配置认证服务器**

新建Spring Boot项目, 关键依赖为:

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-oauth2</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-security</artifactId>
</dependency>
```

同[自定义认证过程](#自定义认证过程), 创建用户类用于保存从数据库获取, 实现UserDetailService.

在Spring Security的配置类上添加`@EnableAuthorizationServer`注解标即可实现认证服务器:

```java
@Configuration
@EnableAuthorizationServer
public class AuthorizationServerConfig extends WebSecurityConfigurerAdapter {
}
```

启动项目后，控制台将打印出随机分配的client-id和client-secret.

可以手动指定这两个值

```yml
security:
  oauth2:
    client:
      client-id: test
      client-secret: test1234
```

对授权模式进行落地:

1.授权码模式获取令牌

浏览器访问, 其中response_type固定为code, client_id为认证服务器配置的client-id, redirect_uri为通过认证后的重定向地址:

```
http://localhost:8080/oauth/authorize?response_type=code&client_id=test&redirect_uri=http://www.xxx.com&scope=all&state=hello
```

将到达认证服务器登录页面, 目前输入正确的username和password仍不能通过认证, 会进行错误提示"At least one redirect_uri must be registered with the client", 提示已经很明显了, 需要在clinet中注册redirect_url:

```yml
security:
  oauth2:
    client:
      client-id: test
      client-secret: test1234
      registered-redirect-uri: http://www.xxx.com
```

继续登录, 认证成功后会提示是否进行授权页面, 允许(Approve)拒绝(Deny), 若选择Approve, 认证服务器将重定向到redirect_url指定的URL上, 并在请求参数中附带授权码参数. 

利用授权码, 可以从认证服务器获取令牌.

利用工具发送POST请求, 其中grant_type固定为authorization_code, code为上一步获取到的授权码, redirect_url和client_id与上一步一样:

```
http://localhost:8080/oauth/token?
grant_type=authorization_code&code=xxx&client=test&redirect_id=http://www.xxx.com&scope=all
```

并在**请求头**添加`Authorization=Basic  <client_id:client_secret进行base64加密后的值>`, 对于test:test1234, 加密后值为dGVzdDp0ZXN0MTlzNA= =, 因此, 请求头为Authorization=Basic dGVzdDp0ZXN0MTlzNA= =(可在http://tool.chinaz.com/Tools/Base64.aspx获取).

发送成功, 即可获取令牌, 包括access_token和refresh_token.

```
{
    "access_token": "950018df-0199-4936-aa80-a3a66183f634",
    "token_type": "bearer",
    "refresh_token": "cc22e8b2-e069-459d-8c24-cfda0bc72128",
    "expires_in": 42827,
    "scope": "all"
}
```

>  一个授权码只能请求一次令牌, 如多次请求, 将返回invalid_grant错误

2.密码模式获取令牌

使用发送POST请求, 其中grant_type固定为password表示直接使用用户名和密码获取令牌：

```?
http://localhost:8080/oauth/token?
grant_type=password&username=admin&password=123456&scope=all
```

并在请求头添加与授权码模式同样的内容, 发送同样可获取令牌

**配置资源服务器**

资源服务器除了进行SpringSecurity正常配置之外, 还需要对校验令牌的认证服务器TokenService进行配置:

```java
@Configuration
@EnableResourceServer
public class ResourceServerConfig  {
    @Override
    public void configure(ResourceServerSecurityConfigurer resources) {
        resources.tokenServices(tokenServices());//.resourceId(resourceId);
    }

    @Bean
    public ResourceServerTokenServices tokenServices() {
        RemoteTokenServices remoteTokenServices = new RemoteTokenServices();
        remoteTokenServices.setCheckTokenEndpointUrl("http://资源服务器Ip:端口/oauth/check_token");
        remoteTokenServices.setClientId("test");
        remoteTokenServices.setClientSecret("test1234");
        return remoteTokenServices;
    }
}
```

根据上一步获取的令牌 , 访问资源服务器,  在请求头添加`Authorization=bearer <access_token>`

>  如果同时设置认证服务器和资源服务器,  可省略tokenService配置仅添加注解即可,  但使用授权码模式获取令牌可能会遇到 Full authentication is required to access this resource错误，可认证服务器的配置类上使用`@Order(1)`标注，在资源服务器的配置类上使用`@Order(2)`标注。

### 手动实现Token获取

模拟Spring Security OAuth2资源服务器获取令牌流程, 核心类为ClientDetailsService和AuthorizationServerTokenServices,

```java
@Configuration
@EnableResourceServer
public class ResourceServerConfig extends ResourceServerConfigurerAdapter {
    @Autowired
    private MyAuthenticationSucessHandler authenticationSucessHandler;
    @Override
    public void configure(HttpSecurity http) throws Exception {
        http.formLogin()
                .loginProcessingUrl("/login")	//配置登录页, 用于认证后获取令牌
                .successHandler(authenticationSucessHandler) 
            .and()
                .authorizeRequests() 
                .anyRequest()
                .authenticated()
            .and()
                .csrf().disable();
    }
}
```

登录成功处理器`MyAuthenticationSucessHandler`里生成令牌并返回：

```java
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
		// base64解码
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

之后请求资源服务器登录的登录url并附带Basic请求头携带base64后的client_id和client_secret, 登录成功后

点击发送后便可以成功获取到令牌：

```
{
    "access_token": "88a3dd6c-ab27-41af-95ee-5cd406fe5ab1",
    "token_type": "bearer",
    "refresh_token": "b316177d-68e9-4fc9-9f4a-804a7367ebc9",
    "expires_in": 43199
}
```

### 验证码获取令牌

因为使用令牌的方式和系统交互,  Session已经不适用了，所以需要使用第三方存储来保存我们的验证码.

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis</artifactId>
</dependency>
```

在Spring Security短信验证码登录和上面手动获取令牌的基础上,定义一个`RedisCodeService`，用于验证码的增删改：

```java
@Service
public class RedisCodeService {
    private final static String SMS_CODE_PREFIX = "SMS_CODE:";
    private final static Integer TIME_OUT = 300;

    @Autowired
    private StringRedisTemplate redisTemplate;
    /**
     * 保存验证码到 redis
     */
    public void save(SmsCode smsCode, ServletWebRequest request, String mobile) throws Exception {
        redisTemplate.opsForValue().set(key(request, mobile), smsCode.getCode(), TIME_OUT, TimeUnit.SECONDS);
    }
    /**
     * 获取验证码
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
	// 根据设备id和手机号构造key
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

```java
@GetMapping("/code/sms")
public void createSmsCode(HttpServletRequest request, HttpServletResponse response, String mobile) throws Exception {
    SmsCode smsCode = createSMSCode();
    redisCodeService.save(smsCode, new ServletWebRequest(request), mobile);
    // 输出验证码到控制台代替短信发送服务
    System.out.println("手机号" + mobile + "的登录验证码为：" + smsCode.getCode() + "，有效时间为120秒");
}
```

在SmsCodeFilter的校验验证码方法处修改为从redis处理:

```java
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

通过`http://资源服务器ip:port/code/sms?mobile=xxxxxx`获取验证码, 请求头中带上deviceId（模拟）：

接着拿验证码去换取令牌:  访问资源服务器验证码登录url, 附带mobile参数和输入的验证码, 并附带Basic请求头携带base64后的client_id和client_secret和deviceId.

### 自定义令牌配置

Spring Security允许自定义令牌配置，如不同的client_id对应不同的令牌，令牌的有效时间，令牌的存储策略等:

```java
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
                .withClient("test1") 	//不同的client_id来获取不同的令牌
                .secret("test1111")	
                .accessTokenValiditySeconds(3600)	//令牌有效时间
                .refreshTokenValiditySeconds(864000)//刷新令牌有效时间
                .scopes("all", "a", "b", "c")		//限定scope
                .authorizedGrantTypes("password")	//限定授权模式
            .and()
                .withClient("test2")
                .secret("test2222")
                .accessTokenValiditySeconds(7200);
    }
}
```

创建一个新的配置类`SecurityConfig`，在里面注册上面需要的`AuthenticationManager`Bean：

```java
@Configuration
public class SecurityConfig extends WebSecurityConfigurerAdapter {
    @Bean(name = BeanIds.AUTHENTICATION_MANAGER)
    @Override
    public AuthenticationManager authenticationManagerBean() throws Exception {
        return super.authenticationManagerBean();
    }
}
```

启动后使用密码模式获取令牌, 根据需要获取令牌的client配置不同的请求头:

```
http://localhost:8080/oauth/token?
grant_type=password&username=test&password=123456&scope=all
```

默认令牌是存储在内存中的, 可更改为保存到Redis:

```java
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

在认证服务器里指定该令牌存储策略：

```java
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

### 使用JWT替换默认令牌

默认令牌为UUID, 可使用JWT替换默认的令牌,  注入JwtTokenStore

```java
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

签名密钥为`test_key`, 即sign, 用来生成和解析jwt, , 在认证服务器里指定tokenStore：

```java
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

启动服务获取令牌，系统将返回如下格式令牌：

```
{
    "access_token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJleHAiOjE1NjE1MzI1MDEsInVzZXJfbmFtZSI6Im1yYmlyZCIsImF1dGhvcml0aWVzIjpbImFkbWluIl0sImp0aSI6IjJkZjY4MGNhLWFmN2QtNGU4Ni05OTdhLWI1ZmVkYzQxZmYwZSIsImNsaWVudF9pZCI6InRlc3QxIiwic2NvcGUiOltdfQ.dZ4SeuU3VWnSJKy5vELGQ0YkVRddcEydUlJAVovlycg",
    "token_type": "bearer",
    "expires_in": 3599,
    "scope": "all",
    "jti": "2df680ca-af7d-4e86-997a-b5fedc41ff0e"
}
```

将`access_token`中的内容复制到https://jwt.io/网站解析下, 可看到jwt中包含的用户认证信息.

**增加额外信息**

如果想在JWT中添加一些额外的信息，我们需要实现`TokenEnhancer`（Token增强器）：

```java
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

```java
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

```java
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

### 刷新令牌

令牌过期后可以使用refresh_token来从系统中换取一个新的可用令牌。要让系统返回refresh_token，需要在认证服务器自定义配置里添加如下配置：

```java
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

假设access_token过期了，可用refresh_token去换取新的令牌:

```
http://localhost:8080/oauth/token?
grant_type=refresh_token&refresh_token=xxxx
```

>  刷新令牌同样需要指定client的请求头

### 单点登录SSO

创建一个maven多模块项目，包含认证服务器和两个客户端, 依赖oauth2即可.

**认证服务器配置**

认证配置, 省略UserService等重复代码.

```java
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

认证服务器配置:

```java
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

    //分配两个客户端,
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

为了方便区分, 配置contextpath;

```
server:
  port: 8080
  servlet:
    context-path: /server
```

**客户端配置**

两个客户端的代码基本一致, 在客户端SpringBoot入口类上添加`@EnableOAuth2Sso`注解，开启SSO的支持.

重点是配置文件application.yml的配置：

```yml
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

```html
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

```java
@RestController
public class UserController {
    @GetMapping("user")
    public Principal user(Principal principal) {
        return principal;
    }
}
```

登录后，页面跳转到了授权页面,  点击Authorize ,到了app1的index,   其中包含了可以跳转到app2的index链接 ,  点击跳转,   结果为页面直接来到授权页，而不需要重新输入用户名密码，继续点击Authorize.

app2也已经成功登录，访问http://127.0.0.1:9091/app2/user看是否能成功获取到用户信息

### 自动授权

在上面过程中需要用户点击Authorize授权，体验并不是很好，我们可以去掉它。修改认证服务器Client配置如下：

```java
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

## Shiro

在Spring Boot中集成Shiro进行用户的认证过程主要可以归纳为以下三点：

1、定义一个ShiroConfig，然后配置SecurityManager Bean，SecurityManager为Shiro的安全管理器，管理着所有Subject；

2、在ShiroConfig中配置ShiroFilterFactoryBean，其为Shiro过滤器工厂类，依赖于SecurityManager；

3、自定义Realm实现，Realm包含`doGetAuthorizationInfo()`和`doGetAuthenticationInfo()`方法.

```xml
 <dependency>
    <groupId>org.apache.shiro</groupId>
    <artifactId>shiro-spring-boot-web-starter</artifactId>
    <version>${shiro.spring.version}</version>
 </dependency>
```

Shiro filterChain基于短路机制，即最先匹配原则，如：

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

### 认证

```java
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

realm

```java
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

```java
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

### RemenberMe

在上面的基础上修改ShiroFilterFactoryBean,SecurityManager,加入 SimpleCookie ,CookieRememberMeManager

```java
@Configuration
public class ShiroConfig {
   @Bean
   public ShiroFilterFactoryBean shiroFilterFactoryBean(SecurityManager securityManager) {
      //...
      //修改权限配置，对filterChainDefinitionMap.put("/**", "authc")进行替换
      //user指的是用户认证通过或者配置了Remember Me记住用户登录状态后可访问。
      filterChainDefinitionMap.put("/**", "user");
      //...
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
   
   // cookie对象
   public SimpleCookie rememberMeCookie() {
      // 设置cookie名称，对应login.html页面的<input type="checkbox" name="rememberMe"/>
      SimpleCookie cookie = new SimpleCookie("rememberMe");
      // 设置cookie的过期时间，单位为秒，这里为一天
      cookie.setMaxAge(86400);
      return cookie;
   }
   
   public CookieRememberMeManager rememberMeManager() {
      CookieRememberMeManager cookieRememberMeManager = new CookieRememberMeManager();
      cookieRememberMeManager.setCookie(rememberMeCookie());
      // rememberMe cookie加密的密钥 
     cookieRememberMeManager.setCipherKey(Base64.decode("3AvVhmFLUs0KTA3Kprsdag=="));
      return cookieRememberMeManager;
   }
}
```

修改登录接口, 加入remenberMe:

```java
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
```

### 鉴权

@RequiresAuthentication：表示当前Subject已经通过login进行了身份验证；即 Subject.isAuthenticated()返回true
@RequiresUser：表示当前 Subject 已经身份验证或者通过记住我登录的。
@RequiresGuest：表示当前Subject没有身份验证或通过记住我登录过，即是游客身份。
@RequiresRoles(value={“admin”, “user”}, logical=Logical.AND)：表示当前 Subject 需要角色 admin 和user
@RequiresPermissions (value={“user:a”, “user:b”},logical= Logical.OR)：表示当前 Subject 需要权限 user:a 或user:b。

使用注解鉴权需要在配置类中注入`AuthorizationAttributeSourceAdvisor`

```java
@Bean
public AuthorizationAttributeSourceAdvisor authorizationAttributeSourceAdvisor(SecurityManager securityManager) {
    AuthorizationAttributeSourceAdvisor authorizationAttributeSourceAdvisor = new AuthorizationAttributeSourceAdvisor();
    authorizationAttributeSourceAdvisor.setSecurityManager(securityManager);
    return authorizationAttributeSourceAdvisor;
}
```

通过realm的doGetAuthorizationInfo返回的角色与权限进行校验:

```java
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

### Shiro-Redis

> 与Cache类似, 关键为创建不同的CacheManager

开源项目https://github.com/alexxiyang/shiro-redis/

该框架给我们实现了RedisManager,RedisCache(实现了Cache接口)以及RedisCacheManager(实现了CacheManager接口)

拓展:我们自己也可以按照此方法实现(novel-plus:admin),https://blog.csdn.net/qq_34021712/article/details/80791219,

```xml
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

```java
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

```java
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

### Shiro+Ehcache

> 与Cache类似, 关键为创建不同的CacheManager

```xml
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

shiro-ehcache.xml, 具体配置可参见持久层Cache一节.

也可以通过配置文件指定xml

```yml
 spring:
   cache:
	 ehcache:
       config: classpath:ehcache.xml
```

在Authrization例修改如下

```java
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
```

### Thymeleaf

标签官方http://shiro.apache.org/web.html#Web-JSP%252FGSPTagLibrary

```xml
<dependency>
	<groupId>com.github.theborakompanioni</groupId>
	<artifactId>thymeleaf-extras-shiro</artifactId>
	<version>2.0.0</version>
</dependency>
```

```xml
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

```java
@Bean
public ShiroDialect shiroDialect() {
    return new ShiroDialect();
}
```

### SessionDao

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

为了能够在Spring Boot中使用`SessionDao`，我们在ShiroConfig中配置该Bean(EhCache):

```java
@Configuration
public class ShiroConfig {
    //...
	@Bean
	public SessionDAO sessionDAO() {
		MemorySessionDAO sessionDAO = new MemorySessionDAO();
		return sessionDAO;
	}
	//SessionDao通过org.apache.shiro.session.mgt.SessionManager进行管理
	@Bean
	public SessionManager sessionManager() {
		DefaultWebSessionManager sessionManager = new DefaultWebSessionManager();
		Collection<SessionListener> listeners = new ArrayList<SessionListener>();
		listeners.add(new ShiroSessionListener());	//添加sessionListener
		sessionManager.setSessionListeners(listeners);
		sessionManager.setSessionDAO(sessionDAO());   //设置sessionDao
		return sessionManager;
	}
}
```

如果想自己实现sessionDao,可以通过实现` AbstractSessionDAO`.

如果使用的是Shiro-Redis作为缓存实现，那么SessionDAO则为`RedisSessionDAO`:

```java
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

使用SessionDao:

```java
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

通过SessionDao的`getActiveSessions()`方法，我们可以获取所有有效的Session，通过Session，还可以获取到当前用户的Principal信息。

当某个用户被踢出后（Session Time置为0），该Session并不会立刻从ActiveSessions中剔除，所以通过其timeout信息来判断该用户在线与否。

如果使用的Redis作为缓存实现，那么，`forceLogout()`方法需要稍作修改：

```java
@Override
public boolean forceLogout(String sessionId) {
    Session session = sessionDAO.readSession(sessionId);
    sessionDAO.delete(session);
    return true;
}
```

### 实现shiro+redis

RedisManager

```java
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

```java
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

```java
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

```java
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

```java
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
