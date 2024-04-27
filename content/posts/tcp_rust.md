+++
title = 'Tcp_rust'
date = 2024-04-27T19:04:32+08:00
draft = false
+++

### TUN/TAP: Linux feature 软件实现的操作系统内核中的虚拟网络设备.

TAP等同于一个**以太网**设备，它操作第二层数据包如以太网数据帧。TUN模拟了**网络层**设备，操作第三层数据包比如IP数据封包。

操作系统通过TUN/TAP设备向绑定该设备的用户空间的程序发送数据，反之，用户空间的程序也可以像操作硬件网络设备那样，通过TUN/TAP设备发送数据。在后种情况下，TUN/TAP设备向操作系统的网络栈投递（或“注入”）数据包，从而模拟从外部接受数据的过程。

正常情况下，Internet 的数据通过 NIC(network interface card)发送到 kernel，user space 程序通过socket 读写 kernel 中的数据。通过 TUN device模拟 NIC, 我们可以直接在user space 和 kernel 之间交流。

```bash
# 查看本机的 NIC
ip address
# 使用 tun0 发送 ping
ping -I tun0 192.168.0.2
# 使用tcp client 从本机向tun0管辖的网段请求建立tcp连接
nc 192.168.0.2 443
# 使用 tshark 查看 tun0 的数据包
tshark -i tun0
```
tun 设备得到的数据（ip 数据）格式如下：
```
Flags [2 bytes]
Proto [2 bytes] # ether type. 0x86dd 是 IPv6, 0x0800 是 IPv4
Raw protocol(IP, IPv6, etc) frame.
```
这里的 Raw protocol frame 就是 IP header + 传输层内容（TCP header + 应用层内容）


tashark 的结果数据包可以根据 TCP Connection State Diagram 看出第一个包是sync
