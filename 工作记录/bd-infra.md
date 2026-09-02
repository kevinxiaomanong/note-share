# Starrocks

## 1、产品简介

### 什么是Starrocks

架构：系统由前端和后端两种组件组成

前端节点称为FE 后端节点有BE和CN（计算节点）两种类型



当使用本地存储数据时需要部署BE,当数据存储在对象存储或HDFS时需要部署CN，SR不依赖任何外部组件，简化了部署和维护，节点可以水平扩展而不影响服务正常运行，且存在元数据和服务数据副本机制保障高可用



且兼容Mysql协议 支持标准SQL 可以通过Mysql客户端连接到SR实时OLAP



### **架构**

**存算一体**

本地存储为实时查询提供了更低的查询延迟，因为避免了数据传输和复制，适合追求最佳查询性能的场景

FE+BE节点 share-nothing架构 （CK就是典型的share-nothing架构）



FE节点：元数据管理和执行计划构建，每个FE用berkeley db java版 在内存中存储和维护元信息的完整副本，具体的角色有

1. leader节点 负责读写元数据 然后使用Raft协议将元数据更改同步到Follow节点和Observer节点，只有在元数据更改同步超一半的follower节点后数据写入才被认为成功，从技术实现上本质也是一个follower节点，是选举出来的，只有超过一半的follower节点活跃才可以主节点选举
2. follower节点 只能读元数据
3. Observer节点 从leader fe同步和重放日志以更新元数据，主要用于增加集群的查询并发，不参与主节点选举



BE节点：

1. 数据存储 FE根据预定义规则将数据分发到各个BE，BE将数据写入所需格式并建立索引
2. SQL执行 FE根据查询语义将SQL解析为逻辑执行计划，然后转为可以在BE上执行的物理计划



**存算分离**

FE+CN+store

对象存储和HDFS提供低成本、高可靠、可扩展的优势，且CN也可以快速扩缩（无状态）

CN仅负责数据计算任务和缓存热数据，数据存储在低成本且可靠的远端存储服务

这种架构降低存储成本且有高度的弹性和可扩展性，需要配置后端对象存储



### 产品特性

fragment：分段   Scatter：分散 shuffle：改组 protocol：协议

materialized view：物化视图

**MPP**

massively parallel processing，

Scatter-Gather

与很多数据分析系统采用的Scatter-Gather分布式执行框架不同，MPP分布式执行框架可以利用更多的资源处理查询请求，在Scatter-Gather框架中只有Gather节点能处理最后一级的汇总计算，而在MPP框架中数据会被Shuffle到多个节点并行汇总，在复杂计算时（高基数的group by、大表join）有明显性能优势



对SR来说，查询被切分成多个Fragment，每个逻辑执行单元由一个或多个物理执行单元实现，节点间通过Exchange算子传递数据（shuffle/broadcast），在复杂计算时，MPP相比Scatter-Gather有明显模式



















**向量化执行引擎**

starrock全面实现向量化引擎，充分发挥CPU的处理能力，能够充分利用CPU的SIMD质量，用更少的指令数目完成更多的数据操作

**存算分离**

存储与计算解耦，各自独立服务扩缩容，解决了在存算一体模式下的计算与存储等比例扩缩容所带来的资源浪费问题，存储可选例如S3/OSS的对象存储，也支持HDFS

**CBO优化器**

在多表关联查询场景下，不同执行计划的复杂度可能差几个数量级，从众多的可能中选择一个最优的计划是一个NP-Hard问题

starrocks从零实现了基于代价的CBO优化器，cost based optimizer，SR会比同类产品更好地支持多表关联查询

**可实时更新的列式存储引擎**

数据按列的方式进行存储，相同类型的数据连续存放，可以用更高效的编码方式获得更高的压缩比，且大部分OLAP场景查询只涉及部分列，列存只需要读取部分列数据，极大降低磁盘IO吞吐，这一点和CK是一样的

StarRocks 能够支持秒级的导入延迟，提供准实时的服务能力。StarRocks 的存储引擎在数据导入时能够保证每一次操作的 ACID。一个批次的导入数据生效是原子性的，要么全部导入成功，要么全部失败。并发进行的各个事务相互之间互不影响，对外提供 Snapshot Isolation 的事务隔离级别。

**智能物化视图**

starrocks的materialized view可以自动根据原始表更新数据，且在查询规划时能定位到最适合的物化视图上进行查询加速

**数据湖分析**

starrocks不仅能高效地分析本地存储的数据，也可以作为计算引擎直接分析数据湖中的数据，用户可以通过Starrocks提供的External Catalog，查询存储在Hive、Iceberg、Hudi、Lake等数据湖上的数据，无需进行数据迁移，支持的存储系统包括HDFS/S3/OSS

在数据湖分析场景，Starrcoks主要负责数据的计算分析，数据湖主要负责数据存储、组织和维护，使用数据湖的优势在于使用开放的存储格式和灵活多变的schema定义方式，可以让BI/ADHOC/报表等业务有统一的single source of truth，而Starrocks作为数据湖的计算引擎可以充分发挥向量化引擎和CBO的优势提升数据湖分析的性能







## 2、最佳实践

### **分区与分桶**

分区是按某个字段的值把数据物理切分成独立目录。

/table/
  dt=2026-07-01/   ← 一个分区目录
    data.orc
  dt=2026-07-02/
    data.orc
  dt=2026-07-03/
    data.orc

查询 WHERE dt = '2026-07-02' 时，直接跳过其他目录，只扫描对应分区 → 分区裁剪（Partition Pruning）。

适合：时间（dt）、地区（region）、业务线（bu）等低基数、常作为过滤条件的字段。

问题：字段基数太高（如 user_id）会产生百万级目录，元数据爆炸。

---

分桶（Bucket / Clustering）

在分区内部，按某个字段做 hash 取模，把数据分散到固定数量的文件（桶）里。

/table/dt=2026-07-01/
  bucket_0.orc   ← user_id % 4 == 0 的数据
  bucket_1.orc   ← user_id % 4 == 1 的数据
  bucket_2.orc
  bucket_3.orc

优势：

- Join 加速：两张表按同一字段分桶，Join 时同号桶直接配对，避免全量 Shuffle（Bucket Map Join）
- 采样高效：TABLESAMPLE(BUCKET 1 OUT OF 4) 直接读 bucket_0，保证数据均匀
- 数据倾斜缓解：hash 打散后每桶数据量相对均衡



两者通常组合使用：先按dt分区控制扫描范围，再按user_id分桶提升Join性能

典型数量：

每个表100-10000分区，每个分区10-120个桶



**分桶**

分桶可以分为Hash Bucketing和Random Bucketing

hash：

通过对一个列或多个列进行哈希，将行分配到tablet，创建后tablet数量固定

 要求：

- 必须预先选择一个稳定、均匀、高基数的键 防止哈希桶之间的数据倾斜
- 初始选择合适的桶大小，理想每个bucket在1-10gb之间

优势：

- 选择性的过滤和join 触达更少的tablet
- shuffle join

劣势：

- 如果数据分布倾斜，容易出现热点tablet
- 数量静态



```
-- 事实表按 (customer_id) 哈希分桶并按天分区
CREATE TABLE sales (
  sale_id bigint,
  customer_id int,
  sale_date date,
  amount decimal(10,2)
) ENGINE = OLAP
DISTRIBUTED BY HASH(customer_id) BUCKETS 48
PARTITION BY date_trunc('DAY', sale_date)
PROPERTIES ("colocate_with" = "group1");

-- 维度表在相同键和桶数上哈希分桶，与销售表共置
CREATE TABLE customers (
  customer_id int,
  region varchar(32),
  status tinyint
) ENGINE = OLAP
DISTRIBUTED BY HASH(customer_id) BUCKETS 48
PROPERTIES ("colocate_with" = "group1");


-- StarRocks 可以进行分桶裁剪
SELECT sum(amount) 
FROM sales
WHERE customer_id = 123

-- StarRocks 可以进行本地聚合
SELECT customer_id, sum(amount) AS total_amount
FROM sales
GROUP BY customer_id
ORDER BY total_amount DESC LIMIT 100;

-- StarRocks 可以进行Colocate Join
SELECT c.region, sum(s.amount)
FROM sales s JOIN customers c USING (customer_id)
WHERE s.sale_date BETWEEN '2025-01-01' AND '2025-01-31'
GROUP BY c.region;
```

- 分桶裁剪 where customer_id = 123这样的customer_id谓词可启用分桶裁剪，使查询仅访问一个tablet
- 本地聚合：当哈希分布键是聚合键的子集时，sr可跳过洗牌聚合阶段
- Colocate join 由于两个表共享桶数量和键，join时连接各自的tablet对



random bucketing

行按轮询分配，无需指定键，会在分区增大时动态拆分tablet，但每次查询会扫描分区内的所有tablet

建议为随机分桶设置合适的桶大小 如1gb以启用自动拆分

对哈希分桶监控tablet大小，单个tablet超过5-10gb之前重分片



Sr支持在一个集群内使用多种存储介质，可以将新数据分区放在SSD盘，旧数据放SATA盘节省数据存储成本





### 表聚簇

即排序键设计，一个设计良好的排序键，用小而可预期的导入开销换来扫描时延、存储效率与CPU利用率的提升

Starrocks的三层组织

- 分区 低基数 如dt、region
- 分桶 高基础 如user_id hash打散到各tablet
- 排序 tablet内部按排序键有序存储





## 3、表设计

### catalog、database、table

Starrocks使用Internal Catalog来管理内部数据，使用External Catalog来连接数据湖中的数据

internal catalog可以包含一个或多个数据库，来存储、管理和操作SR中的数据例如表、物化视图、视图等

每个集群都有且只有一个名为default_catalog的Internal Catalog，包含一个或多个数据库

也可以将Starrocks作为查询引擎，直接查询湖上数据无需导入数据至starrocks，external catalog用于连接数据湖中的数据



database是包含表、视图、物化视图等对象的集合





table在sr里分内部表和外部表

内部表归属于internal catalog的数据库，数据保存在SR中，物理上是按列存储，即一列数据经过分块编码压缩然后持久化存储，根据约束类型分为：主键表、明细表、聚合表和更新表，采用分区+分桶的两级数据分布策略实现均匀分布，且分桶以多副本形式均匀分布至BE节点保证高可用

外部表的实际数据存在外部数据源，SR只保存表的元数据



物化视图：特殊的物理表，存储基于基表的预计算结果，又分为同步物化视图和异步物化视图

视图：也叫逻辑视图，是虚拟表不实际存储数据，每次在查询中引用某个视图时都会运行定义该视图的查询



临时表：在处理数据时可能需要保存中间计算结果以便后续复用，临时表允许您将临时数据暂存在表中（例如 ETL 计算的中间结果），其生命周期与 Session 绑定，并由 StarRocks 管理。Session 结束时，临时表会被自动清除。临时表仅在当前 Session 内可见，不同的 Session 内可以创建同名的临时表。



### 数据分布

现代分布式数据库中常见的数据分布方式有：Round-Robin、Range、List和Hash



分区键和分区粒度

如果表单月数据量很小，其实可以按月分区，减少元数据数量，如果大部分查询精确到天可以按天分区有效分区裁剪

正常一般选择时间进行分区以优化大量删除过期数据带来的性能问题 且方便冷热数据分级存储



分桶设置：

如果查询海量数据且查询时经常使用一些列作为条件列，建议用哈希分桶，这样在查询时只需要扫描和计算查询命中的少量分桶即可，默认是随机分桶

哈希分桶建议使用高基数且常作为查询条件的列，但如果分桶列分布不均，根据二八规则可能造成数据倾斜，导致系统局部的性能瓶颈，需要调整分桶字段将数据打散



短查询：扫描数据量不大、单机就能完成扫描的查询

长查询：扫描数据量大、并行扫描能显著提升性能的查询





对于 StarRocks 而言，分区和分桶的选择是非常关键的。在建表时选择合理的分区键和分桶键，可以有效提高集群整体性能。因此建议在选择分区键和分桶键时，根据业务情况进行调整。

- **数据倾斜**

  如果业务场景中单独采用倾斜度大的列做分桶，很大程度会导致访问数据倾斜，那么建议采用多列组合的方式进行数据分桶。

- **高并发**

  分区和分桶应该尽量覆盖查询语句所带的条件，这样可以有效减少扫描数据，提高并发。

- **高吞吐**

  尽量把数据打散，让集群以更高的并发扫描数据，完成相应计算。

- **元数据管理**

  Tablet 过多会增加 FE/BE 的元数据管理和调度的资源消耗。



### 数据压缩

SR支持对表和索引数据进行压缩，有助于节省存储空间+减少IO，但压缩和解压缩需要额外的CPU资源

只能在创建表时设置后续无法修改，默认使用LZ4压缩算法，具有较为均衡的压缩比和解压缩性能





## 4、数据湖

starrocks湖仓一体方案重点在于：

- 规范的Catalog及元数据服务集成
- 弹性可扩展的计算节点（简称CN）
- 灵活的缓存机制



### Catalog

数据目录，实现在一套系统内同时维护内、外部数据，外部数据源例如Hive、Iceberg、Hudi这种



external catalog：用于连接外部metastore，可以创建不同类型数据源的external catalog，例如Hive catalog/Iceberg catalog/Jdbc catalog，在使用external catalog查询数据时，SR会用到外部数据源的两个组件：

- 元数据服务 用于将元数据暴露给SR的FE进行查询规划
- 存储系统 数据文件以不同格式存储在分布式文件系统或对象存储系统中，当FE将生成的查询计划分发给各个BE或CN后，会并行扫描存储系统的目标数据，



但有的时候我们其实有跨catalog查询的场景，可以通过catalog_name.db_name.table_name的格式引用目标数据，例如一下其实是跨catalog的join查询

```
SELECT * FROM hive_catalog.hive_db.hive_table h JOIN default_catalog.olap_db.olap_table o WHERE h.id = o.id;
```



**Default Catalog**

每个集群有且只有一个Internal Catalog





















# Clickhouse

1、

https://golangguide.top/%E4%B8%AD%E9%97%B4%E4%BB%B6/ClickHouse/%E9%9D%A2%E8%AF%95%E9%A2%98.html#clickhouse-%E7%9A%84%E6%A0%B8%E5%BF%83%E7%89%B9%E6%80%A7

- 基数  在数据库领域，基数是指某一列中不同值的数量，比如对于用户表的userid列来说 有100w个用户就有100w个基数 

因此基数大的列不适合做索引列，因为CK的索引是稀疏索引，它会对数据进行分组，然后记录分组的边界值，当基数特别大时，数据分布会非常分散，索引难以有效地缩小查询范围



2、CK数据写入

在CK里分布式表本身不存储数据，更像一个代理将查询路由到本地表所在的节点上

有以下几种关联方式：

rand():随机 这种可以保证均匀 能均匀分散数据负载 但无法保证数据的局部性不利于关联查询

hash：根据指定列的哈希值将数据分发到不同节点，配置分布式表时可以使用hash函数来实现 有利于关联查询

但容易产生数据倾斜

分片键：指定一个或多个列的值让CK根据分片键的值将数据分发到不同节点



我们是推荐用Local to Local的方式，就是不通过分布式表进行写入，而是直接将数据写入到各个节点的本地表

这样写入性能更好，通过分布式表写入，数据先到分布式表所在节点，然后再将数据转发到各个本地表所在节点

过程是存在网络开销和处理延迟的，直接写入本地表能充分利用各个节点的资源，不会让分布式表所在的节点成为整个集群的写入瓶颈，而且这样写入单节点是独立的



一般我们会为每个节点都创建分布式表并非在一个节点去创建，这样客户端不管连接到哪个节点都可以用该节点上的分布式表发起查询，也保证系统的负载均衡，提高容错性



Replacing merge tree

因为CK是不好做删除或修改的 但我们有些时候存在数据重复

采用replacing merge tree

数据去重：在合并分区时删除重复数据（CK的分区合并是几分钟触发一次）跨分区、跨分片无法去重



cdata主要是开发类埋点 闪退白屏这样的





```
CREATE TABLE ubtprimarychdb.htl_c_app_list_tag_show_all
(
    `arrange` LowCardinality(String),
    `ascityid` LowCardinality(String),
    `attributeid` LowCardinality(String),
    `card_below` LowCardinality(String),
    `checkin` LowCardinality(String),
    `checkout` LowCardinality(String),
    `cityid` LowCardinality(String),
    `comment_keywords` LowCardinality(String),
    `comment_keywordtype` LowCardinality(String),
    `countryname` LowCardinality(String),
    `countrytype` LowCardinality(String),
    `currenthotelcount` LowCardinality(String),
    `dispatchid` LowCardinality(String),
    `dist` LowCardinality(String),
    `distanceremark` LowCardinality(String),
    `disttype` LowCardinality(String),
    `fastfilterlist|fastfilterid` Array(String),
    `fastfilterlist|fastfiltername` Array(String),
    `fastfilterlist|fastfilterrank` Array(Float64),
    `fastfilterlist|fastfiltersubtype` Array(String),
    `fastfilterlist|fastfiltertitle` Array(String),
    `fastfilterlist|fastfiltertype` Array(String),
    `fastfilterlist|filterselectstate` Array(Float64),
    `fastfilterlist|filtervalue` Array(String),
    `fastfilterlist|subtitle` Array(String),
    `fh_price` Float64,
    `filterlist|fastfiltername` Array(String),
    `filterlist|fastfiltersubtype` Array(Float64),
    `filterlist|fastfiltertype` Array(Float64),
    `filterlist|filterid` Array(String),
    `filterlist|filtername` Array(String),
    `filterlist|filterposition` Array(Float64),
    `filterlist|filterselectstate` Array(Float64),
    `filterlist|filtersubtype` Array(String),
    `filterlist|filtertitle` Array(String),
    `filterlist|filtertitleCNname` Array(String),
    `filterlist|filtertype` Array(String),
    `filterlist|filtervalue` Array(String),
    `fq_price` Float64,
    `frompage` LowCardinality(String),
    `hotelcardheight` LowCardinality(String),
    `hotelnum` LowCardinality(String),
    `htlist_recomm_type` LowCardinality(String),
    `imageFilter` LowCardinality(String),
    `inputword` LowCardinality(String),
    `is_location` LowCardinality(String),
    `isads` LowCardinality(String),
    `isfilter` LowCardinality(String),
    `isfull` LowCardinality(String),
    `issproomfilter` LowCardinality(String),
    `issproomorder` LowCardinality(String),
    `issproomshow` LowCardinality(String),
    `keyword` LowCardinality(String),
    `labelid` LowCardinality(String),
    `leave_blank|aword_blank` LowCardinality(String),
    `leave_blank|comm_blank` LowCardinality(String),
    `leave_blank|incentive_blank` LowCardinality(String),
    `leave_blank|line_width` LowCardinality(String),
    `leave_blank|poi_blank` LowCardinality(String),
    `leave_blank|price_blank` LowCardinality(String),
    `leave_blank|pricetag_blank` LowCardinality(String),
    `leave_blank|tag_blank` LowCardinality(String),
    `leave_blank|tax_blank` LowCardinality(String),
    `leave_blank|title_blank` LowCardinality(String),
    `methodid` LowCardinality(String),
    `newtypeid` LowCardinality(String),
    `pagecode` LowCardinality(String),
    `pageindex` LowCardinality(String),
    `place` LowCardinality(String),
    `rank` LowCardinality(String),
    `rankingid` LowCardinality(String),
    `rankVersion` LowCardinality(String),
    `recommendType` LowCardinality(String),
    `regionid` LowCardinality(String),
    `regiontype` LowCardinality(String),
    `roomlist|price` Array(Float64),
    `roomlist|roomid` Array(Float64),
    `searchtype` LowCardinality(String),
    `sessionid` LowCardinality(String),
    `taginfo|clienttaglist|featureID` Array(String),
    `taginfo|clienttaglist|tagfeatureid` Array(String),
    `taginfo|clienttaglist|tagid` Array(String),
    `taginfo|clienttaglist|tagposition` Array(String),
    `taginfo|clienttaglist|tagtitle` Array(String),
    `taginfo|clienttaglist|isshow` Array(UInt8),
    `taginfo|hotelid` Array(String),
    `taginfo|masterhotelid` Array(String),
    `taginfo_up` LowCardinality(String),
    `taxfee|taxfeekey` Array(String),
    `taxfee|taxfeetext` Array(String),
    `tracelogid` LowCardinality(String),
    `type` LowCardinality(String),
    `userPurposeDetails|purpose` LowCardinality(String),
    `userPurposeDetails|score` Float64,
    `visiblePercentage` LowCardinality(String),
    `wordsfeature|featureid` Array(String),
    `wordsfeature|featurename` Array(String),
    `wordsfeature|featuretypeid` Array(String),
    `uid` UInt64,
    `uid_value` String,
    `page_id` LowCardinality(String),
    `app_ver` LowCardinality(String),
    `client_id` LowCardinality(String),
    `os` LowCardinality(String),
    `meta_id` String,
    `vid` String,
    `receive_time` DateTime('Asia/Hong_Kong'),
    `send_time` DateTime('Asia/Hong_Kong'),
    `country` LowCardinality(String),
    `province` LowCardinality(String),
    `city` LowCardinality(String),
    `device_type` LowCardinality(String),
    `app_name` LowCardinality(String),
    `meta_id_num` UInt64,
    `env_buildID` String,
    `gps_country` LowCardinality(String),
    `gps_province` LowCardinality(String),
    `gps_city` LowCardinality(String),
    `d` Date DEFAULT toDate(receive_time, 'Asia/Hong_Kong'),
    `current_time` DateTime DEFAULT now()
)
ENGINE = Distributed('ck100043832', 'ubtprimarychdb', 'htl_c_app_list_tag_show', xxHash64(meta_id))
COMMENT 'create by gao_fei create time is:2023-04-18 17:05:05'
```



```
CREATE TABLE ubtprimarychdb.htl_c_app_list_tag_show
(
    `arrange` LowCardinality(String),
    `ascityid` LowCardinality(String),
    `attributeid` LowCardinality(String),
    `card_below` LowCardinality(String),
    `checkin` LowCardinality(String),
    `checkout` LowCardinality(String),
    `cityid` LowCardinality(String),
    `comment_keywords` LowCardinality(String),
    `comment_keywordtype` LowCardinality(String),
    `countryname` LowCardinality(String),
    `countrytype` LowCardinality(String),
    `currenthotelcount` LowCardinality(String),
    `dispatchid` LowCardinality(String),
    `dist` LowCardinality(String),
    `distanceremark` LowCardinality(String),
    `disttype` LowCardinality(String),
    `fastfilterlist|fastfilterid` Array(String),
    `fastfilterlist|fastfiltername` Array(String),
    `fastfilterlist|fastfilterrank` Array(Float64),
    `fastfilterlist|fastfiltersubtype` Array(String),
    `fastfilterlist|fastfiltertitle` Array(String),
    `fastfilterlist|fastfiltertype` Array(String),
    `fastfilterlist|filterselectstate` Array(Float64),
    `fastfilterlist|filtervalue` Array(String),
    `fastfilterlist|subtitle` Array(String),
    `fh_price` Float64,
    `filterlist|fastfiltername` Array(String),
    `filterlist|fastfiltersubtype` Array(Float64),
    `filterlist|fastfiltertype` Array(Float64),
    `filterlist|filterid` Array(String),
    `filterlist|filtername` Array(String),
    `filterlist|filterposition` Array(Float64),
    `filterlist|filterselectstate` Array(Float64),
    `filterlist|filtersubtype` Array(String),
    `filterlist|filtertitle` Array(String),
    `filterlist|filtertitleCNname` Array(String),
    `filterlist|filtertype` Array(String),
    `filterlist|filtervalue` Array(String),
    `fq_price` Float64,
    `frompage` LowCardinality(String),
    `hotelcardheight` LowCardinality(String),
    `hotelnum` LowCardinality(String),
    `htlist_recomm_type` LowCardinality(String),
    `imageFilter` LowCardinality(String),
    `inputword` LowCardinality(String),
    `is_location` LowCardinality(String),
    `isads` LowCardinality(String),
    `isfilter` LowCardinality(String),
    `isfull` LowCardinality(String),
    `issproomfilter` LowCardinality(String),
    `issproomorder` LowCardinality(String),
    `issproomshow` LowCardinality(String),
    `keyword` LowCardinality(String),
    `labelid` LowCardinality(String),
    `leave_blank|aword_blank` LowCardinality(String),
    `leave_blank|comm_blank` LowCardinality(String),
    `leave_blank|incentive_blank` LowCardinality(String),
    `leave_blank|line_width` LowCardinality(String),
    `leave_blank|poi_blank` LowCardinality(String),
    `leave_blank|price_blank` LowCardinality(String),
    `leave_blank|pricetag_blank` LowCardinality(String),
    `leave_blank|tag_blank` LowCardinality(String),
    `leave_blank|tax_blank` LowCardinality(String),
    `leave_blank|title_blank` LowCardinality(String),
    `methodid` LowCardinality(String),
    `newtypeid` LowCardinality(String),
    `pagecode` LowCardinality(String),
    `pageindex` LowCardinality(String),
    `place` LowCardinality(String),
    `rank` LowCardinality(String),
    `rankingid` LowCardinality(String),
    `rankVersion` LowCardinality(String),
    `recommendType` LowCardinality(String),
    `regionid` LowCardinality(String),
    `regiontype` LowCardinality(String),
    `roomlist|price` Array(Float64),
    `roomlist|roomid` Array(Float64),
    `searchtype` LowCardinality(String),
    `sessionid` LowCardinality(String),
    `taginfo|clienttaglist|featureID` Array(String),
    `taginfo|clienttaglist|tagfeatureid` Array(String),
    `taginfo|clienttaglist|tagid` Array(String),
    `taginfo|clienttaglist|tagposition` Array(String),
    `taginfo|clienttaglist|tagtitle` Array(String),
    `taginfo|clienttaglist|isshow` Array(UInt8),
    `taginfo|hotelid` Array(String),
    `taginfo|masterhotelid` Array(String),
    `taginfo_up` LowCardinality(String),
    `taxfee|taxfeekey` Array(String),
    `taxfee|taxfeetext` Array(String),
    `tracelogid` LowCardinality(String),
    `type` LowCardinality(String),
    `userPurposeDetails|purpose` LowCardinality(String),
    `userPurposeDetails|score` Float64,
    `visiblePercentage` LowCardinality(String),
    `wordsfeature|featureid` Array(String),
    `wordsfeature|featurename` Array(String),
    `wordsfeature|featuretypeid` Array(String),
    `uid` UInt64,
    `uid_value` String,
    `page_id` LowCardinality(String),
    `app_ver` LowCardinality(String),
    `client_id` LowCardinality(String),
    `os` LowCardinality(String),
    `meta_id` String,
    `vid` String,
    `receive_time` DateTime('Asia/Hong_Kong'),
    `send_time` DateTime('Asia/Hong_Kong'),
    `country` LowCardinality(String),
    `province` LowCardinality(String),
    `city` LowCardinality(String),
    `device_type` LowCardinality(String),
    `app_name` LowCardinality(String),
    `meta_id_num` UInt64,
    `env_buildID` String,
    `gps_country` LowCardinality(String),
    `gps_province` LowCardinality(String),
    `gps_city` LowCardinality(String),
    `d` Date DEFAULT toDate(receive_time, 'Asia/Hong_Kong'),
    `current_time` DateTime DEFAULT now()
)
ENGINE = ReplacingMergeTree(current_time)
PARTITION BY (d, os)
ORDER BY (page_id, app_ver, receive_time, meta_id)
TTL d + toIntervalDay(93)
SETTINGS index_granularity = 8192
COMMENT 'create by gao_fei create time is:2023-04-18 17:05:05'
```

这里如果page_id+app_ver+receive_time+meta_id有重复 保留最新一条数据



replacing merge tree数据合并的逻辑是在分区合并的时候 同一分区内的重复数据会被删除

这个分区重复数据删除还是很方便的，因为分区内的数据已经基于order by进行了排序 因此相邻的重复数据是很好找到的

我们设置的是current_time保留最新一条数据

单机同分区去重



其实这里有个问题：

比如你要导入某一天的埋点数据

我想看数据有没有做分片：



我们先查询了2023-08-03这一天10.42.134.19上本地表数量/分布式表数量

```
228285926
```

```
456551499
```

发现一个两亿一个四亿 看起来数据是分了两个分片

SELECT COUNT(*) AS row_count
FROM ubtprimarychdb.htl_c_app_list_tag_show
WHERE d = '2023-08-03';

SELECT COUNT(*) AS row_count
FROM ubtprimarychdb.htl_c_app_list_tag_show_all
WHERE d = '2023-08-03';



然后我们在10.42.136.106上查询：

```
228265573
```

```
456551499
```



本地表时用来写/存数据 按ip写入 而不是分布式表 因为分布式表有分发策略

假如集群有十台机器，分布式表采用rand策略 那就会把数据打散到十台机器

c0文件会特别小而且特别多 后台会做很多次归并排序 对后台性能很大影响

不如就是直接一次写入一个分区 每次归并排序都会对CK产生性能影响 建议直接写本地表 CK的写入qps不高的

维表可以用Dictionary默认放内存方便join



分区不能太多 单次写入一个分区 单次查询设计较少分区 按天

优先使用大宽表减少join 

columns用合适的数据类型 低基数类型 提高压缩比



在千万级别join性能不太好

物化视图做预聚合



用户画像人群包 可用bitmap存储优化



写入频率10qps以内 每批要大 10w 1min一刷这样

如果实在要删就按分区删 不建议删、修改单条数据

如果希望增加写吞吐可以在写入前对数据按排序键预排序  然后单批数据写入不要跨分区

join操作大表在左小表在右（小于千万级）



用户画像例子：

create table ch_label_string{

​	lablename:string,

​	lablevalue:string,

​	uv AggregateFunction(groupBitmap,UInt64)

}

ENGINE=AggregatingMergeTree()

PARTITION BY labelname

ORDER BY(labelname,labelvalue)

SETTINGS index_granularity=128



其实这里就牵扯到一个bitmap的应用了



#### 官网-管理数据

*CREATE* *TABLE* hits_UserID_URL
(
    `UserID` UInt32,
    `URL` String,
    `EventTime` *DateTime*
)
*ENGINE* = MergeTree
*// highlight-next-line*
*PRIMARY* *KEY* (UserID, URL)
*ORDER* *BY* (UserID, URL, EventTime)
SETTINGS index_granularity = 8192, index_granularity_bytes = 0, compress_primary_key = 0;



##### 使用多个主索引

*SELECT* UserID, count(UserID) *AS* Count
*FROM* hits_UserID_URL
*WHERE* URL = 'http://public_search'
*GROUP* *BY* UserID
*ORDER* *BY* Count *DESC*
*LIMIT* 10;

例如对于该sql 其实有点类似mysql联合索引的前缀匹配原则

这里CK只能使用排除搜索算法 并且该算法的有效性取决于URL列与前一个键列UserID之间的基数差异



如果前一个键列的基数较低 那么相同UserID值很可能分布在多个表行和粒度以及索引标记上，对于具有相同UserID的索引标记，其URL值按升序排序

但如果前一个键列的基数较高时，相同的UserID不太可能分布在多个表行和粒度上，所以很多granule都不能被排除，就扫描很多个granule



因此如果我们也想显著加速筛选特定URL行的查询 就需要创建多个主索引

1、创建具有不同主键的第二个表



#### 表引擎

##### mergeTree

当数据插入到表中，会创建单独的数据部件parts，parts是按照primary key排序的，不同分区的数据被分隔到不同部件中，CK后台会不断合并数据部件，每个部件在逻辑上都分成granules粒度，这个粒度是CK在选择数据时最小不可分割的数据集，

CK不需要唯一的primary key 可以插入相同primary key的多行

primary key主键可以是和排序键不同，但必须是排序键的





### Clickhouse部署实践

sudo：Linux中允许普通用户临时以root身份运行命令的机制，不需要知道root密码只需要当前用户即可

此外CK通过RPM包 yum install安装后文件会分布在标准的linux目录中

在官网下ARM包时发现有两种：这其实和CPU架构有关，

- x86_64 对应64位Intel&AMD的处理器 绝大多数Windows PC、服务器
- aarch64 64位ARM处理器 Apple Mac、安卓手机

我们需要下至少两个核心包：

- clickhouse-server
- clickhouse-client
- clickhouse-common-static

介绍一下wget和curl两个命令：



安装server包时提示给default用户设置密码：123456

Password for the default user is saved in file /etc/clickhouse-server/users.d/default-password.xml.



如何在linux上编辑文件--神器vim

- i 插入模式
- esc退出插入模式
- ：wq保存并退出
- q！不保存强制退出



本地连接 ck

首先找出虚拟机的ip：192.168.88.133

此外配置CK允许远程连接



### 为什么CK如此之快

1、数据方向

数据库将数据存储为面向行或面向列，在面向行的DBMS

由于从磁盘到内存的块式存储和传输与分析查询的数据访问模式对齐，因此只有查询所需的列才从磁盘读取。从而避免了对未使用数据的不必要IO

比如说一张宽表有100个字段，你的查询只需要2个字段，那么行式存储必须读取整行，即使你只用2个，剩下98%都浪费了

在大数据场景，磁盘IO是最大瓶颈，减少IO=直接提速

此外同一列的数据类型相同、值相似，可以使用更合适的压缩算法，压缩比很高，从而进一步减少IO和内存占用

****

CK22.8+版本引入了Compact Format格式，对于小数据量的MergeTree表会自动使用Compact格式

即每列独立.bin+.mrk文件 变为 所有列合并到data.bin

如何利用跳表索引skip index去优化非主键列全表扫描的痛点



### OLAP

在线分析处理，可以从技术和业务两个角度来看待

- 从业务上将今年来商业人士开始意识到数据的价值，盲目做决策的公司往往跟不上竞争的步伐，成功公司的数据驱动方法迫使他们收集所有可能对业务决策有用的数据并具备及时分析的机制
- 从技术上看，所有的数据库管理系统其实都可以分为两组：OLAP和OLTP，前者侧重于构建预计大量历史数据的报告，后者通常处理连续的事务流不断修改数据的当前状态



### 二级索引&跳表索引

与其他DB不同的是：CK中的二级索引不指向特定的行或行范围，相反它们使数据库可以预先知道某些数据部分中的所有行都不符合查询过滤条件，并且根本不读取它们，因此它们被称为数据跳过索引（即跳表索引）

对比Mysql二级索引是直接锁定到记录的主键的，而CK是对某个数据块Granule的摘要，只能判断某个Granule是否可能包含目标值

CK常见的二级索引：

- minmax 每个Granule的min/max值
- bloom_filter 每个Granule的布隆过滤器
- tokenbf_v1 字符串分词的布隆过滤器

因此可以针对高频过滤列区创建跳数索引，而主键用于排序+稀疏索引



**Q：为什么CK不做类似Mysql的二级索引**

传统索引的行指针在列存储中没有意义，且CK以Granule为单位处理数据，跳过Granule比定位单行高效

且传统B+数索引写入会严重拖慢写入



### CK数据操作

注意表引擎决定了数据如何&在哪存储，支持哪些查询，数据是否被复制

对于单节点CK服务器的简单表，MergeTree是首选

**主键理解**

CK中的主键对于表中的每一行不是唯一的，主键决定了数据写入磁盘时的排序方式，每8192行（默认granule条数）会在主键索引文件中创建一个条目，这种索引粒度使得其可以轻松容纳在内存中，我们称为稀疏索引，并且granules代表了Select查询期间最小列数据条带

可以通过Primary key去定义主键，如果未指定，则默认为排序键中指定的元组，如果primary key和order by同时指定，则主键必须是排序顺序的前缀

**插入数据**

CK允许每秒插入数百万行，这是通过高度并行化的架构和高效的列式压缩想结合实现的，但在即时一致性上做了妥协，仅提供最终一致性

我们这边其实都是建议大批量插入，理想情况下一般1w-10w行数据，控制写入的qps，最好不要超过5，等数据积攒一批再写入，如果说客户端批处理不可行，可以使用异步插入将批处理委托给clickhouse

**异步插入原理**





**修改与删除数据**



### MergeTree家族引擎

mergeTree引擎是怎样写入&合并数据，不同mergeTree家族的写入与合并表现有何不同

以及如何自动地将数据插入多个表

mergeTree引擎被设计来每秒接收数百万行插入，并存储非常大量的数据，例如数百PB

每次插入会形成一个表部件part，即磁盘上的一个目录其中包含压缩列表文件、mark文件、主索引文件

在后台CK会不断将较小的parts合并成一个较大的parts，合并后原有部件将被删除



mergeTree家族引擎的区别在于它们如何合并数据块

- 普通MergeTree：简单合并-->排序
- replacingMergeTree：CK的数据插入和读取都是极快的，但对大量单行更新的OLTP性能不好，但你可以通过replacingMergeTree来把更新转换为行插入操作，CK很适合干这个事情，把物理更新放在MergeTree部分，具有相同排序键的行会被最新行替换
- AggregatingMergeTree：把数据聚合从查询时间移动到数据插入和合并的时间，这样我们不必在查询时一次又一次地计算聚合，而是在插入和合并时分别计算，在查询时用已经聚合好的数据，在合并时具有相同排序键的所有行替换为包含具有聚合数据类型的所有列的聚合值的单行，列的特定聚合函数数据类型告诉CK作用于哪个列以及是否需要分别存储额外的状态



**简单聚合和非简单聚合**

- 简单聚合：只应用聚合函数即可 例如max
- 非简单聚合：额外存储精确增量聚合所需的一些额外信息&状态 例如AVG



在使用mergeTree引擎的时候要注意部分的合并可能永远不会完全完成，首先CK常用于实时分析场景，其中不断提取新数据，因此不断创建新part

此外CK在合并parts时会在某个时候停止，要么是因为parts变得太大，要么是没有足够的磁盘容量进行另一次合并，而且部分的合并可能永远不会完全完成



这意味着使用replacingmergeTree去替换行也可能不完整，并且使用AggregatingMergeTree引擎聚合行数据

**因此需要在查询时完成该表引擎可能未完成的工作**，这往往只需要消耗查询的一小部分资源



我们在各个表还有未完全合并的数据块的情况下 发起查询：

1、MergeTree基础只是盲目地将所有行从各个数据部件复制到一个新的，在合并期间我们可以使用标准SQL查询从表中获取数据，此查询正在读取并列出所有活动部分的所有行

2、但如果我们用相同的查询对replacingMergeTree时会有问题，因为replacing是不完整的，会包含重复项

为此我们可以在查询的from子句中使用final修饰符，当指定final时CK会在返回结果前在内存中完全合并数据从而执行给定表引擎在常规物理合并期间发生的所有数据转换



**合并与数据分区**

特别注意的是，分区是mergetree引擎中控制part合并范围的核心机制，核心原则是：**Part合并只在同一个分区内进行**



1. 首先分区是天然的数据隔离单元，不同分区数据天然组织在不同磁盘目录中，合并时只需扫描单个分区内数据，可以减少IO和内存压力
2. TTL和删除优化 例如删除整个分区时只需删除目录，无需合并或扫描数据
3. 查询减枝 可以直接在符合范围的分区内查找即可













**场景题:**

 以replacingMergeTree引擎的表为例，我理解一次insert写入Clickhouse会创建一个表部件part，后台是不断得在进部件parts合并的，此时客户端发起了一次查询，可是我的表部件还并没有完全完成，这种时候是否可能出现有相同主键的记录存在于不同的表部件中（此时由于两者还没有完成合并所以replacingMergeTree也没法完成去重），我使用普通select会可能出现这种问题嘛？

**---**使用普通select确实存在这种问题，这是由于ReplacingMergeTree的最终一致性决定的，只要相同主键的记录分散在不同的未合并 `Part` 中（比如刚写入的小 `Part` 还没和旧 `Part` 合并），去重就没完成。

而客户端普通的select执行时，CK会扫所有的Active Part不论是否合并，并将所有Part中的数据直接返回



两种解决方案：

1、OPTIMIZE TABLE t final

手动触发合并，对IO和CPU的消耗很大，会严重影响性能

虽然不影响CK的写入，因为CK的写入是Append-Only的，新的insert会继续生成part

但由于合并操作消耗大量CPU（排序、去重、压缩）IO（读取多个part+写入新part，磁盘IO飙升）而且合并需要内存缓存，optimize final会合并所有的part，期间的写入和查询会很慢，不建议在生产环境使用

**我们进行过压测实验：**

正常状态 写入50w行/s吞吐 查询延迟 100ms CPU load30%

但是执行optimize final 吞吐5w 查询延迟300ms CPU load 95%



2、final修饰符解决

性能不好，不建议频繁查询，本质是查询时模拟后台合并的行为，会读取所有part在查询时进行内存合并

尽量避免final，有限使用argMax等聚合函数去做优化

optimize table hits final这才会真实合并 select+final是不会的

而且要注意的是final修饰符的本质的内存的逻辑合并，不会触发或完成任何后台part合并，只是在查询时的模拟

后台合并线程才是真实做合并的：

- 删除旧的part 创建新part 更新System.parts元数据

对CK来说查询和存储是分离的，查询引擎只负责读取和计算，存储引擎负责合并和优化



3、使用group by+avgmax来解决

即先正常查询后去实时查询





### CK 的类MVCC机制

Q：所有问题还是在于客户端的查询其实无法感知到后台有没有把parts合并完全，但问题是我们在使用mergeTree引擎的时候parts的合并可能永远不会完全完成，首先CK作为实时分析场景，不断有数据写入，因此不断创建新part，此外CK合并parts时在某个时候停止，因为part可能已经变得太大 以上我想说明的是：对于CK来说表部件part的合并可能永远无法完全完成，而同时CK的读写又是不冲突的（我理解没有类似mysql这种mvcc快照读机制），因此CK是怎么处理数据读写可见性的问题的呢？



核心机制：Part-Level MVCC 基于数据片段的多版本并发控制

1. 每个part一旦创建就是不可变的
2. 查询开始时获取当前所有Active part的快照
3. 查询过程中只读取这个快照中的part
4. 新写入的part不会影响正在进行的查询

其实这本质就是MVCC 只不过粒度是part级别而不是mysql的行级别，实现CK读写不冲突



part生命周期：

- Active 正常数据part
- Outdated 被合并替代的part 不可见
- Temporary 正在写入的part 不可见直到写入完成

只有新的写入完成，旧的才会被标记为outdated



理解：part合并永远不会完全完成

1、持续写入 实时场景 或是合并<写入

2、合并策略限制 CK有合并阈值 超大part不会继续合并 避免单词合并耗时过长

3、资源保护 当系统负载高时合并会暂停 因为CK优先保证写入性能和查询稳定性，而不是追求完全合并





### 物化视图与投影

物化视图实际上只对源表中新插入内容做出反应，并且物化视图中的SQL查询仅应用于所有新插入的数据块，以便在将数据插入目标表前重塑数据，CK本质上设计是为了快速且高效地提取和处理数据，如果源表存储了PB级数据，那么从非常大的源表中选择数据将无法扩展

然而对源表的所有插入操作做出反应是可行的，重塑的数据块最终提供给目标表，存储在目标表磁盘目录中

这是一种自动增量数据转换的机制

例如可以做预聚合，把计算更多放在插入和合并阶段而不是查询阶段，这可以加速CK的分析查询，在查询中只需读取几乎最终的结果



而投影无疑是实现自动增量数据转换最现代的方式，和物化视图不用的是投影的底层机制对用户是隐藏的

且用户不必明确使用状态后缀状态来查询，并且所有的查询都发送到源表，CK自动在投影的隐藏目标表查询，这根据优化器决定，但物化视图需要用户显示指定



但投影不能针对性设置DTL，物化视图可以

投影以原子方式与源表一致地更新，因为投影隐藏目标表的数据存储在源表相同part目录



## CK集群相关

### 分片与复制

对CK来说现代服务器很好，在许多情况下CK对硬件的利用率已达到理论极限，向量化指令-->CPU

因此在CK中纵向扩展优先于横向扩展，直到纵向扩展成本超过线性

单个现代CK主机服务器具有足够的CPU内核、足够的内存、大型SSD，可以处理数TB的数据

只有数据量已经超过单服务器的容量时才应考虑横向扩展--分片



由于硬件不可靠需要复制分片数据来保证高可用，新数据可以写入分片的任意副本，其他副本会自动复制更改

，复制实现CK数据完整性和自动故障转移，不至于一台主机挂了或是计划维护丢失数据，重启时会自动同步数据

，为了获取最新版本的数据进行复制需要额外的组件-->Clickhouse Keeper（分布式协调组件）和大名鼎鼎的Zookeeper兼容的





### 分布式表

由于采用share nothing架构，CK的分片方式很低级，例如集群主机直接没有进行协调，并且分片之间完全独立

，因此我们需要创建分布式表，分布式表本身不存储任何数据，但提供单个表接口，来组合访问位于不同主机的远程表，我们可以在集群的一个或多个节点上创建分布式表。



当查询针对分布式表时，会把查询转发到所有主机，等待分片的查询结果，然后计算并返回整体查询结果，可能由于特定的数据分布，也可能转发到一个节点



**写分片表**

1、直接指定节点的远程表写数据，这是最灵活的解决方案，数据可以完全独立地写入不同分片

2、写分布式表，需要设定分片键参数，sharding key，然后分布式表再分发，如果在创建分布式表时没有指定分片键，那么无法通过分布式表直接写入数据，只能通过向各个节点的本地表分别写入

在每个节点都配置一下集群拓扑信息，这些是给分布式表引擎使用的

| 场景                      | 建议                             |
| ------------------------- | -------------------------------- |
| 希望数据分布均匀          | 使用rand()或高基数字段（userid） |
| 希望相同的key落在同一分片 | 使用业务字段                     |
|                           |                                  |



**写入优化**

对比写分布式表与local to local写入

| 对比项   | 分布式写                                   | local to local        |
| -------- | ------------------------------------------ | --------------------- |
| 网络跳数 | 客户端->协调节点->目标节点 两跳            | 客户端到目标节点 一跳 |
| 协调节点 | 所有写流量都经过协调节点，容易成为写入瓶颈 | 无协调节点，负载分散  |
| 写入延迟 | 更高                                       | 更低                  |
| 场景     | 小批量简单写入                             | 高吞吐、生产推荐      |

Local to Local写入核心：客户端负责分片路由，先拉取到CK集群配置，在客户端实现分片算法，然后将数据发送到对应节点的本地表，确保数据分布与分片规则完全一致，此外写入时只需写入一个副本，CK集群会在副本间同步，建议按分片分组数据，每个节点一次批量写入，避免频繁小写，建议是攒够1w条数据写一次

建议封装同一个shardRouter，避免各团队重复实现，然后自动化测试验证，监控集群数据分布，千万不要随便改分片键，扩容需要rehash的

另外如果分片键用rand，其实不能local to local写入了，因为每次rand不同，local to local必须分片要确定









### Clickhouse-keeper

CK集群的搭建还需要设置和配置Clickhouse-keeper集群，提供数据复制的协调系统

建议部署在不同主机上，防止进程之间资源争用，建议至少三台服务器运行keeper

此外Keeper捆绑在clickhouse server的包中 然后还要配置CK节点对于keeper集群节点的可见性



具有keeper相同路径但不同副本名称的表将自动相互复制，当某个分片表获取新数据时，将新条目放入CK路径中的复制日志中，其他副本会去这个路径下拉取日志，发现新的后从源副本拉取新数据



### 存储底层

![image-20251008133820619](D:\编程文档\note-share\工作记录\assets\image-20251008133820619.png)

bin文件是由一个个数据块组成的，数据块分为头文件和压缩数据，头文件记录压缩算法以及压缩前大小&压缩后大小，CK会把64k-1mb的数据作为一个压缩块，



了解到bin文件构造后，就可以去看每个列的mrk文件，mrk文件是为了快速定位到

这里要注意的是Granule是逻辑查询单位，但Compression Block是物理存储单位 mrk文件是二者的桥梁



**理解：为什么CK是压缩数据块而不是每个granule压缩**

压缩效率Compression block默认64-1mb（如果我们批量写 大多是1mb）相比一个granule 8192行 大约几十KB来说，压缩效率更高，

一个granule永远不会跨多个block

以 `SELECT name FROM table1 WHERE id = 657812` 为例：

步骤 1️⃣：主键索引定位 Granule

- `primary.idx` → 确定 `id=657812` 在 **Granule 100**

步骤 2️⃣：.mrk3 文件提供精确定位

- ```
  name.mrk3[100]
  ```

   包含：

  - `offset_in_compressed_file = X` → Compression Block 起始位置
  - `offset_in_decompressed_block = Y` → Granule 100 在解压数据中的偏移

步骤 3️⃣：读取并解压整个 Compression Block

- 从 `name.bin` 的位置 `X` 开始，读取**整个 Compression Block**（约 1MB）
- 解压得到连续数据：`[Granule 95, Granule 96, ..., Granule 105, ...]`

步骤 4️⃣：精确定位到目标 Granule

- 从解压数据的偏移 `Y` 开始，就是 **Granule 100 的起始位置**
- 读取 Granule 100 的 8192 行数据

步骤 5️⃣：在 Granule 内查找目标行

- 由于 Granule 内部**按主键排序**，可以用二分查找快速定位 `id=657812`
- 返回对应的 `name` 值



### 最佳实践

- 空值不使用null 用-1去替换
- 对如用户uid字段如需去重，可用bitmap存储优化，常用在用户画像人群包
- 优先用大宽表 减少join
- 字段用合适的数据类型，尽量低基数类（提高压缩比）
- 排序键：建议按查询频率从前往后排列，尽量用低基数字段作为排序键
- 单次写入一个分区，单次查询涉及较少分区（按天、小时）
- 写入严格控制qps在5以下 每批5-20w，避免出现too many parts和merge slower then insert，写入前建议预排序能有效提高写入吞吐，local to local写入不要走分布式表写，对于分布式hash写入，让相同key同值落到一个shard，加速join和聚合查询，不要建太多物化视图会影响写入吞吐，单次写单分区
- 使用replacingMergeTree等用插入数据的方式去实现数据更新和删除逻辑
- 查询控制频率在400qps以内，定期优化慢查询，Join大表在左小表在右，但建议用物化视图&投影去加速查询，重新组织数据存储，尽量命中排序键和二级索引，不要读整个granule很慢即使是列存储，特别是CK对CPU利用率很高有慢查询很影响整个集群

























# Trino

Trino（原名PrestoSQL）是一款开源的分布式SQL查询引擎，专为OLAP场景设计，最早由Facebook内部开发



核心架构：

client

coordinator 解析SQL、生成执行计划、调度任务

worker 并行执行 内存计算 shuffle数据

connector 对接各种数据源

- share-nothing架构
- 纯内存计算
- MPP模型

一条sql可以跨Hive、Iceberg、mysql等多个数据源JOIN

但长时间大批量ETL任务不如Spark，内存压力大，大JOIN/聚合需要足够内存，否则性能下降明显



**为什么trino查hive会更稳定**

既然分析到瓶颈在hive的hms以及hdfs的namenode 这两个串行的rpc



trino的结构是索引查询走单点coordinator  单点，统一向HMS/NameNode拿元数据，Hive Connector内置元数据缓存，可以缓解查询压力，不会N个并发查询同时爆炸式打HMS

而SR是每个FE节点查询独立去并发拿元数据







# Hive

hive的核心组成：

发起一条sql-->Driver解析sql、生成执行计划--->查表结构Metastore HMS--->生成MapReduce/Spark任务

--->去HDFS上读写数据

默认执行引擎是MapReduce，现在普遍换成Spark

最核心的还是Metastore，本质是一个Mysql/Postgresql + thrift（rpc），记录表A的数据在HDFS哪个目录，有哪些列，按什么字段分区，Hive本身不存数据，数据文件就是普通的HDFS



将结构化数据映射到HDFS文件，提供类SQL的查询接口，让不熟悉MapReduce的用户也能分析大数据

常作用在离线数仓、ETL加工、



存储底座：

- HDFS 大数据存储事实标准、高并发、高容错
- 对象存储 OSS/S3 云原生趋势 存算分离架构核心

表格式层：

数据湖三剑客：

- Iceberg
- Hudi
- Delta

列式存储格式：

- ORC HIVE

  

计算底座：

批处理：

- Spark

流处理：

- Flink
- Spark Streaming



查询引擎：

- Trino/presto 联邦查询，多源统一入口
- Clickhouse 极速列存分析
- Starrocks 国产MPP



消息队列：kafka  标准选择，高吞吐持久化



元数据 Hive Metastore





















# 数仓理论

### SQL执行

首先理解SQL语法的逻辑顺序（写SQL时的语句顺序）和数据库的实际执行顺序（优化器解析后的执行步骤）

1、逻辑顺序

通常写sql的时候：select from where groupby having order by的语法顺序

但数据库例如hive、mysql的优化器会根据执行计划调整实际步骤，目的是减少参与计算的数据量

2、执行顺序

数据库典型的实际执行顺序是：

- FROM 先定位要查询的表
- JOIN 关联其他表
- WHERE 过滤行级数据
- GROUP BY按指定字段分组
- 聚合函数 对分组后的数据计算指标
- HAVING 过滤分组级数据
- SELECT 提取最终需要的字段
- ORDER BY/LIMIT 排序或限制结果行数



### Join

Join操作通过关联条件将两个表中符合条件的记录组合成新纪录，核心逻辑分为“准备”、关联、优化三个阶段

- **驱动表（Driving Table）**：作为关联的 “基准表”，先被扫描，其结果会用来匹配另一张表。通常选择数据量较小的表作为驱动表，减少后续匹配次数。
- **被驱动表（Driven Table）**：被匹配的表，会根据驱动表的结果进行查询和关联。
- **关联条件（Join Condition）**：如`a.id = b.id`，用于判断两表记录是否需要组合。

通常小表作为驱动表



一般根据数据量和底层引擎，JOIN会采用不同的算法，核心有三种：

1、嵌套循环连接

场景：驱动表极小（如几千行之内）被驱动表有索引

逻辑：遍历驱动表，每条记录用关联关系去被驱动表中查询匹配的记录，如果匹配组合成新记录输出

2、哈希连接hash join--大数据场景常用

场景：两表数据量大，且无合适索引

逻辑：遍历驱动表根据关联建计算哈希值，将<key,记录>存储到内存哈希表

遍历被驱动表，对每条记录的关联键计算哈希值，到哈希表中查询相同哈希值记录（若哈希冲突再校验值相等）匹配成功后输出

优势：时间复杂度低 相比嵌套来说从O（N*M）变成O（n+m） 相当于空间换时间

3、排序合并连接

场景：两表数据量大，且关联键已排序

逻辑：将两张表根据关键字排序，同时遍历有点类似归并排序

好处是可以避免哈希表的内存限制



Join的底层执行算法

- Nested Loop Join 嵌套循环连接，小表驱动大表， 外层遍历驱动表，每行都去被驱动表查一次，适用于驱动表小+被驱动表连接键有索引，如果被驱动表无索引那就退化到每次全表扫描
- Hash Join 大表等值连接
- Merge Join 两表已按连接键排序时最优

查询优化器会根据表大小、索引情况自动选择算法



**注意点**

- LEFT JOIN左表是驱动表 
- 连接键加索引 ON条件的字段键索引可大幅加速
- 多表JOIN 超过三张表时注意中间结果集膨胀 考虑拆分查询

如果是inner join是优化器自由选择，一般用过滤后结果集更小的表作为驱动表

如果是left/right join则必须用左/右表作为驱动表，因为语义要求保留左/右表所有行



---

1. Nested Loop Join（嵌套循环连接）

逻辑：外层遍历驱动表，每行去被驱动表查一次。

驱动表: users (小表，1万行)
被驱动表: orders，且 orders.user_id 有索引

FOR each row u IN users:              -- 循环 1万次
    查 orders WHERE user_id = u.id    -- 走索引，每次 O(log n)
    输出匹配行

适用条件： 驱动表小 + 被驱动表连接键有索引
退化情况： 被驱动表无索引 → 每次全表扫描 → O(m×n)，灾难性慢

---

2. Hash Join（哈希连接）

逻辑：对小表建哈希表，大表逐行探测。

阶段1 — Build（构建哈希表）:
  遍历 users（小表），对每行计算 hash(id)，存入内存哈希表
  哈希表: { 1 → {name:"Alice"}, 2 → {name:"Bob"}, ... }

阶段2 — Probe（探测）:
  遍历 orders（大表），对每行计算 hash(user_id)
  去哈希表中查找匹配项，输出结果

  order(id=5001, user_id=2) → hash(2) → 命中 {name:"Bob"} → 输出
  order(id=5002, user_id=9) → hash(9) → 未命中 → 丢弃

适用条件： 大表等值连接 + 内存能放下小表的哈希表
退化情况： 内存不足 → 分批写磁盘（Grace Hash Join），性能下降

---

3. Merge Join（排序合并连接）

逻辑：两表都按连接键排序，然后双指针归并。

前提：两表都按 id/user_id 排好序（或有有序索引）

users:   id=[1, 2, 3, 4, 5...]
orders:  user_id=[1, 1, 2, 3, 3, 3, 5...]

双指针同时推进：
  u.id=1, o.user_id=1 → 匹配，输出；o指针继续
  u.id=1, o.user_id=1 → 匹配，输出；o指针继续
  u.id=1, o.user_id=2 → o>u，u指针前进
  u.id=2, o.user_id=2 → 匹配，输出
  u.id=2, o.user_id=3 → o>u，u指针前进
  ...

适用条件： 连接键上已有排序（如主键、有序索引、上游已排序的子查询）
代价： 若需临时排序，代价是 O(m log m + n log n)，不如 Hash Join



### Corn表达式

常用于定时任务调度，由6个空格分隔的字段组成（部分场景包含第七位年份），从左到右对应

秒 分 时 日 月 星期

例如0 58 11 * * ？

即11:58分0秒 每个月的每天 都调度 其中星期为占位符

因为第四位日和第六位星期是互斥字段，若通过日字段指定了具体日期，则星期字段必须用？占位表示不限制

反之若知道了具体星期则日字段必须用？



日和周不能同时指定有效值，冲突时用？占位







INSERT OVERWRITE TABLE ctripdi_prodb.mid_dna_user_label_initial_8735





INSERT OVERWRITE TABLE ubt_search.cdm_ubt_performance_metrics_table PARTITION (d='${hivevar:dt}')



INSERT OVERWRITE TABLE ubt_search.cdm_ubt_performance_metrics_table PARTITION (d='${hivevar:dt}')









lakehouse stack overview：

自顶向下：

- 应用层
- kyuubi SQL网关 屏蔽底层引擎差异
- spark starrocks trino 计算引擎
- hive、iceberg、hudi 表格式
- hdfs oss/s3 存储层
- datax+mysql/kafka数据接入

从工程上来看hudi的质量以及复杂度不行 慢慢被放弃 更多向icebreg和Paimon上转



湖仓表格式：iceberg、Paimon、hudi 





### OSS、S3

OSS和S3是同一类产品-云对象存储（object storage）只是厂商不同

传统的文件系统用目录树组织文件，例如/data/2024/log.txt 而对象存储的模型更简单：

bucket+key

桶+字符串路径 得到value（二进制文件）+metadata



S3是AWS推出的 Amazon Simple Storage Service，业界事实标准，几乎所有大数据生态都原生支持S3

衍生出了S3协议，现在说兼容S3是指实现了S3的API接口

OSS是阿里云的对象存储产品，国内用得多



### 文件存储和对象存储

从物理介质层面都是磁盘上的0和1，区别在于抽象层的设计哲学

从底层往上拆，起点都是磁盘扇区上的Block块，操作系统管理

操作系统文件系统层在块上建立了一套树形命令空间

- 用inode记录文件元数据
- 用目录项 维护树形结构
- 修改文件需要锁

对象存储直接在块上建一套扁平KV结构

bucket/

- key:  "img/photo.jpg" value: 数据块地址列表+元数据

看起来像目录 其实只是一个带斜杠的key

对象存储一般是不可变的，可以整体替换

在分布式大规模场景在对象存储占优势





文件存储用目录树组织数据，即我们每天用的文件系统

- 有真实的目录层级，ls、cd都是真实操作
- 支持随机读写，可以只修改文件中间某几行
- 访问需要挂载到操作系统



对象存储是用扁平的key-value组织数据，没有真正的目录

- 不支持随机写，只能整体上传覆盖，对象写入后是不可变的
- 通过HTTP REST API访问 无需挂载
- 单个bucket可存数十亿对象，近乎无限扩展
- 存在大量元数据，高度可扩展，海量数据支持

适合大量非结构化数据，图片、视频、日志、数据湖文件



块存储是最底层的，文件存储通常建立在块存储之上，被分成多个block存储到磁盘

对象存储是独立的设计



### 存储后端和表格式

Hive和Iceberg都可以用HDFS 也都可以用S3，存储用什么不是区分它们的核心，真正的区别是Table Format，即数据文件和元数据时怎么组织的



Hive的组织方式：用目录结构表示分区，元数据存在外部的Hive Metastore（一个mysql数据库），查询分区要遍历目录，在S3/OSS上很慢



Iceberg的组织方式：元数据也是文件，和数据文件放一起，每次写入生成新的快照文件，通过快照链实现时间旅行，这和对象存储高度契合



现实中迁移路径往往是HDFS+Hive ---> S3/OSS + Iceberg



**iceberg**

不同于HMS，iceberg把文件元数据也存成文件，放在HDFS/OSS上，和数据一起，无额外服务瓶颈

整体结构：

- catalog 表入口
- metadata file 表级元数据
- manifest list 快照包含哪些manifest file（每个数据文件详细信息）指向data file实际数据ORC

Iceberg把hive依赖外部服务的元数据变成了和数据一起存放的普通文件，用文件的分层索引替代了Hive metastore+namenode的rpc调用



**HDFS**

NameNode：管理元数据 全在内存中

DataNode：实际存储数据块

SR查询Hive高并发瓶颈：

查询路径有两个串行的单点瓶颈

FE节点先查Hive Metastore，获取表分区列表、文件路径等元信息，然后查HDFS的namenode 获取每个文件的Block位置，然后生成执行计划下发给BE节点，BE节点去并发读DataNode

HMS和NameNode RPC 本质都是单进程，N个查询*M个文件产生大量并发RPC，队列堆积响应升到秒级甚至超时，SR拿不到block位置查询失败/超时

HDFS+HMS是为离线批处理设计的，元数据层没有为高并发小查询设计过，和SR的并发理念冲突



### 分区策略

- 全量分区 每个分区存当天的全部数据快照 每天全量覆盖写入 存储开销大 查询简单 不需要合并多个分区 适合DIM维表、数据量不太大
- 增量分区 每个分区只存当天新增或变更的记录 不存没变化的 存储小 但想知道当前状态需要合并所有历史分区 适合ODS 贴近binlog/kafka原始数据 适合数据量大 变化率低
- 快照分区 本质和全量分区一样 也是每个分区存完整数据 但强调不可变性 写入后不再修改 记录某个时间点的业务状态留存 按业务节点做存档 审计  历史定格的场景
- 拉链分区 给每条记录加上start_date和end_date 记录在哪段时间内有效 用一张表存所有历史变化，存储效率最高，实现复杂适合变化不频繁的数据



假设有三天的数据变化：
1月1日：用户A注册，城市=北京
1月2日：用户B注册，用户A改城市为上海
1月3日：用户C注册，用户B注销

全量分区

每个分区存当天的全部数据快照，每天全量覆盖写入。

dt=2024-01-01
  user_id | city  | status
  A       | 北京  | 正常

dt=2024-01-02
  user_id | city  | status
  A       | 上海  | 正常   ← A的城市已更新
  B       | 广州  | 正常

dt=2024-01-03
  user_id | city  | status
  A       | 上海  | 正常
  B       | 广州  | 注销   ← B的状态已更新
  C       | 成都  | 正常



增量分区

每个分区只存当天新增或变更的记录，不存没变化的。

dt=2024-01-01
  user_id | city  | status | op
  A       | 北京  | 正常   | INSERT

dt=2024-01-02
  user_id | city  | status | op
  A       | 上海  | 正常   | UPDATE  ← 只记录变了的
  B       | 广州  | 正常   | INSERT

dt=2024-01-03
  user_id | city  | status | op
  B       | 广州  | 注销   | UPDATE
  C       | 成都  | 正常   | INSERT



拉链表 dwd_user_zip
  user_id | city  | status | start_date | end_date
  A       | 北京  | 正常   | 2024-01-01 | 2024-01-01  ← A在1月1日是北京
  A       | 上海  | 正常   | 2024-01-02 | 9999-12-31  ← A从1月2日起是上海（当前有效）
  B       | 广州  | 正常   | 2024-01-02 | 2024-01-02
  B       | 广州  | 注销   | 2024-01-03 | 9999-12-31
  C       | 成都  | 正常   | 2024-01-03 | 9999-12-31

end_date = 9999-12-31 表示这条记录当前有效

查询技巧：
-- 查当前状态（等同于全量）
WHERE end_date = '9999-12-31'

-- 查1月1日那天A在哪个城市（历史回溯）
WHERE user_id = 'A'
  AND start_date <= '2024-01-01'
  AND end_date >= '2024-01-01'
-- 结果：北京





### 数据仓库-分层

分层的核心目的是：让数据加工过程清晰、可复用、可维护，每层只做一件事下游直接用上游结果，不重复造轮子



ODS-镜像层  原始数据的镜像 与源系统数据结构保持一致

eg：

 ods_orders_di（订单表，每日增量）                                                                                                                                         
    order_id | user_id | sku_id | amount | status | create_time | ...                                                                                                       
    100001   | 8888    | A001   | 299.0  | paid   | 2024-01-01  | ...   





DIM-维表层 描述业务实体属性的参考数据

dim_user（用户维表）
  user_id | name | age | city | register_date | vip_level | ...

dim_product（商品维表）
  sku_id | name | category | brand | price | ...

dim_date（日期维表）
  date | year | month | week | is_holiday | ...



EDW-明细层 

在ODS基础上做清洗和标准化，但仍然保留明细粒度，不做聚合

- 只是处理ODS的脏数据、去重、补空值、统一格式
- 关联DIM层把ID翻译成有意义的属性
- 是整个数仓可信数据的起点

edw_orders_di（清洗后的订单明细）
  order_id | user_id | user_city | sku_id | category | amount
           | 去掉测试账号 | 异常金额过滤 | 统一时区 | jo

ODS 里 amount = -1 的异常记录在这里被过滤掉，sku_id 在这里被关联



CDM-轻度汇总层

按业务主题对明细数据做轻度聚合

- 按某个维度汇总 但不是最终指标 保留足够的灵活性
- 沉淀公共逻辑，避免重复计算
- 多个下游应用都要用的中间结构放在这层

cdm_user_order_1d（用户日粒度订单汇总）
  dt | user_id | order_cnt | pay_amount | category | ...
  2024-01-01 | 8888 | 3 | 599.0 | 手机 | ...

今天用户在手机品类下单了几次、花了多少钱，这个结果既可以给 BI 报表



ADM-应用数据层

面向具体业务需求的最终结果表

- 高度定制化
- 通常是宽表

例子：
adm_daily_sales_report（每日销售报表）
  dt | category | region | gmv | order_cnt | avg_price | yoy | mom | ...
  -- 这张表直接对接 BI 看板，字段是报表需要什么就放什么

adm_user_portrait（用户画像宽表）
  user_id | 30日消费 | 最近购买品类 | 活跃度 | vip等级 | 推荐标签 | ...
  -- 直接给推荐



MID-中间层

- 特点：

  - 通常是某个复杂计算的中间步骤，拆分出来提高可维护性
  - 只被少数下游依赖，不作为公共资产推广
  - 生命周期短，逻辑稳定后可能会被合并掉

  例子：
  mid_order_with_refund（关联了退款信息的订单中间表）
    -- 因为退款逻辑很复杂，单独抽出来，
    -- 下游 cdm/adm 的多张表都要用这



整体数据流

MySQL/Kafka
    │
    ▼
  ODS（镜像，原始数据）
    │
    ├──► DIM（维表，静态属性）
    │         │
    ▼         │
  EDW（清洗明细）◄──┘
    │
    ├──► MID（复杂中间结果）
    │         │
    ▼         ▼
  CDM（轻度汇总，公共复用）
    │
    ▼
  ADM（应用宽表，直接消费）
    │
    ▼
  BI报表 / 推荐系统 / 标签平台



数据湖：存储企业所有原始数据的系统或存储 包含结构化和非结构化数据 其实算ODS 

数据中台：连接数据源和数据消费者，简化企业内部数据共享流程，

数据仓库：支持业务智能BI的系统

数据集市：按业务分主题，快速为特定用户提供所需信息，通常为一个特定部门如营销、财务创建，包含该部门所需全部数据









