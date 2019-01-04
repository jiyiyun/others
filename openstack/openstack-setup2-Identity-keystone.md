Identity service
===

The OpenStack Identity service provides a single point of integration for managing authentication, authorization, and a catalog of services.

The Identity service contains these components:

Server
    A centralized server provides authentication and authorization services using a RESTful interface.
Drivers
    Drivers or a service back end are integrated to the centralized server. They are used for accessing identity information in repositories external to OpenStack, and may already exist in the infrastructure where OpenStack is deployed (for example, SQL databases or LDAP servers).
Modules
    Middleware modules run in the address space of the OpenStack component that is using the Identity service. These modules intercept service requests, extract user credentials, and send them to the centralized server for authorization. The integration between the middleware modules and OpenStack components uses the Python Web Server Gateway Interface. 

Install and configure
---

1 . Use the database access client to connect to the database server as the root user:

```txt
$ mysql -u root -p
```
2 . Create the keystone database:

```txt
    MariaDB [(none)]> CREATE DATABASE keystone;
```
3 . Grant proper access to the keystone database:

```txt
创建keystone数据库，并创建keystone用户，赋予keystone数据库所有权限，本地所有权限和远程所有权限
MariaDB [(none)]> CREATE DATABASE keystone;
MariaDB [(none)]> GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'localhost' IDENTIFIED BY 'KEYSTONE_DBPASS';
MariaDB [(none)]> GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'controller' IDENTIFIED BY 'KEYSTONE_DBPASS';
MariaDB [(none)]> GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'%' IDENTIFIED BY 'KEYSTONE_DBPASS';
MariaDB [(none)]> FLUSH PRIVILEGES; 
```
Replace <code>KEYSTONE_DBPASS</code> with a suitable password.

走到这里先不要往下走，先验证mysql设置成不成功

```txt
1、在本机用keystone用户连接本机IP
[root@controller ~]# mysql -h 192.168.100.64 -u keystone -p
Enter password: 
Welcome to the MariaDB monitor.  Commands end with ; or \g.
Your MariaDB connection id is 9
Server version: 10.1.20-MariaDB MariaDB Server

Copyright (c) 2000, 2016, Oracle, MariaDB Corporation Ab and others.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

MariaDB [(none)]> exit
Bye

2、在本机用keystone用户连接controller 这步最关键

[root@controller ~]# mysql -h controller -u keystone -p
Enter password: 
Welcome to the MariaDB monitor.  Commands end with ; or \g.
Your MariaDB connection id is 10
Server version: 10.1.20-MariaDB MariaDB Server

Copyright (c) 2000, 2016, Oracle, MariaDB Corporation Ab and others.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

MariaDB [(none)]> exit
Bye
[root@controller ~]# 

3 、在conpute1主机上测试连接controller
[root@compute1 ~]# mysql -h controller -u keystone -p
Enter password: 
Welcome to the MariaDB monitor.  Commands end with ; or \g.
Your MariaDB connection id is 14
Server version: 10.1.20-MariaDB MariaDB Server

Copyright (c) 2000, 2016, Oracle, MariaDB Corporation Ab and others.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

MariaDB [(none)]> 

4 、在block1主机上测试连接controller
[root@block1 ~]# mysql -h controller -u keystone -p
Enter password: 
Welcome to the MariaDB monitor.  Commands end with ; or \g.
Your MariaDB connection id is 11
Server version: 10.1.20-MariaDB MariaDB Server

Copyright (c) 2000, 2016, Oracle, MariaDB Corporation Ab and others.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

MariaDB [(none)]> 
```
4 . Exit the database access client.

Install and configure components
---

1 . Run the following command to install the packages:

```txt
    # yum install openstack-keystone httpd mod_wsgi
```
Note：This guide uses the Apache HTTP server with mod_wsgi to serve Identity service requests on ports 5000 and 35357. By default, the keystone service still listens on these ports. Therefore, this guide manually disables the keystone service.

注意：该指南使用Apache Http服务器的mod_wsgi(Python Web Server Gateway Interface)动态访问模块来支持认证服务在5000和35357端口上的请求。keystone service默认就会监听这两个端口，所以，该指南手动的禁用keystone service。

WSGI：Python Web服务器网关接口（Python Web Server Gateway Interface，缩写为WSGI)，是Python应用程序或框架和Web服务器之间的一种接口，已经被广泛接受, 它已基本达成它的可移植性方面的目标

2 . Edit the <code>/etc/keystone/keystone.conf</code> file and complete the following actions:

        In the [database] section, configure database access:

```txt
        [database]
        # ...
        connection = mysql+pymysql://keystone:KEYSTONE_DBPASS@controller/keystone

        mysql_sql_mode = TRADITIONAL

        Replace KEYSTONE_DBPASS with the password you chose for the database.

        In the [token] section, configure the Fernet token provider:

        [token]
        # ...
        provider = fernet

        这个配置文件只改这三处
```
Replace <code>KEYSTONE_DBPASS</code> with a suitable password.

概览keystone配置

```txt 
[root@controller ~]#  cat /etc/keystone/keystone.conf | grep -v ^# | grep -v ^$
[DEFAULT]
[assignment]
[auth]
[cache]
[catalog]
[cors]
[cors.subdomain]
[credential]
[database]
connection = mysql+pymysql://keystone:KEYSTONE_DBPASS@controller/keystone
mysql_sql_mode = TRADITIONAL
[domain_config]
[endpoint_filter]
[endpoint_policy]
[eventlet_server]
[federation]
[fernet_tokens]
[healthcheck]
[identity]
[identity_mapping]
[kvs]
[ldap]
[matchmaker_redis]
[memcache]
[oauth1]
[oslo_messaging_amqp]
[oslo_messaging_kafka]
[oslo_messaging_notifications]
[oslo_messaging_rabbit]
[oslo_messaging_zmq]
[oslo_middleware]
[oslo_policy]
[paste_deploy]
[policy]
[profiler]
[resource]
[revoke]
[role]
[saml]
[security_compliance]
[shadow_users]
[signing]
[token]
provider = fernet
[tokenless_auth]
[trust]
[root@controller ~]#
```
3 . Populate the Identity service database:

```txt
# su -s /bin/sh -c "keystone-manage db_sync" keystone
```
使用sh执行keystone数据库初始化填充指令

注意：写入完验一定要证下有没有填充keystone到数据库中

```txt
[root@block1 ~]# mysql -h controller -u keystone -p
Enter password: 
Welcome to the MariaDB monitor.  Commands end with ; or \g.
Your MariaDB connection id is 11
Server version: 10.1.20-MariaDB MariaDB Server

Copyright (c) 2000, 2016, Oracle, MariaDB Corporation Ab and others.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

MariaDB [(none)]> use keystone
Reading table information for completion of table and column names
You can turn off this feature to get a quicker startup with -A

Database changed
MariaDB [keystone]> show tables;
+------------------------+
| Tables_in_keystone     |
+------------------------+
| access_token           |
| assignment             |
| config_register        |
| consumer               |
| credential             |
| endpoint               |
| endpoint_group         |
| federated_user         |
| federation_protocol    |
| group                  |
| id_mapping             |
| identity_provider      |
```

4 . Initialize Fernet key repositories:

```txt
# keystone-manage fernet_setup --keystone-user keystone --keystone-group keystone
# keystone-manage credential_setup --keystone-user keystone --keystone-group keystone
```
5 . Bootstrap the Identity service:

```txt
# keystone-manage bootstrap --bootstrap-password ADMIN_PASS \
  --bootstrap-admin-url http://controller:35357/v3/ \
  --bootstrap-internal-url http://controller:5000/v3/ \
  --bootstrap-public-url http://controller:5000/v3/ \
  --bootstrap-region-id RegionOne
```
Replace <code>ADMIN_PASS</code> with a suitable password for an administrative user.

Configure the Apache HTTP server¶
---

1 . Edit the <code>/etc/httpd/conf/httpd.conf</code> file and configure the ServerName option to reference the controller node:

```txt
    ServerName controller
```
2 . Create a link to the <code>/usr/share/keystone/wsgi-keystone.conf</code> file:

```txt
    # ln -s /usr/share/keystone/wsgi-keystone.conf /etc/httpd/conf.d/
```
Finalize the installation
---

1 . Start the Apache HTTP service and configure it to start when the system boots:

```txt
    # systemctl enable httpd.service
    # systemctl start httpd.service
```
2 . Configure the administrative account

```txt
    $ export OS_USERNAME=admin
    $ export OS_PASSWORD=ADMIN_PASS
    $ export OS_PROJECT_NAME=admin
    $ export OS_USER_DOMAIN_NAME=Default
    $ export OS_PROJECT_DOMAIN_NAME=Default
    $ export OS_AUTH_URL=http://controller:35357/v3
    $ export OS_IDENTITY_API_VERSION=3
```
Replace <code>ADMIN_PASS</code> with the password used in the keystone-manage bootstrap command in keystone-install-configure

keystone 可以使用了

Create a domain, projects, users, and roles
===


Contents

The Identity service provides authentication services for each OpenStack service. The authentication service uses a combination of domains, projects, users, and roles.

1 . This guide uses a service project that contains a unique user for each service that you add to your environment. Create the service project:

```txt
$ openstack project create --domain default \
  --description "Service Project" service

+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description | Service Project                  |
| domain_id   | default                          |
| enabled     | True                             |
| id          | 24ac7f19cd944f4cba1d77469b2a73ed |
| is_domain   | False                            |
| name        | service                          |
| parent_id   | default                          |
+-------------+----------------------------------+
```
2 . Regular (non-admin) tasks should use an unprivileged project and user. As an example, this guide creates the demo project and user.

    Create the demo project:

```txt
    $ openstack project create --domain default \
      --description "Demo Project" demo

    +-------------+----------------------------------+
    | Field       | Value                            |
    +-------------+----------------------------------+
    | description | Demo Project                     |
    | domain_id   | default                          |
    | enabled     | True                             |
    | id          | 231ad6e7ebba47d6a1e57e1cc07ae446 |
    | is_domain   | False                            |
    | name        | demo                             |
    | parent_id   | default                          |
    +-------------+----------------------------------+
```
Create the demo user:

```txt
$ openstack user create --domain default \
  --password-prompt demo

User Password:
Repeat User Password:
+---------------------+----------------------------------+
| Field               | Value                            |
+---------------------+----------------------------------+
| domain_id           | default                          |
| enabled             | True                             |
| id                  | aeda23aa78f44e859900e22c24817832 |
| name                | demo                             |
| password_expires_at | None                             |
+---------------------+----------------------------------+
```
Create the user role:

```txt
$ openstack role create user

+-----------+----------------------------------+
| Field     | Value                            |
+-----------+----------------------------------+
| domain_id | None                             |
| id        | 997ce8d05fc143ac97d83fdfb5998552 |
| name      | user                             |
+-----------+----------------------------------+
```
Add the user role to the demo project and user:

```txt
$ openstack role add --project demo --user demo user
```
Verify operation
---

1 . For security reasons, disable the temporary authentication token mechanism:

Edit the <code>/etc/keystone/keystone-paste.ini</code> file and remove admin_token_auth from the [pipeline:public_api], [pipeline:admin_api], and [pipeline:api_v3] sections.

2 . Unset the temporary <code>OS_AUTH_URL</code> and <code>OS_PASSWORD</code> environment variable:

```txt
$ unset OS_AUTH_URL OS_PASSWORD
```
3 . As the admin user, request an authentication token:

```txt
$ openstack --os-auth-url http://controller:35357/v3 \
  --os-project-domain-name default --os-user-domain-name default \
  --os-project-name admin --os-username admin token issue

Password:
+------------+----------------------------------------------------------------------------------------------------+
| Field      | Value                                                                                              |
+------------+----------------------------------------------------------------------------------------------------+
| expires    | 2017-03-02T08:13:18+0000                                                                           |
| id         | gAAAAABYt8YOcjBs-zkGWn1PI_UlojHEdWxrQkFp1e8VMuG1SE__8jdajAa_DC7oFeMT4PKrSwGXOuYtI9zdQXX-           |
|            | GcCZgh0NWLTyDeTbyE96Nn6wqDUB6NI-v2r6cmbfx2NNhtS05ZSgfhVQpLvEvPXTfVkumwQcDwdJdfCEQlBIifqUO26HrGg    |
| project_id | 40a83a1c15fa407283bbd5e48b7b3b87                                                                   |
| user_id    | 7ab58453c4684201a202fcda7f2efa9f                                                                   |
+------------+----------------------------------------------------------------------------------------------------+
```
注意：这里用户是admin，所以要输入正确的admin密码才能出现这个结果

4 . As the demo user, request an authentication token:

```txt
$ openstack --os-auth-url http://controller:5000/v3 \
  --os-project-domain-name default --os-user-domain-name default \
  --os-project-name demo --os-username demo token issue

Password:
+------------+----------------------------------------------------------------------------------------------------+
| Field      | Value                                                                                              |
+------------+----------------------------------------------------------------------------------------------------+
| expires    | 2017-03-02T08:13:56+0000                                                                           |
| id         | gAAAAABYt8Y0mujmDnwlOk4TWxoZ7vc3DB7cQJto3K6KzKEWx-JdLFhHHUWWj04dBjFCIMx-                           |
|            | iflyxpRyONGCrB0QRVW7P6v2vTAYQSSumQasfhd6AzZoQbl6NvF5bSoYkXcAsCQwgnUzs-                             |
|            | 6YJTSS_VyOEfkkjYhONbtzfMgG6Gt3hJB2rMjhrIY                                                          |
| project_id | ded444da53f146dda544fe3a85a4118d                                                                   |
| user_id    | de8ed9c250914bc2bae671da46cdfa79                                                                   |
+------------+----------------------------------------------------------------------------------------------------+
```
注意：这里用户是demo 输入正确的demo用户密码才能出现这个结果

Create OpenStack client environment scripts
---

Creating the scripts¶

Create client environment scripts for the admin and demo projects and users. Future portions of this guide reference these scripts to load appropriate credentials for client operations.

1 .  Create and edit the <code> admin-openrc </code> file and add the following content:

```txt
export OS_PROJECT_DOMAIN_NAME=Default
export OS_USER_DOMAIN_NAME=Default
export OS_PROJECT_NAME=admin
export OS_USERNAME=admin
export OS_PASSWORD=ADMIN_PASS
export OS_AUTH_URL=http://controller:35357/v3
export OS_IDENTITY_API_VERSION=3
export OS_IMAGE_API_VERSION=2
```
Replace <code>ADMIN_PASS</code> with the password you chose for the admin user in the Identity service.

2 . Create and edit the <code> demo-openrc </code> file and add the following content:

```txt
export OS_PROJECT_DOMAIN_NAME=Default
export OS_USER_DOMAIN_NAME=Default
export OS_PROJECT_NAME=demo
export OS_USERNAME=demo
export OS_PASSWORD=DEMO_PASS
export OS_AUTH_URL=http://controller:5000/v3
export OS_IDENTITY_API_VERSION=3
export OS_IMAGE_API_VERSION=2
```
Replace <code>DEMO_PASS</code> with the password you chose for the demo user in the Identity service.

雷区警示：上面两个环境变量看似一样其实有很大差别，设置的两个密码<code>ADMIN_PASS</code>、  <code>DEMO_PASS</code> 一定要记住，下面要用到

Using the scripts¶
---

To run clients as a specific project and user, you can simply load the associated client environment script prior to running them. For example:

1 . Load the <code>admin-openrc</code> file to populate environment variables with the location of the Identity service and the admin project and user credentials:

```txt
    $ . admin-openrc
```
2 . Request an authentication token:

```txt
    $ openstack token issue

   +------------+----------------------------------------------------------------------------------------------------+
| Field      | Value                                                                                              |
+------------+----------------------------------------------------------------------------------------------------+
| expires    | 2017-03-02T08:17:12+0000                                                                           |
| id         | gAAAAABYt8b4KFYy-WFj3cv0-D4p31-N9OZtRB35LsNfFWQ1BRj4Ogqa7t-0hdr-5fT52ON_8ATUUJzJPQyx8bAsJ26NArr5lB |
|            | 4l2oorYBGpQukK44dT7CzXbKKqTRXPrMPftuWS31EeX8FZeIZ5rdSyeANl0vxGbXTXNfZxg5hhwb6GxCNyusM              |
| project_id | ded444da53f146dda544fe3a85a4118d                                                                   |
| user_id    | de8ed9c250914bc2bae671da46cdfa79                                                                   |
+------------+----------------------------------------------------------------------------------------------------+
```
第二关：完成

附录:删除以创建的service、project、user、role的方法(实例中创建project、user都是在$权限下，如果用#权限创建建议删除)

```txt
[openstack@controller ~]$ openstack project delete --domain default service
[openstack@controller ~]$ openstack project delete --domain default demo
[openstack@controller ~]$ openstack role delete user
[openstack@controller ~]$ openstack user delete --domain default demo
```
参考资料
---
- https://docs.openstack.org/ocata/install-guide-rdo/keystone-install.html
- http://www.2cto.com/net/201606/516833.html
- http://dolphm.com/openstack-keystone-fernet-tokens/