The Block Storage service (cinder) provides block storage devices to guest instances. The method in which the storage is provisioned and consumed is determined by the Block Storage driver, or drivers in the case of a multi-backend configuration. There are a variety of drivers that are available: NAS/SAN, NFS, iSCSI, Ceph, and more.

The Block Storage API and scheduler services typically run on the controller nodes. Depending upon the drivers used, the volume service can run on controller nodes, compute nodes, or standalone storage nodes.

For more information, see the Configuration Reference.

Block Storage service overview

The OpenStack Block Storage service (cinder) adds persistent storage to a virtual machine. Block Storage provides an infrastructure for managing volumes, and interacts with OpenStack Compute to provide volumes for instances. The service also enables management of volume snapshots, and volume types.

The Block Storage service consists of the following components:

cinder-api
---

Accepts API requests, and routes them to the cinder-volume for action.

cinder-volume
---

Interacts directly with the Block Storage service, and processes such as the cinder-scheduler. It also interacts with these processes through a message queue. The cinder-volume service responds to read and write requests sent to the Block Storage service to maintain state. It can interact with a variety of storage providers through a driver architecture.

cinder-scheduler daemon
---

Selects the optimal storage provider node on which to create the volume. A similar component to the nova-scheduler.

cinder-backup daemon
---

The cinder-backup service provides backing up volumes of any type to a backup storage provider. Like the cinder-volume service, it can interact with a variety of storage providers through a driver architecture.

Messaging queue
---

Routes information between the Block Storage processes.

Install and configure controller node
---

This section describes how to install and configure the Block Storage service, code-named cinder, on the controller node. This service requires at least one additional storage node that provides volumes to instances.

Prerequisites¶

Before you install and configure the Block Storage service, you must create a database, service credentials, and API endpoints.

1 . To create the database, complete these steps:

Use the database access client to connect to the database server as the root user:

```txt
$ mysql -u root -p
Create the cinder database:

MariaDB [(none)]> CREATE DATABASE cinder;

Grant proper access to the cinder database:

MariaDB [(none)]> GRANT ALL PRIVILEGES ON cinder.* TO 'cinder'@'localhost' IDENTIFIED BY 'CINDER_DBPASS';
MariaDB [(none)]> GRANT ALL PRIVILEGES ON cinder.* TO 'cinder'@'controller' IDENTIFIED BY 'CINDER_DBPASS';
MariaDB [(none)]> GRANT ALL PRIVILEGES ON cinder.* TO 'cinder'@'%' IDENTIFIED BY 'CINDER_DBPASS';
MariaDB [(none)]> FLUSH PRIVILEGES; 
```
Replace <code>CINDER_DBPASS</code> with a suitable password.

Exit the database access client.

2 . Source the admin credentials to gain access to admin-only CLI commands:

```txt
$ . admin-openrc
```
To create the service credentials, complete these steps:

3 . Create a cinder user:

```txt
$ openstack user create --domain default --password-prompt cinder

User Password:
Repeat User Password:
+---------------------+----------------------------------+
| Field               | Value                            |
+---------------------+----------------------------------+
| domain_id           | default                          |
| enabled             | True                             |
| id                  | 9d7e33de3e1a498390353819bc7d245d |
| name                | cinder                           |
| options             | {}                               |
| password_expires_at | None                             |
+---------------------+----------------------------------+
```
Add the admin role to the cinder user:

```txt
$ openstack role add --project service --user cinder admin
```
Note : This command provides no output.

Create the cinder and cinderv2 service entities:

```txt
$ openstack service create --name cinder \
  --description "OpenStack Block Storage" volume

+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description | OpenStack Block Storage          |
| enabled     | True                             |
| id          | ab3bbbef780845a1a283490d281e7fda |
| name        | cinder                           |
| type        | volume                           |
+-------------+----------------------------------+

$ openstack service create --name cinderv2 \
  --description "OpenStack Block Storage" volumev2

+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description | OpenStack Block Storage          |
| enabled     | True                             |
| id          | eb9fd245bdbc414695952e93f29fe3ac |
| name        | cinderv2                         |
| type        | volumev2                         |
+-------------+----------------------------------+

Note: The Block Storage services require two service entities.

4 . Create the Block Storage service API endpoints:

```txt
$ openstack endpoint create --region RegionOne \
  volume public http://controller:8776/v1/%\(tenant_id\)s

  +--------------+-----------------------------------------+
  | Field        | Value                                   |
  +--------------+-----------------------------------------+
  | enabled      | True                                    |
  | id           | 03fa2c90153546c295bf30ca86b1344b        |
  | interface    | public                                  |
  | region       | RegionOne                               |
  | region_id    | RegionOne                               |
  | service_id   | ab3bbbef780845a1a283490d281e7fda        |
  | service_name | cinder                                  |
  | service_type | volume                                  |
  | url          | http://controller:8776/v1/%(tenant_id)s |
  +--------------+-----------------------------------------+
```

```txt
$ openstack endpoint create --region RegionOne \
  volume internal http://controller:8776/v1/%\(tenant_id\)s

  +--------------+-----------------------------------------+
  | Field        | Value                                   |
  +--------------+-----------------------------------------+
  | enabled      | True                                    |
  | id           | 94f684395d1b41068c70e4ecb11364b2        |
  | interface    | internal                                |
  | region       | RegionOne                               |
  | region_id    | RegionOne                               |
  | service_id   | ab3bbbef780845a1a283490d281e7fda        |
  | service_name | cinder                                  |
  | service_type | volume                                  |
  | url          | http://controller:8776/v1/%(tenant_id)s |
  +--------------+-----------------------------------------+
```

```txt
$ openstack endpoint create --region RegionOne \
  volume admin http://controller:8776/v1/%\(tenant_id\)s

  +--------------+-----------------------------------------+
  | Field        | Value                                   |
  +--------------+-----------------------------------------+
  | enabled      | True                                    |
  | id           | 4511c28a0f9840c78bacb25f10f62c98        |
  | interface    | admin                                   |
  | region       | RegionOne                               |
  | region_id    | RegionOne                               |
  | service_id   | ab3bbbef780845a1a283490d281e7fda        |
  | service_name | cinder                                  |
  | service_type | volume                                  |
  | url          | http://controller:8776/v1/%(tenant_id)s |
  +--------------+-----------------------------------------+
```

```txt
$ openstack endpoint create --region RegionOne \
  volumev2 public http://controller:8776/v2/%\(tenant_id\)s

+--------------+-----------------------------------------+
| Field        | Value                                   |
+--------------+-----------------------------------------+
| enabled      | True                                    |
| id           | 513e73819e14460fb904163f41ef3759        |
| interface    | public                                  |
| region       | RegionOne                               |
| region_id    | RegionOne                               |
| service_id   | eb9fd245bdbc414695952e93f29fe3ac        |
| service_name | cinderv2                                |
| service_type | volumev2                                |
| url          | http://controller:8776/v2/%(tenant_id)s |
+--------------+-----------------------------------------+
```

```txt
$ openstack endpoint create --region RegionOne \
  volumev2 internal http://controller:8776/v2/%\(tenant_id\)s

+--------------+-----------------------------------------+
| Field        | Value                                   |
+--------------+-----------------------------------------+
| enabled      | True                                    |
| id           | 6436a8a23d014cfdb69c586eff146a32        |
| interface    | internal                                |
| region       | RegionOne                               |
| region_id    | RegionOne                               |
| service_id   | eb9fd245bdbc414695952e93f29fe3ac        |
| service_name | cinderv2                                |
| service_type | volumev2                                |
| url          | http://controller:8776/v2/%(tenant_id)s |
+--------------+-----------------------------------------+
```

```txt
$ openstack endpoint create --region RegionOne \
  volumev2 admin http://controller:8776/v2/%\(tenant_id\)s

+--------------+-----------------------------------------+
| Field        | Value                                   |
+--------------+-----------------------------------------+
| enabled      | True                                    |
| id           | e652cf84dd334f359ae9b045a2c91d96        |
| interface    | admin                                   |
| region       | RegionOne                               |
| region_id    | RegionOne                               |
| service_id   | eb9fd245bdbc414695952e93f29fe3ac        |
| service_name | cinderv2                                |
| service_type | volumev2                                |
| url          | http://controller:8776/v2/%(tenant_id)s |
+--------------+-----------------------------------------+
```
Note : The Block Storage services require endpoints for each service entity.

Install and configure components¶
---

1 . Install the packages:

```txt
# yum install openstack-cinder
```
2 . Edit the <code>/etc/cinder/cinder.conf</code> file and complete the following actions:

In the <code>[database]</code> section, configure database access:

```txt
[database]
# ...
connection = mysql+pymysql://cinder:CINDER_DBPASS@controller/cinder
```
Replace <code>CINDER_DBPASS</code> with the password you chose for the Block Storage database.

In the <code>[DEFAULT]</code> section, configure RabbitMQ message queue access:

```txt
[DEFAULT]
# ...
transport_url = rabbit://openstack:RABBIT_PASS@controller
```
Replace <code>RABBIT_PASS</code> with the password you chose for the openstack account in RabbitMQ.

In the <code>[DEFAULT]</code> and <code>[keystone_authtoken]</code> sections, configure Identity service access:

```txt
[DEFAULT]
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
username = cinder
password = CINDER_PASS
```
Replace <code>CINDER_PASS</code> with the password you chose for the cinder user in the Identity service.

Note : Comment out or remove any other options in the <code>[keystone_authtoken]</code> section.

In the <code>[DEFAULT]</code> section, configure the my_ip option to use the management interface IP address of the controller node:

```txt
[DEFAULT]
# ...
my_ip = 10.0.0.11
```
In the <code>[oslo_concurrency]</code> section, configure the lock path:

```txt
[oslo_concurrency]
# ...
lock_path = /var/lib/cinder/tmp
```
3 . Populate the Block Storage database:

```txt
# su -s /bin/sh -c "cinder-manage db sync" cinder
```
Note : Ignore any deprecation messages in this output.

验证有没有将配置写入数据库

```txt
[root@block1 ~]# mysql -h controller -u cinder -p
Enter password: 
Welcome to the MariaDB monitor.  Commands end with ; or \g.
Your MariaDB connection id is 105
Server version: 10.1.20-MariaDB MariaDB Server

Copyright (c) 2000, 2016, Oracle, MariaDB Corporation Ab and others.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

MariaDB [(none)]> use cinder
Reading table information for completion of table and column names
You can turn off this feature to get a quicker startup with -A

Database changed
MariaDB [cinder]> show tables;
+----------------------------+
| Tables_in_cinder           |
+----------------------------+
| attachment_specs           |
| backups                    |
| cgsnapshots                |
| clusters                   |
| consistencygroups          |
| driver_initiator_data      |
| encryption                 |
| group_snapshots            |
| group_type_projects        |
| group_type_specs           |
| group_types                |
| group_volume_type_mapping  |
| groups                     |
| image_volume_cache_entries |
| messages                   |
| migrate_version            |
| quality_of_service_specs   |
| quota_classes              |
| quota_usages               |
| quotas                     |
| reservations               |
| services                   |
| snapshot_metadata          |
| snapshots                  |
| transfers                  |
| volume_admin_metadata      |
| volume_attachment          |
| volume_glance_metadata     |
| volume_metadata            |
| volume_type_extra_specs    |
| volume_type_projects       |
| volume_types               |
| volumes                    |
| workers                    |
+----------------------------+
34 rows in set (0.00 sec)

MariaDB [cinder]> 
```
Configure Compute to use Block Storage¶
---

Edit the <code>/etc/nova/nova.conf</code> file and add the following to it:

```txt
[cinder]
os_region_name = RegionOne
```

Finalize installation¶
---

Restart the Compute API service:

```txt
# systemctl restart openstack-nova-api.service
```
Start the Block Storage services and configure them to start when the system boots:

```txt
# systemctl enable openstack-cinder-api.service openstack-cinder-scheduler.service
# systemctl start openstack-cinder-api.service openstack-cinder-scheduler.service
```