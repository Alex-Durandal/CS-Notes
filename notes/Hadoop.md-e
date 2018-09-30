# Hadoop

## MapReduce

### 简述mapreduce的工作机制

#### Mapreduce1中的工作机制

角色主要包括:客户端、jobtracker、tasktracker

- Jobtracker:协调作业的运行
- Tasktracker:负责运行作业划分之后的任务

工作流程：

1. 客户端向jobtracker请求一个新的作业，检查作业的输出路径是否存在，若存在则抛出异常。若不存在，jobtracker向客户端返回iob相关资源的提交路径以及jobId
2. 客户端将job所需的资源(jar文件、配置文件)提交到共享文件系统中
3. 告知jobtracker已将job复制到共享文件系统，准备执行
4. jobtracker将提交的job放入内部的任务队列，由作业调度器进行调度，并进行初始化(包括创建一个表示正在运行作业的对象，用于封装任务和记录信息)
5. jobtracker的作业调度器从共享文件系统获取客户端计算好的输入分片，以创建任务运行列表

6. tasktracker通过心跳与jobtracker保持通信，报告自己的状态，以及是否准备好运行。一个task，若已经准备好，则jobtracker通过一定的调度算法从jobtracker中获得一个task分配给tasktracker
7. tasktracker在共享文件系统中获得任务相关资源，实现jar本地化，并创建响应的文件夹以及一个taskrunner运行该任务

8. taskrunner启动一个新的jvm，在新启动的ivm中运行任务

9. 进度与状态的更新有一个独立的线程向tasktracker报告当前任务状态，同时，tasktracker每隔5秒钟向jobtracker通过心跳发送状态。Jobtracker将这些更新合并，发送给客户端

缺点：

1. JobTracker是Map-reduce的集中处理点，存在单点故障
2. JobTracker完成了太多的任务，造成了过多的资源消耗，当map-reduce job非常多的时候，会造成很大的内存开销，也增加了JobTracker fail的风险，这也是业界普遍总结出老Hadoop的Map-Reduce只能支持4000节点主机的上限。
3. 在TaskTracker端，以map/reduce task的数目作为资源的表示过于简单，没有考虑到cpu/内存的占用情况，如果两个大内存消耗的task被调度到了一块，很容易出现OOM
4. 在TaskTracker端，把资源强制划分为map task slot和reduce task slot。Slot是对cpu、mem等系统资源的抽象，每个task不一定使用slot的全部资源，造成浪费。如果当系统中只有map task或者只有reduce task的时候，由于map slot与reduce slot不能共享也会造成资源的浪费。

#### mapreduce2即Yarn中的工作机制

在 Yarn中将JobTracker两个主要的功能：资源管理和任务调度/监控分离成单独的组件，ResourceManager和ApplicationMaster。新的资源管理器全局管理所有应用程序计算资源的分配，每一个应用的ApplicationMaster负责相应的调度和协调。

Yarn中主要角色包括: ResourceManager、 ApplicationMaster和NodeManager

- ResourceManager:启动每一个 Job所属的 ApplicationMaster、另外监控ApplicationMaster以及NodeManager的存在情况，并且负责协调集群上计算资源的分配
- ApplicationMaster:每个job有一个ApplicationMaster，负责运行MapReduce的任务，并负责报告任务状态
- NodeManager:负责启动和管理节点中的容器


工作流程：

1. 客户端向ResourceManager发送Job请求，客户端产生Runjar进程与ResourceManager通过RPC通信
2. ResourceManager向客户端返回Job相关资源的提交路径以及jobID
3. 客户端将job相关的资源提交到相应的共享文件系统的路径下，客户端向ResourceManager提交job
4. ResourceManager通过调度器在NodeManager创建一个容器，并在容器中启用MRAppMaster进程(进程由ResourceManager启动) ，该MRAppMaster进程对作业进行初始化，创建多个对象对作业进行跟踪
5. MRAppMaster从共享文件系统中获得计算得到的输入分片，只获取分片信息，不需要jar等相关资源。为每一个分片创建一个map以及指定数量的reduce对象。之后，MRAppMaster决定如何运行构成mapreduce作业的各个任务。如果作业很小，则与MRAppMaster在同一个JVM上运行；若作业很大，则MRAppMaster会为所有map任务和reduce任务向ResourceManager发起申请容器资源请求，请求中包含了map任务的数据本地化信息以及输入分片等信息
6. Resourcemanager为任务分配了容器之后，MRAppMaster就通过与NodeManager通信启动容器，由MRAppMaster负责分配在哪些NodeManager上运行map(即yarn child进程)和reduce任务
7. 运行map和reduce任务的NodeManager从共享文件系统中获取job相关资源，包括jar文件、配置文件等
8. 运行map和reduce任务，关于状态的检测与更新不经过ResourceManager，任务周期性的向MRAppMaster汇报状态及进度，客户端每秒钟通过查询一次MRAppMaster获取状态更新信息

**注意:**

- 由于ResourceManager负责资源的分配，当NodeManager启动时，会向ResourceManager注册，而注册信息中会包含该节点可分配的CPU和内存总量
- YARN的资源分配过程是异步的，也就是说，资源调度器将资源分配给一个application后，不会立刻push给对应的ApplicaitonMaster，而是暂时放到一个缓冲区中，等待ApplicationMaster通过周期性的RPC函数主动来取

参考链接：

[YARN/MRv2 Resource Manager深入剖析—资源调度器](http://dongxicheng.org/mapreduce-nextgen/yarnmrv2-resource-manager-resource-manager/)

### 简述mapreduce中的shuffle机制，以及存在哪些缺陷

Map的输出作为reduce的输入传给reduce的过程称为shuffle

Map端:

1. 每个map任务维护一个内存缓冲区。当map开始产生输出数据时，先将数据写入内存缓冲区，当内存缓冲区达到设置的阈值之后，会将缓冲区的数据溢出到本地磁盘。
2. 溢出的文件成为spill文件，在将内存缓冲区的数据写入磁盘之前，会先根据reduce的数量对缓冲区的数据进行分区。
3. 在每个分区中对数据按键进行排序，如果有combiner函数，则会在**排序后的输出上运行**，使得map的输出结果更加紧凑。
4. 每次内存缓冲区达到阈值溢出时，便会在磁盘创建一个spill文件，当最后一个spill文件写完之后，会有多个溢出文件，会对多个溢出文件进行合并，形成一个已经分区并排序的大文件。如果有combiner，在**合并多个spill文件时**,也会在输出上运行。
5. 将压缩的map输出写入到磁盘是个很好的主意，可以减少磁盘I/O量。

Reduce端: 

1. Reduce通过HTTP的方式从map获取数据，reduce有少量的复制线程，可以并行的从map上复制数据。
2. Reduce可能需要从多个map任务中获取数据(通过MRAppMaster获取哪些节点有map输出)，因此**只要多个map中的一个完成, reduce便可以从map复制数据**。
3. 如果map的输出数据比较小，会直接复制到内存；如果数据比较大,当达到内存缓冲区阈值，则会溢出到磁盘。
4. 随着磁盘中溢出文件的增加会进行排序合并，最后一次的合并结果作为reduce的输入。最后一次合并不一定会合并成一个文件。有合并因子，默认为10，我们可以自行设置每一趟需要合并的文件数。
5. 对已经排序的输出数据中的每个键调用reduce函数。此阶段的输出直接写入文件系统，一般是HDFS。一般reduce节点也是数据节点，会将第一个副本写入本地磁盘。

缺陷：

- MR的shuffle，每个map可能会有多个spill文件，写入磁盘会产生较多的磁盘I/O。
- 在数据量很小，但是map任务和reduce任务很多时，会产生很多网络I/O

### reduce获取map输出

reduce通过MRAppMaster获取哪些节点有map输出。

首先，MRAppMaster负责任务的调度以及监控，当map执行结束之后，MRAppMaster会得知该map结束

其次，MRAppMaster知道map与reduce任务之间的映射关系，reduce中的一个线程会定期询问MRAppMaster以便获取map输出的位置

### MapReduce优化

- 自定义partition函数,使得key值较为均匀的分布在reducer上
- Map输出使用压缩
- 使用combiner函数

#### mapreduce之间如何选择压缩格式？

对于map与reduce的中间数据的压缩,通常选用snappy压缩,是压缩速率与低cpu开销的结合:而对于reduce输出的1缩通常使用bzip2,尽可能压缩文件. Bzip2支持块级别的压缩,会按照块的边界进行压缩.

#### mapreduce的计数器

Hadoop为每个作业维护了若干内置计数器包括两大类：任务计数器和作业计数器。

- 任务计数器主要用于采集任务的相关信息，由与其关联的任务维护，每个作业的所有任务结果会被聚集起来，比如每个map统计本身输入记录的总数，并在一个作业的所有map上进行聚集，统计所有的map输入记录总数等
- 作业计数器由ApplicationMaster维护，比如统计失败的map数量或者reduce数量等

同时用户也可以定义计数器



#### 切片?

输入切片抽象为inputSplit。包含一个以字节为单位的长度和一组存储位置。用于

mapreduce系统将map任务尽量放在切片附近。外片大小用来排序,优先处理最大的分片、切片不需要直接处理,而是inputformat创建。客户端通过getsplits()i计算分片.Map任务将!切片传给InputFormat的getRecordReader. RecordReader将记录转化为kv对传给map任务,在inputformat.getinputSplit()中计算切片信息的实现过程为;

1.通过listStatus()获取输入文件列表files,其中会遍历输入目录的子目录,并过滤掉部分文件,如文件SUCCESS

2.获取所有的文件大小totalSlze 

3.goalSlze-totalsize/numMaps. numMaps是用户指定的map数目

4.files中取出一个文件file

5.计算splitsize, splitSize=max(minSplitsize,min(file.blocksize,goalSize)),其中minSplitSize是允许的最小分片大小,默认为1B

6后面根据splitSize大小将file分片。在分片的时候,如果剩余的大小不大于splitSize" 1.1,且大于0B的时候,会将该区域整个作为一个分片。这样做是为了防止一个mapper处理的数据太小

7将file的分片加入到splits中

8返回4,直到将files遍历完

9结束,返回splits

#### 如何避免切片?

1) .将切片的最小值设置为大于文件的大小

2),使用FilelnputFormat的具体子类,重写isSplitable ()方法,把返回值设置为FALSE

#### Hadoop中输入切片inputsplit?如何避免切片？

Hadoop中对切片的抽象为inputsplit.InputSplit包含一个以字节为单位的长度和一组存储位置(即一组主机名),注意,一个分片并不包含数据本身,而是指向数据的用reference),存储位置供MapReduce系统使用以便将map任务尽量放在分片致据附近,而长度用来排序分片,以便优先处理最大的分片,从而最小化作业运行时间,Mapreduce的开发人员不需要关注inputsplit. inputsplit是有inputformat来进行创建的,

非将其装换为record

publie interface Inputformatck, v (



Inputspl1t(] getspltstobConf Job, Ant nuntplits) throws 1oxception;



ResordReaderck, v> etRerordReader(Inputar1t split,

Reporter reerter) throws 10fxception;

通过getinputsplit方法计算输入切片,返回一个inoutsplit数组,通过getRecordReader

方法,返回每个inputsplit的recordreader. recordreader是inputsplit上的迭代器. Map任务,

使用recordreader生成键值对,再将其传给map任务. Recordreader通过其next方法,不

断的讲kv传给map任务,当到达输入流的末尾时, next方法返回FALSE, map任务结束

Kkey sreader.createKeyt;i

Vvalue - reader. createvalve():

while (reader.next(key, value))

mapper.nap(key, value, output, reporter);

」

最大的分片大小默认是由Java long类型表示的最大值,这样做的效果是:当它的,

值被设置成小于块大小时,将强制分片比块小.

分片的大小由以下公式计算(参见FilelnputFormat的computeSplitSize()方法),max(mininumsize. min (maximuesize. blocks1ze))

默认情况下:

minimunsize c blocksize saximussize

所以分片的大小就是blocksize,这些参数的不同设置及其如何影响最终分片大

其中minimumSize的默认值为1字节, maximumSize的默认值为Long.maximum.

补充:如果想要将文件当做一个把整个文件作为一条记录处理有时, mapper需要访问,一个文件中的全部内容。即使不分割文件,仍然需要一个RecordReader、来读取文件内容作为record的值. wholeFilelnputFormat展示了如此做的方法. WholeFilelnputFormat重写了isSplite方法和getRecordReader方法。

对于小文件的处理: archive, combinerinputformat,sequencefile

---



## HDFS

### HDFS的读流程

1. 客户端通过FileSystem.open()方法打开想要读取的文件，其实是distributedFileSystem通过RPC与NameNode通信打开文件。创建输入流FSDatalnputStream给客户端。客户端利用这个输入流读取数据。
2. 客户端在读数据时，首先调用getblocklocations()方法，通过RPC调用与NameNode通信，获取文件起始块的位置，包括其副本的节点信息。这些信息根据与客户端的距离进行了简单的排序(排序主要是通过两个节点间的带宽) 。
3. 客户端通过输入流的read()方法在相应的DataNode上读取数据。当到达块的末端时，DFSInputStream会关闭和数据节点的连接，再次通过getBlockLocations()方法获取下一个数据块的节点信息。
4. 读取结束后通过close关闭输入流。
5. 在读数据的过程中如果某个数据块被客户端读取，则向相应的DataNode发送success反馈。每个DataNode有一个数据块扫描器，会周期性校验该DataNode上数据块是否正常。客户端读取数据后如果没错，则会向DataNode发送success的反馈，这样数据块扫描器扫描时，该数据块可以跳过，提高效率。

**读过程发生错误:**

当读过程中节点出现故障，客户端会尝试从其他数据节点读取信息，同时记住故障节点，并通过reportbadblock()方法上报给NameNode。

### HDFS的写文件过程

1),首先客户端通过DistributedFileSystem对象调用create方法创建文件。这时,DistributedFileSystem会创建一个DFSOutPutStream,通过PRC调用让namenode执行同名方法,在文件系统的命名空间中创建一个文件,并记录创建操作到编辑日志中,此时文件还没有相应的数据块。

2)在创建文件时, namenode会检查这个文件是否存在以及客户端是否拥有对该文件的权限。若检查通过namenode会为新创建的文件记录一条记录,否则像客户端抛出异常。

3)远程同名create方法执行结束后DistributedFileSystem会向客户端返回一个封装了DFSOutPutStream的FSDataOutputStream对象,客户端可以开始写入数据。

4),由于此时只是创建以一个空的文件,客户端通过调用addblock方法向namenode申请数据块,返回一个locatedBlock对象,里面的locs数组包含了数据块及其副本的信息。

5)客户端通过DFSOutPutStream与DataNode建立数据流管道。

6),客户端在写之前会将数据分成一个个的数据包,放入数据队列中,发往数据流管道。

7)数据包会写入管线中的第一个DataNode,第一个DataNode将数据写入第二个,依次类推。当数据包写入第一个DataNode后,会在数据队列中删除,加入到确认队列中。

8)同时也维护一个确认队列,当收到了所有的DataNode的写成功的确认信息,则在确认队列中将该数据包删除。

9)当数据块写满后,会通过blockReceive()方法向namenode提交已经写完的数据块。若,此时数据队列中还有数据未写完,则继续通过addblock方法向namenode申请数据块,继续上述步骤。

10)当全部写完之后通过complete方法通知名字节点关闭文件,完成数据的写。

细节:打开一个DFSOutputStream流, Client会写数据流到内部的一个缓冲区中,然后数据被分解成多个Packet,每个Packet大小为64k字节,每个Packet又由一组chunk和这组chunk对应的checksum数据组成,默认chunk大小为512字节,每个checksum是对512字节数据计算的校验和数据。

当Client写入的字节流数据达到一个Packet的长度,这个Packet会被构建出来,然后会被放到队列dataQueue中,接着DataStreamer线程会不断地从dataQueue队列中取出Packet,复制发送到Pipeline中的第一个DataNode上,并将该Packet从dataQueue队列中移到ackQueue队列中。ResponseProcessor线程接收从Datanode发送过来的ack,如果是一个成功的ack,表示复制Pipeline中的所有Datanode都已经接收到这个Packet.ResponseProcessor线程将packet从队列ackQueue中删除。

在发送过程中,如果发生错误,所有未完成的Packet都会从ackQueue队列中移除掉,然后重新创建一个新的Pipeline,排除掉出错的那些DataNode节点,接着DataStreamer线程继续从dataQueue队列中发送Packet.

#### HDFS写入过程的几种可能

1. client崩溃

2. DataNode故障
3. namenode故障

#### HDFS写数据时某一副本出错处理

1).首先会关闭管线,将已经发送到管道中但是没有收到确认的数据包重新写回到数据队列,这样无论哪个节点发生故障,都不会发生数据丢失。这个过程是在确认队列中将未收到确认的数据包删除,写回到数据队列。

2)当前正常工作的数据节点将会被赋予一个新的版本号(利用namenode中租约的信息可以获得最新的时间戳版本) ,这样故障节点恢复后由于版本信息不对,故障DataNode恢复后会被删除。

3).在当前正常的DataNode中根据租约信息选择一个主DataNode,并与其他正常DataNode通信,获取每个DataNode当前数据块的大小,从中选择一个最小值,将每个正常的DataNode同步到该大小。之后重新建立管道。

4)在管线中删除故障节点,并把数据写入管线中余下正常的DataNode,即新的管道。当文件关闭后, namenode发现副本数量不足时会在另一个节点上创建一个新的副本.

#### HDFS写入过程中Client出错处理（租约恢复）

客户端崩溃时,便不可以周期性的更新租约,此时namenode便可以感知到.

当数据写入过程中客户端异常退出时,同一数据块的不同副本可能存在不一致的状态,选择某一副本作为主数据节点,协调其他数据节点,将该数据块恢复到他们中的最小长度.数据块恢复配合租约恢复是HDES中故障恢复的重要机制.

lease recovery算法:

1) NameNode查找lease信息:

2)对于该客户端lease中的每个文件f,令b为f的最后一个block,作如下操作:

2.1)获取b所在的datanode列表,

2.2)令其中一个datanode作为primarydatanode p.

2.3) p从NameNode获取最新的时间戴,

2.4) p从每个DataNode获取block信息

2.5) p计算最小的block长度,

2.61p用最小的block长度和最新的时间戳来更新具有有效时间戳的datanode

2.7)0通知NameNode更新结果:

2.8) NameNode更新Blockinfo

2.9) NameNode从lease中删除f,如果此时该lease中所有文件都已被删除,将删除该

lease.

2.10) Namenode提交修改的EditLog

2.11)当客户端恢复后,重新与namenode通信,此时namenode租约已删除,客户端会,

以append的方式继续写入即可.

#### HDFS写入过程中DataNode出错如何处理

以客户端写入过程中出错,客户端主动发起数据块恢复为例。

当数据节点出现故障后,客户端会调用recoverblock方法进行数据块恢复。该方法会通过与所有参与到该写入过程中的正常节点中选取一个节点作为主节点。主节点需要获取所有各个数据节点上的信息便于进行数据块恢复。

1首先主节点循环创建与其他数据节点的InterDataNodeProtocol实例并调用startblockrecovery()方法获取数据块的恢复信息,因为是数据块恢复节点,必须要将数据节点上对于此数据块操作的线程中断,避免恢复时受这些线程的影响;同时有可能处于数据协恢复状态的数据块的数据文件和校验文件不一致, startblockrecovery在返回数据块信息时先,对数据块进行一次校验,当主节点获取到参与了写操作的所有正常节点的数据块信息后,计,算得到所有的数据节点列表和需要恢复到的数据块长度(取所有数据节点中相应数据块的最,小值,这种情况是在client故障时的处理策略):同步具体过程如下:

2向namenode申请一个新的版本号(根据namenode中的租约信息获得) ,更新所有,正常状态的数据块版本号,以避免故障的数据节点恢复后上报过时的数据块,将其删除。3根据获得到版本号以及数据块长度构建新的数据块信息,然后通过,interDataNodeProtocol.syncblock()方法与其他节点同步数据块.

4同步向namenode上报这次恢复的结果.

5.最后recoverblock会返回一个locatedblock对象,根据其中的locs变量重新建立管线,继续写入数据.

6数据块写完之后, blockreceive上报数据块后,无论是否发生过故障, namenode都会检查该文件当前拥有的副本数p409,通过循环检查文件拥有的所有数据块,若不满足会执行数据块的复制,将其加入到neededReplications中(复制过程见数据块的管理), namenode发现数据块的副本数小于目标值时,名字节点利用DataNodeCommand向一个拥有该数据块的数据节点发送transfer命令,将数据块复制到其他目标节点,这里目标节点可以有多个..也是建立管线通过blocksender往管道里发送数据.

### namenode中维护的元数据中都存储了哪些信息

文件名,副本的数量,每一个块的DataNode位置。Namenode中对数据块的索引的元信息内容包括:文件路径-副本数量---([blk_1:h0,h1,h2],[blk2:ho,h1,h2], [blk3:h0,h1,h2]}

Namenode作用: 1).Namenode管理文件系统的命名空间,维护文件的数据块索引,管理这些信息的文件有两个,分别是Namespace镜像文件(Namespace image)和操作日志文件(edit log),这两个文件也会被持久化存储在本地硬盘2).Namenode记录着每个文件中各个块所在的数据节点的位置信息,但是他并不持久化存储这些信息,因为这些信息会在系统启动时从数据节点重建。

Fsimage用于存储文件系统的目录,元数据以及文件的数据块索引,即每一个文件的数据块列表。后续对这些数据的修改写进editlog.

但是数据块与数据节点的对应关系并不会持久化到本地。而是系统启动后,有DataNode向namenode发送心跳,上报自己所包含的的数据块

### HDFS上的副本放置，如何建立三副本

1.第一个副本一般与客户端在同一节点(如果客户端在集群之外，则随机选择一个节点)

2.第二个副本选择放在与第一个副本不同机架上的随机选择的另外一个机架上的节点

3.第三个副本选择与第二个副本同一机架上的不同节点

4,其他副本放在随机选择的节点上.

我在这里主要说明一下Hadoop的replication policies.

我们知道当我们要write data到datanode时,首先要通过namenode确定文件是否已经,存在,若不存在则DataStreamer会请求namenode确定新分配的block的位置,然后write就行

具体namenode如何确定选择哪个datanode存储数据呢?这里namenode会参考可靠,

性,读写的带宽等因素来确定。具体如下说明:

假设replica factor=3, Hadoop会将第一个replica放到client node里,这里node是随

机选择的,当然hadoop还是想不要选择过于busy过于full的node:

第二个replica会随机选择和第一个不在同一rack的node;

第三个replica放到和第二个一样的rack里,但是随机选择一个不同的node.

如果replica factor更大则其他副本随即在cluster里选择。当然这里hadoop还是随机的,

尽管我们都知道尽量不要吧更多的replica放到同一个rack里,这不仅影响可靠性而且读写

的带宽有可能成为瓶颈。

当replica的location确定之后, write的pipline就会建成,里面是被分解的data packets,

然后按照网络的拓扑结构进行操作。

### HDFS的namenode目录树（namenode的第一关系管理）

-. INode

在名字节点中,对文件和目录的抽象使用INODE对类进行命名.INODE是一个抽象类,,gNodeDirectory和INodeFile的父类。其中INodeDirectory代表了HDFS中的目录,而INodeFile代表i HDFS中的文件。

INode里面包含了文件和目录的共有属性,比如文件目名,父目录,最后修改时间,最后访问时间,访问权限permission等信息。

Inode中的permission是通过一个long的整形,通过利用Java的枚举,将长整型64字"介为三段,实现权限的控制:文件访问权限,文件主标识符,文件所在用户组标识符,但,ti种权限是在一个64位的long类型中存储. HDFS中用户名和用户标识的映射,用户a1,用产组标识的映射存在于SerialNumberManager对象中,通过此对象namenode不需要,INode中记冰字符串形式的用户名和用户组名,节省对象对内存的占用.

1.NodeDirectory有一个子类: INodeDirectoryWithQuota,是带有配额的目录.

在InodeDirectory中调用removeChild时,只是简单的将child列表中该节点删除,但是,删除文件时,存在大量的数据块,怎么才能删除文件所有的数据块?如下:"

Inode中有一个抽象方法collectSubTreeBlocksAndClear(),用来收集INode所有孩子的block.因为INode可能是文件或者目录,目录的话就不含有Block,而文件则有多个Block,它会返回INode所在子目录树中所有文件拥有的数据块。在调用删除节点前,名字节点的处理逻辑( FSDriectory中)会调用此方法, 收集目录拥有的所有数据块。collectSubTreeBlocksAndClear的实现方式是典型的通过递归遍历目录树模式.

INodeDirectory中有一个Inode的成员变量parent,代表父目录:同时还有一个Inode的列表children: INodeDirectory的removeNode方法实际是调用父目录的removeChild方法,在目录中删除代表自身的Inode.同时INodeDirectory中的方法有removeChild. get add等,INodeDirectoryWithQuato用于实现HDFS的配额机制,配额有两种:节点配额与空间配.顺,节点配额用于限制目录下的名字数量:而空间配额用于现在存储在目录中的所有文件的总规模,保证用户不会过多的占用数据节点的资源.

2.INodeFile与INodeFileUnderConstruction:

NodeFile是namenode中对文件的抽象,继承自INode. INodeFile有两个特殊的属性:header与blocks.其中header在一个长整型里面保存了文件的副本系数和文件数据块的大,小:数组blocks存放着文件拥有的数据块,数组元素类型为Blockinfo.

Blockinfo是blockMap的静态内部类,从Blockinfo保存着数据块和文件的映射关系以及,数据块与数据节点的映射关系。是namenode第一关系与第二关系的桥梁.

INodeFileUnderConstruction是INodeFile的子类,是处于构建状态的文件索引节点,当!客户端为写数据打开HDFS文件时,改文件处于构建状态,在HDFS的目录树中就是一个INodefileUnderConstructrion对象.

INodefileUnderConstruction内部成员变量有(未列全P333),比较重要的是租约恢复相关：

Clientname:发起写文件的客户端名称,这个属性也用于租约管理中:

ClientMachine:客户端所在节点索引.

primaryNodetndex和lastRecoveryTime:主要用于由名字节点发起的数据块恢复(租约恢复) ,分别保存租约恢复时的主数据节点索引以及开始恢复时间.



基于jdbc的方法会产生一批insert语句,每个语句向表中插入多条记录)

二. HDFS命名空间镜像与编辑日志?

由于FSImage存储着某一时刻的文件系统的系统镜像文件,如果使其与内存中的元数据时时刻刻保持一致,由于磁盘10等会降低性能。因此HDFS对于元数据的修改记录到EDITLog!中。编辑口志和fsimage一起确定当前时刻文件系统的元数据。

在数据节点中对存储空间的管理由datastorage与fsdataset完成,但是在名字节点中有Fsimage与FSEditlog完成。Fsimage起主导作用,管理存储空间的生存期,同时也负责Fsimage,的保存和加载,还需要与第二名字节点合作。

1.Fsimage继承自Storage, Storage有内部类storageDirectory,可以管理多个存储目录。Fslmage.saveFSimage()会将当前时刻的命名空间镜像,保存在newfile指定的文件中。内容包,括根节点、其他节点、正在构建中的节点、安全信息。

FSimage保存当前时刻命名空间镜像的过程是:

1)首先输出镜像文件的文件头,包括版本号、存储标识、目录树包含的节点数等信息

2)使用savelnode2image()输出根节点

3)使用savelmage()保存目录树中的其他节点。由于用户名与用户组名以整数的形式保存,在Inode.permission中,但是在命名空间镜像中是以字符串的形式保存,因此需要现在serialNumberManager中根据映射关系得到对应字符串。

4)保存构建中的节点, (包含了租约信息)

5)安全信息。

在实际的写入过程中, savelmage()会循环输出当前目录(假设为current代表的节点为foo)的所有子节点,每一个子节点:

1)设置缓冲区的位置,将缓冲区中的内容写为: /foo

2)将当前目录(bar)追加到缓冲区,这时缓冲区保存的是/foo/bar,

3)调用savelnode2image()输出节点信息,

如果foo下有多个输出项,在下一个循环开始时会重新设置缓冲区位置,并将bar在缓,冲区中抹去,重复上述步骤。在整个过程中需要保存父节点的绝对路径,保存在缓冲区parentPrefix中。以/foo/bar为例,当输出bar时,父目录为foo,当输出bar下的目录时父目录为foo/bar/,子节点输出完成之后, savelmage会通过递归调用,输出current下的子目录。



savelnode2image用于输出一个INode对象,在FSImage中存储的都是绝对路径。通过,savelnode2lmage输出的不同类型节点( InodeFileUnderConstruction、InodeFile、INodeDirectory)有不同的结构。 InodeFile与INodeDirectory的输出结构相同,区分的话是利!用其中的数据块数量来区分, INodeDirectory作为目录没有文件数据块,标志位为-1,同时,permission在Inode中是以整数形式存在,而在fsimage中是以字符串形式存在,需要进行,转换(存在SerialNumberManager,保存在映射关系)



2.FSEditLog的保存:包括操作码和操作参数两部分。

EditLog保存的是修改namenode第一关系的事件。日志可以抽象为只允许添加数据的,输出流。

FSEditLog写入时拥有两个缓冲区: bufCurrent日志写入缓冲区与bufReady写文件缓冲区。首先会写入日志写入缓冲区,当日志写入缓冲区的数据需要写入文件时,会交换两个缓,冲区,将日志写入缓冲区变为写文件缓冲区,写文件缓冲区变成日志写入缓冲区。这样可以避免一个缓冲区写满后的延迟

3.FSImage的读取(FSImage.loadimage())



三. HOFS FSDirectory的实现?

在HDFS中,名字节点的业务逻辑是对目录树进行操作,包括增删改查,有些需要写入志文件,同时启动时需要载入fsimage和editlog.为了屏蔽子系统的复杂性,引入Directory为复杂的第一关系提供一个简单的接口。Namenode的其他子系统通过Directory操作目录树。FSDirectory知晓设计目录树的操作应该分派到哪一个子系统。sDirectory有大量的方法列表



1.主要的成员变量有:

FSNameSystem类型的namesystem,主要负责对数据块的操作

Boolean类型的ready,当完成fsimage与editlog的加载之后,可以对目录树进行操作FSimage类型变量fsimage,负责对fsimage与editlog的操作

INodeDirectoryWithQuota类型的变量rootdir,是整个文件系统的根目录。

包含成员变量FSNameSystem, FSDirectory通过FSnamesystem访问数据块管理的功能.



2.成员函数:在clientProtocal中的方法基本都在FSDirectory中找得到。

例如

addToParent方法,在利用FSimage.loadimage()方法载入fsimage时,每载入一条记录都会调用FSDirectory.addToParent方法,向目录树添加相应的Inode.注意,如果此时是文件,则需要向blockmap中添加映射关系。

(即删除文件或者目录的过程) Delete方法删除目录树上的一项。调用FSDirectory.unprotecteddelete()方法执行实际的删除,并往日志文件中记录该操作。首先获得要删除的节点的路径:然后通过父节点的removeChild删除该节点;同时利用INODE.collectSubTreeBlocksAndClear()方法收集该目录下所有的数据块信息;通过FSNamesystem.removePathAndBlocks()方法删除所有数据块和租约信息。



### HDFS的一致性模型

在HDFS上新建一个文件后,它在文件系统的命名空间中立即可见:

当数据块正在写入时,写入的内容不能立即可见。当写入的数据超过一个数据块时,第一个数据块对新的reader就可见了。总之,正在写入的数据块对其他reader不可见。

### HDFS读取文件镜像时，如何判断是文件还是目录？

P341

应为在存储目录与文件,在命名空间中都有一定的格式,由于目录不存在数据块,所以其中的数据块数字段为-1代表着是目录,读取命名空间镜像时根据这个字段判断当前的输入时文件还是目录。

### namenode的数据块和数据节点管理（namenode的第二关系管理）

一,数据结构

我们知道在目录树中InodeFile代表着一个文件,其中有由数组成员变量Blockinfo,很自1然的和第二关系发生关联。

1.BlockMap:管理者名字节点上数据块的元信息。包括数据块所属的INODE以及这些数|据块保存在哪些DataNode。也就是说如果想定位某个数据块在哪些数据节点上,只需要访问blockMap对象. blockMap有一个内部类: blockinfo.

2.DataNodeDescriptor是名字节点中对数据节点的抽象。 DataNodeDescriptor包含三部分信息,一部分是DataNode的状态信息(isAlive,撤销等);一部分用于产生到DataNode的命|令指令(replicateBlock,recoverblock,invalidateblokcs,分别是数据块复制,恢复,删除) ,第

三部分是一个成员变量blocklist(该数据节点保存的数据块队列的头结点):以删除数据块为,例,在invalidateblocks中保存了这个数据节点上等待删除的数据块,在下次心跳中,会发送一个DNA-INVALIDATA操作、让数据节点删除invalidateblocks中的数据块;如果是replicatesblocks,包括两项信息:数据块和目标数据节点列表.

当数据节点成功接收到一个数据块之后,要通过远程方法blockReceived)方法报告给namenode,此时namenode有两个操作:

首先,利用DatanodeDescriptor.addblock()方法,调用blockinfo.addNode()方法将对象自己DataNodeDescirptor加入到数据块所属数据节点列表中

然后,通过DataNodeDescirptor.listinsert()方法将数据块添加到数据节点管理的数据块列表中。

Blockmap与DataNodeDescriptor一起构成了namenode的第二关系

3.Blockinfo:从BlockInfo保存着数据块和文件的映射关系以及数据块与数据节点的映射关系。数据块所在数据节点信息,保存在一个obiect数组中,而不是DataNodeDescritptor对象数组。在这个数组中第i个数据节点信息保存在[3*i,还以双向循环链表的形式保存改数据节点上其他两个数据块的数据块信息对象, 3*i+1保存了当前数据块的前一个数据块,3*i+2保存了后面一个数据块的blockInfo.

4.FSNamesystem:

Blockmap与DataNodeDescriptor一起构成了namenode的第二关系。但是对于数据块的管理(主要是副本状态进行管理)需要FSNamesystem来实现,同时FSNamesystem也管理着DataNode启动时的握手、注册、上报数据块等方法,

fsnamesystem中有DataNodemap保存着名字节点当前管理的所有DataNode,并提供了通过DataNode的storagelD迅速定位到其DataNodeDescriptor的能力。会出现以下故障:1)客户端在写入数据的过程中DataNode发生故障,由客户端进行数据块恢复

2)客户端写入数据过程中崩溃,需要namenode进行租约恢复。

3)数据节点磁盘损坏:

4)数据节点出现故障:

### namenode中对于数据块的管理（增删改查）

1.FSNameSystem.addStoreBlock():该方法用于在blockmap中添加更新数据节点上的数,

据块副本,使用场景: DataNode数据块写成功后会通过blockReceive()方法向namenode上.

报,此时namenode通过namenode.blockReceive()方法更新. namenode.blockReceive()通过,

调用FSNameSystem.addStoreBlock()方法将数据块信息更新到blockmap中。如果是数据块复

制产生的,则可以通过delNodeHint删除源节点上的数据块副本,其次,当数据节点上报数。

据块信息时, namenode通过其更新blockmap中的信息..

2.删除数据块副本,

数据块副本的删除包括以下几种情况:

1),数据块所属的文件被删除,

2),数据块副本数多余副本系数,

3).数据块副本已经损坏:

第一种情况:在删除目录上的一个文件时,通过FSDirectory.UnprotectedDelete()方法,

客户端通过ClientProtocol.delete(String, boolean)方法来删除文件,最终实现是NameNodeRpcServer.delete(String, boolean)方法。最底层:之后调用了FSNamesystem的.delete()来删除namesystem中的相应的文件。具体是通过INodeDirectorv.delete()方实现.NodeDirectory.delete()是通过调用FSDirectory.UnProtectedDelete()方法实现,UnProtectedDelete具体过程首先规范化路径,获取从根节点到要删除节点之间的INODE,然后在父节点上调用removeChild方法删除该INode对象。接下来会通过INode.CollectSubTreeblocksAndClear()方法,获得被删除节点为根节点的子目录下的所有文件拥有的数据块,把这些数据块放入一个ArrayList<Block> blocklist中,最后将规范化路径和,blocklist以参数的形式传入FSnameSystem.removePathAndBlocks中, 最后通过,FSnameSystem.removePathAndBlocks方法,删除所有数据块和可能的租约。具体代码见p364.removePathAndBlocks的具体过程是对于每个要删除的block,首先在blockmap中将其删除:由于数据块已经被删除,因此即使有副本损坏,也无需复制,所以将corruptReplicated中删除。最后调用addTolnvalidates方法在所有拥有副本的数据节点上删除数据块,其实也只是,简单的将待删除的数据块副本信息添加到recentInvalidateSets中。 (在DataNode上是通过FSDataSet.invalidates()方法,对于每一个数据块,通过异步方式删除P285).

第二种:对于副本的删除:

移除多余副本是使用processOverReplicatedBlock()方法,该方法主要是选择出候选数据节点集合,首先需要选择候选数据节点,将其保存在nonExcess中,对于首选数据节点必须要满足三个条件:

1).该节点信息不在excessReplicatedMap中,

2)该节点不是一个处于正在撤销或者已撤销的数据节点,

3)该节点保存的副本不是损坏副本,

其次,通过chooseExcessReplicates()在候选节点集合中选择正确的节点,执行删除。

chooseExcessReplicates使用两个原则选择并删除多余数据块副本:尽量不在同一个机架上放,置同一个副本;尽量删除比较忙的数据节点上的副本。在chooseExcessReplicates方法选择,出后会将其添加到FSNameSystem.excessBlocks中,并将副本信息添加到recentinvalidatedSets中,对比processOverReplicatedBlock和removePathAndBlocks,最终都是讲需要删除的副本放在recentinvalidateSet中,但是processOverReplicatedBlock还会保留在excessBlocks中。第三种情况:对于损坏的副本,客户端读文件或者DataNode的数据块扫描器都可能会发现损坏的副本,他们会将检测到的结果上报给名字节点,最终由markBlockAsCorrupt()进行处理.

以上三种情况,最终都只是将要删除的数据块放入FSNameSystem.recentlnvalidateSet中,那什么时候才会进入DataNodedescriptor中的invalidateBlocks中,并通过名字节点指令在数据节点删除呢?在namenode中有一个线程ReplicatedMonitor会周期性的将recentinvalidateSet中的数据块放入对应的DataNodedescriptor的invalidateBlocks中,并在recentinvalidateSet中删除。

由于在删除多余副本中,要删除的数据块副本还被保存在excessBlocks中,当数据节点,周期性的通过心跳汇报它拥有的数据块时,当namenode发现一些数据块不存在时会在excessBlocks中删除。



3.数据块复制:在数据块复制时,包括四个步骤:参数检查、选择复制的源数据节点、目标数据节点、生成复制请求。

1)参数检查包括如果数据块不属于任何一个文件或者已经打开的文件,则不需要复制;选择数据源或者目标节点时没找到合适的节点,

副本数满足,不需要复制!

2)选择源数据节点有两个原则:尽量选择不忙的节点:如果副本处于正在撤销的节点,优先选择(因为写请求少)

3)选择目的节点:

第一个尽量选择和源在同一个机架上的节点!

第二个选择其他机架的节点.

第三个,若上述两个位于同一个机架,选择其他机架的一个节点,否则选择和源节点相同机架

4)生成复制请求:在生成复制请求是会先通过underReplicateBlocks方法计算数据块优先,级,在将其生成复制请求放入pendingReplication中,并将复制信息写入DataNodeDscritor中. namenode会周期性的检查这些复制请求是否完成,超时未完成的会重新将其加入到neededReplications中重新计算优先级生成复制请求。

### Fsimage与EditLog?Fsimage与EditLog的合并过程

Fsimage文件包含了整个文件系统所有的文件和目录,是文件系统元数据的持久性检查,点,当namenode重启后都需要载入fsimage进入内存,恢复到某一个检查点,再执行检查点后的编辑日志,进行重建。inode是序列化信息,是每个文件或者且录的元数 的内部描,述方式。而在该检查点之后的所有改动则保存在editlog中,Fsimage并不描述DataNode, namenode将这种块映射关系放在内存中。DataNode启,动的通过注册上报。

Fsimage与EditLog合并过程:

1.secondarynamenode通过周期性(五分钟)通过getEditLog获取editlog的大小,当其,达到合并的大小时通过RollEditLog方法进行合并,

2.Namenode停止使用editlog文件,并生成一个新的临时的edit.new文件。

3.Secondarynamenode通过namenode内建的Http服务器,以get的方式获取editlog与

fsimage文件。Get方法中携带者fsimage与editlog的路径

4.Secondarynamenode将fsimage载入内存并逐一执行editlog中的操作,

5·执行结束后,会向namenode发送http请求,通知namenode合并结束, namenode

通过http get的方式获取新fsimage.chk文件。

6.Namenode更新fsimage文件中的记录检查点执行的时间,并改名为fsimage文件7.Edit.new文件更名为edit文件。

注:由此可知namenode与secondarynamenode有着相似的内存需求,因为secondarynamenode也会将fsimage载入内存,因此secondarynamenode需要运行在一台专用机器上。

1,检查点的创建触发条件收两个配置参数控制: secondarynamenode每隔一小时创建-此检查点:当编辑日志达到64M时,即使未到一小时也会创建检查点,系统每五分钟检查,一次编辑日志的大小,

2.Secondarynamenode在为namenode创建检查点的同时也会保留一份检查点数据,在,previous.checkpoint中,作为namenode的备份. Previous.checkpoint, secondarynamenode的current目录、namenode的current目录结构相同,便于namenode故障时从,secondarynamenode恢复数据。有两种方式:

1).将相关目录复制到新namenode中,

2)启动namenode守护进程,将secondarynamenode作为namenode. Namenode目录结构如下:

-current

\- VERSION

editstsimagefstime

#### HDFS namenode的HA的实现?

HA的namenode主要分为共享editLog机制和ZKFC对NameNode状态的控制。

1,集群中存在多个namenode,这些namenode都有状态,处于active或者standby状态,

2.各个namenode之间通过共享文件系统存储编辑日志文件. active master将信息写入,共享存储系统,而standby master则读取该信息以保持与active master的同步,从而减少切换时间。

3.DataNode同时需要向各个namenode发送数据块处理报告.

4.每个namenode运行着一个轻量级的故障转移控制器ZKFC,用于监视和控制NN进程。基于Zookeeper实现的切换控制器ZKFC进程, 主要由两个核心组件构成:ActiveStandbyElector和HealthMonitor,其中, ActiveStandbyElector负责与zookeeper集群交互,通过尝试获取全局锁,以判断所管理的master进入active还是standby状态:HealthMonitor负责监控各个活动master的状态,以根据它们状态进行状态切换。

HealthMonitor初始化完成之后会启动内部的线程来定时调用对应NameNode的HAServiceProtocol RPC接口的方法,对NameNode的健康状态进行检测.

HealthMonitor 如果检测到 NameNode的健康状态发生变化, 会回调ZKFailoverController注册的相应方法进行处理。会先使用ActiveStandbyElector来进行自动的主备选举。

如果ZKFailoverController判断需要进行主备切换,

ActiveStandbyElector与Zookeeper进行交互完成自动的主备选举。

ActiveStandbyElector在主备选举完成后,会回调ZKFailoverController的相应方法来通知当前的NameNode成为主NameNode或备NameNode.

ZKFailoverController调用对应NameNode的HAServiceProtocol RPC接口的方法将NameNode转换为Active状态或Standby状态。

#### 联邦HDFS

是namenode水平扩展方案。该方案允许HDFS创建多个namespace以提高集群的扩展,性和隔离性。联邦HDFS允许每个namenode管理文件系统命名空间的一部分。每个namenode维护一个命名空间,不同namenode之间的命名空间相互独立。数据块池不在切分,因此每个DataNode需要注册到每个namenode.

HDFS的底层存储是可以水平扩展的(解释:底层存储指的是datanode,当集群存储空,间不够时,可简单的添加机器已进行水平扩展) ,但namespace不可以。当前的namespace只能存放在单个namenode上,而namenode在内存中存储了整个分布式文件系统中的元数据信息,这限制了集群中数据块,文件和目录的数目。

1,多个NN共用一个集群里DN上的存储资源,每个NN都可以单独对外提供服务

2每个NN都会定义一个存储池,有单独的id,每个DN都为所有存储池提供存储

3.DN会按照存储池id向其对应的NN汇报块信息,同时, DN会向所有NN汇报本地存,储可用资源情况

4·如果需要在客户端方便的访问若干个NN上的资源,可以使用客户端挂载表,把不同的目录映射到不同的NN,但NN上必须存在相应的目录,

由于使用多个命名空间,对于划分和管理这些命名空间,采用客户端挂载表的形式显示。客户端访问时通过该表会找到所属的自命名空间。

一个block pool由属于同一个namespace的数据块组成,每个datanode可能会存储集群中所有block pool的数据块。

每个block pool内部自治,也就是说各自管理各自的block,不会与其他block pool交流。一个namenode挂掉了,不会影响其他namenode.

某个namenode上的namespace和它对应的block pool一起被称为namespace volume.它是管理的基本单位。当一个namenode/nodespace被删除后,其所有datanode上对应的block pool也会被删除。当集群升级时,每个namespace volume作为一个基本单元进行升级。



### Hadoop处理小文件的方法？

在Hadoop中使用小文件的弊端:

(1)增加map开销,因为每个分片都要执行一次map任务, map操作会造成额外的开销

(2)MapReduce处理数据的最佳速度就是和集群中的传输速度相同,而处理小文件将,

增加作业的寻址次数

(3)浪费namenode的内存

解决方案:

1.使用SequenceFile将这些小文件合并成一个大文件或多个大文件:将文件名作为键,文本内容作为值。

2.但是如果HDFS中已经存在的大批小文件,可以使用CombinerFileinputFormat.

CombinerFilelnputFormat把多个文件打包成一个文件以便每个mapper能够处理更多的数据3.Hadoop archive,主要减少对namenode的负载

### hadoop archive?

Hadoop不适合存储小文件。在namenode中存储中这些文件的元数据。当小文件过多占用大量的namenode堆内存空间。存档文件文件可以大大降低namenode守护节点的内存压力。 Hadoop archive是将众多的小文件打包成一个har文件,允许对文件进行透明的访问,可以作为mapreduce的输入。

当多个文件被存档之后,会在存档文件中生成两个索引文件以及部分文件的集合。部分文件中包含已经链接在一起的大量原始文件的内容,我们可以通过索引文件找到包含在存档,文件中的部分文件,他的起始点以及长度。

2.不足:

1)har文件不可修改。要修改需要从新归档

2).har文件是源文件的一个副本,需要与原文件大小相同的磁盘空间。虽然源文件可以删除

3).虽然har可以减少小文件在namenode中占的空间大小,但仍受限于namenode的空间容量,可以使用联邦namenode.

4).虽然将多个小文件整理为har文件,但是作为mapreduce的输入,不是将多个小文件,作为一个split,在执行时每个源文件一个split,权威指南翻译错误。

### HDFS数据块副本状态的管理

对于每一种数据块副本的操作,fsnamesystem都有相应的成员变量保存相应的数据块和执行操作时需要的附加信息。成员变量包括:

corruptReplicas:保存损坏的数据块副本;保存在损坏副本到数据节点的映射关系,当副本出现损坏时,会调用addToCorrputReplicasMap添加到corruptReplicas中;当损坏副本删|除后会在corruptReplicas中删除相应记录。

recentInvalidateSets:无效数据块副本,即等待删除的数据块副本,比如文件删除时,该文件所有数据块都是无效的,一般来说,损坏的数据块副本也是无效的。是数据节点标识到!-组数据块的映射。

excessReplicateMap:多余副本,减低文件的副本数。多余副本是名字节点在多个副本中选择得到的。是数据节点标识到一组数据块的映射。

neededReplications:等待复制,准备生产复制请求的数据块副本,是为了让数据块的副本数满足文件的副本系数。

pendingReplications已经生成复制请求的数据块副本保存在其中,当复制请求产生后会,



相应的将数据块从neededReplications中取出放入pendingReplications中.

leaseManager:租约管理器,间接保存了处于构建状态或者恢复状态的数据块信息.underReplicatedBlocks:产生需要复制的数据块的优先级等,

#### HDFS什么时候会出现副本数量多于设定值的情况

<http://blog.csdn.net/androidushangderen/artice/detail/507610>

1.以下情况:

ReCommission节点重新上线这类操作是运维操作引起的.节点下线操作会导致大量此节点的block块在集群中大量拷贝,一旦此节点取消下线,之前已拷贝的大量块必然会成为多,余的副本块.

(此情况一般不会出现)名字节点启动时,会进入安全模式,数据节点上报数据块信息,若此时上报的数据块小于设置的阈值, namenode会复制出足够的数据块。有可能原本数据!块充足,只是网络或者其他情况,致集中数据块2金一

人为重新设置block replication副本数,还是以A副本举例.A副本当前满足标准副本数3.个此时用户张三通过使用hdfs的API方法setReplication (或hadoop fs-setrep)人为设置副本数为1.此时也会早A副本数多余2个的情况,即使说HDFS中的副本标准系数还是3个

新添加的block块记录在系统中被丢失这种可能相对于前2种case的情况,是内部因素,造成这些新添加的丢失的block块记录会在BlockManager进行再次扫描检测,防止出现过量,副本的现象.

2.多余副本块的处理分为2个子过程:

多余副本块的选出,

选出的多余副本块的处理(由名字节点在多个副本中选择得到),有两个原则:数据块副本的放置规则;尽量删除剩余空间小的节点。

### 如何知道一个数据块的副本情况

在FSNameSystem.countNodes()方法会返回一个NumbersReplicas对象、保存了存于不同,状态的副本数(正常副本数,位于撤销节点上的副本数,损坏副本数,多余副本数等).countNodes方法主要是利用FSNameSystem的corruptReplications, excessReplications等对象中保存的信息进行简单的统计.

由此便可以通过一个数据块知道其他副本的情况.

### HDFS删除多余副本的过程

<http://blog.csdn.net/androidlushangderen/article/details/50760170>

首先, nonExcess对象其实是一个候选节点的概念,将block副本块所在的节点列表进行

多种条件的再判断和剔除最后就调用到选择最终过量副本块节点的方法。

然后,在候选节点中选择节点删除冗余数据块。选择删除多余副本的节点时有两个选择

依据:副本的放置策略;选择出可用空间最少的,这个也好理解,同样的副本数的节点列表中,

当然要选择可用空间尽可能少的,以便释放出多的空间。

最后执行相应的删除方法。

BlocksMap类维护块(Block)到其元数据的映射表,元数据信息包括块所属的inode.

存储块的Datanode.

描述某些块的副本数量不足块的实体类,而且,对于块设定了优先级,通过一个优先级

队列来管理块副本不足的块的集合。

private UnderReplicatedBlocks neededReplications = new UnderReplicatedBlocks();

//描述当前尚未完成块副本复制的块的列表。

private PendingReplicationBlocks pendingReplications;

### HDFS块的复制

HDFS中对于数据块副本状态的管理都是在FSNameSystem中管理. FSNamesystem中存在多个数据结构(见上) 。等待复制的数据块信息在neededReplications中,当UnderReplications返回等待复制的数据块优先级后,生成复制请求,加入到源数据节点中(DataNodeDescriptor中),由于复制请求可能会失败,所以也会放入pendingReplications中.数据复制请求可能会超时,当超时后会将其重新插入到underReplicatedBlock中,以重新产生复制请求.

UnderReplicatedBlocks是HDFS中关于块复制的一个重要数据结构,在进行复制时,可能存在多个数据块需要复制,才是有三个优先级:

1.当复制源节点时一个等待撤销的节点或者数据块只剩一个副本的情况下优先级最高2.当副本数不到期望值得1/3时,优先级次之3·其他情况优先级最低。

### HDFS数据块复制时节点的选择

和新数据块副本的防止不同

1.第一个目标尽量和源节点在同一个机架上的另一个节点

2.第二个目标选择其他机架上的另一个节点,

3.第三个(很少见)。如果前两个在同一个机架上则避开该机架,否则选择和源同一个

机架选择一个。

### HDFS有一个数据块损坏如何处理？

有两种情况:所有数据块都损坏,部分数据块损坏在namenode中只有所有的数据块副本都损坏,才认为该数据块损坏,当部分副本出现,据害时,名字节点会进行数据块复制,直到数据块副本恢复正常.

在namenode中对于数据块副本的管理是在FSnameSystem中,其中有一个成员变量,corruptReplicats<block,CollectioneDatanodeDescriptor>> , 其中存储着损坏数据块与DataNodeDescriptor的映射。当某个数据块损坏后(DataNode可以通过数据块扫描器获知、通过心跳发送给namenode) ,会将损坏的数据块加到corruptReplicats中;当损坏数据块在,DataNode上删除后会在corrutpReplicates中删除.

当部分副本出现损害时,名字节点会进行数据块复制时,有一个数据结构叫做,underReplicatedBlocks对象。其中有list保存需要进行数据块复制的数据块,对于数据块复a是有优先级的. underReplicatedBlocks的getPriority()方法会返回数据块的优先级。等待复制的对象从underReplicatsBlocks中读取出来之后,会生成复制请求,并将请求放入DataNodeDescriptor的成员变量replicateBlock中.同时会将这个请求放入,FSnameSystem.pendingReplications中. pendingReplications是一个数据块到数据块复制信息,pendingBlockinfo的映射。 pendingBlockinfo保存了数据块复制的时间和复制副本数。如果复制没成功会被重新插入到underReplicatedBlocks对象中,重新产生复制请求

### Client在写入数据块时如果只写入一个数据块，如何处理

在hdfs中存在着数据块的复制,对于数据块的复制时分优先级的,如果某一个数据块只有一个副本,其优先级最高,会先对此类数据块进行复制

### HDFS删除一个文件的过程(源码)? (携程)

<http://m.blog.csdn.net/zhniun5965/article/detail/631827>

客户端通过ClientProtocol.delete(String, boolean)方法来删除文件,最终实现是!NameNodeRpcServer.delete(String, boolean)方法.

最底层:之后调用了FSNamesystem的delete()来删除namesystem中的相应的文件。具,体是通过INodeDirectory.delete()方实现。NodeDirectory.delete()是通过调用UNProtectedDelete()方法实现,具体过程首先获取从根节点到要删除节点之间的INODE,然后在父节点上调用removeChild方法删除该INode对象。接下来会通过INode.CollectSubTreeblocksAndClear()方法,获得被删除节点为根节点的子目录下的所有文件拥有的数据块,最后通过FSnameSystem.removePathAndBlocks方法,删除所有数据块和可能的租约。在removePathAndBlocks删除实际有三部分,对于每一个block,首先将其从corruptReplications中删除,再从blockmap中删除,最后通过addTolnvalidate方法将其添加到ResentinvalidatedSet中。(在blockMap中已经删除,那之后如何确定该block是在哪个DataNodeDescriptor中的呢? ) ResentinvalidatedSet中的元素是kv对,保存着block与DataNodeDescriptor的映射

在namenode中有一个服务线程Monitor,会周期性的将ResentinvalidatedSet中的的block放入对应的DataNodeDescriptor中的invalidateBlocks中,这时, recentinvalidatedSet中的block会被删除。在数据节点上报心跳的处理过程中生成名字节点指令。当DataNode通过心跳收到namenode的指令后,会通过调用FSDataSet.invalidate()方法将相应的数据块删除。



当DataNode收到心跳后,通过switch来判断传过来的是哪种命令,最终进入了FsDataset.invalidate(String, BlockI)方法来从磁盘删除具体的数据块。首先会删除数据块信息和meta信息,删除之后调用 datanode.notifyNamenodeDeletedBlock(block,volume.getStoragelD();向namenode报告最近删除的数据块.

注意:

HDFS中的数据删除也是比较有特点的,并不是直接删除,而是先放在一个类似回收站的地方(/trash) ,可供恢复。

对于用户或者应用程序想要删除的文件, HDFS会将它重命名并移动到/trash中,当过了一定的生命期限以后, HDFS才会将它从文件系统中删除,并由Namenode修改相关的元数据信息。并且只有到这个时候, Datanode上相关的磁盘空间才能节省出来,也就是说,当用户要求删除某个文件以后,并不能马上看出HDFS存储空间的增加,得等到一定的时间周期以后(现在默认为6小时) 。

对于备份数据,有时候也会需要删除,比如用户根据需要下调了Replicaion的个数,那么多余的数据备份就会在下次Beatheart联系中完成删除,对于接受到删除操作的Datanode来说,它要删除的备份块也是先放入/trash中,然后过一定时间后才删除。因此在磁盘空间的查看上,也会有一定的延时。

那么如何立即彻底删除文件呢,可以利用HDFS提供的Shell命令: bin/Hadoop dfsexpunge清空/trash.

BlockManager:维护整个文件系统中与数据块相关的信息及数据块的状态变化:BlocksMap在NameNode内存空间占据很大比例,由BlockManager统一管理,相比Namespace, BlockManager管理的这部分数据要复杂的多。Namespace与BlockManager之间通过前面提到的INodeFile有序Blocks数组关联到一起

### DataNode删除一个数据块的过程

原因很多:数据块损害,多余数据块,无效数据块等。

### HDFS Client删除一个文件是同步还是异步的

http://m.blog.csdn.net/zhangiun5965/article/details/76381587异步: DataNode会异步单独开启线程删除磁盘数据,



### NameNode中的租约管理

租约是名字节点给与租约持有者的,在一定时间内的权限。客户端需要不断的更新租约信息。每一个打开的文件在租约管理器中都会有一条记录,所以已经打开的文件不能再被其他客户端打开,关闭文件时需要释放租约。每一条租约记录包括客户端信息,租约最新更新时间,打开的文件.

租约管理器会定时检查租约,对于长时间没有进行租约更新的文件,名字节点会进行租,约恢复(租约恢复时针对已经打开的文件),关闭文件。租约管理器中有两个租约过期时间:软超时时间(1分钟)和硬超时时间(一小时) ,不可配置。在超过了硬超时时间后会触发租约恢复。

租约恢复过程:

1)首先在正常工作的数据流管道成员中选择一个作为恢复的主节点,其他节点作为参与,节点,并将这些信息加入到主数据节点描述符中,等待该数据节点的心跳上报;

2)名字节点触发的租约恢复会修改恢复文件的租约信息,他们的租约持有者统一

改为NN_Recovery

3)主数据节点收到指令后开始租约恢复,首先联系各个参与节点,开始租约恢复收集数

据块信息。

4)根据收集到的数据块信息,找到最小值的数据块最为数据块恢复后的长度。

5)主数据节点向名字节点重新申请数据块版本号

6)将长度和版本号通过到各个节点,同步结束后这些数据块拥有相同的大小和新的版本

号。更新主要是利用DataNode上的FSDataSet.updata()方法进行更新

7)将恢复的结果上报给namenode.

8)名字节点更新blockMap以及DataNodedescriptor中的信息等

#### HDFS如何添加和撤销数据节点

在HDFS中提供了dfs.hosts (又称为include文件)文件和dfs.exclude文件,对连接到namenode的数据节点进行管理。Include和exclude保存在FSNameSystem中的hostsReader.中Include文件:对连接到NameNode的数据节点进行管理,指定了可以连接到NameNode的数据节点列表,Exclude文件:指定不能连接到namenode的数据节点列表

1添加数据节点:

需要在include中添加相应的记录,并通过dfsadmin工具的refreshnodes命令,刷新名字节点信息,然后启动数据节点,启动时会执行握手,注册,上报相应的行为。

2·删除撤销节点:在exclude文件中添加将要撤销的节点,然后执行refreshnodes命令,名字节点就会开始撤销DataNode,被撤销的节点的数据块会被复制到集群的其他节点,这个过程中数据节点处于正在撤销的状态(可以在DataNodeDescriptor中查询这个状态) ,数据复制完成后才会转移到已撤销状态,并在include中删除相应的记录,就可以关闭相应的,数据节点,

---

## Yarn

### Yarn 的总体架构

<http://dongxicheng.org/mapreduce-nextgen/yarnmr2-resource-manager-infrastructurel>

ResourceManager主要由以下几个部分组成:

1·用户交互: YARN分别针对普通用户,管理员和Web提供了三种对外服务,分别对应,ClientRMService, AdminService和WebApp:

2.NM管理:主要包括:监控nodemanager的死活、接受nodemanager的请求、维护正常节点列表和异常节点列表。具体如下:

NMLivelinessMonitor监控NM是否活着,如果一个NodeManager在一定时间(默认为10min)内未汇报心跳信息,则认为它死掉了,会将其从集群中移除。

NodesListManager维护正常节点和异常节点列表,管理exlude (类似于黑名单)和inlude(类似于白名单)节点列表,这两个列表均是在配置文件中设置的,可以动态加载。

ResourceTrackerService处理来自NodeManager的请求,主要包括两种请求:注册和心跳,其中,注册是NodeManager启动时发生的行为,请求包中包含节点ID,可用的资源上限等信息,而心跳是周期性行为,包含各个Container运行状态,运行的Application列表、节点健康状况(可通过一个脚本设置),而ResourceTrackerService则为NM返回待释放的Container列表、Application列表等。



3.AM管理,即对AM的管理(有三部分组成):主要包括监控appmaster的死活、接受appmaster的请求、与nodemanager通信启动APPmaster,具体如下

AMLivelinessMonitor监控AM是否活着,如果一个ApplicationMaster在一定时间(默认为10min)内未汇报心跳信息,则认为它死掉了,它上面所有正在运行的Container将被认必死亡, AM本身会被重新分配到另外一个节点上(用户可指定每个ApplicationMaster的尝试次数,默认是1次)执行.

ApplicationMasterLauncher与NodeManager通信,要求它为某个应用程序启动,ApplicationMaster.

ApplicationMasterService处理来自ApplicationMaster的请求,主要包括两种请求

注册的心跳,其中,注册是ApplicationMaster启动时发生的行为,包括请求包中包含所;

节点,RPC端口号和tracking URL等信息,而心跳是周期性行为,包含请求资源的类型描述、等待待释放的Container列表等,而AMS则为之返回新分配的Container、失败的Container等信息

4.资源分配:

ResourceScheduler是资源调度器,它按照一定的约束条件(比如队列容量限制等)将年胖中的资源分配给各个应用程序,当前主要考虑内存资源,在3.0版本中将会考虑CPU(https://issues.apache.org/jira/browse/YARN-2) . ResourceScheduler是一个插拔式模块,默认是FIFO实现, YARN还提供了Fair Scheduler和Capacity Scheduler两个多租户调度器.

5.Application管理:

ApplicationACLsMa

ager管理应用程序访问权限,包含两部分权限:查看和修改,查看主要指查看应用程序基本信息,而修改则主要是修改应用程序优先级、杀死应用程序等.应用程序的启动和关闭.

RMAppManager管.

ContainerAllocationEpirerYARN不允许AM获得Container后长时间不对其使用,因为这会降低整个集群的利用率。当AM收到RM新分配的一个Container后,必须在一定的时间内(默认为10min)在对应的NM上启动该Container, 否则, RM会回收该Container.

6.安全管理

### Hadoop yarn中container的概念

htp://dongxicheng.org/mapreduce-nextgen/hadoop-ira-varn-392/

在YARN中,用户提交应用程序后,对应的ApplicationMaster负责将应用程序的资源需求转化成符合特定格式的资源请求(具体格式如下段) ,并发送给ResourceManager.一旦,某个节点出现空闲资源, ResourceManager中的调度器将决定把这些空闲资源分配给哪个应,用程序,并封装成Container返回给对应的ApplicationMaster. ApplicationMaster发送的资源,请求(ResourceRequest)形式如下:

<Priority, Hostname, Resource, #Container>

其中, Priority为资源的优先级、HostName是期望资源所在的节点,可以为节点host名、节点所在rack名或者*(任何节点均可以) , Resource为资源量, #Container为满足上.述三个属性的资源个数.

<http://dongxicheng.org/mapreduce-nextgen/understand-varn-container-concept>

在YARN中, ResourceManager中包含一个插拔式的组件:资源调度器,它负责资源的,管理和调度,是YARN中最核心的组件之一.

当向资源调度器申请资源,需向它发送一个ResourceRequest列表,其中,每个ResourceRequest描述了一个资源单元的详细需求,而资源调度器则为之返回分配到的资源描述Container,每个ResourceRequest可看做一个可序列化Java对象,包含的字段信息:

message ResourceRequestProto (

optional PriorityProto priority 1;//资源优先级

optional string resource-name = 2; //资源名称(期望资源所在的host, rack名称等)optional ResourceProto capability - 3: //资源量(仅支持CPU和内存两种资源)

optional int32 num-containers =4://满足以上条件的资源个数

optional bool relax locality = 5 ldefault = truel; //是否支持本地性松弛(2.1.0-beta.

之后的版本新增加的,具体参考我的这篇文章: Hadoop新特性、改进、优化和Bug分

析系列3: YARN-392)

}



ResoureeNanager

2: ContainersRaarastu..ResoarecReqtests



pplicationiMastes



e



0



3: StariContainerRequestsYCuntalncr LaunsbContesn



nonsxichens.ors



发出资源请求后,资源调度器并不会立马为它返回满足要求的资源,而需要应用程序的,ApplicationMaster不断与ResourceManager通信,探测分配到的资源,并拉去过来使用。-

且分配到资源后, ApplicatioMaster可从资源调度器那获取以Container表示的资源,Container

可看做一个可序列化Java对象,包含的字段信息(直接给出了Protocol Buffers定义)如下:

message ContainerProto {

optional ContainerldProto id = 1; //container id

optional NodeldProto nodeld = 2; //container (资源)所在节点,

optional string node_http_address = 3;

optional ResourceProto resource = 4; //container资源量:

optional PriorityProto priority = 5; //container优先级

optional hadoop.common.TokenProto container token = 6; //container token,用于安全认证,



一般而言,每个Container可用于运行一个任务. ApplicationMaster收到一个或多个Container后,再次将该Container进一步分配给内部的某个任务,一旦确定该任务后,ApplicationMaster需将该任务运行环境(包含运行命令、环境变量、依赖的外部文件等)连同Container中的资源信息封装到ContainertaunchContext对象中,进而与对应的NodeManager通信,以启动该任务. ContainerLaunchContext包含的字段信息(直接给出了Protocol Buffers定义)如下:

message ContainerLaunchContextProto {

repeated StringLocalResourceMapProto localResources a 1; //Container启动以来的外部资,



ptional bytes tokens = 2;

repeated StringBytesMapProto service_data = 3;

repeated StringStringMapProto environment =4; //Container启动所需的环境变量,

repeated string command -5; //Container内部运行的任务启动命令,如果是MapReduce的话, Map/Reduce Task启动命令就在该字段中,

repeated ApplicationACLMapProto application_ACLs = 6;

}

每个ContainerLaunchContext和对应的Container信息(被封装到了ContainerToken中)你再次被封装到StartContainerRequest中,也就是说, ApplicationMaster最终发送给,NodeManager的是StartContainerRequest,每个StartContainerRequest对应一个Container和,任务。



总结上述可知, Container的一些基本概念和工作流程如下:

(1) Container是YARN中资源的抽象,它封装了某个节点上一定量的资源(CPU和内存两类资源) 。它跟Linux Container没有任何关系,仅仅是YARN提出的一个概念(从实,现上看,可看做一个可序列化/反序列化的Java类).

(2) Container由ApplicationMaster向ResourceManager申请的,由ResouceManager中的资源调度器异步分配给ApplicationMaster:

(3) Container的运行是由ApplicationMaster向资源所在的NodeManager发起的,Containeri运行时需提供内部执行的任务命令(可以使任何命令,比如java, Python. C+进程启动命令均可)以及该命令执行所需的环境变量和外部资源(比如词典文件、可执行文件、jar包等).

另外,一个应用程序所需的Container分为两大类,如下:

(1)运行ApplicationMaster的Container:这是由ResourceManager (向内部的资源调度器)申请和启动的,用户提交应用程序时,可指定唯一的ApplicationMaster所需的资源:

(2)运行各类任务的Container:这是由ApplicationMaster向ResourceManager申请:的,并由ApplicationMaster与NodeManager通信以启动之.

### 简述yarn中的资源调度器? 

<http://www.cnblogs.com/BYRans/p/5567650.html>

YARN自带了三种常用的调度器,分别是FIFO, Capacity Scheduler和Fair Scheduler.1.Fair Scheduler允许YARN应用程序在一个大的集群上公平地共享资源。

公平调度是一种为应用程序分配资源的方法,多用户的情况下,强调用户公平地使用资,源。默认情况下Fair Scheduler根据内存资源对应用程序进行公平调度,通过配置可以修改为根据内存和CPU两种资源进行调度。当集群中只有一个应用程序运行时,那么此应用程序占用这个集群资源。当其他的应用程序提交后,那些释放的资源将会被分配给新的应用程,序,所以每个应用程序最终都能获取几乎一样多的资源。

在Fair Scheduler中,不需要预先占用一定的系统资源, Fair Scheduler会动态调

应用

i的资源分配。例如,当第一个大job提交时,只有这一个job在运行,此时它获得了所有集群资源:当第二个小任务提交后, Fair调度器会分配一半资源给这个小任务,让这两个任务公平的共享集群资源。

需要注意的是,从第二个任务提交到获得资源会有一定的延迟,因为它需要等待第一个,住爷释放占用的Container,小任务执行完成之后也会释放自己占用的资源,大任务又获得,i全部的系统资源。

Fair Scheduler将应用程序支持以队列的方式组织,这些队列之间公平的共享资源。默,认,所有的用户共享一个队列。如果应用程序在请求资源时指定了队列,那么请求将会被提交到指定的队列中。也可以通过配置,根据用户名称来分配队列。在每个队列内部,应用程序基于内存公平共享或FIFO共享资源.

举个例子,假设有两个用户A和B,他们分别拥有一个队列。当A启动一个job而B没,有任务时, A会获得全部集群资源;当B启动一个iob后, A的job会继续运行,不过一会,

个任务会各自获得一半的集群资源。如果此时B再启动第二个job并且其它job还,

1之后在运行. 1

1它将会和B的第一个job共享B这个队列的资源,也就是B的两个job会用于四,

)集群资源,而A的job仍然用于集群一半的资源,结果就是资源最终在两个用户之,

分之一

喜的共享。

间平

2.Capacity调度器:

Capacity调度器允许多个组织共享整个集群,每个组织可以获得集群的一部分计算能,力,通过为每个组织分配专门的队列,然后再为每个队列分配一定的集群资源,这样整个集群就可以通过设置多个队列的方式给多个组织提供服务了。除此之外,队列内部又可以,垂直划分,这样一个组织内部的多个成员就可以共享这个队列资源了,在一个队列内部,资源的调度是采用的是先进先出(FIFO)策略.

一个iob可能使用不了整个队列的资源。然而如果这个队列中运行多个job,如果这个,队列的资源够用,那么就分配给这些job,如果这个队列的资源不够用了呢?其实Capacity调度器仍可能分配额外的资源给这个队列,这就是弹性队列(queue elasticity)的概念.

在正常的操作中, Capacity调度器不会强制释放Container,当一个队列资源不够用时,这个队列只能获得其它队列释放后的Container资源。当然,我们可以为队列设置一个最大资源使用量,以免这个队列过多的占用空闲资源,导致其它队列无法使用这些空闲资源,这就是弹性队列需要权衡的地方.

1. Yarn中AppMaster向ResourceMaster申请资源的过程

1当APPmaster获得任务的切片信息以后,会创建相应数量的map任务与reducer任务..这些任务会被封装成resourceRequest列表,其中每个resourceRequest代表一个资源请求单i.

2.APPmaster向resourcemananger申请资源时会向m发送这个resourceRequest列表。RM会返回分配到的资源描述container, resourceRequest在实现上就是一个可序列化反序列化的Java对象,包含一些申请的资源信息:是否资源本地松弛,资源优先级,资源量等等。

3.APPmaster发送请求后,RM不会立即返回给APPmaster,所分配的资源时在APPmaster.a时自动拉取。一旦资源分配之后, APPmaster则会在RM获取表示资源的Container..Container实际也是一个Java对象,包含分配的资源信息:资源节点信息,资源量等等,每个container运行一个任务。

4.APPmaster获得container后会将其与任务信息封装为一个containerLounchContext对象。

5.ContainerLounchContext对象与container信息再次封装为startContainerRequest对象.ApPmaster向nodemanager发送startContainerRequest对象,每个StartContainerRequest对,应一个Container和任务.

注:本地松弛:当一个节点出现空闲资源时,调度器按照调度策略应将该资源分配给jobl,但是job1没有满足locality的任务,考虑到性能问题, i度器暂时跳过该作业,而将空闲资源分配给其他有locality任务的作业,今后集群出现空闲资源时, iob1将一直被跳过,知道它有一个满足locality的任务,或者达到了管理员事先配置的最长跳过时间,这时候不得不将资源分:配给job1 (不能让人家再等了啊,亲),从上面描述可知道, ApplicationMaster申请的node1.上的资源,最终可能得到的是node2上的资源,这在某些应用场景下,是不允许如此操作的.

#### yarn中的ApplicationMaster故障后如何处理？

MRAPPMaster向resourcemanager发送周期性的心跳,当resourcemanager发现!MRAPPMaster故障时,会在一个新的容器(由节点管理器管理)开始一个新的MRAPPMaster.实例,新的MRAPPMaster实例可以恢复故障任务的状态,使其不必重复运行,默认是不可以恢复,可设置。客户端通过心跳局期性的向MRAPPMaster获取进度轮训,因此客户端会重新定位实例位置.

客户端定位MRAPPMster的过程为:在作业初始阶段,客户端会向resourcemanager询!问并缓存MRAPPMster的位置,使其每次想MRAPPMster查询时不需要重载resourcemanager,但是当mrappmaster运行失败,客户端不能获得进度状态时,会重新向resourcemanager询问。

.节点管理器运行失败:节点管理器会停止向resourcemanager发送心跳,并被移除可用节点管理器池

对于resourcemanager,可以通过zookeeper实现HA,避免一个resourcemanager出现单点故障,其脑裂问题是由zookeeper的ACL来实现的. YARN的单点故障指的是,ResourceManager单点问题, ResourceManager负责整个系统的资源管理和调度,内部维护了各个应用程序的ApplictionMaster信息, NodeManager信息,资源使用信息等。考虑到这些信息绝大多数可以动态重构,因此解决YARN单点故障要比HDFS单点容易很多,与HDFS类似, YARN的单点故障仍采用主备切换的方式完成,不同的是,备节点不会同步主节点的信息,而是在切换之后,才从共享存储系统读取所需信息。之所以这样,是因为YARN ResourceManager内部保存的信息非常少,大部分可以重构,且这些信息是动态变化的,很快会变旧。

#### hadoop yarn常见问题和解决方案

(1)默认情况下,各个节点的负载不均衡(任务数目不同),有的节点很多任务在跑,有的没有任务,怎样让各个节点任务数目尽可能均衡呢?

答:默认情况下,资源调度器处于批调度模式下,即一个心跳会尽可能多的分配任务,这样,优先发送心跳过来的节点将会把任务领光(前提:任务数目远小于集群可以同时运行的任务数量) ,为了避免该情况发生,可以按照以下说明配置参数:

如果采用的是fair scheduler,可在yarn-site.xml中,将参数yarn.scheduler.fair.max.assign设置为1(默认是-1,)

如果采用的是capacity scheduler (默认调度器),则不能配置,目前该调度器不带负载均衡之类的功能.

当然,从hadoop集群利用率角度看,该问题不算问题,因为一般情况下,用户任务数目要远远大于集群的并发处理能力的,也就是说,通常情况下,集群时刻处于忙碌状态,没有节点一直空闲着.

(2)某个节点上任务数目太多,资源利用率太高,怎么控制一个节点上的任务数目?



答:一个节点上运行的任务数目主要由两个因表决定,一个是NodeManager可使用的资源总量,一个是单个任务的资源需求量,比如一个NodeManager上可用资源为8GB内存.8cpu.单个任务资源需求量为168内存. Icpu,财该节点最多运行8个任务.

NodeManager上可用资源是由管理员在配置文件varn-site.xml中配置的,相关参数如下。yarn.nodemanager.resource.memory-mb:熟的可用物理内存量、默认是8096varn.nodemanager.resource.cpu-vcores:总的可用CPU数目,默认是8

对于MapReduce而言,每个作业的任务资源量可通过以下参数设置:

mapreduce.map.memory.mb:物理内存量,默认是1024

mapreduce.map.cpu.vcores: CPU数目,默认是1

注:以上这些配置属性的详细介绍可参考文章: Hadoop YARN配置参数剖析(1)--RM与

默认情况,各个调度器只会对内存餐源进行调度,不会考虑CPU资源,你需要在调度

NM相关参数.

器配置文件中进行相关设置,

(4) 用户给任务设置的内存量为1000MB,为何最终外配的内存却是1024MB?答:为了易于管理资源和调度资源, Hadoop YARN内置了资源规整化算法,它规定了最小可申请资源量、最大可申请资源量和资渡规整化因子,如果应用程序申请的资源量小于最小可申请资源量,则YARN会将其大小改为最小可申请量,也就是说,应用程序获得资源不会小于自己申请的资源,但也不一定相等:如果应用程序申请的资源量大于最大可申请资源!量,则会抛出异常,无法申请成功:规整化因子是用来规整化应用程序资源的,应用程序申!请的资源如果不是该因子的整数倍,则将被修改为最小的整数倍对应的值,

资源分配模型:

在YARN中,用户以队列的形式组织,每个用户可属于一个或多个队列,且只能向这些!队列中提交application.每个队列被划分了一定比例的资源.

YARN的资源分配过程是异步的,也就是说,资源调度器将资源分配给一个application后,不会立刻push给对应的ApplicaitonMaster,而是暂时放到一个缓冲区中,等待;ApplicationMaster通过周期性的RPC函数主动来取,也就是说,采用了pull-based模型,而!不是push-based模型,这个与MRv1是一致的.



### Yarn的内存和cpu的资源调度和资源隔离机制

<http://dongxicheng.org/mapreduce-nextgen/yarnmr2-resource-manager-resource-manager/>

资源调度和资源隔离是YARN作为一个资源管理系统,最重要和最基础的两个功能。资源调度由ResourceManager完成,而资源隔离由各个NodeManager实现,

Hadaop YARN同时支持内存和CPU两种资源的调度(默认只支持内存,如果想进一步调度CPU,需要自己进行一些配置) ,本文将介绍YARN是如何对这些资源进行调度和隔离的.

在YARN中,资源管理由ResourceManager和NodeManager共同完成,其中,ResourceManager中的调度器负责资源的分配,而NodeManager则负责资源的供给和隔离.ResourceManager将某个NodeManager上资源分配给任务(这就是所谓的“资源调度”)后,NodeManager需按照要求为任务提供相应的资源,甚至保证这些资源应具有独占性,为任务运行提供基础的保证,这就是所谓的资源隔离.

YARN的资源管理器实际上是一个事件处理器,它需要处理来自外部的6种,SchedulerEvent类型的事件,并根据事件的具体含义进行相应的处理,这6种事件含义如下:

(1) NODE REMOVED

事件NODE REMOVED表示集群中被移除一个计算节点(可能是节点故障或者管理员主动移除) ,资源调度器收到该事件时需要从可分配资源总量中移除相应的资源量.

事件NODE-ADDED表示集群中增加了一个计算节点,资源调度器收到该事件时需要物

2) NODE ADDED

新增的资源量添加到可分配资源总量中。

事件APPLICATION-ADDED表示ResourceManager收到一个新的Application,通常而言.

(3) APPLICATION ADDED

资源管理器需要为每个application维护一个独立的数据结构,以便于统一管理和资源分配。资源管理器需将该Application添加到相应的数据结构中.

事件APPLICATION-REMOVED表示一个Application运行结束(可能成功或者失败),资

(4) APPLICATION REMOVED

源管理器需将该Application从相应的数据结构中清除.

当资源调度器将一个container分配给某个ApplicationMaster后, 如果该

(5) CONTAINER EXPIRED

ApplicationMaster在一定时间间隔内没有使用该container.则资源调度器会对该containg进行再分配.

NodeManager通过心跳机制向ResourceManager汇报各个container运行情况,会触发

(6) NODE UPDATE

一个NODE-UDDATE事件,由于此时可能有新的container得到释放,因此该事件会触发货!源分配,也就是说、该事件是6个事件中最重要的事件,它会触发资源调度器最核心的资源

YARN对内存资源和CPU资源采用了不同的资源隔离方案.

分配机制.

对于内存资源,对于YARN而言, 目前所做的工作是监控每个任务的进程树,如果每个!任务的进程树使用的总物理内存或者总虚拟内存量超过了预先设置值,则依次发送TERM和KILL两个信号将整个进程树杀死. YARN并没有显式地对内存资源进行强制隔离,以免在产

为了能够更灵活的控制内存使用量, YARN采用了进程监控的方案控制内存使用,即每

生内存抖动时。

个NodeManager会启动一个额外监控线程监控每个container内存资源使用量,一旦发现它超过约定的资源量,则会将其杀死。采用这种机制的另一个原因是Java中创建子进程采用了fork()-exec()的方案,子进程启动瞬间,它使用的内存量与父进程一致,从外面看来..个进程使用内存量可能瞬间翻倍,然后又降下来,采用线程监控的方法可防止这种情况下导致swap操作,对于CPU资源,则采用了Cgroups进行资源隔离.

Cpu的隔离:

CPU被划分成虚拟CPU (CPU virtual Core) ,这里的虚拟CPU是YARN自己引入的概念,初衷是,考虑到不同节点的CPU性能可能不同,每个CPU具有的计算能力也是不一样的,比如某个物理CPU的计算能力可能是另外一个物理CPU的2倍,这时候,你可以通过为第一个物理CPU多配置几个虚拟CPU弥补这种差异。用户提交作业时,可以指定每个任务需:要的虚拟CPU个数.

默认情况下, NodeManager不会对CPU资源进行任何隔离,你可以通过启用Cgroups让你支持CPU隔离。

(1) CPU资源按照百分比进行使用和隔离,

(2)限制每个container的CPU资源使用上限。上一一种CPU隔离方式能够保证每个,Contaienr的CPU使用下限,大部分情况下,可能拿到比自己期望的多的CPU资源;而这种隔离则不同,它会严格限制cpu使用上限,比如你希望使用2个CPU,则会限制你只能使用!2个,不能多用,即使同机器上仍有大量空闲CPU资源,也不会允许你使用。





---



1. Hadoop作业调优

1.map作业数量2.Reduce作业数量,3.Shuffle参数调整,4·中间值压缩,注意shuffle与reduce的输出的压缩方式不同。

5.Combiner

1. Combiner ()函数何时被使用?

1.map端内存缓冲区达到阈值溢出成spill文件时

2.将多个spill文件合并成一个大的文件时

1. 以MR任务为例，讲一下YARN的整个过程

Yarn中主要角色包括: resourcemanager, ApplicationMaster, NodeManager

Resourcemanager: 启动每一个 Job所属的 ApplicationMaster、另外监控ApplicationMaster以及nodemanager的存在情况,并且负责协调集群上计算资源的分配。ApplicationMaster:每个job有一个ApplicationMaster,负责运行mapreduce的任务,并负责报告任务状态

NodeManager(负责启动和管理节点中的容器)。

1).客户端向resourcemanager发送job请求,客户端产生Runjar进程与resourcemanager通过rpc通信

2).resourcemanager向客户端返回job相关资源的提交路径以及jobID

3).客户端将job相关的资源提交到相应的共享文件系统的路径下。

4)客户端向resourcemanager提交job

5).resourcemanager通过调度器在nodemanager创建一个容器,并在容器中启用!MRAppMaster进程(该进程由resourcemanager启动) 。

6)该MRAppMaster进程对作业进行初始化,创建多个对象对作业进行跟踪。

7)MRAppMaster从共享文件系统中过得计算得到的输入分片(只获取分片信息,不需要,iar等相关资源) ,为每一个分片创建一个map以及指定数量的reduce对象。之后MRAppMaster决定如何运行构成mapreduce作业的各个任务,如果作业很小,则与MRAppMaster在同一个JvM上运行

8)若作业很大,则MRAppMaster会为所有map任务和reduce任务向resourcemanager发起申请容器资源请求。请求中包含了map任务的数据本地化信息以及输入分片等信息。

9)Resourcemanager为任务分配了容器之后, MRAppMaster就通过与nodemanager通信启动容器, 由MRAppMaster负责分配在哪些nodemanager上运行map (即yarnchild进程)和reduce任务。

10)运行map和reduce任务的nodemanager从共享文件系统中获取job相关资源,包括iar文件,配置文件等。

11)运行map和reduce任务

12).关于状态的检测与更新不经过resourcemanager:任务周期性的向MRAppMaster汇报状态及进度,客户端每秒钟通过查询一次MRAppMaster获取状态更新信息。

注意：

http://dongxicheng.org/mapreduce-nextgen/yarnmrv2-resource-manager-resource-manager/1.由于resourcemanager负责资源的分配, 当NodeManager启动时,会向ResourceManager注册,而注册信息中会包含该节点可分配的CPU和内存总量

YARN的资源分配过程是异步的,也就是说,资源调度器将资源分配给一个application后,不会立刻push给对应的ApplicaitonMaster, 而是暂时放到一个缓冲区中,等待ApplicationMaster通过周期性的RPC函数主动来取。



1. MR对1T数据全排序
2. namenode的高可用实现
3. namendoe高可用中editlog的同步过程

在HA的namenode中,存在多个namenode,其中只有一个处于active状态, active的namenode主要是通过zookeeper选择产生。对于editlog会将其放入到qjournal集群之上,qjournal是利用zookeeper实现,从namenode会在editlog上注册监听,当active namenode修改editlog后,从namenode会通过监听机制感知到,会将editlog的改变同步到本地

1. secondarynamenode的作用
2. namenode底层对整个文件系统命名镜像的抽象
3. HDFS是如何容错和存储信息的?

HDFS容错包括很多方面:

1·写入数据块数据时出错,当数据写入过程中客户端异常退出时,同一数据块的不同副本可能存在不一致的状态(租约实现) ; 

2.客户端与namenode中某一个实体出现奔溃(租约) ; 

3.当客户端与各个DataNode建立连接,发送数据之前的过程中出错

HDFS包括namenode secondarynamenode DataNode.

其中名字节点维护整个文件系统的文件目录树、文件/目录的元信息以及文件数据块的索引信息。这些信息以两种形式存储: fsimage与editlog. Fsimage保存着某一时刻文件系统的文件目录树、文件/目录的元信息以及文件数据块索引,之后的修改保存在editlog中。注意与数据节点相关的信息并不保存在名字节点的本地文件系统中,而是系统启动时,名字节点动态重建这些信息。

Secondanamenode:定期合并fsimage与editlog,不记录任何HDFS的实时变化。

1. 

1. hadoop与spark的区别

Hadoop适合用于离线数据处理,不适合处理实时数据处理。Hadoop将中间结果输出到磁盘,大量的10操作。Spark允许中间结果写入内存。

Hadoop为开发者提供了map, reduce原语,使并行批处理程序变得非常地简单和优美。Spark提供的数据集操作类型有很多种,不像Hadoop只提供了Map和Reduce两种操作。

从容错方面、shuffle方面、速度方面、使用场景方面回答!

1. Hadoop中yarn的任务调度算法和任务队列

Yarn中的任务调度算法有公平调度、容量调度、先进先出

1. 介绍yarn框架

YARN总体上仍然是Master/Slave结构,在整个资源管理框架中, ResourceManager为Master, NodeManager为Slave, ResourceManager负责对各个NodeManager上的资源进行统一管理和调度。当用户提交一个应用程序时,需要提供一个用以跟踪和管理这个程序的ApplicationMaster,它负责向ResourceManager申请资源,并要求NodeManger启动可以占用一定资源的任务。由于不同的ApplicationMaster被分布到不同的节点上,因此它们之间不会相互影响。

YARN主要由ResourceManager、NodeManager, ApplicationMaster构成。

在yarn框架的资源分配与任务调度过程见Hadoop具体章节。

1. combiner函数的使用发生在哪个阶段，以及其使用的条件是什么?Partition的使用阶段？

map端将处理的数据写入环形缓冲区,写到一定阈值之后溢出为spill文件,在溢出之前会先用partition函数进行按照key键分区排序,排序之后调用combiner函数处理。

然后将spill文件写入磁盘。一个map可能会有多个spill文件,会将多个spill文件合并为一个输出文件(可以设置) ,在输出文件输出到磁盘之前会调用,若果至少有三combiner函数。

1. 简述HDFS中的接口类型？

HDFS中的接口主要包括三种类型:

客户端相关接口ClientProtocol (客户端与名字节点) 、ClientDataNodeProtocol (客户端

与数据节点之间)服务器端相关接口: DataNodeprocotol (数节点与名字节点) 、IntelDataNodeProtocol(数据节点之间) 、NameNodeProtocol (名字节点与第二名字节点)

安全相关接口HDFS不同角色之间的通信方式有三种: RPC、 http. socket.

1. HDFS clientProtocol接口

具体参见: D:\学习总练面试问题附件Hadoop!HDFS源码HDFS中与ClientProtocol相关。重要! !

客户端与名字节点的通信协议定义在这个接口中,主要有两种:与Hadoop文件系统相关:一种是与查询、设置HDFS状态等相关.

与Hadoop文件系统相关: clientProtocol接口中定义的与元数据操作相关的方法与,Hadoop抽象文件系统中定义的方法有对应关系,客户端读数据时涉及的操作较少,主要包!括open方法, getBlockLocations (返回LocatedBlocks对象)和reportBadBlocks两个方法.写数据涉及方法较多:用于打开文件的create, append,用于追加数据块的addblock方法!和abandonBlock方法,持久化数据的方法fsync,关闭文件的方法complete,租约相关的方法。P229

1. HDFS DataNodeProtocol接口

具体参见: D:\学习总结面试问题\附件Hadoop\HDFS源码HDFS源码之服务器接口,

数据节点与名字节点之间的通信,数据节点是客户端,名字节点是服务器端。

Namenode维护整个文件系统的文件目录树以及文件对应的数据块信息,这些存于namenode的本地文件系统中。而数据块所在的数据节点信息是在namenode每次启动时重建的。DataNode在启动后需要像namenode汇报自己当前存储的数据块信息,后续工作过程中也会不断向namenode更新数据块信息。

1,基础类: DataNodeRegistration类和Namespacelnfo类.

DataNodelD唯一确定集群中的DataNode. DataNodeRegistration类继承自DataNodelD.同时增加了成员变量storagelnfo, storageinfo是一个类,记录的是整个HDFS集群的信息,和数据节点无关。storagelnfo包括三个成员变量: layoutversion namespacelD Time.

NameSpacelnfo继承自storagelnfo.增加了系统升级前的版本检测的成员变量,distributedupgradeVersion以及系统构架的版本号subversion.

2.握手注册以及数据块上报和心跳:

DataNode启动后过程:握手-注册-上报数据块信息 心跳,

讨程详解: DataNode启动时,会先通过versionRequest无参方法向namenode进行握手,该方法返回一个NameSpacelnfo对象,检查DataNode与namenode的构建版本号是否一致.如不同则DataNode节点退出;构建版本号即HDFS版本是否一致.

握手之后, DataNode会向namenode进行注册,通过DataNodeProtocol.register()方法.输入参数是一个DataNodeRegistration,同时返回一个DataNodeRegistration,便于DataNode后续逻辑处理;DataNodeRegistration可以用于检查数据节点标志DataNodelD和存储系统信息,storagelnfo,检查是否属于同一个HDFS集群。

注册之后, DataNode通过blockReport方法汇报其所拥有的数据块信息,两个参数,第一个参数为DataNodeRegistration,第二个是一个长整型数据,包含块的信息,而不是Block数a-若为block数组, namenode接收到后需创建大量block对象. Block对象有三个长整形成员变量,可以把他们放在一个长整型数组中,这样namenode就不需要创建大量block对象,减少namenode的负担. blockReport方法返回一个DataNodeCommand对象,其携带名字节点的指令,名字节点就是通过DataNodeCommand对象在namenode与DataNode之.间传递命,是所有名字节点对数据节点指令的基类,

上报之后DataNode通过sendHeartBeat方法与namenode通过心跳进行通信,携带自身,信息的datanodeRegistration对象以及当前DataNode的状态,返回一个DataNodeCommand对象.

3.与数据块操作相关的方法,

reportBadBlock()方法。数据块存储在数据节点上,可能由于各种原因导致数据块损坏.HDFS使用循环冗余校验进行错误检验。reportBadBlock上报的是一组LocatedBlocks对象,其包括了block及其位置信息。名字节点收到后根据其删除错误的数据块。

DataNodeProtocol.blockReceived() (重要)方法向namenode报告已经完整的接受了一些数据块。接受的数据源既可以是HDFS客户端也可以是其他DataNode.第一个参数是标识了,数据节点的DataNodeRegistration对象,第二个参数是Block数组. blockRecieved()方法可以一次上报多个已完整接受的数据块。在正常的数据块写入的过程中,数据节点只需要使用blockRecieved方法向namenode确认数据块的接受情况,不涉及其他接口,是比较忙的一个DataNodeprotocol方法。在数据写入的过程中,会向名字节点上报正在写状态的数据块状态,已帮助名字节点进行租约恢复。

1. HDFS InterDataNodeProtocol接口

数据节点与数据节点之间的接口,通过这个接口进行数据块恢复,保证数据的一致性.

1. HDFS NameNodeProtocol接口

调用者有两个,一个是secondarynamenode,一个是均衡器,

1.均衡器使用namenodeprotocol中的getBlocks方法和getBlocksKey方法.

getBlocks可以获取一个数据节点上的一系列的数据块及其位置信息,均衡器根据这些,信息将数据块从该节点移动的其他节点,以平衡各个数据节点数据块数量.

2.secondarynamenode:

getEditlLogsize获得编辑日志文件的大小,如果很小,则不需要合并,

第二命名节点通过rollEditLog通知命名节点开始一个合并操作,此时namenode停止,editlog的使用并创建一个新的editlog, secondarynamenode通过Http的流式接口获取待合并的fsimage与editlog之后进行合并,合并之后secondarynamenode通过http并上传元数据换像。合并结束后远程无参的rollslmage方法被调用, namenode进行一些必要的处理.

Namenode有内建的Http服务器,当需要下载fsimage和editog时. secondarynamenode合并.使用内建的http服务器的get操作获取数据,合并完成后通知namenode, namen的方式从secondarynamenode下载新的fsimage.

当客户端需要读数据时,需要与DataNode建立TCP连接,由于TCP是基于字节流的.

1. HDFS clientDataNodeProrocol接口

该接口只有三个方法: recoverBlock(),getBlockinfo().getBlockLocalinfo().

RecoverBlock():用于客户端在向数据块写入数据的过程中,如果某个副本所在数据节点

出现错误,调用此方法。从正常的数据节点中找到恢复点,然后才继续输出数据。

1. HDFS的安全模式

安全模式是HDFS的一种工作状态,处于安全模式的状态下,只像客户端提供文件的只,读视图,不接受对命名空间的修改;同时名字节点也不会进行数据块的复制或者删除,如副!本数量小于正常水平.

Namenode启动时,首先将fsimage载入内存,并执行编辑日志中的操作。一旦文件系,统元数据建立成功,便会创建一个空的编辑日志,此时namenode开始监听RPC和Http请求。但是此时namenode处于安全模式,只能接受客户端的读请求.

在安全模式下,各个DataNode会向namenode发送自身的数据块列表,当namenode.有足够的数据块信息后,便在30秒后退出安全模式。若namenode发现数据节点过少会启,动数据块复制过程(基本不会).

当Hadoop的NameNode节点启动时,会进入安全模式阶段,在此阶段, DataNode会,向NameNode上传它们数据块的列表,让NameNode得到块的位置信息,并对每个文件对应的数据块副本进行统计。当最小副本条件满足时,即一定比例的数据块都达到最小副本数,系统就会退出安全模式,而这需要一定的延迟时间。当最小副本条件未达到要求时,就会对副本数不足的数据块安排DataNode进行复制,直至达到最小副本数。而在安全模式下,系统会处于只读状态, NameNode不会处理任何块的复制和删除命令。

在启动一个刚刚格式化的HDFS时不会进入安全模式,因为没有数据块.

1. HDFS工具

dfsadmin:查看hdfs状态或者在hdfs上执行管理操作.Fsck:检查hdfs中文件的健康状况.

1. HDFS clientProtocol中出现的基本概念的抽象

1,与数据块相关:

数据块:在HDFS中抽象为: Block,包含三个成员变量: blockid,numbytes(文件数据大小)generationStamp (数据块版本号,每次对数据块的修改版本号都会修改,用于数据一致性检查,具有相同blockid但版本号不同的数据块至少有一个是无效的,需要删除).数据块的名词为blk <blockid>.

LocatedBlock:已经确定了存储位置的数据块,成员变量有; Block、数据块所在节点信息,locs、数据块在对应文件中的偏移量. Locs是一个类型为Datanodelnfo的数组,包含了所有可用的数据块位置.

LocaledBlocks:可用于一次定位多个数据块,包含一系列的LocatedBlock对象,

BlockLocalPathInfo:用于HDFS数据块本地读优化,当数据块与客户端位于同一台机器时。不通过数据节点读数据,而是直接本地读.

2.与DataNode相关: DataNodelD. DataNodelnfo

DataNodelD:用于在集群中唯一确定某一数据节点,可以从中获得数据节点的主机地址.DataNodelnfo:提供附加状态信息,包括容量,已使用容量,剩余容量,数据节点在集群中的位置,数据节点状态等信息.

1. HDFS读数据与写数据流程

没有消息边界的概念,因此在流上建立数据赖并通过赖来交互信息,包括三种数据

写数据时,会将通过addblock方法返回的locatedBLock中的locs信息组成一个管线.

求帧,读应答的应答头和数据包、

建立管线的这程需要用到写请求啦、数据会被拆成数据包写入数据队列,数据队列中的数先写入管线中的第一个节点,第一个节点写入第二.次类推,同时维护一个确认队列,认包由最后一个节点产生迎流向上传递,滑逾的节点只有在确认本地写成功之后才会网海传递,直达数据源,首先客户端需要发送写请求,该帧中没有偏移量,所以只能添加能修改数据。帧中的pipeline指定了管线的大小,如果数据源是客户端,则clentname会赋予客户的name,如果数据源是某个数据节点(意味着正在复制),则为空,网时hasSerDataNode置位,这种情况下紧接着是数据源节点信息DataNodelnfo..为了实现管线.还需要包括numstarget和target. Target是数据目标列表. Numstarget是这个列表的大小若为零,意味着当前节点为管线中的最后一个节点,否则,数据目标列中的第一项就是s数据节点的下一个节点.

写请求应答:由最后一个节点开始,往客户端方向发送,管道上的每一个节点都会等该应答之后才会开始接受数据,应答中有附加信息,包括出错的数据节点信息(结合abandon方法) .

1. HDFS源码有哪些优化

1,在ClientProtocol中的getBlockLocations方法一次返回多个数据块,降低名字节点的负载。

2.在DataNode启动时需要握手、注册、上报数据块信息。在上报数据块信息时, DataNode通过blockReport方法汇报其所拥有的数据块信息,两个参数,第一个参数为DataNodeRegistration,第二个是一个长整型数据,包含块的信息,而不是Block数组,若为block数组, namenode接收到后需创建大量block对象。 Block对象有三个长整型成员变量,可以把他们放在一个长整型数组中,这样namenode就不需要创建大量block对象,减namenode的负担。

3.写数据时的管道的设计,由管道中的上游节点向下游节点传输数据,充分利用网络带宽宽。

4在blockSender读取数据时,可以选择通过零拷贝进行数据的发送,具体见下文。5,数据块扫描器的优化,利用客户端读数据的返回码。

1. HDFS数据节点目录结构

在第一次启动HDFS集群是需要先格式化:对namenode, DataNode与!secondarynamenode不需要。让namenode建立对应的文件结构。

数据节点的目录文件结构:数据节点可以管理多个目录结构,可以通过dfs.data.dir配置指定,这些目录结构作为数据块的保存目录。

Dfs.data.dir包括四个目录和两个文件：

四个目录:

blocksBeingWritten:用于保存正在写入的数据块,与temp不同之处是该目录下是客,户端发起的写。

Temp:用于记录正在写入的数据块,该写入过程是由于数据块复制产生,由一个数,据节点写入另一个数据节点。

Current:用于保存已经写入并提交的数据块,校验信息以及数据块扫描器用到的,文件和version文件(包含HDFS运行时的版本信息),

Detach:与系统升级相关的目录.

数据块最开始写入时处于blocksBeingWritten或者temp状态,当数据顺利写入后,今迁移为current.

两个文件:

Storage:保存在提示信息,系统版本相关

in_use-lock:表明目录已经被使用,是一种锁机制,如果停止数据节点,该目录会,消失,防止两个数据节点实例共享一个目录.

2.细节

Current:该目录中即包含目录也包含文件。大部分文件以blk为前缀。文件有两种类型.一举是以blk为前缀的数据块文件,另一类是校验信息文件,用于保存数据块的校验信息.当目录中的数据块达到一定规模时,会创建一个子目录,用干保存新的数据块和校验信息。.默认是64个数据块(128个文件)。同时一个目录下最多有64个子目录,这样即抱枕了目录的深度不会太深影响检索,同时确保每个目录中的数据块时可控的。

其中current还包括了三个特殊的文件: version,包含了运行的HDFS版本信息(storageinfo中的三个信息) ,节点类型(数据节点还是名字节点)以及namenode对于DataNode的标志信息StoragelD,集群中的每个DataNode拥有不同的storagelD.还有两个,数据块扫描需要的文件.

因为DataNode与namenode都有通用的存储服务,因此继承自storageinfo的抽象子类,storage为数据节古名字节点提供通用服务。

1. HDFS数据节点的实现

业务逻辑不会在这些文件结构上直接操作,而是利用管理这些文件结构的对象上提供的服务。数据节点的文件结构管理包括两部分内容:数据存储datastorage和文件系统数据集,FSDataSet. Datastorage专注与对数据节点存储空间生命周期的管理;而FsDataSet则是对数据节占逻辑相关的存储服务,比如创建数据块文件,打开文件上的输入输出流等等,

1数据存储,

数据存储datastorage继承自抽象类storage,而storage继承自storageinfo. Storageinfo包含集群的信息,包括hdfs的版本号, hdfs的创建时间, hdfs的系统标志,而这些信息保,存在current目录下的version文件中.

在version文件中除了上述三个属性,还包括storageType和storagelD.表示是数据节点还是名字节点,以及每个节点的唯一标志storagelD.集群中的每个节点都有不同的StoragelD.用于名字节点标志数据节点.

因为每个DataNode都可以有多个目录,因此storage有个内部类StorageDirectory,每,个StorageDirectory对象代表一个目录. StorageDirectory中保存着目录的的根目录,目录类型以及lock,我们知道在DataNode的目录结构中, current目录下游in use lock文件锁,避!免同一个目录属于多个DataNode.该文件就是在创建StorageDirectory对象是其内部的,try.lock()方法创建的, current目录下的in-use-lock文件就是在try lock ()中创建的。当,

DataNode大效后会释旅该锁,文件镇的创建是利用java NIO的Filelock 创建fie,同时在iflet上调用java.io.file的deleteOnExist方法、当虚拟机退出时自为删除该锁,即in_use-lock文件.

因为storage继乖自storageinfo我们u道在storageinfo中有三个集群信息,而在current.目录下的version文件中即可获取这 个信息(version还存在其他信息).

DataStorage主要用于对存储空间的生存期进行管理(创建目录和升级)。数据节点启动时会调用Datastorage.Format)创建11求,或者利用Datastorage.doUpdate()进行升级(升级过程见下一题)。

升级过程：

HDFS的升级是利用硬链接实现的,不需要两倍大小的存储空间,系统的升级是在方法DataStorage中实现的,最多保留前一版本的文件系统,

1.将current目录改名为previous.tmp

2.调用linkBlocks()在新创建的current目录ド建立到previous.tmp目录中数据块以及校验,文件的硬链接,

3.在current目录下创建新的version文件:

4.将previous.tmp文件改名为previous.完成升级,

5.此时存在着previous与current两个目录。且包含同样的数据块和校验信息文件,但,是他们各自有自己的version文件。

6.如果需要回滚操作,只需要将previous改名为current就可以完成回滚。回滚时调用,doRollBack()方法。具体过程:先将current改名为remove.tmp:将previous改名为current,再将remove.tmp删除。

7.升级提交:若不回滚,此时调用doFinalize()提交升级。将previous改名为finalize.tmp,然后删除finalize.tmp,完成提交

升级过程中之所以引入大量的临时文件,是因为当发生升级故障时,可以根据这些临时文件判断故障发生在哪个阶段。



2.文件系统数据集(逻辑功能的实现部分)

主要负责DataNode逻辑相关的存储服务。

FSDataSet中的方法分为三类:与数据块操作相关、数据块校验信息相关、其他(关闭等方法)

FSDataSet的实现借鉴了Linux逻辑卷管理的思想。FSDataSet将它管理的存储空间分为三个级别: FSDir, FSValume, FSVolumeSet

FSDir维护着current目录下的子目录.FSDir中的成员遍历children包含了目录下的所有,子目录,其成员变量children包含了目录下的所有子目录。

FSVolume:我们知道在DataNode下有多个存储目录. FSVolume就是dfs.data.dir中的一项。因此系统中会有一个或者多个FSVolume对象。而这些FSVolume对象由FSVolumeSet管理。



由于存在上述内部类,其成员变量有FSVolumeSet volumes,以及ongoingCreates和volumeMap两个hashmap.ongoingCreates是数据块到ActiveFile的映射关系, ongoingCreates的Value值为ActiveFile.记录着写入的线程等信息。volumeMap是block到DataNodeBlockinfo的映射关系。DataNodeBlockinfo保存着数据块的一些额外信息,包括数据块的目录,对应的文件等信息。

FSDataSet中比较重要的方法是写入数据块和提交数据块

FSDataSet在写入数据时会调用writeToBlock方法,该方法会创建两个数据流:一个是写入到数据块的数据流,一个是写到校验文件的数据流,调用者可以使用者两个数据流进行写数据和校验信息.写入过程如下:

1.通过isValidBlock()方法判断数据块是否有效,即满足数据块已存在,且该数据块已提交,若有效则为append操作或者数据块恢复操作、若有效首先执行数据块分离,避免在写入的过程中数据节点崩溃而导致数据不一致问题,数据分离会将需要写入的数据块复制一份.

2.如果数据块已经在写入过程中(通过if判断ongoingCreates是否包含该block) .如果是则不能再创建数据块,只能进行数据恢复,先将ActiveFile中的线程都中断停止释放资源、.在ongoingCreates删除该数据块.

3.开始为数据块创建文件,井在上下文中记录信息,如果不是恢复模式则新建据块(即文件),如果是恢复模式,则复用文件,如果都是不是则append.

4.最后打开数据块的信息会被保存到ongoingCreates中,并创建到数据块文件以及校验信息的输入流.

5.和writeToblock对应的是finalizeBlock.写入完成后会调用finalizeBlock方法提交数据块,是提交到FSDataSet.不同于blockreceive()方法,首先将数据文件和校验文件移动到current目录下,同时将信息保存在volumMap中并删除在ongoingcreate中的信息.

FSDataSet还包含删除数据块的方法: unfinlize()方法和invalidate()方法,用于删除复制不成功的数据块以及已经提交的数据块. Invalidate()方法通过线程池实现异步删除.

1. DataNode流式接口的实现

数据节点通过datastorage和fadataset将数据块的存储抽象为对象上的服务,流式接口就是建立在这个服务之上. HDFS是通过基于tcp流的方式实现数据访问接口,实现文件的读写.

数据节点在启动时会调用DataNode.startDataNode方法,其中会在DataNode中创建,serverSocket对象并绑定到监听地址的监听端口上,然后创建线程组和DataXceiverServer服,务器对象并将serverSocket对象以参数传入.

然后会创建线程组和DataXceiverServer服务器,并将线程组的线程设置为守护线程。DataXceiverServer类是DataNode的辅助类,它最主要是用来实现客户端或其他DataNode与.当前节点通信,接收/发送数据块由Dataxceiver对象处理. DataXceiverServer有一个标志位.shouldrun. DataXceiverServer中主要有两个方法,一个是run方法一个是kill方法,其中run方法,方法内通过while(shouldrun)实现了accept循环, accept会阻塞等待客户端的连接.Kill方法用于关闭监听的socket,首相将shouldrun标志位设置为false,然后关闭socket并通过循环关闭已经打开的被DataXcieve对象使用的socket.当有请求是会创建一个新的线程,即Dataxceive对象. DataXceiverServer是为一客户一线程的方式. DataXceiverServer只处理,客户端的连接请求,对于实际的请求处理和数据交换由DataXceiver对象处理, DataXceiver对象拥有自己独立的线程,该DataXceiver对象和其拥有的线程只处理一个客户端请求.在DataXceiver继承了Runnable接口,在run方法中首先创建输入流,然后检查版本号,再检查是否超出数据节点的能力,然后读入请求码,通过switch根据请求码调用相应的方法. (结合读请求帧与写请求帧)

比如请求帧中的操作码为81,则为读请求,调用DataXceiver.readBlock()方法。

Dataxceiver中还有copyblock与replaceblock等方法。

1. DataXceiver处理读请求？

当请求帧中的操作码为81时,为读,会在DataXceiver对象的run方法的switch中根据操作码调用相应的方法.

DataXceiver.readBlock()给出了数据节点读数据的流式框架。

首先根据socket输入流,读取请求帧的信息,构造数据块发送器blockSender对象,选后通过该对象发送数据. 1.blockSender的构造函数: blockSender的构造函数会进行一些楼查(P292) ,若通过可以在blockSender的构造函数中执行相应的准备操作,比较重要的是打开校验文件,从文件中获得校验方法,校验块的大小,确定应答数据在数据块中的位置.计算应答数据在校验文件中的校验位置等等,此时要主要校验信息在校验文件中是按照块委储的,默认512字节存一个校验信息,因此返回给客户端的数据必须包含用户请求的数据的最小数据块,有可能大于请求的数据。并在最后通过FSDataSet的方法构建输入流,若不通过会通过异常通知readblock方法,给客户端返回错误码。若成功构建了blockSender对象会给客户端返回成功的响应码。然后通过blockSender.sendblock发送数据给客户端.

Blocksender的构造方法只要是一些准备工作。会根据读请求帧计算应答数据在数据埃!中的位置,并打开校验信息文件,获取校验方法以及检验块的大小,这些校验信息存在与验文件的头部。因为校验信息是按照块组织的(不同于block) , 512个字节。为了让客户端进行数据校验,必须返回包含用户读取数据的所有块以便客户端进行数据校验,因此返国的数据可能会大于客户端请求的数据。并在最后通过FSDataSet的方法构建数据块的输入流.至此读数据的准备工作完成。

2.blockSender发送数据有两种方式:普通的方式和NIO的transferTo.后者可以实现零拷贝,避免了不必要的数据拷贝(内核缓冲区到用户缓冲区->用户缓冲区到内核缓冲区)以及用户态向内核态以及内核态向用户态的装换。

blockSender通过sendblock方法和sendChunks()方法发送数据, sendblock会根据缓冲区的大小计算一次发送多少的数据。然后sendblock通过循环调用sendChunks方法发送数据包,sendChunks会在文件中。同时会有节流器控制读写数据的速度,避免10成为系统的瓶颈.在DataXceiver的readblock方法在最后部分,当客户端成功读取并校验数据后会返回-个成功响应码通知数据节点。当数据块读完之后可以根据客户端返回的响应码通知数据块扫!描器,让扫描器标记该数据块正常,因为客户端已经校验, DataNode使用数据块扫描器定!期扫描数据块,以尽早发现错误,是一个比较耗时的操作。通过此方式可以降低扫描器的负!载。注意:客户端只上报正常的数据块,当数据块出现错误时会报给namenode,由namenode决定下一步的处理,而不是通知数据节点,因此客户端读取数据后只会给数据节点返回正常的响应码。

1. DataXceiver处理写请求？

补充:打开一个DFSOutputStream流, Client会写数据流到内部的一个缓冲区中,然后!数据被分解成多个Packet,每个Packet大小为64k字节,每个Packet又由一组chunk和这组chunk对应的checksum数据组成,默认chunk大小为512字节,每个checksum是对512字节数据计算的校验和数据。这也就是客户端在读文件时,返回的数据可能比请求的数据多的原因。

当Client写入的字节流数据达到一个Packet的长度,这个Packet会被构建出来,然后会被放到队列dataQueue中,接着DataStreamer线程会不断地从dataQueue队列中取出Packet,复制发送到Pipeline中的第一个DataNode上,并将该Packet从dataQueue队列中移到ackQueue队列中. ResponseProcessor线程接收从Datanode发送过来的ack,如果是一个成功的ack,表示复制Pipeline中的所有Datanode都已经接收到这个Packet,ResponseProcessor线程将packet从队列ackQueue中删除.

在发送过程中,如果发生错误,所有未完成的Packet都会从ackQueue队列中移除掉,然后重新创建一个新的Pipeline,排除掉出错的那些DataNode节点,接着DataStreamer线p维续从dataQueue队列中发送Packet.

当请求帧中的操作码为80时,为读,会在switch中调用相应的方法.

DataXceiver.writeBlock()给出了数据节点读数据的流式框架,根据写请求帧构造数据块接,收器对象blockReceiver, blockReceiver完成检查后会调用FSDataSet.writeToblock方法,如果需要会创建数据块文件和校验信息文件,并返回到校验文件和数据文件的输出流.

1,根据请求帧中的数据构建blockreceiver()对象, writeblock主要通过该对象

入数据.

2.建立输入流管道:

接着writeblock根据请求信息建立数据流管道,此时需要socket对象:与上游通信和与,下游通信,因此有两个socket对象,其中s用于与管道上游通信(in和replyout) mirrorSock(mirrorout和mirrorin).

如果当前数据节点不是最后一个,则会通过请求帧的目标列表与下一个数据节点建立socket连接,建立后通过mirrorout与下游节点发送写请求,相应改变目标列表和targetnum大小,其他数据一致,当向下一个节点的写请求发送后,该节点会等待相应,这是一个同步的过程,建立数据流管道会花费一定时间,当与下游节点建立连接时出错,下游节点会抛出异常,此时下游的返回码中是error时,上游节点会在响应码的附加信息中会包含第一个出借节点的地址信息.

向上游发送相应的时机:到下游的连接出现问题:为写准备工作完成之后发送的应答.3接受数据(packetResponder部分)

当writeblock处理完客户端的写数据请求建立管道之后,方法blockreceiver.receiveblock.开始接收数据包,校验数据并写入本地的数据块文件和校验信息文件,如果节点是管道中间节点,还需要向下一个数据节点转发数据包,同时节点也需要从下游接受数据包确认.blockReceiver引入了packetResponder线程,它和blockReceive所在的线程一起工作,分别用于从下游接受应答和从上游接受数据。避免一个线程处理两个输入流造成延迟等待。向上游发送确认包需要满足:本身已经处理完数据包;收到了下游的确认包.

由于数据包由blockReceiver线程处理,而接受下游的确认是由packetResponder处理,因此需要通过某种机制告诉packetResponder线程: packetResponder维护一个队列ackqueue.每当blockreceiver.receivepacket()处理完一个数据包将其通过packetResponder.enqueue()放入packetResponder的队列acqueue中,便可实现之间的通信. packetResponder通过java的同步工具wait方法等待ackqueue中的数据,其中wait等待的是enqueue中的notifyAll的通知.如果ackqueue中有数据,则获取第一个数据,接下来等待下游的确认,当收到下游确认时,则将确认包向上游发送,如果处理的是写请求的最后一个数据包的确认,则关闭packetResponder所属的数据块接收器对象,向FSDataSet提交数据块.

4,数据接收(receiveblock部分)

(1)、从上游接收数据主要是通过blockreceiver.receiveblock方法接收,大部分工作是通过,receivePacket方式实现.

receivePacket方法通过调用readNextPacket(),每次最少读入一个写请求数据包,并将,读入的数据放入buffer中,放入buffer时保证buffer的剩余空间可以写入整个数据包.blockreceive.receivePacket)从buffer中读入数据包的包头信息,并赋值给成员变量,调用setBlockPositon()方法设置写数据时的文件位置,由于需要写入两个文件:数据块文件和校验文件, setBlockPosition()会设置这两个位置,写数据块的位置容易确定,但是写入校验文件的位置相对复杂,因为校验文件的校验信息是按块计算校验并写入的,而客户端的数据是以流的形式写入。如果写入的数据落在了某个校验块的中间,需要在确认校验信息文件写位置,的时候计算不完整数据块的校验信息,为后继写校验数据做准备。如果写入位置不是校验块边界, setBlockPosition会调用computeParitialChunkCrc()计算上述校验信息。

computeParitialChunkCrc方法输入三个参数,通过参数可以计算出不完整数据块的长度,铁后分配缓冲区读入数据,计算不完整数据块的校验信息,存入校验文件末尾,同时会被blockreceiver的变量保存。

对于数据块文件,读入数据会在数据块文件后面追加数据,但是对于校验文件,若不是一个完整数据校验块,会更新校验文件的最后部分,而不是简单的追加。

(2),当上述过程结束后, receivepacket方法将数据写入下一个数据节点。通过这种方式数据包会流经管道上所有的数据节点。

5.对于长度为0的数据包,这是一种特殊的数据包,当客户端长时间不发送数据时,会发送该心跳包,用于维护客,户端和数据节点的tcp连接。

6.receivepacket的后续操作就是将数据写入数据块文件和校验信息文件。此时注意在最后一个数据节点写入时会进行数据校验,管道中的其他节点不需要执行,因为若最后一个节点的数据无误则上游节点也无误。若此时校验发送数据有问题,则上报名字节点。

7.Receiveblock循环调用receivepacket方法,处理这次写请求的所有数据包。当所有数,据接受完毕后,发送一个空包给下游节点表示数据发送完毕,然后等待packetResponder线程结束,该线程结束表示所有的响应包已发送到上游节点。

8.最后通过FSDataSet.finalizeBlokc方法提交数据块,写数据请求处理完毕。同时处理结果会上报给namenode。当datanode处理完最后一个packag后, 对通过datanode.NotifyNamenodeBlockReceive()方法向namenode上报。但是由于datanode会频繁写,因此, datanode调用此方法时,并不是真正的向namenode上报, datanode会维护着一个队列,用于存放这些需要上报的block, datanode.NotifyNamenodeBlockReceive()会将写,完的block插入到队列中,最后通过DataNode的服务线程将队列封装为DataNode.Receive()想要的格式,发送给namenode。在这里, NotifyNamenodeBlockReceive方法维护两个队列,一个保存着完成写的数据块信息,另一个队列保存着相应的删除提示信息,因为如果是数据块替换的话,最后需要删除源节点的数据块, namenode会根据这个信息将源节点的数据块删除。

使用FSDataSet的finalize()方法提交数据块(提交到当所有的数据包都处理完, FSDataSet),并通知namenode.

需要注意的是在写入的过程中只有最后一个节点需要进行校验,会调用相应的方法,其他节点不需要校验信息。

#### HDFS DataNode升级过程

DataNode的升级是利用Linux的硬链接来实现的,不需要移动大量的数据, DataNode.

中的版本信息主要在current/version中。升级主要是通过datastorage.doUpgrage()实现,

1:将current文件改名为previous.temp.

2:在新建的current下建立指向previous.temp的硬链接

3在新建的current目录下创建新的version文件,此时, current文件与previous.temp

文件除了version不同外其他相同。

4.将previous.temp改名为previous.

5.升级提交:将previous改名为finalized.temp,然后删除,完成升级的提交,

若升级的过程中出现了故障需要回滚,因为previous中是升级前的系统,因此只需要将

previous恢复至原来的current文件即可.

1,根据previous中version文件中的构建版本号和时间截与升级的current中的version

中的构建版本号和时间裁比较,获得需要回滚的版本。

2.将current改名为remove.temp,将previous改名为current,将remove.temp删除在升级的过程中有大量的temp文件,主要是便于系统知道此时处于升级过程中的哪个阶段

#### HDFS中的distcp和fastcp?

DistCp (Distributed Copy)是用于大规模集群内部或者集群之间的高性能拷贝工具。它使用Map/Reduce实现文件分发,错误处理和恢复,以及报告生成。它把文件和目录的列表作为map任务的输入,每个任务会完成源列表中部分文件的拷贝。

DistCp第一版使用了MapReduce并发拷贝数据,它将整个数据拷贝过程转化为一个map-only Job以加快拷贝速度。由于DistCp本质上是一个MapReduce作业,它需要保证文件中各个block的有序性,因此它的最小数据切分粒度是文件,也就是说,一个文件不能被,切分成不同部分让多个任务并行拷贝,最小只能做到一个文件交给一个任务.

DistCp2针对DistCp1在易用性和性能等方面的不足,提出了一系列改进点,包括通过去,掉不必要的检查缩短了目录扫描时间、动态分配各个Map Task的数据量、可对拷贝限速避,免占用过多网络流量、支持HSFTP等。尤其值得一说的是动态分配Map Task处理数据量。DistCp1的实现跟我们平时写的大部分MapReduce程序一样,每个Map Task的待处理数据量在作业开始运行前已经静态分配好了,这就出现了我们经常看到的拖后腿的现象:由于一个Map Task分配的数据量过多,运行非常缓慢,所有Reduce Task都在等待这个Map Task运行完成。而对于DistCp而言,该现象更加常见,因为最小的数据划分单位是文件,文件有大有小,分到大文件的Map Task将运行的非常慢,比如你有两个待拷贝的文件,一个大小为1GB,另一个大小为1TB,如果你指定了超过2个的Map Task,则该DistCp只会启动两个Map Task,其中一个负责拷贝1GB的文件,另一个负责拷贝1TB的文件,可以想象其中一个任务将运行的非常慢。DistCp2通过动态分配Map Task数据量解决了该问题,它实现了一个DynamiclnputFormat,该InputFormat将待拷贝的目录文件分解成很多的chunk,其中每个chunk的信息(位置,文件名等)写到一个以“.chunk.K" (K是一个数字)结尾的HDFS文件中,这样,每个文件可看做一份“任务”, “任务”数目要远大于启动的Map Task数目,运行快的Map Task能够多领取一些“任务” ,而运行慢得则领取少一些,进而提高数据拷:贝速度。尽管DistCp2中Map Task拷贝数据最小单位仍是文件,但相比于DistCp1,则要高效得多,尤其是在文件数据庞大,且大小差距较大的情况下。

Fastcp采用硬链接的方式,比较适合在联邦namenode中namenode之间的数据copy







#### Hadoop如何实现全局排序

1.设置一个分区。当文件较大时效率较低

2.首先创建一系列排好序的文件;其次,串联这些文件(类似于归并排序);最后得到一个全局有序的文件。主要的思路是使用一个partitioner来描述全局排序的输出。比方说我,们有1000个1-10000的数据,跑10个ruduce任务, 如果我们运行进行partition的时候,.能够将在1-1000中数据的分配到第一个reduce中, 1001-2000的数据分配到第二个reduce中,以此类推。即第n个reduce所分配到的数据全部大于第n-1个reduce中的数据。这样,每个reduce出来之后都是有序的了,我们只要cat所有的输出文件,变成一个大的文件,就都是有序的了。

基本思路就是这样,但是现在有一个问题,就是数据的区间如何划分,在数据量大,还有我们并不清楚数据分布的情况下。一个比较简单的方法就是采样,通过对key空间进行采,样就可以较为均匀的划分数据集。Hadoop有自带的采用工具。

<http://blog.csdn.net/tusing/article/details/9145243>

利用hadoop分而治之的计算模型,可以参照快速排序的思想。在这里我们先简单回忆,一下快速排序。快速排序基本步骤就是需要现在所有数据中选取一个作为支点。然后将大于这个支点的放在一边,小于这个支点的放在另一边。

设想如果我们有N个支点(这里可以称为标尺),就可以把所有的数据分成N+1个part,将这N+1个part丢给reduce, 由hadoop自动排序,最后输出N+1个内部有序的文件,再,把这N+1个文件首尾相连合并成一个文件,收工。

1) 对待排序数据进行抽栏;

2) 对抽样数据进行排庄,产生标尺;

3) Map对输入的每条数据计算其处于哪两个标尺之间;将数据发给对应区间ID的reduce

4) Reduce将获得数据直接输出。

这里还有一点小问题要处理:如何将数据发给一个指定ID的reduce? hadoop提供了多

种分区算法。这些算法根据map输出的数据的key来确定此数据应该发给哪个reduce( reduce

的排序也依赖key) 。因此,如果需要将数据发给某个reduce,只要在输出数据的同时,提供一个key (在上面这个例子中就是reduce的ID+url) ,数据就该去哪儿去哪儿了。





### hadoop的安全认证

1,背景

在Hadoop1.0.0或者CDH3版本之前, hadoop并不存在安全认证一说。默认集;群内所有的节点都是可靠的,值得信赖的。用户与HDFS或者M/R进行交互时并!不需要进行验证。导致存在恶意用户伪装成真正的用户或者服务器入侵到hadoop集群上,恶意的提交作业,修改JobTracker状态,篡改HDFS上的数据,伪装成NameNode或者TaskTracker接受任务等。尽管在版本0.16以后, HDFS增加了文件和目录的权限,但是并没有强认证的保障,这些权限只能对偶然的数据丢失起保护作用。恶意的用户可以轻易的伪装成其他用户来篡改权限,致使权限设置形同虚设。不能够对Hadoop集群起到安全保障。

在Hadoop1.0.0或者CDH3版本后,加入了Kerberos认证机制。使得集群中的节!点就是它们所宣称的,是信赖的。 Kerberos可以将认证的密钥在集群部署时事先放到可靠的节点上。集群运行时,集群内的节点使用密钥得到认证。只有被认证过节点才能正常使用。企图冒充的节点由于没有事先得到的密钥信息,无法与集群内部的节点

通信。防止了恶意的使用或篡改Hadoop集群的问题,确保了Hadoop集群的可靠安全。



\2. Hadoop安全问题

2.1用户到服务器的认证问题,

NameNode, ,JobTracker上没有用户认证

用户可以伪装成其他用户入侵到一个HDFS或者MapReduce集群上。

DataNode上没有认证:

Datanode对读入输出并没有认证。导致如果一些客户端如果知道block的ID,就

可以任意的访问DataNode上block的数据,

JobTracker上没有认证

可以任意的杀死或更改用户的jobs,可以更改JobTracker的工作状态,

2.2服务器到服务器的认证问题!

没有DataNode, TaskTracker的认证

用户可以伪装成datanode ,tasktracker,去接受JobTracker, Namenode的任务指派。



\3. Kerberos能解决的Hadoop安全认证问题

kerberos实现的是机器级别的安全认证,也就是前面提到的服务到服务的认证问,题。事先对集群中确定的机器由管理员手动添加到kerberos数据库中,在KDC上分别产生主机与各个节点的keytab(包含了host和对应节点的名字,还有他们,之间的密钥),并将这些keytab分发到对应的节点上。通过这些keytab文件,节点可以从KDC上获得与目标节点通信的密钥,进而被目标节点所认证,提供相应的服务,防止了被冒充的可能性。

解决服务器到服务器的认证,

由于kerberos对集群里的所有机器都分发了keytab,相互之间使用密钥进行通信,确保不会冒充服务器的情况。集群中的机器就是它们所宣称的,是可靠的。防止了用户伪装成Datanode, Tasktracker,去接受JobTracker, Namenode的任务指派。

解决client到服务器的认证,

Kerberos对可信任的客户端提供认证,确保他们可以执行作业的相关操作。防止用户恶意冒充client提交作业的情况。

用户无法伪装成其他用户入侵到一个HDFS或者MapReduce集群上

用户即使知道datanode的相关信息,也无法读取HDFS上的数据,

用户无法发送对于作业的操作到JobTracker上

对用户级别上的认证并没有实现:

无法控制用户提交作业的操作。不能够实现限制用户提交作业的权限。不能控制哪些用户可以提交该类型的作业,哪些用户不能提交该类型的作业。这些由AC模块控制(参考)



\4. Kerberos工作原理介绍,

4.1基本概念

Princal(安全个体):被认证的个体,有一个名字和口令,

KDC(key distribution center ):是一个网络服务,提供ticket和临时会话密钥

Ticket:一个记录,客户用它来向服务器证明自己的身份,包括客户标识、会

密钥、时间戳。

AS (Authentication Server): 认证服务器TSG(Ticket Granting Server): 许可证服务器,4.2 kerberos工作原理

大数据方向总结,邮箱1ma-xij@163.com.有问题或者大数据相关可联态该邮箱,



4.2.1 Kerberos协议

Kerberos可以分为两个部分:

Client向KDC发送自己的身份信息, KDC从Ticket Granting Service得到TGT(ticket-granting ticket),并用协议开始前Client与KDC之间的密钢将TGT加瓷回复给Client.此时只有真正的Client才能利用它与KDC之间的密钥将加密后的TGT解密,从而获得TGT, (此过程避免了Client直接向KDC发送密码,以求通,

过验证的不安全方式)

Client利用之前获得的TGT向KDC请求其他Service的Ticket,从而通过其他.

Service的身份鉴别,

4.3 Kerberos认证过程

Kerberos协议的重点在于第二部分(即认证过程) :



(1) Client将之前获得TGT和要请求的服务信息(服务名等)发送给KDC, KDC中的Ticket Granting Service将为Client和Service之间生成一个Session Key用于

Service对Client的身份鉴别。然后KDC将这个Session Key和用户名,用户地址

(IP),服务名,有效期,时间戳一起包装成一个Ticket(这些信息最终用于Service对Client的身份鉴别)发送给Service, 不过Kerberos协议并没有直接将Ticket发,送给Service,而是通过Client转发给Service,所以有了第二步。

(2)此时KDC将刚才的Ticket转发给Client,由于这个Ticket是要给Service的.不能让Client看到,所以KDC用协议开始前KDC与Service之间的密钥将Ticket加密后再发送给Client,同时为了让Client和Service之间共享那个密钥(KDC在第一步为它们创建的Session Key), KDC用Client与它之间的密钥将Session Key加,密随加密的Ticket一起返回给Client.

(3)为了完成Ticket的传递, Client将刚才收到的Ticket转发到Service.由于Client.不知道KDC与Service之间的密钥,所以它无法算改Ticket中的信息。同时Client将收到的Session Key解密出来,然后将自己的用户名,用户地址(IP)打包成Authenticator用Session Kev加密也发送给Service.

(4)Service收到Ticket后利用它与KDC之间的密钥将Ticket中的信息解密出来,从而获得Session Key和用户名,用户地址(IP) ,服务名,有效期。然后再用Session Key将Authenticator解密从而获得用户名,用户地址(IP)将其与之前Ticket中解密出来的用户名,用户地址(IP)做比较从而验证Client的身份。

(5)如果Service有返回结果,将其返回给Client.

4.4 kerberos在Hadoop上的应用,Hadoop集群内部使用Kerberos进行认证

具体的执行过程可以举例如下:



4.5使用kerberos进行验证的原因,

可靠Hadoop本身并没有认证功能和创建用户组功能,使用依靠外围的认证系统高效Kerberos使用对称钥匙操作,比SSL的公共密钥快,操作简单用户可以方便进行操作,不需要很复杂的指令。比如废除一个用户只需要从Kerbores的KDC数据库中删除即可。

总结:

要是通过kerberos+委托令牌实现。

lerberos实现的是机器级别的安全认证,也就是前面提到的服务到服务的认证问题。事ti集群中确定的机器由管理员手动添加到kerberos数据库中,在KDC上分别产生主机上《个i点的keytab(包含了host和对应节点的名字,还有他们之间的密钥),并将这些kevtabae到对应的节点上。通过这些keytab文件,节点可以从KDC上获得与目标节点通信的索;i,进而被目标节点所认证,提供相应的服务,防止了被冒充的可能性,

Kerberos可以分为两个部分:

1.Client向KDC发送自己的身份信息, KDC从Ticket Granting Service得到rsiticket-granting ticket), 并用协议开始前Client与KDC之间的密钥将TGT加密回复给,cient.此时只有真正的Client才能利用它与KDC之间的密钥将加密后的TGT解密、从而获,a TGT. (此过程避免了Client直接向KDC发送密码,以求通过验证的不安全方式)

2.Client利用之前获得的TGT向KDC请求其他Service的Ticket,从而通过其他Service的身份鉴别.

Kerberos在Hadoop中的应用:



KDCekeytabe



使用KDc和client之间的密钥请求服务与JobTracker通信返回client与JobTracker的sessionkey.

使用sessionkey请求服务.



client .

obTracker

kextabe



keytab



返回相关信息.



由于在分布式系统中,客户端与服务器端频繁交瓦,如果每次都需要认证,会对KDC造成很大压力。因此, Hadoop采用委托令牌来支持后续的访问。委托令牌有namenode创建,看作为客户端与服务器的秘钥。当客户端首次访问namenode时需要向KDC认证,会从namenode得到委托令牌,之后再访问namenode,不会再去KDC认证、而是通过访问令牌i问namenode、节点之间的距离。

### hadoop如何衡量两个节点之间的距离

### 



