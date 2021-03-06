<h3>前言</h3>
传统商业关系数据库都声称可以做到故障恢复后不丢数据（即RPO为0），跟故障前的数据状态是强一致的，实际是否一定如此？ 开源数据库MySQL在金融核心业务都不敢用，最重要的一个原因是做不到不丢数据。但是有些基于MySQL修改的数据库为何又说自己是强一致的呢？新兴的分布式数据库OceanBase声称是金融级的分布式关系型数据库，强一致，绝对不丢数据，这个是真的吗？

本文分为上下两篇。 上篇分析传统关系数据库Oracle/MySQL在应对故障时保障数据不丢失的机制，以及分析PolarDB和AliSQL在这方面曾经探索的改进措施。 下篇分析蚂蚁的OceanBase在数据安全方面的创新之处。

<h3>OceanBase简介</h3>
<h4>集群</h4>
OceanBase是蚂蚁金服完全自主研发的通用的分布式关系型数据库。OceanBase以集群的形式存在，至少三个节点分布在三个区域（Zone），每个节点上运行一个单进程程序，进程名 observer。每个observer进程都包含两个模块：SQL引擎和存储引擎，所以每个节点地位基本是平等的。稍微特殊的是每个Zone里会有一个节点的observer内还会运行总控服务，三个总控服务内容一样，角色上会有一个Leader两个Follower，只有Leader提供服务。

OceanBase集群支持多租户管理，租户就是给到业务的实例，租户的使用体验，目前对外版本会感觉像MySQL，只是因为OceanBase兼容了MySQL的连接协议和功能，绝对不是用MySQL代码或者其他开源数据库代码修改的。

<div style="text-align:center" align="center">
<img src="/OceanBase/images/揭开数据库RPO等于0的秘密1.png" align="center" />
</div>

<h4>分区</h4>
OceanBase的数据存在每个节点上，observer通过分区管理数据。分区是数据的子集，一个非分区表就是一个分区，一个分区表包含多个分区。一个分区不能跨节点，分区表的不同分区可以跨节点。所以分区表可以做水平跨节点扩展。

分区是数据的子集，是高可用的最小粒度。分析OceanBase是否丢数据，只要分析分区的数据写是否会丢就行。

<h4>OceanBase的读写模式</h4>
OceanBase在初次读入一行数据时会将该行所在块读入到内存的Block Cache中，后面修改的时候并不是直接修改这个block，而是在另外一块内存中分配少量空间记录这笔修改。并且只记录变化的部分，这部分也称为增量数据（Memtable）。前面在Block Cache里的数据称为基线数据（永远不变）。同一行记录如果反复修改多次，多个增量会以链表形式挂在该记录下。

<div style="text-align:center" align="center">
<img src="/OceanBase/images/揭开数据库RPO等于0的秘密2.png" align="center" />
</div>

OceanBase这种设计导致业务修改产生的脏块的量远小于传统数据库，于是OceanBase就索性把这些增量数据尽可能的一直保留在缓存中，也是尽可能的推迟写入到磁盘上。当然，这个增量数据最终是要落盘的。在落盘的时候，增量数据会冻结历史版本（后续写会追加新的版本），然后跟其对应的基线数据在内存中进行合并，生成SSTable（Sorted String Table）格式写入到磁盘数据文件中。这个「合并」动作对资源影响不可忽视，所以OceanBase会尽可能的推迟合并到低峰期（时间是用户设定的）。如果增量内存利用率超过阈值，OceanBase会将增量数据直接以SSTable的格式临时写入到磁盘上。这个动作叫「转储」对资源消耗小很多，释放了内存。转储可以看作数据另一种形式的落盘。

<h4>OceanBase的事务</h4>
初次听说OceanBase增量数据一天只落盘一次的这个设计时，难免会担心如果节点宕机了岂不是会丢数据。实际肯定不会，因为OceanBase在记录这些增量之前也是遵循前面提到WAL机制，先生成相关的事务日志保存在日志缓冲区里。跟Oracle不同的是OceanBase的这些事务日志在事务提交之前会一直在日志缓冲区里。此时节点若宕机，由于事务并没有提交，对业务而言也没有数据丢失。当事务提交的时候，OceanBase也会先做事务日志的持久化动作。

从这个设计可以看出OceanBase事务两个特点。第一是对大事务不是很友好。太大的事务其事务日志可能会占用不少内存空间。OceanBase 2.x版本已经在针对大事务做优化。第二就是OceanBase没有Undo，也不需要Undo。假设业务事务回滚了，在OceanBase里只有一些清理逻辑，所以会很快。

<h4>OceanBase的宕机恢复</h4>
OceanBase节点宕机后，该节点上可能有部分数据（分区）的访问会受影响，但OceanBase集群会很快恢复这些分区的访问。这是OceanBase的可用性特性，不是本文重点这里就不展开。这里接着分析这个宕掉的节点起来后的恢复逻辑。跟传统关系型数据库一样，它会读取事务日志，重做事务。但是不同的地方在于这个时候observer不需要再次读入基线数据，而只需要根据事务日志在增量内存里构建相关分区的Memtable。这个过程也是非常快的。只有在相关分区再次被业务读取时，其对应的基线数据所在块才会被再次读入到Block Cache中。

提前说明一下，OceanBase的备副本应用事务日志逻辑也是同理。

<h4>OceanBase的副本复制技术</h4>
跟Oracle一样，光支持WAL是不足以保障数据安全，OceanBase还要设法保障事务日志的可靠性。除了使用DirectIO持久化到本节点磁盘外，也需要持久化到其他节点上。

跟传统关系数据库主备两副本架构不一样，OceanBase选择了三副本架构，即每份数据至少有三份。当然这里三副本并不是唯一选择，也可以是五副本，七副本，但不能是两副本，四副本等。其原因是如果副本数是偶数，会有传统双机房容灾的脑裂问题。脑裂问题的本质就是全体成员在局部通信中断故障时无法就哪个节点接管服务作出一致性决议。成员数是奇数，才有可能形成多数派。这里就以三副本为例进行分析。

<div style="text-align:center" align="center">
<img src="/OceanBase/images/揭开数据库RPO等于0的秘密3.png" align="center" />
</div>

副本就是分区的别称，一个分区有三份数据，每份是一个副本。副本的内容除了数据还有事务日志。在这里我们只关心事务日志部分。三个副本在角色上是1个Leader（类似于主副本）2个Follower（类似于备副本）。只有Leader副本才会对外提供读写服务，这样就规避了单个分区多个节点同时写入的问题。但是注意每个分区只能单点写入跟OceanBase集群多个节点写入并不矛盾。因为Leader副本是可以分散到所有节点（OBServer）上。跟传统关系数据库一样，OceanBase维持三副本数据的同步是靠传输事务日志（Redo）机制实现的。

所以，为了保障事务日志的可靠性，OceanBase要把Leader副本上的事务日志持久化到本机和其他两个Follower副本上。宏观上表现就是可能存在各个节点彼此互相传输事务日志。这个跟MySQL的Master-Master架构里双向复制并不完全一样。 我们重点看看OceanBase如何认定事务日志可靠了。

策略一,就是所有副本成员都接收到事务日志并持久化到各自节点磁盘就是可靠的。这个确实可靠，但是太过严格，会降低事务提交性能。

策略二,就是其他多个Follower副本至少一个接收到事务日志并持久化到其节点磁盘就是可靠的。在三副本里这个结论通常也是可以的, 但是它是能忍受有Follower副本可以不持久化成功，那么就只有那个成功的Follower副本能作为备选节点。尤其在五副本的情况下，某个Follower副本的地位特殊会加大风险（如果这个Follower副本先挂了，Leader就没有备用副本了）。上篇提到的Oracle一主多备高安全的复制模式和MySQL一主多备半同步的复制技术就是这个策略。这个策略有时候也不可靠。

策略三,使用Paxos协议，各个副本成员就事务日志持久化到磁盘进行表决。只要一半以上成员投票OK，Leader副本上的事务就可以继续提交了，Follower副本才开始应用Redo。这个协议是强制性约束，不够一半成员就会表决失败，Leader副本上事务就会回滚。这里没有类似Oracle或者MySQL的同步降级的做法。此外，剩余少数派成员最终也是要表决成功的，否则就是一个异常状态。

OceanBase选择策略三，会尽力自动去保障三副本成员状态的正常，否则就会告警等运维处理。这点也是强制性的约束，也是跟传统关系数据库不一样的地方。

<h3>容灾场景RPO分析</h3>
故障转移属于高可用和容灾范畴，表面上看跟本文主题数据强一致无关，实际上如果应对方法（即主备切换）不当也可能间接导致数据丢失，所以这里也分析一下。

</h4>主备切换分析</h4>
传统关系数据库的主备通常是同机房或者同城双机房。 主备切换多需要人工操作，运维开发技术实力强的团队可能还开发了针对Oracle或MySQL自动切换的工具。

如果是DBA切换，会依赖DBA的经验能力，当时的状态等。熟练的DBA能够切换成功，也要花费一些时间，Oracle是分钟级别以上，MySQL顺利的话可以几秒。重大业务数据库切换时，如果对备库数据是否有丢失不确认时，还需要等待领导做决策。在实际操作时很多DBA都有过手心冒汗的经历。

如果是工具切换，在面对脑裂情形时工具可能会导致两个主库出现。又或者当主备数据已经不一致的时候工具拒绝切换还好，如果强制切换成功就导致数据丢失。工具自身的可靠性也需要额外验证。

传统关系数据库，特别是MySQL，在做主备架构或异地容灾时，为了性能放弃了一些底线。比如说事务日志务必落盘，事务日志务必同步在备机落盘等。使得主备数据可能面临不一致的情形，而在故障的时候就让「切换」这个动作变得更加艰难。而OceanBase在事务日志持久化到本机和其他机器上的流程是严格规定的，不存在修改某个参数就可以牺牲安全性而去提升性能。

<h4>OceanBase的「切换」</h4>
由于OceanBase的Leader副本可以分布在所有节点上，所以节点就没有主备之说。同时，OceanBase单节点故障时，也只有局部数据访问出现故障，具体就是该节点上的Leader副本的读写出现故障（前提是没有针对Follower副本的访问设置）。

OceanBase的切换的粒度是分区级别的。如果有1000个Leader副本，那就有1000个分区的切换事件（在OceanBase里叫选主，会批量选主）。OceanBase保障选出来的新Leader拥有老Leader的全部事务日志以及数据；否则就选举失败。通常一个分区选举失败并不影响其他分区的选举。

对运维而言，OceanBase集群出现节点故障，只要故障范围没有波及到业务数据的多数派，在可用性恢复方面运维是不需要人为处理的。有可能等运维收到告警并查看告警细节时数据访问就已经恢复了。这个对运维的体验非常好。

<h4>异地容灾的RPO分析</h4>
传统关系数据库的异地容灾，是不能保证数据绝对不丢的，因为主库到异地的备库的数据同步一定是异步高性能模式。 OceanBase集群可以异地部署，只要保证多数派成员之间的网络延时满足业务需求即可。比如说在蚂蚁有些业务是两地三中心三副本（杭州两个机房，上海一个机房），有些业务是三地五中心五副本（杭州两机房，上海两机房，深圳一个机房）。

如果只是做异地容灾，用OceanBase集群就足够了，只是这样没有充分发挥OceanBase优势和充分利用机器资源。蚂蚁的业务跟OceanBase一起做分布式拆分，中间件产品跟OceanBase集群一起实现了异地多活。在保证RPO=0的前提下还提高了机器的利用率，提升了业务处理能力。

<h3>后记</h3>
本文上下篇详细分析了事务日志的可靠性对RPO指标的重要性，以及分析了OceanBase在处理这个细节上是非常严格的，不牺牲安全性去提升性能，尽管OceanBase在性能上离Oracle数据库还有不少差距。

有关OceanBase的使用经验技巧问题等等，都可以来OceanBase论坛（ https://bbs.aliyun.com/thread/439.html ）看看。
