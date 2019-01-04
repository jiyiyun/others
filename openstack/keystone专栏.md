keystone专栏
===

Keystone V3 简介
Keystone 中主要涉及到如下几个概念：User、Tenant、Role、Token。下面对这几个概念进行简要说明。
User：顾名思义就是使用服务的用户，可以是人、服务或者是系统，只要是使用了 Openstack 服务的对象都可以称为用户。
Tenant：租户，可以理解为一个人、项目或者组织拥有的资源的合集。在一个租户中可以拥有很多个用户，这些用户可以根据权限的划分使用租户中的资源。
Role：角色，用于分配操作的权限。角色可以被指定给用户，使得该用户获得角色对应的操作权限。
Token：指的是一串比特值或者字符串，用来作为访问资源的记号。Token 中含有可访问资源的范围和有效时间。


查看keystone 连接mysql数据库出错日志 /var/log/keystone/keystone.log

```txt
2017-03-02 00:21:28.377 747 ERROR keystone   File "/usr/lib/python2.7/site-packages/pymysql/connections.py", line 393, in check_error
2017-03-02 00:21:28.377 747 ERROR keystone     err.raise_mysql_exception(self._data)
2017-03-02 00:21:28.377 747 ERROR keystone   File "/usr/lib/python2.7/site-packages/pymysql/err.py", line 107, in raise_mysql_exception
2017-03-02 00:21:28.377 747 ERROR keystone     raise errorclass(errno, errval)
2017-03-02 00:21:28.377 747 ERROR keystone OperationalError: (pymysql.err.OperationalError) (1045, u"Access denied for user 'keystone'@'yingyunode1' (using password: YES)"
```

keystone 对接mysql数据库成功

```txt
2017-03-02 04:36:45.231 6773 INFO migrate.versioning.api [-] 10 -> 11... 
2017-03-02 04:36:45.330 6773 INFO migrate.versioning.api [-] done
2017-03-02 04:36:45.331 6773 INFO migrate.versioning.api [-] 11 -> 12... 
2017-03-02 04:36:45.429 6773 INFO migrate.versioning.api [-] done
2017-03-02 04:36:45.429 6773 INFO migrate.versioning.api [-] 12 -> 13... 
2017-03-02 04:36:45.697 6773 INFO migrate.versioning.api [-] done
2017-03-02 04:36:45.697 6773 INFO migrate.versioning.api [-] 13 -> 14... 
2017-03-02 04:36:45.973 6773 INFO migrate.versioning.api [-] done
2017-03-02 04:36:45.974 6773 INFO migrate.versioning.api [-] 14 -> 15... 
2017-03-02 04:36:46.043 6773 INFO migrate.versioning.api [-] done
2017-03-02 04:36:46.043 6773 INFO migrate.versioning.api [-] 15 -> 16... 
2017-03-02 04:36:46.053 6773 INFO migrate.versioning.api [-] done
```

```txt
MariaDB [keystone]> show tables;
ERROR 2006 (HY000): MySQL server has gone away
No connection. Trying to reconnect...
Connection id:    3
Current database: keystone

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
| idp_remote_ids         |
| implied_role           |
| local_user             |
| mapping                |
| migrate_version        |
| nonlocal_user          |
| password               |
| policy                 |
| policy_association     |
| project                |
| project_endpoint       |
| project_endpoint_group |
| region                 |
| request_token          |
| revocation_event       |
| role                   |
| sensitive_config       |
| service                |
| service_provider       |
| token                  |
| trust                  |
| trust_role             |
| user                   |
| user_group_membership  |
| user_option            |
| whitelisted_config     |
+------------------------+
38 rows in set (0.06 sec)

MariaDB [keystone]> exit
Bye
```
```txt
[root@yingyunode1 7]# vi /etc/httpd/conf/httpd.conf 
[root@yingyunode1 7]# ln -s /usr/share/keystone/wsgi-keystone.conf /etc/httpd/conf.d/
[root@yingyunode1 7]# systemctl enable httpd.service
Created symlink from /etc/systemd/system/multi-user.target.wants/httpd.service to /usr/lib/systemd/system/httpd.service.
[root@yingyunode1 7]# systemctl start httpd.service
[root@yingyunode1 7]# export OS_USERNAME=admin
[root@yingyunode1 7]# export OS_PASSWORD=admin
[root@yingyunode1 7]# export OS_PROJECT_NAME=admin
[root@yingyunode1 7]# export OS_USER_DOMAIN_NAME=Default
[root@yingyunode1 7]# export OS_PROJECT_DOMAIN_NAME=Default
[root@yingyunode1 7]# export OS_AUTH_URL=http://192.168.15.61:35357/v3
[root@yingyunode1 7]# export OS_IDENTITY_API_VERSION=3
[root@yingyunode1 7]# openstack project create --domain default   --description "Service Project" service
+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description | Service Project                  |
| domain_id   | default                          |
| enabled     | True                             |
| id          | d61e5d062e824d5ba40623643e611cf8 |
| is_domain   | False                            |
| name        | service                          |
| parent_id   | default                          |
+-------------+----------------------------------+
[root@yingyunode1 7]# openstack project create --domain default \
>       --description "Demo Project" demo
+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description | Demo Project                     |
| domain_id   | default                          |
| enabled     | True                             |
| id          | 88a237c2751f40d58d996fc783247d3c |
| is_domain   | False                            |
| name        | demo                             |
| parent_id   | default                          |
+-------------+----------------------------------+
[root@yingyunode1 7]# openstack user create --domain default \
>   --password-prompt demo
User Password:
Repeat User Password:
+---------------------+----------------------------------+
| Field               | Value                            |
+---------------------+----------------------------------+
| domain_id           | default                          |
| enabled             | True                             |
| id                  | c28842962bec4630a7ee6135ff2fdea5 |
| name                | demo                             |
| options             | {}                               |
| password_expires_at | None                             |
+---------------------+----------------------------------+
[root@yingyunode1 7]# openstack role create user
+-----------+----------------------------------+
| Field     | Value                            |
+-----------+----------------------------------+
| domain_id | None                             |
| id        | 4ec13b9069ce4e0a8f8218d531971ff2 |
| name      | user                             |
+-----------+----------------------------------+
[root@yingyunode1 7]# openstack role add --project demo --user demo user
[root@yingyunode1 7]# 
```
参考资料
---

- https://docs.openstack.org/developer/keystone/
- https://wiki.openstack.org/wiki/Keystone
- http://www.ibm.com/developerworks/cn/cloud/library/1506_yuwz_keystonev3/index.html
- http://blog.csdn.net/xuyuefei1988/article/details/19328321