Hadoop名词解释
---
* Common: 一系列组件和接口，用于分布式文件系统和通用I/O(序列化、java RPC和持久化数据结构)
* Avro: 一种序列化系统，支持高效、跨语言的RPC和持久化数据存储
* MapReduce: 分布式数据处理模型和执行环境，运行于大型商用机集群
* HDFS:分布式文件系统，运行于大型商用机集群
* Pig:数据流语音和运行环境，用于探究非常庞大的数据集。pig运行在MapReduce和HDFS集群上
* Hive: 一种分布式的，按列存储的数据仓库。Hive管理HDFS中存储的数据。并提供基于SQL的查询语言(由运行引擎翻译成MapReduce作业)用以查询数据。
* HBase：一种分布式的，按列存储的数据库。HBase使用HDFS作为底层存储，同时支持MapReduce的批量计算和点查询(随机读取)
* ZooKeeper:一种分布式的高可用的协调服务。ZooKeeper提供分布式锁之类 的基本服务用于构建分布式应用
* Sqoop:该工具用于在结构化数据存储(如关系数据库)和HDFS之间高效批量传输数据
* Oozie:该服务用于运行和调度Hadoop作业(如MapReduce，Pig，Hive，及Sqoop作业)