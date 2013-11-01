### 动态路由

我选择了bird+osfp作为本项目的动态路由平台，关于bird信息请参考《bird on openwrt》[]

#### 配置

以`lab`和`corp`为例：

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

`protocol direct`的意思是将直连路由塞入ospf中参与协商，换句话说，通告给相连路由器，我这里有`192.168.44.0/24`和`192.168.77.0/24`这两个子网，请将目标地址属于这两个子网的数据包丢过来给我处理吧。

可以通过birdc4来查看bird平台状态

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

> **NOTE** 手工设置的静态路由优先级最高，动态路由协议所学习到的路由优先级低，故OS优先选择静态路由。

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

l

