## logstash 概述

logstash是一款轻量级的日志搜索处理框架，可以方便的把分散的，多样化的日志搜集起来，并进行自定义的处理，然后输出到指定的位置，比如我们项目中使用到的kafka消息队列

## logstash 工作原理

- logstash使用**管道方式**进行日志的搜集处理和输出，有点类似linux的命令(比如linux命令 ps-ef|grep 'logstash',就是ps -ef执行完了，输出结果传给grep 'logstash')
- 在通常的logstash任务中，包含三个阶段：输入input-->过滤处理filter-->输出output
- 流程图如下：
  ![img](https://images.xiaozhuanlan.com/photo/2018/4ca3abfe6c9c9cf476878305756fd998.png)
- 每个阶段都由很多的插件配合工作，比如file输入插件是logstash从文件读入数据，比如kafka输出插件，是输出数据到kafka等

**优势**
Logstash 主要的有点就是它的灵活性，这还主要因为它有很多插件。然后它清楚的文档已经直白的配置格式让它可以再多种场景下应用。这样的良性循环让我们可以在网上找到很多资源，几乎可以处理任何问题。

**劣势**

Logstash 致命的问题是它的性能以及资源消耗（默认的堆大小是 1GB）。尽管它的性能在近几年已经有很大提升，与它的替代者们相比还是要慢很多的。这里有 [Logstash 与 rsyslog 性能对比](https://link.jianshu.com/?t=https://sematext.com/blog/2015/05/18/tuning-elasticsearch-indexing-pipeline-for-logs/)以及[Logstash 与 filebeat 的性能对比](https://link.jianshu.com/?t=https://sematext.com/blog/2016/04/25/elasticsearch-ingest-node-vs-logstash-performance/)。它在大数据量的情况下会是个问题。

## Flume

Flume 是Apache旗下使用JRuby来构建，所以依赖Java运行环境。Flume本身最初设计的目的是为了把数据传入HDFS中（并不是为了采集日志而设计，这和Logstash有根本的区别。

优势：

Flume已经可以支持一个Agent中有多个不同类型的channel和sink，我们可以选择把Source的数据复制，分发给不同的目的端口

**Logstash:** 

1.插件式组织方式，易于扩展和控制

2.数据源多样不仅限于日志文件，数据处理操作更丰富，可自定义（过滤，匹配过滤，转变，解析......）

3.可同时监控多个数据源（input插件多样），同时也可将处理过的数据同时有不同多种输出（如stdout到控制台，同时存入elasticsearch）

4.安装简单，使用简单，结构也简单，所有操作全在配置文件设定，运行调用配置文件即可

5.管道式的dataSource——input plugin——filter plugin——output plugin——dataDestination

6.有logstash web界面，可搜索日志

7.有一整套的EKL日志追踪技术栈，可收集处理（logstash），存储管理搜索（elasticsearch），图形显示分析（kibana）

8，做到更好的实时监控（插件设置时间间隔属性，对监控的数据源检查更新） 

**Flume (1.x  flume-ng）**

1.分布式的可靠的可用的系统，高效的从不同数据源收集聚合迁移大量数据到一个集中的数据存储

2.安装部署比较logstash复杂

3.同样以配置文件为中心  提供了JavaAPI

4.是一个完整的基于插件的[架构](http://lib.csdn.net/base/architecture) 有独立开发的第三方插件

5.三层架构：source  channel  sink

Flume使用基于事务的数据传递方式来保证事件传递的可靠性。Source和Sink被封装进一个事务。事件被存放在Channel中直到该事件被处理，Channel中的事件才会被移除。这是Flume提供的点到点的可靠机制。
从多级流来看，前一个agent的sink和后一个agent的source同样有它们的事务来保障数据的可靠性。

6，一个agent可指定多个数据源（同一agent内多个source连接到同一个channel上）？

一个agent可将收集的数据输出到多个目的地（HDFS，JMS,agent.....）span-out