Compute service(Nova)
===

Compute service overview

Contents

Use OpenStack Compute to host and manage cloud computing systems. OpenStack Compute is a major part of an Infrastructure-as-a-Service (IaaS) system. The main modules are implemented in Python.

OpenStack Compute interacts with OpenStack Identity for authentication; OpenStack Image service for disk and server images; and OpenStack Dashboard for the user and administrative interface. Image access is limited by projects, and by users; quotas are limited per project (the number of instances, for example). OpenStack Compute can scale horizontally on standard hardware, and download images to launch instances.

OpenStack Compute consists of the following areas and their components:

nova-api service
---

    Accepts and responds to end user compute API calls. The service supports the OpenStack Compute API, the Amazon EC2 API, and a special Admin API for privileged users to perform administrative actions. It enforces some policies and initiates most orchestration activities, such as running an instance.

nova-api-metadata service
---

    Accepts metadata requests from instances. The nova-api-metadata service is generally used when you run in multi-host mode with nova-network installations. For details, see Metadata service in the OpenStack Administrator Guide.

nova-compute service
---

A worker daemon that creates and terminates virtual machine instances through hypervisor APIs. For example:
        XenAPI for XenServer/XCP
        libvirt for KVM or QEMU
        VMwareAPI for VMware
Processing is fairly complex. Basically, the daemon accepts actions from the queue and performs a series of system commands such as launching a KVM instance and updating its state in the database.

nova-scheduler service
---

    Takes a virtual machine instance request from the queue and determines on which compute server host it runs.

nova-conductor module
---

    Mediates interactions between the nova-compute service and the database. It eliminates direct accesses to the cloud database made by the nova-compute service. The nova-conductor module scales horizontally. However, do not deploy it on nodes where the nova-compute service runs. For more information, see Configuration Reference Guide.

nova-cert module
---

    A server daemon that serves the Nova Cert service for X509 certificates. Used to generate certificates for euca-bundle-image. Only needed for the EC2 API.

nova-consoleauth daemon
---

    Authorizes tokens for users that console proxies provide. See nova-novncproxy and nova-xvpvncproxy. This service must be running for console proxies to work. You can run proxies of either type against a single nova-consoleauth service in a cluster configuration. For information, see About nova-consoleauth.

nova-novncproxy daemon
---

    Provides a proxy for accessing running instances through a VNC connection. Supports browser-based novnc clients.

nova-spicehtml5proxy daemon
---

    Provides a proxy for accessing running instances through a SPICE connection. Supports browser-based HTML5 client.

nova-xvpvncproxy daemon
---

    Provides a proxy for accessing running instances through a VNC connection. Supports an OpenStack-specific Java client.

The queue
---

    A central hub for passing messages between daemons. Usually implemented with RabbitMQ, also can be implemented with another AMQP message queue, such as ZeroMQ.

SQL database
---

Stores most build-time and run-time states for a cloud infrastructure, including:
        Available instance types
        Instances in use
        Available networks
        Projects
Theoretically, OpenStack Compute can support any database that SQLAlchemy supports. Common databases are SQLite3 for test and development work, MySQL, MariaDB, and PostgreSQL.

Install and configure controller node
===

Prerequisites¶

Before you install and configure the Compute service, you must create databases, service credentials, and API endpoints.

1 .  To create the databases, complete these steps:

        Use the database access client to connect to the database server as the root user:

```txt
        $ mysql -u root -p
```
        Create the nova_api and nova databases:
        Grant proper access to the databases:

```txt
        MariaDB [(none)]> CREATE DATABASE nova_api;
        MariaDB [(none)]> GRANT ALL PRIVILEGES ON nova_api.* TO 'nova'@'localhost' IDENTIFIED BY 'NOVA_DBPASS';
        MariaDB [(none)]> GRANT ALL PRIVILEGES ON nova_api.* TO 'nova'@'controller' IDENTIFIED BY 'NOVA_DBPASS';
        MariaDB [(none)]> GRANT ALL PRIVILEGES ON nova_api.* TO 'nova'@'%' IDENTIFIED BY 'NOVA_DBPASS';
        MariaDB [(none)]> FLUSH PRIVILEGES; 
        
        MariaDB [(none)]> CREATE DATABASE nova;
        MariaDB [(none)]> GRANT ALL PRIVILEGES ON nova.* TO 'nova'@'localhost' IDENTIFIED BY 'NOVA_DBPASS';
        MariaDB [(none)]> GRANT ALL PRIVILEGES ON nova.* TO 'nova'@'controller' IDENTIFIED BY 'NOVA_DBPASS';
        MariaDB [(none)]> GRANT ALL PRIVILEGES ON nova.* TO 'nova'@'%' IDENTIFIED BY 'NOVA_DBPASS';
        MariaDB [(none)]> FLUSH PRIVILEGES; 
```
Replace <code>NOVA_DBPASS</code> with a suitable password.

Exit the database access client.

2 .   Source the admin credentials to gain access to admin-only CLI commands:

```txt
    $ . admin-openrc
```
3 . To create the service credentials, complete these steps:

Create the nova user:

```txt
    $ openstack user create --domain default \
      --password-prompt nova

    User Password:
Repeat User Password:
+---------------------+----------------------------------+
| Field               | Value                            |
+---------------------+----------------------------------+
| domain_id           | default                          |
| enabled             | True                             |
| id                  | a9bc0fccdcca4b0a88c0e114474bb008 |
| name                | nova                             |
| options             | {}                               |
| password_expires_at | None                             |
+---------------------+----------------------------------+
```

Add the admin role to the nova user:

```txt
    $ openstack role add --project service --user nova admin
```

Create the nova service entity:

```txt
    $ openstack service create --name nova \
      --description "OpenStack Compute" compute
+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description | OpenStack Compute                |
| enabled     | True                             |
| id          | 4bb92cbc5559404d9934480b173d84c3 |
| name        | nova                             |
| type        | compute                          |
+-------------+----------------------------------+
```
4 . Create the Compute service API endpoints:

```txt
$ openstack endpoint create --region RegionOne \
  compute public http://controller:8774/v2.1/%\(tenant_id\)s

+--------------+-------------------------------------------+
| Field        | Value                                     |
+--------------+-------------------------------------------+
| enabled      | True                                      |
| id           | aff2ab740990431d9994f9180ca8ab33          |
| interface    | public                                    |
| region       | RegionOne                                 |
| region_id    | RegionOne                                 |
| service_id   | 4bb92cbc5559404d9934480b173d84c3          |
| service_name | nova                                      |
| service_type | compute                                   |
| url          | http://controller:8774/v2.1/%(tenant_id)s |
+--------------+-------------------------------------------+

$ openstack endpoint create --region RegionOne \
  compute internal http://controller:8774/v2.1/%\(tenant_id\)s

+--------------+-------------------------------------------+
| Field        | Value                                     |
+--------------+-------------------------------------------+
| enabled      | True                                      |
| id           | f45bab3365cf4feeb29f6b476d56d558          |
| interface    | internal                                  |
| region       | RegionOne                                 |
| region_id    | RegionOne                                 |
| service_id   | 4bb92cbc5559404d9934480b173d84c3          |
| service_name | nova                                      |
| service_type | compute                                   |
| url          | http://controller:8774/v2.1/%(tenant_id)s |
+--------------+-------------------------------------------+

$ openstack endpoint create --region RegionOne \
  compute admin http://controller:8774/v2.1/%\(tenant_id\)s

+--------------+-------------------------------------------+
| Field        | Value                                     |
+--------------+-------------------------------------------+
| enabled      | True                                      |
| id           | 4c714b234355493d93829b3bf7b94b91          |
| interface    | admin                                     |
| region       | RegionOne                                 |
| region_id    | RegionOne                                 |
| service_id   | 4bb92cbc5559404d9934480b173d84c3          |
| service_name | nova                                      |
| service_type | compute                                   |
| url          | http://controller:8774/v2.1/%(tenant_id)s |
+--------------+-------------------------------------------+
```

Install and configure components¶
---

1 .  Install the packages:

```txt
    # yum install openstack-nova-api openstack-nova-conductor \
      openstack-nova-console openstack-nova-novncproxy \
      openstack-nova-scheduler
```
2 .   Edit the <code>/etc/nova/nova.conf</code> file and complete the following actions:

In the <code>[DEFAULT]</code> section, enable only the compute and metadata APIs:

```txt
        [DEFAULT]
        # ...
        enabled_apis = osapi_compute,metadata
```
        In the [api_database] and [database] sections, configure database access:

```txt
        [api_database]
        # ...
        connection = mysql+pymysql://nova:NOVA_DBPASS@controller/nova_api

        [database]
        # ...
        connection = mysql+pymysql://nova:NOVA_DBPASS@controller/nova
```
雷区警示：

[api_database]对应的是nova_api数据库，

[database] 参数对应的是nova数据库

Replace <code>NOVA_DBPASS</code> with the password you chose for the Compute databases.

In the <code>[DEFAULT]</code> section, configure RabbitMQ message queue access:

```txt
        [DEFAULT]
        # ...
        transport_url = rabbit://openstack:RABBIT_PASS@controller
```
Replace <code>RABBIT_PASS</code> with the password you chose for the openstack account in RabbitMQ.

In the <code>[api]</code> and <code>[keystone_authtoken]</code> sections, configure Identity service access:

 ```txt
        [api]
        # ...
        auth_strategy = keystone

        [keystone_authtoken]
        # ...
        auth_uri = http://controller:5000
        auth_url = http://controller:35357
        memcached_servers = controller:11211
        auth_type = password
        project_domain_name = default
        user_domain_name = default
        project_name = service
        username = nova
        password = NOVA_PASS
```
Replace <code>NOVA_PASS</code> with the password you chose for the nova user in the Identity service.

In the <code>[DEFAULT]</code> section, configure the my_ip option to use the management interface IP address of the controller node:

```txt
        [DEFAULT]
        # ...
        my_ip = 10.0.0.11
```
In the <code>[DEFAULT]</code> section, enable support for the Networking service:

```txt
    [DEFAULT]
    # ...
    use_neutron = True
    firewall_driver = nova.virt.firewall.NoopFirewallDriver
```
In the <code>[vnc]</code> section, configure the VNC proxy to use the management interface IP address of the controller node:

```txt
    [vnc]
    enabled = true
    # ...
    vncserver_listen = $my_ip
    vncserver_proxyclient_address = $my_ip
```
In the <code>[glance]</code> section, configure the location of the Image service API:

```txt
    [glance]
    # ...
    api_servers = http://controller:9292
```
In the <code>[oslo_concurrency]</code> section, configure the lock path:

```txt
    [oslo_concurrency]
    # ...
    lock_path = /var/lib/nova/tmp
```
Populate the Compute databases:

```
    # su -s /bin/sh -c "nova-manage api_db sync" nova
    # su -s /bin/sh -c "nova-manage db sync" nova
```

```txt
[root@controller openstack]# su -s /bin/sh -c "nova-manage api_db sync" nova
[root@controller openstack]# su -s /bin/sh -c "nova-manage db sync" nova
WARNING: cell0 mapping not found - not syncing cell0.
An error has occurred:
Traceback (most recent call last):
  File "/usr/lib/python2.7/site-packages/nova/cmd/manage.py", line 1594, in main
    ret = fn(*fn_args, **fn_kwargs)
  File "/usr/lib/python2.7/site-packages/nova/cmd/manage.py", line 644, in sync
    return migration.db_sync(version)
  File "/usr/lib/python2.7/site-packages/nova/db/migration.py", line 26, in db_sync
    return IMPL.db_sync(version=version, database=database, context=context)
  File "/usr/lib/python2.7/site-packages/nova/db/sqlalchemy/migration.py", line 53, in db_sync
    current_version = db_version(database, context=context)
  File "/usr/lib/python2.7/site-packages/nova/db/sqlalchemy/migration.py", line 84, in db_version
    _("Upgrade DB using Essex release first."))
NovaException: Upgrade DB using Essex release first.

[root@controller openstack]# 
```
检查是否将配置已经写入数据库

```txt
[root@compute1 ~]# mysql -h controller -u nova -p
Enter password: 
Welcome to the MariaDB monitor.  Commands end with ; or \g.
Your MariaDB connection id is 14
Server version: 10.1.20-MariaDB MariaDB Server

Copyright (c) 2000, 2016, Oracle, MariaDB Corporation Ab and others.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

MariaDB [(none)]> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| nova               |
| nova_api           |
| test               |
+--------------------+
4 rows in set (0.00 sec)

MariaDB [(none)]> use nova_api
Reading table information for completion of table and column names
You can turn off this feature to get a quicker startup with -A

Database changed
MariaDB [nova_api]> show tables;
+------------------------------+
| Tables_in_nova_api           |
+------------------------------+
| aggregate_hosts              |
| aggregate_metadata           |
| aggregates                   |
| allocations                  |
| build_requests               |
| cell_mappings                |
| flavor_extra_specs           |
| flavor_projects              |
| flavors                      |
| host_mappings                |
| instance_group_member        |
| instance_group_policy        |
| instance_groups              |
| instance_mappings            |
| inventories                  |
| key_pairs                    |
| migrate_version              |
| placement_aggregates         |
| project_user_quotas          |
| quota_classes                |
| quota_usages                 |
| quotas                       |
| request_specs                |
| reservations                 |
| resource_classes             |
| resource_provider_aggregates |
| resource_providers           |
+------------------------------+
27 rows in set (0.00 sec)

MariaDB [nova_api]> use nova
Database changed
MariaDB [nova]> show tables;
+--------------------------------------------+
| Tables_in_nova                             |
+--------------------------------------------+
| agent_builds                               |
| aggregate_hosts                            |
| aggregate_metadata                         |
| aggregates                                 |
| allocations                                |
| block_device_mapping                       |
| bw_usage_cache                             |
| cells                                      |
| certificates                               |
| compute_nodes                              |
| console_auth_tokens                        |
| console_pools                              |
| consoles                                   
| task_log                                   |
| virtual_interfaces                         |
| volume_id_mappings                         |
| volume_usage_cache                         |
+--------------------------------------------+
110 rows in set (0.00 sec)
MariaDB [nova]> 
```
Finalize installation¶
---

    Start the Compute services and configure them to start when the system boots:

```txt
    # systemctl enable openstack-nova-api.service \
      openstack-nova-consoleauth.service openstack-nova-scheduler.service \
      openstack-nova-conductor.service openstack-nova-novncproxy.service
    # systemctl start openstack-nova-api.service \
      openstack-nova-consoleauth.service openstack-nova-scheduler.service \
      openstack-nova-conductor.service openstack-nova-novncproxy.service
```

Install and configure a compute node 这个是在运算节点上运行
===

Install and configure components¶
---

1 .  Install the packages:

```txt
    # yum install openstack-nova-compute
```
2 .  Edit the <code>/etc/nova/nova.conf</code> file and complete the following actions:

In the <code>[DEFAULT]</code> section, enable only the compute and metadata APIs:

```txt
        [DEFAULT]
        # ...
        enabled_apis = osapi_compute,metadata
```
In the <code>[DEFAULT]</code> section, configure RabbitMQ message queue access:

```txt
        [DEFAULT]
        # ...
        transport_url = rabbit://openstack:RABBIT_PASS@controller
```
Replace <code>RABBIT_PASS</code> with the password you chose for the openstack account in RabbitMQ.

In the <code>[api]</code> and <code>[keystone_authtoken]</code> sections, configure Identity service access:

```txt
        [api]
        # ...
        auth_strategy = keystone

        [keystone_authtoken]
        # ...
        auth_uri = http://controller:5000
        auth_url = http://controller:35357
        memcached_servers = controller:11211
        auth_type = password
        project_domain_name = default
        user_domain_name = default
        project_name = service
        username = nova
        password = NOVA_PASS
```
Replace <code>NOVA_PASS</code> with the password you chose for the nova user in the Identity service.

In the <code>[DEFAULT]</code> section, configure the my_ip option:

```txt
[DEFAULT]
# ...
my_ip = MANAGEMENT_INTERFACE_IP_ADDRESS
```
Replace <code>MANAGEMENT_INTERFACE_IP_ADDRESS</code> with the IP address of the management network interface on your compute node, typically 10.0.0.31 for the first node in the example architecture.

In the <code>[DEFAULT]</code> section, enable support for the Networking service:

```txt
[DEFAULT]
# ...
use_neutron = True
firewall_driver = nova.virt.firewall.NoopFirewallDriver
```
In the <code>[vnc]</code> section, enable and configure remote console access:

```txt
[vnc]
# ...
enabled = True
vncserver_listen = 0.0.0.0
vncserver_proxyclient_address = $my_ip
novncproxy_base_url = http://controller:6080/vnc_auto.html
```
The server component listens on all IP addresses and the proxy component only listens on the management interface IP address of the compute node. The base URL indicates the location where you can use a web browser to access remote consoles of instances on this compute node.

 Note

If the web browser to access remote consoles resides on a host that cannot resolve the controller hostname, you must replace controller with the management interface IP address of the controller node.

提示：这里要把controller 换成IP 这样远端才能通过浏览器访问

In the <code>[glance]</code> section, configure the location of the Image service API:

```txt
    [glance]
    # ...
    api_servers = http://controller:9292
```
In the <code>[oslo_concurrency]</code> section, configure the lock path:

```txt
    [oslo_concurrency]
    # ...
    lock_path = /var/lib/nova/tmp
```

Finalize installation¶
---

Determine whether your compute node supports hardware acceleration for virtual machines:

```txt
    $ egrep -c '(vmx|svm)' /proc/cpuinfo
```
If this command returns a value of one or greater, your compute node supports hardware acceleration which typically requires no additional configuration.

如果返回值是1或者更大的数字，则系统硬件满足要求，不需要额外配置

If this command returns a value of zero, your compute node does not support hardware acceleration and you must configure libvirt to use QEMU instead of KVM.

如果返回值是0，则需要配置livirt

Edit the <code>[libvirt]</code> section in the <code>/etc/nova/nova.conf</code> file as follows:

```txt
        [libvirt]
        # ...
        virt_type = qemu
```
Start the Compute service including its dependencies and configure them to start automatically when the system boots:

```txt
    # systemctl enable libvirtd.service openstack-nova-compute.service
    # systemctl start libvirtd.service openstack-nova-compute.service
```
Note

If the nova-compute service fails to start, check <code>/var/log/nova/nova-compute.log</code>. The error message AMQP server on controller:5672 is unreachable likely indicates that the firewall on the controller node is preventing access to port 5672. Configure the firewall to open port 5672 on the controller node and restart nova-compute service on the compute node.

Verify operation :Verify operation of the Compute service.
---

1 . Source the admin credentials to gain access to admin-only CLI commands:

```txt
$ . admin-openrc
```
List service components to verify successful launch and registration of each process:

```txt
$ openstack compute service list

+----+--------------------+------------+----------+---------+-------+----------------------------+
| Id | Binary             | Host       | Zone     | Status  | State | Updated At                 |
+----+--------------------+------------+----------+---------+-------+----------------------------+
|  1 | nova-consoleauth   | controller | internal | enabled | up    | 2016-02-09T23:11:15.000000 |
|  2 | nova-scheduler     | controller | internal | enabled | up    | 2016-02-09T23:11:15.000000 |
|  3 | nova-conductor     | controller | internal | enabled | up    | 2016-02-09T23:11:16.000000 |
|  4 | nova-compute       | compute1   | nova     | enabled | up    | 2016-02-09T23:11:20.000000 |
+----+--------------------+------------+----------+---------+-------+----------------------------+
```
3 . List API endpoints in the Identity service to verify connectivity with the Identity service:

```txt
$ openstack catalog list

+----------+----------+--------------------------------------------------------------------------+
| Name     | Type     | Endpoints                                                                |
+----------+----------+--------------------------------------------------------------------------+
| keystone | identity | RegionOne                                                                |
|          |          |   public: http://controller:5000/v3/                                     |
|          |          | RegionOne                                                                |
|          |          |   internal: http://controller:5000/v3/                                   |
|          |          | RegionOne                                                                |
|          |          |   admin: http://controller:35357/v3/                                     |
|          |          |                                                                          |
| glance   | image    | RegionOne                                                                |
|          |          |   admin: http://controller:9292                                          |
|          |          | RegionOne                                                                |
|          |          |   public: http://controller:9292                                         |
|          |          | RegionOne                                                                |
|          |          |   internal: http://controller:9292                                       |
|          |          |                                                                          |
| nova     | compute  | RegionOne                                                                |
|          |          |   admin: http://controller:8774/v2.1/3825a95e0e7f4e2fbfdbd65cc836ecfb    |
|          |          | RegionOne                                                                |
|          |          |   internal: http://controller:8774/v2.1/3825a95e0e7f4e2fbfdbd65cc836ecfb |
|          |          | RegionOne                                                                |
|          |          |   public: http://controller:8774/v2.1/3825a95e0e7f4e2fbfdbd65cc836ecfb   |
|          |          |                                                                          |
+----------+----------+--------------------------------------------------------------------------+
```
4 . List images in the Image service to verify connectivity with the Image service:

```txt
$ openstack image list
+--------------------------------------+-------------+-------------+
| ID                                   | Name        | Status      |
+--------------------------------------+-------------+-------------+
| 9a76d9f9-9620-4f2e-8c69-6c5691fae163 | cirros      | active      |
+--------------------------------------+-------------+-------------+
```