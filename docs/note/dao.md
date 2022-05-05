# 持久层

## DataSource

Spring Boot 2.1默认使用MySQL 8.0的驱动, 采用`com.mysql.cj.jdbc.Driver`, 而不是老的`com.mysql.jdbc.Driver`

**JDBC**

JDBC位于JDK的`java.sql`包中, 主要包括: DriverManager、Driver、Connection、Statement、CallableStatement、SQLException. 其中的内容已经可以实现对数据库的访问.

为了封装对数据库访问参数进行统一封装, 对连接池进行管理, `javax.sql`中包含了DataSource的抽象, 其实现有C3P0、Druid、HikariCP. 在Spring Boot 2.x中, 采用了目前性能最佳的[HikariCP](https://github.com/brettwooldridge/HikariCP)。

- 通用配置：`spring.datasource.*`

```properties
spring.datasource.url=jdbc:mysql://localhost:3306/test
spring.datasource.username=root
spring.datasource.password=123456
spring.datasource.driver-class-name=com.mysql.jdbc.Driver
```

- 数据源连接池配置：`spring.datasource.<数据源名称>.*`, 常用的有

```properties
# 最小空闲连接
spring.datasource.hikari.minimum-idle=10
# 最大连接数
spring.datasource.hikari.maximum-pool-size=10
# 空闲连接超时时间
spring.datasource.hikari.idle-timeout=600000
# 连接最大存活时间
spring.datasource.hikari.max-lifetime=540000
# 连接超时时间
spring.datasource.hikari.connection-timeout=60000
# 用于测试连接是否可用的查询语句
spring.datasource.hikari.connection-test-query=SELECT 1
```

### 多数据源

```java
@Configuration
public class DataSourceConfig {
    @Bean(name = "primaryDataSource")
    @Qualifier("primaryDataSource")
    @ConfigurationProperties(prefix="spring.datasource.primary")
    public DataSource primaryDataSource() {
        return DataSourceBuilder.create().build();
    }

    @Bean(name = "secondaryDataSource")
    @Qualifier("secondaryDataSource")
    @Primary
    @ConfigurationProperties(prefix="spring.datasource.secondary")
    public DataSource secondaryDataSource() {
        return DataSourceBuilder.create().build();
    }
}
```

```properties
spring.datasource.primary.jdbc-url=jdbc:mysql://localhost:3306/test1   
#spring.datasource.primary.url=jdbc:mysql://localhost:3306/test1    1.x版本使用的方式
spring.datasource.primary.username=root
spring.datasource.primary.password=root
spring.datasource.primary.driver-class-name=com.mysql.jdbc.Driver

spring.datasource.primary.jdbc-url=jdbc:mysql://localhost:3306/test2
#spring.datasource.secondary.url=jdbc:mysql://localhost:3306/test2
spring.datasource.secondary.username=root
spring.datasource.secondary.password=root
spring.datasource.secondary.driver-class-name=com.mysql.jdbc.Driver
```

```java
@Bean(name = "primaryJdbcTemplate")
public JdbcTemplate primaryJdbcTemplate(
        @Qualifier("primaryDataSource") DataSource dataSource) {
    return new JdbcTemplate(dataSource);
}

@Bean(name = "secondaryJdbcTemplate")
public JdbcTemplate secondaryJdbcTemplate(
        @Qualifier("secondaryDataSource") DataSource dataSource) {
    return new JdbcTemplate(dataSource);
}
```

### AbstractRoutingDataSource

继承该抽象类, 实现其detemineCurrentLookupKey方法可以实现动态数据源, 该方法返回的dataSourceName即请求数据库的数据源名称, 一般与ThreadLocal< String>和Map 一起使用, 将数据源以kv形式存于Map, 并将当前dataSourceName设置到ThreadLocal, 提供额外的接口修改其值(注解).

### Druid

```xml
<dependency>
    <groupId>com.alibaba</groupId>
    <artifactId>druid-spring-boot-starter</artifactId>
    <version>1.1.21</version>
</dependency>
```

```properties
spring.datasource.druid.url=jdbc:mysql://localhost:3306/test
spring.datasource.druid.username=root
spring.datasource.druid.password=
spring.datasource.druid.driver-class-name=com.mysql.cj.jdbc.Driver
```

配置连接池

```properties
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

监控功能配置

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```

```properties
spring.datasource.druid.stat-view-servlet.url-pattern          ：访问地址规则
spring.datasource.druid.stat-view-servlet.reset-enable         ：是否允许清空统计数据
spring.datasource.druid.stat-view-servlet.login-username       ：监控页面的登录账户
spring.datasource.druid.stat-view-servlet.login-password       ：监控页面的登录密码
```

访问Druid的监控页面`http://localhost:8080/druid/`

输入上面`spring.datasource.druid.stat-view-servlet.login-username`和`spring.datasource.druid.stat-view-servlet.login-password`配置的登录账户与密码，就能看到监控页面

## JPA

```xml
<dependency
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-jpa</artifactId>
</dependency>
```

```properties
spring.datasource.url=jdbc:mysql://localhost:3306/test
spring.datasource.username=root
spring.datasource.password=
spring.datasource.driver-class-name=com.mysql.cj.jdbc.Driver

spring.jpa.properties.hibernate.hbm2ddl.auto=create-drop
// 新版本已经变为 spring.jpa.properties.ddl-auto=create-drop
```

`spring.jpa.properties.hibernate.hbm2ddl.auto`是hibernate的配置属性，其主要作用是：自动创建、更新、验证数据库表结构。该参数的几种配置如下：

- `create`：每次加载hibernate时都会删除上一次的生成的表，然后根据你的model类再重新来生成新表，哪怕两次没有任何改变也要这样执行，这就是导致数据库表数据丢失的一个重要原因。
- `create-drop`：每次加载hibernate时根据model类生成表，但是sessionFactory一关闭,表就自动删除。
- `update`：最常用的属性，第一次加载hibernate时根据model类会自动建立起表的结构（前提是先建立好数据库），以后加载hibernate时根据model类自动更新表结构，即使表结构改变了但表中的行仍然存在不会删除以前的行。要注意的是当部署到服务器后，表结构是不会被马上建立起来的，是要等应用第一次运行起来后才会。
- `validate`：每次加载hibernate时，验证创建数据库表结构，只会和数据库中的表进行比较，不会创建新表，但是会插入新值。
- `none`:默认值

```java
public interface UserRepository extends JpaRepository<User, Long> {
    //Spring-data-jpa的一大特性：通过解析方法名创建查询
    User findByName(String name);
    User findByNameAndAge(String name, Integer age);
    @Query("from User u where u.name=:name")
    User findUser(@Param("name") String name);
}
```

> 大坑: 如果需要使用MySQL事务,需要设置`spring.jpa.database-platform`为MySQL5InnoDBDialect,保证Hibernate自动创建表时使用InnoDB,否则默认使用MyISAM创建.

### 列注解

@Table:可指定表名

@Column:

```java
//设置属性 userName 对应的数据库字段名为 user_name，长度为 32，非空
@Column(name = "user_name", nullable = false, length=32)
private String userName;
//设置字段类型并且加默认值
@Column(columnDefinition = "tinyint(1) default 1")
private Boolean enabled;
```

@Id:标识主键

@GeneratedValue:主键生成,可指定策略,如`strategy = GenerationType.IDENTITY`

```
TABLE:使用一个特定的数据库表格来保存主键
SEQUENCE:某些数据库中不支持主键自增长,如Oracle、PostgreSQL其提供序列机制生成主键
IDENTITY:主键自增长,
AUTO:默认,把主键生成策略交给持久化引擎,即根据数据库自动从上面三者中选择
```

还可以自己制定生成策略, 如:

> ```java
> @Id
> @GeneratedValue(strategy = GenerationType.IDENTITY)
> private Long id;
> ```
> 
> 等价于通过 `@GenericGenerator`声明一个主键策略，然后 `@GeneratedValue`使用这个策略
> 
> ```java
> @Id
> @GeneratedValue(generator = "IdentityIdGenerator")
> @GenericGenerator(name = "IdentityIdGenerator", strategy = "identity")
> private Long id;
> ```

JPA提供的主键生成策略:

```java
public class DefaultIdentifierGeneratorFactory implements MutableIdentifierGeneratorFactory, Serializable, ServiceRegistryAwareService {
    @SuppressWarnings("deprecation")
    public DefaultIdentifierGeneratorFactory() {
        register( "uuid2", UUIDGenerator.class );
        register( "guid", GUIDGenerator.class );            // can be done with UUIDGenerator + strategy
        register( "uuid", UUIDHexGenerator.class );            // "deprecated" for new use
        register( "uuid.hex", UUIDHexGenerator.class );     // uuid.hex is deprecated
        register( "assigned", Assigned.class );
        register( "identity", IdentityGenerator.class );
        register( "select", SelectGenerator.class );
        register( "sequence", SequenceStyleGenerator.class );
        register( "seqhilo", SequenceHiLoGenerator.class );
        register( "increment", IncrementGenerator.class );
        register( "foreign", ForeignGenerator.class );
        register( "sequence-identity", SequenceIdentityGenerator.class );
        register( "enhanced-sequence", SequenceStyleGenerator.class );
        register( "enhanced-table", TableGenerator.class );
    }

    public void register(String strategy, Class generatorClass) {
        LOG.debugf( "Registering IdentifierGenerator strategy [%s] -> [%s]", strategy, generatorClass.getName() );
        final Class previous = generatorStrategyToClassNameMap.put( strategy, generatorClass );
        if ( previous != null ) {
            LOG.debugf( "    - overriding [%s]", previous.getName() );
        }
    }
}
```

@CreationTimestamp:    生成创建时间

@UpdateTimestamp:  更新时间

@CreatedBy:创建者

@LastModifiedBy:最后修改者

@Transient ：声明不需要与数据库映射的字段,下面效果相同

```
static String secrect; // not persistent because of static
final String secrect = "Satish"; // not persistent because of final
transient String secrect; // not persistent because of transient
```

@Lob:声明某个字段为大字段, 可通过@Basic更详细声明

```java
@Lob
//指定 Lob 类型数据的获取策略， FetchType.EAGER 表示非延迟加载，而 FetchType.LAZY 表示延迟加载 ；
@Basic(fetch = FetchType.EAGER)
//columnDefinition 属性指定数据表对应的 Lob 字段类型
@Column(name = "content", columnDefinition = "LONGTEXT NOT NULL")
private String content;
```

@Enumerated: 使用枚举类型的字段需要加的注解

### 审计

只要继承了 `AbstractAuditBase`的类都会自动提供审计功能, 可以省略好多代码.

```java
@Data
@AllArgsConstructor
@NoArgsConstructor
@MappedSuperclass
@EntityListeners(value = AuditingEntityListener.class)
public abstract class AbstractAuditBase {
    @CreatedDate
    @Column(updatable = false)
    @JsonIgnore
    private Instant createdAt;
    @LastModifiedDate
    @JsonIgnore
    private Instant updatedAt;
    @CreatedBy
    @Column(updatable = false)
    @JsonIgnore
    private String createdBy;
    @LastModifiedBy
    @JsonIgnore
    private String updatedBy;
}
```

对应该审计功能的配置类可能是下面这样的（Spring Security 项目）:

```java
@Configuration
@EnableJpaAuditing
public class AuditSecurityConfiguration {
    @Bean
    AuditorAware<String> auditorAware() {
        return () -> Optional.ofNullable(SecurityContextHolder.getContext())
                .map(SecurityContext::getAuthentication)
                .filter(Authentication::isAuthenticated)
                .map(Authentication::getName);
    }
}
```

简单介绍一下上面涉及到的一些注解：

1. `@CreatedDate`: 表示该字段为创建时间字段，在这个实体被 insert 的时候，会设置值

2. `@CreatedBy` :表示该字段为创建人(username)，在这个实体被 insert 的时候，会设置值
   
   `@LastModifiedDate`、`@LastModifiedBy`同理。

`@EnableJpaAuditing`：开启 JPA 审计功能。

### @Query

用户定制化查询

```java
public @interface Query {
    /** 定义被执行的sql或者hql */
    String value() default "";
    /** 分页时用于查询中数量的sql或者hql */
    String countQuery() default "";
    /** 用原生的分页 */
    String countProjection() default "";
    /** 使用原生sql,为false时使用hql hql from对象 sql from表名*/
    boolean nativeQuery() default false;
    /** 定义该查询的名字 */
    String name() default "";
    /** 数量查询返回的别名 */
    String countName() default "";
}
```

`?序号`占位符方式,序号从1开始

```java
@Query("select u.id, u.name from User u where u.id=?1")
List<User> getUserById(Integer id);
```

`:参数名`方式(参数名需要为@Param指定)

```java
@Query("select u.id, u.name from User u where u.name like %:name%")
List<User> getUserByName(@Param("name") String name);
```

原生SQL 只能使用`?序号`方式

```java
@Query(value="select u.id, u.name from t_user u where u.id=?1", nativeQuery = true)
List<User> getUserById(Integer id);
```

SPEL表达式方式

```java
//#{#entityName}会取@Entity的value属性，默认是类名小写，如@Entity(name = “t_user”)，取出的值就是t_user
@Query(value="select u.id, u.name from  #{#entityName} u where u.id=?1", nativeQuery = true)
List<User> getUserById(Integer id);
```

自定义列-属性映射

```java
@Query("select new com.ljw.test.pojo.domain.UserDept(" +
            "u.id, u.name, d.id, d.name ) " +
            "from User u, Dept d " +
            "where u.deptId=d.id")
    List<UserDept> findAllForUserDept();

    @Query("select new map(" +
            "u.id as user_id, u.name as user_name, d.id as dept_id, d.name as dept_name) " +
            "from User u, Dept d " +
            "where u.deptId=d.id")
    List<Map<String, Object>> findAllForMap();
```

### Specification

高级查询,Specification为条件接口

```java
public interface JpaSpecificationExecutor<T> {
    T findOne(Specification<T> var1); //查询一个
    List<T> findAll(Specification<T> var1);//条件产讯
    Page<T> findAll(Specification<T> var1, Pageable var2);//分页查询
    List<T> findAll(Specification<T> var1, Sort var2);//排序查询
    long count(Specification<T> var1);//计数查询
}
```

### 关联与对多查询

@OneToMany：建立一对多的关系映射
属性：
        targetEntityClass：指定多的一方的类的字节码
        mappedBy：指定从表实体类中引用主表对象的名称。
        cascade：指定要使用的级联操作
        fetch：指定是否采用延迟加载
        orphanRemoval：是否使用删除

@ManyToOne :  建立多对一的关系
属性：
        targetEntityClass：指定一的一方实体类字节码
        cascade：指定要使用的级联操作
        fetch：指定是否采用延迟加载
        optional：关联是否可选。如果设置为false，则必须始终存在非空关系。

@JoinColumn: 用于定义主键字段和外键字段的对应关系。
属性：
        name：指定外键字段的名称
        referencedColumnName：指定引用主表的主键字段名称
        unique：是否唯一。默认值不唯一
        nullable：是否允许为空。默认值允许。
        insertable：是否允许插入。默认值允许。
        updatable：是否允许更新。默认值允许。
        columnDefinition：列的定义信息。

@ManyToMany: 用于映射多对多关系
属性：
        cascade：配置级联操作。
        fetch：配置是否采用延迟加载。
        targetEntity：配置目标的实体类。映射多对多的时候不用写。

@JoinTable :  针对中间表的配置
属性：
        nam：配置中间表的名称
        joinColumns：中间表的外键字段关联当前实体类所对应表的主键字段                          
        inverseJoinColumn：中间表的外键字段关联对方表的主键字段

### 多数据应用

在多DataSouce配置下, 配置多EntityManager和TrasactionManagement:

```java
@Configuration
@EnableTransactionManagement
@EnableJpaRepositories(
        entityManagerFactoryRef="entityManagerFactoryPrimary",
        transactionManagerRef="transactionManagerPrimary",
        basePackages= { "com.didispace.domain.p" }) //设置Repository所在位置
public class PrimaryConfig {
    @Autowired 
    @Qualifier("primaryDataSource")
    private DataSource primaryDataSource;

    @Primary
    @Bean(name = "entityManagerPrimary")
    public EntityManager entityManager(EntityManagerFactoryBuilder builder) {
        return entityManagerFactoryPrimary(builder).getObject().createEntityManager();
    }

    @Primary
    @Bean(name = "entityManagerFactoryPrimary")
    public LocalContainerEntityManagerFactoryBean entityManagerFactoryPrimary (EntityManagerFactoryBuilder builder) {
        return builder
                .dataSource(primaryDataSource)
                .properties(getVendorProperties(primaryDataSource))
                .packages("com.didispace.domain.p") //设置实体类所在位置
                .persistenceUnit("primaryPersistenceUnit")
                .build();
    }

    @Autowired
    private JpaProperties jpaProperties;

    private Map<String, String> getVendorProperties(DataSource dataSource) {
        return jpaProperties.getHibernateProperties(dataSource);
    }

    @Primary
    @Bean(name = "transactionManagerPrimary")
    public PlatformTransactionManager transactionManagerPrimary(EntityManagerFactoryBuilder builder) {
        return new JpaTransactionManager(entityManagerFactoryPrimary(builder).getObject());
    }
}
```

```java
@Configuration
@EnableTransactionManagement
@EnableJpaRepositories(
        entityManagerFactoryRef="entityManagerFactorySecondary",
        transactionManagerRef="transactionManagerSecondary",
        basePackages= { "com.didispace.domain.s" }) //设置Repository所在位置
public class SecondaryConfig {
    @Autowired 
    @Qualifier("secondaryDataSource")
    private DataSource secondaryDataSource;

    @Bean(name = "entityManagerSecondary")
    public EntityManager entityManager(EntityManagerFactoryBuilder builder) {
        return entityManagerFactorySecondary(builder).getObject().createEntityManager();
    }

    @Bean(name = "entityManagerFactorySecondary")
    public LocalContainerEntityManagerFactoryBean entityManagerFactorySecondary (EntityManagerFactoryBuilder builder) {
        return builder
                .dataSource(secondaryDataSource)
                .properties(getVendorProperties(secondaryDataSource))
                .packages("com.didispace.domain.s") //设置实体类所在位置
                .persistenceUnit("secondaryPersistenceUnit")
                .build();
    }

    @Autowired
    private JpaProperties jpaProperties;

    private Map<String, String> getVendorProperties(DataSource dataSource) {
        return jpaProperties.getHibernateProperties(dataSource);
    }

    @Bean(name = "transactionManagerSecondary")
    PlatformTransactionManager transactionManagerSecondary(EntityManagerFactoryBuilder builder) {
        return new JpaTransactionManager(entityManagerFactorySecondary(builder).getObject());
    }
}
```

## MyBatis

```xml
<dependency>
    <groupId>org.mybatis.spring.boot</groupId>
    <artifactId>mybatis-spring-boot-starter</artifactId>
    <version>2.1.1</version>
</dependency>
```

```properties
spring.datasource.url=jdbc:mysql://localhost:3306/test
spring.datasource.username=root
spring.datasource.password=
spring.datasource.driver-class-name=com.mysql.cj.jdbc.Driver

mybatis.mapper-locations=classpath:mapper/*.xml
#sql打印
mybatis.configuraion.log-impl=org.apache.ibatis.logging.stuout.StdOutImpl
# 自动将数据库带下划线的表字段值映射到Java类的驼峰字段上
mybatis.configuration.map-underscore-to-camel-case=true
```

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE mapper
        PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN"
        "http://mybatis.org/dtd/mybatis-3-mapper.dtd">
<mapper namespace="com.xxx.xxx.mapper.xxxMapper">
</mapper>
```

### typeHandler

MyBatis 在预处理语句中设置一个参数和从结果集中取出一个值时， 都会类型处理器进行转换.

![img](https://img-blog.csdnimg.cn/20181219120716455.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQzNjUwNzcz,size_16,color_FFFFFF,t_70)

![img](https://img-blog.csdnimg.cn/20181219120846757.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQzNjUwNzcz,size_16,color_FFFFFF,t_70)

通过实现 `org.apache.ibatis.type.TypeHandler `接口 或继承`org.apache.ibatis.type.BaseTypeHandler`可自定义类型转换器, 然后可以选择性地将它映射到一个JDBC类型。

比如：一个Java中的Date数据类型，我想将之存到数据库的时候存成一个1970年至今的毫秒数，取出来时转换成java的Date，即java的Date与数据库的varchar毫秒值之间转换。

```java
public class MyDateTypeHandler extends BaseTypeHandler<Date> {
    //把日期转换成毫秒, 存入数据库
    public void setNonNullParameter(PreparedStatement preparedStatement, int i, Date date, JdbcType type) {
        preparedStatement.setString(i,date.getTime()+"");
    }
    //以下均为将数据库取出的列转换为Data
    public Date getNullableResult(ResultSet resultSet, String s) throws SQLException {
        return new Date(resultSet.getLong(s));
    }
    public Date getNullableResult(ResultSet resultSet, int i) throws SQLException {
        return new Date(resultSet.getLong(i));
    }
    public Date getNullableResult(CallableStatement callableStatement, int i) throws SQLException {
        return callableStatement.getDate(i);
    }
}
```

如果要使用自定义类型转换器, 可:

1.全局注册

```proper
# 注册
mybatis.type-handlers-package=xxx.xxx.xxx
```

2.局部使用(建议)

```xml
<resultMap id="UserResultMap" type="com.entity.User">
    <id column="id" jdbcType="VARCHAR" property="id" />
    <result column="name" jdbcType="VARCHAR" property="name" />
    <result column="date" jdbcType="VARCHAR" property="date"
            javaType="Date" typeHandler="xxx.xxx.xxx.MyDateTypeHandler" />
</resultMap>
```

### pagehelper

```xml
<dependency>
    <groupId>com.github.pagehelper</groupId>
    <artifactId>pagehelper-spring-boot-starter</artifactId>
    <version>1.3.1</version>
</dependency>
<dependency>
       <groupId>com.github.pagehelper</groupId>
       <artifactId>pagehelper-spring-boot-autoconfigure</artifactId>
       <version>1.3.1</version>
</dependency>
```

简单使用

```java
@Test
public void testPageHelper(){
    //设置分页参数
    PageHelper.startPage(1,2);

    List<User> select = userMapper2.select(null);
    for(User user : select){
        System.out.println(user);
    }
}

//其他分页的数据
PageInfo<User> pageInfo = new PageInfo<User>(select);
System.out.println("总条数："+pageInfo.getTotal());
System.out.println("总页数："+pageInfo.getPages());
System.out.println("当前页："+pageInfo.getPageNum());
System.out.println("每页显示长度："+pageInfo.getPageSize());
System.out.println("是否第一页："+pageInfo.isIsFirstPage());
System.out.println("是否最后一页："+pageInfo.isIsLastPage());
```

自定义配置方式1:

```yml
#注意 pagehelper 配置，因为分页插件根据自己的扩展不同，支持的参数也不同，所以不能用固定的对象接收参数，所以这里使用的 Map<String,String>，因此参数名是什么这里就写什么，IDE 也不会有自动提示。
pagehelper:
    helperDialect: mysql
    reasonable: true
    supportMethodsArguments: true
    params: count=countSql
```

自定义配置方式2:

```java
@Configuration
public class PageHelperConfig {
    @Bean
    public PageHelper getPageHelper(){
        PageHelper pageHelper=new PageHelper();
        Properties properties=new Properties();
        properties.setProperty("helperDialect","mysql");
        properties.setProperty("reasonable","true");
        properties.setProperty("supportMethodsArguments","true");
        properties.setProperty("params","count=countSql");
        pageHelper.setProperties(properties);
        return pageHelper;
    }
```

参数详见https://github.com/pagehelper/Mybatis-PageHelper/blob/master/wikis/zh/HowToUse.md

1. `helperDialect`：分页插件会自动检测当前的数据库链接，自动选择合适的分页方式。 你可以配置`helperDialect`属性来指定分页插件使用哪种方言。配置时，可以使用下面的缩写值：
   `oracle`,`mysql`,`mariadb`,`sqlite`,`hsqldb`,`postgresql`,`db2`,`sqlserver`,`informix`,`h2`,`sqlserver2012`,`derby`
   **特别注意：**使用 SqlServer2012 数据库时，需要手动指定为 `sqlserver2012`，否则会使用 SqlServer2005 的方式进行分页。
   你也可以实现 `AbstractHelperDialect`，然后配置该属性为实现类的全限定名称即可使用自定义的实现方法。
2. `offsetAsPageNum`：默认值为 `false`，该参数对使用 `RowBounds` 作为分页参数时有效。 当该参数设置为 `true` 时，会将 `RowBounds` 中的 `offset` 参数当成 `pageNum` 使用，可以用页码和页面大小两个参数进行分页。
3. `rowBoundsWithCount`：默认值为`false`，该参数对使用 `RowBounds` 作为分页参数时有效。 当该参数设置为`true`时，使用 `RowBounds` 分页会进行 count 查询。
4. `pageSizeZero`：默认值为 `false`，当该参数设置为 `true` 时，如果 `pageSize=0` 或者 `RowBounds.limit = 0` 就会查询出全部的结果（相当于没有执行分页查询，但是返回结果仍然是 `Page` 类型）。
5. `reasonable`：分页合理化参数，默认值为`false`。当该参数设置为 `true` 时，`pageNum<=0` 时会查询第一页， `pageNum>pages`（超过总数时），会查询最后一页。默认`false` 时，直接根据参数进行查询。
6. `params`：为了支持`startPage(Object params)`方法，增加了该参数来配置参数映射，用于从对象中根据属性名取值， 可以配置 `pageNum,pageSize,count,pageSizeZero,reasonable`，不配置映射的用默认值， 默认值为`pageNum=pageNum;pageSize=pageSize;count=countSql;reasonable=reasonable;pageSizeZero=pageSizeZero`。
7. `supportMethodsArguments`：支持通过 Mapper 接口参数来传递分页参数，默认值`false`，分页插件会从查询方法的参数值中，自动根据上面 `params` 配置的字段中取值，查找到合适的值时就会自动分页。 使用方法可以参考测试代码中的 `com.github.pagehelper.test.basic` 包下的 `ArgumentsMapTest` 和 `ArgumentsObjTest`。
8. `autoRuntimeDialect`：默认值为 `false`。设置为 `true` 时，允许在运行时根据多数据源自动识别对应方言的分页 （不支持自动选择`sqlserver2012`，只能使用`sqlserver`），用法和注意事项参考下面的**场景五**。
9. `closeConn`：默认值为 `true`。当使用运行时动态数据源或没有设置 `helperDialect` 属性自动获取数据库类型时，会自动获取一个数据库连接， 通过该属性来设置是否关闭获取的这个连接，默认`true`关闭，设置为 `false` 后，不会关闭获取的连接，这个参数的设置要根据自己选择的数据源来决定。
10. `aggregateFunctions`(5.1.5+)：默认为所有常见数据库的聚合函数，允许手动添加聚合函数（影响行数），所有以聚合函数开头的函数，在进行 count 转换时，会套一层。其他函数和列会被替换为 count(0)，其中count列可以自己配置。

### 主键

1.通过insert标签的两个属性:

> useGeneratedKeys = true　　返回自增主键值
> keyProperty = "xxx"　　将值赋给哪个属性，这个属性是方法参数中的

```xml
<insert id = "insert" useGeneratedKeys = "true" keyProperty="userId" parameterType="com.xxx.User">
   ...
</insert>
```

该方式同样适用于返回List的情况.

2.通过selectKey标签自增

> resultType：返回类型
> order：BEFORE在添加之前查询　AFTER在添加之后查询　　//这两个都是全大写
> keyProperty：将取值赋值给方法参数，如果方法参数是实体类，一般赋值给实体类的字段
> keyColumn：对应表的列名

```xml
<insert id = "insertEmp">
    <selectKey resultType = "integer" order = "AFTER" keyProperty = "eid" >
         select last_insert_id()    
    </selectKey>
    insert into dept(id,deptname) values(#{id},#{deptname})
</insert>
```

3.在插入前使用mysql的uuid或其他sql获取id方式作为主键id

```xml
<insert id = "insertDept">
    <selectKey resultType = "string" order = "BEFORE" keyProperty = "id">
        select uuid() as id
    </selectKey>
    insert into dept(id,name) values(#{id},#{name})
</insert>
```

### 注解

```java
@Insert("INSERT INTO USER(NAME, AGE) VALUES(#{name}, #{age})")
int insert(@Param("name") String name, @Param("age") Integer age);
```

**结果绑定**

```java
@Results({
    @Result(property = "name", column = "name"),
    @Result(property = "age", column = "age")
})
@Select("SELECT name, age FROM user")
List<User> findAll();
```

可以给results指定id属性然后使用resultMap复用

```java
@ResultMap("AuthorCodeResult")
Optional<AuthorCode> selectOne(SelectStatementProvider selectStatement);

@Results(id="AuthorCodeResult", value = {
    @Result(column="id", property="id", jdbcType=JdbcType.BIGINT, id=true),
    @Result(column="invite_code", property="inviteCode", jdbcType=JdbcType.VARCHAR),
    @Result(column="validity_time", property="validityTime", jdbcType=JdbcType.TIMESTAMP),
    @Result(column="is_use", property="isUse", jdbcType=JdbcType.TINYINT),
    @Result(column="create_time", property="createTime", jdbcType=JdbcType.TIMESTAMP),
    @Result(column="create_user_id", property="createUserId", jdbcType=JdbcType.BIGINT)
    })
List<AuthorCode> selectMany(SelectStatementProvider selectStatement);
```

**使用Map<String,String>和对象作参数**

```java
@Insert("INSERT INTO USER(NAME, AGE) VALUES(#{name,jdbcType=VARCHAR}, #{age,jdbcType=INTEGER})")
int insertByMap(Map<String, Object> map);

@Insert("INSERT INTO USER(NAME, AGE) VALUES(#{name}, #{age})")
int insertByUser(User user);
```

**一对一**

```java
public interface OrderMapper {
    @Select("select * from orders")
    @Results({
            @Result(id=true,property = "id",column = "id"),
            @Result(property = "ordertime",column = "ordertime"),
            @Result(property = "total",column = "total"),
            @Result(property = "user",column = "uid",
                    javaType = User.class,
                    //select属性 代表查询哪个接口的方法获得数据
                    one = @One(select = "com.itheima.mapper.UserMapper.findById"))
    })
    List<Order> findAll();
}
public interface UserMapper {
    @Select("select * from user where id=#{id}")
    User findById(int id);
}
```

或者联合查询, 也能实现一对一

```java
@Select("select *,o.id oid from orders o,user u where o.uid=u.id")
@Results({
        @Result(column = "oid",property = "id"),
        @Result(column = "ordertime",property = "ordertime"),
        @Result(column = "total",property = "total"),
        @Result(column = "uid",property = "user.id"),
        @Result(column = "username",property = "user.username"),
        @Result(column = "password",property = "user.password")
})
public List<Order> findAll();
```

**一对多**

```java
public interface UserMapper {
    @Select("select * from user")
    @Results({
            @Result(id = true,property = "id",column = "id"),
            @Result(property = "username",column = "username"),
            @Result(property = "password",column = "password"),
            @Result(property = "birthday",column = "birthday"),
            @Result(property = "orderList",column = "id",
                    javaType = List.class,
                    many = @Many(select = "com.itheima.mapper.OrderMapper.findByUid"))
    })
    List<User> findAllUserAndOrder();
}

public interface OrderMapper {
    @Select("select * from orders where uid=#{uid}")
    List<Order> findByUid(int uid);
}
```

### 多数据应用

在多DataSouce配置下, 配置多个SqlSessionFactory和SqlSessionTemplate:

```java
@Configuration
@MapperScan(
        basePackages = "com.xxx.xxx.p",
        sqlSessionFactoryRef = "sqlSessionFactoryPrimary",
        sqlSessionTemplateRef = "sqlSessionTemplatePrimary")
public class PrimaryConfig {
    private DataSource primaryDataSource;

    public PrimaryConfig(@Qualifier("primaryDataSource") DataSource primaryDataSource) {
        this.primaryDataSource = primaryDataSource;
    }

    @Bean
    public SqlSessionFactory sqlSessionFactoryPrimary() throws Exception {
        SqlSessionFactoryBean bean = new SqlSessionFactoryBean();
        bean.setDataSource(primaryDataSource);
        return bean.getObject();
    }

    @Bean
    public SqlSessionTemplate sqlSessionTemplatePrimary() throws Exception {
        return new SqlSessionTemplate(sqlSessionFactoryPrimary());
    }
}
```

```java
@Configuration
@MapperScan(
        basePackages = "com.xxx.xxx.s",
        sqlSessionFactoryRef = "sqlSessionFactorySecondary",
        sqlSessionTemplateRef = "sqlSessionTemplateSecondary")
public class SecondaryConfig {
    private DataSource secondaryDataSource;

    public SecondaryConfig(@Qualifier("secondaryDataSource") DataSource secondaryDataSource) {
        this.secondaryDataSource = secondaryDataSource;
    }

    @Bean
    public SqlSessionFactory sqlSessionFactorySecondary() throws Exception {
        SqlSessionFactoryBean bean = new SqlSessionFactoryBean();
        bean.setDataSource(secondaryDataSource);
        return bean.getObject();
    }

    @Bean
    public SqlSessionTemplate sqlSessionTemplateSecondary() throws Exception {
        return new SqlSessionTemplate(sqlSessionFactorySecondary());
    }
}
```

配置类上使用`@MapperScan`注解来指定当前数据源下定义的Entity和Mapper的包路径；另外需要指定sqlSessionFactory和sqlSessionTemplate，这两个具体实现在该配置类中类中初始化。

### MyBatis Dynamic SQL

## 通用Mapper

```xml
<dependency>
   <groupId>org.mybatis.spring.boot</groupId>
   <artifactId>mybatis-spring-boot-starter</artifactId>
   <version>1.3.1</version>
</dependency>
<dependency>
   <groupId>tk.mybatis</groupId>
   <artifactId>mapper-spring-boot-starter</artifactId>
   <version>1.1.5</version>
</dependency>
```

```yml
mybatis:
  config-location: classpath:config/mybatis-config.xml
  # type-aliases扫描路径
  type-aliases-package: demo.springboot.test.domain
  # mapper xml实现扫描路径
  mapper-locations: classpath:mapper/*.xml
  property:
    order: BEFORE


#mappers 多个接口时逗号隔开
mapper:
  mappers: demo.springboot.test.config.MyMapper
  not-empty: false
  identity: oracle
```

```java
@Table(name = "T_USER")
public class User {
    @Id
    @Column(name = "ID")
    //@GeneratedValue(strategy = GenerationType.IDENTITY)  //oracle没有主键自增功能,需要关闭
    private Long id;
    @Column(name = "USERNAME")
    private String username; 
    @Column(name = "PASSWD")
    private String passwd; 
    @Column(name = "CREATE_TIME")
    private Date createTime;
    @Column(name = "STATUS")
    private String status;
    ...
}
```

```java
import tk.mybatis.mapper.common.Mapper;
import tk.mybatis.mapper.common.MySqlMapper;
public interface MyMapper<T> extends Mapper<T>, MySqlMapper<T> {

}

public interface UserMapper extends MyMapper<User> {
}
```

### Generator

```xml
<build>
    <plugins>
        <plugin>
            <groupId>org.mybatis.generator</groupId>
            <artifactId>mybatis-generator-maven-plugin</artifactId>
            <version>1.3.5</version>
            <dependencies>
                <dependency>
                    <!-- 数据库连接驱动 -->
                    <groupId>com.oracle</groupId>
                    <artifactId>ojdbc6</artifactId>
                    <version>6.0</version>
                </dependency>
                <dependency>
                    <groupId>tk.mybatis</groupId>
                    <artifactId>mapper</artifactId>
                    <version>3.4.0</version>
                </dependency>
            </dependencies>
            <executions>
                <execution>
                    <id>Generate MyBatis Artifacts</id>
                    <phase>package</phase>
                    <goals>
                        <goal>generate</goal>
                    </goals>
                </execution>
            </executions>
            <configuration>
                <!--允许移动生成的文件 -->
                <verbose>true</verbose>
                <!-- 是否覆盖 -->
                <overwrite>true</overwrite>
                <!-- 自动生成的配置 -->
                <configurationFile>src/main/resources/mybatis-generator.xml</configurationFile>
            </configuration>
        </plugin>
    </plugins>
</build>
```

mybatis-generator.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE generatorConfiguration
    PUBLIC "-//mybatis.org//DTD MyBatis Generator Configuration 1.0//EN"
    "http://mybatis.org/dtd/mybatis-generator-config_1_0.dtd">
<generatorConfiguration>
    <context id="oracle" targetRuntime="MyBatis3Simple" defaultModelType="flat">

        <plugin type="tk.mybatis.mapper.generator.MapperPlugin">
            <!-- 该配置会使生产的Mapper自动继承MyMapper -->
            <property name="mappers" value="com.springboot.config.MyMapper" />
            <!-- caseSensitive默认false，当数据库表名区分大小写时，可以将该属性设置为true -->
            <property name="caseSensitive" value="false"/>
        </plugin>

        <!-- 阻止生成自动注释 -->
        <commentGenerator>
            <property name="javaFileEncoding" value="UTF-8"/>
            <property name="suppressDate" value="true"/>
            <property name="suppressAllComments" value="true"/>
        </commentGenerator>

        <!-- 数据库链接地址账号密码 -->
        <jdbcConnection 
            driverClass="oracle.jdbc.driver.OracleDriver"
            connectionURL="jdbc:oracle:thin:@localhost:1521:ORCL"
            userId="admin"
            password="123456">
        </jdbcConnection>

        <javaTypeResolver>
            <property name="forceBigDecimals" value="false"/>
        </javaTypeResolver>

        <!-- 生成Model类存放位置 -->
        <javaModelGenerator targetPackage="com.springboot.bean" targetProject="src/main/java">
            <property name="enableSubPackages" value="true"/>
            <property name="trimStrings" value="true"/>
        </javaModelGenerator>

        <!-- 生成映射文件存放位置 -->
        <sqlMapGenerator targetPackage="mapper" targetProject="src/main/resources">
            <property name="enableSubPackages" value="true"/>
        </sqlMapGenerator>

        <!-- 生成Dao类存放位置 -->
        <!-- 客户端代码，生成易于使用的针对Model对象和XML配置文件的代码
            type="ANNOTATEDMAPPER",生成Java Model 和基于注解的Mapper对象
            type="XMLMAPPER",生成SQLMap XML文件和独立的Mapper接口 -->
        <javaClientGenerator type="XMLMAPPER" targetPackage="com.springboot.mapper" targetProject="src/main/java">
            <property name="enableSubPackages" value="true"/>
        </javaClientGenerator>

        <!-- 配置需要生成的表 -->
        <table tableName="T_USER" domainObjectName="User" enableCountByExample="false" enableUpdateByExample="false" enableDeleteByExample="false" enableSelectByExample="false" selectByExampleQueryId="false">
            <generatedKey column="id" sqlStatement="oralce" identity="true"/>
        </table>
    </context>
</generatorConfiguration>
```

运行命令`mybatis-generator:generate`生成

将自动生成实体类和对应的继承了配置中指定的MyMapper的mapper.

### Example使用

```java
Example example = new Example(User.class);
example.createCriteria().andCondition("username like '%i%'");
example.setOrderByClause("id desc");
List<User> userList = this.userService.selectByExample(example);
for (User u : userList) {
    System.out.println(u.getUsername());
}
```

### 模板service

自定义service统一通用接口.

```java
@Service
public interface IService<T> {
    List<T> selectAll();    
    T selectByKey(Object key);
    int save(T entity);
    int delete(Object key);
    int updateAll(T entity);
    int updateNotNull(T entity);
    List<T> selectByExample(Object example);
}

public abstract class BaseService<T> implements IService<T> {
    @Autowired
    protected Mapper<T> mapper;

    public Mapper<T> getMapper() {
        return mapper;
    }    
    @Override
    public List<T> selectAll() {
        return mapper.selectAll();
    } 
    @Override
    public T selectByKey(Object key) {
        return mapper.selectByPrimaryKey(key);
    }
    @Override
    public int save(T entity) {
        return mapper.insert(entity);
    }
    @Override
    public int delete(Object key) {
        return mapper.deleteByPrimaryKey(key);
    }
    @Override
    public int updateAll(T entity) {
        return mapper.updateByPrimaryKey(entity);
    }
    @Override
    public int updateNotNull(T entity) {
        return mapper.updateByPrimaryKeySelective(entity);
    }
    @Override
    public List<T> selectByExample(Object example) {
        //重点：这个查询支持通过Example类指定查询列，通过selectProperties方法指定查询列
        return mapper.selectByExample(example);
    }
}
```

封装特定查询接口:

```java
public interface UserService extends IService<User>{  
   User findByName(String userName);
}

@Repository("userService")
public class UserServiceImpl extends BaseService<User> implements UserService {
    @Override
    public User findByName(String userName) {
        Example example = new Example(User.class);
        example.createCriteria().andCondition("username=", userName);
        List<User> userList = this.selectByExample(example);
        if (userList.size() != 0)
            return userList.get(0);
        else
            return null;
    }
}
```

## RedisTemplate

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-redis</artifactId>
</dependency>
```

```properties
spring.redis.database=0
spring.redis.host=localhost
spring.redis.port=6379
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

```java
@Autowired  //StringRedisTemplate相当于RedisTemplate<String, String>的实现。
private StringRedisTemplate stringRedisTemplate;
@Test
public void test() throws Exception {
    // 保存字符串
    stringRedisTemplate.opsForValue().set("aaa", "111");
    Assert.assertEquals("111", stringRedisTemplate.opsForValue().get("aaa"));
}
```

通过RedisTemplate也可以获取连接信息:

```java
//在当前连接下执行动作 这里为获取连接信息
Properties info = (Properties) redisTemplate.execute((RedisCallback<Object>) RedisServerCommands::info);
//仅获取命令信息
Properties commandStats = (Properties) redisTemplate.execute((RedisCallback<Object>) connection -> connection.info("commandstats"));
//数据大小 相当于key数量
Object dbSize = redisTemplate.execute((RedisCallback<Object>) RedisServerCommands::dbSize);
```

### RedisSerializer

可以为RedisTemplate指定序列化器, 通常key序列化采用自带的`StringRedisSerializer`即可, 而value的序列化可以实现多种定制，实现RedisSerializer接口即可, 如自定义对象序列化:

```java
//内部利用Converter原生序列化  序列化的对象需要实现serializer接口(不确定)
public class RedisObjectSerializer implements RedisSerializer<Object> {
  private Converter<Object, byte[]> serializer = new SerializingConverter();
  private Converter<byte[], Object> deserializer = new DeserializingConverter();
  static final byte[] EMPTY_ARRAY = new byte[0];

  public Object deserialize(byte[] bytes) {
    if (isEmpty(bytes)) return null;
    return deserializer.convert(bytes);
  }
  public byte[] serialize(Object object) {
    if (object == null) return EMPTY_ARRAY;
    return serializer.convert(object);
  }
  private boolean isEmpty(byte[] data) {
    return (data == null || data.length == 0);
  }
}
```

然后进行配置:

```java
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

也可以利用FastJson实现:

```java
//内部利用fastjson实现
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

或直接使用Jackson提供的Jackson2JsonRedisSerializer:

```
@Bean
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

## Transaction

在Spring Boot中，当我们使用了spring-boot-starter-jdbc或spring-boot-starter-data-jpa依赖的时候，框架会自动默认分别注入DataSourceTransactionManager或JpaTransactionManager。所以我们不需要任何额外配置就可以用@Transactional注解进行事务的使用。(老版本可能需要加`@EnableTransactionManagement`)

回滚日志:

```
o.s.t.c.transaction.TransactionContext   : Rolled back transaction for ...
```

> 使用`@Rollback`注解让每个单元测试都能在结束时回滚

**在多数据源时,指定事务管理器:**

```java
@Transactional(value="transactionManagerPrimary")
```

`@Transactional` 的常用配置参数总结（只列出了 5 个平时比较常用的):

| 属性名         | 说明                                                              |
|:----------- |:--------------------------------------------------------------- |
| propagation | 事务的传播行为，默认值为 REQUIRED                                           |
| isolation   | 事务的隔离级别，默认值采用 DEFAULT, 即使用底层数据库默认隔离级别.                          |
| timeout     | 事务的超时时间，默认值为-1（不会超时）。如果超过该时间限制但事务还没有完成，则自动回滚事务。                 |
| readOnly    | 指定事务是否为只读事务，默认值为 false。                                         |
| rollbackFor | 用于指定能够触发事务回滚的异常类型，并且可以指定多个异常类型。默认只会在遇到`RuntimeException`的时候才会回滚 |

> 传播行为: 如果在开始当前事务之前，一个事务上下文已经存在，此时有若干选项可以指定一个事务性方法的执行行为。即何时要创建一个事务，或者何时使用已有的事务.

`REQUIRED`：默认 . 如果当前存在事务，则加入该事务；如果当前没有事务，则创建一个新的事务。

- `SUPPORTS`：如果当前存在事务，则加入该事务；如果当前没有事务，则以非事务的方式继续运行。
- `MANDATORY`：如果当前存在事务，则加入该事务；如果当前没有事务，则抛出异常。
- `REQUIRES_NEW`：创建一个新的事务，如果当前存在事务，则把当前事务挂起。
- `NOT_SUPPORTED`：以非事务方式运行，如果当前存在事务，则把当前事务挂起。
- `NEVER`：以非事务方式运行，如果当前存在事务，则抛出异常。
- `NESTED`：如果当前存在事务，则创建一个事务作为当前事务的嵌套事务来运行(外部主事务回滚的话，子事务也会回滚，而内部子事务可以单独回滚而不影响外部主事务和其他子事务。)；如果当前没有事务，则该取值等价于`REQUIRED`。

### 事务无效分析

1.MySQL MyISAM

```
spring.jpa.database-platform=org.hibernate.dialect.MySQL5InnoDBDialect
```

若使用JPA时, 默认会使用MySQL的MyISAM, 不支持事务.

2.`@Transactional`注解修饰的函数中`catch`了异常，并没有往方法外抛。

3.`@Transactional`注解修饰的函数不是`public`类型

4.异常类型错误，如果有通过rollbackFor指定回滚的异常类型，那么抛出的异常与指定的是否一致。

5.数据源没有配置事务管理器

6.在一个类中调用自己的方法。如

```java
@Service
public class UserServiceImpl implements UserService {
    //调用此方法 事务将失效
    @Override
    public void saveUserTest(User user) {
        this.saveUser(user);
    }
    @Transactional
    @Override
    public void saveUser(User user) {
        userMapper.save(user);
        // 测试事务回滚
        if (!StringUtils.hasText(user.getUsername())) {
            throw new ParamInvalidException("username不能为空");
        }
    }
}
```

Spring事务控制使用AOP代理实现，通过对目标对象的代理来增强目标方法,  这里this指的是被代理对象.

### 编程式事务管理

上面为通常注解式事务管理, 

通过 `TransactionTemplate`或者`TransactionManager`手动管理事务，实际应用中很少使用

1.使用`TransactionTemplate` 进行编程式事务管理的示例代码如下：

```java
@Autowired
private TransactionTemplate transactionTemplate;
public void testTransaction() {
        transactionTemplate.execute(new TransactionCallbackWithoutResult() {
            @Override
            protected void doInTransactionWithoutResult(TransactionStatus transactionStatus) {
                try {

                    // ....  业务代码
                } catch (Exception e){
                    //回滚
                    transactionStatus.setRollbackOnly();
                }
            }
        });
}
```

2.使用 `TransactionManager` 进行编程式事务管理的示例代码如下：

```java
@Autowired
private PlatformTransactionManager transactionManager;
public void testTransaction() {
  TransactionStatus status = transactionManager.getTransaction(new DefaultTransactionDefinition());
          try {
               // ....  业务代码
              transactionManager.commit(status);
          } catch (Exception e) {
              transactionManager.rollback(status);
          }
}
```

Spring事务最重要的 3 个接口为：

- `PlatformTransactionManager`： （平台）事务管理器，Spring 事务策略的核心。
- `TransactionDefinition`： 事务定义信息(事务隔离级别、传播行为、超时、只读、回滚规则)。
- `TransactionStatus`： 事务运行状态。

## Cache

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-cache</artifactId>
</dependency>
```

在Spring Boot主类中增加`@EnableCaching`注解开启缓存功能

Spring Boot根据下面的顺序去侦测缓存提供者：

Generic, JCache (JSR-107), EhCache 2.x, Hazelcast, Infinispan, Redis, Guava, Simple

或通过配置属性`spring.cache.type`来指定.

当我们不指定具体其他第三方实现的时候，Spring Boot的Cache模块会使用`ConcurrentHashMap`来存储.

> 可通过debug调试查看cacheManager对象的实例来判断当前使用了什么缓存
> 
> 可通过开启SQL打印查询多次验证缓存是否生效

若想编程式使用Cache, 可以直接通过CacheManager使用:

```
@Autowired
private  CacheManager cacheManager;

@Test
public void test1(){
    // 根据名称获取缓存对象
    Cache cache = cacheManager.getCache("test1");
    Element element = cache.get("key1");
    System.out.println(element.getObjectValue());
}
```

### 核心注解

- `@CacheConfig`：主要用于配置该类中会用到的一些共用的缓存配置。如`@CacheConfig(cacheNames = "users")`：配置了该数据访问对象中返回的内容将存储于名为users的缓存对象中(对应cacheManager.getCache(cacheName)方法)，我们也可以不使用该注解，直接通过`@Cacheable`自己配置缓存集的名字来定义。

- `@Cacheable`：配置了findByName函数的返回值将被加入缓存。同时在查询时，会先从缓存中获取，若不存在才再发起对数据库的访问。该注解主要有下面几个参数：
  
  - `value`、`cacheNames`：两个等同的参数，用于指定缓存存储的集合名。由于Spring 4中新增了`@CacheConfig`，因此在Spring 3中原本必须有的`value`属性，也成为非必需项了。
  - `key`：缓存对象存储在Map集合中的key值，非必需，缺省按照函数的所有参数组合作为key值，若自己配置需使用SpEL表达式，比如：`@Cacheable(key = "#p0")`：使用函数第一个参数作为缓存的key值。
  - `condition`：缓存对象的条件，非必需，也需使用SpEL表达式，只有满足表达式条件的内容才会被缓存，比如：`@Cacheable(key = "#p0", condition = "#p0.length() < 3")`，表示只有当第一个参数的长度小于3的时候才会被缓存，若做此配置上面的AAA用户就不会被缓存，读者可自行实验尝试。
  - `unless`：另外一个缓存条件参数，非必需，需使用SpEL表达式。它不同于`condition`参数的地方在于它的判断时机，该条件是在函数被调用之后才做判断的，所以它可以通过对result进行判断。
  - `keyGenerator`：用于指定key生成器，非必需。若需要指定一个自定义的key生成器，我们需要去实现`org.springframework.cache.interceptor.KeyGenerator`接口，并使用该参数来指定。需要注意的是：**该参数与`key`是互斥的**, 也可以通过`CachingConfigurerSupport`进行全局配置。
  - `cacheManager`：用于指定使用哪个缓存管理器，非必需。只有当有多个时才需要使用。
  - `cacheResolver`：用于指定使用那个缓存解析器，非必需。需通过。`org.springframework.cache.interceptor.CacheResolver`接口来实现自己的缓存解析器，并用该参数指定。

- `@CachePut`：配置于函数上，能够根据参数定义条件来进行缓存，它与`@Cacheable`不同的是，它每次都会真实调用函数，所以主要用于数据新增和修改操作上。它的参数与`@Cacheable`类似。

- `@CacheEvict`：配置于函数上，通常用在删除方法上，用来从缓存中移除相应数据。除了同`@Cacheable`一样的参数之外，它还有下面两个参数：
  
  - `allEntries`：非必需，默认为false。当为true时，会移除所有数据。
  - `beforeInvocation`：非必需，默认为false，会在调用方法之后移除数据。当为true时，会在调用方法之前移除数据。

### CachingConfigurerSupport

覆盖该类的方法可实现自定义cache配置, 如自定义KeyGenerator.

```java
@Configuration
public class RedisConfig extends CachingConfigurerSupport {
    // 自定义缓存key生成策略
    @Bean
    @Override
    public KeyGenerator keyGenerator() {
        return new KeyGenerator() {
            @Override
            public Object generate(Object target, java.lang.reflect.Method method, Object... params) {
                StringBuffer sb = new StringBuffer();
                sb.append(target.getClass().getName());
                sb.append(method.getName());
                for (Object obj : params) {
                    sb.append(obj.toString());
                }
                return sb.toString();
            }
        };
    }
    //---------------------------
    // 使用SHA256生成key 
    @Bean
    @Override
    public KeyGenerator keyGenerator() {
        return (target, method, params) -> {
            Map<String,Object> container = new HashMap<>(3);
            Class<?> targetClassClass = target.getClass();
            // 类地址
            container.put("class",targetClassClass.toGenericString());
                // 方法名称
            container.put("methodName",method.getName());
            // 包名称
            container.put("package",targetClassClass.getPackage());
            // 参数列表
            for (int i = 0; i < params.length; i++) {
                container.put(String.valueOf(i),params[i]);
            }
            // 转为JSON字符串
            String jsonString = JSON.toJSONString(container);
            // 做SHA256 Hash计算，得到一个SHA256摘要作为Key
            eturn DigestUtils.sha256Hex(jsonString);
        };
    }
}
```

或者自定义cache异常处理

```java
@Bean
@Override
public CacheErrorHandler errorHandler() {
    // 异常处理，当Redis发生异常时，打印日志，但是程序正常走
       log.info("初始化 -> [{}]", "Redis CacheErrorHandler");
    return new CacheErrorHandler() {
        @Override
        public void handleCacheGetError(RuntimeException e, Cache cache, Object key) {
            log.error("Redis occur handleCacheGetError：key -> [{}]", key, e);
        }
        @Override
        public void handleCachePutError(RuntimeException e, Cache cache, Object key, Object value) {
            log.error("Redis occur handleCachePutError：key -> [{}]；value -> [{}]", key, value, e);
        }
        @Override
        public void handleCacheEvictError(RuntimeException e, Cache cache, Object key) {
            log.error("Redis occur handleCacheEvictError：key -> [{}]", key, e);
        }
        @Override
        public void handleCacheClearError(RuntimeException e, Cache cache) {
            log.error("Redis occur handleCacheClearError：", e);
        }
    };
}
```

### EhCache

在Spring Boot中开启EhCache只需要在工程中加入`ehcache.xml`配置文件并在pom.xml中增加ehcache依赖，框架只要发现该文件，就会创建EhCache的CacheManager。

```xml
<dependency>
    <groupId>net.sf.ehcache</groupId>
    <artifactId>ehcache</artifactId>
</dependency>
```

在`src/main/resources`目录下创建：`ehcache.xml`

```xml
<ehcache xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:noNamespaceSchemaLocation="ehcache.xsd">
    <!-- 磁盘缓存位置,当内存中对象数量达到maxElementsInMemory时，将会写到磁盘中-->
    <diskStore path="user.dir/cachedata" />
    <!-- 默认缓存, 当ehcache找不到指定的缓存对象时，则使用这个缓存策略 -->
    <defaultCache maxElementsInMemory="10000" eternal="false"
        timeToIdleSeconds="120" timeToLiveSeconds="120"
        maxElementsOnDisk="10000000" diskExpiryThreadIntervalSeconds="120"
        memoryStoreEvictionPolicy="LRU">
       <!-- 缓存对象配置 --> 
    <cache name="users"
           maxEntriesLocalHeap="200"
           timeToLiveSeconds="600">
    </cache>
</ehcache>
```

> 如果不适用默认配置文件名, 可通过`spring.cache.ehcache.config`属性来指定

> EhCache可以配置集群, 但因为进程内独立, 即使EhCache提供了集群下的同步策略, 但不同服务器进程仍会出现缓存不一致.

### redis

集中式缓存    

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis</artifactId>
</dependency>
<dependency>
    <groupId>org.apache.commons</groupId>
    <artifactId>commons-pool2</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-redis</artifactId>
</dependency>
```

```properties
#2.x版本中采用了lettuce作为连接池
spring.redis.host=localhost
spring.redis.port=6379
spring.redis.lettuce.pool.max-idle=8
spring.redis.lettuce.pool.max-active=8
spring.redis.lettuce.pool.max-wait=-1ms
spring.redis.lettuce.pool.min-idle=0
spring.redis.lettuce.shutdown-timeout=100ms
#在1.x版本中采用jedis作为连接池
spring.redis.host=localhost
spring.redis.port=6379
spring.redis.pool.max-idle=8
spring.redis.pool.min-idle=0
spring.redis.pool.max-active=8
spring.redis.pool.max-wait=-1
#以上配置均为默认值，实际上生产需进一步根据部署情况与业务要求做适当修改
```

Spring Boot Cache在侦测到存在Redis的依赖并且Redis的配置是可用的情况下，使用`RedisCacheManager`初始化`CacheManager`。

也可以通过配置类自己注入,  以实现自定义cache配置, 如:

```java
@Bean
public CacheManager cacheManager(@SuppressWarnings("rawtypes") RedisTemplate redisTemplate) {
    RedisCacheManager cacheManager = new RedisCacheManager(redisTemplate);
    // 设置缓存过期时间
    cacheManager.setDefaultExpiration(10000);
    return cacheManager;
}
@Bean
public RedisTemplate<String, String> redisTemplate(RedisConnectionFactory factory) {
    StringRedisTemplate template = new StringRedisTemplate(factory);
    //... 这里可以做一些自定义设置, 如指定RedisSerializer
    template.afterPropertiesSet();
    return template;
}
```

## Guava Cache

构建Cache对象有两种形式:

1.CacheLoader : 构造 LoadingCache 的关键在于实现 load 方法,  **在需要访问key不存在的时候 Cache 会自动调用 load 方法将数据加载到 Cache 中**

```java
private static final LoadingCache<String, String> CACHE = CacheBuilder
    .newBuilder()
    // 最大容量为 100 超过容量有对应的淘汰机制
    .maximumSize(100) 
    // 缓存项写入后多久过期  类似的还有expireAfterAccess表示访问多久后过期
    .expireAfterWrite(60 * 5, TimeUnit.SECONDS)
    // 缓存写入后多久自动刷新一次 需要小于过期时间
    .refreshAfterWrite(60, TimeUnit.SECONDS)
    // 创建一个 CacheLoader，load 表示缓存不存在的时候加载到缓存并返回
    .build(new CacheLoader<String, String>() {
        // 加载缓存数据的方法
        @Override
        public String load(String key) {
            return "cache [" + key + "]";//这里为了简单直接省略数据库访问
        }
    });
```

2.Callable : 创建时不指定load, 但可以在获取key时指定载入缓存的方式, 更灵活, 适用于从多个数据源加载缓存的场景.

```java
// 注意返回值是 Cache 类型
private static final Cache<String, String> SIMPLE_CACHE = CacheBuilder
    .newBuilder()
    .build();

public void getTest1() throws Exception {
    String key = "KEY_25487";
    // get 缓存项的时候指定 callable 加载缓存项
    SIMPLE_CACHE.get(key, () -> "cache [" + key + "]");
}
```

> 回顾缓存击穿问题: 即如果某个key过期了或者key不存在于缓存中，而恰巧此时有大量请求过来请求这个key，如果没有保护机制就会导致大量的线程同时请求数据源加载数据.

Guava Cache 在 load 的时候做了并发控制，**在多个线程请求一个不存在或者过期的key时保证只有一个线程进入 load 方法，其他线程等待直到key被生成**，这样就避免了大量的线程击穿缓存直达 DB. 但新的问题是, 大QPS下大量线程阻塞的问题.

配置了refreshAfterWrite主动对key进行刷新(过期时间内),  但**前提是该key存在于cache中**, 因此需要提前预热, 该选项配合expireAfterWrite/expireAfterAccess可避免线程阻塞的同时保证缓存项及时更新. 但新问题是, 如果大量key同时过期或刷新, 会造成大量线程请求DB, 即**缓存雪崩**.

缓存雪崩可通过限制请求线程、控制数据库连接池大小等方法解决,  如果对一致性要求不高, 可以通过Guava Cache的LoadingCache异步加载解决, 即**通过后台线程异步刷新缓存, 所有请求返回旧值**:

```java
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

## 使用oracle序列值实现主键

提供获取序列值的mapper接口,

```java
public interface UserMapper {
    @Select("select ${seqName}.nextval from dual")
    Long getSequence(@Param("seqName") String seqName);
}
```

其中 seqName为我们自定义的序列名称, 在一个表中为了保证序列值不重复, 需要获取需要每次传入的序列名称相同

插入数据的方式:

```java
@Test
public void test() throws Exception {
    User user = new User();
    user.setId(userMapper.getSequence("seq_user"));
    user.setUsername("test");
    user.setPasswd("14123");
    user.setCreateTime(new Date());
    user.setStatus("0");
    this.userMapper.save(user);
}
```
