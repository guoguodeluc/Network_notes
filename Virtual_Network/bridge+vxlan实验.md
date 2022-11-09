---
title:bridge+vxlan实验
---

[TOC]

# 实验环境

为了实现虚机或者容器在不同的host上相互通信。

## HOST环境信息

需要两台host，这里不关心host的IP地址。

```bash
# lsb_release -a
No LSB modules are available.
Distributor ID:	Ubuntu
Description:	Ubuntu 20.04.3 LTS
Release:	20.04
Codename:	focal
# uname -r
5.4.0-92-generic
```

> 如果可以，尽量使用比较新版本的 kernel，以免出现因为内核版本太低导致功能或者性能上的问题。

## 虚拟化环境准备

- 安装kvm

```bash
## ubuntu
apt update
apt install -y qemu qemu-kvm libvirt-daemon-system libvirt-clients bridge-utils virtinst  cpu-checker  libguestfs-tools 
## centos
# yum install -y qemu-kvm qemu-img libvirt virt-install libguestfs-tools libguestfs-bash-completion 
```

- 下载测试镜像

```bash
cd /var/lib/libvirt/images/
wget http://download.cirros-cloud.net/0.5.2/cirros-0.5.2-x86_64-disk.img
```

[cirros测试镜像](http://download.cirros-cloud.net/0.5.2/cirros-0.5.2-x86_64-disk.img)下载，比较小测试起来比较方便。



# bridge+vxlan综合实验

## 网络架构图

```bash
+--------------------------------------------------+
|    host1                                         |
|                                                  |
|     +--------+     +-------+                     |
|     |  vm1   +<--->+       |     +------+     +------+
|     +--------+     |       |     |      |     |      |
|                    |lu-br0 +<--->+vxlan0+<--->+ eth0 +<----------->+
|     +--------+     |       |     |      |     |      |             |
|     |  cn1   +<--->+       |     +------+     +------+             |
|     +--------+     +-------+                     |                 |
|                                                  |                 |
+--------------------------------------------------+                 |
                                                             +-----------------+ 
                                                             |     组播地址     |
                                                             |    239.0.0.1    |
                                                             +-----------------+ 
+--------------------------------------------------+                 |
|    host2                                         |                 |
|                                                  |                 |
|     +--------+     +-------+                     |                 |
|     |  vm2   +<--->+       |     +------+     +------+             |
|     +--------+     |       |     |      |     |      |             |          
|                    |lu-br0 +<--->+vxlan0+<--->+ eth0 +<----------->|
|     +--------+     |       |     |      |     |      |
|     |  cn2   +<--->+       |     +------+     +------+
|     +--------+     +-------+                     |
|                                                  |
+--------------------------------------------------+
```

网口和网桥信息

```bash
# networkctl 
IDX LINK   TYPE     OPERATIONAL SETUP     
  1 lo     loopback carrier     unmanaged 
  2 eth0   ether    routable    configured
  3 vxlan0 vxlan    routable    unmanaged 
  4 lu-br0 bridge   carrier     unmanaged 
  6 veth0  ether    carrier     unmanaged 
  7 vnet0  ether    carrier     unmanaged 

# brctl show
bridge name	bridge id		STP enabled	interfaces
lu-br0		8000.4e00b9fae4d4	no		veth0
							            vnet0
							            vxlan0
```

> NetworkManager工具使用nmcli device status，systemd-networkd使用networkctl工具

## Vxlan网络准备

### 主机1-设置vxlan

```bash
vxlan_id="100"
dstport="4789"
group_ip="239.0.0.1"
host1_vxlan0_ip="192.168.100.1"
host2_vxlan0_ip="192.168.100.2"
## 在主机1上创建vtep设备vxlan0与主机2vtep接口建立隧道，将vxlan0自身的IP地址设为host1_vxlan0_ip，使用的vxlan目标端口为 4789(IANA标准)
ip link add vxlan0 type vxlan \
    id "$vxlan_id" \
    dstport "$dstport" \
    group "$group_ip" \
    dev eth0
## 为vxlan网络设置虚拟网段
ip addr add "$host1_vxlan0_ip"/24 dev vxlan0
## 启用vxlan0设备，会自动生成路由规则
ip link set vxlan0 up
```

> [本地管理组播地址:](http://www.tcpipguide.com/free/t_IPMulticastAddressing.htm#:~:text=So%2C%20the%20full%20range%20of,a%20datagram%3B%20never%20the%20source.)239.0.0.0～239.255.255.255，仅在特定的本地范围内有效。
>
> 指定 VNI 的值，这个值可以在 1 到 2^24 之间

### 主机2-设置vxlan

```bash
vxlan_id="100"
dstport="4789"
group_ip="239.0.0.1"
host1_vxlan0_ip="192.168.100.1"
host2_vxlan0_ip="192.168.100.2"
## 在主机2上同样创建一个vtep设备vxlan0。注意group，vxlan id和dstport必须和前面完全一致。
ip link add vxlan0 type vxlan \
    id "$vxlan_id" \
    dstport "$dstport" \
    group "$group_ip" \
    dev eth0
## 为vxlan网络设置虚拟网段，并启用vxlan0
ip addr add "$host2_vxlan0_ip"/24 dev vxlan0
ip link set vxlan0 up
```

### host2上验证-vxlan

```bash
host1_vxlan0_ip="192.168.100.1"
## 可以在主机2上ping host1_vxlan0_ip试试，应该能收到主机1的回应。
ping -c1 -w 1 $host1_vxlan0_ip
```

### 删除vxlan0上地址

```bash
host1_vxlan0_ip="192.168.100.1"
host2_vxlan0_ip="192.168.100.2"
ip addr del "$host1_vxlan0_ip"/24 dev vxlan0
ip addr del "$host2_vxlan0_ip"/24 dev vxlan0
```

> 将vxlan0添加到br0上后，vxlan0上的ip地址就相互不通了，但是不影响隧道的正常使用和后面的实验。

### 验证完成添加网桥-vxlan

```bash
## 创建br0并将vxlan0绑定上去
ip link add lu-br0 type bridge
ip link set vxlan0 master lu-br0
ip link set vxlan0 up
ip link set lu-br0 up
```

## 创建容器

### 主机1-容器

```bash
host1_container0_ip="192.168.100.11"
host2_container0_ip="192.168.100.12"
## 使用ns模拟将容器加入到网桥中的
ip netns add container0
## 创建veth pair，并把一端加到网桥上
ip link add veth0 type veth peer name veth1
ip link set dev veth0 master lu-br0
ip link set dev veth0 up
## 配置容器内部的网络和IP
ip link set dev veth1 netns container0
ip netns exec container0 ip link set lo up

ip netns exec container0 ip link set veth1 name eth0
ip netns exec container0 ip addr add "$host1_container0_ip"/24 dev eth0
ip netns exec container0 ip link set eth0 up
```

### 主机2-容器

```bash
host1_container0_ip="192.168.100.11"
host2_container0_ip="192.168.100.12"
## 使用ns模拟将容器加入到网桥中的
ip netns add container0
## 创建veth pair，并把一端加到网桥上
ip link add veth0 type veth peer name veth1
ip link set dev veth0 master lu-br0
ip link set dev veth0 up
## 配置容器内部的网络和IP
ip link set dev veth1 netns container0
ip netns exec container0 ip link set lo up

ip netns exec container0 ip link set veth1 name eth0
ip netns exec container0 ip addr add "$host2_container0_ip"/24 dev eth0
ip netns exec container0 ip link set eth0 up
```

### host2上验证-容器

```bash
host1_container0_ip="192.168.100.11"
## 可以在主机2的netns上ping host1_container0_ip，应该能收到主机1上netns的回应。
ip netns exec container0 ping -c1 -w 1 "$host1_container0_ip"
```

> 如果需要端口互连需要注意mtu要设置成1400，本实验仅ping测通连接性，没有修改mtu值。

## 创建虚机

### 创建网络-vm

https://libvirt.org/formatdomain.html#virtual-network

```bash
##创建虚机的br0网络资源，并绑定到br0网桥上 
cat >lu-br0.xml <<EOF
<network>
  <name>lu-br0</name>
  <forward mode="bridge"/>
  <bridge name="lu-br0"/>
</network>
EOF
virsh net-define lu-br0.xml
virsh net-start lu-br0
virsh net-autostart lu-br0
```

### 创建虚机-vm

```bash
virt-install --name testlu-vm \
--memory 512 \
--disk /var/lib/libvirt/images/cirros-0.5.2-x86_64-disk.img,device=disk,bus=virtio,format=qcow2 \
--network network=lu-br0,model=virtio \
--graphics vnc,listen=0.0.0.0,port=5900  --noautoconsole \
--import
```

> --virt-type kvm \

### host2上验证-vm

#### 手动给虚机配置网络

由于创建的网路资源没有设置dhcp服务，创建的虚机是不会自动获取ip地址，需要手动设置ip地址。而且虚机会尝试和metadata服务连接，启动会比较慢。

- host1

```bash
# virsh console testlu-vm
############ debug end   ##############
#  ____               ____  ____
# / __/ __ ____ ____ / __ \/ __/
#/ /__ / // __// __// /_/ /\ \ 
#\___//_//_/  /_/   \____/___/ 
#   http://cirros-cloud.net
#
#
#login as 'cirros' user. default password: 'gocubsgo'. use 'sudo' for root.
#cirros login: cirros
#Password: 
sudo su -
host1_cirros_ip="192.168.100.21"
ip addr add "$host1_cirros_ip"/24 dev eth0
```

- host2

```bash
# virsh console testlu-vm
sudo su -
host2_cirros_ip="192.168.100.22"
ip addr add "$host2_cirros_ip"/24 dev eth0
```

#### 验证-vm

```bash
host1_cirros_ip="192.168.100.21"
## 可以在主机2的vm上ping host1_cirros_ip试试，应该能收到主机1上vm的回应。
ping -c1 -w 1 "$host1_cirros_ip"
```

> 如果需要端口互连需要注意mtu要设置成1400，本实验仅ping测通连接性，没有修改mtu值。

## host2上整体验证

```bash
# visrh console cirros
host1_container0_ip="192.168.100.11"
host2_container0_ip="192.168.100.12"
host1_cirros_ip="192.168.100.21"
host2_cirros_ip="192.168.100.22"
## 可以在主机2的vm上ping host1_cirros_ip试试，应该能收到主机1上vm的回应。
ping -c1 -w 1 "$host1_container0_ip"
ping -c1 -w 1 "$host2_container0_ip"
ping -c1 -w 1 "$host1_cirros_ip"
ping -c1 -w 1 "$host2_cirros_ip"
```

> 如果需要端口互连需要注意mtu要设置成1400，本实验仅ping测通连接性，没有修改mtu值。

## 清理环境

```bash
## 删除虚机
virsh destroy testlu-vm && virsh undefine testlu-vm
virsh net-destroy lu-br0  &&  virsh net-undefine lu-br0 
## 删除netns
ip netns delete container0
## 删除网桥
ip link delete lu-br0
## 删除vxlan
ip link delete vxlan0 
```

> 按照创建相反的顺序来清理环境。



# 附件

## kvm默认nat default网络

```bash
# virsh net-dumpxml default 
<network>
  <name>default</name>
  <uuid>36df69f7-e7ae-463f-ab5d-a53edc6606ea</uuid>
  <forward mode='nat'>
    <nat>
      <port start='1024' end='65535'/>
    </nat>
  </forward>
  <bridge name='virbr0' stp='on' delay='0'/>
  <mac address='52:54:00:cc:26:6a'/>
  <ip address='192.168.122.1' netmask='255.255.255.0'>
    <dhcp>
      <range start='192.168.122.2' end='192.168.122.254'/>
    </dhcp>
  </ip>
</network>

cat >br0.xml <<EOF
<network>
  <name>br0</name>
  <forward mode="bridge"/>
  <bridge name="br0"/>
  <virtualport type="openvswitch"/>
</network>
EOF
```
