准备工作
===

1、安装java jdk 设置好JAVA_HOME

```txt
export JAVA_HOME=/usr/java/jdk1.8.0_121
export JRE_HOME=$JAVA_HOME/jre
export CLASSPATH=.:$JAVA_HOME/lib/dt.jar:$JAVA_HOME/lib/tools.jar:$JAVA_HOME/lib:$CLASSPATH
export PATH=$PATH:$JAVA_HOME/bin:$JRE_HOME/bin
```
2、设置好master节点到所有节点的ssh 无密码登录，包括自己并把自己的id.rsa.pub写到authorized_keys中，修改每台主机的~/.ssh/authorized_keys权限为600

3、修改每台主机的/etc/hosts 文件

格式IP    主机名

```txt
192.168.100.63  master
192.168.100.64  slave1
192.168.100.65  slave
```
4、配置PATH变量

```txt
[hadoop@master2 local]$ vi .bashrc
export PATH=$PATH:/usr/local/hadoop/bin:/usr/local/hadoop/sbin
export HADOOP_COMMON_LIB_NATIVE_DIR=$HADOOP_HOME/lib/native
export PATH=$PATH:/usr/local/hadoop/bin:/usr/local/hadoop/sbin
```
5、把JAVA_HOEME添加到hadoop-env.sh文件中

```TXT
修改hadoop-env.sh文件vi hadoop-env.sh
把JAVA_HOME添加进去
export JAVA_HOME=/usr/java/jdk1.8.0_121
```
开始配置hadoop 配置文件都在/usr/local/hadoop/etc/hadoop文件夹
===

```txt
[hadoop@master ~]$ cd /usr/local/hadoop/etc/hadoop
[hadoop@master hadoop]$ ls
capacity-scheduler.xml  hadoop-env.cmd hadoop-policy.xml httpfs-signature.secret  kms-log4j.properties  mapred-env.sh slaves yarn-env.sh configuration.xsl hadoop-env.sh hdfs-site.xml httpfs-site.xml kms-site.xml mapred-queues.xml.template  ssl-client.xml.example  yarn-site.xml container-executor.cfg  hadoop-metrics2.properties  httpfs-env.sh kms-acls.xml log4j.properties mapred-site.xml ssl-server.xml.example core-site.xml hadoop-metrics.properties   httpfs-log4j.properties  kms-env.sh mapred-env.cmd mapred-site.xml.template  yarn-env.cmd
```
1、文件 slaves
---

```txt
[hadoop@master hadoop]$ vi slaves
slave1
slave2
```
2、文件 core-site.xml 
---

```txt
<configuration>
        <property>
                <name>fs.defaultFS</name>
                <value>hdfs://Master:9000</value>
        </property>
        <property>
                <name>hadoop.tmp.dir</name>
                <value>file:/usr/local/hadoop/tmp</value>
                <description>Abase for other temporary directories.</description>
        </property>
</configuration>
```
3、文件 hdfs-site.xml，dfs.replication 一般设为 3，但我们只有2个 Slave 节点，所以 dfs.replication 的值还是设为2
---

```txt
<configuration>
        <property>
                <name>dfs.namenode.secondary.http-address</name>
                <value>hadoop:50090</value>
        </property>
        <property>
                <name>dfs.replication</name>
                <value>2</value>
        </property>
        <property>
                <name>dfs.namenode.name.dir</name>
                <value>file:/usr/local/hadoop/tmp/dfs/name</value>
        </property>
        <property>
                <name>dfs.datanode.data.dir</name>
                <value>file:/usr/local/hadoop/tmp/dfs/data</value>
        </property>
</configuration>
```
4、 文件 mapred-site.xml （可能需要先重命名，默认文件名为 mapred-site.xml.template），然后配置修改如下：
---

```txt
<configuration>
        <property>
                <name>mapreduce.framework.name</name>
                <value>yarn</value>
        </property>
        <property>
                <name>mapreduce.jobhistory.address</name>
                <value>Master:10020</value>
        </property>
        <property>
                <name>mapreduce.jobhistory.webapp.address</name>
                <value>Master:19888</value>
        </property>
</configuration>
```
5、 文件 yarn-site.xml
---

```txt
<configuration>
        <property>
                <name>yarn.resourcemanager.hostname</name>
                <value>Master</value>
        </property>
        <property>
                <name>yarn.nodemanager.aux-services</name>
                <value>mapreduce_shuffle</value>
        </property>
</configuration>
```
配置好后，将 Master 上的 /usr/local/Hadoop 文件夹复制到各个节点上

首次启动需要先在 Master 节点执行 NameNode 的格式化：
---

```txt
hdfs namenode -format       # 首次运行需要执行初始化，之后不需要
```
接着可以启动 hadoop 了，启动需要在 Master 节点上进行：
---

```txt
start-dfs.sh
start-yarn.sh
mr-jobhistory-daemon.sh start historyserver
```
```txt
[hadoop@master ~]$ start-dfs.sh 
Starting namenodes on [master]
master: starting namenode, logging to /usr/local/hadoop/logs/hadoop-hadoop-namenode-master.out
slave2: datanode running as process 2182. Stop it first.
slave1: datanode running as process 2456. Stop it first.
Starting secondary namenodes [master]
master: starting secondarynamenode, logging to /usr/local/hadoop/logs/hadoop-hadoop-secondarynamenode-master.out
[hadoop@master ~]$ start-yarn.sh 
starting yarn daemons
starting resourcemanager, logging to /usr/local/hadoop/logs/yarn-hadoop-resourcemanager-master.out
slave2: starting nodemanager, logging to /usr/local/hadoop/logs/yarn-hadoop-nodemanager-slave2.out
slave1: starting nodemanager, logging to /usr/local/hadoop/logs/yarn-hadoop-nodemanager-slave1.out
[hadoop@master ~]$ mr-jobhistory-daemon.sh start historyserver
starting historyserver, logging to /usr/local/hadoop/logs/mapred-hadoop-historyserver-master.out
```

```txt
使用jps命令查看
[hadoop@master2 ~]$ jps
14777 Jps
14285 ResourceManager
14637 JobHistoryServer

[hadoop@node1-hadoop1 ~]$ jps
9457 NodeManager
9570 Jps
9320 DataNode

[hadoop@master ~]$ hdfs dfsadmin -report
Configured Capacity: 58951507968 (54.90 GB)
Present Capacity: 53208956928 (49.55 GB)
DFS Remaining: 53208948736 (49.55 GB)
DFS Used: 8192 (8 KB)
DFS Used%: 0.00%
Under replicated blocks: 0
Blocks with corrupt replicas: 0
Missing blocks: 0
Missing blocks (with replication factor 1): 0

-------------------------------------------------
Live datanodes (2):

Name: 192.168.100.65:50010 (slave2)
Hostname: slave2
Decommission Status : Normal
Configured Capacity: 29475753984 (27.45 GB)
DFS Used: 4096 (4 KB)
Non DFS Used: 2871791616 (2.67 GB)
DFS Remaining: 26603958272 (24.78 GB)
DFS Used%: 0.00%
DFS Remaining%: 90.26%
Configured Cache Capacity: 0 (0 B)
Cache Used: 0 (0 B)
Cache Remaining: 0 (0 B)
Cache Used%: 100.00%
Cache Remaining%: 0.00%
Xceivers: 1
Last contact: Thu Feb 16 15:16:44 CST 2017

Name: 192.168.100.64:50010 (slave1)
Hostname: slave1
Decommission Status : Normal
Configured Capacity: 29475753984 (27.45 GB)
DFS Used: 4096 (4 KB)
Non DFS Used: 2870759424 (2.67 GB)
DFS Remaining: 26604990464 (24.78 GB)
DFS Used%: 0.00%
DFS Remaining%: 90.26%
Configured Cache Capacity: 0 (0 B)
Cache Used: 0 (0 B)
Cache Remaining: 0 (0 B)
Cache Used%: 100.00%
Cache Remaining%: 0.00%
Xceivers: 1
Last contact: Thu Feb 16 15:16:44 CST 2017

[hadoop@master ~]$ 
```
也可以通过 Web 页面看到查看 DataNode 和 NameNode 的状态：http://master:50070 如果不成功，可以通过启动日志排查原因。

```txt
http://192.168.100.63:50070
Started:Thu Feb 16 15:12:21 CST 2017 
Version:2.7.3, rbaa91f7c6bc9cb92be5982de4719c1c8af91ccff 
Compiled:2016-08-18T01:41Z by root from branch-2.7.3 
Cluster ID:CID-36045a0d-2d4b-4d85-b2e1-0378b50c782e 
Block Pool ID:BP-1176183291-192.168.14.63-1487228449561

Datanode Information
In operation
Node      Last State  Capacity   Used NonUsed RemaBlocks Blockused Failed Version
slave2:50010 0 InService 27.45GB 8 KB 2.67 GB 24.78 GB 0 8 KB (0%) 0      2.7.3 
slave1:50010 0 InService 27.45GB 8 KB 2.67 GB 24.78 GB 0 8 KB (0%) 0      2.7.3
```
对已经初始化过的主机，清空配置
===

```txt
cd /usr/local
sudo rm -r ./hadoop/tmp     # 删除 Hadoop 临时文件
sudo rm -r ./hadoop/logs/*   # 删除日志文件
tar -zcf ~/hadoop.master.tar.gz ./hadoop   # 先压缩再复制
cd ~
scp ./hadoop.master.tar.gz Slave1:/home/hadoop
```

```txt
sudo rm -r /usr/local/hadoop    # 删掉旧的（如果存在）
sudo tar -zxf ~/hadoop.master.tar.gz -C /usr/local
sudo chown -R hadoop /usr/local/hadoop
```
执行分布式实例
===

1、执行分布式实例过程与伪分布式模式一样，首先创建 HDFS 上的用户目录

```txt
hdfs dfs -mkdir -p /user/hadoop
```
将 /usr/local/hadoop/etc/hadoop 中的配置文件作为输入文件复制到分布式文件系统中：

```txt
[hadoop@master hadoop]$ hdfs dfs -mkdir -p /user/hadoop
[hadoop@master hadoop]$ hdfs dfs -mkdir input
[hadoop@master hadoop]$ hdfs dfs -put /usr/local/hadoop/etc/hadoop/*.xml input
```
通过查看 DataNode 的状态（占用大小有改变），输入文件确实复制到了 DataNode 中

```txt
http://192.168.100.63:50070
Datanode Information
In operation
Node      Last State  Capacity   Used          NonUsed RemaBlocks Blockused Failed Version
slave2:50010 1 InService 27.45GB 35.97 KB (0%) 2.67 GB 24.78 GB 9 35.97 KB (0%) 0  2.7.3 
slave1:50010 1 InService 27.45GB 35.97 KB (0%) 2.67 GB 24.78 GB 9 35.97 KB (0%) 0  2.7.3 
```
接着就可以运行 MapReduce 作业了

```txt
[hadoop@master hadoop]$ hadoop jar /usr/local/hadoop/share/hadoop/mapreduce/hadoop-mapreduce-examples-2.7.3.jar grep input output 'dfs[a-z.]+'
17/02/16 17:02:23 INFO client.RMProxy: Connecting to ResourceManager at master/192.168.100.63:8032
17/02/16 17:02:24 INFO input.FileInputFormat: Total input paths to process : 9
17/02/16 17:02:24 INFO mapreduce.JobSubmitter: number of splits:9
17/02/16 17:02:25 INFO mapreduce.JobSubmitter: Submitting tokens for job: job_1487229244291_0001
17/02/16 17:02:25 INFO impl.YarnClientImpl: Submitted application application_1487229244291_0001
17/02/16 17:02:25 INFO mapreduce.Job: The url to track the job: http://master:8088/proxy/application_1487229244291_0001/
17/02/16 17:02:25 INFO mapreduce.Job: Running job: job_1487229244291_0001
17/02/16 17:02:35 INFO mapreduce.Job: Job job_1487229244291_0001 running in uber mode : false
17/02/16 17:02:35 INFO mapreduce.Job:  map 0% reduce 0%
17/02/16 17:02:42 INFO mapreduce.Job:  map 11% reduce 0%
17/02/16 17:02:54 INFO mapreduce.Job:  map 11% reduce 4%
17/02/16 17:02:57 INFO mapreduce.Job:  map 22% reduce 4%
17/02/16 17:03:00 INFO mapreduce.Job:  map 22% reduce 7%
17/02/16 17:03:10 INFO mapreduce.Job:  map 100% reduce 7%
17/02/16 17:03:12 INFO mapreduce.Job:  map 100% reduce 100%
17/02/16 17:03:12 INFO mapreduce.Job: Job job_1487229244291_0001 completed successfully
17/02/16 17:03:12 INFO mapreduce.Job: Counters: 50
	File System Counters
		FILE: Number of bytes read=153
		FILE: Number of bytes written=1190369
		FILE: Number of read operations=0
		FILE: Number of large read operations=0
		FILE: Number of write operations=0
		HDFS: Number of bytes read=29393
		HDFS: Number of bytes written=263
		HDFS: Number of read operations=30
		HDFS: Number of large read operations=0
		HDFS: Number of write operations=2
	Job Counters 
		Killed map tasks=2
		Launched map tasks=11
		Launched reduce tasks=1
		Data-local map tasks=11
		Total time spent by all maps in occupied slots (ms)=249225
		Total time spent by all reduces in occupied slots (ms)=27011
		Total time spent by all map tasks (ms)=249225
		Total time spent by all reduce tasks (ms)=27011
		Total vcore-milliseconds taken by all map tasks=249225
		Total vcore-milliseconds taken by all reduce tasks=27011
		Total megabyte-milliseconds taken by all map tasks=255206400
		Total megabyte-milliseconds taken by all reduce tasks=27659264
	Map-Reduce Framework
		Map input records=807
		Map output records=5
		Map output bytes=137
		Map output materialized bytes=201
		Input split bytes=1050
		Combine input records=5
		Combine output records=5
		Reduce input groups=5
		Reduce shuffle bytes=201
		Reduce input records=5
		Reduce output records=5
		Spilled Records=10
		Shuffled Maps =9
		Failed Shuffles=0
		Merged Map outputs=9
		GC time elapsed (ms)=2174
		CPU time spent (ms)=4440
		Physical memory (bytes) snapshot=1901674496
		Virtual memory (bytes) snapshot=20760567808
		Total committed heap usage (bytes)=1248497664
	Shuffle Errors
		BAD_ID=0
		CONNECTION=0
		IO_ERROR=0
		WRONG_LENGTH=0
		WRONG_MAP=0
		WRONG_REDUCE=0
	File Input Format Counters 
		Bytes Read=28343
	File Output Format Counters 
		Bytes Written=263
17/02/16 17:03:12 INFO client.RMProxy: Connecting to ResourceManager at master/192.168.100.63:8032
17/02/16 17:03:13 INFO input.FileInputFormat: Total input paths to process : 1
17/02/16 17:03:13 INFO mapreduce.JobSubmitter: number of splits:1
17/02/16 17:03:13 INFO mapreduce.JobSubmitter: Submitting tokens for job: job_1487229244291_0002
17/02/16 17:03:13 INFO impl.YarnClientImpl: Submitted application application_1487229244291_0002
17/02/16 17:03:13 INFO mapreduce.Job: The url to track the job: http://master:8088/proxy/application_1487229244291_0002/
17/02/16 17:03:13 INFO mapreduce.Job: Running job: job_1487229244291_0002
17/02/16 17:03:26 INFO mapreduce.Job: Job job_1487229244291_0002 running in uber mode : false
17/02/16 17:03:26 INFO mapreduce.Job:  map 0% reduce 0%
17/02/16 17:03:32 INFO mapreduce.Job:  map 100% reduce 0%
17/02/16 17:03:40 INFO mapreduce.Job:  map 100% reduce 100%
17/02/16 17:03:40 INFO mapreduce.Job: Job job_1487229244291_0002 completed successfully
17/02/16 17:03:40 INFO mapreduce.Job: Counters: 49
	File System Counters
		FILE: Number of bytes read=153
		FILE: Number of bytes written=237243
		FILE: Number of read operations=0
		FILE: Number of large read operations=0
		FILE: Number of write operations=0
		HDFS: Number of bytes read=392
		HDFS: Number of bytes written=107
		HDFS: Number of read operations=7
		HDFS: Number of large read operations=0
		HDFS: Number of write operations=2
	Job Counters 
		Launched map tasks=1
		Launched reduce tasks=1
		Data-local map tasks=1
		Total time spent by all maps in occupied slots (ms)=3835
		Total time spent by all reduces in occupied slots (ms)=3848
		Total time spent by all map tasks (ms)=3835
		Total time spent by all reduce tasks (ms)=3848
		Total vcore-milliseconds taken by all map tasks=3835
		Total vcore-milliseconds taken by all reduce tasks=3848
		Total megabyte-milliseconds taken by all map tasks=3927040
		Total megabyte-milliseconds taken by all reduce tasks=3940352
	Map-Reduce Framework
		Map input records=5
		Map output records=5
		Map output bytes=137
		Map output materialized bytes=153
		Input split bytes=129
		Combine input records=0
		Combine output records=0
		Reduce input groups=1
		Reduce shuffle bytes=153
		Reduce input records=5
		Reduce output records=5
		Spilled Records=10
		Shuffled Maps =1
		Failed Shuffles=0
		Merged Map outputs=1
		GC time elapsed (ms)=133
		CPU time spent (ms)=1020
		Physical memory (bytes) snapshot=307019776
		Virtual memory (bytes) snapshot=4156833792
		Total committed heap usage (bytes)=165810176
	Shuffle Errors
		BAD_ID=0
		CONNECTION=0
		IO_ERROR=0
		WRONG_LENGTH=0
		WRONG_MAP=0
		WRONG_REDUCE=0
	File Input Format Counters 
		Bytes Read=263
	File Output Format Counters 
		Bytes Written=107
[hadoop@master hadoop]$ 
```
同样可以通过 Web 界面查看任务进度 http://master:8088/cluster，在 Web 界面点击 "Tracking UI" 这一列的 History 连接，可以看到任务的运行信息

```txt
                                All Applications
Cluster       Cluster Metrics
About         Apps      Apps    Apps    Apps      Memory Memory VCores Active
Nodes         Submitted Pending Running Completed Used   Total  Total  Nodes
Node Labels   2         0       0       2         0      16 GB  16     2
Applications  Scheduler Metrics
  NEW         SchedulerType     SchedulingResourceType Minimum Allocation      Maximum Allocation 
  NEW_SAVING  CapacityScheduler [MEMORY]               <memory:1024, vCores:1> <memory:8192, vCores:8>
  SUBMITTED   Show 20 entries
  ACCEPTED    ID                             User   Name     ApplicationType State   FinalStatus
  RUNNING     application_1487229244291_0001 hadoop grep-search MAPREDUCE   FINISHED SUCCEEDED
  FINISHED    application_1487229244291_0002 hadoop grep-sort   MAPREDUCE   FINISHED SUCCEEDED
  FAILED 
  KILLED 
Scheduler 
``` 
关闭 Hadoop 集群也是在 Master 节点上执行的：

```txt
[hadoop@master hadoop]$ stop-yarn.sh 
stopping yarn daemons
stopping resourcemanager
slave2: stopping nodemanager
slave1: stopping nodemanager
slave2: nodemanager did not stop gracefully after 5 seconds: killing with kill -9
slave1: nodemanager did not stop gracefully after 5 seconds: killing with kill -9
no proxyserver to stop

[hadoop@master hadoop]$ stop-dfs.sh 
Stopping namenodes on [master]
master: stopping namenode
slave2: stopping datanode
slave1: stopping datanode
Stopping secondary namenodes [master]
master: stopping secondarynamenode

[hadoop@master hadoop]$ mr-jobhistory-daemon.sh stop historyserver
stopping historyserver

[hadoop@master hadoop]$
```
参考资料
===
- Hadoop集群安装配置教程_Hadoop2.6.0_Ubuntu/CentOS http://dblab.xmu.edu.cn/blog/install-hadoop-cluster/