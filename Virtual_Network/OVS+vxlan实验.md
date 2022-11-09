---
title: OVS+vxlan实验
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

# OVS+vxlan综合实验

[ovs github](https://github.com/openvswitch)，[ovs官网下载](http://www.openvswitch.org/download/)，[ovs命令使用](https://www.xiexianbin.cn/sdn/openvswitch/ovs-vsctl/index.html)

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
IDX LINK       TYPE     OPERATIONAL SETUP     
  1 lo         loopback carrier     unmanaged 
  2 eth0       ether    routable    configured
  8 vxlan0     vxlan    routable    unmanaged 
  9 ovs-system ether    off         unmanaged 
 10 lu-br0     ether    carrier     unmanaged 
 12 veth0      ether    carrier     unmanaged 
 13 vnet0      ether    carrier     unmanaged 
     
# ovs-vsctl show
bc4d3e2f-7eda-4505-b7a1-caf5e5d2ca29
    Bridge lu-br0
        Port lu-br0
            Interface lu-br0
                type: internal
        Port vxlan0
            Interface vxlan0
        Port vnet0
            Interface vnet0
        Port veth0
            Interface veth0
    ovs_version: "2.13.5"
```

> vnet0是连接虚机，veth0是连接容器

## 安装ovs

- centos

[下源码包](https://www.openvswitch.org/releases/openvswitch-2.13.7.tar.gz)，根据[源码包做出rpm包](https://gist.github.com/umardx/a31bf6a13600a55c0d07d4ca33133834)，使用rpm安装。

```bash
## rpm是源码包制作成
yum localinstall -y openvswitch-2.13.7-1.el7.x86_64.rpm
systemctl enable openvswitch.service --now
```

- ubuntu

```bash
apt install openvswitch-switch
```

常用命令

```
ovs-vsctl --version
ovs-vsctl show
```

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
ovs-vsctl add-br lu-br0
ovs-vsctl add-port lu-br0 vxlan0
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
ovs-vsctl add-port lu-br0 veth0
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
ovs-vsctl add-port lu-br0 veth0
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
  <virtualport type="openvswitch"/>
</network>
EOF
virsh net-define lu-br0.xml
virsh net-start lu-br0
virsh net-autostart lu-br0
```

> 创建网络在/etc/libvirt/qemu/networks/可以查看

### 创建虚机-vm

```bash
virt-install --name testlu-vm \
--memory 512 \
--disk /var/lib/libvirt/images/cirros-0.5.2-x86_64-disk.img,device=disk,bus=virtio,format=qcow2 \
--network network=lu-br0,virtualport_type=openvswitch,model=virtio \
--graphics vnc,listen=0.0.0.0,port=5900  --noautoconsole \
--import
```

> --virt-type kvm \
>
> virtualport_type=openvswitch需要添加，否则会报错：ERROR    Unable to add bridge br0 port vnet0: Operation not supported

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
ovs-vsctl del-br lu-br0
## 删除vxlan
ip link delete vxlan0
```

> 按照创建相反的顺序来清理环境。



# 附件



## ovs-vsctl帮助

```bash
# ovs-vsctl --help
ovs-vsctl: ovs-vswitchd management utility
usage: ovs-vsctl [OPTIONS] COMMAND [ARG...]

Open vSwitch commands:
  init                        initialize database, if not yet initialized
  show                        print overview of database contents
  emer-reset                  reset configuration to clean state

Bridge commands:
  add-br BRIDGE               create a new bridge named BRIDGE
  add-br BRIDGE PARENT VLAN   create new fake BRIDGE in PARENT on VLAN
  del-br BRIDGE               delete BRIDGE and all of its ports
  list-br                     print the names of all the bridges
  br-exists BRIDGE            exit 2 if BRIDGE does not exist
  br-to-vlan BRIDGE           print the VLAN which BRIDGE is on
  br-to-parent BRIDGE         print the parent of BRIDGE
  br-set-external-id BRIDGE KEY VALUE  set KEY on BRIDGE to VALUE
  br-set-external-id BRIDGE KEY  unset KEY on BRIDGE
  br-get-external-id BRIDGE KEY  print value of KEY on BRIDGE
  br-get-external-id BRIDGE  list key-value pairs on BRIDGE

Port commands (a bond is considered to be a single port):
  list-ports BRIDGE           print the names of all the ports on BRIDGE
  add-port BRIDGE PORT        add network device PORT to BRIDGE
  add-bond BRIDGE PORT IFACE...  add bonded port PORT in BRIDGE from IFACES
  del-port [BRIDGE] PORT      delete PORT (which may be bonded) from BRIDGE
  port-to-br PORT             print name of bridge that contains PORT

Interface commands (a bond consists of multiple interfaces):
  list-ifaces BRIDGE          print the names of all interfaces on BRIDGE
  iface-to-br IFACE           print name of bridge that contains IFACE

Controller commands:
  get-controller BRIDGE      print the controllers for BRIDGE
  del-controller BRIDGE      delete the controllers for BRIDGE
  [--inactivity-probe=MSECS]
  set-controller BRIDGE TARGET...  set the controllers for BRIDGE
  get-fail-mode BRIDGE       print the fail-mode for BRIDGE
  del-fail-mode BRIDGE       delete the fail-mode for BRIDGE
  set-fail-mode BRIDGE MODE  set the fail-mode for BRIDGE to MODE

Manager commands:
  get-manager                print the managers
  del-manager                delete the managers
  [--inactivity-probe=MSECS]
  set-manager TARGET...      set the list of managers to TARGET...

SSL commands:
  get-ssl                     print the SSL configuration
  del-ssl                     delete the SSL configuration
  set-ssl PRIV-KEY CERT CA-CERT  set the SSL configuration

Auto Attach commands:
  add-aa-mapping BRIDGE I-SID VLAN   add Auto Attach mapping to BRIDGE
  del-aa-mapping BRIDGE I-SID VLAN   delete Auto Attach mapping VLAN from BRIDGE
  get-aa-mapping BRIDGE              get Auto Attach mappings from BRIDGE

Switch commands:
  emer-reset                  reset switch to known good state

Database commands:
  list TBL [REC]              list RECord (or all records) in TBL
  find TBL CONDITION...       list records satisfying CONDITION in TBL
  get TBL REC COL[:KEY]       print values of COLumns in RECord in TBL
  set TBL REC COL[:KEY]=VALUE set COLumn values in RECord in TBL
  add TBL REC COL [KEY=]VALUE add (KEY=)VALUE to COLumn in RECord in TBL
  remove TBL REC COL [KEY=]VALUE  remove (KEY=)VALUE from COLumn
  clear TBL REC COL           clear values from COLumn in RECord in TBL
  create TBL COL[:KEY]=VALUE  create and initialize new record
  destroy TBL REC             delete RECord from TBL
  wait-until TBL REC [COL[:KEY]=VALUE]  wait until condition is true
Potentially unsafe database commands require --force option.
Database commands may reference a row in each table in the following ways:
  AutoAttach:
    by UUID
    via "auto_attach" of Bridge with matching "name"
  Bridge:
    by UUID
    by "name"
  CT_Timeout_Policy:
    by UUID
  CT_Zone:
    by UUID
  Controller:
    by UUID
    via "controller" of Bridge with matching "name"
  Datapath:
    by UUID
  Flow_Sample_Collector_Set:
    by UUID
    by "id"
  Flow_Table:
    by UUID
    by "name"
  IPFIX:
    by UUID
    via "ipfix" of Bridge with matching "name"
  Interface:
    by UUID
    by "name"
  Manager:
    by UUID
    by "target"
  Mirror:
    by UUID
    by "name"
  NetFlow:
    by UUID
    via "netflow" of Bridge with matching "name"
  Open_vSwitch:
    by UUID
    as "."
  Port:
    by UUID
    by "name"
  QoS:
    by UUID
    via "qos" of Port with matching "name"
  Queue:
    by UUID
  SSL:
    by UUID
    as "."
  sFlow:
    by UUID
    via "sflow" of Bridge with matching "name"

Options:
  --db=DATABASE               connect to DATABASE
                              (default: unix:/var/run/openvswitch/db.sock)
  --no-wait                   do not wait for ovs-vswitchd to reconfigure
  --retry                     keep trying to connect to server forever
  -t, --timeout=SECS          wait at most SECS seconds for ovs-vswitchd
  --dry-run                   do not commit changes to database
  --oneline                   print exactly one line of output per command

Output formatting options:
  -f, --format=FORMAT         set output formatting to FORMAT
                              ("table", "html", "csv", or "json")
  -d, --data=FORMAT           set table cell output formatting to
                              FORMAT ("string", "bare", or "json")
  --no-headings               omit table heading row
  --pretty                    pretty-print JSON in output
  --bare                      equivalent to "--format=list --data=bare --no-headings"

Logging options:
  -vSPEC, --verbose=SPEC   set logging levels
  -v, --verbose            set maximum verbosity level
  --log-file[=FILE]        enable logging to specified FILE
                           (default: /var/log/openvswitch/ovs-vsctl.log)
  --syslog-method=(libc|unix:file|udp:ip:port)
                           specify how to send messages to syslog daemon
  --syslog-target=HOST:PORT  also send syslog msgs to HOST:PORT via UDP
  --no-syslog             equivalent to --verbose=vsctl:syslog:warn

Active database connection methods:
  tcp:HOST:PORT           PORT at remote HOST
  ssl:HOST:PORT           SSL PORT at remote HOST
  unix:FILE               Unix domain socket named FILE
Passive database connection methods:
  ptcp:PORT[:IP]          listen to TCP PORT on IP
  pssl:PORT[:IP]          listen for SSL on PORT on IP
  punix:FILE              listen on Unix domain socket FILE
PKI configuration (required to use SSL):
  -p, --private-key=FILE  file with private key
  -c, --certificate=FILE  file with certificate for private key
  -C, --ca-cert=FILE      file with peer CA certificate
  --bootstrap-ca-cert=FILE  file with peer CA certificate to read or create
SSL options:
  --ssl-protocols=PROTOS  list of SSL protocols to enable
  --ssl-ciphers=CIPHERS   list of SSL ciphers to enable

Other options:
  -h, --help                  display this help message
  -V, --version               display version information
```

