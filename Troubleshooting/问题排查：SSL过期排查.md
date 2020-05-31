
## 现象

客户反馈请求一个域名，总是会报 `SSL certificate problem: certificate has expired`。

```
root@pek3b_migration_transit:~# date; curl -v https://business.xxxxxxxx.com/eagle/appver
Sat May 30 23:08:56 CST 2020
* Hostname was NOT found in DNS cache
*   Trying 139.198.xx.yyy...
* Connected to business.xxxxxxxx.com (139.198.xx.yyy) port 443 (#0)
* successfully set certificate verify locations:
*   CAfile: none
  CApath: /etc/ssl/certs
* SSLv3, TLS handshake, Client hello (1):
* SSLv3, TLS handshake, Server hello (2):
* SSLv3, TLS handshake, CERT (11):
* SSLv3, TLS alert, Server hello (2):
* SSL certificate problem: certificate has expired
* Closing connection 0
curl: (60) SSL certificate problem: certificate has expired
More details here: http://curl.haxx.se/docs/sslcerts.html

curl performs SSL certificate verification by default, using a "bundle"
 of Certificate Authority (CA) public keys (CA certs). If the default
 bundle file isn't adequate, you can specify an alternate file
 using the --cacert option.
If this HTTPS server uses a certificate signed by a CA represented in
 the bundle, the certificate verification probably failed due to a
 problem with the certificate (it might be expired, or the name might
 not match the domain name in the URL).
If you'd like to turn off curl's verification of the certificate, use
 the -k (or --insecure) option.
root@pek3b_migration_transit:~#
```

加上 `-k` 后，看到的现象更奇怪，证书里的时间明明是 `2020-08-15 23:59:59 GMT`，并没有过期，但还是显示了
`SSL certificate verify result: certificate has expired (10), continuing anyway`。

```
root@pek3b_migration_transit:~# curl -k -v https://business.xxxxxxxx.com/eagle/appver
* Hostname was NOT found in DNS cache
*   Trying 139.198.xx.yyy...
* Connected to business.xxxxxxxx.com (139.198.xx.yyy) port 443 (#0)
* successfully set certificate verify locations:
*   CAfile: none
  CApath: /etc/ssl/certs
* SSLv3, TLS handshake, Client hello (1):
* SSLv3, TLS handshake, Server hello (2):
* SSLv3, TLS handshake, CERT (11):
* SSLv3, TLS handshake, Server key exchange (12):
* SSLv3, TLS handshake, Server finished (14):
* SSLv3, TLS handshake, Client key exchange (16):
* SSLv3, TLS change cipher, Client hello (1):
* SSLv3, TLS handshake, Finished (20):
* SSLv3, TLS change cipher, Client hello (1):
* SSLv3, TLS handshake, Finished (20):
* SSL connection using ECDHE-RSA-AES256-GCM-SHA384
* Server certificate:
* 	 subject: OU=Domain Control Validated; OU=PositiveSSL Wildcard; CN=*.xxxxxxxx.com
* 	 start date: 2018-08-16 00:00:00 GMT
* 	 expire date: 2020-08-15 23:59:59 GMT
* 	 issuer: C=GB; ST=Greater Manchester; L=Salford; O=COMODO CA Limited; CN=COMODO RSA Domain Validation Secure Server CA
* 	 SSL certificate verify result: certificate has expired (10), continuing anyway.
> GET /eagle/appver HTTP/1.1
> User-Agent: curl/7.35.0
> Host: business.haomoney.com
> Accept: */*
>
< HTTP/1.1 200 OK
* Server Tengine is not blacklisted
< Server: Tengine
< Date: Sat, 30 May 2020 15:06:25 GMT
< Content-Type: application/json;charset=UTF-8
< Transfer-Encoding: chunked
< Connection: close
< Vary: Accept-Encoding
< Access-Control-Allow-Origin: *
< Access-Control-Allow-Methods: POST, GET, OPTIONS, PUT, DELETE
< Access-Control-Max-Age: 0
< Access-Control-Allow-Headers: Origin, X-Requested-With, Content-Type, Accept
<
* Closing connection 0
* SSLv3, TLS alert, Client hello (1):
{"build.tag":"v1.0.7.2020051501","build.module":"eagle","build.revision":"r23-76f3c0a","build.datetime":"2020-05-15 18:57:59","build.hostname":"zof-as-01a1"}root@pek3b_migration_transit:~#
```

## 分析

先检查证书，发现证书里的内容确实显示还未过期。

```
root@pek3br01n05:~/cc# openssl x509 -noout -in 0 -dates
notBefore=Aug 16 00:00:00 2018 GMT
notAfter=Aug 15 23:59:59 2020 GMT
root@pek3br01n05:~/cc#
```

还尝试了抓包，发现确实是证书所在节点收到了包并响应了请求。

那就很奇怪了，为什么仍然显示过期呢？

Google 搜索了一番，有人提到可能是证书并不是一个独立的证书，而是一个证书链。发现客户的这个证书确实是由三个证书组成的，于是把三个证书分别拆开来验证：

```
root@pek3br01n05:~/cc# openssl x509 -noout -in 1 -dates
notBefore=Aug 16 00:00:00 2018 GMT
notAfter=Aug 15 23:59:59 2020 GMT
root@pek3br01n05:~/cc# openssl x509 -noout -in 2 -dates
notBefore=Feb 12 00:00:00 2014 GMT
notAfter=Feb 11 23:59:59 2029 GMT
root@pek3br01n05:~/cc# openssl x509 -noout -in 3 -dates
notBefore=May 30 10:48:38 2000 GMT
notAfter=May 30 10:48:38 2020 GMT
root@pek3br01n05:~/cc#
```

验证到最后一个证书，果然是这个问题，最后一个证书的时间确实已经过期了。

## Q&A

Q4: 如何查看一个域名的证书链？
A4:
```
openssl s_client -showcerts -connect www.baidu.com:443 </dev/null
```

Q3: 如何在查看证书时显示证书链的时间？
TODO

Q2: 如何在发请求时查看证书链的时间？
TODO

Q1: 客户还反馈部分客户端请求没问题，部分客户端请求有问题，这个现象怎么解释？
A1: 猜测可能是openssl版本有关。

## 参考

* [Openssl telling certificate has expired when it has not](https://stackoverflow.com/questions/24992976/openssl-telling-certificate-has-expired-when-it-has-not)
