# OLTP VS. OLAP
联机事务处理(OLTP)

- 短暂的读/写事务。
- 少量资源占用小。
- 重复性操作。

联机分析处理(OLAP)

- 长时间运行的只读查询。
- 复杂连接。
- 探索性查询。

# Decentralized Coordinator Setup分散协调器设置
分布式数据库系统的基本场景是我们有一个应用服务器和多个数据分区。其中一个分区被选为主节点。事务的开始请求从应用服务器发送到主节点，如果没有集中的协调器，查询将直接发送到各个节点。提交请求从应用服务器发送到主节点，主节点负责在所有参与节点中确定是否允许提交。如果所有参与节点都同意提交是安全的，那么我们就可以提交事务。使用两阶段锁定、MVCC、OCC等策略来确定事务是否可以安全地提交到每个单独的节点上。
下面几节将讨论如何确保所有节点同意提交事务，以及如果一个节点失败/消息显示较晚/系统没有等待每个节点同意会发生什么。我们假设分布式DBMS中的所有节点都表现良好，并且在同一个管理域下。
(如果你不信任分布式DBMS中的其他节点，那么你需要为事务使用拜占庭容错协议(例如区块链)。)
# Replication
DBMS可以跨冗余节点复制数据以提高可用性。在primary - replica中，每个对象的所有更新都转到指定的主节点。主服务器在没有原子提交协议的情况下将更新传播到其副本，并协调到达它的所有更新。如果不需要最新的信息，可以允许只读事务访问副本。如果初选失败，则举行选举以选出新的初选。
在Multi-Primary中，事务可以在任何副本上更新数据对象。副本之间必须使用原子提交协议进行同步。
![image.png](https://cdn.nlark.com/yuque/0/2024/png/26927011/1715358271333-b4a81518-f14d-4f68-8a88-6a8e5ec197f8.png#averageHue=%23ecebeb&clientId=u043af470-757a-4&from=paste&height=170&id=u365f32bf&originHeight=213&originWidth=330&originalType=binary&ratio=1.25&rotation=0&showTitle=false&size=42112&status=done&style=none&taskId=u938f1f7e-86f6-46b6-9154-d570c739498&title=&width=264)
## K-Safety
是确定复制数据库容错性的阈值。值K表示每个数据对象必须始终可用的副本数量。如果副本的数量低于这个阈值，那么DBMS将停止执行并使自己脱机。K值越高，丢失数据的风险就越小。它是确定系统可用性的阈值。
## Propagation Scheme传播方案
当一个事务在复制数据库上提交时，DBMS决定是否必须等待该事务的更改传播到其他节点，然后才能向应用程序客户端发送确认。有两种传播级别:同步(强一致性)和异步(最终一致性)。
在同步方案中，主服务器向副本发送更新，然后等待副本确认已完全应用(即记录)更改。然后，主服务器可以通知客户端更新已经成功。它保证了DBMS不会因为强一致性而丢失任何数据。这在传统的DBMS中更为常见。
在异步模式中，主服务器立即向客户端返回确认，而无需等待副本应用更改。在这种方法中可能会发生过期读取，因为在读取时可能不会将更新完全应用于副本。如果可以容忍某些数据丢失，则此选项可能是可行的优化。这在NoSQL系统中很常用。
## Propagation Timing传播时间
传播定时对于连续传播定时，DBMS在生成日志消息时立即发送日志消息。注意，还需要发送提交和中止消息。大多数系统使用这种方法。
对于on commit propagation timing, DBMS只在事务提交后才将事务的日志消息发送到副本。这不会浪费时间来发送终止事务的日志记录。它确实假设事务的日志记录完全适合内存。
## Active vs Passive主动与被动
主动与被动有多种方法可以将更改应用于副本。对于双活，事务在每个副本上独立执行。最后，DBMS需要检查事务在每个副本上是否以相同的结果结束，以查看副本是否正确提交。这很困难，因为现在事务的顺序必须在所有节点之间同步，这使得它不太常见。
对于主动-被动，每个事务在单个位置执行，并将整体更改传播到副本。DBMS既可以发送更改的物理字节(这是更常见的)，也可以发送逻辑SQL查询。大多数系统都是主-被动的。

# Atomic Commit Protocols原子性提交协议
当一个多节点事务完成时，DBMS需要询问所有涉及的节点是否可以安全提交。根据协议的不同，可能需要大多数节点或所有节点进行提交。例如:

- 两阶段提交(20世纪70年代)
- 三阶段提交(1983年)
- Paxos(1989年)
- Raft(2013年)
- ZAB(2008年?Zookeeper原子广播协议，Apache Zookeeper）
- Viewstamped Replication (1988)

如果在发送准备消息后协调器失败，两阶段提交(Two-Phase Commit, 2PC)会阻塞，直到协调器恢复。另一方面，如果大多数参与者都活着，只要有足够长的时间没有进一步的故障，Paxos就是非阻塞的。如果节点位于同一个数据中心，不经常发生故障，并且不是恶意的，那么2PC通常比Paxos更受欢迎，因为2PC通常会减少往返。
## 两阶段提交
客户端向协调器发送提交请求。在该协议的第一阶段，协调器发送Prepare消息，主要是询问参与者节点是否允许提交当前事务。如果给定的参与者验证给定的事务是有效的，他们将向协调器发送一个OK。如果协调器收到所有参与者的OK，系统现在可以进入协议的第二阶段。如果有人向协调器发送Abort，则协调器将向客户端发送Abort。
协调器向所有参与者发送Commit，如果所有参与者都发送OK，则告诉这些节点提交事务。一旦参与者响应OK，协调器就可以告诉客户端事务已提交。如果事务在第一阶段被中止，那么参与者将从协调器收到一个Abort，它们应该用OK来响应这个Abort。要么每个人都提交，要么没有人提交。协调器也可以是系统中的参与者。
此外，在崩溃的情况下，所有节点都将跟踪每个阶段结果的非易失性日志。
节点会阻塞，直到它们想出下一步的行动方案。如果协调器崩溃，参与者必须决定做什么。一个安全的选择就是中止。或者，节点可以相互通信，看看它们是否可以在没有协调器的显式许可的情况下提交。如果参与者崩溃，则协调器假定它使用尚未发送确认的中止响应。
### 优化

- 提前准备投票——如果DBMS将查询发送到一个远程节点，它知道这将是最后一个在那里执行的查询，那么该节点也将返回他们对准备阶段的投票和查询结果。
- 准备后的早期确认-如果所有节点投票提交事务，协调器可以在提交阶段完成之前向客户端发送事务成功的确认。
## Paxos
Paxos(以及Raft)在现代系统中比2PC更为普遍。2PC是Paxos的简并情况，Paxos使用2F + 1个协调器，只要至少F + 1个协调器工作正常，2PC使F = 0。Paxos是一个共识协议，其中协调器提出一个结果(例如，提交或中止)，然后参与者投票决定该结果是否应该成功。如果大多数参与者可用，该协议不会阻塞，并且在最佳情况下具有最小的消息延迟。对于Paxos，协调者被称为提议者，参与者被称为接受者。客户端将向提议者发送提交请求。提议者将向系统中的其他节点或接受者发送提议。如果给定的接受者还没有发送一个更高逻辑时间戳的协议，那么他们将发送一个协议。否则，它们将发送一个Reject。一旦大多数接受者发送了一个“同意”，提议者将发送一个“提交”。提议者必须等待收到来自大多数接受者的Accept，然后才向客户端发送最终消息，表示事务已提交，这与2PC不同。在提议失败后，使用指数后退时间来尝试再次提议，以避免提议者之间的决斗。
### Multi-Paxos:
如果系统选出一个领导者在一段时间内监督提议变更，那么它可以跳过提议阶段。系统定期更新谁是使用另一个Paxos轮的领导者。当出现故障时，DBMS可以恢复到完整的Paxos。

## CAP Theorem
由Eric Brewer提出并于2002年在MIT得到证明的CAP定理解释了分布式系统不可能始终保持一致性、可用性和分区容忍度。这三个属性中只能选择两个。
一致性是所有节点上操作的线性化的同义词。一旦写操作完成，以后的所有读操作都应该返回已应用的写操作的值或稍后应用的写操作的值。此外，一旦返回了一个读操作，以后的读操作应该返回该值或以后应用的写操作的值。NoSQL系统牺牲了这一属性，转而支持后两者。其他系统会倾向于这一属性和后两者之一。
可用性是指所有运行的节点都可以满足所有请求的概念。
分区容忍意味着系统仍然可以正常运行，尽管在试图就值达成一致的节点之间有一些消息丢失。如果为系统选择一致性和分区容忍度，则在大多数节点重新连接之前不允许更新，这通常在传统或NewSQL dbms中完成。
有一个考虑一致性与延迟权衡的现代版本:PACELC定理。对于分布式系统中的网络分区(P)，必须在可用性(a)和一致性(C)之间进行选择，否则(E)，即使系统在没有网络分区的情况下正常运行，也必须在延迟(L)和一致性(C)之间进行选择。

 
