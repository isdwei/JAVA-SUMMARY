# Zookeeper

## 1. ACID与CAP

## 2. 分布式一致性协议

常见的分布式一致性协议有: 两阶段提交协议，三阶段提交协议，向量时钟，RWN协议，paxos协议，Raft协议。 

### 2.1 两阶段提交协议(2PC)

两阶段提交协议，简称2PC，是比较常用的解决分布式事务问题的方式，要么所有参与进程都提交事务，要么都取消事务，即实现ACID中的原子性(A)的常用手段。

#### 2.1.1 提交流程

* **提交事务请求**

  * **事务询问**：协调者向所有参与者发送事务内容，询问是否可以提交，等待个参与者响应；
  * **事务执行**：各参与者执行事务，将Undo、Redo信息记入日志；
  * **反馈响应**：参与者成功执行事务则反馈Yes，否则反馈No。

  这一阶段也被称为投票阶段。

* **执行事务提交**

  * **执行事务提交**：如果所有参与者都反馈Yes，则发送事务提交请求。参与者收到Commit请求后正式执行提交，提交完成反馈Ack消息。协调者收到所有参与者反馈的Ack消息后完成事务。
  * **中断事务**：如果任一个参与者反馈了No响应，或等待超时后协调者无法接收所有参与者的反馈响应，那么就会发送回滚请求Rollback。参与者完成回滚后发送Ack。

简单来说**二阶段将事务提交过程分为投票和执行两个阶段，是一个强一致性的算法。**

#### 2.1.2 优缺点

优点：原理简单、实现方便
缺点：同步阻塞、单点问题、脑裂、太过保守

### 2.2 三阶段提交协议(3PC)

3PC就是在2PC基础上将2PC的提交阶段细分位两个阶段：预提交阶段和提交阶段

#### 2.2.1 提交流程

* **CanCommit**

  * 事务询问
  * 反馈询问

* **PreCommit**

  根据CanCommit阶段反馈，执行事务预提交或中断事务

* **doCommit**

  根据PreCommit阶段反馈执行提交或中断事务

在进入三阶段后，如果协调者出现问题或网络故障，导致参与者无法及时接收到协调者的请求，参与者会在等待超时后继续进行事务提交。

#### 2.2.2 优缺点

优点：相较于2PC，降低了阻塞范围，并能在单点故障后继续达成一致。
缺点：引入新的问题，参与者接收到preCommit消息后如果发生网络故障，仍然会继续提交，会造成数据不一致。

### 2.3 Paxos算法

一种基于消息传递且具有高度容错性的一致性算法，是目前解决分布式一致性问题最有效的算法之一。

 https://chuansongme.com/n/2189245 

#### 2.3.1 提交流程

* **第一阶段 Prepare**

  * **P1a：Proposer 发送 Prepare**

    Proposer 生成全局唯一且递增的提案 ID（Proposalid，以高位时间戳 + 低位机器 IP 可以保证唯一性和递增性），向 Paxos 集群的所有机器发送 PrepareRequest，这里无需携带提案内容，只携带 Proposalid 即可。

  - **P1b：Acceptor 应答 Prepare**
    Acceptor 收到 PrepareRequest 后，做出“两个承诺，一个应答”。

    **两个承诺：**

    - 第一，不再应答 Proposalid 小于等于（注意：这里是 <= ）当前请求的 PrepareRequest；
    - 第二，不再应答 Proposalid 小于（注意：这里是 < ）当前请求的 AcceptRequest

    **一个应答：**

    - 返回自己已经 Accept 过的提案中 ProposalID 最大的那个提案的内容，如果没有则返回空值;

    注意：这“两个承诺”中，蕴含两个要点：就是应答当前请求前，也要按照“两个承诺”检查是否会违背之前处理 PrepareRequest 时做出的承诺；应答前要在本地持久化当前 Propsalid。

- **第二阶段 Accept**

  - **P2a：Proposer 发送 Accept**
    “提案生成规则”：Proposer 收集到多数派应答的 PrepareResponse 后，从中选择proposalid最大的提案内容，作为要发起 Accept 的提案，如果这个提案为空值，则可以自己随意决定提案内容。然后携带上当前 Proposalid，向 Paxos 集群的所有机器发送 AccpetRequest。

  - **P2b：Acceptor 应答 Accept**

    Accpetor 收到 AccpetRequest 后，检查不违背自己之前作出的“两个承诺”情况下，持久化当前 Proposalid 和提案内容。最后 Proposer 收集到多数派应答的 AcceptResponse 后，形成决议。

#### 2.4 ZAB协议（Zookeeper原子消息协议）

ZAB是一种特别为Zookeeper实现的崩溃可恢复的原子消息广播算法。

基于ZAB，Zookeeper使用一个单一的主进程来接收并处理客户端的所有事务请求，并采用ZAB协议将服务器数据的状态变更以事务Proposal的形式广播到所有的副本进程上去。

> 所有的事务请求必须由一个全局唯一的服务器来协调处理，即Leader；
> 而其他服务器作为Follower。Leader将一个客户端事务请求转化为一个事务Proposal，并将其分发所有Follower。之后Leader等待所有Follower的反馈，超过半数正确则会向所有Follower分发Commit消息。

ZAB协议主要用于构建一个高可用的分布式数据主备系统，而Paxos算法则是用于构建一个分布式的一致性状态机系统。

## 3. Zookeeper

Zookeeper 是 一个典型的分布式数据一致性的解决方案。

### 3.1 Zookeeper的典型应用场景

* 数据发布/订阅：将配置文件保存在Zookeeper节点中
* 负载均衡：动态DNS
* 命名服务：利用顺序节点的创建获得全局唯一ID
* 分布式协调/通知：利用Watcher机制完成实时数据复制
* 集群管理
* Master选举：利用Zookeeper的强一致性
* 分布式锁：利用一个数据节点实现锁，
  * 排它锁：获取锁即是创建一个临时节点，释放锁即是删除该节点
  * 共享锁：获取锁是创建一个临时顺序节点。对于读请求，比自己序号小的节点都是读请求，则说明获取到了共享锁，如果比自己小的节点有写请求，需要进入等待（只需向最后一个写请求节点注册监听）；对于写请求，如果自己不是序号最小的节点就要等待（只需对前一个节点注册监听）。释放锁就是删除该节点。
* 分布式队列：
  * FIFO：利用临时顺序节点，类似于一个全写的共享锁模型
  * Barrier：分布式屏障，节点数超过阈值时才可以进行业务处理

#### 3.1.1 Zookeeper在HadoopHA中的应用

##### **主备切换**

* 创建锁节点：所有RM在启动时都会竞争一个临时Lock子节点。Zookeeper能够保证最后只有一个可以成功。
* 注册Watcher监听：所有的RM都会注册一个节点监听，感知Active的RM的运行情况
* 主备切换：当Active的RM挂掉，其在Zookeeper上创建的Lock也会随之被删除，Watch该节点的RM都会收到通知，竞争创建新的锁节点。

##### **Fencing**（隔离）

如果出现假死现象，原先Active的RM恢复后发现Zookeeper上相关节点不是自己创建的，就会自动切换到Standby状态防止脑裂出现。

##### **ResourceManager状态存储**

Zookeeper中，RM的状态信息都存储在/rmstore下。包括各个Application的信息，安全相关的Token信息

#### 3.1.2 Zookeeper在Kafka中的应用

##### Broker注册

Zookeeper中使用/brokers/ids/存储所有节点信息。每个Broker在启动时都会到Zookeeper下创建一个临时节点。

##### Topic注册

每个Topic的存储位置在/brokers/topics。Broker启动后会到对应的Topic节点下注册自己的BrokerID，并写入Topic的分区总数，如`/brokers/topic/login/3->2`。

##### 生产者负载均衡

Kafka生产者会对Zookeeper上Broker的新增与减少、Topic的新增与减少和Broker与Topic关联关系的变化等事件注册监听，这样就可以实现动态的负载均衡机制。

##### 消费者负载均衡

Zookeeper中记录了消息分区与消费者的对应关系，一旦一个消费者缺点了对一个消息分区消费的权利，则将其ConsumerID写入对应消息分区的临时节点上。

现在消费者的数据已经保存在Kafka集群中。

### 3.2 Zookeeper分布式一致性特性

* 顺序一致性：同一个客户端发出的请求会严格的按顺序被应用到Zookeeper中去。
* 原子性：所有事务请求的处理都是原子的。
* 单一视图：无论客户端连接的是哪个Zookeeper服务器，其看到的服务端数据模型都是一致的。
* 可靠性：一旦服务端成功应用了一个事务，其影响会一直被保留下来。
* 实时性：Zookeeper**仅能保证在一定时间内**，客户端最终一定能从服务端读到最新的数据。

### 3.3 Zookeeper设计目标

* 简单的数据模型：Znode数据节点
* 可以构建集群
* 顺序访问：客户端的每一个请求，Zookeeper都会分配全局递增的ID
* 高性能：数据存储于内存中

### 3.4 Zookeeper基本架构

#### 3.4.1 数据模型

Zookeeper的视图结构与Unix相似，但没有引入传统文件系统中目录和文件的概念，而是使用了数据节点ZNode的概念。Zookeeper使用ZNode保存数据和挂载子节点，构成了树形文件系统，ZNode中还保存了其所有的状态信息，包括创建时间、修改时间、版本号等。

##### 事务ID

Zookeeper中的事务表示能够改变Zookeeper服务器状态的操作，一般包括数据节点的创建删除、内容更新和客户端会话的创建与失效。

Zookeeper为每个事务分配全局唯一大的ZXID，通常时64位数字，从ZXID中可以判断事务请求的顺序。

##### 节点类型

* 持久节点（PERSISTENT）节点被创建后就持久化 存在于Zookeeper服务器上；
* 临时节点（EPHEMERAL）临时节点的生命周期与客户端会话绑定在一起。临时节点只能作为叶子节点。
* 顺序节点（SEQUENTIAL）每个父节点会为它的第一级子节点维护一份顺序。Zookeeper会在创建子节点时在节点名后加上一个数字后缀，数字后缀的最大值是`Integer.MAX_VALUE`

##### 版本——保证分布式数据原子性操作

ZNode中保存了三种版本信息，包括：

* version：当前数据节点的数据内容版本号
* cversion：当前数据节点子节点的版本号
* aversion：当前数据节点的ACL变更版本号

版本号表示对数据节点的数据内容、子节点列表或ACL信息的修改次数。

基于版本号，Zookeeper可以实现类似于JDK中CAS的乐观锁机制。Zookeeper会从setDataRequest请求中获取当前请求版本的version，同时检查数据记录nodeRecord中当前版本。如果version是-1，则表示不使用锁；否则只有版本号一致才能成功修改。

##### Watcher——数据变更的通知

客户端在向Zookeeper服务器注册Watcher同时，会将Watcher对象存储在客户端的WatchManager上。当Zookeeper服务器端触发Watcher事件后，会向客户端发送通知，客户端线程从WatcManager中取出对应的Watcher对象来执行回调逻辑。

Watcher具有以下几个特性：

* 一次性：一旦一个Watcher被触发，Zookeeper都会将去从相应的存储中删除。开发时需要反复注册。
* 客户端串行执行：Watcher回调过程是一个串行同步的过程
* 轻量：WatcherEvent时整个Watcher通知机制的最小单元，这个数据结构只包含三部分：通知状态、事件类型、节点路径，而不包含事件的具体内容。

##### ACL——保障数据安全

ACL（Access Control List）权限控制机制保障数据安全。

* 权限模式（Scheme）：用于确定验证过程中使用的检验策略，包括，
  * IP：通过IP地址粒度进行权限控制
  * Digest：最常用的模式，以类似于`username：password`的形式进行权限配置
  * World：最开放的权限控制模式，向所有用户开放
  * Super：超级用户，可对任意数据进行操作
* 授权对象（ID）:权限赋予的用户
* 权限（Permission）:被允许执行的操作。包括，
  * C（Create）：允许在本节点下创建子节点
  * D（Delete）：允许在本节点下删除子节点
  * R（Read）：允许读本数据节点和子节点
  * W（Write）：允许对象对本节点更新
  * A（Admin）：管理权限，允许进行ACL相关的设置

集群角色  Leader，Follower，Observer

#### 3.4.2 会话

Zookeeper的连接与会话就是客户端通过实例化Zookeeper对象来实现客户端与服务器创建并保持TCP连接的过程。

##### 会话状态

一个会话的状态包括CONNECTING、CONNECTED、RECONNECTING、RECONNECTED、CLOSE。

##### 会话创建

会话包括四个基本属性：

* sessionID：long nextSid = ((System.currentTimeMills()<<24) >>> 8)|(id<<56);
* TimeOut、TickTime、isClosing

#### 3.4.3 数据节点Znode

#### 3.4.4 版本

#### 3.4.5 Watcher

#### 3.4.6 ACL



