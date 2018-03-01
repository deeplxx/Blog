---
title: Spark基础
categories: 大数据
tag:
- Spark
- 大数据
mathjax: true
---

## 几个重要的概念：
* RDD：是弹性分布式数据集（Resilient Distributed Dataset）的简称，是分布式内存的一个抽象概念，提供了一种高度受限的共享内存模型；
* DAG：是Directed Acyclic Graph（有向无环图）的简称，反映RDD之间的依赖关系；
* Executor：是运行在工作节点（Worker Node）上的一个进程，负责运行任务，并为应用程序存储数据；
* App：用户编写的Spark应用程序；
* task：运行在Executor上的工作单元；
* job：一个作业包含多个RDD及作用于相应RDD上的各种操作；
* stage：是作业的基本调度单位，一个作业会分为多组任务，每组任务被称为“阶段”，或者也被称为“任务集”
* App:
    * Driver
    * Job -> stage -> task
![spark运行流程图](http://dblab.xmu.edu.cn/blog/wp-content/uploads/2016/11/%E5%9B%BE9-7-Spark%E8%BF%90%E8%A1%8C%E5%9F%BA%E6%9C%AC%E6%B5%81%E7%A8%8B%E5%9B%BE.jpg)

## RDD

> RDD提供了一个抽象的数据架构，我们不必担心底层数据的分布式特性，只需将具体的应用逻辑表达为一系列转换处理，不同RDD之间的转换操作形成依赖关系，可以实现管道化，从而避免了中间结果的存储，大大降低了数据复制、磁盘IO和序列化开销。一个RDD就是一个分布式对象集合，本质上是一个只读的分区记录集合，每个RDD可以分成多个分区，每个分区就是一个数据集片段，并且一个RDD的不同分区可以被保存到集群中不同的节点上，从而可以在集群中的不同节点上进行并行计算。RDD提供了一种高度受限的共享内存模型，即RDD是只读的记录分区的集合，不能直接修改，只能基于稳定的物理存储中的数据集来创建RDD，或者通过在其他RDD上执行确定的转换操作（如map、join和groupBy）而创建得到新的RDD

1. RDD的两种创建方式(均通过SparkContext创建)：
    1. 读取文件中的数据集：`sc.textFile(filePath)`
    2. 在Driver中已经存在的一个集合上创建：`sc.parallelize(array)`
2. RDD创建好后，会进行两种操作：
    1. 转换：对现有数据集进行处理，结果还是数据集。常见转换操作有
        - `filter(func)` :筛选，会返回一个新的RDD，不会改变原来的RDD
        - `map(func)` :对每个元素进行操作
        - `distinct()` :去重
        - `groupByKey()` :专用于键值对。按key进行分组
        - `reduceByKey(func)` :专用于键值对。对相同key的value进行聚合操作
    2. 行动：在数据集上进行运算，产生结果。常见行动操作有
        - `count()` :返回集合中的元素个数
        - `collect()` :以数组形式返回集合中的所有元素(会把所有worker结点上的RDD都抓取过来，有内存溢出的风险，可以先take一部分)
        - `first()` :返回数据集中的第一个元素
        - `take(n)` :以数组形式返回数据集中的前n个元素
        - `reduce(func)` :对数据集进行聚合操作
        - `foreach(func)` :对数据集的每个元素进行操作
    3. 由于惰性机制，每次在有行动操作后才会从头进行计算(优点是可以只计算求结果时真正需要的数据，比如`first`行动会只扫描文件直到找到第一个匹配的行就停止)。为避免多次行动重复计算，可以用对计算结果进行持久化`persist`。调用`cache`方法时会调用`persist(MEMORY_ONLY)`。
    4. 向操作传读函数时，如果传递的时类中的方法，或使用类中的值，会吧整个类一起发送出去(若类不可序列化则会报错)，解决办法是用一个局部变量来保存需要用的类成员
        ```scala
        class SearchFunctions(val query: String) {
            def isMatch(s: String): Boolean = {
                s.contains(query)
            }
            def getMatchesFunctionReference(rdd: RDD[String]): RDD[String] = {
                // 问题: "isMatch"表示"this.isMatch",因此我们要传递整个"this"
                rdd.map(isMatch)
            }
            def getMatchesFieldReference(rdd: RDD[String]): RDD[String] = {
                // 问题: "query"表示"this.query",因此我们要传递整个"this"
                rdd.map(x => x.split(query))
            }
            def getMatchesNoReference(rdd: RDD[String]): RDD[String] = {
                // 安全:只把我们需要的字段拿出来放入局部变量中
                val query_ = this.query
                rdd.map(x => x.split(query_))
            }
        }
        ```
    5. spark的分区partition与hadoop的shuffle本质上都是groupbykey的过程。spark由于其RDD的共享机制减少了网络通信开销
        - 窄依赖：新的RDD生成只依赖上一级RDD的确定分区(partition后)
        - 宽依赖：新的RDD生成依赖上一级RDD的所有分区，此时若按key操作则需要shuffle

## 共享变量

> 默认情况下，当Spark在集群的多个不同节点的多个任务上并行运行一个函数时，它会把函数中涉及到的每个变量，在每个任务上都生成一个副本。但是有时候，需要在多个任务之间共享变量，或者在任务（Task）和任务控制节点（Driver Program）之间共享变量。

1. 广播变量：允许程序开发人员在每个机器上缓存一个只读的变量，而不是为机器上的每个任务都生成一个副本。通过这种方式，就可以非常高效地给每个节点（机器）提供一个大的输入数据集的副本。Spark的“动作”操作会跨越多个阶段（stage），对于每个阶段内的所有任务所需要的公共数据，Spark都会自动进行广播。通过广播方式进行传播的变量，会经过序列化，然后在被任务使用时再进行反序列化。这就意味着，显式地创建广播变量只有在下面的情形中是有用的：当跨越多个阶段的那些任务需要相同的数据，或者当以反序列化方式对数据进行缓存是非常重要的。
2. 累加器：一个数值型的累加器，可以通过调用`sc.longAccumulator()`或`sc.doubleAccumulator()`来创建。运行在集群中的任务，就可以使用add方法来把数值累加到累加器上，但是，这些任务只能做累加操作，不能读取累加器的值，只有任务控制节点（Driver Program）可以使用value方法来读取累加器的值

## 数据操作
1. 文件数据读写
    - `sc.textFile(filePath)`
2. Hbase数据库
    - Hbase是一个分布式，面向列的非关系型分布式数据库，列可以划分列族
    - 通过java API调用
3. Spark SQL
    - 主要结构为`DataFrame`。RDD数据内部结构是不可知的，但是`DataFrame`是一种以RDD为基础的分布式数据集，提供了详细的结构信息。共同点为同样为惰性运算。
    - 创建方式
        - 直接创建: `spark.read`
        - RDD转换：
            - 隐式转换需要导入`spark.implicits._`。利用反射机制推断RDD模式时，需要首先定义一个`case class`，因为，只有`case class`才能被Spark隐式地转换为`DataFrame`
                ```scala
                case class Person(name: String, age: Long)
                val peopleRDD = sc.textFile("filePath")
                val peopleDF = peopleRDD.map(_.split(","))
                    .map(list => Person(list(0), list(1).toLong))
                    .toDF()
                peopleDF.createOrReplaceTempView("people")  // 必须注册为临时表才能进行SQL语句操作
                val queryDF = spark.sql("select name,age from people where age > 20")
                ```
            - 显式转换需要定义结构信息`StructType`
                ```scala
                // 定义模式字符串
                val schemaString = "name age".split(" ")
                // 生成模式
                // fields: Array[org.apache.spark.sql.types.StructField] = Array(StructField(name,StringType,true), StructField(age,LongType,true))
                val fields = Array(StructField(schemaString(0), StringType, nullable = true), StructField(schemaString(1), LongType, nullable = true))
                val schema = StructType(fields)
                // 定义数据
                val rowRDD = peopleRDD.map(_.split(","))
                    .map(list => Row(list(0), list(1).toLong))
                // 生成DataFrame
                val peopleDF = spark.createDataFrame(rowRDD, schema)
                ```
        - 读写Parquet：是一种列式存储格式，可以高效的存储具有嵌套字段的记录。存储非常高效
        - JDBC连接MySQL
        - Hive

## 流计算
1. 静态数据对应批量计算(hadoop)，流数据对应流计算(实时计算)
2. 流计算处理过程：
    - 实时采集
    - 实时计算
    - 实时查询
    ![流计算过程](http://dblab.xmu.edu.cn/blog/wp-content/uploads/2016/11/%E5%9B%BE10-5-%E6%B5%81%E8%AE%A1%E7%AE%97%E7%9A%84%E6%95%B0%E6%8D%AE%E5%A4%84%E7%90%86%E6%B5%81%E7%A8%8B.jpg)
3. Spark Streaming的基本原理是将实时输入数据流以时间片（秒级）为单位进行拆分(DStream)，然后经Spark引擎以类似批处理的方式处理每个时间片数据。
    - DStream内部是一个RDD序列，每个RDD对应一个计算周期。所以行动操作有foreachRDD(func)，其中func是在Driver中执行，但func调用的RDD操作是在worker中执行
    - Dstream和RDD一样是延迟执行，只有遇到action操作才会真正去计算。因此**在Dstream的内部RDD必须包含Action操作才能使接受到的数据得到处理**。即使代码中包含foreachRDD,但在内部却没有action的RDD，SparkStream只会简单地接受数据数据而不进行处理
4. Spark Streaming和Storm最大的区别在于，Spark Streaming无法实现毫秒级的流计算，而Storm可以实现毫秒级响应。
Spark Streaming无法实现毫秒级的流计算，是因为其将流数据按批处理窗口大小（通常在0.5~2秒之间）分解为一系列批处理作业，在这个过程中，会产生多个Spark 作业，且每一段数据的处理都会经过Spark DAG图分解、任务调度过程，因此，无法实现毫秒级相应。Spark Streaming难以满足对实时性要求非常高（如高频实时交易）的场景，但足以胜任其他流式准实时计算场景。相比之下，Storm处理的单位为Tuple，只需要极小的延迟。
5. 编写Spark Streaming程序的基本步骤是：
    1.**通过创建输入DStream来定义输入源**
    2.**通过对DStream应用转换操作和输出操作来定义流计算。**
    3.用streamingContext.start()来开始接收数据和处理流程。
    4.通过streamingContext.awaitTermination()方法来等待处理结束（手动结束或因为错误而结束）。
    5.可以通过streamingContext.stop()来手动结束流计算进程
6. 输入源：
    - 文件流：支持以秒为单位时间监听文件变化，对新增的文件进行实时计算`ssc.textFileStream(filePath)`
    - 套接字流：可以通过socket端口监听文件数据变化`ssc.socketTextStream(socketPath)`
    - RDD队列流：`ssc.queueStream(rddQueue)`
    - 流行日志采集系统：`Kafka`，`Flume`
7. 转换操作
    - 无状态转换：每个批次的处理不依赖以前的处理结果。eg：**监听时，只处理本次监听过程中获取的输入数据，不会与之前监听到的数据处理结果有关系**
    - 有状态转换：当前批次的处理需要使用之前批次的数据或者中间结果。有状态转换包括
        - 基于滑动窗口的转换：eg:`val wordCounts = pair.reduceByKeyAndWindow(_ + _,_ - _,Minutes(2),Seconds(10),2)`。当滑动窗口到达一个新的位置时，原来之前被窗口框住的部分数据离开了窗口，又有新的数据被窗口框住，但是，这时计算窗口内单词的词频时，不需要对当前窗口内的所有单词全部重新执行统计，而是只要把窗口内新增进来的元素，增量加入到统计结果中，把离开窗口的元素从统计结果中减去，这样，就大大提高了统计的效率。
        - 追踪状态变化的转换：在跨批次之间维护状态`updateStateByKey()`
8. 输出操作：
    - 输出到文本：`saveAsTextFiles(filePath)`
    - 写入数据库：`conn = java.sql.DriverManager.getConnection(url, user, password)`
9. 容错通过保存数据副本实现
