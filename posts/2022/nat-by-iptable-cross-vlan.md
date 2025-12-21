---
date: 2022-04-04
title: iptables进行NAT端口映射的踩坑记录
category: 运维
tags:
- iptables
- NAT
- 运维
description: 使用iptables进行NAT端口映射和局限
---
# 使用iptables进行NAT端口映射和局限
## 0x01 背景

最近工作中遇到需要使用centos进行NAT端口映射，那么首相想到的肯定是用iptables来实现，网上看来好多文章，写的都不是很完整，为了以后方便处理决定记一下。

项目网络拓扑如下，要对公网暴露的多个服务在两个vLAN中（即20.0.0.1和30.0.0.1中），做了到20.0.0.1网段与30.0.0.1网段内网打通。

```mermaid
graph
    internet-->firewall[firewall<br/> global:xx.xx.xx.xx<br/> dmz:10.0.0.x]
    firewall --NAT--> B
    subgraph private network
        subgraph vLAN1
            B[serviceA<br/> dmz:10.0.0.x<br/> private:20.0.0.x]
            D[others]
        end

        subgraph vLAN2
            C[serviceB<br/> private:30.0.0.1]
            E[others]
        end    
    end
```

## 0x02

针对多网卡的设备，需进行静态路由的配置，让数据流流向指定的出口网关，这一块就不赘述了。

需要开启开启linux的转发功能。

```shell
echo 1 >/proc/sys/net/ipv4/ip_forward
```

iptables在默认配置中禁用了转发，需要允许。
打开`/etc/sysconfig/iptables` 注释一下内容
```shell
-A FORWARD -j REJECT --reject-with icmp-host-prohibited
```

执行下属命令进行端口映射
```shell
iptables -t nat -A PREROUTING -p tcp -m tcp --dport 30010 -j DNAT --to-destination 30.0.0.1:30010
iptables -t nat -A PREROUTING -p udp -m udp --dport 30010 -j DNAT --to-destination 30.0.0.1:30010

```

执行完后进行测试发现，目标服务器能收到udp包，udp回包客户端收不到，tcp超时，这个现象明显就是回程出现了问题。
既然是回程的问题，那首相想到的是把20.0.0.1这台机器配置成网关。

但是这一步存在一个问题，Linux中网关设备只能是广播可达的设备，在和网络组经过一番~~激烈撕逼~~友好交流后修改了网络方案如下图,在NAT服务器设备上添加了一张30.0.0.1网段的网卡。

```mermaid
graph
    internet-->firewall[firewall<br/> global:xx.xx.xx.xx<br/> dmz:10.0.0.x]
    firewall --NAT--> B
    subgraph private network
        subgraph vLAN1
            B[serviceA<br/> dmz:10.0.0.x<br/> private:20.0.0.x]
            D[others]
        end

        subgraph vLAN2
            B--forward-->C
            C[serviceB<br/> private:30.0.0.1]
            C--route-->B
            E[others]
        end
    end
```

在配置网卡后配置静态路由把目标服务的网关修改为NAT服务器后，tcp和udp双向数据就都通了。
