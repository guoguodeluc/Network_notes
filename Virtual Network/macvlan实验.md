---
title: macvlan实验
---

# macvlan介绍

macvlan允许在主机的一个网络接口上配置多个虚拟的网络接口，这些网络网卡有自己独立的 MAC 地址，也可以配置上 IP 地址进行通信。

> 只要是从macvlan 子接口发来的数据包，物理网卡只接收数据包，不处理数据包，所以这就引出了一个问题：本机macvlan网卡上面的IP无法和物理网卡上面的 IP 通信。

# macvlan模式

当想从容器直接连接到物理网络时，[macvlan](https://developers.redhat.com/blog/2018/10/22/introduction-to-linux-interfaces-for-virtual-networking#macvlan)是一个不错的选择，根据不同的需要选择类型，bridge模式是最常用的。

五种 MACVLAN 类型

1. **Private**：不允许同一物理接口上的macvlan实例之间进行通信，即使外部交换机支持hairpin 模式。
2. **VEPA**：从一个macvlan实例到同一物理接口上的另一个实例的数据通过物理接口传输，连接的交换机需要支持hairpin模式，或者必须有一个 TCP/IP 路由器转发数据包才能允许通信。
3. **Bridge**：所有端点通过物理接口通过简单的网桥直接相互连接。
4. **Passthru**：允许单个 VM 直接连接到物理接口。
5. **Source**：源模式用于根据允许的源 MAC 地址列表过滤流量，以创建基于 MAC 的 VLAN 关联。[source相关信息](https://git.kernel.org/pub/scm/linux/kernel/git/netdev/net.git/commit/?id=79cf79abce71)



# 创建macvlan网卡

```bash
## ip link add link [宿主机网卡] [macvlan名称]  type macvlan mode [macvlan类型]
ip link add link eth0 macvlan0  type macvlan mode bridge
## 添加网卡的通用格式
ip link add [link DEV] [ name ] NAME
		    [ txqueuelen PACKETS ]
		    [ address LLADDR ]
		    [ broadcast LLADDR ]
		    [ mtu MTU ] [index IDX ]
		    [ numtxqueues QUEUE_COUNT ]
		    [ numrxqueues QUEUE_COUNT ]
		    type TYPE [ ARGS ]
```



# macvlan使用场景

**使用 Macvlan：**

- 仅仅需要为虚拟机或容器提供访问外部物理网络的连接。
- Macvlan 占用较少的 CPU，同时提供较高的吞吐量。
- 当使用 Macvlan 时，宿主机无法和 VM 或容器直接进行通讯。

**使用 Bridge：**

- 当在同一台宿主机上需要连接多个虚拟机或容器时。
- 对于拥有多个网桥的混合环境。
- 需要应用高级流量控制，FDB的维护。



Macvlan 是将 VM 或容器通过二层连接到物理网络的近乎理想的方案，但它也有一些局限性：

- Linux 主机连接的交换机可能会限制同一个物理端口上的 MAC 地址数量。虽然你可以让网络管理员更改这些策略，但有时这种方法是无法实行的（比如你要去给客户做一个快速的 PoC 演示）。
- 许多 NIC 也会对该物理网卡上的 MAC地址数量有限制。超过这个限制就会影响到系统的性能。
- IEEE 802.11 不喜欢同一个客户端上有多个 MAC 地址，这意味着你的 Macvlan 子接口在无线网卡或 AP 中都无法通信。可以通过复杂的办法来突破这种限制，但还有一种更简单的办法，那就是使用 `Ipvlan`，感兴趣可以自己查阅相关资料。



# 模拟容器使用macvlan

## 创建macvlan接口

```bash
parent_interface="eth0"
# 开启混杂模式
ip link set "$parent_interface" promisc on
## 创建macvlan接口
ip link add link "$parent_interface" macvlan1 type macvlan mode bridge
ip link add link "$parent_interface" macvlan2 type macvlan mode bridge
```

> parent_interface是 Macvlan 接口的父接口名称，这里使用macvlan的bridge模式，使用`ip -d link show macvlan0`详细查看信息
>
> 服务器网卡都工作在非**混杂模式**下，此时网卡只接受来自网络端口的目的地址指向自己的数据。 当网卡工作在**混杂模式**下时，网卡将来自接口的所有数据都捕获并交给相应的驱动程序。macvlan必须要开启**混杂模式**

## 创建模拟容器

```bash
## 设置ns地址是和宿主机在同一个段
ns1_ip="192.168.88.11"
ns2_ip="192.168.88.12"
## 创建的macvlan接口并放入network namespace中，配置IP地址
## 测试容器1
ip netns add ns1
ip link set macvlan1 netns ns1
ip netns exec ns1 ip addr add "$ns1_ip"/24 dev macvlan1
ip netns exec ns1 ip link set dev macvlan1 up
## 测试容器2
ip netns add ns2
ip link set macvlan2 netns ns2
ip netns exec ns2 ip addr add "$ns2_ip"/24 dev macvlan2
ip netns exec ns2 ip link set dev macvlan2 up
```

## 验证连通性

```bash
## 设置ns地址是和宿主机在同一个段
ns1_ip="192.168.88.11"
ns2_ip="192.168.88.12"
ip netns exec ns2  ping "$ns1_ip"
```

>  当使用macvlan时，宿主机无法和VM或容器直接进行通讯



# 虚机使用macvlan

## 创建虚机

kvm安装部署步骤就不该次实验体现，直接部署虚机

```bash
parent_interface="eth0"
# 开启混杂模式
ip link set "$parent_interface" promisc on
## 创建虚机
virt-install --name testlu-vm \
--virt-type kvm \
--memory 512 \
--disk /var/lib/libvirt/images/cirros-0.5.2-x86_64-disk.img,device=disk,bus=virtio,format=qcow2 \
--network model=virtio,type=direct,source=eth0,source.mode=bridge  \
--graphics vnc,listen=0.0.0.0,port=5900  --noautoconsole \
--import
```

> [cirros测试镜像](http://download.cirros-cloud.net/0.5.2/cirros-0.5.2-x86_64-disk.img)下载，比较小测试起来比较方便。
>
> `virt-install --network ?`查看 network相关支持参数。使用macvtap的bridge模式，默认是vepa模式

virsh查看网卡类型

```bash
# virsh domiflist testlu-vm 
 Interface   Type     Source   Model    MAC
-----------------------------------------------------------
 macvtap0    direct   eth0     virtio   52:54:00:da:6b:9b
# virsh attach-interface testlu-vm --type direct --source bond0  --model virtio
```





## host1

```bash
# virsh console testlu-vm
sudo su -
host1_vm_ip="192.168.88.21"
ip add flush dev eth0
ip addr add "$host1_vm_ip"/24 dev eth0
```

> cirros默认账号cirros密码gocubsgo

## host2

```bash
# virsh console testlu-vm
sudo su -
host2_vm_ip="192.168.88.22"
ip add flush dev eth0
ip addr add "$host2_vm_ip"/24 dev eth0
```

## host2上验证连通性

```bash
host1_vm_ip="192.168.88.21"
host2_vm_ip="192.168.88.22"
ping -c 1 "$host1_vm_ip"
```

>  当使用macvlan时，宿主机无法和VM或容器直接进行通讯

# 清理环境

```bash
ip netns delete ns1
ip netns delete ns2
ip link delete macvlan1
ip link delete macvlan2
virsh destroy testlu-vm && virsh undefine testlu-vm
```



# 附件

https://www.cnblogs.com/yudar/p/4630958.html