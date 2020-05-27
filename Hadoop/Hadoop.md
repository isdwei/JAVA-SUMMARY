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

<img src="picture\read.jpg" alt="1586269880115" style="zoom:67%;" />

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

<img src="picture\write.jpg" alt="1586269992823" style="zoom:67%;" />

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

* Gzip 不支持Split  压缩率较高，速度较快 自带

* Bzip2 支持Split   压缩率很高，自带    压缩解压慢

* Lzo  支持Split   压缩率合理，不自带   压缩解压快 最流行

* Snappy 不支持Split 压缩率比Gzip低，不自带   高速压缩  

#### 2.6 Hadoop 归档

HDFS中的文件以块为单位存储，块的元数据存储在NameNode中。 **HDFS中每个文件、目录、数据块占用150Bytes的空间**，因此大量的小文件会耗尽NameNode的大部分内存。 

HadoopArchives能够将多个小文件打包成一个HAR文件，这样在减少namenode内存使用的同时，仍然允许用户使用一个以har://开头的URL就可以访问HAR文件中的小文件。**使用HAR files可以减少HDFS中的文件数量**。

但访问一个指定的小文件需要访问两层索引文件才能获取小文件在HAR文件中的存储位置，因此，访问一个HAR文件的效率可能会比直接访问HDFS文件要低。**对于一个mapreduce任务来说，如果使用HAR文件作为其输入，仍旧是其中每个小文件对应一个map task，效率低下。所以，HAR files最好是用于文件归档。**

<img src="picture\har.png" alt="1586350902340" style="zoom: 33%;" />

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

### 2. MapReduce中的数据输入与输出

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

#### 2.5 MapReduce输出数据的方法（OutputFormat）

##### 2.5.1 TextOutputFormat

每条记录写为文本行

##### 2.5.2 SequenceFileOutputFormat

序列化格式输出，紧凑容易被压缩。

2.5.3 自定义格式输出（继承OutputFormat）



### 3. MapReduceTask工作流程

#### 3.1 概述

MapReduce框架中，一个Task被分为Map和Reduce两个阶段，每个MapTask处理数据集合中的一个split并将产生的数据溢写入本地磁盘；而每个ReduceTask远程通过HTTP以pull的方式拉取相应的中间数据文件，经过合并计算后将结果写入HDFS。

#### 3.2 MapTask

<img src="picture\shuffle.png" alt="1586412639068" style="zoom: 33%;" />

客户端提交任务，规划切片，并将切片文件，程序jar包与任务配置文件上传，Hadoop集群分配节点执行任务；

* **Read阶段**：MapTask按InputFormat中定义的方法将数据读取如内存；

* **Map阶段**：将内存中的数据按map()函数定义的方法处理成K-V对的形式；

* **Collect阶段**：将K-V对的数据与索引向两个方向写入环形缓冲区（默认100M大小）；

  * 调用Partitioner.getPartition()获取记录的分区号，再将<key,value,partition>传给MapOutputBuffer.collect()做进一步处理；

  * MapOutputBuffer内部使用了一个缓冲区暂时存储用户输出数据。几种不同的缓冲区优劣：

    *  单向缓冲区：生产者像缓冲区中单向写，写满后一次性写磁盘。性能低，不能同时读写数据。
    * 双缓冲区：一个用于写入数据，一个用于溢写磁盘，交替读写。仍会存在读写等待问题。
    * 环形缓冲区：缓冲区使用率达到一定阈值后便开始溢写磁盘，同时生产者仍可以像不断增加的剩余空间中循环写入数据，读写并行。

  * 环形缓冲区实际是通过一个线性缓冲区模拟，通过取模操作实现循环取数据。缓冲区的读写采用的是典型的但生产者消费者模型。

  * Hadoop2.x让kvmeta、kvbuffer共用一个环形缓冲区，Hadoop0.22之前是采用了两级索引结构，涉及三个缓冲区，分别为kvoffsets、kvindices、kvbuffer。

  * 实际实现中元数据信息是从末尾到头的顺序写入数组，元数据16byte，包括index、partition、keystart、valuestart。

    ![1586410952072](picture\kv.png)

* **Spill阶段**：缓冲区内存占用到80%（默认）后溢写入本地磁盘；

  * 溢写入磁盘之前会对缓冲区中的数据先按照分区编号Partition再按key进行排序 ，使用的是Hadoop自己实现的快速排序，Hadoop中快速排序的优化体现在：

    （1）枢轴选择：中位数中元素作为枢轴；

    （2）子序列划分：左右两个扫描的索引遇到元素大于等于/小于等于枢轴元素时才会停止；

    （3）对相同元素的优化：将序列化分为三部分，中间部分为与枢轴相同的元素，不参与后续的递归；

    （4）减少递归次序：子序列中元素数目小于13时使用插入排序，不再继续递归。

  * 按照分区编号由小到大依次将每个分区中的数据写入任务工作目录下的临时文件output/spillN.out（N表示当前溢写次数）中。如果用户设置了Combiner，则写入文件之前，对每个分区中的数据进行一次聚集操作。  

  *  将分区数据的元信息写到内存索引数据结构SpillRecord中，其中每个分区的元信息包括在临时文件中的偏移量、压缩前数据大小和压缩后数据大小。如果当前内存索引大小超过1MB，则将内存索引写到文件output/spillN.out.index中。  

  * 溢写同时仍然可以写入数据。

* **Combiner阶段**：当所有数据处理完成后，MapTask对所有临时文件进行合并、排序，最后只会生成一个数据文件

  * 按分区合并文件，对于某个分区，采用多轮归并的方式排序合并；
  * 每轮合并100（默认）个文件，并将新文件加入待合并列表中，知道得到最后一个大文件。MapTask的输出结果保存为IFile的格式（支持按行压缩，可用Zlib、BZip2等压缩算法减小IO数据量）

#### 3.3 ReduceTask

ReduceTask从各个MapTask上读取数据，排序后以组为单位交给reduce()函数处理并将结果写入HDFS。

<img src="picture\reduce.png" alt="1586412707867" style="zoom:33%;" />

* **Shuffle阶段**：ReduceTask从各个MapTask上拷贝一片数据，针对每一片数据，如果其大小超过阈值，则写入磁盘，否则直接放在内存中。Map阶段完成至少进行5%以后开始拷贝。

* **Merge阶段**：拷贝远程数据的同时，ReduceTask启动两个后台线程对内存和磁盘上的文件合并。（归并排序）

  Shuffle和Merge过程可以分为三个子阶段：

  * GetMapEventsThread线程周期性（心跳间隔）通过RPC从TaskTracker获取已完成的MapTask列表。
  * ReduceTask同时启动多个MapOutputCopier线程（默认5个），通过HTTP Get远程拷贝数据，存入内存或磁盘中。
  * ReduceTask启动LocalFSMerger和InMemFSMergeThread两个线程进行Merge。
  * 当内存中的数据已占有超过66%的堆内存、或内存中文件数超过1000个或阻塞在ShuffleRamManager上的请求数大于拷贝线程数的75%时，内存中的数据会溢写入磁盘。

* **Sort阶段**：Merge阶段已经减少了文件数目，这阶段只需对所有数据进行一次归并排序即可。

* **Reduce阶段**：按key将数据传入reduce()函数中聚合。

  * Sort阶段与Reduce阶段也是并行的。在Sort阶段，ReduceTask为内存和磁盘中的文件建立了小顶堆，保存了指向根节点的迭代器。ReduceTask不断移动迭代器，以将相同key的数据顺次交给reduce()函数处理。移动迭代器的过程其实就是不断调整小顶堆的过程。
  * 这两个阶段必须在数据全部拷贝完后才能进行，是一个性能瓶颈。
  * 这一阶段可以自定义GroupingComparator进行分组排序

* **Write阶段**：将计算结果写入HDFS。

#### 3.4 MapReduce的优化

MapReduce优化方法主要从六个方面考虑：数据输入、Map阶段、Reduce阶段、IO传输、数据倾斜问题和常用的调优参数。

* 数据输入：合并小文件，CombineTextInputFormat输入

* Map：减少Spill次数（调整环形内存大小和触发spill的阈值），减少Merge次数（增大一次Merge的文件数），进行Combiner处理；

* Reduce：合理设置Map，Reduce任务数，设置Map，Reduce共存，规避使用reduce，合理设置Reduce端Buffer(保留一定比例的内存直接给Reduce使用)；

* IO：数据压缩（Snappy，LZO），使用SequenceFile二进制文件，spill次数过多，merge次数过多。

* 数据倾斜：抽样范围分区，自定义分区，Combine，采用MapJoin(避免Reducejoin)。

* 参数调优：

  * 可设置Map和Reduce可用的内存上限、CPU核数、Container占用的CPU、内存资源、环形缓冲区大小、溢写阈值大小、是否压缩、压缩器选择、ReduceTask同时启动的拷贝线程数、Reduce阶段的内存阈值、合并阈值等。

  * 杜克大学Starfish项目：自动参数调优

* 系统调优：

  * 避免排序；
  * Shuffle阶段内部优化：Netty代替Jetty（Hadoop2.X）、Reduce端批拷贝、Shuffle阶段独立出来以分离IO与CPU的资源重叠使用；
  * C++改写；

**HDFS小文件优化方法**

HDFS小文件弊端：HDFS上每个文件都要在NameNode上建立一个索引，这个索引的大小约为150byte，这样当小文件比较多的时候，就会产生很多的索引文件，一方面会大量占用NameNode的内存空间，另一方面就是索引文件过大使得索引速度变慢。

HDFS小文件解决方案

* 在数据采集的时候，就将小文件或小批数据合成大文件再上传HDFS。（归档）

* 在业务处理之前，在HDFS上使用MR对小文件进行合并。（SequenceFile）

* 在MapReduce处理时，可采用CombineTextInputFormat提高效率。

* 开启JVM重用

#### 3.5 MapReduce的一些应用

##### 3.5.1 TopN

数据保存入一个小顶堆（可用Reduce端放置一个TreeMap实现）

##### 3.5.2 join

MapJoin：驱动中加入缓存，重新setup阶段保存小表，性能更优

ReduceJoin：连接键作为key，性能较差





 

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

## 三、Yarn

#### 1. 下一代MapReduce框架的设计思想

Hadoop1.0的弊端：

* Hadoop MapReduce在可扩展性、资源利用率和多框架支持等方面存在不足。
* Hadoop未能将资源调度与应用程序计算的功能分开，造成Hadoop难以支持多种框架。

<img src="picture\h1.png" alt="1586501579751" style="zoom:33%;" />

下一代MapReduce的设计思想：将JobTracker的两个主要功能：资源管理和作业控制拆分成两个独立的进程。其中资源管理进程（YARN）与具体应用程序无关，只负责资源的调度，而作业控制进程（ApplicationMaster）则是直接与应用程序相关的模块，每个作业控制进程只负责管理一个作业。

<img src="E:\JAVA-SUMMARY\Hadoop\picture\h2.png" alt="1586502829601" style="zoom:33%;" />

#### 2. Yarn的组成结构

##### 2.1 ResourceManager

RM是一个全局的资源管理器，由调度器（Scheduler）和应用程序管理器组成（Application Manager，ASM）

* **调度器Scheduler**：根据容量、队列等限制条件将**资源**分配给各个正在运行的应用程序。调度器根据各个应用程序的资源需求进行资源分配，而资源分配用一个抽象的概念Container。Container是一个动态资源分配单位，将内存、CPU、磁盘、网络等资源封装在一起，从而限定每个任务可使用的资源量。
* **应用程序管理器Applications Manager**：负责应用程序的提交、与调度器协商资源以起动AM、监控AM运行状态并在失败时重启它。

##### 2.2 ApplicationMaster

每一个应用程序均会包含一个AM，主要功能包括：

* 与RM调度器协商以获取资源（用Container表示）；
* 将得到的任务进一步分配给内部的任务；
* 与NM通信以启动任务；
* 监控所有任务的运行状态。

YARN自带了两个AM的实现，分别用于演示AM编写方法的示例程序和用于MR的MRApplicationMaster。

##### 2.3 NodeManager

NM是每个节点上的资源和任务管理器，一方面它会定期向RM汇报本节点上资源的使用情况和各个Container的运行状态；另一方面它接受来自AM的Container启动/停止等各种请求。

##### 2.4 Container

Container是YARN中的资源抽象

### 3. Yarn的工作流程

<img src="picture\yarnwork.png" alt="1586507778521" style="zoom:33%;" />

* 用户向YARN中提交应用程序，其中包括ApplicationMaster程序、启动ApplicationMaster的命令、用户程序等。
* ResourceManager为该应用程序分配第一个Container，并与对应的NodeManager通信，要求它在这个Container中启动应用程序的ApplicationMaster。
* ApplicationMaster首先向ResourceManager注册，这样用户可以直接通过ResourceManager查看应用程序的运行状态，然后它将为各个任务申请资源，并监控它的运行状态，直到运行结束。
* ApplicationMaster 采用轮询的方式通过 RPC 协议向 ResourceManager申请和领取资源。
* 一旦 ApplicationMaster 申请到资源后，便与对应的NodeManager通信，要求它启动任务。
* NodeManager为任务设置好运行环境（ 包括环境变量、JAR 包、二进制程序等）后，将任务启动命令写到一个脚本中， 并通过运行该脚本启动任务。
* 各个任务通过某个RPC 协议向ApplicationMaster 汇报自己的状态和进度， 以让 ApplicationMaster 随时掌握各个任务的运行状态， 从而可以在任务失败时重新启动任务。在应用程序运行过程中，用户可随时通过 RPC向ApplicationMaster查询应用程序的当前运行状态。
* 应用程序运行完成后，ApplicationMaster向ResourceManager注销并关闭自己。  

### 4. ResourceManager剖析

ResourceManager通过两个RPC协议与NodeManager和各个ApplicationMaster交互。

#### 4.1 ApplicationMaster管理

ApplicationMaster管理功能主要有三个服务构成：ApplicationMasterLauncher、AMLivelinessMonitor、ApplicationMasterService，他们共同管理应用程序的ApplicationMaster的生存周期。

* 首先用户向YARN ResourceManager提交应用程序，RM收到提交请求后，先向资源调度器申请以启动MpplicationMaster的资源，申请到资源后，再由ApplicationMasterLauncher对应的NodeManager通信，从而启动应用程序的ApplicationMaster。
* ApplicatonMaster启动完成后，ApplicationMasterLauncher会通过事件的形式，将刚刚启动的ApplicationMaster注册到AMLivelinessMonitor，以启动心跳监控；
* ApplicatonMaster启动后，先向ApplicationMasterService注册，并将自己所在host、端口号等信息汇报给它。
* ApplicationMaster运行过程中，周期性地向ApplicationMasterService汇报心跳（心跳中包含所需资源的描述）。
* ApplicationMasterService每次收到心跳信息后将通知AMLivelinessMonitor更新其最近汇报心跳的时间。
* 应用程序完成后，AM向ApplicationMasterService发送请求，注销自己。
* ApplicationMasterService收到注销请求后标记应用程序运行状态为完成，同时通知AMLivelinessMonitor移除对它的心跳监控。

#### 4.3 NodeManager管理

NodeManager管理部分主要有三个服务构成：NMLivelinessMonitor、NodesListManager、ResourceTrackerService，它们共同管理NM的生存周期。

* NMLivelinessMonitor：周期性遍历集群中所有的NodeManager，只要有一个NM在一定时间内（10min）未汇报心跳，则认为其死亡，上面运行的所有Container都被认为失败。但RM不会重新执行这些Container，而是通过心跳机制告诉AM，由AM决定是否重新执行。
* NodesListManager：管理exlude（黑名单）和include（白名单）节点列表。
* ResourceTrackerService：负责处理来自各个NodeManager的请求，主要包括注册和心跳。

#### 4.4 Application管理

* ApplicationACLsManager：负责管理应用程序的访问权限

* RMAppManager：负责应用程序的启动与关闭

* ContainerAllocationExpirer：当一个AM收到RM新分配的Container后，必须在10min钟内在对应的NM上启动该Container，否则RM会强制收回该Container。

### 5. Hadoop HA 基本框架

在Master/Slave架构中，为了解决单点故障问题们通常采用热备方案。
Hadoop2.0采用了基于Zookeeper的方案。

#### 5.1 HDFS HA实现

通过双NameNode消除单点故障。架构组成包括：

* **MasterHADaemon**：与Master服务运行在一个进程中，可接受外部RPC命令；

* **SharedStorage**：共享存储系统，Active Master将信息写入其中，而StandbyMaster读取该信息以保持和Active Master同步。

* **ZKFailoverController**：基于Zookeeper实现的切换控制器，主要由ActiveStandbyElector和HealthMonitor组成。其中ActiveStandbyElector负责与zookeeper集群交互，通过尝试获取全局锁判断所管理的Master是进入Active还是Standby状态；HealthMonitor负责监控各个活动Master的状态。具体来说包含三个功能：

  * **健康监测**，ZKFC使用一个健康检查命令定期地ping与之在相同主机的NameNode，只要该NameNode及时地回复健康状态，ZKFC认为该节点是健康的。如果该节点崩溃，冻结或进入不健康状态，健康监测器标识该节点为非健康的。  

  * **ZooKeeper会话管理**，当本地NameNode是健康的，ZKFC保持一个在ZooKeeper中打开的会话。如果本地NameNode处于active状态，ZKFC也保持一个特殊的znode锁，该锁使用了ZooKeeper对短暂节点的支持，如果会话终止，锁节点将自动删除。  

  * **基于ZooKeeper的选择**，如果本地NameNode是健康的，且ZKFC发现没有其它的节点当前持有znode锁，它将为自己获取该锁。如果成功，则它已经赢得了选择，并负责运行故障转移进程以使它的本地NameNode为Active。故障转移进程与前面描述的手动故障转移相似，首先如果必要保护之前的现役NameNode，然后本地NameNode转换为Active状态。  

* **Zookeeper**：核心功能是维护一把全局锁控制集群只有一个ActiveMaster。

<img src="picture\ha.png" alt="1586526878373" style="zoom: 25%;" />

HDFS HA的实现中要考虑以下几个问题：

* 两个NameNode的内存中各自保存一份元数据，Edits日志只有Active状态的NameNode节点可以做写操作，两个NameNode都可以读取Edits，共享的Edits放在一个共享存储中管理（qjournal和NFS两个主流实现，主要是qjournal）。

* 实现了一个zkfailover，每个zkfailover负责监控自己所在的NN节点，利用zk进行状态标识，当需要进行状态切换时，zkfailover负责切换。为了保证切换过程透明，Hadoop应保证所有客户端和Slave可以自动重定向到新的ActiveMaster上。

* 脑裂（brain-split），主备切换时，由于切换不彻底，导致客户端和Slave误以为出现两个ActiveMaster。通常采用隔离（Fencing）机制解决：

  * 共享存储隔离：只有一个Master可以往共享存储中写数据；
  * 客户端隔离：只有一个Master可以相应客户端请求；
  * Slave隔离：只有一个Master可以向Slave发命令。

  Hadoop提供了两种隔离实现：

  * sshfence，通过SSH登录目标节点使用fuser将进程杀死（TCP端口号定位）

  * shellfence，执行一个用户事先定义的Shell命令完成隔离

#### 5.2 Yarn HA实现

ResourceManager并不会保存已经分配给每个ApplicationMaster的资源信息和每个NodeManager的资源使用信息，这些均可通过心跳重构出来，因此，Yarn HA实现是很轻量级的。

Yarn将共享存储系统抽象成RMStateStore，以保存恢复ResoureManager所需的必备信息，包括Application状态信息，Application对应的每个ApplicationAttempt信息和安全令牌信息。

ResourceManager重启后，已经运行的程序无须重新提交，正在运行的Container将被杀死。

### 6. 资源调度器剖析

资源调度器是YARN中最核心的组件之一，他是RM中的一个插拔式服务组件，负责集群资源的管理和分配。

#### 6.1 资源调度模型

##### 6.1.1 双层资源调度模型

YARN采用了双层资源调度模型：第一层中RM中的资源调度器将资源分配给ApplicationMaster，第二层中ApplicationMaster才将资源分配给他其中的各个任务。YARN的资源分配是异步的，也就是说资源调度器将资源分配给一个应用程序后，不是立即push给对应的AM，而是等AM的心跳来主动获取。

具体资源分配过程如下：

* **NodeManager**通过周期性心跳汇报节点信息；
* **RM**为NM返回一个心跳应答，包括需要释放的Container列表等信息；
* RM收到NM信息后会触发NODE_UPDATE事件;
* **ResourceScheduler**收到NODE_UPDATE事件后，按一定的策略将该节点上的资源分配各应用程序，并将分配结果放到一个内存数据结构中；
* 应用程序的**AM**向RM发送心跳以领取最新分配的**Container**；
* RM收到心跳后，将为其分配的资源以心跳应答的方式返回；
* AM收到Container，并将资源分配给各个任务。

##### 6.1.2 资源保证机制

分布式计算中，有两种资源调度器，当程序申请的资源暂时无法保证时

* 增量资源分配：优先为应用程序预留一个节点上的资源直到空闲资源满足需求；
* 一次性资源分配：暂时放弃当前资源直到出现一个节点剩余资源一次性满足需求。

YARN采用增量资源分配机制，这会降低资源利用率，但不会出现饿死现象。

##### 6.1.3 资源分配算法

YARN采用了主资源公平调度算法（Dominant Resource Fairness，DRF），该算法扩展类最大最小公平算法，使其能够支持多维资源调度。

DRF算法中，将所需份额最大的资源称为主资源，DRF算法总是最大化所有主资源中最小的。

#### 6.2 资源抢占模型

资源调度器中，每个队列可设置一个最小资源量和最大资源量，其中最小资源量是资源紧缺时每个队列需保证的资源量，而最大资源量是极端情况下队列也不能超过的资源使用量。但最小资源并不是硬资源保障，当队列不需要资源时，调度器会将这些资源暂借给别的队列使用。仅当负载较轻的队列需要资源时，调度器才将本属于该队列的资源分配给他。若一段时间后资源仍未归还，则会发生资源抢占。

#### 6.3 Capacity Scheduler

Capacity Scheduler是Yahoo开发的多用户调度器，他以队列为单位划分资源，每个队列设定一定比例的最低保障和资源上限，当一个队列资源有剩余时，可暂时将剩余资源共享给其他队列。

YARN采用了三级资源分配方式，当一个节点上有空闲资源时，他会依次选择队列，应用程序和container请求使用该资源：

<img src="picture\1.png" alt="1586622654134" style="zoom:33%;" />

* **选择队列**：YARN基于优先级的深度优先遍历算法，当遇到子队列是叶子队列，则在其中选择一个Container请求并退出。
* **选择应用程序**：选中一个子队列后，Capacity Scheduler按提交时间对叶子队列的应用程序排序，并遍历排序后的应用程序以找到一个或多个合适的Container请求；
* **选择Container请求**：对于同一个Application，他的请求可能是多样化的，涉及不同的节点、资源和数量。Capacity Scheduler会优先满足优先级高的Container请求。对于同一优先级优先选择满足本地性的Container。

#### 6.4 Fair Schedular

Fair Schedular是Facebook开发的多用户调度器，他与Capacity Schedular区别在于它允许每个队列单独配置调度策略。当前有三种策略可选：FIFO，Fair和DRF。可以认为Fair Schedular具备了Capacity Schedular的所有功能。

### 7. NodeManager剖析

NodeManager是YARN上单个节点的代理，他管理集群中单个计算节点，功能包括：与RM保持通讯，管理Container生命周期，监控Container是资源使用，追踪节点健康状况，管理日志等。