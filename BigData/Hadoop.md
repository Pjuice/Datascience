## <center> Hadoop </center>

### 1.Hadoop、Hive、Spark之间关系

在找实习的时候，看到了很多数据相关的岗位都要求掌握Hadoop，Hive，Spark等，刚开始看到的时候一脸懵逼，这些到底都是些什么玩意~ 为了理清这些基础概念，翻阅资料后结合自己的一些理解写下来作为入门。

大数据本身是个很宽泛的概念，Hadoop生态圈(或者泛生态圈)基本上都是为了处理超过单机尺度的数据处理而诞生的。可以把这个生态圈比作一个厨房，厨房中的生产行为需要各种各样的工具，锅碗瓢盆各有各的用处，互相之间又有重合。你可以用汤锅直接当碗吃饭喝汤，你可以用小刀或者刨子去皮。但是每个工具有自己的特性，虽然奇怪的组合也能工作，但是未必是最佳选择。

#### 1.1 存储大数据

为了对大数据进行操作，首先应该将这些海量的数据进行存储。然而传统的文件系统是单机的，不能横跨不同的机器。HDFS(Hadoop Distributed FileSystem)Hadoop分布式文件系统的设计本质上是为了大量的数据能横跨成百上千台机器，但是你看到的是一个文件系统而不是很多文件系统。比如需要获取某个文件的数据，引用的是一个文件路径，但是实际上数据被存放在很多不同的机器上。作为用户，不需要知道这些，就好比不需要知道单机上的一个文件被分散在什么磁道什么扇区一样。HDFS为用户管理这些数据。

存储完这些海量数据后，需要开始考虑如何处理数据。虽然HDFS可以为用户整体管理不同机器上的数据，但是这些数据太大了。如果仅由一台机器读取，可能需要几天甚至数周的时间，所以需要用多台计算机处理。但是如果使用很多台机器处理，就面临着如何分配工作的问题，或者如果一台机器崩溃如何重新启动相应任务，机器之间如何互相通信交换数据以完成复杂计算等问题。这就是<font color = "red">MapReduce / Tez / Spark</font>的功能。MapReduce是第一代计算引擎，Tez和Spark是第二代。MapReduce的设计，采用了很简化的计算模型，只有Map和Reduce两个计算过程(中间用Shuffle串联)，用这个模型，已经可以处理大数据领域很大一部分问题了。

#### 1.2 什么是Map，什么是Reduce

考虑用户需要统计一个存储在类似HDFS上的巨大的文本文件中各个词的出现频率，需要启动一个MapReduce程序。在Map阶段，几百台机器可以同时读取这个文件的各个部分，分别把各自读到的部分分别统计出词频，产生类似(hello, 12100次)，(world，15214次)等等这样的Pair(此处把Map和Combine放在一起收以便简化)；这几百台机器各自都产生了如上的集合，然后又有几百台机器启动Reduce处理。Reducer机器A将从Mapper机器收到所有以A开头的统计结果，机器B将收到B开头的词汇统计结果(当然实际上不会真的以字母开头做依据，而是用函数产生Hash值以避免数据串化。因为类似X开头的词肯定比其他要少得多，而你不希望数据处理各个机器的工作量相差悬殊)。然后这些Reducer将再次汇总，(hello，12100)+(hello，12311)+(hello，345881)= (hello，370292)。每个Reducer都如上处理，你就得到了整个文件的词频结果。

这看似是个很简单的模型，但很多算法都可以用这个模型描述了。

MapReduce的简单模型虽然好用，但是很笨重，第二代的Tez和Spark增加除了内存Cache之类的新feature，本质上来说，是让Map/Reduce模型更通用，让Map和Reduce之间的界限更模糊，数据交换更灵活，更少的磁盘读写，以便更方便地描述复杂算法，取得更高的吞吐量。

有了MapReduce，Tez和Spark之后，程序员发现，MapReduce的程序写起来真麻烦。他们希望简化这个过程。这就好比你有了汇编语言，虽然你几乎什么都能干了，但是你还是觉得繁琐。你希望有个更高层更抽象的语言层来描述算法和数据处理流程。于是就有了Pig和Hive。Pig是接近脚本方式去描述MapReduce，Hive则用的是SQL。它们把脚本和SQL语言翻译成MapReduce程序，丢给计算引擎去计算，而你就从繁琐的MapReduce程序中解脱出来，用更简单更直观的语言去写程序了。

即 <font color = "red">pig(脚本)+Hive(SQL)</font>翻译为MapReduce。

有了Hive之后，人们发现SQL对比Java有巨大的优势。一个是它太容易写了。刚才词频的东西，用SQL描述就只有一两行，MapReduce写起来大约要几十上百行。而更重要的是，非计算机背景的用户终于感受到了爱：我也会写SQL!于是数据分析人员终于从乞求工程师帮忙的窘境解脱出来，工程师也从写奇怪的一次性的处理程序中解脱出来。大家都开心了。Hive逐渐成长成了大数据仓库的核心组件。甚至很多公司的流水线作业集完全是用SQL描述，因为易写易改，一看就懂，容易维护。

自从数据分析人员开始用Hive分析数据之后，它们发现，Hive在MapReduce上跑，真慢! 流水线作业集也许没啥关系，比如24小时更新的推荐，反正24小时内跑完就算了。但是数据分析，人们总是希望能跑更快一些。比如我希望看过去一个小时内多少人在某个商品页面驻足，分别停留了多久，对于一个巨型网站海量数据下，这个处理过程也许要花几十分钟甚至很多小时。而这个分析也许只是你万里长征的第一步，需要分析用户的特性，向老板汇报。你无法忍受等待的折磨，只能跟帅帅的工程师蝈蝈说，快，快，再快一点!

于是Impala，Presto，Drill诞生了(当然还有无数非著名的交互SQL引擎，就不一一列举了)。三个系统的核心理念是，MapReduce引擎太慢，因为它太通用，太强壮，太保守，我们SQL需要更轻量，更激进地获取资源，更专门地对SQL做优化，而且不需要那么多容错性保证(因为系统出错了大不了重新启动任务，如果整个处理时间更短的话，比如几分钟之内)。这些系统让用户更快速地处理SQL任务，牺牲了通用性稳定性等特性。如果说MapReduce是大砍刀，砍啥都不怕，那上面三个就是剔骨刀，灵巧锋利，但是不能搞太大太硬的东西。

这些系统，说实话，一直没有达到人们期望的流行度。因为这时候又两个异类被造出来了。他们是Hive on Tez / Spark和SparkSQL。它们的设计理念是，MapReduce慢，但是如果我用新一代通用计算引擎Tez或者Spark来跑SQL，那我就能跑的更快。而且用户不需要维护两套系统。这就好比如果你厨房小，人又懒，对吃的精细程度要求有限，那你可以买个电饭煲，能蒸能煲能烧，省了好多厨具。

上面的介绍，基本就是一个数据仓库的构架了。底层HDFS，上面跑MapReduce/Tez/Spark，在上面跑Hive，Pig。或者HDFS上直接跑Impala，Drill，Presto。这解决了中低速数据处理的要求。

#### 1.3 如何更高速处理数据？

如果我是一个类似微博的公司，我希望显示不是24小时热博，我想看一个不断变化的热播榜，更新延迟在一分钟之内，上面的手段都将无法胜任。于是又一种计算模型被开发出来，这就是Streaming(流)计算。Storm是最流行的流计算平台。流计算的思路是，如果要达到更实时的更新，我何不在数据流进来的时候就处理了?比如还是词频统计的例子，我的数据流是一个一个的词，我就让他们一边流过我就一边开始统计了。流计算很牛逼，基本无延迟，但是它的短处是，不灵活，你想要统计的东西必须预先知道，毕竟数据流过就没了，你没算的东西就无法补算了。因此它是个很好的东西，但是无法替代上面数据仓库和批处理系统。

还有一个有些独立的模块是KV Store，比如Cassandra，HBase，MongoDB以及很多很多很多很多其他的(多到无法想象)。所以KV Store就是说，我有一堆键值，我能很快速滴获取与这个Key绑定的数据。比如我用身份证号，能取到你的身份数据。这个动作用MapReduce也能完成，但是很可能要扫描整个数据集。而KV Store专用来处理这个操作，所有存和取都专门为此优化了。从几个P的数据中查找一个身份证号，也许只要零点几秒。这让大数据公司的一些专门操作被大大优化了。比如我网页上有个根据订单号查找订单内容的页面，而整个网站的订单数量无法单机数据库存储，我就会考虑用KV Store来存。KV Store的理念是，基本无法处理复杂的计算，大多没法JOIN，也许没法聚合，没有强一致性保证(不同数据分布在不同机器上，你每次读取也许会读到不同的结果，也无法处理类似银行转账那样的强一致性要求的操作)。但是丫就是快。极快。

每个不同的KV Store设计都有不同取舍，有些更快，有些容量更高，有些可以支持更复杂的操作。必有一款适合你。

除此之外，还有一些更特制的系统/组件，比如Mahout是分布式机器学习库，Protobuf是数据交换的编码和库，ZooKeeper是高一致性的分布存取协同系统，等等。

有了这么多乱七八糟的工具，都在同一个集群上运转，大家需要互相尊重有序工作。所以另外一个重要组件是，调度系统。现在最流行的是Yarn。你可以把他看作中央管理，好比你妈在厨房监工，哎，你妹妹切菜切完了，你可以把刀拿去杀鸡了。只要大家都服从你妈分配，那大家都能愉快滴烧菜。

你可以认为，大数据生态圈就是一个厨房工具生态圈。为了做不同的菜，中国菜，日本菜，法国菜，你需要各种不同的工具。而且客人的需求正在复杂化，你的厨具不断被发明，也没有一个万用的厨具可以处理所有情况，因此它会变的越来越复杂。

作者：Xiaoyu Ma ，大数据工程师

### 2. Hadoop 简介

Hadoop是一个由Apache基金会所开发的分布式系统基础架构。

用户可以在不了解分布式底层细节的情况下，开发分布式程序。充分利用集群的威力进行高速运算和存储。

Hadoop实现了一个分布式文件系统（Hadoop Distributed File System），简称HDFS。HDFS有高容错性的特点，并且设计用来部署在低廉的（low-cost）硬件上；而且它提供高吞吐量（high throughput）来访问应用程序的数据，适合那些有着超大数据集（large data set）的应用程序。HDFS放宽了（relax）POSIX的要求，可以以流的形式访问（streaming access）文件系统中的数据。

Hadoop的框架最核心的设计就是：HDFS和MapReduce。HDFS为海量的数据提供了存储，而MapReduce则为海量的数据提供了计算。

Hadoop得以在大数据处理应用中广泛应用得益于其自身在数据提取、变形和加载(ETL)方面上的天然优势。Hadoop的分布式架构，将大数据处理引擎尽可能的靠近存储，对例如像ETL这样的批处理操作相对合适，因为类似这样操作的批处理结果可以直接走向存储。Hadoop的MapReduce功能实现了将单个任务打碎，并将碎片任务(Map)发送到多个节点上，之后再以单个数据集的形式加载(Reduce)到数据仓库里。

#### 核心框架

Hadoop课运行于一般的商用服务器上，具有高容错、高可靠性、高扩展性等特点特别适合写一次，读多次的场景。

Hadoop的架构如下图所示：

![屏幕快照 2019-04-27 上午11.12.32](https://ws2.sinaimg.cn/large/006tNc79gy1g2h0pzevcjj30gk09gmz4.jpg)

- **HDFS:** 分布式文件存储
- **YARN:** 分布式资源管理
- **MapReduce:** 分布式计算
- **Others:** 利用YARN的资源管理功能实现其他的数据处理方式

### 3. Hadoop HDFS

#### 简介

HDFS即Hadoop Distributed File System，分布式文件系统

#### 架构

![屏幕快照 2019-04-27 上午11.52.48](https://ws2.sinaimg.cn/large/006tNc79gy1g2h1vvjjcbj312q0l6asm.jpg)

**Block数据**

1. 基本存储单位，一般大小为64M（配置大的块主要是因为：1）减少搜寻时间，一般硬盘传输速率比寻道时间要快，大的块可以减少寻道时间；2）减少管理块的数据开销，每个块都需要在NameNode上有对应的记录；3）对数据块进行读写，减少建立网络的连接成本）
2. 一个大文件会被拆分成一个个的块，然后存储于不同的机器。如果一个文件少于Block大小，那么实际占用的空间为其文件的大小
3. 基本的读写类似于磁盘的页，每次都是读写一个块
4. 每个块都会被复制到多台机器，默认复制3份

**NameNode**

1. 存储文件的metadata，运行时所有数据都保存到内存，整个HDFS可存储的文件数受限于NameNode的内存大小
2. 一个Block在NameNode中对应一条记录（一般一个block占用150字节），如果是大量的小文件，会消耗大量内存。同时map task的数量是由splits来决定的，所以用MapReduce处理大量的小文件时，就会产生过多的map task，线程管理开销将会增加作业时间。处理大量小文件的速度远远小于处理同等大小的大文件的速度。因此Hadoop建议存储大文件
3. 数据会定时保存到本地磁盘，但不保存block的位置信息，而是由DataNode注册时上报和运行时维护（NameNode中与DataNode相关的信息并不保存到NameNode的文件系统中，而是NameNode每次重启后，动态重建）
4. NameNode失效则整个HDFS都失效了，所以要保证NameNode的可用性

**Secondary NameNode**

1. 定时与NameNode进行同步（定期合并文件系统镜像和编辑日志;，然后把合并后的传给NameNode，替换其镜像，并清空编辑日志，类似于CheckPoint机制），但NameNode失效后仍需要手工将其设置成主机

**DataNode**

1. 保存具体的block数据
2. 负责数据的读写操作和复制操作
3. DataNode启动时会向NameNode报告当前存储的数据块信息，后续也会定时报告修改信息
4. DataNode之间会进行通信，复制数据块，保证数据的冗余性

#### 3.1 HDFS写文件

![屏幕快照 2019-04-27 下午12.01.29](https://ws3.sinaimg.cn/large/006tNc79gy1g2h24u1d9fj30ya0cqmyj.jpg)

1.客户端将文件写入本地磁盘的临时文件中

2.当临时文件大小达到一个block大小时，HDFS client通知NameNode，申请写入文件

3.NameNode在HDFS的文件系统中创建一个文件，并把该block id和要写入的DataNode的列表返回给客户端

4.客户端收到这些信息后，将临时文件写入DataNodes

- 4.1 客户端将文件内容写入第一个DataNode（一般以4kb为单位进行传输）
- 4.2 第一个DataNode接收后，将数据写入本地磁盘，同时也传输给第二个DataNode
- 4.3 依此类推到最后一个DataNode，数据在DataNode之间是通过pipeline的方式进行复制的
- 4.4 后面的DataNode接收完数据后，都会发送一个确认给前一个DataNode，最终第一个DataNode返回确认给客户端
- 4.5 当客户端接收到整个block的确认后，会向NameNode发送一个最终的确认信息
- 4.6 如果写入某个DataNode失败，数据会继续写入其他的DataNode。然后NameNode会找另外一个好的DataNode继续复制，以保证冗余性
- 4.7 每个block都会有一个校验码，并存放到独立的文件中，以便读的时候来验证其完整性

5.文件写完后（客户端关闭），NameNode提交文件（这时文件才可见，֘如果提交前，NameNode垮掉，那文件也就丢失了。fsync：只保证数据的信息写到NameNode上，但并不保证数据已经被写到DataNode中）

**Rack aware（机架感知）**

通过配置文件指定机架名和DNS的对应关系

假设复制参数是3，在写入文件时，会在本地的机架保存一份数据，然后在另外一个机架内保存两份数据（同机架内的传输速度快，从而提高性能）

整个HDFS的集群，最好是负载平衡的，这样才能尽量利用集群的优势

#### 3.2 HDFS读文件

![屏幕快照 2019-04-27 下午12.16.19](https://ws1.sinaimg.cn/large/006tNc79gy1g2h2k7zj3jj311q0cimyc.jpg)

1. 客户端向NameNode发送读取请求
2. NameNode传回文件的所有block和这些block所在的DataNodes（包括复制节点）
3. 客户端直接从DataNode中读取数据，如果该DataNode读取失败（DataNode失效或校验码不对），则从复制节点中读取（如果读取的数据就在本机，则直接读取，否则通过网络读取）

#### 3.3 HDFS可靠性

1. DataNode可以失效

   DataNode会定时发送心跳到NameNode。如果在一段时间内NameNode没有收到DataNode的心跳消息，则认为其失效。此时NameNode就会将该节点的数据（从该节点的复制节点中获取）复制到另外的DataNode中

2. 数据可以毁坏

   无论是写入时还是硬盘本身的问题，只要数据有问题（读取时通过校验码来检测），都可以通过其他的复制节点读取，同时还会再复制一份到健康的节点中

3. NameNode不可靠

#### 3.4 HDFS命令工具

fsck: 检查文件的完整性

start-balancer.sh: 重新平衡HDFS

hdfs dfs -copyFromLocal 从本地磁盘复制文本到HDFS

### 4. Hadoop YARN

Apache Hadoop YARN （Yet Another Resource Negotiator，另一种资源协调者）是一种新的 Hadoop 资源管理器，它是一个通用资源管理系统，可为上层应用提供统一的资源管理和调度，它的引入为集群在利用率、资源统一管理和数据共享等方面带来了巨大好处。

首先，将旧的MapReduce架构与新的YARN架构进行对比，显示出它的优越性。

**旧的MapReduce架构**

![屏幕快照 2019-04-27 下午4.01.00](https://ws4.sinaimg.cn/large/006tNc79gy1g2h92ao6pjj30s20nak3u.jpg)

- **JobTracker:** 负责资源管理，跟踪资源消耗和可用性，作业生命周期管理（调度作业任务，跟踪进度，为任务提供容错）
- **TaskTracker:** 加载或关闭任务，定时报告认为状态

此架构会有以下问题：

1. JobTracker是MapReduce的集中处理点，存在单点故障
2. JobTracker完成了太多的任务，造成了过多的资源消耗，当MapReduce job 非常多的时候，会造成很大的内存开销。这也是业界普遍总结出老Hadoop的MapReduce只能支持4000 节点主机的上限
3. 在TaskTracker端，以map/reduce task的数目作为的资源利用率表示过于简单，没有考虑到cpu/ 内存的占用情况，如果两个大内存消耗的task被调度到了一块，很容易出现OOM
4. 在TaskTracker端，把资源强制划分为map task slot和reduce task slot, 如果当系统中只有map task或者只有reduce task的时候，会造成资源的浪费，也就集群资源利用的问题

总的来说就是**单点问题**和**资源利用率问题**

**YARN架构**

![屏幕快照 2019-04-27 下午4.05.05](https://ws3.sinaimg.cn/large/006tNc79gy1g2h969jaesj30ru0l8apy.jpg)

![屏幕快照 2019-04-27 下午4.05.31](https://ws1.sinaimg.cn/large/006tNc79gy1g2h96qg2igj30ry0v6dym.jpg)

YARN就是将JobTracker的职责进行拆分，将资源管理和任务调度监控拆分成独立的进程：一个全局的资源管理和一个每个作业的管理（ApplicationMaster） ResourceManager和NodeManager提供了计算资源的分配和管理，而ApplicationMaster则完成应用程序的运行。

- **ResourceManager:** 全局资源管理和任务调度
- **NodeManager:** 单个节点的资源管理和监控
- **ApplicationMaster:** 单个作业的资源管理和任务监控
- **Container:** 资源申请的单位和任务运行的容器

**架构对比**

![屏幕快照 2019-04-27 下午11.51.48](https://ws2.sinaimg.cn/large/006tNc79gy1g2hmnxk5pdj30s00p27gg.jpg)

YARN架构下形成了一个通用的资源管理平台和一个通用的应用计算平台，避免了旧架构的单点问题和资源利用率问题，同时也让在其上运行的应用不再局限于MapReduce形式

**YARN基本流程**

![屏幕快照 2019-04-27 下午11.57.16](https://ws4.sinaimg.cn/large/006tNc79gy1g2hmtk6cwaj30rm0rgwrk.jpg)

![屏幕快照 2019-04-27 下午11.57.34](https://ws3.sinaimg.cn/large/006tNc79gy1g2hmtsxuxuj30s80q6n6z.jpg)

**1. Job submission**

从ResourceManager中获取一个Application ID 检查作业输出配置，计算输入分片 拷贝作业资源（job jar、配置文件、分片信息）到HDFS，以便后面任务的执行

**2. Job initialization**

ResourceManager将作业递交给Scheduler（有很多调度算法，一般是根据优先级）Scheduler为作业分配一个Container，ResourceManager就加载一个application master process并交给NodeManager管理ApplicationMaster主要是创建一系列的监控进程来跟踪作业的进度，同时获取输入分片，为每一个分片创建一个Map task和相应的reduce task Application Master还决定如何运行作业，如果作业很小（可配置），则直接在同一个JVM下运行

**3. Task assignment**

ApplicationMaster向Resource Manager申请资源（一个个的Container，指定任务分配的资源要求）一般是根据data locality来分配资源

**4. Task execution**

ApplicationMaster根据ResourceManager的分配情况，在对应的NodeManager中启动Container 从HDFS中读取任务所需资源（job jar，配置文件等），然后执行该任务

**5. Progress and status update**

定时将任务的进度和状态报告给ApplicationMaster Client定时向ApplicationMaster获取整个任务的进度和状态

**6. Job completion**

Client定时检查整个作业是否完成 作业完成后，会清空临时文件、目录等

#### 4.1 YARN  ResourceManager

负责全局的资源管理和任务调度，把整个集群当成计算资源池，只关注分配，不管应用，且不负责容错。

**资源管理**

1. 以前资源是每个节点分成一个个的Map slot和Reduce slot，现在是一个个Container，每个Container可以根据需要运行ApplicationMaster、Map、Reduce或者任意的程序
2. 以前的资源分配是静态的，目前是动态的，资源利用率更高
3. Container是资源申请的单位，一个资源申请格式：<resource-name, priority, resource-requirement, number-of-containers>, resource-name：主机名、机架名或*（代表任意机器）, resource-requirement：目前只支持CPU和内存
4. 用户提交作业到ResourceManager，然后在某个NodeManager上分配一个Container来运行ApplicationMaster，ApplicationMaster再根据自身程序需要向ResourceManager申请资源
5. YARN有一套Container的生命周期管理机制，而ApplicationMaster和其Container之间的管理是应用程序自己定义的

**任务调度**

1. 只关注资源的使用情况，根据需求合理分配资源
2. Scheluer可以根据申请的需要，在特定的机器上申请特定的资源（ApplicationMaster负责申请资源时的数据本地化的考虑，ResourceManager将尽量满足其申请需求，在指定的机器上分配Container，从而减少数据移动）

**内部结构**

![屏幕快照 2019-04-28 上午12.09.35](https://ws4.sinaimg.cn/large/006tNc79gy1g2hn6fr86uj30xg0nytvs.jpg)

- Client Service: 应用提交、终止、输出信息（应用、队列、集群等的状态信息）
- Adaminstration Service: 队列、节点、Client权限管理
- ApplicationMasterService: 注册、终止ApplicationMaster, 获取ApplicationMaster的资源申请或取消的请求，并将其异步地传给Scheduler, 单线程处理
- ApplicationMaster Liveliness Monitor: 接收ApplicationMaster的心跳消息，如果某个ApplicationMaster在一定时间内没有发送心跳，则被任务失效，其资源将会被回收，然后ResourceManager会重新分配一个ApplicationMaster运行该应用（默认尝试2次）
- Resource Tracker Service: 注册节点, 接收各注册节点的心跳消息
- NodeManagers Liveliness Monitor: 监控每个节点的心跳消息，如果长时间没有收到心跳消息，则认为该节点无效, 同时所有在该节点上的Container都标记成无效，也不会调度任务到该节点运行
- ApplicationManager: 管理应用程序，记录和管理已完成的应用
- ApplicationMaster Launcher: 一个应用提交后，负责与NodeManager交互，分配Container并加载ApplicationMaster，也负责终止或销毁
- YarnScheduler: 资源调度分配， 有FIFO(with Priority)，Fair，Capacity方式
- ContainerAllocationExpirer: 管理已分配但没有启用的Container，超过一定时间则将其回收

#### 4.2 YARN NodeManager

Node节点下的Container管理

1. 启动时向ResourceManager注册并定时发送心跳消息，等待ResourceManager的指令
2. 监控Container的运行，维护Container的生命周期，监控Container的资源使用情况
3. 启动或停止Container，管理任务运行时的依赖包（根据ApplicationMaster的需要，启动Container之前将需要的程序及其依赖包、配置文件等拷贝到本地）

**内部结构**

![屏幕快照 2019-04-28 上午12.12.04](https://ws4.sinaimg.cn/large/006tNc79gy1g2hn8x3xa7j30rw0jsduw.jpg)

- NodeStatusUpdater: 启动向ResourceManager注册，报告该节点的可用资源情况，通信的端口和后续状态的维护

- ContainerManager: 接收RPC请求（启动、停止），资源本地化（下载应用需要的资源到本地，根据需要共享这些资源）

  PUBLIC: /filecache

  PRIVATE: /usercache//filecache

  APPLICATION: /usercache//appcache//（在程序完成后会被删除）

- ContainersLauncher: 加载或终止Container

- ContainerMonitor: 监控Container的运行和资源使用情况

- ContainerExecutor: 和底层操作系统交互，加载要运行的程序

#### 4.3 YARN ApplicationMaster

单个作业的资源管理和任务监控

具体功能描述如下：

1. 计算应用的资源需求，资源可以是静态或动态计算的，静态的一般是Client申请时就指定了，动态则需要ApplicationMaster根据应用的运行状态来决定
2. 根据数据来申请对应位置的资源（Data Locality）
3. 向ResourceManager申请资源，与NodeManager交互进行程序的运行和监控，监控申请的资源的使用情况，监控作业进度
4. 跟踪任务状态和进度，定时向ResourceManager发送心跳消息，报告资源的使用情况和应用的进度信息
5. 负责本作业内的任务的容错

ApplicationMaster可以是用任何语言编写的程序，它和ResourceManager和NodeManager之间是通过ProtocolBuf交互，以前是一个全局的JobTracker负责的，现在每个作业都一个，可伸缩性更强，至少不会因为作业太多，造成JobTracker瓶颈。同时将作业的逻辑放到一个独立的ApplicationMaster中，使得灵活性更加高，每个作业都可以有自己的处理方式，不用绑定到MapReduce的处理模式上

**如何计算资源需求**

一般的MapReduce是根据block数量来定Map和Reduce的计算数量，然后一般的Map或Reduce就占用一个Container

**如何发现数据的本地化**

数据本地化是通过HDFS的block分片信息获取的

#### 4.4 YARN Container

1. 基本的资源单位（CPU、内存等）
2. Container可以加载任意程序，而且不限于Java
3. 一个Node可以包含多个Container，也可以是一个大的Container
4. ApplicationMaster可以根据需要，动态申请和释放Container

#### 4.5 YARN Failover

**失败类型**

1. 程序问题
2. 进程崩溃
3. 硬件问题

**失败处理**：

**1.任务失败**

1. 运行时异常或者JVM退出都会报告给ApplicationMaster
2. 通过心跳来检查挂住的任务(timeout)，会检查多次（可配置）才判断该任务是否失效
3. 一个作业的任务失败率超过配置，则认为该作业失败
4. 失败的任务或作业都会有ApplicationMaster重新运行

**2.ApplicationMaster失败**

1. ApplicationMaster定时发送心跳信号到ResourceManager，通常一旦ApplicationMaster失败，则认为失败，但也可以通过配置多次后才失败
2. 一旦ApplicationMaster失败，ResourceManager会启动一个新的ApplicationMaster
3. 新的ApplicationMaster负责恢复之前错误的ApplicationMaster的状态(yarn.app.mapreduce.am.job.recovery.enable=true)，这一步是通过将应用运行状态保存到共享的存储上来实现的，ResourceManager不会负责任务状态的保存和恢复
4. Client也会定时向ApplicationMaster查询进度和状态，一旦发现其失败，则向ResouceManager询问新的ApplicationMaster

**3.NodeManager失败**

1. NodeManager定时发送心跳到ResourceManager，如果超过一段时间没有收到心跳消息，ResourceManager就会将其移除
2. 任何运行在该NodeManager上的任务和ApplicationMaster都会在其他NodeManager上进行恢复
3. 如果某个NodeManager失败的次数太多，ApplicationMaster会将其加入黑名单（ResourceManager没有），任务调度时不在其上运行任务

**4.ResourceManager失败**

1. 通过checkpoint机制，定时将其状态保存到磁盘，然后失败的时候，重新运行
2. 通过zookeeper同步状态和实现透明的HA

可以看出，**一般的错误处理都是由当前模块的父模块进行监控（心跳）和恢复。而最顶端的模块则通过定时保存、同步状态和zookeeper来ֹ实现HA**

### 5.MapReduce

**简介**

一种分布式的计算方式指定一个Map（映射）函数，用来把一组键值对映射成一组新的键值对，指定并发的Reduce（归约）函数，用来保证所有映射的键值对中的每一个共享相同的键组

**Pattern**

![屏幕快照 2019-04-28 上午12.30.33](https://ws1.sinaimg.cn/large/006tNc79gy1g2hns70tisj30o80dcgp5.jpg)

map: (K1, V1) → list(K2, V2) combine: (K2, list(V2)) → list(K2, V2) reduce: (K2, list(V2)) → list(K3, V3)

Map输出格式和Reduce输入格式一定是相同的

**基本流程**

MapReduce主要是先读取文件数据，然后进行Map处理，接着Reduce处理，最后把处理结果写到文件中

![屏幕快照 2019-04-28 上午12.31.53](https://ws2.sinaimg.cn/large/006tNc79gy1g2hntr1dvsj30yu0a6gmr.jpg)

**详细流程**

![屏幕快照 2019-04-28 上午12.33.25](https://ws4.sinaimg.cn/large/006tNc79gy1g2hnv84hr1j31320i210h.jpg)

**多节点下的流程**

![屏幕快照 2019-04-28 上午12.34.03](https://ws4.sinaimg.cn/large/006tNc79gy1g2hnvtlj5zj30yw0u0qbq.jpg)

**主要过程**

![屏幕快照 2019-04-28 上午12.52.16](https://ws2.sinaimg.cn/large/006tNc79gy1g2hoeta1odj313m074wnr.jpg)

**Map Side**

**Record reader**

记录阅读器会翻译由输入格式生成的记录，记录阅读器用于将数据解析给记录，并不分析记录自身。记录读取器的目的是将数据解析成记录，但不分析记录本身。它将数据以键值对的形式传输给mapper。通常键是位置信息，值是构成记录的数据存储块.自定义记录不在本文讨论范围之内.

**Map**

在映射器中用户提供的代码称为中间对。对于键值的具体定义是慎重的，因为定义对于分布式任务的完成具有重要意义.键决定了数据分类的依据，而值决定了处理器中的分析信息.本书的设计模式将会展示大量细节来解释特定键值如何选择.

**Shuffle and Sort**

ruduce任务以随机和排序步骤开始。此步骤写入输出文件并下载到本地计算机。这些数据采用键进行排序以把等价密钥组合到一起。

**Reduce**

reducer采用分组数据作为输入。该功能传递键和此键相关值的迭代器。可以采用多种方式来汇总、过滤或者合并数据。当ruduce功能完成，就会发送0个或多个键值对。

**输出格式**

输出格式会转换最终的键值对并写入文件。默认情况下键和值以tab分割，各记录以换行符分割。因此可以自定义更多输出格式，最终数据会写入HDFS。类似记录读取，自定义输出格式不在本书范围。

#### 5.1 Mapreduce 读取数据

通过InputFormat决定读取的数据的类型，然后拆分成一个个InputSplit，每个InputSplit对应一个Map处理，RecordReader读取InputSplit的内容给Map

**InputFormat**

决定读取数据的格式，可以是文件或数据库等。

**功能**

1. 验证作业输入的正确性，如格式等
2. 将输入文件切割成逻辑分片(InputSplit)，一个InputSplit将会被分配给一个独立的Map任务
3. 提供RecordReader实现，读取InputSplit中的"K-V对"供Mapper使用

**方法**

**List getSplits():** 获取由输入文件计算出输入分片(InputSplit)，解决数据或文件分割成片问题

**RecordReader <k,v>createRecordReader():</k,v>** 创建RecordReader，从InputSplit中读取数据，解决读取分片中数据问题

**类结构**

![屏幕快照 2019-04-28 上午1.01.08](https://ws3.sinaimg.cn/large/006tNc79gy1g2honzd8ktj314a0pk0yt.jpg)

**TextInputFormat:** 输入文件中的每一行就是一个记录，Key是这一行的byte offset，而value是这一行的内容

**KeyValueTextInputFormat:** 输入文件中每一行就是一个记录，第一个分隔符字符切分每行。在分隔符字符之前的内容为Key，在之后的为Value。分隔符变量通过key.value.separator.in.input.line变量设置，默认为(\t)字符。

**NLineInputFormat:** 与TextInputFormat一样，但每个数据块必须保证有且只有Ｎ行，mapred.line.input.format.linespermap属性，默认为１

**SequenceFileInputFormat:** 一个用来读取字符流数据的InputFormat，<key,value>为用户自定义的。字符流数据是Hadoop自定义的压缩的二进制数据格式。它用来优化从一个MapReduce任务的输出到另一个MapReduce任务的输入之间的数据传输过程。</key,value>

**InputSplit**

代表一个个逻辑分片，并没有真正存储数据，只是提供了一个如何将数据分片的方法

Split内有Location信息，利于数据局部化

一个InputSplit给一个单独的Map处理

```java
public abstract class InputSplit {
      /**
       * 获取Split的大小，支持根据size对InputSplit排序.
       */
      public abstract long getLength() throws IOException, InterruptedException;

      /**
       * 获取存储该分片的数据所在的节点位置.
       */
      public abstract String[] getLocations() throws IOException, InterruptedException;
}
```

**RecordReader**

将InputSplit拆分成一个个<key,value>对给Map处理，也是实际的文件读取分隔对象</key,value>

#### 5.2 MapReduce Mapper

主要是读取InputSplit的每一个Key,Value对并进行处理

```java
public class Mapper<KEYIN, VALUEIN, KEYOUT, VALUEOUT> {
    /**
     * 预处理，仅在map task启动时运行一次
     */
    protected void setup(Context context) throws  IOException, InterruptedException {
    }

    /**
     * 对于InputSplit中的每一对<key, value>都会运行一次
     */
    @SuppressWarnings("unchecked")
    protected void map(KEYIN key, VALUEIN value, Context context) throws IOException, InterruptedException {
        context.write((KEYOUT) key, (VALUEOUT) value);
    }

    /**
     * 扫尾工作，比如关闭流等
     */
    protected void cleanup(Context context) throws IOException, InterruptedException {
    }

    /**
     * map task的驱动器
     */
    public void run(Context context) throws IOException, InterruptedException {
        setup(context);
        while (context.nextKeyValue()) {
            map(context.getCurrentKey(), context.getCurrentValue(), context);
        }
        cleanup(context);
    }
}

public class MapContext<KEYIN, VALUEIN, KEYOUT, VALUEOUT> extends TaskInputOutputContext<KEYIN, VALUEIN, KEYOUT, VALUEOUT> {
    private RecordReader<KEYIN, VALUEIN> reader;
    private InputSplit split;

    /**
     * Get the input split for this map.
     */
    public InputSplit getInputSplit() {
        return split;
    }

    @Override
    public KEYIN getCurrentKey() throws IOException, InterruptedException {
        return reader.getCurrentKey();
    }

    @Override
    public VALUEIN getCurrentValue() throws IOException, InterruptedException {
        return reader.getCurrentValue();
    }

    @Override
    public boolean nextKeyValue() throws IOException, InterruptedException {
        return reader.nextKeyValue();
    }
}
```

#### 5.3 MapReduce Shuffle

对Map的结果进行排序并传输到Reduce进行处理 Map的结果并不直接存放到硬盘,而是利用缓存做一些预排序处理 Map会调用Combiner，压缩，按key进行分区、排序等，尽量减少结果的大小 每个Map完成后都会通知Task，然后Reduce就可以进行处理

![屏幕快照 2019-04-28 上午1.11.00](https://ws4.sinaimg.cn/large/006tNc79gy1g2hoy9bc25j31400hsqap.jpg)

**Map端**

当Map程序开始产生结果的时候，并不是直接写到文件的，而是利用缓存做一些排序方面的预处理操作

每个Map任务都有一个循环内存缓冲区（默认100MB），当缓存的内容达到80%时，后台线程开始将内容写到文件，此时Map任务可以&#x#x7EE7;续输出结果，但如果缓冲区满了，Map任务则需要等待

写文件使用round-robin方式。在写入文件之前，先将数据按照Reduce进行分区。对于每一个分区，都会在内存中根据key进行排序，如果配置了Combiner，则排序后执行Combiner（Combine之后可以减少写入文件和传输的数据）

每次结果达到缓冲区的阀值时，都会创建一个文件，在Map结束时，可能会产生大量的文件。在Map完成前，会将这些文件进行合并和排序。如果文件的数量超过3个，则&##x5408;并后会再次运行Combiner（1、2个文件就没有必要了）

如果配置了压缩，则最终写入的文件会先进行压缩，这样可以减少写入和传输的数据

一旦Map完成，则通知任务管理器，此时Reduce就可以开始复制结果数据

**Reduce端**

Map的结果文件都存放到运行Map任务的机器的本地硬盘中

如果Map的结果很少，则直接放到内存，否则写入文件中

同时后台线程将这些文件进行合并和排序到一个更大的文件中（如果文件是压缩的ÿ#xFF0C;则需要先解压）

当所有的Map结果都被复制和合并后，就会调用Reduce方法

Reduce结果会写入到HDFS中

**调优**

一般的原则是给shuffle分配尽可能多的内存，但前提是要保证Map、Reduce任务有足够的内存

对于Map，主要就是避免把文件写入磁盘，例如使用Combiner，增大io.sort.mb的值

对于Reduce，主要是把Map的结果尽可能地保存到内存中，同样也是要避免把中间结果写入磁盘。默认情况下，所有的内存都是分配给Reduce方法的，如果Reduce方法不怎&##x4E48;消耗内存，可以mapred.inmem.merge.threshold设成0，mapred.job.reduce.input.buffer.percent设成1.0

在任务监控中可通过Spilled records counter来监控写入磁盘的数，但这个值是包括map和reduce的

对于IO方面，可以Map的结果可以使用压缩，同时增大buffer size（io.file.buffer.size，默认4kb）

