Networking service
===

This chapter explains how to install and configure the Networking service (neutron) using the provider networks or self-service networks option.

For more information about the Networking service including virtual networking components, layout, and traffic flows, see the OpenStack Networking Guide.

Networking service overview
---

Contents

OpenStack Networking (neutron) allows you to create and attach interface devices managed by other OpenStack services to networks. Plug-ins can be implemented to accommodate different networking equipment and software, providing flexibility to OpenStack architecture and deployment.

It includes the following components:

neutron-server
---

Accepts and routes API requests to the appropriate OpenStack Networking plug-in for action.

OpenStack Networking plug-ins and agents
----

Plug and unplug ports, create networks or subnets, and provide IP addressing. These plug-ins and agents differ depending on the vendor and technologies used in the particular cloud. OpenStack Networking ships with plug-ins and agents for Cisco virtual and physical switches, NEC OpenFlow products, Open vSwitch, Linux bridging, and the VMware NSX product.

    The common agents are L3 (layer 3), DHCP (dynamic host IP addressing), and a plug-in agent.

Messaging queue
---

Used by most OpenStack Networking installations to route information between the neutron-server and various agents. Also acts as a database to store networking state for particular plug-ins.

OpenStack Networking mainly interacts with OpenStack Compute to provide networks and connectivity for its instances.

Install and configure controller node 在控制节点上操作
---

Prerequisites¶

Before you configure the OpenStack Networking (neutron) service, you must create a database, service credentials, and API endpoints.

1 . To create the database, complete these steps:

Use the database access client to connect to the database server as the root user:

```txt
        $ mysql -u root -p
```
 Create the neutron database:

```txt
        MariaDB [(none)] CREATE DATABASE neutron;
```
Grant proper access to the neutron database, replacing NEUTRON_DBPASS with a suitable password:

```txt
        MariaDB [(none)]> GRANT ALL PRIVILEGES ON neutron.* TO 'neutron'@'localhost' IDENTIFIED BY 'NEUTRON_DBPASS';
        MariaDB [(none)]> GRANT ALL PRIVILEGES ON neutron.* TO 'neutron'@'controller' IDENTIFIED BY 'NEUTRON_DBPASS';
        MariaDB [(none)]> GRANT ALL PRIVILEGES ON neutron.* TO 'neutron'@'%' IDENTIFIED BY 'NEUTRON_DBPASS';
        MariaDB [(none)]> FLUSH PRIVILEGES; 
```
        Exit the database access client.

验证数据库

```txt
[root@controller openstack]# mysql -h controller -u neutron -p
Enter password: 
Welcome to the MariaDB monitor.  Commands end with ; or \g.
Your MariaDB connection id is 14640
Server version: 10.1.20-MariaDB MariaDB Server

Copyright (c) 2000, 2016, Oracle, MariaDB Corporation Ab and others.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

MariaDB [(none)]> exit
Bye
```

2 .  Source the admin credentials to gain access to admin-only CLI commands:

```txt
    $ . admin-openrc
```
3 . To create the service credentials, complete these steps:

Create the neutron user:

```txt
    $ openstack user create --domain default --password-prompt neutron

User Password:
Repeat User Password:
+---------------------+----------------------------------+
| Field               | Value                            |
+---------------------+----------------------------------+
| domain_id           | default                          |
| enabled             | True                             |
| id                  | f0c405573ebc4bcc9157b07ce92cef1c |
| name                | neutron                          |
| options             | {}                               |
| password_expires_at | None                             |
+---------------------+----------------------------------+
```

Add the admin role to the neutron user:

```txt
    $ openstack role add --project service --user neutron admin
```
 Note:This command provides no output.这行无输出

Create the neutron service entity:

```txt
    $ openstack service create --name neutron \
      --description "OpenStack Networking" network

+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description | OpenStack Networking             |
| enabled     | True                             |
| id          | e15e826a1a674dbdac6f366a545b3d59 |
| name        | neutron                          |
| type        | network                          |
+-------------+----------------------------------+
```
4 . Create the Networking service API endpoints:

```txt
$ openstack endpoint create --region RegionOne \
  network public http://controller:9696

+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | 46480ed56b3d417bbbc158fb7dc7629d |
| interface    | public                           |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | e15e826a1a674dbdac6f366a545b3d59 |
| service_name | neutron                          |
| service_type | network                          |
| url          | http://controller:9696           |
+--------------+----------------------------------+

$ openstack endpoint create --region RegionOne \
  network internal http://controller:9696

+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | 094246cea7e444d990623127c1f90236 |
| interface    | internal                         |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | e15e826a1a674dbdac6f366a545b3d59 |
| service_name | neutron                          |
| service_type | network                          |
| url          | http://controller:9696           |
+--------------+----------------------------------+

$ openstack endpoint create --region RegionOne \
  network admin http://controller:9696

+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | 7fe5ac4bbd0540149617c2afdbba9b00 |
| interface    | admin                            |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | e15e826a1a674dbdac6f366a545b3d59 |
| service_name | neutron                          |
| service_type | network                          |
| url          | http://controller:9696           |
+--------------+----------------------------------+
```

Configure networking options¶
---

You can deploy the Networking service using one of two architectures represented by options 1 and 2.配置网络选择1和选择2

Option 1 deploys the simplest possible architecture that only supports attaching instances to provider (external) networks. No self-service (private) networks, routers, or floating IP addresses. Only the admin or other privileged user can manage provider networks.
简单模式：只提供外部网络，不提供私有网络、路由、浮动ip(浮动IP是针对路由选择的)

Option 2 augments option 1 with layer-3 services that support attaching instances to self-service networks. The demo or other unprivileged user can manage self-service networks including routers that provide connectivity between self-service and provider networks. Additionally, floating IP addresses provide connectivity to instances using self-service networks from external networks such as the Internet.

Self-service networks typically use overlay networks. Overlay network protocols such as VXLAN include additional headers that increase overhead and decrease space available for the payload or user data. Without knowledge of the virtual network infrastructure, instances attempt to send packets using the default Ethernet maximum transmission unit (MTU) of 1500 bytes. The Networking service automatically provides the correct MTU value to instances via DHCP. However, some cloud images do not use DHCP or ignore the DHCP MTU option and require configuration using metadata or a script.

Configure the metadata agent
---

安装相关软件

```txt
yum install openstack-neutron openstack-neutron-ml2 \
  openstack-neutron-linuxbridge ebtables
```
The metadata agent provides configuration information such as credentials to instances.

Edit the <code>/etc/neutron/metadata_agent.ini</code> file and complete the following actions:

In the <code>[DEFAULT]</code> section, configure the metadata host and shared secret:

```txt
        [DEFAULT]
        # ...
        nova_metadata_ip = controller
        metadata_proxy_shared_secret = METADATA_SECRET

        Replace METADATA_SECRET with a suitable secret for the metadata proxy
```
Configure the Compute service to use the Networking service¶

Edit the <code>/etc/nova/nova.conf</code> file and perform the following actions:

In the <code>[neutron]</code> section, configure access parameters, enable the metadata proxy, and configure the secret:

```txt
        [neutron]
        # ...
        url = http://controller:9696
        auth_url = http://controller:35357
        auth_type = password
        project_domain_name = default
        user_domain_name = default
        region_name = RegionOne
        project_name = service
        username = neutron
        password = NEUTRON_PASS
        service_metadata_proxy = true
        metadata_proxy_shared_secret = METADATA_SECRET
```
Replace <code>NEUTRON_PASS</code> with the password you chose for the neutron user in the Identity service.

Replace <code>METADATA_SECRET</code> with the secret you chose for the metadata proxy.

Finalize installation¶
---

The Networking service initialization scripts expect a symbolic link <code>/etc/neutron/plugin.ini<code> pointing to the ML2 plug-in configuration file, <code>/etc/neutron/plugins/ml2/ml2_conf.ini<code>. If this symbolic link does not exist, create it using the following command:

```txt
    # ln -s /etc/neutron/plugins/ml2/ml2_conf.ini /etc/neutron/plugin.ini
```
Populate the database:

```txt
    # su -s /bin/sh -c "neutron-db-manage --config-file /etc/neutron/neutron.conf \
      --config-file /etc/neutron/plugins/ml2/ml2_conf.ini upgrade head" neutron
```
Restart the Compute API service:

```txt
# systemctl restart openstack-nova-api.service
```
Start the Networking services and configure them to start when the system boots.

For both networking options:

```txt
# systemctl enable neutron-server.service \
  neutron-linuxbridge-agent.service neutron-dhcp-agent.service \
  neutron-metadata-agent.service
# systemctl start neutron-server.service \
  neutron-linuxbridge-agent.service neutron-dhcp-agent.service \
  neutron-metadata-agent.service
```
For networking option 2, also enable and start the layer-3 service:

```txt
# systemctl enable neutron-l3-agent.service
# systemctl start neutron-l3-agent.service
```
Networking Option 1: Provider networks
---

Install and configure the Networking components on the controller node.在控制节点上

Install the components¶

```txt
# yum install openstack-neutron openstack-neutron-ml2 \
  openstack-neutron-linuxbridge ebtables
```
Configure the server component¶

The Networking server component configuration includes the database, authentication mechanism, message queue, topology change notifications, and plug-in.

Edit the <code>/etc/neutron/neutron.conf</code> file and complete the following actions:

In the <code>[database]</code> section, configure database access:

```txt
    [database]
    # ...
    connection = mysql+pymysql://neutron:NEUTRON_DBPASS@controller/neutron
```
Replace <code>NEUTRON_DBPASS</code> with the password you chose for the database.

In the <code>[DEFAULT]</code> section, enable the Modular Layer 2 (ML2) plug-in and disable additional plug-ins:

```txt
[DEFAULT]
# ...
core_plugin = ml2
service_plugins =
```
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
username = neutron
password = NEUTRON_PASS
```
Replace <code>NEUTRON_PASS</code> with the password you chose for the neutron user in the Identity service.

In the <code>[DEFAULT]</code> and <code>[nova]</code> sections, configure Networking to notify Compute of network topology changes:

```txt
    [DEFAULT]
    # ...
    notify_nova_on_port_status_changes = true
    notify_nova_on_port_data_changes = true

    [nova]
    # ...
    auth_url = http://controller:35357
    auth_type = password
    project_domain_name = default
    user_domain_name = default
    region_name = RegionOne
    project_name = service
    username = nova
    password = NOVA_PASS
```
Replace <code>NOVA_PASS</code> with the password you chose for the nova user in the Identity service.

In the <code>[oslo_concurrency]</code> section, configure the lock path:

```txt
    [oslo_concurrency]
    # ...
    lock_path = /var/lib/neutron/tmp
```
Configure the Modular Layer 2 (ML2) plug-in¶
---

The ML2 plug-in uses the Linux bridge mechanism to build layer-2 (bridging and switching) virtual networking infrastructure for instances.

Edit the <code>/etc/neutron/plugins/ml2/ml2_conf.ini</code> file and complete the following actions:

In the [ml2] section, enable flat and VLAN networks:

```txt
        [ml2]
        # ...
        type_drivers = flat,vlan
```
        In the [ml2] section, disable self-service networks:
```txt
        [ml2]
        # ...
        tenant_network_types =
```
        In the [ml2] section, enable the Linux bridge mechanism:

```txt
        [ml2]
        # ...
        mechanism_drivers = linuxbridge
```      
Warning

After you configure the ML2 plug-in, removing values in the type_drivers option can lead to database inconsistency.

In the <code>[ml2]</code> section, enable the port security extension driver:
```txt
        [ml2]
        # ...
        extension_drivers = port_security
```
In the <code>[ml2_type_flat]</code> section, configure the provider virtual network as a flat network:

```txt
        [ml2_type_flat]
        # ...
        flat_networks = provider
```
In the <code>[securitygroup]</code> section, enable ipset to increase efficiency of security group rules:

```txt
        [securitygroup]
        # ...
        enable_ipset = true
```
Configure the Linux bridge agent¶

The Linux bridge agent builds layer-2 (bridging and switching) virtual networking infrastructure for instances and handles security groups.

Edit the <code>/etc/neutron/plugins/ml2/linuxbridge_agent.ini</code> file and complete the following actions:

In the <code>[linux_bridge]</code> section, map the provider virtual network to the provider physical network interface:

```txt
        [linux_bridge]
        physical_interface_mappings = provider:PROVIDER_INTERFACE_NAME
```
Replace <code>PROVIDER_INTERFACE_NAME</code> with the name of the underlying provider physical network interface. See Host networking for more information.

In the <code>[vxlan]</code> section, disable VXLAN overlay networks:

```txt
        [vxlan]
        enable_vxlan = false
```
In the <code>[securitygroup]</code> section, enable security groups and configure the Linux bridge iptables firewall driver:

```txt
        [securitygroup]
        # ...
        enable_security_group = true
        firewall_driver = neutron.agent.linux.iptables_firewall.IptablesFirewallDriver
```

Configure the DHCP agent¶

The DHCP agent provides DHCP services for virtual networks.

Edit the <code>/etc/neutron/dhcp_agent.ini</code> file and complete the following actions:

In the <code>[DEFAULT]</code> section, configure the Linux bridge interface driver, Dnsmasq DHCP driver, and enable isolated metadata so instances on provider networks can access metadata over the network:

```txt
        [DEFAULT]
        # ...
        interface_driver = linuxbridge
        dhcp_driver = neutron.agent.linux.dhcp.Dnsmasq
        enable_isolated_metadata = true
```
Return to Networking controller node configuration.

Networking Option 2: Self-service networks
---

Install and configure the Networking components on the controller node.

Install the components¶

```txt
# yum install openstack-neutron openstack-neutron-ml2 \
  openstack-neutron-linuxbridge ebtables
```
Configure the server component¶
---
Edit the <code>/etc/neutron/neutron.conf</code> file and complete the following actions:

In the <code>[database]</code> section, configure database access:

```txt
        [database]
        # ...
        connection = mysql+pymysql://neutron:NEUTRON_DBPASS@controller/neutron
```
Replace <code>NEUTRON_DBPASS</code> with the password you chose for the database.
    
Note:Comment out or remove any other connection options in the <code>[database]</code> section.

In the <code>[DEFAULT]</code> section, enable the Modular Layer 2 (ML2) plug-in, router service, and overlapping IP addresses:

```txt
        [DEFAULT]
        # ...
        core_plugin = ml2
        service_plugins = router
        allow_overlapping_ips = true
```
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
        username = neutron
        password = NEUTRON_PASS
```
Replace <code>NEUTRON_PASS</code> with the password you chose for the neutron user in the Identity service.

Note:Comment out or remove any other options in the <code>[keystone_authtoken]</code> section.

In the <code>[DEFAULT]</code> and <code>[nova]</code> sections, configure Networking to notify Compute of network topology changes:

```txt
        [DEFAULT]
        # ...
        notify_nova_on_port_status_changes = true
        notify_nova_on_port_data_changes = true

        [nova]
        # ...
        auth_url = http://controller:35357
        auth_type = password
        project_domain_name = default
        user_domain_name = default
        region_name = RegionOne
        project_name = service
        username = nova
        password = NOVA_PASS
```
Replace <code>NOVA_PASS</code> with the password you chose for the nova user in the Identity service.

In the <code>[oslo_concurrency]</code> section, configure the lock path:

```txt
        [oslo_concurrency]
        # ...
        lock_path = /var/lib/neutron/tmp
```

Configure the Modular Layer 2 (ML2) plug-in¶

The ML2 plug-in uses the Linux bridge mechanism to build layer-2 (bridging and switching) virtual networking infrastructure for instances.

Edit the <code>/etc/neutron/plugins/ml2/ml2_conf.ini</code> file and complete the following actions:In the [ml2] section, enable flat, VLAN, and VXLAN networks:

```txt
        [ml2]
        # ...
        type_drivers = flat,vlan,vxlan
```
In the <code>[ml2]</code> section, enable VXLAN self-service networks:

```txt
        [ml2]
        # ...
        tenant_network_types = vxlan
```
In the <code>[ml2]</code> section, enable the Linux bridge and layer-2 population mechanisms:

```txt
        [ml2]
        # ...
        mechanism_drivers = linuxbridge,l2population
```      
Warning:After you configure the ML2 plug-in, removing values in the type_drivers option can lead to database inconsistency.
Note    :The Linux bridge agent only supports VXLAN overlay networks.

In the <code>[ml2]</code> section, enable the port security extension driver:

```txt
        [ml2]
        # ...
        extension_drivers = port_security
```
In the <code>[ml2_type_flat]</code> section, configure the provider virtual network as a flat network:

```txt
        [ml2_type_flat]
        # ...
        flat_networks = provider
```
In the <code>[ml2_type_vxlan]</code> section, configure the VXLAN network identifier range for self-service networks:

```txt
        [ml2_type_vxlan]
        # ...
        vni_ranges = 1:1000
```
In the <code>[securitygroup]</code> section, enable ipset to increase efficiency of security group rules:

```txt
        [securitygroup]
        # ...
        enable_ipset = true
```
Configure the Linux bridge agent¶

The Linux bridge agent builds layer-2 (bridging and switching) virtual networking infrastructure for instances and handles security groups.

Edit the <code>/etc/neutron/plugins/ml2/linuxbridge_agent.ini</code> file and complete the following actions:

In the <code>[linux_bridge]</code> section, map the provider virtual network to the provider physical network interface:

```txt
        [linux_bridge]
        physical_interface_mappings = provider:PROVIDER_INTERFACE_NAME
```
Replace <code>PROVIDER_INTERFACE_NAME</code> with the name of the underlying provider physical network interface. See Host networking for more information.

In the <code>[vxlan]</code> section, enable VXLAN overlay networks, configure the IP address of the physical network interface that handles overlay networks, and enable layer-2 population:

```txt
        [vxlan]
        enable_vxlan = true
        local_ip = OVERLAY_INTERFACE_IP_ADDRESS
        l2_population = true
```
Replace <code>OVERLAY_INTERFACE_IP_ADDRESS</code> with the IP address of the underlying physical network interface that handles overlay networks. The example architecture uses the management interface to tunnel traffic to the other nodes. Therefore, replace <code>OVERLAY_INTERFACE_IP_ADDRESS</code> with the management IP address of the controller node. See Host networking for more information.

In the <code>[securitygroup]</code> section, enable security groups and configure the Linux bridge iptables firewall driver:

```txt
        [securitygroup]
        # ...
        enable_security_group = true
        firewall_driver = neutron.agent.linux.iptables_firewall.IptablesFirewallDriver
```
Configure the layer-3 agent¶
---

The Layer-3 (L3) agent provides routing and NAT services for self-service virtual networks.

Edit the <code>/etc/neutron/l3_agent.ini</code> file and complete the following actions:

In the <code>[DEFAULT]</code> section, configure the Linux bridge interface driver and external network bridge:

```txt
        [DEFAULT]
        # ...
        interface_driver = linuxbridge
```
Configure the DHCP agent¶

The DHCP agent provides DHCP services for virtual networks.

Edit the <code>/etc/neutron/dhcp_agent.ini</code> file and complete the following actions:

In the <code>[DEFAULT]</code> section, configure the Linux bridge interface driver, Dnsmasq DHCP driver, and enable isolated metadata so instances on provider networks can access metadata over the network:

```txt
        [DEFAULT]
        # ...
        interface_driver = linuxbridge
        dhcp_driver = neutron.agent.linux.dhcp.Dnsmasq
        enable_isolated_metadata = true
```

Install and configure compute node
===

The compute node handles connectivity and security groups for instances.

Install the components¶

```txt
# yum install openstack-neutron-linuxbridge ebtables ipset
```
Configure the common component¶

The Networking common component configuration includes the authentication mechanism, message queue, and plug-in.

Edit the <code>/etc/neutron/neutron.conf</code> file and complete the following actions:

In the <code>[database]</code> section, comment out any connection options because compute nodes do not directly access the database.

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
        username = neutron
        password = NEUTRON_PASS
```
Replace <code>NEUTRON_PASS</code> with the password you chose for the neutron user in the Identity service.
         
Note:Comment out or remove any other options in the <code>[keystone_authtoken]</code> section.

In the <code>[oslo_concurrency]</code> section, configure the lock path:

```txt
        [oslo_concurrency]
        # ...
        lock_path = /var/lib/neutron/tmp
```
Configure networking options¶

Choose the same networking option that you chose for the controller node to configure services specific to it. Afterwards, return here and proceed to Configure the Compute service to use the Networking service.

Networking Option 1: Provider networks

Networking Option 2: Self-service networks

Configure the Compute service to use the Networking service¶

Edit the <code>/etc/nova/nova.conf</code> file and complete the following actions:

In the <code>[neutron]</code> section, configure access parameters:

```
        [neutron]
        # ...
        url = http://controller:9696
        auth_url = http://controller:35357
        auth_type = password
        project_domain_name = default
        user_domain_name = default
        region_name = RegionOne
        project_name = service
        username = neutron
        password = NEUTRON_PASS
```
Replace <code>NEUTRON_PASS</code> with the password you chose for the neutron user in the Identity service.

Finalize installation¶

Restart the Compute service:

```txt
    # systemctl restart openstack-nova-compute.service
```
Start the Linux bridge agent and configure it to start when the system boots:

```txt
    # systemctl enable neutron-linuxbridge-agent.service
    # systemctl start neutron-linuxbridge-agent.service
```
