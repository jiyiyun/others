openstack安装1、Environment
===

1 、创建OpenSSL 密码或者token(官网给的选用，一般用自己设定的密码)

```txt
#openssl rand -hex 10
随机创建一个16进制的十位密码
```
2 、OpenStack要用的密码解释

he following table provides a list of services that require passwords and their associated references in the guide.

```txt
Password name                           Description
Database password (no variable used)    Root password for the database
ADMIN_PASS                              Password of user admin
CINDER_DBPASS                           Database password for the Block Storage service
CINDER_PASS                             Password of Block Storage service user cinder
DASH_DBPASS                             Database password for the Dashboard
DEMO_PASS                               Password of user demo
GLANCE_DBPASS                           Database password for Image service
GLANCE_PASS                             Password of Image service user glance
KEYSTONE_DBPASS                         Database password of Identity service
NEUTRON_DBPASS                          Database password for the Networking service
NEUTRON_PASS                            Password of Networking service user neutron
NOVA_DBPASS                             Database password for Compute service
NOVA_PASS                               Password of Compute service user nova
RABBIT_PASS                             Password of user guest of RabbitMQ
```
Host networking
===

Controller node 
--

Configure name resolution¶

1 . Set the hostname of the node to controller.

2 . Edit the <code>/etc/hosts</code> file to contain the following:

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
Compute node
---

Configure name resolution¶

1 . Set the hostname of the node to compute1.

2 . Edit the <code>/etc/hosts</code> file to contain the following:

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

Block storage node (Optional选用)
---

Configure name resolution¶

1 . Set the hostname of the node to block1.

2 . Edit the <code>/etc/hosts</code> file to contain the following:

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
Verify connectivity
---

1 . From the controller node, test access to the Internet: 测试controller能不能上外网，因为后面要下载OpenStack所需软件

```txt
# ping -c 4 openstack.org

PING openstack.org (174.143.194.225) 56(84) bytes of data.
64 bytes from 174.143.194.225: icmp_seq=1 ttl=54 time=18.3 ms
64 bytes from 174.143.194.225: icmp_seq=2 ttl=54 time=17.5 ms
64 bytes from 174.143.194.225: icmp_seq=3 ttl=54 time=17.5 ms
64 bytes from 174.143.194.225: icmp_seq=4 ttl=54 time=17.4 ms

--- openstack.org ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 3022ms
rtt min/avg/max/mdev = 17.489/17.715/18.346/0.364 ms
```
2 . From the controller node, test access to the management interface on the compute node:相互之间的连接

```txt
# ping -c 4 compute1

PING compute1 (10.0.0.31) 56(84) bytes of data.
64 bytes from compute1 (10.0.0.31): icmp_seq=1 ttl=64 time=0.263 ms
64 bytes from compute1 (10.0.0.31): icmp_seq=2 ttl=64 time=0.202 ms
64 bytes from compute1 (10.0.0.31): icmp_seq=3 ttl=64 time=0.203 ms
64 bytes from compute1 (10.0.0.31): icmp_seq=4 ttl=64 time=0.202 ms

--- compute1 ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 3000ms
rtt min/avg/max/mdev = 0.202/0.217/0.263/0.030 ms
```
3 . From the compute node, test access to the Internet:

```txt
# ping -c 4 openstack.org

PING openstack.org (174.143.194.225) 56(84) bytes of data.
64 bytes from 174.143.194.225: icmp_seq=1 ttl=54 time=18.3 ms
64 bytes from 174.143.194.225: icmp_seq=2 ttl=54 time=17.5 ms
64 bytes from 174.143.194.225: icmp_seq=3 ttl=54 time=17.5 ms
64 bytes from 174.143.194.225: icmp_seq=4 ttl=54 time=17.4 ms

--- openstack.org ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 3022ms
rtt min/avg/max/mdev = 17.489/17.715/18.346/0.364 ms
```
4 . From the compute node, test access to the management interface on the controller node:

```txt
# ping -c 4 controller

PING controller (10.0.0.11) 56(84) bytes of data.
64 bytes from controller (10.0.0.11): icmp_seq=1 ttl=64 time=0.263 ms
64 bytes from controller (10.0.0.11): icmp_seq=2 ttl=64 time=0.202 ms
64 bytes from controller (10.0.0.11): icmp_seq=3 ttl=64 time=0.203 ms
64 bytes from controller (10.0.0.11): icmp_seq=4 ttl=64 time=0.202 ms

--- controller ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 3000ms
rtt min/avg/max/mdev = 0.202/0.217/0.263/0.030 ms
```
Network Time Protocol (NTP所有服务器之间时间同步，如果内网有专门的NTP服务器就不用搭建了，所有的时间同步到NTP服务器上)
---

```txt
[root@controller ~]# systemctl start ntpd
[root@controller ~]# systemctl enable ntpd
```
Created symlink from /etc/systemd/system/multi-user.target.wants/ntpd.service to /usr/lib/systemd/system/ntpd.service.

You should install Chrony, an implementation of NTP, to properly synchronize services among nodes. We recommend that you configure the controller node to reference more accurate (lower stratum) servers and other nodes to reference the controller node.

Controller node
---

Install and configure components¶

1 、Install the packages:

```txt
# yum install chrony
```
2 、Edit the <code>/etc/chrony.conf</code> file and add, change, or remove these keys as necessary for your environment:

```txt
server NTP_SERVER iburst
```
Replace <code>NTP_SERVER</code> with the hostname or IP address of a suitable more accurate (lower stratum) NTP server. The configuration supports multiple server keys.

3 、 To enable other nodes to connect to the chrony daemon on the controller node, add this key to the <code>/etc/chrony.conf</code> file:

```txt
allow 10.0.0.0/24
```
If necessary, replace 10.0.0.0/24 with a description of your subnet.

4 、 Start the NTP service and configure it to start when the system boots:

```txt
# systemctl enable chronyd.service
# systemctl start chronyd.service
```
Other nodes
---

Install and configure components¶

1 . Install the packages:

```txt
# yum install chrony
```
2 . Edit the <code>/etc/chrony.conf</code> file and comment out or remove all but one server key. Change it to reference the controller node:

```txt
server controller iburst
```
3 . Start the NTP service and configure it to start when the system boots:

```txt
# systemctl enable chronyd.service
# systemctl start chronyd.service
```
Verify operation 验证chrony
---

We recommend that you verify NTP synchronization before proceeding further. Some nodes, particularly those that reference the controller node, can take several minutes to synchronize.

Run this command on the controller node:

```txt
# chronyc sources

  210 Number of sources = 2
  MS Name/IP address         Stratum Poll Reach LastRx Last sample
  ===============================================================================
  ^- 192.0.2.11                    2   7    12   137  -2814us[-3000us] +/-   43ms
  ^* 192.0.2.12                    2   6   177    46    +17us[  -23us] +/-   68ms
```
Contents in the Name/IP address column should indicate the hostname or IP address of one or more NTP servers. Contents in the MS column should indicate * for the server to which the NTP service is currently synchronized.

Run the same command on all other nodes:

```txt
# chronyc sources

  210 Number of sources = 1
  MS Name/IP address         Stratum Poll Reach LastRx Last sample
  ===============================================================================
  ^* controller                    3    9   377   421    +15us[  -87us] +/-   15ms
```
Contents in the Name/IP address column should indicate the hostname of the controller node.

OpenStack packages
===

红帽7系统需要添加的

1 . When using RHEL, it is assumed that you have registered your system using Red Hat Subscription Management and that you have the rhel-7-server-rpms repository enabled by default.

For more information on registering the system, see the Red Hat Enterprise Linux 7 System Administrator’s Guide.

2 . In addition to rhel-7-server-rpms, you also need to have the rhel-7-server-optional-rpms, rhel-7-server-extras-rpms, and rhel-7-server-rh-common-rpms repositories enabled:

```txt
# subscription-manager repos --enable=rhel-7-server-optional-rpms \
  --enable=rhel-7-server-extras-rpms --enable=rhel-7-server-rh-common-rpms
```

Enable the OpenStack repository¶ 添加OpenStack更新源
---
On CentOS, the extras repository provides the RPM that enables the OpenStack repository. CentOS includes the extras repository by default, so you can simply install the package to enable the OpenStack repository.

```txt
# yum install centos-release-openstack-ocata
```
On RHEL, download and install the RDO repository RPM to enable the OpenStack repository.

对于红帽系统，还需要再添加RDO更新源这样才能正常使用OpenStack更新源
```txt
# yum install https://rdoproject.org/repos/rdo-release.rpm
```

Finalize the installation¶
---

1 . Upgrade the packages on all nodes:

```txt
# yum upgrade
``` 
 Note

If the upgrade process includes a new kernel, reboot your host to activate it.

2 . Install the OpenStack client:

```txt
# yum install python-openstackclient
```
3 . RHEL and CentOS enable SELinux by default. Install the openstack-selinux package to automatically manage security policies for OpenStack services:

```txt
# yum install openstack-selinux
```

SQL database
---

Install and configure components¶

1 . Install the packages:

```txt
# yum install mariadb mariadb-server python2-PyMySQL
```
2 . Create and edit the <code>/etc/my.cnf.d/openstack.cnf</code> file and complete the following actions:

Create a <code>[mysqld]</code> section, and set the bind-address key to the management IP address of the controller node to enable access by other nodes via the management network. Set additional keys to enable useful options and the UTF-8 character set:

```txt
[mysqld]
bind-address = 10.0.0.11

default-storage-engine = innodb
innodb_file_per_table = on
max_connections = 4096
collation-server = utf8_general_ci
character-set-server = utf8
```
备注：bind-address可以不写

Finalize installation¶
---

1 . Start the database service and configure it to start when the system boots:

```txt
# systemctl enable mariadb.service
# systemctl start mariadb.service
```
2 . Secure the database service by running the mysql_secure_installation script. In particular, choose a suitable password for the database root account:

```
# mysql_secure_installation
```
Message queue (在控制节点上安装)
---

The message queue runs on the controller node.

Install and configure components¶

1 . Install the package:

```txt
# yum install rabbitmq-server
```
2 . Start the message queue service and configure it to start when the system boots:

```txt
# systemctl enable rabbitmq-server.service
# systemctl start rabbitmq-server.service
```
3 . Add the openstack user:

```txt
# rabbitmqctl add_user openstack RABBIT_PASS
```
Creating user "openstack" ...

4 . Replace <code>RABBIT_PASS</code> with a suitable password.

Permit configuration, write, and read access for the openstack user:

```txt
# rabbitmqctl set_permissions openstack ".*" ".*" ".*"
```
Setting permissions for user "openstack" in vhost "/" ...

Memcached (网络术语：缓存;集群) (通常在控制节点上)
---

The Identity service authentication mechanism for services uses Memcached to cache tokens. The memcached service typically runs on the controller node. For production deployments, we recommend enabling a combination of firewalling, authentication, and encryption to secure it.

Install and configure components¶

1 . Install the packages:

```txt
# yum install memcached python-memcached
```
Edit the <code>/etc/sysconfig/memcached</code> file and configure the service to use the management IP address of the controller node. This is to enable access by other nodes via the management network:

```txt
10.0.0.11
```
Finalize installation¶

Start the Memcached service and configure it to start when the system boots:

```txt
# systemctl enable memcached.service
# systemctl start memcached.service
```