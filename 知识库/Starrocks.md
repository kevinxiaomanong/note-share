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

与很多数据分析系统采用的Scatter-Gather分布式执行框架不同，MPP分布式执行框架可以利用更多的资源处理查询请求，在Scatter-Gather框架中只有Gather节点能处理最后一级的汇总计算，而在MPP框架中数据会被Shuffle到多个节点并行汇总，在复杂计算时（高基数的group by、大表join）有明显性能优势

**向量化执行引擎**

starrock全面实现向量化引擎，充分发挥CPU的处理能力，能够充分利用CPU的SIMD质量，用更少的指令数目完成更多的数据操作

**存算分离**

存储与计算解耦，各自独立服务扩缩容，解决了在存算一体模式下的计算与存储等比例扩缩容所带来的资源浪费问题

**CBO优化器**

在多表关联查询场景下，不同执行计划的复杂度可能差几个数量级，从众多的可能中选择一个最优的计划是一个NP-Hard问题

starrocks从零实现了基于代价的CBO优化器，cost based optimizer，SR会比同类产品更好地支持多表关联查询

**可实时更新的列式存储引擎**

数据按列的方式进行存储，相同类型的数据连续存放，可以用更高效的编码方式获得更高的压缩比，且大部分OLAP场景查询只涉及部分列，列存只需要读取部分列数据，极大降低磁盘IO吞吐

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

















