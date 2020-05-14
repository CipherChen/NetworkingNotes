# ARP 浅析

## ARP(Address Resolution Protocol)

网络层（IP 地址）将数据包发送出去时，需要确定链路层地址（MAC），ARP 正是提供这种转换的协议。

## ARP 请求寻址

当主机 A（IP 地址为 IP—A）的主机需要和主机 B（IP 地址为 IP-B）通信时，A 会先查询本机的路由表，决定下一跳发往哪里。

如果 IP-B 和 IP-A 在同一个子网，A 会查本机的 ARP 缓存，得到 IP-B 对应的 mac 地址，并将数据包的目标 mac 改为 IP-B 对应的 mac 地址，从对应网卡发出。如果 A 的 ARP 缓存里没有 IP-B 对应的 mac 地址，A 会先发一个 ARP **广播包(broadcast)**，B 会发一个 **单播包(unicast)** 响应这个 ARP 包；A 收到这个单播包后，会更新本地的 ARP 缓存，并将数据包发出。

如果 IP-B 和 IP-A 不在同一个子网，A 会将数据包发到路由表里指向的下一跳地址。同样的，发往下一跳地址的过程中，也需要通过 ARP 缓存或 ARP 广播来确定下一跳的 mac 地址。

1. ARP 只在同一个二层网中使用。
2. ARP 寻址过程中，发出的请求是广播包，收到的请求是单播包。
3. 广播包会发送给除本机外的所有其它主机。所以如果 arping 自己的 IP，没有人会回包。

## 常用命令

* 主机上抓 ARP 数据包

```
tcpdump -i eth0 arp -v
```

* 查看本机的 arp 缓存

```
ip neighbour

arp -an
```

* （VIP 漂移时）（主动）通知更新 ARP caches

```
arping -U # Unsolicited ARP mode. 发送 ARP REQUEST，更新 ARP caches.

arping -A # Unsolicited ARP mode. 发送 ARP REPLY
```

## 常见问题

Q1: 如何判断一个 LAN 中，IP 地址冲突了？
A: 通过 arping。

```
root@hostA:~# arping -I eth0 -c 1 172.31.20.64
ARPING 172.31.20.64 from 172.31.20.10 eth0
Unicast reply from 172.31.20.64 [03:54:12:D0:40:AF]  0.861ms
Unicast reply from 172.31.20.64 [03:54:12:D1:3B:AF]  0.792ms
Sent 1 probes (1 broadcast(s))
Received 2 response(s)
root@hostA:~#
```

如果出现如上结果，即有不同 MAC 的回包，说明 IP 地址冲突了。

## 参考

* [RFC826](https://tools.ietf.org/html/rfc826/)
* [NETWORK BASICS: LOCAL HOST ARP REQUESTS](http://www.dummies.com/programming/networking/cisco/network-basics-local-host-arp-requests//)
