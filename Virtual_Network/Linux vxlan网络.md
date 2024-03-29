---
title: Linux vxlan网络
---

# Vxlan介绍
VXLAN 通过在 UDP 数据包中封装 2 层帧，将第 2 层网络叠加到第 3 层基础结构上。每个覆盖网络被称为 VXLAN 分段，由称为 VXLAN 网络标识符 (VNI) 的唯一 24 位标识符进行标识。只有同一 VXLAN 中的网络设备才能相互通信。

VXLAN 提供与 VLAN 相同的以太网第 2 层网络服务，但具有更高的可扩展性和灵活性。使用 VXLAN 的两个主要优点如下：

- 更高的可扩展性。服务器虚拟化和云计算架构大大增加了对数据中心中隔离的第 2 层网络的需求。VLAN 规范使用 12 位 VLAN ID 来标识第 2 层网络，因此无法扩展到 4094 VLAN 以外。当需要数千个隔离的第 2 层网络时，该数字可能不足。24 位 VNI 可在同一管理域中容纳多达 1600 万个 VXLAN 段。
- 更高的灵活性。由于 VXLAN 在第 3 层数据包上传送第 2 层数据框，因此 VXLAN 将 L2 网络扩展到数据中心的不同部分和地理上分离的数据中心。托管在数据中心的不同部分和不同数据中心但属于同一 VXLAN 的应用程序将显示为一个连续网络。



# 点对点



# 多播



# VETP



# FDB



# ARP

# 附件
https://cizixs.com/2017/09/28/linux-vxlan/
https://docs.citrix.com/zh-cn/citrix-adc/current-release/networking/vxlans.html
