# 云数据库POLARDB优势解读系列文章之⑤——会话读一致性
<h3>POLARDB架构</h3>
我们知道，POLARDB是一个由多个节点构成的数据库集群，一个主节点，多个读节点。对外默认提供两个地址，一个是集群地址，一个是主地址，推荐使用集群地址，因为它具备读写分离功能可以把所有节点的资源整合到一起对外提供服务。

<div style="text-align:center" align="center">
<img src="/Alibaba Cloud ApsaraDB/images/POLARDB优势1.png" align="center" />
</div>

<h3>MySQL读写分离解决和引入的问题</h3>
用过MySQL的都知道，MySQL的主从复制简单易用，非常流行，通过把主库的Binlog异步地传输到备库并实时应用，一方面可以实现高可用，另一方面备库也可以提供查询，来减轻对主库的压力。

<div style="text-align:center" align="center">
<img src="/Alibaba Cloud ApsaraDB/images/POLARDB优势2.png" align="center" />
</div>

虽然备库可以提供查询，但存在两个问题，一是主库和备库一般提供两个不同的访问地址，应用程序端需要选择使用哪一个，对应用有侵入。二来MySQL的复制是异步的，即使是半同步也没办法做到100%强同步，因此备库的数据并不是最新的，有延迟，无法保证查询的一致性。

为了解决第一个问题，我们引入了读写分离代理，如下图，对应用程序非常友好。一般的实现是，代理会伪造成MySQL与应用程序建立好连接，解析发送进来的每一条SQL，如果是UPDATE、DELETE、INSERT、CREATE等写操作则直接发往主库，如果是SELECT则发送到备库。

<div style="text-align:center" align="center">
<img src="/Alibaba Cloud ApsaraDB/images/POLARDB优势3.png" align="center" />
</div>

但是第二个问题——延迟导致的查询不一致——还是没有解决，使用时，就不可避免地会遇到备库SELECT查询数据不一致的现象（因为主备有延迟）。MySQL负载低的时候延迟可以控制在5秒内，但当负载很高时，尤其是对大表做DDL（比如加字段）或者大批量插入的时候，延迟会非常严重。

POLARDB读写分离的会话读一致性
POLARDB是读写分离的架构，传统的读写分离都只提供最终一致性的保证，主从复制延迟会导致从不同节点查询到的结果不同，比如一个会话内连续执行以下QUERY：

INSERT INTO t1(id, price) VALUES(111, 96);

UPDATE t1 SET price = 100 WHERE id=111;

SELECT price FROM t1;

在读写分离的下，最后一个查询的结果是不确定的，因为读会发到只读库，在执行SELECT时之前的更新是否同步到了只读库时不确定的，因此结果也是不确定的；因为有这个问题，所以就要求应用程序去适应最终一致性，而一般的解决方法是： 将业务做拆分，有高一致性要求的请求直连到主库，可以接受最终一致性的部分走读写分离；显然这样会增加应用开发的负担，还会增大主库的压力，影响读写分离的效果；

为了解决这个问题，在POLARDB中我们提供了会话一致性或者说因果一致性的保证，会话一致性即保证同一个会话内，后面的请求一定能够看到此前更新所产生版本的数据或者比这个版本更新的数据，保证单调性，就很好的解决了上面这个例子里的问题；

<h3>实现原理</h3>
<div style="text-align:center" align="center">
<img src="/Alibaba Cloud ApsaraDB/images/POLARDB优势4.png" align="center" />
</div>

在POLARDB的链路中间层做读写分离的同时，中间层会track各个节点已经apply了的redolog位点即LSN，同时每次更新时会记录此次更新的位点为Session LSN, 当有新请求到来时我们会比较Session LSN 和当前各个节点的LSN，仅将请求发往LSN >= Session LSN的节点，从而保证了会话一致性；表面上看该方案可能导致主库压力大，但是因为POLARDB是物理复制，上一篇已详细介绍过，速度极快，在上述场景中，当更新完成后，返回客户端结果时复制就同步在进行，而当下一个读请求到来时主从极有可能已经完成，然后大多数应用场景都是读多写少，所以经验证在该机制下即保证了会话一致性，也保证了读写分离负载均衡的效果；
