---
author : "jihongwei"
tags : ["clickhouse"]
date : 2021-08-19T11:24:00Z
title : "clickhouse原理"
---

* 主键索引文件：primary.idx
* 数据标记文件：column.mrk
* 数据文件：column.bin

使用方式：

&nbsp;&nbsp;MergeTree会先通过稀疏索引(primary.idx)找到对应数据的偏移量信息(column.mrk)，再通过偏移量直接从数据文件(column.bin)读取数据。所以主键索引和数据标记的间隔粒度相同，均有index_granularity参数决定，数据文件也会依据该参数生成压缩数据块。
![clickhouse](https://demoio.cn:90/blog-image/clickhouse数据块.png)
1. 给定index_granularity的标号，去.mrk文件里面读出压缩块偏移量，然后再找到下一个有变化的偏移量，二者之间就是本个granularity的压缩块的位置；
2. 把压缩块拉到内存里进行解压；
3. 根据.mrk文件中的解压缩块的偏移量（块内偏移量，类似于第一步），扫描相应位置，读到数据。

![clickhouse](https://demoio.cn:90/blog-image/clickhouse索引文件结构.png)

&nbsp;&nbsp;在Clickhouse中，数据表中最终的数据是按列存储在Part目录下的各自对应的列存文件下的。那么，查询时应该如何从不同的列存文件中查找到同属于某一行的数据？

&nbsp;&nbsp;在Clickhouse中，如果使用MergeTree系列的表引擎，则必须指定一个排序键。排序键可以由一列或多列组成，在数据写入磁盘文件中时，数据行的就会按照制定的顺序排列，随后按列拆分写入各个列存文件中。

&nbsp;&nbsp;索引文件和标记文件实际是一对多的关系（主键只有一个，但列有很多（一列一个标记文件）），将索引文件和标记文件剥离后，索引文件大小比较小，可以常驻内存。查询到数据范围后，可以直接计算出数据对应在标记文件中的位置，做最小化查询。

&nbsp;&nbsp;索引文件：

![clickhouse](https://demoio.cn:90/blog-image/索引文件内容示例图.png)

&nbsp;&nbsp;数据标记文件：

![clickhouse](https://demoio.cn:90/blog-image/标记文件示例图.png)


&nbsp;&nbsp;这里的行号其实只是用于关联起索引和标记两个表，而这两个表的数据在行方向其实是一一顺序对应的，因此行号其实是实际上是不需要存在文件中的，这也是Clickhouse追求极致性能，数据尽量精简的一个体现。可以通过od查看一下真实的数据索引文件中和数据标记文件中的数据：
```
// 数据索引文件，存储的是一个个主健的值，这里主键只有一列
root@clickhouse-0:20210110_0_123_3_341# od -l -j 0 -N 80 --width=8 primary.idx
0000000 5670735277560
0000010 24176312979802680
0000020 48658950580167724
0000030 72938406171441414
0000040 96513037981382350
0000050 120656338641242134
0000060 145024009883201898
0000070 169438340458750532
0000100 193384698694174670
0000110 217869890390743588

// 数据标记文件，可以看作三列，分别是数据压缩块位置，数据块内偏移和granule大小
root@clickhouse-0:20210110_0_123_3_341# od -l -j 0 -N 240 --width=24 ./value9.mrk2
0000000 0 0 8192
0000030 0 32768 8192
0000060 65677 0 8192
0000110 65677 32768 8192
0000140 129357 0 8192
0000170 129357 32768 8192
0000220 193106 0 8192
0000250 193106 32768 8192
0000300 258449 0 8192
0000330 258449 32768 8192
```


&nbsp;&nbsp;Mark标识文件：action_id.mrk2、avatar_id.mrk2等都是列存文件中的Mark标记，Mark标记和MergeTree列存中的两个重要概念相关：Granule和Block。

&nbsp;&nbsp;Granule是数据按行划分时用到的逻辑概念。关于多少行是一个Granule这个问题，在老版本中这是用参数index_granularity设定的一个常量，也就是每隔确定行就是一个Granule。在当前版本中有另一个参数index_granularity_bytes会影响Granule的行数，它的意义是让每个Granule中所有列的sum size尽量不要超过设定值。老版本中的定长Granule设定主要的问题是MergeTree中的数据是按Granule粒度进行索引的，这种粗糙的索引粒度在分析超级大宽表的场景中，从存储读取的data size会膨胀得非常厉害，需要用户非常谨慎得设定参数。

&nbsp;&nbsp;Block是列存文件中的压缩单元。每个列存文件的Block都会包含若干个Granule，具体多少个Granule是由参数min_compress_block_size控制，每次列的Block中写完一个Granule的数据时，它会检查当前Block Size有没有达到设定值，如果达到则会把当前Block进行压缩然后写磁盘。
从以上两点可以看出MergeTree的Block既不是定data size也不是定行数的，Granule也不是一个定长的逻辑概念。所以我们需要额外信息快速找到某一个Granule。这就是Mark标识文件的作用，它记录了每个Granule的行数，以及它所在的Block在列存压缩文件中的偏移，同时还有Granule在解压后的Block中的偏移位置。

#### 索引：

&nbsp;&nbsp;主键索引：primary.idx是表的主键索引。ClickHouse对主键索引的定义和传统数据库的定义稍有不同，它的主键索引没用主键去重的含义，但仍然有快速查找主键行的能力。ClickHouse的主键索引存储的是每一个Granule中起始行的主键值，而MergeTree存储中的数据是按照主键严格排序的。所以当查询给定主键条件时，我们可以根据主键索引确定数据可能存在的 ，再结合上面介绍的Mark标识，我们可以进一步确定数据在列存文件中的位置区间。ClickHoue的主键索引是一种在索引构建成本和索引效率上相对平衡的粗糙索引。

&nbsp;&nbsp;MergeTree的主键序列默认是和Order By序列保存一致的，但是用户可以把主键序列定义成Order By序列的部分前缀。

&nbsp;&nbsp;分区键索引：MergeTree存储会把统计每个Data Part中分区键的最大值和最小值，当用户查询中包含分区键条件时，就可以直接排除掉不相关的Data Part，这是一种OLAP场景下常用的分区裁剪技术。

&nbsp;&nbsp;Skipping索引：Merge Tree中 的Skipping Index是一类局部聚合的粗糙索引。用户在定义skipping index的时候需要设定granularity参数，这里的granularity参数指定的是在多少个Granule的数据上做聚合生成索引信息。用户还需要设定索引对应的聚合函数，常用的有minmax、set、bloom_filter、ngrambf_v1等，聚合函数会统计连续若干个Granule中的列值生成索引信息。Skipping索引的思想和主键索引是类似的，因为数据是按主键排序的，主键索引统计的其实就是每个Granule粒度的主键序列MinMax值，而Skipping索引提供的聚合函数种类更加丰富，是主键索引的一种补充能力。另外这两种索引都是需要用户在理解索引原理的基础上贴合自己的业务场景来进行设计的。

&nbsp;&nbsp;对于skipping索引GRANULARITY 是稀疏点选择上的 granule 颗粒度， GRANULARITY 1 表示每 1 个 granule 选取一个：

![clickhouse](https://demoio.cn:90/blog-image/clickhouse_granularity_1.png)

&nbsp;&nbsp;如果定义为GRANULARITY 2 ，则 2 个 granule 选取一个：

![clickhouse](https://demoio.cn:90/blog-image/clickhouse_granularity_2.png)

&nbsp;&nbsp;全景图:

![clickhouse](https://demoio.cn:90/blog-image/clickhouse_索引全景图.png)

检索过程：
MergeTree存储在收到一个select查询时会先抽取出查询中的分区键和主键条件的KeyCondition，KeyCondition类上实现了以下三个方法，用于判断过滤条件可能满足的Mark Range。
```
/// Whether the condition is feasible in the key range.
 
/// left_key and right_key must contain all fields in the sort_descr in the appropriate order.
 
/// data_types - the types of the key columns.
 
bool mayBeTrueInRange(size_t used_key_size, const Field * left_key, const Field * right_key, const DataTypes & data_types) const;
 
/// Whether the condition is feasible in the direct product of single column ranges specified by `parallelogram`.
 
bool mayBeTrueInParallelogram(const std::vector<Range> & parallelogram, const DataTypes & data_types) const;
 
/// Is the condition valid in a semi-infinite (not limited to the right) key range.
 
/// left_key must contain all the fields in the sort_descr in the appropriate order.
 
bool mayBeTrueAfter(size_t used_key_size, const Field * left_key, const DataTypes & data_types) const;
```
&nbsp;&nbsp;MergeTree Data Part中的列存数据是以Granule为粒度被Mark标识数组索引起来的，而Mark Range就表示Mark标识数组里满足查询条件的下标区间。

![clickhouse](https://demoio.cn:90/blog-image/clickhouse_mark_range.png)


&nbsp;&nbsp;索引检索的过程中首先会用分区键KeyCondition裁剪掉不相关的数据分区，然后用主键索引挑选出粗糙的Mark Range，最后再用Skipping Index过滤主键索引产生的Mark Range。用主键索引挑选出粗糙的Mark Range的算法是一个不断分裂Mark Range的过程，返回结果是一个Mark Range的集合。起始的Mark Range是覆盖整个MergeTree Data Part区间的，每次分裂都会把上次分裂后的Mark Range取出来按一定粒度步长分裂成更细粒度的Mark Range，然后排除掉分裂结果中一定不满足条件的Mark Range，最后Mark Range到一定粒度时停止分裂。这是一个简单高效的粗糙过滤算法。

&nbsp;&nbsp;首先根据查询解析到QUERY的PRIMARY KEY范围；
从最大的数据范围内开始进行递归查找，如果QUERY范围和数据范围有交集，则划分成8个子区间（可以配置），如果已经不能拆8份了，即数据范围的markrange步长<8，那么就返回；如果没交集，那直接剪枝掉。
最终把返回的markRange区间合并起来


&nbsp;&nbsp;使用Skipping Index过滤主键索引返回的Mark Range之前，需要构造出每个Skipping Index的IndexCondition，不同的Skipping Index聚合函数有不同的IndexCondition实现，但判断Mark Range是否满足条件的接口和KeyCondition是类似的。

#### 数据扫描：

&nbsp;&nbsp;MergeTree的数据扫描部分提供了三种不同的模式：

1. Final模式：该模式对CollapsingMergeTree、SummingMergeTree等表引擎提供一个最终Merge后的数据视图。前文已经提到过MergeTree基础上的高级MergeTree表引擎都是对MergeTree Data Part采用了特定的Merge逻辑。它带来的问题是由于MergeTree Data Part是异步Merge的过程，在没有最终Merge成一个Data Part的情况下，用户无法看到最终的数据结果。所以ClickHouse在查询是提供了一个final模式，它会在各个Data Part的多条BlockInputStream基础上套上一些高级的Merge Stream，例如DistinctSortedBlockInputStream、SummingSortedBlockInputStream等，这部分逻辑和异步Merge时的逻辑保持一致，这样用户就可以提前看到“最终”的数据结果了。

2. Sorted模式：sort模式可以认为是一种order by下推存储的查询加速优化手段。因为每个MergeTree Data Part内部的数据是有序的，所以当用户查询中包括排序键order by条件时只需要在各个Data Part的BlockInputStream上套一个做数据有序归并的InputStream就可以实现全局有序的能力。

3. Normal模式：这是基础MergeTree表最常用的数据扫描模式，多个Data Part之间进行并行数据扫描，对于单查询可以达到非常高吞吐的数据读取。

Normal模式中几个关键的性能优化点：

1. 并行扫描：传统的计算引擎在数据扫描部分的并发度大多和存储文件数绑定在一起，所以MergeTree Data Part并行扫描是一个基础能力。但是MergeTree的存储结构要求数据不断mege，最终合并成一个Data Part，这样对索引和数据压缩才是最高效的。所以ClickHouse在MergeTree Data Part并行的基础上还增加了Mark Range并行。用户可以任意设定数据扫描过程中的并行度，每个扫描线程分配到的是Mark Range In Data Part粒度的任务，同时多个扫描线程之间还共享了Mark Range Task Pool，这样可以避免在存储扫描中的长尾问题。

2. 数据Cache：MergeTree的查询链路中涉及到的数据有不同级别的缓存设计。主键索引和分区键索引在load Data Part的过程中被加载到内存，Mark文件和列存文件有对应的MarkCache和UncompressedCache，MarkCache直接缓存了Mark文件中的binary内容，而UncompressedCache中缓存的是解压后的Block数据。

3. SIMD反序列化：部分列类型的反序列化过程中采用了手写的sse指令加速，在数据命中UncompressedCache的情况下会有一些效果。
PreWhere过滤：ClickHouse的语法支持了额外的PreWhere过滤条件，它会先于Where条件进行判断。当用户在sql的filter条件中加上PreWhere过滤条件时，存储扫描会分两阶段进行，先读取PreWhere条件中依赖的列值，然后计算每一行是否符合条件。相当于在Mark Range的基础上进一步缩小扫描范围，PreWhere列扫描计算过后，ClickHouse会调整每个Mark对应的Granule中具体要扫描的行数，相当于可以丢弃Granule头尾的一部分行。



#### 动态代码生成Runtime Codegen

&nbsp;&nbsp;在经典的数据库实现中，通常对表达式计算采用火山模型，也即将查询转换成一个个operator，比如HashJoin、Scan、IndexScan、Aggregation等。为了连接不同算子，operator之间采用统一的接口，比如open/next/close。在每个算子内部都实现了父类的这些虚函数，在分析场景中单条SQL要处理数据通常高达数亿行，虚函数的调用开销不再可以忽略不计。另外，在每个算子内部都要考虑多种变量，比如列类型、列的size、列的个数等，存在着大量的if-else分支判断导致CPU分支预测失效。

&nbsp;&nbsp;ClickHouse实现了Expression级别的runtime codegen，动态地根据当前SQL直接生成代码，然后编译执行。如下图例子所示，对于Expression直接生成代码，不仅消除了大量的虚函数调用（即图中多个function pointer的调用），而且由于在运行时表达式的参数类型、个数等都是已知的，也消除了不必要的if-else分支判断。


#### DISTRIBUTED写入

&nbsp;&nbsp;DISTRIBUTED表的数据写入一个shard节点，首先是，把本shard内的数据写入分区，然后把其他shard分区的挑出来，分别写到一个固定位置去，形成分区；本shard内有一个对固定位置的监视器，监测到分区目录变化后，会根据目录名，立即与远程shard建立联系，压缩传递数据。秉着谁执行谁负责的原则，写入shard负责确保数据都已经正确写入，结束返回。

&nbsp;&nbsp;写入的过程可以设置同步/异步，异步不用等待远程写完，同步需要设置超时。
&nbsp;&nbsp;上面的过程只考虑了sharding，没有考虑replica，replica的同步写入可以有两种模式，通过配置文件写死配置。第一种可以通过上述的方式，由distribute引擎写入replica，然而这种方式，写入节点要传输和写入的replica太多了，容易造成单点瓶颈；另一种方式是通过Repliacted-MergeTree，利用zk传输日志来进行同步，这样写入节点只需在每个shard选一个replica写入即可，具体选哪个可以根据一个全局计数器errors_count来选择。

#### DISTRIBUTED查询

&nbsp;&nbsp;分布式查询，要在每一个shard选择一个replica，这就涉及一个负载均衡算法，由参数控制，有以下四种
1. random：默认算法。选择errors_count最小的replica，相等的话就随机选择；
2. nearest_hostname：还是首先选择errors_count最小的，然后选hostname最接近的；

3. in_order：先选errors_count，相等的话看配置文件的顺序；
4. first_or_random：先选errors_count，然后按配置文件的第一个看，如果第一个不可用就随机选了。

&nbsp;&nbsp;可想而知，查询也是谁执行谁负责，谁是入口查询节点，谁就要串联整个查询过程。包括分割分布式查询为本地子查询，选择连接其他shard的节点，传递SQL，收到结果数据，UNION返回结果。
使用GLOBAL优化分布式子查询。

&nbsp;&nbsp;对于SQL中的子查询和JOIN，很可能在一条SQL中出现两次分布式表，那么就会出现SQL被反复传递以获取信息，导致查询请求幂次扩大的问题，为了解决这个问题，在第二/N次出现分布式表的时候，加入GLOBAL字段，查询中首先执行GLOBAL子查询，得到结果返回，构成内存表，再广播给各个节点，最后执行主查询。

&nbsp;&nbsp;由于这种分布式IN/JOIN方式，子句返回的结果不能太大，要在内存中放得下，因此最好要提前DISTINCT或者筛选掉一部分。


>https://blog.csdn.net/night_zw/category_10751307.html
https://my.oschina.net/u/2000675/blog/4655098
https://www.jianshu.com/p/3fdafe7ecb20
https://www.cnblogs.com/eedbaa/p/14512803.html
https://blog.csdn.net/weixin_45320660/article/details/112761790
https://mp.weixin.qq.com/s?__biz=Mzg3NTI2MzMzMw%3D%3D&chksm=cec56e78f9b2e76e967d3d7090eb6dad6df5f5d3fc5e5e11fb1a42934e36369ca9c66b3c6ac6&idx=1&mid=2247486066&scene=21&sn=072a302c459eaf55f7910bfc0fefadf9#wechat_redirect