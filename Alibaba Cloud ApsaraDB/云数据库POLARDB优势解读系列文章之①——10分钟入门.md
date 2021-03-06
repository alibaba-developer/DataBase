# 云数据库POLARDB优势解读系列文章之①——10分钟入门
<h3>什么是POLARDB</h3>
POLARDB 是阿里云自研的下一代关系型分布式数据库，100%兼容MySQL，之前使用MySQL的应用程序不需要修改一行代码，即可使用POLARDB。

POLARDB在运行形态上是一个多节点集群，集群中有一个Writer节点（主节点）和多个Reader节点，他们之间节点间通过分布式文件系统（PolarFileSystem）共享底层的同一份存储（PolarStore）。

POLARDB通过内部的代理层（Proxy）对外提供服务，也就是说所有的应用程序都先经过这层代理，然后才访问到具体的数据库节点。Proxy不仅可以做安全认证（Authorization）和保护（Protection），还可以解析SQL，把写操作（比如事务、Update、Insert、Delete、DDL等）发送到Writer节点，把读操作（比如Select）均衡地分发到多个Reader节点，这个也叫读写分离。

POLARDB对外默认提供了两个数据库地址，一个是集群地址（Cluster），一个是主地址（Primary），推荐使用集群地址，因为它具备读写分离功能可以把所有节点的资源整合到一起对外提供服务。主地址是永远指向主节点，访问主地址的SQL都被发送到主节点，当发生主备切换（Failover）时，主地址也会在30秒内自动漂移到新的主节点上，确保应用程序永远连接的都是可写可读的主节点。

<div style="text-align:center" align="center">
<img src="/Alibaba Cloud ApsaraDB/images/文章之1.png" align="center" />
</div>

如上图，底层一套存储，节省成本，是『合』；中间多个节点，提高扩展性，是『分』；上层一套代理层，统一入口，使用简单，也是『合』。如此『合-分-合』的架构，在扩展性和使用便捷性之间保持了平衡，使得对于上层应用程序来说，就像使用一个单点的MySQL数据库一样简单。

<h3>如何使用</h3>
POLARDB部署在云端，创建时先选择使用的地域可用区和具体的VPC网络，然后指定节点的数量（从 2个 到 16 个）和配置（从 2核 到 88核）即可，存储空间不用提前配置，也不需要关心容量大小，系统会根据实际的使用量自动收取费用。

创建过程可能持续5-10分钟，然后配置好白名单、创建完高权限账号就可以使用了。逻辑DB和账号User，可以在控制台创建，也可以通过高权限账号登录到数据库执行SQL创建，二者效果完全一样，没有区别。

如果您需要迁移老的数据库到POLARDB，推荐使用DTS。不管源库是在RDS，还是在ECS自建MySQL，甚至是在云下有公网地址可访问的MySQL，都可以通过DTS做在线平滑迁移，停机时间5-10分钟。

<h3>特点</h3>
除了可以像使用MySQL一样使用POLARDB，这里还有一些传统MySQL数据库不具备的优势。

<h4>容量大</h4>
最高100T，不再因为单机容量的天花板而去购买多个MySQL实例做Sharding，甚至也不需要考虑分库分表，简化应用开发，降低运维负担。
<h4>高性价比</h4>
多个节点只收取一份存储的钱，也就是说只读实例越多越划算。
<h4>分钟级弹性</h4>
存储与计算分离的架构，再加上共享存储，使得快速升级成为现实。
<h4>读一致性</h4>
集群的读写分离地址，利用LSN（Log Sequence Number）确保读取数据时的全局一致性，避免因为主备延迟引起的不一致问题。
<h4>毫秒级延迟——物理复制</h4>
利用基于Redo的物理复制代替基于Binlog的逻辑复制，提升主备复制的效率和稳定性。即使是加索引、加字段的大表DDL操作，也不会对数据库造成延迟。
<h4>无锁备份</h4>
利用存储层的快照，可以在60秒内完成2T数据量大小的数据库的备份。并且这个备份过程不需要对数据库加锁，对应用程序几乎无影响，全天24小时均可进行备份。
<h4>复杂SQL查询加速</h4>
内置并行查询引擎，对执行时长超过1分钟的复杂分析类SQL加速效果明显。该功能需要额外连接地址。
