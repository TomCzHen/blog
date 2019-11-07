---
title: Nginx 学习笔记——基础
date: 2019-10-26T14:00:00+08:00
categories:
    - Linux
tags:
    - NGINX
---

NGINX (engine x) 可以用作 HTTP 服务、反向代理、邮件代理和 TCP/UDP 代理，NGINX 有开源版本以及商业支持的 Plus 版本，在高并发场景下，具有高性能、低资源的优点。

## 安装

安装有两种选择，使用发行版的包管理器安装，或者从源码编译。

> [NGINX: Linux packages](http://nginx.org/en/linux_packages.html)

二进制包的源码可以从 [http://hg.nginx.org/pkg-oss](http://hg.nginx.org/pkg-oss) 获取，选择需要的版本，并下载 zip 或 gz 包即可。

> [NGINX: Building NGINX from Sources](http://nginx.org/en/docs/configure.html)

nginx 开源版本源码可以从 [NGINX: download](https://nginx.org/en/download.html) 下载。

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

为了便于维护配置文件，按不同功能模块来划分配置文件，然后使用 `include` 指令引入配置文件，。
```
user             nobody;
error_log        logs/error.log notice;
worker_processes 1;
include /etc/nginx/conf.d/http.conf;
include /etc/nginx/conf.d/stream.conf;
```

以下指令则必须包含有上下文：

* events – General connection processing
* http – HTTP traffic
* mail – Mail traffic
* stream – TCP and UDP traffic

在这些指令之外的内容都被称为 main context。

#### MIME Type

在默认配置文件中可以看到有这样的段对 mime 进行配置：

```
include /etc/nginx/mime.types;
default_type application/octet-stream;
```

HTTP 客户端——通常是浏览器，会根据 `Content-Type` 响应头对响应内容进行处理，比如 `text/html` 则会作为 HTML 内容进行渲染。

`mime.types` 中保存了不同资源类型对应的 `Content-Type` 响应值，如果没有对应的资源类型，则取 `default_type` 指令中配置的值。