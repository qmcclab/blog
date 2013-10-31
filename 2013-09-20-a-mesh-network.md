!!MODIFIED!!

mesh network看起来很牛，实际上也的确非常牛。这个小项目的初衷源于一个小小的需求：允许外网用户透过公司网络访问用户侧的一台设备。后来我对这张网络的期望越来越大，需求不断的叠加，最后演变成一个mesh network。由于各个互联节点分散在内外网，故需要用到vpn，所以题目就叫a mesh vpn network project。

<!-- more -->

整个系列分为以下几个部分：

# 需求及配置

最初的需求是：允许维护人员的笔记本或手机通过互联网访问位于客户2（User2）的网管服务器，第一时间获知故障，缩短修复时长。

::top-orig-purpose.jpg::

公司办公室（corp）可以连接互联网和内网这两张网络，互联网和内网的防火墙均不在我的控制范围之内，corp节点的终端仅拥有tcp/udp的outbound访问权限。那么唯一可行的方案就是首先在互联网上找一个拥有固定IP地址的节点（lab），然后使用vpn技术，分别在lab节点与corp节点，corp节点与User2节点之间建立两条点对点隧道，最后打通corp节点的路由转发，从而实现最初的需求。

::top-orig-tunnel.jpg::

随后便想到，我1、有时在家（home）需要连回公司处理业务；2、在公司需经常处理客户1（User1）的网络故障；索性把这几个节点全部连起来，于是需求变成了：

## 需求

::tinc-vpn-top.jpg::

### 功能需求

1. 实现lab、User1、User2、corp和home五个节点之间的互联。
2. corp节点下挂的终端可以同时访问内网和互联网，提供condition dns forward，即根据不同的域名，前转至不同的DNS server进行解析；
3. User2节点网关可通过corp节点访问互联网；

### 安全策略

5个节点的安全要求不尽相同，User1和User2的安全性要求最高，lab次之，corp和home最低，需要定义严格的访问策略。

默认策略为拒绝

- “corp”节点
    - 允许节点内主机访问互联网；
    - 允许“User2”节点网关访问本节点网关的DNS服务，并通过本节点访问互联网；
    - 开启“User1”、“User2”、“lab”、“home”之间的路由转发功能；

- “home”节点
    - 允许终端访问“lab”、“lab”、“User1”和“User2”节点的内部网段；

- “lab”节点
    - 允许“home”、“corp”访问SSH、web服务；
    - 终端可以自由的访问“User1”和“User2”的内部网段；

- “User2”节点
    - 允许“lab”、“corp”和“home”特定主机访问节点内网的SSH、web服务；
    - 允许本节点网关访问“corp”节点的DNS服务；
    - 允许本节点网关通过“corp”节点访问互联网，用于软件升级；

- “User1”节点
    - 允许“lab”、“corp”和“home”特定主机访问节点内网的SSH、web服务；

- 所有节点
    - 允许节点网关之间互ping

## 软硬件配置

1. lab、User1和User2均有虚拟化环境，为了方便起见，选择虚拟机（debian squeeze）作为节点网关。
2. 考虑到噪音和能耗，选择桌面路由器（支持OpenWRT固件）担任corp和home节点网关；
3. corp节点处于整张网的中心，vpn的加解密工作颇为频繁，对设备性能要求较高，因而选择了一台MikroTik的RB450G。home仅作为末梢节点接入，因而Linksys WRT54G（64M ram，16M flash）足矣。

# tinc VPN

毫无疑问，需要构建一个VPN network，才能实现五个节点的互联。

第一个想到的是OpenVPN，它成熟、稳定、文档丰富、社区庞大，出问题都能很快找到解决的办法，然而它有一个缺点：组网不够灵活。OpenVPN的组网类型要么是点对点，要么是spoke-and-hub（星状），像本次组网，假如各节点均具备双向访问权限，那么将corp设为中心节点，其它四个节点设为分节点，就可以实现整网的互联互通。问题恰恰就在于corp和User2，其中corp只有outbound的权限，无法成为中心节点；User2只具有inbound权限，无法成为分节点。

当然，如果非要用OpenVPN来实现也不是不可能，但是组网结构比较别扭，可将lab设为server，User1、corp和home作为client。接着，在corp和User2之间再创建一条点对点隧道。这种组网结构的配置会比较繁琐，排错困难。而且一旦lab节点宕机，则互联网用户将无法访问User2，既然采用spoke-and-hub的组网方式那么困难，那有没有一种去中心化的VPN组网呢？

有，那就是mesh VPN，tinc、ZeroRouter、hamachi、cloudVPN和socialVPN是其中的代表。经过比较，本项目选择tinc。实际上tinc的诞生已逾10年，同OpenVPN一样基于OpenSSL，不过一直生存在OpenVPN的阴影之下，未被世人知晓。

以下的介绍来自tinc官网：

> **What is tinc?**

> tinc is a Virtual Private Network (VPN) daemon that uses tunnelling and encryption to create a secure private network between hosts on the Internet. tinc is Free Software and licensed under the GNU General Public License version 2 or later. Because the VPN appears to the IP level network code as a normal network device, there is no need to adapt any existing software. This allows VPN sites to share information with each other over the Internet without exposing any information to others. In addition, tinc has the following features:

> - **Encryption, authentication and compression**

>      All traffic is optionally compressed using zlib or LZO, and OpenSSL is used to encrypt the traffic and protect it from alteration with message authentication codes and sequence numbers.

> - **Automatic full mesh routing**

>     Regardless of how you set up the tinc daemons to connect to each other, VPN traffic is always (if possible) sent directly to the destination, without going through intermediate hops.

> - **Easily expand your VPN**

>     When you want to add nodes to your VPN, all you have to do is add an extra configuration file, there is no need to start new daemons or create and > configure new devices or network interfaces.

> - **Ability to bridge ethernet segments**

>     You can link multiple ethernet segments together to work like a single segment, allowing you to run applications and games that normally only work on a LAN over the Internet.

> - **Runs on many operating systems and supports IPv6**

>     Currently Linux, FreeBSD, OpenBSD, NetBSD, MacOS/X, Solaris, Windows 2000, XP, Vista and Windows 7 and 8 platforms are supported. See our section about supported platforms for more information about the state of the ports. tinc has also full support for IPv6, providing both the possibility of tunneling IPv6 traffic over its tunnels and of creating tunnels over existing IPv6 networks. 

我觉得最厉害的功能是`automatic full mesh routing`和`easily expand`。 

:: top-physical-link ::

上图是物理连接，tinc启动后，会自动创建一个full mesh network，无需人工接入。譬如home和corp节点，在通常情况下是无法直接创建隧道的，原因是：

1. 我缺乏corp互联网防火墙的控制权，不能开端口映射；
2. home用的是移动宽带，无法对其它运营商用户开服务；

然而借助lab这个中间节点，tinc就可以让corp和home之间直接创建加密隧道，后续的业务流量直接在该隧道中跑了。

tinc有路由和交换两种模式。路由模式的效率更高，broadcast被阻断。switch模式更方便，不需要配置网段信息。

本项目一共尝试了3种组合，每种组合均有适合的应用场景：

1. 路由模式+静态路由

* 少量节点，譬如实现公司和家两点的互联；
* 大量节点，但节点内主机不参与通信，譬如多人联网玩局域网有戏；

2. 交换模式+动态路由

* 大量节点，节点内主机参与通信，同时管理员拥有节点内网的IP地址规划和控制核心路由器的权限；

3. 路由模式+NAT

* 大量节点，节点内主机参与通信，但管理员缺乏或仅拥有少部分节点内网IP地址规划和控制核心路由器的权限；

在构建VPN之前，需先普及一下tinc的安装流程。

## tinc安装和调试

在debian和openwrt中安装tinc非常简单

### 安装

**debian**

`aptitude update && aptitude -t squeeze-backports install tinc`

**openwrt**

`opkg update && opkg install tinc`

> **NOTE:** 在debian squeeze-backports和openwrt 12.09中，tinc的版本均为1.0.19。

### 启动

**debian**

先修改启动参数

```
# echo "EXTRA="--debug=3"" > /etc/default/tinc
# echo "<network id>" >> /etc/tinc/nets.boot
```
然后启动
`/etc/init.d/tinc start`

> **说明：**
> 
> 1. debian跟其它发行版不同，喜欢在`/etc/default`中放守护进程的运行参数。默认情况下，tinc的log级别为1，当需要更详细的日志信息时，可以通过EXTRA="--debug=3"现象来提升级别。日志输出到/var/log/syslog中。也可以通过"--logfile"将log重定向到/var/log/tinc.<network id>.log中，但奇怪的是，tinc仍会将一份log写到/var/log/syslog，可能是一个bug。
> 
> 2. debian package还提供了一个`/etc/tinc/net.boots`的控制文件，当用户需要开启某个vpn进程时，需要将该vpn进程的`network id`添加到该文件中。

**openwrt**

openwrt package附送了一个tinc.initd的启动脚本，然而只能识别uci配置，由于tinc涉及到的配置文件较多，因而我决定采用文本配置方式，tinc.initd就不适用了，所以还得另行创建一个启动脚本。

```
# mv /etc/init.d/tinc /etc/init.d/tinc.orig
# echo <<'EOF' > /etc/init.d/tinc

#!/bin/sh /etc/rc.common

START=46
STOP=46

start() {
        tincd -n <network id> -d3
}

stop() {
        killall tincd
}
EOF
```

> **说明：** 全网节点的`network id`建议保持一致。

`root@corp:~ # /etc/init.d/tinc enable`

这就可以实现开机自启动了。

手工启动的命令是`root@corp:~ # /etc/init.d/tinc start`


### 排错

- 查看tun0网口

    `# ip addr show tun0`

- 查看connection

    `# netstat -atun | grep 655`

- 查看log

    `# tail -f /var/log/syslog | grep tinc`

- 检查对端655端口

    `# telnet <wan ip> 655`

- ping+tcpdump

    `# tcpdump -nvi tun0 icmp`

万事俱备，只欠东风，下面开始a mesh vpn network之旅。

## 路由+静态路由

以lab和User1为例，这两个节点均有静态互联网IP，网关均为debian squeeze虚拟机。

**lab**

1. 创建一个network

    `# mkdir -p /etc/tinc/mgmt/hosts`

    tinc可以同时开启多个vpn进程，每个进程的相关配置存放到一个被称为`network id`的文件夹中，本文的`network id`取名为mgmt。

2. 生成hosts配置

        # echo <<'EOF' > /etc/tinc/mgmt/hosts/lab
        Address = 221.182.254.165

        ## vpn subnet
        Subnet = 10.8.0.254/32
    
        ## lab's real subnet
        Subnet = 192.168.33.0/24
        Subnet = 192.168.66.0/24
        EOF

3. 生成证书

    `# tincd -n mgmt -K4096`

    tinc将会提示公私钥的存放位置，默认情况下，私钥以独立文件方式存放在`/etc/tinc/<network>`目录，公钥会附在hosts文件末尾。

        # cat /etc/tinc/mgmt/hosts/lab

        Address = 221.xx.xx.165

        ## vpn subnet
        Subnet = 10.8.0.254/32

        ## lab's real subnet
        Subnet = 192.168.33.0/24
        Subnet = 192.168.66.0/24

        -----BEGIN RSA PUBLIC KEY-----
        ...
        -----END RSA PUBLIC KEY-----# cat /etc/tinc/mgmt/hosts/lab

4. 生成tinc.conf

        # echo <<'EOF' > /etc/tinc/mgmt/tinc.conf
        Name = lab
        AddressFamily = ipv4
        ConnectTo = User1
        Interface = tun0
        EOF

5. 创建tinc-up脚本

        # echo <<'EOF' > /etc/tinc/mgmt/tinc-up
        #!/bin/sh

        User1_VIP="10.8.0.253"
        lab_VIP="10.8.0.254"

        ip link set $INTERFACE up
        ip address add dev $INTERFACE ${lab_VIP}/24

        ip route add $User1_VIP via $lab_VIP

        ## to User1's real subnet
        ip route add 10.168.1.0/24 via $User1_VIP
        ip route add 10.168.9.0/24 via $User1_VIP
        ip route add 10.1.0.0/24 via $User1_VIP
        ip route add 10.2.0.0/24 via $User1_VIP
        ip route add 10.3.0.0/16 via $User1_VIP
        ip route add 192.168.200.0/24 via $User1_VIP
        EOF

> **说明：**
> 
> 1. OS需添加相关静态路由，tinc才能转发数据包；
> 2. trick：先添加一条点对点路由，然后再将对端网段指向对端网关IP地址；

**User1**

1. 创建hosts文件

        # echo <<'EOF' > /etc/tinc/mgmt/hosts/User1
        Address = 221.xx.xx.44
        
        # VPN subnet
        Subnet = 10.8.0.253/32

        # User1's real subnet
        Subnet = 10.168.1.0/24
        Subnet = 10.168.9.0/24
        Subnet = 10.1.0.0/24
        Subnet = 10.2.0.0/24
        Subnet = 10.3.0.0/16
        Subnet = 192.168.200.0/24
        EOF

2. 生成证书

    `# tincd -n mgmt`

3. 创建tinc-up脚本

```bash
root@User1:~ # echo <<'EOF' > /etc/tinc/mgmt/tinc-up
#!/bin/sh

User1_VIP="10.8.0.253"
lab_VIP="10.8.0.254"

ip link set $INTERFACE up
ip address add dev $INTERFACE ${User1_VIP}/24

ip route add $lab_VIP via $User1_VIP

# to lab's real subnet
ip route add 192.168.33.0/24 via $lab_VIP
ip route add 192.168.55.0/24 via $lab_VIP
ip route add 192.168.66.0/24 via $lab_VIP
ip route add 192.168.88.0/24 via $lab_VIP
EOF
```

在启动前，需要把hosts文件分别拷贝到对方的hosts目录下：

`root@User1:~ # scp /etc/tinc/mgmt/hosts/User1 lab_host_ip:/etc/tinc/mgmt/hosts`

`root@lab:~ # scp /etc/tinc/mgmt/hosts/lab User1_host_ip:/etc/tinc/mgmt/hosts`

最后，分别启动tinc

`root@lab:~ # /etc/init.d/tinc start`

`root@User1:~ # /etc/init.d/tinc start`

这样就在lab和User1之间创建了一条加密隧道，非常简单。

路由+静态路由模式的一个缺陷就是，随着节点的增多，需要手工配置的路由越来越多，维护工作量也越来越大。下面是`corp`节点的`tinc-up`

```bash
root@corp:~ # cat /etc/tinc/mgmt/tinc-up
#!/usr/sh
home_VIP="10.8.0.250"
User1_VIP="10.8.0.253"
User2_VIP="10.8.0.251"
corp_VIP="10.8.0.252"
lab_VIP="10.8.0.254"

ip link set $INTERFACE up
ip address add dev $INTERFACE ${corp_VIP}/24

ip route add $User1_VIP via $corp_VIP
ip route add $lab_VIP via $corp_VIP
ip route add $User2_VIP via $corp_VIP
ip route add $home_VIP via $corp_VIP


# to User1's real subnet
ip route add 10.168.1.0/24 via $User1_VIP
ip route add 10.168.9.0/24 via $User1_VIP
ip route add 10.1.0.0/24 via $User1_VIP
ip route add 10.2.0.0/24 via $User1_VIP
ip route add 10.3.0.0/16 via $User1_VIP
ip route add 192.168.200.0/24 via $User1_VIP

# to lab's real subnet
ip route add 192.168.33.0/24 via $lab_VIP
ip route add 192.168.55.0/24 via $lab_VIP
ip route add 192.168.66.0/24 via $lab_VIP
ip route add 192.168.88.0/24 via $lab_VIP

# to User2's real subnet
ip route add 10.10.115.0/24 via $User2_VIP
ip route add 10.10.80.0/24 via $User2_VIP
ip route add 10.10.84.0/24 via $User2_VIP
ip route add 10.10.88.0/24 via $User2_VIP

# to home's real subnet
ip route add 192.168.22.0/24 via $home_VIP
```

这还不算完，各节点的核心路由器也要添加相应的静态路由[^note1]，它们的内网方能互相通信。这就不能忍了，之所以抛弃OpenVPN而选择tinc，不就是看到它自我标榜的`easily expand`吗？怎么办呢？上动态路由吧。

## 交换+动态路由

嗯，这又是一个庞大的话题。我对动态路由的认识远在2005年，考完CCNP后就丢掉了。因而，本章就不误人子弟了，只有简单的说明，技术细节请自行参考cisco出的通俗读物。

现在，我们需要在所有节点网关上跑动态路由协议，它们之间互相协商，根据链路的通断动态调整路由指向，最后形成一张（某个路由器）临时路由表。有了这个功能，我们就不需要再手工维护路由表了，just leave them alone。

### tinc配置

引入动态路由后，tinc-up的配置得到极大的简化：

**lab**

```bash
root@lab:~ # cat /etc/tinc/mgmt/tinc-up
#!/bin/sh
ip link set $INTERFACE up
ip address add dev $INTERFACE 10.8.0.254
```

**corp**

```bash
root@corp:~ # cat /etc/tinc/mgmt/tinc-up
#!/bin/sh
ip link set $INTERFACE up
ip address add dev $INTERFACE $10.8.0.252
```

然而，引入动态路由后，`tinc.conf`还需要手工添加`Subnet=本节点内网`，各节点之间才能正常的通信。解决办法是将tinc的组网模式需要从router变更为switch（在`tinc.conf`中添加`Mode = switch`），switch的性能比route要差一些，因为还要处理broadcast和arp请求等数据包，但是可以不用再费心去维护各节点的子网信息了。以一点性能损失来换取维护的便利还是值得的。

全部节点的`tinc.conf`和`tinc-up`都得到简化，顿时感觉清爽了不少，真正做到leave them alone！

### 动态路由

动态路由包含路由平台和动态路由协议。路由平台运行在linux服务器上，而动态路由协议则运行在路由平台上，可以将运行着路由平台的服务器看作一台路由器。

#### 选择

每次选择开源软件都很真痛苦，因为不知道选中的软件能活多久。譬如我之前为n2n歌功颂德，没几天，作者就说没时间维护，请关注fork，这不是坑爹吗？幸好开源的路由平台也没有太多的选择：quagga or bird。10年前曾试过zebra，稳定性实在太差，所以对quagga也不太感冒，加上bird已经在欧洲多个Internet exchanger部署，所以就选了bird。bird目前仅支持传统的RIP、BGP、OSPF，尚不支持babel、olsr等mesh network协议，因而路由协议就选了ospf，没有选择也是一种幸福。

#### 安装

**debian**

`aptitude update && aptitude -t squeeze-backports install bird`

**openwrt**

`opkg update && opkg install bird4 birdc4`


#### 配置

这里以`lab`和`corp`为例：

**lab**

```bash
root@root@lab:~ # cat /etc/bird.conf
log syslog { debug, trace, info, remote, warning, error, auth, fatal, bug };
router id 10.8.0.254;
debug protocols all;

protocol kernel {
        persist;                # Don't remove routes on bird shutdown
        scan time 20;           # Scan kernel routing table every 20 seconds
        export all;             # Default is export none
}

protocol device {
        scan time 10;           # Scan interfaces every 10 seconds
}

protocol direct {
        interface "eth0";
}

protocol static {
       debug all;
}

protocol ospf {
        import all;
        export all;
        area 0.0.0.0 {
                interface "tun0" {
                        hello 9;
                        retransmit 6;
                        cost 10;
                        transmit delay 5;
                        dead count 60;
                        wait 10;
                        type pointopoint;
                };
        };
}
```

**corp**

bird的配置与lab几乎一致，区别是

```
route id 10.8.0.252;
protocol direct {
        interface "br-lan", "wlan0";
}
```

protocol direct的意思是将直连路由塞入ospf中参与协商，换句话说，通告给相连路由器，我这里有192.168.44.0/24和192.168.77.0/24这两个子网，请将目标地址属于这两个子网的数据包丢过来给我处理吧。

#### 常用指令

**启动**

debian：

`# /etc/init.d/bird start`

openwrt：

`# /etc/init.d/bird4 start`

**自启动**

debian：

`# /etc/init.d/bird enable`

openwrt：

`# /etc/init.d/bird4 enable`

**查看状态**

可以通过birdc/birdc4来查看bird平台状态

```
root@corp:~# birdc4 
BIRD 1.3.7 ready.
bird> show route
0.0.0.0/0          via 10.8.0.251 on tun0 [ospf1 Jul29] ! E2 (150/10/10000) [10.8.0.251]
10.2.0.0/24        via 10.8.0.253 on tun0 [ospf1 Jul28] * E2 (150/10/10000) [10.8.0.253]
10.3.0.0/16        via 10.8.0.253 on tun0 [ospf1 Jul28] * E2 (150/10/10000) [10.8.0.253]
172.168.0.0/24     via 10.8.0.251 on tun0 [ospf1 Jul29] * E2 (150/10/10000) [10.8.0.251]
10.1.0.0/24        via 10.8.0.253 on tun0 [ospf1 Jul28] * E2 (150/10/10000) [10.8.0.253]
10.10.10.0/24      via 10.8.0.253 on tun0 [ospf1 Jul28] * E2 (150/10/10000) [10.8.0.253]
10.8.0.0/24        dev tun0 [ospf1 Jul28] * I (150/10) [10.8.0.252]
192.168.77.0/24    dev wlan0 [direct1 Jul28] * (240)
192.168.33.0/24    via 10.8.0.254 on tun0 [ospf1 Jul29] * E2 (150/10/10000) [10.8.0.254]
192.168.44.0/24    dev br-lan [direct1 Jul28] * (240)
10.10.115.0/25     via 10.8.0.251 on tun0 [ospf1 Jul29] * E2 (150/10/10000) [10.8.0.251]
10.10.80.0/24      via 10.8.0.251 on tun0 [ospf1 Jul29] ! E2 (150/10/10000) [10.8.0.251]
10.10.84.0/24      via 10.8.0.251 on tun0 [ospf1 Jul29] ! E2 (150/10/10000) [10.8.0.251]
10.10.88.0/24      via 10.8.0.251 on tun0 [ospf1 Jul29] ! E2 (150/10/10000) [10.8.0.251]
10.168.9.0/24      via 10.8.0.253 on tun0 [ospf1 Jul28] * E2 (150/10/10000) [10.8.0.253]
10.168.1.0/24      via 10.8.0.253 on tun0 [ospf1 Jul28] * E2 (150/10/10000) [10.8.0.253]
bird>exit
```

再用`ip route show`查看，就会发现bird已经将路由写入到kernel路由表中了，每条路由后面带有`proto bird`的后缀。

> **NOTE** 手工设置的路由优先级最高，bird无法覆盖。

**log**

`tail -f /var/log/syslog | grep bird`

动态路由的引入极大地拓展tinc的应用场景，譬如在全国拥有大量分支机构的企业，就可以利用“交换+动态路由”模式在互联网上构建一张灵活的full mesh network。当然，tinc+动态路由并非万能，需满足以下条件：

1. 各节点的IP地址段统一分配；
2. 各节点内网的核心路由器/交换机也需开启动态路由协议。

否则会出现：

1. 节点的IP地址段容易冲突；
2. 各节点核心路由器/交换机的路由表需手工维护，工作量大[^footnote2]；

在本项目中，corp、User1、User2的网络规划和网络设备都不归我负责，故动态路由注定只是匆匆过客。虽然它的出现给tinc带来了无限的可能，但此时我只能无情地say no，重新寻找新的解决方案。不久，NAT在不经意中出现了，欲知后事，详见下一章节“路由+NAT”。

## 路由+NAT

上一章节中提到希望通过NAT的方式来“解决IP地址冲突”和“缓解节点核心路由器路由表维护工作”这两个问题，其思路是：

将一个C类（10.8.0.0/24）切割成多个子网段，每个分节点分配一个子网段，子网段的第一个可用IP地址作为该节点的互联IP地址，剩余的以1:1 NAT方式映射至节点内主机。

假设vpn地址分配&映射表如下：

地点     | 网段
-------- | ----------
lab      | 10.8.0.0/27
User1    | 10.8.0.32/27
corp     | 10.8.0.64/27
User2    | 10.8.0.96/27
home     | 10.8.0.128/27

下面以lab:monitor与User2:storage之间的通信为例，monitor和storage这两个主机的IP地址处于冲突状态，原本无法相互通信。

lab:monitor -> User2:storage

数据包IP层为src ip: 192.168.33.78/dst ip: 10.8.0.97，当数据包经过lab网关时，src ip被替换成10.8.0.2，当数据包到达User2网关时，dst ip被替换成10.10.115.3，数据包IP层变成了src ip: 10.8.0.2/dst ip: 192.168.33.78，最后到达storage；

User2:storage -> lab:monitor

数据包IP层为src ip: 10.8.0.2/dst ip: 192.168.33.78，当数据包经过User2网关时，src ip被替换成10.8.0.97，当数据包达到lab网关时，dst ip被替换成192.168.33.68，数据包IP层变成了src ip: 10.8.0.97/dst ip: 192.168.33.78，最后到达monitor；

使用1:1 NAT技术后，

1. 在vpn中，源和目标地址均为10.8.0.0/24，充分利用了tinc的`auto routing`功能特性，实现tinc vpn的路由不需要配置路由；
2. 在节点内，核心路由器只需要配置一条静态路由，将10.8.0.0/24指向本节点网关的内网IP即可。
也就是说，利用1:1 NAT技术可以实现：

1. 规避节点内网路由；
2. 规避各节点内网段的冲突问题；
3. 减轻节点内的核心路由器路由表的维护工作；

1:1 NAT是节点网关的第一个功能需求，第二个功能需求则是实现以下的安全访问策略：

**“corp”节点**

- 允许节点内主机访问互联网；
- 允许“User2”节点网关访问本节点网关的DNS服务，并通过本节点访问互联网；
- 开启“User1”、“User2”、“lab”、“home”之间的路由转发功能；

**“home”节点**

- 允许终端访问“lab”、“lab”、“User1”和“User2”节点的内部网段；

**“lab”节点**

- 允许“home”、“corp”访问节点网关SSH、web服务；
- 允许“home”、“corp”访问节点内网的所有服务；
- 终端可以自由的访问“User1”和“User2”的内部网段；

**“User2”节点**

- 允许“lab”、“corp”和“home”特定主机访问节点内网的SSH、web服务；
- 允许本节点网关访问“corp”节点的DNS服务；
- 允许本节点网关通过“corp”节点访问互联网，用于软件升级；

**“User1”节点**

* 允许“lab”、“corp”和“home”特定主机访问节点内网的SSH、web服务；

**所有节点**

* 允许节点网关之间互ping；

毫无疑问，需要在各节点启用iptables/netfilter，但自从用了pf后，我对iptables的语法深恶痛绝，故在debian安装了shorewall，OpenWRT则用了的shorewall-lite。

在玩弄防火墙之前，还是先调整tinc的配置。

### tinc配置

**lab**

tinc需要修改两个地方：

1. /etc/tinc/mgmt/hosts/lab

        Address = 221.xx.xx.165

        # VPN subnet
        Subnet = 10.8.0.0/27

        -----BEGIN RSA PUBLIC KEY-----
        ...
        -----END RSA PUBLIC KEY-----

2. /etc/tinc/mgmt/tinc.conf

        ...
        # Mode = switch

将Mode由switch恢复为route，tinc会自动将目标地址为10.8.0.0/27网段丢到10.8.0.1。

**corp**

tinc需要修改两个地方：

1. /etc/tinc/mgmt/hosts/lab

        Address = 221.xx.xx.165

        # VPN subnet
        Subnet = 10.8.0.0/27

        -----BEGIN RSA PUBLIC KEY-----
        ...
        -----END RSA PUBLIC KEY-----

2. /etc/tinc/mgmt/tinc.conf

        ...
        # Mode = switch

### 防火墙

下面分别以lab和corp为例说明debian shorewall和OpenWRT shorewall-lite的安装和配置。

shorewall主要是要理解zones，policy和rules，具体的就不献丑了，官网文档很详尽。另外，为了更好的排错，我用ulogd替代了syslog，默认情况下，netfilter利用syslog来记录，一是效率低，二是跟`/var/log/syslog`混在一起，改用ulogd后效率更高，而且可以将日志存放在单独的文件中。

#### shorewall on debian

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

关于移动宽带

由于运营商的不对称管制，导致移动宽带用户的互联网访问感知很差，原因是大部分的IDC资源均在电信侧。为了缓解这种矛盾，移动公司将部分互联网流量通过第三方通道流向chinanet，使用感知得到了提升，然而却给终端用户带来一些麻烦，特别是无法正常使用动态域名。原因是：

1. 由于通过不同链路出去，故拥有多个互联网IP地址；
2. 第三方提供的互联网ip只允许outbound，不允许inbound，甚至禁ping。

好在tinc既可以做server，也可以做client。当作为client的时候，本机的hosts文件中的Address即便不是真实的，也无关紧要。因而移动宽带用户即便没有使用动态域名解析，也能正常使用tinc client。


[^footnote2]: **脚注2**

倘若希望lab和User1之间内部网段的互访，各自核心交换机需将对方网段指向本节点的tinc网关：

**lab huawei S5328**

```
ip route 192.168.44.0 255.255.255.0 <lab lan ip> desc to corp
ip route 192.168.77.0 255.255.255.0 <lab lan ip> desc to corp

ip route 10.1.0.0 255.255.255.0 <User1 lan ip> desc to User1
ip route 10.2.0.0 255.255.255.0 <User1 lan ip> desc to User1
ip route 10.3.0.0 255.255.0.0 <User1 lan ip> desc to User1
ip route 10.168.1.0 255.255.255.0 <User1 lan ip> desc to User1
ip route 10.168.9.0 255.255.255.0 <User1 lan ip> desc to User1
ip route 192.168.200.0 255.255.255.0 <User1 lan ip> desc to User1

ip route 10.8.0.0 255.255.255.0 <lab lan ip> desc to local tinc gw
```

**User1 H3C S5500**

```
ip route 192.168.44.0 255.255.255.0 <User1 lan ip> desc to corp
ip route 192.168.77.0 255.255.255.0 <User1 lan ip> desc to corp

ip route 192.168.33.0 255.255.255.0 <User1 lan ip> desc to lab
ip route 192.168.55.0 255.255.255.0 <User1 lan ip> desc to lab
ip route 192.168.66.0 255.255.255.0 <User1 lan ip> desc to lab
ip route 192.168.88.0 255.255.255.0 <User1 lan ip> desc to lab
ip route 10.8.0.0 255.255.255.0 <User1 lan ip> desc to local tinc gw
```
