# Background
前面对查询执行的讨论假设查询是使用单个工作线程（即线程）执行的。然而，在实践中，查询通常与多个工作线程并行执行。
并行执行为 DBMS 提供了许多关键优势： 

- 提高吞吐量（每秒更多查询）和延迟（每次查询时间更短）方面的性能。
- 从 DBMS 外部客户端的角度来看，提高了响应能力和可用性。
- 可能降低总拥有成本(TCO)。此成本包括硬件采购和软件许可，以及部署 DBMS 的人工开销和运行机器所需的能源。

DBMS 支持两种类型的并行性：查询间并行性和查询内并行性。

# Parallel vs Distributed Databases
在并行和分布式系统中，数据库分布在多个“资源”中以提高并行性。这些资源可以是计算资源（例如，CPU 核心、CPU 插槽、GPU、附加机器）或存储资源（例如，磁盘、内存）。
区分并行系统和分布式系统很重要：

- 并行DBMS 在并行DBMS 中，资源或节点在物理上彼此靠近。这些节点通过高速互连进行通信。假设资源之间的通信不仅快速，而且廉价且可靠。
- 分布式DBMS 在分布式DBMS 中，资源可能彼此相距很远。这可能意味着数据库跨越世界不同地区的机架或数据中心。因此，资源使用较慢的互连（通常通过公共网络）进行通信。节点之间的通信成本较高，且故障不容忽视。

尽管数据库可能在物理上划分为多个资源，但对于应用程序来说，它仍然显示为单个逻辑数据库实例。因此，针对单节点 DBMS 执行的 SQL 查询应该在并行或分布式 DBMS 上生成相同的结果。

# Process Models
DBMS 进程模型定义系统如何支持来自多用户应用程序/环境的并发请求。 DBMS 由一个或多个工作人员组成，负责代表客户端执行任务并返回结果。应用程序可能会同时发送大型请求或多个请求，这些请求必须分配给不同的工作人员。
 
## Process per Worker
最基本的方法是process per worker。在这里，每个worker都是一个单独的操作系统进程，因此依赖于操作系统调度程序。应用程序发送请求并打开与数据库系统的连接。某个调度程序接收请求并选择其worker进程之一来管理连接。然后，应用程序直接与负责执行查询所需请求的worker进程进行通信。该事件序列如图 1 所示。![image.png](https://cdn.nlark.com/yuque/0/2024/png/26927011/1715321934823-7226a9c7-1432-4f98-8998-d68898923fcb.png#averageHue=%23e2e1e1&clientId=u0ec69082-e701-4&from=paste&height=252&id=Z2fbL&originHeight=302&originWidth=723&originalType=binary&ratio=1.1979166269302368&rotation=0&showTitle=false&size=63236&status=done&style=none&taskId=u3aef0dda-1155-49c7-9e0f-8cefb078270&title=&width=603.5478461074115)
依靠操作系统进行调度有效地减少了DBMS对执行的控制。
此外，该模型依赖共享内存来维护全局数据结构或依赖消息传递，这具有较高的开销。
每个工作线程方法的一个优点是，进程崩溃不会破坏整个系统，因为每个工作线程都在自己的操作系统进程的上下文中运行。
这种流程模型提出了多个工作人员在不同流程上制作同一页面的大量副本的问题。最大化内存使用的解决方案是对全局数据结构使用共享内存，以便它们可以由运行在不同进程中的工作人员共享。
使用process-per-worker进程模型的系统示例包括 IBM DB2、Postgres 和 Oracle。当开发这些 DBMS 时，threading 尚未成为标准线程模型。
threading的语义因操作系统而异，而 fork() 的定义更好。
注：这里的threading模型并不是指现代计算机上的线程模型，他们出现的比计算机的线程模型更早。
## Thread per Worker
如今最常见的模型是thread per worker。每个数据库系统只有一个具有多个工作线程的进程，而不是让不同的进程执行不同的任务。在这种环境下，DBMS 可以完全控制任务和线程，它可以管理自己的调度。多线程模型可以使用也可以不使用调度程序线程。每个工作线程模型的图如图 2 所示。
![image.png](https://cdn.nlark.com/yuque/0/2024/png/26927011/1715322901517-9d37447a-bcd9-48fa-bf87-203e5d828c28.png#averageHue=%23e3e2e2&clientId=u0ec69082-e701-4&from=paste&height=247&id=u721047e0&originHeight=296&originWidth=759&originalType=binary&ratio=1.1979166269302368&rotation=0&showTitle=false&size=63111&status=done&style=none&taskId=u767e2872-79b0-4c30-a7cc-9e623f6dc3e&title=&width=633.600021017324)
使用多线程架构具有一定的优势。其一，每次上下文切换的开销更少。此外，不必维护共享模型。但是，线程崩溃可能会终止整个数据库进程。此外，每个工作线程模型并不一定意味着 DBMS 支持查询内并行性。
过去 20 年来创建的几乎所有 DBMS 都使用这种方法，包括 Microsoft SQL Server 和 MySQL。 IBM DB2 和 Oracle 也更新了他们的模型来为这种方法提供支持。
Postgres 和 Postgres 派生的数据库很大程度上仍然使用基于流程的方法。

## Scheduling 调度
总之，对于每个查询计划，DBMS 必须决定在何处、何时以及如何执行。相关问题包括： 
• 应使用多少个任务？ 
• 应使用多少个CPU 核心？ 
• 任务应在哪些CPU 内核上执行？ 
• 任务应该在哪里存储其输出？
在做出有关查询计划的决策时，DBMS 总是比操作系统了解更多，因此应该优先考虑。

## Embedded DBMS 嵌入式数据库管理系统
数据库的一种非常不同的使用模式,DBMS和应用程序的同一地址空间中运行系统，这与数据库独立于应用程序的客户端-服务器模型相反。嵌入式指的是DBMS嵌入到前端应用中。
在这种情况下，应用程序将设置要在数据库系统上运行的线程和任务。应用程序本身将主要负责调度。图 3 显示了嵌入式 DBMS 的调度行为图。
![image.png](https://cdn.nlark.com/yuque/0/2024/png/26927011/1715323603660-caaaf1a4-a777-4970-95d5-325c72804fcf.png#averageHue=%23d0cfcf&clientId=u0ec69082-e701-4&from=paste&height=270&id=ue3fb9af0&originHeight=324&originWidth=739&originalType=binary&ratio=1.1979166269302368&rotation=0&showTitle=false&size=53707&status=done&style=none&taskId=u0762b65e-7929-43ab-b91e-6818fe3d821&title=&width=616.9043682895948)
DuckDB、SQLite 和 RocksDB 是最著名的嵌入式 DBMS。

# Inter-Query Parallelism 查询间并行性
在查询间并行中，DBMS 同时执行不同的查询。由于多个worker同时运行请求，因此整体性能得到了提高。这增加了吞吐量并减少了延迟。
如果查询是只读的，则查询之间几乎不需要协调。但是，如果多个查询同时更新数据库，则会出现更复杂的冲突。这些问题将在第 15 讲中进一步讨论。

# Intra-Query parallelism 查询内并行性


在查询内并行中，DBMS 并行执行单个查询的操作。这减少了长时间运行的查询的延迟。
查询内并行性的组织可以根据生产者/消费者范例来考虑。
每个操作节点都是数据的生产者，也是其下运行的某个操作节点的数据的消费者。
每个关系运算节点都存在并行算法。 DBMS 可以让多个线程访问集中式数据结构，也可以使用分区来划分工作。
在查询内并行中，存在三种类型的并行：运算节点内并行、运算节点间并行和密集并行。这些方法并不相互排斥。 DBMS 的责任是将这些技术结合起来，以优化给定工作负载的性能。

## Intra-Operator Parallelism (Horizontal) 运算节点内并行性（水平）
在运算节点内并行性中，查询计划的运算节点被分解为独立的片段，这些片段对不同（不相交）的数据子集执行相同的功能。
DBMS 将交换运算节点插入到查询计划中以合并子运算节点的结果。交换运算节点阻止 DBMS 执行计划中位于其上方的运算节点，直到它收到来自子级的所有数据。图 4 显示了一个示例。
![image.png](https://cdn.nlark.com/yuque/0/2024/png/26927011/1715323747620-618a583a-f8dc-43ab-9f2c-918e67823222.png#averageHue=%23efedec&clientId=u0ec69082-e701-4&from=paste&height=527&id=dhrHE&originHeight=631&originWidth=1011&originalType=binary&ratio=1.1979166269302368&rotation=0&showTitle=false&size=166491&status=done&style=none&taskId=u26ee920d-d6f4-471b-bd29-2d1b27993bb&title=&width=843.9652453867122)
一般来说，交换操作节点分为三类：

- 收集：将多个工作人员的结果合并到一个输出流中。这是并行 DBMS 中最常用的类型。
- 分发：将单个输入流拆分为多个输出流。
- 重新分区：跨多个输出流重新组织多个输入流。这允许 DBMS 获取以一种方式分区的输入，然后以另一种方式重新分配它们。

## Inter-Operator Parallelism (Vertical) 操作间并行性（垂直）
在运算节点间并行性中，DBMS 重叠运算节点，以便将数据从一个阶段输送到下一阶段，而无需具体化。这有时称为流水线并行性。请参见图 5 中的示例。
![image.png](https://cdn.nlark.com/yuque/0/2024/png/26927011/1715324466130-a1c61661-a66e-4738-a947-3f414b14fe79.png#averageHue=%23edeceb&clientId=u0ec69082-e701-4&from=paste&height=402&id=u3a74c958&originHeight=481&originWidth=843&originalType=binary&ratio=1.1979166269302368&rotation=0&showTitle=false&size=97448&status=done&style=none&taskId=uba580aac-9a2c-4d33-b050-c4f0fb53d25&title=&width=703.7217624737868)
这种方法广泛用于流处理系统，流处理系统是对输入元组流持续执行查询的系统。

## Bushy Parallelism
Bushy 并行性是运算符内并行性和运算符间并行性的混合体，其中工作线程同时执行来自查询计划的不同部分的多个运算符。

DBMS 仍然使用交换运算符来组合这些段的中间结果。图 6 显示了一个示例。
![image.png](https://cdn.nlark.com/yuque/0/2024/png/26927011/1715325861878-035061e0-c952-4f6c-b963-39924198de74.png#averageHue=%23f0efee&clientId=u0ec69082-e701-4&from=paste&height=433&id=u0849d680&originHeight=519&originWidth=846&originalType=binary&ratio=1.1979166269302368&rotation=0&showTitle=false&size=101092&status=done&style=none&taskId=u833a93aa-b62a-49ae-936f-a1ae849e16b&title=&width=706.2261103829462)

# I/O Parallelism
如果磁盘始终是主要瓶颈，那么使用额外的进程/线程并行执行查询不会提高性能。因此，能够将数据库拆分到多个存储设备上非常重要。
为了解决这个问题，DBMS 使用 I/O 并行性将安装分散到多个设备上。 I/O 并行的两种方法是多磁盘并行和数据库分区。
## Multi-Disk Parallelism
在多磁盘并行中，操作系统/硬件配置为跨多个存储设备存储 DBMS 的文件。这可以通过存储设备或 RAID 配置来完成。所有存储设置对 DBMS 都是透明的，因此工作人员无法在不同的设备上操作，因为 DBMS 不知道底层的并行性。
## Database Partitioning
在数据库分区中，数据库被分成不相交的子集，这些子集可以分配给离散的磁盘。
某些 DBMS 允许指定每个单独数据库的磁盘位置。如果 DBMS 将每个数据库存储在单独的目录中，则在文件系统级别很容易做到这一点。所做更改的日志文件通常是共享的。
逻辑分区的思想是将单个逻辑表分割成不相交的物理段，并单独存储/管理。理想情况下，这种分区对于应用程序来说是透明的。也就是说，应用程序应该能够访问逻辑表，而无需关心事物的存储方式。
我们将在本课程晚些时候讨论分布式数据库时介绍这些方法。

 
