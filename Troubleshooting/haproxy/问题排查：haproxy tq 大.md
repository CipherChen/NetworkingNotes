
### 现象 2020-07-06

发现部分日志中，HTTP请求响应的`Tq`时间特别大。以下述例子为例，`Tq`达到了 14394ms。

```
Jul  2 13:35:16 198.19.27.79 haproxy[62013]: 183.210.210.116:15726 [02/Jul/2020:13:35:02.441] lbl-e02bf09x~ lbl-e02bf09x_default/lbb-z59wc7ch 14394/0/1/2/14397 304 134 - - ---- 445/445/1/1/0 0/0 "GET /R?sw=1&aVersion=44 HTTP/1.1"
```

### 原因

  - If "Tq" is close to 3000, a packet has probably been lost between the
    client and the proxy. This is very rare on local networks but might happen
    when clients are on far remote networks and send large requests. It may
    happen that values larger than usual appear here without any network cause.
    Sometimes, during an attack or just after a resource starvation has ended,
    haproxy may accept thousands of connections in a few milliseconds. The time
    spent accepting these connections will inevitably slightly delay processing
    of other connections, and it can happen that request times in the order of
    a few tens of milliseconds are measured after a few thousands of new
    connections have been accepted at once. Using one of the keep-alive modes
    may display larger request times since "Tq" also measures the time spent
    waiting for additional requests.

这里提到一点，在 http-alive 模式下，一个连接可能处理多个请求，所以 `Tq` 可能会很大。于是按连接来搜索日志：

```
# cat 1 | grep 183.210.210.116:15726                                                                                                                                                                 130 ↵
Jul  2 13:35:02 198.19.27.79 haproxy[62013]: 183.210.210.116:15726 [02/Jul/2020:13:35:02.406] lbl-e02bf09x~ lbl-e02bf09x_default/lbb-aatolnup 21/0/1/13/35 304 295 - - ---- 399/399/0/1/0 0/0 "GET /api/chat/unread HTTP/1.1"
Jul  2 13:35:16 198.19.27.79 haproxy[62013]: 183.210.210.116:15726 [02/Jul/2020:13:35:02.441] lbl-e02bf09x~ lbl-e02bf09x_default/lbb-z59wc7ch 14394/0/1/2/14397 304 134 - - ---- 445/445/1/1/0 0/0 "GET /R?sw=1&aVersion=44 HTTP/1.1"
Jul  2 13:35:16 198.19.27.79 haproxy[62013]: 183.210.210.116:15726 [02/Jul/2020:13:35:16.839] lbl-e02bf09x~ lbl-e02bf09x_default/lbb-aatolnup 99/0/1/36/138 200 12498 - - ---- 440/440/1/1/0 0/0 "GET /api/sheet?code=21VbdW1G HTTP/1.1"
Jul  2 13:35:18 198.19.27.79 haproxy[62013]: 183.210.210.116:15726 [02/Jul/2020:13:35:16.977] lbl-e02bf09x~ lbl-e02bf09x_default/lbb-z59wc7ch 1114/0/2/3/1121 200 1466 - - ---- 450/449/0/1/0 0/0 "GET /sw.js HTTP/1.1"
Jul  2 13:35:18 198.19.27.79 haproxy[62013]: 183.210.210.116:15726 [02/Jul/2020:13:35:18.098] lbl-e02bf09x~ lbl-e02bf09x_default/lbb-z59wc7ch 570/0/0/3/573 200 232 - - ---- 455/454/2/2/0 0/0 "POST /api/media/play?code=AD1BQzpk HTTP/1.1"
```

果然，14394ms 的时间，是从 13:35:02 - 13:35:16 花费的 14s。

### 解决

在 `option http-keep-alive` 模式下，`Tq` 特别大不一定是问题。

### 现象 2019-11-27

client -> vpc1 -> vpc2 -> lbc，但 Tq 特别大。

```
[root@i-w63ocrrn 1]# cat 12.log | grep lbl-u3kts8ax | grep iotgateway | egrep "\/[0-9]{4,5} " | head -1
Nov 27 12:02:03 bogon haproxy[47820]: 139.198.0.108:45389 [27/Nov/2019:12:02:02.624] lbl-u3kts8ax~ lbl-u3kts8ax_default/lbb-fhelr0yj 1280/0/0/13/1293 201 272 - - ---- 20/17/17/6/0 0/0 "POST /iotgateway/webhook HTTP/1.1"
[root@i-w63ocrrn 1]# cat 12.log | grep lbl-u3kts8ax | grep iotgateway | egrep "\/[0-9]{4,5} " | wc -l
321
```

### 原因

从拓扑来看，vpc1/vpc2 很容易成为瓶颈。

vpc1/vpc2 是个同一个zone的两个路由器，网络链路很短，所以问题很有可能就在 vpc1/vpc2 上。

在调整 lbc 为公网 lbc 后，绕过了 vpc2，但问题依旧存在，说明瓶颈可能出在 vpc1 上。

后有幸观察到现场监控数据，发现 Tq 大时，vpc1 总是有一些数据同步的流量，虽然流量不大，但持续时间和Tq问题的时间吻合。

vpc1 实际上是个最小规格的路由器，所以在这种情况下，vpc1 就是瓶颈。

### 解决

升级 vpc1；或者用 natgw；或者用 border 连接 vpc1/vpc2。
