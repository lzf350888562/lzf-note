**因为Spring Boot 2.1.x默认使用了MySQL 8.0的驱动，所以采用`com.mysql.cj.jdbc.Driver`，而不是老的`com.mysql.jdbc.Driver`。**

# JDBC与数据源

Java数据库连接（Java Database Connectivity，简称JDBC）是Java语言中用来规范客户端程序如何来访问数据库的应用程序接口，提供了诸如查询和更新数据库中数据的方法。JDBC也是Sun Microsystems的商标。我们通常说的JDBC是面向关系型数据库的。

JDBC API主要位于JDK中的`java.sql`包中（之后扩展的内容位于`javax.sql`包中），主要包括（斜体代表接口，需驱动程序提供者来具体实现）：

- DriverManager：负责加载各种不同驱动程序（Driver），并根据不同的请求，向调用者返回相应的数据库连接（Connection）。
- Driver：驱动程序，会将自身加载到DriverManager中去，并处理相应的请求并返回相应的数据库连接（Connection）。
- Connection：数据库连接，负责与进行数据库间通讯，SQL执行以及事务处理都是在某个特定Connection环境中进行的。可以产生用以执行SQL的Statement。
- Statement：用以执行SQL查询和更新（针对静态SQL语句和单次执行）。PreparedStatement：用以执行包含动态参数的SQL查询和更新（在服务器端编译，允许重复执行以提高效率）。
- CallableStatement：用以调用数据库中的存储过程。
- SQLException：代表在数据库连接的建立和关闭和SQL语句的执行过程中发生了例外情况（即错误）。



在`java.sql`中并没有数据源（Data Source）的概念。这是由于在`java.sql`中包含的是JDBC内核API，另外还有个`javax.sql`包，其中包含了JDBC标准的扩展API。而关于数据源（Data Source）的定义，就在`javax.sql`这个扩展包中。

实际上，在JDBC内核API的实现下，就已经可以实现对数据库的访问了，那么我们为什么还需要数据源呢？主要出于以下几个目的：

1. 封装关于数据库访问的各种参数，实现统一管理
2. 通过对数据库的连接池管理，节省开销并提高效率

在Java这个自由开放的生态中，已经有非常多优秀的开源数据源可以供大家选择，比如：DBCP、C3P0、Druid、HikariCP等。

而在Spring Boot 2.x中，对数据源的选择也紧跟潮流，采用了目前性能最佳的[HikariCP](https://github.com/brettwooldridge/HikariCP)。接下来，我们就来具体说说，这个Spring Boot中的默认数据源配置。



# Hikari

- 通用配置：以`spring.datasource.*`的形式存在，主要是对一些即使使用不同数据源也都需要配置的一些常规内容。比如：数据库链接地址、用户名、密码等。这里就不做过多说明了，通常就这些配置：

```
spring.datasource.url=jdbc:mysql://localhost:3306/test
spring.datasource.username=root
spring.datasource.password=123456
spring.datasource.driver-class-name=com.mysql.jdbc.Driver
```

- 数据源连接池配置：以`spring.datasource.<数据源名称>.*`的形式存在，比如：Hikari的配置参数就是`spring.datasource.hikari.*`形式。下面这个是我们最常用的几个配置项及对应说明：

```
spring.datasource.hikari.minimum-idle=10
spring.datasource.hikari.maximum-pool-size=20
spring.datasource.hikari.idle-timeout=500000
spring.datasource.hikari.max-lifetime=540000
spring.datasource.hikari.connection-timeout=60000
spring.datasource.hikari.connection-test-query=SELECT 1
```

- `spring.datasource.hikari.minimum-idle`: 最小空闲连接，默认值10，小于0或大于maximum-pool-size，都会重置为maximum-pool-size
- `spring.datasource.hikari.maximum-pool-size`: 最大连接数，小于等于0会被重置为默认值10；大于零小于1会被重置为minimum-idle的值
- `spring.datasource.hikari.idle-timeout`: 空闲连接超时时间，默认值600000（10分钟），大于等于max-lifetime且max-lifetime>0，会被重置为0；不等于0且小于10秒，会被重置为10秒。
- `spring.datasource.hikari.max-lifetime`: 连接最大存活时间，不等于0且小于30秒，会被重置为默认值30分钟.设置应该比mysql设置的超时时间短
- `spring.datasource.hikari.connection-timeout`: 连接超时时间：毫秒，小于250毫秒，否则被重置为默认值30秒
- `spring.datasource.hikari.connection-test-query`: 用于测试连接是否可用的查询语句

| **name**                  | **描述**                                                     | **构造器默认值**               | **默认配置validate之后的值** | **validate重置**                                             |
| :------------------------ | :----------------------------------------------------------- | :----------------------------- | :--------------------------- | :----------------------------------------------------------- |
| autoCommit                | 自动提交从池中返回的连接                                     | TRUE                           | TRUE                         | –                                                            |
| connectionTimeout         | 等待来自池的连接的最大毫秒数                                 | SECONDS.toMillis(30) = 30000   | 30000                        | 如果小于250毫秒，则被重置回30秒                              |
| idleTimeout               | 连接允许在池中闲置的最长时间                                 | MINUTES.toMillis(10) = 600000  | 600000                       | 如果idleTimeout+1秒>maxLifetime 且 maxLifetime>0，则会被重置为0（代表永远不会退出）；如果idleTimeout!=0且小于10秒，则会被重置为10秒 |
| maxLifetime               | 池中连接最长生命周期                                         | MINUTES.toMillis(30) = 1800000 | 1800000                      | 如果不等于0且小于30秒则会被重置回30分钟                      |
| connectionTestQuery       | 如果您的驱动程序支持JDBC4，我们强烈建议您不要设置此属性      | null                           | null                         | –                                                            |
| minimumIdle               | 池中维护的最小空闲连接数                                     | -1                             | 10                           | minIdle<0或者minIdle>maxPoolSize,则被重置为maxPoolSize       |
| maximumPoolSize           | 池中最大连接数，包括闲置和使用中的连接                       | -1                             | 10                           | 如果maxPoolSize小于1，则会被重置。当minIdle<=0被重置为DEFAULT_POOL_SIZE则为10;如果minIdle>0则重置为minIdle的值 |
| metricRegistry            | 该属性允许您指定一个 Codahale / Dropwizard MetricRegistry 的实例，供池使用以记录各种指标 | null                           | null                         | –                                                            |
| healthCheckRegistry       | 该属性允许您指定池使用的Codahale / Dropwizard HealthCheckRegistry的实例来报告当前健康信息 | null                           | null                         | –                                                            |
| poolName                  | 连接池的用户定义名称，主要出现在日志记录和JMX管理控制台中以识别池和池配置 | null                           | HikariPool-1                 | –                                                            |
| initializationFailTimeout | 如果池无法成功初始化连接，则此属性控制池是否将 fail fast     | 1                              | 1                            | –                                                            |
| isolateInternalQueries    | 是否在其自己的事务中隔离内部池查询，例如连接活动测试         | FALSE                          | FALSE                        | –                                                            |
| allowPoolSuspension       | 控制池是否可以通过JMX暂停和恢复                              | FALSE                          | FALSE                        | –                                                            |
| readOnly                  | 从池中获取的连接是否默认处于只读模式                         | FALSE                          | FALSE                        | –                                                            |
| registerMbeans            | 是否注册JMX管理Bean（MBeans）                                | FALSE                          | FALSE                        | –                                                            |
| catalog                   | 为支持 catalog 概念的数据库设置默认 catalog                  | driver default                 | null                         | –                                                            |
| connectionInitSql         | 该属性设置一个SQL语句，在将每个新连接创建后，将其添加到池中之前执行该语句。 | null                           | null                         | –                                                            |
| driverClassName           | HikariCP将尝试通过仅基于jdbcUrl的DriverManager解析驱动程序，但对于一些较旧的驱动程序，还必须指定driverClassName | null                           | null                         | –                                                            |
| transactionIsolation      | 控制从池返回的连接的默认事务隔离级别                         | null                           | null                         | –                                                            |
| validationTimeout         | 连接将被测试活动的最大时间量                                 | SECONDS.toMillis(5) = 5000     | 5000                         | 如果小于250毫秒，则会被重置回5秒                             |
| leakDetectionThreshold    | 记录消息之前连接可能离开池的时间量，表示可能的连接泄漏       | 0                              | 0                            | 如果大于0且不是单元测试，则进一步判断：(leakDetectionThreshold < SECONDS.toMillis(2) or (leakDetectionThreshold > maxLifetime && maxLifetime > 0)，会被重置为0 . 即如果要生效则必须>0，而且不能小于2秒，而且当maxLifetime > 0时不能大于maxLifetime |
| dataSource                | 这个属性允许你直接设置数据源的实例被池包装，而不是让HikariCP通过反射来构造它 | null                           | null                         | –                                                            |
| schema                    | 该属性为支持模式概念的数据库设置默认模式                     | driver default                 | null                         | –                                                            |
| threadFactory             | 此属性允许您设置将用于创建池使用的所有线程的java.util.concurrent.ThreadFactory的实例。 | null                           | null                         | –                                                            |
| scheduledExecutor         | 此属性允许您设置将用于各种内部计划任务的java.util.concurrent.ScheduledExecutorService实例 | null                           | null                         | –                                                            |



# druid

```
<dependency>
    <groupId>com.alibaba</groupId>
    <artifactId>druid-spring-boot-starter</artifactId>
    <version>1.1.21</version>
</dependency>
```

```
spring.datasource.druid.url=jdbc:mysql://localhost:3306/test
spring.datasource.druid.username=root
spring.datasource.druid.password=
spring.datasource.druid.driver-class-name=com.mysql.cj.jdbc.Driver
```

配置连接池

```
spring.datasource.druid.initialSize=10
spring.datasource.druid.maxActive=20
spring.datasource.druid.maxWait=60000
spring.datasource.druid.minIdle=1
spring.datasource.druid.timeBetweenEvictionRunsMillis=60000
spring.datasource.druid.minEvictableIdleTimeMillis=300000
spring.datasource.druid.testWhileIdle=true
spring.datasource.druid.testOnBorrow=true
spring.datasource.druid.testOnReturn=false
spring.datasource.druid.poolPreparedStatements=true
spring.datasource.druid.maxOpenPreparedStatements=20
spring.datasource.druid.validationQuery=SELECT 1
spring.datasource.druid.validation-query-timeout=500
spring.datasource.druid.filters=stat
```

| 置                                        | 缺省值             | 说明                                                         |
| :---------------------------------------- | :----------------- | :----------------------------------------------------------- |
| name                                      |                    | 配置这个属性的意义在于，如果存在多个数据源，监控的时候可以通过名字来区分开来。如果没有配置，将会生成一个名字，格式是：”DataSource-“ + System.identityHashCode(this). 另外配置此属性至少在1.0.5版本中是不起作用的，强行设置name会出错。[详情-点此处](http://blog.csdn.net/lanmo555/article/details/41248763)。 |
| url                                       |                    | 连接数据库的url，不同数据库不一样。例如： mysql : jdbc:mysql://10.20.153.104:3306/druid2 oracle : jdbc:oracle:thin:@10.20.149.85:1521:ocnauto |
| username                                  |                    | 连接数据库的用户名                                           |
| password                                  |                    | 连接数据库的密码。如果你不希望密码直接写在配置文件中，可以使用ConfigFilter。[详细看这里](https://github.com/alibaba/druid/wiki/使用ConfigFilter) |
| driverClassName                           | 根据url自动识别    | 这一项可配可不配，如果不配置druid会根据url自动识别dbType，然后选择相应的driverClassName |
| initialSize                               | 0                  | 初始化时建立物理连接的个数。初始化发生在显示调用init方法，或者第一次getConnection时 |
| maxActive                                 | 8                  | 最大连接池数量                                               |
| maxIdle                                   | 8                  | 已经不再使用，配置了也没效果                                 |
| minIdle                                   |                    | 最小连接池数量                                               |
| maxWait                                   |                    | 获取连接时最大等待时间，单位毫秒。配置了maxWait之后，缺省启用公平锁，并发效率会有所下降，如果需要可以通过配置useUnfairLock属性为true使用非公平锁。 |
| poolPreparedStatements                    | false              | 是否缓存preparedStatement，也就是PSCache。PSCache对支持游标的数据库性能提升巨大，比如说oracle。在mysql下建议关闭。 |
| maxPoolPreparedStatementPerConnectionSize | -1                 | 要启用PSCache，必须配置大于0，当大于0时，poolPreparedStatements自动触发修改为true。在Druid中，不会存在Oracle下PSCache占用内存过多的问题，可以把这个数值配置大一些，比如说100 |
| validationQuery                           |                    | 用来检测连接是否有效的sql，要求是一个查询语句，常用select ‘x’。如果validationQuery为null，testOnBorrow、testOnReturn、testWhileIdle都不会起作用。 |
| validationQueryTimeout                    |                    | 单位：秒，检测连接是否有效的超时时间。底层调用jdbc Statement对象的void setQueryTimeout(int seconds)方法 |
| testOnBorrow                              | true               | 申请连接时执行validationQuery检测连接是否有效，做了这个配置会降低性能。 |
| testOnReturn                              | false              | 归还连接时执行validationQuery检测连接是否有效，做了这个配置会降低性能。 |
| testWhileIdle                             | false              | 建议配置为true，不影响性能，并且保证安全性。申请连接的时候检测，如果空闲时间大于timeBetweenEvictionRunsMillis，执行validationQuery检测连接是否有效。 |
| keepAlive                                 | false （1.0.28）   | 连接池中的minIdle数量以内的连接，空闲时间超过minEvictableIdleTimeMillis，则会执行keepAlive操作。 |
| timeBetweenEvictionRunsMillis             | 1分钟（1.0.14）    | 有两个含义： 1) Destroy线程会检测连接的间隔时间，如果连接空闲时间大于等于minEvictableIdleTimeMillis则关闭物理连接。 2) testWhileIdle的判断依据，详细看testWhileIdle属性的说明 |
| numTestsPerEvictionRun                    | 30分钟（1.0.14）   | 不再使用，一个DruidDataSource只支持一个EvictionRun           |
| minEvictableIdleTimeMillis                |                    | 连接保持空闲而不被驱逐的最小时间                             |
| connectionInitSqls                        |                    | 物理连接初始化的时候执行的sql                                |
| exceptionSorter                           | 根据dbType自动识别 | 当数据库抛出一些不可恢复的异常时，抛弃连接                   |
| filters                                   |                    | 属性类型是字符串，通过别名的方式配置扩展插件，常用的插件有： 监控统计用的filter:stat 日志用的filter:log4j 防御sql注入的filter:wall |
| proxyFilters                              |                    | 类型是List<com.alibaba.druid.filter.Filter>，如果同时配置了filters和proxyFilters，是组合关系，并非替换关系 |

## 监控

```
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```

```
spring.datasource.druid.stat-view-servlet.url-pattern          ：访问地址规则
spring.datasource.druid.stat-view-servlet.reset-enable         ：是否允许清空统计数据
spring.datasource.druid.stat-view-servlet.login-username       ：监控页面的登录账户
spring.datasource.druid.stat-view-servlet.login-password       ：监控页面的登录密码
```

访问Druid的监控页面`http://localhost:8080/druid/`

输入上面`spring.datasource.druid.stat-view-servlet.login-username`和`spring.datasource.druid.stat-view-servlet.login-password`配置的登录账户与密码，就能看到监控页面

## 多数据源

```
server:
  context-path: /web

spring:
  datasource:
    druid:
      # 数据库访问配置, 使用druid数据源
      # 数据源1 mysql
      mysql:
        type: com.alibaba.druid.pool.DruidDataSource
        driver-class-name: com.mysql.jdbc.Driver
        url: jdbc:mysql://localhost:3306/test?useUnicode=true&characterEncoding=UTF-8&rewriteBatchedStatements=true&autoReconnect=true&failOverReadOnly=false&zeroDateTimeBehavior=convertToNull
        username: root
        password: 123456
      # 数据源2 oracle
      oracle: 
        type: com.alibaba.druid.pool.DruidDataSource
        driver-class-name: oracle.jdbc.driver.OracleDriver
        url: jdbc:oracle:thin:@localhost:1521:ORCL
        username: test
        password: 123456
        
      # 连接池配置
      initial-size: 5
      min-idle: 5
      max-active: 20
      # 连接等待超时时间
      max-wait: 30000
      # 配置检测可以关闭的空闲连接间隔时间
      time-between-eviction-runs-millis: 60000
      # 配置连接在池中的最小生存时间
      min-evictable-idle-time-millis: 300000
      validation-query: select '1' from dual
      test-while-idle: true
      test-on-borrow: false
      test-on-return: false
      # 打开PSCache，并且指定每个连接上PSCache的大小
      pool-prepared-statements: true
      max-open-prepared-statements: 20
      max-pool-prepared-statement-per-connection-size: 20
      # 配置监控统计拦截的filters, 去掉后监控界面sql无法统计, 'wall'用于防火墙
      filters: stat,wall
      # Spring监控AOP切入点，如x.y.z.service.*,配置多个英文逗号分隔
      aop-patterns: com.springboot.servie.*
      
    
      # WebStatFilter配置
      web-stat-filter:
        enabled: true
        # 添加过滤规则
        url-pattern: /*
        # 忽略过滤的格式
        exclusions: '*.js,*.gif,*.jpg,*.png,*.css,*.ico,/druid/*'
      
      # StatViewServlet配置 
      stat-view-servlet:
        enabled: true
        # 访问路径为/druid时，跳转到StatViewServlet
        url-pattern: /druid/*
        # 是否能够重置数据
        reset-enable: false
        # 需要账号密码才能访问控制台
        login-username: druid
        login-password: druid123
        # IP白名单
        # allow: 127.0.0.1
        #　IP黑名单（共同存在时，deny优先于allow）
        # deny: 192.168.1.218
      
      # 配置StatFilter
      filter: 
        stat: 
          log-slow-sql: true 
```

```
@Configuration
@MapperScan(basePackages = MysqlDatasourceConfig.PACKAGE, sqlSessionFactoryRef = "mysqlSqlSessionFactory")
public class MysqlDatasourceConfig {

   // mysqldao扫描路径
   static final String PACKAGE = "com.springboot.mysqldao";
   // mybatis mapper扫描路径
   static final String MAPPER_LOCATION = "classpath:mapper/mysql/*.xml";

   @Primary
   @Bean(name = "mysqldatasource")
   @ConfigurationProperties("spring.datasource.druid.mysql")
   public DataSource mysqlDataSource() {
      return DruidDataSourceBuilder.create().build();
   }

   @Bean(name = "mysqlTransactionManager")
   @Primary
   public DataSourceTransactionManager mysqlTransactionManager() {
      return new DataSourceTransactionManager(mysqlDataSource());
   }

   @Bean(name = "mysqlSqlSessionFactory")
   @Primary
   public SqlSessionFactory mysqlSqlSessionFactory(@Qualifier("mysqldatasource") DataSource dataSource)
         throws Exception {
      final SqlSessionFactoryBean sessionFactory = new SqlSessionFactoryBean();
      sessionFactory.setDataSource(dataSource);
      //如果不使用xml的方式配置mapper，则可以省去下面这行mapper location的配置。
      sessionFactory.setMapperLocations(
            new PathMatchingResourcePatternResolver().getResources(MysqlDatasourceConfig.MAPPER_LOCATION));
      return sessionFactory.getObject();
   }
}
```

```
@Configuration
@MapperScan(basePackages = OracleDatasourceConfig.PACKAGE, 
   sqlSessionFactoryRef = "oracleSqlSessionFactory")
public class OracleDatasourceConfig {
   
   // oracledao扫描路径
   static final String PACKAGE = "com.springboot.oracledao"; 
   // mybatis mapper扫描路径
   static final String MAPPER_LOCATION = "classpath:mapper/oracle/*.xml";
   
   @Bean(name = "oracledatasource")
   @ConfigurationProperties("spring.datasource.druid.oracle")
   public DataSource oracleDataSource() {
      return DruidDataSourceBuilder.create().build();
   }
   
   @Bean(name = "oracleTransactionManager")
    public DataSourceTransactionManager oracleTransactionManager() {
        return new DataSourceTransactionManager(oracleDataSource());
    }
 
    @Bean(name = "oracleSqlSessionFactory")
    public SqlSessionFactory oracleSqlSessionFactory(@Qualifier("oracledatasource") DataSource dataSource) throws Exception {
        final SqlSessionFactoryBean sessionFactory = new SqlSessionFactoryBean();
        sessionFactory.setDataSource(dataSource);
        //如果不使用xml的方式配置mapper，则可以省去下面这行mapper location的配置。
        sessionFactory.setMapperLocations(new PathMatchingResourcePatternResolver()
                .getResources(OracleDatasourceConfig.MAPPER_LOCATION));
        return sessionFactory.getObject();
    }
}
```

## 若依动态数据源

```
public enum DataSourceType
{
    /**
     * 主库
     */
    MASTER,

    /**
     * 从库
     */
    SLAVE
}


/**
 * 数据源切换处理
 * 
 * @author ruoyi
 */
public class DynamicDataSourceContextHolder
{
    public static final Logger log = LoggerFactory.getLogger(DynamicDataSourceContextHolder.class);

    /**
     * 使用ThreadLocal维护变量，ThreadLocal为每个使用该变量的线程提供独立的变量副本，
     *  所以每一个线程都可以独立地改变自己的副本，而不会影响其它线程所对应的副本。
     */
    private static final ThreadLocal<String> CONTEXT_HOLDER = new ThreadLocal<>();

    /**
     * 设置数据源的变量
     */
    public static void setDataSourceType(String dsType)
    {
        log.info("切换到{}数据源", dsType);
        CONTEXT_HOLDER.set(dsType);
    }

    /**
     * 获得数据源的变量
     */
    public static String getDataSourceType()
    {
        return CONTEXT_HOLDER.get();
    }

    /**
     * 清空数据源变量
     */
    public static void clearDataSourceType()
    {
        CONTEXT_HOLDER.remove();
    }
}

public class DynamicDataSource extends AbstractRoutingDataSource
{
    public DynamicDataSource(DataSource defaultTargetDataSource, Map<Object, Object> targetDataSources)
    {
        super.setDefaultTargetDataSource(defaultTargetDataSource);
        super.setTargetDataSources(targetDataSources);
        super.afterPropertiesSet();
    }

    @Override
    protected Object determineCurrentLookupKey()
    {
        return DynamicDataSourceContextHolder.getDataSourceType();
    }
}
```

```
import org.springframework.boot.autoconfigure.condition.ConditionalOnProperty;
import org.springframework.boot.context.properties.ConfigurationProperties;
import org.springframework.boot.web.servlet.FilterRegistrationBean;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.context.annotation.Primary;
import com.alibaba.druid.pool.DruidDataSource;
import com.alibaba.druid.spring.boot.autoconfigure.DruidDataSourceBuilder;
import com.alibaba.druid.spring.boot.autoconfigure.properties.DruidStatProperties;
import com.alibaba.druid.util.Utils;
import com.ruoyi.common.enums.DataSourceType;
import com.ruoyi.common.utils.spring.SpringUtils;
import com.ruoyi.framework.config.properties.DruidProperties;
import com.ruoyi.framework.datasource.DynamicDataSource;

/**
 * druid 配置多数据源
 * 
 * @author ruoyi
 */
@Configuration
public class DruidConfig
{
    @Bean
    @ConfigurationProperties("spring.datasource.druid.master")
    public DataSource masterDataSource(DruidProperties druidProperties)
    {
        DruidDataSource dataSource = DruidDataSourceBuilder.create().build();
        return druidProperties.dataSource(dataSource);
    }

    @Bean
    @ConfigurationProperties("spring.datasource.druid.slave")
    @ConditionalOnProperty(prefix = "spring.datasource.druid.slave", name = "enabled", havingValue = "true")
    public DataSource slaveDataSource(DruidProperties druidProperties)
    {
        DruidDataSource dataSource = DruidDataSourceBuilder.create().build();
        return druidProperties.dataSource(dataSource);
    }

    @Bean(name = "dynamicDataSource")
    @Primary  //必要
    public DynamicDataSource dataSource(DataSource masterDataSource)
    {
        Map<Object, Object> targetDataSources = new HashMap<>();
        targetDataSources.put(DataSourceType.MASTER.name(), masterDataSource);
        setDataSource(targetDataSources, DataSourceType.SLAVE.name(), "slaveDataSource");
        return new DynamicDataSource(masterDataSource, targetDataSources);
    }
    
    /**
     * 设置数据源
     * 
     * @param targetDataSources 备选数据源集合
     * @param sourceName 数据源名称
     * @param beanName bean名称
     */
    public void setDataSource(Map<Object, Object> targetDataSources, String sourceName, String beanName)
    {
        try
        {
            DataSource dataSource = SpringUtils.getBean(beanName);
            targetDataSources.put(sourceName, dataSource);
        }
        catch (Exception e)
        {
        }
    }

    /**
     * 配置Druid的监控 去除监控页面底部的广告
     */
    @SuppressWarnings({ "rawtypes", "unchecked" })
    @Bean
    @ConditionalOnProperty(name = "spring.datasource.druid.statViewServlet.enabled", havingValue = "true")
    public FilterRegistrationBean removeDruidFilterRegistrationBean(DruidStatProperties properties)
    {
        // 获取web监控页面的参数
        DruidStatProperties.StatViewServlet config = properties.getStatViewServlet();
        // 提取common.js的配置路径
        String pattern = config.getUrlPattern() != null ? config.getUrlPattern() : "/druid/*";
        String commonJsPattern = pattern.replaceAll("\\*", "js/common.js");
        final String filePath = "support/http/resources/js/common.js";
        // 创建filter进行过滤
        Filter filter = new Filter()
        {
            @Override
            public void init(javax.servlet.FilterConfig filterConfig) throws ServletException
            {
            }
            @Override
            public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain)
                    throws IOException, ServletException
            {
                chain.doFilter(request, response);
                // 重置缓冲区，响应头不会被重置
                response.resetBuffer();
                // 获取common.js
                String text = Utils.readFromResource(filePath);
                // 正则替换banner, 除去底部的广告信息
                text = text.replaceAll("<a.*?banner\"></a><br/>", "");
                text = text.replaceAll("powered.*?shrek.wang</a>", "");
                response.getWriter().write(text);
            }
            @Override
            public void destroy()
            {
            }
        };
        FilterRegistrationBean registrationBean = new FilterRegistrationBean();
        registrationBean.setFilter(filter);
        registrationBean.addUrlPatterns(commonJsPattern);
        return registrationBean;
    }
}

```

```
# 数据源配置
spring:
    datasource:
        type: com.alibaba.druid.pool.DruidDataSource
        driverClassName: com.mysql.cj.jdbc.Driver
        druid:
            # 主库数据源
            master:
                url: jdbc:mysql://localhost:3306/ry-vue?useUnicode=true&characterEncoding=utf8&zeroDateTimeBehavior=convertToNull&useSSL=true&serverTimezone=GMT%2B8
                username: root
                password: 350562
            # 从库数据源
            slave:
                # 从数据源开关/默认关闭
                enabled: false
                url: 
                username: 
                password: 
            # 初始连接数
            initialSize: 5
            # 最小连接池数量
            minIdle: 10
            # 最大连接池数量
            maxActive: 20
            # 配置获取连接等待超时的时间
            maxWait: 60000
            # 配置间隔多久才进行一次检测，检测需要关闭的空闲连接，单位是毫秒
            timeBetweenEvictionRunsMillis: 60000
            # 配置一个连接在池中最小生存的时间，单位是毫秒
            minEvictableIdleTimeMillis: 300000
            # 配置一个连接在池中最大生存的时间，单位是毫秒
            maxEvictableIdleTimeMillis: 900000
            # 配置检测连接是否有效
            validationQuery: SELECT 1 FROM DUAL
            testWhileIdle: true
            testOnBorrow: false
            testOnReturn: false
            webStatFilter: 
                enabled: true
            statViewServlet:
                enabled: true
                # 设置白名单，不填则允许所有访问
                allow:
                url-pattern: /druid/*
                # 控制台管理用户名和密码
                login-username: ruoyi
                login-password: 123456
            filter:
                stat:
                    enabled: true
                    # 慢SQL记录
                    log-slow-sql: true
                    slow-sql-millis: 1000
                    merge-sql: true
                wall:
                    config:
                        multi-statement-allow: true
```



```
/**
 * 多数据源处理
 * 
 * @author ruoyi
 */
@Aspect
@Order(1)
@Component
public class DataSourceAspect
{
    protected Logger logger = LoggerFactory.getLogger(getClass());

    @Pointcut("@annotation(com.ruoyi.common.annotation.DataSource)"   //方法上持有
            + "|| @within(com.ruoyi.common.annotation.DataSource)")   //任何地方持有
    public void dsPointCut()
    {

    }

    @Around("dsPointCut()")
    public Object around(ProceedingJoinPoint point) throws Throwable
    {
        DataSource dataSource = getDataSource(point);

        if (StringUtils.isNotNull(dataSource))
        {
            DynamicDataSourceContextHolder.setDataSourceType(dataSource.value().name());
        }

        try
        {
            return point.proceed();
        }
        finally
        {
            // 销毁数据源 在执行方法之后
            DynamicDataSourceContextHolder.clearDataSourceType();
        }
    }

    /**
     * 获取需要切换的数据源
     * 因为@DataSource可写在方法或者类上
     */
    public DataSource getDataSource(ProceedingJoinPoint point)
    {
        MethodSignature signature = (MethodSignature) point.getSignature();
        DataSource dataSource = AnnotationUtils.findAnnotation(signature.getMethod(), DataSource.class);
        if (Objects.nonNull(dataSource))
        {
            return dataSource;
        }
        //getDeclaringType()
        return AnnotationUtils.findAnnotation(signature.getDeclaringType(), DataSource.class);
    }
}
```

使用方式:在需要使用多数据源方法或类上添加`@DataSource`注解，其中`value`用来表示数据源

```java
@DataSource(value = DataSourceType.SLAVE)
public List<SysUser> selectUserList(SysUser user)
{
	return userMapper.selectUserList(user);
}
```

```java
@Service
@DataSource(value = DataSourceType.SLAVE)
public class SysUserServiceImpl
```

手动切换数据源

在需要切换数据源的方法中使用`DynamicDataSourceContextHolder`类实现手动切换

```java
public List<SysUser> selectUserList(SysUser user)
{
	DynamicDataSourceContextHolder.setDataSourceType(DataSourceType.SLAVE.name());
	List<SysUser> userList = userMapper.selectUserList(user);
	DynamicDataSourceContextHolder.clearDataSourceType();
	return userList;
}
```



# redis



```
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-redis</artifactId>
</dependency>
```

```
# REDIS (RedisProperties)
# Redis数据库索引（默认为0）
spring.redis.database=0
# Redis服务器地址
spring.redis.host=localhost
# Redis服务器连接端口
spring.redis.port=6379
# Redis服务器连接密码（默认为空）
spring.redis.password=
# 连接池最大连接数（使用负值表示没有限制）
spring.redis.pool.max-active=8
# 连接池最大阻塞等待时间（使用负值表示没有限制）
spring.redis.pool.max-wait=-1
# 连接池中的最大空闲连接
spring.redis.pool.max-idle=8
# 连接池中的最小空闲连接
spring.redis.pool.min-idle=0
# 连接超时时间（毫秒）
spring.redis.timeout=0
```

**其中spring.redis.database的配置通常使用0即可，Redis在配置的时候可以设置数据库数量，默认为16，可以理解为数据库的schema**

```
@RunWith(SpringJUnit4ClassRunner.class)
@SpringApplicationConfiguration(Application.class)
public class ApplicationTests {
	@Autowired  //StringRedisTemplate就相当于RedisTemplate<String, String>的实现。
	private StringRedisTemplate stringRedisTemplate;
	@Test
	public void test() throws Exception {
		// 保存字符串
		stringRedisTemplate.opsForValue().set("aaa", "111");
		Assert.assertEquals("111", stringRedisTemplate.opsForValue().get("aaa"));
    }
}
```

## RedisTemplate封装

```
@SuppressWarnings(value = { "unchecked", "rawtypes" })
@Component
public class RedisCache
{
    @Autowired
    public RedisTemplate redisTemplate;

    /**
     * 缓存基本的对象，Integer、String、实体类等
     *
     * @param key 缓存的键值
     * @param value 缓存的值
     */
    public <T> void setCacheObject(final String key, final T value)
    {
        redisTemplate.opsForValue().set(key, value);
    }

    /**
     * 缓存基本的对象，Integer、String、实体类等
     *
     * @param key 缓存的键值
     * @param value 缓存的值
     * @param timeout 时间
     * @param timeUnit 时间颗粒度
     */
    public <T> void setCacheObject(final String key, final T value, final Integer timeout, final TimeUnit timeUnit)
    {
        redisTemplate.opsForValue().set(key, value, timeout, timeUnit);
    }

    /**
     * 设置有效时间
     *
     * @param key Redis键
     * @param timeout 超时时间
     * @return true=设置成功；false=设置失败
     */
    public boolean expire(final String key, final long timeout)
    {
        return expire(key, timeout, TimeUnit.SECONDS);
    }

    /**
     * 设置有效时间
     *
     * @param key Redis键
     * @param timeout 超时时间
     * @param unit 时间单位
     * @return true=设置成功；false=设置失败
     */
    public boolean expire(final String key, final long timeout, final TimeUnit unit)
    {
        return redisTemplate.expire(key, timeout, unit);
    }

    /**
     * 获得缓存的基本对象。
     *
     * @param key 缓存键值
     * @return 缓存键值对应的数据
     */
    public <T> T getCacheObject(final String key)
    {
        ValueOperations<String, T> operation = redisTemplate.opsForValue();
        return operation.get(key);
    }

    /**
     * 删除单个对象
     *
     * @param key
     */
    public boolean deleteObject(final String key)
    {
        return redisTemplate.delete(key);
    }

    /**
     * 删除集合对象
     *
     * @param collection 多个对象
     * @return
     */
    public long deleteObject(final Collection collection)
    {
        return redisTemplate.delete(collection);
    }

    /**
     * 缓存List数据
     *
     * @param key 缓存的键值
     * @param dataList 待缓存的List数据
     * @return 缓存的对象
     */
    public <T> long setCacheList(final String key, final List<T> dataList)
    {
        Long count = redisTemplate.opsForList().rightPushAll(key, dataList);
        return count == null ? 0 : count;
    }

    /**
     * 获得缓存的list对象
     *
     * @param key 缓存的键值
     * @return 缓存键值对应的数据
     */
    public <T> List<T> getCacheList(final String key)
    {
        return redisTemplate.opsForList().range(key, 0, -1);
    }

    /**
     * 缓存Set
     *
     * @param key 缓存键值
     * @param dataSet 缓存的数据
     * @return 缓存数据的对象
     */
    public <T> BoundSetOperations<String, T> setCacheSet(final String key, final Set<T> dataSet)
    {
        BoundSetOperations<String, T> setOperation = redisTemplate.boundSetOps(key);
        Iterator<T> it = dataSet.iterator();
        while (it.hasNext())
        {
            setOperation.add(it.next());
        }
        return setOperation;
    }

    /**
     * 获得缓存的set
     *
     * @param key
     * @return
     */
    public <T> Set<T> getCacheSet(final String key)
    {
        return redisTemplate.opsForSet().members(key);
    }

    /**
     * 缓存Map
     *
     * @param key
     * @param dataMap
     */
    public <T> void setCacheMap(final String key, final Map<String, T> dataMap)
    {
        if (dataMap != null) {
            redisTemplate.opsForHash().putAll(key, dataMap);
        }
    }

    /**
     * 获得缓存的Map
     *
     * @param key
     * @return
     */
    public <T> Map<String, T> getCacheMap(final String key)
    {
        return redisTemplate.opsForHash().entries(key);
    }

    /**
     * 往Hash中存入数据
     *
     * @param key Redis键
     * @param hKey Hash键
     * @param value 值
     */
    public <T> void setCacheMapValue(final String key, final String hKey, final T value)
    {
        redisTemplate.opsForHash().put(key, hKey, value);
    }

    /**
     * 获取Hash中的数据
     *
     * @param key Redis键
     * @param hKey Hash键
     * @return Hash中的对象
     */
    public <T> T getCacheMapValue(final String key, final String hKey)
    {
        HashOperations<String, String, T> opsForHash = redisTemplate.opsForHash();
        return opsForHash.get(key, hKey);
    }

    /**
     * 获取多个Hash中的数据
     *
     * @param key Redis键
     * @param hKeys Hash键集合
     * @return Hash对象集合
     */
    public <T> List<T> getMultiCacheMapValue(final String key, final Collection<Object> hKeys)
    {
        return redisTemplate.opsForHash().multiGet(key, hKeys);
    }

    /**
     * 获得缓存的基本对象列表
     *
     * @param pattern 字符串前缀
     * @return 对象列表
     */
    public Collection<String> keys(final String pattern)
    {
        return redisTemplate.keys(pattern);
    }
}
```



## RedisSerializer

redisTemplate可以指定序列化器(通常key序列化采用自带的`StringRedisSerializer`即可, 而value的序列化可以实现多种定制)，实现RedisSerializer接口即刻,

```
public class User implements Serializable {
    private static final long serialVersionUID = -1L;
    private String username;
    private Integer age;
    public User(String username, Integer age) {
        this.username = username;
        this.age = age;
    }
    // 省略getter和setter
}
```

```
//配置对象序列化器  实现redis序列化接口
public class RedisObjectSerializer implements RedisSerializer<Object> {
  private Converter<Object, byte[]> serializer = new SerializingConverter();
  private Converter<byte[], Object> deserializer = new DeserializingConverter();

  static final byte[] EMPTY_ARRAY = new byte[0];

  public Object deserialize(byte[] bytes) {
    if (isEmpty(bytes)) {
      return null;
    }

    try {
      return deserializer.convert(bytes);
    } catch (Exception ex) {
      throw new SerializationException("Cannot deserialize", ex);
    }
  }

  public byte[] serialize(Object object) {
    if (object == null) {
      return EMPTY_ARRAY;
    }

    try {
      return serializer.convert(object);
    } catch (Exception ex) {
      return EMPTY_ARRAY;
    }
  }

  private boolean isEmpty(byte[] data) {
    return (data == null || data.length == 0);
  }
}
```

配置

```

@Configuration
public class RedisConfig {

    @Bean
    JedisConnectionFactory jedisConnectionFactory() {
        return new JedisConnectionFactory();
    }

    @Bean
    public RedisTemplate<String, User> redisTemplate(RedisConnectionFactory factory) {
        RedisTemplate<String, User> template = new RedisTemplate<String, User>();
        template.setConnectionFactory(jedisConnectionFactory());
        template.setKeySerializer(new StringRedisSerializer());
        template.setValueSerializer(new RedisObjectSerializer());
        return template;
    }


}
```

```
//测试
@RunWith(SpringJUnit4ClassRunner.class)
@SpringApplicationConfiguration(Application.class)
public class ApplicationTests {

	@Autowired
	private RedisTemplate<String, User> redisTemplate;

	@Test
	public void test() throws Exception {

		// 保存对象
		User user = new User("超人", 20);
		redisTemplate.opsForValue().set(user.getUsername(), user);

		user = new User("蝙蝠侠", 30);
		redisTemplate.opsForValue().set(user.getUsername(), user);

		user = new User("蜘蛛侠", 40);
		redisTemplate.opsForValue().set(user.getUsername(), user);

		Assert.assertEquals(20, redisTemplate.opsForValue().get("超人").getAge().longValue());
		Assert.assertEquals(30, redisTemplate.opsForValue().get("蝙蝠侠").getAge().longValue());
		Assert.assertEquals(40, redisTemplate.opsForValue().get("蜘蛛侠").getAge().longValue());

	}

}
```

**使用Jackson提供的Jackson2JsonRedisSerializer**

```
@Bean
    @SuppressWarnings("all")
    public RedisTemplate<String, Object> redisTemplate(RedisConnectionFactory factory) {
        RedisTemplate<String, Object> template = new RedisTemplate<String, Object>();
        template.setConnectionFactory(factory);
        Jackson2JsonRedisSerializer jackson2JsonRedisSerializer = new Jackson2JsonRedisSerializer(Object.class);
        ObjectMapper om = new ObjectMapper();
        om.setVisibility(PropertyAccessor.ALL, JsonAutoDetect.Visibility.ANY);
        om.enableDefaultTyping(ObjectMapper.DefaultTyping.NON_FINAL);
        jackson2JsonRedisSerializer.setObjectMapper(om);
        StringRedisSerializer stringRedisSerializer = new StringRedisSerializer();
        // key采用String的序列化方式
        template.setKeySerializer(stringRedisSerializer);
        // hash的key也采用String的序列化方式
        template.setHashKeySerializer(stringRedisSerializer);
        // value序列化方式采用jackson
        template.setValueSerializer(jackson2JsonRedisSerializer);
        // hash的value序列化方式采用jackson
        template.setHashValueSerializer(jackson2JsonRedisSerializer);
        template.afterPropertiesSet();
        return template;
    }
```

**使用fastjson序列化**

```
public class FastJsonRedisSerializer<T> implements RedisSerializer<T> {

    private final Class<T> clazz;

    FastJsonRedisSerializer(Class<T> clazz) {
        super();
        this.clazz = clazz;
    }

    @Override
    public byte[] serialize(T t) {
        if (t == null) {
            return new byte[0];
        }
        return JSON.toJSONString(t, SerializerFeature.WriteClassName).getBytes(StandardCharsets.UTF_8);
    }

    @Override
    public T deserialize(byte[] bytes) {
        if (bytes == null || bytes.length <= 0) {
            return null;
        }
        String str = new String(bytes, StandardCharsets.UTF_8);
        return JSON.parseObject(str, clazz);
    }

}
```





## 获取连接信息

```
//在当前连接下执行动作 这里为获取连接信息
        Properties info = (Properties) redisTemplate.execute((RedisCallback<Object>) RedisServerCommands::info);
        //仅获取命令信息
        Properties commandStats = (Properties) redisTemplate.execute((RedisCallback<Object>) connection -> connection.info("commandstats"));
        //数据大小 相当于key数量
        Object dbSize = redisTemplate.execute((RedisCallback<Object>) RedisServerCommands::dbSize);
```



# EhCache

```
<dependency>
    <groupId>net.sf.ehcache</groupId>
    <artifactId>ehcache</artifactId>
</dependency>
```

```
spring.cache.ehcache.config.classpath:ehcache.xml
```

```
<ehcache xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
   xsi:noNamespaceSchemaLocation="ehcache.xsd">
   <!--timeToIdleSeconds 当缓存闲置n秒后销毁 -->
   <!--timeToLiveSeconds 当缓存存活n秒后销毁 -->
   <!-- 缓存配置 name:缓存名称。 maxElementsInMemory：缓存最大个数。 eternal:对象是否永久有效，一但设置了，timeout将不起作用。 
      timeToIdleSeconds：设置对象在失效前的允许闲置时间（单位：秒）。仅当eternal=false对象不是永久有效时使用，可选属性，默认值是0，也就是可闲置时间无穷大。 
      timeToLiveSeconds：设置对象在失效前允许存活时间（单位：秒）。最大时间介于创建时间和失效时间之间。仅当eternal=false对象不是永久有效时使用，默认是0.，也就是对象存活时间无穷大。 
      overflowToDisk：当内存中对象数量达到maxElementsInMemory时，Ehcache将会对象写到磁盘中。 diskSpoolBufferSizeMB：这个参数设置DiskStore（磁盘缓存）的缓存区大小。默认是30MB。每个Cache都应该有自己的一个缓冲区。 
      maxElementsOnDisk：硬盘最大缓存个数。 diskPersistent：是否缓存虚拟机重启期数据 Whether the disk 
      store persists between restarts of the Virtual Machine. The default value 
      is false. diskExpiryThreadIntervalSeconds：磁盘失效线程运行时间间隔，默认是120秒。 memoryStoreEvictionPolicy：当达到maxElementsInMemory限制时，Ehcache将会根据指定的策略去清理内存。默认策略是 
      LRU（最近最少使用）。你可以设置为FIFO（先进先出）或是LFU（较少使用）。 clearOnFlush：内存数量最大时是否清除。 maxEntriesLocalHeap="1000" 
      : 堆内存中最大缓存对象数,0没有限制(必须设置) maxEntriesLocalDisk="1000" : 硬盘最大缓存个数。 -->
   <!-- 磁盘缓存位置  user.dir获取当前项目路径的地址-->
   <diskStore path="user.dir/cachedata" />

   <!-- 默认缓存 -->
   <defaultCache maxElementsInMemory="10000" eternal="false"
      timeToIdleSeconds="120" timeToLiveSeconds="120"
      maxElementsOnDisk="10000000" diskExpiryThreadIntervalSeconds="120"
      memoryStoreEvictionPolicy="LRU">

      <persistence strategy="localTempSwap" />
   </defaultCache>


   <cache name="util_cache" maxEntriesLocalHeap="0" eternal="true"
         overflowToDisk="true" diskPersistent="true" />
</ehcache>
```

## 工具类

```
@Service
public class EhCacheUtil implements CommonCacheUtil {

	@Autowired
	private  CacheManager cacheManager ;
	private static final String CACHE_NAME = "util_cache";
	/**
	 * 获得一个Cache，没有则创建一个。
	 * @return
	 */
	private Cache getCache(){
	
		Cache cache = cacheManager.getCache(CACHE_NAME);
		return cache;
	}
		
	@Override
	public String get(String key) {
		Element element = getCache().get(key);
		return element==null?null:(String)element.getObjectValue();
	}
	@Override
	public void set(String key, String value) {
		Element element = new Element(key, value);
		Cache cache = getCache();
		//不过期
		cache.getCacheConfiguration().setEternal(true);
		cache.put(element);

	}
	@Override
	public void set(String key, String value, long timeout) {
		Element element = new Element(key, value);
		element.setTimeToLive((int) timeout);
		Cache cache = getCache();
		cache.put(element);

	}
	@Override
	public void del(String key) {
		getCache().remove(key);


	}
	@Override
	public boolean contains(String key) {
		return getCache().isKeyInCache(key);
	}
	@Override
	public void expire(String key, long timeout) {
		Element element = getCache().get(key);
		if (element != null) {
			Object value = element.getValue();
			element = new Element(key, value);
			element.setTimeToLive((int)timeout);
			Cache cache = getCache();
			cache.put(element);
		}
	}
	/**
	 * 根据key获取缓存的Object类型数据
	 */
	@Override
	public Object getObject(String key) {
		Element element = getCache().get(key);
		return element==null?null:element.getObjectValue();
	}
	/**
	 * 设置Object类型的缓存
	 */
	@Override
	public void setObject(String key, Object value) {
		Element element = new Element(key, value);
		Cache cache = getCache();
		//不过期
		cache.getCacheConfiguration().setEternal(true);
		cache.put(element);
		
	}
	/**
	 * 设置一个有过期时间的Object类型的缓存,单位秒
	 */
	@Override
	public void setObject(String key, Object value, long timeout) {
		Element element = new Element(key, value);
		element.setTimeToLive((int) timeout);
		Cache cache = getCache();
		cache.put(element);
		
	}
	/**
	 * 刷新过期时间
	 * */
	@Override
	public void refresh(String key) {
		Element element = getCache().get(key);
		if (element != null) {
			Object value = element.getValue();
			int timeToLive = element.getTimeToLive();
			element = new Element(key, value);
			element.setTimeToLive(timeToLive);
			Cache cache = getCache();
			cache.put(element);
		}
		
	}
}
```

# Guava cache

进程内缓存

**构造 Cache 对象**

方式1 CacheLoader

构造 LoadingCache 的关键在于实现 load 方法，也就是在需要**访问的缓存项不存在的时候 Cache 会自动调用 load 方法将数据加载到 Cache** **中**。

除了实现 load 方法之外还可以配置缓存相关的一些性质，比如过期加载策略、刷新策略 。

```

private static final LoadingCache<String, String> CACHE = CacheBuilder
    .newBuilder()
    // 最大容量为 100 超过容量有对应的淘汰机制，下文详述
    .maximumSize(100)
    // 缓存项写入后多久过期，下文详述
    .expireAfterWrite(60 * 5, TimeUnit.SECONDS)
    // 缓存写入后多久自动刷新一次，下文详述
    .refreshAfterWrite(60, TimeUnit.SECONDS)
    // 创建一个 CacheLoader，load 表示缓存不存在的时候加载到缓存并返回
    .build(new CacheLoader<String, String>() {
        // 加载缓存数据的方法
        @Override
        public String load(String key) {
            return "cache [" + key + "]";
        }
    });

public void getTest() throws Exception {
    CACHE.get("KEY_25487");
}
```

方式2 Callable

除了在构造 Cache 对象的时候指定 load 方法来加载缓存外，我们亦可以在**获取缓存项时指定载入缓存的方法**，并且可以根据使用场景在不同的位置采用不同的加载方式。比如在某些位置可以通过二级缓存加载不存在的缓存项，而有些位置则可以直接从 DB 加载缓存项。

```

// 注意返回值是 Cache
private static final Cache<String, String> SIMPLE_CACHE = CacheBuilder
    .newBuilder()
    .build();

public void getTest1() throws Exception {
    String key = "KEY_25487";
    // get 缓存项的时候指定 callable 加载缓存项
    SIMPLE_CACHE.get(key, () -> "cache [" + key + "]");
}
```

## 机制

**数据项加载机制**

如果某个缓存过期了或者缓存项不存在于缓存中，而恰巧此此时有大量请求过来请求这个缓存项，如果没有保护机制就会导致**大量的线程同时请求数据源加载数据并生成缓存项，这就是所谓的 “缓存击穿”** 。

举个简单的例子，某个时刻有 100 个请求同时请求 KEY_25487 这个缓存项，而不巧这个缓存项刚好失效了，那么这 100 个线程（如果有这么多机器和流量的话）就会同时从 DB 加载这个数据，很可怕的点在于就算某一个线程率先获取到数据生成了缓存项，**其他的线程还是继续请求 DB 而不会走到缓存**。

如果多个线程过来如果**我们只让一个线程去加载数据生成缓存项，其他线程等待然后读取生成好的缓存项**岂不是就完美解决。那么恭喜你在这个问题上，和 Google 工程师的思路是一致的。不过采用这个方案，问题是解了但没有完全解，后面会说到它的缺陷。

其实 Guava Cache 在 load 的时候做了并发控制，在多个线程请求一个不存在或者过期的缓存项时保证只有一个线程进入 load 方法，其他线程等待直到缓存项被生成，这样就避免了大量的线程击穿缓存直达 DB 。不过试想下如果有上万 QPS 同时过来会有**大量的线程阻塞导致线程无法释放，甚至会出现线程池满的尴尬场景，这也是说为什么这个方案解了 “缓存击穿” 问题但又没完全解**。

上述机制其实就是 expireAfterWrite/expireAfterAccess 来控制的，如果你配置了过期策略对应的缓存项在过期后被访问就会走上述流程来加载缓存项。



**缓存项刷新机制**

缓存项的刷新和加载看起来是相似的，都是让缓存数据处于最新的状态。区别在于：

1. **缓存项加载是一个被动**的过程，而**缓存刷新是一个主动触发**动作。如果缓存项不存在或者过期只有下次 get 的时候才会触发新值加载。而缓存刷新则更加主动替换缓存中的老值。
2. 另外一个很重要点的在于，**缓存刷新的项目一定是存在缓存中**的，他是对老值的替换而非是对 NULL 值的替换。

由于缓存项刷新的前提是该缓存项存在于缓存中，那么缓存的刷新就不用像缓存加载的流程一样让其他线程等待而是允许一个线程去数据源获取数据，**其他线程都先返回老值直到异步线程生成了新缓存项**。

这个方案完美解决了上述遇到的 “缓存击穿” 问题，不过**他的前提是已经生成缓存项了**。在实际生产情况下我们可以做 缓存预热 ，提前生成缓存项，避免流量洪峰造成的线程堆积。

这套机制在 Guava Cache 中是通过 refreshAfterWrite 实现的，在配置刷新策略后，对应的缓存项会按照设定的时间定时刷新，避免线程阻塞的同时保证缓存项处于最新状态。

但他也不是完美的，比如他的限制是缓存项已经生成，并且**如果恰巧你运气不好，大量的缓存项同时需要刷新或者过期， 就会有大量的线程请求 DB，这就是常说的 “缓存血崩”**。



**缓存项异步刷新机制**

上面说到缓存项大面积失效或者刷新会导致雪崩，那么就只能限制访问 DB 的数量了，位置有三个地方：

1. 源头：因为加载缓存的线程就是前台请求线程，所以如果**控制请求线程数量**的确是减少大面积失效对 DB 的请求，那这样一来就不存在高并发请求，就算不用缓存都可以。
2. 中间层缓冲：因为请求线程和访问 DB 的线程是同一个，假如在**中间加一层缓冲，通过一个后台线程池去异步刷新缓存**所有请求线程直接返回老值，这样对于 DB 的访问的流量就可以被后台线程池的池大小控住。
3. 底层：直接**控 DB 连接池的池大小**，这样访问 DB 的连接数自然就少了，但是如果大量请求到连接池发现获取不到连接程序一样会出现连接池满的问题，会有大量连接被拒绝的异常。

所以比较合适的方式是通过添加一个异步线程池异步刷新数据，在 **Guava Cache 中实现方案是重写 CacheLoader 的 reload 方法**。

```
private static final LoadingCache<String, String> ASYNC_CACHE = CacheBuilder.newBuilder()
    .build(
    CacheLoader.asyncReloading(new CacheLoader<String, String>() {
        @Override
        public String load(String key) {
            return key;
        }

        @Override
        public ListenableFuture<String> reload(String key, String oldValue) throws Exception {
            return super.reload(key, oldValue);
        }
    }, new ThreadPoolExecutor(5, Integer.MAX_VALUE,
                              60L, TimeUnit.SECONDS,
                              new SynchronousQueue<>()))
);
```

## 源码

先整体看下 Cache 的类结构，下面的这些子类表示了不同的创建方式本质还都是 LocalCache

![](./picture/guava-cacahe-01.jpg)

核心代码都在 LocalCache 这个文件中，并且通过这个继承关系可以看出 Guava Cache 的本质就是 ConcurrentMap。

![guava-cache-02](./picture/guava-cache-02.png)



![guava-cache-03](./picture/guava-cache-03.jpg)

**核心Get的流程**

LocalCache#get

```
V get(K key, CacheLoader<? super K, V> loader) throws ExecutionException {
    int hash = hash(checkNotNull(key));
    // 根据 hash 获取对应的 segment 然后从 segment 获取具体值
    return segmentFor(hash).get(key, hash, loader);
}
```

Segment#get

```
V get(K key, int hash, CacheLoader<? super K, V> loader) throws ExecutionException {
    checkNotNull(key);
    checkNotNull(loader);
    try {
        // count 表示在这个 segment 中存活的项目个数
        if (count != 0) {
            // 获取 segment 中的元素 (ReferenceEntry) 包含正在 load 的数据
            ReferenceEntry<K, V> e = getEntry(key, hash);
            if (e != null) {
                long now = map.ticker.read();
                // 获取缓存值，如果是 load，invalid，expired 返回 null，同时检查是否过期了,过期移除并返回 null
                V value = getLiveValue(e, now);
                if (value != null) {
                    // 记录访问时间
                    recordRead(e, now);
                    // 记录缓存命中一次
                    statsCounter.recordHits(1);
                    // 刷新缓存并返回缓存值 ，后面展开
                    return scheduleRefresh(e, key, hash, value, now, loader);
                }
                ValueReference<K, V> valueReference = e.getValueReference();
                // 如果在 loading 等着 ，后面展开
                if (valueReference.isLoading()) {
                    return waitForLoadingValue(e, key, valueReference);
                }
            }
        }

        // 走到这说明从来没写入过值 或者 值为 null 或者 过期（数据还没做清理），后面展开
        return lockedGetOrLoad(key, hash, loader);
    } catch (ExecutionException ee) {
        Throwable cause = ee.getCause();
        if (cause instanceof Error) {
            throw new ExecutionError((Error) cause);
        } else if (cause instanceof RuntimeException) {
            throw new UncheckedExecutionException(cause);
        }
        throw ee;
    } finally {
        postReadCleanup();
    }
}
```

**Segment#scheduleRefresh**

```
V scheduleRefresh(
    ReferenceEntry<K, V> entry,
    K key,
    int hash,
    V oldValue,
    long now,
    CacheLoader<? super K, V> loader) {
    
    if (
        // 配置了刷新策略 refreshAfterWrite
        map.refreshes()
        // 到刷新时间了
        && (now - entry.getWriteTime() > map.refreshNanos)
        // 没在 loading
        && !entry.getValueReference().isLoading()) {
        // 开始刷新，下面展开
        V newValue = refresh(key, hash, loader, true);
        if (newValue != null) {
            return newValue;
        }
    }
    return oldValue;
}



V refresh(K key, int hash, CacheLoader<? super K, V> loader, boolean checkTime) {
    // 插入 loading 节点
    final LoadingValueReference<K, V> loadingValueReference =
        insertLoadingValueReference(key, hash, checkTime);
    
    if (loadingValueReference == null) {
        return null;
    }

    // 异步刷新，下面展开
    ListenableFuture<V> result = loadAsync(key, hash, loadingValueReference, loader);
    if (result.isDone()) {
        try {
            return Uninterruptibles.getUninterruptibly(result);
        } catch (Throwable t) {
            // don't let refresh exceptions propagate; error was already logged
        }
    }
    return null;
}

// com.google.common.cache.LocalCache.Segment#loadAsync

ListenableFuture<V> loadAsync(
    final K key,
    final int hash,
    final LoadingValueReference<K, V> loadingValueReference,
    CacheLoader<? super K, V> loader) {
    // 通过 loader 异步加载数据，下面展开
    final ListenableFuture<V> loadingFuture = loadingValueReference.loadFuture(key, loader);
    loadingFuture.addListener(
        new Runnable() {
            @Override
            public void run() {
                try {
                    getAndRecordStats(key, hash, loadingValueReference, loadingFuture);
                } catch (Throwable t) {
                    logger.log(Level.WARNING, "Exception thrown during refresh", t);
                    loadingValueReference.setException(t);
                }
            }
        },
        directExecutor());
    return loadingFuture;
}

// com.google.common.cache.LocalCache.LoadingValueReference#loadFuture

public ListenableFuture<V> loadFuture(K key, CacheLoader<? super K, V> loader) {
    try {
        stopwatch.start();
        // oldValue 指在写入 loading 节点前这个位置的值，如果这个位置之前没有值 oldValue 会被赋值为 UNSET
        // UNSET.get() 值为 null ，所以这个缓存项从来没有进入缓存需要同步 load 具体原因前面提到了，如果通过
        // 异步 reload ，由于没有老值会导致其他线程返回的都是 null
        V previousValue = oldValue.get();
        if (previousValue == null) {
            V newValue = loader.load(key);
            return set(newValue) ? futureValue : Futures.immediateFuture(newValue);
        }
        // 异步 load
        ListenableFuture<V> newValue = loader.reload(key, previousValue);
        if (newValue == null) {
            return Futures.immediateFuture(null);
        }
        // To avoid a race, make sure the refreshed value is set into loadingValueReference
        // *before* returning newValue from the cache query.
        return transform(
            newValue,
            new com.google.common.base.Function<V, V>() {
                @Override
                public V apply(V newValue) {
                    LoadingValueReference.this.set(newValue);
                    return newValue;
                }
            },
            directExecutor());
    } catch (Throwable t) {
        ListenableFuture<V> result = setException(t) ? futureValue : fullyFailedFuture(t);
        if (t instanceof InterruptedException) {
            Thread.currentThread().interrupt();
        }
        return result;
    }
}
```

 **Segment#waitForLoadingValue**

```

V waitForLoadingValue(ReferenceEntry<K, V> e, K key, ValueReference<K, V> valueReference)
    throws ExecutionException {
    // 首先你要是一个 loading 节点
    if (!valueReference.isLoading()) {
        throw new AssertionError();
    }

    checkState(!Thread.holdsLock(e), "Recursive load of: %s", key);
    // don't consider expiration as we're concurrent with loading
    try {
        V value = valueReference.waitForValue();
        if (value == null) {
            throw new InvalidCacheLoadException("CacheLoader returned null for key " + key + ".");
        }
        // re-read ticker now that loading has completed
        long now = map.ticker.read();
        recordRead(e, now);
        return value;
    } finally {
        statsCounter.recordMisses(1);
    }
}

// com.google.common.cache.LocalCache.LoadingValueReference#waitForValue

public V waitForValue() throws ExecutionException {
    return getUninterruptibly(futureValue);
}

// com.google.common.util.concurrent.Uninterruptibles#getUninterruptibly

public static <V> V getUninterruptibly(Future<V> future) throws ExecutionException {
    boolean interrupted = false;
    try {
        while (true) {
            try {
                // hang 住，如果该线程被打断了继续回去 hang 住等结果，直到有结果返回
                return future.get();
            } catch (InterruptedException e) {
                interrupted = true;
            }
        }
    } finally {
        if (interrupted) {
            Thread.currentThread().interrupt();
        }
    }
}
```

**Segment#lockedGetOrLoad**

```

V lockedGetOrLoad(K key, int hash, CacheLoader<? super K, V> loader) throws ExecutionException {
    ReferenceEntry<K, V> e;
    ValueReference<K, V> valueReference = null;
    LoadingValueReference<K, V> loadingValueReference = null;
    boolean createNewEntry = true;

    // 要对 segment 写操作 ，先加锁
    lock();
    try {
        // re-read ticker once inside the lock
        long now = map.ticker.read();
        preWriteCleanup(now);

        // 这里基本就是 HashMap 的代码，如果没有 segment 的数组下标冲突了就拉一个链表
        int newCount = this.count - 1;
        AtomicReferenceArray<ReferenceEntry<K, V>> table = this.table;
        int index = hash & (table.length() - 1);
        ReferenceEntry<K, V> first = table.get(index);

        for (e = first; e != null; e = e.getNext()) {
            K entryKey = e.getKey();
            if (e.getHash() == hash
                && entryKey != null
                && map.keyEquivalence.equivalent(key, entryKey)) {
                valueReference = e.getValueReference();

                // 如果在加载中 不做任何处理
                if (valueReference.isLoading()) {
                    createNewEntry = false;
                } else {
                    V value = valueReference.get();
                    // 如果缓存项为 null 数据已经被删除，通知对应的 queue 
                    if (value == null) {
                        enqueueNotification(
                            entryKey, hash, value, valueReference.getWeight(), RemovalCause.COLLECTED);
                    // 这个是 double check 如果缓存项过期 数据没被删除，通知对应的 queue 
                    } else if (map.isExpired(e, now)) {
                        // This is a duplicate check, as preWriteCleanup already purged expired
                        // entries, but let's accommodate an incorrect expiration queue.
                        enqueueNotification(
                            entryKey, hash, value, valueReference.getWeight(), RemovalCause.EXPIRED);
                    // 再次看到的时候这个位置有值了直接返回 
                    } else {
                        recordLockedRead(e, now);
                        statsCounter.recordHits(1);
                        return value;
                    }

                    // immediately reuse invalid entries
                    writeQueue.remove(e);
                    accessQueue.remove(e);
                    this.count = newCount; // write-volatile
                }
                break;
            }
        }

        // 没有 loading ，创建一个 loading 节点
        if (createNewEntry) {
            loadingValueReference = new LoadingValueReference<>();

            if (e == null) {
                e = newEntry(key, hash, first);
                e.setValueReference(loadingValueReference);
                table.set(index, e);
            } else {
                e.setValueReference(loadingValueReference);
            }
        }
    } finally {
        unlock();
        postWriteCleanup();
    }

    if (createNewEntry) {
        try {
            // Synchronizes on the entry to allow failing fast when a recursive load is
            // detected. This may be circumvented when an entry is copied, but will fail fast most
            // of the time.
            synchronized (e) {
                return loadSync(key, hash, loadingValueReference, loader);
            }
        } finally {
            statsCounter.recordMisses(1);
        }
    } else {
        // The entry already exists. Wait for loading.
        return waitForLoadingValue(e, key, valueReference);
    }
}
```

总结:**首先判断缓存是否过期然后判断是否需要刷新，如果过期了就调用 loading 去同步加载数据（其他线程阻塞），如果是仅仅需要刷新调用 reloading 异步加载（其他线程返回老值）**。

如果 refreshTime > expireTime 意味着永远走不到缓存刷新逻辑，缓存刷新是为了在缓存有效期内尽量保证缓存数据一致性所以在配置刷新策略和过期策略时一定保证 refreshTime < expireTime 。

最佳实践:

```
1.如果刷新时间配置的较短一定要重载 reload 异步加载数据的方法，传入一个自定义线程池保护 DB
2.失效时间一定要大于刷新时间
3.如果是常驻内存的一些少量数据失效时间可以配置的较长刷新时间配置短一点 (根据业务对缓存失效容忍度)
```



# mongodb

连接工具MongodbCompass

我们可以直接用MongoDB来存储键值对类型的数据，如：验证码、Session等；由于MongoDB的横向扩展能力，也可以用来存储数据规模会在未来变的非常巨大的数据，如：日志、评论等；由于MongoDB存储数据的弱类型，也可以用来存储一些多变json数据，如：与外系统交互时经常变化的JSON报文。而对于一些对数据有复杂的高事务性要求的操作，如：账户交易等就不适合使用MongoDB来存储。

```
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-mongodb</artifactId>
</dependency>
```

若MongoDB的安装配置采用默认端口，那么在自动配置的情况下，我们不需要做任何参数配置;

否则spring.data.mongodb.uri=mongodb://name:pass@localhost:27017/test

mongodb 2.x 也支持 :spring.data.mongodb.host=localhost spring.data.mongodb.port=27017

或者

```
spring:
  data:
    mongodb:
      host: localhost
      port: 27017
      database: testdb
```



```
//创建要存储的User实体，包含属性：id、username、age
public class User {
    @Id
    private Long id;
    private String username;
    private Integer age;
    public User(Long id, String username, Integer age) {
        this.id = id;
        this.username = username;
        this.age = age;
    }
    // 省略getter和setter
}
```

```
//在Spring Data的抽象下，同其他Spring Data子项目一样的简洁、好用、易学！
public interface UserRepository extends MongoRepository<User, Long> {

    User findByUsername(String username);
}
```

测试

```
@Test
	public void test() throws Exception {

		// 创建三个User，并验证User总数
		userRepository.save(new User(1L, "didi", 30));
		userRepository.save(new User(2L, "mama", 40));
		userRepository.save(new User(3L, "kaka", 50));
		Assertions.assertEquals(3, userRepository.findAll().size());
		//Assert.assertEquals(3, userRepository.findAll().size());

		// 删除一个User，再验证User总数
		User u = userRepository.findOne(1L);
		userRepository.delete(u);
		Assertions.assertEquals(3, userRepository.findAll().size());
		//Assert.assertEquals(2, userRepository.findAll().size());

		// 删除一个User，再验证User总数
		u = userRepository.findByUsername("mama");
		userRepository.delete(u);
		Assertions.assertEquals(3, userRepository.findAll().size());
		/Assert.assertEquals(1, userRepository.findAll().size());

	}
```

其他参数默认值

```
spring.data.mongodb.option.min-connection-per-host=0
spring.data.mongodb.option.max-connection-per-host=100
spring.data.mongodb.option.threads-allowed-to-block-for-connection-multiplier=5
spring.data.mongodb.option.server-selection-timeout=30000
spring.data.mongodb.option.max-wait-time=120000
spring.data.mongodb.option.max-connection-idle-time=0
spring.data.mongodb.option.max-connection-life-time=0
spring.data.mongodb.option.connect-timeout=10000
spring.data.mongodb.option.socket-timeout=0

spring.data.mongodb.option.socket-keep-alive=false
spring.data.mongodb.option.ssl-enabled=false
spring.data.mongodb.option.ssl-invalid-host-name-allowed=false
spring.data.mongodb.option.always-use-m-beans=false

spring.data.mongodb.option.heartbeat-socket-timeout=20000
spring.data.mongodb.option.heartbeat-connect-timeout=20000
spring.data.mongodb.option.min-heartbeat-frequency=500
spring.data.mongodb.option.heartbeat-frequency=10000
spring.data.mongodb.option.local-threshold=15
```

## 多条件查询

其实``通过继承`MongoRepository`已经具有了JPA的特性，我们可以通过方法名来构建多查询条件的SQL。

在输入`findBy`后，IDEA会根据实体对象的属性和SQL的各种关键字自动组合提示：

```
@Repository
public interface UserDao extends MongoRepository<User, String> {
    /**
     * 根据年龄段来查找
     * @param from from
     * @param to   to
     * @return List<User>
     */
    List<User> findByAgeBetween(Integer from, Integer to);
}
```

更复杂的:

```
/**
 * 通过年龄段，用户名，描述（模糊查询）
 * @param from        from
 * @param to          to
 * @param name        name
 * @param description description
 * @return List<User>
 */
List<User> findByAgeBetweenAndNameEqualsAndDescriptionIsLike(Integer from, Integer to, String name, String description);
```

方法参数个数需要和方法名中所需要的参数个数对应上。

## 排序和分页

排序和分页需要使用`MongoTemplate`(也可指定多条件)对象来完成，在`UserService`里新增一个`getUserByCondition`方法：

```
@Autowired
private MongoTemplate template;

public Page<User> getUserByCondition(int size, int page, User user) {
    Query query = new Query();
    Criteria criteria = new Criteria();

    if (!StringUtils.isEmpty(user.getName())) {
        criteria.and("name").is(user.getName());
    }
    if (!StringUtils.isEmpty(user.getDescription())) {
        criteria.and("description").regex(user.getDescription());
    }

    query.addCriteria(criteria);

    Sort sort = new Sort(Sort.Direction.DESC, "age");
    Pageable pageable = PageRequest.of(page, size, sort);

    List<User> users = template.find(query.with(pageable), User.class);
    return PageableExecutionUtils.getPage(users, pageable, () -> template.count(query, User.class));
}
```

`size`表示每页显示的条数，`page`表示当前页码数，0表示第一页。上面的方法通过`name`和`description`（模糊查询）来查询用户分页信息，并且查询结果使用`age`字段降序排序。方法返回`Page`对象。

## 优化

为充分挖掘MongoDB性能，简单列举以下Mongodb优化方式：

1. 文档中的_id键推荐使用默认值，禁止向_id中保存自定义的值。
   ​解读： MongoDB文档中都会有一个“_id”键，默认是个ObjectID对象（标识符中包含时间戳、机器ID、进程ID和计数器）。MongoDB在指定_id与不指定_id插入时 速度相差很大，指定_id会减慢插入的速率。

2. 推荐使用短字段名。
   解读：与关系型数据库不同，MongoDB集合中的每一个文档都需要存储字段名，长字段名会需要更多的存储空间。

3. MongoDB索引可以提高文档的查询、更新、删除、排序操作，所以结合业务需求，适当创建索引。

4. 每个索引都会占用一些空间，并且导致插入操作的资源消耗，因此，建议每个集合的索引数尽量控制在5个以内。

5. 对于包含多个键的查询，创建包含这些键的复合索引是个不错的解决方案。复合索引的键值顺序很重要，理解索引最左前缀原则。
   解读：例如在test集合上创建组合索引{a:1,b:1,c:1}。执行以下7个查询语句：
   db.test.find({a:”hello”}) // 1
   db.test.find({b:”sogo”, a:”hello”}) // 2
   db.test.find({a:”hello”,b:”sogo”, c:”666”}) // 3
   db.test.find({c:”666”, a:”hello”}) // 4
   db.test.find({b:”sogo”, c:”666”}) // 5
   db.test.find({b:”sogo” }) // 6
   db.test.find({c:”666”}) // 7
   以上查询语句可能走索引的是1、2、3、4
   查询应包含最左索引字段，以索引创建顺序为准，与查询字段顺序无关。
   最少索引覆盖最多查询。

6. TTL 索引（time-to-live index，具有生命周期的索引），使用TTL索引可以将超时时间的文档老化，一个文档到达老化的程度之后就会被删除。
   解读：创建TTL的索引必须是日期类型。TTL索引是一种单字段索引，不能是复合索引。TTL删除文档后台线程每60s移除失效文档。不支持定长集合。

7. 需要在集合中某字段创建索引，但集合中大量的文档不包含此键值时，建议创建稀疏索引。
   解读：索引默认是密集型的，这意味着，即使文档的索引字段缺失，在索引中也存在着一个对应关系。在稀疏索引中，只有包含了索引键值的文档才会出现。

8. 创建文本索引时字段指定text，而不是1或者-1。每个集合只有一个文本索引，但是它可以为任意多个字段建立索引。
   解读：文本搜索速度快很多，推荐使用文本索引替代对集合文档的多字段的低效查询。

9. 使用findOne在数据库中查询匹配多个项目，它就会在自然排序文件集合中返回第一个项目。如果需要返回多个文档，则使用find方法。

10. 如果查询无需返回整个文档或只是用来判断键值是否存在，可以通过投影（映射）来限制返回字段，减少网络流量和客户端的内存使用。
    解读：既可以通过设置{key:1}来显式指定返回的字段，也可以设置{key:0}指定需要排除的字段。

11. 除了前缀样式查询，正则表达式查询不能使用索引，执行的时间比大多数选择器更长，应节制性地使用它们。

12. 批量插入（batchInsert）可以减少数据向服务器的提交次数，提高性能。但是批量提交的BSON Size不超过48MB。

13. 禁止一次取出太多的数据进行排序，MongoDB目前支持对32M以内的结果集进行排序。如果需要排序，请尽量限制结果集中的数据量。



# Springboot嵌入式数据库

嵌入式数据库通常用于开发和测试环境，不推荐用于生产环境。Spring Boot提供自动配置的嵌入式数据库有H2、HSQL、Derby，你不需要提供任何连接配置就能使用。

比如,我们可以在`pom.xml`中引入如下配置使用HSQL

```
<dependency>
    <groupId>org.hsqldb</groupId>
    <artifactId>hsqldb</artifactId>
    <scope>runtime</scope>
</dependency>
```

# 时序数据库InfluxDB

什么是时序数据库？全称为时间序列数据库。时间序列数据库主要用于指处理带时间标签（按照时间的顺序变化，即时间序列化）的数据，带时间标签的数据也称为时间序列数据。
时间序列数据主要由电力行业、化工行业等各类型实时监测、检查与分析设备所采集、产生的数据，这些工业数据的典型特点是：产生频率快（每一个监测点一秒钟内可产生多条数据）、严重依赖于采集时间（每一条数据均要求对应唯一的时间）、测点多信息量大（常规的实时监测系统均有成千上万的监测点，监测点每秒钟都产生数据，每天产生几十GB的数据量）。虽然关系型数据库也可以存储基于时间序列的数据，但由于存储结构上的劣势，使得这些数据无法高效的实现高频存储和查询统计，因此就诞生了一种专门针对时间序列来做存储和优化的数据库，以满足更高的效率要求。 – 参考：[百度百科：时序数据库](https://baike.baidu.com/item/时序数据库/922671)

InfluxDB就是目前比较流行的开源时序数据库（官网地址：https://www.influxdata.com/），我们比较常见的使用场景就是一些与时间相关的高频的数据记录和统计需要，比如：监控数据的存储和查询。

在进行下面的动手环节之前，先了解一下InfluxDB中的几个重要名词：

- database：数据库
- measurement：类似于关系数据库中的table（表）
- points：类似于关系数据库中的row（一行数据）

其中，一个Point由三个部分组成：

- time：时间戳
- fields：记录的值
- tags：索引的属性

```
<dependency>
    <groupId>org.influxdb</groupId>
    <artifactId>influxdb-java</artifactId>
</dependency>
```

注意：这里因为Spring Boot 2.x版本的parent中有维护InfluxDB的SDK版本，所以不需要手工指明版本信息。如果使用的Spring Boot版本比较老，那么可能会缺少version信息，就需要手工写了。

```
spring.influx.url=http://localhost:8086
spring.influx.user=admin
spring.influx.password=
```

三个属性分别代表：连接地址、用户名、密码。到这一步，基础配置就完成了。

注意：虽然没有spring data的支持，但spring boot 2.x版本中也实现了InfluxDB的自动化配置，所以只需要写好配置信息，就可以使用了。具体配置属性可以查看源码：`org.springframework.boot.autoconfigure.influx.InfluxDbProperties`。

创建定时任务，模拟上报数据，并写入InfluxDB:

```
@Service
@AllArgsConstructor
@Slf4j
public class Monitor {

    private InfluxDB influxDB;

    @Scheduled(fixedRate = 5000)
    public void writeQPS() {
        // 模拟要上报的统计数据
        int count = (int) (Math.random() * 100);

        Point point = Point.measurement("ApiQPS")     // ApiQPS表
                .tag("url", "/hello")  // url字段
                .addField("count", count)        // 统计数据
                .time(System.currentTimeMillis(), TimeUnit.MILLISECONDS)  // 时间
                .build();
        // 往test库写数据
        influxDB.write("test", "autogen", point);

        log.info("上报统计数据：" + count);
    }
}
```

测试验证:

```
进入InfluxDB
$ influx

> show databases
> create database "test"
```

启动Spring Boot应用，在定时任务的作用下

```
2021-08-03 01:52:47.732  INFO 94110 --- [           main] c.d.chapter63.Chapter63Application       : Started Chapter63Application in 2.326 seconds (JVM running for 3.027)
2021-08-03 01:52:47.764  INFO 94110 --- [   scheduling-1] com.didispace.chapter63.Monitor          : 上报统计数据：25
2021-08-03 01:52:52.736  INFO 94110 --- [   scheduling-1] com.didispace.chapter63.Monitor          : 上报统计数据：30
2021-08-03 01:52:57.737  INFO 94110 --- [   scheduling-1] com.didispace.chapter63.Monitor          : 上报统计数据：38
2021-08-03 01:53:02.739  INFO 94110 --- [   scheduling-1] com.didispace.chapter63.Monitor          : 上报统计数据：51
2021-08-03 01:53:07.739  INFO 94110 --- [   scheduling-1] com.didispace.chapter63.Monitor          : 上报统计数据：31
```

验证:

```
> select * from ApiQPS order by time desc;

name: ApiQPS
time                count url
----                ----- ---
1627926787730000000 31    /hello
1627926782730000000 51    /hello
1627926777729000000 38    /hello
1627926772727000000 30    /hello
1627926767728000000 25    /hello
```

可以看到，已经存在与日志中一样的数据了。

# mysql

目录:
windows:

工程目录默认在C:/Program Files/MySQL/MySQL Server8.0/

数据目录默认在C:/ProgramData/MySQL/MySQL Server8.0/

数据目录下的my.ini为mysql配置文件,linux为,my.cnf

![](D:\Study notes\LZFNotes\utils\note\picture\sql-join.jpg)

## 表操作

```
-- 创建表
    CREATE [TEMPORARY] TABLE[ IF NOT EXISTS] [库名.]表名 ( 表的结构定义 )[ 表选项]
        每个字段必须有数据类型
        最后一个字段后不能有逗号
        TEMPORARY 临时表，会话结束时表自动消失
        对于字段的定义：
            字段名 数据类型 [NOT NULL | NULL] [DEFAULT default_value] [AUTO_INCREMENT] [UNIQUE [KEY] | [PRIMARY] KEY] [COMMENT 'string']
-- 表选项
    -- 字符集
        CHARSET = charset_name
        如果表没有设定，则使用数据库字符集
    -- 存储引擎
        ENGINE = engine_name
        表在管理数据时采用的不同的数据结构，结构不同会导致处理方式、提供的特性操作等不同
        常见的引擎：InnoDB MyISAM Memory/Heap BDB Merge Example CSV MaxDB Archive
        不同的引擎在保存表的结构和数据时采用不同的方式
        MyISAM表文件含义：.frm表定义，.MYD表数据，.MYI表索引
        InnoDB表文件含义：.frm表定义，表空间数据和日志文件
        SHOW ENGINES -- 显示存储引擎的状态信息
        SHOW ENGINE 引擎名 {LOGS|STATUS} -- 显示存储引擎的日志或状态信息
    -- 自增起始数
    	AUTO_INCREMENT = 行数
    -- 数据文件目录
        DATA DIRECTORY = '目录'
    -- 索引文件目录
        INDEX DIRECTORY = '目录'
    -- 表注释
        COMMENT = 'string'
    -- 分区选项
        PARTITION BY ... (详细见手册)
-- 查看所有表
    SHOW TABLES[ LIKE 'pattern']
    SHOW TABLES FROM  库名
-- 查看表结构
    SHOW CREATE TABLE 表名 （信息更详细）
    DESC 表名 / DESCRIBE 表名 / EXPLAIN 表名 / SHOW COLUMNS FROM 表名 [LIKE 'PATTERN']
    SHOW TABLE STATUS [FROM db_name] [LIKE 'pattern']
-- 修改表
    -- 修改表本身的选项
        ALTER TABLE 表名 表的选项
        eg: ALTER TABLE 表名 ENGINE=MYISAM;
    -- 对表进行重命名
        RENAME TABLE 原表名 TO 新表名
        RENAME TABLE 原表名 TO 库名.表名 （可将表移动到另一个数据库）
        -- RENAME可以交换两个表名
    -- 修改表的字段机构（13.1.2. ALTER TABLE语法）
        ALTER TABLE 表名 操作名
        -- 操作名
            ADD[ COLUMN] 字段定义       -- 增加字段
                AFTER 字段名          -- 表示增加在该字段名后面
                FIRST               -- 表示增加在第一个
            ADD PRIMARY KEY(字段名)   -- 创建主键
            ADD UNIQUE [索引名] (字段名)-- 创建唯一索引
            ADD INDEX [索引名] (字段名) -- 创建普通索引
            DROP[ COLUMN] 字段名      -- 删除字段
            MODIFY[ COLUMN] 字段名 字段属性     -- 支持对字段属性进行修改，不能修改字段名(所有原有属性也需写上)
            CHANGE[ COLUMN] 原字段名 新字段名 字段属性      -- 支持对字段名修改
            DROP PRIMARY KEY    -- 删除主键(删除主键前需删除其AUTO_INCREMENT属性)
            DROP INDEX 索引名 -- 删除索引
            DROP FOREIGN KEY 外键    -- 删除外键
-- 删除表
    DROP TABLE[ IF EXISTS] 表名 ...
-- 清空表数据
    TRUNCATE [TABLE] 表名
-- 复制表结构
    CREATE TABLE 表名 LIKE 要复制的表名
-- 复制表结构和数据
    CREATE TABLE 表名 [AS] SELECT * FROM 要复制的表名
-- 检查表是否有错误
    CHECK TABLE tbl_name [, tbl_name] ... [option] ...
-- 优化表
    OPTIMIZE [LOCAL | NO_WRITE_TO_BINLOG] TABLE tbl_name [, tbl_name] ...
-- 修复表
    REPAIR [LOCAL | NO_WRITE_TO_BINLOG] TABLE tbl_name [, tbl_name] ... [QUICK] [EXTENDED] [USE_FRM]
-- 分析表
    ANALYZE [LOCAL | NO_WRITE_TO_BINLOG] TABLE tbl_name [, tbl_name] ...
```



## 配置文件

```
port：端口
server-id：数据库唯一标识码
datadir：数据目录路径
basedir：基本路径
socket：指定socket文件路径
skip-name-resolve：禁止mysql对外部连接进行DNS解析，启用时所有远程连接都要使用ip地址方
式
skip-external-locking：避免外部锁，默认开启
skip-grant-tables：mysql启动时不使用grant-tables授权表，常用于忘记密码情况下
user：使用的用户
sql_mode：约束SQL语法模式
show_query_log_file：慢日志存储路径，5.6及以上版本，log-slow-queries是5.6以下版本
long_query_time：设置慢查询时间，默认10s
log-queries-not-using-indexes：记录未使用索引查询
log_throttle_queries_not_using_indexs：5.6.5版本新增参数，设定每分钟记录到日志未使用索引
的语句数目，超过则之后只记录语句数量和花费总时间
log_output：日志存储方式，默认FILE，TABLE表示写入到mysql.slow_log表
expire_logs_days：二进制日志保留的时间
log-bin：设置日志文件路径及名称
log-bin-index：日志文件后缀形式
relay-bin：中继日志文件
relay-bin-index：中继日志文件后缀形式
max_connections：mysql最大连接数（超出会报Too many connections）
back_log：mysql暂存的连接数量(每个连接占用256kb)
wait_timeout：关闭一个非交互的连接之前所需要等待的秒数
max_connect_errors：最大错误连接数，flush host进行解禁
max_allowed_packet：接受数据包大小，需要是才会分配内存，设置过大会造成内存溢出
max_heap_table_size：用户可以创建的内存表大小
tmp_table_size：mysql的heap表缓冲大小
read_rnd_buffer_size：mysql随机读缓冲区大小
sort_buffer_size：mysql执行排序使用的缓冲大小
join_buffer_size：联合查询操作所能使用的缓冲区大小
```

修改方式:

1.修改参数文件, 然后重启mysqld;

2.在命令行里通过SET来设置(如果要修改全局变量, 必须要显示指定"GLOBAL"或者"@@global.", 同时必须要有SUPER权限.):

```
mysql> set global wait_timeout=10;
mysql> set @@global.wait_timeout=10;
```

查看是否设置成功:

```
mysql> select @@global.wait_timeout=10;
or
mysql> show global variables like 'wait_timeout';
```

如果查询时使用的是show variables的话, 会发现设置并没有生效, 除非重新登录再查看. 这是因为使用 show variables的话就等同于使用show session variables, 查询的是会话变量, 只有使用show global variables查询的才是全局变量. 如果仅仅想修改会话变量的话, 可以使用类似set wait_timeout=10;或者 set session wait_timeout=10;这样的语法.

3.设置回话变量:  如果要修改会话变量值, 可以指定"SESSION"或者"@@session."或者"@@"或者"LOCAL"或者"@@local.", 或者什么都不使用.

```
mysql> set wait_timeout=10;
mysql> set session wait_timeout=10;
mysql> set local wait_timeout=10;
mysql> set @@wait_timeout=10;
mysql> set @@session.wait_timeout=10;
mysql> set @@local.wait_timeout=10;
```

4.会话变量和全局变量转换方法:

```
# 将会话变量值设置为对应的全局变量值呢:
mysql> set @@session.wait_timeout=@@global.wait_timeout;
# 将会话变量值设置为MySQL编译时的默认值(wait_timeout=28800):
mysql> set wait_timeout=DEFAULT;
```

要注意的是, 并不是所有的系统变量都能被设置为DEFAULT, 如果设置这些变量为DEFAULT则会返回 错误.

**sql_mode**

 sql_mode定义了对Mysql中sql语句语法的校验规则！

sql_mode是个很容易被忽视的变量，如果设置为空值（非严格模式），在这种情况下是可以允许一些非法操作的，比如允许一些非法数据的插入。在生产环境必须将这个值设置为严格模式，所以开发、测试环境的数据库也必须要设置，这样在开发测试阶段就可以发现问题。

sql_mode常用的值

| ONLY_FULL_GROUP_BY         | 对于GROUP BY聚合操作，如果在SELECT中的列，没有在GROUP  BY中出现，那么这个SQL是不合法的，因为列不在GROUP BY从句中 |
| -------------------------- | ------------------------------------------------------------ |
| NO_AUTO_VALUE_ON_ZERO      | 该值影响自增长列的插入。默认设置下，插入0或NULL代表生成下一个自增长值。如果用户 希望插入的值为0，而该列又是自增长的，那么这个选项就有用了 |
| STRICT_TRANS_TABLES        | 在该模式下，如果一个值不能插入到一个事务表中，则中断当前的操作，对非事务表不做限制 |
| NO_ZERO_IN_DATE            | 在严格模式下，不允许日期和月份为零                           |
| NO_ZERO_DATE               | 设置该值，mysql数据库不允许插入零日期，插入零日期会抛出错误而不是警告 |
| ERROR_FOR_DIVISION_BY_ZERO | 在INSERT或UPDATE过程中，如果数据被零除，则产生错误而非警告。如 果未给出该模式，那么数据被零除时MySQL返回NULL |
| NO_AUTO_CREATE_USER        | 禁止GRANT创建密码为空的用户                                  |
| NO_ENGINE_SUBSTITUTION     | 如果需要的存储引擎被禁用或未编译，那么抛出错误。不设置此值时，用默认的存储引擎替代，并抛出一个异常 |
| PIPES_AS_CONCAT            | 将"\|\|"视为字符串的连接操作符而非或运算符，这和Oracle数据库是一样的，也和字符串的拼接函数Concat相类似 |
| ANSI_QUOTES                | 启用ANSI_QUOTES后，不能用双引号来引用字符串，因为它被解释为识别符 |
| ORACLE                     | 设置等同于PIPES_AS_CONCAT,  ANSI_QUOTES, IGNORE_SPACE, NO_KEY_OPTIONS, NO_TABLE_OPTIONS,  NO_FIELD_OPTIONS, NO_AUTO_CREATE_USER |

```
# 查看当前的sql_mode
mysql> select @@sql_mode;
# 临时修改 sql_mode
mysql> set @@sql_mode='';
# 永久修改，需要在配置文件my.cnf中修改
[root@hadoop102 ~]$ vim /etc/my.cnf
#添加下列配置，然后重启mysql即可
[mysqld]
sql_mode=''
# sql_mode的影响案例：group by 查询语法错误！
CREATE TABLE mytbl2 (id INT,NAME VARCHAR(200),age INT,dept INT);
INSERT INTO mytbl2 VALUES(1,'zhang3',33,101);
INSERT INTO mytbl2 VALUES(2,'li4',34,101);
INSERT INTO mytbl2 VALUES(3,'wang5',34,102);
INSERT INTO mytbl2 VALUES(4,'zhao6',34,102);
INSERT INTO mytbl2 VALUES(5,'tian7',36,102);

查询每个dept中年龄最大的人：
错误结果：
SELECT NAME,dept,MAX(age) FROM mytbl2 GROUP BY dept;
正确结果：
SELECT id,name,ab.dept,ab.maxage FROM mytbl2 m INNER JOIN(SELECT dept,MAX(age)maxage FROM mytbl2 GROUP BY dept)ab ON ab.dept=m.dept AND m.age=ab.maxage;
```

**大小写是否敏感**

win系统默认大小写不敏感, linux系统大小写敏感

查看大小写是否敏感:

```
show variables like '%lower_case_table_names%'
```

设置大小写不敏感

```
vim /etc/my.cnf
[mysqld]
lower_case_table_names = 1
# 0		大小写敏感
# 1		大小写不敏感
# 2		创建的表和DB依据语句上格式存放，凡是查找都是转换为小写进行
```

注意：如果要设置属性为大小写不敏感，要在重启数据库实例之前就需要将原来的数据库和表转换为小写，否则将找不到数据库名。在进行数据库参数设置之前，需要掌握这个参数带来的影响，切不可盲目设置。



## explain

explain关键字可以模拟MySQL优化器执行SQL语句，可以很好的分析SQL语句或表结构的性能 瓶颈。

```
mysql> explain select * from subject where id = 1
******************************************************
id: 1
select_type: SIMPLE
table: subject
partitions: NULL
type: const
possible_keys: PRIMARY
key: PRIMARY
key_len: 4
ref: const
rows: 1
filtered: 100.00
Extra: NULL
```

```
1. id //select查询的序列号，包含一组数字，表示查询中执行select子句或操作表的顺序
2. select_type //查询类型 主要用于区别普通查询、联合查询、子查询等的复杂查询
3. table //这个数据是基于哪张表的
4. partitions //匹配的分区
5. type //访问的类型  这个数据是基于哪张表的
6. possible_keys //显示可能应用在这张表中的索引，一个或多个，但不一定实际使用到
7. key //实际使用到的索引，如果为NULL，则没有使用索引
8. key_len //表示索引中使用的字节数，可通过该列计算查询中使用的索引的长度.在不损失精确度的情况下，长度越短越好
9. ref //显示索引的哪一列被使用了，如果可能的话，是一个常数，哪些列或常量被用于查找索引列上的值
10. rows //根据表统计信息及索引选用情况，大致估算出找到所需的记录所需读取的行数
11. filtered //查询的表行占表的百分比
12. Extra //包含不适合在其它列中显示但十分重要的额外信息
```

**id字段**

1.id相同

执行顺序从上至下

2.id不同

如果是子查询，id的序号会递增，id的值越大优先级越高，越先被执行

3.id相同又不同

id如果相同，可以认为是一组，从上往下顺序执行 在所有组中，id值越大，优先级越高，越先执行

**select_type字段**

1.SIMPLE

简单查询，不包含子查询或Union查询

2.PRIMARY

查询中若包含任何复杂的子部分，最外层查询则被标记为主查询

3.SUBQUERY

在select或where中包含子查询

4.DERIVED

在FROM列表中包含的子查询被标记为DERIVED（衍生），MySQL 会递归执行这些子查询，把结果放在临时表中

```
MySQL5.7+ 进行优化了，增加了derived_merge（派生合并），默认开启，可加快查询效率
```

5.UNION

若第二个select出现在uion之后，则被标记为UNION

6.UNION RESULT

从UNION表获取结果的select

**type字段**

以下顺序从好到差

1.NULL

MySQL能够在优化阶段分解查询语句，在执行阶段用不着再访问表或索引，比如通过id没有找到

explain select min(id) from subject;

2.system

表只有一行记录（等于系统表），这是const类型的特列，平时不大会出现，

3.const

表示通过索引一次就找到了，const用于比较primary key或uique索引，因为只匹配一行数据，所以很 快，如主键置于where列表中，MySQL就能将该查询转换为一个常量.

4.eq_ref

唯一性索引扫描，对于每个索引键，表中只有一条记录与之匹配，常见于主键或唯一索引扫描

5.ref

非唯一性索引扫描，返回匹配某个单独值的所有行 

本质上也是一种索引访问，返回所有匹配某个单独值的行

 然而可能会找到多个符合条件的行，应该属于查找和扫描的混合体

6.ref_or_null

类似ref, 但是可以搜索值为null的行

7.index_merge

表示使用了索引合并的优化方法

8.range

只检索给定范围的行，使用一个索引来选择行，key列显示使用了哪个索引 一般就是在你的where语句中出现between、<>、in等的查询。

9.index

Full index Scan

如explain select id from subject;

10.ALL

Full Table Scan，将遍历全表以找到匹配行.

Index与All区别：index只遍历索引树，通常比All快 因为索引文件通常比数据文件小，也就是虽然all和index都是读全表，但index是从索引中读取的，而all 是从硬盘读的。

**Extra字段**

1.Using filesort

说明MySQL会对数据使用一个外部的索引排序，而不是按照表内的索引顺序进行读取 

MySQL中无法利用索引完成的排序操作称为“文件排序”

```
优化方式,给排序字段建索引，并使用where或limit激活
```

2.Using temporary

使用了临时表保存中间结果，MySQL在对结果排序时使用临时表，常见于排序order by 和分组查询group by

```
优化方式,给分组字段建索引
```

3.Using index

表示相应的select操作中使用了**覆盖索引**(Covering Index  ,  就是select的数据列只用从索引中就能够取得，不必从数据表中读取，换句话说查询列要被所使用的索引覆盖)，  避免访问了表的数据行，效率不错！

4.Using where

使用了where条件,表示优化器需要通过索引回表查询数据

5.Using join buffer

使用了连接缓存 

```
explain select student.*,teacher.*,subject.* from student,teacher,subject;

explain select * from emp ,dept where emp.empno = dept.ceo ;
```

6.impossible where

where子句的值总是false，不能用来获取任何元组

7.distinct

一旦mysql找到了与行相联合匹配的行，就不再搜索了

如左连接里的右表

8.Select tables optimized away

```
explain select * from emp ,dept where emp.empno = dept.ceo ;
explain select min(id) from subject;
```



## sql优化

**关联查询**

**left join**

在优化关联查询时，只有在匹配表上建立索引才有效。

   left join时，左侧的为驱动表，右侧为匹配表(被驱动表)！

```
1)	保证被驱动表的join字段已经被索引
2)	Left Join 或者 right join时，选择小表作为驱动表，大表作为被驱动表
3)	inner join 时，mysql会自己把小表选为驱动表
4)	子查询尽量不要放在被驱动表，有可能使用不到索引
5)	能直接多表关联就尽量直接关联，不用子查询
```

**子查询**

在范围判断时，尽量不要使用not in和not exists，使用 left join on  xxx  where xxx is null代替。



### 执行周期

```
#查看profile是否开启
show variables  like '%profiling%';
#开启profiling
set profiling =1 ;
#在MySQL中执行两条相同的SQL
select * from mytbl2;
select * from mytbl2;
#查看最近执行的sql
show profiles;
#根据上一步查询出来的Query_ID,查看sql的具体执行周期
show profile cpu,block io for query 1 ;
show profile cpu,block io for query 2 ;
```



## 触发器

1.单条业务逻辑触发器

```
CREATE TRIGGER 触发器名称 BEFORE|AFTER INSERT|UPDATE|DELETE ON 表名
FOR EACH ROW 业务逻辑
```

```
#当b_user表中插入数据后，b_log表中也插入一条数据
CREATE TRIGGER trigger_insert AFTER INSERT ON b_user
FOR EACH ROW INSERT INTO b_log(字段) VALUES('插入数据')
```

2.多条业务逻辑触发器

```
DELIMITER $
CREATE TRIGGER 触发器名称 BEFORE|AFTER INSERT|UPDATE|DELETE ON 表名
FOR EACH ROW
BIGIN
INSERT...;
UPDATE...;
END;$
```

```
#在b_user表中插入数据前，b_log表中插入2条数据
DELIMITER $
CREATE TRIGGER trigger_ insert_before BEFORE INSERT ON b_user
FOR EACH ROW
BEGIN
INSERT INTO b_log(comments,name) values('insert1' ，NEW.name);
INSERT INTO b_log(comments,name) values('insert2' , NEW.name) ;
END;$
```

```
NEW.columnname：新增行的某列数据
OLD.columnname：删除行的某列数据
```

3.删除触发器

```
DROP TRIGGER 触发器名称
```

## 数据脚本

**生成随机字符串**

```sql
DELIMITER $$
CREATE FUNCTION rand_string(n INT) RETURNS VARCHAR(255)
BEGIN    
DECLARE chars_str VARCHAR(100) DEFAULT 'abcdefghijklmnopqrstuvwxyzABCDEFJHIJKLMNOPQRSTUVWXYZ';
 DECLARE return_str VARCHAR(255) DEFAULT '';
 DECLARE i INT DEFAULT 0;
 WHILE i < n DO  
 SET return_str =CONCAT(return_str,SUBSTRING(chars_str,FLOOR(1+RAND()*52),1));  
 SET i = i + 1;
 END WHILE;
 RETURN return_str;
END $$
```

**生成随机数字**

```sql
#用于随机产生多少到多少的编号
DELIMITER $$
CREATE FUNCTION  rand_num (from_num INT ,to_num INT) RETURNS INT(11)
BEGIN   
 DECLARE i INT DEFAULT 0;  
 SET i = FLOOR(from_num +RAND()*(to_num -from_num+1))   ;
RETURN i;  
 END$$
```

**批量插入数据**

先创建表

```sql
CREATE TABLE `dept` (
 `id` INT(11) NOT NULL AUTO_INCREMENT,
 `deptName` VARCHAR(30) DEFAULT NULL,
 `address` VARCHAR(40) DEFAULT NULL,
  ceo INT NULL ,
  PRIMARY KEY (`id`)
) ENGINE=INNODB AUTO_INCREMENT=1 DEFAULT CHARSET=utf8;

CREATE TABLE `emp` (
 `id` INT(11) NOT NULL AUTO_INCREMENT,
 `empno` INT NOT NULL ,
 `name` VARCHAR(20) DEFAULT NULL,
 `age` INT(3) DEFAULT NULL,
 `deptId` INT(11) DEFAULT NULL,
  PRIMARY KEY (`id`)
 #CONSTRAINT `fk_dept_id` FOREIGN KEY (`deptId`) REFERENCES `t_dept` (`id`)
) ENGINE=INNODB AUTO_INCREMENT=1 DEFAULT CHARSET=utf8;
```

开启log_bin_trust_function_creators  , 它控制是否可以信任存储函数创建者

```
show variables like 'log_bin_trust_function_creators'; //查询
mysql> set global log_bin_trust_function_creators=1;  //开启

设置永久生效方式：修改/etc/my.cnf 文件
[mysqld]
log_bin_trust_function_creators=1
```

创建批量插入存储过程,中间需要调用上面随机生成函数

```sql
DELIMITER $$
CREATE PROCEDURE  insert_emp(  START INT ,  max_num INT )
BEGIN  
DECLARE i INT DEFAULT 0;   
#set autocommit =0 把autocommit设置成0  
 SET autocommit = 0;    
 REPEAT  
 SET i = i + 1;  
 INSERT INTO emp (empno, NAME ,age ,deptid ) VALUES ((START+i) ,rand_string(6)   , rand_num(30,50),rand_num(1,10000));  
 UNTIL i = max_num  
 END REPEAT;  
 COMMIT;  
 END$$ 
 
#删除
# DELIMITER ;
# drop PROCEDURE insert_emp;

#执行存储过程，往dept表添加随机数据
DELIMITER $$
CREATE PROCEDURE `insert_dept`(  max_num INT )
BEGIN  
DECLARE i INT DEFAULT 0;   
 SET autocommit = 0;    
 REPEAT  
 SET i = i + 1;  
 INSERT INTO dept ( deptname,address,ceo ) VALUES (rand_string(8),rand_string(10),rand_num(1,500000));  
 UNTIL i = max_num  
 END REPEAT;  
 COMMIT;  
 END$$
 
#删除
# DELIMITER ;
# drop PROCEDURE insert_dept;
```

**调用存储过程**

```
#执行存储过程，往dept表添加1万条数据
DELIMITER ;
CALL insert_dept(10000);
#执行存储过程，往emp表添加50万条数据
DELIMITER ;
CALL insert_emp(100000,500000);
```

**删除表的所有索引**

```sql
DELIMITER $$
CREATE  PROCEDURE `proc_drop_index`(dbname VARCHAR(200),tablename VARCHAR(200))
BEGIN
       DECLARE done INT DEFAULT 0;
       DECLARE ct INT DEFAULT 0;
       DECLARE _index VARCHAR(200) DEFAULT '';
       DECLARE _cur CURSOR FOR  SELECT   index_name   FROM information_schema.STATISTICS   WHERE table_schema=dbname AND table_name=tablename AND seq_in_index=1 AND    index_name <>'PRIMARY'  ;
       DECLARE  CONTINUE HANDLER FOR NOT FOUND set done=2 ;      
        OPEN _cur;
        FETCH   _cur INTO _index;
        WHILE  _index<>'' DO 
               SET @str = CONCAT("drop index ",_index," on ",tablename ); 
               PREPARE sql_str FROM @str ;
               EXECUTE  sql_str;
               DEALLOCATE PREPARE sql_str;
               SET _index=''; 
               FETCH   _cur INTO _index; 
        END WHILE;
   CLOSE _cur;
   END$$
```



## 主从同步

MySQL主从复制的基础是主服务器对数据库修改记录二进制日志，从服务器通过主服务器的二进制日志 自动执行更新。

原理:

```
Master 数据库只要发生变化，立马记录到Binary log 日志文件中
Slave数据库启动一个I/O thread连接Master数据库，请求Master变化的二进制日志
Slave I/O获取到的二进制日志，保存到自己的Relay log 日志文件中。
Slave 有一个 SQL thread定时检查Realy log是否变化，变化那么就更新数据
```

不管使用何种存储引擎，在server层都可以开启binlog日志功能。binlog会记录所有的逻辑操作，并且是采取追加写的形式，将写操作命令，记录在一个二进制文件中。因此binlog日志通常用于恢复数据，或者是主从复制。

配置

master:  192.168.120.200

```
#mysql master1 config
[mysqld]
server-id = 1 # 节点ID，确保唯一
# log config
log-bin = master-bin #开启mysql的binlog日志功能
sync_binlog = 1 #控制数据库的binlog刷到磁盘上去 , 0 不控制，性能最好，1每次事物提交都会刷到日志文件中，性能最差，最安全
binlog_format = mixed #binlog日志格式，mysql默认采用statement，建议使用mixed
expire_logs_days = 7 #binlog过期清理时间
max_binlog_size = 100m #binlog每个日志文件大小
binlog_cache_size = 4m #binlog缓存大小
max_binlog_cache_size= 512m #最大binlog缓存大
binlog-ignore-db=mysql #不生成日志文件的数据库，多个忽略数据库可以用逗号拼接，或者 复制这句话，写多行
auto-increment-offset = 1 # 自增值的偏移量
auto-increment-increment = 1 # 自增值的自增量
slave-skip-errors = all #跳过从库错误
```

slave:192.168.120.201

```
[mysqld]
server-id=2
log-bin=mysql-bin
relay-log=mysql-relay-bin
replicate-wild-ignore-table=mysql.%
replicate-wild-ignore-table=test.%
replicate-wild-ignore-table=information_schema.%
```

重启mysql后 ,给master创建复制用户并授权

```
CREATE USER repl IDENTIFIED BY 'Root12345_';

grant replication slave on *.* to 'repl'@'192.168.120.200' identified by 'Root12345_';

FLUSH PRIVILEGES;
```

查看master状态

```
show master status;
```

配置slave

```
CHANGE MASTER TO
MASTER_HOST = '192.168.120.200',
MASTER_USER = 'repl',
MASTER_PASSWORD = 'Root12345_',
MASTER_PORT = 3306,
MASTER_LOG_FILE='master-bin.000001',
MASTER_LOG_POS=120,
MASTER_RETRY_COUNT = 60,
MASTER_HEARTBEAT_PERIOD = 10000;

# 根据上面查看master状态显示后
# MASTER_LOG_FILE与主库File 保持一致
# MASTER_LOG_POS=120 , #与主库Position 保持一致
```

启动从库进程

```
slave start;
```

**用户和权限**

用户操作,MySQL的用户管理在 mysql库中的user表中

  需要了解的列: Host,User, authentication_string( mysql 5.7 的密码保存到 authentication_string 字段中不再使用password 字段)等， 可通过 desc user 查看user表结构

| 命令                                                         | 描述                                                         | 备注                                                         |
| ------------------------------------------------------------ | ------------------------------------------------------------ | ------------------------------------------------------------ |
| create user zhang3 identified by '123123';                   | 创建名称为zhang3的用户，密码设为123123；                     |                                                              |
| select host,user,password,select_priv,insert_priv,drop_priv  from mysql.user; | 查看用户和权限的相关信息                                     |                                                              |
| set password  =password('123456')                            | 修改当前用户的密码                                           |                                                              |
| update  mysql.user set authentication_string=password('123456') where user='li4'; | 修改其他用户的密码  注意:mysql 5.7 通过authentication_string表示密码列 | 所有通过user表的修改，必须用flush privileges;命令才能生效    |
| update  mysql.user set user='li4' where user='wang5';        | 修改用户名                                                   | 所有通过user表的修改，必须用flush privileges;命令才能生效    |
| drop  user li4                                               | 删除用户                                                     | 不要通过delete from user  u where user='li4' 进行删除，系统会有残留信息保留。 |

权限操作

权限授予

| 命令                                                         | 描述                                                         |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| grant 权限1,权限2,…权限n  on 数据库名称.表名称 to  用户名@用户地址 identified by ‘连接口令’ | 该权限如果发现没有该用户，则会直接新建一个用户。  示例：  grant select,insert,delete,drop on atguigudb.* to  li4@localhost ;  给li4用户用本地命令行方式下，授予atguigudb这个库下的所有表的插删改查的权限。 |
| grant all privileges on *.* to joe@'%' identified by '123';  | 授予通过网络方式登录的的joe用户 ，对所有库所有表的全部权限，密码设为123. |

收回权限

| 命令                                                         | 描述                                | 备注 |
| ------------------------------------------------------------ | ----------------------------------- | ---- |
| show grants                                                  | 查看当前用户权限                    |      |
| revoke [权限1,权限2,…权限n]  on   库名.表名   from 用户名@用户地址 ; | 收回权限命令                        |      |
| REVOKE ALL PRIVILEGES ON mysql.* FROM joe@localhost;         | 收回全库全表的所有权限              |      |
| REVOKE select,insert,update,delete ON mysql.* FROM  joe@localhost; | 收回mysql库下的所有表的插删改查权限 |      |

提示：权限收回后，必须用户重新登录后，才能生效。

**Keepalived**

Keepalived是基于VRRP（Virtual Router Redundancy Protocol，虚拟路由器冗余协议）协议的一款高可用软件。Keepailived有一台主服务器（master）和多台备份服务器（backup），在主服务器和备份服务器上面部署相同的服务配置，使用一个虚拟IP地址对外提供服务，当主服务器出现故障时，虚拟IP地址会自动漂移到备份服务器。

**互为主从**



# 查询

**查找所有上级或所有下级**

```
select * from table [start with condition1]
    connect by [prior] id=parentid
    
select * from table [start with condition1]
    connect by id= [prior] parentid
```

**分组统计后排序**

```
select name,count(name)  from tb_test group by name order by 2 desc
select name,count(1)  from tb_test group by name order by 2 desc
```

