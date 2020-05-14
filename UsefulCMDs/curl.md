
# curl

* `-x`，指定代理。但只支持 HTTP 代理。

    如果有 HTTPS 代理，可以用环境变量：http_proxy/https_proxy
    用法：`https_proxy=192.168.1.5:8000 curl 'https://www.example.com' -d ''`

* `-w`

```
    root@i-hfnpyny2:~# cat curl-time.txt
              http: %{http_code}\n
               dns: %{time_namelookup}s\n
          redirect: %{time_redirect}s\n
      time_connect: %{time_connect}s\n
   time_appconnect: %{time_appconnect}s\n
  time_pretransfer: %{time_pretransfer}s\n
time_starttransfer: %{time_starttransfer}s\n
     size_download: %{size_download}bytes\n
    speed_download: %{speed_download}B/s\n
                  ----------\n
        time_total: %{time_total}s\n
root@i-hfnpyny2:~# curl -w "@curl-time.txt" 192.168.0.7  -o /dev/null
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   612  100   612    0     0   512k      0 --:--:-- --:--:-- --:--:--  597k
              http: 200
               dns: 0.000s
          redirect: 0.000s
      time_connect: 0.001s
   time_appconnect: 0.000s
  time_pretransfer: 0.001s
time_starttransfer: 0.001s
     size_download: 612bytes
    speed_download: 524871.000B/s
                  ----------
        time_total: 0.001s
root@i-hfnpyny2:~#
```

## curl 字段含义

```
content_type：The Content-Type of the requested document, if there was any.
filename_effective：The ultimate filename that curl writes out to. This is only meaningful if curl is told to write to a file with the --remote-name or --output option. It's most useful in combination with the --remote-header-name option.
ftp_entry_path：The initial path curl ended up in when logging on to the remote FTP server.
http_code：The numerical response code that was found in the last retrieved HTTP(S) or FTP(s) transfer. In 7.18.2 the alias response_code was added to show the same info.
http_connect：The numerical code that was found in the last response (from a proxy) to a curl CONNECT request.
local_ip：The IP address of the local end of the most recently done connection - can be either IPv4 or IPv6
local_port：The local port number of the most recently done connection
num_connects：Number of new connects made in the recent transfer.
num_redirects：Number of redirects that were followed in the request.
redirect_url：When an HTTP request was made without -L to follow redirects, this variable will show the actual URL a redirect would take you to.
remote_ip：The remote IP address of the most recently done connection - can be either IPv4 or IPv6
remote_port：The remote port number of the most recently done connection
size_download：The total amount of bytes that were downloaded.
size_header：The total amount of bytes of the downloaded headers.
size_request：The total amount of bytes that were sent in the HTTP request.
size_upload：The total amount of bytes that were uploaded.
speed_download：The average download speed that curl measured for the complete download. Bytes per second.
speed_upload：The average upload speed that curl measured for the complete upload. Bytes per second.
ssl_verify_result：The result of the SSL peer certificate verification that was requested. 0 means the verification was successful.
time_appconnect：The time, in seconds, it took from the start until the SSL/SSH/etc connect/handshake to the remote host was completed.
time_connect：The time, in seconds, it took from the start until the TCP connect to the remote host (or proxy) was completed.
time_namelookup：The time, in seconds, it took from the start until the name resolving was completed.
time_pretransfer：The time, in seconds, it took from the start until the file transfer was just about to begin. This includes all pre-transfer
commands and negotiations that are specific to the particular protocol(s) involved.
time_redirect：The time, in seconds, it took for all redirection steps include name lookup, connect, pretransfer and transfer before the final transaction was started. time_redirect shows the complete execution time for multiple redirections.
time_starttransfer：The time, in seconds, it took from the start until the first byte was just about to be transferred. This includes time_pre‐transfer and also the time the server needed to calculate the result.
time_total：The total time, in seconds, that the full operation lasted. The time will be displayed with millisecond resolution.
url_effective：The URL that was fetched last. This is most meaningful if you've told curl to follow location: headers.
``
