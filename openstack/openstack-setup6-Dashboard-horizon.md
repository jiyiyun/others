The Dashboard (horizon) is a web interface that enables cloud administrators and users to manage various OpenStack resources and services.

This example deployment uses an Apache web serve

Note

This section assumes proper installation, configuration, and operation of the Identity service using the Apache HTTP server and Memcached service as described in the Install and configure the Identity service section

Install and configure components¶

 Note:尽量添加而不要动原来默认的配置

 Default configuration files vary by distribution. You might need to add these sections and options rather than modifying existing sections and options. Also, an ellipsis (...) in the configuration snippets indicates potential default configuration options that you should retain.

 Install the packages:
---

```txt
# yum install openstack-dashboard
```
Edit the <code>/etc/openstack-dashboard/local_settings</code> file and complete the following actions:

Configure the dashboard to use OpenStack services on the controller node:

```txt
OPENSTACK_HOST = "controller"
```
Allow your hosts to access the dashboard:

```txt
ALLOWED_HOSTS = ['one.example.com', 'two.example.com']
```
Note: 允许设置成* 允许所有主机来访问

<code>ALLOWED_HOSTS</code> can also be <code>[‘*’]</code> to accept all hosts. This may be useful for development work, but is potentially insecure and should not be used in production. See https://docs.djangoproject.com/en/dev/ref/settings/#allowed-hosts for further information.

Configure the <code>memcached</code> session storage service:

```txt
SESSION_ENGINE = 'django.contrib.sessions.backends.cache'

CACHES = {
    'default': {
         'BACKEND': 'django.core.cache.backends.memcached.MemcachedCache',
         'LOCATION': 'controller:11211',
    }
}
```
Enable the Identity API version 3:

```txt
OPENSTACK_KEYSTONE_URL = "http://%s:5000/v3" % OPENSTACK_HOST
Enable support for domains:

OPENSTACK_KEYSTONE_MULTIDOMAIN_SUPPORT = True
Configure API versions:

OPENSTACK_API_VERSIONS = {
    "identity": 3,
    "image": 2,
    "volume": 2,
}
```
Configure default as the default domain for users that you create via the dashboard:

```txt
OPENSTACK_KEYSTONE_DEFAULT_DOMAIN = "default"
Configure user as the default role for users that you create via the dashboard:

OPENSTACK_KEYSTONE_DEFAULT_ROLE = "user"
```
If you chose networking option 1, disable support for layer-3 networking services:

```txt
OPENSTACK_NEUTRON_NETWORK = {
    ...
    'enable_router': False,
    'enable_quotas': False,
    'enable_distributed_router': False,
    'enable_ha_router': False,
    'enable_lb': False,
    'enable_firewall': False,
    'enable_vpn': False,
    'enable_fip_topology_check': False,
}
```
Optionally, configure the time zone:

```txt
TIME_ZONE = "TIME_ZONE"
```
Replace <code>TIME_ZONE</code> with an appropriate time zone identifier. For more information, see the list of time zones.

Finalize installation¶

Restart the web server and session storage service:

```txt
# systemctl restart httpd.service memcached.service
```
Note:The systemctl restart command starts each service if not currently running.

Verify operation
---

Verify operation of the dashboard.

Access the dashboard using a web browser at http://controller/dashboard.

Authenticate using admin or demo user and default domain credentials.

备注：默认域、默认用户名、默认密码都在admin-openrc文件中