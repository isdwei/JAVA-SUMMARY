# Hadoop学习笔记

文件配置

如何配置安装Hadoop

Linux环境

Ssh免密码登录

安装jdk

安装hadoop,配置文件

hadoop namenode –format

Hadoop中的配置文件

Core-site.xml，Hadoop-env.sh，Hdfs-site.xml，Mapred-site.xml

Hadoop中启动的进程

NameNode, SecondaryNameNode, DataNode, ResourceManage, NodeManage

默认端口号

50070，8088

HDFS

文件块（Block）

Hadoop2.x 128M hadoop1.x 64M 通过dfs.blocksize规定

HDFS写数据流程

![img](file:///C:/Users/weitu/AppData/Local/Temp/msohtmlclip1/01/clip_image002.png)

HDFS读数据流程

![img](file:///C:/Users/weitu/AppData/Local/Temp/msohtmlclip1/01/clip_image004.png)

SecondaryNameNode工作机制

![img](file:///C:/Users/weitu/AppData/Local/Temp/msohtmlclip1/01/clip_image006.png)

SecondaryNameNode与NameNode的区别

1） NameNode负责管理整个文件系统的元数据，以及每一个路径所对应的数据块信息。

2） SecondaryNameNode用于等起合并命名空间镜像和编辑日志。

3） 在NameNode故障时SecondaryNameNode可以用于恢复（有损）。

MapReduce

Hadoop序列化与反序列化

1） 必须实现Writable接口；

2） 反序列化时需要用到反射，所以应有空参构造器；

3） 重写序列化/反序列化方法（顺序必须完全一致）

4） 若自定义Bean放在Key中，需实现Comparable接口。

FileInputFormat切片机制

FileInputFormat切片源码解析(input.getSplits(job))

![img](file:///C:/Users/weitu/AppData/Local/Temp/msohtmlclip1/01/clip_image008.png)

如何决定Map和Reduce的数量

1） map的数量：splitSize=max(minSize,min{maxSize,blockSize}) map个数由切片数决定

2） reduce的数量：job.setNumReduceTasks(x);默认为1

MapReduce工作机制

![img](file:///C:/Users/weitu/AppData/Local/Temp/msohtmlclip1/01/clip_image010.png)

图4-6 MapReduce详细工作流程（一）

![img](file:///C:/Users/weitu/AppData/Local/Temp/msohtmlclip1/01/clip_image012.png)

MapReduce中的排序

快速排序，归并排序

Shuffle阶段的工作流程，如何优化shuffle

分区，排序，溢写，拷贝到对应的reduce机器上，

优化：增加combiner（不可对），增加环形缓冲区大小（默认100M），增大溢写的阈值（默认80%），增大合并溢写文件的此处，reduce端可部分数据存放在内存中，对中间文件压缩。

Combiner的作用，使用场景，和reduce的区别

作用：对每一个mapTask进行局部汇总，减小io

使用场景：不能影响最终业务逻辑，如求均值不可，汇总可以；combiner的输出（k,v）类型要和reduce的输入（k,v）类型对应起来。

区别：combiner在每一个mapTask所在的节点运行，reducer是在接收全局所有的Mapper的输出结果。

Partition作用，默认的Partition方法

作用：不同的分区号会对应不同的reduceTask。

默认：每一条数据的key的hashCode值%redduce的数量。

TopN如何实现

小顶堆，Reduce端放入TreeMap

Hadoop任务输出到多个目录中/MySQL/HBase...

自定义OutputFormat，改写recordwriter.write()。

Hadoop实现join的几种方法

Reduce join（连接键作为k）

map join(推荐)(驱动中加载缓存，重写setup阶段)

map端二级排序

重写compareTo()

Hdfs分块

文件在hdfs中分块，一旦超过块大小就会新增一个块

mr的 split中分片，如果超出1.1倍才会被分片

Hadoop中RecordReader作用是什么

在FileInputFormat中，用来读文件，默认按行读（LineRecordReader）。

手撕MapReduce代码

 

Yarn

hadoop1.x hadoop2.x异同

加入yarn解决了资源调度的问题

加入了对zookeeper的支持实现了比较高的高可用

yarn的优势

解决用户程序与yarn框架的完全解耦，yarn上可以运行各种分布式运算程序

MR作业job的提交流程

![img](file:///C:/Users/weitu/AppData/Local/Temp/msohtmlclip1/01/clip_image014.png)

![img](file:///C:/Users/weitu/AppData/Local/Temp/msohtmlclip1/01/clip_image016.png)

HDFS中的数据压缩算法

Gzip 不支持Spill  压缩率较高，自带

Bzip2 支持Spill   压缩率很高，自带    压缩解压慢

Lzo  支持Spill   压缩率合理，不自带   压缩解压快 最流行

Snappy 不支持Spill 压缩率比Gzip低，不自带   高速压缩  

资源调度器

Hadoop作业调度器主要有三种：FIFO、Capacity Scheduler和Fair Scheduler。

Hadoop2.7.2默认的资源调度器是Capacity Scheduler。（容量调度器）

支持多队列，每个队列分配一定的资源，每个队列FIFO。

MR推测执行原理

![img](file:///C:/Users/weitu/AppData/Local/Temp/msohtmlclip1/01/clip_image018.png)

不能启用推测执行机制情况

  （1）任务间存在严重的负载倾斜；

  （2）特殊任务，比如任务向数据库中写数据。

Mapreduce优化

Mapreduce跑的慢的原因

1.计算机性能

2.I/O操作优化：数据倾斜，map/reduce任务数设计不合理，map运行时间太长，小文件过多，大量不可分的超大文件，spill次数过多，merge次数过多

Mapreduce优化方法

MapReduce优化方法主要从六个方面考虑：数据输入、Map阶段、Reduce阶段、IO传输、数据倾斜问题和常用的调优参数。

数据输入：合并小文件，CombineTextInputFormat输入

Map：减少Spill次数（调整环形内存大小和触发spill的阈值），减少Merge次数（增大一次Merge的文件数），进行Combiner处理；

Reduce：合理设置Map，Reduce任务数，设置Map，Reduce共存，规避使用reduce，合理设置Reduce端Buffer(保留一定比例的内存直接给Reduce使用)；

IO：数据压缩（Snappy，LZO），使用SequenceFile二进制文件；

数据倾斜：抽样范围分区，自定义分区，Combine，采用MapJoin(避免Reducejoin)。

调优参数：设置Map和Reduce可用的内存上限，CPU核数，Container占用的CPU、内存资源

HDFS小文件优化方法

HDFS小文件弊端：HDFS上每个文件都要在NameNode上建立一个索引，这个索引的大小约为150byte，这样当小文件比较多的时候，就会产生很多的索引文件，一方面会大量占用NameNode的内存空间，另一方面就是索引文件过大使得索引速度变慢。

HDFS小文件解决方案

（1）在数据采集的时候，就将小文件或小批数据合成大文件再上传HDFS。（归档）

（2）在业务处理之前，在HDFS上使用MR对小文件进行合并。（SequenceFile）

（3）在MapReduce处理时，可采用CombineTextInputFormat提高效率。

（4）开启JVM重用