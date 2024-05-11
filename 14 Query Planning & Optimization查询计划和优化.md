# Overview
由于 SQL 是声明性的，查询仅告诉 DBMS 要计算什么，而不是如何计算。因此，DBMS 需要将 SQL 语句转换为可执行查询计划。但是，执行查询计划中的每个运算符的方法不同（例如，连接算法），并且这些计划之间的性能会有所不同。DBMS 优化器的工作是为任何给定的查询选择最佳计划。 查询优化器的第一个实现是 IBM System R，设计于 20 世纪 70 年代。在此之前，人们并不相信 DBMS 能够比人类更好地构建查询计划。System R 优化器的许多概念和设计决策至今仍在使用。 查询优化有两种高级策略。 第一种方法是使用静态规则或启发式方法。启发式方法将查询的各部分与已知模式进行匹配以制定计划。这些规则转换查询以消除低效率。虽然这些规则可能需要查阅目录以了解数据的结构，但它们永远不需要检查数据本身。 另一种方法是使用基于成本的搜索来读取数据并估算执行等效计划的成本。成本模型选择成本最低的计划。 查询优化是构建 DBMS 最困难的部分。一些系统尝试应用机器学习来提高优化器的准确性和效率，但目前没有一个主要的 DBMS 部署基于此技术的优化器。
## Logical vs. Physical Plans
优化器生成逻辑代数表达式到最佳等效物理代数表达式的映射。逻辑计划大致相当于查询中的关系代数表达式。
物理运算符使用查询计划中不同运算符的访问路径定义特定的执行策略。物理计划可能取决于所处理的数据的物理格式（即排序、压缩）。
并不总是存在从逻辑计划到物理计划的一对一映射。

# Logical Query Optimization 逻辑查询优化
一些选择优化包括： 

- 尽早执行过滤器（谓词下推）。
- 对谓词重新排序，以便 DBMS 首先应用最具选择性的谓词。
- 分解复杂的谓词并将其下推（拆分连接谓词）。

![image.png](https://cdn.nlark.com/yuque/0/2024/png/26927011/1715326421441-a4d74ba0-f7c4-44a9-a80d-da627f8d8b20.png#averageHue=%23f0eeed&clientId=ua08488c8-9dcd-4&from=paste&height=481&id=ua8fbc733&originHeight=576&originWidth=862&originalType=binary&ratio=1.1979166269302368&rotation=0&showTitle=false&size=170449&status=done&style=none&taskId=u9494fe27-5fd0-45c3-8168-b36acaaa300&title=&width=719.5826325651295)
 图 2 显示了谓词下推的示例。
一些投影优化包括： 

- 尽早执行投影以创建更小的元组并减少中间结果（投影下推）。
- 投影出除请求或需要的属性之外的所有属性。

![image.png](https://cdn.nlark.com/yuque/0/2024/png/26927011/1715326491484-d784d321-fd10-4dfc-8f53-fd9747dbeece.png#averageHue=%23f0efef&clientId=ua08488c8-9dcd-4&from=paste&height=369&id=u91ffc9da&originHeight=442&originWidth=877&originalType=binary&ratio=1.1979166269302368&rotation=0&showTitle=false&size=89220&status=done&style=none&taskId=ub506615f-ecaf-4204-94ce-5aeecebf428&title=&width=732.1043721109265)
投影下推的示例如图 3 所示。
一些查询重写优化包括：

- 删除不可能或不必要的谓词。在此优化中，DBMS 省略了对表中每个元组的结果不会改变的谓词的评估。绕过这些谓词可以降低计算成本。
- 合并谓词，如图 4 所示。

![image.png](https://cdn.nlark.com/yuque/0/2024/png/26927011/1715326985397-d11de0ee-653e-4d4a-8e80-be316ff5131e.png#averageHue=%23edeceb&clientId=ua08488c8-9dcd-4&from=paste&height=248&id=u3cd2efa6&originHeight=333&originWidth=865&originalType=binary&ratio=1.1979166269302368&rotation=0&showTitle=false&size=81342&status=done&style=none&taskId=u82141ce8-34c9-4192-87c6-6ec33be96cd&title=&width=643)
• 通过去关联和/或展平嵌套子查询来重写查询。一个例子是如图5所示。
![image.png](https://cdn.nlark.com/yuque/0/2024/png/26927011/1715327032159-d716eea1-4f07-4a97-bb1a-9a9b288cfa14.png#averageHue=%23eeeded&clientId=ua08488c8-9dcd-4&from=paste&height=409&id=ueea339dc&originHeight=490&originWidth=865&originalType=binary&ratio=1.1979166269302368&rotation=0&showTitle=false&size=104878&status=done&style=none&taskId=uc29d3311-e3e4-4fb1-a317-1def606ee06&title=&width=722.0869804742889)
• 分解嵌套查询并将结果存储到临时表中。图 6 显示了一个示例。
![image.png](https://cdn.nlark.com/yuque/0/2024/png/26927011/1715327056980-955acb8e-cd76-4c45-bb16-38522e2b043f.png#averageHue=%23f2f0ef&clientId=ua08488c8-9dcd-4&from=paste&height=427&id=ua739503c&originHeight=511&originWidth=858&originalType=binary&ratio=1.1979166269302368&rotation=0&showTitle=false&size=144138&status=done&style=none&taskId=uf83bf3b6-ba05-4214-bc34-29e5b96034b&title=&width=716.2435020195837)

- JOIN 操作的顺序是查询性能的关键决定因素。穷举所有可能的连接顺序是低效的，因此连接顺序优化需要成本模型。但是，我们仍然可以使用启发式优化方法消除不必要的连接。图 7 显示了连接消除的示例。

![image.png](https://cdn.nlark.com/yuque/0/2024/png/26927011/1715327218801-989c21fa-3606-4807-8b9f-e5e70493a449.png#averageHue=%23f1f0ef&clientId=ua08488c8-9dcd-4&from=paste&height=239&id=ucae8ca9a&originHeight=286&originWidth=902&originalType=binary&ratio=1.1979166269302368&rotation=0&showTitle=false&size=60545&status=done&style=none&taskId=u9fe8e21f-7241-44ac-96e0-09a901feaca&title=&width=752.973938020588)

# Cost Estimations 代价估算
DBMS 使用成本模型来估计执行计划的成本。这些模型评估查询的等效计划，以帮助 DBMS 选择最佳的计划。
查询的成本取决于物理成本和逻辑成本之间的几个基础指标，包括：

- CPU：成本虽小，但难以估计。
- 磁盘I/O：块传输的数量。
- 内存：使用的DRAM 量。
- 网络：发送的消息数。

详尽枚举查询的所有有效计划对于优化器来说执行速度太慢。仅对于可交换和关联的连接来说，每个 n 路连接有 4^^n 种不同的顺序。
优化器必须限制其搜索空间才能有效工作。
为了估算查询成本，DBMS 在其内部目录中维护有关表、属性和索引的内部统计信息。不同的系统以不同的方式维护这些统计数据。大多数系统试图通过维护内部统计表来避免动态计算。然后可以在后台更新这些内部表。
对于每个关系 R，DBMS 维护以下信息：

- NR：R 中元组的数量
- V (A, R)：属性 A 的不同值的数量

利用上面列出的信息，优化器可以导出选择基数 SC(A, R) 统计量。选择基数是给定 NR/V (A,R) 的属性 A 具有值的记录的平均数。请注意，这假设数据一致。这个假设通常是不正确的，但它简化了优化过程

## Selection Statistics
选择基数可用于确定将为给定输入选择的元组的数量。
唯一键上的相等谓词很容易估计（参见图 8）。图 9 显示了一个更复杂的谓词。
![image.png](https://cdn.nlark.com/yuque/0/2024/png/26927011/1715330060056-151d3e9f-40f4-4fdf-8f92-e38a80af4dac.png#averageHue=%23f3f2f1&clientId=ua08488c8-9dcd-4&from=paste&height=194&id=u340cde16&originHeight=232&originWidth=872&originalType=binary&ratio=1.1979166269302368&rotation=0&showTitle=false&size=59252&status=done&style=none&taskId=u4bdef8fa-2951-4b0b-ad3f-3a1edd9c2fd&title=&width=727.9304589289942)
![image.png](https://cdn.nlark.com/yuque/0/2024/png/26927011/1715330155965-64468821-f66a-4777-910e-dcd27c5dfc2e.png#averageHue=%23f3f2f1&clientId=ua08488c8-9dcd-4&from=paste&height=198&id=u5cf312ed&originHeight=237&originWidth=878&originalType=binary&ratio=1.1979166269302368&rotation=0&showTitle=false&size=62429&status=done&style=none&taskId=u71eec78a-0440-42dd-a757-a43e9eb3eb6&title=&width=732.939154747313)
谓词 P 的选择性 (sel) 是符合条件的元组的分数。用于计算选择性的公式取决于谓词的类型。复杂谓词的选择性很难准确估计，这可能会给某些系统带来问题。图 10 显示了选择性计算的示例。
![image.png](https://cdn.nlark.com/yuque/0/2024/png/26927011/1715330180342-0b30d49d-8ef9-4d12-9e90-d111e7683f7a.png#averageHue=%23f7f6f5&clientId=ua08488c8-9dcd-4&from=paste&height=368&id=u1b829825&originHeight=441&originWidth=867&originalType=binary&ratio=1.1979166269302368&rotation=0&showTitle=false&size=100000&status=done&style=none&taskId=u790736f9-0071-42a7-ba52-243f11f54d7&title=&width=723.7565457470619)
观察到谓词的选择性等于该谓词的概率。在许多选择性计算中应用的概率规则。这在处理复杂谓词时特别有用。例如，如果我们假设连词中涉及的多个谓词是独立的，我们可以将连词的总选择性计算为各个谓词选择性的乘积。
## Selectivity Computation Assumptions
在计算谓词的选择基数时，使用以下三个假设。

- 统一数据：值的分布（除了重要的因素）是相同的。
- 独立谓词：属性上的谓词是独立的。
- 包含原则：连接键的域重叠，使得内部关系中的每个键也存在于外表中。

实际数据往往无法满足这些假设。例如，相关属性打破了谓词独立性的假设。
# Histograms
真实数据通常是有偏差的，并且很难做出假设。然而，存储数据集的每个值都是昂贵的。减少内存使用量的一种方法是将数据存储在直方图中以将值分组在一起。图 11 显示了带有存储桶的图表示例。
另一种方法是使用等深度直方图来改变桶的宽度，以便每个桶的出现总数大致相同。图 12 显示了一个示例。
一些系统可以使用草图来代替直方图来生成有关数据集的近似统计数据。
 ![image.png](https://cdn.nlark.com/yuque/0/2024/png/26927011/1715330530942-76cf48a7-933a-4049-a791-7cc56b6a1a73.png#averageHue=%23f8f7f7&clientId=ua08488c8-9dcd-4&from=paste&height=554&id=ub5a4a436&originHeight=664&originWidth=993&originalType=binary&ratio=1.1979166269302368&rotation=0&showTitle=false&size=169186&status=done&style=none&taskId=u619b67a9-22d0-40df-a29e-96ae64824b8&title=&width=828.939157931756)

# Sampling
DBMS 可以使用采样将谓词应用到具有类似分布的较小表副本（参见图 13）。每当基础表的更改量超过某个阈值（例如元组的 10%）时，DBMS 就会更新样本。
![image.png](https://cdn.nlark.com/yuque/0/2024/png/26927011/1715330624626-5840e7ae-1f10-4b1a-831a-7d762da2fef7.png#averageHue=%23f1f0ef&clientId=ua08488c8-9dcd-4&from=paste&height=353&id=u54cf7696&originHeight=423&originWidth=868&originalType=binary&ratio=1.1979166269302368&rotation=0&showTitle=false&size=98103&status=done&style=none&taskId=uc82f6d7d-dc46-4c11-8258-31406de2dc8&title=&width=724.5913283834483)

# Plan Enumeration
执行基于规则的重写后，DBMS 将枚举不同的查询计划并估计其成本。然后，在用尽所有计划或超时后，它会为查询选择最佳计划。

 
# Single-Relation Query Plans
对于单关系查询计划，最大的障碍是选择最佳的访问方法（即顺序扫描、二分搜索、索引扫描等）。大多数新的数据库系统只是使用启发式方法，而不是复杂的成本模型来选择访问方法。
对于 OLTP 查询，这尤其容易，因为它们是可控制的（Search Argument Able），这意味着存在可以为查询选择的最佳索引。这也可以通过简单的启发式方法来实现。

# Multi-Relation Query Plans
对于多关系查询计划，随着连接数量的增加，备选计划的数量迅速增加。 因此，限制搜索空间以便能够在合理的时间内找到最佳计划非常重要。有两种方法可以解决此搜索问题： 

- 自下而上：从零开始，然后制定计划以获得所需的结果。 示例：IBM System R、DB2、MySQL、Postgres、大多数开源 DBMS。 
- 自上而下：从所需的结果开始，然后沿着树向下工作以找到实现该目标的最佳计划。示例：MSSQL、Greenplum、CockroachDB、Volcano
## 自下而上优化示例 - System R
使用静态规则执行初始优化。然后使用动态规划通过分而治之搜索方法确定表的最佳连接顺序。 

- 将查询分解为块并为每个块生成逻辑运算符 
- 对于每个逻辑运算符，生成一组实现它的物理运算符 
- 然后，迭代构建一个“左深”树，以最小化执行计划的估计工作量
## 自上而下的优化示例 - Volcano
从我们想要的查询的逻辑计划开始。通过将逻辑运算符转换为物理运算符，执行分支定界搜索来遍历计划树。 

- 在搜索过程中跟踪全局最佳计划。 
- 在规划过程中将数据的物理属性视为一流实体。
 
