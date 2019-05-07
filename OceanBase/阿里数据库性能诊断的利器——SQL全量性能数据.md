# 阿里数据库性能诊断的利器——SQL全量性能数据.md

<b>概述</b>

在业务数据库调优过程中，如果数据库能记录执行过的每个SQL的性能信息，那对应用诊断性能异常问题会很有帮助。传统商业数据库在这方面做了一些探索。

如ORACLE的AWR或ASH视图里记录的SQL都是参数化的SQL，并且还有去重。所以不能准确关联到有问题的业务SQL上。ORACLE的诊断思路是如果SQL性能不好，就执行计划上找原因，相应的解决方案就是调整索引、收集统计信息或者用大纲(OUTLINE)修改或固定执行计划。

MySQL没有这样的SQL历史视图，有慢日志(Slow Log)可以收集执行时间超过指定阈值(long_query_time)的SQL记录在慢日志里。MySQL的思路是只关注执行慢的SQL。不过这个记录的性能数据信息并不是很多。此外记录的SQL也只是业务SQL的很小部分。此外MySQL有查询日志(Query Log)，默认是关闭的。如果打开，MySQL的性能会下降50%以上，所以生产环境基本不敢用。

在生产中，有时会碰到这种情形就是并不复杂的业务SQL突然变得很慢，其执行计划也并没有走错。单独跑这个SQL又很快。这个就让人很困惑。要分析这个问题需要认识到两点：

业务SQL在数据库的响应时间 等于SQL排队等被调度过程中的等待时间加上SQL的执行时间。当SQL请求非常高的时候，SQL工作线程非常繁忙时会引起排队。
我们从客户端监控到的SQL响应时间多是平均响应时间，只是一个时间段内全部SQL执行时间的统计值。对于具体的每个SQL，其响应时间可能在这个均值之下或者之上。或者说全部SQL的响应时间实际呈现一个类似正太分布的。
所以，如果数据库内核能记录每个SQL详细的执行信息，就能观察到上面两点。如总的时间、等待IO时间、锁等待时间和服务时间、逻辑读、物理读信息等，甚至更多。有了这些基础信息后，数据库性能诊断可以自动化，不再单纯依赖DBA的精力和能力。

阿里数据库内核的SQL全量功能
AliSQL的SQL全量日志
AliSQL是阿里巴巴数据库内核团队曾经维护的一个开源MySQL的分支，针对MySQL内核做了很多加强和优化。其中一个独特的功能就是SQL全量信息。
AliSQL的内核会在SQL执行结束时拿到执行性能信息后就答复客户端。然后异步的将SQL文本和执行性能信息以字符流形式写入一个SQL日志缓冲区。然后一个单独的日志线程循环读取该SQL日志缓冲区内容并写入到磁盘上一个管道文件。管道文件每个MySQL实例一个(实例监听端口不一样)，格式为：/u01/my$port/run/mysql.fifo。这个管道文件很特殊，如果写满了需要有其他客户端连接到这个管道并消费(读取)数据。否则，内核会停止输出(写入)SQL全量性能信息。这个设计很巧妙，不用担心SQL全量日志会占用空间(管道文件的大小很小)。它需要有客户端不停的读取这个管道文件。这个客户端就是MySQL的运维平台的监控客户端。

SQL全量输出的信息格式如下：

#time:1464861208504797
#user@host:root[root] @ [127.0.0.1]
#db:chuck
#table_name:tt
select * from tt;
#query_time:0.000232
#lock_time:0.000000
#rows_sent:0
#rows_examined:0
#rows_affected:0
#innodb_pages_read:0
#innodb_pages_io_read:0
#id:342
SQL全量输出通过参数log_sql_info控制。默认是false，开启就设置为true。开启后要先保证监控客户端在读取管道文件。通过命令show Sql_log_info_status 可以查看SQL全量日志输出状态。即使在天猫双11大促期间，这个功能也是开启的，对性能的影响在2%以内，完全可以接受。

OceanBase的SQL全量日志
OceanBase是阿里巴巴和蚂蚁金服完全自主研发的通用的分布式关系型数据库，其在SQL执行和性能诊断方面的逻辑大量参考了ORACLE的设计思路。所以在OceanBase里也有执行计划、硬解析和软解析，以及类似AWR设计的性能视图等。同时OceanBase还有自己的创新就是提供了一个SQL审计的功能。

OceanBase的视图(v$sql_audit)会以类似队列形式缓存当前集群内运行的所有SQL的执行性能信息，并且包括那些执行报错的SQL(报错原因很多如内部执行超时、锁等待超时、违反约束等各种原因)。这部份数据全部在内存里，内存大小由参数控制(sql_audit_memory_limit)，默认是3G。超出这个大小的SQL审计信息就遵循先入先出原则，并不会保存在磁盘文件里。当然有人可能会说如果OceanBase集群挂了这个数据不就丢失了吗？实际上由于OceanBase独特的高可用特性，不是那么容易挂掉的。此外，OceanBase运维平台(OCP)也会部署监控客户端定时拉取这个性能数据用作后面分析。
OceanBase的SQL审计功能由参数(enable_sql_audit)控制，可以针对每个节点设置是否开启。默认值是开启的(true)。同样即使在蚂蚁双11大促期间，这个参数也可以不关闭，由此可见开启这个对性能并没有什么影响。

<div style="text-align:center" align="center">
<img src="/OceanBase/images/阿里数据库性能诊断的利器——SQL全量性能数据1.png" align="center" />
</div>
</br>

OceanBase的SQL审计包含了SQL文本以及执行的详细信息，如执行节点、总耗时、等待时间、服务时间、逻辑读、影响行数、等待事件及其参数和其他信息，内容非常丰富。如下图

<div style="text-align:center" align="center">
<img src="/OceanBase/images/阿里数据库性能诊断的利器——SQL全量性能数据2.jpeg" align="center" />
</div>
</br>

下面是查询某个节点上某个租户的某个用户的的最近的 100条SQL执行信息

SELECT /*+ read_consistency(weak) ob_querytimeout(100000000) */ substr(usec_to_time(request_time),1,19) request_time_, s.svr_ip, s.client_Ip, s.sid,s.tenant_id, s.tenant_name, s.user_name, s.db_name, s.query_sql, s.affected_rows, s.return_rows, s.ret_code, s.event, s.elapsed_time, s.queue_time, s.execute_time, round(s.request_memory_used/1024/1024,2) req_mem_mb, is_executor_rpc, is_inner_sql
FROM gv$sql_audit s
WHERE  s.tenant_id=1001 and user_name='testuser'  and svr_ip in ('11.166.84.78')
ORDER BY request_time DESC
LIMIT 100;

<div style="text-align:center" align="center">
<img src="/OceanBase/images/阿里数据库性能诊断的利器——SQL全量性能数据3.png" align="center" />
</div>
</br>

SQL全量日志的用处
数据库诊断引擎 CloudDBA
在电商有上万的AliSQL实例，上千的数据库菜鸟开发，上百个高流量敏感的核心业务。然后支持业务的DBA团队只有十几个人，其中还不乏兼职的运维平台开发人员。数据库的性能问题不能再依赖DBA去一一分析，更别提提前发现预防性能问题。当AliSQL的运维平台收集了所有实例7*24小时的SQL全量执行信息后，接入到一个数据库智能诊断平台中，就能依据一定的规则和机器学习算法去识别性能异常并告警给开发。同时平台自动读出该SQL的详细性能数据、执行计划甚至给出优化建议(如调整索引等)给到开发。极大的降低了研发同学做数据库诊断的技术门槛。在电商，这个产品叫CloudDBA，在阿里云数据库上也有一个类似的产品，思路是一样的。

下面是CloudDBA曾经某个版本的一个功能示意图。能看到全部SQL的QPS和RT,以及每个SQL在不同区间的分布状况。 

<div style="text-align:center" align="center">
<img src="/OceanBase/images/阿里数据库性能诊断的利器——SQL全量性能数据4.png" align="center" />
</div>
</br>

进一步点击SQL前面的id值，显示该SQL的不同时间段的执行时间RT分布。

<div style="text-align:center" align="center">
<img src="/OceanBase/images/阿里数据库性能诊断的利器——SQL全量性能数据5.png" align="center" />
</div>
</br>

蓝色的是链接，还可以进一步做性能下钻分析。

CloudDBA的价值在于降低了业务研发做数据库诊断的技术门槛，同时解放了业务DBA的人力，专心去做更难的事情。不过也不能过度夸大其作用。作为一个大规模的性能诊断平台，无论是基于规则(DBA的经验)还是基于机器学习(计算机的经验)，都是基于统计的，都可能存在”不命中“或者”误判“的情形。

OceanBase数据库云平台 OCP
同样，OCP采集了各个OceanBase集群的SQL审计信息后也提供页面可以查看TOP SQL信息，并提供SQL性能分析下钻功能。

<div style="text-align:center" align="center">
<img src="/OceanBase/images/阿里数据库性能诊断的利器——SQL全量性能数据6.png" align="center" />
</div>
</br>

点击 蓝色序号链接，可以查看SQL明细

<div style="text-align:center" align="center">
<img src="/OceanBase/images/阿里数据库性能诊断的利器——SQL全量性能数据7.png" align="center" />
</div>
</br>

这些还只是的简单的展示。在SQL审计日志信息上也可以继续发展自动诊断，定位问题SQL，并给出优化建议。
