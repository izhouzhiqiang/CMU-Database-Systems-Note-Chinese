# Introduction to Distributed Databases
分布式DBMS将单个逻辑数据库划分为多个物理资源。应用程序(通常)不知道数据在不同的硬件之间被分割。该系统依赖于单节点dbms的技术和算法来支持分布式环境中的事务处理和查询执行。设计分布式DBMS的一个重要目标是容错(即，避免单个节点故障导致整个系统崩溃)。
并行数据库与分布式数据库的不同之处在于:
并行数据库:

- 节点在物理上彼此靠近。
- 节点通过高速局域网(快速、可靠的通信结构)连接。
- 假设节点间的通信开销很小。因此，在设计内部协议时不需要担心节点崩溃或数据包丢失。

分布式数据库:

- 节点之间可以相距很远。
- 节点可能通过公共网络连接，这可能很慢且不可靠。
- 通信成本和连接问题不容忽视(即节点可能崩溃，数据包可能丢失)。
# System Architectures系统架构
DBMS的系统体系结构指定cpu可以直接访问哪些共享资源。它影响cpu如何相互协调，以及它们在数据库中检索和存储对象的位置。
单节点DBMS使用所谓的共享一切架构。这个节点在本地CPU上使用自己的本地内存地址空间和磁盘执行worker。
## Shared Memory
在分布式系统中，共享一切架构的另一种选择是共享内存。cpu可以通过快速互连访问公共内存地址空间。cpu也共享同一个磁盘。
在实践中，大多数dbms不使用这种体系结构，因为它是在操作系统/内核级别提供的。这也会导致问题，因为每个进程的内存范围是相同的内存地址空间，这可以由多个进程修改。
每个处理器都有一个所有内存数据结构的全局视图。处理器上的每个DBMS实例都必须“知道”其他实例。

## Shared Disk
在共享磁盘体系结构中，所有cpu都可以通过互连直接对单个逻辑磁盘进行读写，但是每个cpu都有自己的私有内存。每个计算节点上的本地存储可以充当缓存。这种方法在基于云的dbms中更为常见。
DBMS的执行层可以独立于存储层进行扩展。添加新的存储节点或执行节点不会影响另一层数据的布局或位置。
节点之间必须发送消息以了解其他节点的当前状态。也就是说，由于内存是本地的，如果数据被修改，如果数据块在其他cpu的主存中，则必须将更改通知给其他cpu。
节点有自己的缓冲池，被认为是无状态的。节点崩溃不会影响数据库的状态，因为数据库是单独存储在共享磁盘上的。存储层在崩溃的情况下持久化状态。

## Shared Nothing
在无共享环境中，每个节点都有自己的CPU、内存和磁盘。节点之间只通过网络进行通信。在云存储平台兴起之前，无共享架构曾被认为是构建分布式dbms的正确方法。
在这种体系结构中增加容量更加困难，因为DBMS必须将数据物理地移动到新节点。确保DBMS中所有节点的一致性也很困难，因为节点必须在事务状态上相互协调。然而，它的优点是无共享DBMS可能比其他类型的分布式DBMS体系结构实现更好的性能和更高效。
![image.png](https://cdn.nlark.com/yuque/0/2024/png/26927011/1715357852870-0b619eea-add8-42f1-8891-030f61766f6e.png#averageHue=%23f9f8f7&clientId=ua13af67f-5c9e-4&from=paste&height=344&id=uedf59db7&originHeight=430&originWidth=637&originalType=binary&ratio=1.25&rotation=0&showTitle=false&size=62193&status=done&style=none&taskId=u332ac9c4-f368-4f77-8552-f6726810708&title=&width=509.6)

# Design Issues设计问题
分布式dbms的目标是维护数据的透明性，这意味着不应该要求用户知道数据的物理位置，或者表是如何分区或复制的。如何存储数据的细节对应用程序是隐藏的。换句话说，在单节点DBMS上工作的SQL查询应该在分布式DBMS上工作。
分布式数据库系统必须解决的关键设计问题如下:

- 应用程序如何查找数据?
- 如何在分布式数据上执行查询?应该将查询推到数据所在的位置吗?还是应该将数据汇集到一个公共位置以执行查询?
- DBMS如何确保正确性?要做出的另一个设计决策涉及决定节点如何在其集群中交互。两个选项是同构节点和异构节点，它们都在现代系统中使用。

同构节点 Homogeneous Nodes :集群中的每个节点都可以执行相同的任务集(尽管可能是在不同的数据分区上)，这使其非常适合无共享架构。这使得配置和故障转移“更容易”。失败的任务被分配给可用的节点。
异构节点 Homogeneous Nodes :节点被分配特定的任务，因此节点之间必须进行通信以执行给定的任务。这允许单个物理节点承载多个“虚拟”节点类型，用于可以从一个节点独立扩展到另一个节点的专用任务。一个例子是MongoDB，它有路由器节点将查询路由到分片，并配置服务器节点存储从键到分片的映射。

# 分区方案
分布式系统必须跨多个资源(包括磁盘、节点、处理器)对数据库进行分区。这个过程在NoSQL系统中有时被称为分片。当DBMS接收到查询时，它首先分析查询计划需要访问的数据。DBMS可能会将查询计划的片段发送到不同的节点，然后将结果组合以生成单个答案。
分区方案的目标是最大化单节点事务，或者只访问包含在一个分区上的数据的事务。这使得DBMS不需要协调在其他节点上运行的并发事务的行为。另一方面，分布式事务访问一个或多个分区上的数据。这需要昂贵而困难的协调，将在下一节讨论。
对于逻辑分区节点，特定节点负责访问来自共享磁盘的特定元组。对于物理分区节点，每个shared nothing节点读取和更新它在自己的本地磁盘上包含的元组。
## Implementation
对表进行分区的最简单方法是初始数据分区。假设给定节点有足够的存储空间，每个节点存储一个表。这很容易实现，因为查询只是路由到特定的分区。这可能很糟糕，因为它是不可伸缩的。如果经常查询一个表，而不使用所有可用节点，则可能耗尽该分区的资源。图2给出了一个示例。
![image.png](https://cdn.nlark.com/yuque/0/2024/png/26927011/1715357970885-15a9ac79-5d18-4869-b26f-881ce8fde26c.png#averageHue=%23f2f1f0&clientId=ua13af67f-5c9e-4&from=paste&height=205&id=u64b43fc2&originHeight=256&originWidth=634&originalType=binary&ratio=1.25&rotation=0&showTitle=false&size=48184&status=done&style=none&taskId=u8728eb20-b617-476e-839a-93b9c8b7500&title=&width=507.2)
另一种分区方式是垂直分区，它将表的属性划分为单独的分区。
每个分区还必须存储用于重建原始记录的元组信息。
更常见的是水平分区，它将表的元组分成不相交的子集。选择在大小、负载或使用方面平均划分数据库的列，称为分区键。

DBMS可以根据散列、数据范围或谓词对数据库进行物理分区(不共享)或逻辑分区(共享磁盘)。图3给出了一个示例。散列分区的问题是，当添加或删除节点时，必须对大量数据进行洗牌。对此的解决方案是一致哈希。
![image.png](https://cdn.nlark.com/yuque/0/2024/png/26927011/1715357996902-6cd38647-f3d3-43fe-823f-c0c45697bab8.png#averageHue=%23f5f4f2&clientId=ua13af67f-5c9e-4&from=paste&height=250&id=u6afa1c5b&originHeight=312&originWidth=649&originalType=binary&ratio=1.25&rotation=0&showTitle=false&size=89353&status=done&style=none&taskId=u240d4b7c-21d4-4778-87e2-ccf5ce60018&title=&width=519.2)
一致性哈希将每个节点分配到某个逻辑环上的一个位置。然后，每个分区键的哈希映射到环上的一个位置。顺时针方向上离键最近的节点负责该键。图4给出了一个示例。当添加或删除节点时，键仅在与新节点/删除节点相邻的节点之间移动，因此仅移动1/n部分键。
复制因子k表示每个键在顺时针方向的k个最近的节点上被复制。
逻辑分区:节点负责一组键，但它实际上并不存储这些键。这通常用于共享磁盘体系结构。
物理分区:节点负责一组键，并物理地存储这些键。这通常用于无共享架构。
![image.png](https://cdn.nlark.com/yuque/0/2024/png/26927011/1715358048815-6a65f373-835b-4e4c-ba89-529a460bdac9.png#averageHue=%23fafaf9&clientId=ua13af67f-5c9e-4&from=paste&height=277&id=u1c56d5b6&originHeight=346&originWidth=625&originalType=binary&ratio=1.25&rotation=0&showTitle=false&size=49192&status=done&style=none&taskId=udab2d52a-7837-4230-8de9-65486c48145&title=&width=500)

 
# 分布式并发控制
分布式事务访问一个或多个分区上的数据，这需要昂贵的协调。
## Centralized coordinator
集中式协调器充当全局“交通警察”，协调所有行为。请参见图5中的图表。
## Middleware
集中式协调器可以用作中间件，它接受查询请求并将查询路由到正确的分区。
## Decentralized coordinator
在去中心化的方法中，节点自我组织。客户端直接向其中一个分区发送查询。这个主分区将把结果发送回客户端。主分区负责与其他分区通信并相应地提交。
在多个客户机试图获取相同分区上的锁的情况下，集中式方法会产生瓶颈。但是，它更适合分布式2PL，因为它有锁的中心视图，可以更快地处理死锁。这对于去中心化的方法来说是非常重要的。

 
