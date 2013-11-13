---
layout: post
title: "OpenWRT网络配置"
date: 2013-04-26 16:00
comments: true
categories: [openwrt, ap]
---

# 软硬件环境

- 硬件：Linksys WRT54G修改版，8M FLASH/64M RAM
- OS：OpenWRT ATTITUDE ADJUSTMENT (12.09-rc1, r34185)

# 需求

- WAN口处于闲置状态，未连接任何网络
- LAN IP: 192.168.44.1/24，wifi IP: 192.168.77.1/24
- wifi与LAN这两个网段之间可互相访问
- 在wifi口启用DHCP服务
- wifi处于AP mode，SSID为OpenWRT，启用WPS-PSK2认证

# 配置

```
# cat /etc/config/network

config switch 'eth0'
	option enable '1'

config switch_vlan 'eth0_0'
	option device 'eth0'
	option vlan '0'
	option ports '0 1 2 3 5'

config switch_vlan 'eth0_1'
	option device 'eth0'
	option vlan '1'
	option ports '4 5'

config interface 'loopback'
	option ifname 'lo'
	option proto 'static'
	option ipaddr '127.0.0.1'
	option netmask '255.0.0.0'

config interface 'lan'
	option type 'bridge'
	option ifname 'eth0.0'
	option proto 'static'
	option netmask '255.255.255.0'
	option ipaddr '192.168.44.1'

config interface 'wan'
	option ifname 'eth0.1'
	option proto 'dhcp'
	
config interface 'wifi'
	option _orig_ifname 'radio0.network1'
	option _orig_bridge 'false'
	option proto 'static'
	option ipaddr '192.168.77.1'
	option netmask '255.255.255.0'

```

> **NOTE** 在openwrt官网中，wifi section缺乏`_orig_ifname`和`_orig_bridge`这两个参数，无法正常工作。

```
# cat /etc/config/wireless 

config wifi-device 'radio0'
	option type 'mac80211'
	option channel '11'
	option macaddr '00:13:10:e5:bc:53'
	option hwmode '11g'
	option txpower '20'
	option country '00'

config wifi-iface
	option device 'radio0'
	option network 'wifi'
	option mode 'ap'
	option encryption 'psk2'
	option key 'secret'
	option ssid 'OpenWRT'
```


```
# cat /etc/config/dhcp

config dnsmasq
	option domainneeded '1'
	option boguspriv '1'
	option localise_queries '1'
	option rebind_protection '1'
	option rebind_localhost '1'
	option local '/lan/'
	option domain 'lan'
	option expandhosts '1'
	option authoritative '1'
	option readethers '1'
	option leasefile '/tmp/dhcp.leases'
	option resolvfile '/tmp/resolv.conf.auto'
	list server '192.168.44.254'

config dhcp 'lan'
	option interface 'lan'
	option ignore '1'

config dhcp 'wan'
	option interface 'wan'
	option ignore '1'

config dhcp 'wifi'
	option interface 'wifi'
	option start     '100'
	option limit     '20'
	option leasetime '12h'
```

配置完成后重启`wifi`和`dnsmasq`

```
# ifup wifi
# /etc/init.d/dnsmasq restart
```
