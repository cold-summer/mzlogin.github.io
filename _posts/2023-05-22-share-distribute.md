---
layout: post
title: 分布式理论
categories: 分布式
description: 分布式理论
keywords: 分布式, distribute
topmost: false
---

### CAP定理

一致性 （Consistency）：所有节点访问时都是同一份最新的数据副本

可用性 (Availability)：每次请求都有响应，但是不保证获取的数据为最新数据。

分区容错性 (Partition tolerance)：分布式系统在遇到任何网络分区故障的时候，仍然能够对外提供满足一致
性和可用性的服务，除非整个网络环境都发生了故障



CA: 不能容忍网络错误或节点错误,整个系统就会拒绝写请求。

CP：关注的是系统里大多数人 的一致性协议。少数的结点会在没有同步到最新版 本的数据时变成不可用的状态，也就是部分请求响应会超时或报错。

AP ：不能达成一致性，需要给出数据冲突，给出数据冲突就需要维护数据版本



etcd： 使用raft

CAP:

1. CA: RDBMS
2. CP:MongoDB、 Hbase、Redis
3. AP: CouchDB、DynamoDB



1. 弱一致性
    1. 最终一致性
        1. Dns
        2. Gossip（Cassandra通信协议）
2. 强一致性
    1. 同步
    2. Paxos
    3. Raft (multi-paxos)
    4. ZAB ( multi-paxes)



ps： 一致性指强一致性。矛盾主要在一致性和分区容错性

#### base定理

全称:Basically Available(基本可用)，Soft state(软状态),和 Eventually consistent(最终一 致性)三个短语的缩写。

一、基本可用：

- 响应时间变长
- 服务降级、熔断

二、软状态：

- 允许系统中的数据存在中间状态

三、最终一致性：

- 时间延期后，数据最终一致



核心思想：无法做到强一致性



### 一、**两阶段提交协议**(2PC)

**要么所有参与进程都提交事务，要么都取消事务，即实现ACID中的原子性(A)的常用手段**

![image-20230315003637513](http://images.comid.top/image/202307161348084.png)

| 步骤 | 描述                             |
| ---- | -------------------------------- |
| 1.2  | 执行事务 (写本地的Undo/Redo日志) |
| 2.1  | 全部成功则提交，有失败的全部回滚 |



优点：无

缺点：

- 同步阻塞
    - 在二阶段提交的执行过程中，所有参与该事务操作的逻辑都处于阻塞状态，即当参与者占有公
      共资源时，其他节点访问公共资源会处于阻塞状态
- 单点问题
    - 若协调器出现问题，那么整个二阶段提交流程将无法运转，若协调者是在阶段二中出现问题
      时，那么其他参与者将会一直处于锁定事务资源的状态中，而无法继续完成事务操作
- 数据不一致
    - 在阶段二中，执行事务提交的时候，当协调者向所有的参与者发送Commit请求之后，发生了 局部网络异常或者是协调者在尚未发送完Commit请求之前自身发生了崩溃，导致最终只有部 分参与者收到了Commit请求，于是会出现数据不一致的现象。
- 太过保守
    - 没有完善的容错机制，**任意一个结点的失败都会导致整个事务的失败**
    - 参与者出现故障而导致协调者始终无法获取到所有参与者的响 应信息的话，协调者只能依靠自身的超时机制来判断是否需要中断事务

### 二、三阶段提交协议（3PC）

是 2PC 的改进版，将 2PC 的 “提交事务请求” 过程一分为二，共 形成了由CanCommit、PreCommit和doCommit三个阶段组成的事务处理协议。

![image-20230315010212394](http://images.comid.top/image/202411041914051.png)

相对于二阶段的升级点：

1. 三阶段提交协议中 **协调者和参与者**都设置了超时机制
    1. 在2PC中，**只有协调者拥有超时机制**，即如果在一定 时间内没有收到参与者的消息则默认失败
2. 在第一阶段和第二阶段中，引入了一个准备阶段。保证了在最后提交阶段之前各参与节点的状态是
   一致的



**问题点：**

一旦进入阶段三，可能会出现 2 种故障:

- 协调者出现问题
- 协调者和参与者之间的网络故障

出现了任一一种情况，最终都会导致参与者无法收到 doCommit 请求或者 abort 请求，针对 这种情况，参与者都会在等待超时之后，**继续进行事务提交**。

缺点：此时如果原本是abort的请求，到参与者会变成提交事务。





**对比**

|                       | 2PC  | 3PC  | 说明                                                     |
| --------------------- | ---- | ---- | -------------------------------------------------------- |
| 协调者超时机制        | 对   | 对   |                                                          |
| 参与者超时机制        | ❌    | 对   | 解决参与者无法释放资源问题（2PC）,释放行为：默认提交事务 |
| PreCommit**缓冲阶段** | ❌    | 对   | 保证了在最后提交阶段之前各参与节点的状态是一致的         |
|                       |      |      |                                                          |





### 三、NWR协议

NWR是一种在分布式存储系统中用于控制一致性级别的一种策略。在亚马逊的云存储系统中，就应用 NWR来控制一致性。

N:在分布式存储系统中，有多少份备份数据

W:代表一次成功的更新操作要求至少有w份数据写入成功

R: 代表一次成功的读数据操作要求至少有R份数据成功读取



NWR值的不同组合会产生不同的一致性效果

#### 一、**强一致性 W+R>N**

**以常见的**N=3、W=2、R=2为例

- N=3，表示，任何一个对象都必须有三个副本
- W=2 表示，对数据的修改操作只需要在3个副本中的2个上面完成就返回
- R=2  表示，从三个对象中要读取到2个数据对象，才能返回

分布式系统中，通常工业界通常把N设置为3。

![image-20230315011421515](http://images.comid.top/image/202411041914651.png)

可以保证一定能够读取到最新版本的更新数据

在 满足数据一致性协议的前提下，R或者W设置的越大，则系统延迟越大，因为这取决于最慢的那份 备份数据的响应时间。

#### 二、R+W<=N，无法保证数据的强一致性

![image-20230315011610206](http://images.comid.top/image/202411041914533.png)



因为成功写和成功读集合可能不存在交集，这样读操作无法读取到最新的更新数值，也就无法保证数据的强一致性。



### 四、**Gossip** **协议**

详解：https://www.jianshu.com/p/37231c0455a9

**是弱一致性**



不存在中心节点、主从什么的关系

Anti-entropy（反熵），熵混乱程度，反熵就是指 数据达到一致。  传播节点上的所有的数据

Rumor-Mongering（传谣），是传播节点上的新到达的、或者变更的数据

说白了，一个是全量数据，一个是增量数据。



反熵模式下，会同步节点的全部数据，以消除各节点之间的差异，目标是整个网络各节点完全的一致。

传谣模式是以传播消息为目标，仅仅发送新到达节点的数据，即只对外发送变更信息，这样消息数据量将显著缩减，网络开销也相对较小。





关于“传谣”和“反熵”，再借用周志明老师《凤凰架构》里面的正经一点的描述，是这样的：

http://icyfenix.cn/distribution/consensus/gossip.html

达到一致性耗费的时间与网络传播中消息冗余量这两个缺点存在一定对立，如果要改善其中一个，就会恶化另外一个。

https://flopezluis.github.io/gossip-simulator/



同时，gossip 协议它也具备容错性。

传播此时公式

f: 每个节点传播次数

N： 节点数

x：传播多少轮循环

$\log_{f} N \approx x$

例如：

$\log_{4} 100 \approx 3.32193$





新节点加入时又是如何知道集群中一个节点信息的呢？

- 人工指定

Redis 集群采用的就是 gossip 协议来交换信息。当有新节点要加入到集群的时候，需要用到一个 meet 命令。

http://www.redis.cn/commands/cluster-meet.html



1967年，哈佛大学的心理学教授Stanley Milgram想要描绘一个连结人与社区的人际连系网。做过一次连锁信实验，结果发现了“六度分隔”现象。简单地说：“你和任何一个陌生人之间所间隔的人不会超过六个，也就是说，最多通过六个人你就能够认识任何一个陌生人。

**六度分割理论，也叫小世界理论。这其实和 Gossip 协议也有千丝万缕的联系**。

小破站上看到一个相关的视频，我觉得解释的还是挺清楚的：

https://www.bilibili.com/video/BV1b7411B7D2?t=31

$\log_{10} 7800000000 \approx 9.8921$

$\log_{20} 7800000000 \approx 7.6033$

$\log_{44} 7800000000 \approx 6.0191$

$\log_{100} 7800000000 \approx 4.946$

### 五、 Paxos协议

Paxos协议其实说的就是Paxos算法,

Paxos算法是基于**消息传递**且具有**高度容错特性**的**一致性算法**，

是目前公认的解决**分布式一致性**问题**最有效**的算法之一



解决的问题：

1. 机器宕机
2. 网络异常
    1. 延迟、重复、丢失
    2. 网络分区

**目的**： 快速且正确地在集群内部对**某个数据**达成**一致**，并且保证不论发生以上任何异常，都不会破 坏整个系统的一致性。



3PC问题： 协调者为单节点，一旦宕机，参与者会在超时后提交事务，如果原本为回滚则会数据不一致

2PC：参与者没有超时机制，无法释放资源

解决方案：

一、引入多个协调者

![](http://images.comid.top/image/202411041914862.png)

二、引入主协调者,以他的命令为基准

![image-20230315012808131](http://images.comid.top/image/202411041914359.png)



**其实在引入多个协调者之后又引入主协调者。那么这个就是最简单的一种Paxos** 算法

Paxos的版本有: Basic Paxos , Multi Paxos, Fast-Paxos

**具体落地有Raft 和zk的ZAB协议**。

![image-20230315012924978](http://images.comid.top/image/202411041914561.png)



#### 一、**Basic Paxos**

1. 角色介绍
    - Client:客户端
        - 客户端向分布式系统发出请求，并等待响应。例如，对分布式文件服务器中文件的写请求
    - Proposer:提案发起者
        - 提案者提倡客户端请求，试图说服Acceptor对此达成一致，并在发生冲突时充当协调者以推 动协议向前发展
    - Acceptor: 决策者，
        - 可以批准提案Acceptor可以接受(accept)提案;并进行投票, 投票结果是否通过以多数派为准。
        - 以如果某个提案被选定，那么该提案里的value就被选定
    - Learner: 最终决策的学习者
        - 学习者充当该协议的复制因素(不参与投票)

![image-20230315090734396](http://images.comid.top/image/202411041914274.png)



3. basic paxos流程
   basic paxos流程一共分为4个步骤:

    1. Prepare
       Proposer提出一个提案,编号为N, 此N大于这个Proposer之前提出所有提出的编号, 请求Accpetor的多数人接受这个提案

    2. Promise
       如果编号N大于此Accpetor之前接收的任提案编号则接收, 否则拒绝

    3. Accept
       如果达到**多数派**,   即 $ \frac{N}{2} +1$  . N为节点数

       Proposer会发出accept请求, 此请求包含提案编号和对应的内容

    4. Accepted

       如果此Accpetor在此期间没有接受到任何大于N的提案,则接收此提案内容, 否则忽略



#### 二、Basic Paxos流程图

1. 正常流程，无故障的basic Paxos

   ![image-20230315090927262](http://images.comid.top/image/202411041915952.png)

2. Acceptor失败时的basic Paxos

   在下图中，多数派中的一个Acceptor发生故障，因此多数派大小变为2。在这种情况下，Basic Paxos协议仍然成功。

   ![image-20230315091007523](http://images.comid.top/image/202411041915651.png)

3. Proposer失败时的basic Paxos

   Proposer在提出提案之后但在达成协议之前失败。

   传递到Acceptor的时候失败了,这个时候需要选出新的Proposer(提案人),那么 Basic Paxos协议仍然成功

   ![image-20230315091128616](http://images.comid.top/image/202411041915223.png)

4. 当多个提议者发生冲突时的basic Paxos

   最复杂的情况是多个Proposer都进行提案,导致Paxos的**活锁问题**.


![image-20230315091304538](http://images.comid.top/image/202411041915784.png)



解决：只需要在每个Proposer再去提案的时候随机加上一个等待时间即可



basic paxos缺点：

- 首先就是流程复杂,实现及其困难
- 其次效率低(达成一致性需要2轮RPC调用)



#### 三、Multi-Paxos流程图

核心：

- 两轮选举第一次流程选举leader
- 第二次由leader决定

**一、选举leader**

![image-20230315091711523](http://images.comid.top/image/202411041915188.png)

二、leader决定

![image-20230315091804429](http://images.comid.top/image/202411041915542.png)



工业中的multi-Paxos 有 raft 和 ZAB协议。

实际运用会将Proposer，Acceptor和Learner的角色合并统称为“服务器”。

因此， 最后只有“客户端”和“服务器”。

![image-20230315091934295](http://images.comid.top/image/202411041915172.png)



总结：

Paxos 是论证了一致性协议的可行性，但缺少必要的实现细节。

广为人知实现只有 zk 的实现 zab 协议。

后面又出现了Raft协议。



明确问题：

1. 数据不能单点
2. 数据复制：数据复制的算法
3. paxos共识算法



### 六、raft协议算法

然后斯坦福大学RamCloud项目中提出了易实现，易理解的分布式一致性复制协议 Raft。

Java， C++，Go 等都有其对应的实现之后出现的Raft相对要简洁很多。

**引入主节点，通过竞选确定主节点。节 点类型:Follower、Candidate 和Leader**

Leader 会周期性的发送心跳包给 Follower。每个 Follower 都设置了一个随机的竞选超时时间，一 般为 150ms~300ms，

如果在这个时间内没有收到 Leader 的心跳包，就会变成 Candidate，

进入竞选 阶段, 通过竞选阶段的投票多的人成为Leader。



基本流程：

![image-20230316100714117](http://images.comid.top/image/202411041915636.png)



**相关概念**

- Leader(主节点)

    - 接受 client 更新请求，写入本地后，然后同步到其他副本中

- Follower(从节点)

    - 从 Leader 中接受更新请求，然后写入本地日志文件。对客户端提供读 请求

- Candidate(候选节点)

    - 如果 follower 在一段时间内未收到 leader 心跳。则判断 leader可能故障，发起选主提议。节点状态从 Follower 变为 Candidate 状态，直到选主结束

- termId 任期号

    - 时间被划分成一个个任期，每次选举后都会产生一个新的 termId，一个任期内只有一个 leader



引入主节点，通过竞选确定主节点。节 点类型:Follower、Candidate 和Leader

动画演示地址 ：https://raft.github.io/



#### leader选举

一、正常使用，Leader宕机，选出新leader

1. 剩余4个节点中，最先到超时时间的会变成 candidate状态，并将自己的term+1，并向其他节点发送Request cote Rpc请求。
2. 其余3个节点都投赞成票，加上自己一票。就是4票，达成共识，从candidate变成Leader。
3. 正常使用，Leader不断向其余节点发送心跳



二、投票分裂时，选取新Leader

1. 在Leader宕机后，有2个的节点同时超时（例如3、4）。并同时发送拉票请求，都自己投给自己一票。term都+1，接受节点也+1
2. 此时，剩余2个节点（1、2）投票。1->3 , 2-> 4。 此时3和4都2票。 选不出leader
3. 34节点一直向宕机的5节点拉票。
4. 此时1节点超时时间到了。 变成candinate，term+1 变为最新。并向所有节点拉票，此时位candinate的的节点term比1节点低。所有都会投同意票。此时1节点位Leader。

####  日志复制

nextIndex 图中小箭头

mathIndex 图中圆点

虚线：写入本地，但还未向Leader提交，为未提交状态

实现： 日志以提交

框内数字：版本为 N任期的日志，控制新旧及leader选举



1. 2个节点之间的复制（未超过半数节点）
    1. 日志不会提交

2. 3个节点之间提交日志
    1. 从第一条日志开始复制
3. 宕机节点恢复
    1. 新leader会先统一所有节点的nextIndex的值，有新节点加入，leader会做一致性检查，统一nextIndex
    2. 遍历往前检查节点数据。然后往前匹配前一个日志是否一样。直至数据一致为止
    3. 并开始逐条复制

#### 日志不一致修复

1. 如果主节点新入任期为3的日志，宕机节点恢复，先进行日志检查。

    1. 主节点数据强一致性

    1. term > index。  优先选term大的节点进行日志复制。term一致，选index大的节点。

    1. 此时就节点数据被覆盖为新任期数据

#### 日志安全

主节点请求日志后 宕机了。剩余节点开始重新选举

1. 通过比较日志term和index大小，确定新旧日志
    1. term > index。  优先选term大的节点进行日志复制。term一致，选index大的节点
    2. 次数规避旧节点（不符合数据最新要求），重新选取新节点
2. 重复上述逐条match，并逐条复制



etcd： 使用raft



### 七、ZAB协议

基本与Raft相同，在一些名词叫起来是有区别的

ZAB将Leader的一个生命周期叫做epoch，而Raft称之为term

实现上也有些许不同，如raft保证日志的连续性，心跳是Leader向Follower发送，而ZAB方向与之相反

术语说明：

ZXID：高32位是epoch，表示Leader周期，单调递增；低32位，Leader产生一个新的事务proposer时，该计数器会加1(在一个Leader周期里是单调递增，epoch变更后从0开始)

SID：服务器ID，用来唯一标识一台ZooKeeper集群中的服务器，每台机器不能重复，和myid的值一致。

vote_sid:接收到的投票信息中所推举的Leader服务器的SID

vote_zxid:接收到的投票信息中所推举的Leader服务器的ZXID

self_sid:当前服务器自己的SID

self_zxid:当前服务器自己的ZXID

electionEpoch：当前服务器的选举轮次。

Leader选举
1.变更状态：如果是运行过程中Leader挂了，则会有这一步，非Observer服务器会将自己的服务器状态变更为LOOKING（寻找Leader状态），然后进入Leader选举流程

2.每个Server会发出一个投票 vote[self_id，self_zxid]

3.接收来自各个Server的投票 vote[vote_sid,vote_zxid]

4.处理投票

先比较ZXID，ZXID比较大的服务器优先作为Leader

         如果vote_zxid > self_zxid，就认可收到的投票，并再次将该投票发送出去
    
         如果vote_zxid < self_zxid，就坚持自己的投票，不做任何变更

如果ZXID相同的话，那么比较myid，myid比较大的服务器作为Leader服务器。

          如果vote_sid > self_sid，就认可收到的投票，并再次将该投票发送出去
    
          如果vote_sid < self_sid，那么就坚持自己的投票，不做任何变更

5.统计投票：判断是否已经有过半的机器Quorum接收到相同的投票信息，如果存在一个服务器得到过半的票数，那么得到过半票数的机器作为Leader，终止投票。否则进入步骤3，进入到新一轮的投票选举中。

6.改变服务器状态：确定Leader后，每个服务器会更新自己的状态，如果是Follower，那么就变更为FOLLOWING（跟随者状态），如果是Leader，那么就变更为LEADING（领导者状态）

         注意：选举出一个leader后，服务器不会马上更新状态，而是会等待一段时间(默认是200毫秒)来确定是否有新的更优的投票。

Leader同步
由于上述选举过程保证了 Leader 必然拥有最大zxid, Leader 只需要向Follower 同步自己的历史提案即可。

若Follower 拥有 Leader 没有的提案，则 truncat掉。
若Follower的epoch = 当前leader的epoch，落后则根据log, reply 历史transcation
若落后太多，则直接同步 snapshot，再replay transaction log.