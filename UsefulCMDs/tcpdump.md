# tcpdump

## 概览：tcpdump 参数和表达式

tcpdump 是一款强大的基于命令行的网络流量包分析工具。

如果对于TCP/IP协议没有基本的了解，tcpdump的输出看起来就像是梵文一般深奥难懂。

初次接触tcpdump，首先先介绍一些基本的最为常用的参数，让你试图了解tcpdump的输出。

tcpdump的基本格式：

```
tcpdump options expression
```

### options

`-n` 表示不将域名转换成名字。`-nn` 表示不将域名/端口等转换为名字，如 80 -> http 等。

TCP协议中包含sequence numbers，tcpdump默认将输出相对的（相对于发出此包的进程的包数）。
加上 `-S` 后，则会输出绝对的sequence numbers.

tcpdump默认只会打印出每个包的部分信息：源地址、源端口、目标地址、目标端口、seq、length等。
若想进一步探究整个数据包，可以加上 `-X` 或 `-XX` . 这样tcpdump会输出包含ASCII和hex的详细包内容。

当然，默认情况下tcpdump只会打印出65535字节的包内容。若想得到更多包信息，加上 `-s snaplen bytes` 参数
可以选择打印指定字节数的内容。当使用 `-s 0` 时，会打出整个包的内容。

还有一个常用的参数是 `-c count`：在捕捉到count个包后结束tcpdump程序。
我个人习惯于在抓包时，强制加上 `-c`，避免由于表达式不够准确，抓到过多的包，导致刷屏且很难终止。

```
-n/-nn
-X/-XX
-S
-s
-c

-w/-r
```

## 示例

那我们就以捕捉ping包作为例子学习tcpdump的使用方法。我们都知道ping包是基于ICMP协议的。

实验环境：多张网卡，p5p1连接到外网。

* 捕捉任意网卡上的icmp协议包：

    ```
    tcpdump -i any icmp
    ```

* 捕捉一个完整的ping交互：

    ```
    tcpdump -i p5p1 icmp -c 2 -S
    ```

* 查看真实的地址IP：

    ```
    tcpdump -i p5p1 icmp -c 2 -S -nn
    ```

* 查看完整的ICMP包：

    ```
    tcpdump -i p5p1 icmp -S -nn -s 0 -XX
    ```

* 查看所有80端口（包括源端口和目标端口）的请求：

    ```
    tcpdump -i p5p1 port 80
    ```

* 查看所有请求

    ```
    # 我个人习惯性在抓包时指定 -c，避免由于包过多而刷屏。
    tcpdump -i p5p1 -c 100
    ```

### expression

用来选择捕获哪些流量包。

如果不加expression，所有的网络流量包都会被捕获。

否则，只捕获那些使expression为真的流量包。

表达式包含三种可能的qualifier：

* type
    * host
    * net
    * port
    * portrange
* dir
    * src
    * dst
    * src or dst
    * src and dst
* proto
    * ip/ip6
    * arp/rarp
    * tcp
    * udp
    * ether
    * fddi
    * tr
    * decnet

此外，还有一些关键字不符合这些表达式的，比如gateway、broadcast等。

而这些primitives可以使用一些combinator来连接：

* `and`
* `or`
* `not`
* `greater`
* `less`

或者使用符号：
* `&&`
* `||`
* `>/>=`
* `</<=`
* `=/!=`
* `!`

对于复杂的表达式分组，pcap还提供了一系列默认的优先级组合。
比如 `not/!` 具有最高的优先级，`and/&&`、`or/||` 具有同等优先级且是从左到右进行组合。

比如如果标识符没有任何关键字，那么使用最近的关键字。E.g:

```
not host vs and ace
```

表示

```
not host vs and host ace
```

而不是

```
not (host vs or ace)。
```

看到这里是不是有点头大而想起了大学挂掉的C语言操作符优先级？

不用担心，但凡遇到可能使逻辑变得复杂的跟优先级相关的操作符，就使用括号来分组。

上例中 `not host vs and host ace中not` 到底是跟 `host vs` 还是跟 `host ace` 组合的，还是同时组合的呢？

使用 `(not host vs) and (not host ace)` 或者 `not (host vs and host ace)` 来表达出你想要的效果来吧。

### Flags

* `F` : FIN - 结束; 结束会话
* `S` : SYN - 同步; 表示开始会话请求
* `R` : RST - 复位;中断一个连接
* `P` : PUSH - 推送; 数据包立即发送
* `A` : ACK - 应答
* `U` : URG - 紧急
* `E` : ECE - 显式拥塞提醒回应
* `W` : CWR - 拥塞窗口减少

## WireShark

* 过滤HTTP关键字

```
http contains "GET /uidm/sso/login"
```

### Wireshark 常见提示及说明

https://community.emc.com/community/support/chinese/teamblog/blog/2016/04/26/大咖讲网络-wireshark的提示

## 参考

* `man 8 tcpdump`
* `man 7 pcap-filter`

# 总结

推荐阅读：如何编写tcpdump的过滤规则。

----

# Q&A

Q1: 如何获取系统所有的网卡？

* `tcpdump -D`
* `ifconfig -s`
* `netstat -i`

----
