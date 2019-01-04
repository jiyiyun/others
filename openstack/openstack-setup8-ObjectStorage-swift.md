The Object Storage services (swift) work together to provide object storage and retrieval through a REST API.

This chapter assumes a working setup of OpenStack following the OpenStack Installation Tutorial.

Your environment must at least include the Identity service (keystone) prior to deploying Object Storage.

The OpenStack Object Storage is a multi-tenant object storage system. It is highly scalable and can manage large amounts of unstructured data at low cost through a RESTful HTTP API.

It includes the following components:

Proxy servers (swift-proxy-server)
---

Accepts OpenStack Object Storage API and raw HTTP requests to upload files, modify metadata, and create containers. It also serves file or container listings to web browsers. To improve performance, the proxy server can use an optional cache that is usually deployed with memcache.

Account servers (swift-account-server)
---

Manages accounts defined with Object Storage.

Container servers (swift-container-server)
---

Manages the mapping of containers or folders, within Object Storage.

Object servers (swift-object-server)
---

Manages actual objects, such as files, on the storage nodes.

Various periodic processes
---

Performs housekeeping tasks on the large data store. The replication services ensure consistency and availability through the cluster. Other periodic processes include auditors, updaters, and reapers.

WSGI middleware
---

Handles authentication and is usually OpenStack Identity.

swift client
---

Enables users to submit commands to the REST API through a command-line client authorized as either a admin user, reseller user, or swift user.

swift-init
---

Script that initializes the building of the ring file, takes daemon names as parameter and offers commands. Documented in http://docs.openstack.org/developer/swift/admin_guide.html#managing-services.

swift-recon
---

A cli tool used to retrieve various metrics and telemetry information about a cluster that has been collected by the swift-recon middleware.

swift-ring-builder
---

Storage ring build and rebalance utility. Documented in http://docs.openstack.org/developer/swift/admin_guide.html#managing-the-rings.

块存储&文件存储&对象存储
---

块存储：传统的存储结构，典型的代表是SAN。对于用户而言，块存储好比一块大磁盘，用户可以根据需要将裸设备格式化成想要的文件系统来使用。其可扩展性较差。

文件存储：典型代表是NAS。对于用户而言，文件存储好比一个共享文件夹，文件系统已存在，用户的数据以文件的形式存在于系统中。但由于以文件为传输协议，开销较大，不适用于高性能的集群搭建。

对象存储：典型代表是swift、s3。用户的每个数据对象既包含文件的元数据又包含存储数据。

数据结构       Ring（环）：用于记录存储对象与物理位置间的映射关系。在涉及查询Account、Container、Object信息时，就需要查询集群的Ring信息。Ring使用Zone、Device、Partition和Replica来维护这些映射信息。Ring中每个Partition在集群中都（默认）有3个Replica。每个Partition的位置由Ring来维护，并存储在映射中。Ring文件在系统初始化时创建，之后每次增减存储节点时，需要重新平衡一下Ring。

（Zone：物理位置分区，Device：物理设备，Partition：虚拟出的物理设备，Replica：冗余副本）

创建Ring

Ring共有三种，分别为Account Ring、Container Ring、Object  账户（account）、容器（container）、对象（object）、环（rings）

Ring。Ring需要在整个集群中保持完全相同，因此需要在某一台PC上创建Ring文件，然后复制到其他PC上。我们将在PC1上进行Ring的创建，然后复制到PC2上。
      
首先，使用如下命令创建三个Ring。其中，18表示Ring的分区数为2^18；2表示对象副本数为2；1表示分区数据的迁移时间为1小时（这个解释有待证实）。

```txt
$ cd /etc/swift/
$ swift-ring-builder account.builder create 18 2 1
$ swift-ring-builder container.builder create 18 2 1
$ swift-ring-builder object.builder create 18 2 1

注：builder参数的值表示2^18次方的值也就是大小，设置总的存储量，预计整个环使用这种“分割权力”的值。2值为表示每个对象的副本的数量，最后一个值是小时数用来限制移动分区次数
```

```txt
/etc/swift/account-server.conf     bind_port = 6002
/etc/swift/container-server.conf   bind_port = 6001
/etc/swift/object-server.conf      bind_port = 6000
这三个配置文件的端口号不同
```

创建和分发初始环 Rings

在这个阶段进行之前，要确保 Controller 与两台 Storage 节点都确定设定与安装完成 Swift。若没问题的话，回到Controller节点，并进入 /etc/swift 目录来完成以下步骤。

```txt
$ cd /etc/swift
建立 Account Ring

Account Server 使用 Account Ring 来维护容器的列表。首先透过以下指令建立一个 account.builder ：

```txt
$ sudo swift-ring-builder account.builder create 10 3 1
```
然后新增每一个 Storage 节点的 Account Server 资讯到 Ring 中：

```txt
# Object1 sdb sdc
$ sudo swift-ring-builder account.builder \
add --region 1 --zone 1 --ip 192.168.8.155 \
--port 6002 --device sdb --weight 100
$ sudo swift-ring-builder account.builder \
add --region 1 --zone 1 --ip 192.168.8.155 \
--port 6002 --device sdc --weight 100

 # Object2 sdb sdc
$ sudo swift-ring-builder account.builder \ 
add --region 1 --zone 2 --ip 192.168.8.156 \ 
--port 6002 --device sdb --weight 100 

.....依次类推
```
当完成后，即可透过以下指令来验证是否正确：

```txt
$ sudo swift-ring-builder account.builder
```
1.对每一个存储node创建更改/srv/node并加入到这个ring

```txt
$ZONE                    # 设置该zone编号的存储设备
$STORAGE_LOCAL_NET_IP    # 添加StorageNodeIP地址
$DEVICE                  # 存储设备磁盘挂载分区
$WEIGHT                  # 权重
```

```txt
$ swift-ring-builder account.builder add z$ZONE-$STORAGE_LOCAL_NET_IP:6002/$DEVICE  $WEIGHT

$ swift-ring-builder container.builder add z$ZONE-$STORAGE_LOCAL_NET_IP:6001/$DEVICE $WEIGHT

$ swift-ring-builder object.builder add z$ZONE-$STORAGE_LOCAL_NET_IP:6000/$DEVICE $WEIGHT
```
2.验证每个ring中的内容

```txt
$ swift-ring-builder account.builder

$ swift-ring-builder container.builder

$ swift-ring-builder object.builder
```
3.重新平衡ring
---

```txt
$ swift-ring-builder account.builder rebalance

$ swift-ring-builder container.builder rebalance

$ swift-ring-builder object.builder rebalance
```
4.复制ring.gz
---

复制刚刚生成的account.ring.gz，container.ring.gz，object.ring.gz文件到每一个proxy和storage nodes的/etc/swift文件里,确保所有的配置文件的用户拥有swift用户

```txt
$chown -R swift:swift /etc/swift
```
启动proxy server

```txt
$ swift-init proxy start
```
```txt
# cd /etc/swift
# swift-ring-builder account.builder add z1-192.168.3.52:6002/sdb1 100
# swift-ring-builder container.builder add z1-192.168.3.52:6001/sdb1 100
# swift-ring-builder object.builder add z1-192.168.3.52:6000/sdb1 100
 
# swift-ring-builder account.builder add z2-192.168.3.53:6002/sdb1 100
# swift-ring-builder container.builder add z2-192.168.3.53:6001/sdb1 100
# swift-ring-builder object.builder add z2-192.168.3.53:6000/sdb1 100
```
Ring文件创建完毕后，可以通过以下命令来查看刚才添加的信息，以验证是否输入正确。若发现错误，以Account Ring为例，可以使用swift-ring-builder account.builder的删除方法删除已添加的设备，然后重新添加。

```txt
# cd /etc/swift
# swift-ring-builder account.builder
# swift-ring-builder container.builder
# swift-ring-builder object.builder
```
完成设备的添加后，我们还需要创建Ring的最后一步，即平衡环。这个过程需要消耗一些时间。成功之后，会在当前目录生成account.ring.gz、container.ring.gz和object.ring.gz三个文件，这三个文件就是所有节点（包括Proxy Server和Storage Server）要用到的Ring文件。我们需要将这三个文件拷贝到PC2中的/etc/swift目录下。

```txt
# cd /etc/swift
# swift-ring-builder account.builder rebalance
# swift-ring-builder container.builder rebalance
# swift-ring-builder object.builder rebalance
```

```txt
# cd /etc/swift
# swift-ring-builder account.builder create 18 2 1
# swift-ring-builder container.builder create 18 2 1
# swift-ring-builder object.builder create 18 2 1
```
然后向三个Ring中添加存储设备。其中z1和z2表示zone1和zone2；sdb1为Swift使用的存储空间，即上文挂在的回环设备；100代表设备的权重。

![ring](http://upload.server110.com/image/20151114/14341Q211-0.jpg)


角色

认证节点：用户身份认证，官方文档说支持keystone、tempauth及swauth认证方式。本人亲测keystone与tempauth均成功，swauth没有测试过。

代理节点：节点运行代理服务，处理http请求。

存储节点：存储数据对象，其上运行account（账号）、container（容器）、和object（对象）服务。

系统架构

![act](http://upload.server110.com/image/20151114/14341UZ9-1.jpg)
Configure networking
---

Before you start deploying the Object Storage service in your OpenStack environment, configure networking for two additional storage nodes.

这部分的在存储Storage节点上
===

First node¶

```txt
Configure network interfaces¶
Configure the management interface:
IP address: 10.0.0.51
Network mask: 255.255.255.0 (or /24)
Default gateway: 10.0.0.1
Configure name resolution¶
Set the hostname of the node to object1.
```

Edit the <code>/etc/hosts</code> file to contain the following:

```txt
# controller
10.0.0.11       controller

# compute1
10.0.0.31       compute1

# block1
10.0.0.41       block1

# object1
10.0.0.51       object1

# object2
10.0.0.52       object2
```
Reboot the system to activate the changes.

Second node¶

```txt
Configure network interfaces¶
Configure the management interface:
IP address: 10.0.0.52
Network mask: 255.255.255.0 (or /24)
Default gateway: 10.0.0.1
Configure name resolution¶
Set the hostname of the node to object2.
```
Edit the <cpde>/etc/hosts</cpde> file to contain the following:

```txt
# controller
10.0.0.11       controller

# compute1
10.0.0.31       compute1

# block1
10.0.0.41       block1

# object1
10.0.0.51       object1

# object2
10.0.0.52       object2
```
Reboot the system to activate the changes.

Warning : Some distributions add an extraneous entry in the <code>/etc/hosts</code> file that resolves the actual hostname to another loopback IP address such as 127.0.1.1. You must comment out or remove this entry to prevent name resolution problems. Do not remove the 127.0.0.1 entry.


这部分的在控制Controller节点上
===

Install and configure the controller node

This section describes how to install and configure the proxy service that handles requests for the account, container, and object services operating on the storage nodes.

Note that installation and configuration vary by distribution.

Install and configure the controller node for openSUSE and SUSE Linux Enterprise
Install and configure the controller node for Red Hat Enterprise Linux and CentOS
Install and configure the controller node for Ubuntu
Install and configure the controller node for Debian

Install and configure the controller node for Red Hat Enterprise Linux and CentOS
---

This section describes how to install and configure the proxy service that handles requests for the account, container, and object services operating on the storage nodes. For simplicity, this guide installs and configures the proxy service on the controller node. However, you can run the proxy service on any node with network connectivity to the storage nodes. Additionally, you can install and configure the proxy service on multiple nodes to increase performance and redundancy. For more information, see the Deployment Guide.

This section applies to Red Hat Enterprise Linux 7 and CentOS 7.

Prerequisites¶

The proxy service relies on an authentication and authorization mechanism such as the Identity service. However, unlike other services, it also offers an internal mechanism that allows it to operate without any other OpenStack services. Before you configure the Object Storage service, you must create service credentials and an API endpoint.

Note : The Object Storage service does not use an SQL database on the controller node. Instead, it uses distributed SQLite databases on each storage node.

提示：对象存储swift没有使用SQL数据库，而是使用分布式的sqlite数据库，所以这里不用配置MYSQL数据库

Source the admin credentials to gain access to admin-only CLI commands:

```txt
$ . admin-openrc
```
To create the Identity service credentials, complete these steps:

Create the swift user:

```txt
$ openstack user create --domain default --password-prompt swift
User Password:
Repeat User Password:
+---------------------+----------------------------------+
| Field               | Value                            |
+---------------------+----------------------------------+
| domain_id           | default                          |
| enabled             | True                             |
| id                  | 2d6651f9075945f99601feac3ef0218d |
| name                | swift                            |
| options             | {}                               |
| password_expires_at | None                             |
+---------------------+----------------------------------+
```
Add the admin role to the swift user:

```txt
$ openstack role add --project service --user swift admin
```
Note ： This command provides no output.

Create the swift service entity:

```txt
$ openstack service create --name swift \
  --description "OpenStack Object Storage" object-store
+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description | OpenStack Object Storage         |
| enabled     | True                             |
| id          | c96fcd4379ed40fb95ad825cb4d3e20d |
| name        | swift                            |
| type        | object-store                     |
+-------------+----------------------------------+

Create the Object Storage service API endpoints:

$ openstack endpoint create --region RegionOne \
  object-store public http://controller:8080/v1/AUTH_%\(tenant_id\)s
+--------------+----------------------------------------------+
| Field        | Value                                        |
+--------------+----------------------------------------------+
| enabled      | True                                         |
| id           | e4cc01b395484688ae6b6908405eb046             |
| interface    | public                                       |
| region       | RegionOne                                    |
| region_id    | RegionOne                                    |
| service_id   | c96fcd4379ed40fb95ad825cb4d3e20d             |
| service_name | swift                                        |
| service_type | object-store                                 |
| url          | http://controller:8080/v1/AUTH_%(tenant_id)s |
+--------------+----------------------------------------------+

$ openstack endpoint create --region RegionOne \
  object-store internal http://controller:8080/v1/AUTH_%\(tenant_id\)s
+--------------+----------------------------------------------+
| Field        | Value                                        |
+--------------+----------------------------------------------+
| enabled      | True                                         |
| id           | 2cc9cbda90334cfbaf5d533b07a7cc9b             |
| interface    | internal                                     |
| region       | RegionOne                                    |
| region_id    | RegionOne                                    |
| service_id   | c96fcd4379ed40fb95ad825cb4d3e20d             |
| service_name | swift                                        |
| service_type | object-store                                 |
| url          | http://controller:8080/v1/AUTH_%(tenant_id)s |
+--------------+----------------------------------------------+

$ openstack endpoint create --region RegionOne \
  object-store admin http://controller:8080/v1
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | 5e793d7f54074092bdb3153332cfa238 |
| interface    | admin                            |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | c96fcd4379ed40fb95ad825cb4d3e20d |
| service_name | swift                            |
| service_type | object-store                     |
| url          | http://controller:8080/v1        |
+--------------+----------------------------------+
```
Install and configure components¶

Note : Default configuration files vary by distribution. You might need to add these sections and options rather than modifying existing sections and options. Also, an ellipsis (...) in the configuration snippets indicates potential default configuration options that you should retain.对于默认配置尽量添加配置信息不要修改

Install the packages:

```txt
# yum install openstack-swift-proxy python-swiftclient \
  python-keystoneclient python-keystonemiddleware \
  memcached
```
Note : Complete OpenStack environments already include some of these packages.

Obtain the proxy service configuration file from the Object Storage source repository:

```txt
# curl -o /etc/swift/proxy-server.conf https://git.openstack.org/cgit/openstack/swift/plain/etc/proxy-server.conf-sample?h=stable/newton
```
Edit the <code>/etc/swift/proxy-server.conf</code> file and complete the following actions:

In the <code>[DEFAULT]</code> section, configure the bind port, user, and configuration directory:

```txt
[DEFAULT]
...
bind_port = 8080
user = swift
swift_dir = /etc/swift
```
In the <code>[pipeline:main]</code> section, remove the tempurl and tempauth modules and add the authtoken and keystoneauth modules:

```txt
[pipeline:main]
pipeline = catch_errors gatekeeper healthcheck proxy-logging cache container_sync bulk ratelimit authtoken keystoneauth container-quotas account-quotas slo dlo versioned_writes proxy-logging proxy-server
```
Note : Do not change the order of the modules.

Note : For more information on other modules that enable additional features, see the Deployment Guide.

In the <code>[app:proxy-server]</code> section, enable automatic account creation:

```txt
[app:proxy-server]
use = egg:swift#proxy
...
account_autocreate = True
```
In the <code>[filter:keystoneauth]</code> section, configure the operator roles:

```txt
[filter:keystoneauth]
use = egg:swift#keystoneauth
...
operator_roles = admin,user
```
In the <code>[filter:authtoken]</code> section, configure Identity service access:

```txt
[filter:authtoken]
paste.filter_factory = keystonemiddleware.auth_token:filter_factory
...
auth_uri = http://controller:5000
auth_url = http://controller:35357
memcached_servers = controller:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = swift
password = SWIFT_PASS
delay_auth_decision = True
```
Replace <code>SWIFT_PASS</code> with the password you chose for the swift user in the Identity service.

Note : Comment out or remove any other options in the <code>[filter:authtoken]</code> section.

In the <code>[filter:cache]</code> section, configure the memcached location:

```txt
[filter:cache]
use = egg:swift#memcache
...
memcache_servers = controller:11211
```

这部分的在存储Storage节点上
===

Install and configure the storage nodes
---

Install and configure the storage nodes for Red Hat Enterprise Linux and CentOS

Contents
Prerequisites
Install and configure components
This section describes how to install and configure storage nodes that operate the account, container, and object services. For simplicity, this configuration references two storage nodes, each containing two empty local block storage devices. The instructions use /dev/sdb and /dev/sdc, but you can substitute different values for your particular nodes.

Although Object Storage supports any file system with extended attributes (xattr), testing and benchmarking indicate the best performance and reliability on XFS. For more information on horizontally scaling your environment, see the Deployment Guide.

This section applies to Red Hat Enterprise Linux 7 and CentOS 7.

Prerequisites¶
---

Before you install and configure the Object Storage service on the storage nodes, you must prepare the storage devices.

Note : Perform these steps on each storage node.

Install the supporting utility packages:
---

```txt
# yum install xfsprogs rsync
```
Format the <code>/dev/sdb</code> and <code>/dev/sdc</code> devices as XFS:

```txt
# mkfs.xfs /dev/sdb
# mkfs.xfs /dev/sdc
```
Create the mount point directory structure:

```txt
# mkdir -p /srv/node/sdb
# mkdir -p /srv/node/sdc
```
Edit the <code>/etc/fstab</code> file and add the following to it:

```txt
/dev/sdb /srv/node/sdb xfs noatime,nodiratime,nobarrier,logbufs=8 0 2
/dev/sdc /srv/node/sdc xfs noatime,nodiratime,nobarrier,logbufs=8 0 2
```
Mount the devices:

```txt
# mount /srv/node/sdb
# mount /srv/node/sdc
```
Create or edit the <code>/etc/rsyncd.conf</code> file to contain the following:

```txt
uid = swift
gid = swift
log file = /var/log/rsyncd.log
pid file = /var/run/rsyncd.pid
address = MANAGEMENT_INTERFACE_IP_ADDRESS

[account]
max connections = 2
path = /srv/node/
read only = False
lock file = /var/lock/account.lock

[container]
max connections = 2
path = /srv/node/
read only = False
lock file = /var/lock/container.lock

[object]
max connections = 2
path = /srv/node/
read only = False
lock file = /var/lock/object.lock
```
Replace <code>MANAGEMENT_INTERFACE_IP_ADDRESS</code> with the IP address of the management network on the storage node.

Note : The rsync service requires no authentication, so consider running it on a private network in production environments.

Start the rsyncd service and configure it to start when the system boots:

```txt
# systemctl enable rsyncd.service
# systemctl start rsyncd.service
```

这部分的在控制Controller节点上
===

Install and configure components¶
---

Note : Default configuration files vary by distribution. You might need to add these sections and options rather than modifying existing sections and options. Also, an ellipsis (...) in the configuration snippets indicates potential default configuration options that you should retain.

Note : Perform these steps on each storage node.

Install the packages:

```txt
# yum install openstack-swift-account openstack-swift-container \
  openstack-swift-object
```
Obtain the accounting, container, and object service configuration files from the Object Storage source repository:

```txt
# curl -o /etc/swift/account-server.conf https://git.openstack.org/cgit/openstack/swift/plain/etc/account-server.conf-sample?h=stable/newton
# curl -o /etc/swift/container-server.conf https://git.openstack.org/cgit/openstack/swift/plain/etc/container-server.conf-sample?h=stable/newton
# curl -o /etc/swift/object-server.conf https://git.openstack.org/cgit/openstack/swift/plain/etc/object-server.conf-sample?h=stable/newton
```
Edit the <code>/etc/swift/account-server.conf</code> file and complete the following actions:

In the <code>[DEFAULT]</code> section, configure the bind IP address, bind port, user, configuration directory, and mount point directory:

```txt
[DEFAULT]
...
bind_ip = MANAGEMENT_INTERFACE_IP_ADDRESS
bind_port = 6002
user = swift
swift_dir = /etc/swift
devices = /srv/node
mount_check = True
```
Replace <code>MANAGEMENT_INTERFACE_IP_ADDRESS</code> with the IP address of the management network on the storage node.

In the <code>[pipeline:main]</code> section, enable the appropriate modules:

```txt
[pipeline:main]
pipeline = healthcheck recon account-server
```
Note : For more information on other modules that enable additional features, see the Deployment Guide.

In the <code>[filter:recon]</code> section, configure the recon (meters) cache directory:

```
[filter:recon]
use = egg:swift#recon
...
recon_cache_path = /var/cache/swift
```
Edit the <code>/etc/swift/container-server.conf</code> file and complete the following actions:

In the <code>[DEFAULT]</code> section, configure the bind IP address, bind port, user, configuration directory, and mount point directory:

```txt
[DEFAULT]
...
bind_ip = MANAGEMENT_INTERFACE_IP_ADDRESS
bind_port = 6001
user = swift
swift_dir = /etc/swift
devices = /srv/node
mount_check = True
```
Replace <code>MANAGEMENT_INTERFACE_IP_ADDRESS</code> with the IP address of the management network on the storage node.

In the <code>[pipeline:main]</code> section, enable the appropriate modules:

```txt
[pipeline:main]
pipeline = healthcheck recon container-server
```
Note : For more information on other modules that enable additional features, see the Deployment Guide.

In the <code>[filter:recon]</code> section, configure the recon (meters) cache directory:

```txt
[filter:recon]
use = egg:swift#recon
...
recon_cache_path = /var/cache/swift
```
Edit the <code>/etc/swift/object-server.conf</code> file and complete the following actions:

In the <code>[DEFAULT]</code> section, configure the bind IP address, bind port, user, configuration directory, and mount point directory:

```txt
[DEFAULT]
...
bind_ip = MANAGEMENT_INTERFACE_IP_ADDRESS
bind_port = 6000
user = swift
swift_dir = /etc/swift
devices = /srv/node
mount_check = True
```
Replace <code>MANAGEMENT_INTERFACE_IP_ADDRESS</code> with the IP address of the management network on the storage node.

In the <code>[pipeline:main]</code> section, enable the appropriate modules:

```txt
[pipeline:main]
pipeline = healthcheck recon object-server
```
Note : For more information on other modules that enable additional features, see the Deployment Guide.

In the <code>[filter:recon]</code> section, configure the recon (meters) cache and lock directories:

```txt
[filter:recon]
use = egg:swift#recon
...
recon_cache_path = /var/cache/swift
recon_lock_path = /var/lock
```
Ensure proper ownership of the mount point directory structure:

```txt
# chown -R swift:swift /srv/node
```
Create the recon directory and ensure proper ownership of it:

```txt
# mkdir -p /var/cache/swift
# chown -R root:swift /var/cache/swift
# chmod -R 775 /var/cache/swift
```

这部分的在控制Controller节点上
===

Create and distribute initial rings 创建分布式的令牌环
---

Before starting the Object Storage services, you must create the initial account, container, and object rings. The ring builder creates configuration files that each node uses to determine and deploy the storage architecture. For simplicity, this guide uses one region and two zones with 2^10 (1024) maximum partitions, 3 replicas of each object, and 1 hour minimum time between moving a partition more than once. For Object Storage, a partition indicates a directory on a storage device rather than a conventional partition table. For more information, see the Deployment Guide.

Note :Perform these steps on the controller node.

Create account ring¶ 创建account令牌环
---

The account server uses the account ring to maintain lists of containers.

Change to the <code>/etc/swift</code> directory. 

进入<code>/etc/swift/</code>目录

Create the base account.builder file:
---

```txt
# swift-ring-builder account.builder create 10 3 1
```
Note : This command provides no output.

Add each storage node to the ring: 添加所有节点到令牌环上
---

```txt
# swift-ring-builder account.builder \
  add --region 1 --zone 1 --ip STORAGE_NODE_MANAGEMENT_INTERFACE_IP_ADDRESS --port 6002 \
  --device DEVICE_NAME --weight DEVICE_WEIGHT
```
Replace <code>STORAGE_NODE_MANAGEMENT_INTERFACE_IP_ADDRESS</code> with the IP address of the management network on the storage node. Replace <code>DEVICE_NAME</code> with a storage device name on the same storage node. For example, using the first storage node in Install and configure the storage nodes with the <code>/dev/sdb</code> storage device and weight of 100:

```txt
# swift-ring-builder account.builder add \
  --region 1 --zone 1 --ip 10.0.0.51 --port 6002 --device sdb --weight 100
```
Repeat this command for each storage device on each storage node. In the example architecture, use the command in four variations:

```txt
# swift-ring-builder account.builder add \
  --region 1 --zone 1 --ip 10.0.0.51 --port 6002 --device sdb --weight 100
Device d0r1z1-10.0.0.51:6002R10.0.0.51:6002/sdb_"" with 100.0 weight got id 0
# swift-ring-builder account.builder add \
  --region 1 --zone 1 --ip 10.0.0.51 --port 6002 --device sdc --weight 100
Device d1r1z2-10.0.0.51:6002R10.0.0.51:6002/sdc_"" with 100.0 weight got id 1
# swift-ring-builder account.builder add \
  --region 1 --zone 2 --ip 10.0.0.52 --port 6002 --device sdb --weight 100
Device d2r1z3-10.0.0.52:6002R10.0.0.52:6002/sdb_"" with 100.0 weight got id 2
# swift-ring-builder account.builder add \
  --region 1 --zone 2 --ip 10.0.0.52 --port 6002 --device sdc --weight 100
Device d3r1z4-10.0.0.52:6002R10.0.0.52:6002/sdc_"" with 100.0 weight got id 3
```
Verify the ring contents:
```txt
# swift-ring-builder account.builder
account.builder, build version 4
1024 partitions, 3.000000 replicas, 1 regions, 2 zones, 4 devices, 100.00 balance, 0.00 dispersion
The minimum number of hours before a partition can be reassigned is 1
The overload factor is 0.00% (0.000000)
Devices:    id  region  zone      ip address  port  replication ip  replication port      name weight partitions balance meta
             0       1     1       10.0.0.51  6002       10.0.0.51              6002      sdb  100.00          0 -100.00
             1       1     1       10.0.0.51  6002       10.0.0.51              6002      sdc  100.00          0 -100.00
             2       1     2       10.0.0.52  6002       10.0.0.52              6002      sdb  100.00          0 -100.00
             3       1     2       10.0.0.52  6002       10.0.0.52              6002      sdc  100.00          0 -100.00
```
Rebalance the ring: 再平衡平衡令牌环
---

```txt
# swift-ring-builder account.builder rebalance
```
Reassigned 1024 (100.00%) partitions. Balance is now 0.00.  Dispersion is now 0.00


Create container ring¶ 创建容器令牌环
---

The container server uses the container ring to maintain lists of objects. However, it does not track object locations.

Change to the <code>/etc/swift</code> directory. 

进入<code>/etc/swift/</code>目录

Create the base container.builder file:

```txt
# swift-ring-builder container.builder create 10 3 1
```
Note : This command provides no output.

Add each storage node to the ring:添加所有节点到令牌环上
---

```txt
# swift-ring-builder container.builder \
  add --region 1 --zone 1 --ip STORAGE_NODE_MANAGEMENT_INTERFACE_IP_ADDRESS --port 6001 \
  --device DEVICE_NAME --weight DEVICE_WEIGHT
```
Replace <code>STORAGE_NODE_MANAGEMENT_INTERFACE_IP_ADDRESS</code> with the IP address of the management network on the storage node. Replace <code>DEVICE_NAME</code> with a storage device name on the same storage node. For example, using the first storage node in Install and configure the storage nodes with the <code>/dev/sdb</code> storage device and weight of 100:

```txt
# swift-ring-builder container.builder add \
  --region 1 --zone 1 --ip 10.0.0.51 --port 6001 --device sdb --weight 100
```
Repeat this command for each storage device on each storage node. In the example architecture, use the command in four variations:

```txt
# swift-ring-builder container.builder add \
  --region 1 --zone 1 --ip 10.0.0.51 --port 6001 --device sdb --weight 100
Device d0r1z1-10.0.0.51:6001R10.0.0.51:6001/sdb_"" with 100.0 weight got id 0
# swift-ring-builder container.builder add \
  --region 1 --zone 1 --ip 10.0.0.51 --port 6001 --device sdc --weight 100
Device d1r1z2-10.0.0.51:6001R10.0.0.51:6001/sdc_"" with 100.0 weight got id 1
# swift-ring-builder container.builder add \
  --region 1 --zone 2 --ip 10.0.0.52 --port 6001 --device sdb --weight 100
Device d2r1z3-10.0.0.52:6001R10.0.0.52:6001/sdb_"" with 100.0 weight got id 2
# swift-ring-builder container.builder add \
  --region 1 --zone 2 --ip 10.0.0.52 --port 6001 --device sdc --weight 100
Device d3r1z4-10.0.0.52:6001R10.0.0.52:6001/sdc_"" with 100.0 weight got id 3
```

Verify the ring contents:

```txt
# swift-ring-builder container.builder
container.builder, build version 4
1024 partitions, 3.000000 replicas, 1 regions, 2 zones, 4 devices, 100.00 balance, 0.00 dispersion
The minimum number of hours before a partition can be reassigned is 1
The overload factor is 0.00% (0.000000)
Devices:    id  region  zone      ip address  port  replication ip  replication port      name weight partitions balance meta
             0       1     1       10.0.0.51  6001       10.0.0.51              6001      sdb  100.00          0 -100.00
             1       1     1       10.0.0.51  6001       10.0.0.51              6001      sdc  100.00          0 -100.00
             2       1     2       10.0.0.52  6001       10.0.0.52              6001      sdb  100.00          0 -100.00
             3       1     2       10.0.0.52  6001       10.0.0.52              6001      sdc  100.00          0 -100.00
```

Rebalance the ring: 令牌环的再平衡
---

```txt
# swift-ring-builder container.builder rebalance
```
Reassigned 1024 (100.00%) partitions. Balance is now 0.00.  Dispersion is now 0.00


Create object ring¶ 创建对象令牌环
---

The object server uses the object ring to maintain lists of object locations on local devices.

Change to the <code>/etc/swift</code> directory.

进入<code>/etc/swift/</code>目录

Create the base object.builder file:

```txt
# swift-ring-builder object.builder create 10 3 1
```
Note : This command provides no output.

Add each storage node to the ring:  添加存储服务器、磁盘、到令牌环
---

```txt
# swift-ring-builder object.builder \
  add --region 1 --zone 1 --ip STORAGE_NODE_MANAGEMENT_INTERFACE_IP_ADDRESS --port 6000 \
  --device DEVICE_NAME --weight DEVICE_WEIGHT
```
Replace <code>STORAGE_NODE_MANAGEMENT_INTERFACE_IP_ADDRESS</code> with the IP address of the management network on the storage node. Replace <code>DEVICE_NAME</code> with a storage device name on the same storage node. For example, using the first storage node in Install and configure the storage nodes with the <code>/dev/sdb</code> storage device and weight of 100:

```txt
# swift-ring-builder object.builder add \
  --region 1 --zone 1 --ip 10.0.0.51 --port 6000 --device sdb --weight 100
```
Repeat this command for each storage device on each storage node. In the example architecture, use the command in four variations:

```txt
# swift-ring-builder object.builder add \
  --region 1 --zone 1 --ip 10.0.0.51 --port 6000 --device sdb --weight 100
Device d0r1z1-10.0.0.51:6000R10.0.0.51:6000/sdb_"" with 100.0 weight got id 0
# swift-ring-builder object.builder add \
  --region 1 --zone 1 --ip 10.0.0.51 --port 6000 --device sdc --weight 100
Device d1r1z2-10.0.0.51:6000R10.0.0.51:6000/sdc_"" with 100.0 weight got id 1
# swift-ring-builder object.builder add \
  --region 1 --zone 2 --ip 10.0.0.52 --port 6000 --device sdb --weight 100
Device d2r1z3-10.0.0.52:6000R10.0.0.52:6000/sdb_"" with 100.0 weight got id 2
# swift-ring-builder object.builder add \
  --region 1 --zone 2 --ip 10.0.0.52 --port 6000 --device sdc --weight 100
Device d3r1z4-10.0.0.52:6000R10.0.0.52:6000/sdc_"" with 100.0 weight got id 3
```

Verify the ring contents: 验证

```txt
# swift-ring-builder object.builder
object.builder, build version 4
1024 partitions, 3.000000 replicas, 1 regions, 2 zones, 4 devices, 100.00 balance, 0.00 dispersion
The minimum number of hours before a partition can be reassigned is 1
The overload factor is 0.00% (0.000000)
Devices:    id  region  zone      ip address  port  replication ip  replication port      name weight partitions balance meta
             0       1     1       10.0.0.51  6000       10.0.0.51              6000      sdb  100.00          0 -100.00
             1       1     1       10.0.0.51  6000       10.0.0.51              6000      sdc  100.00          0 -100.00
             2       1     2       10.0.0.52  6000       10.0.0.52              6000      sdb  100.00          0 -100.00
             3       1     2       10.0.0.52  6000       10.0.0.52              6000      sdc  100.00          0 -100.00
```

Rebalance the ring:令牌环再平衡
---

```txt
# swift-ring-builder object.builder rebalance
```
Reassigned 1024 (100.00%) partitions. Balance is now 0.00.  Dispersion is now 0.00
Distribute ring configuration files¶


Copy the <code>account.ring.gz, container.ring.gz, and object.ring.gz</code> files to the <code>/etc/swift</code> directory on each storage node and any additional nodes running the proxy service.
将这三个压缩文件放到所有存储节点的<code>/etc/swift/</code>目录


Finalize installation for Red Hat Enterprise Linux and CentOS
===

Note : Default configuration files vary by distribution. You might need to add these sections and options rather than modifying existing sections and options. Also, an ellipsis (...) in the configuration snippets indicates potential default configuration options that you should retain.

This section applies to Red Hat Enterprise Linux 7 and CentOS 7.

Obtain the <code>/etc/swift/swift.conf</code> file from the Object Storage source repository:

```txt
# curl -o /etc/swift/swift.conf \
  https://git.openstack.org/cgit/openstack/swift/plain/etc/swift.conf-sample?h=stable/newton
```
Edit the <code>/etc/swift/swift.conf</code> file and complete the following actions:

In the <code>[swift-hash]</code> section, configure the hash path prefix and suffix for your environment.

```txt
[swift-hash]
...
swift_hash_path_suffix = HASH_PATH_SUFFIX
swift_hash_path_prefix = HASH_PATH_PREFIX
```
Replace <code>HASH_PATH_PREFIX</code> and <code>HASH_PATH_SUFFIX</code> with unique values.

Warning : Keep these values secret and do not change or lose them.

In the <code>[storage-policy:0]</code> section, configure the default storage policy:

```txt
[storage-policy:0]
...
name = Policy-0
default = yes
```
Copy the <code>swift.conf</code> file to the /etc/swift directory on each storage node and any additional nodes running the proxy service.

On all nodes, ensure proper ownership of the configuration directory:

```txt
# chown -R root:swift /etc/swift
```
On the controller node and any other nodes running the proxy service, start the Object Storage proxy service including its dependencies and configure them to start 

when the system boots:

```txt
# systemctl enable openstack-swift-proxy.service memcached.service
# systemctl start openstack-swift-proxy.service memcached.service
```

  
这部分的在存储Storage节点上
===

On the storage nodes, start the Object Storage services and configure them to start when the system boots:

```
# systemctl enable openstack-swift-account.service openstack-swift-account-auditor.service \
  openstack-swift-account-reaper.service openstack-swift-account-replicator.service
# systemctl start openstack-swift-account.service openstack-swift-account-auditor.service \
  openstack-swift-account-reaper.service openstack-swift-account-replicator.service
# systemctl enable openstack-swift-container.service \
  openstack-swift-container-auditor.service openstack-swift-container-replicator.service \
  openstack-swift-container-updater.service
# systemctl start openstack-swift-container.service \
  openstack-swift-container-auditor.service openstack-swift-container-replicator.service \
  openstack-swift-container-updater.service
# systemctl enable openstack-swift-object.service openstack-swift-object-auditor.service \
  openstack-swift-object-replicator.service openstack-swift-object-updater.service
# systemctl start openstack-swift-object.service openstack-swift-object-auditor.service \
  openstack-swift-object-replicator.service openstack-swift-object-updater.service
```

这部分的在控制Controller节点上
===

Verify operation
---

Verify operation of the Object Storage service.

Note : Perform these steps on the controller node.

Warning : If you are using Red Hat Enterprise Linux 7 or CentOS 7 and one or more of these steps do not work, check the /var/log/audit/audit.log file for SELinux messages indicating denial of actions for the swift processes. If present, change the security context of the /srv/node directory to the lowest security level (s0) for the swift_data_t type, object_r role and the system_u user:

```txt
# chcon -R system_u:object_r:swift_data_t:s0 /srv/node
```
Source the demo credentials:

$ . demo-openrc
Show the service status:

```txt
$ swift stat
                        Account: AUTH_ed0b60bf607743088218b0a533d5943f
                     Containers: 0
                        Objects: 0
                          Bytes: 0
    X-Account-Project-Domain-Id: default
                    X-Timestamp: 1444143887.71539
                     X-Trans-Id: tx1396aeaf17254e94beb34-0056143bde
         X-Openstack-Request-Id: tx1396aeaf17254e94beb34-0056143bde
                   Content-Type: text/plain; charset=utf-8
                  Accept-Ranges: bytes
```
Create container1 container:

```txt
$ openstack container create container1
+---------------------------------------+------------+------------------------------------+
| account                               | container  | x-trans-id                         |
+---------------------------------------+------------+------------------------------------+
| AUTH_ed0b60bf607743088218b0a533d5943f | container1 | tx8c4034dc306c44dd8cd68-0056f00a4a |
+---------------------------------------+------------+------------------------------------+
```
Upload a test file to the container1 container:

```txt
$ openstack object create container1 FILE
+--------+------------+----------------------------------+
| object | container  | etag                             |
+--------+------------+----------------------------------+
| FILE   | container1 | ee1eca47dc88f4879d8a229cc70a07c6 |
+--------+------------+----------------------------------+
```
Replace <code>FILE</code> with the name of a local file to upload to the container1 container.

List files in the container1 container:

```txt
$ openstack object list container1
+------+
| Name |
+------+
| FILE |
+------+
```
Download a test file from the container1 container:

```txt
$ openstack object save container1 FILE
```
Replace <code>FILE</code> with the name of the file uploaded to the container1 container.

Note ：This command provides no output.

Your OpenStack environment now includes Object Storage.

参考资料：
---

- https://docs.openstack.org/project-install-guide/object-storage/ocata/get_started.html
- https://docs.openstack.org/developer/swift/
- https://xieyugui.wordpress.com/2016/08/11/%e6%90%ad%e5%bb%baopenstack-swift-%e4%ba%91%e5%ad%98%e5%82%a8%e4%ba%8c/
- OpenStack swift安装部署过程图文教程 http://www.server110.com/openstack/201511/11385.html