

### 现象

公有云客户经常反馈 nacos 经常抛异常，异常多种多样，有连接异常，有超时等。

并且该问题有几个难点：

1. 异常的种类比较多
2. 异常的时间点不确定，没有规律，并且复现概率很低（一天几次）
3. 如果经过负载均衡器，则一直没有问题；而经过负载均衡器后，就有可能出现。（这样还很容易给客户端一种主观上认为“负载均衡器很不稳定”的印象。）

### 分析

#### 2020.04.27 04:48

2020.04.27 04:48 生产和测试LB出现大量超时
原因是该时间点LB的后端所依赖的数据库连不上，数据库连不上的原因是数据库所在的两个可用区的专线出异常，专线出异常的原因是交换机故障。

#### 2020.05.08 07:21

2020.05.08 07:21 测试LB客户端的报错：`java.net.ConnectException: no available server, currentServerAddr: http://172.26.11.252:8848`
原因是测试LB的规格是最小的，节点内存只有512MB，内存跑满后，被`haproxy_agent`发现，自动重启了`haproxy`，重启过程中，影响了客户端。

#### 2020.05.11 06:25

1. 同样的拓扑，生产的LB没问题，测试的LB频繁报
2. 如果不经过LB，而是直连后端，也没有问题
3. 抓包对比生产、测试的LB后端，测试LB后端的RST包更多

3# 的原因，是因为测试LB的健康检查周期更短，是2s，生产的LB是10s，所以测试LB的后端更频繁的收到LB的健康检查探测包。每次探测都包含SYN/SYN+ACK/RST，所以RST更多。

1# 2020.05.11 06:25 生产LB出现了 `NACOSSocketTimeoutException httpPostcurrentServerAddr: http://172.20.10.253:8848, err: Read timed out` 现象

#### 2020.05.11 15:54

2020.05.11 15:54:01 另一个测试LB又出现如下异常：`java.net.ConnectException: no available server, currentServerAddr`

```
@timestamp 2020-05-11 @ 15:54:01.932
  t @version 1
  t _id K6-5AnIBSsGAnGoD-WTP
  t _index logstash_2020.05.11
  # _score -  
  t _type _doc
  t app tss-gateway
  t host 172.26.13.13
  t level ERROR
  # level_value 40,000
  t logger_name com.alibaba.nacos.client.config.impl.ClientWorker
  t message longPolling error :  
  t namespace sh-test
  # port 18,873
  t stack_trace java.net.ConnectException: no available server, currentServerAddr : http://nacostest.localdomain:8848 at com.alibaba.nacos.client.config.http.ServerHttpAgent.httpPost(ServerHttpAgent.java:170) at com.alibaba.nacos.client.config.http.MetricsHttpAgent.httpPost(MetricsHttpAgent.java:64) at com.alibaba.nacos.client.config.impl.ClientWorker.checkUpdateConfigStr(ClientWorker.java:377) at com.alibaba.nacos.client.config.impl.ClientWorker.checkUpdateDataIds(ClientWorker.java:352) at com.alibaba.nacos.client.config.impl.ClientWorker$LongPollingRunnable.run(ClientWorker.java:512) at java.util.concurrent.Executors$RunnableAdapter.call(Executors.java:511) at java.util.concurrent.FutureTask.run(FutureTask.java:266) at java.util.concurrent.ScheduledThreadPoolExecutor$ScheduledFutureTask.access$201(ScheduledThreadPoolExecutor.java:180) at java.util.concurrent.ScheduledThreadPoolExecutor$ScheduledFutureTask.run(ScheduledThreadPoolExecutor.java:293) at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1149) at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:624) at java.lang.Thread.run(Thread.java:748)  
  t thread_name com.alibaba.nacos.client.Worker.longPolling.fixed-nacostest.localdomain_8848-aacafbf3-3481-45da-a8b3-7e54c7e6b8c3
```

但这个问题，目前从负载均衡器的日志来看，又特别的奇怪，LB 的日志显示，对应时间段，所有的请求响应码都是 200，压根就没有失败的请求。

```
May 11 15:54:00 172.26.11.4 haproxy[37590]: 172.26.13.18:16126 [11/May/2020:15:53:31.284] lbl-9clckaj8 lbl-9clckaj8_default/lbb-hlabg1an 2/0/0/29503/29505 200 119 - - ---- 19/19/18/6/0 0/0 "POST /nacos/v1/cs/configs/listener HTTP/1.1"
May 11 15:54:01 172.26.11.3 haproxy[24966]: 172.26.13.7:6045 [11/May/2020:15:53:31.546] lbl-9clckaj8 lbl-9clckaj8_default/lbb-3ezlcsuc 2/0/2/29504/29508 200 119 - - ---- 19/19/18/6/0 0/0 "POST /nacos/v1/cs/configs/listener HTTP/1.1"
May 11 15:54:02 172.26.11.3 haproxy[24966]: 172.26.13.13:47806 [11/May/2020:15:53:31.901] lbl-9clckaj8 lbl-9clckaj8_default/lbb-py6yxsp1 1/0/1/30135/30137 200 119 - - ---- 19/19/18/7/0 0/0 "POST /nacos/v1/cs/configs/listener HTTP/1.1"
May 11 15:54:03 172.26.11.3 haproxy[24966]: 172.26.13.18:52144 [11/May/2020:15:53:34.045] lbl-9clckaj8 lbl-9clckaj8_default/lbb-hlabg1an 2/0/1/29506/29509 200 119 - - ---- 18/18/17/6/0 0/0 "POST /nacos/v1/cs/configs/listener HTTP/1.1"
May 11 15:54:04 172.26.11.3 haproxy[24966]: 172.26.13.12:55391 [11/May/2020:15:53:34.725] lbl-9clckaj8 lbl-9clckaj8_default/lbb-3ezlcsuc 1/0/3/29504/29508 200 119 - - ---- 19/19/18/7/0 0/0 "POST /nacos/v1/cs/configs/listener HTTP/1.1"
May 11 15:54:04 172.26.11.4 haproxy[37590]: 172.26.13.7:10511 [11/May/2020:15:53:35.449] lbl-9clckaj8 lbl-9clckaj8_default/lbb-py6yxsp1 2/0/0/29502/29504 200 119 - - ---- 19/19/18/7/0 0/0 "POST /nacos/v1/cs/configs/listener HTTP/1.1"
```

看来只剩最后一条路走了，只能查 nacos 的源码。

所幸源码里报 `no available server, currentServerAddr` 的地方并不多，在 github 上一搜就搜到了唯一一个地方。

从源码分析来看，`https://github.com/alibaba/nacos/blob/994da9cbebc45d20508bdbc62f5cb57d0126f8d3/client/src/main/java/com/alibaba/nacos/client/config/http/ServerHttpAgent.java`，
```
    @Override
    public HttpResult httpPost(String path, List<String> headers, List<String> paramValues, String encoding,
                               long readTimeoutMs) throws IOException {
        final long endTime = System.currentTimeMillis() + readTimeoutMs;
        boolean isSSL = false;
        injectSecurityInfo(paramValues);
        String currentServerAddr = serverListMgr.getCurrentServerAddr();
        int maxRetry = this.maxRetry;

        do {

            try {
                List<String> newHeaders = getSpasHeaders(paramValues);
                if (headers != null) {
                    newHeaders.addAll(headers);
                }

                HttpResult result = HttpSimpleClient.httpPost(
                    getUrl(currentServerAddr, path), newHeaders, paramValues, encoding,
                    readTimeoutMs, isSSL);
                if (result.code == HttpURLConnection.HTTP_INTERNAL_ERROR
                    || result.code == HttpURLConnection.HTTP_BAD_GATEWAY
                    || result.code == HttpURLConnection.HTTP_UNAVAILABLE) {
                    LOGGER.error("[NACOS ConnectException] currentServerAddr: {}, httpCode: {}",
                        currentServerAddr, result.code);
                } else {
                    // Update the currently available server addr
                    serverListMgr.updateCurrentServerAddr(currentServerAddr);
                    return result;
                }
            } catch (ConnectException ce) {
                LOGGER.error("[NACOS ConnectException httpPost] currentServerAddr: {}, err : {}", currentServerAddr, ce.getMessage());
            } catch (SocketTimeoutException stoe) {
                LOGGER.error("[NACOS SocketTimeoutException httpPost] currentServerAddr: {}， err : {}", currentServerAddr, stoe.getMessage());
            } catch (IOException ioe) {
                LOGGER.error("[NACOS IOException httpPost] currentServerAddr: " + currentServerAddr, ioe);
                throw ioe;
            }

            if (serverListMgr.getIterator().hasNext()) {
                currentServerAddr = serverListMgr.getIterator().next();
            } else {
                maxRetry--;
                if (maxRetry < 0) {
                    throw new ConnectException("[NACOS HTTP-POST] The maximum number of tolerable server reconnection errors has been reached");
                }
                serverListMgr.refreshCurrentServerAddr();
            }

        } while (System.currentTimeMillis() <= endTime);

        LOGGER.error("no available server, currentServerAddr : {}", currentServerAddr);
        throw new ConnectException("no available server, currentServerAddr : " + currentServerAddr);
    }

```
抛这个异常不是因为HTTP的响应码不对，而是完全是因为超时了。

又从 `https://www.jianshu.com/p/dae5ae7a50da` 读到，

> 可以看到一个现象，在配置没有发生变化的情况下，客户端会等29.5s以上，才请求到服务器端的结果。然后客户端拿到服务器端的结果之后，在做后续的操作。
如果在配置变更的情况下，由于客户端基于长轮训的连接保持，所以返回的时间会非常的短，我们可以做个小实验，在nacos console中频繁修改数据然后再观察一下config-client-request.log 的变化

由此猜测，可能是由于请求在客户端超时了。于是回过头去搜索客户端 `172.26.13.13` 在相应时间段的日志。

果然发现了端倪。

```
May 11 15:54:00 172.26.11.4 haproxy[37590]: 172.26.13.18:16126 [11/May/2020:15:53:31.284] lbl-9clckaj8 lbl-9clckaj8_default/lbb-hlabg1an 2/0/0/29503/29505 200 119 - - ---- 19/19/18/6/0 0/0 "POST /nacos/v1/cs/configs/listener HTTP/1.1"
May 11 15:54:01 172.26.11.3 haproxy[24966]: 172.26.13.7:6045 [11/May/2020:15:53:31.546] lbl-9clckaj8 lbl-9clckaj8_default/lbb-3ezlcsuc 2/0/2/29504/29508 200 119 - - ---- 19/19/18/6/0 0/0 "POST /nacos/v1/cs/configs/listener HTTP/1.1"
# 其实这个客户端确实发了请求，而且LB也正常转发了200的响应。但响应时长并不是平常的29.5s，而是30s。
May 11 15:54:02 172.26.11.3 haproxy[24966]: 172.26.13.13:47806 [11/May/2020:15:53:31.901] lbl-9clckaj8 lbl-9clckaj8_default/lbb-py6yxsp1 1/0/1/30135/30137 200 119 - - ---- 19/19/18/7/0 0/0 "POST /nacos/v1/cs/configs/listener HTTP/1.1"
May 11 15:54:03 172.26.11.3 haproxy[24966]: 172.26.13.18:52144 [11/May/2020:15:53:34.045] lbl-9clckaj8 lbl-9clckaj8_default/lbb-hlabg1an 2/0/1/29506/29509 200 119 - - ---- 18/18/17/6/0 0/0 "POST /nacos/v1/cs/configs/listener HTTP/1.1"
May 11 15:54:04 172.26.11.3 haproxy[24966]: 172.26.13.12:55391 [11/May/2020:15:53:34.725] lbl-9clckaj8 lbl-9clckaj8_default/lbb-3ezlcsuc 1/0/3/29504/29508 200 119 - - ---- 19/19/18/7/0 0/0 "POST /nacos/v1/cs/configs/listener HTTP/1.1"
May 11 15:54:04 172.26.11.4 haproxy[37590]: 172.26.13.7:10511 [11/May/2020:15:53:35.449] lbl-9clckaj8 lbl-9clckaj8_default/lbb-py6yxsp1 2/0/0/29502/29504 200 119 - - ---- 19/19/18/7/0 0/0 "POST /nacos/v1/cs/configs/listener HTTP/1.1"
```

其实这个客户端确实发了请求，而且LB也正常转发了200的响应。但响应时长并不是平常的29.5s，而是30s。

所以当前的问题现象就比较清晰了：该抛异常的原因，在于客户端设置的29.5s在大部分情况没有问题，但出于 "配置可能发生变化的情况"，响应的时长稍微大了些，虽然服务端（LB）正常响应了，但客户端已经超时了。

接下来的分析思路就简单了，还需要两个方向的深入排查：

1. 这个请求，在 lbb-py6yxsp1 上，为什么慢了 0.5s？这个后端的日志，是否也能看到这个现象？如果是，说明确实是后端慢导致的。
2. 能否改大这个 29.5s 的超时时间？

深入排查涉及到nacos及用户业务，这里就不深入展开了。

### 参考

* [public HttpResult httpPost](https://github.com/alibaba/nacos/blob/994da9cbebc45d20508bdbc62f5cb57d0126f8d3/client/src/main/java/com/alibaba/nacos/client/config/http/ServerHttpAgent.java)
* [Nacos 配置实时更新原理分析](https://www.jianshu.com/p/acb9b1093a54)

### 结论

这个问题比较经典，因为现象的底层，是由多种原因导致的。每一次出现错误的，都可能是不同的原因。

这些问题交织在一块，给了客户一种“业务经过负载均衡器后很不稳定”的印象。这些印象无形之中也会给排查带来一定的压力，一定要hold住。

并且笔者对 nacos 并不熟悉，可以说是由于该客户反馈，才第一次听说，也给排查带来了一定的难度，最终只能查源码来分析抛异常行为。
