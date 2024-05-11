# Crash Recovery崩溃恢复
DBMS依赖于它的恢复算法来确保数据库一致性、事务原子性和故障时的持久性。每个恢复算法由两部分组成:

- 正常事务处理期间的操作，以确保DBMS可以从故障中恢复
- 失败后的操作，将数据库恢复到确保事务的原子性、一致性和持久性的状态。

数据库弹性的关键是事务完整性和持久性的管理，特别是在故障场景中。这个基本概念为ARIES恢复算法的引入奠定了基础。
利用语义进行恢复和隔离的算法(ARIES Algorithms for Recovery and Isolation Exploiting Semantics)是IBM研究院在20世纪90年代初为DB2系统开发的一种恢复算法。
在ARIES恢复协议中有三个关键概念:

- 提前写入日志:在将数据库更改写入磁盘之前，任何更改都记录在稳定存储的日志中(STEAL + NO-FORCE)。
- 在重做期间重复历史:在重启时，回溯操作并恢复数据库到崩溃前的确切状态。
- 撤销期间的日志更改:记录撤销操作到日志中，以确保在重复失败的情况下不会重复操作。
# WAL Records 预写式日志记录
预写日志记录扩展了DBMS的日志记录格式，以包含一个全局唯一的日志序列号(LSN)。所有日志记录都有一个LSN。图1显示了如何编写带有LSN的日志记录的高级图表。
每个数据页都包含一个pageLSN，这是该页最近更新的LSN。每次事务修改页面中的记录时，都会更新pageLSN。
DBMS还跟踪到目前为止刷新的最大LSN (flushedLSN)。每次DBMS将WAL缓冲区写入磁盘时，都会更新内存中的flushedLSN。
系统中的各种组件跟踪与它们相关的lsn。这些lsn的表如下所示。
![image.png](https://cdn.nlark.com/yuque/0/2024/png/26927011/1715357135506-a5dbdb72-a901-473c-894b-c9c2076bb634.png#averageHue=%23eae7e4&clientId=ua39ceebf-6ced-4&from=paste&height=118&id=uc6546d36&originHeight=148&originWidth=721&originalType=binary&ratio=1.25&rotation=0&showTitle=false&size=46421&status=done&style=none&taskId=ub1298369-2a9e-4983-886e-b13604356bb&title=&width=576.8)
![image.png](https://cdn.nlark.com/yuque/0/2024/png/26927011/1715357155462-6a9f44b1-61de-4da6-bfaa-89b0dcc48ecc.png#averageHue=%23f4f2f0&clientId=ua39ceebf-6ced-4&from=paste&height=402&id=ubfcc7584&originHeight=502&originWidth=652&originalType=binary&ratio=1.25&rotation=0&showTitle=false&size=132165&status=done&style=none&taskId=ub5a58d4f-5cb3-4bd9-9850-93913f54224&title=&width=521.6)
# Normal Execution正常执行
每个事务调用一系列的读写操作，然后是提交或中止。恢复算法必须具备的就是这一系列事件。
## 事务提交
当事务提交时，DBMS首先将Commit记录写入内存中的日志缓冲区。然后，DBMS将所有日志记录(包括事务的COMMIT记录)刷新到磁盘。请注意，这些日志刷新是顺序的、同步的磁盘写入。每个日志页可以有多个日志记录。事务提交的关系图如图2所示。
![image.png](https://cdn.nlark.com/yuque/0/2024/png/26927011/1715357270757-47e62b68-6981-4e16-8e50-5f0b4a939fc6.png#averageHue=%23f4f3f2&clientId=ua39ceebf-6ced-4&from=paste&height=390&id=ubcc41c27&originHeight=487&originWidth=650&originalType=binary&ratio=1.25&rotation=0&showTitle=false&size=114193&status=done&style=none&taskId=uf3057130-6190-455d-bc98-3232a54f8cf&title=&width=520)
一旦COMMIT记录被安全地存储在磁盘上，DBMS就会向应用程序返回事务已提交的确认信息。稍后，DBMS将写一条特殊的TXN-END记录到日志中。这表明该事务在系统中已经完全完成，并且不再有关于它的日志记录。这些TXN-END记录用于内部簿记，不需要立即刷新。
## 事务中止
中止事务是仅应用于一个事务的ARIES撤销操作的特殊情况。
日志记录中增加了一个名为prevLSN的附加字段。对应上一个LSN
对于交易。DBMS使用这些prevLSN值为每个事务维护一个链表
这使得在日志中查找记录变得更加容易。
还引入了一种称为补偿日志记录(CLR)的新记录类型。CLR描述
为撤消先前更新记录的操作而采取的操作。它包含更新日志记录的所有字段以及undoNext指针(即下一个要撤消的LSN)。DBMS将clr添加到日志中，就像任何其他记录一样，但它们永远不需要撤消。此外，DBMS在通知应用程序事务已中止之前，不会等待clr被刷新到磁盘。这种方法确保了高效的事务管理，特别是在涉及事务回滚的场景中。
要中止事务，DBMS首先向内存中的日志缓冲区追加一条abort记录。然后，它以相反的顺序撤消事务的更新，以从数据库中删除其影响。对于每个未完成的更新，DBMS在日志中创建CLR条目并恢复旧值。在所有被终止的事务的更新都被逆转之后，DBMS就会写一条TXN-END日志记录。图3显示了这一过程的示意图。

# Checkpointing
DBMS会定期设置检查点，将缓冲池中的脏页写到磁盘上。这用于最小化在恢复时必须重放的日志量。
下面讨论的前两个阻塞检查点方法在检查点过程中暂停事务。这个暂停是必要的，以确保DBMS在检查点期间不会错过对页面的更新。然后，提出了一种更好的方法，该方法允许事务在检查点期间继续执行，但要求DBMS记录额外的信息，以确定它可能错过了哪些更新。
![image.png](https://cdn.nlark.com/yuque/0/2024/png/26927011/1715357384721-b99a9c03-6957-40de-a755-d91a0a43304e.png#averageHue=%23e6e0de&clientId=ua39ceebf-6ced-4&from=paste&height=347&id=uf456ebb6&originHeight=434&originWidth=641&originalType=binary&ratio=1.25&rotation=0&showTitle=false&size=157182&status=done&style=none&taskId=uf94b5f4e-7021-4e59-b7ad-f15d6353f06&title=&width=512.8)
## Non-Fuzzy检查点
当DBMS采用检查点以确保将数据库的一致快照写入磁盘时，它会停止事务和查询的执行。这与上一讲中讨论的方法相同:

- 停止任何新事务的开始。
- 等待所有活动事务完成执行。
- 将脏页刷新到磁盘。

虽然这个过程会影响运行时性能，但它极大地简化了恢复。
## Slightly Better Blocking Checkpoints
与以前的检查点方案类似，只是DBMS不必等待活动事务完成执行。DBMS现在记录检查点开始时的内部系统状态。

- 停止任何新事务的开始。
- 在DBMS执行检查点时暂停事务。

检查点进程需要在其开始时记录内部状态。这包括两个关键组件:活动事务表(ATT)，它跟踪正在进行的事务;脏页表(DPT)，它列出所有尚未写入磁盘的修改页。
活动事务表(Active Transaction Table, ATT): ATT表示在DBMS中活动运行的事务的状态。在DBMS完成事务的提交/中止过程后，将删除事务的条目。对于每个事务条目，ATT包含以下信息:•transactionId:唯一的事务标识符

- status:事务的当前“模式”(Running, committed, Undo Candidate)。
- lastLSN:最近由事务写的LSN。注意，ATT包含没有TXN-END日志记录的所有事务。这包括提交或中止的事务。

脏页表(Dirty Page Table, DPT): DPT包含有关缓冲池中被未提交事务修改的页面的信息。每个脏页都有一个包含recsn的条目(即，首先导致该页变脏的日志记录的LSN)。
DPT包含缓冲池中所有脏的页面。更改是由正在运行、提交还是中止的事务引起的并不重要。
总的来说，《禁止核武器条约》和《禁止核武器条约》在检查点和恢复过程中都至关重要。在检查点期间，它们捕获数据库的当前状态，ATT跟踪活动事务，DPT列出未刷新的修改页面。在恢复中，例如使用ARIES协议，这些表有助于将数据库恢复到崩溃前的一致状态。

## Fuzzy Checkpoints
模糊检查点是DBMS允许其他事务继续运行的地方。这就是ARIES在它的协议中使用的。
DBMS使用额外的日志记录来跟踪检查点边界:

- < checkpoint - begin >:表示检查点的开始。此时，DBMS获取当前ATT和DPT的快照，它们在记录中被引用。在检查点启动之后开始的事务不包括在ATT中。
- < checkpoint - end >:当检查点完成时。它包含ATT + DPT，在写入日志记录时捕获。

在检查点完成后，< checkpoint - begin >记录的LSN记录在主记录中。
# ARIES Recovery
ARIES协议由三个阶段组成。在崩溃后启动时，DBMS将执行以下阶段，如图4所示:

1.  Analysis:读取WAL以识别在崩溃时缓冲池中的脏页和活动事务。在分析阶段结束时，ATT告诉DBMS哪些事务在崩溃时处于活动状态。DPT告诉DBMS哪些脏页可能没有保存到磁盘上。 
2.  Redo:从日志中的适当点开始重复所有操作(甚至是将中止的txns)。 
3.  Undo:撤销在崩溃前未提交的事务的操作。 

![image.png](https://cdn.nlark.com/yuque/0/2024/png/26927011/1715357619515-fbf993d0-5cba-458f-81ae-2ce2e25941cf.png#averageHue=%23f8f6f4&clientId=ua39ceebf-6ced-4&from=paste&height=446&id=u039c1fea&originHeight=557&originWidth=638&originalType=binary&ratio=1.25&rotation=0&showTitle=false&size=108718&status=done&style=none&taskId=u0cea1c5f-692d-45a5-8824-7a4610b27bf&title=&width=510.4)
## 分析阶段
从通过数据库的MasterRecord LSN找到的最后一个检查点开始。

1.  从检查点向前扫描日志。 
2.  如果DBMS发现TXN-END记录，则从ATT中删除其事务。 
3.  所有其他记录，将事务以UNDO状态添加到ATT，并在提交时将事务状态更改为commit。 
4.  对于UPDATE日志记录，如果页P不在DPT中，则将P添加到DPT中，并将P的recsn设置为日志记录的LSN。 
## REDO重做阶段
这个阶段的目标是让DBMS重复历史来重建它在崩溃之前的状态。它将重新应用所有更新(甚至终止的事务)并重做clr。
DBMS从DPT中包含最小recsn的日志记录向前扫描。对于具有给定LSN的每个更新日志记录或CLR, DBMS将重新应用更新，除非:

- 受影响的页面不在DPT中，或者
- 受影响的页面在DPT中，但该记录的LSN小于DPT中该页的recsn，或者
- 受影响的页面elsn(在磁盘上)≥LSN

要重做一个操作，DBMS重新应用日志记录中的更改，然后将受影响页面的pageLSN设置为该日志记录的LSN。
在重做阶段结束时，为状态为COMMIT的所有事务写入TXN-END日志记录，并将它们从ATT中删除。
## Undo阶段
在最后一个阶段，DBMS反转在崩溃时处于活动状态的所有事务。这些都是分析阶段之后ATT中具有UNDO状态的事务。
DBMS使用最后一个LSN以反向LSN顺序处理事务，以加快遍历速度。在每个步骤中，选择ATT中所有事务中最大的lastLSN。当它反转事务的更新时，DBMS为每个修改写入一个CLR条目到日志中。
一旦成功终止了最后一个事务，DBMS就会清空日志，然后准备开始处理新的事务。
