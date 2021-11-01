1 设计目标和理由
1.1 Redis Cluster goals


高性能可线性扩展至最多1000节点。集群中没有代理，（集群节点间）使用异步复制，没有归并操作(merge operations on values)
可接受的写入安全:系统尝试(采用best-effort方式)保留所有连接到master节点的client发起的写操作。通常会有一个小的时间窗，时间窗内的已确认写操作可能丢失(即，在发生failover之前的小段时间窗内的写操作可能在failover中丢失)。而在(网络)分区故障下，对少数派master的写入，发生写丢失的时间窗会很大。
可用性：Redis Cluster在以下场景下集群总是可用：大部分master节点可用，并且对少部分不可用的master，每一个master至少有一个当前可用的slave。更进一步，通过使用 replicas migration 技术，当前没有slave的master会从当前拥有多个slave的master接受到一个新slave来确保可用性。

1.2 Clients and Servers roles in the Redis Cluster protocol


Redis Cluster的节点负责维护数据，和获取集群状态，这包括将keys映射到正确的节点。集群节点同样可以自动发现其他节点、检测不工作节点、以及在发现故障发生时晋升slave节点到master
所有集群节点通过由TCP和二进制协议组成的称为 Redis Cluster Bus 的方式来实现集群的节点自动发现、故障节点探测、slave升级为master等任务。每个节点通过cluster bus连接所有其他节点。节点间使用gossip协议进行集群信息传播，以此来实现新节点发现，发送ping包以确认对端工作正常，以及发送cluster消息用来标记特定状态。cluster bus还被用来在集群中创博Pub/Sub消息，以及在接收到用户请求后编排手动failover。

1.3 Write safety


Redis Cluster在节点间采用了异步复制，以及 last failover wins 隐含合并功能(implicit merge function)（【译注】不存在合并功能，而是总是认为最近一次failover的节点是最新的）。这意味着最后被选举出的master所包含的数据最终会替代（同一前master下）所有其他备份(replicas/slaves)节点（包含的数据）。当发生分区问题时，总是会有一个时间窗内会发生写入丢失。然而，对连接到多数派master（majority of masters）的client，以及连接到少数派master（mimority of masters）的client，这个时间窗是不同的。
相比较连接到少数master(minority of masters)的client，对连接到多数master(majority of masters)的client发起的写入，Redis cluster会更努力地尝试将其保存。 下面的场景将会导致在主分区的master上，已经确认的写入在故障期间发生丢失：

写入请求达到master，但是当master执行完并回复client时，写操作可能还没有通过异步复制传播到它的slave。如果master在写操作抵达slave之前挂了，并且master无法触达(unreachable)的时间足够长而导致了slave节点晋升，那么这个写操作就永远地丢失了。通常很难直接观察到，因为master尝试回复client(写入确认)和传播写操作到slave通常几乎是同时发生。然而，这却是真实世界中的故障方式。（【译注】不考虑返回后宕机的场景，因为宕机导致的写入丢失，在单机版redis上同样存在，这不是redis cluster引入的目的及要解决的问题）
另一种理论上可能发生写入丢失的模式是：

master因为分区原因不可用（unreachable）
该master被某个slave替换(failover)
一段时间后，该master重新可用
在该old master变为slave之前，一个client通过过期的路由表对该节点进行写入。




上述第二种失败场景通常难以发生，因为：1）少数派master(minority master)无法与多数派master(majority master)通信达到一定的时间后，它将拒绝写入，并且当分区恢复后，该master在重新与多数派master建立连接后，还将保持拒绝写入状态一小段时间来感知集群配置变化。留给client可写入的时间窗很小。2）发生这种错误还有一个前提是，client一直都在使用过期的路由表（而实际上集群因为发生了failover，已有slave发生了晋升）。
写入少数派master(minority side of a partition)会有一个更长的时间窗会导致数据丢失。因为如果最终导致了failover，则写入少数派master的数据将会被多数派一侧(majority side)覆盖（在少数派master作为slave重新接入集群后）。
特别地，如果要发生failover，master必须至少在NODE_TIMEOUT时间内无法被多数masters(majority of maters)连接，因此如果分区在这一时间内被修复，则不会发生写入丢失。当分区持续时间超过NODE_TIMEOUT时，所有在这段时间内对少数派master(minority side)的写入将会丢失。然而少数派一侧(minority side)将会在NODE_TIMEOUT时间之后如果还没有连上多数派一侧，则它会立即开始拒绝写入，因此对少数派master而言，存在一个进入不可用状态的最大时间窗。在这一时间窗之外，不会再有写入被接受或丢失。

1.4 可用性(Availability)


Redis Cluster在少数派分区侧不可用。在多数派分区侧，假设由多数派masters存在并且不可达的master有一个slave，cluster将会在NODE_TIMEOUT外加重新选举所需的一小段时间(通常1～2秒)后恢复可用。
这意味着，Redis Cluster被设计为可以忍受一小部分节点的故障，但是如果需要在大网络分裂(network splits)事件中(【译注】比如发生多分区故障导致网络被分割成多块，且不存在多数派master分区)保持可用性，它不是一个合适的方案(【译注】比如，不要尝试在多机房间部署redis cluster，这不是redis cluster该做的事)。
假设一个cluster由N个master节点组成并且每个节点仅拥有一个slave，在多数侧只有一个节点出现分区问题时，cluster的多数侧(majority side)可以保持可用，而当有两个节点出现分区故障时，只有 1-(1/(N*2-1)) 的可能性保持集群可用。
也就是说，如果有一个由5个master和5个slave组成的cluster，那么当两个节点出现分区故障时，它有 1/(5*2-1)=11.11%的可能性发生集群不可用。
Redis cluster提供了一种成为 Replicas Migration 的有用特性特性，它通过自动转移备份节点到孤master节点，在真实世界的常见场景中提升了cluster的可用性。在每次成功的failover之后，cluster会自动重新配置slave分布以尽可能保证在下一次failure中拥有更好的抵御力。

2.1.5 性能(Performance)


Redis Cluster不会将命令路由到其中的key所在的节点，而是向client发一个重定向命令 (- MOVED) 引导client到正确的节点。
最终client会获得一个最新的cluster(hash slots分布)展示，以及哪个节点服务于命令中的keys，因此clients就可以获得正确的节点并用来继续执行命令。
因为master和slave之间使用异步复制，节点不需要等待其他节点对写入的确认（除非使用了WAIT命令）就可以回复client。
同样，因为multi-key命令被限制在了临近的key(near keys)(【译注】即同一hash slot内的key，或者从实际使用场景来说，更多的是通过hash tag定义为具备相同hash字段的有相近业务含义的一组keys)，所以除非触发resharding，数据永远不会在节点间移动。
普通的命令(normal operations)会像在单个redis实例那样被执行。这意味着一个拥有N个master节点的Redis Cluster，你可以认为它拥有N倍的单个Redis性能。同时，query通常都在一个round trip中执行，因为client通常会保留与所有节点的持久化连接（连接池），因此延迟也与客户端操作单台redis实例没有区别。
在对数据安全性、可用性方面提供了合理的弱保证的前提下，提供极高的性能和可扩展性，这是Redis Cluster的主要目标

1.6 为何要避免合并(merge)操作


Redis Cluster设计上避免了在多个拥有相同key-value对的节点上的版本冲突（及合并/merge），因为在redis数据模型下这是不需要的。Redis的值同时都非常大；一个拥有数百万元素的list或sorted set是很常见的。同样，数据类型的语义也很复杂。传输和合并这类值将会产生明显的瓶颈，并可能需要对应用侧的逻辑做明显的修改，比如需要更多的内存来保存meta-data等。
这里(【译注】刻意避免了merge)并没有严格的技术限制。CRDTs或同步复制状态机可以塑造与redis类似的复杂的数据类型。然而，这类系统运行时的行为与Redis Cluster其实是不一样的。Redis Cluster被设计用来支持非集群redis版本无法支持的一些额外的场景。

2 Redis Cluster主要模块介绍

2.1 分布式Keys模型


key空间被分为16384个slot，有效地设置了一个集群的最大上限为16384个master（然而一般建议最大节点数少于1000）
每个master节点处理16384个hash slot中的一部分。在没有集群重新配置的任务(比如，正在执行将一个slot hash从一个节点转移到另一个的任务)时，cluster是稳定的。当cluster处于稳定状态时，每个hash slot只会由一个节点提供服务(当然服务节点可以有一个或多个slave，用于在网络分裂或单点故障是替代它，或是在允许读取过期数据的前提下用来扩展读操作)。
将key映射到hash slot的算法如下：

HASH_SLOT = CRC16(key) mod 16384


具体算法介绍及示例代码请参考原文


2.2 Keys hash tags


Hash tags提供了一种途径，用来将多个(相关的)key分配到相同的hash slot中。这时Redis  Cluster中实现multi-key操作的基础。
hash tag规则如下，如果：

key包含一个{字符

并且 如果在这个{的右面有一个}字符

并且 如果在{和}之间存在至少一个字符


那么，{和}之间的字符将用来计算HASH_SLOT，以保证这样的key保存在同一个slot中。
例如：


{user1000}.following和{user1000}.followers这两个key会被hash到相同的hash slot中，因为只有user1000会被用来计算hash slot值。

foo{}{bar}这个key不会启用hash tag因为第一个{和}之间没有字符。

foo{{bar}}zap这个key中的{bar部分会被用来计算hash slot

foo{bar}{zap}这个key中的bar会被用来计算计算hash slot，而zap不会



2.3 Cluster nodes属性


每个节点在cluster中有一个唯一的名字。这个名字由160bit随机十六进制数字表示，并在节点启动时第一次获得(通常通过/dev/urandom)。节点在配置文件中保留它的ID，并永远地使用这个ID，直到被管理员使用CLUSTER RESET HARD命令hard reset这个节点。
节点ID被用来在整个cluster中标识每个节点。一个节点可以修改自己的IP地址而不需要修改自己的ID。Cluster可以检测到IP /port的改动并通过运行在cluster bus上的gossip协议重新配置该节点。
节点ID不是唯一与节点绑定的信息，但是他是唯一的一个总是保持全局一致的字段。每个节点都拥有一系列相关的信息。一些信息时关于本节点在集群中配置细节，并最终在cluster内部保持一致的。而其他信息，比如节点最后被ping的时间，是节点的本地信息。
每个节点维护着集群内其他节点的以下信息：node id, 节点的IP和port，节点标签，master node id（如果这是一个slave节点），最后被挂起的ping的发送时间(如果没有挂起的ping则为0)，最后一次收到pong的时间，当前的节点configuration epoch ，链接状态，以及最后是该节点服务的hash slots。
对节点字段更详细的描述，可以参考对命令 CLUSTER NODES的描述。

CLUSTER NODES命令可以被发送到集群内的任意节点，他会提供基于该节点视角(view)下的集群状态以及每个节点的信息。
下面是一个发送到一个拥有3个节点的小集群的master节点的CLUSTER NODES输出的例子。

$ redis-cli cluster nodes

d1861060fe6a534d42d8a19aeb36600e18785e04 127.0.0.1:6379 myself - 0 1318428930 1 connected 0-1364
3886e65cc906bfd9b1f7e7bde468726a052d1dae 127.0.0.1:6380 master - 1318428930 1318428931 2 connected 1365-2729
d289c575dcbc4bdd2931585fd4339089e461a27d 127.0.0.1:6381 master - 1318428931 1318428931 3 connected 2730-4095
在上面的例子中，按顺序列出了不同的字段：
node id, address:port, flags, last ping sent, last pong received, configuration epoch, link state, slots.

2.4 Cluster总线


每个Redis Cluster节点有一个额外的TCP端口用来接受其他节点的连接。这个端口与用来接收client命令的普通TCP端口有一个固定的offset。该端口等于普通命令端口加上10000.例如，一个Redis街道口在端口6379坚挺客户端连接，那么它的集群总线端口16379也会被打开。
节点到节点的通讯只使用集群总线，同时使用集群总线协议：有不同的类型和大小的帧组成的二进制协议。集群总线的二进制协议没有被公开文档话，因为他不希望被外部软件设备用来预计群姐点进行对话。当然你可以通过Redis Cluster的源码中的cluster.h和cluster.c获得更多的细节。

2.5 集群拓扑


Redis Cluster是一张全网拓扑，节点与其他每个节点之间都保持着TCP连接。
在一个拥有N个节点的集群中，每个节点由N-1个TCP传出连接，和N-1个TCP传入连接。
这些TCP连接总是保持活性(be kept alive)。当一个节点在集群总线上发送了ping请求并期待对方回复pong，（如果没有得到回复）在等待足够成时间以便将对方标记为不可达之前，它将先尝试重新连接对方以刷新与对方的连接。
而在全网拓扑中的Redis Cluster节点，节点使用gossip协议和配置更新机制来避免在正常情况下节点之间交换过多的消息，因此集群内交换的消息数目(相对节点数目)不是指数级的。

2.6 节点握手


节点总是接受集群总线端口的链接，并且总是会回复ping请求，即使ping来自一个不可信节点。然而，如果发送节点被认为不是当前集群的一部分，所有其他包将被抛弃。
节点认定其他节点是当前集群的一部分有两种方式：

如果一个节点出现在了一条MEET消息中。一条meet消息非常像一个PING消息，但是它会强制接收者接受一个节点作为集群的一部分。节点只有在接收到系统管理员的如下命令后，才会向其他节点发送MEET消息：
CLUSTER MEET ip port

如果一个被信任的节点gossip了某个节点，那么接收到gossip消息的节点也会那个节点标记为集群的一部分。也就是说，如果在集群中，A知道B，而B知道C，最终B会发送gossip消息到A，告诉A节点C是集群的一部分。这时，A会把C注册未网络的一部分，并尝试与C建立连接。


这意味着，一旦我们把某个节点加入了连接图(connected graph)，它们最终会自动形成一张全连接图(fully connected graph)。这意味着只要系统管理员强制加入了一条信任关系（在某个节点上通过meet命令加入了一个新节点），集群可以自动发现其他节点。

3 Redirection and resharding

3.1 MOVED Redirection


一个redis client可以随意地向集群里的任意节点发送查询请求，包括slave节点。节点将会分析请求，如果它是可以接受的（也就是说，请求只涉及一个key，或是涉及了多个属于同一hash slot的keys），它会查找哪个节点负责命令中的key所属的hash slot。
如果hash slot正好由当前节点服务，那么请求会直接被执行，否则节点会检查它内部的hash slot到节点的映射，并会恢复client一个MOVED错误，就像下面的例子：

$ redis-cli -p 7000 get foo10449  
(error) MOVED 4995 127.0.0.1:7003


错误包含了key所属的hash slot(3999)和为该hash slot服务的节点ip:port。客户端需要想指定节点的IP地址和端口补发该请求。即使client在补发请求前等待了很长一段时间，并且在此期间集群的配置发生了变化，目标节点依旧会再次发送一个 MOVED 错误告知3999 hash slot的所有权现在已经转移到了另一个节点。
此外，尽管从集群角度看节点是通过ID被标记的，为了简化与client的接口，我们还是向客户端暴露了一个hash slot到ip:port指定的redis节点。
规范没有对client做出强制要求，但是redis client (在收到了MOVED错误之后)应该能记录下目前hash slot 3999由节点127.0.0.1:6381提供服务。这样如果一旦有一个新的命令需要发送，它就可以计算出目标key的hash slot并有很大的机会选择一个正确的节点。
另一个可以选择的方式是每当接收到一个MOVED重定向，客户端就通过CLUSTER NODES或  CLUSTER SLOTS`命令刷新客户端侧的全部集群信息。当client遇到一个重定向错误，那么更有可能有多个hash slots被重新配置而不是仅仅这一个，所以尽快更新客户端配置通常是最有效的策略。
注意到当cluster时稳定的(没有正在执行的配置变化)，最终所有的客户端可以会的一个hash slots -> nodes的映射表，这样clieng就可以直接定位到正确的节点而不需要重定向、路由或其它单点故障，从而使整个集群更有效率。
Redis client必须能够正确处理 ASK 重定向，否则就不是一个完整的Redis Cluster client。

3.2 Cluster live reconfiguration


Redis cluster支持在运行时添加和删除节点。这可以抽象成如下操作：将一个hash slot从一个节点转移到另一个节点。这意味着，同样的机制可以被用作集群rebalance、添加节点、删除节点等。

添加节点：将一个新的空节点添加到集群病从已有节点转移一系列slot set到该新节点。
删除节点：将待删除节点的所有hash slot转移到其他节点。
节点rebalance：将给定的一系列hash slots在节点间移动。


该实现的核心是为集群提供移动hash slots的能力。从一个特别的角度来看，一个hash slot就是一系列的keys，因此在resharding期间，Redis Cluster真正做的就是将keys从一个实例转移到另一个。移动一个hash slot意味着将那些正好hash到该hash slot的所有keys进行移动。
为了更好的理解这一过程，我们先看一看我们用来操作slots转移的 CLUSTER 子命令。
以下cluster子命令可以被用来达到上述目的：


CLUSTER ADDSLOTS slot1 [slot2] ... [slotN]

将指定的slot分配给redis节点
所有指定的slot必须是集群内未分配的，否则命令执行失败
主要用于：

新建集群的hash slot配置
修复被损坏的集群时分配未指定的slots


不建议直接使用，应该在集群编排应用程序中使用，比如redis-trib.rb



CLUSTER DELSLOTS slot1 [slot2] ... [slotN]

使 指定的cluster节点 忘记某个主节点正在负责指定的hash slots
删除成功的slot在该节点将进入unbound状态等待分配
在该 指定的cluster节点 删除hash slots后将使该节点进入cluster_state:fail状态，所有针对该节点的redis操作都将失败。

每个节点必须知道所有16384个hash slot的分配，才能对外提供服务。
多节点之间的slot分配信息可能因为delslots/addslots而不同，造成 集群失步！！！



当从其他节点接收到一个心跳包并得知该hash slot已被其他节点负责后，会重新建立关系
极少被使用，建议只用在debug场景中
目前redis-trib.rb中没有使用该命令


CLUSTER SETSLOT slot NODE node
CLUSTER SETSLOT slot MIGRATING node
CLUSTER SETSLOT slot IMPORTING node


前两个命令，ADDSLOTS和DELSLOTS，只是用来简单地在redis节点上分配或移除slot。分配slot意味着告诉一个master node他讲负责保存和服务指定hash slot的内容。
在分配了hash slots之后，节点会通过gossip协议在集群中传播这些信息。
命令ADDSLOTS通常用在创建一个新cluster时为每个master节点指定16384个hash slots的一个子集。
命令DELSLOTS主要用在手动修改集群配置，或者用在调试相关的任务：在实际场景中这个命令极少被用到。

当一个slot被设置为MIGRATING，如果某个命令的keys在当前节点存在，那么节点将会接受对该slot的所有操作；如果有节点不存在，就会使用 -ASK 重定向到迁移的目标节点。
当一个slot被设置为IMPORTING，则如果某个命令紧跟着一个ASKING命令，那么该命令将会被执行。如果client没有给出ASKING命令，该操作将会被-MOVED重定向到它真正的hash slot所有者。


为了更好地理解这一过程，我们看看下面的hash slot迁移的例子。假设我们有两个Redis master节点，称为 A 和 B。我们希望将hash slot 8从A转移到B，因此我们发出如下指令：

向B发送： CLUSTER SETSLOT 8 IMPORINT A

向A发送： CLUSTER SETSLOT 8 MIGRATING B



所有其他节点在收到一个对属于hash slot 8的key的查询时，将会继续向client指出重定向到A。

所有对现有keys的查询将会被"A"处理
所有在"A"上对不存在的keys的查询都会被"B"处理，因为"A"会将client重定向到"B"。


这样，我们不会再在"A"上创建新keys。同时，有一个称为"redis-trib"的特别的程序，它通常被用来resharding以及Redis Cluster配置，会对所有hash slot 8中已经存在的keys进行迁移。这会用到如下命令：

CLUSTER GETKEYSINSLOT slot count


上述命令将会返回指定hash slot中的 count 个keys。对返回的每个key，"redis-trib"向"A"发送一个MIGRATE命令，这会将指定的key从"A"自动迁移到"B"。在迁移过程中，"A"和"B"两个实例都会被锁定很短的时间以确保没有竞争条件。这是MIGRATE的工作方式：

MIGRATE target_host target_port key target_datebase id timeout


命令MIGRATE将会连接对端实例，序列化并发送该key，一旦收到对方的OK回复，就会在本地删除旧的key。从外部client角度看，在任意给定时间，一个key只会存在于A或者B。
在Redis Cluster中，没有必要指定一个0以外的database，但是MIGRATE是一个通用命令，它也用于非cluster环境。MIGRATE命令针对移动复杂key，比如很大的list，做过优化，以便能够尽可能快地在集群间移动key。尽管如此，如果使用redis的应用对延时由要求，对存在大key的集群进行重新配置仍然被认为是个不明智的举动。
当歉意过程最终结束，SETSLOT <slot> NODE <node-id>命令将会被发送到迁移涉及的两个节点，以便将他们的slots状态设置回普通状态。同样的命令通常也会被发送到集群中所有其他节点，以避免新配置在集群间自然传播的等待时间。

3.2.1 CLUSTER SETSLOT命令

MIGRATING

CLUSTER SETSLOT <slot> MIGRATING <destination-node-id>
将一个hash slot设置为migrating状态
当一个slot处于migrating状态时，

如果一个命令的key存在，则执行该命令
如果一个命令的key不存在，则产生一个ASK重定向
如果一个命令包含多个key

如果所有key都存在，则执行
如果只有部分key存在，则产生一个TRYAGAIN错误提示等迁移完成后再试 (在笔者4.0本地测试环境中，只会产生ASK错误，留待继续研究)






IMPORTING

CLUSTER SETSLOT <slot> IMPORTING <source-node-id>
将一个hash slot设置为importing状态
当一个hash slot处于importing状态时，

对所有命令，都会产生一个 MOVED 重定向
如果命令紧跟在ASKING命令之后的命令，则会被按照如下规则执行：

如果在某个key迁移前，在importing方通过asking插入该key，则会造成后续migrating失败
新key只能在target(imporing一方)创建
对于已经完成迁移的key，命令可以背正确执行并保证一致性






STABLE

CLUSTER SETSLOT <slot> STABLE
清除当前hash slot的migrating/importing状态。
所有已经通过MIGRATE命令转移过的key无法恢复


NODE

该子命令的语法最为复杂。它将hash slot与特定节点相关联，但是该命令只能在特定条件下才会被执行并且根据slot状态会产生不同的结果：

如果当前hash slot的owner是接受命令的节点，此时尝试通过本命令将slot分配给一个不同的节点，则

如果该slot为空，执行成功
如果该slot含有任何key，泽返回错误


如果slot处于migrating状态的节点接收到该命令，当该slot被assign给其他key之后，清除migrating状态
如果slot处于importing状态的节点接收到该命令，则并且将这个slotassign给自己（通常发生在针对某个hash slot的resharding结束时），则该命令产生如下影响：

清除imporing状态
如果本节点的config epoch在及群众不是最大的，泽产生一个心的值并将这个config epoch分配给自己。这样相对之前由failover或slot迁移产生的配置，本节点能确保它赢得这个新hash slot的拥有权。






Redis Cluster live resharding过程

在source node上把该hash slot设置为IMPORTING状态
CLUSTER SETSLOT <slot> IMPORTING <source-node-id>

在destination node上把该hash slot设置为IMPORTING状态
CLUSTER SETSLOT <slot> MIGRATING <destination-node-id>

在source node上通过以下命令迁移所有key

CLUSTER GETKEYSINSLOT <slot> <count>
MIGRATE host port "" destination-db timeout [KEYS key [key ...]]


在source或destination节点执行以下命令assign slot到指定节点

CLUSTER SETSLOT <slot> NODE <destination-node-id>





3.3 ASK重定向


在过去的章节，我们已经提及了 ASK 重定向。为何我们不能简单地使用 MOVED 重定向？因为使用 MOVED 意味着我们认为hash slot已经永久的服务于一个不同的节点，并且下一次查询应该尝试那个指定节点，而 AKS 意味着只是下一次查询需要发往指定的节点。
这是有必要的，因为下一次我们可能会对一个属于hash slot 8但目前仍然在"A"上的key进行操作，因此我们总是希望client先尝试访问"A"然后在必要的时候再访问"B"。因为这只会偶尔(resharding或reconfiguring期间)发生在16384个hash slot中的一个上，因此对cluster的性能影响是可以接受的。
我们需要强制规定client的行为，以确保client只有在A已经尝试过之后再尝试B。如果client在查询前发送了ASKING命令，节点B只有在该key所属的slot处于IMPORTING状态时才会接受改命令。
一般来说，ASKING命令会为client设置一个单次标签(one-time flag)，以允许该client可以访问一个IMPORTING的slot一次。
从client视角，ASK重定向的语义如下：

如果收到了 ASK 重定向命令，仅仅将这条查询重定向到某个节点的命令发送到指定的新节点，之后的命令还是继续发送给老的节点。
重定向查询必须以一条ASKING命令开始
(目前)不要在本地将hash slot 8对应的服务节点指向B。


一旦hash slot 8的迁移完成，A会发送一个MOVED消息，而client会永久更新hash slot 8的映射到新的IP:port。逐一，如果一个有bug的client提前修改了本地映射，这也不会成为问题，因为这个client不会在查询前带上ASKING命令，这样B会通过MOVED将client重定向回A节点。

3.4 客户端首次连接和对重定向的处理


一个Redis Cluster client是可以不将slots配置（slot号到服务节点的地址的映射）记录在本地内存中的，这样它只需要随机找一个节点访问，并根据回复的重定向找到正确的服务节点，当然这样的client是非常没有效率的。
Redis Cluster client应该通过缓存slots配置而变得尽可能聪明。当然，这个配置并非必须总是最新的。因为跟错误的节点通信后会简单地获得一个重定向，并且这会出发一次客户端视图的更新。

Clients通常需要在下面两个场景中获取一次全量的slots/nodes映射信息：

在启动阶段为了生成slots配置的初始化信息
在接收到一个MOVED重定向信息时


逐一client在接收到MOVED重定向时，可以只更新单个slot，当然这通常不是很有效率，因为一般来说，每次配置变动通常会设计多个slots(比如发生了一次slave晋升，则所有该节点服务的slots都会被重新映射)。而在收到MOVED重定向时重新获取所有slots的映射处理起来更为简单。
为了获取slots配置，Redis Cluster除了提供了CLUSTER NODES命令外，还提供了一个新选择，这个新的命令只提供了client需要的信息，并且不需要client(对接收到的数据)进行解析。
这个新命令就是CLUSTER SLOTS，它提供了一个slots范围数组，并关联了服务于对应范围的master和slave节点。

下面是CLUSTER SLOTS输出的例子：

127.0.0.1:7000> cluster slots
1)　1)　(integer) 5461
　　2)　(integer) 10922
　　3)　1) "127.0.0.1"

2) (integer) 7001
　　4)　1) "127.0.0.1"
　　　　2) (integer) 7004
2)　1)　(integer) 0
　　2)　(integer) 5460
　　3)　1) "127.0.0.1"
　　　　2) (integer) 7000
　　4)　1) "127.0.0.1"
　　　　2) (integer) 7003
3)　1)　(integer) 10923
　　2)　(integer) 16383
　　3)　1) "127.0.0.1"
　　　　2) (integer) 7002
　　4)　1) "127.0.0.1"
　　　　2) (integer) 7005  d

更多对该命令的解释请参考 CLUSTER SLOTS
该命令(CLUSTER SLOTS)不保证可以反悔16384 slots中的所有信息，如果slots配置缺失，clients应该将其初始化未NULL，并在用户尝试在这些未分配slots上执行命令式上报错误。
在返回一个错误给调用者之前，当一个slot被发现没有分配，client应该再次尝试获取slots配置以检查当前cluster已经正确地配置了。

3.5 多key操作


对拥有相同hash tag的key，总是可以使用multi-key操作
当resharding时：

如果此时所有的key都正好处于相同的节点，则可以成功进行multi-key操作
否则，multi-key操作不可用，产生一个 "-TRYAGAIN" 错误。



3.6 用slave节点扩展读操作


slave节点默认不可读，对slave的读操作将产生 MOVED 错误
通过READONLY命令，可以将slave节点设置为可读
通过READWRITE命令，可以清除该节点的只读flags值

4 容错

4.1 心跳包和gossip消息


Redis Cluster节点持续地交换ping和pong包。这两种包拥有相同的结构，并且都会携带重要的配置信息。它们唯一的不同就是消息的类型字段。我们通常把ping和pong包统称为心跳包(heartbeat packets)。
通常节点发送的ping包将会触发接收者回复一个pong包。当然这并不总是必须的。也有可能节点会向其他节点直接发送包含重要配置信息的pong包，而不需要触发回复。这是有用的，例如，在将自己的心配置尽可能快地广播出去。
通常，每一秒里，一个节点会随机挑选一些节点并发送ping包，因此每个节点(指定时间内)发送的ping包(以及接收到的pong包)的总是是一个常熟，不管集群中有多少节点。
当然，每个节点一定会主动ping那些自己在NODE_TIMEOUT/2时间内没有发送过ping或从之接收过pong的节点。在NODE_TIMEOUT耗尽之前，节点同样会尝试重新连接哪个节点，以确保这不是由于当前TCP连接问题造成的。
如果NODE_TIMEOUT被设置为一个较小的数字，同时节点数目又很大，从全局看，交换的消息数目会是很可观的，因为在NODE_TIMEOUT一半的时间内，每个节点会尝试ping其他所有节点。
例如，在一个拥有100个节点的集群里，如果把节点过期设置为60秒，那么每个节点在30秒内将会发送99个ping，也就是每秒3.3个ping。考虑到有100个节点，也就是集群内部每秒会生成330个ping包。
还是有一些办法来降低消息总数的，然而到目前为止还没有人报告因为Redis Cluster故障检测而导致的带宽问题。也必须注意，即使在上面的例子中，每秒330个包交换也是被平分到100个不同的节点上的，因此对每个节点来说接收到的流量是可以接受的。

4.2 心跳包内容


ping和pong除了包含一个同样会被用于所有其他类型的包(例如请求failover选票的包)的包头(header)外，还有一个特别的Gossip Section。Gossip Section只存在于Ping和Pong包中。
通用包头包含了以下信息：


发送者的Node ID——这是一个160bit随机字符串它在一个节点第一次被创建时生成，并在该Redis Cluster节点的整个生命周期中保持不变。

发送者的currentEpoch和configEpoch字段——由Redis Cluster用来加载分布式算法（下一节中会详细描述）。对slave，configEpoch就是它的master的configEpoch。

发送者的node flags——用来表明节点是slave，还是master，以及其他一些由单bit表示的信息。

发送者的hash slots的bitmap——如果是一个slave，则代表其master的hash slots的bitmap

发送者的TCP和port——（port指的是接收普通命令的端口，即+10000就是redis cluster bus端口）

发送者视角下的集群状态——down或者ok
发送者的master节点ID（如果这是一个slave）


ping和pong包同样包含一个gossip段。该段为接收方提供了这样一个视图——发送方节点视角的集群中其他节点的状态。Gossip段只包含了发送方知道的一系列其他节点中的随机几个节点的信息。Gossip段中包含的节点的数目与集群大小成正比。
每个被放入gossip段的节点包含了如下字段：
- Node ID
- 该节点的IP和port
- 该节点的Node flags
Gossip段允许接收方可以得到发送方视角的集群状态。这在故障检测和节点发现时很有用处。

4.3 故障检测


Redisu Cluster故障检测用来识别合适一个master或slave节点对集群中多数节点来说不再可达(no longer reachable)并在之后提升一个slave节点为master。当无法进行slave晋升时，集群将会被设置为error状态，并停止接受客户端的命令。
之前已经提及，每个节点持有每个已知的其他节点的一系列标签。有两个标签用于故障检测，他们是PFAIL和FAIL。PFAIL标签意味着 可能故障 ，是一个不需要确认的故障类型。FAIL意味着一个节点已经失败，他必须在一段固定的时间内由多数master进行确认。
PFAIL flag (possible failure)

当节点发现某个节点失联超过NODE_TIMEOUT时间后，就会将该节点标记为PFAIL。无论master还是slave都可以姜其他节点标记为PFAIL，而不管对方的类型。
对一个Redis Cluster节点来说，不可达概念的定义为，我们有一个 活跃的ping（指的是我们发送出但是没有接收到对方回复的ping）挂起超过了 NODE_TIMEOUT。为了让这一机制正确工作，NODE_TIMEOUT必须大于一个网络正常往返的时间。为了增加可靠性，在NODE_TIMEOUT时间过去一半时，如果节点还没有得到回复，它会尝试重新连接其他节点。这一机制确保了连接保持活跃，因此损坏的链接通常不会到之错误的在节点间报告失败。


FAIL flag

PFAIL flag只是每个节点针对其他节点状态的本地信息，它不足以被用来触发slave晋升。为了确认一个节点确实down了，PFAIL条件必须升级为FAIL条件。
每个节点发出的gossip消息中会随机包含一部分自己已知的其他节点的状态信息，每一个节点最终会接收到每个其他节点的一系列node flags。这为每个节点提供了一种机制来向其他节点发送自己检测到的节点失效事件。


当如下一系列条件满足时，PFAIL条件就会升级称为FAIL条件：

某个节点(我们称之为A)，已经将另一个节点B标识为PFAIL
节点A通过gossip段收集集群中多数master(majority master)对该节点的标注
多数master在 NODE_TIMEOUT * FAIL_REPORT_VALIDITY_MULT (在当前redis实现中，FAIL_REPORT_VALIDITY_MULTI被设置为2且不可配置) 这段时间内将节点A标注为 PFAIL 或 FAIL。


如果上述条件满足，则节点A将：

把失联节点标记为FAIL

发送一个FAIL消息给所有可达节点



FAIL消息将会强制所有接收到的节点将失联节点（节点B）标记为FAIL，而不管当前自己是否已将其标记为PFAIL

【注意】FAIL flag总是单向的，即一个节点可以从PFAIL变为FAIL，但是不能反向转变。FLAG标签只有在以下情况下才会被清除：

节点是slave，并且重新可达。这种情况下 FAIL 标签可以清除因为slave节点不会发生故障转移。
节点是master且没有服务于任何slot，重新可达。这种情况下，FAIL标签可以被清除，该master继续等待被配置后加入集群
节点是master，重新可达，且在一段较长时间 (N倍NODE_TIMEOUT) 内 没有被检测到有slave节点被晋升。显然此时它应该被作为master将重新加入集群。


在从 PFAIL -> FAIL 转变的过程中，使用了弱一致机制(weak agreeement)：

节点在一段时间内收集其他节点的视图(views)，所以即使多数派master需要达成一致，事实上这只是说明我们在不同的时间，从不同的节点收集到了这一结果，我们既无法确定，也无法要求，在什么时刻获得了多数master的一致结果。然而，因为我们会抛弃老旧（过期）的失败报告，所以多数派master一定是在某个时间窗内对某个节点的失败达成了一致。
即使每个节点检测到了 FAIL 条件并通过 FAIL 消息强制集群中其他节点接受该条件，还是无法保证消息可以倍所有节点接收到，因为此时可能因为分区问题导致某些节点不可达。


当然，redis cluster的失败检测还有一个活性需求(liveness requirement)：最终所有的节点需要(should)对一个给定节点的状态达成一致。下面是两个可能由(集群)脑裂引起的场景(case)：一些少数派节点认为某个节点已经 FAIL，或是一些少数派节点认定某个节点不在FAIL状态。这两种状态下集群对某个节点最终一定会(在集群全局)有一个唯一的视图(view)。

场景1： 如果多数派masters通过失败检测及其产生的影响链，最终将一个节点标记为FAIL，所有其他节点将最终将会把这个master标记为FAIL，因为在指定的时间窗内，集群里会有足够多的失败被报告。
场景2： 如果只是少数派master将一个节点标记为FAIL，slave晋升将不会发生，所有节点将会根据上述的FAIL状态清除规则清除该节点的FAIL状态（例如通过"在N倍NODE_TIMEOUT内没有晋升动作"这一条规则）。



FAIL flag只是用来作为一个触发机制，它将触发执行slave晋升算法的安全部分，以便将slave晋升。理论上slave独立地运作并在发现它的master不可达后启动一次晋升，并在多数masters可以触达该master时等待其他master拒绝认可。然而，PFAIL -> FAIL 状态变迁、弱一致、 在cluster可达节点间通过FAIL消息强制状态的生成，这些额外的复杂度的引入，在实践上是有它的优势的。因为这些机制的引入，使得集群(可以意识到自己)在处于一个error状态下，所有节点可以拒绝写入操作。从从使用redis cluster的应用的角度看，这是一个必要的特性。同时这样也可以避免由于slave自己的问题导致无法连接master，进而导致错误的选举尝试。

5 配置执行，传播，和故障转移 (Configuration handling,propagation, and failovers)

5.1 集群当前代(cluster current epoch)


Redis Cluster使用了一个类似Raft算法的"term"的概念，称为"epoch"(代)。它用来为事件提供递增的版本号。当多个节点提供了相互冲突的信息，它让其他节点可以正确的理解哪一个状态是最新的。

currentEpoch是一个64bit无符号整数。
在节点创建时，所有的Redis Cluster节点，包括slave和master节点，都把自己的currentEpoch设置为0。
每当从其他节点接收到一个包时，如果发送方的epoch(包含在cluster总线消息头中)大于本地节点的epoch，则本地节点将自己的currentEpoch更新为发送方的epoch。
因为这一语义，最终所有节点将会认同集群中拥有最大configEpoch的节点(提出的主张)。（【译注】current epoch是用来标识集群epoch的，集群epoch取自所有节点中configEpoch最大的那个节点的configEpoch）
该信息被用在：当集群状态发生变化，并且一个节点正在请求其他节点的同意来执行一些操作时(比如slave晋升)。
当前的实现里，currentEpoch仅仅被用在slave晋升。简单来说，epoch是集群的一个本地时钟，拥有大epoch的消息总是能赢得拥有相对小的epoch的消息。

5.2 配置代(Configuration epoch)


每个master总是在ping和pong包中向它的slave广播自己的configEpoch，和其服务的hash slots的bitmap信息。
当一个新master节点被创建时，configEpoch被设置为0.
一个新的configEpoch将会在slave选举中被创建。在尝试替代失败的master时，slave会增加他的epoch并尝试得到多数masters的授权。当一个slave被选中，一个新的唯一的configEpoch会被创建，同时该slave会使用这个新的configEpoch并转变为一个master。
后续章节将会解释，当不同的节点主张存在分歧时（可能由于网络分区或节点失败导致），configEpoch时如何帮助解决冲突的。
slave节点同样会在ping和pong包中声明configEpoch，此时的configEpoch是他的master在上一次包交换中携带的configEpoch。这允许其他实例检测到这个slave有一个老的配置并且需要更新（master节点将不会向一个拥有旧配置的slave授权选票）。
每当一些已知节点的configEpoch发生变化，它就会被所有收到这条信息的节点永久地保存在各自的node.conf文件里。同理，currentEpoch也会被保存。Redis保证会在执行下一个操作前保存这两个变量并同步到磁盘。

configEpoch的值在failover时使用一个简单的算法来保证产生一个新的、递增的、唯一的值。

5.3 Slave选举和晋升


选举和晋升总是由slave节点发起和处理，在这期间，master节点会提供一些帮助，它们会为晋升哪个slave而进行投票。当一个master在至少一个slave的视图中处于FAIL状态，并且该slave已经要求晋升为master时，slave选举就会发生。
为了将自己晋升为master，slave需要开启一个选举并赢得该选举。如果一个master处于FAIL状态，那么它的所有slave都可以开始一个选举，但是只有一个slave会赢得最终的选举并晋升为master。
当以下条件符合时，slave将开始一个选举：

该slave的master处于FAIL状态
master正在为至少一个以上slot服务
slave与master的复制连接断开时间少于一个给定值，这用来保证晋升的slave的数据是尽可能新的。这个值可以由用户配置。


为了被选中，一个slave首先要做的就是增加自己的currentEpoch计数，并向master实例请求选票。
Slave通过向集群的每一个master节点广播一个 FAILOVER_AUTH_REQUEST 包请求选票。然后它会在不超过2倍NODE_TIMEOUT时间内等待所有master的回复（一般至少等2秒）。
一旦一个master将选票投给了某个slave，主动回复了 FAILOVER_AUTH_ACK，在时间窗 NODE_TIMEOUT * 2 以内，它就再也不能向该master的任何其他slave投票了。从安全性保证上来说，这一规则不是必须的，但是它有助于避免多个slave在几乎相同的时间内同时被选中（哪怕它们的configEpoch是不同的），而这显然不是期望得到的结果。
slave抛弃那些epoch比自己在发送选票请求时的currentEpoch小的AUTH_ACK。这保证了它不会计算用于上一次选举的选票。
一旦slave受到了多数master的ACK，它就赢得了选举。否则如果在 2 * NODE_TIMEOUT  时间窗（至少2s）内没有得到多数的回复，选举就会中止，而一次新的尝试将在 NODE_TIMEOUT * 4 (至少4s)之后尝试。

5.4 Slave排序


一旦一个master处于FAIL状态，一个slave会在尝试选举前等待一小段时间。等待的时间按照如下公式计算：
DELAY = 500 milliseconds + random delay between 0 and 500 milliseconds + SLAVE_RANK * 1000 milliseconds

固定的DELAY用来保证FAIL状态扩散到整个集群，否则slave可能会在多数master不知道该FAIL时请求选举并被拒绝投票。
随机的DELAY用来使slave之间异步，避免在同一时间同时发起选举。

SLAVE_RANK是该slave针对它从master获得的备份数据的总量的排序。在master失败之后，slave之间通过交换消息来创建一个（最大努力）排序：拥有最新备份offset的slave获得排序0，第二个更新为1，以此类推。这样最新的slave会尝试最先开始选举。
排序的顺序并没有严格强制。如果一个拥有最高排序的slave再选剧终失败了，其他slave会很快进行重试。
一旦某个slave赢得了选举，它就获得了一个新的唯一的递增的configEPoch，这个configEpoch会比所有其他现存的master更大。它会在ping和pong包中作为master广播自己，同时提供自己的服务slot和比之前的master更大的configEpoch。
为了加速重配置，新master会向集群内所有节点直接发送pong包。
当前不可达的节点，最终也将会被重新配置，比如它重新连接后接收到了其他节点发来的ping和pong包，或者通过它自己发送的心跳包被其他节点检测到已过期并回复了UPDATE包之后。
其他节点会检测到有一个新的master服务于之前master相同的slots，但是拥有一个更大的configEpoch，之后它们会更新自己的配置。旧master的其他slave（包括重新接入的旧master自己），不但会更新配置，而且会重新从新的master同步所有数据。

5.5 Masters回复slave选举请求


Master接收到slave的FAILOVER_AUTH_REQUEST后就会开始一次选举。
只有符合如下条件，master才会授予选票：

master针对每一个epoch只会投票一次，一旦投票后就会拒绝所有更小的epoch：每个master有一个lastVoteEpoch字段，并且会拒绝对currentEpoch小于该值的请求投票。一旦master对投票请求回复确认，lasterVoteEpoch就会同步更新并安全地保存到磁盘。
只有当slave所属的master被标记为FAIL时，master才会投票给该slave
如果一个FAIL_AUTH_REQUEST的cureentEpoch的值小于master的currentEpoch，那么该选举请求将被忽略。因此，master的回复总是与FAIL_AUTH_REQUEST拥有相同的currentEpoch。如果同样的slave再次请求选票，并增加了currentEpoch，这可以保证针对旧请求的DELAY的投票不会在新投票请求中被接受。




如果一个master在上一轮选举中投票过，那么它在NODE_TIMEOUT*2时间窗内，不会为该master的任意slave再次投票。这不是严格的需求，因为两个slave不可能在一个相同的epoch中同时获胜。然而，在实践中，它保证了当一个slave被选举后，它拥有足够的时间通知其他slave并避免其他slave赢得新一轮选举的可能性，否则这会造成有一次没有必要的failover。
master不会做任何尝试来保证选出最好的slave。如果slave的master处于FAIL状态，且master没有在当前的term(任期，代?)中投票过，那么它一定会将授予自己的投票。最好的slave总是更可能启动一次选举并在其他slave之前赢得选举，因为由于它拥有更高的排名，它总是会先于其他slave发起选举。
当一个master拒绝为某个slave投票，那么它会简单地忽略该请求，而不会发出一个负面的响应。
master不会投票给这样的slaves，它们发送的configEpoch小于master表中为slave宣称的slot服务的master的configEpoch。记得之前提起过，slave发送的消息中使用它的master的configEpoch，以及它的master服务的slots。这意味着请求选票的slave必须拥有它打算failover的master的slot配置，并且这个配置需要比授权选票的master更新或至少相等。

5.6 一个分区问题中epoch配置有效性实际例子


本节解释了epoch概念如何使slave在晋升过程中更加能够容忍分区(partitions)错误

一个master无限期失联。它拥有三个slave：A，B，C
Slave A赢得了选举并晋升为master
一个网络分区问题导致A对集群的多数节点不可用
SLave B赢得了选举并晋升为master
一个分区问题导致B对集群多数节点不可用
网络问题修复，并且A也恢复可用。


此时，B忽然down机并且A恰好以master身份恢复可用(实际上UPDATE消息会及时重新配置它，但我们假设所有的UPDATE消息也丢失了)。这时，slave C尝试选图来替代B。接下来：

C尝试选举并成功，因为对多数master来说，它的master的确挂了。它将获得一个新的增加了的configEpoch。
A不能生成自己时这些hash slots的master，因为其他服务于相同hash slots的节点已经有了一个更大的configuration epoch。
所以，所有的节点将会更新它们的配置表并将这些hash slots分配给C，集群可以继续运行。


在下一节中，你会看到，一个旧节点重新加入cluster，它将立刻被通知到配置的变更，因为一旦它主动ping其他节点，接收方就会检测到它有一个陈旧的集群信息，并向它发送一条UPDATE消息。

5.7 Hash slots配置传播


Redis Cluster中很重要的部分就是提供一种机制，用来传播集群中哪个节点服务于哪些hash slot信息。这无论是在集群启动还是在slave晋升时都非常重要。
同样的机制保证了节点在不限期遇到分区问题后能以一种明智的方式重新加入集群。

有两种生成hash slots配置的方式：

heartbeat消息。ping和pong消息的发送者总是添加自己或自己的master所服务的hash slots信息。

UPDATE消息。因为每个heartbeat包都有发送方的configEpoch和其服务的hash slots，如果接收方发现发送方的信息过期，它姜发送一个包含了新信息的包，强制过期节点更新自己的信息。



心跳包消息或UPDATE消息的接收方使用一种简单的规则来更新表映射中的hash slots到对应的节点。当一个新的Redis Cluster 节点被创建，它的本地hash slot表被初始化为NULL，这样每个hash slot就不会被绑定到任何节点。它看上去就像这样：

0 -> NULL
1 -> NULL
...
16383 -> NULL



配置传播的规则如下：


Rule 1: 如果一个hash slot没有被分配(NULL)，这时如果有一个已知节点声明了该slot，则修改本地hash slot table并将声明的hash slots关联到该节点。

当一个新的cluster被创建，系统管理员需要手动分配(使用命令CLUSTER ADDSLOTS，一般通过redis-trib命令行工具，或其它类似的工具)所有的slots给它的master，这一信息会很快在整个集群中传播。


Rule 2: 如果一个hash slot已经被分配，并且一个已知节点广播消息中的configEpoch比当前拥有该slot的master的configEpoch更大，则重新绑定hash slot到新节点。
因为rule 2，最终所有节点一定会通过节点间的消息广播就configEpoch最大的节点获得slot的拥有权达成一致。
这一机制被称为 last failover wins(最后故障转移者胜)

同样的情况也发生在resharding(重分片)。当一个节点importing一个hash slot并完成，它的configuration epoch将被增加以确保改动会被扩散到整个集群。



5.8 UPDATE消息，a closer look


Node A在一段时间后重新加入集群。它将发送heartbeat包并声明自己服务于hash slots1和2，而configuration epoch为3。所有更新了最新集群信息的接收者却看到相同的hash slots已经被关联到了节点B，并且节点B拥有更高的configuration epoch。因此它们会发送一条UPDATE消息给A，同时带上这些slots的心配置。A将会根据rule2更新自己的配置。

5.9 节点如何重新加入集群


同样的机制也被用于一个节点重新加入集群。继续上面的例子，节点A会被通知hash slots1和2现在被节点B服务。假设A之前只服务这两个hash slots，那么目前A服务的slots数目就会将为0。因此A将会重新配置为新master的slave。
实际上遵循的规则比上述场景更复杂一些。同上这在A经过很长时间的断开后重新加入集群时更容易发生，这时A发现它之前服务的hash slots目前被多个节点服务，例如hash slot 1由节点B服务，而hash slot 2被节点C服务。
所以真是场景中Redis Cluster节点角色切换的规则是：

一个master节点(在重新加入集群后)将会修改自己的配置为：自动从属于(be slave of)之前该master服务的最后一个hash slot的新master


通过重新配置，最终该节点服务的所有hash slots将会被丢弃，且该节点会被重新配置。
对slave而言也是一样的：它们会将自己配置为之前master的最后一个hash slot的新master的备份节点。

5.10 备份迁移(Replica migration)


Redis Cluster实现了一个称为备份迁移(replica migration)的概念(特性)，用来提升系统可用性。在一个由master-slave组成的集群中，如果slaves和masters之间的映射关系是固定的，那么集群的可用性随着时间的推移，姜会因为单个节点的失败而变得越来越差。
例如，在一个每个master拥有一个slave的集群中，集群在任意一个master或slave失败后还能继续运作，但是如果master和slave同时失败，则集群就无法继续使用了。不幸的是，由于硬件或软件的问题，总是会有一类错误造成但个节点的失败，并且随着时间而积累。比如：

Master A只有一个slave A1
Master A发生了故障了，A1被晋升为新的master
三小时后，由于各种问题，A1也发生了故障。此时已经没有其他slave可以被晋升了，因为A和A1已经全部宕机。Cluster这时会进入error状态而不能继续提供服务。


如果masters和slaves之间的映射关系是固定的，那么唯一能保证集群更稳定的方法就是为每个master添加更多的slave，然而这样做的成本非常昂贵，因为它需要添加更多的Redis实例，以及更多的内存等。
另一个可用方案是在集群中创建不对称(slave)，并允许集群布局随着时间自动发生变化。例如，集群可以拥有三个masters：A，B，C。A和B分别有一个slave，A1和B1。而master C与之不同，它拥有两个slaves：C1和C2。
备份迁移是一种slave自动重配的过程，它用来将备份节点迁移到当前已经没有可用slave的master上。通过备份迁移，上述场景变为：

Master A发生故障。A1被晋升。
C2迁移为A1的slave，否则A1将会没有任何slave。
三小时后A1也发生了故障
C2被晋升为新的master以替代A1
此时，集群还能正常提供服务。



5.11 备份迁移算法


歉意算法不需要使用任何的协商(agreement)，因为在Redis Cluster中，slave布局不是集群配置的一部分，他不需要通过config epoch提供任何的一致性和/或版本化保证。相反，在master没有回归之前，它使用了一种避免slave块迁移(mass-migration)的算法。这种算法保证了最终(一旦集群配置变得稳定后)每个master至少可以拥有一个slave。
这就是该算法的工作方式。我们先从『什么是一个好的slve(a good slave)』的定义开始：一个好的slave指的是，从某个节点角度，处于非FAIL状态的slave节点。
每个slave一旦检测到目前至少有一个master没有好的slave，就会触发该算法。然而，在所有检测到这一情况的slave中，只有一个子集会真正行动(act)。这个子集实际上通常只有一个slave，除非不同的slaves在特定时刻对其他节点的失败状态有一个略微不同的视图。

行动slave(acting slave) 是一个这样的slave：它是集群中拥有最多slave的master的一个ID最小的slave。
例如，对于这样的一个集群，集群中有10个masters每个拥有1个slave，有2个masters每个拥有5个slaves，那么即将发生迁移的slave是 —— 在2个拥有5个slaves的master中 - 拥有最小节点ID的那个。确认这点不需要使用任何协商，只是有可能在集群配置不稳定的时候，发生竞争条件，这时多个slave会认为自己是拥有更小节点ID的非失败节点 (实际上这很难发生)。如果发生了，结果就是会有多个slaves被迁移到同一master，这本身并没有害处。如果竞争导致失去slave的master变成孤节点，一旦集群稳定后，该算法会再次被执行并转移一个可用slave给该节点。
最终每个master将会拥有至少一个slave。然而，通常行为都会是，只有一个slave从其拥有多个slaves的master迁移到另一个孤master上。
算法会被一个称为cluster-migration-barrier的用户配置参数控制，该参数指定了在发生备份迁移之前，一个master必须拥有的好slave的数目。例如，如果该参数被设置为2，那么只有在某个master拥有2个可工作slaves时，其中一个slave才能尝试迁移(【译注】通过检查代码，应该是拥有2个或2个以上可工作slaves时，其中一个可以发生迁移)。

5.12 configEpoch冲突解决算法


在failover阶段，slave晋升中会产生一个新的configEpoch值，并且这个值被保证是唯一的。
然而如果有两个不同的events通过不安全的方式分别生成新的configEpoch值，它们将会仅仅把本地的currentEpoch自增并寄希望于同一时间内不会有冲突。比如，系统管理员同时触发了两个events：

带TAKEOVER选项的CLUSTER FAILOVER会手动晋升一个slave节点到master，并且不需要多数master的同意。这是个有用的操作，比如，在多数据中心创建时
为集群rebalancing而迁移slots，为了性能考虑，这同样会在节点内产生新的configuration epochs而不需要其他节点的同意。


特别的，在手动重分片(reshardings)期间，当一个hash slot被从节点A迁移到节点B时，重分片程序将强制要求B更新它的configuration epoch为集群中的最大值加1（除非此时B的configuration epoch已经是集群中最大的），而不需要请求其他节点的同意。
通常一次真实世界的resharding会设计数百hash slot的迁移（尤其在较小的集群里）。在resharding期间，如果每一个slot迁移都需要为生成新的configuration epoch而请求其他节点的同意，这将会很没有效率。而且，这回要求集群中的节点每次都通过fsync来保存这一新的配置（configuration epoch）。因为这些原因，我们采用如下行为处理reshardings时的configEpoch：我们只需要在第一个hash slot迁移时生成一个新的configEpoch，这在生产环境中会更加有效率。
然而，因为上述两种情况，有可能发生（虽然很难）多个节点拥有相同的configuration epoch的情况。一个resharding操作有管理员发起，同时一次failover恰巧发生，外加一点坏运气，如果各自的epoch没有在足够短的时间内扩散开，这将会会导致currentEpoch冲突。
更进一步来说，软件bug和文件系统破坏，这同样可能导致多个节点拥有相同的configuration epoch。
当服务于不同hash slots的master拥有相同的configEpoch时，这不会有什么问题。相对而言，更重要的是，当slave故障恢复一个master时，它需要拥有唯一的configuration epoch。
也就是说，手动干预或resharding会用一种不同的方式改变集群配置。Redis Cluster主要的liveness property要求，slot配置总是汇聚的，因此我们总是期望所有场景下，所有的master节点总是拥有不同的configEpoch。
为了强制达到上述目的，当两个节点以相同的configEpoch结束某个操作时， 冲突解决算法 将会被用来处理这一场景。

如果一个master节点检测到其他master节点正在广播一个像同的configEpoch


并且 如果本节点拥有一个按字典排序更小的节点ID

那么 它会将自己的currentEpoch增加1，并用这一新值作为自己的configEpoch。


如果还有任意数目的节点拥有相同的configEpoch，那么除了拥有最大ID的节点，所有其他节点都将被前移，来保证最终每个节点都拥有一个唯一的configEpoch。
这一机制同样也保证了，在一个新的集群被创建后，所有的节点都会以一个不同的configEpoch开始，尽管它们并没有被用到，因为redis-trib会保证使用CONFIG SET-CONFIG-EPOCH为每一个节点设置不同的ID。
然而，如果因为某些原因一个节点没有被配置，它还是会自动地更新自己的配置到一个不同的configuration epoch（因为有冲突解决算法的保证）。

5.13 节点reset


节点可以被软件重置(software reset)，而不需要重启，以便用于不同的role或不同的cluster。

CLUSTER RESET命令包含连个变种：

CLUSTER RESET SOFT
CLUSTER RESET HARD


命令必须被直接发送给待reset的节点。默认使用soft reset。
下面是一个reset命令所做的操作：

SOFT/HARD: 如果节点是slave，将其转化为master，删除所有数据。如果节点是master且包含keys，则reset失败
SOFT/HARD: 释放所有的slots，重置手动failover状态
SOFT/HARD: 删除该节点上node table中所有其他节点，即该节点不再知道任何其他节点的信息
HARD only: 重置currentEpoch，configEpoch，和lastVoteEpoch为0
HARD only: Node ID变更为一个新的随机ID。


拥有非空数据集的master节点不能被reset，因为一般而言你可能希望先reshard数据到其他分片。或者，在某些特定场景下，比如当一个cluster已经完全被破坏而需要创建一个新cluster时，可以用FLUSHALl先清空数据，然后再reset。

5.14 从集群移除节点


如果希望移除一个节点，从实践上来说，resharding它的所有数据到其他节点（如果这是一个master），然后关闭它，这样是可行的。但是，其他节点姜还会技术该节点的node ID和地址，并会尝试连接它。
因此，当一个节点需要被移除时，我们希望把它彻底从其他节点的node table中一并移除。这可以通过CLUSTER FORGET <node-id>命令来实现。
该命令做了两件事情：

它把该节点从其他节点的nodes table中移除
它设置了一个60秒的禁用期，来防止一个节点使用相同的node id重新加入集群。


上述第二条时必要的，因为Redis Cluster使用gossip来自动发现节点，因此从节点A删除节点X，会导致节点B从节点A处通过gossip重新发现该节点（X）。而通过引入60秒禁用期，Redis Cluster管理工具就可以有60秒的时间来逐一在所有节点删除X，并防止它被重新发现。

6. Publish/Subscribe


在Redis Cluster集群中，client可以在任何节点上订阅(subscribe)消息，也可以向任何节点发布(publish)消息。如果需要，集群会保证发布的消息被正确转发。
当前的实现中，发布的消息将会被简单地广播到所有节点，但有时候他是可以使用Bloom过滤器或其他算法进行优化的。

