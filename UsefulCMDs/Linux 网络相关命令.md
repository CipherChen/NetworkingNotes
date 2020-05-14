
《TCP/IP 详解 卷一》

http://linux.vbird.org/linux_server/0140networkcommand.php

## 链路层检测

* `arping`

* `ip neighbor`

## 网络层检测

* `ping`

* `traceroute`

    http://www.cnblogs.com/peida/archive/2013/03/07/2947326.html

* `mtr`

    mtr 上看到的不丢包，就一定不丢包；如果有丢包，不一定是网络，有可能是禁 ping 了，这种情况 mtr 输出也不会 100% loss.

* `ifconfig`

* `route`

* `ip`

* `netstat`
* `ss`

* `tcpdump`

## DNS

* `host`
* `nslookup`
* `dig`

## 应用层监测

* `telnet`

* `dstst`

* `wget`

    `wget -e 'http_proxy=http://127.0.0.1:80' http://www.example.com`

* `curl`

* `links`

* `nc`/`netcat`
    * `nc -uvc 127.0.0.1 10000` (udp 端口监听测试)

* `lsof`
    * `lsof -nP | grep deleted`

## 其它

* `ipcalc`

* `nethogs`
* `iftop`
* `iptraf`
* `bmon`
* `slurm`
* `tcptrack`
* `ngrep`

查看哪些 iptables rules 命中了
* `watch -n 2 -d 'iptables -vS | grep 1079fb71'`

* `ip netns`
    * `ip netns ls`
    * `ip netns add`
    * `ip netns exec $NS ip route`
    * `ip netns exec $NS bash`

--------------------------------------------------------------------------------
