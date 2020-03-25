# Spark内核源码阅读笔记

# **【Spark 内核】 Spark 内核解析-上**

Spark内核泛指Spark的核心运行机制，包括Spark核心组件的运行机制、Spark任务调度机制、Spark内存管理机制、Spark核心功能的运行原理等，熟练掌握Spark内核原理，能够帮助我们更好地完成Spark代码设计，并能够帮助我们准确锁定项目运行过程中出现的问题的症结所在。

## Spark 内核概述

### Spark 核心组件回顾

#### Driver

Spark驱动器节点，用于执行Spark任务中的main方法，负责实际代码的执行工作。Driver在Spark作业执行时主要负责：

1. 将用户程序转化为作业（job）；
2. 在Executor之间调度任务(task)；
3. 跟踪Executor的执行情况；
4. 通过UI展示查询运行情况；

#### Executor

Spark Executor节点是一个JVM进程，负责在 Spark 作业中运行具体任务，任务彼此之间相互独立。Spark 应用启动时，Executor节点被同时启动，并且始终伴随着整个 Spark 应用的生命周期而存在。如果有Executor节点发生了故障或崩溃，Spark 应用也可以继续执行，会将出错节点上的任务调度到其他Executor节点上继续运行。

Executor有两个核心功能:

1. 负责运行组成Spark应用的任务，并将结果返回给驱动器进程；
2. 它们通过自身的块管理器（Block Manager）为用户程序中要求缓存的 RDD 提供内存式存储。RDD 是直接缓存在Executor进程内的，因此任务可以在运行时充分利用缓存数据加速运算。

### Spark 通用运行流程概述

<img src="http://img4.sycdn.imooc.com/5e1c416e0001d6d208350622.jpg" alt="Spark核心运行流程" style="zoom: 60%;" />
图为Spark通用运行流程，不论Spark以何种模式进行部署，任务提交后，都会先启动Driver进程，随后Driver进程向集群管理器注册应用程序，之后集群管理器根据此任务的配置文件分配Executor并启动，当Driver所需的资源全部满足后，Driver开始执行main函数，Spark查询为懒执行，当执行到action算子时开始反向推算，根据宽依赖进行stage的划分，随后每一个stage对应一个taskset，taskset中有多个task，根据本地化原则，task会被分发到指定的Executor去执行，在任务执行的过程中，Executor也会不断与Driver进行通信，报告任务运行情况。

## Spark 部署模式

Spark支持3种集群管理器（Cluster Manager），分别为：

1. Standalone：独立模式，Spark原生的简单集群管理器，自带完整的服务，可单独部署到一个集群中，无需依赖任何其他资源管理系统，使用Standalone可以很方便地搭建一个集群；
2. Apache Mesos：一个强大的分布式资源管理框架，它允许多种不同的框架部署在其上，包括yarn；
3. Hadoop YARN：统一的资源管理机制，在上面可以运行多套计算框架，如mapreduce、storm等，根据driver在集群中的位置不同，分为yarn client和yarn cluster。
   实际上，除了上述这些通用的集群管理器外，Spark内部也提供了一些方便用户测试和学习的简单集群部署模式。由于在实际工厂环境下使用的绝大多数的集群管理器是Hadoop YARN，因此我们关注的重点是Hadoop YARN模式下的Spark集群部署。
   Spark的运行模式取决于传递给SparkContext的MASTER环境变量的值，个别模式还需要辅助的程序接口来配合使用，目前支持的Master字符串及URL包括：

| **Master URL**        | **Meaning**                                                  |
| :-------------------- | :----------------------------------------------------------- |
| **local**             | 在本地运行，只有一个工作进程，无并行计算能力。               |
| **local[K]**          | 在本地运行，有K个工作进程，通常设置K为机器的CPU核心数量。    |
| **local[\*]**         | 在本地运行，工作进程数量等于机器的CPU核心数量。              |
| **spark://HOST:PORT** | 以Standalone模式运行，这是Spark自身提供的集群运行模式，默认端口号: 7077。详细文档见:Spark standalone cluster。 |
| **mesos://HOST:PORT** | 在Mesos集群上运行，Driver进程和Worker进程运行在Mesos集群上，部署模式必须使用固定值:–deploy-mode cluster。详细文档见:MesosClusterDispatcher. |
| **yarn-client**       | 在Yarn集群上运行，Driver进程在本地，Executor进程在Yarn集群上，部署模式必须使用固定值:–deploy-mode client。Yarn集群地址必须在HADOOP_CONF_DIR or YARN_CONF_DIR变量里定义。 |
| **yarn-cluster**      | 在Yarn集群上运行，Driver进程在Yarn集群上，Work进程也在Yarn集群上，部署模式必须使用固定值:–deploy-mode cluster。Yarn集群地址必须在HADOOP_CONF_DIR or YARN_CONF_DIR变量里定义。 |

用户在提交任务给Spark处理时，以下两个参数共同决定了Spark的运行方式。
· –master MASTER_URL ：决定了Spark任务提交给哪种集群处理。
· –deploy-mode DEPLOY_MODE：决定了Driver的运行方式，可选值为Client或者Cluster。

### Standalone 模式运行机制

Standalone集群有四个重要组成部分，分别是:

1. Driver：是一个进程，我们编写的Spark应用程序就运行在Driver上，由Driver进程执行；
2. Master(RM)：是一个进程，主要负责资源的调度和分配，并进行集群的监控等职责；
3. Worker(NM)：是一个进程，一个Worker运行在集群中的一台服务器上，主要负责两个职责，一个是用自己的内存存储RDD的某个或某些partition；另一个是启动其他进程和线程（Executor），对RDD上的partition进行并行的处理和计算。
4. Executor：是一个进程，一个Worker上可以运行多个Executor，Executor通过启动多个线程（task）来执行对RDD的partition进行并行计算，也就是执行我们对RDD定义的例如map、flatMap、reduce等算子操作。

#### Standalone Client 模式

![Standalone Client 模式](http://img2.sycdn.imooc.com/5e1c417000016e2910660548.jpg)

在Standalone Client模式下，Driver在任务提交的本地机器上运行，Driver启动后向Master注册应用程序，Master根据submit脚本的资源需求找到内部资源至少可以启动一个Executor的所有Worker，然后在这些Worker之间分配Executor，Worker上的Executor启动后会向Driver反向注册，所有的Executor注册完成后，Driver开始执行main函数，之后执行到Action算子时，开始划分stage，每个stage生成对应的taskSet，之后将task分发到各个Executor上执行。

#### Standalone Cluster模式

![Standalone Cluster模式](http://img3.sycdn.imooc.com/5e1c417200013e9410290470.jpg)
在Standalone Cluster模式下，任务提交后，Master会找到一个Worker启动Driver进程， Driver启动后向Master注册应用程序，Master根据submit脚本的资源需求找到内部资源至少可以启动一个Executor的所有Worker，然后在这些Worker之间分配Executor，Worker上的Executor启动后会向Driver反向注册，所有的Executor注册完成后，Driver开始执行main函数，之后执行到Action算子时，开始划分stage，每个stage生成对应的taskSet，之后将task分发到各个Executor上执行。
**注意**
Standalone的两种模式下（client/Cluster），Master在接到Driver注册Spark应用程序的请求后，会获取其所管理的剩余资源能够启动一个Executor的所有Worker，然后在这些Worker之间分发Executor，此时的分发只考虑Worker上的资源是否足够使用，直到当前应用程序所需的所有Executor都分配完毕，Executor反向注册完毕后，Driver开始执行main程序。

### Yarn 模式运行机制

#### Yarn Client 模式

![Yarn Client 模式](http://img2.sycdn.imooc.com/5e1c41740001ff9712450741.jpg)
在YARN Client模式下，Driver在任务提交的本地机器上运行，Driver启动后会和ResourceManager通讯申请启动ApplicationMaster，随后ResourceManager分配container，在合适的NodeManager上启动ApplicationMaster，此时的ApplicationMaster的功能相当于一个ExecutorLaucher，只负责向ResourceManager申请Executor内存。

ResourceManager接到ApplicationMaster的资源申请后会分配container，然后ApplicationMaster在资源分配指定的NodeManager上启动Executor进程，Executor进程启动后会向Driver反向注册，Executor全部注册完成后Driver开始执行main函数，之后执行到Action算子时，触发一个job，并根据宽依赖开始划分stage，每个stage生成对应的taskSet，之后将task分发到各个Executor上执行。

#### Yarn Cluster 模式

![Yarn Cluster 模式](http://img3.sycdn.imooc.com/5e1c4175000117b812010499.jpg)

在YARN Cluster模式下，任务提交后会和ResourceManager通讯申请启动ApplicationMaster，随后ResourceManager分配container，在合适的NodeManager上启动ApplicationMaster，此时的ApplicationMaster就是Driver。

Driver启动后向ResourceManager申请Executor内存，ResourceManager接到ApplicationMaster的资源申请后会分配container，然后在合适的NodeManager上启动Executor进程，Executor进程启动后会向Driver反向注册，Executor全部注册完成后Driver开始执行main函数，之后执行到Action算子时，触发一个job，并根据宽依赖开始划分stage，每个stage生成对应的taskSet，之后将task分发到各个Executor上执行。

## Spark 通讯架构

### Spark 通信架构概述

Spark2.x版本使用Netty通讯框架作为内部通讯组件。spark 基于netty新的rpc框架借鉴了Akka的中的设计，它是基于Actor模型，如下图所示：
<img src="http://img1.sycdn.imooc.com/5e1c41760001f0b009270612.jpg" alt="Actor" style="zoom: 50%;" />
Spark通讯框架中各个组件（Client/Master/Worker）可以认为是一个个独立的实体，各个实体之间通过消息来进行通信。具体各个组件之间的关系图如下：
<img src="http://img2.sycdn.imooc.com/5e1c41770001d37e07690553.jpg" alt="Spark通讯架构" style="zoom: 67%;" />
Endpoint（Client/Master/Worker）有1个InBox和N个OutBox（N>=1，N取决于当前Endpoint与多少其他的Endpoint进行通信，一个与其通讯的其他Endpoint对应一个OutBox），Endpoint接收到的消息被写入InBox，发送出去的消息写入OutBox并被发送到其他Endpoint的InBox中。

### Spark 通讯架构解析

Spark通信架构如下图所示：
![Spark通信架构-2](http://img2.sycdn.imooc.com/5e1c417800018c1609640575.jpg)

1. RpcEndpoint：RPC端点，Spark针对每个节点（Client/Master/Worker）都称之为一个Rpc端点，且都实现RpcEndpoint接口，内部根据不同端点的需求，设计不同的消息和不同的业务处理，如果需要发送（询问）则调用Dispatcher；
2. RpcEnv：RPC上下文环境，每个RPC端点运行时依赖的上下文环境称为RpcEnv；
3. Dispatcher：消息分发器，针对于RPC端点需要发送消息或者从远程RPC接收到的消息，分发至对应的指令收件箱/发件箱。如果指令接收方是自己则存入收件箱，如果指令接收方不是自己，则放入发件箱；
4. Inbox：指令消息收件箱，一个本地RpcEndpoint对应一个收件箱，Dispatcher在每次向Inbox存入消息时，都将对应EndpointData加入内部ReceiverQueue中，另外Dispatcher创建时会启动一个单独线程进行轮询ReceiverQueue，进行收件箱消息消费；
5. RpcEndpointRef：RpcEndpointRef是对远程RpcEndpoint的一个引用。当我们需要向一个具体的RpcEndpoint发送消息时，一般我们需要获取到该RpcEndpoint的引用，然后通过该应用发送消息。
6. OutBox：指令消息发件箱，对于当前RpcEndpoint来说，一个目标RpcEndpoint对应一个发件箱，如果向多个目标RpcEndpoint发送信息，则有多个OutBox。当消息放入Outbox后，紧接着通过TransportClient将消息发送出去。消息放入发件箱以及发送过程是在同一个线程中进行；
7. RpcAddress：表示远程的RpcEndpointRef的地址，Host + Port。
8. TransportClient：Netty通信客户端，一个OutBox对应一个TransportClient，TransportClient不断轮询OutBox，根据OutBox消息的receiver信息，请求对应的远程TransportServer；
9. TransportServer：Netty通信服务端，一个RpcEndpoint对应一个TransportServer，接受远程消息后调用Dispatcher分发消息至对应收发件箱；
   根据上面的分析，Spark通信架构的高层视图如下图所示：
   <img src="http://img3.sycdn.imooc.com/5e1c41790001ef9e06980614.jpg" alt="Spark通信架构高层视图" style="zoom:67%;" />

## Spark 任务调度机制

在工厂环境下，Spark集群的部署方式一般为YARN-Cluster模式，之后的内核分析内容中我们默认集群的部署方式为YARN-Cluster模式。

### Spark 任务提交流程

![Yarn-Cluster 任务提交流程](http://img1.sycdn.imooc.com/5e1c417a000117b812010499.jpg)
下面的时序图清晰地说明了一个Spark应用程序从提交到运行的完整流程：
![Spark任务提交时序图](http://img1.sycdn.imooc.com/5e1c417a000188be09820796.jpg)
提交一个Spark应用程序，首先通过Client向ResourceManager请求启动一个Application，同时检查是否有足够的资源满足Application的需求，如果资源条件满足，则准备ApplicationMaster的启动上下文，交给ResourceManager，并循环监控Application状态。

当提交的资源队列中有资源时，ResourceManager会在某个NodeManager上启动ApplicationMaster进程，ApplicationMaster会单独启动Driver后台线程，当Driver启动后，ApplicationMaster会通过本地的RPC连接Driver，并开始向ResourceManager申请Container资源运行Executor进程（一个Executor对应与一个Container），当ResourceManager返回Container资源，ApplicationMaster则在对应的Container上启动Executor。

Driver线程主要是初始化SparkContext对象，准备运行所需的上下文，然后一方面保持与ApplicationMaster的RPC连接，通过ApplicationMaster申请资源，另一方面根据用户业务逻辑开始调度任务，将任务下发到已有的空闲Executor上。

当ResourceManager向ApplicationMaster返回Container资源时，ApplicationMaster就尝试在对应的Container上启动Executor进程，Executor进程起来后，会向Driver反向注册，注册成功后保持与Driver的心跳，同时等待Driver分发任务，当分发的任务执行完毕后，将任务状态上报给Driver。

从上述时序图可知，Client只负责提交Application并监控Application的状态。对于Spark的任务调度主要是集中在两个方面: 资源申请和任务分发，其主要是通过ApplicationMaster、Driver以及Executor之间来完成。

### Spark 任务调度概述

当Driver起来后，Driver则会根据用户程序逻辑准备任务，并根据Executor资源情况逐步分发任务。在详细阐述任务调度前，首先说明下Spark里的几个概念。一个Spark应用程序包括Job、Stage以及Task三个概念：

* Job是以Action方法为界，遇到一个Action方法则触发一个Job；
* Stage是Job的子集，以RDD宽依赖(即Shuffle)为界，遇到Shuffle做一次划分；
* Task是Stage的子集，以并行度(分区数)来衡量，分区数是多少，则有多少个task。
  Spark的任务调度总体来说分两路进行，一路是Stage级的调度，一路是Task级的调度，总体调度流程如下图所示：
  ![Spark 任务调度概述](http://img3.sycdn.imooc.com/5e1c417c000175ff11310873.jpg)

Spark RDD通过其Transactions操作，形成了RDD血缘关系图，即DAG，最后通过Action的调用，触发Job并调度执行。DAGScheduler负责Stage级的调度，主要是将job切分成若干Stages，并将每个Stage打包成TaskSet交给TaskScheduler调度。TaskScheduler负责Task级的调度，将DAGScheduler给过来的TaskSet按照指定的调度策略分发到Executor上执行，调度过程中SchedulerBackend负责提供可用资源，其中SchedulerBackend有多种实现，分别对接不同的资源管理系统。有了上述感性的认识后，下面这张图描述了Spark-On-Yarn模式下在任务调度期间，ApplicationMaster、Driver以及Executor内部模块的交互过程：
![Job提交和Task拆分](http://img4.sycdn.imooc.com/5e1c417d0001375710550707.jpg)

Driver初始化SparkContext过程中，会分别初始化DAGScheduler、TaskScheduler、SchedulerBackend以及HeartbeatReceiver，并启动SchedulerBackend以及HeartbeatReceiver。SchedulerBackend通过ApplicationMaster申请资源，并不断从TaskScheduler中拿到合适的Task分发到Executor执行。HeartbeatReceiver负责接收Executor的心跳信息，监控Executor的存活状况，并通知到TaskScheduler。

### Spark Stage级调度

Spark的任务调度是从DAG切割开始，主要是由DAGScheduler来完成。当遇到一个Action操作后就会触发一个Job的计算，并交给DAGScheduler来提交，下图是涉及到Job提交的相关方法调用流程图。
![Job提交调用栈](http://img2.sycdn.imooc.com/5e1c417e0001e90212360374.jpg)

Job由最终的RDD和Action方法封装而成，SparkContext将Job交给DAGScheduler提交，它会根据RDD的血缘关系构成的DAG进行切分，将一个Job划分为若干Stages，具体划分策略是，由最终的RDD不断通过依赖回溯判断父依赖是否是宽依赖，即以Shuffle为界，划分Stage，窄依赖的RDD之间被划分到同一个Stage中，可以进行pipeline式的计算，如上图紫色流程部分。划分的Stages分两类，一类叫做ResultStage，为DAG最下游的Stage，由Action方法决定，另一类叫做ShuffleMapStage，为下游Stage准备数据，下面看一个简单的例子WordCount。
![WordCount实例](http://img1.sycdn.imooc.com/5e1c417f000111b608310305.jpg)
Job由saveAsTextFile触发，该Job由RDD-3和saveAsTextFile方法组成，根据RDD之间的依赖关系从RDD-3开始回溯搜索，直到没有依赖的RDD-0，在回溯搜索过程中，RDD-3依赖RDD-2，并且是宽依赖，所以在RDD-2和RDD-3之间划分Stage，RDD-3被划到最后一个Stage，即ResultStage中，RDD-2依赖RDD-1，RDD-1依赖RDD-0，这些依赖都是窄依赖，所以将RDD-0、RDD-1和RDD-2划分到同一个Stage，即ShuffleMapStage中，实际执行的时候，数据记录会一气呵成地执行RDD-0到RDD-2的转化。不难看出，其本质上是一个深度优先搜索算法。

**一个Stage是否被提交，需要判断它的父Stage是否执行，只有在父Stage执行完毕才能提交当前Stage，如果一个Stage没有父Stage，那么从该Stage开始提交。**Stage提交时会将Task信息（分区信息以及方法等）序列化并被打包成TaskSet交给TaskScheduler，一个Partition对应一个Task，另一方面TaskScheduler会监控Stage的运行状态，只有Executor丢失或者Task由于Fetch失败才需要重新提交失败的Stage以调度运行失败的任务，其他类型的Task失败会在TaskScheduler的调度过程中重试。

相对来说DAGScheduler做的事情较为简单，仅仅是在Stage层面上划分DAG，提交Stage并监控相关状态信息。TaskScheduler则相对较为复杂，下面详细阐述其细节。


作者：南风意未起链接：http://www.imooc.com/article/299663?block_id=tuijian_wz来源：慕课网