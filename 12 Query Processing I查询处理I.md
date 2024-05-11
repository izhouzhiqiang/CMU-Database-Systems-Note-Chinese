# Query Plan
DBMS 将 SQL 语句转换为查询计划。查询计划中的运算节点以树的形式排列。 数据从这棵树的叶子流向根。树中根节点的输出是查询的结果。运算节点通常是二进制的（1-2 个子节点）。同一个查询计划可以用多种方式执行。
# Processing Models
DBMS 处理模型定义系统如何执行查询计划。它指定了诸如评估查询计划的方向以及沿途在运算节点之间传递什么类型的数据等内容。有不同的处理模型模型，针对不同的工作负载有不同的权衡。
这些模型还可以实现为从上到下或从下到上调用运算节点。尽管自上而下的方法更为常见，但自下而上的方法可以允许对管道中的缓存/寄存器进行更严格的控制。
我们考虑的三种执行模型是：

- 迭代器模型 Iterator Model
- 物化模型 Materialization Model
- 矢量化/批处理模型 Vectorized / Batch Model
## 迭代器模型 Iterator Model
迭代器模型，也称为火山或管道模型，是最常见的处理模型，几乎每个（基于行的）DBMS 都使用它。
迭代器模型的工作原理是为数据库中的每个运算节点实现 Next 函数。查询计划中的每个节点都会对其子节点调用 Next，直到到达叶节点，叶节点开始将元组发送到其父节点进行处理。然后，在检索下一个元组之前，尽可能沿计划处理每个元组。这在基于磁盘的系统中非常有用，因为它允许我们在访问下一个元组或页面之前充分使用内存中的每个元组。迭代器模型的示例图如图 1 所示。
![image.png](https://cdn.nlark.com/yuque/0/2024/png/26927011/1715308570433-69569ca9-de3d-4754-bbe6-462a880ee448.png#averageHue=%23f3f1f0&clientId=ubd0674d9-9f7c-4&from=paste&height=437&id=u0ba70628&originHeight=524&originWidth=894&originalType=binary&ratio=1.1979166269302368&rotation=0&showTitle=false&size=168458&status=done&style=none&taskId=u2ecc3a44-0f15-4506-8332-5397943ac3e&title=&width=746.2956769294964)
迭代器模型中的查询计划运算节点具有高度可组合性且易于推理，因为只要每个运算节点实现了 Next 函数，就可以独立于查询计划树中的父运算节点或子运算节点来实现，如下所示：

- 每次调用Next 时，该运算节点将返回单个元组，或者如果没有更多元组要发出，则返回空标记。
- 运算节点节点 实现一个循环，调用其子项的Next 来检索其元组，然后处理它们。这样，在父级上调用 Next 会在其子级上调用 Next。作为响应，子节点将返回父节点必须处理的下一个元组
 
## 物化模型 Materialization Model
物化模型是迭代器模型的特化，其中每个运算节点一次处理其所有输入，然后一次发出其输出。每个运算节点每次到达时都会返回其所有元组，而不是使用返回单个元组的 next 函数。为了避免扫描太多元组，DBMS 可以将有关需要多少元组的信息向下传播给后续运算节点（例如 LIMIT）。运算节点将其输出“具体化”为单个结果。输出可以是整个元组 (NSM) 或列的子集 (DSM)。物化模型图如图 2 所示。
![image.png](https://cdn.nlark.com/yuque/0/2024/png/26927011/1715309787547-fe635303-00cd-4cf2-8a61-98a36fe4f4c6.png#averageHue=%23f3f3f2&clientId=ubd0674d9-9f7c-4&from=paste&height=384&id=VZ3GT&originHeight=460&originWidth=858&originalType=binary&ratio=1.1979166269302368&rotation=0&showTitle=false&size=117447&status=done&style=none&taskId=u575d7ff1-73d1-4318-86ab-20492403b8e&title=&width=716.2435020195837)
每个查询计划运算节点都实现一个输出函数：

- 运算节点立即处理其子项的所有元组。
- 该函数的返回结果是运算节点将发出的所有元组。当运算节点执行完毕后，DBMS 永远不需要返回它来检索更多数据。

此方法更适合 OLTP 工作负载，因为查询通常一次仅访问少量元组。因此，检索元组的函数调用较少。物化模型不适合具有大量中间结果的 OLAP 查询，因为 DBMS 可能必须在运算节点之间将这些结果溢出到磁盘。
## 矢量化/批处理模型 Vectorized / Batch Model
与迭代器模型一样，矢量化模型中的每个运算节点都实现一个 Next 函数。然而，每个运算节点都会发出一批（即向量）数据而不是单个元组。该运算节点的内部循环实现针对处理批量数据而不是一次处理单个项目进行了优化。批次的大小可能因硬件或查询属性而异。有关矢量化模型的示例，请参见图 3。
![image.png](https://cdn.nlark.com/yuque/0/2024/png/26927011/1715310040999-5d9b8cad-7208-4004-a7c7-7cb75d4c16a6.png#averageHue=%23f2f1f0&clientId=ubd0674d9-9f7c-4&from=paste&height=368&id=u0dd8c7f8&originHeight=441&originWidth=868&originalType=binary&ratio=1.1979166269302368&rotation=0&showTitle=false&size=121948&status=done&style=none&taskId=u6d2967b7-c864-4569-87fb-b65e4000b5b&title=&width=724.5913283834483)
矢量化模型方法非常适合必须扫描大量元组的 OLAP 查询，因为对 Next 函数的调用较少。
矢量化模型允许操作员更轻松地使用矢量化 (SIMD) 指令来处理批量元组。

## Processing Direction 处理逻辑方向

- 方法#1：自上而下：从根开始，将数据从子项“拉”到父项 – 元组始终通过函数调用传递 
- 方法#2：自下而上：从叶节点开始，将数据从子节点“推送”到父节点 – 允许对操作管道中的缓存/寄存器进行更严格的控制

 
# Access Methods
访问方法是 DBMS 访问存储在表中的数据的方式。一般来说，有两种访问模型的方法；数据可以从通过顺序扫描从表中读取，也可以索引中读取。
## Sequential Scan 顺序扫描
顺序扫描运算节点迭代表中的每个页面并从缓冲池中检索它。当扫描迭代每个页面上的所有元组时，它会评估谓词以决定是否将元组发送给下一个运算节点。
DBMS 维护一个内部游标，用于跟踪它检查的最后一个页/槽。
顺序表扫描几乎总是 DBMS 执行查询的效率最低的方法。
有许多优化可帮助加快顺序扫描速度：

- 预取：提前获取接下来的几页，以便DBMS 在访问每个页面时不必阻塞存储I/O。
- 缓冲池旁路：扫描操作节点将从磁盘获取的页面存储在本地内存中，而不是缓冲池中，以避免顺序泛洪。
- 并行化：使用多个线程/进程并行执行扫描。
- 延迟实现：DSM DBMS 可以延迟将元组拼接在一起，直到查询计划的上部部分。这允许每个操作员将所需的最少量信息传递给下一个操作节点（例如记录 ID、列中记录的偏移量）。这仅在列存储系统中有用。
- 堆集群：元组按照集群索引指定的顺序存储在堆页中。
- 近似查询（有损数据跳过）：对整个表的采样子集执行查询以生成近似结果。这通常是为了在允许低错误产生近乎准确的答案的情况下计算聚合而完成的。
- 区域映射（无损数据跳过）：为页面中的每个元组属性预先计算聚合。然后，DBMS 可以通过首先检查其区域映射来决定是否需要访问页面。每个页面的区域映射存储在单独的页面中，并且每个区域映射页面中通常有多个条目。因此，可以减少顺序扫描中检查的页面总数。区域地图在云数据库系统中特别有价值，因为通过网络传输数据会产生更大的成本。有关区域图的示例，请参见图 4。

![image.png](https://cdn.nlark.com/yuque/0/2024/png/26927011/1715310429889-b529f9fe-6561-4f47-8d3d-e790b425dc7f.png#averageHue=%23f0eeed&clientId=ubd0674d9-9f7c-4&from=paste&height=407&id=ue17209c1&originHeight=488&originWidth=1111&originalType=binary&ratio=1.1979166269302368&rotation=0&showTitle=false&size=147810&status=done&style=none&taskId=u7d985fc7-5b99-493d-9a07-c924b6919fe&title=&width=927.4435090253584)
顺序模型的局限性包括函数开销和缺乏并行化（例如，无法利用向量运算）

## Index Scan
在索引扫描中，DBMS 选择一个索引来查找查询所需的元组。
DBMS 的索引选择过程涉及许多因素，包括： 

- 索引包含哪些属性 
- 查询引用哪些属性 
- 属性的值域 
- 谓词组成 
- 索引是否具有唯一或非唯一键

一个简单的示例索引扫描如图 5 所示。
![image.png](https://cdn.nlark.com/yuque/0/2024/png/26927011/1715310646239-28546c00-844c-453e-80e2-38cc2d7c8423.png#averageHue=%23eeedec&clientId=ubd0674d9-9f7c-4&from=paste&height=482&id=gg6Xl&originHeight=577&originWidth=1102&originalType=binary&ratio=1.1979166269302368&rotation=0&showTitle=false&size=178241&status=done&style=none&taskId=u3707d1ae-8406-4b20-a199-fa0b3c481d4&title=&width=919.9304652978802)
更高级的 DBMS 支持多索引扫描。当对查询使用多个索引时，DBMS 使用每个匹配索引计算记录 ID 集，根据查询的谓词组合这些集，并检索记录并应用可能保留的任何谓词。 DBMS 可以使用位图、哈希表或布隆过滤器通过集合交集来计算记录 ID。请参阅图 6 了解使用多索引扫描的示例。

# Modification Queries
修改数据库的操作节点（INSERT、UPDATE、DELETE）负责检查约束和更新索引。对于 UPDATE/DELETE，子操作节点传递目标元组的记录 ID，并且必须跟踪以前看到的元组。
关于如何处理 INSERT 运算节点，有两种实现选择：

- 选择#1：在运算节点内部具体化元组。
- 选择#2：运算节点插入从子运算节点传入的任何元组。

![image.png](https://cdn.nlark.com/yuque/0/2024/png/26927011/1715320055165-9a1008bc-af4f-4ca7-a302-25611593dcc0.png#averageHue=%23f1f0ef&clientId=ubd0674d9-9f7c-4&from=paste&height=602&id=u25a75ab3&originHeight=721&originWidth=1232&originalType=binary&ratio=1.1979166269302368&rotation=0&showTitle=false&size=234979&status=done&style=none&taskId=u9a660127-1307-4de6-b6b8-b7b0a2d8196&title=&width=1028.4522080281201)

## Halloween Problem
万圣节问题是一种异常情况，其中更新操作更改了元组的物理位置，导致扫描操作节点多次访问该元组。这可能发生在聚簇表或索引扫描上。
这一现象最初是由 IBM 研究人员在 1976 年万圣节构建 System R 时发现的。该问题的解决方案是跟踪每个查询的修改记录 ID。

# Expression Evaluation 表达式估计
DBMS 将 WHERE 子句表示为表达式树（参见图 7 中的示例）。树中的节点代表不同的表达式类型。
![image.png](https://cdn.nlark.com/yuque/0/2024/png/26927011/1715321113459-ac57bece-8588-4968-998b-beedb05aafc1.png#averageHue=%23f9f9f8&clientId=ubd0674d9-9f7c-4&from=paste&height=333&id=ufa8fccf5&originHeight=399&originWidth=996&originalType=binary&ratio=1.1979166269302368&rotation=0&showTitle=false&size=70012&status=done&style=none&taskId=uf8523c5d-1323-4125-b870-881e11be954&title=&width=831.4435058409153)
可存储在树节点中的表达式类型的一些示例：

- 比较（=、<、>、!=） 
- 合取（AND）、析取（OR） 
- 算术运算节点（+、-、*、/、%） 
- 常量和参数值 
- 元组属性引用

为了在运行时计算表达式树，DBMS 维护一个上下文句柄，其中包含执行的元数据，例如当前元组、参数和表模式。然后，DBMS 遍历树来评估其运算节点并生成结果。
以这种方式评估谓词的速度很慢，因为 DBMS 必须遍历整个树并确定每个运算节点要采取的正确操作。更好的方法是直接计算表达式（想想 JIT 编译）。基于内部成本模型，DBMS 将确定是否采用代码生成来加速查询。

## Scheduler
上述查询处理模型给出了数据流的清晰描述，而控制流则相当隐式。调度程序将数据流和控制流完全分离。它在批处理模式下运行良好，尤其是矢量化模型。调度器创建调度组件（工单），不是从上到下调用算子树，而是遍历树并将调度工作放入调度队列。然后工作线程从队列中获取请求并执行它们。

 ![image.png](https://cdn.nlark.com/yuque/0/2024/png/26927011/1715321443889-48fef1fa-ce40-414a-9d33-31363ac0ea69.png#averageHue=%23ebeae8&clientId=ubd0674d9-9f7c-4&from=paste&height=366&id=u46488211&originHeight=438&originWidth=885&originalType=binary&ratio=1.1979166269302368&rotation=0&showTitle=false&size=217899&status=done&style=none&taskId=u600925d3-5384-4336-a57f-793543db0cc&title=&width=738.7826332020181)
一般来说，它具有更清晰的抽象、动态优化、内置查询暂停、更好的 p9x 性能以及更好的可管理性和调试能力。

 
 
 
