Image service
===

Image service overview
---

Contents

The Image service (glance) enables users to discover, register, and retrieve virtual machine images. It offers a REST API that enables you to query virtual machine image metadata and retrieve an actual image. You can store virtual machine images made available through the Image service in a variety of locations, from simple file systems to object-storage systems like OpenStack Object Storage.

Important

```txt
For simplicity, this guide describes configuring the Image service to use the file back end, which uploads and stores in a directory on the controller node hosting the Image service. By default, this directory is /var/lib/glance/images/.

Before you proceed, ensure that the controller node has at least several gigabytes of space available in this directory. Keep in mind that since the file back end is often local to a controller node, it is not typically suitable for a multi-node glance deployment.
```
glance-api
---

    Accepts Image API calls for image discovery, retrieval, and storage.

glance-registry
---
    Stores, processes, and retrieves metadata about images. Metadata includes items such as size and type.

```
Warning

    The registry is a private internal service meant for use by OpenStack Image service. Do not expose this service to users.
```
Database
    Stores image metadata and you can choose your database depending on your preference. Most deployments use MySQL or SQLite.

Storage repository for image files
    Various repository types are supported including normal file systems (or any filesystem mounted on the glance-api controller node), Object Storage, RADOS block devices, VMware datastore, and HTTP. Note that some repositories will only support read-only usage.

Metadata definition service
    A common API for vendors, admins, services, and users to meaningfully define their own custom metadata. This metadata can be used on different types of resources like images, artifacts, volumes, flavors, and aggregates. A definition includes the new property’s key, description, constraints, and the resource types which it can be associated with

Install and configure
---

Prerequisites¶

1 . Before you install and configure the Image service, you must create a database, service credentials, and API endpoints.

    To create the database, complete these steps:

        Use the database access client to connect to the database server as the root user:

```txt
        $ mysql -u root -p
```
        Create the glance database:
```txt
        MariaDB [(none)]> CREATE DATABASE glance;
```
        Grant proper access to the glance database:

```txt
        MariaDB [(none)]> GRANT ALL PRIVILEGES ON glance.* TO 'glance'@'localhost' IDENTIFIED BY 'GLANCE_DBPASS';
        MariaDB [(none)]> GRANT ALL PRIVILEGES ON glance.* TO 'glance'@'controller' IDENTIFIED BY 'GLANCE_DBPASS';
        MariaDB [(none)]> GRANT ALL PRIVILEGES ON glance.* TO 'glance'@'%' IDENTIFIED BY 'GLANCE_DBPASS';
        MariaDB [(none)]> FLUSH PRIVILEGES;
```
Replace <code>GLANCE_DBPASS</code> with a suitable password.

Exit the database access client.

数据库创建好了一定要验证，包括本机验证和在其它节点验证

验证glance数据库

```
在本机验证成功
[openstack@controller ~]$ mysql -h controller -u glance -p
Enter password: 
Welcome to the MariaDB monitor.  Commands end with ; or \g.
Your MariaDB connection id is 26
Server version: 10.1.20-MariaDB MariaDB Server

Copyright (c) 2000, 2016, Oracle, MariaDB Corporation Ab and others.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

MariaDB [(none)]> 

在block节点验证
[root@block1 ~]# mysql -h controller -u glance -p
Enter password: 
Welcome to the MariaDB monitor.  Commands end with ; or \g.
Your MariaDB connection id is 23
Server version: 10.1.20-MariaDB MariaDB Server

Copyright (c) 2000, 2016, Oracle, MariaDB Corporation Ab and others.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

MariaDB [(none)]> 
```
2 . Source the admin credentials to gain access to admin-only CLI commands:

```txt
    $ . admin-openrc
```
3 . To create the service credentials, complete these steps:

    Create the glance user:

```txt
[openstack@controller ~]$ openstack user create --domain default --password-prompt glance
User Password:
Repeat User Password:
+---------------------+----------------------------------+
| Field               | Value                            |
+---------------------+----------------------------------+
| domain_id           | default                          |
| enabled             | True                             |
| id                  | 767550d1e8744d81a37b0d4309d4c46e |
| name                | glance                           |
| options             | {}                               |
| password_expires_at | None                             |
+---------------------+----------------------------------+

```
    Add the admin role to the glance user and service project:

```txt
[openstack@controller ~]$ openstack role add --project service --user glance admin
```
Create the glance service entity:

```txt
$ openstack service create --name glance \
  --description "OpenStack Image" image

+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description | OpenStack Image                  |
| enabled     | True                             |
| id          | bd07a89ceba144c6ba6fe74e612fe610 |
| name        | glance                           |
| type        | image                            |
+-------------+----------------------------------+
```
4 . Create the Image service API endpoints:

```txt
$ openstack endpoint create --region RegionOne \
  image public http://controller:9292

+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | bdaca7f6b9b84924a231d5a6204eede1 |
| interface    | public                           |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | bd07a89ceba144c6ba6fe74e612fe610 |
| service_name | glance                           |
| service_type | image                            |
| url          | http://controller:9292           |
+--------------+----------------------------------+

$ openstack endpoint create --region RegionOne \
  image internal http://controller:9292

+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | 6f01183152ac4f62a069c43b169bc63c |
| interface    | internal                         |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | bd07a89ceba144c6ba6fe74e612fe610 |
| service_name | glance                           |
| service_type | image                            |
| url          | http://controller:9292           |
+--------------+----------------------------------+

$ openstack endpoint create --region RegionOne \
  image admin http://controller:9292

+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | 4c5979344272457f919cd6f3ddbe5a1c |
| interface    | admin                            |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | bd07a89ceba144c6ba6fe74e612fe610 |
| service_name | glance                           |
| service_type | image                            |
| url          | http://controller:9292           |
+--------------+----------------------------------+
```
Install and configure components¶
---

1 . Install the packages:

```txt
    # yum install openstack-glance
```
2 . Edit the <code>/etc/glance/glance-api.conf</code> file and complete the following actions:

In the <code>[database]</code> section, configure database access:

```txt
        [database]
        # ...
        connection = mysql+pymysql://glance:GLANCE_DBPASS@controller/glance
```
Replace <code>GLANCE_DBPASS</code> with the password you chose for the Image service database.

In the <code>[keystone_authtoken]</code> and <code>[paste_deploy]</code> sections, configure Identity service access:

```txt
        [keystone_authtoken]
        # ...
        auth_uri = http://controller:5000
        auth_url = http://controller:35357
        memcached_servers = controller:11211
        auth_type = password
        project_domain_name = default
        user_domain_name = default
        project_name = service
        username = glance
        password = GLANCE_PASS

        [paste_deploy]
        # ...
        flavor = keystone
```
Replace <code>GLANCE_PASS</code> with the password you chose for the glance user in the Identity service.

In the <code>[glance_store]</code> section, configure the local file system store and location of image files:

```txt
    [glance_store]
    # ...
    stores = file,http
    default_store = file
    filesystem_store_datadir = /var/lib/glance/images/
```
3 . Edit the <code>/etc/glance/glance-registry.conf</code> file and complete the following actions:

In the <code>[database]</code> section, configure database access:

```txt
    [database]
    # ...
    connection = mysql+pymysql://glance:GLANCE_DBPASS@controller/glance
```
Replace <code>GLANCE_DBPASS</code> with the password you chose for the Image service database.

In the <code>[keystone_authtoken]</code> and <code>[paste_deploy]</code> sections, configure Identity service access:

```txt
    [keystone_authtoken]
    # ...
    auth_uri = http://controller:5000
    auth_url = http://controller:35357
    memcached_servers = controller:11211
    auth_type = password
    project_domain_name = default
    user_domain_name = default
    project_name = service
    username = glance
    password = GLANCE_PASS

    [paste_deploy]
    # ...
    flavor = keystone
```
Replace <code>GLANCE_PASS</code> with the password you chose for the glance user in the Identity service.

4 .  Populate the Image service database:

```txt
# su -s /bin/sh -c "glance-manage db_sync" glance
```

```txt
[root@controller openstack]# su -s /bin/sh -c "glance-manage db_sync" glance
Option "verbose" from group "DEFAULT" is deprecated for removal.  Its value may be silently ignored in the future.
/usr/lib/python2.7/site-packages/oslo_db/sqlalchemy/enginefacade.py:1241: OsloDBDeprecationWarning: EngineFacade is deprecated; please use oslo_db.sqlalchemy.enginefacade
  expire_on_commit=expire_on_commit, _conf=conf)
INFO  [alembic.runtime.migration] Context impl MySQLImpl.
INFO  [alembic.runtime.migration] Will assume non-transactional DDL.
INFO  [alembic.runtime.migration] Running upgrade  -> liberty, liberty initial
INFO  [alembic.runtime.migration] Running upgrade liberty -> mitaka01, add index on created_at and updated_at columns of 'images' table
INFO  [alembic.runtime.migration] Running upgrade mitaka01 -> mitaka02, update metadef os_nova_server
INFO  [alembic.runtime.migration] Running upgrade mitaka02 -> ocata01, add visibility to and remove is_public from images
INFO  [alembic.runtime.migration] Context impl MySQLImpl.
INFO  [alembic.runtime.migration] Will assume non-transactional DDL.
Upgraded database to: ocata01, current revision(s): ocata01

```
注意：验证有没有写入到数据库中

验证配置信息有没有写到glance数据库中

```txt
[root@block1 ~]# mysql -h controller -u glance -p
Enter password: 
Welcome to the MariaDB monitor.  Commands end with ; or \g.
Your MariaDB connection id is 23
Server version: 10.1.20-MariaDB MariaDB Server

Copyright (c) 2000, 2016, Oracle, MariaDB Corporation Ab and others.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

MariaDB [(none)]> use glance
Reading table information for completion of table and column names
You can turn off this feature to get a quicker startup with -A

Database changed
MariaDB [glance]> show tables;
+----------------------------------+
| Tables_in_glance                 |
+----------------------------------+
| alembic_version                  |
| artifact_blob_locations          |
| artifact_blobs                   |
| artifact_dependencies            |
| artifact_properties              |
| artifact_tags                    |
| artifacts                        |
| image_locations                  |
| image_members                    |
| image_properties                 |
| image_tags                       |
| images                           |
| metadef_namespace_resource_types |
| metadef_namespaces               |
| metadef_objects                  |
| metadef_properties               |
| metadef_resource_types           |
| metadef_tags                     |
| migrate_version                  |
| task_info                        |
| tasks                            |
+----------------------------------+
21 rows in set (0.01 sec)
```
已经正确写入

Finalize installation¶
---

    Start the Image services and configure them to start when the system boots:

```txt
[root@controller openstack]# systemctl enable openstack-glance-api.service   openstack-glance-registry.service
[root@controller openstack]# systemctl start openstack-glance-api.service   openstack-glance-registry.service
```
Verify operation (用一个小镜像CirrOS做验证)
---

Verify operation of the Image service using CirrOS, a small Linux image that helps you test your OpenStack deployment.

For more information about how to download and build images, see OpenStack Virtual Machine Image Guide. For information about how to manage images, see the OpenStack End User Guide.

1 . Source the admin credentials to gain access to admin-only CLI commands:

```txt
$ . admin-openrc
```
2 . Download the source image:

```txt
$ wget http://download.cirros-cloud.net/0.3.4/cirros-0.3.4-x86_64-disk.img
```
3 . Upload the image to the Image service using the QCOW2 disk format, bare container format, and public visibility so all projects can access it:

```txt
$ openstack image create "cirros" \
  --file cirros-0.3.4-x86_64-disk.img \
  --disk-format qcow2 --container-format bare \
  --public

+------------------+------------------------------------------------------+
| Field            | Value                                                |
+------------------+------------------------------------------------------+
| checksum         | ee1eca47dc88f4879d8a229cc70a07c6                     |
| container_format | bare                                                 |
| created_at       | 2017-03-03T01:33:07Z                                 |
| disk_format      | qcow2                                                |
| file             | /v2/images/536b0640-4a72-4570-9b10-22c6b44fa35a/file |
| id               | 536b0640-4a72-4570-9b10-22c6b44fa35a                 |
| min_disk         | 0                                                    |
| min_ram          | 0                                                    |
| name             | cirros                                               |
| owner            | 40a83a1c15fa407283bbd5e48b7b3b87                     |
| protected        | False                                                |
| schema           | /v2/schemas/image                                    |
| size             | 13287936                                             |
| status           | active                                               |
| tags             |                                                      |
| updated_at       | 2017-03-03T01:33:13Z                                 |
| virtual_size     | None                                                 |
| visibility       | public                                               |
+------------------+------------------------------------------------------+
```
For information about the openstack image create parameters, see Create or update an image (glance) in the OpenStack User Guide.

For information about disk and container formats for images, see Disk and container formats for images in the OpenStack Virtual Machine Image Guide.

上传镜像出错检查日志

```txt
[root@controller openstack]# cat /var/log/glance/api.log
[root@controller openstack]# cat /var/log/glance/registry.log
```
```txt
[root@controller openstack]# cat /var/log/glance/api.log
2017-03-02 16:21:51.639 5583 WARNING keystonemiddleware.auth_token [-] AuthToken middleware is set with keystone_authtoken.service_token_roles_required set to False. This is backwards compatible but deprecated behaviour. Please set this to True

[root@controller openstack]# cat /var/log/glance/registry.log
2017-03-03 00:45:46.397 1058 WARNING keystonemiddleware.auth_token [-] AuthToken middleware is set with keystone_authtoken.service_token_roles_required set to False. This is backwards compatible but deprecated behaviour. Please set this to True
```
4 . Confirm upload of the image and validate attributes:

```txt
[openstack@controller ~]$ openstack image list
+--------------------------------------+--------+--------+
| ID                                   | Name   | Status |
+--------------------------------------+--------+--------+
| 536b0640-4a72-4570-9b10-22c6b44fa35a | cirros | active |
+--------------------------------------+--------+--------+
```

参考资料
---

- https://docs.openstack.org/ocata/install-guide-rdo/glance-install.html#install-and-configure-components
- http://blog.csdn.net/jmilk/article/details/51724360
