shorewall-lite on openwrt

##### 安装

```
root@lab:~ # echo "" >> /etc/apt/sources.list
root@lab:~ # aptitude update && aptitude install shorewall ulogd
```

##### 配置

/etc/default/shorewall

```
startup=1
...
```

shorewall.conf

```bash
# 修改以下参数：
LOGFILE=/var/log/ulog/syslogemu.log
LOGFORMAT=""
# 将所有info替换成$LOG

...
```

/etc/shorewall/params

```
INT_IF=eth0
VPN_IF=tun0
LOG=ULOG
```

/etc/shorewall/zones

```
fw      firewall
net     ipv4
vpn     ipv4
```

/etc/shorewall/interfaces

```
net     $INT_IF            dhcp,tcpflags,logmartians,nosmurfs,sourceroute=0
vpn     $VPN_IF            dhcp,tcpflags,logmartians,nosmurfs,sourceroute=0
```

/etc/shorewall/policy

```
$FW             net             ACCEPT
$FW             vpn             ACCEPT
vpn             $FW             ACCEPT
net             all             DROP            $LOG
all             all             REJECT          $LOG
```

/etc/shorewall/rules

```
###############
# net2fw

# allow ssh,http,https,655,80,2003 from net to $FW
ACCEPT          net             $FW             tcp             ssh,http,https,2003 
ACCEPT          net             $FW             udp,tcp         655

###############
# vpn2net

# allow ssh,http from USER1 to LAB 
ACCEPT     vpn:10.8.0.32/27     net:192.168.33.0/24,192.168.66.0/24  tcp     ssh,http

# allow all from CORP to LAB
ACCEPT          vpn:10.8.0.64/27        net:192.168.33.0/24     all     
ACCEPT          vpn:10.8.0.64/27        net:192.168.55.0/24     all     
ACCEPT          vpn:10.8.0.64/27        net:192.168.66.0/24     all     
ACCEPT          vpn:10.8.0.64/27        net:192.168.88.0/24     all     
ACCEPT          vpn:10.8.0.64/27        net:192.168.100.0/24    all     
ACCEPT          vpn:10.8.0.64/27        net:172.16.33.0/24      all     

###############
# net2vpn

ACCEPT  net:192.168.33.0/24     vpn     all
ACCEPT  net:192.168.66.0/24     vpn     all
ACCEPT  net:10.5.1.0/24         vpn     all
ACCEPT  net:172.16.33.0/24      vpn     all

###############
# all2all

Ping(ACCEPT)   all             all
```

/etc/shorewall/nat

```
10.8.0.2       tun0            192.168.33.231     No               No
10.8.0.3       tun0            192.168.33.232     No               No
10.8.0.4       tun0            192.168.33.233     No               No
10.8.0.5       tun0            192.168.33.234     No               No
10.8.0.6       tun0            192.168.66.21      No               No
10.8.0.7       tun0            192.168.66.22      No               No
10.8.0.8       tun0            192.168.66.23      No               No
10.8.0.9       tun0            192.168.55.120     No               No
10.8.0.10      tun0            192.168.88.120     No               No
```

```
root@lab:~ # shorewall check
root@lab:~ # shorewall restart
root@lab:~ # /etc/init.d/ulogd start
```

> **NOTE** 别把自己关在外面，希望我的提醒还不至于太晚。

至此，a mesh vpn network完成了。

####　shorewall-lite on OpenWRT

shorewall依赖于perl，对于OpenWRT来说太庞大了。此外，假如有多个防火墙，则需要一套机制进行统一管理，于是诞生了shorewall-lite。

今天，我们就在OpenWRT上体验一下shorewall-lite的魔力。
图片

admin(administrative system)安装了shorewall，firewall的配置文件均在admin中完成，随后通过shorewall compile来生成脚本，接着通过scp将该脚本拷贝至firewall的/etc/shorewall-lite/state目录，然后使用ssh远程执行firewall的shorewall-lite，将脚本转换成iptables rules。

这就是shorewall-lite的运作原理。

所以，首先安装admin的shorewall

root@shorewall-centre-d6:/ # apt-get update && apt-get install shorewall

    * 为每个firewall创建一个export目录

root@shorewall-centre-d6:/ # make -p export/rb450g && cd export/rb450g

    * 准备firewall配置文件

对于debian系，需下载tarball，解压后将/usr/share/shorewall/configfiles中的文件拷贝至export目录。

    * 调整firewall配置文件

拷贝过来的配置文件均是空文件，需要自行调整配置。

/etc/shorewall/export/rb450g/params

```
WAN_IF=eth0
LAN_IF=br-lan
OA_IF=br-oa
VPN_IF=tun0
LOG=ULOG
```

/etc/shorewall/export/rb450g/zones

```
fw      firewall
oa      ipv4
lan     ipv4
wan     ipv4
vpn     ipv4
```

/etc/shorewall/export/rb450g/interfaces

```
wan             $WAN_IF                 dhcp,tcpflags,logmartians,nosmurfs,sourceroute=0
lan             $LAN_IF                 tcpflags,logmartians,nosmurfs,sourceroute=0
vpn             $VPN_IF                 tcpflags,logmartians,nosmurfs,sourceroute=0
oa              $OA_IF                  tcpflags,logmartians,nosmurfs,sourceroute=0
```

/etc/shorewall/export/rb450g/policy

```
$FW     all     ACCEPT
lan     all     ACCEPT
wan     all     DROP            $LOG    10/sec:40
all     all     REJECT
```

/etc/shorewall/export/rb450g/rules

```
SECTION NEW
Invalid(DROP)   wan             all

###############
# vpn2fw

Ping(ACCEPT)    vpn             $FW
SSH(ACCEPT)     vpn             $FW
HTTP(ACCEPT)    vpn             $FW

###############
# wan2fw

ACCEPT          wan             $FW     tcp     655
ACCEPT          wan             $FW     udp     655
SSH(ACCEPT)     wan             $FW
```

/etc/shorewall/export/rb450g/masq

```
$OA_IF          192.168.44.0/24 10.199.27.17
$VPN_IF         192.168.44.0/24 10.8.0.65
$WAN_IF         192.168.44.0/24 192.168.7.21
```

OpenWRT的准备工作

创建`state`

root@RB450G:/ # mkdir /etc/shorewall-lite/state

禁用firewall。

```bash
root@RB450G:/ # /etc/init.d/firewall disable
root@RB450G:/ # /etc/init.d/firewall stop
```

启用shorewall-lite

```bash
root@RB450G:/# /etc/init.d/shorewall-lite enable
```

    * 生成firewall脚本

虽然可用`shorewall compile`来生成firewall脚本，然而Thomas M. Eastep自己写了个Makefile，然后通过`make`和`make install`这两个linux管理员耳熟能详的指令来编译和部署。

root@shorewall-centre-d6:/etc/shorewall/export/rb450g# wget http://www1.shorewall.net/pub/shorewall/contrib/Shorewall-lite/

然后调整Makefile中的HOST，域名和IP地址均可。若用域名，则需要确保可以解析。

```bash
root@shorewall-centre-d6:/etc/shorewall/export/rb450g# make
shorewall compile -e . firewall
Compiling...
Processing /etc/shorewall/export/rb450g/params ...
Processing /etc/shorewall/export/rb450g/shorewall.conf...
   WARNING: Your capabilities file is out of date -- it does not contain all of the capabilities defined by Shorewall version 4.5.5.3
Compiling /etc/shorewall/export/rb450g/zones...
Compiling /etc/shorewall/export/rb450g/interfaces...
Determining Hosts in Zones...
Locating Action Files...
Compiling /usr/share/shorewall/action.Drop for chain Drop...
Compiling /usr/share/shorewall/action.Broadcast for chain Broadcast...
Compiling /usr/share/shorewall/action.Invalid for chain Invalid...
Compiling /usr/share/shorewall/action.NotSyn for chain NotSyn...
Compiling /usr/share/shorewall/action.Reject for chain Reject...
Compiling /etc/shorewall/export/rb450g/policy...
Compiling /etc/shorewall/export/rb450g/notrack...
Running /etc/shorewall/export/rb450g/initdone...
Adding Anti-smurf Rules
Adding rules for DHCP
Compiling TCP Flags filtering...
Compiling Kernel Route Filtering...
Compiling Martian Logging...
Compiling Accept Source Routing...
Compiling /etc/shorewall/export/rb450g/tcrules...
Compiling /etc/shorewall/export/rb450g/masq...
Compiling MAC Filtration -- Phase 1...
Compiling /etc/shorewall/export/rb450g/rules...
Compiling /usr/share/shorewall/action.Invalid for chain %Invalid...
Compiling MAC Filtration -- Phase 2...
Applying Policies...
Generating Rule Matrix...
Creating iptables-restore input...
Shorewall configuration compiled to /etc/shorewall/export/rb450g/firewall
将会在当前目录生成firewall脚本，然后采用make install部署至firewall：
root@shorewall-centre-d6:/etc/shorewall/export/rb450g# make install
scp firewall firewall.conf root@192.168.44.1:/etc/shorewall-lite/state
root@192.168.44.1's password:
firewall                                                                                                                             100%   79KB  79.3KB/s   00:00   
firewall.conf                                                                                                                        100%  862     0.8KB/s   00:00   
ssh root@192.168.44.1 "/sbin/shorewall-lite restart"
root@192.168.44.1's password:Restarting Shorewall Lite....
Initializing...
Processing init user exit ...
Processing tcclear user exit ...
Setting up Route Filtering...
Setting up Martian Logging...
Setting up Accept Source Routing...
Setting up Proxy ARP...
Setting up Traffic Control...
Preparing iptables-restore input...
Running /usr/sbin/iptables-restore...
IPv4 Forwarding EnabledProcessing start user exit ...
Processing started user exit ...
done.
touch: /var/lock/subsys/shorewall: No such file or directory
```

执行`make install`时，admin会将firewall、firewall.conf通过scp拷贝到firewall(rb450g)的/etc/shorewall-lite/state目录下，在firewall(rb450g)中执行/etc/init.d/shorewall-lite stop|start|restart均与该目录下的firewall脚本打交道。

以下是make install执行成功后，/etc/shorewall-lite/state的文件列表

```bash
root@RB450G:/etc/shorewall-lite/state# ls -alh
drwxr-xr-x    1 root     root        2.0K Sep  3 13:49 .
drwxr-xr-x    1 root     root        2.0K Sep  3 11:05 ..
-rw-------    1 root     root           0 Sep  3 13:49 .dynamic
-rw-------    1 root     root        9.8K Sep  3 13:49 .iptables-restor
-rw-------    1 root     root        3.4K Sep  3 13:49 .modules
-rw-------    1 root     root          12 Sep  3 13:49 .modulesdir
-rw-r--r--    1 root     root        1.0K Sep  3 11:09 capabilities
-rwx------    1 root     root       79.3K Sep  3 13:43 firewall
-rw-------    1 root     root         862 Sep  3 13:43 firewall.conf
-rw-------    1 root     root         162 Sep  3 13:49 marks
-rw-------    1 root     root           0 Sep  3 13:49 nat
-rw-------    1 root     root         740 Sep  3 13:49 policies
-rw-------    1 root     root           0 Sep  3 13:49 proxyarp
-rw-------    1 root     root          29 Sep  3 13:49 restarted
-rw-------    1 root     root          74 Sep  3 13:49 state
-rw-------    1 root     root         110 Sep  3 13:49 zones
```

### tricks

**慎用iptables -F**

在清除规则之前，请先确认Chain INPUT的默认poliy，假如是：`Chain INPUT (policy DROP 0 packets, 0 bytes)`，则需要先`iptables -P INPUT ACCEPT`，然后`iptables -F`，否则会把自己锁在系统之外。

**涉及的网卡需起来**

在测试的过程中，发现tinc还没起来，导致shorewall make install失败。
将tinc配置好后，再make install就成功了。

**log**

选用shorewall/shorewall-lite的一个重要原因是shorewall log可根据`源zone+目的zones`分组，使得管理员可以迅速定位出错的规则。举个例子：

从lab中telnet corp的tinc vpn地址失败，于是查看双方的log日志，lab的日志无异样，corp则找到蛛丝马迹：

```bash
# tail -f /var/log/ulogd.syslogemu | grep 10.8.0.65
Sep  3 13:27:14 corp Shorewall:vpn2fw:REJECT: IN=tun0 OUT= MAC= SRC=10.8.0.1 DST=10.8.0.65 LEN=60 TOS=10 PREC=0x00 TTL=64 ID=19084 DF PROTO=TCP SPT=41748 DPT=23 SEQ=1825983957 ACK=0 WINDOW=5840 SYN URGP=0 
```

该log表明，10.8.0.1 telnet 10.8.0.65不满足vpn2fw的policy或rules，于是，我们只要在`/etc/shorewall/export/rb450g/rules`中的vpn2fw区域添加一条`TELNET(ACCEPT)    vpn    $FW`即可。

以下是shorewall的log配置

debian

```bash
# sudo apt-get update
# sudo apt-get install iptables-mod-ulog kmod-ipt-ulog ulogd ulogd-mod-extra
# /etc/init.d/ulogd start
```

OpenWRT

```bash
# opkg update
# opkg install iptables-mod-ulog kmod-ipt-ulog ulogd ulogd-mod-extra
# /etc/init.d/ulogd start
```

排错时再开启ulogd 默认情况下，会将log写到/var/log/ulogd.syslogemu中

shorewall极大简化了iptables的管理，然而与pf相比还是稍逊一筹，主要是shorewall的配置文件太多了。pf只需要一个配置文件，要简单、可爱得多。我认为它是世界上最优雅的防火墙。以lab为例：

/etc/pf.conf

```
wan_if = eth0
vpn_if = tun0
lab_vpn_net = "10.8.0.0/24"
User1_vpn_net = "10.8.0.32/27"
corp_vpn_net = "10.8.0.64/27"

table <lab_net> { "192.168.33.0/24", "192.168.55.0/24"
                  "192.168.66.0/24", "192.168.88.0/24"
                  "192.168.100.0/24", "172.16.33.0/24"
}

###############
# 1:1 nat

pass on tun0 from 192.168.33.231 to any binat-to 10.8.0.2
pass on tun0 from 192.168.33.232 to any binat-to 10.8.0.3
pass on tun0 from 192.168.33.233 to any binat-to 10.8.0.4
pass on tun0 from 192.168.33.234 to any binat-to 10.8.0.5
pass on tun0 from 192.168.66.21 to any binat-to 10.8.0.6
pass on tun0 from 192.168.66.22 to any binat-to 10.8.0.7
pass on tun0 from 192.168.66.23 to any binat-to 10.8.0.8
pass on tun0 from 192.168.55.120 to any binat-to 10.8.0.9
pass on tun0 from 192.168.88.120 to any binat-to 10.8.0.10

###############
# masq

# lab中不需要用到masq，故以下配置被注释掉
# match out on $ext_if from !($vpn_if) to any nat-to ($vpn_if)

block all

###############
# rules

# wan2fw
permit proto { udp, tcp } from any to $wan_if port 655

# vpn2net
permit proto tcp from $User1_vpn_net to <lab_net> port { 22,80,443 }
permit from $corp_vpn_net to <lab_net>

# all2all
permit proto icmp all
```

pf.conf支持table、list，甚至嵌套，大大减少了rules的条数，易于维护。

完成防火墙配置后，还要调整tinc配置。


## openbsd&pfsense

需开放openvpn访问tinc vpn的权限
pass log inet proto { tcp, udp } from <vpn_net> to <tinc_net>

在“nat”的“openvpn”tab中，新建一条目标地址段为tinc_net的允许访问策略

[^footnote1]: **脚注1**


