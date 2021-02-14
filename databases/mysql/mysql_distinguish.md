
### 前言
```
在互联网还未崛起的时代,我们的传统应用都有这样一个特点：访问量、数据量都比较小，单库单表都完全可以支撑整个业务。
随着互联网的发展和用户规模的迅速扩大,对系统的要求也越来越高。因此传统的MySQL单库单表架构的性能问题就暴露出来了。而有下面几个因素会影响数据库性能:

```

+ 数据量
```
MySQL单库数据量在5000万以内性能比较好,超过阈值后性能会随着数据量的增大而变弱。
```

```
MySQL单表的数据量是500w-1000w之间性能比较好,超过1000w性能也会下降。
```

+ 磁盘
```
因为单个服务的磁盘空间是有限制的,如果并发压力下,所有的请求都访问同一个节点,肯定会对磁盘IO造成非常大的影响。
```

+ 数据库连接
```
数据库连接是非常稀少的资源,如果一个库里既有用户、商品、订单相关的数据,当海量用户同时操作时,数据库连接就很可能成为瓶颈。
```

```
为了提升性能,所以我们必须要解决上述几个问题,那就有必要引进分库分表。
```

### 垂直拆分 or 水平拆分？
```
当我们单个库太大时,我们先要看一下是因为表太多还是数据量太大，如果是表太多,则应该将部分表进行迁移(可以按业务区分),这就是所谓的垂直切分。
如果是数据量太大,则需要将表拆成更多的小表,来减少单表的数据量,这就是所谓的水平拆分。

```


### 垂直拆分

+ 垂直分库
```
垂直分库针对的是一个系统中的不同业务进行拆分,比如用户一个库,商品一个库,订单一个库。 一个购物网站对外提供服务时,会同时对用户、商品、订单表进行操作。没拆分之前, 全部都是落到单一的库上的,这会让数据库的单库处理能力成为瓶颈。如果垂直分库后还是将用户、商品、订单放到同一个服务器上,只是分到了不同的库,这样虽然会减少单库的压力,但是随着用户量增大,这会让整个数据库的处理能力成为瓶颈,还有单个服务器的磁盘空间、内存也会受非常大的影响。 所以我们要将其拆分到多个服务器上，这样上面的问题都解决了，以后也不会面对单机资源问题。

```


+ 垂直分表
```
也就是“大表拆小表”，基于列字段进行的。一般是表中的字段较多，将不常用的， 数据较大，长度较长（比如text类型字段）的拆分到“扩展表“。一般是针对那种几百列的大表，也避免查询时，数据量太大造成的“跨页”问题。

```

### 水平拆分

+ 水平分表

```
和垂直分表有一点类似,不过垂直分表是基于列的,而水平分表是基于全表的。水平拆分可以大大减少单表数据量,提升查询效率。
```

+ 水平分库分表

```
将单张表的数据切分到多个服务器上去，每个服务器具有相应的库与表，只是表中数据集合不同。 水平分库分表能够有效的缓解单机和单库的性能瓶颈和压力，突破IO、连接数、硬件资源等的瓶颈。
```

### 几种常用的分库分表的策略

+ HASH取模
```
假设有用户表user,将其分成3个表user0,user1,user2.路由规则是对3取模,当uid=1时,对应到的是user1,uid=2时,对应的是user2。
```

+ 范围分片

```
从1-10000一个表,10001-20000一个表。
```

+ 地理位置分片

```
华南区一个表,华北一个表。
```

+ 时间分片

```
按月分片，按季度分片等等,可以做到冷热数据。
```


### 分库分表后引入的问题

+ 分布式事务问题
```
如果我们做了垂直分库或者水平分库以后,就必然会涉及到跨库执行SQL的问题,这样就引发了互联网界的老大难问题-"分布式事务"。那要如何解决这个问题呢？
1.使用分布式事务中间件 2.使用MySQL自带的针对跨库的事务一致性方案(XA),不过性能要比单库的慢10倍左右。3.能否避免掉跨库操作(比如将用户和商品放在同一个库中)

```


+ 跨库join的问题

```
分库分表后表之间的关联操作将受到限制，我们无法join位于不同分库的表，也无法join分表粒度不同的表， 结果原本一次查询能够完成的业务，可能需要多次查询才能完成。粗略的解决方法： 全局表：基础数据，所有库都拷贝一份。 字段冗余：这样有些字段就不用join去查询了。 系统层组装：分别查询出所有，然后组装起来，较复杂。

```

+ 横向扩容的问题

```
当我们使用HASH取模做分表的时候,针对数据量的递增,可能需要动态的增加表,此时就需要考虑因为reHash导致数据迁移的问题。
```

+ 结果集合并、排序的问题

```
因为我们是将数据分散存储到不同的库、表里的,当我们查询指定数据列表时,数据来源于不同的子库或者子表,就必然会引发结果集合并、排序的问题。如果每次查询都需要排序、合并等操作,性能肯定会受非常大的影响。走缓存可能一条路!

```

### 使用分库分表中间件



+ Mycat

```
Mycat发展到现在，适用的场景已经很丰富，而且不断有新用户给出新的创新性的方案，以下是几个典型的应用场景：
单纯的读写分离，此时配置最为简单，支持读写分离，主从切换
分表分库，对于超过1000万的表进行分片，最大支持1000亿的单表分片
多租户应用，每个应用一个库，但应用程序只连接Mycat，从而不改造程序本身，实现多租户化报表系统，借助于Mycat的分表能力，处理大规模报表的统计
替代Hbase，分析大数据作为海量数据实时查询的一种简单有效方案，比如100亿条频繁查询的记录需要在3秒内查询出来结果，除了基于主键的查询，还可能存在范围查询或其他属性查询，此时Mycat可能是最简单有效的选择。

```


+ Sharding-JDBC

```
当当网开发的简单易用、轻量级的中间件。
```

```
此外还有淘宝的TDDL,支付宝的OneProxy,360的Atlas等。
```




















## [回到上一级](./index.md)