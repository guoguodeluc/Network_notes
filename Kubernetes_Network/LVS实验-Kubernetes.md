[TOC]

# LVS实验-Kubernetes

LVS (Linux Virtual Server)是基于内核态的 netfilter 框架实现的 IPVS (IP Virtual Server)功能。对比于Nginx等代理，LVS所有工作都在内核态运行，不涉及数据拷贝和内核态/用户态切换，性能是其最大的优势。

> [LVS官网](http://www.linuxvirtualserver.org/)

## 准备和说明

本文主要是模拟kubernetes的kube-proxy的ipvs模式，并且是LVS的LVS-NAT模式，实验是从简单入难。实验1是简单的ipvs的实现，实验2加入了ipset的ip集管理，实验3加入dummy网卡，实验4是比较综合实验加入calico网络实现。

> ipvs 是一个内核态的四层负载均衡，支持NAT、Gateway以及IPIP隧道模式，Gateway模式性能最好，但LB和RS不能跨子网，IPIP性能次之，通过ipip隧道解决跨网段传输问题，因此能够支持跨子网。而NAT模式没有限制，这也是唯一一种支持端口映射的模式。
>
> 由于 Kubernetes Service 需要使用端口映射功能，因此kube-proxy必然只能使用ipvs的NAT模式。

```bash
VIP:               169.254.1.100/169.254.2.100
Port:              http  80/TCP
Endpoints1:        10.130.17.30:8888,10.130.17.31:8888
Endpoints2:        172.16.0.10:8888,172.16.1.11:8888
```

>  Endpoints1是Host启动服务，Endpoints2是namespace启动服务。VIP是可以设置任意ip，原则上不冲突HOST和NS的地址。
>
> 169.254.0.0/16是一个本地链接地址段，在IP网络里，每台主机都需要一个IP地址，通常情况下是通过DHCP服务器自动分配，但某些特殊情况下，DHCP分配失败或者没有DHCP服务器时，机器可以自己分配一个IP来完成这个工作。[rfc3927](https://datatracker.ietf.org/doc/html/rfc3927)

### 安装包

```bash
yum install -y ipvsadm curl wget tcpdump ipset conntrack-tools
```

> ipvsadm是ipvs管理工具；ipset 是ip集管理工具。
>
> ipvs ：工作于内核空间，主要用于使用户定义的策略生效
>
> ipvsadm : 工作于用户空间，主要用于用户定义和管理集群服务的工具

### 设置内核参数

```
sysctl --write net.ipv4.ip_forward=1
sysctl --write net.ipv4.vs.conntrack=1
```

> k8s 的 snat 是在 mangle 表的 POSTROUTING 链上进行的，需要开启以下参数net.ipv4.vs.conntrack=1，才能实现这一功能。

### 清理环境

```bash
cat > clean-environment.sh <<EOF
vip=169.254.1.100
host_interface=eth0
ipvsadm --clear
iptables -F -tnat && iptables -X -tnat
ipset destroy
pkill python
ip netns delete ns0
ip link delete dummy-ipvs0
ip link delete dummy0
ip address del "\$vip"/32 dev "\$host_interface"
EOF
```

> 每次实验完，都可以执行该命令，来进行其他实验。以上命令都是危险操作。bash clean-environment.sh

### 查看现有配置状态

```bash
cat > show-enviroment.sh <<EOF
ipvsadm -ln
iptables -S -tnat
ipset list
ps aux |grep python
ip netns
ip a
EOF
```

> 需要时来查看实验结果。

### 启动后端服务

```bash
[ -d  /tmp/lu-`hostname -I` ] || mkdir /tmp/lu-`hostname -I |awk '{print $1}'`
cd /tmp/lu-`hostname -I` && echo `hostname -I` > lu
## host启动服务
nohup python -m SimpleHTTPServer 8888 &
## namespace启动服务
ip netns exec ns0 nohup python -m SimpleHTTPServer 8888 &
```

> 后端使用python的SimpleHTTPServer启动的，也可以使用其他工具启动。

## 实验1-Host-ipvs-iptables

### 实验1说明

ipvs实验，vip配置在单个Host的真实的网卡上，ipvs实现dnat，iptables实现snat。后端使用真实的Host启动简单的web服务，在配置vip的节点访问vip，并验证联通性。

> 进行实验前先清除环境，然后在继续。

### 实验1架构图

```ini
## ip地址分布
|------------Host0------------|            |------------Host1------------|
|                             |            |                             |
|  [10.130.17.30/24]eth0      |<- Network->|  [10.130.17.31/24]eth0      |
|  [169.254.1.100/32]eth0 vip |            |  [169.254.1.100/32]eth0 vip |
|                             |            |                             |
|------------------------------------------------------------------------|
## 代理数据流
                   |——> 10.130.17.30 realserver
169.254.1.100VIP-->|
                   |——> 10.130.17.31 realserver
## 命令结果                  
# ipvsadm -ln
TCP  169.254.1.100:80 rr
  -> 10.130.17.30:8888  
  -> 10.130.17.31:8888  
# iptables -S -tnat
-A POSTROUTING -m ipvs --vaddr 169.254.1.100 --vport 80 -j MASQUERADE
```

> 如果是openstack开启的虚机需要设置allowed-address-pairs

### 实验1命令

设置ipvs-->设置iptables-->启服务

- 设置变量

```bash
vip=169.254.1.100
vip_interface=eth0
vip_port=80
rip_port=8888
rip1=10.130.17.30
rip2=10.130.17.31
```

> rip1和rip2是Host地址，vip可以任意设置，只要不和Host地址冲突。
>

- 执行命令

```bash
ipvsadm --add-service --tcp-service "$vip":"$vip_port" --scheduler rr
ipvsadm --add-server --tcp-service "$vip":"$vip_port"  --real-server "$rip1":"$rip_port"  --masquerading 
ipvsadm --add-server --tcp-service "$vip":"$vip_port"  --real-server "$rip2":"$rip_port"  --masquerading 
ip address add "$vip"/32 dev "$vip_interface"

iptables -t nat -A POSTROUTING -m ipvs --vaddr "$vip" --vport "$vip_port" -j MASQUERADE
sysctl -w net.ipv4.vs.conntrack=1

# 启动后端服务
[ -d  /tmp/lu-`hostname -I` ] || mkdir /tmp/lu-`hostname -I |awk '{print $1}'`
cd /tmp/lu-`hostname -I` && echo `hostname -I` > lu
## host启动服务
nohup python -m SimpleHTTPServer 8888 &
```

> lvs 做了 DNAT 并没有做 SNAT ，所以我们利用 iptables 做 SNAT ;
>
> nat 是依赖 conntrack 的，而 IPVS 默认不会记录 conntrack，我们需要开启 IPVS 的 conntrack 才可以让 MASQUERADE 生效;
>
> iptables的ipvs帮助：iptables -m ipvs --help

### 实验1验证

```bash
# curl 169.254.1.100/lu
10.130.17.31 169.254.1.100
# curl 169.254.1.100/lu
10.130.17.30 169.254.1.100
```

## 实验2-Host-ipvs-iptables-ipset

### 实验2说明

ipvs实验，vip配置在两个Host的真实的网卡上，ipvs实现dnat，iptables和ipset实现snat。后端使用真实的Host启动简单的web服务，在配置vip的节点访问vip，并验证联通性。

> 进行实验前先清除环境，然后在继续。

### 实验2架构图

```ini
## ip地址分布
|------------Host0------------|            |------------Host1------------|
|                             |            |                             |
|  [10.130.17.30/24]eth0      |<- Network->|  [10.130.17.31/24]eth0      |
|  [169.254.1.100/32]eth0 vip |            |  [169.254.1.100/32]eth0 vip |
|                             |            |                             |
|------------------------------------------------------------------------|
## 代理数据流
                   |——> 10.130.17.30 realserver
169.254.1.100VIP-->|
                   |——> 10.130.17.31 realserver
## 命令结果
# ipvsadm -ln
TCP  169.254.1.100:80 rr
  -> 10.130.17.30:8888  
  -> 10.130.17.31:8888  
# iptables -S -tnat
-P PREROUTING ACCEPT
-P INPUT ACCEPT
-P OUTPUT ACCEPT
-P POSTROUTING ACCEPT
-N LU-MARK-MASQ
-N LU-POSTROUTING
-N LU-SERVICES
-A PREROUTING -m comment --comment "lu service portals" -j LU-SERVICES
-A OUTPUT -m comment --comment "lu service portals" -j LU-SERVICES
-A POSTROUTING -m comment --comment "lu postrouting rules" -j LU-POSTROUTING
-A LU-MARK-MASQ -j MARK --set-xmark 0x2000/0x2000
-A LU-POSTROUTING -m comment --comment "lu service traffic requiring SNAT" -m mark --mark 0x2000/0x2000 -j MASQUERADE
-A LU-SERVICES -m comment --comment "lu service cluster ip + port for masquerade purpose" -m set --match-set LU-SERVICE-IP dst,dst -j LU-MARK-MASQ
# ipset list
Name: LU-SERVICE-IP
Type: hash:ip,port
Members:
169.254.1.100,tcp:80
```

> 如果是openstack开启的虚机需要设置allowed-address-pairs

#### 访问详解

##### 访问本地的后端

在Host上访问本机的 IP 端口的时候是内核直接转发给本地进程，这种数据包会经过 OUTPUT 链，发现是本地ip地址，经过INPUT链，返回结果，不会过 PREROUTING链。

Client-->OUTPUT 链--> Server-->INPUT链 --> Client 

##### 访问非本地的后端

Client--> OUTPUT 链-->POSTROUTING链-->Server--> PREROUTING链-->INPUT链--Client链

##### nat表

看nat表设置路径

PREROUTING链-->自建链LU-SERVICES(匹配ipset hash表)-->自建链LU-MARK-MASQ-->MARK打标签

POSTROUTING链-->自建链LU-POSTROUTING-->MARK-->MASQUERADE

OUTPUT链-->自建链LU-SERVICES(匹配ipset hash表)-->自建链LU-MARK-MASQ-->MARK打标签



### 实验2命令

设置ipvs-->设置iptables--设置ipset-->启服务

- 设置变量

```bash
# 实验1变量会在实验2中继续使用
vip=169.254.1.100
vip_interface=eth0
vip_port=80
rip_port=8888
rip1=10.130.17.30
rip2=10.130.17.31
# 实验2新增部分
chain_services=LU-SERVICES
chain_mark_masq=LU-MARK-MASQ
chain_postrouting=LU-POSTROUTING
ipset_service_name=LU-SERVICE-IP
```

- 执行命令

```bash
# 实验1命令会在实验2中继续使用
ipvsadm --add-service --tcp-service "$vip":"$vip_port" --scheduler rr
ipvsadm --add-server --tcp-service "$vip":"$vip_port"  --real-server "$rip1":"$rip_port"  --masquerading 
ipvsadm --add-server --tcp-service "$vip":"$vip_port"  --real-server "$rip2":"$rip_port"  --masquerading 
ip address add "$vip"/32 dev "$vip_interface"

# 实验2新增部分
echo 1 > /proc/sys/net/ipv4/vs/conntrack
iptables -t nat -N "$chain_services"
iptables -t nat -A PREROUTING -m comment --comment "lu service portals" -j "$chain_services"
iptables -t nat -N "$chain_mark_masq"
##先创建ipset，在创建iptables规则
ipset create "$ipset_service_name" hash:ip,port -exist
iptables -t nat -A "$chain_services" -m comment --comment "lu service cluster ip + port for masquerade purpose" -m set --match-set "$ipset_service_name" dst,dst -j "$chain_mark_masq"
iptables -t nat -A "$chain_mark_masq" -j MARK --set-xmark 0x2000/0x2000

iptables -t nat -N "$chain_postrouting"
iptables -t nat -A POSTROUTING -m comment --comment "lu postrouting rules" -j "$chain_postrouting"
iptables -t nat -A "$chain_postrouting" -m comment --comment "lu service traffic requiring SNAT" -m mark --mark 0x2000/0x2000 -j MASQUERADE

iptables -t nat -A OUTPUT -m comment --comment "lu service portals" -j "$chain_services"

ipset add "$ipset_service_name" "$vip",tcp:"$vip_port" -exist

# 启动后端服务
[ -d  /tmp/lu-`hostname -I` ] || mkdir /tmp/lu-`hostname -I |awk '{print $1}'`
cd /tmp/lu-`hostname -I` && echo `hostname -I` > lu
## host启动服务
nohup python -m SimpleHTTPServer 8888 &
```

> -->PREROUTING 阶段处理，提供一个入口链，而不是直接添加在 PREROUTING 链上，在 PREROUTING 子链里去 ipset 匹配，跳转到我们 mark 的链，创建专门mark的链
> -->POSTROUTING 阶段处理，提供一个入口链，而不是直接添加在 POSTROUTING 链上，在 POSTROUTING 阶段做 snat
> -->添加ipvs的vip地址和端口
>
> MARK标记用于将特定的数据包打上标签，供iptables配合TC做QOS流量限制或应用策略路由。

### 实验2验证

```bash
# curl 169.254.1.100/lu
10.130.17.31 169.254.1.100
# curl 169.254.1.100/lu
10.130.17.30 169.254.1.100
```



## 实验3-dummy-ipvs-iptables-ipset

### 实验3说明

ipvs实验，vip配置在两个node的dummy网卡上，ipvs实现dnat，iptables和ipset实现snat。后端使用真实的Host启动简单的web服务，两个Host的节点访问vip，并验证联通性。

dummy接口说明：是完全虚拟的虚拟接口，和环回接口类似。dummy接口目的是提供一个设备来路由数据包，而不实际传输数据包。使用dummy接口可以使一个不活跃的SLIP（串行线互联网协议）地址看起来像一个真实的地址，用于本地程序。现在多用于测试和调试。

> 进行实验前先清除环境，然后在继续。
>
> [dummy_interface](https://developers.redhat.com/blog/2018/10/22/introduction-to-linux-interfaces-for-virtual-networking#dummy_interface)

### 实验3架构图

```ini
## ip地址分布
|------------Host0------------|            |------------Host1------------|
|                             |            |                             |
|  [10.130.17.30/24]eth0      |<- Network->|  [10.130.17.31/24]eth0      |
|[169.254.2.100/32]dummy-ipvs0|            |[169.254.2.100/32]dummy-ipvs0|
|             vip             |            |            vip              |
|------------------------------------------------------------------------|
## 代理数据流
                   |——> 10.130.17.30 realserver
169.254.2.100VIP-->|
                   |——> 10.130.17.31 realserver
## 命令结果
# ipvsadm -ln
TCP  169.254.1.100:80 rr
  -> 10.130.17.30:8888  
  -> 10.130.17.31:8888  
# iptables -S -tnat
-P PREROUTING ACCEPT
-P INPUT ACCEPT
-P OUTPUT ACCEPT
-P POSTROUTING ACCEPT
-N LU-MARK-MASQ
-N LU-POSTROUTING
-N LU-SERVICES
-A PREROUTING -m comment --comment "lu service portals" -j LU-SERVICES
-A OUTPUT -m comment --comment "lu service portals" -j LU-SERVICES
-A POSTROUTING -m comment --comment "lu postrouting rules" -j LU-POSTROUTING
-A LU-MARK-MASQ -j MARK --set-xmark 0x2000/0x2000
-A LU-POSTROUTING -m comment --comment "lu service traffic requiring SNAT" -m mark --mark 0x2000/0x2000 -j MASQUERADE
-A LU-SERVICES -m comment --comment "lu service cluster ip + port for masquerade purpose" -m set --match-set LU-SERVICE-IP dst,dst -j LU-MARK-MASQ
# ipset list
Name: LU-SERVICE-IP
Type: hash:ip,port
Members:
169.254.2.100,tcp:80
```

> 如果是openstack开启的虚机需要设置allowed-address-pairs

### 实验3命令

创建dummy网卡和设置ip-->设置ipvs-->设置iptables--设置ipset-->启服务

- 设置变量

```bash
dummy_ip=169.254.2.100
rip1=10.130.17.30
rip2=10.130.17.31
rip_port=8888
# 实验2变量部分
chain_services=LU-SERVICES
chain_mark_masq=LU-MARK-MASQ
chain_postrouting=LU-POSTROUTING
ipset_service_name=LU-SERVICE-IP
vip=169.254.1.100
vip_port=80
```

> vip变量不使用

- 执行命令

```bash
ip link add dev dummy-ipvs0 type dummy
ip addr add "$dummy_ip"/32 dev dummy-ipvs0
ipvsadm --add-service --tcp-service "$dummy_ip":"$vip_port" --scheduler rr
ipvsadm --add-server --tcp-service "$dummy_ip":"$vip_port"  --real-server "$rip1":"$rip_port"  --masquerading 
ipvsadm --add-server --tcp-service "$dummy_ip":"$vip_port"  --real-server "$rip2":"$rip_port"  --masquerading 

# 实验2部分
echo 1 > /proc/sys/net/ipv4/vs/conntrack
iptables -t nat -N "$chain_services"
iptables -t nat -A PREROUTING -m comment --comment "lu service portals" -j "$chain_services"
iptables -t nat -N "$chain_mark_masq"

ipset create "$ipset_service_name" hash:ip,port -exist
iptables -t nat -A "$chain_services" -m comment --comment "lu service cluster ip + port for masquerade purpose" -m set --match-set "$ipset_service_name" dst,dst -j "$chain_mark_masq"
iptables -t nat -A "$chain_mark_masq" -j MARK --set-xmark 0x2000/0x2000

iptables -t nat -N "$chain_postrouting"
iptables -t nat -A POSTROUTING -m comment --comment "lu postrouting rules" -j "$chain_postrouting"
iptables -t nat -A "$chain_postrouting" -m comment --comment "lu service traffic requiring SNAT" -m mark --mark 0x2000/0x2000 -j MASQUERADE

iptables -t nat -A OUTPUT -m comment --comment "lu service portals" -j "$chain_services"

# 修改的实验2部分
ipset add "$ipset_service_name" "$dummy_ip",tcp:"$vip_port" -exist

#启动后端服务
[ -d  /tmp/lu-`hostname -I` ] || mkdir /tmp/lu-`hostname -I |awk '{print $1}'`
cd /tmp/lu-`hostname -I` && echo `hostname -I` > lu
## host启动服务
nohup python -m SimpleHTTPServer 8888 &
```

### 实验3验证

```shell
# curl 169.254.2.100/lu
10.130.17.30
# curl 169.254.2.100/lu
10.130.17.31
```

## 实验4-namespace-dummy-ipvs-iptables-ipset

### 实验4说明

ipvs和calico网络模拟实验，vip配置在两个node的dummy网卡上，ipvs实现dnat，iptables和ipset实现snat。其中后端在namespace中启动web服务，Host设置相关路由来满足网络的连通性。

> 进行实验前先清除环境，然后在继续。

### 实验4架构图

```ini
## ip地址分布
|------------Host0------------|            |------------Host1------------|
|                             |            |                             |
|  [network ns0]              |            |  [network ns1]              |
|  [172.16.0.10/24]veth0      |            |  [172.16.1.11/24]veth0      |
|             \               |            |              /              |
|         [cali0]-- [eth0] -- |<- Network->|--[eth0] -- [cali0]          |
|                10.130.17.30 |            | 10.130.17.30                |
|------------------------------------------------------------------------|
# 路由
172.16.0.10 dev cali0 scope link           172.16.1.11 dev cali0 scope link
172.16.1.0/24 via 10.130.17.31 dev eth0    172.16.1.0/24 via 10.130.17.31 dev eth0
## 代理数据流
                   |——> 172.16.0.10 realserver
169.254.2.100VIP-->|
                   |——> 172.16.1.11 realserver
## 命令结果
# ipvsadm -ln
TCP  169.254.2.100:80 rr
  -> 172.16.0.10:8888     
  -> 172.16.1.11:8888     
# iptables -S -tnat
-P PREROUTING ACCEPT
-P INPUT ACCEPT
-P OUTPUT ACCEPT
-P POSTROUTING ACCEPT
-N LU-MARK-MASQ
-N LU-POSTROUTING
-N LU-SERVICES
-A PREROUTING -m comment --comment "lu service portals" -j LU-SERVICES
-A OUTPUT -m comment --comment "lu service portals" -j LU-SERVICES
-A POSTROUTING -m comment --comment "lu postrouting rules" -j LU-POSTROUTING
-A LU-MARK-MASQ -j MARK --set-xmark 0x2000/0x2000
-A LU-POSTROUTING -m comment --comment "lu service traffic requiring SNAT" -m mark --mark 0x2000/0x2000 -j MASQUERADE
-A LU-SERVICES -m comment --comment "lu service cluster ip + port for masquerade purpose" -m set --match-set LU-SERVICE-IP dst,dst -j LU-MARK-MASQ
# ipset list
Name: LU-SERVICE-IP
Type: hash:ip,port
Members:
169.254.2.100,tcp:80
```

> 如果是openstack开启的虚机需要设置allowed-address-pairs

### 实验4命令

创建veth pair--> 创建命名空间-->分配地址和设置路由-->设置ipvs-->设置iptables--设置ipset-->启服务

- 变量设置

```bash
# 这三个变量需要跟进主机修改
destion_host_ip=10.130.17.31 
destion_ns_cidr=172.16.1.0/24
local_ns_ip=172.16.0.10
# 不需要修改
ns_name=ns0

# 实验3变量部分，rip1和rip2被修改成namespace地址
dummy_ip=169.254.2.100
rip1=172.16.0.10
rip2=172.16.1.11
rip_port=8888
# 实验2变量部分
chain_services=LU-SERVICES
chain_mark_masq=LU-MARK-MASQ
chain_postrouting=LU-POSTROUTING
ipset_service_name=LU-SERVICE-IP
vip=169.254.1.100
vip_port=80
```

> 变量destion_host_ip和destion_ns_cidr是**目标端的信息**，非执行命令的主机的信息。
>
> 变量local_ns_ip是执行命令的主机的namespace内的ip地址。

- 执行命令

```bash
ip link add cali0 type veth peer name veth0
ip netns add "$ns_name"
ip link set veth0 netns "$ns_name"
ip netns exec "$ns_name" ip a add "$local_ns_ip"/24 dev veth0
ip netns exec "$ns_name" ip link set veth0 up
ip netns exec "$ns_name" ip link set lo up
ip netns exec "$ns_name" ip route add 169.254.1.1 dev veth0 scope link
ip netns exec "$ns_name" ip route add default via 169.254.1.1 dev veth0
ip link set cali0 up
ip route add "$local_ns_ip" dev cali0 scope link
ip route add "$destion_ns_cidr" via "$destion_host_ip" dev eth0
echo 1 > /proc/sys/net/ipv4/conf/cali0/proxy_arp
echo 1 > /proc/sys/net/ipv4/ip_forward

# 实验3部分
ip link add dev dummy-ipvs0 type dummy
ip addr add "$dummy_ip"/32 dev dummy-ipvs0
ipvsadm --add-service --tcp-service "$dummy_ip":"$vip_port" --scheduler rr
ipvsadm --add-server --tcp-service "$dummy_ip":"$vip_port"  --real-server "$rip1":"$rip_port"  --masquerading 
ipvsadm --add-server --tcp-service "$dummy_ip":"$vip_port"  --real-server "$rip2":"$rip_port"  --masquerading 

# 实验2部分
echo 1 > /proc/sys/net/ipv4/vs/conntrack
iptables -t nat -N "$chain_services"
iptables -t nat -A PREROUTING -m comment --comment "lu service portals" -j "$chain_services"
iptables -t nat -N "$chain_mark_masq"

ipset create "$ipset_service_name" hash:ip,port -exist
iptables -t nat -A "$chain_services" -m comment --comment "lu service cluster ip + port for masquerade purpose" -m set --match-set "$ipset_service_name" dst,dst -j "$chain_mark_masq"
iptables -t nat -A "$chain_mark_masq" -j MARK --set-xmark 0x2000/0x2000

iptables -t nat -N "$chain_postrouting"
iptables -t nat -A POSTROUTING -m comment --comment "lu postrouting rules" -j "$chain_postrouting"
iptables -t nat -A "$chain_postrouting" -m comment --comment "lu service traffic requiring SNAT" -m mark --mark 0x2000/0x2000 -j MASQUERADE

iptables -t nat -A OUTPUT -m comment --comment "lu service portals" -j "$chain_services"

# 修改的实验2部分
ipset add "$ipset_service_name" "$dummy_ip",tcp:"$vip_port" -exist

#启动后端服务
[ -d  /tmp/lu-`hostname -I` ] || mkdir /tmp/lu-`hostname -I |awk '{print $1}'`
cd /tmp/lu-`hostname -I` && echo `hostname -I` > lu
## namespace启动服务
ip netns exec ns0 nohup python -m SimpleHTTPServer 8888 &
```

### 实验4验证

```bash
## web服务验证
# curl 169.254.2.100/lu
10.130.17.30
# curl 169.254.2.100/lu
10.130.17.31

## ping验证
# ip netns exec ns0 ping -c1 172.16.0.10 
PING 172.16.0.10 (172.16.0.10) 56(84) bytes of data.
64 bytes from 172.16.0.10: icmp_seq=1 ttl=64 time=0.058 ms

--- 172.16.0.10 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.058/0.058/0.058/0.000 ms
# ip netns exec ns0 ping -c1 172.16.1.11 
PING 172.16.1.11 (172.16.1.11) 56(84) bytes of data.
64 bytes from 172.16.1.11: icmp_seq=1 ttl=62 time=0.347 ms

--- 172.16.1.11 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.347/0.347/0.347/0.000 ms

## tracepath验证
# ip netns exec ns0 tracepath -n 172.16.1.11 
 1?: [LOCALHOST]                                         pmtu 1500
 1:  10.130.17.30                                          0.122ms 
 1:  10.130.17.30                                          0.035ms 
 2:  10.130.17.30                                          0.031ms pmtu 1450
 2:  10.130.17.31                                          0.369ms 
 3:  172.16.1.11                                           0.303ms reached
     Resume: pmtu 1450 hops 3 back 3 
```



## 附件

### lvs概念

```bash
Director：负载均衡器,也称VS(Virtual Server)
RS:真实服务器(RealServer)
CIP：Client IP客户端IP
VIP: Client所请求的，提供虚拟服务的IP，可以用Keepalive做高可用
DIP: director IP 在Director实现与RS通信的IP
RIP:RealServer IP

NAT：Network address translation，网络地址转换
DR：Direct routing，直接路由
TUN：IP tunneling，IP隧道
```

### ipset匹配

| ipset type        | iptables match-set | Packet fields                                      |
| ----------------- | ------------------ | -------------------------------------------------- |
| hash:net,port,net | src,dst,dst        | src IP CIDR address, dst port, dst IP CIDR address |
| hash:net,port,net | dst,src,src        | dst IP CIDR address, src port, src IP CIDR address |
| hash:ip,port,ip   | src,dst,dst        | src IP address, dst port, dst IP address           |
| hash:ip,port,ip   | dst,src,src        | dst IP address, src port, src ip address           |
| hash:mac          | src                | src mac address                                    |
| hash:mac          | dst                | dst mac address                                    |
| hash:ip,mac       | src,src            | src IP address, src mac address                    |
| hash:ip,mac       | dst,dst            | dst IP address, dst mac address                    |
| hash:ip,mac       | dst,src            | dst IP address, src mac address                    |

Supported set types:
    list:set		3	skbinfo support
    list:set		2	comment support
    list:set		1	counters support
    list:set		0	Initial revision
    hash:mac		0	Initial revision
    hash:ip,mac		0	Initial revision
    hash:net,iface	6	skbinfo support
    hash:net,iface	5	forceadd support
    hash:net,iface	4	comment support
    hash:net,iface	3	counters support
    hash:net,iface	2	/0 network support
    hash:net,iface	1	nomatch flag support
    hash:net,iface	0	Initial revision
    hash:net,port	7	skbinfo support
    hash:net,port	6	forceadd support
    hash:net,port	5	comment support
    hash:net,port	4	counters support
    hash:net,port	3	nomatch flag support
    hash:net,port	2	Add/del range support
    hash:net,port	1	SCTP and UDPLITE support
    hash:net,port,net	2	skbinfo support
    hash:net,port,net	1	forceadd support
    hash:net,port,net	0	initial revision
    hash:net,net	2	skbinfo support
    hash:net,net	1	forceadd support
    hash:net,net	0	initial revision
    hash:net		6	skbinfo support
    hash:net		5	forceadd support
    hash:net		4	comment support
    hash:net		3	counters support
    hash:net		2	nomatch flag support
    hash:net		1	Add/del range support
    hash:net		0	Initial revision
    hash:ip,port,net	7	skbinfo support
    hash:ip,port,net	6	forceadd support
    hash:ip,port,net	5	comment support
    hash:ip,port,net	4	counters support
    hash:ip,port,net	3	nomatch flag support
    hash:ip,port,net	2	Add/del range support
    hash:ip,port,net	1	SCTP and UDPLITE support
    hash:ip,port,ip	5	skbinfo support
    hash:ip,port,ip	4	forceadd support
    hash:ip,port,ip	3	comment support
    hash:ip,port,ip	2	counters support
    hash:ip,port,ip	1	SCTP and UDPLITE support
    hash:ip,mark	2	skbinfo support
    hash:ip,mark	1	forceadd support
    hash:ip,mark	0	initial revision
    hash:ip,port	5	skbinfo support
    hash:ip,port	4	forceadd support
    hash:ip,port	3	comment support
    hash:ip,port	2	counters support
    hash:ip,port	1	SCTP and UDPLITE support
    hash:ip		4	skbinfo support
    hash:ip		3	forceadd support
    hash:ip		2	comment support
    hash:ip		1	counters support
    hash:ip		0	Initial revision
    bitmap:port		3	skbinfo support
    bitmap:port		2	comment support
    bitmap:port		1	counters support
    bitmap:port		0	Initial revision
    bitmap:ip,mac	3	skbinfo support
    bitmap:ip,mac	2	comment support
    bitmap:ip,mac	1	counters support
    bitmap:ip,mac	0	Initial revision
    bitmap:ip		3	skbinfo support
    bitmap:ip		2	comment support
    bitmap:ip		1	counters support
    bitmap:ip		0	Initial revision



[kube-proxy IPVS 模式的工作原理](https://fuckcloudnative.io/posts/ipvs-how-kubernetes-services-direct-traffic-to-pods/)

[K8S-ipvs负载均衡原理](http://www.4k8k.xyz/article/sdgdsczs/120066177)

[在非容器环境上实现散装的 IPVS SVC](https://cloudnative.to/blog/create-a-ipvs-svc-without-container/)

[深入理解 Kubernetes 网络模型：自己实现 Kube-Proxy 的功能](https://cloudnative.to/blog/k8s-node-proxy/)

iptables:

　　[ https://www.jianshu.com/p/67744d680286](https://www.jianshu.com/p/67744d680286) 这里对iptables讲解的

IPvs  

   [ https://www.jianshu.com/p/7cff00e253f4](https://www.jianshu.com/p/7cff00e253f4) 这里对ipvs讲解的比较好