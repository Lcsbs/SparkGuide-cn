# Spark Streaming编程指南

- [总览](http://spark.apache.org/docs/latest/streaming-programming-guide.html#overview)
- [一个简单的例子](http://spark.apache.org/docs/latest/streaming-programming-guide.html#a-quick-example)
- 基本概念
  - [连结中](http://spark.apache.org/docs/latest/streaming-programming-guide.html#linking)
  - [初始化StreamingContext](http://spark.apache.org/docs/latest/streaming-programming-guide.html#initializing-streamingcontext)
  - [离散流（DStreams）](http://spark.apache.org/docs/latest/streaming-programming-guide.html#discretized-streams-dstreams)
  - [输入DStreams和接收器](http://spark.apache.org/docs/latest/streaming-programming-guide.html#input-dstreams-and-receivers)
  - [DStreams上的转换](http://spark.apache.org/docs/latest/streaming-programming-guide.html#transformations-on-dstreams)
  - [DStreams上的输出操作](http://spark.apache.org/docs/latest/streaming-programming-guide.html#output-operations-on-dstreams)
  - [DataFrame和SQL操作](http://spark.apache.org/docs/latest/streaming-programming-guide.html#dataframe-and-sql-operations)
  - [MLlib操作](http://spark.apache.org/docs/latest/streaming-programming-guide.html#mllib-operations)
  - [缓存/持久化](http://spark.apache.org/docs/latest/streaming-programming-guide.html#caching--persistence)
  - [检查点](http://spark.apache.org/docs/latest/streaming-programming-guide.html#checkpointing)
  - [累加器，广播变量和检查点](http://spark.apache.org/docs/latest/streaming-programming-guide.html#accumulators-broadcast-variables-and-checkpoints)
  - [部署应用](http://spark.apache.org/docs/latest/streaming-programming-guide.html#deploying-applications)
  - [监控应用](http://spark.apache.org/docs/latest/streaming-programming-guide.html#monitoring-applications)
- 性能调优
  - [减少批处理时间](http://spark.apache.org/docs/latest/streaming-programming-guide.html#reducing-the-batch-processing-times)
  - [设置正确的批次间隔](http://spark.apache.org/docs/latest/streaming-programming-guide.html#setting-the-right-batch-interval)
  - [内存调优](http://spark.apache.org/docs/latest/streaming-programming-guide.html#memory-tuning)
- [容错语义](http://spark.apache.org/docs/latest/streaming-programming-guide.html#fault-tolerance-semantics)
- [从这往哪儿走](http://spark.apache.org/docs/latest/streaming-programming-guide.html#where-to-go-from-here)

# 总览

Spark Streaming是核心Spark API的扩展，可实现实时数据流的可扩展，高吞吐量，容错流处理。数据可以从像Kafka，Kinesis，或TCP sockets许多来源摄入，并且可以使用与像高级别功能表达复杂的算法来处理`map`，`reduce`，`join`和`window`。最后，可以将处理后的数据推送到文件系统，数据库和实时仪表板。实际上，您可以在数据流上应用Spark的 [机器学习](http://spark.apache.org/docs/latest/ml-guide.html)和 [图形处理](http://spark.apache.org/docs/latest/graphx-programming-guide.html)算法。

![火花流](http://spark.apache.org/docs/latest/img/streaming-arch.png)

在内部，它的工作方式如下。Spark Streaming接收实时输入数据流，并将数据分成批处理，然后由Spark引擎进行处理，以生成批处理的最终结果流。

![火花流](http://spark.apache.org/docs/latest/img/streaming-flow.png)

Spark Streaming提供了称为*离散流*或*DStream*的高级抽象，它表示连续的数据流。可以根据来自Kafka和Kinesis等来源的输入数据流来创建DStream，也可以通过对其他DStream应用高级操作来创建DStream。在内部，DStream表示为[RDD](http://spark.apache.org/docs/latest/api/scala/org/apache/spark/rdd/RDD.html)序列 。

本指南向您展示如何开始使用DStreams编写Spark Streaming程序。您可以使用Scala，Java或Python（Spark 1.2中引入）编写Spark Streaming程序，本指南中介绍了所有这些程序。在本指南中，您会找到一些选项卡，可让您在不同语言的代码段之间进行选择。

**注意：**有一些API可能不同，或者在Python中不可用。在本指南中，您会发现**Python API**标签突出了这些差异。

------

# 一个简单的例子

在详细介绍如何编写自己的Spark Streaming程序之前，让我们快速看一下简单的Spark Streaming程序的外观。假设我们要计算从侦听TCP sockets的数据服务器接收到的文本数据中的单词数。您需要做的如下。

- [**Scala**](http://spark.apache.org/docs/latest/streaming-programming-guide.html#tab_scala_0)
- [**Java**](http://spark.apache.org/docs/latest/streaming-programming-guide.html#tab_java_0)
- [**Python**](http://spark.apache.org/docs/latest/streaming-programming-guide.html#tab_python_0)

首先，我们将Spark Streaming类的名称以及从StreamingContext进行的一些隐式转换导入到我们的环境中，以便为我们需要的其他类（如DStream）添加有用的方法。[StreamingContext](http://spark.apache.org/docs/latest/api/scala/org/apache/spark/streaming/StreamingContext.html)是所有流功能的主要入口点。我们创建具有两个执行线程和1秒批处理间隔的本地StreamingContext。

```scala
import org.apache.spark._
import org.apache.spark.streaming._
import org.apache.spark.streaming.StreamingContext._ // not necessary since Spark 1.3

// Create a local StreamingContext with two working thread and batch interval of 1 second.
// The master requires 2 cores to prevent a starvation scenario.

val conf = new SparkConf().setMaster("local[2]").setAppName("NetworkWordCount")
val ssc = new StreamingContext(conf, Seconds(1))
```

使用此上下文，我们可以创建一个DStream，它表示来自TCP源的流数据，指定为主机名（例如`localhost`）和端口（例如`9999`）。

```scala
// Create a DStream that will connect to hostname:port, like localhost:9999
val lines = ssc.socketTextStream("localhost", 9999)
```

此`lines`DStream表示将从数据服务器接收的数据流。此DStream中的每个记录都是一行文本。接下来，我们要用空格将行分割成单词。

```scala
// Split each line into words
val words = lines.flatMap(_.split(" "))
```

`flatMap`是一对多DStream操作，它通过从源DStream中的每个记录生成多个新记录来创建新的DStream。在这种情况下，每行将拆分为多个单词，单词流表示为 `words`DStream。接下来，我们要计算这些单词。

```scala
import org.apache.spark.streaming.StreamingContext._ // not necessary since Spark 1.3
// Count each word in each batch
val pairs = words.map(word => (word, 1))
val wordCounts = pairs.reduceByKey(_ + _)

// Print the first ten elements of each RDD generated in this DStream to the console
wordCounts.print()
```

将`words`DStream进一步映射（一对一转换）到`(word, 1)`成对的DStream，然后将其减小以获取每批数据中单词的频率。最后，`wordCounts.print()`将打印每秒产生的一些计数。

请注意，执行这些行时，Spark Streaming仅设置启动时将执行的计算，并且尚未开始任何实际处理。在完成所有转换后，要开始处理，我们最终调用

```scala
ssc.start()             // Start the computation
ssc.awaitTermination()  // Wait for the computation to terminate
```

完整的代码可以在Spark Streaming示例 [NetworkWordCount中找到](https://github.com/apache/spark/blob/v3.0.1/examples/src/main/scala/org/apache/spark/examples/streaming/NetworkWordCount.scala)。

如果您已经[下载](http://spark.apache.org/docs/latest/index.html#downloading)并[构建了](http://spark.apache.org/docs/latest/index.html#building)Spark，则可以按以下方式运行此示例。您首先需要通过使用以下命令将Netcat（在大多数类Unix系统中找到的一个小实用程序）作为数据服务器运行

```scala
$ nc -lk 9999
```

然后，在另一个终端中，您可以通过使用

- [**Scala**](http://spark.apache.org/docs/latest/streaming-programming-guide.html#tab_scala_1)
- [**Java**](http://spark.apache.org/docs/latest/streaming-programming-guide.html#tab_java_1)
- [**Python**](http://spark.apache.org/docs/latest/streaming-programming-guide.html#tab_python_1)

```scala
$ ./bin/run-example streaming.NetworkWordCount localhost 9999
```

然后，将对运行netcat服务器的终端中键入的任何行进行计数并每秒打印一次。它看起来像以下内容。

| `# TERMINAL 1: # Running Netcat $ nc -lk 9999 hello world  ...` |      | [**Scala**](http://spark.apache.org/docs/latest/streaming-programming-guide.html#tab_scala_2)[**Java**](http://spark.apache.org/docs/latest/streaming-programming-guide.html#tab_java_2)[**Python**](http://spark.apache.org/docs/latest/streaming-programming-guide.html#tab_python_2)`# TERMINAL 2: RUNNING NetworkWordCount $ ./bin/run-example streaming.NetworkWordCount localhost 9999 ... ------------------------------------------- Time: 1357008430000 ms ------------------------------------------- (hello,1) (world,1) ...` |
| ------------------------------------------------------------ | ---- | ------------------------------------------------------------ |
|                                                              |      |                                                              |

------

------

# 基本概念

接下来，我们将超越简单的示例，并详细介绍Spark Streaming的基础。

## 连结中

与Spark相似，可以通过Maven Central使用Spark Streaming。要编写自己的Spark Streaming程序，您必须将以下依赖项添加到SBT或Maven项目中。

- [**马文**](http://spark.apache.org/docs/latest/streaming-programming-guide.html#tab_Maven_3)
- [**SBT**](http://spark.apache.org/docs/latest/streaming-programming-guide.html#tab_SBT_3)

```xml
<dependency>
    <groupId>org.apache.spark</groupId>
    <artifactId>spark-streaming_2.12</artifactId>
    <version>3.0.1</version>
    <scope>provided</scope>
</dependency>
```

要从Spark Streaming核心API中不存在的，从诸如Kafka和Kinesis之类的源中获取数据，您必须将相应的工件添加`spark-streaming-xyz_2.12`到依赖项中。例如，一些常见的如下。

| 资源   | 神器                                              |
| :----- | :------------------------------------------------ |
| Kafka  | spark-streaming-kafka-0-10_2.12                   |
| 运动学 | spark-streaming-kinesis-asl_2.12 [Amazon软件许可] |
|        |                                                   |

有关最新列表，请参阅 [Maven存储库](https://search.maven.org/#search|ga|1|g%3A"org.apache.spark" AND v%3A"3.0.1") ，以获取受支持的源和工件的完整列表。

------

## 初始化StreamingContext

要初始化Spark Streaming程序，必须创建**StreamingContext**对象，该对象是所有Spark Streaming功能的主要入口点。

- [**Scala**](http://spark.apache.org/docs/latest/streaming-programming-guide.html#tab_scala_4)
- [**Java**](http://spark.apache.org/docs/latest/streaming-programming-guide.html#tab_java_4)
- [**Python**](http://spark.apache.org/docs/latest/streaming-programming-guide.html#tab_python_4)

甲[的StreamingContext](http://spark.apache.org/docs/latest/api/scala/org/apache/spark/streaming/StreamingContext.html)对象可以从被创建[SparkConf](http://spark.apache.org/docs/latest/api/scala/org/apache/spark/SparkConf.html)对象。

```
import org.apache.spark._
import org.apache.spark.streaming._

val conf = new SparkConf().setAppName(appName).setMaster(master)
val ssc = new StreamingContext(conf, Seconds(1))
```

该`appName`参数是您的应用程序显示在集群UI上的名称。 `master`是[Spark，Mesos，Kubernetes或YARN群集URL](http://spark.apache.org/docs/latest/submitting-applications.html#master-urls)，或者是特殊的**“ local [\*]”**字符串以在本地模式下运行。实际上，当在集群上运行时，您将不希望`master`在程序中进行硬编码，而是在其中[启动应用程序`spark-submit`](http://spark.apache.org/docs/latest/submitting-applications.html)并在其中接收。但是，对于本地测试和单元测试，您可以传递“ local [*]”以在内部运行Spark Streaming（检测本地系统中的内核数）。请注意，这会在内部创建一个[SparkContext](http://spark.apache.org/docs/latest/api/scala/org/apache/spark/SparkContext.html)（所有Spark功能的起点），可以通过访问`ssc.sparkContext`。

必须根据应用程序的延迟要求和可用群集资源来设置批处理间隔。有关 更多详细信息，请参见[性能调整](http://spark.apache.org/docs/latest/streaming-programming-guide.html#setting-the-right-batch-interval)部分。

甲`StreamingContext`目的还可以从现有的创建的`SparkContext`对象。

```
import org.apache.spark.streaming._

val sc = ...                // existing SparkContext
val ssc = new StreamingContext(sc, Seconds(1))
```

定义上下文后，必须执行以下操作。

1. 通过创建输入DStream定义输入源。
2. 通过将转换和输出操作应用于DStream来定义流计算。
3. 开始接收数据并使用进行处理`streamingContext.start()`。
4. 等待使用停止处理（手动或由于任何错误）`streamingContext.awaitTermination()`。
5. 可以使用手动停止处理`streamingContext.stop()`。

##### 要记住的要点：

- 一旦启动上下文，就无法设置新的流计算或将其添加到该流计算中。
- 上下文一旦停止，就无法重新启动。
- JVM中只能同时激活一个StreamingContext。
- StreamingContext上的stop（）也会停止SparkContext。要仅停止的StreamingContext，设置可选的参数`stop()`叫做`stopSparkContext`假。
- 只要在创建下一个StreamingContext之前停止（而不停止SparkContext）上一个StreamingContext，即可将SparkContext重用于创建多个StreamingContext。

------

## 离散流（DStreams）

**离散流**或**DStream**是Spark Streaming提供的基本抽象。它表示连续的数据流，可以是从源接收的输入数据流，也可以是通过转换输入流生成的已处理数据流。在内部，DStream由一系列连续的RDD表示，这是Spark对不可变的分布式数据集的抽象（有关更多详细信息，请参见[Spark编程指南](http://spark.apache.org/docs/latest/rdd-programming-guide.html#resilient-distributed-datasets-rdds)）。DStream中的每个RDD都包含来自特定间隔的数据，如下图所示。

![火花流](http://spark.apache.org/docs/latest/img/streaming-dstream.png)

在DStream上执行的任何操作都转换为对基础RDD的操作。例如，在将行流转换为单词的[较早示例](http://spark.apache.org/docs/latest/streaming-programming-guide.html#a-quick-example)中，该`flatMap`操作应用于`lines`DStream中的每个RDD，以生成DStream的 `words`RDD。如下图所示。

![火花流](http://spark.apache.org/docs/latest/img/streaming-dstream-ops.png)

这些基础的RDD转换由Spark引擎计算。DStream操作隐藏了大多数这些细节，并为开发人员提供了更高级别的API，以方便使用。这些操作将在后面的部分中详细讨论。

------

## 输入DStreams和接收器

输入DStream是表示从流源接收的输入数据流的DStream。在[快速示例中](http://spark.apache.org/docs/latest/streaming-programming-guide.html#a-quick-example)，`lines`输入DStream代表从netcat服务器接收的数据流。每个输入DStream（文件流除外，本节稍后将讨论）都与一个**Receiver对象** （[Scala doc](http://spark.apache.org/docs/latest/api/scala/org/apache/spark/streaming/receiver/Receiver.html)， [Java doc](http://spark.apache.org/docs/latest/api/java/org/apache/spark/streaming/receiver/Receiver.html)）关联，该对象从源接收数据并将其存储在Spark的内存中以进行处理。

Spark Streaming提供了两类内置的流媒体源。

- *基本来源*：可直接在StreamingContext API中获得的来源。示例：文件系统和套接字连接。
- *高级资源*：可以通过其他实用程序类获得诸如Kafka，Kinesis等资源。如[链接](http://spark.apache.org/docs/latest/streaming-programming-guide.html#linking)部分所述，这些要求针对额外的依赖项进行 [链接](http://spark.apache.org/docs/latest/streaming-programming-guide.html#linking)。

我们将在本节后面的每个类别中讨论一些资源。

请注意，如果要在流应用程序中并行接收多个数据流，则可以创建多个输入DStream（在“[性能调整”](http://spark.apache.org/docs/latest/streaming-programming-guide.html#level-of-parallelism-in-data-receiving)部分中进一步讨论）。这将创建多个接收器，这些接收器将同时接收多个数据流。但是请注意，Spark工作程序/执行程序是一项长期运行的任务，因此它占用了分配给Spark Streaming应用程序的核心之一。因此，重要的是要记住，需要为Spark Streaming应用程序分配足够的内核（或线程，如果在本地运行），以处理接收到的数据以及运行接收器。

##### 要记住的要点

- 在本地运行Spark Streaming程序时，请勿使用“ local”或“ local [1]”作为主URL。这两种方式均意味着仅一个线程将用于本地运行任务。如果您使用基于接收方的输入DStream（例如套接字，Kafka等），则将使用单个线程来运行接收方，而不会留下任何线程来处理接收到的数据。因此，在本地运行时，请始终使用“ local [ *n* ]”作为主URL，其中*n* >要运行的接收者数（有关如何设置主服务器的信息，请参见[Spark属性](http://spark.apache.org/docs/latest/configuration.html#spark-properties)）。
- 为了将逻辑扩展到在集群上运行，分配给Spark Streaming应用程序的内核数必须大于接收器数。否则，系统将接收数据，但无法处理它。

### 基本资料

我们已经`ssc.socketTextStream(...)`在[快速示例中](http://spark.apache.org/docs/latest/streaming-programming-guide.html#a-quick-example)查看了，该[示例](http://spark.apache.org/docs/latest/streaming-programming-guide.html#a-quick-example) 根据通过TCP套接字连接接收的文本数据创建DStream。除了套接字外，StreamingContext API还提供了从文件作为输入源创建DStream的方法。

#### 文件流

要从与HDFS API兼容的任何文件系统（即HDFS，S3，NFS等）上的文件中读取数据，可以通过创建DStream `StreamingContext.fileStream[KeyClass, ValueClass, InputFormatClass]`。

文件流不需要运行接收器，因此不需要分配任何内核来接收文件数据。

对于简单的文本文件，最简单的方法是`StreamingContext.textFileStream(dataDirectory)`。

- [**Scala**](http://spark.apache.org/docs/latest/streaming-programming-guide.html#tab_scala_5)
- [**Java**](http://spark.apache.org/docs/latest/streaming-programming-guide.html#tab_java_5)
- [**Python**](http://spark.apache.org/docs/latest/streaming-programming-guide.html#tab_python_5)

```
streamingContext.fileStream[KeyClass, ValueClass, InputFormatClass](dataDirectory)
```

对于文本文件

```
streamingContext.textFileStream(dataDirectory)
```

##### 如何监控目录

Spark Streaming将监视目录`dataDirectory`并处理在该目录中创建的所有文件。

- 可以监视一个简单目录，例如`"hdfs://namenode:8040/logs/"`。发现后，将直接处理该路径下的所有文件。
- 甲[POSIX glob模式](http://pubs.opengroup.org/onlinepubs/009695399/utilities/xcu_chap02.html#tag_02_13_02)可以被提供，例如 `"hdfs://namenode:8040/logs/2017/*"`。在这里，DStream将包含与模式匹配的目录中的所有文件。也就是说：它是目录的模式，而不是目录中的文件。
- 所有文件必须具有相同的数据格式。
- 根据文件的修改时间而不是创建时间，将其视为时间段的一部分。
- 处理后，在当前窗口中对文件的更改不会导致重新读取该文件。也就是说：*忽略更新*。
- 目录下的文件越多，扫描更改所需的时间就越长-即使未修改任何文件。
- 如果使用通配符来标识目录（例如）`"hdfs://namenode:8040/logs/2016-*"`，则重命名整个目录以匹配路径会将目录添加到受监视目录列表中。流中仅包含目录中修改时间在当前窗口内的文件。
- 调用[`FileSystem.setTimes()`](https://hadoop.apache.org/docs/current/api/org/apache/hadoop/fs/FileSystem.html#setTimes-org.apache.hadoop.fs.Path-long-long-) 修复时间戳是一种在以后的窗口中拾取文件的方法，即使其内容没有更改。

##### 使用对象存储作为数据源

HDFS之类的“完整”文件系统倾向于在创建输出流后立即对其文件设置修改时间。当打开文件时，甚至在完全写入数据之前，它也可能包含在`DStream`-之后，将忽略同一窗口中对该文件的更新。也就是说：更改可能会丢失，流中会省略数据。

为确保在窗口中进行更改，请将文件写入一个不受监视的目录，然后在关闭输出流后立即将其重命名为目标目录。如果重命名的文件在创建窗口期间出现在扫描的目标目录中，则将提取新数据。

相反，由于实际复制了数据，因此诸如Amazon S3和Azure存储之类的对象存储通常具有较慢的重命名操作。此外，重命名的对象可能具有`rename()`操作时间作为其修改时间，因此可能不被视为原始创建时间所暗示的窗口的一部分。

需要对目标对象存储进行仔细的测试，以验证存储的时间戳行为与Spark Streaming期望的一致。直接写入目标目录可能是通过所选对象存储流传输数据的适当策略。

有关此主题的更多详细信息，请参阅[Hadoop Filesystem Specification](https://hadoop.apache.org/docs/stable2/hadoop-project-dist/hadoop-common/filesystem/introduction.html)。

#### 基于自定义接收器的流

可以使用通过自定义接收器接收的数据流来创建DStream。有关更多详细信息，请参见《[定制接收器指南》](http://spark.apache.org/docs/latest/streaming-custom-receivers.html)。

#### RDD队列作为流

为了使用测试数据测试Spark Streaming应用程序，还可以使用，基于RDD队列创建DStream `streamingContext.queueStream(queueOfRDDs)`。推送到队列中的每个RDD将被视为DStream中的一批数据，并像流一样进行处理。

有关套接字和文件中流的更多详细信息，请参阅[StreamingContext](http://spark.apache.org/docs/latest/api/scala/org/apache/spark/streaming/StreamingContext.html) for Scala，[JavaStreamingContext](http://spark.apache.org/docs/latest/api/java/index.html?org/apache/spark/streaming/api/java/JavaStreamingContext.html) for Java和[StreamingContext](http://spark.apache.org/docs/latest/api/python/pyspark.streaming.html#pyspark.streaming.StreamingContext) for Python中相关功能的API文档 。

### 进阶资源

**Python API**从Spark 3.0.1开始，在这些来源中，Python API中提供了Kafka和Kinesis。

这类资源需要与外部非Spark库进行接口，其中一些库具有复杂的依存关系（例如，Kafka）。因此，为了最大程度地减少与依赖项版本冲突相关的问题，从这些源创建DStream的功能已移至单独的库，可以在必要时显式[链接](http://spark.apache.org/docs/latest/streaming-programming-guide.html#linking)到这些库。

请注意，这些高级源在Spark Shell中不可用，因此无法在Shell中测试基于这些高级源的应用程序。如果您真的想在Spark shell中使用它们，则必须下载相应的Maven工件的JAR及其依赖项，并将其添加到类路径中。

这些高级资源如下。

- **Kafka：** Spark Streaming 3.0.1与0.10或更高版本的Kafka代理兼容。有关更多详细信息，请参见《[Kafka集成指南》](http://spark.apache.org/docs/latest/streaming-kafka-0-10-integration.html)。
- **Kinesis：** Spark Streaming 3.0.1与Kinesis Client Library 1.2.1兼容。有关更多详细信息，请参见《[Kinesis集成指南》](http://spark.apache.org/docs/latest/streaming-kinesis-integration.html)。

### 自订来源

**Python API Python**尚不支持此功能。

输入DStreams也可以从自定义数据源中创建。您所需要做的就是实现一个用户定义的**接收器**（请参阅下一节以了解其含义），该接收器可以接收来自自定义源的数据并将其推送到Spark中。有关详细信息，请参见《[定制接收器指南](http://spark.apache.org/docs/latest/streaming-custom-receivers.html)》。

### 接收器可靠性

根据数据*可靠性，*可以有两种数据源。源（如Kafka）允许确认已传输的数据。如果从这些*可靠*来源接收数据的系统正确地确认了接收到的数据，则可以确保不会由于任何类型的故障而丢失任何数据。这导致两种接收器：

1. *可靠的接收器*-*可靠的接收器*在接收到数据并通过复制将其存储在Spark中后，可以正确地将确认发送到可靠的源。
2. *不可靠的接收器*-一个*不可靠的接收器*并*没有*发送确认的资源等。可以将其用于不支持确认的来源，甚至可以用于不希望或不需要进入确认复杂性的可靠来源。

《[定制接收器指南》](http://spark.apache.org/docs/latest/streaming-custom-receivers.html)中讨论了如何编写可靠的接收器的详细信息 。

------

## DStreams上的转换

与RDD相似，转换允许修改来自输入DStream的数据。DStream支持普通Spark RDD上可用的许多转换。一些常见的方法如下。

| 转型                                        | 含义                                                         |
| :------------------------------------------ | :----------------------------------------------------------- |
| **地图**（*func*）                          | 通过将源DStream的每个元素传递给函数*func来*返回新的DStream 。 |
| **flatMap**（*func*）                       | 与map相似，但是每个输入项可以映射到0个或多个输出项。         |
| **过滤器**（*func*）                        | 通过仅选择*func*返回true的源DStream的记录来返回新的DStream 。 |
| **重新分区**（*numPartitions*）             | 通过创建更多或更少的分区来更改此DStream中的并行度。          |
| **联合**（*otherStream*）                   | 返回一个新的DStream，其中包含源DStream和*otherDStream*中的元素的并 *集*。 |
| **数**（）                                  | 通过计算源DStream的每个RDD中的元素数，返回一个新的单元素RDD DStream。 |
| **减少**（*func*）                          | 通过使用函数*func*（带有两个参数并返回一个）来聚合源DStream的每个RDD中的元素，从而返回一个单元素RDD的新DStream 。该函数应具有关联性和可交换性，以便可以并行计算。 |
| **countByValue**（）                        | 在类型为K的元素的DStream上调用时，返回一个新的（K，Long）对的DStream，其中每个键的值是其在源DStream的每个RDD中的频率。 |
| **reduceByKey**（*func*，[ *numTasks* ]）   | 在（K，V）对的DStream上调用时，返回一个新的（K，V）对的DStream，其中使用给定的reduce函数聚合每个键的值。**注意：**默认情况下，这使用Spark的默认并行任务数（本地模式为2，而在集群模式下，此数量由config属性确定`spark.default.parallelism`）进行分组。您可以传递一个可选`numTasks`参数来设置不同数量的任务。 |
| **加入**（*otherStream*，[ *numTasks* ]）   | 在（K，V）和（K，W）对的两个DStream上调用时，返回一个新的（K，（V，W））对的DStream，其中每个键都有所有元素对。 |
| **协同组**（*otherStream*，[ *numTasks* ]） | 在（K，V）和（K，W）对的DStream上调用时，返回一个新的（K，Seq [V]，Seq [W]）元组的DStream。 |
| **转换**（*func*）                          | 通过将RDD-to-RDD函数应用于源DStream的每个RDD，返回一个新的DStream。这可用于在DStream上执行任意RDD操作。 |
| **updateStateByKey**（*func*）              | 返回一个新的“状态” DStream，在该DStream中，通过在键的先前状态和键的新值上应用给定函数来更新每个键的状态。这可用于维护每个键的任意状态数据。 |
|                                             |                                                              |

其中一些转换值得更详细地讨论。

#### UpdateStateByKey操作

该`updateStateByKey`操作使您可以保持任意状态，同时不断用新信息更新它。要使用此功能，您将必须执行两个步骤。

1. 定义状态-状态可以是任意数据类型。
2. 定义状态更新功能-使用功能指定如何使用输入流中的先前状态和新值来更新状态。

在每个批次中，Spark都会对所有现有密钥应用状态更新功能，无论它们是否在批次中具有新数据。如果更新函数返回，`None`则将删除键值对。

让我们用一个例子来说明。假设您要保持在文本数据流中看到的每个单词的连续计数。此处，运行计数是状态，它是整数。我们将更新函数定义为：

- [**Scala**](http://spark.apache.org/docs/latest/streaming-programming-guide.html#tab_scala_6)
- [**Java**](http://spark.apache.org/docs/latest/streaming-programming-guide.html#tab_java_6)
- [**Python**](http://spark.apache.org/docs/latest/streaming-programming-guide.html#tab_python_6)

```
def updateFunction(newValues: Seq[Int], runningCount: Option[Int]): Option[Int] = {
    val newCount = ...  // add the new values with the previous running count to get the new count
    Some(newCount)
}
```

这适用于包含单词的DStream（例如，在[前面的示例中](http://spark.apache.org/docs/latest/streaming-programming-guide.html#a-quick-example)`pairs`包含`(word, 1)`对的DStream ）。

```
val runningCounts = pairs.updateStateByKey[Int](updateFunction _)
```

将为每个单词调用更新函数，每个单词`newValues`的序列为1（来自各`(word, 1)`对），并`runningCount`具有先前的计数。

请注意，使用`updateStateByKey`需要配置检查点目录，这将在[检查点](http://spark.apache.org/docs/latest/streaming-programming-guide.html#checkpointing)部分中详细讨论。

#### 转型运营

该`transform`操作（以及类似的变体`transformWith`）允许将任意RDD-to-RDD函数应用于DStream。它可用于应用DStream API中未公开的任何RDD操作。例如，将数据流中的每个批次与另一个数据集连接在一起的功能未直接在DStream API中公开。但是，您可以轻松地使用它`transform`来执行此操作。这实现了非常强大的可能性。例如，可以通过将输入数据流与预先计算的垃圾邮件信息（也可能由Spark生成）结合在一起，然后基于该信息进行过滤来进行实时数据清除。

- [**Scala**](http://spark.apache.org/docs/latest/streaming-programming-guide.html#tab_scala_7)
- [**Java**](http://spark.apache.org/docs/latest/streaming-programming-guide.html#tab_java_7)
- [**Python**](http://spark.apache.org/docs/latest/streaming-programming-guide.html#tab_python_7)

```
val spamInfoRDD = ssc.sparkContext.newAPIHadoopRDD(...) // RDD containing spam information

val cleanedDStream = wordCounts.transform { rdd =>
  rdd.join(spamInfoRDD).filter(...) // join data stream with spam information to do data cleaning
  ...
}
```

请注意，在每个批处理间隔中都会调用提供的函数。这使您可以执行随时间变化的RDD操作，即可以在批之间更改RDD操作，分区数，广播变量等。

#### 窗口操作

Spark Streaming还提供了*窗口计算*，可让您在数据的滑动窗口上应用转换。下图说明了此滑动窗口。

![火花流](http://spark.apache.org/docs/latest/img/streaming-dstream-window.png)

如该图所示，每当窗口*滑动*在源DSTREAM，落入窗口内的源RDDS被组合及操作以产生RDDS的窗DSTREAM。在这种特定情况下，该操作将应用于数据的最后3个时间单位，并以2个时间单位滑动。这表明任何窗口操作都需要指定两个参数。

- *窗口长度*-*窗口*的持续时间（图中3）。
- *滑动间隔*-进行窗口操作的间隔（图中为2）。

这两个参数必须是源DStream的批处理间隔的倍数（图中为1）。

让我们用一个例子来说明窗口操作。假设您想扩展 [前面的示例](http://spark.apache.org/docs/latest/streaming-programming-guide.html#a-quick-example)，方法是每10秒在数据的最后30秒生成一次字数统计。为此，我们必须在最后30秒的数据`reduceByKey`上对`pairs`DStream `(word, 1)`对应用该操作。这是通过操作完成的`reduceByKeyAndWindow`。

- [**Scala**](http://spark.apache.org/docs/latest/streaming-programming-guide.html#tab_scala_8)
- [**Java**](http://spark.apache.org/docs/latest/streaming-programming-guide.html#tab_java_8)
- [**Python**](http://spark.apache.org/docs/latest/streaming-programming-guide.html#tab_python_8)

```
// Reduce last 30 seconds of data, every 10 seconds
val windowedWordCounts = pairs.reduceByKeyAndWindow((a:Int,b:Int) => (a + b), Seconds(30), Seconds(10))
```

一些常见的窗口操作如下。所有这些操作都采用上述两个参数*-windowLength*和*slideInterval*。

| 转型                                                         | 含义                                                         |
| :----------------------------------------------------------- | :----------------------------------------------------------- |
| **窗口**（*windowLength*，*slideInterval*）                  | 返回基于源DStream的窗口批处理计算的新DStream。               |
| **countByWindow**（*windowLength*，*slideInterval*）         | 返回流中元素的滑动窗口计数。                                 |
| **reduceByWindow**（*func*，*windowLength*，*slideInterval*） | 返回一个新的单元素流，该流是通过使用*func*在滑动间隔内聚合流中的元素而创建的。该函数应该是关联的和可交换的，以便可以并行正确地计算它。 |
| **reduceByKeyAndWindow**（*func*，*windowLength*，*slideInterval*，[ *numTasks* ]） | 在（K，V）对的DStream上调用时，返回新的（K，V）对的DStream，其中使用给定的reduce函数*func* 在滑动窗口中的批处理上汇总每个键的值。**注意：**默认情况下，这使用Spark的默认并行任务数（本地模式为2，而在集群模式下，此数量由config属性确定`spark.default.parallelism`）进行分组。您可以传递一个可选 `numTasks`参数来设置不同数量的任务。 |
| **reduceByKeyAndWindow**（*func*，*invFunc*，*windowLength*， *slideInterval*，[ *numTasks* ]） | 上面一种更有效的版本，`reduceByKeyAndWindow()`其中，使用前一个窗口的减少值递增地计算每个窗口的减少值。这是通过减少进入滑动窗口的新数据，然后“反减少”离开窗口的旧数据来完成的。一个示例是在窗口滑动时“增加”和“减少”键的计数。但是，它仅适用于“可逆归约函数”，即具有相应“逆归约”函数（作为参数*invFunc*）的归约函数。像in中一样`reduceByKeyAndWindow`，reduce任务的数量可以通过可选参数配置。请注意，必须启用[检查点](http://spark.apache.org/docs/latest/streaming-programming-guide.html#checkpointing)才能使用此操作。 |
| **countByValueAndWindow**（*windowLength*， *slideInterval*，[ *numTasks* ]） | 在（K，V）对的DStream上调用时，返回新的（K，Long）对的DStream，其中每个键的值是其在滑动窗口内的频率。像in中一样 `reduceByKeyAndWindow`，reduce任务的数量可以通过可选参数配置。 |
|                                                              |                                                              |

#### 加盟运营

最后，值得一提的是，您可以轻松地在Spark Streaming中执行各种类型的联接。

##### 流流连接

流可以很容易地与其他流合并。

- [**Scala**](http://spark.apache.org/docs/latest/streaming-programming-guide.html#tab_scala_9)
- [**Java**](http://spark.apache.org/docs/latest/streaming-programming-guide.html#tab_java_9)
- [**Python**](http://spark.apache.org/docs/latest/streaming-programming-guide.html#tab_python_9)

```
val stream1: DStream[String, String] = ...
val stream2: DStream[String, String] = ...
val joinedStream = stream1.join(stream2)
```

在此，在每个批处理间隔中，由生成的RDD`stream1`将与生成的RDD合并`stream2`。你也可以做`leftOuterJoin`，`rightOuterJoin`，`fullOuterJoin`。此外，在流的窗口上进行联接通常非常有用。这也很容易。

- [**Scala**](http://spark.apache.org/docs/latest/streaming-programming-guide.html#tab_scala_10)
- [**Java**](http://spark.apache.org/docs/latest/streaming-programming-guide.html#tab_java_10)
- [**Python**](http://spark.apache.org/docs/latest/streaming-programming-guide.html#tab_python_10)

```
val windowedStream1 = stream1.window(Seconds(20))
val windowedStream2 = stream2.window(Minutes(1))
val joinedStream = windowedStream1.join(windowedStream2)
```

##### 流数据集联接

这已经在前面解释`DStream.transform`操作时显示过了。这是将窗口流与数据集结合在一起的另一个示例。

- [**Scala**](http://spark.apache.org/docs/latest/streaming-programming-guide.html#tab_scala_11)
- [**Java**](http://spark.apache.org/docs/latest/streaming-programming-guide.html#tab_java_11)
- [**Python**](http://spark.apache.org/docs/latest/streaming-programming-guide.html#tab_python_11)

```
val dataset: RDD[String, String] = ...
val windowedStream = stream.window(Seconds(20))...
val joinedStream = windowedStream.transform { rdd => rdd.join(dataset) }
```

实际上，您还可以动态更改要加入的数据集。`transform`每个批次间隔都会评估提供给该函数的功能，因此将使用当前的数据集`dataset`参考所指向。

API文档中提供了DStream转换的完整列表。有关Scala API，请参见[DStream](http://spark.apache.org/docs/latest/api/scala/org/apache/spark/streaming/dstream/DStream.html) 和[PairDStreamFunctions](http://spark.apache.org/docs/latest/api/scala/org/apache/spark/streaming/dstream/PairDStreamFunctions.html)。有关Java API，请参见[JavaDStream](http://spark.apache.org/docs/latest/api/java/index.html?org/apache/spark/streaming/api/java/JavaDStream.html) 和[JavaPairDStream](http://spark.apache.org/docs/latest/api/java/index.html?org/apache/spark/streaming/api/java/JavaPairDStream.html)。有关Python API，请参见[DStream](http://spark.apache.org/docs/latest/api/python/pyspark.streaming.html#pyspark.streaming.DStream)。

------

## DStreams上的输出操作

输出操作允许将DStream的数据推出到外部系统，例如数据库或文件系统。由于输出操作实际上允许外部系统使用转换后的数据，因此它们会触发所有DStream转换的实际执行（类似于RDD的操作）。当前，定义了以下输出操作：

| 输出操作                                  | 含义                                                         |
| :---------------------------------------- | :----------------------------------------------------------- |
| **列印**（）                              | 在运行流应用程序的驱动程序节点上，打印DStream中每批数据的前十个元素。这对于开发和调试很有用。 **Python API**在Python API中称为 **pprint（）**。 |
| **saveAsTextFiles**（*前缀*，[*后缀*]）   | 将此DStream的内容另存为文本文件。基于产生在每批间隔的文件名*的前缀*和*后缀*：*“前缀TIME_IN_MS [.suffix]”*。 |
| **saveAsObjectFiles**（*前缀*，[*后缀*]） | 将此DStream的内容保存为`SequenceFiles`序列化Java对象的内容。基于产生在每批间隔的文件名*的前缀*和 *后缀*：*“前缀TIME_IN_MS [.suffix]”*。 **Python API**这在Python API中不可用。 |
| **saveAsHadoopFiles**（*前缀*，[*后缀*]） | 将此DStream的内容另存为Hadoop文件。基于产生在每批间隔的文件名*的前缀*和*后缀*：*“前缀TIME_IN_MS [.suffix]”*。 **Python API**这在Python API中不可用。 |
| **foreachRDD**（*func*）                  | 最通用的输出运算符，将函数*func*应用于从流生成的每个RDD。此功能应将每个RDD中的数据推送到外部系统，例如将RDD保存到文件或通过网络将其写入数据库。请注意，函数*func*在运行流应用程序的驱动程序进程中执行，并且通常在其中具有RDD操作，这将强制计算流RDD。 |
|                                           |                                                              |

### 使用foreachRDD的设计模式

`dstream.foreachRDD`是一个强大的原语，可以将数据发送到外部系统。但是，重要的是要了解如何正确有效地使用此原语。应避免的一些常见错误如下。

通常，将数据写入外部系统需要创建一个连接对象（例如，到远程服务器的TCP连接），并使用该对象将数据发送到远程系统。为此，开发人员可能会无意间尝试在Spark驱动程序中创建连接对象，然后尝试在Spark辅助程序中使用该对象以将记录保存在RDD中。例如（在Scala中），

- [**Scala**](http://spark.apache.org/docs/latest/streaming-programming-guide.html#tab_scala_12)
- [**Java**](http://spark.apache.org/docs/latest/streaming-programming-guide.html#tab_java_12)
- [**Python**](http://spark.apache.org/docs/latest/streaming-programming-guide.html#tab_python_12)

```
dstream.foreachRDD { rdd =>
  val connection = createNewConnection()  // executed at the driver
  rdd.foreach { record =>
    connection.send(record) // executed at the worker
  }
}
```

这是不正确的，因为这要求将连接对象序列化并从驱动程序发送给工作程序。这样的连接对象很少能在机器之间转移。此错误可能表现为序列化错误（连接对象不可序列化），初始化错误（连接对象需要在工作程序中初始化）等。正确的解决方案是在工作程序中创建连接对象。

但是，这可能会导致另一个常见错误-为每个记录创建一个新的连接。例如，

- [**Scala**](http://spark.apache.org/docs/latest/streaming-programming-guide.html#tab_scala_13)
- [**Java**](http://spark.apache.org/docs/latest/streaming-programming-guide.html#tab_java_13)
- [**Python**](http://spark.apache.org/docs/latest/streaming-programming-guide.html#tab_python_13)

```
dstream.foreachRDD { rdd =>
  rdd.foreach { record =>
    val connection = createNewConnection()
    connection.send(record)
    connection.close()
  }
}
```

通常，创建连接对象会浪费时间和资源。因此，为每个记录创建和销毁连接对象会导致不必要的高开销，并且会大大降低系统的整体吞吐量。更好的解决方案是使用 `rdd.foreachPartition`-创建单个连接对象，并使用该连接在RDD分区中发送所有记录。

- [**Scala**](http://spark.apache.org/docs/latest/streaming-programming-guide.html#tab_scala_14)
- [**Java**](http://spark.apache.org/docs/latest/streaming-programming-guide.html#tab_java_14)
- [**Python**](http://spark.apache.org/docs/latest/streaming-programming-guide.html#tab_python_14)

```
dstream.foreachRDD { rdd =>
  rdd.foreachPartition { partitionOfRecords =>
    val connection = createNewConnection()
    partitionOfRecords.foreach(record => connection.send(record))
    connection.close()
  }
}
```

这将分摊许多记录上的连接创建开销。

最后，可以通过在多个RDD /批次之间重用连接对象来进一步优化。与将多个批次的RDD推送到外部系统时可以重用的连接对象相比，它可以维护一个静态的连接对象池，从而进一步减少了开销。

- [**Scala**](http://spark.apache.org/docs/latest/streaming-programming-guide.html#tab_scala_15)
- [**Java**](http://spark.apache.org/docs/latest/streaming-programming-guide.html#tab_java_15)
- [**Python**](http://spark.apache.org/docs/latest/streaming-programming-guide.html#tab_python_15)

```
dstream.foreachRDD { rdd =>
  rdd.foreachPartition { partitionOfRecords =>
    // ConnectionPool is a static, lazily initialized pool of connections
    val connection = ConnectionPool.getConnection()
    partitionOfRecords.foreach(record => connection.send(record))
    ConnectionPool.returnConnection(connection)  // return to the pool for future reuse
  }
}
```

请注意，应按需延迟创建池中的连接，如果一段时间不使用，则会超时。这样可以最有效地将数据发送到外部系统。

##### 其他要记住的要点：

- DStream由输出操作延迟执行，就像RDD由RDD操作延迟执行一样。具体来说，DStream输出操作内部的RDD动作会强制处理接收到的数据。因此，如果您的应用程序没有任何输出操作，或者`dstream.foreachRDD()`内部没有任何RDD操作，则不会执行任何操作。系统将仅接收数据并将其丢弃。
- 默认情况下，输出操作一次执行一次。它们按照在应用程序中定义的顺序执行。

------

## DataFrame和SQL操作

您可以轻松地对流数据使用[DataFrames和SQL](http://spark.apache.org/docs/latest/sql-programming-guide.html)操作。您必须使用StreamingContext使用的SparkContext创建一个SparkSession。此外，必须这样做，以便可以在驱动程序故障时重新启动它。这是通过创建SparkSession的延迟实例化单例实例来完成的。在下面的示例中显示。它修改了前面的[单词计数示例，](http://spark.apache.org/docs/latest/streaming-programming-guide.html#a-quick-example)以使用DataFrames和SQL生成单词计数。每个RDD都转换为一个DataFrame，注册为临时表，然后使用SQL查询。

- [**Scala**](http://spark.apache.org/docs/latest/streaming-programming-guide.html#tab_scala_16)
- [**Java**](http://spark.apache.org/docs/latest/streaming-programming-guide.html#tab_java_16)
- [**Python**](http://spark.apache.org/docs/latest/streaming-programming-guide.html#tab_python_16)

```
/** DataFrame operations inside your streaming program */

val words: DStream[String] = ...

words.foreachRDD { rdd =>

  // Get the singleton instance of SparkSession
  val spark = SparkSession.builder.config(rdd.sparkContext.getConf).getOrCreate()
  import spark.implicits._

  // Convert RDD[String] to DataFrame
  val wordsDataFrame = rdd.toDF("word")

  // Create a temporary view
  wordsDataFrame.createOrReplaceTempView("words")

  // Do word count on DataFrame using SQL and print it
  val wordCountsDataFrame = 
    spark.sql("select word, count(*) as total from words group by word")
  wordCountsDataFrame.show()
}
```

请参阅完整的[源代码](https://github.com/apache/spark/blob/v3.0.1/examples/src/main/scala/org/apache/spark/examples/streaming/SqlNetworkWordCount.scala)。

您还可以在来自不同线程的流数据上定义的表上运行SQL查询（即与正在运行的StreamingContext异步）。只需确保将StreamingContext设置为记住足够的流数据即可运行查询。否则，不知道任何异步SQL查询的StreamingContext将在查询完成之前删除旧的流数据。例如，如果您要查询最后一批，但是查询可能需要5分钟才能运行，然后调用`streamingContext.remember(Minutes(5))`（使用Scala或其他语言的等效语言）。

请参阅[DataFrames和SQL](http://spark.apache.org/docs/latest/sql-programming-guide.html)指南以了解有关DataFrames的更多信息。

------

## MLlib操作

您还可以轻松使用[MLlib](http://spark.apache.org/docs/latest/ml-guide.html)提供的机器学习算法。首先，有流机器学习算法（例如，[流线性回归](http://spark.apache.org/docs/latest/mllib-linear-methods.html#streaming-linear-regression)，[流KMeans](http://spark.apache.org/docs/latest/mllib-clustering.html#streaming-k-means)等），可以同时从流数据中学习并将模型应用于流数据。除此之外，对于更多种类的机器学习算法，您可以离线学习学习模型（即使用历史数据），然后在线将模型应用于流数据。有关更多详细信息，请参见[MLlib](http://spark.apache.org/docs/latest/ml-guide.html)指南。

------

## 缓存/持久化

与RDD相似，DStreams还允许开发人员将流的数据持久存储在内存中。也就是说，`persist()`在DStream上使用该方法将自动将该DStream的每个RDD持久存储在内存中。如果DStream中的数据将被多次计算（例如，对同一数据进行多次操作），这将很有用。对于和的基于窗口的操作`reduceByWindow`和 `reduceByKeyAndWindow`和的基于状态的操作`updateStateByKey`，这都是隐含的。因此，由基于窗口的操作生成的DStream会自动保存在内存中，而无需开发人员调用`persist()`。

对于通过网络接收数据的输入流（例如Kafka，套接字等），默认的持久性级别设置为将数据复制到两个节点以实现容错。

请注意，与RDD不同，DStream的默认持久性级别将数据序列化在内存中。[性能调整](http://spark.apache.org/docs/latest/streaming-programming-guide.html#memory-tuning)部分将对此进行进一步讨论。有关不同持久性级别的更多信息，请参见《[Spark编程指南》](http://spark.apache.org/docs/latest/rdd-programming-guide.html#rdd-persistence)。

------

## 检查点

流应用程序必须24/7全天候运行，因此必须对与应用程序逻辑无关的故障（例如系统故障，JVM崩溃等）具有弹性。为此，Spark Streaming需要将足够的信息*检查点*指向容错存储系统，以便可以从故障中恢复。检查点有两种类型的数据。

- 元数据检查点

  -将定义流计算的信息保存到HDFS等容错存储中。这用于从运行流应用程序的驱动程序的节点的故障中恢复（稍后详细讨论）。元数据包括：

  - *配置*-用于创建流应用程序的配置。
  - *DStream操作*-定义流应用程序的DStream操作集。
  - *不完整的批次*-作业排队但尚未完成的批次。

- *数据检查点*-将生成的RDD保存到可靠的存储中。在一些*有状态*转换中，这需要跨多个批次合并数据，这是必需的。在此类转换中，生成的RDD依赖于先前批次的RDD，这导致依赖项链的长度随时间不断增加。为了避免恢复时间的这种无限制的增加（与依存关系链成比例），有状态转换的中间RDD定期 *检查点*到可靠的存储（例如HDFS）以切断依存关系链。

总而言之，从驱动程序故障中恢复时，主要需要元数据检查点，而如果使用有状态转换，则即使是基本功能，也需要数据或RDD检查点。

#### 何时启用检查点

必须为具有以下任一要求的应用程序启用检查点：

- *有状态转换的用法*-如果在应用程序中使用`updateStateByKey`或`reduceByKeyAndWindow`（带有反函数），则必须提供检查点目录以允许定期进行RDD检查点。
- *从运行应用程序的驱动程序故障中恢复*-元数据检查点用于恢复进度信息。

注意，没有前述状态转换的简单流应用程序可以在不启用检查点的情况下运行。在这种情况下，从驱动程序故障中恢复也将是部分的（某些丢失但未处理的数据可能会丢失）。这通常是可以接受的，并且许多都以这种方式运行Spark Streaming应用程序。预计将来会改善对非Hadoop环境的支持。

#### 如何配置检查点

可以通过在容错，可靠的文件系统（例如，HDFS，S3等）中设置目录来启用检查点，将检查点信息保存到该目录中。这是通过使用完成的`streamingContext.checkpoint(checkpointDirectory)`。这将允许您使用上述有状态转换。此外，如果要使应用程序从驱动程序故障中恢复，则应重写流应用程序以具有以下行为。

- 程序首次启动时，它将创建一个新的StreamingContext，设置所有流，然后调用start（）。
- 失败后重新启动程序时，它将根据检查点目录中的检查点数据重新创建StreamingContext。

- [**Scala**](http://spark.apache.org/docs/latest/streaming-programming-guide.html#tab_scala_17)
- [**Java**](http://spark.apache.org/docs/latest/streaming-programming-guide.html#tab_java_17)
- [**Python**](http://spark.apache.org/docs/latest/streaming-programming-guide.html#tab_python_17)

使用可简化此行为`StreamingContext.getOrCreate`。如下使用。

```
// Function to create and setup a new StreamingContext
def functionToCreateContext(): StreamingContext = {
  val ssc = new StreamingContext(...)   // new context
  val lines = ssc.socketTextStream(...) // create DStreams
  ...
  ssc.checkpoint(checkpointDirectory)   // set checkpoint directory
  ssc
}

// Get StreamingContext from checkpoint data or create a new one
val context = StreamingContext.getOrCreate(checkpointDirectory, functionToCreateContext _)

// Do additional setup on context that needs to be done,
// irrespective of whether it is being started or restarted
context. ...

// Start the context
context.start()
context.awaitTermination()
```

如果`checkpointDirectory`存在，则将根据检查点数据重新创建上下文。如果该目录不存在（即首次运行），则将`functionToCreateContext`调用该函数以创建新上下文并设置DStreams。请参阅Scala示例 [RecoverableNetworkWordCount](https://github.com/apache/spark/tree/master/examples/src/main/scala/org/apache/spark/examples/streaming/RecoverableNetworkWordCount.scala)。本示例将网络数据的字数附加到文件中。

除了使用`getOrCreate`驱动程序外，还需要确保驱动程序进程在发生故障时自动重新启动。这只能通过用于运行应用程序的部署基础结构来完成。这将在“[部署”](http://spark.apache.org/docs/latest/streaming-programming-guide.html#deploying-applications)部分中进一步讨论 。

请注意，RDD的检查点会导致保存到可靠存储的成本。这可能会导致RDD获得检查点的那些批次的处理时间增加。因此，需要仔细设置检查点的间隔。在小批量（例如1秒）时，每批检查点可能会大大降低操作吞吐量。相反，检查点太少会导致沿袭和任务规模增加，这可能会产生不利影响。对于需要RDD检查点的有状态转换，默认间隔为批处理间隔的倍数，至少应为10秒。可以使用设置 `dstream.checkpoint(checkpointInterval)`。通常，DStream的5-10个滑动间隔的检查点间隔是一个很好的尝试设置。

------

## 累加器，广播变量和检查点

无法从Spark Streaming中的检查点恢复[累加器](http://spark.apache.org/docs/latest/rdd-programming-guide.html#accumulators)和[广播变量](http://spark.apache.org/docs/latest/rdd-programming-guide.html#broadcast-variables)。如果启用检查点并同时使用 [Accumulators](http://spark.apache.org/docs/latest/rdd-programming-guide.html#accumulators)或[Broadcast变量](http://spark.apache.org/docs/latest/rdd-programming-guide.html#broadcast-variables) ，则必须为[Accumulators](http://spark.apache.org/docs/latest/rdd-programming-guide.html#accumulators)和[Broadcast变量](http://spark.apache.org/docs/latest/rdd-programming-guide.html#broadcast-variables)创建延迟实例化的单例实例， 以便在驱动程序发生故障重新启动后可以重新实例化它们。在下面的示例中显示。

- [**Scala**](http://spark.apache.org/docs/latest/streaming-programming-guide.html#tab_scala_18)
- [**Java**](http://spark.apache.org/docs/latest/streaming-programming-guide.html#tab_java_18)
- [**Python**](http://spark.apache.org/docs/latest/streaming-programming-guide.html#tab_python_18)

```
object WordBlacklist {

  @volatile private var instance: Broadcast[Seq[String]] = null

  def getInstance(sc: SparkContext): Broadcast[Seq[String]] = {
    if (instance == null) {
      synchronized {
        if (instance == null) {
          val wordBlacklist = Seq("a", "b", "c")
          instance = sc.broadcast(wordBlacklist)
        }
      }
    }
    instance
  }
}

object DroppedWordsCounter {

  @volatile private var instance: LongAccumulator = null

  def getInstance(sc: SparkContext): LongAccumulator = {
    if (instance == null) {
      synchronized {
        if (instance == null) {
          instance = sc.longAccumulator("WordsInBlacklistCounter")
        }
      }
    }
    instance
  }
}

wordCounts.foreachRDD { (rdd: RDD[(String, Int)], time: Time) =>
  // Get or register the blacklist Broadcast
  val blacklist = WordBlacklist.getInstance(rdd.sparkContext)
  // Get or register the droppedWordsCounter Accumulator
  val droppedWordsCounter = DroppedWordsCounter.getInstance(rdd.sparkContext)
  // Use blacklist to drop words and use droppedWordsCounter to count them
  val counts = rdd.filter { case (word, count) =>
    if (blacklist.value.contains(word)) {
      droppedWordsCounter.add(count)
      false
    } else {
      true
    }
  }.collect().mkString("[", ", ", "]")
  val output = "Counts at time " + time + " " + counts
})
```

请参阅完整的[源代码](https://github.com/apache/spark/blob/v3.0.1/examples/src/main/scala/org/apache/spark/examples/streaming/RecoverableNetworkWordCount.scala)。

------

## 部署应用

本部分讨论了部署Spark Streaming应用程序的步骤。

### 要求

要运行Spark Streaming应用程序，您需要具备以下条件。

- *使用集群管理器进行集群*-这是任何Spark应用程序的一般要求，并且在[部署指南](http://spark.apache.org/docs/latest/cluster-overview.html)中进行了详细讨论。

- *将应用程序JAR打包*-您必须将流式应用程序编译为JAR。如果您[`spark-submit`](http://spark.apache.org/docs/latest/submitting-applications.html)用于启动应用程序，则无需在JAR中提供Spark和Spark Streaming。但是，如果您的应用程序使用[高级资源](http://spark.apache.org/docs/latest/streaming-programming-guide.html#advanced-sources)（例如Kafka），则必须将它们链接到的额外工件以及它们的依赖项打包在用于部署应用程序的JAR中。例如，使用的应用程序`KafkaUtils` 必须`spark-streaming-kafka-0-10_2.12`在应用程序JAR中包含及其所有传递依赖项。

- *为执行者配置足够的内存*-由于必须将接收到的数据存储在内存中，因此必须为执行者配置足够的内存来保存接收到的数据。请注意，如果您要执行10分钟的窗口操作，则系统必须在内存中至少保留最后10分钟的数据。因此，应用程序的内存要求取决于应用程序中使用的操作。

- *配置检查点*-如果流应用程序需要它，则必须将Hadoop API兼容的容错存储中的目录（例如，HDFS，S3等）配置为检查点目录，并且将流应用程序配置为可以通过检查点信息用于故障恢复。有关更多详细信息，请参见[检查点](http://spark.apache.org/docs/latest/streaming-programming-guide.html#checkpointing)部分。

- 配置应用程序驱动程序的自动重新启动

  -若要从驱动程序故障中自动恢复，用于运行流式应用程序的部署基础结构必须监视驱动程序进程，并在驱动程序失败时重新启动。不同的

  集群管理器

   具有不同的工具来实现这一目标。

  - *Spark Standalone-*可以提交Spark应用程序驱动程序以在Spark Standalone集群中运行（请参阅 [集群部署模式](http://spark.apache.org/docs/latest/spark-standalone.html#launching-spark-applications)），即，应用程序驱动程序本身在工作程序节点之一上运行。此外，可以指示独立群集管理器*监督*驱动程序，并在驱动程序由于非零退出代码或由于运行该驱动程序的节点故障而失败时重新启动它。有关更多详细信息，请参见[Spark Standalone指南](http://spark.apache.org/docs/latest/spark-standalone.html)中的 *集群模式*和*监督*。
  - *YARN* -Yarn支持自动重启应用程序的类似机制。请参阅YARN文档以获取更多详细信息。
  - *Mesos* -[马拉松](https://github.com/mesosphere/marathon)已经使用Mesos来实现这一目标。

- *配置预写日志*-自Spark 1.2起，我们引入了*预写日志*以实现强大的容错保证。如果启用，则将从接收器接收的所有数据写入配置检查点目录中的预写日志。这样可以防止驱动程序恢复时丢失数据，从而确保零数据丢失（在“[容错语义”](http://spark.apache.org/docs/latest/streaming-programming-guide.html#fault-tolerance-semantics)部分中进行了详细讨论 ）。这可以通过设置来启用[配置参数](http://spark.apache.org/docs/latest/configuration.html#spark-streaming) `spark.streaming.receiver.writeAheadLog.enable`来`true`。但是，这些更强的语义可能以单个接收器的接收吞吐量为代价。可以通过运行来更正[并行更多接收器](http://spark.apache.org/docs/latest/streaming-programming-guide.html#level-of-parallelism-in-data-receiving) 增加总吞吐量。另外，由于启用了预写日志，因此建议禁用Spark中接收数据的复制，因为该日志已经存储在复制的存储系统中。可以通过将输入流的存储级别设置为来完成此操作`StorageLevel.MEMORY_AND_DISK_SER`。在将S3（或任何不支持刷新的文件系统）用于*预写日志时*，请记住启用 `spark.streaming.driver.writeAheadLog.closeFileAfterWrite`和 `spark.streaming.receiver.writeAheadLog.closeFileAfterWrite`。有关更多详细信息，请参见 [Spark Streaming配置](http://spark.apache.org/docs/latest/configuration.html#spark-streaming)。请注意，启用I / O加密后，Spark不会加密写入预写日志的数据。如果需要对预写日志数据进行加密，则应将其存储在本身支持加密的文件系统中。

- *设置最大接收速率*-如果群集资源不足以使流应用程序能够以最快的速度处理数据，则可以通过设置记录/秒的最大速率限制来限制接收器的速率。请参阅接收器和 Direct Kafka方法的[配置参数](http://spark.apache.org/docs/latest/configuration.html#spark-streaming) 。在Spark 1.5中，我们引入了一个称为*背压*的功能，该功能消除了设置此速率限制的需要，因为Spark Streaming会自动计算出速率限制，并在处理条件发生变化时动态调整它们。这个背压可以通过设置来启用[配置参数](http://spark.apache.org/docs/latest/configuration.html#spark-streaming)来。`spark.streaming.receiver.maxRate``spark.streaming.kafka.maxRatePerPartition` `spark.streaming.backpressure.enabled``true`

### 升级应用程序代码

如果需要使用新的应用程序代码升级正在运行的Spark Streaming应用程序，则有两种可能的机制。

- 升级后的Spark Streaming应用程序将启动并与现有应用程序并行运行。一旦新的（接收与旧的数据相同）的数据被预热并准备好进行黄金时段，就可以关闭旧的数据。请注意，对于支持将数据发送到两个目标的数据源（即，较早和升级的应用程序），可以这样做。
- 现有应用程序正常关闭（请参阅 [`StreamingContext.stop(...)`](http://spark.apache.org/docs/latest/api/scala/org/apache/spark/streaming/StreamingContext.html) 或[`JavaStreamingContext.stop(...)`](http://spark.apache.org/docs/latest/api/java/index.html?org/apache/spark/streaming/api/java/JavaStreamingContext.html) （用于正常关闭选项），以确保在关闭之前已完全处理已接收的数据。然后可以启动升级的应用程序，它将从较早的应用程序停止的同一点开始进行处理。请注意，只能使用支持源端缓冲的输入源（例如Kafka）来完成此操作，因为在上一个应用程序关闭且升级的应用程序尚未启动时，需要缓冲数据。并且无法完成从升级前代码的较早检查点信息重新启动的操作。检查点信息本质上包含序列化的Scala / Java / Python对象，尝试使用经过修改的新类反序列化对象可能会导致错误。在这种情况下，请使用其他检查点目录启动升级的应用程序，或者删除先前的检查点目录。

------

## 监控应用

除了Spark的[监视功能](http://spark.apache.org/docs/latest/monitoring.html)外，Spark Streaming还具有其他特定功能。当使用StreamingContext时， [Spark Web UI会](http://spark.apache.org/docs/latest/monitoring.html#web-interfaces)显示一个附加`Streaming`选项卡，其中显示有关正在运行的接收器（接收器是否处于活动状态，接收到的记录数，接收器错误等）和已完成的批处理（批处理时间，排队延迟等）的统计信息。 ）。这可用于监视流应用程序的进度。

Web UI中的以下两个指标特别重要：

- *处理时间*-处理每批数据的时间。
- *调度延迟*-批生产在队列中等待前几批处理完成的时间。

如果批处理时间始终大于批处理时间间隔和/或排队延迟持续增加，则表明系统无法像生成批处理一样快处理批处理，并且落后于此。在这种情况下，请考虑 [减少](http://spark.apache.org/docs/latest/streaming-programming-guide.html#reducing-the-batch-processing-times)批处理时间。

还可以使用[StreamingListener](http://spark.apache.org/docs/latest/api/scala/org/apache/spark/streaming/scheduler/StreamingListener.html)界面监视Spark Streaming程序的进度，该 界面允许您获取接收器状态和处理时间。请注意，这是一个开发人员API，将来可能会得到改进（即，报告了更多信息）。

------

------

# 性能调优

要在集群上的Spark Streaming应用程序中获得最佳性能，需要进行一些调整。本节说明了可以调整以提高应用程序性能的许多参数和配置。从高层次上讲，您需要考虑两件事：

1. 通过有效使用群集资源减少每批数据的处理时间。
2. 设置正确的批处理大小，以便可以在接收到批处理数据后尽快对其进行处理（也就是说，数据处理与数据摄取保持同步）。

## 减少批处理时间

在Spark中可以进行许多优化，以最大程度地减少每批的处理时间。这些已在“[调优指南”](http://spark.apache.org/docs/latest/tuning.html)中详细讨论。本节重点介绍一些最重要的内容。

### 数据接收中的并行度

通过网络（例如Kafka，套接字等）接收数据需要将数据反序列化并存储在Spark中。如果数据接收成为系统的瓶颈，请考虑并行化数据接收。请注意，每个输入DStream都会创建一个接收器（在工作计算机上运行），该接收器接收单个数据流。因此，可以通过创建多个输入DStream并将其配置为从源接收数据流的不同分区来实现接收多个数据流。例如，可以将接收两个主题数据的单个Kafka输入DStream拆分为两个Kafka输入流，每个输入流仅接收一个主题。这将运行两个接收器，从而允许并行接收数据，从而提高了总体吞吐量。这些多个DStream可以结合在一起以创建单个DStream。然后，可以将应用于单个输入DStream的转换应用于统一流。这样做如下。

- [**Scala**](http://spark.apache.org/docs/latest/streaming-programming-guide.html#tab_scala_19)
- [**Java**](http://spark.apache.org/docs/latest/streaming-programming-guide.html#tab_java_19)
- [**Python**](http://spark.apache.org/docs/latest/streaming-programming-guide.html#tab_python_19)

```
val numStreams = 5
val kafkaStreams = (1 to numStreams).map { i => KafkaUtils.createStream(...) }
val unifiedStream = streamingContext.union(kafkaStreams)
unifiedStream.print()
```

应考虑的另一个参数是接收机的块间隔，该间隔由[配置参数](http://spark.apache.org/docs/latest/configuration.html#spark-streaming)确定 `spark.streaming.blockInterval`。对于大多数接收器，接收到的数据在存储在Spark内存中之前会合并为数据块。每批中的块数确定了将在类似地图的转换中用于处理接收到的数据的任务数。每批接收器中每个接收器的任务数大约为（批处理间隔/块间隔）。例如，200 ms的块间隔将每2秒批处理创建10个任务。如果任务数太少（即少于每台计算机的核心数），那么它将效率低下，因为将不会使用所有可用的核心来处理数据。要增加给定批处理间隔的任务数，请减小阻止间隔。但是，建议的块间隔最小值约为50毫秒，在此之下，任务启动开销可能是个问题。

使用多个输入流/接收器接收数据的另一种方法是显式地对输入数据流进行分区（使用`inputStream.repartition(<number of partitions>)`）。在进一步处理之前，这会将接收到的数据批分布在群集中指定数量的计算机上。

对于直接流，请参阅[Spark Streaming + Kafka集成指南](http://spark.apache.org/docs/latest/streaming-kafka-0-10-integration.html)

### 数据处理中的并行度

如果在计算的任何阶段使用的并行任务数量不够高，则群集资源可能无法得到充分利用。例如，对于像`reduceByKey` 和这样的分布式归约操作`reduceByKeyAndWindow`，并行任务的默认数量由`spark.default.parallelism` [configuration属性](http://spark.apache.org/docs/latest/configuration.html#spark-properties)控制。您可以将并行性级别作为参数传递（请参阅 [`PairDStreamFunctions`](http://spark.apache.org/docs/latest/api/scala/org/apache/spark/streaming/dstream/PairDStreamFunctions.html) 文档），或将`spark.default.parallelism` [配置属性](http://spark.apache.org/docs/latest/configuration.html#spark-properties)设置为更改默认值。

### 数据序列化

可以通过调整序列化格式来减少数据序列化的开销。在流传输的情况下，有两种类型的数据正在序列化。

- **输入数据**：默认情况下，通过Receivers接收的输入数据通过[StorageLevel.MEMORY_AND_DISK_SER_2](http://spark.apache.org/docs/latest/api/scala/org/apache/spark/storage/StorageLevel$.html)存储在执行程序的内存中。也就是说，数据被序列化为字节以减少GC开销，并被复制以容忍执行器故障。同样，数据首先保存在内存中，并且仅在内存不足以容纳流计算所需的所有输入数据时才溢出到磁盘。显然，这种序列化会产生开销–接收器必须对接收到的数据进行反序列化，然后使用Spark的序列化格式对其进行重新序列化。
- **流操作生成的持久RDD**：流计算生成的RDD可以保留在内存中。例如，窗口操作会将数据保留在内存中，因为它们将被多次处理。但是，与Spark Core默认的[StorageLevel.MEMORY_ONLY](http://spark.apache.org/docs/latest/api/scala/org/apache/spark/storage/StorageLevel$.html)不同，默认情况下，由流计算生成的持久性RDD与[StorageLevel.MEMORY_ONLY_SER](http://spark.apache.org/docs/latest/api/scala/org/apache/spark/storage/StorageLevel.html$)（即序列化）保持一致，以最大程度地减少GC开销。

在这两种情况下，使用Kryo序列化都可以减少CPU和内存的开销。有关更多详细信息，请参见《[Spark Tuning Guide》](http://spark.apache.org/docs/latest/tuning.html#data-serialization)。对于Kryo，请考虑注册自定义类，并禁用对象引用跟踪（请参阅《[配置指南》中](http://spark.apache.org/docs/latest/configuration.html#compression-and-serialization)与Kryo相关的[配置](http://spark.apache.org/docs/latest/configuration.html#compression-and-serialization)）。

在流应用程序需要保留的数据量不大的特定情况下，将数据（两种类型）保留为反序列化对象是可行的，而不会产生过多的GC开销。例如，如果您使用的是几秒钟的批处理间隔并且没有窗口操作，那么您可以尝试通过显式设置存储级别来禁用持久化数据中的序列化。这将减少由于序列化导致的CPU开销，从而可能在没有太多GC开销的情况下提高性能。

### 任务启动开销

如果每秒启动的任务数量很高（例如，每秒50个或更多），那么向从服务器发送任务的开销可能会很大，并且将难以实现亚秒级的延迟。可以通过以下更改来减少开销：

- **执行模式**：在独立模式或粗粒度Mesos模式**下**运行Spark可以比细粒度Mesos模式缩短任务启动时间。有关更多详细信息，请参阅[“在Mesos](http://spark.apache.org/docs/latest/running-on-mesos.html)上 [运行”指南](http://spark.apache.org/docs/latest/running-on-mesos.html)。

这些更改可以将批处理时间减少100毫秒，从而使亚秒级的批处理大小可行。

------

## 设置正确的批次间隔

为了使在群集上运行的Spark Streaming应用程序稳定，系统应该能够处理接收到的数据。换句话说，应尽快处理一批数据。可以通过[监视](http://spark.apache.org/docs/latest/streaming-programming-guide.html#monitoring-applications)流式Web UI中的处理时间来发现这是否适用于应用程序 ，其中批处理时间应小于批处理间隔。

根据流计算的性质，所使用的批处理间隔可能会对数据速率产生重大影响，而数据速率可以由应用程序在固定的一组群集资源上维持。例如，让我们考虑前面的WordCountNetwork示例。对于特定的数据速率，系统可能能够跟上每2秒（即2秒的批处理间隔）但不是每500毫秒报告一次字数的情况。因此，需要设置批次间隔，以便可以维持生产中的预期数据速率。

找出适合您的应用程序的正确批处理大小的一种好方法是使用保守的批处理间隔（例如5-10秒）和低数据速率进行测试。要验证系统是否能够跟上数据速率，您可以检查每个已处理批处理经历的端到端延迟的值（在Spark驱动程序log4j日志中查找“ Total delay”（总延迟），或使用 [流侦听器](http://spark.apache.org/docs/latest/api/scala/org/apache/spark/streaming/scheduler/StreamingListener.html) 接口）。如果延迟保持与批次大小相当，则系统是稳定的。否则，如果延迟持续增加，则意味着系统无法跟上，因此不稳定。一旦有了稳定配置的想法，就可以尝试提高数据速率和/或减小批处理大小。注意，由于暂时的数据速率增加而引起的延迟的瞬时增加可能是好的，只要延迟减小回到较低的值（即，小于批大小）即可。

------

## 内存调优

在“[调优指南”中](http://spark.apache.org/docs/latest/tuning.html#memory-tuning)详细讨论了如何[调优](http://spark.apache.org/docs/latest/tuning.html#memory-tuning)Spark应用程序的内存使用情况和GC行为。强烈建议您阅读。在本节中，我们将专门在Spark Streaming应用程序的上下文中讨论一些调整参数。

Spark Streaming应用程序所需的群集内存量在很大程度上取决于所使用的转换类型。例如，如果要对最后10分钟的数据使用窗口操作，则群集应具有足够的内存以将10分钟的数据保留在内存中。或者，如果您想使用`updateStateByKey`大量的按键，则所需的存储空间会很大。相反，如果您想执行一个简单的map-filter-store操作，则所需的内存将很少。

通常，由于通过接收器接收的数据存储在StorageLevel.MEMORY_AND_DISK_SER_2中，因此无法容纳在内存中的数据将溢出到磁盘上。这可能会降低流应用程序的性能，因此建议您提供流应用程序所需的足够内存。最好尝试以小规模查看内存使用情况并据此进行估计。

内存调整的另一个方面是垃圾回收。对于需要低延迟的流应用程序，不希望由于JVM垃圾收集而导致较大的停顿。

有一些参数可以帮助您调整内存使用和GC开销：

- **DStream的持久性级别**：如前面的“[数据序列化”](http://spark.apache.org/docs/latest/streaming-programming-guide.html#data-serialization)部分所述，默认情况下，输入数据和RDD被持久化为序列化字节。与反序列化的持久性相比，这减少了内存使用和GC开销。启用Kryo序列化可进一步减少序列化的大小和内存使用量。通过压缩（请参见Spark配置`spark.rdd.compress`）可以进一步减少内存使用，但会占用CPU时间。
- **清除旧数据**：默认情况下，将自动清除DStream转换生成的所有输入数据和持久性RDD。Spark Streaming根据使用的转换来决定何时清除数据。例如，如果您使用10分钟的窗口操作，那么Spark Streaming将保留最后10分钟的数据，并主动丢弃较旧的数据。通过设置可以将数据保留更长的时间（例如，以交互方式查询较旧的数据）`streamingContext.remember`。
- **CMS垃圾收集器**：强烈建议使用并发标记清除GC，以使与GC相关的暂停时间始终保持较低。尽管已知并发GC会降低系统的整体处理吞吐量，但仍建议使用它来实现更一致的批处理时间。确保在驱动程序（使用`--driver-java-options`中`spark-submit`）和执行程序（使用[Spark配置](http://spark.apache.org/docs/latest/configuration.html#runtime-environment) `spark.executor.extraJavaOptions`）上都设置了CMS GC 。
- **其他提示**：为了进一步减少GC开销，请尝试以下更多提示。
  - 使用`OFF_HEAP`存储级别持久化RDD 。请参阅《[Spark编程指南》](http://spark.apache.org/docs/latest/rdd-programming-guide.html#rdd-persistence)中的更多详细信息。
  - 使用更多具有较小堆大小的执行程序。这将减少每个JVM堆中的GC压力。

------

##### 要记住的要点：

- DStream与单个接收器关联。为了获得读取并行性，需要创建多个接收器，即多个DStream。接收器在执行器中运行。它占据了一个核心。预订接收器插槽后，请确保有足够的内核可用于处理，即`spark.cores.max`应考虑接收器插槽。接收者以循环方式分配给执行者。
- 当从流源接收数据时，接收器会创建数据块。每blockInterval毫秒生成一个新的数据块。在batchInterval期间创建了N个数据块，其中N = batchInterval / blockInterval。这些块由当前执行器的BlockManager分发给其他执行器的块管理器。之后，驱动程序上运行的网络输入跟踪器将被告知有关块的位置，以进行进一步处理。
- 在驱动程序上为在batchInterval期间创建的块创建了RDD。在batchInterval期间生成的块是RDD的分区。每个分区都是一个任务。blockInterval == batchinterval意味着将创建一个分区，并且可能在本地对其进行处理。
- 块上的映射任务在执行器中进行处理（一个执行器接收该块，另一个执行器复制该块），该执行器具有与块间隔无关的块，除非执行非本地调度。除非有更大的块间隔，这意味着更大的块。较高的值会`spark.locality.wait`增加在本地节点上处理块的机会。需要在这两个参数之间找到平衡，以确保较大的块在本地处理。
- 您可以通过调用来定义分区数，而不是依赖于batchInterval和blockInterval `inputDstream.repartition(n)`。这会随机重新随机排列RDD中的数据以创建n个分区。是的，以获得更大的并行度。尽管以洗牌为代价。RDD的处理由驾驶员的Jobscheduler安排为作业。在给定的时间点，只有一项作业处于活动状态。因此，如果一个作业正在执行，则其他作业将排队。
- 如果您有两个dstream，将形成两个RDD，并且将创建两个作业，这些作业将一个接一个地调度。为避免这种情况，可以合并两个dstream。这将确保为dstream的两个RDD形成单个unionRDD。然后将此unionRDD视为一项工作。但是，RDD的分区不受影响。
- 如果批处理时间超过batchinterval，那么显然接收方的内存将开始填满，并最终引发异常（最有可能是BlockNotFoundException）。当前，无法暂停接收器。使用SparkConf配置`spark.streaming.receiver.maxRate`，可以限制接收器的速率。

------

------

# 容错语义

在本节中，我们将讨论发生故障时Spark Streaming应用程序的行为。

## 背景

为了理解Spark Streaming提供的语义，让我们记住Spark的RDD的基本容错语义。

1. RDD是一个不变的，确定性可重新计算的分布式数据集。每个RDD都会记住在容错输入数据集上用于创建它的确定性操作的沿袭。
2. 如果由于工作节点故障而导致RDD的任何分区丢失，则可以使用操作沿袭从原始容错数据集中重新计算该分区。
3. 假设所有RDD转换都是确定性的，则最终转换后的RDD中的数据将始终相同，而不管Spark集群中的故障如何。

Spark在容错文件系统（例如HDFS或S3）中的数据上运行。因此，从容错数据生成的所有RDD也是容错的。但是，Spark Streaming并非如此，因为大多数情况下是通过网络接收数据的（使用时除外 `fileStream`）。为了对所有生成的RDD实现相同的容错属性，将接收到的数据复制到集群中工作节点中的多个Spark执行程序中（默认复制因子为2）。这导致系统中发生故障时需要恢复的两种数据：

1. *接收和复制的*数据-由于该数据的副本存在于其他节点之一上，因此该数据在单个工作节点发生故障时仍可幸免。
2. *已接收但已缓冲数据以进行复制*-由于不进行复制，因此恢复此数据的唯一方法是从源重新获取数据。

此外，我们应该关注两种故障：

1. *工作节点发生故障*-运行执行程序的任何工作节点都可能发生故障，并且这些节点上的所有内存数据都将丢失。如果有任何接收器在发生故障的节点上运行，那么它们的缓冲数据将丢失。
2. *驱动程序节点发生故障*-如果运行Spark Streaming应用程序的驱动程序节点发生故障，则显然SparkContext会丢失，并且所有执行程序及其内存中的数据也会丢失。

有了这些基本知识，让我们了解Spark Streaming的容错语义。

## 定义

流式系统的语义通常是根据系统可以处理每个记录多少次来捕获的。系统在所有可能的操作条件下（尽管有故障等）可以提供三种保证。

1. *最多一次*：每个记录将被处理一次或根本不处理。
2. *至少一次*：每个记录将被处理一次或多次。它比*最多一次*强*，*因为它确保不会丢失任何数据。但是可能会有重复。
3. *恰好一次*：每个记录将被恰好处理一次-不会丢失任何数据，也不会多次处理任何数据。这显然是三者中最强有力的保证。

## 基本语义

概括地说，在任何流处理系统中，处理数据都需要三个步骤。

1. *接收数据*：使用接收器或其他方式从源接收数据。
2. *转换数据*：使用DStream和RDD转换对接收到的数据进行转换。
3. *推送数据*：将最终转换后的数据推送到外部系统，例如文件系统，数据库，仪表板等。

如果流应用程序必须获得端到端的精确一次保证，那么每个步骤都必须提供精确一次保证。也就是说，每条记录必须被接收一次，被转换一次，并被推送到下游系统一次。让我们在Spark Streaming的上下文中了解这些步骤的语义。

1. *接收数据*：不同的输入源提供不同的保证。下一部分将对此进行详细讨论。
2. *转换数据*：由于RDD提供的保证，所有接收到的数据将只处理*一次*。即使出现故障，只要可以访问接收到的输入数据，最终转换后的RDD将始终具有相同的内容。
3. *推出数据*：默认情况下，输出操作确保*至少一次*语义，因为它取决于输出操作的类型（是否为幂等）和下游系统的语义（是否支持事务）。但是用户可以实现自己的事务处理机制来实现*一次*语义。本节稍后将对此进行详细讨论。

## 接收数据的语义

不同的输入源提供不同的保证，范围从*至少一次*到*恰好一次*。阅读更多详细信息。

### 带文件

如果所有输入数据已经存在于诸如HDFS之类的容错文件系统中，则Spark Streaming始终可以从任何故障中恢复并处理所有数据。这提供 *了一次精确的*语义，这意味着无论发生什么故障，所有数据都会被精确处理一次。

### 使用基于接收器的源

对于基于接收方的输入源，容错语义取决于故障情况和接收方的类型。正如我们所讨论的[前面](http://spark.apache.org/docs/latest/streaming-programming-guide.html#receiver-reliability)，有两种类型的接收器：

1. *可靠的接收器*-这些接收器仅在确保已复制接收到的数据后才确认可靠的来源。如果这样的接收器发生故障，则源将不会收到对缓冲的（未复制的）数据的确认。因此，如果重新启动接收器，则源将重新发送数据，并且不会由于失败而丢失任何数据。
2. *不可靠的接收器*-此类接收器*不*发送确认，因此当由于工作程序或驱动程序故障而失败时，*可能*会丢失数据。

根据所使用的接收器类型，我们可以实现以下语义。如果工作节点发生故障，那么可靠的接收器不会造成数据丢失。如果接收器不可靠，则接收到但未复制的数据可能会丢失。如果驱动程序节点发生故障，则除了这些丢失之外，所有已接收并复制到内存中的过去数据都将丢失。这将影响有状态转换的结果。

为了避免丢失以前收到的数据，Spark 1.2引入了预*写日志*，该*日志*将收到的数据保存到容错存储中。使用[启用预写日志](http://spark.apache.org/docs/latest/streaming-programming-guide.html#deploying-applications)和可靠的接收器，数据丢失为零。就语义而言，它至少提供了一次保证。

下表总结了失败时的语义：

| 部署方案                                                     | 工人失败                                                     | 驱动故障                                                     |
| :----------------------------------------------------------- | :----------------------------------------------------------- | :----------------------------------------------------------- |
| *Spark 1.1或更早版本，*或者 *Spark 1.2或更高版本，没有预写日志* | 接收器不可靠 导致缓冲数据丢失接收器不可靠导致缓冲数据丢失 至少一次语义 | 不可靠的接收器缓冲的数据丢失 过去的所有接收器丢失的数据 未定义语义 |
| *Spark 1.2或更高版本具有预写日志*                            | 可靠的接收器实现零数据丢失 至少一次语义                      | 可靠的接收器和文件实现零数据丢失 至少一次语义                |
|                                                              |                                                              |                                                              |

### 使用Kafka Direct API

在Spark 1.3中，我们引入了新的Kafka Direct API，它可以确保Spark Streaming一次接收所有Kafka数据。同时，如果您执行一次精确的输出操作，则可以实现端到端的一次精确保证。《[Kafka集成指南》中](http://spark.apache.org/docs/latest/streaming-kafka-0-10-integration.html)进一步讨论了这种方法。

## 输出操作的语义

输出操作（如`foreachRDD`）*至少具有一次*语义，也就是说，在工作程序失败的情况下，转换后的数据可能多次写入外部实体。尽管使用`saveAs***Files`操作将其保存到文件系统是可以接受的 （因为文件将被相同的数据简单地覆盖），但可能需要付出额外的努力才能实现一次精确的语义。有两种方法。

- *幂等更新*：多次尝试总是写入相同的数据。例如，`saveAs***Files`始终将相同的数据写入生成的文件。

- *事务性更新*：所有更新都是以事务方式进行的，因此原子更新仅进行一次。一种做到这一点的方法如下。

  - 使用批处理时间（可在中找到`foreachRDD`）和RDD的分区索引来创建标识符。该标识符唯一地标识流应用程序中的Blob数据。

  - 使用标识符通过事务（即，原子地一次）更新此Blob的外部系统。也就是说，如果尚未提交标识符，则自动提交分区数据和标识符。否则，如果已经提交，则跳过更新。

    ```
    dstream.foreachRDD { (rdd, time) =>
      rdd.foreachPartition { partitionIterator =>
        val partitionId = TaskContext.get.partitionId()
        val uniqueId = generateUniqueId(time.milliseconds, partitionId)
        // use this uniqueId to transactionally commit the data in partitionIterator
      }
    }
    ```

------

------

# 从这往哪儿走

- 其他指南
  - [Kafka集成指南](http://spark.apache.org/docs/latest/streaming-kafka-0-10-integration.html)
  - [Kinesis集成指南](http://spark.apache.org/docs/latest/streaming-kinesis-integration.html)
  - [自定义接收器指南](http://spark.apache.org/docs/latest/streaming-custom-receivers.html)
- 第三方DStream数据源可以在[第三方项目中](https://spark.apache.org/third-party-projects.html)找到
- API文档
  - Scala文档
    - [StreamingContext](http://spark.apache.org/docs/latest/api/scala/org/apache/spark/streaming/StreamingContext.html)和 [DStream](http://spark.apache.org/docs/latest/api/scala/org/apache/spark/streaming/dstream/DStream.html)
    - [KafkaUtils](http://spark.apache.org/docs/latest/api/scala/org/apache/spark/streaming/kafka/KafkaUtils$.html)， [KinesisUtils](http://spark.apache.org/docs/latest/api/scala/org/apache/spark/streaming/kinesis/KinesisInputDStream.html)，
  - Java文档
    - [JavaStreamingContext](http://spark.apache.org/docs/latest/api/java/index.html?org/apache/spark/streaming/api/java/JavaStreamingContext.html)， [JavaDStream](http://spark.apache.org/docs/latest/api/java/index.html?org/apache/spark/streaming/api/java/JavaDStream.html)和 [JavaPairDStream](http://spark.apache.org/docs/latest/api/java/index.html?org/apache/spark/streaming/api/java/JavaPairDStream.html)
    - [KafkaUtils](http://spark.apache.org/docs/latest/api/java/index.html?org/apache/spark/streaming/kafka/KafkaUtils.html)， [KinesisUtils](http://spark.apache.org/docs/latest/api/java/index.html?org/apache/spark/streaming/kinesis/KinesisInputDStream.html)
  - Python文档
    - [StreamingContext](http://spark.apache.org/docs/latest/api/python/pyspark.streaming.html#pyspark.streaming.StreamingContext)和[DStream](http://spark.apache.org/docs/latest/api/python/pyspark.streaming.html#pyspark.streaming.DStream)
    - [KafkaUtils](http://spark.apache.org/docs/latest/api/python/pyspark.streaming.html#pyspark.streaming.kafka.KafkaUtils)
- [Scala](https://github.com/apache/spark/tree/master/examples/src/main/scala/org/apache/spark/examples/streaming) 和[Java](https://github.com/apache/spark/tree/master/examples/src/main/java/org/apache/spark/examples/streaming) 和[Python中的](https://github.com/apache/spark/tree/master/examples/src/main/python/streaming)更多示例
- 描述Spark Streaming的[论文](http://www.eecs.berkeley.edu/Pubs/TechRpts/2012/EECS-2012-259.pdf)和[视频](http://youtu.be/g171ndOHgJ0)。