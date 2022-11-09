[TOC]

# Calico原理和实验-Kubernetes

## 使用场景

Calico是容器网络互通的一种解决方案，主要用于k8s环境中的pod之间通信和隔离。

## 设计思路

Calico没有使用隧道或NAT来实现转发，而是巧妙的把所有的二/三层流量转成三层流量后通过HOST上的路由配置完成转发。

> Calico 通过一个巧妙的方法将 workload 的所有流量引导到一个特殊的网关 169.254.1.1，从而引流到主机的 calixxx 网络设备上，最终将二三层流量全部转换成三层流量来转发。
>
> 在主机上通过开启代理 ARP 功能来实现 ARP 应答，使得 ARP 广播被抑制在主机上，抑制了广播风暴，也不会有 ARP 表膨胀的问题。

## 特点

### 优势

- **性能好**  Calico使用Linux eBPF或Linux内核的高度优化的标准网络来提供高性能的网络。Calico的网络选项非常灵活，可以在大多数环境中不使用overlays而运行，避免了数据包封装/解密的开销。Calico的控制平面和策略引擎已经在多年的生产使用中进行了微调，以最大限度地减少CPU的总体使用。

- **可扩展性**  Calico的核心设计原则是利用最佳实践的云原生设计模式，结合经过验证的基于标准的网络协议，受到全球最大的互联网运营商的信任。其结果是一个具有卓越的可扩展性的解决方案，已经在生产中大规模运行了多年。Calico的开发测试周期包括定期测试数以千计的节点集群。无论你运行的是10个节点的集群，还是100个节点的集群，或者更多。

- **互通性** Calico使Kubernetes工作负载和非Kubernetes或遗留工作负载能够无缝、安全地进行通信。Kubernetes pods是网络上的重要角色，能够与网络上的任何其他工作负载进行通信。此外，Calico可以无缝扩展，以确保您现有的基于主机的工作负载（无论是在公共云还是在内部的虚拟机或裸机服务器上）与Kubernetes一起。使用 Network Policy 来实现 Pod 与 Pod 之间的访问控制。

- **完全支持Kubernetes网络策略**Calico的网络策略引擎是Kubernetes网络策略在API开发过程中的原始参考实现。Calico的与众不同之处在于，它实现了API定义的全部功能，为用户提供了定义API时设想的所有能力和灵活性。对于需要更多功能的用户，Calico支持一套扩展的网络策略功能，与Kubernetes API无缝衔接，使用户在定义网络策略时具有更大的灵活性。

### 劣势

- **多租户**   不易持多租户，而overlay中vxlan模式的VID实现了多租户的功能 (仅公司自用场景可行)。
- **路由条目 ** 路由条目与POD数量相同，非常容易超过路由器、三层交换、甚至Node的处理能力，从而限制整个网络的扩张
- **BGP限制** BGP模式要求宿主机处于同一个2层网络

## 说明

```ini
由于各物理机中为容器分配不同网段，因此完全可将这些物理机视为路由器并将容器的地址信息生成路由表。
Calico默认不使用overlay网络模型，而是用虚拟路由替代虚拟交换，因此是纯三层方案 ( 纯三层SDN实现，创建并管理一个三层平面网络 ) 
集群所有节点利用Linux内核实现模拟的路由器 VRouter 进行数据的转发，所有 VRouter 通过BGP协议将自身 Workload 的IP路由信息向整个网络传播。
小规模部署时可直接互联，大规模时可通过额外的 BGP route reflector (反射) 实现，从而避免BGP的全互联。
它自带的基于iptables的ACL管理组件非常灵活，能满足复杂的安全隔离需求
综上所述：Calico利用了Linux内核原生的路由及iptables防火墙功能。

BGP
边界网关协议 Border Gateway Protocol 是互联网核心的去中心化的自治路由协议。
它不使用传统IGP协议的度量指标，而是基于路径、网络策略、规则集来决定路由。
该协议通过维护IP路由表或CIDR前缀表来实现自治系统 (AS) 间的互通，属于矢量路由协议 (在自治系统之间提供路由)。
BGP系统的主要功能是和其他的BGP系统交换网络可达信息。网络可达信息包括列出的自治系统（AS）的信息。
BGP通过携带AS路径信息来标记途经的AS，带有本地AS的编号的路由将被丢弃，从而避免域间产生环路。
BGP在小规模集群中可直接互联，在大规模集群中可通过额外的 BGP route reflector 反射完成
当BGP处在全互联的模式时: 每个 BIRD 都要和其他 BIRD 建立TCP/179的连接，这样连接总数是 N^2 (不建议超过100台)
当BGP处在路由反射模式时: 指定个别 BIRD 作为 BGP Route Reflector，每个BIRD只要与其建立连接即可学得全网路由

Calico的两种网络模式:
1.IPinIP    将所有节点间的路由创建隧道再进行连接，因此该模式会在Node中创建名为 tunl0 的虚拟接口 (overlay)
2.BGP       直接把节点作为虚拟路由路器 vRouter，因此不存在Tunnel
```

## 组成

```ini
etcd *                   # 负责网络元数据，基于其提供的watch机制实现Calico网络状态的准确、实时性 (可与kubernetes共用)
Felix *                  # 运行在所有节点，负责更新节点路由、ACL等信息并管理容器 Endpoint (接入) 资源，并将运行状态写入etcd
Calico/node (BIRD) *     # 运行在所有节点，负责把Felix写入内核的路由信息分发到整个Calico网络来实现集群节点间POD的通信有效性，即 VRouter
BGP Route Reflector (RR) # 即BGP路由反射器，仅大规模部署时使用 (通过其实现集中式的路由分发、学习)
Orchestrator plugin *    # 提供与各种云计算编排工具的紧密集成和同步支持，针对不同的平台、架构有着不同的Plugin
Dikastes/Envoy           # 可选的 sidecars，通过互相TLS验证来保护工作负载到工作负载的通信，并增加应用层控制策略
calicoctl *              # 命令行管理工具，通过其实现高级的策略和网络
Endpoint                 # 接入到Calico网络中的网卡称为Endpoint (这里即POD) 
AS                       # 网络自治系统，通过BGP协议与其它的AS交换路由信息 (自治网络拥有独立交换机、路由器等，可独立运转)
IBGP                     # AS内部的BGP_Speaker，与相同AS的ibgp、ebgp交换路由信息
EBGP                     # AS边界的BGP_Speaker，与相同AS的ibgp、以及不同AS的ebgp交换路由信息


# 在kubernetes的架构中当Calico作为网络策略和网络组件时实际运行了两类容器
# kubectl get po -n kube-system | grep calico
calico-kube-controllers-6f899557cd-pbd7c   1/1     Running   2          15d
calico-node-6c8kn                          2/2     Running   0          19d
calico-node-8lmz4                          2/2     Running   2          19d
```

## 工作原理

```ini
etcd
  |
[Felix & Calico node]

Calico与weave在拓扑方面类似: 都是在所有主机启动虚拟机路由器从而将它们作为路由器运行，最终形成互联互通
它把集群中的每个物理节点视为路由器，然后把所有的容器认为是连在这个路由器上的网络终端，即Endpoint
在这些虚拟路由器之间跑的是标准的BGP协议，它们自己学习并最终形成可转发的网络拓扑，因此基于Calico的方案其实是纯L3方案 ...
综上所述，因此每个节点都会运行两个主要的程序:
1.Felix：      # 监听etcd事件，如: 分配了POD、添加了IP，并确保容器 Endpoint (接入) ...
2.BGP Client： # 也称为 calico node，作为虚拟的路由器通过BGP进行路由的传播，使外界可直接访问容器IP而不需做任何类似NAT的操作

Example ...
1.POD创建时Calico为POD生成veth pair设备并将其一端设为容器网络命名空间内的网卡并设置IP/mask等信息
2.同时veth pair的另一端直接暴露在宿主机cali前缀的设备上 (没有网桥) 并设置路由规则将这个POD地址直接暴露到宿主机的通信路由之上
3.于此同时Calico为每个节点分配不同的子网做POD分配的IP范围，这样即可根据子网CIDR为每个节点生成比较固定的路由规则 (存疑)
4.根据容器要访问的目的IP所在的子网CIDR及该节点上的路由规则找到下一跳要到达的宿主机IP
5.流量到达目标节点之后，再根据当前宿主机内的路由表直接到达容器的veth pair插在宿主机的一端，从而互通
这里最核心的下一跳的路由规则是由Calico的Felix进程维护的，而路由信息则是通过BIRD组件使用TCP/BGP协议来传播的
该过程全部在L3完成 () 
```

## Calico两种封装类型

### BGP

边界网关协议（Border Gateway Protocol, BGP）是互联网上一个核心的去中心化自治路由协议。它通过维护IP路由表或‘前缀’表来实现自治系统（AS）之间的可达性，属于矢量路由协议。BGP不使用传统的内部网关协议（IGP）的指标，而使用基于路径、网络策略或规则集来决定路由。因此，它更适合被称为矢量性协议，而不是路由协议。

### IPIP

把 IP 层封装到 IP 层的一个 tunnel。它的作用其实基本上就相当于一个基于IP层的网桥！一般来说，普通的网桥是基于mac层的，根本不需 IP，而这个 ipip 则是通过两端的路由做一个 tunnel，把两个本来不通的网络通过点对点连接起来。ipip 的源代码在内核 net/ipv4/ipip.c 中可以找到。

## 部署

```ini
环境需求: https://docs.projectcalico.org/getting-started/kubernetes/requirements
快速安装: https://docs.projectcalico.org/getting-started/kubernetes/quickstart
官方答疑: https://docs.projectcalico.org/reference/faq
指定节点IP作为calico使用的地址：https://docs.projectcalico.org/networking/ip-autodetection

默认安装以下资源
1.calico/node使用DaemonSet在每个主机上运行，关于其声明中的环境变量参数：https://docs.projectcalico.org/reference/node/configuration
2.使用DaemonSet在每台Node安装Calico CNI二进制文件和网络配置
3.calico/kube-controllers作为Deployment资源实例运行
4.calico-etcd-secrets，可选的，提供TLS ETCD
5.calico-configConfigMap，包含用于配置安装的参数: calico_backend(后端使用)、cni_network_config(CNI网络配置安装在每个节点)
```

## BGP数据流

```
pod中eth0-->calixxxxxxxxxxx(11位数字+字母)-->Host中eth0
```

>  calixxxxxxxxxxx网卡启用：proxy_arp=1，物理机启用：ip_forward=1

```ini
| -------------- Host_0 ---------------|                   | -------------- Host_1 ---------------|
|                                      |                   |                                      |
|  [network Namespace_0]               |                   |               [network Namespace_1]  |
|  [172.16.0.10/24]eth0                | ----<Network>---- |               [172.16.1.11/24]eth0   |
|                \        192.168.1.10 |                   | 192.168.1.11      /                  |
|         [calixxxxxxxxxxx]-- [eth0] --|                   |--[eth0] -- [calixxxxxxxxxxx]         |
|-------------------------------------------------------------------------------------------------|
```

其 veth pair 的一端在 Pod 内，设置为 pod 的 IP，另一端在宿主机中，没有设置 IP，也没有接入 bridge，但是设置了 proxy_arp=1。

pod 内有以下的路由表：

```bash
# ip r          ## pod内
default via 169.254.1.1 dev eth0 
169.254.1.1 dev eth0  scope link 
```

169.254.0.0/16 是一个特殊的 IP 段，只会在主机内出现。不过这里这个 IP 并不重要，只是为了防止冲突才选择了这个特殊值。当 Pod 要访问其他 IP 时，如果该 IP 在同一个网段，那就需要获取该 IP 的 MAC 地址。如果不在一个网段，那么根据路由表，就要获取网关的 IP 地址。所以无论如何，arp 请求都会到达calixxxxxxxxxx。

因为 calixxxxxxxxxx 设置了 proxy_arp=1，所以就会返回自己的 MAC 地址，然后 Pod 的流量就发到了主机的网络协议栈。到达网络协议栈之后，被转发到对端的主机上。

流量到达对端主机后，主机上直接设置了 pod 的路由：

```bash
# ip r   ## Host内
172.16.0.10 dev calixxxxxxxxxxx scope link
172.16.1.0/24 via 192.168.1.11 dev eth0 proto bird
```

## 实验模拟

### BGP模式

#### BGP实验架构图

```ini
| -------------- Host0  ---------------|                   | -------------- Host1  ---------------|
|                                      |                   |                                      |
|  [network Namespace0]                |                   |               [network Namespace1]   |
|  [172.16.0.10/24]veth0               | ----<Network>---- |               [172.16.1.11/24]veth0  |
|                \        192.168.1.10 |                   | 192.168.1.11      /                  |
|                [cali0]  -- [eth0] -- |                   | --[eth0] -- [cali1]                  |
|-------------------------------------------------------------------------------------------------|
[Host0]                                                    [Host1]
172.16.0.10 dev cali0 scope link                           172.16.1.11 dev cali1 scope link 
172.16.1.0/24 via 192.168.1.11 dev eth0				       172.16.0.0/24 via 192.168.1.10 dev eth0 
```

Host0 192.168.1.10和Host1192.168.1.11在同一个二层网络

#### BGP实验命令

```ini
Host添加veth--> Host添加namespace并把veth添加到ns --> ns内veth设置ip和路由--> Host添加路由--> Host开启ns外veth的ARP代理和Host路由转发
```

如果是openstack环境需要设置allowed-address-pairs

```bash
neutron port-update <port_id> --allowed-address-pairs type=dict list=true ip_address=0.0.0.0/0
```

##### Host0

###### 手动设置

```bash
destion_ip=192.168.1.11
ip link add cali0 type veth peer name veth0                         # 创建veth设备并为其两端命名为cali0与veth0
ip netns add ns0                                                    # 创建网络命名空间: ns0
ip link set veth0 netns ns0                                         # 将宿主机的veth0添加到ns0虚拟网络环境 ...
ip netns exec ns0 ip a add 172.16.0.10/24 dev veth0                 # 进入容器的网络名称空间，设置容器的IP
ip netns exec ns0 ip link set veth0 up                              # 启用在容器一端的veth设备
ip netns exec ns0 ip route add 169.254.1.1 dev veth0 scope link     # 设置路由
ip netns exec ns0 ip route add default via 169.254.1.1 dev veth0    # ns0网络空间的所有数据都转发到一个虚拟的IP:169.254.1.1发送ARP请求
ip link set cali0 up                                                # 启用宿主机一端的 veth 设备
ip route add 172.16.0.10 dev cali0 scope link                       # 宿主机中设备本机容器IP的路由信息
ip route add 172.16.1.0/24 via $destion_ip dev eth0                # 宿主机中设置其他节点的容器IP的路由信息 (下一跳指向了Host1)
echo 1 > /proc/sys/net/ipv4/conf/cali0/proxy_arp                    # 开启宿主机的ARP代理功能
echo 1 > /proc/sys/net/ipv4/ip_forward                              # 开启宿主机路由转发
```

###### 脚本设置

```bash
## 添加
# bash  set-ns-veth-calico.sh set 172.16.0.10 192.168.1.11  172.16.1.0/24
Create namepace successfull!!
## 删除
# bash set-ns-veth-calico.sh -d
Delete namepace successfull!!

## 使用方法
# bash set-ns-veth-calico.sh -h
  Operate ns and veth pair, ip and route.
  Usage:
    bash set-ns-veth-calico.sh [ option ] <local namespace ip> <destion host ip> <destion namespace network>
  Option:
    -s set     Add namespace/veth pair,  set ip and route.
       bash set-ns-veth-calico.sh set [ local namespace ip ] [ destion host ip  ] [ destion namespace network  ]
    -d delete  Delete namespace and route.
  Example:
    bash set-ns-veth-calico.sh set 172.16.0.2 192.168.1.100  172.16.1.0/24
    bash set-ns-veth-calico.sh delete
```

> 脚本见附加set-ns-veth-calico.sh，脚本内设置的ns和veth pair名称Host0和Host1是一致的。

##### HOST1

###### 手动设置

```bash
destion_ip=192.168.1.10
ip link add cali1 type veth peer name veth0
ip netns add ns1
ip link set veth0 netns ns1
ip netns exec ns1 ip a add 172.16.1.11/24 dev veth0
ip netns exec ns1 ip link set veth0 up
ip netns exec ns1 ip route add 169.254.1.1 dev veth0 scope link
ip netns exec ns1 ip route add default via 169.254.1.1 dev veth0
ip link set cali1 up
ip route add 172.16.1.11 dev cali1 scope link
ip route add 172.16.0.0/24 via $destion_ip dev eth0
echo 1 > /proc/sys/net/ipv4/conf/cali1/proxy_arp
echo 1 > /proc/sys/net/ipv4/ip_forward
```

###### 脚本设置

```bash
## 添加
# bash  set-ns-veth-calico.sh set 172.16.1.11 192.168.1.10  172.16.0.0/24
Create namepace successfull!!
## 删除
# bash set-ns-veth-calico.sh -d
Delete namepace successfull!!
```

#### BGP实验验证

```bash
## host0
# ip netns exec ns0 ping -c1 172.16.1.11
PING 172.16.1.11 (172.16.1.11) 56(84) bytes of data.
64 bytes from 172.16.1.11: icmp_seq=1 ttl=62 time=0.289 ms

--- 172.16.1.11 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.289/0.289/0.289/0.000 ms

## Host1
# ip netns exec ns1 ping -c1 172.16.0.10
PING 172.16.0.10 (172.16.0.10) 56(84) bytes of data.
64 bytes from 172.16.0.10: icmp_seq=1 ttl=62 time=0.291 ms

--- 172.16.0.10 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.291/0.291/0.291/0.000 ms

## 路径
# ip netns exec ns0 tracepath -n 172.16.1.11
 1?: [LOCALHOST]                                         pmtu 1500
 1:  192.168.1.10                                          0.048ms 
 1:  192.168.1.10                                          0.016ms 
 2:  192.168.1.10                                          0.025ms pmtu 1450
 2:  192.168.1.11                                          0.386ms 
 3:  172.16.1.11                                           0.291ms reached
     Resume: pmtu 1450 hops 3 back 3 
```

#### BGP转发过程

```ini
1.Host0 的veth端收到ARP请求时通过开启网卡的代理ARP功能直接把自己的MAC返回给ns0的容器 (这是重点)
2.ns0的容器发送的数据表中目的地址为对端ns1中的容器IP
3.因为使用了169.254.1.1这样的保留IP，宿主机判断为基于L3的路由转发，根据其路由表 172.16.1.0/24 via 192.168.1.11 dev eth0 发往 Host1
4.如果配置了BGP，这里就会看到 proto 协议为 BIRD
5.当Host1收到来自容器172.16.0.10的数据包时，根据其路由表 172.16.1.11 dev cali1  scope link，转发到对应的 cali1 端从而到达 ns1
6.回程相同
```

### IPIP模式

#### IPIP实验架构图

```ini
| -------------- Host0  ---------------|                   | -------------- Host1  ---------------|
|                                      |                   |                                      |
|  [network Namespace0]                |                   |               [network Namespace1]   |
|  [172.16.0.10/24]veth0               | ----<Network>---- |               [172.16.1.11/24]veth0  |
|                \        192.168.0.10 |                   | 192.168.1.11        /                |
|            [cali0]--[tunl0]--[eth0]--|                   |--[eth0]--[tunl1]--[cali1]            |
|-------------------------------------------------------------------------------------------------|
[Host0]                                                    [Host1]
172.16.0.10 dev cali0 scope link                           172.16.1.11 dev cali1 scope link 
172.16.1.0/24 via 192.168.1.11 dev eth0				       172.16.0.0/24 via 192.168.0.10 dev eth0 
```

> Host0 192.168.0.10和Host1192.168.1.11不在同一个二层网络，但是Host0和Host1网络互通。

#### IPIP实验命令

```ini
Host添加veth--> Host添加namespace并把veth添加到ns --> ns内veth设置ip和路由-->添加tunnel网卡-->Host添加路由--> Host开启ns外veth的ARP代理和Host路由转发
```

##### Host0

```bash
destion_ip=192.168.1.11
ip link add cali0 type veth peer name veth0
ip netns add ns0
ip link set veth0 netns ns0
ip netns exec ns0 ip a add 172.16.0.10/24 dev veth0
ip netns exec ns0 ip link set veth0 up
ip netns exec ns0 ip route add 169.254.1.1 dev veth0 scope link
ip netns exec ns0 ip route add default via 169.254.1.1 dev veth0
ip link set cali0 up
ip route add 172.16.0.10 dev cali0 scope link
echo 1 > /proc/sys/net/ipv4/conf/cali0/proxy_arp
echo 1 > /proc/sys/net/ipv4/ip_forward

ip tunnel add mode ipip
ip link set tunl0 up
ip route add 172.16.1.0/24 via $destion_ip dev tunl0 proto bird onlink
```

##### Host1

```bash
destion_ip=192.168.0.10
ip link add cali1 type veth peer name veth0
ip netns add ns1
ip link set veth0 netns ns1
ip netns exec ns1 ip a add 172.16.1.11/24 dev veth0
ip netns exec ns1 ip link set veth0 up
ip netns exec ns1 ip route add 169.254.1.1 dev veth0 scope link
ip netns exec ns1 ip route add default via 169.254.1.1 dev veth0
ip link set cali1 up
ip route add 172.16.1.11 dev cali1 scope link
echo 1 > /proc/sys/net/ipv4/conf/cali1/proxy_arp
echo 1 > /proc/sys/net/ipv4/ip_forward

ip tunnel add mode ipip
ip link set tunl0 up
ip route add 172.16.0.0/24 via $destion_ip dev tunl0 proto bird onlink
```

> ip tunnel add mode ipip会添加内核模块ipip，使用lsmod |grep ipip查看。

#### IPIP实验验证

```bash
## host0
# ip netns exec ns0 ping -c1 172.16.1.11
PING 172.16.1.11 (172.16.1.11) 56(84) bytes of data.
64 bytes from 172.16.1.11: icmp_seq=1 ttl=62 time=0.416 ms

--- 172.16.1.11 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.416/0.416/0.416/0.000 ms
## host1
# ip netns exec ns1 ping -c1 172.16.0.10
PING 172.16.0.10 (172.16.0.10) 56(84) bytes of data.
64 bytes from 172.16.0.10: icmp_seq=1 ttl=62 time=0.473 ms

--- 172.16.0.10 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.473/0.473/0.473/0.000 ms

## Host1抓包
# tcpdump -i tunl0 host 172.16.0.10 
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on tunl0, link-type RAW (Raw IP), capture size 262144 bytes
14:56:50.039860 IP 172.16.0.10 > 172.16.1.11: ICMP echo request, id 7323, seq 32, length 64
14:56:50.039910 IP 172.16.1.11 > 172.16.0.10: ICMP echo reply, id 7323, seq 32, length 64
```

#### IPIP转发过程

```ini
1.Host0 的veth端收到ARP请求时通过开启网卡的代理ARP功能直接把自己的MAC返回给ns0的容器 (这是重点)
2.ns0的容器发送的数据表中目的地址为对端ns1中的容器IP
3.因为使用了169.254.1.1这样的保留IP，宿主机判断为基于L3的路由转发，根据其路由表 172.16.1.0/24 via 10.130.161.43 dev tunl0 proto bird onlink，数据包到tunl0，在通过eth0发往到 Host1 
4.如果配置了BGP，这里就会看到 proto 协议为 BIRD
5.当Host1收到来自容器172.16.0.10的数据包时，根据其路由表 172.16.1.11 dev cali1  scope link，转发到对应的 cali1 端从而到达 ns1
6.回程相同
```

### 实验数据清理

清除namespace，清除主机上路由，清除隧道网卡

```bash
ip netns delete ns0
ip r delete 172.16.0.0/24
rmmod ipip
```



## 概念

**IPAM** IP地址管理，两种主流方法--基于CIDR的IP地址段分配或精确地为每一个容器分配IP，为每一个调度单元分配一个全局唯一IP地址。

**Overlay** 在现有二层/三层网络之上通过额外的网络协议，将原IP报文封装起来形成逻辑上的新网络

**IPSesc** 一个点对点的加密通信协议

**IP隧道(IP Tunnel)** 是将一个IP报文封装在另一个IP报文的技术，可以使得目标为一个IP地址的数据报文能被封装和转发到另一个IP地址。主要拥有移动主机和虚拟私有网络(VPN, Virtual Private Network)，在其中隧道都是静态建立的，隧道一段有一个IP地址，另一端也有唯一的IP地址

移动IPv4主要有三种隧道技术: IP in IP、最小封装及通用路由封装

IP隧道技术(IP封装技术): 是路由器把一种网络层协议封装到另一协议中以跨过网络传送到另一个网络的处理过程

**IP-in-IP** 一种IP隧道协议(IP Tunnel)，将一个IP数据包封装进另一个IP数据包中

**VXLAN** 由VMware、Cisco、RedHat等联合提出的一个解决方案，主要解决VLAN支持虚拟网络数量(4096)过少的问题，在公有云上每一个租户都有不同的VPC，4096个明显不够用，vxLAN可以支持1600万个虚拟网络，基本上够公有云使用

**BGP(Border Gateway Protocol)边界路由协议** 主干网上一个核心去中心化自治网络路由协议，互联网由很多小的自治网络构成，其间三层路由是由BGP实现

**SDN** 软件定义网络

**路由工作负载IP地址** 当网络知道工作负载IP地址时，它可以直接在工作负载之间路由流量



## 其他

### NetworkManager和Calico

```
Linux中NetworkManager对Calico接口的影响
NetworkManager管理主机默认网络命名空间中接口的路由表的功能，这可能会干扰Calico正确处理网络路由的能力
为了确保Calico能够管理主机上的cali和tunl接口，如果主机上运行着NetworkManager则需按下面的方法配置NetworkManager
创建一个配置文件 /etc/NetworkManager/conf.d/calico.conf 来制止这种干扰

cat > /etc/NetworkManager/conf.d/calico.conf <<EOF
[keyfile]
unmanaged-devices=interface-name:cali*;interface-name:tunl*
EOF
```



## 附件

### set-ns-veth-calico.sh

```bash
#!/bin/bash
## Author: lujianguo
## Email：guoguodelu@126.com
## Date: 20211208
## Version: v1.0
## Description:  Simulate calico network environment.
## 中文描述
## 共识: 变量-小写字母加下划线，函数-开头字母大写加下划线
############################################################

############################################################
### Environmental configuration ###
############################################################
export PATH=$PATH:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/root/bin

############################################################
### Define globle variables ###
############################################################
veth_in_ns=${default:-veth0}
veth_out_ns=${default:-cali0}
ns=${default:-ns0}
ns_gw=${default:-169.254.1.1}
host_interface=${default:-eth0}

ns_ip=${2:-172.16.0.10}
destion_ip=${3}
destion_cidr=${4:-172.16.1.0/24}

############################################################
### Define functions ###
############################################################
function Echo_Color() {
## 31=red,32=green,33=yellow,34=blue
  echo -e "\e[$1;1m$2\e[0m"
}

function Usage_Help(){
  cat <<EOF
  Operate ns and veth pair, ip and route.
  Usage:
    bash $(basename $0) [ option ] <local namespace ip> <destion host ip> <destion namespace network>
  Option:
    -s set     Add namespace/veth pair,  set ip and route.
       bash $(basename $0) set [ local namespace ip ] [ destion host ip  ] [ destion namespace network  ]
    -d delete  Delete namespace and route.
  Example:
    bash $(basename $0) set 172.16.0.2 192.168.1.100  172.16.1.0/24
    bash $(basename $0) delete

EOF
exit 1
}

function Handle_Ns(){
    ip netns exec "$ns" ip a add "$ns_ip"/24 dev "$veth_in_ns"
    ip netns exec "$ns" ip link set "$veth_in_ns" up
    ip netns exec "$ns" ip route add "$ns_gw" dev "$veth_in_ns" scope link
    ip netns exec "$ns" ip route add default via "$ns_gw" dev "$veth_in_ns"
}

function Handle_Host(){
  ip netns | grep -w "$ns" &> /dev/null 
  if [ $? -ne 0 ] ; then
    ip link add "$veth_out_ns" type veth peer name "$veth_in_ns"
    ip netns add "$ns"
    ip link set "$veth_in_ns"  netns "$ns"
    ip link set "$veth_out_ns" up
    ip route add "$ns_ip" dev "$veth_out_ns" scope link
    ip route add "$destion_cidr" via "$destion_ip" dev "$host_interface"
    echo 1 > /proc/sys/net/ipv4/conf/"$veth_out_ns"/proxy_arp
    echo 1 > /proc/sys/net/ipv4/ip_forward
    Echo_Color 32 "Create namepace successfull!!"
  else
    Echo_Color 31 "Namespace $ns  exists!!" && exit 1
  fi
}

function Handle_Delete() {
  ip netns | grep -w "$ns" &> /dev/null && ip netns delete $ns && Echo_Color 32 "Delete namepace successfull!!"
  ip route delete "$destion_cidr"
}

function main(){
case $1 in 
  set|-s ) [ x$destion_ip == "x" ] && Usage_Help  ||  Handle_Host && Handle_Ns ;;
  delete|-d )  Handle_Delete ;;
  * ) Usage_Help ;;
esac
}

############################################################
### Run Functions ###
############################################################
main "$@"

```

[kubernetes 网络组件 calico 运行原理分析](https://codeantenna.com/a/ZqrnjV0Cza)

[calico官网](https://projectcalico.docs.tigera.io/getting-started/kubernetes/quickstart)

[Calico 网络插件运维完全指南](https://freewechat.com/a/MzU1MzY4NzQ1OA==/2247497046/2)