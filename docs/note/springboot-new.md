# Spring尝鲜

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

| 注解名称                   | 功能                                                         |
| -------------------------- | ------------------------------------------------------------ |
| @Null                      | 检查该字段为空                                               |
| @NotNull                   | 不能为null                                                   |
| @NotBlank                  | 不能为空，常用于检查空字符串                                 |
| @NotEmpty                  | 不能为空，多用于检测list是否size是0                          |
| @Max                       | 该字段的值只能小于或等于该值                                 |
| @Min                       | 该字段的值只能大于或等于该值                                 |
| @Past                      | 检查该字段的日期是在过去                                     |
| @Future                    | 检查该字段的日期是否是属于将来的日期                         |
| @Email                     | 检查是否是一个有效的email地址                                |
| @Pattern(regex=,flag=)     | 被注释的元素必须符合指定的正则表达式                         |
| @Range(min=,max=,message=) | 被注释的元素必须在合适的范围内                               |
| @Size(min=, max=)          | 检查该字段的size是否在min和max之间，可以是字符串、数组、集合、Map等 |
| @Length(min=,max=)         | 检查所属的字段的长度是否在min和max之间,只能用于字符串        |
| @AssertTrue                | 用于boolean字段，该字段只能为true                            |
| @AssertFalse               | 该字段的值只能为false                                        |
| @Future                    | 该字段的值必须时一个将来的日期                               |

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