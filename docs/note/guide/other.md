# 数据业务

## 高并发下乐观锁解决并发数据冲突

> 悲观锁性能差,用户体验差 ,且现在主流场景如电商都是读多写少

通过给表添加version字段实现, 每次修改前先查询出版本,修改数据时先判断版本并版本加一.

如果遇到冲突怎么办?

1. 前端应用提示"数据正在处理 , 请稍后再试";
2. 附加spring-retry在service上进行方法重试

```
@Transactional
//重试异常  最大重试次数  重试间隔默认1S
@Retryable(value = {VersionException.class}, maxAttempts = 3)
public void updateBal(){
    Account acc =  执行：”select id,bal,_version from acc where id = 1001”;
    acc.setBal(acc.getBal() + 400)；
    int count = 执行：“update acc set bal = ${acc.bal} , _version=_version + 1 
		where id = 1001 and _version=${acc.version}”；
    if(count == 0) { throw new VersionException(“产生并发异常”) }；
}

```

> 额外问题: 为什么不直接一条语句update即可?   尽量不要将计算放在sql语句中

## 逻辑删除与唯一索引

**逻辑删除:**实际业务中, 对删除的数据给它设置一个标记"已删除", 在后续正常查询中,会过滤这条数据. 

有利于排查问题与补救.

通常设置标记的字段数据类型有布尔类型(mysql中可以使用tinyint) ; 字符串类型(y/n).

而逻辑删除与唯一索引共同使用时, 比如博客系统中,标题+作者是唯一的, 用户发布了博客后不满意进行删除, 之后又发布一篇标题相同的博客,这时候就会引发唯一索引带来的冲突.

解决方案为 :  对于使用了逻辑删除的表,唯一索引上必须要加上逻辑删除的字段.

但是这样设计同样有问题 , 如果用户删除两次相同标题的博客, 同样会引发唯一索引冲突.

最终解决方案为: 使用时间戳来表示已删除, 使用0来表示未删除.

## 访问分布式下的公共表

在分布式架构下被其他业务模块共享的表为公共表.

可以为公共表构建公共服务操作公共表, 并通过Restful | RPC 方式给其他模块使用. 统一指定接口规范 , 实现数据层面解耦 .

# Nginx+keepalived+DNS实现web高可用

在通常的一nginx多web服务器的架构下(反向代理+负载均衡) ,

为了实现高可用, 可以再配置一台Nginx ,并利用linux上的VIP屏蔽两台机器的实际ip  ,  外加keepalived利用VIP定时向nginx发送心跳, 如果一台挂掉, keepalived自动将VIP漂移到另一台上.

上述情况下 , 在第一台nginx没挂之前, 第二台nginx都是闲置状态 ,为了让另外一台nginx也投入使用,  可利用DNS轮询机制给一个域名绑定两个VIP ,  一个VIP指向第一台nginx ,一个VIP指向另一台nginx  . 而任一nginx挂掉, 都不影响使用.

DNS轮询缺点:

1.只负责IP轮询获取,不保证节点可用;

2.DNS IP列表更新有延迟;

3.外网IP占用严重(所以中间通过nginx转发);

4.安全性降低



# 规约

## 为什么不使用外键

> 【阿里JAVA规范】不得使用外键与级联，一切外键概念必须在应用层解决。

1.每次做DELETE 或者UPDATE都必须考虑外键约束，会导致开发的时候很痛苦,测试数据极为不方便。

2.性能问题, 在外键关联表插入数据是会带来额外的数据一致性校验查询 .

3.并发问题,外界约束会启用行级锁 ,主表写入时外键关联表会进入阻塞.

4.级联删除问题, 多层级联删除会让数据变得不可控, 触发器也严格被禁用.

5.数据耦合, 数据库层面数据关系产生耦合, 数据迁移维护困难.

## 禁止三表join

>【强制】超过三个表禁止join . 需要join的字段 , 数据类型必须绝对一致 ; 多表关联查询时, 保证被关联的字段需要有索引。
>
>说明: 即使双表join也需要注意表索引,sql性能.

产品强制要求: 阿里OceanBase(MySql改造) 只允许两表关联 , MyCat只支持两表关联.

原因:

1.Mysql自身设计缺陷, 超过三表关联时优化器做的不好 ; 

> 银行业务无此需求 , oracle无mysql的此缺陷

2.NLJ( 嵌套循环关联 )多级嵌套性能差, 贪心与动态规划算法计算量激增;

> NLJ采用小表关联大表(小表作为驱动表,大表作为匹配表)

3.依赖数据源特性获取数据, 数据迁移改造难;

比如有商品库和订单库,一条商品记录对应对条订单记录.

当订单量巨大时, 可能需要单独迁移出一个模块分开维护, 这时候商品服务需要restful|RPC调用订单服务,  此时单条join无法完成; 

或者当订单量巨大时 需要分表存储 , 此时单条join也无法完成 ,需要使用多条in语句进行查询(在商品表中查询出所有订单表的对应信息, 然后在每个订单表分片中使用in 查询) , 而使用in只适用于数据量小(最大1000),且只支持inner join.

解决方案

反范式表:将所有关联的表拼接成一个完整的大表(数据冗余) .

在数据变更时, 将数据插入反范式表中 , 但随着数据增多, 该表同样会变得臃肿.

> 为什么不使用视图 ,视图只是逻辑表 ,查询时还是需要结合多张表查询

**最终解决方案 数据集市**

通过ETL(抽取&转换&加载)每天(T+1日终处理,第二天凌晨)对数据源进行抽取 , 在数据仓库中进行加工处理生成额外的表.

> 在ETL数据集市中,通常还会使用生成倒排表的形式关联, 比如 在对订单明细表分片时, 不根据订单明细id进行分片, 而是根据订单id进行分片, 当需要获取指定订单id的订单明细时,  不需要遍历所有分片, 只需要对订单id取模获取其对应订单明细的分片.
>
> 通过每次日终额外生成倒排表提高数据提取效率 , 典型的空间换时间.

## 禁止使用存储过程

> 【强制】禁止使用存储过程,  存储过程难以调试和扩展, 更没有移植性。

1.为什么银行都在使用存储过程?

银行业务以数据为核心 , Oracle、DB2一统江湖，存储过程与语言无关 , 预算充足，好多个W采购小型机满足性能要求 ,存储过程几乎是每一个信息科技处开发员工的入职要求. 而银行被Oracle、DB2绑架 , 数据好迁移，存储过程要全部重写  ,谁来/谁敢承担核心业务的风险？

![image-20211206203335275](picture/image-20211206203335275.png)

2.**存储过程在互联网分布式场景的问题**

① 分片场景下存储过程只能作用在局部数据(本分片);

② 数据库压力激增 

③ 无法保证分布式全局事务

![image-20211206204754565](picture/image-20211206204754565.png)
