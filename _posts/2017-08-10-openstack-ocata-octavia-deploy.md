---
layout: post
title: "openstack ocata octavia deploy"
date: 2017-08-10 17:40:22
categories: [openstack,LB]
tags: octavia
---

### octavia组件图

![](https://docs.openstack.org/octavia/latest/_images/octavia-component-overview.svg)

部署分布如下：

* 控制节点：Octavia API
* 网络节点：Octavia Worker, Health Manager, Housekeeping Manager

---

### octavia数据流

![](/assets/octavia/stream.png)
octavia ocata 版本一个LB会在每个租户网络分配两个ip

* 一个IP绑定在eth1:0上，用来绑定VIP
* 一个IP绑定在eth1上，用来和后端通信

---

### octavia 手动部署

#### 控制中心

(1) 配置mysql数据库
~~~sh
mysql -u root -p
~~~

~~~
CREATE DATABASE octavia;
GRANT ALL PRIVILEGES ON octavia.* TO 'octavia'@'localhost'  IDENTIFIED BY 'octavia';
GRANT ALL PRIVILEGES ON octavia.* TO 'octavia'@'%'  IDENTIFIED BY 'octavia';
~~~

(2) 配置keystone
~~~
. admin-openrc
~~~

~~~sh
openstack user create --domain default --password-prompt octavia

openstack role add --project service --user octavia admin

openstack service create --name octavia --description "Octavia Load Balancing Service" octavia
openstack endpoint create --region RegionOne octavia public http://controller:9876
openstack endpoint create --region RegionOne octavia internal http://controller:9876
openstack endpoint create --region RegionOne octavia admin http://controller:9876
~~~

(3) 编译以及上传amphora镜像

配置好centos yum源
~~~sh
git clone --branch ocata https://github.com/openstack/octavia
cd octavia/diskimage-create
sh diskimage-create.sh -i centos -s 2G
openstack image create --tag amphora --public --container-format=bare \
    --disk-format qcow2 --file `pwd`/amphora-x64-haproxy.qcow2 amphora-x64-haproxy
~~~

这边创建完镜像后，可以修改镜像密码，不用秘钥登陆
~~~
LIBGUESTFS_BACKEND=direct guestfish --rw -a amphora-x64-haproxy.qcow2
~~~
~~~
><fs> run
><fs> mount /dev/sda1 /
><fs> vi /etc/cloud/cloud.cfg
><fs> exit
~~~

修改如下：
* lock_passwd: false
* plain_text_passwd: centos

(4) 创建flavor
~~~sh
openstack flavor create --id auto --ram 1024 --disk 2 --vcpus 1 --private m1.amphora -f value -c id
~~~

(5) 安装neutron lbaas 和 octavia api
~~~sh
yum install -y openstack-neutron-lbaas penstack-neutron-lbaas-ui openstack-octavia-api
~~~

(6) 创建互信
~~~sh
ssh-keygen -b 2048 -t rsa -N "" -f /etc/octavia/.ssh/octavia_ssh_key
openstack keypair create --public-key /etc/octavia/.ssh/octavia_ssh_key.pub octavia_ssh_key 
~~~

(7) 创建lb管理网络
~~~sh
openstack network create lb-mgmt-net
openstack subnet create --subnet-range 192.168.0.0/24 --allocation-pool \
    start=192.168.0.2,end=192.168.0.200 --network lb-mgmt-net lb-mgmt-subnet

openstack security group create lb-mgmt-sec-grp
openstack security group rule create --protocol icmp lb-mgmt-sec-grp
openstack security group rule create --protocol tcp --dst-port 22 lb-mgmt-sec-grp
openstack security group rule create --protocol tcp --dst-port 9443 lb-mgmt-sec-grp

openstack security group create lb-health-mgr-sec-grp
openstack security group rule create --protocol udp --dst-port 5555 lb-health-mgr-sec-grp
~~~

(8) 修改neutron配置

/etc/neutron/neutron.conf
~~~
service_plugins = router,neutron_lbaas.services.loadbalancer.plugin.LoadBalancerPluginv2
[octavia]
base_url = http://controller:9876
request_poll_timeout = 3000
~~~

/etc/neutron/neutron_lbaas.conf
~~~sh
[DEFAULT]
[certificates]
[quotas]
[service_auth]
auth_url = http://controller:5000/v2.0
admin_user = admin
admin_tenant_name = admin
admin_password = admin
auth_version = 2
[service_providers]
service_provider = LOADBALANCERV2:Octavia:neutron_lbaas.drivers.octavia.driver.OctaviaDriver:default
~~~

(9) 修改octavia配置

/etc/octavia/octavia.conf
~~~
[DEFAULT]
bind_host = 10.x.x.x
bind_port = 9876
api_handler = queue_producer
host = controller
transport_url = rabbit://openstack:openstack@controller
debug = True
[database]
connection = mysql+pymysql://octavia:octavia@controller:3306/octavia
[keystone_authtoken]
auth_uri = http://controller:5000
auth_url = http://controller:35357
memcached_servers = controller:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = octavia
password = octavia
[certificates]
ca_certificate = /etc/octavia/certs/ca_01.pem
ca_private_key = /etc/octavia/certs/private/cakey.pem
ca_private_key_passphrase = foobar
[anchor]
[networking]
[haproxy_amphora]
base_path = /var/lib/octavia
base_cert_dir = /etc/octavia/certs
connection_max_retries = 1500
connection_retry_interval = 1
rest_request_conn_timeout = 10
rest_request_read_timeout = 120
key_path = /etc/octavia/.ssh/octavia_ssh_key
client_cert = /etc/octavia/certs/client.pem
server_ca = /etc/octavia/certs/ca_01.pem
user_group = haproxy
bind_host = 0.0.0.0
bind_port = 9443
[controller_worker]
amphora_driver = amphora_haproxy_rest_driver
compute_driver = compute_nova_driver
network_driver = allowed_address_pairs_driver
amp_flavor_id = 89a2474c-551a-4a3d-b74b-3d3195e69488 #from step 4
amp_active_retries = 100
amp_active_wait_sec = 2
amp_image_tag = amphora
amp_ssh_key_name = octavia_ssh_key
amp_boot_network_list = 5dd973b0-a283-4e6a-86b3-cd9b77e77880 #from step 7
[task_flow]
[oslo_messaging]
rpc_thread_pool_size = 2
topic = octavia_prov
[house_keeping]
amphora_expiry_age = 3600
load_balancer_expiry_age = 3600
[amphora_agent]
[keepalived_vrrp]
[service_auth]
auth_uri = http://controller:5000
auth_url = http://controller:35357
memcached_servers = controller:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = admin
username = admin
password = admin
[nova]
[glance]
[neutron]
[quotas]
~~~

(10) 重启服务
~~~sh
systemctl restart neutron-server
systemctl enable octavia-api
systemctl start octavia-api
~~~

---

#### 网络节点

(1) 安装octavia rpm 包
~~~sh
yum install -y openstack-octavia-common  openstack-octavia-health-manager \
    openstack-octavia-housekeeping openstack-octavia-worker  python-octavia
~~~

(2) 创建lb管理端口
~~~sh
neutron port-create --name octavia-health-manager-listen-port \
    --security-group lb-health-mgr-sec-grp --device-owner \
	Octavia:health-mgr --binding:host_id=<network_node_host_name> lb-mgmt-net

ovs-vsctl -- --may-exist add-port br-int o-hm0 \
    -- set Interface o-hm0 type=internal -- set Interface o-hm0 \
	external-ids:iface-status=active -- set Interface o-hm0 \
	external-ids:attached-mac=$mac -- set Interface o-hm0 external-ids:iface-id=$portid

ip link set dev o-hm0 address $mac
iptables -I INPUT -i o-hm0 -p udp --dport 5555 -j ACCEPT
ifconfig o-hm0 $ip
~~~
不建议使用dhcp获取ip，会更改默认路由，导致网络断连

(3) 拷贝cert文件

将https://github.com/openstack/octavia/tree/master/devstack/pregenerated/certs
拷贝到/etc/octavia/certs

(4) 创建/var/lib/octavia目录
~~~sh
mkdir -p /var/lib/octavia
chown octavia:octavia /var/lib/octavia/
~~~

(5) octavia配置

/etc/octavia/octavia.conf
~~~
[DEFAULT]
bind_host = 10.x.x.x
bind_port = 9876
api_handler = queue_producer
host = controller
transport_url = rabbit://openstack:openstack@controller
debug = True
[database]
connection = mysql+pymysql://octavia:octavia@controller:3306/octavia
[keystone_authtoken]
auth_uri = http://controller:5000
auth_url = http://controller:35357
memcached_servers = controller:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = octavia
password = octavia
[certificates]
ca_certificate = /etc/octavia/certs/ca_01.pem
ca_private_key = /etc/octavia/certs/private/cakey.pem
ca_private_key_passphrase = foobar
[anchor]
[networking]
[health_manager]
heartbeat_key = insecure
bind_ip = $ip
bind_port = 5555
controller_ip_port_list = $ip:5555
[haproxy_amphora]
base_path = /var/lib/octavia
base_cert_dir = /etc/octavia/certs
connection_max_retries = 1500
connection_retry_interval = 1
rest_request_conn_timeout = 10
rest_request_read_timeout = 120
key_path = /etc/octavia/.ssh/octavia_ssh_key
client_cert = /etc/octavia/certs/client.pem
server_ca = /etc/octavia/certs/ca_01.pem
user_group = haproxy
bind_host = 0.0.0.0
bind_port = 9443
[controller_worker]
amphora_driver = amphora_haproxy_rest_driver
compute_driver = compute_nova_driver
network_driver = allowed_address_pairs_driver
amp_flavor_id = 89a2474c-551a-4a3d-b74b-3d3195e69488
amp_active_retries = 100
amp_active_wait_sec = 2
amp_image_tag = amphora
amp_ssh_key_name = octavia_ssh_key
amp_boot_network_list = 5dd973b0-a283-4e6a-86b3-cd9b77e77880
[task_flow]
[oslo_messaging]
rpc_thread_pool_size = 2
topic = octavia_prov
[house_keeping]
amphora_expiry_age = 3600
load_balancer_expiry_age = 3600
[amphora_agent]
[keepalived_vrrp]
[service_auth]
auth_uri = http://controller:5000
auth_url = http://controller:35357
memcached_servers = controller:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = admin
username = admin
password = admin
[nova]
[glance]
[neutron]
[quotas]
~~~

(6) 启动服务
~~~sh
systemctl enable octavia-worker octavia-health-manager octavia-housekeeping
systemctl start octavia-worker octavia-health-manager octavia-housekeeping
~~~

#### 测试
~~~
. demo-openrc
~~~

~~~
neutron lbaas-loadbalancer-create --name test-lb demo-subnet
neutron lbaas-listener-create --name test-lb-http --loadbalancer e2329654-9b47-48d5-aac5-0ad7be6868fa --protocol HTTP --protocol-port 80
~~~

---

注: octavia手动配置步骤比较多，如有错漏，请指正。
