hadoop安装前准备工作
---
1、 创建hadoop用户

- sudo useradd -m hadoop -s /bin/bash   #创建hadoop用户
- sudo passwd hadoop                    #设置hadoop用户密码
- vi /etc/sudoers                       #将hadoop用户添加到sudoer中

```txt
vi /etc/sudoers
## Allow root to run any commands anywhere 
root    ALL=(ALL)       ALL
hadoop  ALL=(ALL)       ALL
```
2、设置免密码登录

``` txt
[hadoop@node1-hadoop1 .ssh]$ cat id_rsa.pub >> authorized_keys
[hadoop@node1-hadoop1 .ssh]$ ls
authorized_keys  id_rsa  id_rsa.pub
```
3、 设置java home

检查有没有安装jdk

``` txt
# rpm -qa|grep openjdk
java-1.8.0-openjdk-headless-1.8.0.111-2.b15.el7_3.x86_64
java-1.8.0-openjdk-1.8.0.111-2.b15.el7_3.x86_64
```
javac
---
- oracle java 默认安装目录 /usr/java/jdk1.8.0_121
- openjdk     默认安装目录 /usr/lib/jvm/

4、 如果用oracle java 就需要卸载掉openjdk

```txt
# rpm --nodeps -e java-1.8.0-openjdk-headless-1.8.0.111-2.b15.el7_3.x86_64
# rpm --nodeps -e java-1.8.0-openjdk-1.8.0.111-2.b15.el7_3.x86_64
```
5、 oracle java 已经安装好

``` txt
# rpm -ivh jdk-8u121-linux-x64.rpm 
Preparing...                          ################################# [100%]
Updating / installing...
   1:jdk1.8.0_121-2000:1.8.0_121-fcs  ################################# [100%]
Unpacking JAR files...
	tools.jar...
	plugin.jar...
	javaws.jar...
	deploy.jar...
	rt.jar...
	jsse.jar...
	charsets.jar...
	localedata.jar...
```
6、 设置环境变量JAVA_HOME /usr/java/jdk1.8.0_121/bin/

```txt
#set java environment
export JAVA_HOME=/usr/java/jdk1.8.0_121
export CLASSPATH=.:$JAVA_HOME/lib/dt.jar:$JAVA_HOME/lib/tools.jar
export PATH=$PATH:$JAVA_HOME/bin
```
7、 # source /etc/profile   #立即生效

8、 检验一下 JAVA_HOME 是否设置正确

```txt
# echo $JAVA_HOME

[root@node1-hadoop1 ~]# source /etc/profile   #立即生效
[root@node2-hadoop2 jdk1.8.0_121]# vi /etc/profile
[root@node2-hadoop2 jdk1.8.0_121]# source /etc/profile

[root@node2-hadoop2 jdk1.8.0_121]# java -version
openjdk version "1.8.0_111"
OpenJDK Runtime Environment (build 1.8.0_111-b15)
OpenJDK 64-Bit Server VM (build 25.111-b15, mixed mode)

[root@node2-hadoop2 jdk1.8.0_121]# echo $JAVA_HOME
/usr/java/jdk1.8.0_121

[root@node2-hadoop2 jdk1.8.0_121]# $JAVA_HOME/bin/java -version
java version "1.8.0_121"
Java(TM) SE Runtime Environment (build 1.8.0_121-b13)
Java HotSpot(TM) 64-Bit Server VM (build 25.121-b13, mixed mode)

两个不一致所以要继续修改JAVA_HOME参数
```
9、 将hadoop-2.7.3.tar.gz解压放到  /usr/local/ 目录中

10、 设置hadoop文件夹权限

``` txt
chown -R hadoop:hadoop hadoop/
[root@node1-hadoop1 local]# ll
total 0
drwxr-xr-x. 2 root   root     6 Nov  5 23:38 bin
drwxr-xr-x. 2 root   root     6 Nov  5 23:38 etc
drwxr-xr-x. 2 root   root     6 Nov  5 23:38 games
drwxr-xr-x  9 hadoop hadoop 139 Aug 18 09:49 hadoop
```
11、测试hadoop

```txt
[root@node2-hadoop2 hadoop]# ./bin/hadoop version
Hadoop 2.7.3
Subversion https://git-wip-us.apache.org/repos/asf/hadoop.git -r baa91f7c6bc9cb92be5982de4719c1c8af91ccff
Compiled by root on 2016-08-18T01:41Z
Compiled with protoc 2.5.0
From source with checksum 2e4ce5f957ea4db193bce3734ff29ff4
This command was run using /usr/local/hadoop/share/hadoop/common/hadoop-common-2.7.3.jar
```
测试OK

/usr/local/hadoop 为当前目录，./bin/hadoop version 是在/usr/local/hadoop 中，的相对路径

```txt
Hadoop 的配置文件位于 /usr/local/hadoop/etc/hadoop/ 中

如果启动 Hadoop 时遇到输出非常多“ssh: Could not resolve hostname xxx”的异常情况
在 ~/.bashrc 中，增加如下两行内容（设置过程与 JAVA_HOME 变量一样，其中 HADOOP_HOME 为 Hadoop 的安装目录）：

export HADOOP_HOME=/usr/local/hadoop
export HADOOP_COMMON_LIB_NATIVE_DIR=$HADOOP_HOME/lib/native
```
配置完成后，执行 NameNode 的格式化:


参考资料
---
- 1 、[Linux安装JDK1.8] http://www.cnblogs.com/mstk/archive/2014/08/25/3934356.html

``` txt
1. 安装前，最好先删除Linux自带的OpenJDK：

(1)运行java-version，会发现Linux自带的OpenJDK，运行rpm -qa | grep OpenJDK，找出自带的OpenJDK名称；

(2)运行rpm - nodeps -e OpenJDK名称，删除OpenJDK；

2. 下载jdk-8u20-linux-x64.rpm，运行rpm -ivh jdk-8u20-linux-x64.rpm安装；

3. 运行vim /etc/profile，在文件末尾输入以下几行：

export JAVA_HOME=/usr/java/jdk1.8.0_20
export CLASSPATH=.:$JAVA_HOME/lib/dt.jar:$JAVA_HOME/lib/tools.jar
export PATH=$PATH:$JAVA_HOME/bin

保存，退出；

4. 运行source /etc/profile，使/etc/profile文件生效，或者重启；

5. 运行java -version，返回结果如下：

java version "1.8.0_20"
Java(TM) SE Runtime Environment (build 1.8.0_20-b26)
Java HotSpot(TM) 64-Bit Server VM (build 25.20-b23, mixed mode)

说明JDK1.8已经安装成功！
```
- 2、[Hadoop安装教程_单机/伪分布式配置_Hadoop2.6.0/Ubuntu14.04] http://dblab.xmu.edu.cn/blog/install-hadoop/
- 3、[linux环境下java版本的升级和卸载] http://blog.csdn.net/jiahehao/article/details/6834214
- 4、[Linux下修改/设置环境变量JAVA_HOME] http://blog.csdn.net/jillliang/article/details/8216308
- 5、[oracle java] http://www.oracle.com/technetwork/java/javase/downloads/jdk8-downloads-2133151.html