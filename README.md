# Home_NAT
配置家里网络环境时的一些记录

## 软路由
基础配置
- CPU 2 CPUs x Intel(R) Celeron(R) CPU 3965U @ 2.20GHz
- 内存 8G
- 存储 128GB + 2T(NAS)
- 操作系统 ESXI(v7.0) + RouterOs(v6.44.5) + OpenWrt(OpenWrt R20.7.20 / LuCI Master (git-20.191.36863-eee6bae)) 

## 光猫
如果想要尝试多拨的话，需要在软路由内拨号上网。现在家用的宽带都是光猫拨号，然后连线出来就可以直接用了，需要获取光猫的管理员账户然后改成桥接的方式才能在软路由内拨号，同时你也可以看到网络是否支持 ipv6，至于光猫的管理员账户怎么获取可以找运营商或者万能的闲鱼。

## ROS
淘宝卖家以及网上的分享大多数是 ikuai + LEDE 或者 ROS + LEDE 的双软路由系统，为了未来自己不再乱折腾尽量一部到位。ROS 算是主路由，在我的环境下主要功能就是拨号、多拨、DNS缓存，还有额外可以宽带叠加以及 vpn、ddns 之类的我并没有使用。

[双软路由 ROS 的一些配置](https://www.youtube.com/watch?v=mkJxDSMPlPU&t=46s)
### 一些细节
- 虚拟机配置的网络适配器类型尽量选择 VMXNET 3 (linux 版本选最新)
- 内存 1G 肯定够用
- LAN_01234 用 bridge 桥接起来，方便后续用软路由的其他网口
- 好像是支持在线更新的，但是怕授权的问题暂时不更新

### 多拨脚本
多拨我目前使用的网络大概率只能拨两次，有时候支持3拨。
```bash
/interface vrrp
add name=vrrp1 arp=enabled authentication=none disabled=no interface=WAN_5 interval=1 mtu=1500 preemption-mode=yes priority=100 vrid=1
add name=vrrp2 arp=enabled authentication=none disabled=no interface=WAN_5 interval=1 mtu=1500 preemption-mode=yes priority=100 vrid=2
add name=vrrp3 arp=enabled authentication=none disabled=no interface=WAN_5 interval=1 mtu=1500 preemption-mode=yes priority=100 vrid=3

/ip address
add address=1.1.1.10/24 disabled=no interface=vrrp1
add address=1.1.1.11/24 disabled=no interface=vrrp2
add address=1.1.1.12/24 disabled=no interface=vrrp3

/interface pppoe-client #填上你的宽带账号ID和密码
add name="pppoe-out1" interface="vrrp1" user="xxxx" password="123456" disabled=no
add name="pppoe-out2" interface="vrrp2" user="xxxx" password="123456" disabled=no
add name="pppoe-out3" interface="vrrp3" user="xxxx" password="123456" disabled=no

/ip firewall mangle
add action=change-mss chain=forward comment=change-mss disabled=no new-mss=1440 protocol=tcp tcp-flags=syn

/ip firewall mangle
add chain=prerouting action=mark-connection dst-address-type=!local in-interface=LAN_0 per-connection-classifier=both-addresses:3/0 new-connection-mark=PCC_1 passthrough=yes comment="PCC1"
add action=mark-routing chain=prerouting connection-mark=PCC_1 disabled=no in-interface=LAN_0 new-routing-mark=PCC_ROUT1 passthrough=yes

add chain=prerouting action=mark-connection dst-address-type=!local in-interface=LAN_0 per-connection-classifier=both-addresses:3/1 new-connection-mark=PCC_2 passthrough=yes comment="PCC2"
add action=mark-routing chain=prerouting connection-mark=PCC_2 disabled=no in-interface=LAN_0 new-routing-mark=PCC_ROUT2 passthrough=yes

add chain=prerouting action=mark-connection dst-address-type=!local in-interface=LAN_0 per-connection-classifier=both-addresses:3/2 new-connection-mark=PCC_3 passthrough=yes comment="PCC3"
add action=mark-routing chain=prerouting connection-mark=PCC_3 disabled=no in-interface=LAN_0 new-routing-mark=PCC_ROUT3 passthrough=yes


/ip firewall mangle
add action=mark-connection chain=input disabled=no in-interface=pppoe-out1 new-connection-mark=PCC_1 passthrough=yes comment="INOUT1"
add action=mark-routing chain=output connection-mark=PCC_1 disabled=no new-routing-mark=PCC_ROUT1 passthrough=yes

add action=mark-connection chain=input disabled=no in-interface=pppoe-out2 new-connection-mark=PCC_2 passthrough=yes comment="INOUT2"
add action=mark-routing chain=output connection-mark=PCC_2 disabled=no new-routing-mark=PCC_ROUT2 passthrough=yes

add action=mark-connection chain=input disabled=no in-interface=pppoe-out3 new-connection-mark=PCC_3 passthrough=yes comment="INOUT3"
add action=mark-routing chain=output connection-mark=PCC_3 disabled=no new-routing-mark=PCC_ROUT3 passthrough=yes


/ip firewall nat
add action=masquerade chain=srcnat comment=1 disabled=no out-interface=pppoe-out1
add action=masquerade chain=srcnat comment=2 disabled=no out-interface=pppoe-out2
add action=masquerade chain=srcnat comment=3 disabled=no out-interface=pppoe-out3

/ip route
add comment=1 dst-address=0.0.0.0/0 gateway=pppoe-out1 routing-mark=PCC_ROUT1 check-gateway=ping disabled=no distance=1
add comment=2 dst-address=0.0.0.0/0 gateway=pppoe-out2 routing-mark=PCC_ROUT2 check-gateway=ping disabled=no distance=1
add comment=3 dst-address=0.0.0.0/0 gateway=pppoe-out3 routing-mark=PCC_ROUT3 check-gateway=ping disabled=no distance=1

add comment=1 dst-address=0.0.0.0/0 gateway=pppoe-out1 check-gateway=ping disabled=no distance=1
add comment=2 dst-address=0.0.0.0/0 gateway=pppoe-out2 check-gateway=ping disabled=no distance=2
add comment=3 dst-address=0.0.0.0/0 gateway=pppoe-out3 check-gateway=ping disabled=no distance=3
```

## OpenWRT
[双软路由 OpenWRT 的一些配置](https://www.youtube.com/watch?v=n0aqV8rbKmE)

这一部分主要负责功能，比如飞机、docker、去广告之类的。爬墙的插件很多，可以自己协调选择合适的，个人使用的就是比较简单的 ssrp，clash 的规则可能和 surge 类似，但是会使用较多的 cpu 资源，速度也比较慢弃之。

可能比较有意思值得折腾的是 docker，目前在纠结是否需要额外的 nas 虚拟机，因为 OpenWRT 直接支持挂载硬盘和 smb 协议，还有一堆下载插件 aria2，qbittorrent，transmission 之类，还有 docker 可以安装其他有意思的虚拟机。如果额外加一个 nas 虚拟机，在 nas 里面也是一样的操作，安装下载插件，安装 docker 才让 nas 更灵活，反而多了一个系统资源消耗，网络传输，感觉没有必要。

所以目前个人选择可能是放弃 nas 虚拟机，等软路由内的这些东西玩多了，有概率买一个白群晖来替代。
