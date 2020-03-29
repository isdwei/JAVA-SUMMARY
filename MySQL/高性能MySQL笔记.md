MySQL复习

```mysql
SELECT DISTINCT <select_list>
FROM <left_table> <JOIN TYPE> 
JOIN <right_table> ON <join_condition>
WHERE <where_condition> 
GROUP BY <group_by_list>
HAVING <having_condition>
ORDER BY <order_by_condition>
LIMIT <limit_number>
```

对应的机器处理顺序

```mysql
FROM <left_table>
ON <join_condition>
<JOIN TYPE> JOIN <right_table>
WHERE <where_condition> 
GROUP BY <group_by_list>
HAVING <having_condition>
SELECT DISTINCT <select_list>
ORDER BY <order_by_condition>
LIMIT <limit_number>
```

七种JOIN要会写

![](C:\Users\weitu\Desktop\图像.jpeg)





## 高性能MySQL笔记

### 1. 架构和历史

#### 1.1 MySQL的安装注意事项

```shell
rpm -qa|grep mysql  #查看是否已经安装
#先安装服务端再安装客户端
#mysql的启停
service mysql start
service mysql stop
#设置开机自启动
chkconfig mysql on
```

MySQL默认字符集为latin1，中文会乱码，配置文件中改为utf8

MySQL默认连接数151，默认端口2206

MySQL的数据文件：

* .frm 存放表结构
* .myd 存放表数据
* .myi 存放表索引

#### 1.2 MySQL的分层架构

分层架构 连接层+服务层+查询执行引擎+存储引擎

* 连接层包含客户端与一些连接服务 : Connectors
* 服务层完成大多数核心服务，如SQL接口，缓存查询，SQL的分析优化，所有跨存储引擎的功能都在这一层实现 : Management Serveices & Utilities, Connection Pool, SQL Interface, Parser, Optimizer, Cache&Buffer
* 引擎层真正负责MySQL中数据的存储读取，服务层通过API与引擎层交互
* 存储层将数据存储在文件系统中

#### 1.3 事务基础

一些术语：并发控制 读写锁 锁粒度  表锁 行锁

##### 事务 ACID（原子性atomicity，一致性consistency，隔离性isolation，持久性durability）

- 原子性：使用 undo log ，从而达到回滚
- 持久性：使用 redo log，从而达到故障后恢复
- 隔离性：使用锁以及MVCC,运用的优化思想有读写分离，读读并行，读写并行
- 一致性：通过回滚，以及恢复，和在并发环境下的隔离做到一致性。

##### 隔离级别与对幻读的理解

* 未提交读 Read Uncommitted，未提交也可见，很少使用，会造成脏读（Dirty Read）

* 提交读Read Committed，读取已提交的内容，会造成不可重复读（Nonrepeatable Read）

* 可重复读Repeatable Read，同一事务多次读取结果一致，MySQL默认，但会造成幻读（Phantom Read）

  > 幻读错误的理解：说幻读是 事务A 执行两次 select 操作得到不同的数据集，即 select 1 得到 10 条记录，select 2 得到 11 条记录。这其实并不是幻读，这是不可重复读的一种，只会在 R-U R-C 级别下出现，而在 MySQL 默认的 RR 隔离级别是不会出现的。 

  > 幻读，并不是说两次读取获取的结果集不同，幻读侧重的方面是某一次的 select 操作得到的结果所表征的数据状态无法支撑后续的业务操作。更为具体一些：select 某记录是否存在，不存在，准备插入此记录，但执行 insert 时发现此记录已存在，无法插入，此时就发生了幻读。 

* 可串行化Serializable，强制事务串行执行，解决了幻读问题，很少用

##### 对于幻读与不可重复读在原理上的解释

Mysql提供了多版本读取能力，也就是说表中一份存储的数据行会有多个版本。这个版本对应的version就是transaction id。当行数据被某个transaction变更时，这个行会有个隐含的hidden column中记录了这个更新者的transaction id。旧的数据在被覆盖的同时，会存储到一个undolog的文件中，这个文件中就是保存着每一行的版本数据链表,沿着这个链表走我们可以走过这一行数据之前的版本变更。

**可重复读:**

当开始一个新的transaction，这个transaction会分配一个新的transactionid，此时在这个transaction内部如果我们执行一条普通的select语句，根据select语句的condition,我们会找到一些满足条件的行记录,先不着急返回:

每个匹配的行记录，我们从hidden column中找到对应的最近一次执行更新操作的操作者transaction id，我们将这个id(也就是版本)和我们当前的transaction id进行比较，如果其大于执行语句所属的transaction的id，那么就需要去undolog文件中去寻找旧的版本，一直找到小于当前transaction id的版本的行记录作为快照数据返回。

也就是说，即使在我们当前这个事务执行过程中，有其它事务执行插入了新的数据能够满足我们的查询条件，但因为这个数据的版本(transaction id)大于我们当前事务的id，说明这个事务应该在当前事务之后发生，我们是查询不到这个数据的。

**幻读:**

幻读的字面意思很简单，就是在事务内两次查询语句(注意是select ...for...update语句,不是前面的普通查询语句\<consistent read>)，读取到了不同的数据。

比如在一个事务中，我们要查询满足condition: a>5 and a<10的记录，目前只有一条记录a=7，此时另一个事务向这个condition中插入了一条a=8的数据，那么在有版本号的情况下，我们用普通的查询是读不到这条数据的，但如果我们想要插入一条a=8的数据，则会报错，这就是幻读。

**解决办法:**

数据库解决这个问题的办法就是，对这个区间都加锁，不仅仅是已有的行记录，空的那些a的值[8,9]都会加上锁，也就是gap lock。这个时候我们当前的事务就达到了对这个区间独占的目的，其它的事务在我们处理过程中就无法插入新的数据了（阻塞）。

MySQL将每个数据库（schema）保存为目录下的一个子目录。创建表时，会创建一个与表同名的*.frm*文件保存表的定义。

#### 1.4 MyISAM与InnoDB

* InnoDB存储引擎 ，间隙锁防止幻读， 元数据.frm ，数据文件 .ibd 所有表都保存在系统表空间文件 

* MyISAM存储引擎，不支持事务和行锁，数据文件*.myd* 索引文件*.myi*

  |        |       MyISAM       |      InnoDB      |
  | :----: | :----------------: | :--------------: |
  | 主外键 |       不支持       |       支持       |
  |  事务  |       不支持       |       支持       |
  | 行表锁 | 表锁，不适合高并发 | 行锁，适合高并发 |
  |  索引  |   不保存真实数据   | 索引、数据均保存 |
  |  MVCC  |       不支持       |       支持       |
  | 关注点 |        性能        |       事务       |
  |  其他  |         Y          |        Y         |

* Percona Server存储引擎

### 2.  MySQL索引优化  

#### 2.1 MySQL性能下降的原因

索引失效，join太多，服务器调优及各个参数设置

#### 2.2创建高性能的索引

> 什么是索引？ 索引是帮助MySQL高效获取数据的数据结构。索引是数据结构！

```mysql
#创建索引
CREATE [UNIQUE] INDEX indexName ON mytable(columname(length));
ALTER mytable ADD [UNIQUE]  INDEX [indexName] ON(columnname(length));
#ALTER 添加索引
ALTER TABLE tb_name ADD PRIMARY KEY (column_list);#添加主键
ALTER TABLE tb_name ADD UNIQUE index_name(column_list);#唯一索引，可为NULL
ALTER TABLE tb_name ADD INDEX index_name(column_list);#普通索引，索引值可出现多次
ALTER TABLE tb_name ADD FULL TEXT index_name(column_list);#全文索引
#删除索引
DROP INDEX [indexName] ON mytable;
#查看索引
SHOW INDEX FROM table_name\G;
```

#### 2.3 索引的类型

索引分类

* 单值索引  一个索引只包含单个列，如主键 
* 唯一索引 索引列的值必须唯一，允许空值
* 主键索引 一种特殊的唯一索引，不允许有空值
* 复合索引 多个字段联合起来建立索引

索引原理

* B-Tree /B+Tree   MyISAM  InnoDB
* 哈希索引              Memory
* 空间数据索引R-Tree      MyISAM
* 全文索引

**B+Tree索引的特点，与有序数组，Hash，AVL-Tree，B-Tree的优势**

1.  B+树中，非叶节点不保存数据相关信息，只保存关键字和子节点的引用。 所有记录节点存放在叶子节点上，且是顺序存放，由各叶子节点指针进行连接。如果从最左边的叶子节点开始顺序遍历，能得到所有键值的顺序排序。 

   * B+树的高度一般为2-4层，叶子节点存储的数据通常是一页（默认16k）或一页的整倍数（MySQL每次读取都是读一页）。

   * 范围查找时，能通过叶子节点的指针获取数据。例如查找大于等于3的数据，当在叶子节点中查到3时，通过3的尾指针便能获取所有数据，而不需要再像二叉树一样再获取到3的父节点。

2. 有序数组则是节点存储数据无限大的树，数据量大时太耗内存，而树可以一个节点一个节点加载。

3. Hash索引底层是哈希表，哈希表是一种以key-value存储数据的结构，所以多个数据在存储关系上是完全没有任何顺序关系的，所以：

   * 哈希索引适合等值查询，但是无法进行范围查询 

   * 哈希索引没办法利用索引完成排序 

   * 哈希索引不支持多列联合索引的最左匹配规则 

   * 如果有大量重复键值的情况下，哈希索引的效率会很低，因为存在哈希碰撞问题 

4.  相对于AVL树，B+Tree一个节点可以存储多个元素，相对于完全平衡二叉树所以整棵树的高度就降低了，磁盘IO效率提高了

5.   B树不管叶子节点还是非叶子节点，都会保存数据，并且B树由于叶子节点之间没有链表连接，范围查询时会有局部中序遍历、跨层遍历

> 表中的数据删除后，索引上对应的索引值是不会删除的，特别是在一性次删除大批量数据后，会造成大量的dead leaf挂到索引树上。

#### 2.4  聚簇索引和非聚簇索引/主键索引和辅助索引/覆盖索引

1.   聚簇索引，就是指主索引文件和数据文件为同一份文件，聚簇索引在InnoDB存储引擎中存在。 
2. MYISAM中B+Tree索引叶子节点的数据区域存储的是数据记录的地址， MyISAM存储引擎在使用索引查询数据时，会先根据索引查找到数据地址，再根据地址查询到具体的数据。并且主键索引和辅助索引没有太多区别。 
3. 在 InnoDB 里，主键索引B+ Tree的叶子节点存储了整行数据， 辅助索引存储的是主键值（都不是地址）。InnoDB 的一个表一定要有主键索引，如果一个表没有手动建立主键索引，InnoDB会查看有没有唯一索引，如果有则选用唯一索引作为主键索引，如果连唯一索引也没有，则会默认建立一个隐藏的主键索引（用户不可见）。另外，InnoDB 的主键索引要比MyISAM的主键索引查询效率要高（少一次磁盘IO），并且比辅助索引也要高很多。所以，我们在使用InnoDB 作为存储引擎时，我们最好：
   1. 手动建立主键索引
   2. 尽量利用主键索引查询

4. 覆盖索引,即索引包含有所需要的数据

#### 2.5 高性能索引策略

> 哪些情况需要建立索引？

* 主键自动建立唯一索引
* 频繁作为查询条件的字段应建立索引
* 查询中与其他表关联的字段、外键应建立索引
* 高并发下倾向创建组合索引
* 查询中排序、统计、分组的字段

> 哪些情况不应建立索引？

* 记录很少的表
* 频繁更新的字段不应建立索引
* Where中用不到的字段不应建立索引
* 数据重复且分布平均的表字段

#### 2.6 SQL性能分析——Explain指令

explain关键字可以模拟优化器执行SQL语句，分析性能瓶颈。

```mysql
explain select id, name from table stu_info where ....
```

![1584083684169](C:\Users\weitu\AppData\Roaming\Typora\typora-user-images\1584083684169.png)

**解释**

1. id select查询的序列号，复合查询中:

   * id相同的语句，执行顺序从上到下

   * id不同，如果是子查询，id序号会递增，id越大，优先级越高，越早执行

     ![1584083859383](C:\Users\weitu\AppData\Roaming\Typora\typora-user-images\1584083859383.png)

2. select_type 查询的类型，包括：

   * SIMPLE 简单的select查询 ，奴包含子查询或UNION
   * PRIMARY 查询中包含子查询，则外层被标记为PRIMARY
   * SUBQUERY 在select或where中的子查询
   * DERIVED 在from中的子查询（衍生）
   * UNION 子查询在union后
   * UNION　RESULT　从union表获取结果的select

3. table 表示这一行的数据是关于哪张表的

4. type 表示查询使用了何种类型,从好到差依次是: system>const>eq_ref>ref>range>index>ALL

   * system: 表中只有一行记录,相当于系统表
   * const: 通过索引一次就找到了(如将主键置于where中)
   * eq_ref: 唯一性索引,每个索引键表中只有一行数据与之匹配
   * ref:非唯一扫描,返回匹配某个单独值的所有行
   * range: 只检索给定范围的行,使用一个索引来选择行(即where中对索引使用了between,<,>,in等)
   * index: 遍历索引树
   * all: 遍历全表

5. possible_keys 显示可能应用的这张表中的索引

6. key 实际查询使用到的索引

7. key_len 索引最大可能使用的字节数,越短越好

8. ref 显示索引的哪一列被使用了,哪些列或常量被用于查找索引列上的值

9. rows 大致估计找到所需记录应读取的行数

10. Extra:

    1) Using filesort   使用文件排序,未使用索引,性能低

    2)Using temporary 使用了临时表保存中间结果,性能极低

    3)Using index 使用了覆盖索引,性能高

    4)Using where 使用了where过滤

![1584086280453](C:\Users\weitu\AppData\Roaming\Typora\typora-user-images\1584086280453.png)

#### 2.7 索引优化

1. 索引失效
   * 最佳左前缀法则
   * 不在索引列上做任何操作(计算、函数、自动或手动类型转换)，会导致索引失效转向全表扫描(如字符串不加单引号导致自动转为INT)
   * 存储引擎不能使用索引中范围条件右边的列
   * 尽量使用覆盖索引,避免select *
   * MySQL在使用 != 或 <> 时无法使用索引会导致全表扫描
   * is null , is not null也无法使用索引
   * like以通配符开头会导致索引失效,全表扫描(解决办法: 主键索引,覆盖索引)
   * or 连接时会索引失效
2. 一般性建议
   * 对于单键索引，尽量选择针对当前query过滤性更好的索引
   * 在选择组合索引的时候，当前Query中过滤性最好的字段在索引字段顺序中，位置越靠前越好。
   * 在选择组合索引的时候，尽量选择可以能包含当前query中的where子句中更多字段的索引
   * 尽可能通过分析统计信息和调整query的写法来达到选择合适索引的目的

>   [SELECT *] 和[SELECT 全部字段]的 2 种写法有何优缺点?  

1. 前者要解析数据字典，后者不需要
2. 结果输出顺序，前者与建表列顺序相同，后者按指定字段顺序。
3. 表字段改名，前者不需要修改，后者需要改
4. 后者可以建立索引进行优化，前者无法优化
5. 后者的可读性比前者要高  

### 3. 查询截取分析

#### 3.1 查询优化的4种方法

1. 慢查询的开启并捕获
2. explain+慢查询语句 分析
3. show profiles查询 SQL 在MySQL服务器里的执行细节和生命周期情况
4. SQL数据库的参数调优

#### 3.2 慢查询日志

* 对当前会话有效: set global slow_query_log = 1

* 永久生效: 修改 my.cnf 文件

> 开启慢查询日志会影响性能
>
> 默认 long_query_time = 10, 即大于10秒会被记录

#### 3.3 show profiles

MySQL提供用来分析当前会话中语句执行的资源消耗情况，默认关闭，会保存最近15次运行结果

* 分析步骤

  指令：show profiles;

  ​			show profile cpu,block io ... for query 2; 

  需要注意的几个结果

  converting HEAP to MyISAM  查询结果太大，内存不够用，往磁盘上存

  Creating tmp table 创建临时表

  Creating to tmp data on disk  把内存中的临时表复制到磁盘，危险！！

  locked

* 全局查询日志

  general_log = 1

### 4. 主从复制

* 复制的基本原理

1. master 将改变记录到二进制日志（binary log）
2. slave 将 master 的二进制日志拷贝到他的中继日志（relay log）
3. slave 重做中继日志中的时间，将改变应用到自己的数据库中。MySQL的主从复制是异步且串行化的（最大问题：存在延时）

* 主从复制的基本原则

  每个slave只能有一个master，每个master可以有多个slave

  mysql 支持的复制类型?

1. 基于语句的复制： 在主服务器上执行的 SQL 语句，在从服务器上执行同样的语句。 MySQL 默认采用基于语句的复制，效率比较高。 一旦发现没法精确复制时，会自动选着基于行的复制。
2. 基于行的复制：把改变的内容复制过去，而不是把命令在从服务器上执行一遍. 从 mysql5.0 开始支持
3. 混合类型的复制: 默认采用基于语句的复制，一旦发现基于语句的无法精确的复制时，就会采用基于行的复制。  

### 5. MySQL中的锁

#### 5.1 从对数据操作类型分

**共享锁/读锁**

```sql
SELECT * FROM `test` WHERE `id` = 1 LOCK IN SHARE MODE;
```

1. 多个事务的查询语句可以共用一把共享锁；
2. 如果只有一个事务拿到了共享锁，则该事务可以对数据进行 UPDATE DETELE 等操作；
3. 如果有多个事务拿到了共享锁，则所有事务都不能对数据进行 UPDATE DETELE 等操作。

**排他锁/写锁**

```sql
SELECT * FROM `test` WHERE `id` = 1 FOR UPDATE;
```

1. 只有一个事务能获取该数据的排它锁；
2. 一旦有一个事务获取了该数据的排它锁之后，其余事务对于该数据的操作将会被阻塞，直至锁释放。

自增锁

 如果表中存在自增字段，MySQL便会自动维护一个自增锁 

> 为什么主键设置成自增

 主键上设置自增属性，可以保证每次插入都是插入到最后面，可以有效的减少索引页的分裂和数据的移动。 

自增锁会导致出现幻读的情况（某个事务中途插入后会导致在本事务中主键不连续）

#### 5.2从操作粒度分

**行锁/记录锁（Record Locks）**   SELECT * FROM `test` WHERE `id`=1 FOR UPDATE;  锁住一行记录

*   InnoDB 行锁是通过给索引上的索引项加锁来实现的 ，这意味着只有通过索引条件检索数据，InnoDB 才使用行锁 。即**操作未使用索引，行锁会升级为表锁**
* **间隙锁（Gap Lock）**
  1. 间隙锁只有在事务隔离级别 RR 中才会产生；
  2. 唯一索引只有锁住多条记录或者一条不存在的记录的时候，才会产生间隙锁，指定给某条存在的记录加锁的时候，只会加记录锁，不会产生间隙锁；
  3. 普通索引不管是锁住单条，还是多条记录，都会产生间隙锁；
  4. 间隙锁会封锁该条记录相邻两个键之间的空白区域，防止其它事务在这个区域内插入、修改、删除数据，这是为了防止出现幻读现象；

**表锁** 偏向MyISAM存储引擎，开销小，加锁快，无死锁，锁定粒度大，发生锁冲突的概率最高，并发最低

#### 5.3 数据库是如何加锁的

大体的举个例子如下：

一个sql ：select * from user where id=1;

数据库收到sql后会，判断id是不是索引：如果是索引，数据库则就行进行检索索引，对这条索引进行加锁，加锁加的是行锁；如果Id不是索引，则加的是表锁。

如果加的是行锁，会判断根据sql，添加不同的类型的行锁：如果是查询语句，所加的共享锁（S锁，读锁）；如果是增删改语句，则加的是排他锁。

如果加的表锁，则会根据加的是表锁，则会判断表是否存在其他的锁。如果存在锁且是S锁，则可以进行加表锁。如果加的是X锁，则无法加表锁。

在加表锁的时候，是如何判断有锁的？重新检索索引，来判断是加锁？那是不可能的，因为那样效率会低的令人发指，尤其是表级锁，需要检索完所有的数据才能知道是表级别锁，效率可想而知。这个时候就有了意向锁。

意向锁，只是一个意向，他的意思是：这个表，里面的数据或索引已经加锁了。其实他就是一个标志，来告诉数据库已经加锁的标志。**意向锁的存在说明该结点的下层结点正在被加锁**。

**对任一元组加锁时，必须先对它所在的关系加意向锁**。所以正确的是，加共享锁（S锁，读锁）时候，会在之前加上意向共享锁（IS锁）。告诉下一个要操作这张表的事物，这张表的数据加了共享锁。

 通过锁实现的是，读的时候不能写（允许多个线程同时读，即共享锁，S锁），写的时候不能读（一次最多只能有一个线程对同一份数据进行写操作，即排它锁，X锁）。这样的加锁访问，只能实现并发的读，因为它最终实现的是读写串行化，这样就大大降低了数据库的读写性能。加锁访问其实就是和MVCC相对的LBCC，即基于锁的并发控制（Lock-Based Concurrent Control），是四种隔离级别中级别最高的Serialize隔离级别。

### 6. MVCC( Multi-Version Concurrency Control 多版本并发控制 )

MVCC 是一种并发控制的方法，一般在数据库管理系统中，实现对数据库的并发访问；在编程语言中实现事务内存 。通俗的讲就是MVCC通过保存数据的历史版本，根据比较版本号来处理数据的是否显示，从而达到读取数据的时候不需要加锁就可以保证事务隔离性的效果。

- MVCC每次更新操作都会复制一条新的记录，新纪录的创建时间为当前事务id
- 优势为读不加锁，读写不冲突
- InnoDb存储引擎中，每行数据包含了一些隐藏字段 DATA_TRX_ID，DATA_ROLL_PTR，DB_ROW_ID，DELETE BIT
- DATA_TRX_ID 字段记录了数据的创建和删除时间，这个时间指的是对数据进行操作的事务的id
- DATA_ROLL_PTR 指向当前数据的undo log记录，回滚数据就是通过这个指针
- DELETE BIT位用于标识该记录是否被删除，这里的不是真正的删除数据，而是标志出来的删除。真正意义的删除是在mysql进行数据的GC，清理历史版本数据的时候。

具体的DML：

- INSERT：创建一条新数据，DB_TRX_ID中的创建时间为当前事务id，DB_ROLL_PT为NULL
- DELETE：将当前行的DB_TRX_ID中的删除时间设置为当前事务id，DELETE BIT设置为1
- UPDATE：复制了一行，新行的DB_TRX_ID中的创建时间为当前事务id，删除时间为空，DB_ROLL_PT指向了上一个版本的记录，事务提交后DB_ROLL_PT置为NULL

为了提高并发度，InnoDb提供了这个「非锁定读」，即不需要等待访问行上的锁释放，又解决了幻读。

#### 6.1MVCC与隔离级别 

- Read Uncommitted每次都读取记录的最新版本，会出现脏读，未实现MVCC
- Serializable对所有读操作都加锁，读写发生冲突，不会使用MVCC
- SELECT 
  - (RR级别)InnoDB检查每行数据，确保它们符合两个标准：
  - 只查找创建时间早于当前事务id的记录，这确保当前事务读取的行都是事务之前已经存在的，或者是由当前事务创建或修改的行
  - 行的DELETE BIT为1时，查找删除时间晚于当前事务id的记录，确定了当前事务开始之前，行没有被删除
  - (RC级别)每次重新计算read view，read view的范围为InnoDb中最大的事务id，为避免脏读读取的是DB_ROLL_PT指向的记录

简单的SELECT不加条件的查询在RR下肯定是读不到隔壁事务提交的数据的。但是仍然可能在执行INSERT/UPDATE时遇到幻读现象。因为SELECT 不加锁的快照读行为是无法限制其他事务对新增重合范围的数据的插入的。

所以还要引入第二个机制。

#### 6.2 Next-Key Lock 

其实更多的幻读现象是通过写操作来发现的，如SELECT了3条数据，UPDATE的时候可能返回了4个成功结果，或者INSERT某条不在的数据时忽然报错说唯一索引冲突等。

首先来了解一下InnoDB的锁机制，InnoDB有三种行锁：

- Record Lock：单个行记录上的锁
- Gap Lock：间隙锁，锁定一个范围，但不包括记录本身。GAP锁的目的，是为了防止同一事务的两次当前读，出现幻读的情况
- Next-Key Lock：前两个锁的加和，锁定一个范围，并且锁定记录本身。对于行的查询，都是采用该方法，主要目的是解决幻读的问题

如果是带排他锁操作（除了INSERT/UPDATE/DELETE这种，还包括SELECT FOR UPDATE/LOCK IN SHARE MODE等），它们默认都在操作的记录上加了Next-Key Lock。只有使用了这里的操作后才会在相应的记录周围和记录本身加锁，即Record Lock + Gap Lock，所以会导致有冲突操作的事务阻塞进而超时失败。

隔离级别越高并发度越差，性能越差，虽然MySQL默认的是RR，但是如果业务不需要严格的没有幻读现象，是可以降低为RC的或修改配置innodb_locks_unsafe_for_binlog为1 来避免Gap Lock的。 注意有的时候MySQL会自动对Next-Key Lock进行优化，退化为只加Record Lock，不加Gap Lock，如相关条件字段为主键时直接加Record Lock。

### 7.面试题整理

#### 查找每个部门工资最高的员工

> Employee 表包含所有员工信息，每个员工有其对应的 Id, salary 和 department Id。
>
> +----+-------+--------+--------------+
> | Id | Name  | Salary | DepartmentId |
> +----+-------+--------+--------------+
> | 1  | Joe   | 70000  | 1            |
> | 2  | Henry | 80000  | 2            |
> | 3  | Sam   | 60000  | 2            |
> | 4  | Max   | 90000  | 1            |
> +----+-------+--------+--------------+
> Department 表包含公司所有部门的信息。
> +----+----------+
> | Id | Name     |
> +----+----------+
> | 1  | IT       |
> | 2  | Sales    |
> +----+----------+
> 编写一个 SQL 查询，找出每个部门工资最高的员工。例如，根据上述给定的表格，Max 在 IT 部门有最高工资，Henry 在 Sales 部门有最高工资。
>
> +------------+----------+--------+
> | Department | Employee | Salary |
> +------------+----------+--------+
> | IT         | Max      | 90000  |
> | Sales      | Henry    | 80000  |
> +------------+----------+--------+
>

```sql

解法一：
SELECT 
Department.Name as Department, 
Employee.Name as Employee, 
MAX(Employee.Salary) as Salary 
FROM Employee 
LEFT JOIN 
Department 
ON 
Employee.DepartmentId = Department.Id 
GROUP BY Employee.DepartmentId;
 
输入：
{"headers": {"Employee": ["Id", "Name", "Salary", "DepartmentId"], "Department": ["Id", "Name"]}, "rows": {"Employee": [[1, "Joe", 70000, 1], [2, "Jim", 90000, 1], [3, "Henry", 80000, 2], [4, "Sam", 60000, 2], [5, "Max", 90000, 1]], "Department": [[1, "IT"], [2, "Sales"]]}}
输出：
{"headers":["Department","Employee","Salary"],"values":[["IT","Joe",90000],["Sales","Henry",80000]]}
预期：
{"headers":["Department","Employee","Salary"],"values":[["IT","Jim",90000],["Sales","Henry",80000],["IT","Max",90000]]}
 
没有达到预期结果，原因group by 进行了去重，如果有同部门薪水想同多个员工得不到预期结果。
 
解法二：
select 
d.Name as Department,
e.Name as Employee ,
e.Salary 
from Employee e 
join 
Department d 
on e.DepartmentId = d.Id 
where (e.DepartmentId ,e.Salary) 
in 
	(select DepartmentId ,
 	max(Salary) as Salary  
 	from Employee 
 	group by DepartmentId);
 
正确。
先根据部门DepartmentId group by 出部门id与最高的薪水值作为筛选条件。
select DepartmentId ,max(Salary) as Salary  from Employee group by DepartmentId
 
在两表left join 根据得出的条件筛选满足DepartmentId 与 Salary  最大值的员工。
```

####   分布式数据库如何保证数据可靠性  

在传统数据库中，有几种常用的手段来保证数据可靠性：
1）Redo Log
2）主从热备
3）备份/恢复
4）存储层数据校验 

5） 分布式一致性协议 