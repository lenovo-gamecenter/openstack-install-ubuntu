# openstack-install-ubuntu
## Prepare for juno resource on ubuntu
```
apt-get install python-software-properties -y
add-apt-repository cloud-archive:juno -y
apt-get update -y
apt-get dist-upgrade -y
```
## install mysql
```
apt-get install mysql-server python-mysqldb
```
```
vi /etc/mysql/my.cnf
[mysqld]
default-storage-engine = innodb
innodb_file_per_table
collation-server = utf8_general_ci
init-connect = 'SET NAMES utf8'
character-set-server = utf8
```
## install rabbitmq
```
apt-get install rabbitmq-server
```
## install keystone
### create database for keystone(mysql)
```
create database keystone;
grant all privileges on keystone.* to keystone@'localhost' identified by 'openstack';
grant all privileges on keystone.* to keystone@'%' identified by 'openstack';
```
### install keystone
```
apt-get install keystone python-keystoneclient
```
### generate a random token
```
openssl rand -hex 10 >~/ADMIN_TOKEN
```
```
vi /etc/keystone/keystone.conf
[default]
token=
verbose=true
[database]
connection=mysql://keystone:openstack@openstack/keystone
[token]
provider=keystone.token.providers.uuid.Provider
# Token persistence backend driver. (string value)
driver=keystone.token.persistence.backends.sql.Token
```
### sync database for keystone
```
rm -rf /var/lib/keystone/keystone.db
su -s /bin/sh -c "keystone-manage db_sync" keystone
service keystone restart
```
### set env for admin user to create user,tenant,role with keystone command
```
export OS_SERVICE_ENDPOINT=http://openstack:35357/v2.0
export OS_SERVICE_TOKEN=
```
```
keystone tenant-create --name admin --description "Admin Tenant"
keystone user-create --name admin --pass admin --email feiy_2015@sina.com
keystone role-create --name admin
keystone user-role-add --user admin --tenant admin --role admin

keystone role-create --name _member_
keystone tenant-create --name hadoop --description "Hadoop Tenant"
keystone user-create --name hadoop --pass hadoop --email feiy_2015@sina.com
keystone user-role-add --tenant hadoop --user hadoop --role _member_

keystone tenant-create --name service --description "Service Tenant"

keystone service-create --name keystone --type identity --description "Openstack Identity"
+-------------+----------------------------------+
|   Property  |              Value               |
+-------------+----------------------------------+
| description |        Openstack Identity        |
|   enabled   |               True               |
|      id     | 0ef5609b61e34f328f76b89aa8821345 |
|     name    |             keystone             |
|     type    |             identity             |
+-------------+----------------------------------+

keystone endpoint-create --service-id 0ef5609b61e34f328f76b89aa8821345 \
--publicurl http://openstack:5000/v2.0 \
--internalurl http://openstack:5000/v2.0 \
--adminurl http://openstack:35357/v2.0 \
--region regionOne

+-------------+----------------------------------+
|   Property  |              Value               |
+-------------+----------------------------------+
|   adminurl  |   http://openstack:35357/v2.0    |
|      id     | 86249bc723d44fc18b7306aaf7bf46c3 |
| internalurl |    http://openstack:5000/v2.0    |
|  publicurl  |    http://openstack:5000/v2.0    |
|    region   |            regionOne             |
|  service_id | 0ef5609b61e34f328f76b89aa8821345 |
+-------------+----------------------------------+
```
验证:
unset OS_SERVICE_TOKEN OS_SERVICE_ENDPOINT
```
root@openstack:~# keystone --os-tenant-name admin --os-username admin --os-password admin --os-auth-url http://openstack:35357/v2.0 service-list
+----------------------------------+----------+----------+--------------------+
|                id                |   name   |   type   |    description     |
+----------------------------------+----------+----------+--------------------+
| 0ef5609b61e34f328f76b89aa8821345 | keystone | identity | Openstack Identity |
+----------------------------------+----------+----------+--------------------+
root@openstack:~# keystone --os-tenant-name admin --os-username admin --os-password admin --os-auth-url http://openstack:35357/v2.0 token-get
+-----------+----------------------------------+
|  Property |              Value               |
+-----------+----------------------------------+
|  expires  |       2016-11-25T06:32:24Z       |
|     id    | ab2519953b8f48168e6c68c1344f2d13 |
| tenant_id | df6143104b41460798b05d5e4cbbc655 |
|  user_id  | c036a241a331440abd9b9ff1327590d4 |
+-----------+----------------------------------+
root@openstack:~# keystone --os-tenant-name admin --os-username admin --os-password admin --os-auth-url http://openstack:35357/v2.0 role-list
+----------------------------------+----------+
|                id                |   name   |
+----------------------------------+----------+
| 71c260b0a2594737bb4a590836a7510b | _member_ |
| f6c0bcbb4ba84a638e1c1d8eb3d23715 |  admin   |
+----------------------------------+----------+
```
## install glance
### create database for glance(mysql)
```
create database glance;
grant all privileges on glance.* to glance@'localhost' identified by 'openstack';
grant all privileges on glance.* to glance@'%' identified by 'openstack';
```

export OS_SERVICE_TOKEN=655e98d348e4a64ad7f0
export OS_SERVICE_ENDPOINT=http://openstack:35357/v2.0

### set env for admin
```
vi keystonerc_admin.sh
export OS_TENANT_NAME=admin
export OS_USERNAME=admin
export OS_PASSWORD=admin
export OS_AUTH_URL=http://192.168.61.122:35357/v2.0
```
```
keystone user-create --name glance --pass glance --email feiy_2015@sina.com

keystone user-role-add --user glance --tenant service --role admin
keystone service-create --name glance --type image --description "Openstack Image Service"
+-------------+----------------------------------+
|   Property  |              Value               |
+-------------+----------------------------------+
| description |     Openstack Image Service      |
|   enabled   |               True               |
|      id     | 26d0493bdf0c42e8846343ee5ecc4b3a |
|     name    |              glance              |
|     type    |              image               |
+-------------+----------------------------------+
keystone endpoint-create --service-id=26d0493bdf0c42e8846343ee5ecc4b3a --publicurl http://openstack:9292 --internalurl http://openstack:9292 --adminurl http://openstack:9292 --region regionOne
```
### install glance
```
apt-get install glance python-glanceclient
```
```
vi /etc/glance/glance-api.conf
[database]
connection = mysql://glance:openstack@openstack/glance
[keystone_authtoken]
auth_uri=http://openstack:5000/v2.0
identity_uri = http://openstack:35357
admin_tenant_name = service
admin_user = glance
admin_password = glance

[paste_deploy]
flavor=keystone

[glance-store]

filesystem_store_datadir=/var/lib/glance/images
```
```
vi /etc/glance/glance-registy.conf
//一样的配置
[database]
connection = mysql://glance:openstack@openstack/glance
[keystone_authtoken]
auth_uri=http://openstack:5000/v2.0
identity_uri = http://openstack:35357
admin_tenant_name = service
admin_user = glance
admin_password = glance

[paste_deploy]
flavor=keystone
```

### sync database for glance
```
su -s /bin/sh -c "glance-manage db_sync" glance
service glance-registry restart
service glance-api restart
```
### create image with local disk img
```
source .keystonerc_admin

root@openstack:~# glance image-list
Request returned failure status 500.
HTTPInternalServerError (HTTP 500)
//修改配置，让两个配置文件都使用一样的配置


glance image-create --name "cirros" --file "/home/hadoop/cirros-0.3.3-x86_64-disk.img" --disk-format qcow2 --container-format bare --is-public True --progress
[=============================>] 100%
+------------------+--------------------------------------+
| Property         | Value                                |
+------------------+--------------------------------------+
| checksum         | 133eae9fb1c98f45894a4e60d8736619     |
| container_format | bare                                 |
| created_at       | 2017-02-19T19:02:07                  |
| deleted          | False                                |
| deleted_at       | None                                 |
| disk_format      | qcow2                                |
| id               | e90f1aed-58f9-427e-ba24-c440c3dbc04e |
| is_public        | True                                 |
| min_disk         | 0                                    |
| min_ram          | 0                                    |
| name             | cirros-0.3.3                         |
| owner            | 23a84fb4bf9f4078a54adc63627ea224     |
| protected        | False                                |
| size             | 13200896                             |
| status           | active                               |
| updated_at       | 2017-02-19T19:02:09                  |
| virtual_size     | None                                 |
+------------------+--------------------------------------+
root@openstack:~# glance image-list
+--------------------------------------+--------------+-------------+------------------+----------+--------+
| ID                                   | Name         | Disk Format | Container Format | Size     | Status |
+--------------------------------------+--------------+-------------+------------------+----------+--------+
| e90f1aed-58f9-427e-ba24-c440c3dbc04e | cirros-0.3.3 | qcow2       | bare             | 13200896 | active |
+--------------------------------------+--------------+-------------+------------------+----------+--------+
```
##install nova
###create database for nova(mysql)
```
create database nova;
grant all privileges on nova.* to nova@'localhost' identified by 'openstack';
grant all privileges on nova.* to nova@'%' identified by 'openstack';
```
###create nova user with keystone command in cli
```
keystone user-create --name nova --pass nova --email feiy_2015@sina.com
keystone user-role-add --user  nova --tenant service --role admin

keystone service-create --name nova --type compute --description "Openstack Nova Compute"

keystone endpoint-create --service-id=13223e8b93fc488e8d9bbc16b359ed70 --publicurl http://openstack:8774/v2/%\(tenant_id\)s --internalurl http://openstack:8774/v2/%\(tenant_id\)s --adminurl http://openstack:8774/v2/%\(tenant_id\)s --region regionOne

keystone endpoint-create --service-id=15f88fd8b9b1439da7c8a4fb7649f422 \
--publicurl http://openstack:8774/v2/%\(tenant_id\)s \
 --internalurl http://openstack:8774/v2/%\(tenant_id\)s \
 --adminurl http://openstack:8774/v2/%\(tenant_id\)s \
 --region regionOne
```
### install nova
```
apt-get install nova-api nova-cert nova-conductor nova-consoleauth nova-scheduler nova-novncproxy python-novaclient
```
```
vi /etc/nova/nova.conf
[default]
verbose=True
auth_strategy=keystone
rpc_backend=rabbit
rabbit_host=openstack
rabbit_password=guest
my_ip=192.168.61.122
vnc_enabled=True
vncserver_listen=192.168.61.122
vncserver_proxyclient_address=192.168.61.122
[database]
connection=mysql://nova:nova@openstack/nova
[keystone_authtoken]
auth_uri=http://openstack:5000
identity_uri=http://openstack:35357
admin_tenant_name=service
admin_user=nova
admin_password=nova

[glance]
host=openstack
```
### sync database for nova
```
su -s /bin/sh -c "nova-manage db sync" nova
```
### install nova-compute for nova
```
apt-get install nova-compute python-novaclient
```
vi /etc/nova/nova-compute.conf

## install neutron
### create database for neutron(mysql)
```
create database neutron
grant all privileges on neutron.* to neutron@'localhost' identified by 'openstack';
grant all privileges on neutron.* to neutron@'%' identified by 'openstack';
```
### create neutron user with keystone for neutron
```
keystone user-create --name neutron --pass neutron --email feiy_2015@sina.com
keystone user-role-add --user neutron --tenant service --role admin

keystone service-create --name neutron --type network --description "Openstack Network"
keystone endpoint-create --service-id=f85120e3200c4eeaa9b02bdafcc260fb --publicurl http://openstack:9696 --internalurl http://openstack:9696 --adminurl http://openstack:9696 --region regionOne
```
### install neutron
```
apt-get install neutron-server neutron-plugin-ml2 python-neutronclient -y

apt-get install openvswitch-switch openvswitch-datapath-dkms -y
apt-get install neutron-plugin-openvswitch-agent neutron-l3-agent neutron-dhcp-agent ipset -y
```
### add bridge with openvswitch
```
ovs-vsctrl add-br br0
ovs-vsctrl add-port br0 eth0
```
update system variable
```
vi /etc/sysctl.conf
net.ipv4.conf.default.rp_filter = 0
net.ipv4.conf.all.rp_filter = 0
net.ipv4.ip_forward = 1
```
### make it work
```
sysctl -p
```
```
vi /etc/nova/nova.conf
[DEFAULT]
dhcpbridge_flagfile=/etc/nova/nova.conf
dhcpbridge=/usr/bin/nova-dhcpbridge
logdir=/var/log/nova
state_path=/var/lib/nova
lock_path=/var/lock/nova
force_dhcp_release=True
libvirt_use_virtio_for_bridges=True
verbose=True
ec2_private_dns_show_ip=True
api_paste_config=/etc/nova/api-paste.ini
enabled_apis=ec2,osapi_compute,metadata

auth_strategy=keystone
rpc_backend=rabbit
rabbit_host=openstack
rabbit_password=guest

my_ip=192.168.61.122
vncserver_listen=192.168.61.122
vncserver_proxyclient_address=192.168.61.122
novncproxy_base_url=http://openstack:6080/vnc_auto.html

service_neutron_metadata_proxy=true
neutron_metadata_proxy_shared_secret=neutron

network_api_class=nova.network.neutronv2.api.API
security_group_api=neutron
linuxnet_interface_driver=nova.network.linux_net.LinuxOVSInterfaceDriver
firewall_driver=nova.virt.firewall.NoopFirewallDriver
[database]
connection=mysql://nova:nova@openstack/nova
[keystone_authtoken]
auth_uri=http://openstack:5000
identity_uri=http://openstack:35357
admin_tenant_name=service
admin_user=nova
admin_password=nova
[glance]
host=openstack
[neutron]
url=http://openstack:9696
auth_strategy=keystone
admin_auth_url=http://openstack:35357/v2.0
admin_tenant_name=service
admin_username=neutron
admin_password=neutron
```
```
vi /etc/neutron.conf
[DEFAULT]
verbose = True
core_plugin = ml2
service_plugins =router
auth_strategy = keystone
allow_overlapping_ips = True

notify_nova_on_port_status_changes = True
notify_nova_on_port_data_changes=True
nova_url = http://openstack:8774/v2
nova_admin_auth_url=http://openstack:35357/v2.0
nova_region_name =regionOne
nova_admin_username =nova
nova_admin_tenant_id =a4363a87992a4be7aba64be211338b5c
nova_admin_password =nova
nova_admin_auth_url =http://openstack:35357/v2.0

rabbit_host=openstack
rabbit_password=guest
rpc_backend=rabbit

[keystone_authtoken]
auth_host = 192.168.61.122
auth_port = 35357
auth_protocol = http
admin_tenant_name = service
admin_user = neutron
admin_password = neutron

[database]
connection = mysql://neutron:neutron@openstack/neutron
```
```
vi /etc/neutron/plugins/ml2/ml2_conf.ini

[ml2]
type_drivers = local,flat,vlan,gre,vxlan
tenant_network_types = vlan
mechanism_drivers = openvswitch,linuxbridge

[ml2_type_vlan]
network_vlan_ranges = physnet1:1000:2999


[securitygroup]
enable_security_group = True

enable_ipset = True
firewall_driver=neutron.agent.linux.iptables_firewall.OVSHybridIptablesFirewallDriver
[ovs]
local_ip=192.168.56.145
tenant_network_type=vlan
integration_bridge=br-int
network_vlan_ranges=physnet1:1000:2999
bridge_mappings=physnet1:br0
```
```
vi /etc/neutron/l3_agent.ini
[DEFAULT]
verbose=True
interface_driver = neutron.agent.linux.interface.OVSInterfaceDriver
use_namespaces = True
external_network_bridge = br0
```
```
vi /etc/neutron/dhcp_agent.ini
[DEFAULT]
verbose=True
interface_driver = neutron.agent.linux.interface.OVSInterfaceDriver
dhcp_driver = neutron.agent.linux.dhcp.Dnsmasq
use_namespaces = True
```
```
vi /etc/neutron/metadata_agent.ini
[DEFAULT]
verbose=True
auth_url = http://openstack:5000/v2.0
auth_region = regionOne
admin_tenant_name = service
admin_user = neutron
admin_password = neutron
nova_metadata_ip = 192.168.56.145
metadata_proxy_shared_secret =neutron
```
### sync database for neutron
```
su -s /bin/sh -c "neutron-db-manage --config-file /etc/neutron/neutron.conf \
    --config-file /etc/neutron/plugins/ml2/ml2_conf.ini upgrade juno" neutron
```
###nova restart
```
service nova-api restart
service nova-cert restart
service nova-conductor restart
service nova-consoleauth restart
service nova-scheduler restart
service nova-novncproxy restart
service nova-compute restart
```
###neutron restart
```
service neutron-server restart
```
###neutron-agent restart
```
service openvswitch-switch restart
service neutron-plugin-openvswitch-agent restart
service neutron-dhcp-agent restart
service neutron-l3-agent restart
service neutron-metadata-agent restart
```

## install horizon and remove ubuntu theme
```
apt-get install -y openstack-dashboard apache2 libapache2-mod-wsgi memcached python-memcache

 apt-get remove --purge openstack-dashboard-ubuntu-theme
```

##create network
//公有网络
```
root@openstack:~# source keystonerc_admin.sh 
root@openstack:~# neutron net-create public-vlan --router:external=True
Created a new network:
+---------------------------+--------------------------------------+
| Field                     | Value                                |
+---------------------------+--------------------------------------+
| admin_state_up            | True                                 |
| id                        | c3c63eab-2fcc-44fd-a4a4-0b46cba55377 |
| name                      | public-vlan                          |
| provider:network_type     | vlan                                 |
| provider:physical_network | physnet1                             |
| provider:segmentation_id  | 1080                                 |
| router:external           | True                                 |
| shared                    | False                                |
| status                    | ACTIVE                               |
| subnets                   |                                      |
| tenant_id                 | 23a84fb4bf9f4078a54adc63627ea224     |
+---------------------------+--------------------------------------+
root@openstack:~# neutron subnet-create public-vlan --name public-subnet --allocation-pool start=192.168.61.200,end=192.168.61.230 --disable-dhcp --gateway 192.168.61.2 192.168.61.0/24 --dns-nameserver 192.168.61.2
Created a new subnet:
+-------------------+------------------------------------------------------+
| Field             | Value                                                |
+-------------------+------------------------------------------------------+
| allocation_pools  | {"start": "192.168.61.200", "end": "192.168.61.230"} |
| cidr              | 192.168.61.0/24                                      |
| dns_nameservers   | 192.168.61.2                                         |
| enable_dhcp       | False                                                |
| gateway_ip        | 192.168.61.2                                         |
| host_routes       |                                                      |
| id                | 24467a95-5c8f-4fbc-a133-8432b220c5c3                 |
| ip_version        | 4                                                    |
| ipv6_address_mode |                                                      |
| ipv6_ra_mode      |                                                      |
| name              | public-subnet                                        |
| network_id        | c3c63eab-2fcc-44fd-a4a4-0b46cba55377                 |
| tenant_id         | 23a84fb4bf9f4078a54adc63627ea224                     |
+-------------------+------------------------------------------------------+
```

### user network
//用户网络
```
hadoop@openstack:~$ neutron net-create hadoop-vlan 
Created a new network:
+-----------------+--------------------------------------+
| Field           | Value                                |
+-----------------+--------------------------------------+
| admin_state_up  | True                                 |
| id              | 6fa7187f-18cc-46b2-afaa-7636c5acacbd |
| name            | hadoop-vlan                          |
| router:external | False                                |
| shared          | False                                |
| status          | ACTIVE                               |
| subnets         |                                      |
| tenant_id       | 668a527fd9384b639447deaca1cf2c48     |
+-----------------+--------------------------------------+
hadoop@openstack:~$ neutron subnet-create hadoop-vlan --name hadoop-subnet --allocation-pool start=10.0.1.1,end=10.0.1.253 --disable-dhcp --gateway 10.0.1.254 10.0.1.0/24 --dns-nameserver 192.168.61.2 
Created a new subnet:
+-------------------+--------------------------------------------+
| Field             | Value                                      |
+-------------------+--------------------------------------------+
| allocation_pools  | {"start": "10.0.1.1", "end": "10.0.1.253"} |
| cidr              | 10.0.1.0/24                                |
| dns_nameservers   | 192.168.61.2                               |
| enable_dhcp       | False                                      |
| gateway_ip        | 10.0.1.254                                 |
| host_routes       |                                            |
| id                | 30b75ef9-4a57-45b2-9110-5a71188ccd2c       |
| ip_version        | 4                                          |
| ipv6_address_mode |                                            |
| ipv6_ra_mode      |                                            |
| name              | hadoop-subnet                              |
| network_id        | 6fa7187f-18cc-46b2-afaa-7636c5acacbd       |
| tenant_id         | 668a527fd9384b639447deaca1cf2c48           |
+-------------------+--------------------------------------------+
hadoop@openstack:~$ neutron router-create hadoop-router
Created a new router:
+-----------------------+--------------------------------------+
| Field                 | Value                                |
+-----------------------+--------------------------------------+
| admin_state_up        | True                                 |
| external_gateway_info |                                      |
| id                    | 3de91186-f5ae-44e5-8602-a2dda86dae25 |
| name                  | hadoop-router                        |
| routes                |                                      |
| status                | ACTIVE                               |
| tenant_id             | 668a527fd9384b639447deaca1cf2c48     |
+-----------------------+--------------------------------------+
hadoop@openstack:~$ neutron router-interface-add hadoop-router hadoop-subnet
Added interface c48530e3-b317-4cf7-a0cd-1f6ffb79cc72 to router hadoop-router.
hadoop@openstack:~$ neutron router-gateway-set hadoop-router public-vlan
Set gateway for router hadoop-router
```
