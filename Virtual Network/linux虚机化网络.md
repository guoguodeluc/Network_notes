---
title: linux 虚拟化网络
---

Linux有丰富的虚拟化网络功能，这些功能作为支持虚拟机和容器的基础。

# 虚拟网络接口

在云计算时代，虚拟机和容器已经成为标配。背后的网络管理都离不开虚拟网络设备，而tap/tun 就是在云计算时代非常重要的网络设备。

虚拟网络设备也归内核的网络设备管理子系统管理。对于Linux内核网络设备管理模块来说，虚拟设备和物理设备没有区别，都是网络设备，都能配置IP，从网络设备来的数据，都会转发给协议栈，协议栈过来的数据，也会交由这些网络设备发送出去，至于是如何发送、发送到哪，这都由设备驱动来完成，跟Linux内核就没关系了。

Linux tun/tap驱动程序为应用程序提供了两种交互方式：虚拟网络接口和字符设备/dev/net/tun。写入字符设备/dev/net/tun的数据会发送到虚拟网络接口中；发送到虚拟网络接口中的数据也会出现在该字符设备上。

tap/tun 驱动程序包括两个部分，一个是字符设备驱动，一个是网卡驱动。字符驱动负责数据包在内核空间和用户空间的传送，网卡驱动负责数据包在 TCP/IP 网络协议栈上的传输和处理。

/dev/net/tun 管理的虚拟网络接口有两种类型：

- TAP 接口传输以太网帧（第 2 层）

- TUN 接口传输 IP 数据包（第 3 层）

虚拟网卡的两个主要功能是：

- 连接其它设备(虚拟网卡或物理网卡)和虚拟交换机(bridge)
- 提供用户空间程序去收发虚拟网卡上的数据

基于这两个功能，**tap设备通常用来连接其它网络设备(它更像网卡)，tun设备通常用来结合用户空间程序实现再次封装**。换句话说，tap设备通常接入到虚拟交换机(bridge)上作为局域网的一个节点，tun设备通常用来实现三层的ip隧道。

虽然它们的工作方式完全相同，但还是有一些区别：

- 户层程序通过tun设备只能读写IP数据包，而通过tap设备能读写链路层数据包;

- tun设备相当于是一个三层设备，，它无法与物理网卡做 bridge，但是可以通过三层交换（如 ip_forward）与物理网卡连通。可以使用ifconfig之类的命令给该设备设定 IP 地址；

- tap设备则相当于是一个二层设备，可以工作在数据链路层，拥有 MAC 层功能，可以与物理网卡做 bridge，支持 MAC 层广播。可以通过ifconfig之类的命令给该设备设定 IP 地址，甚至还可以给它设定 MAC 地址。

## tap

tun/tap 是操作系统内核中的虚拟网络设备，他们为用户层程序提供数据的接收与传输。

普通的物理网络接口如 eth0，它的两端分别是内核协议栈和外面的物理网络。

### tap创建

```bash
ip tuntap add dev tap0 mode tap
```

> 其他创建方式，都需要额外安装工具包
>
> - `openvpn --mktun --dev tap0`
> - `tunctl -t tap0`

> openvpn创建时时根据--dev tunX|tapX 名称来创建对应的tap或者tun类型，其他名称会报错：I don't recognize device xxx as a tun or tap device
>
> --mktun         : Create a persistent tunnel.
> --rmtun         : Remove a persistent tunnel.
> --dev tunX|tapX : tun/tap device

### tap查看

```bash
# ip -d link show dev tap0
5: tap0: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN mode DEFAULT group default qlen 100
    link/ether aa:d2:d6:b4:45:13 brd ff:ff:ff:ff:ff:ff promiscuity 0 minmtu 68 maxmtu 65521 
    tun type tap pi off vnet_hdr off persist on addrgenmode eui64 numtxqueues 1 numrxqueues 1 gso_max_size 65536 gso_max_segs 65535 
```

### tap删除

```bash
ip link delete dev tap0
```

> `tunctl -d tap0`

### tap使用场景



https://www.zhaohuabing.com/post/2020-02-24-linux-taptun/

https://www.cnblogs.com/4geek/p/12611095.html

## tun

tun 和 tap 的区别在于工作的网络层次不同，用户程序通过 tun设备只能读写网络层的 IP 数据包，而tap设备则支持读写链路层的数据包（通常是以太网数据包，带有 Ethernet headers）。

tun与tap的关系，就类似于 socket 和 raw socket.

### tun创建

```bash
ip tuntap add dev tun0 mode tun
```

> 其他创建方式，都需要额外安装工具包
>
> - `openvpn --mktun --dev tun0`
> - `tunctl -t tun0`

### tun查看

```bash
# ip -d link show dev tun0
11: tun0: <POINTOPOINT,MULTICAST,NOARP> mtu 1500 qdisc noop state DOWN mode DEFAULT group default qlen 500
    link/none  promiscuity 0 minmtu 68 maxmtu 65535 
    tun type tun pi off vnet_hdr off persist on addrgenmode eui64 numtxqueues 1 numrxqueues 1 gso_max_size 65536 gso_max_segs 65535
```

### tun删除

```
ip link delete dev tun0
```

> tunctl -3  -d tun0

### tun使用场景

tun/tap应用最多的场景是 VPN 代理，比如:

1. [clash](https://github.com/ryan4yin/clash): 一个支持各种规则的隧道，也支持 TUN 模式
2. [tun2socks](https://github.com/xjasonlyu/tun2socks/wiki): 一个全局透明代理，和 VPN 的工作模式一样，它通过创建虚拟网卡+修改路由表，在第三层网络层代理系统流量。

## veth

veth 接口总是成对出现，一对 veth 接口就类似一根网线，从一端进来的数据会从另一端出去。

同时 veth 又是一个虚拟网络接口，因此它和 TUN/TAP 或者其他物理网络接口一样，也都能配置 mac/ip 地址（但是并不是一定得配 mac/ip 地址）。

其主要作用就是连接不同的网络，比如在容器网络中，用于将容器的 namespace 与 root namespace 的网桥 br0 相连。 容器网络中，容器侧的 veth 自身设置了 ip/mac 地址并被重命名为 eth0，作为容器的网络接口使用，而主机侧的 veth 则直接连接在 docker0/br0 上面。

使用 veth 实现容器网络，需要结合下一小节介绍的 bridge，在下一小节将给出容器网络结构图





# 虚机网桥

## bridge

Linux Bridge 是工作在链路层的网络交换机，由 Linux 内核模块 `brige` 提供，它负责在所有连接到它的接口之间转发链路层数据包。添加到 Bridge 上的设备被设置为只接受二层数据帧并且转发所有收到的数据包到 Bridge 中。 在 Bridge 中会进行一个类似物理交换机的查MAC端口映射表、转发、更新MAC端口映射表这样的处理逻辑，从而数据包可以被转发到另一个接口/丢弃/广播/发往上层协议栈，由此 Bridge 实现了数据转发的功能。

Linux 网桥的行为类似于网络交换机。它在连接到它的接口之间转发数据包。它通常用于在路由器、网关或虚拟机与主机上的网络命名空间之间转发数据包。它还支持 STP、VLAN 过滤器和组播侦听。

当您想在虚拟机、容器和主机之间建立通信通道时，请使用网桥。

### bridge创建

```bash
## 添加网桥
ip link add testlu-br type bridge
## 添加tap
ip tuntap add mode tap name tap0
ip tuntap add mode tap name tap1
## 将tap添加到网桥
ip link set tap0 master testlu-br
ip link set tap1 master testlu-br
```

> [网桥操作](https://unix.stackexchange.com/questions/255484/how-can-i-bridge-two-interfaces-with-ip-iproute2)

### bridge查看

```bash
brctl show
bridge link
```



### bridge删除

```bash
ip link del testlu-br
ip link del tap0
ip link del tap1
```

### bridge使用场景

在虚拟机、容器和主机之间建立通信通道时，通常会使用网桥



## ovs



# 子接口

## macvlan 



## ipvlan





# overlay

## vxlan

**VXLAN**（Virtual eXtensible Local Area Network，虚拟扩展局域网），是由IETF定义的NVO3（Network Virtualization over Layer 3）标准技术之一，是对传统VLAN协议的一种扩展。

*VTEP*（*VXLAN* Tunnel Endpoints）： *VTEP*是*VXLAN*隧道端点，封装在NVE中，用于*VXLAN*报文的封装和解封装。



## geneve



# 附件

参考资料：[redhat官网](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/8/html/configuring_and_managing_networking/assembly_using-a-vxlan-to-create-a-virtual-layer-2-domain-for-vms_configuring-and-managing-networking) [参考1](https://thiscute.world/posts/linux-virtual-network-interfaces/#vxlan-geneve)  [参考2](https://cizixs.com/2017/09/28/linux-vxlan/)

[在 Linux 上配置 VXLAN 网络](https://fuckcloudnative.io/posts/vxlan-linux/)

[kvm ovs vxlan 实验](https://www.bbsmax.com/topic/kvm-ovs-vxlan-%E5%AE%9E%E9%AA%8C/)

https://developers.redhat.com/blog/2018/10/22/introduction-to-linux-interfaces-for-virtual-networking

https://github.com/jpirko/libteam/wiki/Bonding-vs.-Team-features

https://gist.github.com/mtds/4c4925c2aa022130e4b7c538fdd5a89f

## 网络专业术语



**fdb** 英文： Forwarding Database  中文： MAC地址表

**ARP** 英文：[Address Resolution Protocol](http://en.wikipedia.org/wiki/Address_Resolution_Protocol)  中文：地址解析协议

ARP表：IP和MAC的对应关系；3层设备使用，例如：路由器，交换机，服务器，桌面等

FDB表：MAC+VLAN和PORT的对应关系；2 层设备使用，例如：交换机/网桥，用于存储已获知的 MAC 地址以及获知 MAC 地址的端口

最大的区别在于ARP是三层转发，FDB是用于二层转发。两个设备不在一个网段或者没配IP，只要两者之间的链路层是连通的，就可以通过FDB表进行数据的转发！

**IGMP** 英文：Internet Group Management Protocol 中文：Internet 组管理协议称为协议。该协议运行在主机和组播路由器之间

