

## HTTP2.0

NPN（Next Protocol Negotiation，下一代协议协商），是一个 TLS 扩展，由 Google 在开发 SPDY 协议时提出。随着 SPDY 被 HTTP/2 取代，NPN 也被修订为 ALPN（Application Layer Protocol Negotiation，应用层协议协商）。二者目标一致，但实现细节不一样，相互不兼容。以下是它们主要差别：
* NPN 是服务端发送所支持的 HTTP 协议列表，由客户端选择；而 ALPN 是客户端发送所支持的 HTTP 协议列表，由服务端选择；
* NPN 的协商结果是在 Change Cipher Spec 之后加密发送给服务端；而 ALPN 的协商结果是通过 Server Hello 明文发给客户端；

### HTTP2.0 vs. HTTP1.x

1. 数据以二进制(binary)而非文本(text)的形式传输
    数据更紧凑、更容易解析；而不像HTTP1那样，通过换行、空格等特殊的分隔符来解析。
1. 可以多路复用(multiplexing)，而不是排序(ordered)和阻塞(blocked)
    浏览器在HTTP1.1下，对每个域名的请求并发数有限制（Firefox/Chrome是6），传统的解决办法是，对于不同的资源（CSS/HTML/JS等），使用不同的
    域名（Domain Sharding），来绕开这个限制。
1. 可以只用一个连接实现并行，节省了多连接的连接建立时间
1. 支持头部压缩(head compression)来降低载荷
    gzip 由于 [CRIME](https://en.wikipedia.org/wiki/CRIME) 攻击，所以HTTP2里不用 gzip，而是用 HPACK。
1. 支持服务端推送(server pushing)
    允许服务端在客户端在只请求HTML文件时，还同时主动返回CSS文件(即优先级更高的，对页面渲染影响更大的资源文件)

### 解决了什么问题？

HTTP2 适用于要并发加载多个资源时的场景，因为多路复用，可以降低连接的延迟；
Server pushing 可以实现 client 在请求 HTML 时，server 主动同时返回 CSS 文件。

HTTP2 适用于 WEB（如浏览器等）需要加载多个、多种资源后再做渲染的场景。

### 测试什么场景才能比较出HTTP2 VS. HTTP1 的优势？

1. 从 `h2load` 的结果来看，要测试单连接
2. 从 HTTP2 的原理来看，要测试WEB下的多文件（以及多种文件，如css/image等）
3. 对于HTTP API场景，不要用 HTTP2

### 结论

HTTP2 不是银弹，不要简单的认为打开了 HTTP2，网站速度就一定会得到优化。
从 [HTTP/2 Theory and Practice in NGINX Stable, Part 2] 和我本人使用 `h2load` 的测试结果来看，有几个结论：
1. 在测试 nginx 静态文件的场景下，http2 并没有加快网页的访问，甚至比 http1 还慢一些（都是 SSL）
2. SSL HTTP2 由于有 SSL，多引入了一层负载，所以性能相对也会更低一些
3. HTTP2 的头部压缩可以有效降低 header 的请求量（`space savings 39.58%`），HTTP1 这部分是 0%。

    ```
    finished in 632.93ms, 6319.82 req/s, 4.50MB/s
    requests: 4000 total, 4000 started, 4000 done, 4000 succeeded, 0 failed, 0 errored, 0 timeout
    status codes: 4000 2xx, 0 3xx, 0 4xx, 0 5xx
    traffic: 2.85MB (2984050) total, 453.13KB (464001) headers (space savings 39.58%), 2.33MB (2448000) data

    finished in 5.45s, 7344.45 req/s, 5.23MB/s
    requests: 40000 total, 40000 started, 40000 done, 40000 succeeded, 0 failed, 0 errored, 0 timeout
    status codes: 40000 2xx, 0 3xx, 0 4xx, 0 5xx
    traffic: 28.46MB (29840050) total, 4.43MB (4640001) headers (space savings 39.58%), 23.35MB (24480000) data
    ```

### Q&A

Q3: `h2load` 测试并发，报错 `Process Request Failure`

是 nginx 上对于 http2 的并发限制 `http2_max_requests`，表示 http2 的单个连接会处理多少请求，默认是 1000。
实际超过该阈值时，现有连接会被 nginx 主动中断，客户端需要新建连接。
周期性关闭连接可以确保内存及时回收，因此该配置并不是越大越好。

* [H2LOAD PROCESS REQUEST FAILURE](https://firestuff.org/2019-04-28-h2load-process-request-failure.html)
* [http2_max_requests](https://nginx.org/en/docs/http/ngx_http_v2_module.html#http2_max_requests)

Q2: 对于普通的高并发场景（单 client，`Connection: close` or `Connection: keep-alive`），是否有显著优化？

Q1: 复用连接的话，是否意味着多个请求可以复用源端口，从而减少连接数？

### 参考

#### HTTP2 Demo

* [HTTP/2 vs HTTP/1](https://www.thewebmaster.com/hosting/2015/dec/14/what-is-http2-and-how-does-it-compare-to-http1-1/)
* [HTTP/2 vs HTTP/1 demo](https://http2.akamai.com/demo)
* [Performance difference between HTTP2 and HTTP1.1](https://imagekit.io/demo/http2-vs-http1)
* [HTTP/2 TECHNOLOGY DEMO](http://www.http2demo.io/)

#### 性能测试

* [7 Tips for Faster HTTP/2 Performance](https://www.nginx.com/blog/7-tips-for-faster-http2-performance/)
* [HTTP/2 Theory and Practice in NGINX Stable, Part 2](https://www.nginx.com/blog/http2-theory-and-practice-in-nginx-stable-part-2/)

#### 其它

* [谈谈 Nginx 的 HTTP/2 POST Bug](https://imququ.com/post/nginx-http2-post-bug.html)
* [HTTP/2 资料汇总](https://imququ.com/post/http2-resource.html)

* [How To Set Up Nginx with HTTP/2 Support on Ubuntu 18.04](https://www.digitalocean.com/community/tutorials/how-to-set-up-nginx-with-http-2-support-on-ubuntu-18-04)
* [Module ngx_http_v2_module](http://nginx.org/en/docs/http/ngx_http_v2_module.html)

* [HTTP2 not speed up](https://discourse.haproxy.org/t/http2-not-speed-up/5219)
