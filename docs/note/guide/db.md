# OLAP OLTP

OLAP联机分析处理:

对海量的历史数据进行分析操作,要求产生决策性的影响,不要求在极短的时间内返回结果

 --> 数据仓库  如hive 采用 hash索引

--> clickhoust/druid/kylin

OLTP联机事务处理:

为了支撑业务系统的需要,必须在极短时间内返回对应的结果

-->关系型数据库

# mysql

**局部性原理**:之前被查询过的数据很可能再次被查询;

**磁盘预读:**在磁盘与内存进行交互的时候,有一个最小逻辑单元称之为页,datapage,与操作系统相关,一般为4k或8k,每次在进行数据交互的时候,一定读取的是页的整数倍.

mysql的innodb在进行数据读取的时候也是跟页相关的,datapagesize,默认情况下是16kb.,包括树的节点.

如innodb节点大小可通过:查看

```
show global status like 'Innodb_page_size' 
```

**mysql结构**

![MySQL架构图](picture/v2-7692847118da7b1fd1835066a9e4e273_1440w.jpg)

server中的查询缓存在mysql8版本之后失效:命中率低,查询一次清空.

简单来说 MySQL 主要分为 Server 层和存储引擎层：

**Server 层**：主要包括连接器、查询缓存、分析器、优化器、执行器等，所有跨存储引擎的功能都在这一层实现，比如存储过程、触发器、视图，函数等，还有一个通用的日志模块 binglog 日志模块。

**存储引擎**： 主要负责数据的存储和读取，采用可以替换的插件式架构，支持 InnoDB、MyISAM、Memory 等多个存储引擎，其中 InnoDB 引擎有自有的日志模块 redolog 模块。



## 索引

> 存储在磁盘中,但是会预先加载到内存中,而且在进行加载的时候是分页加载;

链表:时间复杂度高,效率低;

hash表:需要优秀的hash算法;不支持顺序和范围查询;需要大量内存空间(哈希碰撞全部加载),memory使用hash索引,innodb支持自适应hash;

BST(二叉搜索树):数据不平衡时会退化成链表;

AVL(平衡二叉搜索树):为保证平衡经常需要旋转操作,耗费性能;

红黑树(最长子树高度不超过最短子树两倍,减少了avl树旋转的次数):当插入的数据足够多的时候,树会变得很高(因为是二叉),io次数就会变多,查询变慢;

B树: 节点(一个磁盘块大小)中除了指向子节点的指针外,存储的数据占用节点大量空间,范围查询需要中序遍历;

B+树:所有数据存储在叶子节点,非叶子只存储指针和索引值,每个节点能索引的范围更大更精确. 叶节点增加的链指针,加强了区间访问性:

> 每一个节点最多存储degree-1个数据,degree越大(datapagesize和索引大小决定),每个节点存储的数据越多,io次数越少
>
> datapagesize(16) / (索引大小+地址指针大小) = 每个节点存储索引的个数(degree -1) 

> 为了使b+树足够矮,尽量减少索引值的占用空间(比如varchar大于4B用int|使用前缀短索引), 通过千万级别数据3-4层足以(结合datapagesize,数据量,索引大小由mysql确定).

> 复合索引的结构只是先计算左边的值然后再比较后面的值;
>
> 创建索引的时候,要保证唯一的值足够多,这样才有意义,一般情况下是80%
>
> 与其说是数据库只支持一条查询语句只使用一个索引，倒不如说N条独立索引同时在一条语句使用的消耗比只使用一个索引还要慢。
>
> 读取索引的时候一层一层的加载到内存.	

### 聚簇索引

聚簇索引具有唯一性, 表中行的物理顺序即聚簇索引中行的物理顺序;

聚簇索引默认是主键,唯一非空索引,rowid.

访问同一数据页(datapage)不同 行记录时,都会在一次IO 操作中被读取到缓存中,下一次访问直接在内存操作.

而MyISAM因为没有聚簇索引,数据随机存储,获取每条数据都要进行一次io

> 如果已经设置了主键为聚簇索引又希望再单独设置聚簇索引，必须先删除主键，然后添加我们想要的聚簇索引， 最后恢复设置主键即可。

innode数据+索引文件:  `.idb`

myisam数据+索引: `.myd` `.myi`



### 索引维护

**页分裂**

当固定磁盘块的数据节点如[1,2,3,5]要插入一个4,需要进行页分裂,但是页分裂不仅仅是分裂叶子节点这么简单,其上面的上层节点也要进行维护(超出大小也要分裂).

所以自增id每次插在最后而不会影响之前产生的树.

**页合并**

当删了一行记录时，实际上记录并没有被物理删除，记录被标记（flaged）为删除并且它的空间变得允许被其他记录声明使用。当页中删除的记录达到`MERGE_THRESHOLD`（默认页体积的50%），InnoDB会开始寻找最靠近的页（前或后）看看是否可以将两个页合并以优化空间使用。

### 索引覆盖

回表效率不高;

在id为主键的情况下,对于

```
select id,name from table where name=zhangsan;
```

可对name建立索引达到索引覆盖

对于

```
select id,name,age from table where name=zhangsan;
```

可对(name,age)作为复合索引

### 最左匹配原则

需要先匹配左边,再匹配右边,直到遇到范围查询(>、<、between、 like)就停止匹配.

如果我们创建了(area, age, salary)的复合索引，那么其实相当于创建了(area,age,salary)、(area,age)、(area)三个索引.  因此我们在创建复合索引时应该将最常用作限制条件的列放在最左边，依次递减。

对于复合索引(name,age)

```
select * from table where name=?   √
select * from table where name=? and age=?  √
select * from table where age=> and  name=?   
select * from table where age=?  ×
```

对于第三种情况, 在优化器组件的作用下通过调整顺序来匹配索引.



最左匹配原则在mysql优化下可能不会生效

对于字段id,a,b,c ,主键索引id和复合索引(a,b,c);

```
select * from table where b=2 and c=3;
可能会走复合索引,因为走索引可以达到索引覆盖
```

而对于字段id,a,b,c,d ,主键索引id和复合索引(a,b,c);

```
select * from table where b=2 and c=3;
可能不会走复合索引,因为走索引要回表
```



### 索引下推

- 索引条件下推(Index Condition Pushdown),简称ICP。MySQL5.6新添加，用于优化数据的查询。
- 当你不使用ICP,通过使用非主键索引（普通索引or二级索引）进行查询，存储引擎通过索引检索数据，然后返回给MySQL服务器，服务器再判断是否符合条件。
- 使用ICP，当存在索引的列做为判断条件时，MySQL服务器将这一部分判断条件传递给存储引擎，然后存储引擎通过判断索引是否符合MySQL服务器传递的条件，只有当索引符合条件时才会将数据检索出来返回给MySQL服务器。

```
select * from table where name like '%aa' and age=?
```

在没有索引下推之前,因为索引失效先根据name去存储引擎中拉取复合结果的数据,返回到server层,在server层中对age的条件进行过滤;

有了索引下推之后,根据name,age两个条件直接从存储引擎中拉去结果,不需要在server层做条件过滤;

在mysql5.7以后默认开启.

> mysql server端由连接器,分析器,优化器,执行器组成

查看 show variables like '%switch%'

index_condition_push=on

当使用explan进行分析时，如果使用了索引条件下推，Extra会显示**Using index condition**。

### 索引失效

1.OR  (可使用union all 或 union 代替);

2.复合索引不符合最左匹配;

3.like以%开头;

4.索引列上做计算,函数,类型转换;

5.如果mysql觉得全表扫描更快时（数据少）;



## 执行计划explain

explain关键字可以模拟MySQL优化器执行SQL语句，可以很好的分析SQL语句或表结构的性能 瓶颈。

> explain extended 可查看相关警告信息

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
1. id 				//select查询的序列号，包含一组数字，表示查询中执行select子句或操作表的顺序
2. select_type 		//查询类型 主要用于区别普通查询、联合查询、子查询等的复杂查询
3. table 			//这个数据是基于哪张表的
4. partitions 		//匹配的分区
5. type 			//访问的类型
6. possible_keys 	//与当前查询相关备选的索引有哪些, PRIMARY代表主键
7. key 				//代表当前实际使用的索引是哪个
8. key_len 			//代表单个索引值的长度
9. ref 				//显示使用哪个列或常数与key一起从表中选择行
10. rows 			//本次查询所扫描的行数, 不一定一致, 扫描行数越少越好
11. filtered 		//查询的表行占表的百分比
12. Extra 			//包含不适合在其它列中显示但十分重要的额外信息
```

**id字段**

```
1.id相同 : 执行顺序从上至下  (最上方的为驱动)
2.id不同 : 如果是子查询，id的序号会递增，id的值越大优先级越高，越先被执行
3.id相同又不同 : id如果相同，可以认为是一组，从上往下顺序执行 在所有组中，id值越大，优先级越高，越先执行
```

**select_type字段**(关键)

```
1.SIMPLE : 简单查询，不包含子查询或Union查询
2.PRIMARY : 查询中若包含任何复杂的子部分，最外层查询则被标记为主查询
3.SUBQUERY : 在select或where中包含子查询
4.DERIVED : 在FROM列表中包含的子查询被标记为DERIVED（衍生），MySQL 会递归执行这些子查询，把结果放在临时表中
			MySQL5.7+ 进行优化了，增加了derived_merge（派生合并），默认开启，可加快查询效率
5.UNION : 若第二个select出现在union之后，则被标记为UNION
6.UNION RESULT : 从UNION表获取结果的select (去重)
```

**type字段**(关键)

```
表示索引检索类型, 以下顺序从好到差
1.NULL : MySQL能够在优化阶段分解查询语句，在执行阶段用不着再访问表或索引.
比如通过id没有找到 explain select min(id) from subject;
2.system : 表只有一行记录（等于系统表），这是const类型的特列，平时不大会出现，
3.const : 表示通过索引一次就找到了，const用于比较primary key或uique索引，因为只匹配一行数据，所以很快，如主键置于where列表中，MySQL就能将该查询转换为一个常量.
4.eq_ref :唯一性索引扫描，对于每个索引键，表中只有一条记录与之匹配，常见于主键或唯一索引扫描
5.ref : 非唯一性索引扫描，返回匹配某个单独值的所有行, 本质上也是一种索引访问，返回所有匹配某个单独值的行,然而可能会找到多个符合条件的行，应该属于查找和扫描的混合体
6.ref_or_null : 类似ref, 但是可以搜索值为null的行
7.index_merge : 表示使用了索引合并的优化方法
8.range : 只检索给定范围的行，使用一个索引来选择行，key列显示使用了哪个索引 一般就是在你的where语句中出现between、<>、in等的查询。
9.index : Full index Scan
如explain select id from subject; 
10.ALL : Full Table Scan，将遍历全表以找到匹配行.
Index与All区别：index只遍历索引树，通常比All快 因为索引文件通常比数据文件小，也就是虽然all和index都是读全表，但index是从索引中读取的，而all 是从硬盘读的。
```

**Extra字段**(关键)

```
1.Using filesort : 说明MySQL会对数据使用一个外部的索引排序，而不是按照表内的索引顺序进行读取 
	MySQL中无法利用索引完成的排序操作称为“文件排序”
	优化方式,给排序字段建索引，并使用where或limit激活
2.Using temporary : 使用了临时表保存中间结果，MySQL在对结果排序时使用临时表，常见于排序order by 和分组查询group by
	优化方式,给分组字段建索引
3.Using index : 表示相应的select操作中使用了覆盖索引(Covering Index ,就是select的数据列只用从索引中就能够取得，不必从数据表中读取，换句话说查询列要被所使用的索引覆盖)，避免访问了表的数据行，效率不错！
4.Using where : 使用了where条件,表示优化器需要通过索引回表查询数据
5.Using join buffer :使用了连接缓存 
	explain select student.*,teacher.*,subject.* from student,teacher,subject;
	explain select * from emp ,dept where emp.empno = dept.ceo ;
6.impossible where : where子句的值总是false，不能用来获取任何元组
7.distinct : 一旦mysql找到了与行相联合匹配的行，就不再搜索了, 如左连接里的右表
8.Select tables optimized away
	explain select * from emp ,dept where emp.empno = dept.ceo ;
	explain select min(id) from subject;
9.using MMR : MMR(Multi Range Read), 当查询通过二级索引得到的主键值进行回表时,如果回表的主键值为二级索引的顺序,对应的主键就是乱序的, 回表的时候就需要进行大量随机IO; MRR则在内存中将二级索引中命中的主键值进行根据主键列进行排序后才进行回表,这样回表则是顺序IO,极大提高了性能.
```

### NLJ

 NLJ(Nested Loop Join 嵌套循环关联)

与编程中的二层嵌套类似,  驱动表中的每一条记录与被驱动表总的记录进行比较, 驱动表的选择决定了查询性能的高低.

> mysql会自动选择最优驱动表, 但是在多级关联情况下有可能会出现选择问题

案例:

在mysql8多表关联查询语句下:

![image-20211211180444460](picture/image-20211211180444460.png)

执行计划为:

![image-20211211180520940](picture/image-20211211180520940.png)

按上下顺序, 最顶端的是驱动表, 此时未加索引, h表不幸被选中作为驱动表 , 进行全表关联, 效率极差.

此过程先对h表全表扫描576931行, 然后根据ref的关联在被驱动表中查询.

此时, 如果只给筛选条件的字段m.role和c.series_id加索引(小白做法):

```
create index idx_series_id on blog_chapter(series_id);
create index idx_role on blog_menber(role);
```

再查看执行计划, 会发现与没加索引结果一样.

> 结论:如果在多表关联时只在筛选条件上加字段索引, 没卵用.

此时, 再给关联条件的外键增加索引:

```
create index idx_chapter_id on blog_browse_history(chapter_id);  	#新增
create index idx_menber_id on blog_browse_history(menber_id);		#新增
create index idx_series_id on blog_chapter(series_id);
create index idx_role on blog_menber(role);
```

再查看执行计划:

![image-20211211182325320](picture/image-20211211182325320.png)

此时的驱动顺序为: c -> h -> m

先在驱动表c中通过对series_id进行const筛选(103),  这里的Using index表示通过series_id二级索引查询到主键chapter_id后并未回表, 直接使用主键与被驱动表关联.

然后h表根据c表查询出来的主键(chapter_id)通过外键关联进行查询(由于h表对外键chapter_id已建立索引, 所以查询效率高) , 这里的Using where表示通过二级索引chapter_id查询到h表的主键后进行回表.

再然后m表根据h表查询出来的外键(menber_id)直接进行主键查询. (但是不懂这里为什么Using where)

> 结论:只有正确的在外键上建立索引(关联的主键自带索引), 在关联表的索引才能生效, 查询优化器才能正确决定用哪个表作为驱动表是最优解



另外

1.在索引正确的情况下, 如果将多表关联改为where in 子查询,  在查询优化器的驱使执行计划不会改变, 仍然使用NLJ高效查询.

![image-20211211185611638](picture/image-20211211185611638.png)

查看查询计划与上面结果一样.

2.在索引正确的情况下, 如果将多表关联改为from子句筛选, 在查询优化器的驱使执行计划不会改变, 仍然使用NLJ高效查询.

![image-20211211185819137](picture/image-20211211185819137.png)

查看查询计划与上面结果一样.



3.例外:如果select包含子查询, 会出现DEPENDENT SUBQUERY, 代表依赖子查询, 也属于NLJ范畴

![image-20211211190254986](picture/image-20211211190254986.png)

该方式在每查询出一条记录后,就将这条记录的某个属性拿去进行子查询.

### 其他执行计划

- DERIVED 子查询中出现多结果集运算![image-20211211191021781](picture/image-20211211191021781.png)

id越大的子查询越先执行:

id=4和id=3时: 通过两个根据series_id查询形成一个结果集

id相同时从上往下执行:

< derived3 >表示指向id为3的执行计划以及向后的延伸. 这里为id=4和id=3的结果集,

id=1的执行计划先以derived3为驱动表c,通过c表的chapter_id查询h表二级索引,然后回表获取h表数据,然后再根据h表的menber_id查询blog_menber表数据(没有走二级索引)

- UNION RESULT 少用UNION, 多用UNION ALL.



![image-20211211193250977](picture/image-20211211193250977.png)

这里的UNION RESULT执行计划表示对union3和4进行去重, 通过类型可以看到这个操作为全表扫描 , 关键的是Using temporary

> UNION去重基于临时表, 临时表特性时如果缓存够使用内容, 缓存不够自动创建MyISAM引擎表(磁盘), IO效率变差.

## 锁

数据库锁机制简单来说，就是数据库为了保证数据的一致性，使各种 共享资源 在被访问时变得 有 序而设计 的一种规则。

MysQL的锁机制比较简单最著的特点是不同的存储引擎支持不同的锁机制。 InoDB支持行锁,(有时 也会升级为表锁）MyISAM只支持表锁。

表锁 的特点就是开销小、加锁快，不会出现死锁。锁粒度大，发生锁冲突的概率小，并发度相对 低。

行锁 的特点就是开销大、加锁慢，会出现死锁。锁粒度小，发生锁冲突的概率高，并发度搞。

**表锁**

表锁由 MySQL服务器实现，无论存储引擎是什么，都可以使用表锁。一般在执行 DDL 语句时，譬如 ALTER TABLE 时就会对整个表进行加锁。在执行 SQL 语句时，也可以明确对某个表加锁。

给表显式加锁语句为：

```
#加表读锁
lock table {tableName} read;

#释放表锁
unlock tables；

#查看表锁
show open table;

#加表写锁
lock table {tableName} write;

#释放表读锁
unlock tables；
```

在使用MYISAM引擎时，通常我们不需要手动加锁，因为MYISAM引擎会针对我们的sql语句自动进行加锁，整个过程不需要用户干预：

1. 查询语句（select）：会自动给涉及的表加读锁；
2. 更新语句（update、delete、insert）：会自动给涉及的表加写锁。

**InnoDB行锁种类**

查看隔离级别

```
show variables like '%iso%';
```

注意:大多数数据库默认的事务隔离级别为读已提交RC , 而innoDB默认的事务隔离级别为可重复度RR

1.记录锁

```
不加索引，两个事务修改同一行记录,不能同时;不加索引，两个事务修改同一表非同行记录,不能同时;锁的是表↑加索引，修改同一行记录，不能同时;加索引，修改同表的不同行，可以同时;锁的是行,证明行锁是加在索引上的，这是标准的行级锁。
```

InnoDB的行锁是通过对**索引项加锁**实现的，这表示只有通过索引查询记录时才会使用行锁，如果不走索引查询数据将使用表锁，则性能会大打折扣。

2.间隙锁(GPA锁)

间隙锁是一种区间锁。锁加在不存在的空闲空间上，或者两个索引记录之间，或者第一个索引记录，或者最后一个索引之后的空间，用来表示只锁住一段范围（一般在进行范围查询时且隔离级别在RR或Serializable隔时）。

一般在RR隔离级别下会使用到GAP锁。使用GAP锁，主要是为了防止幻读产生，在被GAP锁锁住的区间，不允许插入数据或者更新数据。

间隙锁区域定义：

```
根据检索条件向左寻找最靠近的值A，作为左区间，向右寻找最靠近的值B，作为右区间，间隙锁为（A，B）向左找不到最近的值A，也是就无穷小，作为左区间，向右寻找最靠近的值B，作为右区间，间隙锁为（无穷小，B）向左找到最近的值A，作为左区间，向右寻找不到最近的值B，也就是无穷大，作为右区间，间隙锁为（A，无穷大）
```

MYSQL的锁机制非常复杂，在实际的开发工作中，对于隔离级别的设置都需要非常谨慎，比如RR级别会比RC级别多出一个间隙锁，这就可能导致严重的性能问题。

3.组合

当InnoDb扫描索引时会先对索引记录加上记录锁，在对索记录两边加上间 隙锁。

**InnoDB锁类型**

1.读锁(共享锁,  shared lock,s锁)

一个事务获取了一个数据行的读锁，其他事务能获得该行对应 的读锁但不能获得写锁，即一个事务在读取一个数据行时，其他事务也可以读，但不能对该数行增删改 的操作。

**简而言之：就是可以多个事务读，但只能一个事务写。**

读锁有两种select方式的应用： 

1. 第一种是自动提交模式下的select查询语句，不需加任何锁,直接返回查询结果，这就是一致性非锁 定读。 
2. 第二种就是通过select.... lock in share mode被读取的行记录或行记录的范围上加一个读锁,让其 他事务可以读,但是要想申请加写锁,那就会被阻塞

**注意:**

```
在行（也就是记录）上加锁，并不是锁住该条记录，而是在记录对应的索引上加锁。如果where条件中不走索引，则会对所有的记录加锁。这里所说的读，是指当前读，快照读是无需加锁的。普通select读一般都是快照读，除了select...lock in share mode这样的显式加锁语句下会变成当前读，在InnoDB引擎的serializable级别下，普通select读也会变成当前读。
```

2.写锁(排他锁,Exclusive Lock ,x锁)

写锁，也叫排他锁，或者叫独占所，简称x锁。一个事务获取了一个数据行的写锁，其他事务就不能再 获取该行的其他锁与锁优先级最高。

**简而言之：就是只能有一个事务操作这个数据，别的事务都不行。**

写锁的应用就很简单了，有以下两种情况：

 (1) insert/update/delete语句，会自动在该条记录上加上写锁

如两个事务对同一行update;

 (2) 比较特殊的就是select for update，它会对读取的行记录上加一个写锁，那么其他任何事务戴不 能对被锁定的行上加任何锁了，要不然会被阻塞。

**注意:写锁也是加在索引上的**

**但是发现上述两种情况其他事务还能对该行读,这是因为mysql实现了MVCC模型**

3.MDL锁

MySQL 5.5引入了meta data lock，简称MDL锁，用于保证表中 元数据 的信息  ,系统级的锁无法主动控制。主要是为了在并发环境下对DDL、DML同时操作下保持元数据的一致性。

在事务A中，开启了表查询事务，会自动获得一个MDL锁，

事务B就不可以执行任何DDL语句，比如为表中添加字段 ,删除表.

1. 对表中的记录进行增删改查（DML操作）的时候，自动加MDL读锁；
2. 对表的结构（DDL操作）进行修改的时候，自动加MDL写锁。

根据MDL锁的粒度和模式,MDL锁还分成了很多种类

4.意向锁

在mysql的innodb引擎中，意向锁是表级锁，是一种不与行级锁冲突的表级锁，表示表中的记录所需要的锁(S锁或X锁)的类型（其实就是告诉你，这张表中已经存在了行锁（行锁的类型），所以叫意向锁）。InnoDB支持多种粒度的锁，允许行级锁和表级锁的共存。

意向锁有两种 

意向共享锁（IS） 是指在给一个数据行加共享锁前必须获取该表的意向共享锁

意向排它锁（IX） 是指在给一个数据行加排他锁前必须获取该表的意向排他锁

**意向锁与意向锁之间是不互斥的，但是意向锁与其他表锁之间存在一定的兼容互斥**

|               | 表意向共享锁（IS） | 表意向排他锁（IX） |
| ------------- | ------------------ | ------------------ |
| 表共享锁（S） | 兼容               | 互斥               |
| 表排他锁（X） | 互斥               | 互斥               |

比如:

```
事务 A 获取了某一行的排他锁，并未提交：begin;	SELECT * FROM test_user WHERE id = 28 FOR UPDATE;# 此时事务A获取了两把锁：test_user 表上的意向排他锁与 id 为 28 的数据行上的排他锁。事务 B 想要获取表的表共享锁；begin;LOCK TABLE test_user READ;# 因为共享锁与排他锁互斥，所以事务 B 在试图对 test_user 表加共享锁的时候，必须保证：1.当前没有其他事务持有 test_user 表的排他锁（表排他锁）。2.当前没有其他事务持有 test_user 表中任意一行的排他锁（行排他锁）。# 为了检测是否满足第二个条件，如果没有意向锁，事务 B 必须在确保 test_user表不存在任何排他锁的前提下，去检测表中的每一行是否存在排他锁；但是如果有了意向锁之后，事务 B 只需要检测事务 A 是否持有 test_user 表的意向排他锁，就可以得知事务 A 必然持有该表中某些数据行的排他锁，那么事务 B 对 test_users 表的加锁请求就会被排斥（阻塞），从而无需去检测表中的每一行数据是否存在排他锁。事务 C 也想获取表中某一行的排他锁：begin;	SELECT * FROM test_user WHERE id = 31 FOR UPDATE;# 因为意向锁之间并不互斥，所以事务C获取到了test_user的意向排他锁# 因为id 为 31 的数据行上不存在任何排他锁，最终事务 C 成功获取到了该数据行上的排他锁。
```

5.自增锁

在InnoDB存储引擎中，针对每个自增长的字段都设置了一个自增长的计数器。我们可以执行下面的语句来得到这个计数器的当前值：

```
select max(自增长列) from table;
```

当我们进行插入操作的时候，该操作会根据这个自增长的计数器的当前值进行+1操作，并赋予自增长的列，这个操作我们称之为auto-inc Locking，也就是自增长锁，这种锁其实采用的是特殊的表锁机制，如果insert操作出现在一个事务中，这个锁是在insert操作完成之后立即释放，而不是等待事务提交。

6.页级锁

MySQL中锁定粒度介于行级锁和表级锁中间的一种锁。表级锁速度快，但冲突多，行级冲突少，但速度慢。所以取了折衷的页级，一次锁定相邻的一组记录。不同的存储引擎支持不同的锁机制。

页级锁是MySQL中比较独特的一种锁定级别，**应用于BDB引擎**，并发度一般，页级锁定的特点是锁定颗粒度介于行级锁定与表级锁之间，所以获取锁定所需要的资源开销，以及所能提供的并发处理能力也同样是介于上面二者之间。另外，页级锁定和行级锁定一样，会发生死锁。

锁定粒度大小比较：表级锁 > 页级锁 > 行级锁

**乐观锁和悲观锁**

1.乐观锁

乐观锁大多是基于数据版本记录机制实现，一般是给数据库表增加一个"version"字段。读取数据时，将 此版本号一同读出，之后更新时，对此版本号加一。此时将提交数据的版本数据与数据库表对应记录的 当前版本信息进行比对，如果提交的数据版本号大于数据库表当前版本号，则予以更新，否则认为是过 期数据。

2.悲观锁

悲观锁依靠数据库提供的锁机制实现。MySQL中的共享锁和排它锁都是悲观锁。数据库的增删改操作默 认都会加排他锁，而查询不会加任何锁。此处不赘述

**锁等待和死锁**

锁等待是指一个事务过程中产生的锁，其他事务需要等待上一个事务释放它的锁，才能占用该资源。如 果该事务一直不释放，就需要持续等待下去，直到超过了锁等待时间，会报一个等待超时的错误。

MysQL中通过innodb_lock_wait_timeout参数控制,单位是秒。

死锁是指两个或两个以上的进程在执行过程中，因争夺资源而造成的一种互相等待的现象，就是所谓的 锁资源请求产生了回路现象，即死循环。

```
死锁的条件1. 两行记录，至少两个事务2. 事务A 操作 第n行数据，并加锁 update teacher set name = 'a' where id = 1;3. 事务B 操作 第m行数据，并加锁 update teacher set name = 'b' where id = 2;4. 事务A 操作 第m行数据 update teacher set name = 'c' where id = 2;5. 事务B 操作 第n行数据 update teacher set name = 'd' where id = 1;6. 形成死锁 Deadlock found when trying to get lock; try restartingtransaction
```

InnoDB引擎可以自动检测死锁并 回滚该事务 好不容易执行了一个业务给我回滚了，所以死锁尽量不要 出现。

## 事务

ACID特性

**原子性**（`Atomicity`） ： 事务是最小的执行单位，不允许分割。事务的原子性确保动作要么全部完成，要么完全不起作用；

**一致性**（`Consistency`）： 执行事务前后，数据保持一致，例如转账业务中，无论事务是否成功，转账者和收款人的总额应该是不变的；

**隔离性**（`Isolation`）： 并发访问数据库时，一个用户的事务不被其他事务所干扰，各并发事务之间数据库是独立的；

**持久性**（`Durability`）： 一个事务被提交之后。它对数据库中数据的改变是持久的，即使数据库发生故障也不应该对其有任何影响。



并发事务问题:

**脏读（Dirty read）:** 脏读是指事务读取到其他事务没提交的数据

![image-20211207194859328](picture/image-20211207194859328.png)

**丢失修改（Lost to modify）:** 指在一个事务读取一个数据时，另外一个事务也访问了该数据，那么在第一个事务中修改了这个数据后，第二个事务也修改了这个数据。这样第一个事务内的修改结果就被丢失，因此称为丢失修改。 例如：事务 1 读取某表中的数据 A=20，事务 2 也读取 A=20，事务 1 修改 A=A-1，事务 2 也修改 A=A-1，最终结果 A=19，事务 1 的修改被丢失。

**不可重复读（Unrepeatable read）:** 不可重复读是指在同一次事务中前后查询不一致的问题

![image-20211207195037888](picture/image-20211207195037888.png)

**幻读（Phantom read）:** 幻读是一次事务中前后数据量发生变化用户产生不可预料的问题

![image-20211207195053772](picture/image-20211207195053772.png)

**不可重复读和幻读区别：**

不可重复读的重点是修改比如多次读取一条记录发现其中某些列的值被修改，幻读的重点在于新增或者删除比如多次读取一条记录发现记录增多或减少了(伤害不大)。



**隔离级别**

**READ-UNCOMMITTED(读取未提交)：** 最低的隔离级别，允许读取其他事务尚未提交的数据变更，**可能会导致脏读、幻读或不可重复读**。

**READ-COMMITTED(读取已提交)：** 允许读取并发事务已经提交的数据，**可以阻止脏读，但是幻读或不可重复读仍有可能发生**。

**REPEATABLE-READ(可重复读)：** 对同一字段的多次读取结果都是一致的，除非数据是被本身事务自己所修改，**可以阻止脏读和不可重复读，但幻读仍有可能发生**。

**SERIALIZABLE(可串行化)：** 最高的隔离级别，完全服从 ACID 的隔离级别。所有的事务依次逐个执行，这样事务之间就完全不可能产生干扰，也就是说，**该级别可以防止脏读、不可重复读以及幻读**。**此隔离级别是唯一使用加锁读的方式的隔离级别.**

> MySQL InnoDB 的 REPEATABLE-READ（可重读）可避免幻读，需要应用使用加锁读来保证。而这个加锁度使用到的机制就是 Next-Key Locks(MVCC)。

> 因为隔离级别越低，事务请求的锁越少，所以大部分数据库系统的隔离级别都是 **READ-COMMITTED(读取提交内容)** ，但是InnoDB 存储引擎默认使用 **REPEATABLE-READ（可重读）** 并不会有任何性能损失。

 InnoDB 存储引擎的默认支持的隔离级别是 **REPEATABLE-READ（可重读）**。

我们可以通过`SELECT @@tx_isolation;` 或者 `SHOW VARIABLES LIKE 'transaction_isolation';`查看

通过 `SET SESSION TRANSACTION ISOLATION LEVEL READ UNCOMMITTED;` 设置当前会话的隔离级别.

> 通常我们不需要修改mysql隔离级别

## MVCC

MVCC ，全称 Multi-Version Concurrency Control ，即多版本并发控制。MVCC是基于”数据版本”对并发事务进行访问.

一致性的非锁定行读（consistent nonlocking read，简称CNR）是指InnoDB存储引擎通过行多版本控制（multi versioning）的方式来读取当前执行时间数据库中运行的数据

https://www.cnblogs.com/xuwc/p/13873611.html

**一致性非锁定读**

对于 [一致性非锁定读（Consistent Nonlocking Reads）](https://dev.mysql.com/doc/refman/5.7/en/innodb-consistent-read.html)的实现，通常做法是加一个版本号或者时间戳字段，在更新数据的同时版本号 + 1 或者更新时间戳。查询时，将当前可见的版本号与对应记录的版本号进行比对，如果记录的版本小于可见版本，则表示该记录可见

在 `InnoDB` 存储引擎中，[多版本控制 (multi versioning)](https://dev.mysql.com/doc/refman/5.7/en/innodb-multi-versioning.html) 就是对非锁定读的实现。如果读取的行正在执行 `DELETE` 或 `UPDATE` 操作，这时读取操作不会去等待行上锁的释放。相反地，`InnoDB` 存储引擎会去读取行的一个快照数据，对于这种读取历史数据的方式，我们叫它快照读 (snapshot read)

在 `Repeatable Read` 和 `Read Committed` 两个隔离级别下，如果是执行普通的 `select` 语句（不包括 `select ... lock in share mode` ,`select ... for update`）则会使用 `一致性非锁定读（MVCC）`。并且在 `Repeatable Read` 下 `MVCC` 实现了可重复读和防止部分幻读

**锁定读**

如果执行的是下列语句，就是 [锁定读（Locking Reads）](https://dev.mysql.com/doc/refman/5.7/en/innodb-locking-reads.html)

- `select ... lock in share mode`
- `select ... for update`
- `insert`、`update`、`delete` 操作

在锁定读下，读取的是数据的最新版本，这种读也被称为 `当前读（current read）`。锁定读会对读取到的记录加锁：

- `select ... lock in share mode`：对记录加 `S` 锁，其它事务也可以加`S`锁，如果加 `x` 锁则会被阻塞
- `select ... for update`、`insert`、`update`、`delete`：对记录加 `X` 锁，且其它事务不能加任何锁

在一致性非锁定读下，即使读取的记录已被其它事务加上 `X` 锁，这时记录也是可以被读取的，即读取的快照数据。上面说了，在 `Repeatable Read` 下 `MVCC` 防止了部分幻读，这边的 “部分” 是指在 `一致性非锁定读` 情况下，只能读取到第一次查询之前所插入的数据（根据 Read View 判断数据可见性，Read View 在第一次查询时生成）。但是！如果是 `当前读` ，每次读取的都是最新数据，这时如果两次查询中间有其它事务插入数据，就会产生幻读。所以， **`InnoDB` 在实现`Repeatable Read` 时，如果执行的是当前读，则会对读取的记录使用 `Next-key Lock` ，来防止其它事务在间隙间插入数据**.



**innodb对mvcc实现**

`MVCC` 的实现依赖于：**隐藏字段、Read View、undo log**。在内部实现中，`InnoDB` 通过数据行的 `DB_TRX_ID` 和 `Read View` 来判断数据的可见性，如不可见，则通过数据行的 `DB_ROLL_PTR` 找到 `undo log` 中的历史版本。每个事务读到的数据版本可能是不一样的，在同一个事务中，用户只能看到该事务创建 `Read View` 之前已经提交的修改和该事务本身做的修改..

**隐藏字段**

在内部，`InnoDB` 存储引擎为每行数据添加了三个 [隐藏字段  (opens new window)](https://dev.mysql.com/doc/refman/5.7/en/innodb-multi-versioning.html)：

- `DB_TRX_ID（6字节）`：表示最后一次插入或更新该行的事务 id。此外，`delete` 操作在内部被视为更新，只不过会在记录头 `Record header` 中的 `deleted_flag` 字段将其标记为已删除
- `DB_ROLL_PTR（7字节）` 回滚指针，指向该行的 `undo log` 。如果该行未被更新，则为空
- `DB_ROW_ID（6字节）`：如果没有设置主键且该表没有唯一非空索引时，`InnoDB` 会使用该 id 来生成聚簇索引



**Read view**

```c++
class ReadView {
	private:  
		trx_id_t m_low_limit_id;      			/* 大于等于这个 ID 的事务均不可见 */  
		trx_id_t m_up_limit_id;       			/* 小于这个 ID 的事务均可见 */  
		trx_id_t m_creator_trx_id;    			/* 创建该 Read View 的事务ID */  
		trx_id_t m_low_limit_no;      			/* 事务 Number, 小于该 Number 的 Undo Logs 均可以被 Purge */  
		ids_t m_ids;                  			/* 创建 Read View 时的活跃事务列表 */  
		m_closed;                     			/* 标记 Read View 是否 close */
}
```

[`Read View` ](https://github.com/facebook/mysql-8.0/blob/8.0/storage/innobase/include/read0types.h#L298) 主要是用来做可见性判断，里面保存了 “当前对本事务不可见的其他活跃事务”

主要有以下字段：

- `m_low_limit_id`：目前出现过的最大的事务 ID+1，即下一个将被分配的事务 ID。大于等于这个 ID 的数据版本均不可见
- `m_up_limit_id`：活跃事务列表 `m_ids` 中最小的事务 ID，如果 `m_ids` 为空，则 `m_up_limit_id` 为 `m_low_limit_id`。小于这个 ID 的数据版本均可见
- `m_ids`：`Read View` 创建时其他未提交的活跃事务 ID 列表。创建 `Read View`时，将当前未提交事务 ID 记录下来，后续即使它们修改了记录行的值，对于当前事务也是不可见的。`m_ids` 不包括当前事务自己和已提交的事务
- `m_creator_trx_id`：创建该 `Read View` 的事务 ID

![trans_visible](picture/trans_visible.jpg)



**undo log**

`undo log` 主要有两个作用：

- 当事务回滚时用于将数据恢复到修改前的样子
- 另一个作用是 `MVCC` ，当读取记录时，若该记录被其他事务占用或当前版本对该事务不可见，则可以通过 `undo log` 读取之前的版本数据，以此实现非锁定读

**在 `InnoDB` 存储引擎中 `undo log` 分为两种： `insert undo log` 和 `update undo log`：**

1. **`insert undo log`** ：指在 `insert` 操作中产生的 `undo log`。因为 `insert` 操作的记录只对事务本身可见，对其他事务不可见，故该 `undo log` 可以在事务提交后直接删除。不需要进行 `purge` 操作

**`insert` 时的数据初始状态：**

![img](picture/317e91e1-1ee1-42ad-9412-9098d5c6a9ad.png)

2.**`update undo log`** ：`update` 或 `delete` 操作中产生的 `undo log`。该 `undo log`可能需要提供 `MVCC` 机制，因此不能在事务提交时就进行删除。提交时放入 `undo log` 链表，等待 `purge线程` 进行最后的删除.

**数据第一次被修改时：**

![img](picture/c52ff79f-10e6-46cb-b5d4-3c9cbcc1934a.png)

**数据第二次被修改时：**

![img](picture/6a276e7a-b0da-4c7b-bdf7-c0c7b7b3b31c.png)

不同事务或者相同事务的对同一记录行的修改，会使该记录行的 `undo log` 成为一条链表，链首就是最新的记录，链尾就是最早的旧记录。

> mysql确保UNDO_LOG版本链数据不再被“引用”后再进行删除。



**数据可见性算法**

在 `InnoDB` 存储引擎中，创建一个新事务后，执行每个 `select` 语句前，都会创建一个快照（Read View），**快照中保存了当前数据库系统中正处于活跃（没有 commit）的事务的 ID 号**。其实简单的说保存的是系统中当前不应该被本事务看到的其他事务 ID 列表（即 m_ids）。当用户在这个事务中要读取某个记录行的时候，`InnoDB` 会将该记录行的 `DB_TRX_ID` 与 `Read View` 中的一些变量及当前事务 ID 进行比较，判断是否满足可见性条件

[具体的比较算法](https://github.com/facebook/mysql-8.0/blob/8.0/storage/innobase/include/read0types.h#L161)如下：[图源](https://leviathan.vip/2019/03/20/InnoDB的事务分析-MVCC/#MVCC-1)

![img](picture/8778836b-34a8-480b-b8c7-654fe207a8c2.png)

1. 如果记录 DB_TRX_ID < m_up_limit_id，那么表明最新修改该行的事务（DB_TRX_ID）在当前事务创建快照之前就提交了，所以该记录行的值对当前事务是可见的
2. 如果 DB_TRX_ID >= m_low_limit_id，那么表明最新修改该行的事务（DB_TRX_ID）在当前事务创建快照之后才修改该行，所以该记录行的值对当前事务不可见。跳到步骤 5
3. m_ids 为空，则表明在当前事务创建快照之前，修改该行的事务就已经提交了，所以该记录行的值对当前事务是可见的
4. 如果 m_up_limit_id <= DB_TRX_ID < m_low_limit_id，表明最新修改该行的事务（DB_TRX_ID）在当前事务创建快照的时候可能处于“活动状态”或者“已提交状态”；所以就要对活跃事务列表 m_ids 进行查找（源码中是用的二分查找，因为是有序的）
   - 如果在活跃事务列表 m_ids 中能找到 DB_TRX_ID，表明：① 在当前事务创建快照前，该记录行的值被事务 ID 为 DB_TRX_ID 的事务修改了，但没有提交；或者 ② 在当前事务创建快照后，该记录行的值被事务 ID 为 DB_TRX_ID 的事务修改了。这些情况下，这个记录行的值对当前事务都是不可见的。跳到步骤 5
   - 在活跃事务列表中找不到，则表明“id 为 trx_id 的事务”在修改“该记录行的值”后，在“当前事务”创建快照前就已经提交了，所以记录行对当前事务可见
5. 在该记录行的 DB_ROLL_PTR 指针所指向的 `undo log` 取出快照记录，用快照记录的 DB_TRX_ID 跳到步骤 1 重新开始判断，直到找到满足的快照版本或返回空



**RC 和 RR 隔离级别下 MVCC 的差异**

在事务隔离级别 `RC` 和 `RR` （InnoDB 存储引擎的默认事务隔离级别）下，`InnoDB` 存储引擎使用 `MVCC`（非锁定一致性读），但它们生成 `Read View` 的时机却不同

- 在 RC 隔离级别下的 **`每次select`** 查询前都生成一个`Read View` (m_ids 列表)
- 在 RR 隔离级别下只在事务开始后 **`第一次select`** 数据前生成一个`Read View`（m_ids 列表）



**MVCC 解决不可重复读问题**

虽然 RC 和 RR 都通过 `MVCC` 来读取快照数据，但由于 **生成 Read View 时机不同**，从而在 RR 级别下实现可重复读

举个例子：

![img](picture/6fb2b9a1-5f14-4dec-a797-e4cf388ed413.png)

  

**RC 下 ReadView 生成情况**

1.**`假设时间线来到 T4 ，那么此时数据行 id = 1 的版本链为`：**

![img](picture/a3fd1ec6-8f37-42fa-b090-7446d488fd04.png)

由于 RC 级别下每次查询都会生成`Read View` ，并且事务 101、102 并未提交，此时 `103` 事务生成的 `Read View` 中活跃的事务 **`m_ids` 为：[101,102]** ，`m_low_limit_id`为：104，`m_up_limit_id`为：101，`m_creator_trx_id` 为：103

- 此时最新记录的 `DB_TRX_ID` 为 101，m_up_limit_id <= 101 < m_low_limit_id，所以要在 `m_ids` 列表中查找，发现 `DB_TRX_ID` 存在列表中，那么这个记录不可见
- 根据 `DB_ROLL_PTR` 找到 `undo log` 中的上一版本记录，上一条记录的 `DB_TRX_ID` 还是 101，不可见
- 继续找上一条 `DB_TRX_ID`为 1，满足 1 < m_up_limit_id，可见，所以事务 103 查询到数据为 `name = 菜花`

2.**`时间线来到 T6 ，数据的版本链为`：**

![markdown](picture/528559e9-dae8-4d14-b78d-a5b657c88391.png)

因为在 RC 级别下，重新生成 `Read View`，这时事务 101 已经提交，102 并未提交，所以此时 `Read View` 中活跃的事务 **`m_ids`：[102]** ，`m_low_limit_id`为：104，`m_up_limit_id`为：102，`m_creator_trx_id`为：103

- 此时最新记录的 `DB_TRX_ID` 为 102，m_up_limit_id <= 102 < m_low_limit_id，所以要在 `m_ids` 列表中查找，发现 `DB_TRX_ID` 存在列表中，那么这个记录不可见
- 根据 `DB_ROLL_PTR` 找到 `undo log` 中的上一版本记录，上一条记录的 `DB_TRX_ID` 为 101，满足 101 < m_up_limit_id，记录可见，所以在 `T6` 时间点查询到数据为 `name = 李四`，与时间 T4 查询到的结果不一致，不可重复读！

3.**`时间线来到 T9 ，数据的版本链为`：**

![markdown](picture/6f82703c-36a1-4458-90fe-d7f4edbac71a.png)

重新生成 `Read View`， 这时事务 101 和 102 都已经提交，所以 **m_ids** 为空，则 m_up_limit_id = m_low_limit_id = 104，最新版本事务 ID 为 102，满足 102 < m_low_limit_id，可见，查询结果为 `name = 赵六`

> **总结：** **在 RC 隔离级别下，事务在每次查询开始时都会生成并设置新的 Read View，所以导致不可重复读**



**在 RR 下 ReadView 生成情况**

**在可重复读级别下，只会在事务开始后第一次读取数据时生成一个 Read View（m_ids 列表）**

1. **`在 T4 情况下的版本链为`：**

![markdown](picture/0e906b95-c916-4f30-beda-9cb3e49746bf.png)

在当前执行 `select` 语句时生成一个 `Read View`，此时 **`m_ids`：[101,102]** ，`m_low_limit_id`为：104，`m_up_limit_id`为：101，`m_creator_trx_id` 为：103

此时和 RC 级别下一样：

- 最新记录的 `DB_TRX_ID` 为 101，m_up_limit_id <= 101 < m_low_limit_id，所以要在 `m_ids` 列表中查找，发现 `DB_TRX_ID` 存在列表中，那么这个记录不可见
- 根据 `DB_ROLL_PTR` 找到 `undo log` 中的上一版本记录，上一条记录的 `DB_TRX_ID` 还是 101，不可见
- 继续找上一条 `DB_TRX_ID`为 1，满足 1 < m_up_limit_id，可见，所以事务 103 查询到数据为 `name = 菜花`

1. **`时间点 T6 情况下`：**

   ![markdown](picture/79ed6142-7664-4e0b-9023-cf546586aa39.png)

   在 RR 级别下只会生成一次`Read View`，所以此时依然沿用 **`m_ids` ：[101,102]** ，`m_low_limit_id`为：104，`m_up_limit_id`为：101，`m_creator_trx_id` 为：103

- 最新记录的 `DB_TRX_ID` 为 102，m_up_limit_id <= 102 < m_low_limit_id，所以要在 `m_ids` 列表中查找，发现 `DB_TRX_ID` 存在列表中，那么这个记录不可见
- 根据 `DB_ROLL_PTR` 找到 `undo log` 中的上一版本记录，上一条记录的 `DB_TRX_ID` 为 101，不可见
- 继续根据 `DB_ROLL_PTR` 找到 `undo log` 中的上一版本记录，上一条记录的 `DB_TRX_ID` 还是 101，不可见
- 继续找上一条 `DB_TRX_ID`为 1，满足 1 < m_up_limit_id，可见，所以事务 103 查询到数据为 `name = 菜花`

1. **时间点 T9 情况下：**

![markdown](picture/cbbedbc5-0e3c-4711-aafd-7f3d68a4ed4e.png)

此时情况跟 T6 完全一样，由于已经生成了 `Read View`，此时依然沿用 **`m_ids` ：[101,102]** ，所以查询结果依然是 `name = 菜花`

> 连续多次快照读，ReadView会产生复用，没有幻读问题.
>
> 特例: 如果事务中两次快照读中出现了修改其他事务新增的当前读,  会重新生成ReadView ,  导致出现幻读.



**MVCC➕Next-key-Lock 防止幻读**

`InnoDB`存储引擎在 RR 级别下通过 `MVCC`和 `Next-key Lock` 来解决幻读问题：

**1、执行普通 `select`，此时会以 `MVCC` 快照读的方式读取数据**

在快照读的情况下，RR 隔离级别只会在事务开启后的第一次查询生成 `Read View` ，并使用至事务提交。所以在生成 `Read View` 之后其它事务所做的更新、插入记录版本对当前事务并不可见，实现了可重复读和防止快照读下的 “幻读”

**2、执行 select...for update/lock in share mode、insert、update、delete 等当前读**

在当前读下，读取的都是最新的数据，如果其它事务有插入新的记录，并且刚好在当前事务查询范围内，就会产生幻读！`InnoDB` 使用 [Next-key Lock  (opens new window)](https://dev.mysql.com/doc/refman/5.7/en/innodb-locking.html#innodb-next-key-locks) 来防止这种情况。当执行当前读时，会锁定读取到的记录的同时，锁定它们的间隙，防止其它事务在查询范围内插入数据。只要我不让你插入，就不会发生幻读







## 日志

1.日志类型

物理日志:存储了数据被修改的值

逻辑日志:存储了逻辑sql修改语句

`redo log` 它是物理日志，记录内容是“在某个数据页上做了什么修改”，属于 `InnoDB` 存储引擎，恢复速度远快于逻辑日志。让`InnoDB`存储引擎拥有了崩溃恢复能力。

`binlog` 是逻辑日志，记录内容是语句的原始逻辑，类似于“给 ID=2 这一行的 c 字段加 1”，属于`MySQL Server` 层。不管用什么存储引擎，只要发生了表数据更新，都会产生 `binlog` 日志。保证了`MySQL`集群架构的数据一致性。



### binlog(归档日志)

`MySQL`数据库的**数据备份、主备、主主、主从**都离不开`binlog`，需要依靠`binlog`来同步数据，保证数据一致性。![img](picture/01.png)

`binlog`会记录所有涉及更新数据的逻辑操作，并且是顺序写。

**记录格式**

`binlog` 日志有三种格式，可以通过`binlog_format`参数指定。

- **statement**

记录的内容是`SQL`语句原文，比如执行一条`update T set update_time=now() where id=1`.

![img](picture/02.png)

同步数据时，会执行记录的`SQL`语句，但是有个问题，`update_time=now()`这里会获取当前系统时间，直接执行会导致与原库的数据不一致。

- **row**

为了解决`statement`存在的问题，我们需要指定为`row`，记录的内容不再是简单的`SQL`语句了，还包含操作的具体数据

![img](picture/03-16379862797752.png)

`row`格式记录的内容看不到详细信息，要通过`mysqlbinlog`工具解析出来。

`update_time=now()`变成了具体的时间`update_time=1627112756247`，条件后面的@1、@2、@3 都是该行数据第 1 个~3 个字段的原始值（**假设这张表只有 3 个字段**）。

这样就能保证同步数据的一致性，通常情况下都是指定为`row`，这样可以为数据库的恢复与同步带来更好的可靠性。

但是这种格式，需要更大的容量来记录，比较占用空间，恢复与同步时会更消耗`IO`资源，影响执行速度。

- **mixed**

`statement`和`row`的折中方案,

`MySQL`会判断这条`SQL`语句是否可能引起数据不一致，如果是，就用`row`格式，否则就用`statement`格式。



**写入机制**

事务执行过程中，先把日志写到`binlog cache`，事务提交的时候，再把`binlog cache`写到`binlog`文件中。

因为一个事务的`binlog`不能被拆开，无论这个事务多大，也要确保一次性写入，所以系统会给每个线程分配一个块内存作为`binlog cache`。

我们可以通过`binlog_cache_size`参数控制单个线程 binlog cache 大小，如果存储内容超过了这个参数，就要暂存到磁盘（`Swap`）。

![img](picture/04-16379865689264.png)

- **上图的 write，是指把日志写入到文件系统的 page cache，并没有把数据持久化到磁盘，所以速度比较快**
- **上图的 fsync，才是将数据持久化到磁盘的操作**

`write`和`fsync`的时机，可以由参数`sync_binlog`控制，默认是`0`。表示每次提交事务都只`write`，由系统自行判断什么时候执行`fsync`。

![img](picture/05-16379867069256.png)

虽然性能得到提升，但是机器宕机，`page cache`里面的 binglog 会丢失。

为了安全起见，可以设置为`1`，表示每次提交事务都会执行`fsync`，就如同**binlog 日志刷盘流程**一样。

最后还有一种折中方式，可以设置为`N(N>1)`，表示每次提交事务都`write`，但累积`N`个事务后才`fsync`。

![img](picture/06.png)

在出现`IO`瓶颈的场景里，将`sync_binlog`设置成一个比较大的值，可以提升性能。

同样的，如果机器宕机，会丢失最近`N`个事务的`binlog`日志。



### redolog(重做日志)

 保证事务的**持久性**

InnoDB的修改数据的基本流程:  当我们想要修改DB上某一行数据的时候， InnoDB是把**数据页(16KB)从磁盘读取到内存的缓冲池**上进行修改。数据在内存中被修改，与磁盘中相比就存在 了差异，我们称这种有差异的数据为脏页。

**InnoDB对脏页的处理不是每次生成脏页就将脏页刷新回磁盘，这样会产生海量的IO操作，严重影响InnoDB的处理性能**。

既然脏页与磁盘中的数据存在差异，那么如果在这期间DB 出现故障就会造成数据的丢失。为了解决这个问题，redo log就应运而生了。innodb把每次更新的数据写入redo log buffer(还在innodb中,没有持久化到磁盘)

![img](picture/03.png)

>每条 redo 记录由“表空间号+数据页号+偏移量+修改数据长度+具体修改的数据”组成

**刷盘时机**

`InnoDB` 存储引擎为 `redo log` 的刷盘策略提供了 `innodb_flush_log_at_trx_commit` 参数，它支持三种策略：

- **0** ：设置为 0 的时候，表示每次事务提交时不进行刷盘操作
- **1** ：设置为 1 的时候，表示每次事务提交时都将进行刷盘操作（默认值）
- **2** ：设置为 2 的时候，表示每次事务提交时都只把 redo log buffer 内容写入 page cache

`innodb_flush_log_at_trx_commit` 参数默认为 1 ，也就是说当事务提交时会调用 `fsync` 对 redo log 进行刷盘

`InnoDB` 存储引擎有一个后台线程(master thread)，每隔`1` 秒，就会把 `redo log buffer` 中的内容写到文件系统缓存（`page cache`），然后调用 `fsync` 刷盘。(所以一个没有提交事务的`redo log`记录,也可能会刷盘)

![img](picture/05.png)

除了后台线程每秒`1`次的轮询操作，还有一种情况，当 `redo log buffer` 占用的空间即将达到 `innodb_log_buffer_size` 一半的时候，后台线程会主动刷盘。

在第一种刷盘策略0下 : 因为仅通过后台线程1s刷盘一次,如果宕机或mysql挂了(因为`redo log buffer属于innodb`),可能会丢失1s内的数据;

在第二种刷盘策略1下 :   只要事务提交成功，`redo log`记录就一定在硬盘里，不会有任何数据丢失。如果事务执行期间`MySQL`挂了或宕机，这部分日志丢了，但是事务并没有提交，所以日志丢了也不会有损失;

在第三种刷盘策略2下 : 只要事务提交成功，`redo log buffer`中的内容只写入文件系统缓存（`page cache`）。如果仅仅只是`MySQL`挂了不会有任何数据丢失，但是宕机可能会有`1`秒数据的丢失。

**日志文件组**

硬盘上存储的 `redo log` 日志文件不只一个，而是以一个**日志文件组**的形式出现的，每个的`redo`日志文件大小都是一样的。

比如可以配置为一组`4`个文件，每个文件的大小是 `1GB`，整个 `redo log` 日志文件组可以记录`4G`的内容。

它采用的是环形数组形式，从头开始写，写到末尾又回到头循环写，如下图所示。

![img](picture/10.png)

在个**日志文件组**中还有两个重要的属性，分别是 `write pos、checkpoint`

- **write pos** 是当前记录的位置，一边写一边后移
- **checkpoint** 是当前要擦除的位置，也是往后推移

每次刷盘 `redo log` 记录到**日志文件组**中，`write pos` 位置就会后移更新。

每次 `MySQL` 加载**日志文件组**恢复数据时，会清空加载过的 `redo log` 记录，并把 `checkpoint` 后移更新。

`write pos前` 和 `checkpoint后` 之间的还空着的部分可以用来写入新的 `redo log` 记录。

![img](picture/11.png)

如果 `write pos` 追上 `checkpoint` ，表示**日志文件组**满了，这时候不能再写入新的 `redo log` 记录，`MySQL` 得停下来，清空一些记录，把 `checkpoint` 推进一下。

![img](picture/12.png)

### 二阶段提交

`redo log`（重做日志）让`InnoDB`存储引擎拥有了崩溃恢复能力。

`binlog`（归档日志）保证了`MySQL`集群架构的数据一致性。

虽然它们都属于持久化的保证，但是侧重点不同。

在执行更新语句过程，会记录`redo log`与`binlog`两块日志，以基本的事务为单位，`redo log`在事务执行过程中可以不断写入，而`binlog`只有在提交事务时才写入，所以`redo log`与`binlog`的写入时机不一样。

![img](picture/01-16379872985449.png)

如果`redo log`与`binlog`两份日志之间的逻辑不一致，会出现什么问题？

我们以`update`语句为例，假设`id=2`的记录，字段`c`值是`0`，把字段`c`值更新成`1`，`SQL`语句为`update T set c=1 where id=2`。

**假设执行过程中写完`redo log`日志后，`binlog`日志写期间发生了异常**，会出现什么情况呢？

![img](picture/02-163798743991011.png)

由于`binlog`没写完就异常，这时候`binlog`里面没有对应的修改记录。因此，之后用`binlog`日志恢复数据时(从库同步)，就会少这一次更新，恢复出来的这一行`c`值是`0`，而原库因为`redo log`日志恢复，这一行`c`值是`1`，最终数据不一致。

![img](picture/03-163798748831313.png)

为了解决两份日志之间的逻辑一致问题，`InnoDB`存储引擎使用**两阶段提交**方案。

原理很简单，将`redo log`的写入拆成了两个步骤`prepare`和`commit`，这就是**两阶段提交**。

![img](picture/04-163798757064415.png)

使用**两阶段提交**后，写入`binlog`时发生异常也不会有影响，因为`MySQL`根据`redo log`日志恢复数据时，发现`redo log`还处于`prepare`阶段，并且没有对应`binlog`日志，就会回滚该事务。![img](picture/05-163798761159717.png)

**假如`redo log`设置`commit`阶段发生异常**，那会不会回滚事务呢？

并不会回滚事务，它会执行上图框住的逻辑，虽然`redo log`是处于`prepare`阶段，但是能通过事务`id`找到对应的`binlog`日志，所以`MySQL`认为是完整的，就会提交事务恢复数据。



### undolog(回滚日志)

保证事务的**原子性**。

所有事务进行的修改都会先记录到**回滚日志**中，然后再执行相关的操作。如果执行过程中遇到异常的话，我们直接利用 **回滚日志** 中的信息将数据回滚到修改之前的样子即可！并且，回滚日志会先于数据持久化到磁盘上。这样就保证了即使遇到数据库突然宕机等情况，当用户再次启动数据库的时候，数据库还能够通过查询回滚日志来回滚将之前未完成的事务。

另外，`MVCC` 的实现依赖于：**隐藏字段、Read View、undo log**。

## 慢查询日志

为了定位sql的性能瓶颈, 我们需要开启mysql的慢查询日志, 把超过指定时间的sql语句, 单独记录下来, 方便以后分析和定位问题.

sql中慢查询阈值为long_query_time=10s, 即当sql执行时间大于10s就会被记录到慢sql日志中, 一般建议缩小到1s.

开启慢查询可通过 `set命令` 或 `my.cnf配置文件` 来设置下列属性:

```
show_query_log = ON
show_query_log_file = /usr/local/mysql/log/show.log
long_query_time = 2
```

## ngram全文检索

对于需要模糊查询的情况, 通常使用like查询会导致索引失效而全文检索, 进而可以加入ElasticSearch(Canal异构同步订阅binlog).

但引入ES除了需要更高的成本外, 数据一致性需要保证, 高可用和维护工作需要保证.

MySQL折中方案:从 MySQL 5.7.6 开始，MySQL内置了[ngram全文解析器](https://www.cnblogs.com/miracle-luna/p/11147859.html)

允许对短文本进行全文检索查询，以替代like关键字 ( 分词效果不太行 )

>  对于复杂业务场景的全文检索查询，还是要用ES



# Redis

像 MySQL 这类的数据库的 QPS 大概都在 1w 左右（4 核 8g）, Redis虽然是单进程单线程模型，但是读写性能非常优异，单机可支持10wQPS，原因主要有以下几点：

1. 纯内存操作，避免了与磁盘的交互
2. 用全局hash table作为键空间，查找任意的key只需O(1)
3. 单线程，天生的队列模式，避免了因多线程竞争而导致的上下文切换和抢锁的开销
4. 事件机制，Redis服务器将所有处理的任务分为两类事件，一类是采用I/O多路复用处理客户端请求的网络事件；一类是处理定时任务的时间事件，包括更新统计信息、清理过期键、持久化、主从同步等；

> 单线程事件机制也是有缺陷的，所有的事件都是串行执行，一旦某个事件比较重就会阻塞其它事件，从而导致整个系统的吞吐率下降。比如某个客户端执行了一个比较重的lua函数、或者使用了诸如keys*、zrange(0,-1)、hgetall等全集合扫描的操作，又或者删除的过期键是个big key，又或者使用了较多内存的redis实例进行bgsave时，都会导致服务器一定程度的阻塞，一般伴随会有相应的慢日志。所以我们在实际使用redis的过程中，必须要给每一次的操作分配合理的时间片。

> redis客户端通过对管道中的指令序列改变读写顺序减少I/O来回次数, 大幅节省了IO时间. 具体见《Redis深度历险》- 管道

**分布式缓存的选择**

相比Memcached , redis优点:

Redis 支持更丰富的数据类型;

Redis 支持数据的持久化，可以将内存中的数据保持在磁盘中，重启的时候可以再次加载进行使用(也可以容灾),而 Memecache 把数据全部存在内存之中。

Redis 在服务器内存使用完之后，可以将不用的数据放到磁盘上。Memcached 在服务器内存使用完之后，就会直接报异常。

Memcached 没有原生的集群模式，需要依靠客户端来实现往集群中分片写入数据；但是 Redis 目前是原生支持 cluster 模式的。

Memcached 是多线程，非阻塞 IO 复用的网络模型；Redis 使用单线程的多路 IO 复用模型。 （Redis 6.0 引入了多线程 IO ）

Redis 支持发布订阅模型、Lua 脚本、事务等功能，而 Memcached 不支持。并且，Redis 支持更多的编程语言。

Memcached 过期数据的删除策略只用了惰性删除，而 Redis 同时使用了惰性删除与定期删除。

## 数据类型

RedisObject通过encoding属性设置对象所使用的编码，type属性设置对象类型,  而不是为特定类型的对象关联一种固定的编码， 极大地提升了 Redis 的灵活性和效率.

执行命令前, Redis会先检查该key的值对象是否为执行命令锁需的类型, 如果是, 还会根据该key的值对象的编码来选择正确的命令实现(命令多态) 

```
struct RedisObject {
 int4 type; // 4bits
 int4 encoding; // 4bits
 int24 lru; // 24bits
 int32 refcount; // 4bytes 
 void *ptr; // 8bytes，64-bit system
} robj;
```

**1.String**

> Redis键也是SDS

虽然 Redis 是用 C 语言写的，但并没有使用 C 的字符串表示，而是自己构建了一种 **简单动态字符串SDS**。

其数据结构为带容量和长度的**字节**数组(可以存储二进制数据)的结构体.

SDS 遵循 C 字符串以空字符结尾的惯例是为了兼容部分C字符串函数, 但实际上SDS是使用len属性而不是空字符来判断字符串是否结束. 

> C语言通过遍历到结尾符获取字符串长度使用strlen函数, 复杂度O(n), 这对于单线程redis无法接受.

SDS是可以修改的字符串，内部结构实现上类似于 Java 的 ArrayList，采用预分配冗余空间的方式来减少内存的频繁分配, 避免C字符串API存在的溢出问题(SDS追加时先判断free空间是否足够)

如果是整数, 则使用int编码; 当Redis字符串的长度小于等于44时, 使用embstr编码; 当字符串长度超过44时, 使用raw编码存储. 因为内存分配器最多分配64字节空间, RedisObject固定占用16字节, SDS固定属性占用3字节, 字符串结尾符占用1字节, 还剩下最多64-20=44字节, 超过44则为大字符串, 需要采用raw形式.

embstr与raw的区别为: embstr将 RedisObject 和 SDS 对 象连续存在一起，使用 malloc 方法一次分配。而 raw 存储形式不一样，它需要两次 malloc，两个对象结构在内存地址上一般是不连续的。

> 对于浮点数, redis会转换为字符串存储, 但计算时还是对浮点数操作, 如INCRBYFLOAT

> int和embstr编码在特定条件下回转换为raw, 另外embstr为只读不支持修改, 如果要修改embstr编码的字符串, 需要转换为raw

每次追加字符串前通过free属性判断是否需要扩容, 杜绝c字符串的缓冲区溢出问题.

扩容策略为: 在字符串长度小于1M之前, 扩容空间采用加倍策略, 即保留100%的冗余空间; 当超过1M之后, 为避免加倍后的冗余空间过大导致浪费, 每次扩容只会多分配1M的空间. (扩容必定N次变为了最多N次)

应用场景： 一般常用在需要计数的场景，比如用户的访问次数、热点文章的点赞转发数量等等。

相关命令: `set,get,strlen,exists,decr,incr,setex,setnx`

> Redis 中除了字符串类型有自己独有设置过期时间的命令 `setex` 外，其他方法都需要依靠 `expire` 命令来设置过期时间 。另外， `persist` 命令可以移除一个键的过期时间。

> 可通过object encoding key 查看该value的数据结构

**2.List**

类采用ziplist或linkedlist(双端队列)实现, 为优化存储和减少内存碎片,  后续版本被quicklist替换(ziplist和linkedlist的结合体)). 

linkedlist为一个双端队列, 默认情况下, 当list对象中所有字符串元素的长度都小于64字节且元素个数小于512时, 会采用ziplist编码, 可通过配置文件改变上限值.

ziplist有一个zltail属性记录尾结点偏移量, 每个节点通过previous_entry_length记录前一个节点的长度结合zltail来支持反向遍历, 每个节点通过encoding记录节点的content属性所保存的数据类型(字节数组或整数类型)以及长度.

> 因为前一节点长度小于254时previous_entry_length需要1字节记录, 大于等于254时previous_entry_length需要5字节记录, 所以ziplist在添加和删除元素时若存在多个连续长度介于250-253字节的节点将会导致连锁更新

应用场景:  发布与订阅或者说**异步消息队列**, 比如微博、朋友圈和公众号消息流(lpush)、慢查询。

> 将需要延后处理的任务结构体序列化成字符串塞进 Redis 的列表，另一个线程从这个列表中轮询数据进行处理。

相关命令: `rpush,lpop,lpush,rpop,lrange,llen,brpop(阻塞队列)`

其他慢操作命令:` lindex(通过索引获取值),ltrim(范围截取) `

> list存在索引, 正向索引从0开始, 反向索引从-1开始

**3.Hash**

hash 类似于 JDK1.8 前的 HashMap(数组 + 链表)。包括hashtable(字典)和ziplist两种实现.

字典dict采用渐进式rehash, 包含两个hash表指针,  在 rehash 的同时，保留新旧两个 hash 结构，查询时会同时查询两个 hash 结构，然后在后续的定时任务中以及 hash 的子指令中，循序渐进地将旧 hash 的内容 一点点迁移到新的 hash 结构中。

当hash对象中键值对的键和值字符串长度都小于64字节且键值对数量小于512个时, redis会采用ziplist实现hash.可通过配置文件改变上限值. 

ziplist插入键值对时先推入key到ziplist末尾, 再推入value到ziplist末尾. 因此同一键值对紧挨存储, 后添加的键值对放在尾部/

应用场景: 系统中对象数据的存储, 如电商购物车(用户id为key, 商品id为field, 商品数量为value)。

相关命令:`hset,hmset,hexists,hget,hgetall,hkeys,hvals`

> 同string一样, hash 结构中的单个子 key 也可以进行计数，它对应的指令是 hincrby， 和 incr 使用基本一样。

> 如何权衡string与hash?

**4.set**

类似 Java 中的 `HashSet` 。通过intse(如果只有整数且元素不超过512个)t或hashtable(值为null)实现.

intset可以通过判断新加入的元素长度动态扩展每个元素的空间: content数组为int8_t类型, encoding支持int16_t,int32_t和int64_t编码改变每个元素在content数组中的字节数, 只可升级不可降级.

应用场景:需要存放的数据不能重复以及需要获取多个数据源交集(将用户关注存到set,利用sinterstore实现共同关注)和并集等场景. 比如抽奖(srandmember key [count]/spop key [count] 随机抽取count个)、 点赞(可通过交集显示共同好友).

相关命令:`sadd,spop,smembers,sismember,scard,sinterstore,sunion`

**5.sorted set** 

增加了一个权重参数 score，元素按 score 排列，还可以通过 score 的范围来获取元素的列表。有ziplist和skiplist两种实现.

> 在zset元素个数小于128个且元素长度小于64字节时使用ziplist编码, 可修改

ziplist对每个zset元素紧挨存储, 元素存储在前面, 权重存储在后面, 集合元素按权重从小到大排序.

skiplist中每个节点通过链表连接, 并包含一个后退指针指向前面一个节点以支持反向遍历, 但不能跳跃.  每一层的跳表节点包含一个跨度值记录两个节点之间的记录, 将查询某个节点时的对沿途跨度进行累加用于score rank实现. 

另外, 为了提高查询效率, skiplist增加了从元素到分值的映射字典.

相关命令:`zadd,zcard,zscore,zrange,zrangebyscore,zrem`

应用场景:需要对数据根据某个权重进行排序的场景。比如在直播系统中，实时排行信息包含直播间在线用户列表，各种礼物排行榜，弹幕消息（可以理解为按消息维度的消息排行榜）等信息.

```
> zadd myZset 3.0 value1 2.0 value2 1.0 value3  #插入值和其权重
(integer) 3
> zcard myZset 									#查看元素数量
(integer) 3
> zscore myZset value1 							#查看某个值的权重
"3"
> zrange  myZset 0 -1         #顺序输出某个范围区间的元素，0 -1 表示输出所有元素
1) "value3"
2) "value2"
3) "value1"
> zrevrange  myZset 0 1 						#逆序输出
1) "value1"
2) "value2"
```

**6.bitmap**

 bitmap 存储的是连续的二进制数字（0 和 1），通过 bitmap, 只需要一个 bit 位来表示某个元素对应的值或者状态，key 就是对应元素本身 。

相关命令:`setbit,getbit,bitcount,bitop` 

```
# bitop 对一个或多个保存二进制位的字符串 key 进行位元操作，并将结果保存到 destkey 上。
# 支持 AND 、 OR 、 NOT 、 XOR 这四种操作中的任意一种参数
BITOP operation destkey key [key ...]
```

应用场景: 适合需要保存状态信息（比如是否签到、是否登录...）并需要进一步对这些信息进行分析的场景。比如用户签到情况、活跃用户情况、用户行为统计（比如是否点赞过某个视频）。

```
# setbit 会返回之前位的值（默认是 0）
> setbit mykey 7 1
(integer) 0
> bitcount mykey  #统计被被设置为 1 的位的数量
(integer) 1

#使用场景一：用户行为分析 很多网站为了分析你的喜好，需要研究你点赞过的内容。
# 记录你喜欢过 001 号小姐姐
127.0.0.1:6379> setbit beauty_girl_001 uid 1

#使用场景二：统计活跃用户
# 使用时间作为 redis key，然后用户ID为map key，如果当日活跃过就设1(暂且约定，统计时间内只要有一天在线就称为活跃)
127.0.0.1:6379> setbit 20210308 1 1
(integer) 0
127.0.0.1:6379> setbit 20210308 2 1
(integer) 0
127.0.0.1:6379> setbit 20210309 1 1
(integer) 0
//统计两天都活跃的人
127.0.0.1:6379> bitop and desk1 20210308 20210309
(integer) 1
127.0.0.1:6379> bitcount desk1
(integer) 1
//统计两天内活跃了的人
127.0.0.1:6379> bitop or desk2 20210308 20210309
(integer) 1
127.0.0.1:6379> bitcount desk2
(integer) 2
```

## 事务

Redis 事务其实是一组命令的集合, 缓存在服务器的一个事务队列中, 收到exec指令才执行整个事务队列.

使用 `MULTI` 命令后可以输入多个命令。Redis 不会立即执行这些命令，而是将它们放到队列，当调用了 `EXEC` 

可以通过 `DISCARD`命令取消一个事务，它会清空事务队列中保存的所有命令。

`WATCH` 命令用于监听指定的键，当调用 `EXEC` 命令执行事务时，如果一个被 `WATCH` 命令监视的键被修改的话，整个事务都不会执行，直接返回失败。**+重试 = 乐观锁**

你可以将 Redis 中的事务就理解为 ：**Redis 事务提供了一种将多个命令请求打包的功能。然后，再按顺序执行打包的所有命令，并且不会被中途打断。**

Redis 事务是不支持 roll back 的，因而不满足原子性的（而且不满足持久性）, 比如事务中有命令报错并不会回滚而是会继续执行下去 。 Redis事务仅满足隔离性(串行化)

## 持久化

**Redis 的持久化方式有快照（snapshotting，RDB），和只追加文件（append-only file, AOF）**。

### RDB

RDB快照持久化是 Redis 默认采用的持久化方式，在 Redis.conf 配置文件中默认有此下配置：

```
save 900 1           #在900秒(15分钟)之后，如果至少有1个key发生变化，Redis就会自动触发BGSAVE命令创建快照。
save 300 10          #在300秒(5分钟)之后，如果至少有10个key发生变化，Redis就会自动触发BGSAVE命令创建快照。
save 60 10000        #在60秒(1分钟)之后，如果至少有10000个key发生变化，Redis就会自动触发BGSAVE命令创建快照
```

**触发机制**

1、save的规则满足的情况下

2、执行save, flushall 命令

3、退出redis

**恢复rdb文件** 

1、只需要将rdb文件放在我们redis启动目录就可以，redis启动的时候会自动检查dump.rdb 恢复其中的数据(阻塞载入)

**特点**

1、适合大规模的数据恢复 

2、对数据的完整性要不高. 如果redis意外宕机了，只能保留到最后一个的save的地方, 可能会丢失大量数据。



通过创建快照来获得存储在内存里面的数据在某个时间点上的副本(全量), 是内存数据的**二进制**序列化形式，在存储上非常**紧凑**。

Redis会单独创建（fork）一个子进程来进行持久化，会先将数据写入到一个临时文件中，待持久化过程都结束了，再用这个临时文件替换上次持久化好的文件。整个过程中，主进程是不进行任何IO操作的。(写时复制思想?)

rdb保存的文件是dump.rdb, 一个完整RDB文件包含五个部分:

1.REDIS: 文件的开头, 是常量, 用于标识RDB文件, 5字节, 保存二进制的"REDIS"五个字符 

2.db_version: 文件的版本号, 4字节, 是一个字符串表示的整数

3.databases: 各个数据库中的键值对数据. 如果各个数据库没有内容, 则为空.

> databases部分中每个非空数据库又分为三个部分:
>
> 1.SELECTDB: 常量, 表示接下来的内容为数据库号码, 1字节
>
> 2.db_number: 数据库号码, 1/2/5字节, 服务器读入该部分后, 会调用SELECT命令切换数据库
>
> 3.key_value_pairs: 数据库中所有键值对数据, 如果key存在过期时间, 也会一同保存.
>
> 在不带过期时间下key_value_pairs由三部分组成:
>
> TYPE, key和value, 其中TYPE表示value的类型, 1字节. 
>
> 服务器读入rdb键值对时会根据TYPE来解析value.
>
> 如果带过期时间, 会在头部增加EXPIREIMTE_MS和ms. 前者标识接下来的内容为过期时间, 1字节; ms为8字节带符号整数, 毫秒UNIX时间戳, 为键值对的过期时间.
>
> 具体的不同类型value结构见《Redis设计与实现》

4.EOF: 标识文件正文内容(所有数据库的键值对)的结束, 1字节.

5.check_sum: 前面四部分计算后的校验和, 是8字节无符号整数. 务器在载入 RDB 文件时， 会将载入数据所计算出的校验和与 `check_sum` 所记录的校验和进行对比， 以此来检查 RDB 文件是否有出错或者损坏的情况出现。





### AOF

AOF持久化的实时性更好，因此已成为主流的持久化方案。默认情况下 Redis 没有开启 AOF（append only file）方式的持久化，可以通过 appendonly 参数开启：

```
appendonly yes
```

`AOF`以日志的形式来记录每个写操作，将Redis执行过的所有指令记录下来（读操作不记录），只许追加文件，但不可以改写文件。

aof保存的是 appendonly.aof 文件

如果一个事件循环中执行了写命令，Redis 就会将该命令写入到**内存缓存** `server.aof_buf` 中(redis单线程串行写入)，然后再根据 `appendfsync` 配置来决定何时将其同步到硬盘中的 AOF 文件(fsync命令, 独立线程**异步**刷回, 真正的磁盘IO, 耗时!)。

> Redis服务器进程就是一个事件循环, 伪代码如下:
>
> ```
> def eventLoop():
>     while True:
>         # 处理文件事件，接收命令请求以及发送命令回复
>         # 处理命令请求时可能会有新内容被追加到 aof_buf 缓冲区中
>         processFileEvents()
>         # 处理时间事件
>         processTimeEvents()
>         # 考虑是否要将 aof_buf 中的内容写入和保存到 AOF 文件里面
>         flushAppendOnlyFile()
> ```
>
> 

在 Redis 的配置文件中存在三种不同的 AOF 持久化方式，它们分别是：

```text
appendfsync always    #每个事件循环都会同步AOF文件,这样会严重降低Redis的速度
appendfsync everysec  #每个事件循环判断距离上次同步aof文件是否超过1s, 如果是, 则同步AOF文件
appendfsync no        #让操作系统决定何时进行同步
```

AOF 日志在长期的运行过程中会变的无比庞大，数据库**重启时(空的redis实例)需要加载 AOF 日志进行指令重放(顺序执行所有指令)**，这个时间就会无比漫长。 所以需要定期进行 AOF 重写，给 AOF 日志进行瘦身。

> AOF重写(瘦身):  bgrewriteaof 指令
>
> 其原理为**开辟一个子进程**对内存进行遍历转换成一系列 Redis 的操作指令，序列化到一个新的 AOF 日志文件中。 序列化完毕后再将操作期间发生的增量 AOF 日志追加到这个新的 AOF 日志文件中，追加完毕后就立即替代旧的 AOF 日志文件了，瘦身工作就完成了。

**Redis4.0混合持久化**

因为rdb重放会丢失大量数据, aof重放性能较慢, 所以引入了混合持久化, 将rdb和aof文件存在一起, 此时的aof日志不再是全量的追加日志, 而是rdb持久化开始到结束这段时间的追加日志(通常这部分aof日志很小).

当Redis 重启的时候先加载 rdb 的内容，再重放增量 AOF 日志就可以完全替代之前的 AOF 全量文件重放，重启效率因此大幅得到提升。

## 过期删除

**过期时间保存**

Redis 通过一个叫做过期字典（可以看作是 hash 表）来保存数据过期的时间。过期字典的键指向 Redis 数据库中的某个 key(键)，过期字典的值是一个 long long 类型的整数，这个整数保存了 key 所指向的数据库键的过期时间（毫秒精度的 UNIX 时间戳）。![redis过期字典](picture/redis过期时间.7fb881f9.png)

过期字典是存储在 redisDb 这个结构里的：

```c
typedef struct redisDb {
    dict *dict;     //数据库键空间,保存着数据库中所有键值对
    dict *expires   //过期字典,保存着键的过期时间
    ...
} redisDb;
```

**过期数据删除策略**

**惰性删除** ：只会在取出 key 的时候才对数据进行过期检查。这样对 CPU 最友好，但是可能会造成太多过期 key 没有被删除。

**定期删除** ： 每隔一段时间抽取一批 key 执行删除过期 key 操作。并且，Redis 底层会通过限制删除操作执行的时长和频率来减少删除操作对 CPU 时间的影响。

定期删除对内存更加友好，惰性删除对 CPU 更加友好。两者各有千秋，所以 Redis 采用的是 **定期删除+惰性/懒汉式删除** 。

> redis的删除操作采用的是异步内存回收, 主线程将key从字典中摘除后, 将这个key的内存回收操作包装成一个任务塞进异步队列中后台清理.

但是，仅仅通过给 key 设置过期时间还是有问题的。因为还是可能存在定期删除和惰性删除漏掉了很多过期 key 的情况。这样就导致大量过期 key 堆积在内存里，然后就 Out of memory 了。 所以有了**内存淘汰机制**

> 如果同时有大量key过期的话，极可能导致主线程阻塞。一般可以通过做散列来优化处理。

## 内存淘汰

内存淘汰为当redis执行写操作时, 如果内存超出maxmemory就会执行一次内存淘汰算法.

https://redis.io/topics/lru-cache

Redis 提供 6 种数据淘汰策略：

1. **volatile-lru（least recently used）**：从已设置过期时间的数据集（server.db[i].expires）中挑选最近最少使用的数据淘汰
2. **volatile-ttl**：从已设置过期时间的数据集（server.db[i].expires）中挑选将要过期的数据淘汰
3. **volatile-random**：从已设置过期时间的数据集（server.db[i].expires）中任意选择数据淘汰
4. **allkeys-lru（least recently used）**：当内存不足以容纳新写入数据时，在键空间中，移除最近最少使用的 key（这个是最常用的）
5. **allkeys-random**：从数据集（server.db[i].dict）中任意选择数据淘汰
6. **no-eviction**：禁止驱逐数据，也就是说当内存不足以容纳新写入数据时，新写入操作会报错。这个应该没人使用吧！

4.0 版本后增加以下两种：

1. **volatile-lfu（least frequently used）**：从已设置过期时间的数据集（server.db[i].expires）中挑选最不经常使用的数据淘汰
2. **allkeys-lfu（least frequently used）**：当内存不足以容纳新写入数据时，在键空间中，移除最不经常使用的 key

> 如果redis服务器内存分配不足, 可能会导致热点数据覆盖度不足, 频繁触发LRU清除, 导致缓存穿透, 这样通常可通过扩大redis服务内存或更换key存储结构实现以提高QPS

### LRU

近似LRU:

在严格LRU下,  假设Redis当前有50W规模的key，先通过Keys 遍历获得所有Key，然后比对出空闲时间最长的某个key，最后执行淘汰,  非常耗时

在Redis中它采用了近似LRU的实现，它随机采样5个(可设置)Key，淘汰掉其中空闲时间最长的那个。近似LRU实现起来更简单、成本更低，在效果上接近严格LRU。它的缺点是存在一定的几率淘汰掉最近被访问的Key

> 样本数越多效果越接近严格LRU, 代价为一些额外的CPU开销
>
> Redis3.0使用10样本非常接近严格LRU

RedisObject的lru属性记录了对象最后一次被命令程序访问的时间, OBJECT IDLETIME 命令可以打印出给定键的空转时长， 空转时长就是通过将当前时间减去键的值对象的 `lru` 时间.  

> OBJECT IDLETIME 命令访问对象时不会修改lru属性.

**LRU简单代码**

```
//注意, Node需要保存key, 用于删除时
class Node {
	public int key, val;
	public Node next, prev;
	public Node(int k, int v) {
		this.key = k;
		this.val = v;
	}
}

class LRUCache {
	private HashMap<Integer, Node> map;
	private Queue<Node> cache;
	private int cap;
	public LRUCache(int capacity) {
		this.cap = capacity;
		map = new HashMap<>();
		cache = new LinkedList<>();
	}
	public int get(int key) {
		if (!map.containsKey(key))
			return -1;
		int val = map.get(key).val;
		// 利⽤ put ⽅法把该数据提前
		put(key, val);
		return val;
	}
	public void put(int key, int val) {
		// 先把新节点 x 做出来
		Node x = new Node(key, val);
		if (map.containsKey(key)) {
			// 删除旧的节点，新的插到头部
			cache.remove(map.get(key));
			cache.addFirst(x);
			// 更新 map 中对应的数据
			map.put(key, x);
		} else {
			if (cap == cache.size()) {
				// 删除链表最后⼀个数据
				Node last = cache.removeLast();
				map.remove(last.key);
			}
			// 直接添加到头部
			cache.addFirst(x);
			map.put(key, x);
		}
	}
}
```



## 缓存穿透

指请求缓存和数据库中都没有的数据

而**缓存击穿** 为在高并发下, 大量请求的 key 根本不存在于缓存中，导致请求直接到了数据库上，根本没有经过缓存这一层。

解决方案:

**1）缓存null值**

如果缓存和数据库都查不到某个 key 的数据就写一个到 Redis 中去并设置过期时间.

这种方式可以解决请求的 key 变化不频繁的情况，如果黑客恶意攻击，每次构建不同的请求 key，会导致 Redis 中缓存大量无效的 key 。很明显，这种方案并不能从根本上解决此问题。如果非要用这种方式来解决穿透问题的话，尽量将无效的 key 的过期时间设置短一点比如 1 分钟。

**2）布隆过滤器**

把所有可能存在的请求的值都存放在布隆过滤器中，当用户请求过来，先判断用户发来的请求的值是否存在于布隆过滤器中。不存在的话，直接返回请求参数错误信息给客户端，存在的话才会走后续的缓存至数据库流程。

### 布隆过滤器

**当一个元素加入布隆过滤器中的时候，会进行如下操作：**

1. 使用布隆过滤器中的哈希函数对元素值进行计算，得到哈希值（有几个哈希函数得到几个哈希值）。
2. 根据得到的哈希值，在位数组中把对应下标的值置为 1。

**当我们需要判断一个元素是否存在于布隆过滤器的时候，会进行如下操作：**

1. 对给定元素再次进行相同的哈希计算；
2. 得到值之后判断位数组中的每个元素是否都为 1，如果值都为 1，那么说明这个值在布隆过滤器中，如果存在一个值不为 1，说明该元素不在布隆过滤器中。

> 布隆过滤器说某个元素存在，小概率会误判。布隆过滤器说某个元素不在，那么这个元素一定不在

> 布隆过滤器只会误判没有遇见过的元素.

**应用场景**

判断给定数据是否存在：比如判断一个数字是否存在于包含大量数字的数字集中（数字集很大，5 亿以上！）、 防止缓存穿透（判断请求的数据是否有效避免直接绕过缓存请求数据库）、邮箱的垃圾邮件过滤、黑名单、内容推荐的历史记录(历史记录允许误判, 使用没误判的不存在)功能、爬虫系统记录已爬过的URL(错过少量URL)等等。

去重：比如爬给定网址的时候对已经爬取过的 URL 去重。

布隆过滤器在项目缓存中防止缓存穿透的使用流程:

![image-20211204225755493](picture/image-20211204225755493.png)

**问题**:假如布隆过滤器初始化后的某个商品被删除了怎么办?

布隆过滤器因为某一位二进制可能被多个编号Hash引用,因此布隆过滤器无法直接处理删除数据的情况.

解决办法:

1.定时异步重建布隆过滤器.

2.计数Bloom Filter(记录该位被几个数据引用).



**实现**

1.BitSet

2.Guava BloomFilter

```
// 创建布隆过滤器对象,最多存放 1500 个整数的布隆过滤器，并且我们可以容忍误判的概率为百分之
BloomFilter<Integer> filter = BloomFilter.create(
    Funnels.integerFunnel(),
    1500,
    0.01);
// 判断指定元素是否存在
System.out.println(filter.mightContain(2));
// 将元素添加进布隆过滤器
filter.put(1);
//当 mightContain() 方法返回 true 时，我们可以 99％确定该元素在过滤器中，当过滤器返回 false 时，我们可以 100％确定该元素不存在于过滤器中。
```

3.Rdission

```
Config config = new Config();
config.useSingleServer().setAddress("redis://127.0.0.1:6379");
//构造Redisson
RedissonClient redisson = Redisson.create(config);
RBloomFilter<String> bloomFilter = redisson.getBloomFilter("bloom");
//初始化布隆过滤器：预计元素为1000000L,误判率为1%
bloomFilter.tryInit(1000000L,0.01);
bloomFilter.add("1"); //增加数据
//判断指定编号是否在布隆过滤器中
System.out.println(bloomFilter.contains("1")); //输出true
System.out.println(bloomFilter.contains("8888"));//输出false
```

4.RedisBloom

Redis v4.0 之后有了 Module（模块/插件） 功能，Redis Modules 让 Redis 可以使用外部模块扩展其功能 。布隆过滤器就是其中的 Module。布隆过滤 器作为一个插件加载到 Redis Server 中.

**命令**

1. **`BF.ADD`**：将元素添加到布隆过滤器中，如果该过滤器尚不存在，则创建该过滤器。格式：`BF.ADD {key} {item}`。
2. **`BF.MADD`** : 将一个或多个元素添加到“布隆过滤器”中，并创建一个尚不存在的过滤器。该命令的操作方式`BF.ADD`与之相同，只不过它允许多个输入并返回多个值。格式：`BF.MADD {key} {item} [item ...]` 。
3. **`BF.EXISTS`** : 确定元素是否在布隆过滤器中存在。格式：`BF.EXISTS {key} {item}`。
4. **`BF.MEXISTS`** ： 确定一个或者多个元素是否在布隆过滤器中存在格式：`BF.MEXISTS {key} {item} [item ...]`。

另外， `BF. RESERVE` 命令需要单独介绍一下：

这个命令的格式如下：

`BF. RESERVE {key} {error_rate} {capacity} [EXPANSION expansion]` 。

1. key：布隆过滤器的名称
2. error_rate : 期望的误报率。该值必须介于 0 到 1 之间。例如，对于期望的误报率 0.1％（1000 中为 1），error_rate 应该设置为 0.001。期望的错误率越低内存消耗越大，并且每个操作的 CPU 使用率越高。
3. capacity: 过滤器的容量。当实际存储的元素个数超过这个值之后，性能将开始下降, 误判率上升。

可选参数：

- expansion：如果创建了一个新的子过滤器，则其大小将是当前过滤器的大小乘以`expansion`。默认扩展值为 2。这意味着每个后续子过滤器将是前一个子过滤器的两倍。

## 缓存击穿

缓存击穿，是指一个key非常热点，在不停的扛着大并发，大并发集中对这一个点进行访问，当这个key在失效的瞬间，持续的大并发就穿破缓存，直接请求数据库，就像在一个屏障上凿开了一个洞。

为了防止缓存击穿, 可通过互斥锁, 在缓存不存在时, 阻塞其他线程只让一个线程去读取数据库;

### 缓存雪崩

**缓存在同一时间大面积的失效，后面的请求都直接落到了数据库上，造成数据库短时间内承受大量请求。** 

比如缓存服务宕机不可用, 可以	

1. 采用 Redis 集群，避免单机出现问题整个缓存服务都没办法使用。
2. 限流，避免同时处理大量的请求。

**有一些被大量访问数据（热点缓存）在某一时刻大面积失效，导致对应的请求直接落到了数据库上.**

比如秒杀开始 12 个小时之前，我们统一存放了一批商品到 Redis 中，设置的缓存过期时间也是 12 个小时，那么秒杀开始的时候，这些秒杀的商品的缓存直接就失效了。可以

1. 设置不同的失效时间比如随机设置缓存的失效时间。
2. 缓存永不失效



## 高可用

Redis的高可用，主要通过主从复制机制以及Sentinel集群来实现。

1. 主从复制 分为两个阶段，首先，当从服务器发起**SYNC**(异步, 存在不一致)命令后，主服务器会生成最新的RDB文件发送给从服务器，并使用一个缓冲区来记录从此刻开始主服务器执行的所有写命令；待RDB文件传输完之后，再将该缓冲区的数据再发送给从服务器，这样就完成了复制。旧的Redis版本有个缺陷是，如果在第二个阶段发生失败，需要从第一个阶段重新开始同步，而这个阶段的操作会消耗大量的CPU、内存和磁盘I/O以及网络带宽资源，太过耗费资源。所以从2.8版本开始，实现了部分重同步，通过主从服务器各维护一个复制偏移量来实现。
2. Sentinel 由一个或多个Sentinel实例组成的哨兵系统，可以监视任意多个主从服务器，并完成Failover的操作。Sentinal其实是一个运行在特殊模式下的Redis服务器，运行期间，会与各服务器建立网络连接，以检测服务器的状态；同时会与其它Sentinel服务器创建连接，完成信息交换，比如发现某个主服务器心跳异常时，会互相询问心跳结果，当超过一定数量时即可判定为客观下线；一旦主服务器被判定为客观下线状态，那么Sentinel集群会通过raft协议选举，选出一个Leader来执行Failover。
3. Failover 一般来说，会先选出优先级最高的从服务器，然后再从中选出复制偏移量最大的实例，作为新的主服务器；最后将其它从和旧的主都切换为新主的从。



当从服务器有2个或者多个时，Redis的主从架构可以有两种形式。一种是，所有的从服务器直接挂在主服务器上，这种模式的优点是，所有从服务器复制的延迟相对较低，而缺点在于加大了主服务器的复制压力；另一种形式，是采用级联的方式，S1从M复制，S2从S1复制，以此类推，这种模式的优点是，将主服务器的复制压力分摊到多个服务器上，而缺点在于越处于级联下游的从实例，复制延迟就越大。



从主从复制模式可以看出，Redis的数据只能保证最终一致，不能保证强一致性。

### 主从复制和sentinel

解决单点故障, 哨兵是监控

> 目前 Cluster可以完全用来替代Sentinel

**在redis主从同步方式下**

![image-20211205153409550](picture/image-20211205153409550.png)

当主节点挂了,需要考虑可用性问题. 引入Sentinel (哨兵)

![image-20211205153735959](picture/image-20211205153735959.png)

**故障发现**:客观下线即认为其真正宕机.

![image-20211205153935302](picture/image-20211205153935302.png)

> 通常设置为有N/2+1 个sentinel确认master主观下线时就标注master为客观下线

**故障转移**:先剔除无效slave, 再选出slave

![image-20211205154614692](picture/image-20211205154614692.png)

当选举出slave后 , sentinel leader控制改变master.

![image-20211205155353562](picture/image-20211205155353562.png)

> 为了防止应用实例重启以后之前的工作内容消失不见,所以所有redis节点在进行完主从迁移之后会自动对本地配置文件进行更新. 从而在重启后工作内容信息不会丢失.

因为sentinel相当重要,所以也需要有相应的高可用解决方案:

sentinel之间通过redis master 感知其他sentinel的存在

![image-20211205155851734](picture/image-20211205155851734.png)

>  哨兵模式每个节点保留全量数据,且数据保持一致, 仅为单节点redis提供高可用 , 存在性能瓶颈

## Scan

keys命令用来查找key, 并支持正则, 但是其存在缺点:

1. 无法进行如limit、offset的限制, 不利于浏览;
2. 遍历算法, 复杂度O(n).
3. 阻塞redis server.

为了解决此问题, redis 2.8引入了scan指令, 采用游标的方式分步进行而不会阻塞线程, 并可限制最大返回数量.

如要在10000个key中查找key99开头的key:

```
# 第一次遍历cursor从0开始 最大返回1000条 将返回结果的第一个整数作为下一次遍历的cursor, 直到返回0
127.0.0.1:6379> scan 0 match key99* count 1000
1) "13976"
2) 	1) "key9911"
 	2) "key9974"
 	3) "key9994"
 	4) "key9910"
 	...
127.0.0.1:6379> scan 13976 match key99* count 1000
1) "1996"
2) 	1) "key9982"
 	2) "key9997"
 	3) "key9963"
 	4) "key996"
 	...
127.0.0.1:6379> scan 1996 match key99* count 1000
1) "12594"
2) 	1) "key9939"
 	2) "key9941"
 	3) "key9967"
 	...
127.0.0.1:6379> scan 11687 match key99* count 1000
1) "0"
```

> limit1000并不意味着会返回1000个key, 只是限定redis单词遍历的字典槽位数量, 在上面每1000个槽位返回10个左右的key

> 返回的结果可能重复, 所以客户端需要手动去重, 并且如果遍历的过程中有数据修改，改动后的数据能不能遍历到是不确定的.

**字典与游标**

在 Redis 中所有的 key 都存储在一个很大的字典中，这个字典的结构和 Java 中的 HashMap 一样，是一维数组 + 二维链表结构，第一维数组的大小总是 2^n(n>=0)，扩容一 次数组大小空间加倍，也就是 n++。

scan 指令返回的游标就是第一维数组的位置索引，我们将这个位置索引称为槽 (slot)。 如果不考虑字典的扩容缩容，直接按数组下标挨个遍历就行了。limit 参数就表示需要遍历的 槽位数，之所以返回的结果可能多可能少，是因为不是所有的槽位上都会挂接链表，有些槽 位可能是空的，还有些槽位上挂接的链表上的元素可能会有多个。每一次遍历都会将 limit 数量的槽位上挂接的所有链表元素进行模式匹配过滤后，一次性返回给客户端。

scan 的遍历顺序非常特别。它不是从第一维数组的第 0 位一直遍历到末尾，而是采用 了高位进位加法来遍历。之所以使用这样特殊的方式进行遍历，是考虑到字典的扩容和缩容 时避免槽位的遍历重复和遗漏。

详细见《Redis深度历险》

>  另外,scan命令可以用来扫描大key, 步骤为将scan扫描出来的每一个 key，使用 type 指令获得 key 的类型，然后使用相应数据结构的 size 或者 len 方法来得到 它的大小, 最后统计出大key.
>
> 上面这个过程需要编写脚本, 繁琐, redis-cli提供了对应的指令:
>
> redis-cli -h 127.0.0.1 -p 7001 –-bigkeys

## Cluster

Redis Cluster是分布式架构：即Redis Cluster中有多个节点，每个节点都负责进行数据读写操作, 且每个节点之间会进行通信。

> cluster利用多台服务器构建集群提高超大规模数据处理能力 ,同时提供高可用支持

•Cluster模式是Redis3.0开始推出

•采用无中心结构，每个节点保存数据和整个集群状态, 每个节点都和其他所有节点连接

•官方要求：至少6个节点才可以保证高可用，即3主3从；扩展性强、更好做到高可用

•各个节点会互相通信，采用gossip协议交换节点元数据信息

•数据分散存储到各个节点上

**Cluster如何将数据分散存储?**

Redis Cluster 集群采用Hash Slot（哈希槽）分配

Redis集群预分好16384个槽，初始化集群时平均规划给每一台Redis Master

```
crc16(key)%16384
```

cluster模式集群中槽分配的元数据会不断的在集群中分发(gossip协议) ,以保证所有节点都知晓槽的分配情况.

**为什么时16384?**

主节点个数通常远远达不到16384 

> 16384即16k ,在发送心跳包时使用char进行bitmap压缩后是2k( 2 * 8 ( 8bit) * 1024 ( 1k ) = 16k) .

**构建配置**

redis-cluster.conf

![image-20211207193419534](picture/image-20211207193419534.png)

**创建命令**

指定所有机器ip和端口 , 最后指定副本数, redis默认自动配置规则会将前面的3台作为主节点,

```
./src/redis-cli -a 123456 --cluster create 
192.168.31.102:6379 192.168.31.103:6379 192.168.31.104:6379  
192.168.31.110:6379 192.168.31.111:6379 192.168.31.112:6379 
--cluster-replicas 1
```

启动之后日志

![image-20211207193714585](picture/image-20211207193714585.png)

操作结果中也会显示槽的定位转发

![image-20211207193814192](picture/image-20211207193814192.png)

**故障日志**

![image-20211207194103490](picture/image-20211207194103490.png)

![image-20211207194107617](picture/image-20211207194107617.png)

如果一对主从都挂了 , 整个集群变为fail状态而不可用 . 所以redis规定最少一个副本 , 条件允许可以弄两个副本.

## 大Key与热Key

通常我们会将含有较大数据或含有大量成员、列表数的Key称之为**大Key**.

在某个Key接收到的访问次数、显著高于其它Key时，我们可以将其称之为**热Key**

**大Key与热Key带来的问题**

1 大Key带来的常见问题 

- Client发现Redis变慢；

- Redis内存不断变大引发OOM，或达到maxmemory设置值引发写阻塞或重要Key被逐出；

- Redis Cluster中的某个node内存远超其余node，但因Redis Cluster的数据迁移最小粒度为Key而无法将node上的内存均衡化；

- 大Key上的读请求使Redis占用服务器全部带宽，自身变慢的同时影响到该服务器上的其它服务；

- 删除一个大Key造成主库较长时间的阻塞并引发同步中断或主从切换；

2 热Key带来的常见问题

- 热Key占用大量的Redis CPU时间使其性能变差并影响其它请求；

- Redis Cluster中各node流量不均衡造成Redis Cluster的分布式优势无法被Client利用，一个分片负载很高而其它分片十分空闲从而产生读/写热点问题；

- 在抢购、秒杀活动中，由于商品对应库存Key的请求量过大超出Redis处理能力造成超卖；

- 热Key的请求压力数量超出Redis的承受能力造成缓存击穿，此时大量强求将直接指向后端存储将其打挂并影响到其它业务；

**大Key与热Key产生原因**

业务规划不足、Redis不正确的使用、无效数据的堆积、访问突增等都会产生大Key与热Key，如：

1. 将Redis用在并不适合其能力的场景，造成Key的value过大，如使用String类型的Key存放大体积二进制文件型数据（大Key）；
2. 业务上线前规划设计考虑不足没有对Key中的成员进行合理的拆分，造成个别Key中的成员数量过多（大Key）；
3. 没有对无效数据进行定期清理，造成如HASH类型Key中的成员持续不断的增加（大Key）；
4. 预期外的访问量陡增，如突然出现的爆款商品、访问量暴涨的热点新闻、直播间某大主播搞活动带来的大量刷屏点赞、游戏中某区域发生多个工会间的战斗涉及大量玩家等（热Key）；
5. 使用LIST类型Key的业务消费侧代码故障，造成对应Key的成员只增不减（大Key）；

**如何处理?**

第一步:**找出大Key与热Key**

大Key与热Key的分析并不困难，我们有多种途径和手段来对Redis中的Key进行分析并找出其中的“问题”Key，如Redis的内置功能、开源工具、阿里云Redis控制台中的Key分析功能等。

1 **使用Redis内置功能发现大Key及热Key**

Redis内置的一些命令、工具都可以帮助我们来发现这些问题Key。当你对Redis的大Key热Key已有明确的分析目标时，可以通过如下命令对对应Key进行分析。

①通过Redis内置命令对目标Key进行分析

可能你会选择使用debug object命令对Key进行分析。该命令能够根据传入的对象（Key的名称）来对Key进行分析并返回大量数据，其中serializedlength的值为该Key的序列化长度，你可能会选择通过该数据来判断对应Key是否符合你的大Key判定标准。

需要注意的是，Key的序列化长度并不等同于它在内存空间中的真实长度，此外，debug object属于调试命令，运行代价较大，并且在其运行时，进入Redis的其余请求将会被阻塞直到其执行完毕。而该命令的运行的时间长短取决于传入对象（Key名）序列化长度的大小，因此，在线上环境中并不推荐使用该命令来分析大Key，这可能引发故障。

Redis自4.0起提供了MEMORY USAGE命令来帮助分析Key的内存占用，相对debug object它的执行代价更低，但由于其时间复杂度为O(N)因此在分析大Key时仍有阻塞风险。

我们建议通过风险更低方式来对Key进行分析，Redis对于不同的数据结构提供了不同的命令来返回其长度或成员数量，如下表：

![图片](picture/640.webp)

通过以上Redis内置命令我们可以方便且安全的对Key进行分析而不会影响线上服务，但由于它们返回的结果非Key的真实内存占用数据，因此不够精确，仅可作为参考。

②通过Redis官方客户端redis-cli的bigkeys参数发现大Key

如果你并无明确的目标Key用于分析，而是希望通过工具找出整个Redis实例中的大Key，此时redis-cli的bigkeys参数能够方便的帮你实现这个目标。

Redis提供了bigkeys参数能够使redis-cli以遍历的方式分析整个Redis实例中的所有Key并汇总以报告的方式返回结果。该方案的优势在于方便及安全，而缺点也非常明显：分析结果不可定制化。

bigkeys仅能分别输出Redis六种数据结构中的最大Key，如果你想只分析STRING类型或是找出全部成员数量超过10的HASH Key，那么bigkeys在此类需求场景下将无能为力。

GitHub上有大量的开源项目能够实现bigkeys的加强版使结果能够按照配置定制化输出，另外你可也以动手使用SCAN + TYPE并配合上文表格中的命令自己实现一个Redis实例级的大Key分析工具。

同样，该方案的实现方式及返回结果使其不具备精确性与实时性，建议仅作为参考。

③通过Redis官方客户端redis-cli的hotkeys参数发现热Key

Redis自4.0起提供了hotkeys参数来方便用户进行实例级的热Key分析功，该参数能够返回所有Key的被访问次数，它的缺点同样为不可定制化输出报告，大量的信息会使你在分析结果时复杂度较大，另外，使用该方案的前提条件是将redis-server的maxmemory-policy参数设置为LFU。

④通过业务层定位热Key

指向Redis的每一次访问都来自业务层，因此我们可以通过在业务层增加相应的代码对Redis的访问进行记录并异步汇总分析。该方案的优势为能够准确并及时的分析出热Key的存在，缺点为业务代码复杂度的增加，同时可能会降低一些性能。

⑤使用monitor命令在紧急情况时找出热Key

Redis的monitor命令能够忠实的打印Redis中的所有请求，包括时间信息、Client信息、命令以及Key信息。在发生紧急情况时，我们可以通过短暂执行monitor命令并将输出重定向至文件，在关闭monitor命令后通过对文件中请求进行归类分析即可找出这段时间中的热Key。

由于monitor命令对Redis的CPU、内存、网络资源均有一定的占用。因此，对于一个已处于高压状态的Redis，monitor可能会起到雪上加霜的作用。同时，这种异步收集并分析的方案的时效性较差，并且由于分析的精确度取决于monitor的执行时间，因此在多数无法长时间执行该命令的线上场景中本方案的精确度也不够好。

2 **使用开源工具发现大Key**

Redis的高度流行使我们能够方便的找到大量开源方案来解决我们当前遇到的难题：在不影响线上服务的同时得到精确的分析报告。

①使用redis-rdb-tools工具以定制化方式找出大Key

如果你希望按照自己的标准精确的分析一个Redis实例中所有Key的真实内存占用并避免影响线上服务，在分析结束后得到一份简洁易懂的报告，redis-rdb-tools是非常好的选择。

该工具能够对Redis的RDB文件进行定制化的分析，但由于分析RDB文件为离线工作，因此对线上服务不会有任何影响，这是它的最大优点但同时也是它的最大缺点：离线分析代表着分析结果的较差时效性。对于一个较大的RDB文件，它的分析可能会持续很久很久。

3 **依靠公有云的Redis分析服务发现大Key及热Key**

如果你期望能够实时的对Redis实例中的所有Key进行分析并发现当前存在的大Key及热Key、了解Redis在运行时间线中曾出现过哪些大Key热Key，使自己对整个Redis实例的运行状态有一个全面而又准确的判断，那么公有云的Redis控制台将能满足这个需求。

①阿里云Redis控制台中的CloudDBA 

CloudDBA是阿里云的数据库智能服务系统，它支持Redis大Key与热Key的实时分析、发现。

大Key及热Key分析底层为阿里云Redis内核的Key分析功能，该功能通过Redis内核直接发现并输出大Key热Key的相关信息，因此，该功能的分析结果准确性高效且对性能几乎无任何影响，你可以通过点击CloudDBA中的“Key分析”进入该功能，如下图1-1：

![图片](picture/640-16393265217212)

Key分析功能共有两个页面，它们允许在不同的时间维度对对应Redis实例中的Key进行分析：

- - 实时：对当前实例立即开始分析当前实例，展示当前存在的所有大Key及热Key。

- - 历史：展示该实例近期曾出现过的大Key及热Key，在历史页面中，所有出现过的大Key及热Key都会被记录，哪怕这些Key当前已经不存在。该功能能够很好的反映Redis的历史Key状态，帮助追溯过去或现场已遭破坏的问题。



**如何处理?**

1.**对于大Key**

①对大Key进行拆分;

如将一个含有数万成员的HASH Key拆分为多个HASH Key，并确保每个Key的成员数量在合理范围，在Redis Cluster结构中，大Key的拆分对node间的内存平衡能够起到显著作用。

②对大Key进行清理;

将不适合Redis能力的数据存放至其它存储，并在Redis中删除此类数据。需要注意的是，我们已在上文提到一个过大的Key可能引发Redis集群同步的中断，Redis自4.0起提供了UNLINK命令，该命令能够以非阻塞的方式缓慢逐步的清理传入的Key，通过UNLINK，你可以安全的删除大Key甚至特大Key。

③时刻监控Redis的内存水位

突然出现的大Key问题会让我们措手不及，因此，在大Key产生问题前发现它并进行处理是保持服务稳定的重要手段。我们可以通过监控系统并设置合理的Redis内存报警阈值来提醒我们此时可能有大Key正在产生，如：Redis内存使用率超过70%，Redis内存1小时内增长率超过20%等。

通过此类监控手段我们可以在问题发生前解决问题，如：LIST的消费程序故障造成对应Key的列表数量持续增长，将告警转变为预警从而避免故障的发生。

④对失效数据进行定期清理

例如我们会在HASH结构中以增量的形式不断写入大量数据而忽略了这些数据的时效性，这些大量堆积的失效数据会造成大Key的产生，可以通过定时任务的方式对失效数据进行清理。在此类场景中，建议使用HSCAN并配合HDEL对失效数据进行清理，这种方式能够在不阻塞的前提下清理无效数据。

⑤使用阿里云的Tair(Redis企业版)服务避开失效数据的清理工作

2.**对于热Key**

①在Redis Cluster结构中对热Key进行复制

在Redis Cluster中，热Key由于迁移粒度问题造成请求无法打散使单一node的压力无法下降。此时可以将对应热Key进行复制并迁移至其他node，例如为热Key foo复制出3个内容完全一样的Key并名为foo2，foo3，foo4，然后将这三个Key迁移到其他node来解决单一node的热Key压力。

该方案的缺点在于代码需要联动修改，同时，Key一变多带来了数据一致性挑战：由更新一个Key演变为需要同时更新多个Key，在很多时候，该方案仅建议用来临时解决当前的棘手问题。

②使用读写分离架构

如果热Key的产生来自于读请求，那么读写分离是一个很好的解决方案。在使用读写分离架构时可以通过不断的增加从节点来降低每个Redis实例中的读请求压力。

然而，读写分离架构在业务代码复杂度增加的同时，同样带来了Redis集群架构复杂度的增加：我们不仅要为多个从节点提供转发层（如Proxy，LVS等）来实现负载均衡，还要考虑从节点数量显著增加后带来的故障率增加的问题，Redis集群架构变更的同时为监控、运维、故障处理带来了更大的挑战。

③使用阿里云Tair的QueryCache特性



# MongoDB

文档型非结构化数据库, 可直接存储json文档.

可动态添加新字段, 修改时应用和数据库均无需下线.

并且支持使用JSON Schema来规范数据模式. 在保证模式灵活动态的前提下 , 提供了数据治理能力.

全量存储, 不需要多表形式. 对一个文档进行的是顺序读取, 速度快.

程序API自然 , 开发迅速 , 比如直接插入json字符串 , 通过json字符串查询.

## 高可用

1.采用复制策略保证集群中多个节点数据同步, 支持2-50个集群成员.

> 建议采用3,5,7,9奇数 ,因为mongoDB采用选举策略

2.自恢复: 主从关系, 程序访问主库, 主从同步. 并自身提供了读写分离, 故障转移的功能.

3.多中心容灾能力.

4.滚动服务-最小化服务终端(更新不停用).

5.天然支持分布式(关系型数据库因为表关联, 分布式实现困难),  多分片集群 , 并提供多种数据分布策略.

6.轻松支持TB-PB数量级.



