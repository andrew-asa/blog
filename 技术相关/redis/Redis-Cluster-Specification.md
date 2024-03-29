# Redis Cluster Specification

## 设计的主要特性和基本原理
### Redis集群目标
+ Redis Cluster是Redis的分布式实现，具有以下目标，按照设计中的重要性顺序排列
    + 高性能和线性可扩展性，最多1000个节点。没有代理，使用异步复制，并且不对值执行合并操作。
    + 可接受的写入安全程度：系统尝试（以尽力而为的方式）保留源自与大多数主节点连接的客户端的所有写入。通常有小窗口可以丢失确认的写入。当客户端处于少数分区时，窗口丢失已确认的写入会更大。
    + 可用性：Redis Cluster能够在大多数主节点可访问的分区中存活，并且每个主节点至少有一个可访问的从节点不再可访问。此外，使用副本迁移，任何从属设备不再复制的主设备将从多个从设备覆盖的主设备接收一个主设备。

    本文档中描述的内容在Redis 3.0或更高版本中实现。
    
+  实施子集
    +  Redis Cluster实现Redis的非分布式版本中可用的所有单个键命令。执行复杂的多键操作（如Set类型联合或交叉）的命令也可以实现，只要这些键都属于同一个节点。
    +  Redis Cluster实现了一个称为哈希标记（hash tags）的概念，可用于强制某些key储在同一节点中。但是，在手动重新分片期间，多键操作可能会在一段时间内不可用，而单键操作始终可用。
    +  Redis Cluster不支持多个数据库存储位置，例如Redis的独立版本。只有数据库0，不允许使用SELECT命令。
+ Redis群集协议中的客户端和服务器角色
    + 在Redis群集中，节点负责保存数据并获取群集的状态，包括将键映射到正确的节点。群集节点还能够自动发现其他节点，检测非工作节点，并在需要时促进从节点变成主节点，以便在发生故障时继续运行。
    + 要执行任务，所有群集节点都使用TCP总线和二进制协议连接，称为Redis群集总线。（Cluster Bus.）每个节点都使用集群总线连接到集群中的每个其他节点。
    + 节点使用八卦（gossip protocol）协议传播有关群集的信息，以便发现新节点，发送ping数据包以确保所有其他节点正常工作，以及发送发出特定条件信号所需的群集消息。
    + 群集总线还用于在群集中传播发布/订阅（Pub/Sub ）消息并协调手动故障转移当用户请求时（手动故障转移是故障转移，不是由Redis群集故障检测器启动，而是由系统管理员直接启动）。
    + 由于群集节点无法代理请求，因此可以使用重定向错误-MOVED和-ASK将客户端重定向到其他节点。理论上，客户端可以自由地向集群中的所有节点发送请求，并在需要时重定向，因此客户端不需要保持集群的状态。但是，能够在key和节点之间缓存映射的客户端可以以合理的方式提高性能。
+ 写安全
    + Redis Cluster使用节点之间的异步复制，上次故障转移赢得隐式合并功能。 （last failover wins implicit merge function）这意味着最后选出的主节点的数据最终将替换所有其他从节点。
    + 在分区期间可能会丢失写入时，总会有一个时间窗口。然而，在连接到大多数主设备的客户端和连接到少数主设备的客户端的情况下，这些窗口是非常不同的。
    + 与少数端执行的写操作相比，Redis Cluster更加努力地保留由连接到大多数主服务器的客户端执行的写操作。以下是导致在故障期间在多数分区中收到的已确认写入丢失的情况示例：
        + 写入可以连接的主节点，但是当节点能够回复客户端时，写入可能不会通过主节点和从属节点之间使用的异步复制传播到从设备。如果主节点在没有写入从节点的时候死亡，则写入将永久丢失。
        + 另一种理论上可能出现写入丢失的故障模式如下：
            + 由于分区，主节点无法访问。
            + 主节点的一个从节点上升为主节点。
            + 一段时间后，主节点又恢复正常。
            + 具有过时路由表的客户端可以在群集将其转换为（新主节点的）从节点之前写入旧主节点
    + 第二种故障模式不太可能发生，因为主节点无法与大多数其他主设备通信足够的时间进行故障转移将不再接受写入，并且当分区被修复时，写入仍然会在少量时间内被拒绝，以允许其他节点通知配置更改。此故障模式还要求客户端的路由表尚未更新。
    + 针对分区的少数端的写入有一个更大的窗口可以丢失。例如，Redis Cluster在有少数主服务器和至少一个或多个客户端的分区上丢失了非常重要的写入次数，因为如果主节点在多数方面进行故障转移，发送给主节点的所有写入都可能会丢失。
    + 具体来说，对于要进行故障转移的主节点，大多数主节点必须至少无法访问NODE_TIMEOUT因此，如果在该时间之前修复分区，则不会丢失写入。当分区持续超过NODE_TIMEOUT时，在少数端执行的所有写操作可能会丢失。但是，一旦NODE_TIMEOUT时间过去而没有与大多数人接触，Redis群集的少数端将开始拒绝写入，因此有一个最大窗口，之后少数群体变得不再可用。因此，在此之后不接受或丢失写入。
    
+ 可用性
    + Redis Cluster在分区的少数节点不可用。在分区的大多数方面，假设每个无法访问的主节点至少有大多数主节点和从节点，在NODE_TIMEOUT时间加上从节点获得选举并故障转移成为主节点所需的几秒钟之后，群集将再次可用（故障转移通常在1或2秒内执行）。这意味着Redis Cluster旨在经受群集中几个节点的故障，但对于需要在大型网络分裂时需要可用性的应用程序而言，它不是合适的解决方案。
    + 在由N个主节点组成的集群示例中，每个节点都有一个从节点，只要单个节点被分区，集群的大多数端将保持可用状态，当两个节点被分开时，将保持可用的概率为1-（1 /（N * 2-1））（在第一个节点出现故障后，我们总共剩下N * 2-1个节点，并且概率为没有副本失败的唯一主人是1 /（N * 2-1））。
    + 例如，在每个节点有5个节点和单个从站的集群中，有一个1 /（5 * 2-1）= 11.11％的概率，在两个节点与大多数节点分开后，集群将不再可用。
    + 由于Redis Cluster称为复制副本迁移的功能（replicas migration），因此复制副本迁移到孤立主节点（主节点不再具有副本）这一事实可以改善许多真实场景​​中的群集可用性。因此，在每个成功的故障事件中，集群可以重新配置从节点，以便更好地抵抗下一个故障。
    
+ 性能
    + 在Redis群集中，节点不会将命令代理到负责给定key值的节点，而是将客户端重定向到key值负责的节点。
    + 最终客户端获得集群的最新表示以及哪个节点处理哪些key子集，因此在正常操作期间，客户端直接联系正确的节点以发送给定命令。
    + 此外，由于多键命令仅限于近键（near keys 指同一个节点保存的键值），因此除了重新分片之外，数据永远不会在节点之间移动。
    + 正常操作的处理方式与单个Redis实例完全相同。这意味着在具有N个主节点的Redis群集中，您可以期望与单个Redis实例相同的性能乘以N，因为设计会线性扩展。同时，查询通常在单个往返中执行，因为客户端通常保留与节点的持久连接，因此延迟数字也与单个独立Redis节点情况相同。
    + Redis Cluster的主要目标是提供极高的性能和可扩展性，同时保留弱但合理的数据安全性和可用性。
    + 为什么避免合并操作
        + Redis群集设计避免了多个节点中相同键值对的冲突版本，就像Redis数据模型的情况一样，这并不总是令人满意的。
        + Redis中的值通常非常大;通常会看到包含数百万个元素的列表或排序集。
        + 数据类型在语义上也很复杂。转移和合并这些值可能是主要瓶颈和/或可能需要应用程序端逻辑的非平凡参与，存储元数据的附加存储器等等。
        + 这里没有严格的技术限制。 CRDT或同步复制的状态机可以模拟类似于Redis的复杂数据类型。但是，此类系统的实际运行时行为与Redis Cluster不同。Redis Cluster的设计旨在涵盖非集群Redis版本的确切用例。
        
## Redis Cluster主要组件概述 
    + key分发模型
    + key空间分为16384个槽，有效地设置了16384个主节点的簇大小的上限（但建议的最大节点大小约为1000个节点）。
    + 集群中的每个主节点处理16384个散列槽的子集。当没有正在进行的群集重新配置时，群集是稳定的（即，散列槽从一个节点移动到另一个节点）。
    + 当群集稳定时，单个节点将提供单个散列槽但是，在网络分裂或故障的情况下，服务节点可以具有一个或多个将替换它的从属节点，并且可以用于扩展读取操作，其中读取陈旧数据是可接受的）。
    + 用于将键映射到散列槽的基本算法如下（读取此规则的散列标记异常的下一段）：
        `HASH_SLOT = CRC16(key) mod 16384`
        
    + 键哈希标签
        + 计算用于实现散列标记的散列槽有一个例外。
        + 散列标记是一种确保在同一散列槽中分配多个key的方法。这用于在Redis群集中实现多键操作。
        + 为了实现散列标签，在某些条件下以稍微不同的方式计算key的散列槽。
        + 如果密钥包含“{...}”模式，则仅对{and}之间的子字符串进行散列以获得散列槽。
        + 但是，由于可能存在多次{或}，因此以下规则可以很好地指定算法：
            + 如果key包含{字符。
            + 如果{右侧有}个字符
            + 如果第一次出现{和第一次出现}之间有一个或多个字符。
            + 然后，不是对密钥进行哈希处理，而是仅对第一次出现{和第一次出现的}之间的内容进行哈希处理。
            + 添加哈希标记异常，以下是Ruby和C语言中HASH_SLOT函数的实现。
    ```
    def HASH_SLOT(key)
    s = key.index "{"
    if s
        e = key.index "}",s+1
        if e && e != s+1
            key = key[s+1..e-1]
        end
    end
    crc16(key) % 16384
end
    ```
    
    C示例代码：
    
    ```
    unsigned int HASH_SLOT(char *key, int keylen) {
    int s, e; /* start-end indexes of { and } */

    /* Search the first occurrence of '{'. */
    for (s = 0; s < keylen; s++)
        if (key[s] == '{') break;

    /* No '{' ? Hash the whole key. This is the base case. */
    if (s == keylen) return crc16(key,keylen) & 16383;

    /* '{' found? Check if we have the corresponding '}'. */
    for (e = s+1; e < keylen; e++)
        if (key[e] == '}') break;

    /* No '}' or nothing between {} ? Hash the whole key. */
    if (e == keylen || e == s+1) return crc16(key,keylen) & 16383;

    /* If we are here there is both a { and a } on its right. Hash
     * what is in the middle between { and }. */
    return crc16(key+s+1,e-s-1) & 16383;
}
    ```
    
+ 群集节点属性
    + 每个节点在群集中都有唯一的名称。节点名称是160位随机数的十六进制表示，在第一次启动节点时获得（通常使用/dev/urandom）。
    + 节点将其ID保存在节点配置文件中，并将永久使用相同的ID，或者至少只要系统管理员没有删除节点配置文件，或者通过CLUSTER RESET命令请求硬复位。
    + 节点ID用于标识整个群集中的每个节点。给定节点可以更改其IP地址，而无需也更改节点ID。群集还能够检测IP /端口的变化，并使用在群集总线上运行的八卦协议进行重新配置。
    + 节点ID不是与每个节点关联的唯一信息，而是唯一始终全局一致的信息。每个节点还具有以下相关信息集。某些信息是关于此特定节点的群集配置详细信息，并且最终在群集中保持一致。其他一些信息，例如上次节点被ping时，对每个节点来说都是本地的。
    + 每个节点都维护有关群集中知道的其他节点的以下信息：节点ID，节点的IP和端口，一组标志什么是节点的主节点，如果它被标记为从节点，上次节点被ping并且最后一次接收到pong节点的当前配置时期（在本说明书中稍后解释），链路状态以及最终服务的散列时隙集合。
    + CLUSTER NODES文档中描述了所有节点字段的详细说明。
    + CLUSTER NODES命令可以发送到集群中的任何节点，并根据查询节点对集群的本地视图提供集群的状态和每个节点的信息。
    + 以下是发送到三个节点的小型集群中的主节点的CLUSTER NODES命令的示例输出。
    
    ```
    $ redis-cli cluster nodes
d1861060fe6a534d42d8a19aeb36600e18785e04 127.0.0.1:6379 myself - 0 1318428930 1 connected 0-1364
3886e65cc906bfd9b1f7e7bde468726a052d1dae 127.0.0.1:6380 master - 1318428930 1318428931 2 connected 1365-2729
d289c575dcbc4bdd2931585fd4339089e461a27d 127.0.0.1:6381 master - 1318428931 1318428931 3 connected 2730-4095
    ```
    
    在上面的列表中，不同的字段按顺序排列：节点id，地址：端口，标志，最后ping发送，最后接收到的pong，配置时期，链路状态，时隙。
    
 + 集群总线
     + 每个Redis群集节点都有一个额外的TCP端口，用于接收来自其他Redis群集节点的传入连接。此端口与用于接收来自客户端的传入连接的普通TCP端口相距固定偏移量。要获取Redis群集端口，应将10000添加到普通命令端口。例如，如果Redis节点正在侦听端口6379上的客户端连接，则还将打开群集总线端口16379。
     + 节点到节点的通信仅使用集群总线和集群总线协议进行：由不同类型和大小的帧组成的二进制协议。群集总线二进制协议未公开记录，因为它不是用于外部软件设备使用此协议与Redis群集节点通信。但是，您可以通过读取Redis群集源代码中的cluster.h和cluster.c文件来获取有关群集总线协议的更多详细信息。
+ 群集拓扑
    + Redis Cluster是一个完整的网格，其中每个节点使用TCP连接与每个其他节点连接。
    + Redis Cluster是一个完整的网格，其中每个节点使用TCP连接与每个其他节点连接。 在N个节点的集群中，每个节点具有N-1个传出TCP连接和N-1个传入连接。
    + 这些TCP连接始终保持活动状态，不按需创建。当节点期望响应集群总线中的ping时发出pong回复，在等待足够长的时间以将节点标记为无法访问之前，它将尝试通过从头开始重新连接来刷新与节点的连接。
    + 而Redis Cluster节点形成一个完整的网格，节点使用八卦协议和配置更新机制，以避免在正常情况下在节点之间交换太多消息，因此交换的消息数量不是指数级的。

+ 节点握手
    + 节点始终接受群集总线端口上的连接，甚至在收到ping时也会回复ping，即使ping节点不受信任也是如此。但是，如果发送节点不被视为群集的一部分，则接收节点将丢弃所有其他分组。
    + 节点将仅以两种方式接受另一个节点作为群集的一部分：
        + 如果节点为自己显示MEET消息。满足消息与PING消息完全相同，但强制接收者接受节点作为群集的一部分。仅当系统管理员通过以下命令请求时，节点才会将MEET消息发送到其他节点：CLUSTER MEET ip port
        + 如果已经信任的节点将闲聊该另一个节点，则节点还将另一个节点注册为集群的一部分。
        + 因此，如果A知道B，并且B知道C，则最终B将向A发送关于C的八卦消息。当发生这种情况时，A将注册C作为网络的一部分，并将尝试与C连接。
        + 这意味着只要我们连接任何连接图中的节点，它们最终将自动形成完全连接的图。这意味着群集能够自动发现其他节点，但前提是系统管理员强制建立了信任关系
        + 此机制使群集更加健壮，但可防止不同的Redis群集在更改IP地址或其他网络相关事件后意外混合。

## 重定向和重新分片
    + MOVED重定向
    + Redis客户端可以自由地向集群中的每个节点发送查询，包括从节点。节点将分析查询，如果它是可接受的（也就是说，查询中只提到一个密钥，或者提到的多个密钥都属于同一个哈希槽），它将查找哪个节点负责一个或多个密钥所属的哈希槽。
    + 如果散列槽由节点提供，则简单地处理查询，否则节点将检查其内部散列槽到节点映射，并将回复具有MOVED错误的客户端，如下例所示
    
    ```
    GET x
    -MOVED 3999 127.0.0.1:6381
    ```
    
    + 该错误包括key的哈希槽（3999）和可以为查询提供服务的节点的ip：端口。
    + 客户端需要将查询重新发出到指定节点的IP地址和端口。
    + 即使客户端在重新发出查询之前等待很长时间，同时集群配置也发生了变化，如果散列槽3999现在由另一个节点提供服务，则目标节点将再次回复MOVED错误。如果联系的节点没有更新的信息，则会发生相同的情况
    + 因此，从群集节点的角度来看，我们尝试通过ID来简化我们与客户端的接口，只是在哈希槽和由IP：端口对识别的Redis节点之间公开地图。
    + 客户端不是必需的，但应该尝试记住127.0.0.1:6381提供的哈希槽3999。
    + 这样，一旦需要发出新命令，它就可以计算目标密钥的散列槽并且更有可能选择正确的节点。
    + 另一种方法是在收到MOVED重定向时使用CLUSTER NODES或CLUSTER SLOTS命令刷新整个客户端群集布局。
    + 遇到重定向时，可能会重新配置多个插槽而不是一个，因此尽快更新客户端配置通常是最佳策略。
    + 当群集稳定（配置中没有持续更改）时，最终所有客户端都将获得哈希槽的映射 - >节点，使集群高效，客户端直接寻址正确的节点，无需重定向，代理或其他单点故障实体。
    + 客户端还必须能够处理本文档后面描述的-ASK重定向，否则它不是完整的Redis群集客户端。

+ 群集实时重新配置
    + Redis Cluster支持在群集运行时添加和删除节点的功能。添加或删除节点被抽象为相同的操作：将哈希槽从一个节点移动到另一个节点。这意味着可以使用相同的基本机制来重新平衡群集，添加或删除节点等。
        + 要向集群添加新节点，会向集群添加空节点，并将一些散列插槽集从现有节点移动到新节点。
        + 要从群集中删除节点，分配给该节点的哈希槽将移动到其他现有节点。
        + 为了重新平衡群集，在节点之间移动一组给定的散列槽。
    + 实现的核心是移动哈希槽的能力。从实际的角度来看，哈希槽只是一组keys，因此Redis Cluster在重新分片期间的确实做的是将密钥从一个实例移动到另一个实例。 移动哈希槽意味着将碰巧哈希的所有密钥移动到此哈希槽中。
    + 要了解其工作原理，我们需要显示用于操作Redis群集节点中的插槽转换表的CLUSTER子命令。
    + 可以使用以下子命令（在这种情况下，其他子命令无用）：
        + CLUSTER ADDSLOTS slot1 [slot2] ... [slotN]
        + CLUSTER DELSLOTS slot1 [slot2] ... [slotN]
        + CLUSTER SETSLOT slot NODE node
        + CLUSTER SETSLOT slot MIGRATING node
        + CLUSTER SETSLOT slot IMPORTING node

    + 前两个命令ADDSLOTS和DELSLOTS仅用于将插槽分配（或删除）到Redis节点。分配时隙意味着告诉给定主节点它将负责存储和提供指定散列槽的内容。
    + 在分配散列槽之后，它们将使用八卦协议在群集中传播，如稍后在配置传播部分中所指定的。
    + 当从头开始创建新集群时，通常使用ADDSLOTS命令，以便为每个主节点分配所有可用的16384个散列插槽的子集。
    + DELSLOTS主要用于手动修改集群配置或调试任务：实际上很少使用它
    + 如果使用SETSLOT <slot> NODE表单，则SETSLOT子命令用于将槽分配给特定节点ID。否则，可以将插槽设置​​为两种特殊状态MIGRATING和IMPORTING。使用这两个特殊状态是为了将散列槽从一个节点迁移到另一个节点。
        + 当一个槽设置为MIGRATING时，该节点将接受有关此哈希槽的所有查询，但仅当存在相关key时，否则使用-ASK重定向将查询转发到迁移目标节点。
        + 当插槽设置为IMPORTING时，节点将接受有关此散列插槽的所有查询，但仅当请求前面有ASKING命令时。如果客户端未提供ASKING命令，则会通过-MOVED重定向错误将查询重定向到真正的哈希槽所有者，这通常会发生。
    + 通过哈希槽迁移的例子来说明这一点。假设我们有两个Redis主节点，称为A和B.我们想将散列槽8从A移动到B，所以发出如下命令：
        + 发送B：CLUSTER SETSLOT 8进口A.
        + 发送A：CLUSTER SETSLOT 8 MIGRATING B
    + 每次使用属于散列槽8的密钥查询客户端时，所有其他节点将继续将客户端指向节点“A”，因此会发生以下情况：
        + 有关现有KEY的所有查询都由“A”处理。 
        + 关于A中不存在的KEY的所有查询都由“B”处理，因为“A”将客户端重定向到“B”。
    + 这样就不再在“A”中创建新KEY了。与此同时，在重新分片和Redis群集配置期间使用的称为redis-trib的特殊程序将把哈希槽8中的现有密钥从A迁移到B. 这是使用以下命令执行的：`CLUSTER GETKEYSINSLOT slot count`
    + 上面的命令将返回指定散列槽中的计数键。对于返回的每个键，redis-trib向节点“A”发送MIGRATE命令，这将以原子方式将指定的key从A迁移到B（两个实例都被锁定一段时间（通常是非常小的时间）来迁移key，因此没有竞争条件）。这就是MIGRATE的工作原理：`MIGRATE target_host target_port key target_database id timeout`
    + MIGRATE将连接到目标实例，发送key的序列化版本，一旦收到OK代码，将删除其自己的数据集中的旧key。从外部客户端的角度来看，密钥在任何给定时间存在于A或B中。
    + 在Redis群集中，不需要指定0以外的数据库，但MIGRATE是一个通用命令，可用于不涉及Redis群集的其他任务。即使在移动复杂键（如长列表）时，MIGRATE也会尽可能快地进行优化当移动复杂key（如长列表）时，但在Redis群集中，如果使用数据库的应用程序中存在延迟约束，则重新配置存在大key的群集不被视为明智的过程。
    + 当迁移过程最终完成时，SETSLOT <slot> NODE <node-id>命令被发送到迁移中涉及的两个节点，以便再次将槽设置为其正常状态。通常会将相同的命令发送到所有其他节点，以避免等待新配置在群集中的自然传播。
+ ASK重定向
    + 在上一节中，我们简要介绍了ASK重定向。为什么我们不能简单地使用MOVED重定向？因为虽然MOVED意味着我们认为哈希槽是由不同节点永久服务的，并且应该针对指定节点尝试下一个查询，但ASK意味着仅将下一个查询发送到指定节点。
    + 这是必需的，因为关于散列槽8的下一个查询可以是关于仍在A中的key，因此我们总是希望客户端尝试A，然后在需要时尝试B. 由于这仅发生在16384可用的一个散列槽中，因此群集上的性能可以接受。
    + 我们需要强制该客户端行为，因此为了确保客户端在A尝试之后只尝试节点B，如果客户端在发送查询之前发送ASKING命令，则节点B将仅接受设置为IMPORTING的插槽的查询。
    + 基本上，ASKING命令在客户端上设置一次性标志，强制节点提供有关IMPORTING槽的查询。
    + 从客户端的角度来看，ASK重定向的完整语义如下：
        + 如果收到ASK重定向，则仅发送重定向到指定节点的查询，但继续向旧节点发送后续查询。
        + 使用ASKING命令启动重定向查询。
        + 还没有更新本地客户端表以将哈希插槽8映射到B.
   + 一旦散列槽8迁移完成，A将发送MOVED消息，并且客户端可以将散列槽8永久映射到新的IP和端口对。如果有错误的客户端先前执行了映射，这不是问题，因为它在发出查询之前不会发送ASKING命令，因此B将使用MOVED重定向错误将客户端重定向到A.
   + 插槽迁移以类似的术语解释，但在CLUSTER SETSLOT命令文档中使用不同的措辞（为了文档中的冗余）。
+ 客户端首次连接和处理重定向
    + 虽然可以使Redis群集客户端实现不记得插槽配置（插槽号和服务节点地址之间的映射）在内存中，只能通过联系等待重定向的随机节点来工作，这样的客户端效率非常低。
    + Redis群集客户端应该尝试足够智能以记住插槽配置。但是，此配置不需要是最新的。由于联系错误的节点只会导致重定向，因此应该触发客户端视图的更新。
    + 客户端通常需要在两种不同的情况下获取完整的插槽列表和映射的节点地址：
        + 在启动时，为了填充初始插槽配置。
        + 当收到MOVED重定向时。
    + 客户端可以通过仅更新其表中移动的插槽来处理MOVED重定向，但这通常效率不高，因为通常会立即修改多个插槽的配置(例如，如果将从属设备提升为主设备，则将重新映射旧主设备所服务的所有插槽）
    + 通过从头开始向节点提取完整的插槽映射，对MOVED重定向做出反应要简单得多。
    + 为了检索插槽配置，Redis Cluster提供了不需要解析的CLUSTER NODES命令的替代方法，并且仅提供客户端严格需要的信息。
    + 新命令称为CLUSTER SLOTS，它提供一个插槽范围数组，以及服务于指定范围的关联主节点和从属节点。
    + 以下是CLUSTER SLOTS输出的示例：

    ```
    127.0.0.1:7000> cluster slots
1) 1) (integer) 5461
   2) (integer) 10922
   3) 1) "127.0.0.1"
      2) (integer) 7001
   4) 1) "127.0.0.1"
      2) (integer) 7004
2) 1) (integer) 0
   2) (integer) 5460
   3) 1) "127.0.0.1"
      2) (integer) 7000
   4) 1) "127.0.0.1"
      2) (integer) 7003
3) 1) (integer) 10923
   2) (integer) 16383
   3) 1) "127.0.0.1"
      2) (integer) 7002
   4) 1) "127.0.0.1"
      2) (integer) 7005
    ```
    
    + 返回数组的每个元素的前两个子元素是范围的起始端插槽。附加元素表示地址 - 端口对。第一个地址端口对是服务于时隙的主设备，而附加的地址端口对是服务于相同时隙的所有从设备，它们不处于错误状态（即未设置FAIL标志）。
    + 例如，输出的第一个元素表示从5461到10922（包括起始和结束）的插槽由127.0.0.1:7001提供服务，并且可以在127.0.0.1:7004处缩放与从站联系的只读负载。
    + 如果群集配置错误，则无法保证CLUSTER SLOTS返回覆盖整个16384插槽的范围因此，客户端应使用NULL对象初始化填充目标节点的插槽配置映射，并在用户尝试执行有关属于未分配插槽的密钥的命令时报告错误。
    + 因此，客户端应使用NULL对象初始化填充目标节点的插槽配置映射，并在用户尝试执行有关属于未分配插槽的密钥的命令时报告错误。

+ 多键操作
    + 使用哈希标记，客户端可以自由使用多键操作。例如，以下操作有效：
    `MSET {user:1000}.name Angela {user:1000}.surname White`
    + 当key所属的散列槽的重新分片正在进行时，多key作可能变得不可用。
    + 更具体地说，即使在重新分片期间，仍然可以获得所有存在且仍然在同一节点（源节点或目的节点）中的目标密钥的多key操作。
    + 对于不存在或在重新分片期间在源节点和目标节点之间拆分的key的操作将生成-TRYAGAIN错误。客户端可以在一段时间后尝试操作，或报告错误。
    + 一旦指定的散列槽的迁移终止，所有多键操作再次可用于该散列槽。

+ 使用从节点缩放读取
    + 通常，从节点会将客户端重定向到给定命令中涉及的散列槽的权威主节点，但是客户端可以使用从属节点来使用READONLY命令扩展读取。
    + READONLY告诉Redis群集从属节点客户端可以读取可能过时的数据，并且对运行写入查询不感兴趣。
    + 当连接处于只读模式时，仅当操作涉及未由从属主节点提供的key时，群集才会向客户端发送重定向。这可能是因为：
        + 客户端发送了一个关于从未由该从属服务器的主服务器提供服务的散列槽的命
        + 群集被重新配置（例如重新配置），并且从属设备不再能够为给定的哈希槽提供命令。
+ 发生这种情况时，客户端应更新其散列图映射，如前面部分所述。 可以使用READWRITE命令清除连接的只读状态。 

## 容错
+ 心跳和八卦(gossip)消息
    + Redis群集节点不断交换ping和pong数据包。这两种数据包具有相同的结构，并且都携带重要的配置信息。唯一的实际区别是消息类型字段。我们将ping和pong包的总和称为心跳包。
    + 通常节点发送ping数据包，触发接收器回复pong数据包。然而，这不一定是真的。
    
