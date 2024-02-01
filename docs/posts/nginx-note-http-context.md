---
title: Nginx 学习笔记 —— HTTP Context
authors:
  - tomczhen
date: 2019-10-26T14:00:00+08:00
tags:
  - NGINX
---

需要配置 NGINX 为 Web 服务器或使用反向代理时，大部分配置都需要在 HTTP Context 定义。此外，HTTP Context 还定义 NGINX 如何处理 HTTP/HTTPS 连接。 

<!-- more -->

## 内置变量

> 文档: [NGINX - Core functionality - Embedded Variables](http://nginx.org/en/docs/http/ngx_http_core_module.html#variables)

变量 | 描述
---|---
`$arg_name`                     |
`$args`                         |
`$binary_remote_addr`           |
`$body_bytes_sent`              |
`$bytes_sent`                   |
`$connection`                   |
`$connection_requests`          |
`$content_length`               |
`$content_type`                 |
`$cookie_name`                  |
`$document_root`                |
`$document_uri`                 |
`$host`                         |
`$hostname`                     |
`$http_name`                    |
`$https`                        |
`$is_args`                      |
`$limit_rate`                   |
`$msec`                         |
`$nginx_version`                |
`$pid`                          |
`$pipe`                         |
`$proxy_protocol_addr`          |
`$proxy_protocol_port`          |
`$proxy_protocol_server_addr`   |
`$proxy_protocol_server_port`   |
`$query_string`                 |
`$realpath_root`                |
`$remote_addr`                  |
`$remote_port`                  |
`$remote_user`                  |
`$request`                      |
`$request_body`                 |
`$request_body_file`            |
`$request_completion`           |        
`$request_filename`             |    
`$request_id`                   |
`$request_length`               |
`$request_method`               |
`$request_time`                 |
`$request_uri`                  |
`$scheme`                       |
`$sent_http_name`               |
`$sent_trailer_name`            |
`$server_addr`                  |
`$server_name`                  |
`$server_port`                  |
`$server_protocol`              |
`$status`                       |
`$tcpinfo_rtt`                  |
`$tcpinfo_rttvar`               |
`$tcpinfo_snd_cwnd`             |
`$tcpinfo_rcv_space`            |
`$time_iso8601`                 |
`$time_local`                   |
`$uri`                          |


## HTTP Context

> [NGINX Module - ngx_http_core_module](http://nginx.org/en/docs/http/ngx_http_core_module.html)


HTTP Context 包含下面几个 block：

* server - 定义虚拟服务器。
* upstream - 用于反代、负载均衡、高可用，可以定义多个服务器。

基本结构如下:
```
# main context

event {
    # event context
}

http {
    # http context
    
    upstream first_backend {
        # backend upstream context
    }
    
    upsteam second_backend {
        # backend upstream context
    }

    server {
        # first server context
    }

    server {
        # second server context
    }

}
```
### 基本配置

#### `default_type`

>Syntax: default_type mime-type;  
Default: default_type text/plain;  
Context: http, server, location

> [MIME 类型 - HTTP | MDN](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Basics_of_HTTP/MIME_types)

配置默认响应资源的类型。对于文本资源，一般情况下使用 `text/plain`，二进制文件则使用 `application/octet-stream`。

HTTP 协议中与 MIME 类型相关的有 `Content-Type` 与 `Accept`。`Accept` 表示可以接受的响应资料类型，`Content-Type` 则表示响应资源类型，服务端与客户端可以根据对应的请求头或响应头进行处理。

另外，HTML 标签也会限定响应资源类型，只有当类型匹配时浏览器才会正确渲染，比如 `<video>` 或 `<audio>`。

#### `server`

>Syntax: server { ... }  
Default: —  
Context: http

用于配置虚拟服务器，有基于IP和基于名称两种管理方式。通过 `listen` 指令配置监听地址和端口，通过 `server_name` 指令配置虚拟服务器的名称。

#### `server_name`

>Syntax: server_name name ...;  
Default: server_name "";  
Context: server

为虚拟服务器配置名称。对应请求头 `Host` 中的字段，一般是域名。

```
server {
    # same as:
    # server_name .example.com;
    server_name example.com www.example.com;
}
```
`server_name` 支持通配符与正则，并且可以使用正则参数。

```
# 通配符
server {
    server_name example.com *.example.com www.example.*;
}
```

以 `～` 开头表示使用正则。

```
# 正则
server {
    server_name ~^(www\.)?(.+)$;

    location / {
        root /sites/$2;
    }
}

server {
    server_name ~^(www\.)?(?<domain>.+)$;

    location / {
        root /sites/$domain;
    }
}
```

更多关于 `server_name` 的配置可以查看 [Server names - Nginx](http://nginx.org/en/docs/http/server_names.html)。

#### `listen`

>Syntax: listen address[:port] [default_server] [ssl] [http2 | spdy] [proxy_protocol] [setfib=number] [fastopen=number] [backlog=number] [rcvbuf=size] [sndbuf=size] [accept_filter=filter] [deferred] [bind] [ipv6only=on|off] [reuseport] [so_keepalive=on|off|[keepidle]:[keepintvl]:[keepcnt]];
listen port [default_server] [ssl] [http2 | spdy] [proxy_protocol] [setfib=number] [fastopen=number] [backlog=number] [rcvbuf=size] [sndbuf=size] [accept_filter=filter] [deferred] [bind] [ipv6only=on|off] [reuseport] [so_keepalive=on|off|[keepidle]:[keepintvl]:[keepcnt]];
listen unix:path [default_server] [ssl] [http2 | spdy] [proxy_protocol] [backlog=number] [rcvbuf=size] [sndbuf=size] [accept_filter=filter] [deferred] [bind] [so_keepalive=on|off|[keepidle]:[keepintvl]:[keepcnt]];  
Default: listen *:80 | *:8000;  
Context: server

配置 `server` 接受请求的来源，或者监听IP与端口，默认监听 80 端口。支持 ip 或者 unix socket。

```
listen 127.0.0.1:8000;
listen 127.0.0.1;
listen 8000;
listen *:8000;
# hostname
listen localhost:8000;
# ipv6
listen [::]:8000;
listen [::1];
# unix socket
listen unix:/var/run/nginx.sock;
```

* `default`

    可以使用 `default` 来指定默认 `server`，如果没有指定，则默认为首个 `server` 为默认。

* `ssl` `http2` `spdy`

    分别对应支持 ssl / http2 / spdy 连接，需要注意的是，由于 http2 合并了 spdy，而浏览器方面均未支持非 SSL 下的 HTTP2 协议，所以当需要 HTTP2 时，均需要配置为 `listen 443 ssl http2`，不过 NGINX 支持无 SSL 的 HTTP2 协议。

    关于HTTP 协议发展历程可以参考 [HTTP的发展 - HTTP | MDN](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Basics_of_HTTP/Evolution_of_HTTP)。

* `proxy_protocol`

    [PROXY protocol](http://www.haproxy.org/download/2.0/doc/proxy-protocol.txt) 是 
    HAProxy 使用的代理协议，`proxy_protocol` 也可以在 `stream` 指令中使用。

    ```
    # https://docs.nginx.com/nginx/admin-guide/load-balancer/using-proxy-protocol/
    http {
    #...
    server {
        listen 80   proxy_protocol;
        listen 443  ssl proxy_protocol;
        #...
        }
    }
   
    stream {
        #...
        server {
            listen 12345 proxy_protocol;
            #...
        }
    }
    ```

* `fastopen`
  
    `fastopen` 需要内核支持 (Linux 3.7+) 并开启 [TCP Fast Open](http://en.wikipedia.org/wiki/TCP_Fast_Open)。通过 `cat /proc/sys/net/ipv4/tcp_fastopen` 可以查看当前配置，1 表示为客户端开启，2 表示为服务端开启，3 则表示都开启。

    `fastopen` 配置值表示等待完成三次握手连接的队列长度。

    ```
    listen 127.0.0.1:8000 fastopen=512
    ```

    TCP Fast Open(TFO) 是针对提高 TCP 三次握手速度而产生的方案，对提高首次响应速度有一定的帮助。

* `deferred`
  
    `deferred` 也是针对 TCP 连接的改善参数，与 TFO 不同，是改善高负载下的应用压力。

    > [tcp - TCP protocol | Linux Programmer's Manual](http://man7.org/linux/man-pages/man7/tcp.7.html)  
    > TCP_DEFER_ACCEPT  
    > Allow a listener to be awakened only when data arrives on the socket.  Takes an integer value (seconds), this can bound the maximum number of attempts TCP will make to complete the connection.  This option should not be used in code intended to be portable.

    `deferred` 是针对 TCP 连接握手成功后，在应用层数据传输完成前，worker 进程不会马上进行处理连接，可以减少 worker 进程的等待引起的调用压力。详细的工作原理可以查看 [Take advantage of TCP/IP options to optimize data transmission](https://www.techrepublic.com/article/take-advantage-of-tcp-ip-options-to-optimize-data-transmission/)。

* `reuseport`
* `rcvbuf` `sndbuf`

    与这两个参数相关的是 TCP 中 `SO_SNDBUF` 与 `SO_RCVBUF` 标志。需要注意，内核对 TCP 的缓冲池有自动调整功能，也有限制，另外缓冲池合理大小也与带宽、延迟相关（不小于带宽与延迟的乘积）。

    > [tcp - TCP protocol | Linux Programmer's Manual](http://man7.org/linux/man-pages/man7/tcp.7.html)  
    > SO_SNDBUF  
    > Sets or gets the maximum socket send buffer in bytes.  The kernel doubles this value (to allow space for bookkeeping overhead) when it is set using setsockopt(2), and this doubled value is returned by getsockopt(2).  The default value is set by the /proc/sys/net/core/wmem_default file and the maximum allowed value is set by the /proc/sys/net/core/wmem_max file.The minimum (doubled) value for this option is 2048.
      
    > SO_RCVBUF  
    > Sets or gets the maximum socket receive buffer in bytes.  The kernel doubles this value (to allow space for bookkeeping overhead) when it is set using setsockopt(2), and this doubled value is returned by getsockopt(2).  The default value is set by the /proc/sys/net/core/rmem_default file, and the maximum allowed value is set by the /proc/sys/net/core/rmem_max file.The minimum (doubled) value for this option is 256.
    
* `bind`
* `ipv6only`
* `so_keepalive`

### 重定向

HTTP 协议中重定向是通过 `Location` 响应头返回到客户端（通常是浏览器），根据相应的响应状态码与 `Location` 头的值，客户端会重新发起新的请求。

#### `absolute_redirect`

>Syntax: absolute_redirect on | off;  
Default: absolute_redirect on;  
Context: http, server, location

默认使用绝对重定向，关闭时，重定向时使用相对路径。

#### `server_name_in_redirect`

>Syntax: server_name_in_redirect on | off;  
Default: server_name_in_redirect off;  
Context: http, server, location

定义在重定向时是否显示 server_name。

#### `port_in_redirect`

>Syntax: port_in_redirect on | off;  
Default: port_in_redirect on;  
Context: http, server, location

定义在重定向时是否显示端口。

#### 配置实例

可以通过实际配置来进行测试，使用 `curl -I` 向对应的 URI 发起请求可以看到 `Location` 的差异。

```
# host ip is 192.168.33.219
server {
    listen 8000 default_server;
    listen [::]:8000 default_server;
    server_name _;
    location /default {
        # Location: http://192.168.33.219:8000/redirect/default

        absolute_redirect on;
        server_name_in_redirect off;
        port_in_redirect on;
        return 302 /redirect$uri;
    }

    location /absolute_redirect_off {
        # Location: /redirect/absolute_redirect_off
        absolute_redirect on;
        server_name_in_redirect off;
        port_in_redirect on;
        return 302 /redirect$uri;
    }

    location /server_name_in_redirect_on {
        # Location: http://_:8000/redirect/server_name_in_redirect_on
        absolute_redirect on;
        server_name_in_redirect on;
        port_in_redirect on;
        return 302 /redirect$uri;
    }

    location /port_in_redirect_off {
        # Location: http://192.168.33.219/redirect/port_in_redirect_off
        absolute_redirect on;
        server_name_in_redirect off;
        port_in_redirect off;
        return 302 /redirect$uri;
    }
}
```

### 异步 IO

> [Thread Pools in NGINX Boost Performance 9x!](https://www.nginx.com/blog/thread-pools-boost-performance-9x/)  
> [Boost application performance using asynchronous I/O - IBM](https://developer.ibm.com/articles/l-async/#system-tuning-for-aio)

Linux 下异步 IO 可以有效的减少因 IO 阻塞增加的响应时间。NGINX 与 AIO 相关的还有 main context 中的 `thread_pool` `worker_aio_requests` 指令。

`aio` 并非“银弹”，性能的提高是有前提存在的：首先是内存不足以缓存；其次，带来的性能提升要高于因线程池切换带来的影响。在系统内存足够，页面缓存工作良好时，`aio` 是不需要开启的。另外 FreeBSD 下因为内核异步接口性能已经够好，所以也不需要开启。

换句话讲，只有在 Linux 平台下，HTTP 服务需要提供大文件，并且磁盘 IO 性能足够时，`aio` 才能带来巨大的性能提升。

#### `aio`

> Syntax: aio on | off | threads[=*pool*];  
Default: aio off;  
Context: http, server, location

在 Linux 平台需要 2.6.22+ 内核才能开启 `aio`，同时还需要开启 `directio`，以免造成读取阻塞：

```
location /video/ {
    aio            on;
    directio       512;
    output_buffers 1 128k;
}
```

通过在编译时添加 `--with-threads` 参数，可以开启 NGINX 多线程支持。可以进一步配置多线程下的 `aio` 来避免产生 IO 阻塞。

```
location /video/ {
    sendfile       on;
    aio            threads=pool$disk;
}
```

#### `aio_write`

>Syntax: aio_write on | off;  
Default: aio_write off;  
Context: http, server, location

当 `aio` 开启时，用于指定是否使用 `aio` 模式写入文件。只有使用 `aio thread` 从入代理服务器数据并写入缓存时有效。

#### `directio`

>Syntax: directio *size* | off;  
Default: directio off;  
Context: http, server, location

在 Linux 平台启用 `aio` 或者提供大文件访问时会非常有用。

> [Linux Programmer's Manual - open(2)](http://man7.org/linux/man-pages/man2/open.2.html)

根据文档说明，在 Linux / FreeBSD 平台当 `aio` 与 `sendfile` 同时开启时，会使用 `O_DIRECT` 标志读取大于等于 `size` 的文件，来避免阻塞 IO。当文件小于 `size` 或者关闭 `directio` 时，则使用 `sendfile`。

#### `directio_alignment`

>Syntax: directio_alignment *size*;  
Default: directio_alignment 512;  
Context: http, server, location

为 `directio` 配置块对齐大小，默认为 512 byte，使用 XFS 文件系统时，这个值应该配置为 4K 的倍数。

#### `sendfile`

>Syntax: sendfile on | off;  
Default: sendfile off;  
Context: http, server, location, if in location

> [Linux Programmer's Manual - sendfile(2)](http://man7.org/linux/man-pages/man2/sendfile.2.html)


开启 `sendfile`时 NGINX 会使用调用 `sendfile()` 来读取文件，对提高 IO 性能有帮助。

#### `sendfile_max_chunk`

>Syntax: sendfile_max_chunk *size*;  
Default: sendfile_max_chunk 0;  
Context: http, server, location

限制单个 `sendfile()` 调用传输的数据大小，避免单个连接消耗掉所有资源。默认为 0,表示无限制。
