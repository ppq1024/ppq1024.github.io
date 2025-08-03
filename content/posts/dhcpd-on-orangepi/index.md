---
date: 2025-08-03
name: dhcpd-on-orangepi
title: 在香橙派上配置 dhcpd
draft: false
tags:
  - 计算机网络
  - 香橙派
---

前段时间入手了一块香橙派 AIpro 20T，不知道各位有类似设备（之前也入过一块树莓派 4B）都是怎么连过去的，最常见的应该就是借助家用路由器连接到同一个网络下。不过还有一种稍微小众一点的办法，直接拿一根网线连起来。就是一种非常简单的网络结构，之所以小众大概是因为对大多数设备来说需要额外的配置。

不过香橙派的官方手册推荐的就是这种做法，因为镜像中已经设置好了静态 IP，只需要在 PC 端手动配置静态 IP 就可以了。不过就算是这样我还是觉得麻烦，就打算在香橙派上跑一个 DHCP 服务器，来给我的 PC 自动分配 IP。

## 环境信息

### 硬件设备：

- PC
- 香橙派
- 网线一根
- 公网路由

### 操作环境：

- openEuler 22.03
- OpenSSH（Windows 系统组件，版本未知）

### 网络架构

PC 通过网线连接到香橙派的 eth0，香橙派通过 wlan0 连接到公网。

## dhcpd 配置

首先是安装 dhcpd，其它发行版请自行使用对应的包管理器

{{< notice tip >}}
在 openEuler 中，dhcpd 包含在 dhcp 包中。
{{< /notice >}}

```bash
sudo dnf install dhcp
```

然后是修改配置文件，顺便给我的 PC 配置了静态 IP

```ini
# /etc/dhcp/dhcpd.conf
subnet 192.168.17.0 netmask 255.255.255.0 {
    range 192.168.17.16 192.168.17.191;
    # option routers 192.168.17.1;
    # option domain-name-servers 8.8.8.8, 8.8.4.4;
    option subnet-mask 255.255.255.0;
    default-lease-time 600;
    max-lease-time 7200;
}

host device0 {
    hardware ethernet xx:xx:xx:xx:xx:xx;
    fixed-address 192.168.17.2;
}
```

因为仅访问私网，故没有配置路由和 DNS，当然也可以加上把香橙派当成是软路由用，之前用树莓派这么干过，不过记得额外配置转发规则。


之后启动 dhcpd 服务

```bash
sudo systemctl enable dhcpd
sudo systemctl start dhcpd
```

在 PC 端检查一下 DHCP 是否正常，似乎一切都完成了。

## 还没有结束

本来一切都应该正常进行，但是在我第二天打算连接到香橙派的时候却出现了点问题

```bash
❯ ssh ppq-opi
ssh: connect to host ppq-opi port 22: Connection timed out
```

ping 一下也不通
```bash
❯ ping ppq-opi

正在 Ping ppq-opi [192.168.17.1] 具有 32 字节的数据:
请求超时。
请求超时。
请求超时。
请求超时。

192.168.17.1 的 Ping 统计信息:
    数据包: 已发送 = 4，已接收 = 0，丢失 = 4 (100% 丢失)，
```

*解析到 IP 地址是因为我写到 hosts 文件里了，避免走 WiFi，那边带宽比较低，过程中不涉及 mDNS*

当我检查 PC 端网络配置的时候，却发现没有 IPv4 地址。没办法只能先用 IPv6 连进去看一下了（香橙派那边设置静态 fe80::1 方便调用）。

检查一下 dhcpd 确实没有启动，看了一下 dhcpd日志，没有指定监听设备

{{< notice tip >}}
我只是去复现一下这个问题，所以不要在意日志中的时间。

日志开始日期异常是由于设备和 RTC 同步较晚，因为没有什么危害就没管。
{{< /notice >}}

```bash
[ppq@ppq-opi ~]$ journalctl -u dhcpd -b
-- Journal begins at Thu 1970-01-01 08:00:03 CST, ends at Sun 2025-08-03 12:19:01 CST. --
Aug 03 12:14:52 ppq-opi dhcpd[1400]: Internet Systems Consortium DHCP Server 4.4.2
Aug 03 12:14:52 ppq-opi dhcpd[1400]: Copyright 2004-2020 Internet Systems Consortium.
Aug 03 12:14:52 ppq-opi dhcpd[1400]: All rights reserved.
Aug 03 12:14:52 ppq-opi dhcpd[1400]: For info, please visit https://www.isc.org/software/dhcp/
Aug 03 12:14:52 ppq-opi dhcpd[1400]: ldap_gssapi_principal is not set,GSSAPI Authentication for LDAP will not be used
Aug 03 12:14:52 ppq-opi dhcpd[1400]: Not searching LDAP since ldap-server, ldap-port and ldap-base-dn were not specified in the config file
Aug 03 12:14:52 ppq-opi dhcpd[1400]: Config file: /etc/dhcp/dhcpd.conf
Aug 03 12:14:52 ppq-opi dhcpd[1400]: Database file: /var/lib/dhcpd/dhcpd.leases
Aug 03 12:14:52 ppq-opi dhcpd[1400]: PID file: /var/run/dhcpd.pid
Aug 03 12:14:52 ppq-opi dhcpd[1400]: Source compiled to use binary-leases
Aug 03 12:14:52 ppq-opi dhcpd[1400]: Wrote 0 deleted host decls to leases file.
Aug 03 12:14:52 ppq-opi dhcpd[1400]: Wrote 0 new dynamic host decls to leases file.
Aug 03 12:14:52 ppq-opi dhcpd[1400]: Wrote 1 leases to leases file.
Aug 03 12:14:52 ppq-opi dhcpd[1400]:
Aug 03 12:14:52 ppq-opi dhcpd[1400]: No subnet declaration for eth1 (no IPv4 addresses).
Aug 03 12:14:52 ppq-opi dhcpd[1400]: ** Ignoring requests on eth1.  If this is not what
Aug 03 12:14:52 ppq-opi dhcpd[1400]:    you want, please write a subnet declaration
Aug 03 12:14:52 ppq-opi dhcpd[1400]:    in your dhcpd.conf file for the network segment
Aug 03 12:14:52 ppq-opi dhcpd[1400]:    to which interface eth1 is attached. **
Aug 03 12:14:52 ppq-opi dhcpd[1400]:
Aug 03 12:14:52 ppq-opi dhcpd[1400]:
Aug 03 12:14:52 ppq-opi dhcpd[1400]: No subnet declaration for eth0 (no IPv4 addresses).
Aug 03 12:14:52 ppq-opi dhcpd[1400]: ** Ignoring requests on eth0.  If this is not what
Aug 03 12:14:52 ppq-opi dhcpd[1400]:    you want, please write a subnet declaration
Aug 03 12:14:52 ppq-opi dhcpd[1400]:    in your dhcpd.conf file for the network segment
Aug 03 12:14:52 ppq-opi dhcpd[1400]:    to which interface eth0 is attached. **
Aug 03 12:14:52 ppq-opi dhcpd[1400]:
Aug 03 12:14:52 ppq-opi dhcpd[1400]:
Aug 03 12:14:52 ppq-opi dhcpd[1400]: Not configured to listen on any interfaces!
Aug 03 12:14:52 ppq-opi dhcpd[1400]:
Aug 03 12:14:52 ppq-opi dhcpd[1400]: This version of ISC DHCP is based on the release available
Aug 03 12:14:52 ppq-opi dhcpd[1400]: on ftp.isc.org. Features have been added and other changes
Aug 03 12:14:52 ppq-opi dhcpd[1400]: have been made to the base software release in order to make
Aug 03 12:14:52 ppq-opi dhcpd[1400]: it work better with this distribution.
Aug 03 12:14:52 ppq-opi dhcpd[1400]:
Aug 03 12:14:52 ppq-opi dhcpd[1400]: Please report issues with this software via:
Aug 03 12:14:52 ppq-opi dhcpd[1400]: https://gitee.com/src-openeuler/dhcp/issues
Aug 03 12:14:52 ppq-opi dhcpd[1400]:
Aug 03 12:14:52 ppq-opi dhcpd[1400]: exiting.
```

不过重启一下 dhcpd 倒是正常，但重启系统还是不行。

### 原因分析

dhcpd 配置中通过 subnet 来确定监听的网卡，若 dhcpd 启动时网卡未初始化，没有正确分配 IP，就会找不到要监听的网卡，从而启动失败。解决办法也很简单，等待网卡初始化完成再启动 dhcpd 就可以了

```bash
sudo systemctl edit dhcpd
```

```ini
[Unit]
After=network-online.target sys-subsystem-net-devices-eth0.device
Wants=network-online.target sys-subsystem-net-devices-eth0.device
```

需要确保 eth0 的静态 IP 已生效，wlan0 生效是不行的。之后就一切正常了。
