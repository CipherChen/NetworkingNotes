

### 现象

部分线上机器 ssh 到一台跳板的 CentOS 主机时，总是连接不上，但其它客户端是能正常ssh到这台跳板机的。

加上 timeout 之后，发现总是会超时。

```
root@host:~# timeout 5 ssh root@1.2.3.4 -p 2222
root@host:~#
```

### 分析

通过其它地方跳转到CentOS server，抓包分析并查看连接，发现

1. 其实三次握手已经成功建立连接，但连接建立后，两端在反复的发送接收流量，似乎在传输什么数据或者确认什么信息：

```
# 前三个包，说明已经完成了握手
22:23:46.194993 IP 10.16.150.17.53924 > 100.67.76.58.EtherNet/IP-1: Flags [S], seq 4277261782, win 28690, options [mss 1510,nop,nop,TS val 470231038 ecr 0,nop,wscale 10], length 0
22:23:46.195046 IP 100.67.76.58.EtherNet/IP-1 > 10.16.150.17.53924: Flags [S.], seq 3085339803, ack 4277261783, win 28960, options [mss 1460,nop,nop,TS val 186346509 ecr 470231038,nop,wscale 7], length 0
22:23:46.195133 IP 10.16.150.17.53924 > 100.67.76.58.EtherNet/IP-1: Flags [.], ack 1, win 29, options [nop,nop,TS val 470231038 ecr 186346509], length 0
22:23:46.195329 IP 10.16.150.17.53924 > 100.67.76.58.EtherNet/IP-1: Flags [P.], seq 1:50, ack 1, win 29, options [nop,nop,TS val 470231039 ecr 186346509], length 49
22:23:46.195348 IP 100.67.76.58.EtherNet/IP-1 > 10.16.150.17.53924: Flags [.], ack 50, win 227, options [nop,nop,TS val 186346510 ecr 470231039], length 0
22:23:46.207342 IP 100.67.76.58.EtherNet/IP-1 > 10.16.150.17.53924: Flags [P.], seq 1:24, ack 50, win 227, options [nop,nop,TS val 186346522 ecr 470231039], length 23
22:23:46.207416 IP 10.16.150.17.53924 > 100.67.76.58.EtherNet/IP-1: Flags [.], ack 24, win 29, options [nop,nop,TS val 470231051 ecr 186346522], length 0
22:23:46.207770 IP 10.16.150.17.53924 > 100.67.76.58.EtherNet/IP-1: Flags [P.], seq 50:1458, ack 24, win 29, options [nop,nop,TS val 470231051 ecr 186346522], length 1408
22:23:46.208860 IP 100.67.76.58.EtherNet/IP-1 > 10.16.150.17.53924: Flags [P.], seq 24:1664, ack 1458, win 249, options [nop,nop,TS val 186346523 ecr 470231051], length 1640
22:23:46.208945 IP 10.16.150.17.53924 > 100.67.76.58.EtherNet/IP-1: Flags [.], ack 1664, win 32, options [nop,nop,TS val 470231052 ecr 186346523], length 0
22:23:46.210745 IP 10.16.150.17.53924 > 100.67.76.58.EtherNet/IP-1: Flags [P.], seq 1458:1506, ack 1664, win 32, options [nop,nop,TS val 470231054 ecr 186346523], length 48
22:23:46.221765 IP 100.67.76.58.EtherNet/IP-1 > 10.16.150.17.53924: Flags [P.], seq 1664:1944, ack 1506, win 249, options [nop,nop,TS val 186346536 ecr 470231054], length 280
22:23:46.224305 IP 10.16.150.17.53924 > 100.67.76.58.EtherNet/IP-1: Flags [P.], seq 1506:1522, ack 1944, win 35, options [nop,nop,TS val 470231067 ecr 186346536], length 16
22:23:46.264170 IP 100.67.76.58.EtherNet/IP-1 > 10.16.150.17.53924: Flags [.], ack 1522, win 249, options [nop,nop,TS val 186346579 ecr 470231067], length 0
22:23:46.264247 IP 10.16.150.17.53924 > 100.67.76.58.EtherNet/IP-1: Flags [P.], seq 1522:1566, ack 1944, win 35, options [nop,nop,TS val 470231107 ecr 186346579], length 44
22:23:46.264264 IP 100.67.76.58.EtherNet/IP-1 > 10.16.150.17.53924: Flags [.], ack 1566, win 249, options [nop,nop,TS val 186346579 ecr 470231107], length 0
22:23:46.264340 IP 100.67.76.58.EtherNet/IP-1 > 10.16.150.17.53924: Flags [P.], seq 1944:1988, ack 1566, win 249, options [nop,nop,TS val 186346579 ecr 470231107], length 44
22:23:46.264416 IP 10.16.150.17.53924 > 100.67.76.58.EtherNet/IP-1: Flags [P.], seq 1566:1626, ack 1988, win 35, options [nop,nop,TS val 470231108 ecr 186346579], length 60
22:23:46.264894 IP 100.67.76.58.EtherNet/IP-1 > 10.16.150.17.53924: Flags [P.], seq 1988:2032, ack 1626, win 249, options [nop,nop,TS val 186346579 ecr 470231108], length 44
22:23:46.264984 IP 10.16.150.17.53924 > 100.67.76.58.EtherNet/IP-1: Flags [P.], seq 1626:1990, ack 2032, win 35, options [nop,nop,TS val 470231108 ecr 186346579], length 364
22:23:46.271079 IP 100.67.76.58.EtherNet/IP-1 > 10.16.150.17.53924: Flags [P.], seq 2032:2356, ack 1990, win 271, options [nop,nop,TS val 186346586 ecr 470231108], length 324
22:23:46.272353 IP 10.16.150.17.53924 > 100.67.76.58.EtherNet/IP-1: Flags [P.], seq 1990:2626, ack 2356, win 37, options [nop,nop,TS val 470231115 ecr 186346586], length 636
```

2. 这时查看 netstat 也能看到连接是 ESTABLISHED

```
tcp        0     76 100.67.76.58:2222       10.16.150.17:53924      ESTABLISHED 6995/sshd: root [pr  on (0.20/0/0)
```

3. 忽然想起ssh还有verbose参数，遂加上，发现如下日志：

```
root@ap3ar03n03:~# timeout 5 ssh root@100.67.76.58 -p 2222 -v
OpenSSH_7.7p1 Ubuntu-4ubuntu0.1+pitrix1, OpenSSL 1.0.2g  1 Mar 2016
debug1: Reading configuration data /etc/ssh/ssh_config
debug1: /etc/ssh/ssh_config line 19: Applying options for *
debug1: /etc/ssh/ssh_config line 55: Deprecated option "useroaming"
debug1: Connecting to 100.67.76.58 [100.67.76.58] port 2222.
debug1: Connection established.
debug1: permanently_set_uid: 0/0
debug1: identity file /root/.ssh/id_rsa type 0
debug1: key_load_public: No such file or directory
debug1: identity file /root/.ssh/id_rsa-cert type -1
debug1: key_load_public: No such file or directory
debug1: identity file /root/.ssh/id_dsa type -1
debug1: key_load_public: No such file or directory
debug1: identity file /root/.ssh/id_dsa-cert type -1
debug1: key_load_public: No such file or directory
debug1: identity file /root/.ssh/id_ecdsa type -1
debug1: key_load_public: No such file or directory
debug1: identity file /root/.ssh/id_ecdsa-cert type -1
debug1: key_load_public: No such file or directory
debug1: identity file /root/.ssh/id_ed25519 type -1
debug1: key_load_public: No such file or directory
debug1: identity file /root/.ssh/id_ed25519-cert type -1
debug1: key_load_public: No such file or directory
debug1: identity file /root/.ssh/id_xmss type -1
debug1: key_load_public: No such file or directory
debug1: identity file /root/.ssh/id_xmss-cert type -1
debug1: Local version string SSH-2.0-OpenSSH_7.7p1 Ubuntu-4ubuntu0.1+pitrix1
debug1: Remote protocol version 2.0, remote software version OpenSSH_6.6.1
debug1: match: OpenSSH_6.6.1 pat OpenSSH_6.6.1* compat 0x04000000
debug1: Authenticating to 100.67.76.58:2222 as 'root'
debug1: SSH2_MSG_KEXINIT sent
debug1: SSH2_MSG_KEXINIT received
debug1: kex: algorithm: curve25519-sha256@libssh.org
debug1: kex: host key algorithm: ecdsa-sha2-nistp256
debug1: kex: server->client cipher: chacha20-poly1305@openssh.com MAC: <implicit> compression: none
debug1: kex: client->server cipher: chacha20-poly1305@openssh.com MAC: <implicit> compression: none
debug1: expecting SSH2_MSG_KEX_ECDH_REPLY
debug1: Server host key: ecdsa-sha2-nistp256 SHA256:MTFlctqbVPTZGqzDJbMEzgNUtZ3oKUheYmB45T2tuAc
debug1: Host '[100.67.76.58]:2222' is known and matches the ECDSA host key.
debug1: Found key in /root/.ssh/known_hosts:52
debug1: rekey after 134217728 blocks
debug1: SSH2_MSG_NEWKEYS sent
debug1: expecting SSH2_MSG_NEWKEYS
debug1: SSH2_MSG_NEWKEYS received
debug1: rekey after 134217728 blocks
debug1: SSH2_MSG_SERVICE_ACCEPT received
debug1: Authentications that can continue: publickey,gssapi-keyex,gssapi-with-mic
debug1: Next authentication method: gssapi-keyex
debug1: No valid Key exchange context
debug1: Next authentication method: gssapi-with-mic
debug1: Authentications that can continue: publickey,gssapi-keyex,gssapi-with-mic
debug1: Authentications that can continue: publickey,gssapi-keyex,gssapi-with-mic
debug1: Authentications that can continue: publickey,gssapi-keyex,gssapi-with-mic
debug1: Authentications that can continue: publickey,gssapi-keyex,gssapi-with-mic
debug1: Authentications that can continue: publickey,gssapi-keyex,gssapi-with-mic
debug1: Authentications that can continue: publickey,gssapi-keyex,gssapi-with-mic
debug1: Authentications that can continue: publickey,gssapi-keyex,gssapi-with-mic
debug1: Authentications that can continue: publickey,gssapi-keyex,gssapi-with-mic
debug1: Authentications that can continue: publickey,gssapi-keyex,gssapi-with-mic
...
# 而后一直这样死循环
```

加`-vvv`能看到，这里在反复重试gssapi-with-mic。于是搜索该关键字，发现不少人也碰到了这个问题，但碰到的没这么严重。

一般都是报登录CentOS时，登录会比较慢，或者有明显的等待，而不会像我碰到的这样，直接就连不上，最后timeout了。

### 解决

要解决这个问题，有两种解决思路，分别是客户端和服务端：

1. 客户端

    a. ssh 时，命令行可以加上 `-o GSSAPIAuthentication=no` 参数；

    b. 或者可以配置在客户端的 `/etc/ssh/ssh_config` 里。

2. 服务端

修改 `/etc/ssh/sshd_config`，把 `GSSAPIAuthentication` 关掉，并重启 `sshd` 服务。

```
# 如果是yes，要改成no。或者直接把这两个配置注释掉即可。
# GSSAPIAuthentication no
# GSSAPICleanupCredentials no
```

### 说明

GSSAPI：Generic Security Services Application Program Interface，GSSAPI 本身是一套 API，由 IETF 标准化。
其最主要也是著名的实现是基于 Kerberos 的。一般说到 GSSAPI 都暗指 Kerberos 实现。
GSSAPI 是一套通用网络安全系统接口。该接口是对各种不同的客户端服务器安全机制的封装，以消除安全接口的不同，降低编程难度。

### 参考

* [诊断并解决 SSH 连接慢的方法](https://linux.cn/article-5851-1.html)
* [SSH 中的 GSSAPI 相关选项](https://jaminzhang.github.io/linux/GSSAPI-related-options-in-ssh-configuration/)
