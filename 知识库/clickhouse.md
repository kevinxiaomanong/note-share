https://xie.infoq.cn/article/9f325fb7ddc5d12362f4c88a8（这个写得很棒 深入浅出 作者肯定研究过源码的）

### 一、概述与背景

目前业务中有大量实时分析需求，随着数据量的增加，基于行存储的OLTP数据库已经无法满足性能的需求，调研决定引入Clickhouse作为新系统的OLAP方案，以下简称CK

CK是一款列式存储的数据库管理系统，相比于传统行式数据库系统，列式存储数据库更适合OLAP的场景，注意这里存的概念，行式的存是指在磁盘上是以行为单位放一起存的，列式是指以列为单位存储，我们知道我们的业务是基于一条记录的，我们要改字段就找到行即可，可是数据分析这种事情往往是基于列的，这也意味着CK其实不适合来做业务修改，因为你要改信息 可能要遍历很多的列

列式数据库是以列相关存储架构进行数据存储的数据库，主要适合于批量数据处理和 OLAP。相对应的是行式数据库，数据以行相关的存储体系架构进行空间分配，主要适合于小批量的数据处理，常用于 OLTP。

这里是一个非常容易混淆的概念，对于CK来说，逻辑上的一行数据依旧是由多个不同的column组成，但列存储会将二维表中同一列的数据组织在一起，以Clickhouse为例：一列数据在物理硬盘上对应于一组文件，落盘存储前CK会对列数据进行压缩，相同类型的数据在一起压缩能提供更高的压缩比；读取时也会优先对SQL进行解析和裁剪，只加载指定列数据，能有效降低数据从文件系统中加载的耗时，以下为简化的存储格式

```undefined
1       |zhang        |san           |shanghai   |182xxxxxxxx|male
2       |li           |si            |beijing    |182xxxxxxxx|female
3       |wang         |wu            |shenzhen   |182xxxxxxxx|male
4       |he           |liu           |guangzhou  |182xxxxxxxx|unknow 

  ...   |   ...       |   ...        |   ...     | ...       | ...
  
row.bin |last_name.bin|first_name.bin|address.bin|phone.bin  |sex.bin
```

此外CK提供了丰富的表引擎来应对不同的需要，其中MergeTree引擎作为Clickhouse的核心



### 二、部署与上手

照着官网进行安装，然后使用systemctl start clickhouse-server,跑起来之后可以通过clickhouse-client连接

然后写sql



```undefined
insert into id_test2 values
(1,'607589cdcbc95900017ddf01')
(2,'607589cdcbc95900017ddf02')
(3,'607589cdcbc95900017ddf03')
(4,'607589cdcbc95900017ddf04')
(5,'607589cdcbc95900017ddf05')
(6,'607589cdcbc95900017ddf06')
(7,'607589cdcbc95900017ddf07')
(8,'607589cdcbc95900017ddf08')
(9,'607589cdcbc95900017ddf09')
```



通过这两个查询语句可以很轻松查询到数据库、存储位置的信息

sql
   SELECT name, data_path
   FROM system.databases;

sql
   SELECT database, name, data_paths FROM system.tables;



### 三、文件结构&索引

在ck里每个表都对应文件系统中的一个目录，目录中不同的文件保存了表数据与相关的属性信息

如果对表进行了分区，那么每个分区都是表分区下的一个子分区，

主键索引：primary.idx 用于存放稀疏索引的数据 通过查询条件和稀疏索引能快速过滤无用的数据，减少需要加载的数据量

{column}.bin:列数据的存储文件 默认采用lz4压缩格式

{column}.mrk2:列数据的标记信息，记录了数据快在bin文件中的偏移量，记录了某个压缩块在bin文件中的相对位置，以及稀疏索引对应数据在列存储文件中的位置

CK会先通过索引文件定位到标记信息，再根据标记信息直接从bin数据文件中读取数据



CK里是没有全局索引的，只有稀疏索引，通过将数据排序并建立稀疏索引的方式来加速数据的定位与查询，默认的索引粒度是8192，这意味着即使是1亿条数据只需要建立12208条索引，占用空间小，因此对ck来说primary.idx的数据是可以常驻内存的



那稀疏索引的缺点也明显 即只能定位到大概范围 但作为一个旨在处理海量数据的系统来说 这是合理的

所以CK很重要一点调优是在于索引粒度和索引字段的设置 一定要选择好区分度高的字段来做索引



clickhouse中列数据的存储分为两个文件{column}.bin 和 .mrk2 

在理解这两个文件前要先理解MergeTree存储里的granule和mark的概念，即CK会根据index_granularity的设置将数据分成多个granule，每个granule中索引列的第一个记录会作为索引写入到primary.idx 其他非索引列也会使用相同的策略生产一条mark数据写入其对应的mrk2文件中，并于主键索引对应

此外bin文件的内容是压缩存放的，为了保证读写的效率，压缩的粒度有两个参数控制，max_compress_block_size和min_compress_block_size 默认大小分别为1mb和64kb

假设每次写入的数据大小为x，如果x<64kb 会等待下一批数据 

如果x在64kb和1mb之间 会直接生成一个压缩块 如果x>1mb会进行分割为多个压缩块

这会导致一个granule的数据可能在多个压缩块中 同时一块压缩可能存有多个granule数据 需要mark记录保存好对应关系





## CK接入--CHPASS

OLAP平台由CHPASS（集群统一平台）和IData（统一数据中台）组成，支持将Mysql、Hive，Kafka等离线/实时的数据通过IData的离线作业任务将数据写入CK集群

接入大致流程：在CHPASS中新建集群，在IData中申请相关权限，然后可以通过各种数据服务（artNova/个人应用app）来使用数据









