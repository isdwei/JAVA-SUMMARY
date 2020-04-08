# Hadoop学习笔记

## 安装与配置

Linux环境—>ssh免密码登录—>安装jdk—>安装hadoop,配置文件—>hdfs namenode –format

**Hadoop中的配置文件**

Core-site.xml，Hadoop-env.sh，Hdfs-site.xml，Mapred-site.xml

**Hadoop中启动的进程**

NameNode, SecondaryNameNode, DataNode, ResourceManage, NodeManage

**默认端口号**

50070，8088

**datanode和namenode进程同时只能有一个工作问题分析**

第一次启动没有问题，第二次启动时，原来的datanode数据并没有删掉，他在与新namenode通信时连接不上，导致集群不能正常启动。

解决办法：在格式化之前，删除datanode中的信息



## 一、Hadoop Distributed FileSystem（HDFS）

### 1. HDFS的设计

HDFS时为以**流式数据访问模式**存储**超大文件**而设计的文件系统，在商用硬件集群上运行。

#### 1.1 文件块（Block）

HDFS以块为单位保存文件，在Hadoop2.x版本中块的大小默认为128M（在hadoop1.x中64M，通过dfs.blocksize规定）。一个小于块大小的文件不会占据整个块空间。

##### 1.1.1 如何确定HDFS中块的大小？

**HDFS中块比磁盘大，目的是减少寻址开销**，从而传输一个由多个块组成的文件的时间就取决于磁盘的传输速率。
**如果块设计的太大，传输数据时间会增大；如果块设计的太小，会增加寻址时间。**

**块大小的设计原则：寻址时间为传输时间的1%**

目前磁盘的传输速率普遍为100MB/s，若希望寻址时间在10ms以内，那么传输时间为1s，Block大小为100MB，取2的整数次幂得到128MB。

##### 1.1.2 采用块的好处

* 减少了寻址时间，提高处理大文件的效率
* 一个文件可以大于网络中任意一个磁盘的容量，不需要存储在同一块磁盘上
* 简化存储管理（块的大小固定），元数据可以不与块一同储存。
* 便于提高系统的容错和实用性，方便复制

#### 1.2 NameNode，Secondary NameNode & DataNode

HDFS集群有两种节点，以Master-Slave模式运行。

##### NameNode&Secondary NameNode

**NameNode作用：管理整个文件系统的命名空间；配置副本策略；管理Block的映射信息；处理客户端读写请求**
**SeconaryNameNode作用：复制NameNode，定期合并Fsimage，Edits；辅助恢复NameNode**

> NameNode中并不保留Block的位置信息，而是DataNode启动后定期汇报。

**工作机制**：

* 第一阶段：NameNode启动
  * 第一次启动NameNode格式化后，创建Fsimage和Edits文件。如果不是第一次启动，直接**加载编辑日志和镜像文件到内存**。
  * 客户端对元数据进行增删改的请求：**NameNode先记录操作日志，更新滚动日志再内存中对数据进行增删改。**
* 第二阶段：Secondary NameNode工作
  * Secondary NameNode询问NameNode是否需要CheckPoint，返回NameNode是否检查结果。（ **触发CheckPoint需要满足两个条件中的任意一个：1小时时间到和每隔一分种检查Edits中操作次数是否达到100万**）  
  * 若需要CheckPoint则请求执行：
    * NameNode**滚动正在写的Edits日志，生成一个空的edits.inprogress**，将滚动前的编辑日志和镜像文件拷贝到Secondary NameNode；
    * Secondary NameNode**加载编辑日志和镜像文件到内存，并合并，生成新的镜像文件**fsimage.chkpoint。
    * 拷贝fsimage.chkpoint到NameNode，NameNode将fsimage.chkpoint重新命名成fsimage。

Secondary NameNode恢复NameNode的方法：

* 直接拷贝
* 使用`hdfs namenode -importCheckpoint`启动NameNode守护进程，从而将SecondaryNameNode中数据拷贝到NameNode目录中  

**集群安全模式：**

* NameNode启动时，首先将镜像文件（Fsimage）载入内存，并执行编辑日志（Edits）中的各项操作。一旦在内存中成功建立文件系统元数据的映像，则创建一个新的Fsimage文件和一个空的编辑日志。此时，NameNode开始监听DataNode请求。**这个过程期间，NameNode一直运行在安全模式，即NameNode的文件系统对于客户端来说是只读的。**
* 系统中的**数据块的位置并不是由NameNode维护的，而是以块列表的形式存储在DataNode中**。在系统的正常操作期间，NameNode会在内存中保留所有块位置的映射信息。在安全模式下，各个DataNode会向NameNode发送最新的块列表信息，NameNode了解到足够多的块位置信息之后，即可高效运行文件系统。
* 如果**满足“最小副本条件”**，**NameNode会在30秒钟之后就退出安全模式**。所谓的最小副本条件指的是在整个文件系统中99.9%的块满足最小副本级别（默认值：dfs.replication.min=1）。在**启动一个刚刚格式化的HDFS集群时，因为系统中还没有任何块，所以NameNode不会进入安全模式**。

##### DataNode

**作用：储存实际数据块；执行读写操作**

* 一个数据块在DataNode上以文件形式存储在磁盘上，包括两个文件，**一个是数据本身，一个是元数据包括数据块的长度，块数据的校验和，以及时间戳。**DataNode会**周期性检验文件的校验和是否与创建时相同以保证文件的正确性；**
* **DataNode启动后向NameNode注册**，通过后，**周期性（1小时）的向NameNode上报所有的块信息**。
* **心跳是每3秒一次**，**心跳返回结果带有NameNode给该DataNode的命令**如复制块数据到另一台机器，或删除某个数据块。**如果超过10分钟（默认超时时长10分钟+30秒）没有收到某个DataNode的心跳，则认为该节点不可用**。
* 集群运行中可以安全加入和退出一些机器。



### 2. HDFS数据流

#### 2.1 文件读取流程剖析（重要）

![1586269880115](picture\read.jpg)

```java
public static void getFileFromHDFS() throws IOException, InterruptedException, URISyntaxException {
        // 1 获取文件系统
        Configuration configuration = new Configuration();
        FileSystem fs = FileSystem.get(new URI("hdfs://hadoop001:9000"), configuration, "root");
        // 2 获取输入流
        FSDataInputStream fis = fs.open(new Path("/in.txt"));
        // 3 获取输出流
        FileOutputStream fos = new FileOutputStream(new File("in.txt"));
        // 4 流的对拷
        IOUtils.copyBytes(fis, fos, configuration);
        // 5 关闭资源
        IOUtils.closeStream(fos);
        IOUtils.closeStream(fis);
        fs.close();
    }
```

1. **客户端实例化一个FileSystem对象**（其实是DistributedFileSystem实例，DistributedFileSystem继承于FileSystem），调用**DistributedFileSystem.open()方法通过RPC向NameNode请求下载文件**，**NameNode通过查询元数据**，找到**文件开头部分块（第一个块）所有副本所在的DataNode地址**并传给客户端，并且**每个块的副本都基于距离远近排序**。

   ```java
    @Override
     public FSDataInputStream open(Path f, final int bufferSize)
         throws IOException {
       statistics.incrementReadOps(1);
       Path absF = fixRelativePart(f);
       return new FileSystemLinkResolver<FSDataInputStream>() {
         @Override
         public FSDataInputStream doCall(final Path p)
             throws IOException, UnresolvedLinkException {
           final DFSInputStream dfsis =
             dfs.open(getPathName(p), bufferSize, verifyChecksum);
           return dfs.createWrappedInputStream(dfsis);
         }
         @Override
         public FSDataInputStream next(final FileSystem fs, final Path p)
             throws IOException {
           return fs.open(p, bufferSize);
         }
       }.resolve(this, absF);
     }
   ```

2. **open()方法返回的是一个FSDataInputStream对象（支持文件定位的输入流）给客户端读取数据**。这个FSDataInputStream其实是一个被包装的DFSInputStream对象。

3. 客户端调用这个**输入流的read()方法**。文件开头部分的块的数据节点地址的DFSInputStream随即与这些块最近的数据节点相连接，**数据从数据节点返回客户端**（以Packet为单位来做校验，先在本地缓存，默认buffer为4096位，然后写入目标文件）。当到达块的末端时，DFSInputStream会关闭与数据节点之间的联系，然后为下一个块找到最佳的数据节点。客户端只需要读取一个连续的流，这些对于客户端都是透明的。

   ```java
   /**
      * Copies from one stream to another.
      * 
      * @param in InputStrem to read from
      * @param out OutputStream to write to
      * @param buffSize the size of the buffer 
      */
     public static void copyBytes(InputStream in, OutputStream out, int buffSize) 
       throws IOException {
       PrintStream ps = out instanceof PrintStream ? (PrintStream)out : null;
       byte buf[] = new byte[buffSize];
       int bytesRead = in.read(buf);
       while (bytesRead >= 0) {
         out.write(buf, 0, bytesRead);
         if ((ps != null) && ps.checkError()) {
           throw new IOException("Unable to write to output stream.");
         }
         bytesRead = in.read(buf);
       }
     }
   ```

4. 客户端从流中读取数据时，块是按DFSInputStream打开与数据节点的新连接的顺序读取的。它也会调用NameNode检索下一组需要的块的位置。一旦完成读取，就会对FSDataInputStream调用close()。

#### 2.2 文件写入流程剖析（重要）

![1586269992823](picture\write.jpg)

```java
public void putFileToHDFS() throws IOException, InterruptedException, URISyntaxException {
	// 1 获取文件系统
	Configuration configuration = new Configuration();
	FileSystem fs = FileSystem.get(new URI("hdfs://hadoop001:9000"), configuration, "root");
	// 2 创建输入流
	FileInputStream fis = new FileInputStream(new File("in.txt"));
	// 3 获取输出流
	FSDataOutputStream fos = fs.create(new Path("/in.txt"));
	// 4 流对拷
	IOUtils.copyBytes(fis, fos, configuration);
	// 5 关闭资源
	IOUtils.closeStream(fos);
	IOUtils.closeStream(fis);
    fs.close();
}

```

1. 客户端通过**DistributedFileSystem对象调用create()向NameNode请求上传文件**，**NameNode检查目标文件**是否已存在，父目录是否存在，以及客户端是否有权限创建文件，如果检查通过NameNode就会生成一个新的文件记录**“file.copying”**，并**返回一个文件系统输出流**，否则会向客户端返回一个IOException异常。

2. 客户端将所要上传的文件转换为输出流FSDataOutputStream；

3. 当客户端开始写入文件的时候，**客户端会将文件切分成多个 packets（ 默认64kB ）**，并在**内部以数据队列“data queue（数据队列）”的形式管理这些 packets**，并**向 namenode 申请 blocks**，获取用来存储 replicas 的合适的 datanode 列表，**列表的大小根据 namenode 中 replication 的设定而定，并按距离远近排序**； 

4. 开始**以 pipeline（管道）的形式将 packet 写入所有的 replicas 中**。客户端把 packet 以流的方式写入第一个 datanode，该 datanode 把该 packet 存储之后，再将其传递给在此 pipeline 中的下一个 datanode，直到最后一个 datanode，这种**写数据的方式呈流水线的形式**；

   >  客户端会根据返回的三个节点和第一个节点建立一个socket连接（只会和第一个节点建立），第一个节点又会和第二个节点建立socket连接，由第二个节点又会和第三个节点建立一个socket连接，这种连接的方式叫Pipeline 

5. DataNode完成接收block块后，block的metadata（MD5校验码）通过一个心跳将信息汇报给NameNode 

6. **最后一个 datanode 成功存储之后会返回一个ack packet（确认包）**，在 pipeline 里传递至客户端，在客户端的开发库内部维护着**"ack queue"**，**成功收到 datanode 返回的ack packet 后会从"data queue"移除相应的 packet；**

7. 如果传输过程中，有**某个 datanode 出现了故障，那么当前的 pipeline 会被关闭**，**出现故障的 datanode 会从当前的 pipeline 中移除，剩余的 block 会继续剩下的 datanode 中继续 以 pipeline 的形式传输**，同时 **namenode通过心跳检测机制会发现副本数不足，并分配一个新的datanode，保持replicas设定的数量**；

8. 客户端完成数据的写入后，会对数据流调用 close()方法，关闭数据流。并且NameNode若收到所有DataNode的汇报，NameNode会将元数据后的.copying去掉成为正式文件。

#### 2.3 副本节点的选择——机架感知

第一个副本选择在Client所在节点上，或者随机选一个；
第二个副本选择在第一个副本同机架的任一节点；
第三个副本选择在同集群另一个机架中的任意节点。

#### 2.4 数据完整性

CRC-32（cyclic redundancy check，循环冗余检查）

HDFS会验证数据校验和，针对每512字节创建一个校验和（默认值，通过`io.bytes.per,checksum`改变，CRC检验和的大小为4个字节，存储开销小于1%）。

#### 2.5 HDFS中的数据压缩算法

* Gzip 不支持Spill  压缩率较高，自带

* Bzip2 支持Spill   压缩率很高，自带    压缩解压慢

* Lzo  支持Spill   压缩率合理，不自带   压缩解压快 最流行

* Snappy 不支持Spill 压缩率比Gzip低，不自带   高速压缩  

#### 2.6 Hadoop 归档

HDFS中的文件以块为单位存储，块的元数据存储在NameNode中。 **HDFS中每个文件、目录、数据块占用150Bytes的空间**，因此大量的小文件会耗尽NameNode的大部分内存。 

HadoopArchives能够将多个小文件打包成一个HAR文件，这样在减少namenode内存使用的同时，仍然允许用户使用一个以har://开头的URL就可以访问HAR文件中的小文件。**使用HAR files可以减少HDFS中的文件数量**。

但访问一个指定的小文件需要访问两层索引文件才能获取小文件在HAR文件中的存储位置，因此，访问一个HAR文件的效率可能会比直接访问HDFS文件要低。**对于一个mapreduce任务来说，如果使用HAR文件作为其输入，仍旧是其中每个小文件对应一个map task，效率低下。所以，HAR files最好是用于文件归档。**

<img src="picture\har.png" alt="1586350902340" style="zoom: 50%;" />

————————————————
参考：CSDN博主「寒江一叶舟」的[原创文章](https://blog.csdn.net/zyd94857/article/details/79946773)，遵循 CC 4.0 BY-SA 版权协议

## 二、MapReduce

### 1. Hadoop序列化与反序列化

Hadoop中节点之间的进程通信是RPC来实现的，RPC协议使用序列化将消息编码为二进制流。

#### 1.1 Writable接口

对于一个在MapReduce程序中流动的对象，必须满足：

* 实现Writable接口；

* 反序列化时需要用到反射，所以应有空参构造器；

* 重写序列化/反序列化方法（顺序必须完全一致）

* 若自定义Bean放在Key中，需实现Comparable接口。

**代码：**

```java
// 1 实现writable接口
public class FlowBean implements Writable{
	private long upFlow;
	private long downFlow;	
	//2  反序列化时，需要反射调用空参构造函数，所以必须有
	public FlowBean() {
		super();
	}
	public FlowBean(long upFlow, long downFlow) {
		super();
		this.upFlow = upFlow;
		this.downFlow = downFlow;
		this.sumFlow = upFlow + downFlow;
	}	
	//3  写序列化方法
	@Override
	public void write(DataOutput out) throws IOException {
		out.writeLong(upFlow);
		out.writeLong(downFlow);
	}	
	//4 反序列化方法
	//5 反序列化方法读顺序必须和写序列化方法的写顺序必须一致
	@Override
	public void readFields(DataInput in) throws IOException {
		this.upFlow  = in.readLong();
		this.downFlow = in.readLong();
	}
	//...
}
```

#### 1.2 为什么不使用JDK中的序列化方法

Java的序列化是一个重量级序列化框架（Serializable），一个对象被序列化后，会附带很多额外的信息（各种校验信息，Header，继承体系等），不便于在网络中高效传输，不便于扩展。所以，Hadoop自己开发了一套序列化机制（Writable）。

### 2. MapReduce中的数据提交

#### 2.1 文件切片

##### 2.1.1 什么是切片

数据块（Block）：HDFS中数据保存的单位，HDFS在物理上将数据分为一个一个Block管理

数据切片（Split）：在逻辑上对Map任务输入数据的切片。

##### 2.1.2 为什么要切片

将输入文件分为多片可以并行进行Map阶段的计算，提高Job的运行速度。一份数据切片就会有一个MapTask。

##### 2.1.3 文件的切片机制

* 简单的按照文件的内容长度切片，切片大小默认为Block的大小（128M）,但每次切片完都会判断剩下的部分是否大于块的1.1倍，不大于1.1倍就归入上一个切片
* 切片时不会考虑数据集整体，而是单独针对每一份文件切片（默认）

#### 2.2 任务提交流程

```java
//在客户端Driver中提交任务
job.waitForCompletion()
submit();
// 1建立连接
	connect();	
		// 1）创建提交Job的代理
		new Cluster(getConfiguration());
			// （1）判断是本地yarn还是远程
			initialize(jobTrackAddr, conf); 
// 2 提交job
submitter.submitJobInternal(Job.this, cluster)
	// 1）创建给集群提交数据的Stag路径
	Path jobStagingArea = JobSubmissionFiles.getStagingDir(cluster, conf);
	// 2）获取jobid ，并创建Job路径
	JobID jobId = submitClient.getNewJobID();
	// 3）拷贝jar包到集群
copyAndConfigureFiles(job, submitJobDir);	
	rUploader.uploadFiles(job, jobSubmitDir);
// 4）计算切片，生成切片规划文件
writeSplits(job, submitJobDir);
		maps = writeNewSplits(job, jobSubmitDir);
		input.getSplits(job);
			long minSize = Math.max(getFormatMinSplitSize(), getMinSplitSize(job));
    		long maxSize = getMaxSplitSize(job);
			for (FileStatus file: files) 
                long splitSize = computeSplitSize(blockSize, minSize, maxSize);
					//blockSize与给定的最大值的最小值与给定的Split最小值取最大值
					//最小值默认为1，最大值默认为Long.MAX_VALUE
					Math.max(minSize, Math.min(maxSize, blockSize));
// 5）向Stag路径写XML配置文件
writeConf(conf, submitJobFile);
	conf.writeXml(out);
// 6）提交Job,返回提交状态
status = submitClient.submitJob(jobId, submitJobDir.toString(), job.getCredentials());
```

<img src="picture\submit.png" alt="1586359954724" style="zoom: 50%;" />

1. Job.submit运行，获得一个JobSubmitter对象；
2. 找到数据存储的目录，获取JobId等信息；
3. 遍历所有文件，按要求规划切片信息，并将切片信息写入job.split文件中（只包含了元数据信息）
4. 获取相关参数的.xml文件和任务jar包；
5. 提交到YARN，根据切片信息分配MapTask。

**如何决定Map和Reduce的数量**

1） map的数量：splitSize=max(minSize,min{maxSize,blockSize}) map个数由切片数决定

2） reduce的数量：job.setNumReduceTasks(x);默认为1

#### 2.3 几种不同的切片方法

##### TextInputFormat(默认)

不管文件多小，都会单独规划切片，如果有大量小文件，就会产生大量的MapTask，处理效率极其低下。

##### CombineTextInputFormat

会将多个小文件从逻辑上规划到一个切片中。

`CombineTextInputFormat.setMaxInputSplitSize(job, 4194304);// 4m`

#### 2.4 MapTask读取数据的方法（FileInputFormat）

MapTask将文件读取如内存并按K-V对的形式保存，具体实现依靠FileInputFormat的实现类

##### 2.4.1 TextInputFormat（默认）

按行读取：

键是该行起始位置在整个文件中字节偏移量，LongWritable类型；
值是这行内容，不包括换行符和回车符，Text类型。

##### 2.4.2 KeyValueTextInputFormat

按行读取：

被分隔符分割为K-V对，默认分隔符为'\t'。

##### 2.4.3 NlineInputFormat

文件分片方式不同：每个MapTask不再按Block划分，而是按NlineInputFormat指定的行数N来划分。
键值对读取与默认相同

##### 2.4.4 自定义InputFormat实现类

案例：可以通过这种方法读取小文件合并为一个SequenceFile文件实现大量小文件的合并。

流程：

* 自定义类继承FileInputFormat；
  * 重写isSplitable()返回false不可分割；
  * 重写CreateRecordReader，创建自定义的Reader对象；
* 改写RecordReader，实现一次读取一个文件，封装为KV对；
  * IO流一次读入一个文件，保存为value；
  * 文件路径信息+文件名，保存为key；

```java
// 定义类继承FileInputFormat
public class WholeFileInputformat extends FileInputFormat<Text, BytesWritable>{	
	@Override
	protected boolean isSplitable(JobContext context, Path filename) {
		return false;
	}
	@Override
	public RecordReader<Text, BytesWritable> createRecordReader(InputSplit split, TaskAttemptContext context)	throws IOException, InterruptedException {		
		WholeRecordReader recordReader = new WholeRecordReader();
		recordReader.initialize(split, context);		
		return recordReader;
	}
}
public class WholeRecordReader extends RecordReader<Text, BytesWritable>{
	private Configuration configuration;
	private FileSplit split;
	private boolean isProgress= true;
	private BytesWritable value = new BytesWritable();
	private Text k = new Text();
	@Override
	public void initialize(InputSplit split, TaskAttemptContext context) throws IOException, InterruptedException {		
		this.split = (FileSplit)split;
		configuration = context.getConfiguration();
	}
	@Override
	public boolean nextKeyValue() throws IOException, InterruptedException {		
		if (isProgress) {
			// 1 定义缓存区
			byte[] contents = new byte[(int)split.getLength()];			
			FileSystem fs = null;
			FSDataInputStream fis = null;			
			try {
				// 2 获取文件系统
				Path path = split.getPath();
				fs = path.getFileSystem(configuration);				
				// 3 读取数据
				fis = fs.open(path);				
				// 4 读取文件内容
				IOUtils.readFully(fis, contents, 0, contents.length);				
				// 5 输出文件内容
				value.set(contents, 0, contents.length);
				// 6 获取文件路径及名称
				String name = split.getPath().toString();
				// 7 设置输出的key值
				k.set(name);
			} catch (Exception e) {				
			}finally {
				IOUtils.closeStream(fis);
			}			
			isProgress = false;			
			return true;
		}		
		return false;
	}
	@Override
	public Text getCurrentKey() throws IOException, InterruptedException {
		return k;
	}
	@Override
	public BytesWritable getCurrentValue() throws IOException, InterruptedException {
		return value;
	}
	@Override
	public float getProgress() throws IOException, InterruptedException {
		return 0;
	}
	@Override
	public void close() throws IOException {
	}
}
public class SequenceFileMapper extends Mapper<Text, BytesWritable, Text, BytesWritable>{	
	@Override
	protected void map(Text key, BytesWritable value,Context context)throws IOException, InterruptedException {
		context.write(key, value);
	}
}
public class SequenceFileReducer extends Reducer<Text, BytesWritable, Text, BytesWritable> {
	@Override
	protected void reduce(Text key, Iterable<BytesWritable> values, Context context)		throws IOException, InterruptedException {
		context.write(key, values.iterator().next());
	}
}
public class SequenceFileDriver {
	public static void main(String[] args) throws IOException, ClassNotFoundException, InterruptedException {		
       // 输入输出路径需要根据自己电脑上实际的输入输出路径设置
		args = new String[] { "e:/input/inputinputformat", "e:/output1" };
       // 1 获取job对象
		Configuration conf = new Configuration();
		Job job = Job.getInstance(conf);
       // 2 设置jar包存储位置、关联自定义的mapper和reducer
		job.setJarByClass(SequenceFileDriver.class);
		job.setMapperClass(SequenceFileMapper.class);
		job.setReducerClass(SequenceFileReducer.class);
       // 7设置输入的inputFormat
		job.setInputFormatClass(WholeFileInputformat.class);
       // 8设置输出的outputFormat
	 	job.setOutputFormatClass(SequenceFileOutputFormat.class);       
		// 3 设置map输出端的kv类型
		job.setMapOutputKeyClass(Text.class);
		job.setMapOutputValueClass(BytesWritable.class);		
       // 4 设置最终输出端的kv类型
		job.setOutputKeyClass(Text.class);
		job.setOutputValueClass(BytesWritable.class);
       // 5 设置输入输出路径
		FileInputFormat.setInputPaths(job, new Path(args[0]));
		FileOutputFormat.setOutputPath(job, new Path(args[1]));
       // 6 提交job
		boolean result = job.waitForCompletion(true);
		System.exit(result ? 0 : 1);
	}
}

```

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