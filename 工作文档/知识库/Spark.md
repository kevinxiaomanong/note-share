### 一、概述

Spark是基于内存的大数据分析计算引擎



**对比Hadoop**

Hadoop是用java编写的，在分布式服务器集群存储海量数据并运行分布式分析应用的开源框架



Spark是由Scala语言开发的，由Spark Core提供Spark最基础与核心的功能

Spark Sql是Spark操作结构化数据的组件，我们可以用sql或hive的sql方言查询

然后由Spark Streaming做流式计算



Spark的核心技术是弹性分布式数据集RDD，两者的根本差异是多个作业之间的数据通信问题，Spark多个作业之间数据通信基于内存，而Hadoop基于磁盘



Spark在绝大多数的大数据计算场景中确实会比MapReduce会有优势，但是Spark是基于内存的，在实际生产过程中可能由于内存资源不够导致Job执行失败



**模块**

![image-20240202162126451](D:\工作文档\知识库\assets\image-20240202162126451.png)

其余都是在Spark Core基础上扩展