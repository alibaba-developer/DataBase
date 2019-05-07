# OceanBase分区表有什么不同？.md

<b>概述</b>

分区表是ORACLE从8.0开始引入的功能，也是第一个支持物理分区的数据库，随后其他数据库纷纷跟进。分区表是一种“分而治之”的思想，通过将大表、索引分成可以独立管理的、小的片段(Segment)，为海量数据访问提供了可伸缩的性能。自从 Oracle 引入分区技术以来，Oracle 公司在每次推出重要版本时都会对分区方法或功能上有所增强。在ORACLE 12c之前，ORACLE分区表始终在一个实例范围内，无法突破实例的存储空间瓶颈，12c之后ORACLE SHARDING 支持分区表的不同分区可以分布在不同的ORACLE实例里。这实际上是ORACLE吸收了其他分布式数据库中间件的优点而做的一个改进。

OceanBase是阿里巴巴和蚂蚁金服完全自主研发的通用的分布式关系型数据库，从1.0版本开始OceanBase就支持分区表，功能逐步跟ORACLE分区表兼容，并且支持不同分区分布在集群的不同节点(机器)上。本文是对OceanBase分区表的能力做一个详细介绍。

<b>分区在OceanBase中的重要性</b>

虽然“分区”的概念不是很新，但是“分区”对理解OceanBase的很多原理却是非常重要的。

<b>分区是一种水平拆分方案</b>

从水平拆分设计上说，目前分布式数据库产品里有三种拆分途径。一是以Google、TiDB为代表的在存储层按定长块切片的，称为Region，拆分细节对业务完全透明。二是以ORACLE、OceanBase为代表的使用分区表的多分区拆分，业务需要指定拆分策略和分片数，使用上基本上跟单表一样。三是以DRDS、TDSQL等为代表的分布式数据库中间件的分库分表拆分，业务使用的是一个逻辑表，实际数据存放在多个结构相同命名或位置不同的物理表上。

OceanBase里一个非分区表只有一个分区，一个分区表有多个分区。分区就是表的子集。OceanBase里单个分区只能在一个节点上，不同分区可以在不同节点上。

分区的好处有：

提高可扩展性。分区表的不同分区可以分布在不同的机器上，使得单表能获得多机的处理能力，并且使得单表的容量可以超过单机的容量。性能也是同理。
提高可管理性。对于数据操作的粒度可以控制在单个分区。例如按照时间分区的数据，可以通过drop一个分区来实现数据过期功能。
提高性能。通过分区裁剪，可以快速定位到用户需要查询的分区。提高查询性能。
分区是数据同步的最小单元
在OceanBase里，每个数据有三份，每个具体的分区也有三份，分布在不同的Zone里的不同节点上。每个分区有三份副本，副本内容相同，角色上有区分，是1个leader副本和2个follower副本。有时候会简单说1个主副本2个备副本。但是主备的概念容易引起误解。

默认业务只有leader副本提供读写服务，follower副本只同步数据，不提供服务。特殊场景下，业务SQL使用弱一致性读Hint(即read_consistency(weak))可以就近读取follower副本。此外OBProxy的LDC设置也可以做到就近读取某个合适的follower副本(这个以后再细说)。数据的变更在leader副本，事务提交的时候，leader副本会就Redo落盘发起表决，使用Paxos协议。具体就是除了自己把Redo落盘，同时还发往两个follower副本，follower副本收到redo落盘后表决“成功”。同时Follower副本开始应用该Redo。三副本里只要有一半以上成员(2个副本)表决落盘成功，leader副本上的业务的事务就提交成功返回消息给客户端。

每个分区的三副本组成一个独立的Paxos小组，相应的Redo在副本之间传输。所以说分区是数据同步的最小单元。并且这种Redo同步是自动的，不需要也不能干预的。

<b>分区是高可用的最小单元</b>

每个分区的三副本会保持数据同步，目地是为了保证在Leader副本不可用的时候选举出新的Leader副本拥有全部的数据。Paxos协议保证了Redo会在至少一个Follower副本里有(最终会所有Follower副本都有)。三副本会跟OceanBase集群的rootservice服务维持心跳，当Leader副本不可用时，经过2个租约时间后rootservice会选举出新的Leader出来，在应用完Redo后新Leader提供读写服务。

分区的选举是自动的，只要多数派存活，就不需要人工介入。所以说“分区”是高可用的最小单元。OceanBase的“切换”指的就是一个个Leader分区重新选举的过程，并不是实例级别的“切换”。当一个机器节点挂掉后， 严格的说，其影响只是局部的数据(Leader副本)的读写访问短暂中断)。在OceanBase里，一般不会说某台机器是主，某台机器是备，因为理论上所有的机器都可能存在Leader副本，都能提供读写服务。

<b>OceanBase的分区表概念</b>

<b>分区键</b>

数据表中每一行中用于计算这一行属于哪一个分区的列的集合叫做分区键。由分区键构成的用于计算这一行属于哪一个分区的表达式叫做分区表达式。

由于OceanBase的表是索引组织表(IOT)，为了保证主键的含义：给定主键的查询能很快定位到所在的分区。所以分区键必须是主键的子集。如果这个表里面还含有唯一索引，那么分区键就必须是所有唯一索引列（包括主键列）交集的子集。

Oracle的IOT表和索引分区也有这个要求，堆表没这个要求。

<b>索引</b>

分区表的索引分局部索引和全局索引。

局部索引是局限在单个分区内的索引。对于非分区的普通表，它的索引存储是整个表的行按照索引键进行排序后的结果。对于分区表的局部索引是每个分区分别自己存储自己的行按照索引键进行排序后的结果，不是全局有序的。创建语句中需要带local关键字，举个例子：

CREATE TABLE `sbtest` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `k` int(11) NOT NULL DEFAULT '0',
  `c` char(120) NOT NULL DEFAULT '',
  `pad` char(60) NOT NULL DEFAULT '',
  PRIMARY KEY (`id`, k)
) PARTITION BY HASH(k) PARTITIONS 8;

CREATE INDEX k_ind1 ON `sbtest`(`k`) LOCAL;
OceanBase 1.x版本由于不支持“全局一致性快照”功能，所以只支持局部索引，不支持全局索引。OceanBase 2.x版本支持全局索引。去掉上面的LOCAL关键字就是全局索引。

<b>分区模式</b>

根据数据分区的策略，分为三大类：

hash分区/key分区。
range分区/range columns分区。
list分区/list columns分区。
按照分区的维度，可以分为：

一级分区。
二级分区。
二级分区是一级分区的二次拆分。所以一级分区有一个拆分键，二级分区有两个拆分键，并且两次拆分的策略可以不一样。

二级分区支持下面几种分区策略组合：

hash + hash
hash + range
hash + key
key + key
range + range

<b>OceanBae分区表使用示例</b>

<b>hash分区</b>

Hash分区需要指定分区键和分区个数。通过hash的分区表达式计算得到一个整数，这个结果再跟分区个数取模得到具体这行数据属于那个分区。通常用于给定分区键的点查询，例如按照用户id来分区。hash分区通常能消除热点查询。

示例：

CREATE TABLE `sbtest` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `k` int(11) NOT NULL DEFAULT '0',
  `c` char(120) NOT NULL DEFAULT '',
  `pad` char(60) NOT NULL DEFAULT '',
  PRIMARY KEY (`id`, k)
) PARTITION BY HASH(k) PARTITIONS 8;
hash分区表达式不支持向量，即不支持 hash by (c1,c2)。

<b>key分区</b>

key分区跟hash分区类似，也是通过对分区个数取模的方式来确定数据属于哪个分区。不同的是系统会对key分区键做一个内部默认默认的hash函数后再取模。所以用户通常没有办法自己通过简单的计算来得知某一行属于哪个分区。

示例：

CREATE TABLE `sbtest_x1` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `k` int(11) NOT NULL DEFAULT '0',
  `c` char(120) NOT NULL DEFAULT '',
  `pad` char(60) NOT NULL DEFAULT '',
  PRIMARY KEY (`id`, c)
) PARTITION BY KEY(c) PARTITIONS 8; 
key分区跟hash分区的区别：

hash分区表达式结果必须是整数，key分区没有这个要求，支持字符列。
hash分区的分区键支持表达式，key分区不支持。
range分区和range columns分区:
range分区是按照分区表达式的范围来划分分区。通常用于对分区键需要按照范围的查询。例如通过按照时间字段进行范围分区，还有价格区间等一些分区方式。

示例：

create table t1 (c1 int, c2 int) 
partition by range(c1) (
    partition p0 values less than(100), 
    partition p1 values less than(500), 
    partitions p2 values less than(maxvalue)
);
range分区的限制和要求:

分区表达式的结果必须是整数类型。
不能写向量，例如partition by range(c1, c2)。
range columns和range的区别是：

range columns分区不要求是整数，可以是任意类型。
range columns分区不能写表达式，但是可以是向量。
目前提供对range分区的分区操作功能，能add/drop分区。add分区现在只能加在最后，所以最后不能是maxvalue的分区。如果是maxvalue的分区要增加一个分区，只能做分区分裂(split)。分区分裂还在研发中。

<b>list分区和list columns分区</b>

list分区是根据枚举类型的值来划分分区的。主要用于枚举类型。

示例：

create table t1 (c1 int, c2 int) 
partition by list(c1) (
    partition p0 values in (1,2,3), 
    partition p1 values in (5, 6)， 
    partition p2 values in (default)
);
list分区的限制和要求：

分区表达式的结果必须是整数。
不能写向量，例如partition by list(c1, c2)。
list columns和list的区别是：

list columns分区不要求是整数，可以是任意类型。
list columns分区不能写表达式，但可以是向量。
二级分区
按照两个维度来把数据拆分成分区。
最常用的地方就是类似用户账单领域，会按照user_id做hash分区，按照账单创建时间做range分区

hash/key + range/range columns分区
range/range columns + hash/key分区
示例：

CREATE TABLE history_t(
    user_id INT, gmt_create DATETIME, info VARCHAR(20), 
    PRIMARY KEY(user_id, gmt_create))
PARTITION BY HASH(user_id)
    SUBPARTITION BY RANGE COLUMNS(gmt_create) 
    SUBPARTITION TEMPLATE (
    SUBPARTITION p0 VALUES LESS THAN ('2014-11-11'), 
    SUBPARTITION p1 VALUES LESS THAN ('2015-11-11'), 
    SUBPARTITION p2 VALUES LESS THAN ('2016-11-11'), 
    SUBPARTITION p3 VALUES LESS THAN (MAXVALUE)
    )
    PARTITIONS 3; 
Oracle的二级分区支持每个一级分区的二级分区分区定义不一样。支持类似下面这种

create table t1 (c1 int, c2 int) 
partition by range(c1) 
subpartition by hash(c2) (
    partition p0 values less than(100) (subpartition sp0), 
    partition p1 values less than(200) (subpartition sp2, subpartition sp3)
) ;
一级分区p0下面有一个分区，而一级分区p1下面有两个分区。

OceanBase暂时不支持这样的分区方式，必须用template的分区方式。这导致对于range分区的分区操作add/drop，必须是range分区做为一级分区的方式。所以强烈建议用range + hash的分区方式，而不是hash + range.

<b>分区查询优化建议</b>

<b>分区裁剪</b>

当用户访问分区表时，往往只需要访问其中部分的分区。优化器根据SQL中所带的条件，避免访问无关分区的优化过程我们称之为“分区裁剪”(Partition Pruning)。分区裁剪是分区表提供的重要优化手段，通过分区的裁剪，SQL的执行效率可以得到大幅度的提升。用户可以利用分区裁剪的特性，在访问中加入定位分区的条件，避免访问无关数据，优化查询性能。

分区裁剪本身是一个比较复杂的过程。优化器需要根据用户表的分区信息和SQL中给定的条件，抽取出相关的分区信息，由于SQL中的条件往往比较复杂，整个抽取逻辑的复杂性也随之增加，这一过程由OceanBase中的Query Range子模块完成。

示例：

<div style="text-align:center" align="center">
<img src="/images/OceanBase分区表有什么不同？1.png" align="center" />
</div>
</br>
