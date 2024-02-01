---
title: Nginx 学习笔记 —— 基础
authors:
  - tomczhen
date: 2019-10-26T14:00:00+08:00
categories:
    - Linux
tags:
    - NGINX
---

NGINX (engine x) 可以用作 HTTP 服务、反向代理、邮件代理和 TCP/UDP 代理，NGINX 有开源版本以及商业支持的 Plus 版本，在高并发场景下，具有高性能、低资源的优点。

NGINX 使用 C 语言开发，在 Linux 平台使用 epoll、BSD 平台使用的 kqueue 异步机制。需要注意的是 Windows 平台虽然有 IOCP 异步 API，但 NGINX 在 Windows 平台使用的是阻塞 IO 模型 select，因此在 Windows 平台 NGINX 性能上无法参照 Linux/BSD 平台，除此之外 NGINX 还针对 Linux/BSD 平台有着 IO 优化，这些也会对性能提高有着帮助。

<!-- more -->

Windows 平台下 IIS 由于使用了 IOCP 异步机制，并且 `http.sys` 处于内核态运行，理论没有第三方 Web Server 能在 Windows 上性能超过 IIS。并且，IIS 可以通过 Application Request Routing （ARR）功能模块来实现负载均衡、反向代理等功能。

## 安装

安装有两种选择，使用发行版的包管理器安装，或者从源码编译。

> [NGINX: Linux packages](http://nginx.org/en/linux_packages.html)

二进制包的源码可以从 [http://hg.nginx.org/pkg-oss](http://hg.nginx.org/pkg-oss) 获取，选择需要的版本，并下载 zip 或 gz 压缩包即可。

> [NGINX: Building NGINX from Sources](http://nginx.org/en/docs/configure.html)

NGINX 开源版本源码可以从 [NGINX: download](https://nginx.org/en/download.html) 下载。

需要注意发行版仓库中分发的版本编译参数与 Module 可能会与源码版本中默认配置不一样，不能视为一致。

### Docker 镜像

官方镜像可以在  [NGINX - Docker Hub](https://hub.docker.com/_/nginx) 上查看，通过 [Dockerfile](https://github.com/nginxinc/docker-nginx/blob/master/mainline/buster/Dockerfile) 可以看出，官方镜像是从源码编译安装的。

当然，也可以在构建镜像时直接使用包管理器安装，但是需要修改 NGINX 的启动方式。

### 第三方 Module

NGINX 开源版本可以在编译时自行添加第三方 Module [NGINX 3rd Party Modules](https://www.nginx.com/resources/wiki/modules/)，[Compiling Modules | NGINX](https://www.nginx.com/resources/wiki/extending/compiling/) 中有相关编译说明。

## 命令行参数

* `-v` - 查看 nginx 版本信息。
* `-V` - 查看详细的 nginx 版本信息，包括编译参数，功能模块。
* `-c <filename>` - 指定配置文件，默认路径为 `/etc/nginx/nginx.conf`，可以在编译时指定。
* `-t` - 检查配置文件是否有效。 
* `-T` - 插件配置文件是否有效并输出。
* `-g` - 在配置文件外指定配置信息。
* `-s signal` - 通过发送 signal 到主进程控制 nginx。
  * `quit` – 优雅的退出 nginx
  * `reload` – 重新加载配置文件，开启新的 worker 进程，并优雅的退出旧的 worker 进程
  * `reopen` – 重新打开 log 文件
  * `stop` – 立即停止 nginx

### 使用

* 检测指定的配置文件

```
nginx -t -c /etc/nginx/conf.d/example.conf
```

* 检测并查看当前的配置信息

```
nginx -T
```

* 配置文件有效时重载配置

```
nginx -tq && nginx -s reload || echo "invalid config"
```

## 目录结构

以 Debian 10 为例，使用包管理器安装 NGINX 后，配置路径默认在 `/etc/nginx` 下，日志默认输出在 `/var/log/nginx` 路径。

```
/etc/nginx
|-- conf.d
|-- fastcgi.conf
|-- fastcgi_params
|-- koi-utf
|-- koi-win
|-- mime.types
|-- modules-available
|-- modules-enabled
|   |-- 50-mod-http-auth-pam.conf -> /usr/share/nginx/modules-available/mod-http-auth-pam.conf
|   |-- 50-mod-http-dav-ext.conf -> /usr/share/nginx/modules-available/mod-http-dav-ext.conf
|   |-- 50-mod-http-echo.conf -> /usr/share/nginx/modules-available/mod-http-echo.conf
|   |-- 50-mod-http-geoip.conf -> /usr/share/nginx/modules-available/mod-http-geoip.conf
|   |-- 50-mod-http-image-filter.conf -> /usr/share/nginx/modules-available/mod-http-image-filter.conf
|   |-- 50-mod-http-subs-filter.conf -> /usr/share/nginx/modules-available/mod-http-subs-filter.conf
|   |-- 50-mod-http-upstream-fair.conf -> /usr/share/nginx/modules-available/mod-http-upstream-fair.conf
|   |-- 50-mod-http-xslt-filter.conf -> /usr/share/nginx/modules-available/mod-http-xslt-filter.conf
|   |-- 50-mod-mail.conf -> /usr/share/nginx/modules-available/mod-mail.conf
|   `-- 50-mod-stream.conf -> /usr/share/nginx/modules-available/mod-stream.conf
|-- nginx.conf
|-- proxy_params
|-- scgi_params
|-- sites-available
|   `-- default
|-- sites-enabled
|   `-- default -> /etc/nginx/sites-available/default
|-- snippets
|   |-- fastcgi-php.conf
|   `-- snakeoil.conf
|-- uwsgi_params
`-- win-utf
```

根据不同的功能模块划分了不同的路径以及配置文件，通过 `nginx -T` 可以查看完整的配置信息。

### 配置文件

> 参考：  
> * [How to Configure NGINX](https://www.linode.com/docs/web-servers/nginx/how-to-configure-nginx/)  


默认主配置文件可以通过 `nginx -V` 中显示的 `--conf-path` 参数可以获取，通过 `systemctl status nginx` 可以查看是否指定了配置文件路径。

另外，可以通过一些在线工具网站来生成配置文件，比如 [NGINX Config.io](https://nginxconfig.io/) 和 [Mozilla SSL Configuration Generator](https://ssl-config.mozilla.org/)。

#### 配置文件结构

以下是 Debian 10 中安装 NGINX 后 `/etc/nginx/nginx.conf` 部分内容。

```
# /etc/nginx/nginx.conf
user www-data;
worker_processes auto;
pid /run/nginx.pid;
include /etc/nginx/modules-enabled/*.conf;

events {
        worker_connections 768;
        # multi_accept on;
}

http {

        ##
        # Basic Settings
        ##

        sendfile on;
        tcp_nopush on;
        tcp_nodelay on;
        keepalive_timeout 65;
        types_hash_max_size 2048;
        # server_tokens off;

        # server_names_hash_bucket_size 64;
        # server_name_in_redirect off;

        include /etc/nginx/mime.types;
        default_type application/octet-stream;

        ##
        # SSL Settings
        ##

        ssl_protocols TLSv1 TLSv1.1 TLSv1.2; # Dropping SSLv3, ref: POODLE
        ssl_prefer_server_ciphers on;

        ##
        # Logging Settings
        ##

        access_log /var/log/nginx/access.log;
        error_log /var/log/nginx/error.log;

        ##
        # Gzip Settings
        ##

        gzip on;

        # gzip_vary on;
        # gzip_proxied any;
        # gzip_comp_level 6;
        # gzip_buffers 16 8k;
        # gzip_http_version 1.1;
        # gzip_types text/plain text/css application/json application/javascript text/xml application/xml application/xml+rss text/javascript;

        ##
        # Virtual Host Configs
        ##

        include /etc/nginx/conf.d/*.conf;
        include /etc/nginx/sites-enabled/*;
}
```

NGINX 配置由指令与参数组成，对于单行指令每行都会以 `;` 作为结尾。包含内容的指令则使用 `{}` 包含其他指令，通常都会按不同功能划分为各个块进行配置。

#### main Context

> 参考:  
> * [NGINX - Core functionality](https://nginx.org/en/docs/ngx_core_module.html)

`main` context 可以包含以下 block:

* events – General connection processing
* http – HTTP traffic
* mail – Mail traffic
* stream – TCP and UDP traffic

##### `deamon`

>Syntax: daemon on | off;  
Default: daemon on;

配置是否以守护进程运行 NGINX，通常只会在开发环境下使用。

##### `env`

>Syntax: env variable[=value];  
Default: env TZ;

默认情况下，NGINX 父进程会移除 `TZ` 之外的环境变量。通过 `env` 指令可以定义保留的环境变量或者修改变量值、创建新的变量。

> 参考:  
> * [TZ Variable (The GNU C Library)](https://www.gnu.org/software/libc/manual/html_node/TZ-Variable.html)

使用示例：

```
env MALLOC_OPTIONS;
env PERL5LIB=/data/site/modules;
env OPENSSL_ALLOW_PROXY_CERTS=1;
```

注意：NGINX 内部环境变量无法由用户设置值。

#### `include`

>Syntax: include *file* | *mask*;

`include` 用于引入其他配置文件，可以在任意 context 中使用。

比如：

```
include /etc/nginx/mime.types;
include /etc/nginx/conf.d/*.conf;
```

为了便于维护配置文件，可以按不同功能模块来划分配置文件，然后使用 `include` 指令引入配置文件。

##### `load_module`

加载一个动态模块。

比如：

```
load_module modules/ngx_mail_module.so;
```

##### `master_process`

> Syntax: master_process on | off;  
Default: master_process on;

在开发环境中使用 `master_process off` 可以让 NGINX 不开启 master 进程运行，永远不要在生产环境中使用。

##### `pcre_jit`

>Syntax: pcre_jit on | off;  
Default: pcre_jit off;

定义是否启用 PCRE JIT。开启 PCRE JIT 可以有效提供正则性能，但是需要在编译时使用 `--with-pcre=` `--with-pcre-jit` 参数。

##### `pid`

>Syntax: pid *file*;  
Default: pid logs/nginx.pid;

定义保存 NGINX 主进程 PID 的文件。

##### `ssl_engine`

>Syntax: ssl_engine *device*;  
Default: —

定义 SSL 硬件加速器。

##### `thread_pool`

>Syntax: thread_pool *name* threads=*number* [max_queue=*number*];  
Default: thread_pool default threads=32 max_queue=65536;

定义线程池名称，大小，线程队列长度，在使用非阻塞模式时可以通过 `aio` 等指令来进行配置，解决无法缓存到内存时产生的 IO 阻塞带来的性能问题。

> 参考：  
> * [Thread Pools in NGINX Boost Performance 9x!](https://www.nginx.com/blog/thread-pools-boost-performance-9x/)

##### `timer_resolution`

>Syntax: timer_resolution *interval*;  
Default: —

NGINX 有自己的时间缓存机制存在，未配置时，由 NGINX 根据指定的内核事件时调用更新时间缓存。为了避免因为缓存造成时间精度问题，可以通过定义 `timer_resolution` 指令来配置更新间隔，例如：`timer_resolution 100ms`。

##### `user`

>Syntax: user user [group];  
Default: user nobody nobody;

定义 worker 进程所属的用户以及组。

##### `worker_cpu_affinity`

>Syntax: worker_cpu_affinity *cpumask* ...;  
worker_cpu_affinity auto [*cpumask*];

定义 worker 进程绑定 CPU 核心，通过该配置可以提供多核 CPU 的利用率，有助于提升性能。

`cpumask` 根据 `worker_processes` 和 CPU 逻辑核心数决定，使用二进制数字表示。

使用示例：

```
worker_processes    4;
worker_cpu_affinity 0001 0010 0100 1000;
```

```
worker_processes    2;
worker_cpu_affinity 0101 1010;
```

```
worker_cpu_affinity auto 01010101;
```

需要注意的是，使用 Docker 容器运行时，NGINX 获取的是物理主机资源信息，而非容器可用资源，如果确实需要使用 `worker_cpu_affinity` 还需要根据实际情况调整。

##### `worker_priority`

>Syntax: worker_priority *number*;  
Default: worker_priority 0;

定义 worker 进程调度优先级，`number` 允许的值为 -20 到 20，数字越小，优先级越高。

##### `worker_processes`

>Syntax: worker_processes *number* | auto;  
Default: worker_processes 1;

定义 worker 进程的数量。

决定最佳值的因素很多，包括 CPU 逻辑核心数、存储等等。通常会设置为 CPU 核心数一致，或者使用 `auto`，NGINX 会检测可用的 CPU 核心数并自动配置。

##### `worker_rlimit_core`

>Syntax: worker_rlimit_core *size*;

可以不重启 main 进程调整 worker 进程生产的 dump core 文件大小。

dump core 可用于调试，相关信息可以查看 [Core dump - ArchWiki](https://wiki.archlinux.org/index.php/Core_dump)。

##### `worker_rlimit_nofile`

>Syntax: worker_rlimit_nofile *number*;

调整 worker 进程打开文件的最大限制，没有配置时使用内核的限制。

`worker_rlimit_nofile` 还受到系统参数限制，可以通过 `ulimit -Sn` 与 `ulimit -Hn` 命令查看。可以通过 `cat /proc/$(cat logs/nginx.pid)/limits` 查看 NGINX 当前使用的受限资源。

>参考:  
> * [通过 ulimit 改善系统性能](https://www.ibm.com/developerworks/cn/linux/l-cn-ulimit/index.html)  
> * [Is it safe to increase operating system ulimits?](https://www.ibm.com/support/pages/it-safe-increase-operating-system-ulimits)

针对 NGINX 可以使用两种方式来调整 `ulimit` 限制：

* 通过 `/etc/security/limits.conf` 文件配置
* 通过在 systemctl 配置文件中添加 `LimitNOFILE=65536` 进行配置

此外，`fs.file-max` `fs.file-nr` 内核参数也会影响到打开的文件句柄数量，不过通常这个值是足够大的。

##### `worker_shutdown_timeout`

>Syntax: worker_shutdown_timeout *time*;

配置 worker 进程退出超时时间。

正常情况下 worker 进程退出时会进行等待（定时事务、业务逻辑处理）完成，当配置 `worker_shutdown_timeout` 后，到达指定的时间就会强制关闭连接，退出 worker 进程，以释放系统资源。

##### `working_directory`

>Syntax: working_directory directory;

定义 worker 进程的当前工作路径。通常用于写入 dunmp core 文件，worker 进程需要有写入权限。

#### event Context

##### `accept_mutex`

>Syntax: accept_mutex on | off;  
Default: accept_mutex off;

启用 `accept_mutex` 命令时，worker 进程会轮流处理新连接，未启用时，则由所有 worker 进程来争抢新连接。

`accept_mutex` 用意是解决[惊群问题](https://zh.wikipedia.org/zh-hans/%E6%83%8A%E7%BE%A4%E9%97%AE%E9%A2%98),因此在 `1.11.3` 之前的版本中，该命令是默认开启的。

Linux 4.5, glibc 2.24 增加了 `EPOLLEXCLUSIVE` 在内核 epoll 中解决了这个问题，因此在 `1.11.3` 版本之后，`accept_mutex` 默认是关闭的。另外，当使用了 `reuseport` 参数时，该选项也无需开启。

注意：一些企业发行版比如 SuSE 和 RedHat 会将高版本内核特性 backport 到内核中，内核版本低于 4.5 时也可能支持。

##### `accept_mutex_delay`

>Syntax: accept_mutex_delay *time*;  
Default: accept_mutex_delay 500ms;

在启用 `accept_mutex` 时，可以通过 `accept_mutex_delay` 配置 worker 进程获取新连接的间隔时间。

##### `multi_accept`

>Syntax: multi_accept on | off;  
Default: multi_accept off;

启用 `multi_accept` 时 worker 进程会一次尽量多的接受连接，否则一次接受一条新连接。此外，如果一次接受的连接数比 `worker_connections` 设置得少，会产生错误。

官方推荐是配置为关闭，除非确定开启会有帮助。

*云推测：如果是小请求，在负载合理的前提下，开启应该对缩短响应速度是有帮助的，但是大请求则未必。*

##### `use`

>Syntax: use *method*;

一般情况下不需要指定，NGINX 默认会使用最佳的值，在 Linux 上通常是 epoll，BSD 平台则是 kqueue。

##### `worker_aio_requests`

>Syntax: worker_aio_requests *number*;  
Default: worker_aio_requests 32;

指定每个 worker 线程可使用 (aio)异步 IO 操作数量。

>参考:  
>* [Boost application performance using asynchronous I/O - IBM](https://developer.ibm.com/articles/l-async/#system-tuning-for-aio)

##### `worker_connections`

>Syntax: worker_connections *number*;  
Default: worker_connections 512;

设置单个 worker 进程可以同时打开的最大连接数量。

这个连接数量不仅仅是指与客户端的连接，而是包括代理连接等。另外，`worker_connections` 实际可用值还受 `worker_rlimit_nofile` 限制，需要保证 `worker_processes` * `worker_connections` 小于或等于 ulimit 中配置的可用文件句柄数。