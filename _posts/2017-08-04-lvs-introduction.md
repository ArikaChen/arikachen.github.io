---
layout: post
title: "LVS introduction"
date: 2017-08-04 15:12:13
categories: [Linux, LB]
tags: LVS
---

### LVS介绍

LVS是linux virtual server的简写linux虚拟服务器，是一个虚拟的服务器集群系统，它是全球最流行的四层负载均衡开源软件，
可以在unix/linux平台下实现负载均衡集群功能。该项目在1998年5月由章文嵩博士组织成立。

**IPVS发展史**

早在2.2内核时，IPVS就已经以内核补丁的形式出现。

从2.4.23版本开始ipvs软件就是合并到linux内核的常用版本的内核补丁的集合。

从2.4.24以后IPVS已经成为linux官方标准内核的一部分

---

### LVS部署示意图

![](/assets/lvs/deploy.png)

---

### LVS工作模式


#### NAT

![](/assets/lvs/nat.png)

|-------------------------+++++++++-----------------------------------+
| 优点                    |||||||||不足                               | 
|-------------------------|||||||||-----------------------------------|
| 只需在LVS服务器配置VIP  |||||||||效率低，后端server支持数量少       |
| 支持IP端口转换          |||||||||RS上网关地址必须是LB的的内网地址   |
|-------------------------+++++++++-----------------------------------+

* * *

#### TUN

![](/assets/lvs/tun.png)


|-----------------------------+++++++++-----------------------------------+
| 优点                        |||||||||不足                               | 
|-----------------------------|||||||||-----------------------------------|
| WLAN环境加密数据            |||||||||部署复杂，需要隧道支持             |
| 返回数据无需经过LB，并发量大|||||||||返回数据无需经过LB，并发量大       |
|-----------------------------+++++++++-----------------------------------+

* * *

#### DR

![](/assets/lvs/dr.png)

|-----------------------------+++------------------------------------------------+
| 优点                        |||不足                                            | 
|-----------------------------|||------------------------------------------------|
| 修改mac进行转发，性能最高   |||不能跨VLAN，所有RS节点和调度器在一个局域网      |
|                             |||RS上绑定VIP，风险大                             |
|                             |||RS需要配置ARP抑制                               |
|                             |||不支持端口转换                                  |
|-----------------------------+++------------------------------------------------+

* * *

#### FULLNAT

LVS支持NAT/DR/TUNNEL三种转发模式，上述模式在多VLAN网络环境下部署时，存在网络拓扑复杂，运维成本高的问题。

阿里新增转发模式FULLNAT，实现LVS-RealServer间跨VLAN通讯。

![](/assets/lvs/fullnat.png)


|-----------------------------+++++++++-----------------------------------+
| 优点                        |||||||||不足                               | 
|-----------------------------|||||||||-----------------------------------|
| 支持跨vlan通信              |||||||||性能不及NAT模式                    |
| 部署方便                    |||||||||                                   |
|-----------------------------+++++++++-----------------------------------+

---

### LVS 调度算法

#### rr

轮询算法(round robin)，它将请求依次分配给不同的rs节点，也就是RS节点中均摊分配。这种算法简单，但只适合于RS节点处理性能差不多
的情况

#### wrr

加权轮训调度，它将依据不同RS的权值分配任务。权值较高的RS将优先获得任务，并且分配到的连接数将比权值低的RS更多。相同权值的RS得到相同数目
的连接数。

#### dh

目的地址哈希调度（destination hashing）以目的地址为关键字查找一个静态hash表来获得需要的RS

#### sh
源地址哈希调度（source hashing）以源地址为关键字查找一个静态hash表来获得需要的RS

#### lc

最小连接数调度（least-connection）,IPVS表存储了所有活动的连接。LB会比较将连接请求发送到当前连接最少的RS.

#### wlc

加权最小连接数调度，假设各台RS的全职依次为Wi，当前tcp连接数依次为Ti，依次去Ti/Wi为最小的RS作为下一个分配的RS

#### lblc

基于地址的最小连接数调度（locality-based least-connection）：将来自同一个目的地址的请求分配给同一台RS，此时这
台服务器是尚未满负荷的。否则就将这个请求分配给连接数最小的RS，并以它作为下一次分配的首先考虑。

#### lblcr

带复制的基于局部性最少链接(Locality-Based Least Connections with Replication): 调度算法也
是针对目标IP地址的负载均衡｡它与LBLC算法的不同之处是它要维护从一个目标IP地址到一组服务器的映射，而LBLC算法维护从一个目标IP地址到
一台服务器的映射｡该算法根据请求的目标IP地址找出该目标IP地址对应的服务器组，按”最小连接”原则从服务器组中选出一台服务器，若服务器没有超载，
将请求发送到该服务器；若服务器超载，则按“最小连接”原则从这个集群中选出一台服务器，将该服务器加入到服务器组中，将请求发送到该服务器。同时，
当该服务器组有一段时间没有被修改，将最忙的服务器从服务器组中删除，以降低复制的程度。

#### sed

最短延迟调度（Shortest Expected Delay ）: 在WLC基础上改进，Overhead = （ACTIVE+1）*256/加
权，不再考虑非活动状态，把当前处于活动状态的数目+1来实现，数目最小的，接受下次请求，+1的目的是为了考虑加权的时候，非活动连接过多缺陷：当权
限过大的时候，会倒置空闲服务器一直处于无连接状态。

#### nq

永不排队/最少队列调度（Never Queue Scheduling NQ）: 无需队列。如果有台 realserver的连接数＝0就直接分配
过去，不需要再进行sed运算，保证不会有一个主机很空闲。

---

### LVS FULLNAT工作原理

![](/assets/lvs/transfer.png)

引入Local address(内网IP地址)

* 接收到VIP访问地址cip->vip
* LB将cip->vip 转换成lip->rip
* LB同RS通过内网地址通信
* LB收到RS返回的报文rip->lip, 通过映射关系转换成vip->cip发送给客户端


![](/assets/lvs/hook.png)

**下行**

~~~
ip_vs_in->ip_vs_fnat_xmit->tcp_fnat_in_handle->dst_out
~~~

**上行**

~~~
ip_vs_in->handle_response->ip_vs_fnat_response_xmit->ip_vs_fast_response_xmit
~~~

***

#### TOA

FULLNAT模式存在一个问题是会屏蔽掉客户端IP信息

对此在RS端增加一个TOA模块。LVS将客户端IP以及port存放到报文TCP OPTION中RS端HOOK内核函数，获取客户端信息。

![](/assets/lvs/toa.png)

***

#### SynProxy

原生LVS和商用负载均衡设备（F5）相比，缺少安全攻击防御模块

SynProxy参照Linux TCP协议栈中SYN cookie的思想，LVS构造特殊seq的synack包，验证ack包中的ack_seq
是否合法，实现了LVS代理TCP三次握手。

![](/assets/lvs/synproxy.png)

* Client发送SYN包给LVS。
* LVS构造特殊SEQ的SYN ACK包给Client。
* Client回复ACK给LVS。
* LVS验证ACK包中ack_seq是否合法。
* 如果合法，则LVS再和Realserver建立3次握手。

接收客户端SYN报文，构造SYNACK报文
~~~
ip_vs_pre_routing->ip_vs_synproxy_syn_rcv
~~~

接收客户端ACK报文，校验ack_seq是否合法并构造syn报文发送RS
~~~
ip_vs_in->tcp_conn_schedule->ip_vs_synproxy_ack_rcv->syn_proxy_send_rs_syn
~~~

接收RS SYNACK报文并发送ACK报文完成握手
~~~
ip_vs_in->handle_response->ip_vs_synproxy_synack_rcv
~~~

---

### 其他

**性能优化**

* 网卡多队列中断绑定
* 内核参数调优
* percpu


**集群部署**

* keepalived结合ospf进行LVS集群部署，等价路由增强LVS健壮性，可横向扩展。参考[lbmanager][lbmanager]

[lbmanager]: https://github.com/ArikaChen/lbmanager

**统计信息**

* 支持连接数，入报文数，出报文数，入口流量，出口流量统计
* 支持cps, inpps, outpps, inbps, outbps统计

---

### 参考

* [LVS专题](http://atong.blog.51cto.com/2393905/1347895)
* [阿里云负载均衡技术浅析](https://help.aliyun.com/knowledge_detail/39444.html)
* [腾讯云负载均衡](https://www.qcloud.com/document/product/214/530#3.-.E9.AB.98.E5.8F.AF.E9.9D.A0.E5.AE.9E.E7.8E.B0)
* [阿里FULLNAT](http://kb.linuxvirtualserver.org/wiki/IPVS_FULLNAT_and_SYNPROXY)
* [FULLNAT部署](http://navyaijm.blog.51cto.com/4647068/1346053/)

