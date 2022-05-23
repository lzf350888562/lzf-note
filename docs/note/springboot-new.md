# 其他注解编程

## 请求参数校验

Bean Validation是Java定义的一套基于注解的数据校验规范，但没有实现，比如@Null、@NotNull、@Pattern等，它们位于 javax.validation.constraints这个包下。而hibernate validator是对这个规范的实现，并增加了一些其他校验注解，如 @NotBlank、@NotEmpty、@Length等，它们位于org.hibernate.validator.constraints这个包下。

```xml
<!--spring-boot 项目已经集成在starter-web中-->
<dependency>
    <groupId>org.hibernate.validator</groupId>
    <artifactId>hibernate-validator</artifactId>
    <version>6.0.8.Final</version>
</dependency>
```

1.单参数校验

该方式需要在controller类上添加@Validated注解

```java
public Result deleteUser(@NotNull(message = "id不能为空") Long id) {
  // do something
}
```

或者

```java
@NotBlank String username = user.getUsername();
```

2.对象参数校验

对象参数校验使用时，需要先在对象的校验属性(或gettor)上添加注解，然后在Controller方法的对象参数前添加@Validated 注解

```java
@PostMapping
public AjaxResult add(@Validated @RequestBody SysDept dept){

}

public class SysDept{
    @NotBlank(message = "部门名称不能为空")
    @Size(min = 0, max = 30, message = "部门名称长度不能超过30个字符")
    private String deptName;
    //或者
    @NotBlank(message = "部门名称不能为空")
    @Size(min = 0, max = 30, message = "部门名称长度不能超过30个字符")
    public String getDeptName(){
        return deptName;
    }
}
```

| 注解名称                       | 功能                                       |
| -------------------------- | ---------------------------------------- |
| @Null                      | 检查该字段为空                                  |
| @NotNull                   | 不能为null                                  |
| @NotBlank                  | 不能为空，常用于检查空字符串                           |
| @NotEmpty                  | 不能为空，多用于检测list是否size是0                   |
| @Max                       | 该字段的值只能小于或等于该值                           |
| @Min                       | 该字段的值只能大于或等于该值                           |
| @Past                      | 检查该字段的日期是在过去                             |
| @Future                    | 检查该字段的日期是否是属于将来的日期                       |
| @Email                     | 检查是否是一个有效的email地址                        |
| @Pattern(regex=,flag=)     | 被注释的元素必须符合指定的正则表达式                       |
| @Range(min=,max=,message=) | 被注释的元素必须在合适的范围内                          |
| @Size(min=, max=)          | 检查该字段的size是否在min和max之间，可以是字符串、数组、集合、Map等 |
| @Length(min=,max=)         | 检查所属的字段的长度是否在min和max之间,只能用于字符串           |
| @AssertTrue                | 用于boolean字段，该字段只能为true                   |
| @AssertFalse               | 该字段的值只能为false                            |
| @Future                    | 该字段的值必须时一个将来的日期                          |

> 当输入不能满足条件是，就会抛出异常，通常由统一由异常中心处理.

### 注解分组

当同一个参数对象在不同的场景下有不同的校验规则时

比如，在创建对象时不需要传入id字段，但是在修改对象时就必须要传入id字段。在这样的场景下就需要对注解进行分组。

1）组件有个默认分组Default.class, 所以我们可以再创建一个分组UpdateAction：

```java
public interface UpdateAction {
}
```

2）在参数类中需要校验的属性上，在注解中添加groups属性：

```java
public class UserAO {
    @NotNull(groups = UpdateAction.class, message = "id不能为空")
    private Long id;

    @NotBlank
    private String name;
    @NotNull
    private Integer age;
    ……
}
```

如上所示，就表示只在UpdateAction分组下校验id字段，在默认情况下就会校验name字段和age字段。

然后在controller的方法中，在@Validated注解里指定哪种场景即可，没有指定就代表采用Default.class，采用其他分组就需要显示指定,可同时指定多个分组

```java
public Result addUser(@Validated UserAO userAo) {
  // do something
}
public Result updateUser(@Validated({Default.class, UpdateAction.class}) UserAO userAo) {
  // do something
}
```

### 对象嵌套

如果需要校验的参数对象中还嵌套有一个对象属性，而该嵌套的对象属性也需要校验，那么就需要在该对象属性上增加@Valid注解。

```java
public class UserAO {
    @NotNull(groups = UpdateAction.class, message = "id不能为空")
    private Long id;  
    @NotBlank
    private String name;
    @NotNull
    private Integer age;

    @Valid
    private Phone phone;
}

public class Phone {
    @NotBlank
    private String operatorType;
    @NotBlank
    private String phoneNum;
}
```

### 统一处理参数校验异常

1.单参数校验失败 ConstraintViolationException:对应校验注解写在方法参数上面(方法所在类需要加@Validated)的情况:

```java
@RestControllerAdvice
@Order(value = Ordered.HIGHEST_PRECEDENCE)
public class GlobalExceptionHandler {
    @ExceptionHandler(value = ConstraintViolationException.class)
    @ResponseStatus(HttpStatus.BAD_REQUEST)
    public String handleConstraintViolationException(ConstraintViolationException e) {
        StringBuilder message = new StringBuilder();
        Set<ConstraintViolation<?>> violations = e.getConstraintViolations();
        for (ConstraintViolation<?> violation : violations) {
            Path path = violation.getPropertyPath();
            //获取校验不通过的参数名称
            String[] pathArr = StringUtils.splitByWholeSeparatorPreserveAllTokens(path.toString(), ".");
            //拼接上提示信息
            message.append(pathArr[1]).append(violation.getMessage()).append(",");
        }
        message = new StringBuilder(message.substring(0, message.length() - 1));
        return message.toString();
    }
```

2.get请求的对象参数校验失败后抛出的异常是BindException:

```java
@ExceptionHandler(BindException.class)
@ResponseStatus(HttpStatus.BAD_REQUEST)
public String validExceptionHandler(BindException e) {
    StringBuilder message = new StringBuilder();
    List<FieldError> fieldErrors = e.getBindingResult().getFieldErrors();
    for (FieldError error : fieldErrors) {
        message.append(error.getField()).append(error.getDefaultMessage()).append(",");
    }
    message = new StringBuilder(message.substring(0, message.length() - 1));
    return message.toString();

}
```

3.post请求的对象参数校验失败后抛出的异常是MethodArgumentNotValidException

```java
if (e instanceof MethodArgumentNotValidException){
      // post请求的对象参数校验异常
      Result result = Result.buildErrorResult(ErrorCodeEnum.PARAM_ILLEGAL);
      List<ObjectError> errors = ((MethodArgumentNotValidException) e).getBindingResult().getAllErrors();
      String msg = getValidExceptionMsg(errors);
      if (StringUtils.isNotBlank(msg)){
        result.setMessage(msg);
      }
      return result;
}
```

4.缺少参数抛出的异常是MissingServletRequestParameterException

```java
e.getParameterName()  //获取缺少的参数名
```

## @Async

`@Async`注解就能将原来的同步函数变为异步函数,

为了让@Async注解能够生效，还需要配置@EnableAsync

```java
@Component
public class Task {
    public static Random random =new Random();
    @Async
    public void doTaskOne() throws Exception {
        System.out.println("开始做任务一");
        long start = System.currentTimeMillis();
        Thread.sleep(random.nextInt(10000));
        long end = System.currentTimeMillis();
        System.out.println("完成任务一，耗时：" + (end - start) + "毫秒");
    }
    @Async
    public void doTaskTwo() throws Exception {
        System.out.println("开始做任务二");
        long start = System.currentTimeMillis();
        Thread.sleep(random.nextInt(10000));
        long end = System.currentTimeMillis();
        System.out.println("完成任务二，耗时：" + (end - start) + "毫秒");
    }
}
```

```java
@SpringBootApplication
@EnableAsync
public class Application {
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}

@RunWith(SpringJUnit4ClassRunner.class)
@SpringApplicationConfiguration(classes = Application.class)
public class ApplicationTests {
    @Autowired
    private Task task;
    @Test
    public void test() throws Exception {
        task.doTaskOne();
        task.doTaskTwo();
    }
}
```

此时反复执行单元测试会遇到各种不同的结果，比如：

- 没有任何任务相关的输出
- 有部分任务相关的输出
- 乱序的任务相关的输出

原因是主程序在异步调用三个函数之后并不会理会这三个函数是否执行完成了，由于没有其他需要执行的内容，所以程序就自动结束了，导致了不完整或是没有输出任务相关内容的情况。

> @Async所修饰的函数不要定义为static类型，这样异步调用不会生效

### 异步回调

为了让异步函数能正常结束，可使用`Future<T>`来返回异步调用的结果.

```java
@Component
public class Task {
    public static Random random =new Random();
    @Async
    public Future<String> doTaskOne() throws Exception {
        System.out.println("开始做任务一");
        long start = System.currentTimeMillis();
        Thread.sleep(random.nextInt(10000));
        long end = System.currentTimeMillis();
        System.out.println("完成任务一，耗时：" + (end - start) + "毫秒");
        return new AsyncResult<>("任务一完成");
    }
}
```

```java
@Test
public void test() throws Exception {
    long start = System.currentTimeMillis();

    Future<String> task1 = task.doTaskOne();
    Future<String> task2 = task.doTaskTwo();

    while(true) {
        if(task1.isDone() && task2.isDone()) {
            // 二个任务都调用完成，退出循环等待
            break;
        }
        Thread.sleep(1000);
    }
    long end = System.currentTimeMillis();
    System.out.println("任务全部完成，总耗时：" + (end - start) + "毫秒");
}
```

### 配置默认线程池

异步任务存在的问题

```java
@RestController
public class HelloController {
    @Autowired
    private AsyncTasks asyncTasks;

    @GetMapping("/hello")
    public String hello() {
        // 将可以并行的处理逻辑，拆分成三个异步任务同时执行
        CompletableFuture<String> task1 = asyncTasks.doTaskOne();
        CompletableFuture<String> task2 = asyncTasks.doTaskTwo();
        CompletableFuture<String> task3 = asyncTasks.doTaskThree();

        CompletableFuture.allOf(task1, task2, task3).join();
        return "Hello World";
    }
}
```

多次调用接口会频繁创建线程执行可能会出现内存溢出。

因为Spring Boot默认用于异步任务的线程池是这样配置的：

```java
public static class Pool{
    private int queueCapacity = Integer.MAX_VALUE; //缓冲队列的容量
    private int maxSize = Integer.MAX_VALUE; //允许的最大线程数
    //...
}
```

配置默认线程池:

```properties
spring.task.execution.pool.core-size=2
spring.task.execution.pool.max-size=5
spring.task.execution.pool.queue-capacity=10
spring.task.execution.pool.keep-alive=60s
spring.task.execution.pool.allow-core-thread-timeout=true
spring.task.execution.thread-name-prefix=task-
```

参数解释

- `spring.task.execution.pool.core-size`：线程池创建时的初始化线程数，默认为8
- `spring.task.execution.pool.max-size`：线程池的最大线程数，默认为int最大值
- `spring.task.execution.pool.queue-capacity`：用来缓冲执行任务的队列，默认为int最大值
- `spring.task.execution.pool.keep-alive`：线程终止前允许保持空闲的时间
- `spring.task.execution.pool.allow-core-thread-timeout`：是否允许核心线程超时
- `spring.task.execution.shutdown.await-termination`：是否等待剩余任务完成后才关闭应用
- `spring.task.execution.shutdown.await-termination-period`：等待剩余任务完成的最大时间
- `spring.task.execution.thread-name-prefix`：线程名的前缀，设置好了之后可以方便我们在日志中查看处理任务所在的线程池

### 自定义线程池

由于@Async默认每个任务创建一个线程, 造成资源浪费并可能OOM, 所以可通过自定义线程池的方式:

```java
@SpringBootApplication
public class Application {
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }

    @EnableAsync
    @Configuration
    class TaskPoolConfig {
        @Bean("taskExecutor")
        public Executor taskExecutor() {
            ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
            executor.setCorePoolSize(10);
            executor.setMaxPoolSize(20);
            executor.setQueueCapacity(200);
            executor.setKeepAliveSeconds(60);
            executor.setThreadNamePrefix("taskExecutor-");
            executor.setRejectedExecutionHandler(new ThreadPoolExecutor.CallerRunsPolicy());
            return executor;
        }
    }
}
```

参数:

- `corePoolSize`：线程池核心线程的数量，默认值为1（这就是默认情况下的异步线程池配置使得线程不能被重用的原因）。
- `maxPoolSize`：线程池维护的线程的最大数量，只有当核心线程都被用完并且缓冲队列满后，才会开始申超过请核心线程数的线程，默认值为`Integer.MAX_VALUE`。
- `queueCapacity`：缓冲队列。
- `keepAliveSeconds`：超出核心线程数外的线程在空闲时候的最大存活时间，默认为60秒。
- `threadNamePrefix`：线程名前缀。
- `waitForTasksToCompleteOnShutdown`：是否等待所有线程执行完毕才关闭线程池，默认值为false。
- `awaitTerminationSeconds`：`waitForTasksToCompleteOnShutdown`的等待的时长，默认值为0，即不等待。
- `rejectedExecutionHandler`：当没有线程可以被使用时的处理策略（拒绝任务），默认策略为`abortPolicy`，包含下面四种策略：

```
callerRunsPolicy：用于被拒绝任务的处理程序，它直接在 execute 方法的调用线程中运行被拒绝的任务；如果执行程序已关闭，则会丢弃该任务。
abortPolicy：直接抛出java.util.concurrent.RejectedExecutionException异常。
discardOldestPolicy：当线程池中的数量等于最大线程数时、抛弃线程池中最后一个要执行的任务，并执行新传入的任务。
discardPolicy：当线程池中的数量等于最大线程数时，不做任何动作。
```

在`@Async`注解中指定线程池名让异步调用的执行任务使用这个线程池中的资源来运行:

```java
@Slf4j
@Component
public class Task {
    public static Random random = new Random();
    @Async("taskExecutor")
    public void doTaskOne() throws Exception {
        log.info("开始做任务一");
        long start = System.currentTimeMillis();
        Thread.sleep(random.nextInt(10000));
        long end = System.currentTimeMillis();
        log.info("完成任务一，耗时：" + (end - start) + "毫秒");
    }
    @Async("taskExecutor")
    public void doTaskTwo() throws Exception {
        log.info("开始做任务二");
        long start = System.currentTimeMillis();
        Thread.sleep(random.nextInt(10000));
        long end = System.currentTimeMillis();
        log.info("完成任务二，耗时：" + (end - start) + "毫秒");
    }
}
```

测试: 这里主线程使用join方法等待子线程执行完毕

```java
@RunWith(SpringJUnit4ClassRunner.class)
@SpringBootTest
public class ApplicationTests {
    @Autowired
    private Task task;

    @Test
    public void test() throws Exception {
        task.doTaskOne();
        task.doTaskTwo();

        Thread.currentThread().join();
    }
}
```

### 线程池隔离

以创建多个线程池,通过@Async指定线程池名称使得不同的任务使用不同的线程池.

```java
@EnableAsync
@Configuration
public class TaskPoolConfig {

    @Bean
    public Executor taskExecutor1() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(2);
        executor.setMaxPoolSize(2);
        executor.setQueueCapacity(10);
        executor.setKeepAliveSeconds(60);
        executor.setThreadNamePrefix("executor-1-");
        executor.setRejectedExecutionHandler(new ThreadPoolExecutor.CallerRunsPolicy());
        return executor;
    }

    @Bean
    public Executor taskExecutor2() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(2);
        executor.setMaxPoolSize(2);
        executor.setQueueCapacity(10);
        executor.setKeepAliveSeconds(60);
        executor.setThreadNamePrefix("executor-2-");
        executor.setRejectedExecutionHandler(new ThreadPoolExecutor.CallerRunsPolicy());
        return executor;
    }
}
```

```java
@Slf4j
@Component
public class AsyncTasks {

    public static Random random = new Random();

    @Async("taskExecutor1")
    public CompletableFuture<String> doTaskOne(String taskNo) throws Exception {
        log.info("开始任务：{}", taskNo);
        long start = System.currentTimeMillis();
        Thread.sleep(random.nextInt(10000));
        long end = System.currentTimeMillis();
        log.info("完成任务：{}，耗时：{} 毫秒", taskNo, end - start);
        return CompletableFuture.completedFuture("任务完成");
    }

    @Async("taskExecutor2")
    public CompletableFuture<String> doTaskTwo(String taskNo) throws Exception {
        log.info("开始任务：{}", taskNo);
        long start = System.currentTimeMillis();
        Thread.sleep(random.nextInt(10000));
        long end = System.currentTimeMillis();
        log.info("完成任务：{}，耗时：{} 毫秒", taskNo, end - start);
        return CompletableFuture.completedFuture("任务完成");
    }
}
```

```java
@Slf4j
@SpringBootTest
public class Chapter77ApplicationTests {

    @Autowired
    private AsyncTasks asyncTasks;

    @Test
    public void test() throws Exception {
        long start = System.currentTimeMillis();

        // 线程池1
        CompletableFuture<String> task1 = asyncTasks.doTaskOne("1");
        CompletableFuture<String> task2 = asyncTasks.doTaskOne("2");
        CompletableFuture<String> task3 = asyncTasks.doTaskOne("3");

        // 线程池2
        CompletableFuture<String> task4 = asyncTasks.doTaskTwo("4");
        CompletableFuture<String> task5 = asyncTasks.doTaskTwo("5");
        CompletableFuture<String> task6 = asyncTasks.doTaskTwo("6");

        // 一起执行
        CompletableFuture.allOf(task1, task2, task3, task4, task5, task6).join();

        long end = System.currentTimeMillis();

        log.info("任务全部完成，总耗时：" + (end - start) + "毫秒");
    }
}
```

### 修改默认线程池

> Spring中使用的@Async注解，底层是基于SimpleAsyncTaskExecutor去执行任务，只不过它不是线程池，而是每次都新开一个线程。

实现AsyncConfigurer,  只需要加`@Async`注解就可以，不用在注解属性指定线程池。

```java
@Slf4j
@Configuration
public class NativeAsyncTaskExecutePool implements AsyncConfigurer{

    //注入配置类
    @Autowired
    TaskThreadPoolConfig config;

    @Override
    public Executor getAsyncExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        //核心线程池大小
        executor.setCorePoolSize(config.getCorePoolSize());
        //最大线程数
        executor.setMaxPoolSize(config.getMaxPoolSize());
        //队列容量
        executor.setQueueCapacity(config.getQueueCapacity());
        //活跃时间
        executor.setKeepAliveSeconds(config.getKeepAliveSeconds());
        //线程名字前缀
        executor.setThreadNamePrefix("MyExecutor-");

        // setRejectedExecutionHandler：当pool已经达到max size的时候，如何处理新任务
        // CallerRunsPolicy：不在新线程中执行任务，而是由调用者所在的线程来执行
        executor.setRejectedExecutionHandler(new ThreadPoolExecutor.CallerRunsPolicy());
        executor.initialize();
        return executor;
    }

    /**
     *  异步任务中异常处理
     * @return
     */
    @Override
    public AsyncUncaughtExceptionHandler getAsyncUncaughtExceptionHandler() {
        return new AsyncUncaughtExceptionHandler() {

            @Override
            public void handleUncaughtException(Throwable arg0, Method arg1, Object... arg2) {
                log.error("=========================="+arg0.getMessage()+"=======================", arg0);
                log.error("exception method:"+arg1.getName());
            }
        };
    }
}
```

## AOP

Spring 中的 AOP 模块中：如果目标对象实现了接口，则默认采用 JDK 动态代理，否则采用 CGLIB 动态代理(5.X开始默认使用CGLIB, 基于字节码效率更高)。

- Pointcut（切点），指定在什么情况下才执行 AOP，例如方法被打上某个注解的时候
- JoinPoint（连接点），程序运行中的执行点，例如一个方法的执行或是一个异常的处理；并且在 Spring AOP 中，只有方法连接点
- Advice（增强），对连接点进行增强（代理）：在方法调用前、调用后 或者 抛出异常时，进行额外的处理
- Aspect（切面），由 Pointcut 和 Advice 组成，可理解为：要在什么情况下（Pointcut）对哪个目标（JoinPoint）做什么样的增强（Advice）

回顾代理模式:

1.java proxy

```java
Proxy.newProxyInstance(ClassLoader loader, Class<?>[] interfaces, InvocationHandler h)

loader :类加载器，用于加载代理对象。
interfaces : 被代理类实现的一些接口；
h : 实现了 InvocationHandler 接口的对象；
```

2.cglib(多种拦截器接口自查资料)

```java
Enhancer enhancer = new Enhancer();
enhancer.setClassLoader(SampleClass.class.getClassLoader())
enhancer.setSuperclass(SampleClass.class);
enhancer.setCallback(new MethodInterceptor() {
    @Override
    public Object intercept(Object obj, Method method, Object[] args, MethodProxy proxy) throws Throwable {
            System.out.println("before method run...");
            Object result = proxy.invokeSuper(obj, args);
            System.out.println("after method run...");
            return result;
        }
    });
SampleClass sample = (SampleClass) enhancer.create();
       sample.test();
```

切面包含了5个通知方法：

- 前置通知（@Before）：在目标方法被调用之前调用通知功能；
- 后置通知（@After）：在目标方法完成之后调用通知，此时不会关心方法的输出是什么；
- 返回通知（@AfterReturning）：在目标方法成功执行之后调用通知；
- 异常通知（@AfterThrowing）：在目标方法抛出异常后调用通知。
- 环绕通知  (@Aroud)  :  在目标方法执行之前和之后都可以执行额外代码的通知。

这几个通知的顺序在不同的Spring版本中有所不同：

1. Spring4.x
   
   - 正常情况：环绕前置  —->  @Before —-> 目标方法 —-> 环绕返回  -->环绕最终(finally)—-> @After —-> @AfterReturning
   - 异常情况：环绕前置  —-> @Before —-> 目标方法 —-> 环绕异常  -->环绕最终(finally)—-> @After —-> @AfterThrowing

2. Spring5.x
   
   - 正常情况：环绕前置  —-> @Before —-> 目标方法 —-> @AfterReturning —-> @After  —-> 环绕返回  -->环绕最终(finally)
   - 异常情况：环绕前置  —-> @Before —-> 目标方法 —-> @AfterThrowing —-> @After  —-> 环绕异常  -->环绕最终(finally)
   
   在有多个切面时,可以通过@Order或Ordered接口指定顺序.
   
   基于注解的方式实现AOP需要在配置类中添加注解@EnableAspectJAutoProxy
   
   ```java
   // exposeProxy = true表示通过aop框架暴露该代理对象到AOP上下文(Thr中,AopContext能够访问
   //(通过AopContext的ThreadLocal实现)
   @EnableAspectJAutoProxy(exposeProxy = true)
   //可以调用代理对象的方法
   //应用场景,当想在serivce没有@Transactional的方法中调用带有@Transactional注解的方法时,要使@Transactional生效时,使用this.xxx调用的不是代理之后的方法,而下面的方式可以调用代理方法
   ((T) AopContext.currentProxy()).xxx;
   ```
   
   因为spring采用动态代理机制来实现事务控制，而动态代理(jdk代理,因为service实现了接口)最终都是要调用原始对象的，而原始对象在去调用方法时，是不会再触发代理了.
   
   可通过注解的proxyTargetClass=true指定使用cglib代理方式.(有误, 同样不会触发代理) 

> pointcut表达式支持更直观的操作, 如
> 
> ```
> @Pointcut("com.xyz.myapp.CommonPointcuts.dataAccessOperation() && args(account,..)")
> private void accountDataAccessOperation(Account account) {}
> 
> @Before("accountDataAccessOperation(account)")
> public void validateAccount(Account account) {
>  // ...
> }
> ```
> 
> args（account,..） 部分有两个用途。首先，它将匹配限制为仅那些方法执行，其中方法至少采用一个参数，并且传递给该参数的是 Account 的实例。其次，它通过 account 参数使实际的 Account 对象可用于advice。
> 
> 又或者
> 
> ```
> @Before("com.xyz.lib.Pointcuts.anyPublicMethod() && @annotation(auditable)")
> public void audit(Auditable auditable) {
>  AuditCode code = auditable.value();
> }
> ```

JointPoint使用

```java
//通过joinPoint获取方法
Signature signature = joinPoint.getSignature();
MethodSignature methodSignature = (MethodSignature) signature;
Method method = methodSignature.getMethod();
//获取目标对象
joinPoint.getTarget().getClass()
//通过joinPoint获取方法参数 s
joinPoint.getArgs()
//获取代理对象
joinPoint.getThis():
```

### 脚手架

> 摘自阿里技术微信公众号文章

**定义方法增强处理器**

我们先定义出 ”代理“ 的抽象：方法增强处理器 MethodAdviceHandler 。之后我们定义的每一个注解，都绑定一个对应的 MethodAdviceHandler 的实现类，当目标方法被代理时，由对应的 MethodAdviceHandler 的实现类来处理该方法的代理访问。

```java
/**
 * 方法增强处理器
 *
 * @param <R> 目标方法返回值的类型
 */
public interface MethodAdviceHandler<R> {

    /**
     * 目标方法执行之前的判断，判断目标方法是否允许执行。默认返回 true，即 默认允许执行
     *
     * @param point 目标方法的连接点
     * @return 返回 true 则表示允许调用目标方法；返回 false 则表示禁止调用目标方法。
     * 当返回 false 时，此时会先调用 getOnForbid 方法获得被禁止执行时的返回值，然后
     * 调用 onComplete 方法结束切面
     */
    default boolean onBefore(ProceedingJoinPoint point) { return true; }

    /**
     * 禁止调用目标方法时（即 onBefore 返回 false），执行该方法获得返回值，默认返回 null
     *
     * @param point 目标方法的连接点
     * @return 禁止调用目标方法时的返回值
     */
    default R getOnForbid(ProceedingJoinPoint point) { return null; }

    /**
     * 目标方法抛出异常时，执行的动作
     *
     * @param point 目标方法的连接点
     * @param e     抛出的异常
     */
    void onThrow(ProceedingJoinPoint point, Throwable e);

    /**
     * 获得抛出异常时的返回值，默认返回 null
     *
     * @param point 目标方法的连接点
     * @param e     抛出的异常
     * @return 抛出异常时的返回值
     */
    default R getOnThrow(ProceedingJoinPoint point, Throwable e) { return null; }

    /**
     * 目标方法完成时，执行的动作
     *
     * @param point     目标方法的连接点
     * @param startTime 执行的开始时间
     * @param permitted 目标方法是否被允许执行
     * @param thrown    目标方法执行时是否抛出异常
     * @param result    执行获得的结果
     */
    default void onComplete(ProceedingJoinPoint point, long startTime, boolean permitted, boolean thrown, Object result) { }
}
```

为了方便 MethodAdviceHandler 的使用，我们定义一个抽象类，提供一些常用的方法。

```java
public abstract class BaseMethodAdviceHandler<R> implements MethodAdviceHandler<R> {

    protected final Logger logger = LoggerFactory.getLogger(this.getClass());

    /**
     * 抛出异常时候的默认处理
     */
    @Override
    public void onThrow(ProceedingJoinPoint point, Throwable e) {
        String methodDesc = getMethodDesc(point);
        Object[] args = point.getArgs();
        logger.error("{} 执行时出错，入参={}", methodDesc, JSON.toJSONString(args, true), e);
    }

    /**
     * 获得被代理的方法
     *
     * @param point 连接点
     * @return 代理的方法
     */
    protected Method getTargetMethod(ProceedingJoinPoint point) {
        // 获得方法签名
        Signature signature = point.getSignature();
        // Spring AOP 只有方法连接点，所以 Signature 一定是 MethodSignature
        return ((MethodSignature) signature).getMethod();
    }

    /**
     * 获得方法描述，目标类名.方法名
     *
     * @param point 连接点
     * @return 目标类名.执行方法名
     */
    protected String getMethodDesc(ProceedingJoinPoint point) {
        // 获得被代理的类
        Object target = point.getTarget();
        String className = target.getClass().getSimpleName();

        Signature signature = point.getSignature();
        String methodName = signature.getName();

        return className + "." + methodName;
    }
}
```

**定义方法切面的抽象**

同理，将方法切面的公共逻辑抽取出来，定义出方法切面的抽象 —— 后续每定义一个注解，对应的方法切面继承自这个抽象类就好.

```java
/**
 * 方法切面抽象类，由子类来指定切点和绑定的方法增强处理器的类型
 */
public abstract class BaseMethodAspect implements ApplicationContextAware {

    /**
     * 切点，通过 @Pointcut 指定相关的注解
     */
    protected abstract void pointcut();

    /**
     * 对目标方法进行环绕增强处理，子类需通过 pointcut() 方法指定切点
     *
     * @param point 连接点
     * @return 方法执行返回值
     */
    @Around("pointcut()")
    public Object advice(ProceedingJoinPoint point) {
        // 获得切面绑定的方法增强处理器的类型
        Class<? extends MethodAdviceHandler<?>> handlerType = getAdviceHandlerType();
        // 从 Spring 上下文中获得方法增强处理器的实现 Bean
        MethodAdviceHandler<?> adviceHandler = appContext.getBean(handlerType);
        // 使用方法增强处理器对目标方法进行增强处理
        return advice(point, adviceHandler);
    }

    /**
     * 获得切面绑定的方法增强处理器的类型
     */
    protected abstract Class<? extends MethodAdviceHandler<?>> getAdviceHandlerType();

    /**
     * 使用方法增强处理器增强被注解的方法
     *
     * @param point   连接点
     * @param handler 切面处理器
     * @return 方法执行返回值
     */
    private Object advice(ProceedingJoinPoint point, MethodAdviceHandler<?> handler) {
        // 执行之前，返回是否被允许执行
        boolean permitted = handler.onBefore(point);

        // 方法返回值
        Object result;
        // 是否抛出了异常
        boolean thrown = false;
        // 开始执行的时间
        long startTime = System.currentTimeMillis();

        // 目标方法被允许执行
        if (permitted) {
            try {
                // 执行目标方法
                result = point.proceed();
            } catch (Throwable e) {
                // 抛出异常
                thrown = true;
                // 处理异常
                handler.onThrow(point, e);
                // 抛出异常时的返回值
                result = handler.getOnThrow(point, e);
            }
        }
        // 目标方法被禁止执行
        else {
            // 禁止执行时的返回值
            result = handler.getOnForbid(point);
        }

        // 结束
        handler.onComplete(point, startTime, permitted, thrown, result);

        return result;
    }

    private ApplicationContext appContext;

    @Override
    public void setApplicationContext(ApplicationContext applicationContext) throws BeansException {
        appContext = applicationContext;
    }
}
```

此时，我们基于 AOP 的代理模式小架子就已经搭好了。之所以需要这个小架子，是为了后续新增注解时，能够进行横向的扩展：每次新增一个注解（XxxAnno），只需要实现一个新的方法增强处理器（XxxHandler）和新的方法切面 （XxxAspect），而不会修改现有代码，从而完美符合 **对修改关闭，对扩展开放** 设计模式理念。

**使用**:

**定义一个注解**

```java
/**
 * 用于产生调用记录的注解，会记录下方法的出入参、调用时长
 */
@Target({ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
public @interface InvokeRecordAnno {
    /**
     * 调用说明
     */
    String value() default "";
}
```

**方法增强处理器的实现**

```java
@Component
public class InvokeRecordHandler extends BaseMethodAdviceHandler<Object> {

    /**
     * 记录方法出入参和调用时长
     */
    @Override
    public void onComplete(ProceedingJoinPoint point, long startTime, boolean permitted, boolean thrown, Object result) {
        String methodDesc = getMethodDesc(point);
        Object[] args = point.getArgs();
        long costTime = System.currentTimeMillis() - startTime;

        logger.warn("\n{} 执行结束，耗时={}ms，入参={}, 出参={}",
                    methodDesc, costTime,
                    JSON.toJSONString(args, true),
                    JSON.toJSONString(result, true));
    }

    @Override
    protected String getMethodDesc(ProceedingJoinPoint point) {
        Method targetMethod = getTargetMethod(point);
        // 获得方法上的 InvokeRecordAnno
        InvokeRecordAnno anno = targetMethod.getAnnotation(InvokeRecordAnno.class);
        String description = anno.value();

        // 如果没有指定方法说明，那么使用默认的方法说明
        if (StringUtils.isBlank(description)) {
            description = super.getMethodDesc(point);
        }

        return description;
    }
}
```

**方法切面的实现**

```java
@Aspect
@Order(1)
@Component
public class InvokeRecordAspect extends BaseMethodAspect {

    /**
     * 指定切点（处理打上 InvokeRecordAnno 的方法）
     */
    @Override
    @Pointcut("@annotation(xyz.mizhoux.experiment.proxy.anno.InvokeRecordAnno)")
    protected void pointcut() { }

    /**
     * 指定该切面绑定的方法切面处理器为 InvokeRecordHandler
     */
    @Override
    protected Class<? extends MethodAspectHandler<?>> getHandlerType() {
        return InvokeRecordHandler.class;
    }
}
```

**其他**

1.@Aspect 用来告诉 Spring 这是一个切面，然后 Spring 在启动会时扫描 @Pointcut 匹配的方法，然后对这些目标方法进行织入处理：即使用切面中打上 @Around 的方法来对目标方法进行增强处理。

2.@Order 是用来标记这个切面应该在哪一层，数字越小，则在越外层（越先进入，越后结束） —— 方法调用记录的切面很明显应该在大气层（小编：王者荣耀术语，即最外层），因为方法调用记录的切面应该最后结束，所以我们给一个小点的数字。

**测试**

```java
@RestController
@RequestMapping("proxy")
public class ProxyTestController {

    @GetMapping("test")
    @InvokeRecordAnno("测试代理模式")
    public Map<String, Object> testProxy(@RequestParam String biz,
                                         @RequestParam String param) {
        Map<String, Object> result = new HashMap<>(4);
        result.put("id", 123);
        result.put("nick", "之叶");

        return result;
    }
}
```

**扩展**

假设我们要在目标方法抛出异常时进行处理：抛出异常时，把异常信息异步发送到邮箱或者钉钉，然后根据方法的返回值类型，返回相应的错误响应。

**定义相应的注解**

```java
/**
 * 用于异常处理的注解
 */
@Target({ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
public @interface ExceptionHandleAnno { }
```

**实现方法增强处理器**

```java
@Component
public class ExceptionHandleHandler extends BaseMethodAdviceHandler<Object> {

    /**
     * 抛出异常时的处理
     */
    @Override
    public void onThrow(ProceedingJoinPoint point, Throwable e) {
        super.onThrow(point, e);
        // 发送异常到邮箱或者钉钉的逻辑
    }

    /**
     * 抛出异常时的返回值
     */
    @Override
    public Object getOnThrow(ProceedingJoinPoint point, Throwable e) {
        // 获得返回值类型
        Class<?> returnType = getTargetMethod(point).getReturnType();

        // 如果返回值类型是 Map 或者其子类
        if (Map.class.isAssignableFrom(returnType)) {
            Map<String, Object> result = new HashMap<>(4);
            result.put("success", false);
            result.put("message", "调用出错");

            return result;
        }

        return null;
    }
}
```

如果返回值的类型是个 Map，那么我们就返回调用出错情况下的对应 Map 实例（真实情况一般是返回业务系统中的 Response）。

**实现方法切面**

```java
@Aspect
@Order(10)
@Component
public class ExceptionHandleAspect extends BaseMethodAspect {

    /**
     * 指定切点（处理打上 ExceptionHandleAnno 的方法）
     */
    @Override
    @Pointcut("@annotation(xyz.mizhoux.experiment.proxy.anno.ExceptionHandleAnno)")
    protected void pointcut() { }

    /**
     * 指定该切面绑定的方法切面处理器为 ExceptionHandleHandler
     */
    @Override
    protected Class<? extends MethodAdviceHandler<?>> getAdviceHandlerType() {
        return ExceptionHandleHandler.class;
    }
}
```

异常处理一般是非常内层的切面，所以我们将@Order 设置为 10，让 ExceptionHandleAspect 在 InvokeRecordAspect 更内层（即之后进入、之前结束），从而外层的 InvokeRecordAspect 也可以记录到抛出异常时的返回值。修改测试用的方法，加上 @ExceptionHandleAnno：

```java
@RestController
@RequestMapping("proxy")
public class ProxyTestController {

    @GetMapping("test")
    @ExceptionHandleAnno
    @InvokeRecordAnno("测试代理模式")
    public Map<String, Object> testProxy(@RequestParam String biz,
                                         @RequestParam String param) {
        if (biz.equals("abc")) {
            throw new IllegalArgumentException("非法的 biz=" + biz);
        }

        Map<String, Object> result = new HashMap<>(4);
        result.put("id", 123);
        result.put("nick", "之叶");

        return result;
    }
}
```

测试结果：异常处理的切面先结束，方法调用记录的切面后结束。

**其他问题**

问：抛出异常时， InvokeRecordHandler 的 onThrow 方法没有执行，为什么呢？

答：因为 InvokeRecordAspect 比 ExceptionHandleAspect 在更外层，外层的 InvokeRecordAspect 在执行时，执行的已经是内层的 ExceptionHandleAspect 代理过的方法，而对应的 ExceptionHandleHandler 已经把异常 “消化” 了，即 ExceptionHandleAspect 代理过的方法已经不会再抛出异常。

问:  如果我们要 限制单位时间内方法的调用次数，比如 3s 内用户只能提交表单 1 次，似乎也可以通过这个代理模式的套路来实现。

答： 首先定义好注解（注解可以包含单位时间、最大调用次数等参数），然后在方法切面处理器的 onBefore 方法里面，使用缓存记录下单位时间内用户的提交次数，如果超出最大调用次数，返回 false，那么目标方法就不被允许调用了；然后在 getOnForbid 的方法里面，返回这种情况下的响应。