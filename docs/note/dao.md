# jdbcTemplate

简单多数据源

```
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

```
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

```
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

**说明与注意**：

1. 前两个Bean是数据源的创建，通过`@ConfigurationProperties`可以知道这两个数据源分别加载了`spring.datasource.primary.*`和`spring.datasource.secondary.*`的配置。
2. `@Primary`注解指定了主数据源，就是当我们不特别指定哪个数据源的时候，就会使用这个Bean
3. 后两个Bean是每个数据源对应的`JdbcTemplate`。可以看到这两个`JdbcTemplate`创建的时候，分别注入了`primaryDataSource`数据源和`secondaryDataSource`数据源

# JPA

```
<dependency
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-jpa</artifactId>
</dependency>
```

```
spring.datasource.url=jdbc:mysql://localhost:3306/test
spring.datasource.username=root
spring.datasource.password=
spring.datasource.driver-class-name=com.mysql.cj.jdbc.Driver

spring.jpa.properties.hibernate.hbm2ddl.auto=create-drop
新版本已经变为
spring.jpa.properties.ddl-auto=none
```

`spring.jpa.properties.hibernate.hbm2ddl.auto`是hibernate的配置属性，其主要作用是：自动创建、更新、验证数据库表结构。该参数的几种配置如下：

- `create`：每次加载hibernate时都会删除上一次的生成的表，然后根据你的model类再重新来生成新表，哪怕两次没有任何改变也要这样执行，这就是导致数据库表数据丢失的一个重要原因。
- `create-drop`：每次加载hibernate时根据model类生成表，但是sessionFactory一关闭,表就自动删除。
- `update`：最常用的属性，第一次加载hibernate时根据model类会自动建立起表的结构（前提是先建立好数据库），以后加载hibernate时根据model类自动更新表结构，即使表结构改变了但表中的行仍然存在不会删除以前的行。要注意的是当部署到服务器后，表结构是不会被马上建立起来的，是要等应用第一次运行起来后才会。
- `validate`：每次加载hibernate时，验证创建数据库表结构，只会和数据库中的表进行比较，不会创建新表，但是会插入新值。
- `none`:默认值

```
public interface UserRepository extends JpaRepository<User, Long> {

    User findByName(String name);

    User findByNameAndAge(String name, Integer age);

    @Query("from User u where u.name=:name")
    User findUser(@Param("name") String name);

}
```

如果需要使用mysql事务,需要设置`spring.jpa.database-platform`为MySQL5InnoDBDialect,保证Hibernate自动创建表时使用innodb,否则默认使用MyISAM创建.

Spring-data-jpa的一大特性：**通过解析方法名创建查询**。

## 列注解

@Table:可指定表名

@Column:

```
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
	 /**
     * 使用一个特定的数据库表格来保存主键
     * 持久化引擎通过关系数据库的一张特定的表格来生成主键,
     */
    TABLE,
    /**
     *在某些数据库中,不支持主键自增长,比如Oracle、PostgreSQL其提供了一种叫做"序列(sequence)"的机制生成主键
     */
    SEQUENCE,
    /**
     * 主键自增长
     */
    IDENTITY,
    /**
     *默认,把主键生成策略交给持久化引擎(persistence engine),
     *持久化引擎会根据数据库在以上三种主键生成 策略中选择其中一种
     */
    AUTO
```

>```java
>@Id
>@GeneratedValue(strategy = GenerationType.IDENTITY)
>private Long id;
>```
>
>等价于通过 `@GenericGenerator`声明一个主键策略，然后 `@GeneratedValue`使用这个策略
>
>```
>@Id
>@GeneratedValue(generator = "IdentityIdGenerator")
>@GenericGenerator(name = "IdentityIdGenerator", strategy = "identity")
>private Long id;
>```

jpa提供的主键生成策略:

```
public class DefaultIdentifierGeneratorFactory
		implements MutableIdentifierGeneratorFactory, Serializable, ServiceRegistryAwareService {

	@SuppressWarnings("deprecation")
	public DefaultIdentifierGeneratorFactory() {
		register( "uuid2", UUIDGenerator.class );
		register( "guid", GUIDGenerator.class );			// can be done with UUIDGenerator + strategy
		register( "uuid", UUIDHexGenerator.class );			// "deprecated" for new use
		register( "uuid.hex", UUIDHexGenerator.class ); 	// uuid.hex is deprecated
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

@CreationTimestamp:	生成创建时间

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

```
@Lob
//指定 Lob 类型数据的获取策略， FetchType.EAGER 表示非延迟加载，而 FetchType.LAZY 表示延迟加载 ；
@Basic(fetch = FetchType.EAGER)
//columnDefinition 属性指定数据表对应的 Lob 字段类型
@Column(name = "content", columnDefinition = "LONGTEXT NOT NULL")
private String content;
```

@Enumerated: 使用枚举类型的字段需要加的注解

### 审计

只要继承了 `AbstractAuditBase`的类都会默认加上下面四个字段。

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

我们对应的审计功能对应地配置类可能是下面这样的（Spring Security 项目）:

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

2. `@CreatedBy` :表示该字段为创建人，在这个实体被 insert 的时候，会设置值

   `@LastModifiedDate`、`@LastModifiedBy`同理。

`@EnableJpaAuditing`：开启 JPA 审计功能。

## @Query

定制化查询

```
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

```
@Query("select u.id, u.name from User u where u.id=?1")
    List<User> getUserById(Integer id);
```

`:参数名`方式(参数名需要为@Param指定)

```
@Query("select u.id, u.name from User u where u.name like %:name%")
List<User> getUserByName(@Param("name") String name);
```

原生SQL 只能使用`?序号`方式

```
@Query(value="select u.id, u.name from t_user u where u.id=?1", nativeQuery = true)
List<User> getUserById(Integer id);
```

SPEL表达式方式

```
//#{#entityName}会取@Entity()的值，默认是类名小写，可以申请如@Entity(name = “t_user”)，取出的值就是t_user
@Query(value="select u.id, u.name from  #{#entityName} u where u.id=?1", nativeQuery = true)
List<User> getUserById(Integer id);
```

**@Modifying注解代表允许修改或删除**

返回自定义字段

```
@Query("select u.id, u.name, d.id, d.name " +
            "from User u, Dept d " +
            "where u.deptId=d.id", nativeQuery = true)
    List<Object[]> findAllForUserDept();
```

返回自定义对象

```
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

## JpaSpecificationExecutor

高级查询,Specification为条件接口

```
public interface JpaSpecificationExecutor<T> {
    T findOne(Specification<T> var1); //查询一个

    List<T> findAll(Specification<T> var1);//条件产讯

    Page<T> findAll(Specification<T> var1, Pageable var2);//分页查询

    List<T> findAll(Specification<T> var1, Sort var2);//排序查询

    long count(Specification<T> var1);//计数查询
}
```

## 一对多,多对多

```
@OneToMany:
   	作用：建立一对多的关系映射
    属性：
    	targetEntityClass：指定多的多方的类的字节码
    	mappedBy：指定从表实体类中引用主表对象的名称。
    	cascade：指定要使用的级联操作
    	fetch：指定是否采用延迟加载
    	orphanRemoval：是否使用删除

@ManyToOne
    作用：建立多对一的关系
    属性：
    	targetEntityClass：指定一的一方实体类字节码
    	cascade：指定要使用的级联操作
    	fetch：指定是否采用延迟加载
    	optional：关联是否可选。如果设置为false，则必须始终存在非空关系。

@JoinColumn
     作用：用于定义主键字段和外键字段的对应关系。
     属性：
    	name：指定外键字段的名称
    	referencedColumnName：指定引用主表的主键字段名称
    	unique：是否唯一。默认值不唯一
    	nullable：是否允许为空。默认值允许。
    	insertable：是否允许插入。默认值允许。
    	updatable：是否允许更新。默认值允许。
    	columnDefinition：列的定义信息。
    	
@ManyToMany
	作用：用于映射多对多关系
	属性：
		cascade：配置级联操作。
		fetch：配置是否采用延迟加载。
    	targetEntity：配置目标的实体类。映射多对多的时候不用写。

@JoinTable
    作用：针对中间表的配置
    属性：
    	nam：配置中间表的名称
    	joinColumns：中间表的外键字段关联当前实体类所对应表的主键字段			  			
    	inverseJoinColumn：中间表的外键字段关联对方表的主键字段
```



## 多数据源例

多数据源 在jdbctemplate上面的注入两个DataSource的基础上

```
@Configuration
@EnableTransactionManagement
@EnableJpaRepositories(
        entityManagerFactoryRef="entityManagerFactoryPrimary",
        transactionManagerRef="transactionManagerPrimary",
        basePackages= { "com.didispace.domain.p" }) //设置Repository所在位置
public class PrimaryConfig {

    @Autowired @Qualifier("primaryDataSource")
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

```
@Configuration
@EnableTransactionManagement
@EnableJpaRepositories(
        entityManagerFactoryRef="entityManagerFactorySecondary",
        transactionManagerRef="transactionManagerSecondary",
        basePackages= { "com.didispace.domain.s" }) //设置Repository所在位置
public class SecondaryConfig {

    @Autowired @Qualifier("secondaryDataSource")
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

用`@Primary`区分主数据源。

```
@Entity
public class User {
    @Id
    @GeneratedValue
    private Long id;

    @Column(nullable = false)
    private String name;

    @Column(nullable = false)
    private Integer age;

    public User(){}

    public User(String name, Integer age) {
        this.name = name;
        this.age = age;
    }
    // 省略getter、setter
}
```

```
public interface UserRepository extends JpaRepository<User, Long> {

}
```



# mybatis

[**代码生成**](http://mybatis.org/generator/)

```
<dependency>
		<groupId>org.mybatis.spring.boot</groupId>
		<artifactId>mybatis-spring-boot-starter</artifactId>
		<version>2.1.1</version>
	</dependency>
```

```
spring.datasource.url=jdbc:mysql://localhost:3306/test
spring.datasource.username=root
spring.datasource.password=
spring.datasource.driver-class-name=com.mysql.cj.jdbc.Driver
```

```
@MapperScan("com.xxx.*.mapper")

mybatis.mapper-locations=classpath:mapper/*.xml

<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE mapper
        PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN"
        "http://mybatis.org/dtd/mybatis-3-mapper.dtd">
<mapper namespace="com.xxx.xxx.mapper.UserMapper">
</mapper>
```

```
mybatis.mapper-locations=classpath:mapper/*Mapper.xml
mybatis.configuraion.log-impl=org.apache.ibatis.logging.stuout.StdOutImpl

# 自动将数据库带下划线的表字段值映射到Java类的驼峰字段上
mybatis.configuration.map-underscore-to-camel-case=true
```

## typeHandler

无论是 MyBatis 在预处理语句（PreparedStatement）中设置一个参数时，还是从结果集中取出一个值时， 都会用类型处理器将获取的值以合适的方式转换成 Java 类型。下表描述了一些默认的类型处理器。

![img](https://img-blog.csdnimg.cn/20181219120716455.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQzNjUwNzcz,size_16,color_FFFFFF,t_70)

![img](https://img-blog.csdnimg.cn/20181219120846757.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQzNjUwNzcz,size_16,color_FFFFFF,t_70)

自定义类型转换器

你可以重写类型处理器或创建你自己的类型处理器来处理不支持的或非标准的类型。具体做法为：实现 org.apache.ibatis.type.TypeHandler 接口， 或继承一个很便利的类 org.apache.ibatis.type.BaseTypeHandler， 然后可以选择性地将它映射到一个JDBC类型。例如需求：一个Java中的Date数据类型，我想将之存到数据库的时候存成一个1970年至今的毫秒数，取出来时转换成java的Date，即java的Date与数据库的varchar毫秒值之间转换。

开发步骤：

①定义转换类继承类BaseTypeHandler<T>

②覆盖4个未实现的方法，其中setNonNullParameter为java程序设置数据到数据库的回调方法，getNullableResult为查询时 mysql的字符串类型转换成 java的Type类型的方法

③在MyBatis核心配置文件中进行注册

```
public class MyDateTypeHandler extends BaseTypeHandler<Date> {
    //把日期转换成毫秒(相当于System.currentTimeMillis()方法)
    //返回自1970年1月1日 00-00-00GMT 以来此Date 对象表示的毫秒数。
    public void setNonNullParameter(PreparedStatement preparedStatement, int i, Date date, JdbcType type) {
        preparedStatement.setString(i,date.getTime()+"");
    }
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

```
<!--注册类型自定义转换器-->
<typeHandlers>
    <typeHandler handler="xxx.xxx.xxx.MyDateTypeHandler"></typeHandler>
</typeHandlers>
或使用properties/yml文件指定typehandler所在的包
mybatis.type-handlers-package=xxx.xxx.xxx
```

若不注册类型转换器,则在mapper文件中指定

```
<resultMap id="UserResultMap" type="com.entity.User">
	<id column="id" jdbcType="VARCHAR" property="id" />
	<result column="name" jdbcType="VARCHAR" property="name" />
	<result column="date" jdbcType="VARCHAR" property="date"
	        javaType="Date" typeHandler="xxx.xxx.xxx.MyDateTypeHandler" />
</resultMap>
```

## pagehelper

```
<dependency>
    <groupId>com.github.pagehelper</groupId>
    <artifactId>pagehelper</artifactId>
    <version>3.7.5</version>
</dependency>
```

```
mapper.xml
<!-- 注意：分页拦截器的插件  配置在通用馆mapper之前 -->
<plugin interceptor="com.github.pagehelper.PageHelper">
    <!-- 指定方言 -->
    <property name="dialect" value="mysql"/>
</plugin>
```

```
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

spring

```
<bean id="sqlSessionFactory" class="org.mybatis.spring.SqlSessionFactoryBean">
  <!-- 注意其他配置 -->
  <property name="plugins">
    <array>
      <bean class="com.github.pagehelper.PageInterceptor">
        <property name="properties">
          <!--使用下面的方式配置参数，一行配置一个 -->
          <value>
            params=value1
          </value>
        </property>
      </bean>
    </array>
  </property>
</bean>
```

springboot

```
<dependency>
    <groupId>com.github.pagehelper</groupId>
    <artifactId>pagehelper-spring-boot-starter</artifactId>
    <version>1.3.1</version>
</dependency>
相关依赖
<dependency>
       <groupId>com.github.pagehelper</groupId>
       <artifactId>pagehelper-spring-boot-autoconfigure</artifactId>
       <version>1.3.1</version>
</dependency>

```

```
# 配置pagehelper参数 pagehelper.propertyName=propertyValue
#注意 pagehelper 配置，因为分页插件根据自己的扩展不同，支持的参数也不同，所以不能用固定的对象接收参数，所以这里使用的 Map<String,String>，因此参数名是什么这里就写什么，IDE 也不会有自动提示。
pagehelper:
    helperDialect: mysql
    reasonable: true
    supportMethodsArguments: true
    params: count=countSql
```

参数详见https://hub.fastgit.org/pagehelper/Mybatis-PageHelper/blob/master/wikis/zh/HowToUse.md

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

或使用配置类配置

```

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

```
//分页时，实际返回的结果list类型是Page<E>，如果想取出分页信息，需要强制转换为Page<E>
((Page) list).getTotal()
```

## 标签

**foreach**

```
<select id="selectByPrimaryKeys" resultMap="BaseResultMap">
    select
    <include refid="Base_Column_List"/>
    from tb_newbee_mall_goods_info
    where goods_id in
    <foreach item="id" collection="list" open="(" separator="," close=")">
        #{id}
    </foreach>
    order by field(goods_id,
    <foreach item="id" collection="list" separator=",">
        #{id}
    </foreach>
    );
</select>
```

**choose选择器**

        <if test="orderBy!=null and orderBy!=''">
            <choose>
                <when test="orderBy == 'new'">
                    <!-- 按照发布时间倒序排列 -->
                    order by goods_id desc
                </when>
                <when test="orderBy == 'price'">
                    <!-- 按照售价从小到大排列 -->
                    order by selling_price asc
                </when>
                <otherwise>
                    <!-- 默认按照库存数量从大到小排列 -->
                    order by stock_num desc
                </otherwise>
            </choose>
        </if>

**association一对一**

```
<resultMap id="orderMap" type="com.itheima.domain.Order">
        <id column="oid" property="id"></id>
        <result column="ordertime" property="ordertime"></result>
        <result column="total" property="total"></result>
                <!--手动指定字段与实体属性的映射关系
            column: 数据表的字段名称
            property：实体的属性名称
        -->
        <result column="uid" property="user.id"></result>
        <result column="username" property="user.username"></result>
        <result column="password" property="user.password"></result>
        <result column="birthday" property="user.birthday"></result>
    </resultMap>
```

2

```
<resultMap id="orderMap" type="com.itheima.domain.Order">
    <result property="id" column="id"></result>
    <result property="ordertime" column="ordertime"></result>
    <result property="total" column="total"></result>
            <!--
            property: 当前实体(order)中的属性名称(private User user)
            javaType: 当前实体(order)中的属性的类型(User)
        -->
    <association property="user" javaType="com.itheima.domain.User">
        <result column="uid" property="id"></result>
        <result column="username" property="username"></result>
        <result column="password" property="password"></result>
        <result column="birthday" property="birthday"></result>
    </association>
</resultMap>
```

**collection一对多**

```
<mapper namespace="com.itheima.mapper.UserMapper">
    <resultMap id="userMap" type="com.itheima.domain.User">
        <id column="id" property="id"></id>
        <result column="username" property="username"></result>
        <result column="password" property="password"></result>
        <result column="birthday" property="birthday"></result>
        <!--配置集合信息
            property:集合名称
            ofType：当前集合中的数据类型
        -->
        <collection property="orderList" ofType="com.itheima.domain.Order">
            <result column="oid" property="id"></result>
            <result column="ordertime" property="ordertime"></result>
            <result column="total" property="total"></result>
        </collection>
    </resultMap>
    <select id="findAll" resultMap="userMap">
        select *,o.id oid from user u left join orders o on u.id=o.uid
    </select>
</mapper>
```

通常collection标签 将结合resultMap标签的子标签id一起使用,  指定唯一确定一条记录的 id 列，MyBatis 根据 `<id>` 列值来完成多条记录的去重复功能. 不使用id标签结果一样,但是使用id标签会提高性能.

## 插入返回主键

1.自增主键情况下:

insert标签的两个属性

```
useGeneratedKeys = true　　//是否返回自增主键值
keyProperty = "xxx"　　//将值赋给哪个属性，这个属性是方法参数中的
```

```
<insert id = "insert" useGeneratedKeys = "true" keyProperty="userId" parameterType="com.chenzhou.mybatis.User">
    SQL语句
</insert>
```

```
User user = new User();  
user.setUserName("chenzhou");  
user.setPassword("xxxx");  
user.setComment("测试插入数据返回主键功能");  
System.out.println("插入前主键为："+user.getUserId());
userDao.insertAndGetId(user);//插入操作  
System.out.println("插入后主键为："+user.getUserId()); 
```

2.主键非自增的情况下

1)id已经在方法参数实体对象中传入

selectKey标签属性

```
resultType：返回类型
order：BEFORE在添加之前查询　AFTER在添加之后查询　　//这两个都是全大写
keyProperty：将取值赋值给方法参数，如果方法参数是实体类，一般赋值给实体类的字段
keyColumn：对应表的列名
```

```
<insert id = "insertEmp">
    <selectKey resultType = "integer" order = "AFTER" keyProperty = "eid" >
         select last_insert_id()    //查询最后一次添加的主键,mysql函数
    </selectKey>
    insert into dept(id,deptname) values(#{id},#{deptname})
</insert>
```

2)在插入前使用mysql的uuid或其他sql获取id方式作为主键id

```
<insert id = "insertDept">
    <selectKey resultType = "string" order = "BEFORE" keyProperty = "id">
        select uuid() as id
    </selectKey>
    insert into dept(id,name) values(#{id},#{name})
</insert>
```

3.批量插入 返回主键列表

传入list,执行完list会自动返回主键至list

```
<insert id="addUserByList" useGeneratedKeys="true" keyProperty="id">
    insert into user(name, phone, owner_star, in_time)
    values
    <foreach collection="list" item="item" index="index" separator=",">
        (#{item.name}, #{item.phone}, #{item.ownerStar}, NOW())
    </foreach>
</insert>
```



## 注解

### @Param

```
@Insert("INSERT INTO USER(NAME, AGE) VALUES(#{name}, #{age})")
int insert(@Param("name") String name, @Param("age") Integer age);
```

### 注解一对一

```
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

或者联合查询  和非注解方式类似

```
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

### 注解一对多

```
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

### 结果绑定

```
@Results({
    @Result(property = "name", column = "name"),
    @Result(property = "age", column = "age")
})
@Select("SELECT name, age FROM user")
List<User> findAll();
```

可以给results指定id属性然后使用resultMap复用

```
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



### 使用Map<String,String>和对象作参数

```
@Insert("INSERT INTO USER(NAME, AGE) VALUES(#{name,jdbcType=VARCHAR}, #{age,jdbcType=INTEGER})")
int insertByMap(Map<String, Object> map);

@Insert("INSERT INTO USER(NAME, AGE) VALUES(#{name}, #{age})")
int insertByUser(User user);
```

## 多数据源例

多数据源 在jdbctemplate上面的注入两个DataSource的基础上

```
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

```
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

### 动态数据源

通过继承AbstractRoutingDataSource实现数据源路由映射

见ruoyi





## MyBatis Dynamic SQL

https://mybatis.org/mybatis-dynamic-sql/docs

https://blog.csdn.net/zhongguoyu27/article/details/115833914

**官方例**

实体类

```
public class Person {
    private Integer id;
    private String firstName;
    private LastName lastName;
    private Date birthDate;
    private Boolean employed;
    private String occupation;
    private Integer addressId;
    
    // getters and setters omitted
}
```

dynamic

```
public final class PersonDynamicSqlSupport {
    public static final Person person = new Person();
    public static final SqlColumn<Integer> id = person.id;
    public static final SqlColumn<String> firstName = person.firstName;
    public static final SqlColumn<LastName> lastName = person.lastName;
    public static final SqlColumn<Date> birthDate = person.birthDate;
    public static final SqlColumn<Boolean> employed = person.employed;
    public static final SqlColumn<String> occupation = person.occupation;
    public static final SqlColumn<Integer> addressId = person.addressId;

    public static final class Person extends SqlTable {
        public final SqlColumn<Integer> id = column("id", JDBCType.INTEGER);
        public final SqlColumn<String> firstName = column("first_name", JDBCType.VARCHAR);
        public final SqlColumn<LastName> lastName = column("last_name", JDBCType.VARCHAR, "examples.simple.LastNameTypeHandler");
        public final SqlColumn<Date> birthDate = column("birth_date", JDBCType.DATE);
        public final SqlColumn<Boolean> employed = column("employed", JDBCType.VARCHAR, "examples.simple.YesNoTypeHandler");
        public final SqlColumn<String> occupation = column("occupation", JDBCType.VARCHAR);
        public final SqlColumn<Integer> addressId = column("address_id", JDBCType.INTEGER);

        public Person() {
            super("Person");
        }
    }
}
```

mapper

```
@Mapper
public interface PersonMapper {
    @SelectProvider(type = SqlProviderAdapter.class, method = "select")
    @Results(id = "PersonResult", value = {
            @Result(column = "A_ID", property = "id", jdbcType = JdbcType.INTEGER, id = true),
            @Result(column = "first_name", property = "firstName", jdbcType = JdbcType.VARCHAR),
            @Result(column = "last_name", property = "lastName", jdbcType = JdbcType.VARCHAR, typeHandler = LastNameTypeHandler.class),
            @Result(column = "birth_date", property = "birthDate", jdbcType = JdbcType.DATE),
            @Result(column = "employed", property = "employed", jdbcType = JdbcType.VARCHAR, typeHandler = YesNoTypeHandler.class),
            @Result(column = "occupation", property = "occupation", jdbcType = JdbcType.VARCHAR),
            @Result(column = "address_id", property = "addressId", jdbcType = JdbcType.INTEGER)
    })
    List<PersonRecord> selectMany(SelectStatementProvider selectStatement);

    @SelectProvider(type = SqlProviderAdapter.class, method = "select")
    @ResultMap("PersonResult")
    Optional<PersonRecord> selectOne(SelectStatementProvider selectStatement);
}
```

使用,这里静态导入了上面的SqlSupport类

```
import static xxx.xxx.PersonDynamicSqlSupport;

@Test
void testGeneralSelect() {
    try (SqlSession session = sqlSessionFactory.openSession()) {
        PersonMapper mapper = session.getMapper(PersonMapper.class);

        SelectStatementProvider selectStatement = select(id.as("A_ID"), firstName, lastName, birthDate, employed,
            occupation, addressId)
        .from(person)
        .where(id, isEqualTo(1))
        .or(occupation, isNull())
        .build()
        .render(RenderingStrategies.MYBATIS3);

        List<PersonRecord> rows = mapper.selectMany(selectStatement);
        assertThat(rows).hasSize(3);
    }
}
```





为了支持Mybatis Dynamic Sql，**mybatis官方扩展了原有的mybatis generator**，只要将配置文件中的targetRuntime指定为MyBatis3DynamicSql即可生成DynamicSql风格的Model和Mapper文件..

生成的这些Model和Mapper都是java类，没有xml的半格式化mapper映射文件。**这种方式生成的mapper天然支持limit的物理分页，以及批量插入，但是依然不支持包含枚举的自定义类型，也不支持自动按join关系生成Model和Mapper**:

```
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE generatorConfiguration PUBLIC "-//mybatis.org//DTD MyBatis Generator Configuration 1.0//EN" "http://mybatis.org/dtd/mybatis-generator-config_1_0.dtd">
<generatorConfiguration>
    <!--Mybatis Generator目前有5种运行模式，分别为：MyBatis3DynamicSql、MyBatis3Kotlin、MyBatis3、MyBatis3Simple、MyBatis3DynamicSqlV1。-->
    <context id="springboot-base" targetRuntime="MyBatis3DynamicSql">
        <commentGenerator>
            <!-- 是否去除自动生成的注释 true：是 ： false:否 -->
            <property name="suppressAllComments" value="true" />
        </commentGenerator>
        <jdbcConnection
                connectionURL="jdbc:mysql://127.0.0.1:3306/novel_plus?tinyInt1isBit=false&amp;useUnicode=true&amp;characterEncoding=utf-8&amp;serverTimezone=Asia/Shanghai&amp;nullCatalogMeansCurrent=true"
                driverClass="com.mysql.jdbc.Driver" password="test123456"
                userId="root" />

        <!-- 默认false，把JDBC DECIMAL 和 NUMERIC 类型解析为 Integer， 为 true时把JDBC DECIMAL
            和 NUMERIC 类型解析为java.math.BigDecimal -->
        <javaTypeResolver>
            <property name="forceBigDecimals" value="false" />
        </javaTypeResolver>

        <!-- targetProject:生成PO类的位置 -->
        <javaModelGenerator
                targetPackage="com.java2nb.novel.entity"
                targetProject="novel-common/src/main/java">
            <!-- enableSubPackages:是否让schema作为包的后缀 -->
            <property name="enableSubPackages" value="false" />
            <!-- 从数据库返回的值被清理前后的空格 -->
            <property name="trimStrings" value="true" />
        </javaModelGenerator>

        <!-- targetProject:mapper映射文件生成的位置 -->
        <sqlMapGenerator targetPackage="mybatis.mapping"
                         targetProject="novel-common/src/main/resources">
            <!-- enableSubPackages:是否让schema作为包的后缀 -->
            <property name="enableSubPackages" value="false" />
        </sqlMapGenerator>

        <!-- targetPackage：mapper接口生成的位置 -->
        <javaClientGenerator
                targetPackage="com.java2nb.novel.mapper"
                targetProject="novel-common/src/main/java" type="XMLMAPPER">
            <!-- enableSubPackages:是否让schema作为包的后缀 -->
            <property name="enableSubPackages" value="false" />
        </javaClientGenerator>

        <!--生成全部表tableName设为%-->
        <table tableName="book_index"/>

        <!-- 指定数据库表 -->
        <!--<table schema="jly" tableName="job_position" domainObjectName="JobPositionTest"/>-->
    </context>
</generatorConfiguration>
```

```
@SneakyThrows
    public static void main(String[] args) {
        //MBG 执行过程中的警告信息
        List<String> warnings = new ArrayList<>();
        //读取我们的 MBG 配置文件
        InputStream is = Generator.class.getResourceAsStream("/mybatis/generatorConfig.xml");
        ConfigurationParser cp = new ConfigurationParser(warnings);
        Configuration config = cp.parseConfiguration(is);
        is.close();
        //当生成的代码重复时，不要覆盖原代码
        DefaultShellCallback callback = new DefaultShellCallback(false);
        //创建 MBG
        MyBatisGenerator myBatisGenerator = new MyBatisGenerator(config, callback, warnings);
        //执行生成代码
        myBatisGenerator.generate(null);
        //输出警告信息
        for (String warning : warnings) {
            System.out.println(warning);
        }
    }
```

### 自定义一对一、一对多的多表关联关系

https://blog.csdn.net/zhongguoyu27/article/details/113730755

Myabtis DS Generator最为主要的功能就是对join关系进行了支持。

Mybatis DS Generator以一个补充的xml配置文件进行配置，Mybatis Generator自身也有一个xml格式的配置文件(即上文提到的mybatis-generator-config.xml)，所以总共有两个配置文件，这两个配置文件名称可自定义。

对于sql

```
select ... from exam_class_grade left join exam_student on exam_class_grade.id = exam_student.grade_id left join on exam_class_grade.regulator_id on exam_teacher.id
```

先在mybatis-generator-config.xml中增加Join方法插件：

```xml
<plugin type="com.catyee.generator.plugin.JoinMethodPlugin"/>
```

在ds-generator-config.xml中定义join关系：

```
<joinConfig targetPackage="com.catyee.mybatis.example.mapper" targetProject="src/main/resources">
    <joinEntry leftTable="exam_class_grade">
        <joinTarget rightTable="exam_student" property="students" leftTableColumn="id" rightTableColumn="grade_id" joinType="MORE"/>
        <joinTarget rightTable="exam_teacher" property="regulator" leftTableColumn="regulator_id" rightTableColumn="id" joinType="ONE"/>
    </joinEntry>
</joinConfig>
```

注意：暂不支持多级关联关系，比如A和B是一对多的关联关系，B和C又是一对多的关联关系，这种场景暂不支持。

### 自定义类型，并支持泛型

Mybatis Generator可以在table标签下通过columnOverride的子标签对特定列进行设置，从而配置特定的java类型以及类型handler，但是无法处理泛型，Myabtis DS Generator对泛型进行了支持，使用方式如下：
先在mybatis-generator-config.xml中使用javaTypeResolver：

```
<javaTypeResolver type="com.catyee.generator.resolver.JavaTypeCustomResolver">
    <property name="forceBigDecimals" value="false"/>
    <property name="useJSR310Types" value="true"/>
</javaTypeResolver>
```

然后在ds-generator-config.xml的customTypeConfig标签下增加自定义类型的配置：

```
<!--枚举类型-->
<customType columnName="grade_type" javaType="com.catyee.mybatis.example.custom.entity.GradeType"/>

<!--List<String>类型-->
<customType columnName="tech_courses" javaType="java.util.List" typeHandler="com.catyee.mybatis.example.custom.handler.StringListHandler">
    <genericType javaType="String"/>
</customType>

<!--Map<String, Integer>类型-->
<customType columnName="score" javaType="java.util.Map" typeHandler="com.catyee.mybatis.example.custom.handler.ScoreMapHandler">
    <genericType javaType="String"/>
    <genericType javaType="Integer"/>
</customType>
```

注意：暂不支持多层级的泛型，比如Map< Integer, List< String>>

### 其他插件

**lombok支持**

lombok可以有效的减少get、set等模板代码的编写，提升代码简洁性的利器，相关lombok的使用方式及原理可以自行查看资料。Myabtis DS Generator在Model类上集成了四个最为常用的lombok注解，这四个注解已经满足绝大多数的使用场景。

```
@Data // 提供get、set、hashcode、equals、toString方法
@Builder // 提供Builder静态类，方便构建对象
@AllArgsConstructor // 提供全参构造器
@NoArgsConstructor // 提供无参构造器
```

在generator中使用lombok的方式：
在mybatis-generator-config.xml中增加lombok插件，如下：

```
<plugin type="com.catyee.generator.plugin.AnnotationPlugin"/>
```

**去除多余注释**

Mybatis Gernerator会生成很多的多余的注释，比如这个类是怎么生成的，生成的时间，针对的是哪张表或者哪个字段，其实这些注释没有特别大的用处，反而影响代码的简洁度。Mybatis DS Generator中实现了无注释插件，可以生成没有注释的model和mapper，使用方式：
在mybatis-generator-config.xml中使用无注释插件：

```
<commentGenerator type="com.catyee.generator.comment.NonCommentGenerator"/>
```



# tk.mybatis

```
<!--mybatis-->
<dependency>
   <groupId>org.mybatis.spring.boot</groupId>
   <artifactId>mybatis-spring-boot-starter</artifactId>
   <version>1.3.1</version>
</dependency>
<!--通用mapper-->
<dependency>
   <groupId>tk.mybatis</groupId>
   <artifactId>mapper-spring-boot-starter</artifactId>
   <version>1.1.5</version>
</dependency>
```









## tk+pagehelper+Generator

```
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

```
import tk.mybatis.mapper.common.Mapper;
import tk.mybatis.mapper.common.MySqlMapper;
public interface MyMapper<T> extends Mapper<T>, MySqlMapper<T> {
   
}

public interface UserMapper extends MyMapper<User> {
}
```

配置文件

```
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
  
 #pagehelper
pagehelper: 
  helperDialect: oracle
  reasonable: true
  supportMethodsArguments: true
  params: count=countSql
```

使用MyBatis Geneator来自动生成实体类，Mapper接口和Mapper xml代码

*src/main/resources/mybatis-generator.xml*为生成器的配置, 在路径src/main/resources/下新建mybatis-generator.xml：，

```
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
            userId="scott"
            password="6742530">
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

运行命令*mybatis-generator:generate*生成

可以看到生成的实体类

```
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

和继承了MyMapper<User>的UserMapper ,

只包含resultMap的UserMapper.xml.

**Mapper自带的CRUD方法，所以UserMapper接口中无需定义任何方法。**

注意:要让Spring Boot扫描到Mapper接口，需要在Spring Boot入口类中加入`@MapperScan("com.springboot.mapper")`注解。

### 自定义通用service

我们可以自定义通用service

```
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

```
public interface UserService extends IService<User>{  //这里的extendsg
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

### 使用oracle序列值实现主键

其中 seqName为我们自定义的序列名称,如果在一个表中为了不保证重复获取需要每次传入的序列名称不变

```
public interface SeqenceMapper {
    @Select("select ${seqName}.nextval from dual")
    Long getSequence(@Param("seqName") String seqName);
}
```

在IService和BaseService中加入方法:

```
@Service
public interface IService<T> {
    Long getSequence(@Param("seqName") String seqName);
    // ...
}

public abstract class BaseService<T> implements IService<T> {
	// ...
    @Autowired
    protected SeqenceMapper seqenceMapper;
    @Override
    public Long getSequence(@Param("seqName") String seqName){
        return seqenceMapper.getSequence(seqName);
    }
    // ...
}
```

插入数据的方式:

```
@Test
    public void test() throws Exception {
        User user = new User();
        user.setId(userService.getSequence("seq_user"));
        user.setUsername("scott");
        user.setPasswd("ac089b11709f9b9e9980e7c497268dfa");
        user.setCreateTime(new Date());
        user.setStatus("0");
        this.userService.save(user);
    }
```

### Example查询

```
Example example = new Example(User.class);
example.createCriteria().andCondition("username like '%i%'");
example.setOrderByClause("id desc");
List<User> userList = this.userService.selectByExample(example);
for (User u : userList) {
    System.out.println(u.getUsername());
}
```

## pagehelper

```
PageHelper.startPage(2, 2);
List<User> list = userService.selectAll();
PageInfo<User> pageInfo = new PageInfo<User>(list);
List<User> result = pageInfo.getList();
for (User u : result) {
    System.out.println(u.getUsername());
}
```

查看打印的sql发现自动添加了分页

# transaction

在Spring Boot中，当我们使用了spring-boot-starter-jdbc或spring-boot-starter-data-jpa依赖的时候，框架会自动默认分别注入DataSourceTransactionManager或JpaTransactionManager。所以我们不需要任何额外配置就可以用@Transactional注解进行事务的使用。(如果是老版,可能需要加*`@EnableTransactionManagement`*)

回滚日志:

```
o.s.t.c.transaction.TransactionContext   : Rolled back transaction for ...
```

> 通常我们单元测试为了保证每个测试之间的数据独立，会使用`@Rollback`注解让每个单元测试都能在结束时回滚。而真正在开发业务逻辑时，我们通常在service层接口中使用`@Transactional`来对各个业务逻辑进行事务管理的配置.

**`@Transactional` 的常用配置参数总结（只列出了 5 个平时比较常用的）：**

| 属性名      | 说明                                                         |
| :---------- | :----------------------------------------------------------- |
| propagation | 事务的传播行为，默认值为 REQUIRED，可选的值在上面介绍过      |
| isolation   | 事务的隔离级别，默认值采用 DEFAULT，可选的值在上面介绍过     |
| timeout     | 事务的超时时间，默认值为-1（不会超时）。如果超过该时间限制但事务还没有完成，则自动回滚事务。 |
| readOnly    | 指定事务是否为只读事务，默认值为 false。                     |
| rollbackFor | 用于指定能够触发事务回滚的异常类型，并且可以指定多个异常类型。 |

在多数据源时 使用value指定事务管理器

```
@Transactional(value="transactionManagerPrimary")
```

isolation指定隔离级别

- `DEFAULT`：这是默认值，表示使用底层数据库的默认隔离级别。对大部分数据库而言，通常这值就是：`READ_COMMITTED`。
- `READ_UNCOMMITTED`：该隔离级别表示一个事务可以读取另一个事务修改但还没有提交的数据。该级别不能防止脏读和不可重复读，因此很少使用该隔离级别。
- `READ_COMMITTED`：该隔离级别表示一个事务只能读取另一个事务已经提交的数据。该级别可以防止脏读，这也是大多数情况下的推荐值。
- `REPEATABLE_READ`：该隔离级别表示一个事务在整个过程中可以多次重复执行某个查询，并且每次返回的记录都相同。即使在多次查询之间有新增的数据满足该查询，这些新增的记录也会被忽略。该级别可以防止脏读和不可重复读。
- `SERIALIZABLE`：所有的事务依次逐个执行，这样事务之间就完全不可能产生干扰，也就是说，该级别可以防止脏读、不可重复读以及幻读。但是这将严重影响程序的性能。通常情况下也不会用到该级别。

```
@Transactional(isolation = Isolation.DEFAULT)
```

**传播行为**

所谓事务的传播行为是指，如果在开始当前事务之前，一个事务上下文已经存在，此时有若干选项可以指定一个事务性方法的执行行为。即何时要创建一个事务，或者何时使用已有的事务.

- `REQUIRED`：默认 . 如果当前存在事务，则加入该事务；如果当前没有事务，则创建一个新的事务。
- `SUPPORTS`：如果当前存在事务，则加入该事务；如果当前没有事务，则以非事务的方式继续运行。
- `MANDATORY`：如果当前存在事务，则加入该事务；如果当前没有事务，则抛出异常。
- `REQUIRES_NEW`：创建一个新的事务，如果当前存在事务，则把当前事务挂起。
- `NOT_SUPPORTED`：以非事务方式运行，如果当前存在事务，则把当前事务挂起。
- `NEVER`：以非事务方式运行，如果当前存在事务，则抛出异常。
- `NESTED`：如果当前存在事务，则创建一个事务作为当前事务的嵌套事务来运行(外部主事务回滚的话，子事务也会回滚，而内部子事务可以单独回滚而不影响外部主事务和其他子事务。)；如果当前没有事务，则该取值等价于`REQUIRED`。

指定方法：通过使用`propagation`属性设置，例如：

```
@Transactional(propagation = Propagation.REQUIRED)
```

**异常回滚**

在 `@Transactional` 注解中如果不配置`rollbackFor`属性,那么事务只会在遇到`RuntimeException`的时候才会回滚，加上 `rollbackFor=Exception.class`,可以让事务在遇到非运行时异常时也回滚。 

## 事务无效原因

1.mysql MyISAM

```
spring.jpa.database-platform=org.hibernate.dialect.MySQL5InnoDBDialect
```

这里的`spring.jpa.database-platform`配置主要用来设置hibernate使用的方言。这里特地采用了`MySQL5InnoDBDialect`，主要为了保障在使用Spring Data JPA时候，Hibernate自动创建表的时候使用InnoDB存储引擎，不然就会以默认存储引擎MyISAM来建表，而MyISAM存储引擎是没有事务的。

2.`@Transactional`注解修饰的函数中`catch`了异常，并没有往方法外抛。

3.`@Transactional`注解修饰的函数不是`public`类型

4.异常类型错误，如果有通过rollbackFor指定回滚的异常类型，那么抛出的异常与指定的是否一致。

5.数据源没有配置事务管理器

6.在一个类中调用自己的方法。如

```
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

**这种情况下事务失效的原因为：Spring事务控制使用AOP代理实现，通过对目标对象的代理来增强目标方法。而上面例子直接通过this调用本类的方法的时候，this的指向并非代理类，而是该类本身。**

**因为spring采用动态代理机制来实现事务控制，而(jdk)动态代理最终都是要调用原始对象的，而原始对象在去调用方法时，是不会再触发代理了**

## 编程式事务管理

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



Spring 框架中，事务管理相关最重要的 3 个接口如下：

- **`PlatformTransactionManager`**： （平台）事务管理器，Spring 事务策略的核心。
- **`TransactionDefinition`**： 事务定义信息(事务隔离级别、传播行为、超时、只读、回滚规则)。
- **`TransactionStatus`**： 事务运行状态。

我们可以把 **`PlatformTransactionManager`** 接口可以被看作是事务上层的管理者，而 **`TransactionDefinition`** 和 **`TransactionStatus`** 这两个接口可以看作是事务的描述。

**`PlatformTransactionManager`** 会根据 **`TransactionDefinition`** 的定义比如事务超时时间、隔离级别、传播行为等来进行事务管理 ，而 **`TransactionStatus`** 接口则提供了一些方法来获取事务相应的状态比如是否新事务、是否可以回滚等等。



**PlatformTransactionManager:事务管理接口**

**Spring 并不直接管理事务，而是提供了多种事务管理器** 。Spring 事务管理器的接口是： **`PlatformTransactionManager`** 。

通过这个接口，Spring 为各个平台如 JDBC(`DataSourceTransactionManager`)、Hibernate(`HibernateTransactionManager`)、JPA(`JpaTransactionManager`)等都提供了对应的事务管理器，但是具体的实现就是各个平台自己的事情了。

```
public interface PlatformTransactionManager {
    //获得事务
    TransactionStatus getTransaction(@Nullable TransactionDefinition var1) throws TransactionException;
    //提交事务
    void commit(TransactionStatus var1) throws TransactionException;
    //回滚事务
    void rollback(TransactionStatus var1) throws TransactionException;
}
```



 **TransactionDefinition:事务属性**

事务管理器接口 **`PlatformTransactionManager`** 通过 **`getTransaction(TransactionDefinition definition)`** 方法来得到一个事务，这个方法里面的参数是 **`TransactionDefinition`** 类 ，这个类就定义了一些基本的事务属性。

那么什么是 **事务属性** 呢？

事务属性可以理解成事务的一些基本配置，描述了事务策略如何应用到方法上。包括隔离级别,传播行为,回滚规则,是否只读,事务超时.

```
public interface TransactionDefinition {
    int PROPAGATION_REQUIRED = 0;
    int PROPAGATION_SUPPORTS = 1;
    int PROPAGATION_MANDATORY = 2;
    int PROPAGATION_REQUIRES_NEW = 3;
    int PROPAGATION_NOT_SUPPORTED = 4;
    int PROPAGATION_NEVER = 5;
    int PROPAGATION_NESTED = 6;
    int ISOLATION_DEFAULT = -1;
    int ISOLATION_READ_UNCOMMITTED = 1;
    int ISOLATION_READ_COMMITTED = 2;
    int ISOLATION_REPEATABLE_READ = 4;
    int ISOLATION_SERIALIZABLE = 8;
    int TIMEOUT_DEFAULT = -1;
    // 返回事务的传播行为，默认值为 REQUIRED。
    int getPropagationBehavior();
    //返回事务的隔离级别，默认值是 DEFAULT
    int getIsolationLevel();
    // 返回事务的超时时间，默认值为-1。如果超过该时间限制但事务还没有完成，则自动回滚事务。
    int getTimeout();
    // 返回是否为只读事务，默认值为 false
    boolean isReadOnly();

    @Nullable
    String getName();
}
```



**TransactionStatus:事务状态**

`TransactionStatus`接口用来记录事务的状态 该接口定义了一组方法,用来获取或判断事务的相应状态信息。

`PlatformTransactionManager.getTransaction(…)`方法返回一个 `TransactionStatus` 对象。

```
public interface TransactionStatus{
    boolean isNewTransaction(); // 是否是新的事务
    boolean hasSavepoint(); // 是否有恢复点
    void setRollbackOnly();  // 设置为只回滚
    boolean isRollbackOnly(); // 是否为只回滚
    boolean isCompleted; // 是否已完成
}
```



# cache

## springboot使用

```
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-cache</artifactId>
</dependency>
```

在Spring Boot主类中增加`@EnableCaching`注解开启缓存功能

以jpa为例 ,在数据访问接口中

```
@CacheConfig(cacheNames = "users")
public interface UserRepository extends JpaRepository<User, Long> {
    @Cacheable
    User findByName(String name);
}
```

测试中调用两次查询 查看打印执行sql(sql可配置spring.jpa.show-sql=true    1.x为spring.jpa.properties.hibernate.show_sql=true)

为了可以更好的观察，缓存的存储，我们可以在单元测试中注入cacheManager。

```
@Autowired
private CacheManager cacheManager;
```

使用debug模式运行单元测试，观察cacheManager中的缓存集users以及其中的User对象的缓存加深理解。

### 缓存配置

Spring Boot根据下面的顺序去侦测缓存提供者：

- Generic
- JCache (JSR-107)
- EhCache 2.x
- Hazelcast
- Infinispan
- Redis
- Guava
- Simple

除了按顺序侦测外，我们也可以通过配置属性`spring.cache.type`来强制指定。我们可以通过debug调试查看cacheManager对象的实例来判断当前使用了什么缓存。

当我们不指定具体其他第三方实现的时候，Spring Boot的Cache模块会使用`ConcurrentHashMap`来存储。而实际生产使用的时候，因为我们可能需要更多其他特性，往往就会采用其他缓存框架

注解说明:

- `@CacheConfig`：主要用于配置该类中会用到的一些共用的缓存配置。在这里`@CacheConfig(cacheNames = "users")`：配置了该数据访问对象中返回的内容将存储于名为users的缓存对象中，我们也可以不使用该注解，直接通过`@Cacheable`自己配置缓存集的名字来定义。
- `@Cacheable`：配置了findByName函数的返回值将被加入缓存。同时在查询时，会先从缓存中获取，若不存在才再发起对数据库的访问。该注解主要有下面几个参数：
  - `value`、`cacheNames`：两个等同的参数（`cacheNames`为Spring 4新增，作为`value`的别名），用于指定缓存存储的集合名。由于Spring 4中新增了`@CacheConfig`，因此在Spring 3中原本必须有的`value`属性，也成为非必需项了
  - `key`：缓存对象存储在Map集合中的key值，非必需，缺省按照函数的所有参数组合作为key值，若自己配置需使用SpEL表达式，比如：`@Cacheable(key = "#p0")`：使用函数第一个参数作为缓存的key值，更多关于SpEL表达式的详细内容可参考[官方文档](http://docs.spring.io/spring/docs/current/spring-framework-reference/html/cache.html#cache-spel-context)
  - `condition`：缓存对象的条件，非必需，也需使用SpEL表达式，只有满足表达式条件的内容才会被缓存，比如：`@Cacheable(key = "#p0", condition = "#p0.length() < 3")`，表示只有当第一个参数的长度小于3的时候才会被缓存，若做此配置上面的AAA用户就不会被缓存，读者可自行实验尝试。
  - `unless`：另外一个缓存条件参数，非必需，需使用SpEL表达式。它不同于`condition`参数的地方在于它的判断时机，该条件是在函数被调用之后才做判断的，所以它可以通过对result进行判断。
  - `keyGenerator`：用于指定key生成器，非必需。若需要指定一个自定义的key生成器，我们需要去实现`org.springframework.cache.interceptor.KeyGenerator`接口，并使用该参数来指定。需要注意的是：**该参数与`key`是互斥的**
  - `cacheManager`：用于指定使用哪个缓存管理器，非必需。只有当有多个时才需要使用
  - `cacheResolver`：用于指定使用那个缓存解析器，非必需。需通过`org.springframework.cache.interceptor.CacheResolver`接口来实现自己的缓存解析器，并用该参数指定。

- `@CachePut`：配置于函数上，能够根据参数定义条件来进行缓存，它与`@Cacheable`不同的是，它每次都会真是调用函数，所以主要用于数据新增和修改操作上。它的参数与`@Cacheable`类似，具体功能可参考上面对`@Cacheable`参数的解析
- `@CacheEvict`：配置于函数上，通常用在删除方法上，用来从缓存中移除相应数据。除了同`@Cacheable`一样的参数之外，它还有下面两个参数：
  - `allEntries`：非必需，默认为false。当为true时，会移除所有数据
  - `beforeInvocation`：非必需，默认为false，会在调用方法之后移除数据。当为true时，会在调用方法之前移除数据。

### ehcache

在Spring Boot中开启EhCache非常简单，只需要在工程中加入`ehcache.xml`配置文件并在pom.xml中增加ehcache依赖，框架只要发现该文件，就会创建EhCache的缓存管理器。

- 在`src/main/resources`目录下创建：`ehcache.xml`

```
<ehcache xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:noNamespaceSchemaLocation="ehcache.xsd">
    <cache name="users"
           maxEntriesLocalHeap="200"
           timeToLiveSeconds="600">
    </cache>
</ehcache>
```

复杂点的为:

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
	<!-- 磁盘缓存位置 -->
	<diskStore path="user.dir/cachedata" />

	<!-- 默认缓存 -->
	<defaultCache maxElementsInMemory="10000" eternal="false"
		timeToIdleSeconds="120" timeToLiveSeconds="120"
		maxElementsOnDisk="10000000" diskExpiryThreadIntervalSeconds="120"
		memoryStoreEvictionPolicy="LRU">

		<persistence strategy="localTempSwap" />
	</defaultCache>

	<cache name="user" eternal="false" maxElementsInMemory="10000"
		   overflowToDisk="false" diskPersistent="false" timeToIdleSeconds="0"
		   timeToLiveSeconds="0" memoryStoreEvictionPolicy="LFU"/>
</ehcache>
```

引入

```
<dependency>
    <groupId>net.sf.ehcache</groupId>
    <artifactId>ehcache</artifactId>
</dependency>
在Spring Boot的parent管理下，不需要指定具体版本，会自动采用Spring Boot中指定的版本号。
```

完成上面的配置之后，再通过debug模式运行单元测试，观察此时CacheManager已经是EhCacheManager实例，说明EhCache开启成功了。

对于EhCache的配置文件也可以通过`application.properties`文件中使用`spring.cache.ehcache.config`属性来指定，比如：

```
spring.cache.ehcache.config=classpath:config/another-config.xml
```

运行两次同样的查询测试结果:结果相同,查询sql只执行一次

运行两次同样的查询在中间加一次更新(age)测试:结果不相同,sql执行了两次,缓存及时进行了更新.(区分redis)

#### ehcache集群

尝试手工组建集群的方式，不同实例在网络相关配置上会产生不同的配置信息，所以我们建立不同的配置文件给不同的实例使用.

实例1，使用`ehcache-1.xml`

```
<ehcache xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:noNamespaceSchemaLocation="ehcache.xsd">

    <cache name="users"
           maxEntriesLocalHeap="200"
           timeToLiveSeconds="600">
        <cacheEventListenerFactory
                class="net.sf.ehcache.distribution.RMICacheReplicatorFactory"
                properties="replicateAsynchronously=true,
            replicatePuts=true,
            replicateUpdates=true,
            replicateUpdatesViaCopy=false,
            replicateRemovals=true "/>
    </cache>

    <cacheManagerPeerProviderFactory
            class="net.sf.ehcache.distribution.RMICacheManagerPeerProviderFactory"
            properties="hostName=10.10.0.100,
                        port=40001,
                        socketTimeoutMillis=2000,
                        peerDiscovery=manual,
                        rmiUrls=//10.10.0.101:40001/users" />

</ehcache>
```

```
<ehcache xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:noNamespaceSchemaLocation="ehcache.xsd">

    <cache name="users"
           maxEntriesLocalHeap="200"
           timeToLiveSeconds="600">
        <cacheEventListenerFactory
                class="net.sf.ehcache.distribution.RMICacheReplicatorFactory"
                properties="replicateAsynchronously=true,
            replicatePuts=true,
            replicateUpdates=true,
            replicateUpdatesViaCopy=false,
            replicateRemovals=true "/>
    </cache>

    <cacheManagerPeerProviderFactory
            class="net.sf.ehcache.distribution.RMICacheManagerPeerProviderFactory"
            properties="hostName=10.10.0.101,
                        port=40001,
                        socketTimeoutMillis=2000,
                        peerDiscovery=manual,
                        rmiUrls=//10.10.0.100:40001/users" />

</ehcache>
```

配置说明：

- cache标签中定义名为users的缓存，这里我们增加了一个子标签定cacheEventListenerFactory

  ，这个标签主要用来定义缓存事件监听的处理策略，它有以下这些参数用来设置缓存的同步策略：

  - replicatePuts：当一个新元素增加到缓存中的时候是否要复制到其他的peers。默认是true。
  - replicateUpdates：当一个已经在缓存中存在的元素被覆盖时是否要进行复制。默认是true。
  - replicateRemovals：当元素移除的时候是否进行复制。默认是true。
  - replicateAsynchronously：复制方式是异步的指定为true时，还是同步的，指定为false时。默认是true。
  - replicatePutsViaCopy：当一个新增元素被拷贝到其他的cache中时是否进行复制指定为true时为复制，默认是true。
  - replicateUpdatesViaCopy：当一个元素被拷贝到其他的cache中时是否进行复制指定为true时为复制，默认是true。

- 新增了一个cacheManagerPeerProviderFactory标签的配置，用来指定组建的集群信息和要同步的缓存信息，其中：

  - hostName：是当前实例的主机名
  - port：当前实例用来同步缓存的端口号
  - socketTimeoutMillis：同步缓存的Socket超时时间
  - peerDiscovery：集群节点的发现模式，有手工与自动两种，这里采用了手工指定的方式
  - rmiUrls：当peerDiscovery设置为manual的时候，用来指定需要同步的缓存节点，如果存在多个用`|`连接

  **最后:打包部署与启动**:打包没啥大问题，主要缓存配置内容存在一定差异，所以在指定节点的模式下，需要单独拿出来，然后使用启动参数来控制读取不同的配置文件。比如这样：

  ```
  -Dspring.cache.ehcache.config=classpath:ehcache-1.xml
  -Dspring.cache.ehcache.config=classpath:ehcache-2.xml
  ```

  **验证逻辑:**

  1. 启动通过第三步说的命令参数，启动两个实例
  2. 调用实例1的`/create`接口，创建一条数据
  3. 调用实例1的`/find`接口，实例1缓存User，同时同步缓存信息给实例2，在实例1中会存在SQL查询语句
  4. 调用实例2的`/find`接口，由于缓存集群同步了User的信息，所以在实例2中的这次查询也不会出现SQL语句

  问题:数据更新之后怎么办?

  1. `save`操作增加`@CachePut`注解，让更新操作完成之后将结果再put到缓存中
  2. 保证缓存事件监听的replicateUpdates=true，这样数据在更新之后可以保证复制到其他节点.

  这样就可以防止缓存的脏数据了，但是这种方法还并不是很好，因为缓存集群的同步依然需要时间，会存在短暂的不一致。同时进程内的缓存要在每个实例上都占用，如果大量存储的话始终不那么经济。所以，很多时候进程内缓存不会作为主要的缓存手段。

### redis集中式缓存

虽然EhCache已经能够适用很多应用场景，但是由于EhCache是进程内的缓存框架，在集群模式下时，各应用服务器之间的缓存都是独立的，因此在不同服务器的进程间会存在缓存不一致的情况。即使EhCache提供了集群环境下的缓存同步策略，但是同步依然需要一定的时间，短暂的缓存不一致依然存在。

在一些要求高一致性（任何数据变化都能及时的被查询到）的系统和应用中，就不能再使用EhCache来解决了，这个时候使用集中式缓存是个不错的选择，因此本文将介绍如何在Spring Boot的缓存支持中使用Redis进行数据缓存。	

```
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis</artifactId>
</dependency>
<dependency>
    <groupId>org.apache.commons</groupId>
    <artifactId>commons-pool2</artifactId>
</dependency>
//1.x
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-redis</artifactId>
</dependency>
```

```
spring.redis.host=localhost
spring.redis.port=6379
spring.redis.lettuce.pool.max-idle=8
spring.redis.lettuce.pool.max-active=8
spring.redis.lettuce.pool.max-wait=-1ms
spring.redis.lettuce.pool.min-idle=0
spring.redis.lettuce.shutdown-timeout=100ms
#1.x 在1.x版本中采用jedis作为连接池，而在2.x版本中采用了lettuce作为连接池
spring.redis.host=localhost
spring.redis.port=6379
spring.redis.pool.max-idle=8
spring.redis.pool.min-idle=0
spring.redis.pool.max-active=8
spring.redis.pool.max-wait=-1
#以上配置均为默认值，实际上生产需进一步根据部署情况与业务要求做适当修改
```

，Spring Boot会在侦测到存在Redis的依赖并且Redis的配置是可用的情况下，使用`RedisCacheManager`初始化`CacheManager`。

测试运行两次同样的查询在中间加一次更新(age)测试:结果相同 (问题)

**为什么同样的逻辑在EhCache中没有问题，但是到Redis中会出现这个问题呢？**

在EhCache缓存时没有问题，主要是由于EhCache是进程内的缓存框架，第一次通过select查询出的结果被加入到EhCache缓存中，第二次查询从EhCache取出的对象与第一次查询对象实际上是同一个对象（可以在工程中，观察u1==u2来看看是否是同一个对象），因此我们在更新age的时候，实际已经更新了EhCache中的缓存对象。

而Redis的缓存独立存在于我们的Spring应用之外，我们对数据库中数据做了更新操作之后，没有通知Redis去更新相应的内容，因此我们取到了缓存中未修改的数据，导致了数据库与缓存中数据的不一致。

**因此我们在使用缓存的时候，要注意缓存的生命周期，利用好cache注解来做好缓存的更新、删除**

针对上面的问题，我们只需要在更新age的时候，通过`@CachePut`来让数据更新操作同步到缓存中，就像下面这样：

```
@CacheConfig(cacheNames = "users")
public interface UserRepository extends JpaRepository<User, Long> {

    @Cacheable(key = "#p0")
    User findByName(String name);
	//以name属性作为条件进行缓存,意味着如果非name属性更新的化缓存也进行更新
    @CachePut(key = "#p0.name")
    User save(User user);

}
```



**执行插入语句时候,也会将对象缓存到cache,因此,在插入后马上进行查询不执行sql并访问cache**

#### 发布订阅功能

在发布订阅模式中有个重要的角色，一个是发布者Publisher，另一个订阅者Subscriber。本质上来说，发布订阅模式就是一种生产者消费者模式，Publisher负责生产消息，而Subscriber则负责消费它所订阅的消息。这种模式被广泛的应用于软硬件的系统设计中。比如：配置中心的一个配置修改之后，就是通过发布订阅的方式传递给订阅这个配置的订阅者来实现自动刷新的。

问题:发布订阅模式中的两个概念与观察者模式中的两个概念似乎干的是一样的事情？所以：Publisher就是观察者模式中的Subject？Subscriber就是观察者模式中的Observer？

区别:

![](https://blog.didispace.com/images/pasted-515.png)

![](https://blog.didispace.com/images/pasted-516.png)

**发布订阅模式在两个角色中间是一个中间角色来过渡的，发布者并不直接与订阅者产生交互**。

回想一下生产者消费者模式，这个中间过渡区域对应的就是是缓冲区。因为这个缓冲区的存在，发布者与订阅者的工作就可以实现更大程度的解耦。发布者不会因为订阅者处理速度慢，而影响自己的发布任务，它只需要快速生产即可。而订阅者也不用太担心一时来不及处理，因为有缓冲区在，可以一点点排队来完成（也就是我们常说的“削峰填谷”效果）。

而我们所熟知的RabbitMQ、Kafka、RocketMQ这些中间件的本质其实就是实现发布订阅模式中的这个中间缓冲区。而Redis也提供了简单的发布订阅实现，当我们有一些简单需求的时候，也是可以一用的！

```
@SpringBootApplication
public class Chapter55Application {
    private static String CHANNEL = "didispace";
    public static void main(String[] args) {
        SpringApplication.run(Chapter55Application.class, args);
    }

    @RestController
    static class RedisController {
        private RedisTemplate<String, String> redisTemplate;
        
        public RedisController(RedisTemplate<String, String> redisTemplate) {
            this.redisTemplate = redisTemplate;
        }
        
        @GetMapping("/publish")
        public void publish(@RequestParam String message) {
            // 发送消息
            redisTemplate.convertAndSend(CHANNEL, message);
        }
    }
}
```

```
@Slf4j
@Service
static class MessageSubscriber {

    public MessageSubscriber(RedisTemplate redisTemplate) {
        RedisConnection redisConnection = redisTemplate.getConnectionFactory().getConnection();
        redisConnection.subscribe(new MessageListener() {
            @Override
            public void onMessage(Message message, byte[] bytes) {
                // 收到消息的处理逻辑
                log.info("Receive message : " + message);
            }
        }, CHANNEL.getBytes(StandardCharsets.UTF_8));
    }
}
```

验证结果

1. 启动应用Spring Boot主类
2. 通过curl或其他工具调用接口`curl localhost:8080/publish?message=hello`
3. 观察控制台，可以看到打印了收到的message参数

#### redis cache自定义配置

```
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

	// 缓存管理器
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
		setSerializer(template);// 设置序列化工具
		template.afterPropertiesSet();
		return template;
	}

	private void setSerializer(StringRedisTemplate template) {
		@SuppressWarnings({ "rawtypes", "unchecked" })
		Jackson2JsonRedisSerializer jackson2JsonRedisSerializer = new Jackson2JsonRedisSerializer(Object.class);
		ObjectMapper om = new ObjectMapper();
		om.setVisibility(PropertyAccessor.ALL, JsonAutoDetect.Visibility.ANY);
		om.enableDefaultTyping(ObjectMapper.DefaultTyping.NON_FINAL);
		jackson2JsonRedisSerializer.setObjectMapper(om);
		template.setValueSerializer(jackson2JsonRedisSerializer);
	}
}
```

```
@CacheConfig(cacheNames = "student")
public interface StudentService {
   @CachePut(key = "#p0.sno")
   Student update(Student student);

   @CacheEvict(key = "#p0", allEntries = true)
   void deleteStudentBySno(String sno);
   
   @Cacheable(key = "#p0")
   Student queryStudentBySno(String sno);
}
```

##### SHA256生成key

org.apache.commons

```
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
            return DigestUtils.sha256Hex(jsonString);
        };
    }
```

##### cache操作异常处理

```
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



### Redisson分布式锁

解决集群缓存环境下线程安全问题

```
<dependency>
            <groupId>org.redisson</groupId>
            <artifactId>redisson-spring-boot-starter</artifactId>
            <version>${redisson.version}</version>
        </dependency>
```

```
redisson:
  singleServerConfig:
    address: 127.0.0.1:6379
```

```
 @Autowired
 private final RedissonClient redissonClient;
```

```
RLock lock = redissonClient.getLock("visitCount");
lock.lock();
//...
lock.unlock();
```



# Flyway管理数据库版本

```
<dependency>
	<groupId>org.flywaydb</groupId>
	<artifactId>flyway-core</artifactId>
	<version>5.0.3</version>
</dependency>
```

简单使用

按Flyway的规范在工程的`src/main/resources`目录下创建`db`目录,

在`db`目录下创建版本化的SQL脚本`V1__Base_version.sql`

```
DROP TABLE IF EXISTS user ;
CREATE TABLE `user` (
  `id` bigint(20) NOT NULL AUTO_INCREMENT COMMENT '主键',
  `name` varchar(20) NOT NULL COMMENT '姓名',
  `age` int(5) DEFAULT NULL COMMENT '年龄',
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

在`application.properties`文件中配置Flyway要加载的SQL脚本位置。按第二步创建的结果配置如下：

```
spring.flyway.locations=classpath:/db
#1.x
flyway.locations=classpath:/db
```

执行单元测试使用该数据库,第一次执行没问题,Flyway监测到需要运行版本脚本来初始化数据库，因此执行了`V1__Base_version.sql`脚本，从而创建了user表，这才得以让一系列单元测试（对user表的CRUD操作）通过。

如果第二次执行测试,由于初始化脚本已经执行过，所以这次执行就没有再去执行`V1__Base_version.sql`脚本来重建user表。

如果第三次修改脚本,比如修改name字段长度后再执行测试,由于初始化脚本的改动，Flyway校验失败，认为当前的`V1__Base_version.sql`脚本与上一次执行的内容不同，提示报错并终止程序，以免造成更严重的数据结构破坏。

# 数据脚本初始化

Datasource初始化机制:

2.5以前为:org.springframework.boot.autoconfigure.jdbc.DataSourceProperties类的属性配置内容.

对应配置文件

```
spring.datasource.schema=
spring.datasource.schema-username=
spring.datasource.schema-password=
...
//这些配置主要用来指定数据源初始化之后要用什么用户、去执行哪些脚本、遇到错误是否继续等功能。
```

2.5后

org.springframework.boot.autoconfigure.sql.init.SqlInitializationProperties

```
spring.datasource.url=jdbc:mysql://localhost:3306/test
spring.datasource.username=root
spring.datasource.password=
spring.datasource.driver-class-name=com.mysql.cj.jdbc.Driver

# Spring Boot 2.5.0 init schema & data
# 执行初始化脚本的用户名称
spring.sql.init.username=root
# 执行初始化脚本的用户密码
spring.sql.init.password=
# 初始化的schema脚本位置
spring.sql.init.schema-locations=classpath*:schema-all.sql

```

- 根据上面配置的定义，接下来就在`resource`目录下，创建脚本文件`schema-all.sql`，并写入一些初始化表结构的脚本

- 完成上面步骤之后，启动应用。然后打开MySQL客户端，可以看到在`test`库下，多了一个`user_info`表

**配置详解**

除了上面用到的配置属性之外，还有一些其他的配置，下面详细讲解一下作用。

- `spring.sql.init.enabled`：是否启动初始化的开关，默认是true。如果不想执行初始化脚本，设置为false即可。通过-D的命令行参数会更容易控制。
- `spring.sql.init.username`和`spring.sql.init.password`：配置执行初始化脚本的用户名与密码。这个非常有必要，因为安全管理要求，通常给业务应用分配的用户对一些建表删表等命令没有权限。这样就可以与datasource中的用户分开管理。
- `spring.sql.init.schema-locations`：配置与schema变更相关的sql脚本，可配置多个（默认用`;`分割）
- `spring.sql.init.data-locations`：用来配置与数据相关的sql脚本，可配置多个（默认用`;`分割）
- `spring.sql.init.encoding`：配置脚本文件的编码
- `spring.sql.init.separator`：配置多个sql文件的分隔符，默认是`;`
- `spring.sql.init.continue-on-error：如果执行脚本过程中碰到错误是否继续，默认是`false`；所以，上面的例子第二次执行的时候会报错并启动失败，因为第一次执行的时候表已经存在。



联合Flyway一同使用，通过`org.springframework.jdbc.datasource.init.DataSourceInitializer`来定义更复杂的执行逻辑。

# LDAP管理用户与组织数据

https://blog.didispace.com/spring-boot-learning-24-6-2/
