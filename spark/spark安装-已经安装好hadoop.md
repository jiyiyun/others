spark安装(已经安装好hadoop)
===
spark官网介绍：

Spark™: A fast and general compute engine for Hadoop data. Spark provides a simple and expressive programming model that supports a wide range of applications, including ETL, machine learning, stream processing, and graph computation.

配置环境变量(跟hadoop类似，这样可以直接在shell命令行中使用spark命令)只在master上设置
---

```txt
vim .bashrc                        #可以写在当前用户的.bashrc中，生效的范围只是当前用户，写在/etc/profile生效的范围是所有用户
export SPARK_HOME=/usr/local/spark
export PATH=$PATH:$SPARK_HOME/bin:$SPARK_HOME/sbin
```
Spark配置
===

在Master节点主机上进行如下操作：
---

配置slaves文件

将 slaves.template 拷贝到 slaves

```txt
[hadoop@master local]$ cd /usr/local/spark/conf
[hadoop@master conf]$ cp slaves.template slaves
[hadoop@master conf]$ vi slaves     #跟hadoop中/etc/hadoop/slaves文件类似

slave1
slave2
```
配置spark-env.sh文件
---

将 spark-env.sh.template 拷贝到 spark-env.sh 编辑spark-env.sh,添加如下内容：

```txt
export SPARK_DIST_CLASSPATH=$(/usr/local/hadoop/bin/hadoop classpath)
export HADOOP_CONF_DIR=/usr/local/hadoop/etc/hadoop
export SPARK_MASTER_IP=192.168.100.63

SPARK_MASTER_IP 指定 Spark 集群 Master 节点的 IP 地址；
```

配置好后，将Master主机上的/usr/local/spark文件夹复制到各个节点上。在Master主机上执行如下命令(和hadoop类似)
---

```txt
[hadoop@master ~]$ cd /usr/local/
[hadoop@master local]$ sudo tar -cvf spark.tar spark/
[hadoop@master local]$ scp spark.tar slave1:/home/hadoop
spark.tar                                                          100%  137MB 137.2MB/s   00:01    
[hadoop@master local]$ scp spark.tar slave2:/home/hadoop
spark.tar                                                          100%  137MB 137.2MB/s   00:01    
[hadoop@master local]$ 

在slave1节点上
[hadoop@slave1 ~]$ tar -xvf spark.tar
[hadoop@slave1 ~]$ sudo mv spark /usr/local/

在slave2节点上
[hadoop@slave2 ~]$ tar -xf spark.tar 
[hadoop@slave2 ~]$ sudo mv spark /usr/local/
[sudo] password for hadoop: 
[hadoop@slave2 ~]$ 
```
启动Spark集群前，要先启动Hadoop集群。在Master节点主机上运行如下命令：
---

```txt
启动Hadoop集群

[hadoop@master ~]$ cd /usr/local/hadoop

[hadoop@master hadoop]$ ls
bin  etc  include  lib  libexec  LICENSE.txt  logs  NOTICE.txt  README.txt  sbin  share  tmp

[hadoop@master hadoop]$ sbin/start-all.sh 
This script is Deprecated. Instead use start-dfs.sh and start-yarn.sh
Starting namenodes on [master]
master: starting namenode, logging to /usr/local/hadoop/logs/hadoop-hadoop-namenode-master.out
slave1: starting datanode, logging to /usr/local/hadoop/logs/hadoop-hadoop-datanode-slave1.out
slave2: starting datanode, logging to /usr/local/hadoop/logs/hadoop-hadoop-datanode-slave2.out
Starting secondary namenodes [master]
master: starting secondarynamenode, logging to /usr/local/hadoop/logs/hadoop-hadoop-secondarynamenode-master.out
starting yarn daemons
starting resourcemanager, logging to /usr/local/hadoop/logs/yarn-hadoop-resourcemanager-master.out
slave1: starting nodemanager, logging to /usr/local/hadoop/logs/yarn-hadoop-nodemanager-slave1.out
slave2: starting nodemanager, logging to /usr/local/hadoop/logs/yarn-hadoop-nodemanager-slave2.out
[hadoop@master hadoop]$ 
```
启动Spark集群
---

```txt
启动Master节点

[hadoop@master spark]$ ls
bin  conf  data  examples  jars  LICENSE  licenses  NOTICE  python  R  README.md  RELEASE  sbin  yarn

[hadoop@master spark]$ sbin/start-master.sh 
starting org.apache.spark.deploy.master.Master, logging to /usr/local/spark/logs/spark-hadoop-org.apache.spark.deploy.master.Master-1-master.out

[hadoop@master spark]$ jps   #使用jps命令查看多个master进程
2320 NameNode
2662 ResourceManager
2952 Master
3004 Jps
2510 SecondaryNameNode
[hadoop@master spark]$ 

启动所有Slave节点(在master主机上执行命令)
[hadoop@master spark]$ sbin/start-slaves.sh
slave2: starting org.apache.spark.deploy.worker.Worker, logging to /usr/local/spark/logs/spark-hadoop-org.apache.spark.deploy.worker.Worker-1-slave2.out
slave1: starting org.apache.spark.deploy.worker.Worker, logging to /usr/local/spark/logs/spark-hadoop-org.apache.spark.deploy.worker.Worker-1-slave1.out
[hadoop@master spark]$ 

分别在slave0、slave2节点上运行jps命令，可以看到多了个Worker进程
[hadoop@slave1 ~]$ jps
2278 NodeManager
2486 Jps
2167 DataNode
2439 Worker
[hadoop@slave1 ~]$

[hadoop@slave2 ~]$ jps
2272 NodeManager
2480 Jps
2161 DataNode
2433 Worker
[hadoop@slave2 ~]$
```
在浏览器上查看Spark独立集群管理器的集群信息,在master主机上打开浏览器，访问http://master:8080
---

```txt
http://192.168.100.63:8080/
Spark Master at spark://master:7077 
URL: spark://master:7077
REST URL: spark://master:6066  (cluster mode) 
Alive Workers: 2
Cores in use: 2 Total, 0 Used
Memory in use: 2.0 GB Total, 0.0 B Used
Applications: 0 Running, 0 Completed 
Drivers: 0 Running, 0 Completed 
Status: ALIVE

Workers 
Worker Id                                  Address             State Cores      Memory
worker-20170216213004-192.168.14.65-50715  192.168.14.65:50715 ALIVE 1 (0 Used) 1024.0 MB (0.0 B Used)  
worker-20170216213005-192.168.14.64-37075  192.168.14.64:37075 ALIVE 1 (0 Used) 1024.0 MB (0.0 B Used)  

Running Applications
Application ID  Name Cores  Memory per Node  Submitted Time  User  State  Duration
Completed Applications
Application ID  Name Cores  Memory per Node  Submitted Time  User  State  Duration
```
关闭Spark集群 (先关spark 后关hadoop 跟开启spark目录一样，顺序相反)
---

```txt
关闭Master节点

[hadoop@master spark]$ sbin/stop-master.sh 
stopping org.apache.spark.deploy.master.Master
[hadoop@master spark]$

关闭Worker节点

[hadoop@master spark]$ sbin/stop-slaves.sh 
slave2: stopping org.apache.spark.deploy.worker.Worker
slave1: stopping org.apache.spark.deploy.worker.Worker

关闭Hadoop集群，在/usr/local/hadoop 目录中

[hadoop@master hadoop]$ sbin/stop-all.sh 
This script is Deprecated. Instead use stop-dfs.sh and stop-yarn.sh
Stopping namenodes on [master]
master: stopping namenode
slave2: stopping datanode
slave1: stopping datanode
Stopping secondary namenodes [master]
master: stopping secondarynamenode
stopping yarn daemons
stopping resourcemanager
slave2: stopping nodemanager
slave1: stopping nodemanager
no proxyserver to stop
[hadoop@master hadoop]$
```
参考资料
---
- Spark 2.0分布式集群环境搭建 http://dblab.xmu.edu.cn/blog/1187-2/

