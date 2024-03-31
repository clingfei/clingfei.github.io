---
declare: true
title: MapReduce--Simplified Data Processing on Large Clusters
categories: [论文阅读, Distributed Systems]
tags:
- Distributed Systems
- 论文阅读
---

---



## Abstract

mapreduce是一个用于处理和生成大数据集的编程模型。用户指定一个用于处理键值对的函数来生成键值对集合的中间表示，和一个reduce函数来合并与相同的key相关联的中间值。许多真实任务可以使用此模型来表达。

按照这种函数模型编写的程序可以在计算机集群上并行运行。运行时系统负责划分输入数据的细节，在不同的机器上调度程序的执行，处理机器错误和管理内部机器之间的通信。这允许程序员不需要任何并行与分布式系统的只是就能够利用大量的分布式系统资源。

论文中的mapreduce实现在大型集群中运行，并具有高度的可扩展性：经典的MapReduce在数千台机器上处理许多TB字节的数据。并且MapReduce已经被证明易于使用：超过一千个MapReduce作业每天都在Google集群上执行。

## 1 Introduction

在过去的五年中，作者和Google的同事实现了数千个用于处理大量的元数据（例如爬取的文件、页面请求日志等）的计算任务，来计算不同类型的派生数据，例如倒排索引，web页面的不同的图结构表示，每台主机每天爬取的页面的总结、最频繁的请求数量等。大部分这样的计算是概念清晰的，但是输入数据通常非常庞大，并且计算过程必须被分发到数百或数千台机器上以在可接受的时间内完成。需要用大量的代码来解决并行计算、分配数据和处理故障等问题，从而掩盖了原本的简单计算。

为了解决这样的复杂性问题，设计了一个新的抽象，这一抽象能够运行我们表达原本的简单计算过程，并将大量的并发、容错和数据分发、负载均衡的细节隐藏在库中。我们的抽象来自于Lisp和许多其他函数时语言中存在的map和reduce原语的启发。我们意识到大部分计算过程包括对每个输入中的逻辑记录应用map操作，来计算中间键值对的集合，然后对所有使用映射到相同key的value进行reduce操作，来将派生出的数据恰当地结合起来。我们使用的用户指定map和reduce操作的函数模型能够轻易地实现并行化大型计算和使用重新执行作为容错的主要机制。

MapReduce的主要贡献是提供了一个能够实现自动并行化和大规模计算的分发的接口，结合这个接口的实现，能够在大规模的商用机集群中实现高性能计算。

## 2 Programming Model

MapReduce输入和输出均为键值对的集合，MapReduce库的用户使用Map和Reduce两个函数来表达计算过程。

Map由用户提供，对于输入的键值对，产生中间的键值对表示。MapReduce将所有的宇相同的中间key I相关联的值划分为同一组，并将其传递给Reduce函数。

Reduce函数同样由用户提供，接收中间key I和与I相关的value的集合作为输入。将这些value合并以产生一个更小的value的集合。通常，每次reduce调用只产生零个或一个输出。中间值通过迭代器提供给Reduce函数使用，因而能够处理规模太大而无法放入内存的值列表。

### 2.1 Example

考虑计算在大规模的文档集合中计数每个单词出现次数的问题。用户可能会有如下伪代码：

```
map(String key, String value):
	// key: document name
	// value: document contents
	for each word w in value:
		EmitInttermediate(w, "1");
		
reduce(String key, Iterator values):
	//key: aword
	// values: a list of counts
	int result = 0;
	for each v in values:
		result += ParseInt(v);
	Emit(AsString(result));
```

map函数对于value中的每个word，执行EmitInttermediate(加上word的出现次数)，reduce函数把所有的对于与某一个word相关联的发射的所有出现次数求和。

此外，用户使用输入和输出文件的名字和可选的调整参数来填充*mapreduce specification*对象。然后调用MapReduce函数并将specification对象传递给MapReduce。用户代码通过MapReduce库链接在一起。

### 2.2 Types

从概念上讲，用户所提供的map和reduce函数具有相关联的类型：

- map	                 (k1, k2)								-> list(k2, v2)
- reduce                 (k2, list(v2))                        ->list(v2)

即输入的key和value来自与输出的key和value不同的域。此外，中间key和value域输出的key和value具有相同的域。

*怎么理解域？  为什么上面说具有相同的域？对于输入的（k1, k2)，map生成了（k2, v2)这样的键值集合，而reduce将相同的k2对应地value合并成了list，并最终输出list(v2)?*

## 3 Implementation

MapReduce接口有很多可能的实现方式，具体实现方式取决于具体的环境。例如，某种方式可能适用于小型共享内存机器，而另一种可能适用于大型的NUMA多处理机，或是用于更大规模的多台机器组成的互联网络。

本节描述的实现使用于Google广泛使用的计算环境：通过交换以太网连接在一起的大型商用PC集群。

- 每台PC通常使用双路X86处理器(dual-processor x86 processors)，具有2~4GB内存，运行Linux操作系统。
- 使用商用的网络硬件：通常机器级100Mb/s或1Gb/s，但平均的bit带宽会小很多
- 由数百或数千台机器组成的集群，因此会有很多机器出错
- 存储由直接插入到每台机器上的IDE硬盘来提供。内部的分布式文件系统用于管理这些硬盘存储，文件系统使用备份在不可信的硬件上提供可用性和可靠性。
- 用户向调度系统提交作业，每个作业包括一系列任务，并被调度器映射到集群中一系列空闲的机器中运行。

### 3.1 Execution Overview

Map调用通过自动将输入数据划分为M个split的集合来被分布到多个机器上。输入的split可以被不同机器并行处理，Reduce调用通过使用划分函数(例如hash(key) mod R)将intermediate key划分为R个不同的片来分发到不同的机器上。划分数量R和划分函数由用户指定。

下图展示了MapReduce的完整过程。在用户程序调用MapReduce时，将会依次执行如下过程（图中的序号代表了对应的编号）：

![image-20220922210631823.png](https://s2.loli.net/2022/09/28/TAerbiMgsCVUjmk.png)

1. 用户程序中的MapReduce首先将输入文件划分为M个片，通常每个片的大小为16~64MB（由用户通过可选参数来确定）。然后在集群中启动程序（每台机器运行一份程序的拷贝）。
2. 其中一个程序的拷贝比较特殊，成为master。其他的程序由master安排任务，称为worker。共有M个Map任务和R个Reduce任务供master分配。master选择一个空闲的worker，然后为其分配一个Reduce或是Map任务。
3. 执行map任务的worker读取相关的输入split的内容，从输入的数据中解析出key/value对，并将每个key/value对传递给用户定义的Map函数。Map函数所产生的intermediate key/value对缓存在内存中。
4. 缓存的键值对周期性地写入到本地磁盘中，通过划分函数划分为R个块（*划分成R个块的目的是什么？*）。这些在本地磁盘上的键值对的位置被传回给master，master负责将这些位置转发给执行reduce函数的worker。
5. 当reduce worker收到master传来的地址后，使用远端过程调用(RPC)来从map worker的本地磁盘中读取缓存的键值对。当reduce worker读取了所有的中间数据后，通过intermediate key来进行排序，所以具有相同key的所有value被分到相邻的位置。由于通常许多不同的key映射到相同的reduce任务，因此排序是必须的（*什么叫不同的key映射到相同的reduce任务？ 因为一个Reduce处理的intermediate key/value中不可能只有一个key*）如果数据量太大而不能全部装入内存，则使用外部排序。
6.  reduce worker在排序后的中间数据上迭代，并且对于遇到的每个唯一的intermediate key，将key和对应的intermediate value的集合传递给用户的Reduce函数。Reduce函数的输出被附加到这个reduce partition的最终输出文件后。
7. 在所有的map任务和reduce任务完成后，master唤醒用户程序，此时，用户程序对于MapReduce的调用结束，控制权重新回到用户程序。

在MapReduce成功结束运行后，MapReduce的输出在R个输出文件中（每个Reduce任务生成一个输出文件，一共分成了R个Reduce任务，输出文件名由用户指定）。通常用户不需要将R个输出文件合并为一个，而是将R个输出文件作为另一个MapReduce调用或者其他的分布式应用的输入。

### 3.2 Master Data Structures

master使用了多个数据结构。master需要保存每个map task和reduce task的状态：idle, in-progress和completed，对于不处于idle状态的worker机器，还需要保存其身份（map/reduce？）。

master是将中间文件区域的位置从map task传递到reduce task的枢纽。因此，对于每个处于completed状态的map task，master保存了map task产生的R个中间文件区域的位置和大小。master在map task完成时收到对于这些文件区域的大小和位置的更新，这些信息被逐步发送给正在进行reduce task的worker机器。

### 3.3 Fault Tolerance

因为MapReduce的主要目的是使用数百上千台机器来帮助处理大量数据，因此必须能够容忍机器损坏。

#### Worker Failure

master周期性ping每台worker机器，如果在规定的时间内没有收到来自worker的相应，那么master就将这台worker标记为failed. 任何被worker完成的map task都将被重置为初始的空闲状态，并因此这些task将会能够在其他的worker上进行调度。(*调度的单位是task而不是worker?*)类似地，在failed worker上运行的map或reduce task也将会被重置到idle状态以重新调度。

处于完成状态的map task在任务失败时重新执行，因为其输出储存在failed machine的本地硬盘上，对于执行reduce task的worker来说时不可达的。而处于完成状态的Reduce task不需要重新执行，因为输出储存在全局文件系统中。

当一个map task首先被A执行，在A failed之后被重新调度为被B执行时，，master将会通知所有的正在执行reduce task的worker。所有的还没有从A读取数据的reduce task都将会从B读取。

MapReduce可以容忍大量的worker failure。例如，在MapReduce计算过程中， 网络维护造成了80台机器在几分钟内同时离线。master将会选择合适的worker来重新执行被不可达的worker完成的map task，最终完成MapReduce.

#### Master Failure

很容易让master来周期性地保存上述master数据结构的checkpoint。如果master task离线，那么一个新的备份将会从上一个checkpointd state恢复。但是由于只有一个master，其故障地可能性并不大，因此如果master离线，会终止MapReduce运行。客户端可以检查这种情况并在需要时重新进入MapReduce操作。

#### Semantics in the Presence of Failures

当用户提供的map和reduce是关于输入的确定性函数时，我们的分布式实现将会产生与串行的程序相同的输出。

我们依靠原子化的map commits和reduce task的输出来实现这一特性。每个task将输出写入到私有的临时文件中。reduce task创建一个这样的文件，而map task创建R个这样的文件（*相当于为每个reduce task创建一个，并且这R个文件应该完全相同？*）当map task完成时，发送一条消息通知master，其中包含了R个临时文件的文件名。如果master收到了一个来自已经处于completed状态的map task的完成消息，将其忽略（*因为重复？为什么会引起重复？*）否则将会在master 数据结构中记录R个文件的名字。

当reduce task完成时，reduce worker原子地将临时输出文件重命名为最终的输出文件。如果相同的reduce task在多台机器上被执行，那么将会同时进行多个重命名操作，并且重命名的为一个文件。依赖于底层文件系统所提供的原子化的重命名操作来保证最终文件系统状态仅包含由一个reduce task执行所产生的数据。

（*我的疑惑在于，map创建了R个临时文件，这R个文件彼此之间是相同的吗？如果不是，这R个文件之间的关系是什么？如果是，那么每个reduce task都需要读取M个map task的输出，相当于每个reduce task进行的是重复计算，会出现相当大的冗余。对应地，reduce创建的是最终文件，那么相当于R个reduce每个创建了1/R个最终文件，拼起来才是完整的输出？*）

*map创建的R个临时文件，彼此并不相同，而是使用key % nReduce将其映射到不同的reduce worker来处理。这R个临时文件组合起来构成了这个map worker对于其输入文件处理后产生输出的集合。每个reduce读取与其task id相同的中间输出文件，并将文件中具有相同的key的value合并，输出mr-out-id文件，将所有的reduce worker的输出文件合并，就是完整的输出文件*

我们的map和reduce操作的大部分是确定性的，并且我们的MapReduce语义与顺序执行是等价的，因此对于程序员来说可以容易地理解他们程序的行为。当map和reduce是不确定的时，我们提供weaker but still reasonable语义。对于不确定的操作来说，reduce task R1的输出等价于由这个不确定程序顺序执行的R1的输出。然而，另一个不同的reduce task R2的输出可能对应于由非确定性程序的不同顺序执行产生的R2的输出。

 考虑map task M 和reduce task R1和R2，将已经commit的Ri的执行定义为e(Ri)。weaker的语义之所以出现，是因为e(R1)已经读取了M的一次执行产生的输出，而e(R2)可能已经读取了M的另一次不同执行的输出。

### 3.4 Locality

网络带宽在我们的计算环境中是一种稀缺的资源，因此利用输入数据（由GFS管理）保存在组成集群的机器的本地磁盘上这一事实来节省带宽。GFS将每个文件划分成64MB的块，并在不同的机器上保存每个块的备份（通常是每个块保存3个备份）。MapReduce master考虑输入文件的位置信息并且尝试在已经包含输入数据 副本的机器上调度map task（这样可以减少输入数据在不同机器之间的传输）。如果未能够成功调度，那么尝试在离具有副本的机器更近的worker上调度map task（例如选择与存有副本的机器连接到同一台交换机的worker）。在集群中的大部分worker上运行MapReduce时，大部分输入数据都是本地读取的，因此并不会产生网络带宽。

### 3.5 Task Granularity

我们将map阶段划分成M个片，将reduce阶段划分成R个片。理论上，M和R应该会比worker的数量大很多。让每个worker执行不同的task提升了动态负载均衡的效果，并且同样在一个worker离线时加速了恢复过程：这台离线的worker所完成的map任务可以分布在所有其他的worker上。（*我又不懂了：不同worker所执行的map之间的关系是什么？为什么说可以分布在其他的worker上？* *不同worker所执行的map之间是相互独立的，master可以任意地将一个task安排给不同的worker来执行*）

在实践上M和R有着固定的边界：因为master必须进行O（M+R）次调度并在内存中保存O（M\*R）个状态。（实际上内存占用的常量因子很小：O(M\*R)个状态由每个大约1字节的map task/reduce task对组成）。

此外，R通常被用户限制，因为每个reduce task的输出是一个单独的文件。实际上我们倾向于选择M以使每个单独的任务使用大约16MB到64MB的输入数据（从而与上述局部性优化有效），并将 R设置为需要使用的worker数量的比较小的倍数。在使用2000台worker时，通常将M设置为200000，R设置为5000.

### 3.6 Backup Tasks

一个常见的造成MapReduce总的执行时间延长的原因是，在计算过程中往往存在一个straggler，需要使用很长时间去完成最后几个map或reduce任务。产生straggler的原因有很多，例如由于硬盘损坏导致读性能从30MB/s降到1MB/s；集群调度系统可能调度其他task到这台机器上，由于CPU、内存、本地磁盘和网络带宽的竞争造成其执行MapReduce的效率降低。我们最近遇到的一个问题是由于机器初始化代码bug导致的处理器cache被禁用，受影响机器上的计算速度降低到不到原来的1%。

我们有一个通用的避免straggler问题的机制。当MapReduce接近完成时，master调度剩余的正在进行中任务的备份执行。在primary或是备份完成时，就将这个任务标记为已完成。我们调整了这个机制，使其一般情况下使用的计算资源增加不超过几个百分点。这显著减少了完成大型MapReduce操作的时间。例如如果禁用备份任务机制，那么5.3描述的排序任务所需的时间将会提升44%。

## 4 Refinements

尽管简单的Map和Reduce在大部分情况下已经足够有效，我们依然发现了一些有用的扩展。

### 4.1 Partioning Function

MapReduce的用户将reduce task/输出文件的值设置为R。通过在intermediate key上使用划分函数将数据划分到这些reduce task中。默认的划分函数是使用哈希算法hash(key) mod R. 哈希算法通常会产生公平的划分，但是在某些情况下，使用其他的关于key的函数会非常有效。例如，又是输出的key是URL，并且我们希望单个主机的所有条目都在同一个输出文件中。为了能够支持这种情况，用户可以提供特殊的划分函数，例如hash(Hostname(urlkey)) mod R，从而能够将具有相同host的所有的url都能够输出到同一个输出文件中。

### 4.2 Ordering Guarantees

我们确保对于给定的划分函数，中间键值对按照key的递增序被处理。这种顺序保证了每个划分都能够生成一个有序的输出文件，因而在输出文件格式需要支持高效的对于key随机访问查询或是用户需要排序后的数据时非常有效。

### 4.3 Combiner Function

在某些情况下，每个map task所产生的中间key具有大量的重复，而用户指定的reduce函数是可交换的和可关联的。一个例子是Section 2.1中提到的count，因为单词频率往往遵循Zipf分布（只有少数单词经常被使用，大部分的单词很少被使用），每个map task将会产生数百上千个<the, 1>这样的记录。所有的对于the的记录都将会在通过网络发送到一个reduce task然后被这个reduce task加到一起。因此用户可以通过指定一个Combiner函数，在map task将记录在发送到网络中之前首先进行部分合并，从而减少对于带宽的占用。

Combiner函数在每个运行map task的机器上执行，通常combiner和reduce函数的代码是一致的，二者之间唯一的差别是MapReduce如何处理函数的输出。reduce函数的输出被写入到最终的输出文件中，而combiner函数的输出被写入到map的中间文件中，然后被发送到reduce task作为reduce的输入。

实践证明部分合并能够极大提高某些类型的MapReduce的运行效率。

### 4.4 Input and Output Types

MapReduce提供了对于多种读入格式的支持。例如，text类型的输入将每一行视为键值对，key是当前行在文件中的偏移，而value是当前行的内容。另一种常见的受支持格式存储按键排序的键值对序列。每种输入类型的实现知道如何将其自身划分成有意义的范围，以便作为单独的映射任务进行处理，例如对于文本模式来说，范围划分确保仅在行边界处进行划分。用户可以通过提供reader接口的来实现对新的输入类型的支持，尽管大部分用户仅仅使用预定义的输入类型:slightly_smiling_face:。​

reader并不一定需要提供从文件读取数据的功能。例如，从数据库中读取记录或是从一个映射到内存的数据结构读取都是可行的。

输出类型与输入类似。

### 4.5 Side-effects

某些情况下，MapReduce用户发现生成辅助文件作为其map和reduce的附加输出非常方便。我们依靠应用程序编写者来使这些side-effects具有幂等性和原子性。通常，这些应用写入一个临时文件，并在写入完成后原子地将临时文件重命名。

我们不支持单个任务生成的多个输出文件的原子two-phase commits。因此，生成多个输出文件并且具有跨文件一致性要求的任务应该是确定性的。（*什么叫two-phase commits? 上文一直在说的commit指的是什么？* *我的理解是，对于map task，输出文件必须一次性写入到tempfile中，然后原子地对其重命名，对于reduce worker，同样将合并后的结果写入到mr-out文件中，而不允许先写入一半，等后续处理完后再提交另一半*)

### 4.6 Skipping Bad Records

用户代码中可能存在导致Map或Reduce函数在某些记录上崩溃的bug。这种错误将会导致MapReduce无法正确完成。我们提供了一种可选的执行模式，其中MapReduce库检测哪些记录会导致确定性崩溃并跳过这些记录以使程序继续向前推进。

每个worker进程安装一个能够捕获分段错误和总线错误的信号处理程序。在调用用户的Map或Reduce函数之前，MapReduce在全局变量中保存参数的序列号。如果用户代码生成了一个信号，信号处理程序向master发送一个包含了序列号的"last gasp" UDP报文。当master看到在某个记录上出现多次错误时，master在重新执行相应的Map或Reduce任务时将这个记录标记为应该被跳过的。

### 4.7 Local Execution

调试Map或者Reduce函数通常非常困难，因为实际的计算过程通常发生在包括数千台机器的分布式系统中，工作分配决策由master动态做出。为了帮助调试分析和小规模测试，我么你开发了MapReduce的替代实现，在本地机器上按顺序执行MapReduce的所有工作。控制前提供给用户，因此可以用来执行特定的map task。用户通过特定的flag调用他们的程序，并因此可以使用任何调试或测试工具来帮助测试。

### 4.8 Status Information

master运行一个内部的HTTP服务器，并导出一组状态页面以供使用。状态页面显示了当前计算的进度，例如多少个任务已经完成，多少个进程正在执行，输入的字节数，中间数据的字节数，输出数据的字节数，处理速率等。同样包含到每个任务生成的标准错误和标准输出文件的的链接。用户可以使用这些数据来预测计算需要多长时间，以及是否应该添加更多的计算资源，还可以帮助确定是否计算比预期更慢。

另外，顶层的状态页面显示哪个worker处于离线状态以及离线时正在处理的map/reduce任务。这一信息可以用来帮助查找用户代码中的bug。

## Section 4.9 & Chapter 5 & 6 & 7 & 8

略。



## Lab 1 MapReduce 设计与实现

### coordinator

coordinator的作用为论文中提到的master，用于为worker分配任务和处理错误。主要包括几个部分：

#### 1. RPC

1. Get(args int, reply *GetTaskReply) error 

   用于处理worker对于任务的请求，调用checkTask来选择一个未完成的任务分配给worker。

2. Register (args UNUSED, reply *int) error

   worker启动时调用Register在Coordinator中完成注册，UNUSED为空结构体，表示不需要参数

3. MapReport(args MapReport, reply *UNUSED) error

   在Map任务完成后，调用MapReport通知Coordinator任务已完成，使用Coordinator分配的任务号作为参数。Coordinator在收到report后，将对应的任务状态设置为END，并将输入文件状态设置为FINISHED。由于MapReduce要求所有的Map任务完成后再开始Reduce任务，因此Coordinator需要遍历MapTask来判断是否进行任务阶段的切换。如果所有的任务都已完成，则调用awakeRoutine来唤醒正处于等待状态的worker（如果有）进入下一阶段的处理。

4. ReduceReport(args ReduceReport, reply *UNUSED) error

   Coordinator收到ReduceReport后，首先将对应的任务状态设置为END，然后检查ReduceTask，如果所有的任务都已经完成，则将done设置为true，表示MapReduce已经完成。

#### 2. schedule

1. tickSchedule()

   周期性启动一个goroutine来执行schedule，直到所有的任务都已完成为止。

2. schedule()

   遍历TaskMap，判断每个任务是否已经超时，如果超时，则将对应的任务状态设置为FAILED，并调用taskReschedule函数选择一个worker来重新运行此任务。

3. awakeRoutine()

   仅需调用c.cond.Broadcast()来唤醒所有在此条件变量上wait的goroutine，goroutine被唤醒后自行调用checkTask来选择合适的任务交付给worker执行。

4. taskReschedule(taskid int)

   仅当schedule发现某个任务运行超时时被调用，因此taskReschedule的主要功能就是重置任务状态，并调用awakeRoutine唤醒等待中的routine。

5. checkTask(workerid int, reply *GetTaskReply)

   这是整个Coordinator中设计最为复杂的一部分。

   我分别设置了ReduceTask和MapTask用于跟踪对应的任务状态， 其中分别有nReduce和len(fileMap)个任务，因为Reduce Task最多有nReduce个，MapTask的数量与输入的文件相同。同时将其中的任务初始化为UNSTARTED。

   那么如何从MapTask和ReduceTask中选取合适的任务？首先定义了mCounter和rCounter分别记录已经分配的任务数，如果任务数没有超过最大允许的任务数量，那么很简单，只需要使用counter作为索引，将对应的任务设置为RUNNING，并传给worker任务号和文件名即可。如果任务数超过了最大允许的任务数量，那么说明前面分配的任务出现了FAILED的情况，因此遍历TaskMap，找出对应的任务分配给worker执行。如果没有找到，说明当前所有的任务都已经有worker执行，使用cond.Wait()将当前goroutine阻塞，等待schedule发现任务超时时唤醒。

#### 如何处理并发

因为coordinator结构体中大部分都是被多个协程共享的变量，因此需要使用sync.Mutex来实现互斥。go 语言标准库中的RPC会在连接到来时自动启动一个协程来处理，因此实际上需要自己启动协程的部分只有tickSchedule和schedule两部分。server中使用go http.Serve启动一个协程来监听worker发来的请求，MakeCoordinator使用go tickSchedule来周期性调用go schedule，检查任务执行情况。

一个困扰我很久的问题是在worker的请求到来时，如果没有合适的任务应该怎么处理？一开始使用channel机制来完成阻塞，为每一个任务初始化一个channel，在awakeRoutine中使用channel <-true来唤醒阻塞的routine，但是在测试时遇到了一个问题，如果采用容量为1的channel，那么会存在上一个写入的true没有取出，而新的awakeRoutine又要写入新的内容的情况，这样就会导致整个程序死锁。但是如果采用容量不为1的channel，一方面容量不好确定，另一方面无法保证阻塞。最后选择了使用条件变量来完成这一功能。遇到的另一个坑在于，cond.Wait()在结束时会自动加锁，然后调用checkTask函数进行分配，而我们的checkTask在一开始也会加锁，因此又会产生死锁。所以需要在checkTask之前手动进行解锁。

### Worker

worker在接收RPC分配的任务后，本质上是串行的，因此总体上比较简单，只需要死循环，不断地请求任务、完成任务、报告（什么牛马）。

在debug的过程中，我才真正理解了读论文时感到迷惑的Map的输出和Reduce的输入的关系。我们假设有M个Map任务，R个Reduce任务，那么每个Map都需要产生R个中间文件，其中保存的是Map处理后产生的键值对，每个键值对的key通过ihash产生一个编号（编号的范围是0~R），这个编号就对应的是后续对其进行处理的Reduce Task的编号，按照映射将键值对写入到合适的输出中。而在Reduce时，就需要遍历M个Map的输出，将其产生的与当前任务编号对应的输出文件读取、排序、合并、处理，最后输出到out文件中。由于没有理解mrsequential中Reduce的输出过程，我刚开始的sort是在Map中sort的，而Reduce就无法正确地将具有相同key的来自不同Map的键值对合并，从而导致了indexer和wc test失败。

## Reference

[6.824 Lab 1: MapReduce (mit.edu)](http://nil.csail.mit.edu/6.824/2022/labs/lab-mr.html)

[RPC and Threads](http://nil.csail.mit.edu/6.824/2022/notes/l-rpc.txt)

[Debugging by Pretty Printing (josejg.com)](https://blog.josejg.com/debugging-pretty/)