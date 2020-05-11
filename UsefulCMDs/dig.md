
### dig output explanation

```
$ dig example.com @dns1.p01.nsone.net
; <<>> DiG 9.8.3-P1 <<>> example.com @dns1.p01.nsone.net a
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 60796
;; flags: qr aa rd; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 0
;; WARNING: recursion requested but not available
;; QUESTION SECTION:
;example.com.            IN    A
;; ANSWER SECTION:
# 3600 是 TTL
example.com.        3600    IN    A    104.20.48.182
;; Query time: 8 msec
;; SERVER: 198.51.44.1#53(198.51.44.1)
;; WHEN: Fri Jul  8 10:55:40 2016
;; MSG SIZE  rcvd: 45
```

state:
* NOERROR - Everything's cool. The zone is being served from the requested authority without issues.
* SERVFAIL - The name that was queried exists, but there's no data or invalid data for that name at the requested authority.
* NXDOMAIN - The name in question does not exist, and therefore there is no authoritative DNS data to be served.
* REFUSED - Not only does the zone not exist at the requested authority, but their infrastructure is not in the business of serving things that don't exist at all.

* [DECODING DIG OUTPUT](https://ns1.com/blog/decoding-dig-output)

### dig usage samples

* dig 默认不拼接 `search`，要显式指定 `+search`。

```
dig www.google.com

dig www.google.com +short

提供trace信息
dig www.google.com +trace

指定DNS服务器进行查询
dig @114.114.114.114 www.google.com

查询mx纪录
dig mx www.google.com

查询nameservers
dig www.google.com ns

查询PTR纪录
dig -x 216.58.221.100
```
