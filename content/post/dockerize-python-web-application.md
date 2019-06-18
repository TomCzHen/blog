---
title: Dockerize Python Web 应用
date: 2019-06-09T21:00:00+08:00
categories:
    - Linux
tags:
    - Docker
    - Python
---

虽然“人生苦短，我用 Python”，但是很多时候一个 Python 新手写完第一个 Web 项目之后会陷入 WSGI 是什么？接下来要干啥的蒙蔽状态中。不过好在有 Docker 这个神器，相信了解它之后，就能体验 Python + Docker 的双倍快乐~~并不~~。

本文只是一个向导，基于本地编排，一步一步来实现一个 Flask 应用的容器化，想要能顺畅的阅读，至少需要了解一些 Docker 的基本知识，基本的镜像构建命令。

<!--more-->

> TL;DR: 一个可用样例项目 [TomCzHen/Dockerize-Python-Web-Application](https://github.com/TomCzHen/Dockerize-Python-Web-Application)

典型的 Python Web (Flask) 项目的文件结构大致是这样的：

```
.
├── app.py
├── Pipfile
├── Pipfile.lock
├── .gitignore
└── .venv
```

* app.py

```python
#!/usr/bin/env python3
from flask import Flask
app = Flask(__name__)

@app.route("/")
def hello():
    return "Hello World From Docker!"
```

注：基于个人偏好而使用了 pipenv 管理 Python 包，如果使用其他 Python 包管理方式，请自行替换对应文件并修改相关的 Dockerfile 内容。

## Dockerfile

> 参考资料
> 
> * [Dockerfile reference | Docker Documentation](https://docs.docker.com/engine/reference/builder/
)
> * [Production-ready Docker images](https://pythonspeed.com/docker/)

使用 Dockerfile 构建镜像时默认会将当前路径中所有文件作为 context 发送到 Docker deamon，需要用 .dockerignore 配置构建过程中忽略的路径。

```
# .dockerignore
.env
.git
.venv
__pycache__
```

### 基础镜像

```Dockerfile
FROM python:3.7.3-slim
ENV PIP_NO_CACHE_DIR=1 \
    PYTHONDONTWRITEBYTECODE=1 \
    FLASK_APP="app"
COPY . /app
WORKDIR /app
RUN pip install pipenv && pipenv install --deploy --system
CMD ["flask","run"]
```
上面的 Dockerfile 构建的镜像是可以运行的，不过有一个问题——每次项目代码发生改变时都会执行安装依赖，即便依赖并没有变化。

可以将依赖安装与更新代码分离开，当依赖没有发生变化时就会直接使用构建缓存而不是重新安装。

```Dockerfile
FROM python:3.7.3-slim
ENV PIP_NO_CACHE_DIR=1 \
    PYTHONDONTWRITEBYTECODE=1 \
    FLASK_APP="app"
COPY ["Pipfile","Pipfile.lock","/app/"]
WORKDIR /app
RUN pip install pipenv && pipenv install --deploy --system
COPY . /app
CMD ["flask","run"]
```

#### 构建缓存

在 Docker 构建镜像时会利用缓存加快构建，对缓存机制不注意时会产生一些问题。使用下面这个 Dockerfile 构建镜像时，如果`docker build` 不使用 `--no-cache` 参数，也没删除已经构建过的镜像，构建多个镜像输出时间是一样的。

```Dockerfile
FROM debian
RUN echo $(date) > test.txt
CMD ["cat","test.txt"]
```

同样，在 Dockerfile 中使用 `git clone` 获取代码也会有一样的情况。

```Dockerfile
FROM python:3.7.3-slim
ENV PIP_NO_CACHE_DIR=1 \
    PYTHONDONTWRITEBYTECODE=1 \
    FLASK_APP="app"
RUN apt-get update && apt-get install -yqq git \
    && git clone --depth=1 https://github.com/example/flask-example.git /app

WORKDIR /app

RUN pip install pipenv && pipenv install --deploy --system

CMD ["flask","run"]
```

首先，在构建过程中通过 `git clone` 获取代码是可行的，但由于 Docker 镜像构建缓存，代码不一定是最新的，清除缓存虽然可以避免这个问题，但是会增加构建时间。这样做还产生另一个问题——从私有库获取代码时需要将私钥添加到镜像中。

可以采用以下几种解决方案：

* 在 CI/CD 平台中获取代码，然后从平台本地路径加入代码到镜像。

通过 Jenkins 或 GitLab 的 CI 脚本拉取代码，这样可以避免因构建缓存而无法获取最新代码的问题，同时也可以有效利用缓存加快构建。

* 下载指定版本的代码归档文件构建。

```Dockerfile
FROM python:3.7.3-slim
ENV PIP_NO_CACHE_DIR=1 \
    PYTHONDONTWRITEBYTECODE=1 \
    FLASK_APP="app"
ARG version=1.0.0
RUN apt-get update && apt-get install -yqq curl tar
WORKDIR /app
RUN curl -L https://github.com/example/flask-example/releases/download/${version/flask-example-${version}.tar.gz \
    | tar xz -C /app --strip-components 1 \
    && pip install pipenv \
    && pipenv install --deploy --system
CMD ["flask","run"]
```

* 使用多阶段构建

多阶段构建仍然存在缓存，不过这里主要目的是保护私钥。

```Dockerfile
FROM debian as builder
ARG tag="1.0.0"
ARG ssh_key=""
RUN apt-get update && apt-get install -yqq git \
    && echo ${ssh_key} > ~/.ssh/id.rsa \
    && chmod 700 ~/.ssh/id_rsa \
    && git clone --depth=1 -b ${tag} ssh://github.com/example/flask-example.git /app \
    && rm -rf app/.git

FROM python:3.7.3-slim
ENV PIP_NO_CACHE_DIR=1 \
    PYTHONDONTWRITEBYTECODE=1 \
    FLASK_APP="app"
COPY --from=builder ["/app","/"]
COPY . /app
WORKDIR /app
RUN pip install pipenv && pipenv install --deploy --system
CMD ["flask","run"]
```

无论采取何种方式，如果想确保每次构建获取的都是最新的代码，要么使用 `--no-cache` 参数，要么在获取代码前引入参数变化。

#### 优化结构

当存在多个项目时上面的 Dockerfile 会产生非常多的冗余内容，而且需要更新一些环境工具依赖时需要修改多处 Dockerfile 才能实现。

可以在 FROM 时使用自己构建的镜像作为基础镜像来解决这个问题。

```Dockerfile
# tomczhen/python-pipenv-base
FROM python:3.7.3-slim
ENV PIP_NO_CACHE_DIR=1 \
    PYTHONDONTWRITEBYTECODE=1
RUN apt-get update && apt-get install -yqq git \
    && pip install -U pipenv
WORKDIR /app
ONBUILD COPY ["Pipfile","Pipfile.lock","/app"]
ONBUILD RUN pipenv install --deploy --system
CMD ["flask","run"]
```

假设构建后的镜像标签为 `tomczhen/python-pipenv-base:3.7.3

```Dockerfile
FROM tomczhen/python-pipenv-base:3.7.3
ENV FLASK_APP="app"
COPY . /app
CMD ["flask","run"]
```

#### 优化体积

需要说明的是 `docker images` 显示的体积并非传输时的大小。其次，基于 Docker 镜像分层的机制，传输和保存时会复用已经存在的分层，所以体积并非一个很大的问题。

需要尽量减少镜像体积时可以选择 `python：3.7.3-alpine3.9` 这个基于 alpine 构建的 python 基础镜像。

```Dockerfile
FROM python：3.7.3-alpine3.9
ENV PIP_NO_CACHE_DIR=1 \
    PYTHONDONTWRITEBYTECODE=1 \
    FLASK_APP="app"
COPY ["Pipfile","Pipfile.lock","/app/"]
WORKDIR /app
RUN pip install pipenv && pipenv install --deploy --system
COPY . /app
CMD ["flask","run"]
```

##### 但是，代价是什么？

> 参考资料：
>
> * [musl libc - Functional differences from glibc](https://wiki.musl-libc.org/functional-differences-from-glibc.html)

alpine 使用的是 musl-libc 而不是 glibc，对于 CPython ，有些函数使用了 C library 的基础功能，不同的实现有差异存在——包括性能和运行结果。而 alpine 体积小的优势是依靠大量精简来实现的，比如默认只有 sh 作为解释器，一些常见外部命令也无法使用，需要自己安装运行依赖来确保代码正常运行。

考虑到因此带来的代价，只有当体积是一个不得不解决的问题时，才使用 alpine 作为基础镜像。否则，除非有能力解决 musl-libc 差异引起的问题，并且愿意接受因此带来的时间成本与风险才使用它。

#### 编译安装

Python 包正常使用的依赖区分为两种：安装依赖、运行依赖——当安装依赖有问题时，安装会出错；当运行依赖不存在时运行会出错。

在 alpine 中未安装 `ca-certificates` 之前，系统没有配置 CA 证书，如果 Python 包依赖系统 CA 证书，那么 HTTPS 连接会出现校验错误。如果 Python 包存在 Building wheel 阶段，缺少编译依赖时会无法安装，比如 psycopg2。

注：当然，可以安装 psycopg2-binary 来解决问题，不过官方文档明确的说明：

> The binary package is a practical choice for development and testing but in production it is advised to use the package built from sources.

当 Python 包中存在 build wheel 阶段并且需要安装 gcc 之类的编译环境时有两种选择。

* 发行版包管理器

    发行版中一般存在 python3-xxxx 的包，可以通过发行版的包管理器进行安装。如果符合需要，那么直接通过包管理安装也是很好的选择，可以节省很多时间。

* 编译安装

    发行版提供的二进制安装包版本不能满足需要，或者干脆就没有提供（alpine 常见）对应的包，而 pip 安装又有 build wheel 阶段，那么只能选择根据文档安装好编译依赖。

#### 多阶段构建

在需要编译安装的场合，推荐使用多阶段构建的方式构建 Docker 镜像。同时，可以使用虚拟环境来进一步压缩镜像体积，以安装 psycopg2 为例。

```Dockerfile
FROM python:3.7.3-slim as builder
ENV PIP_NO_CACHE_DIR=1 \
    PYTHONDONTWRITEBYTECODE=1
RUN apt-get update && apt-get install -yqq gcc \
    python3-dev \
    libpq-dev \
    && pip install pipenv

COPY ["Pipfile","Pipfile.lock","/app/"]
WORKDIR /app
RUN mkdir .venv \
    && pipenv install --deploy

FROM python:3.7.3-slim
ENV PATH="/app/.venv/bin:${PATH}" \
    PYTHONDONTWRITEBYTECODE=1
    FLASK_APP="app"
RUN apt-get update && apt-get install libpq5 -yqq
COPY --from=builder ["/app","/"]
COPY . /app
CMD ["flask", "run"]
```

### WSGI Server

> 参考资料
>
> * [Deploy to Production - Flask documentation](flask.pocoo.org/docs/tutorial/deploy/)
> * [Deploying Django | Django documentation](https://docs.djangoproject.com/en/dev/howto/deployment/)

实现镜像构建之后，开发环境中项目文件结构大致会变成这样：

```
.
├── Dockerfile
├── app.py
├── Pipfile
├── Pipfile.lock
├── .dockergitignore
├── .gitignore
└── .venv
```

显然部署 Python Web 项目时是不能使用 `flask run` 方式运行的，一般会使用 Nginx/Caddy + WSGI Server 的方式，所以 Dockerfile 还需要进行修改。另外还有一个问题是 Docker 中默认以 root 用户权限运行，为了安全方面的考量，需要切换为非 root 用户。

根据一个服务一个容器的原则，Web Server 容器和 WSGI Server 容器需要分开运行。但这不是必须的，如果确实有必要，将多个应用放在同一个容器中运行也是可行的。

为了后续多个 Dockerfile 的管理，需要调整一下项目结构：

```
.
├── docker
│   ├── Dockerfile
│   ├── docker-entrypoint.sh
│   └── caddy
│       └── Dockerfile
├── app.py
├── docker-compose.yaml
├── Pipfile
├── Pipfile.lock
├── .dockergitignore
├── .env
├── .gitignore
└── .venv
```

注：在 .gitignore .dockergitignore 中添加忽略 .env。

#### uWSGI

> 参考资料：
>
> * [Configuring uWSGI - uWSGI 2.0 documentation](https://uwsgi-docs.readthedocs.io/en/latest/Configuration.html#environment-variables)
> * [Add WSGI directive for serving Python apps · Issue #176 · mholt/caddy](https://github.com/mholt/caddy/issues/176)

注：由于 Caddy 的 uwsgi 协议支持只是还是非正式的，所以 uWSGI 的配置与使用 Nginx uwsgi module 时有些差别。

```Dockerfile
ARG python_version="3.7.3"
FROM python:${python_version}-slim as builder
ENV PIP_NO_CACHE_DIR=1 \
    PYTHONDONTWRITEBYTECODE=1
RUN apt-get update && apt-get install -yqq gcc \
    python3-dev \
    libpq-dev \
    && pip install pipenv 

COPY ["Pipfile","Pipfile.lock","/app/"]
WORKDIR /app
ARG uwsig_version="2.0.18"
RUN mkdir .venv \
    && pipenv install --deploy \
    && pipenv install uwsgi==${uwsig_version} --skip-lock

FROM python:${python_version}-slim
RUN apt-get update && apt-get install libpq5 -yqq

WORKDIR /app

ENV PATH="/app/.venv/bin:${PATH}" \
    PYTHONDONTWRITEBYTECODE=1 \
    UWSGI_HTTP_SOCKET=":3031" \
    UWSGI_MASTER=1 \
    UWSGI_WORKERS=2 \
    UWSGI_THREADS=4 \
    UWSGI_WSGI_FILE="app.py" \
    UWSGI_CALLABLE="app" \
    UWSGI_VIRTUALENV="/app/.venv" \
    UWSGI_STATUS=":9191"
COPY --from=builder ["/app","/app"]
COPY . /app
EXPOSE 3031 9191
USER nobody
CMD ["uwsgi"]
```

#### ENTRYPOINT

通过 `ENV` 可以为容器添加环境变量，前面已经使用了 `PIP_NO_CACHE_DIR` 与 `PYTHONDONTWRITEBYTECODE` 来配置 pip 使用 cache，Python 不产生字节码文件，uWSGI/Gunicorn 也支持环境变量来配置，Flask 也有很多变量可以用于配置。

除此之外，可能还需要在启动之前做一些处理，那么就需要 `ENTRYPOINT` 了，一个典型的场景是等待数据库服务可用后才启动。

```shell
#!/usr/bin/env bash
# docker/docker-entrypoint.sh

retry_times=0
sleep_sec=5

check_database_uri() {
    local database_uri="${DATABASE_DRIVER}://${DATABASE_USER}:${DATABASE_PASSWORD}@${DATABASE_HOST}:${DATABASE_PORT}/${DATABASE_NAME}"

    if [[ -z "${database_uri}" ]]; then
        echo "SQLALCHEMY_DATABASE_URI not set or empty"
        exit 1
    fi

    python -c "from sqlalchemy import create_engine;engine = create_engine('${database_uri}');conn=engine.connect();conn.execute('SELECT 1');conn.close()"

}

main() {

    if [[ ${1} = "uwsgi" && ${#} = 1 ]];then

        until check_database_uri; do
            if [[ ${retry_times} -lt 3 ]]; then
                >&2 echo "Database server is unavailable - retry after ${sleep_sec} sec"
                retry_times=$((${retry_times} + 1))
                sleep ${sleep_sec}
            else
                exit 1
            fi
        done
    fi

    exec $@
}

main "$@"
```

```Dockerfile
ARG python_version="3.7.3"
FROM python:${python_version}-slim as builder
ENV PIP_NO_CACHE_DIR=1 \
    PYTHONDONTWRITEBYTECODE=1
RUN apt update && apt install -yqq gcc \
    python3-dev \
    libpq-dev \
    && pip install pipenv 

COPY ["Pipfile","Pipfile.lock","/app/"]
WORKDIR /app
ARG uwsig_version="2.0.18"
RUN mkdir .venv \
    && pipenv install --deploy \
    && pipenv install uwsgi==${uwsig_version} --skip-lock

FROM python:${python_version}-slim

COPY --chown=root:root ["docker/docker-entrypoint.sh","/usr/local/bin/"]
RUN apt update && apt install libpq5 -yqq \
    && chmod +x "/usr/local/bin/docker-entrypoint.sh"

WORKDIR /app

ENV PATH="/app/.venv/bin:${PATH}" \
    PYTHONDONTWRITEBYTECODE=1 \
    UWSGI_HTTP_SOCKET=":3031" \
    UWSGI_MASTER=1 \
    UWSGI_WORKERS=2 \
    UWSGI_THREADS=4 \
    UWSGI_WSGI_FILE="app.py" \
    UWSGI_CALLABLE="app" \
    UWSGI_VIRTUALENV="/app/.venv" \
    UWSGI_STATUS=":9191"

COPY --from=builder ["/app","/app"]
COPY . /app
ENTRYPOINT ["docker-entrypoint.sh"]
EXPOSE 3031 9191
USER nobody
CMD ["uwsgi"]
```

注意：需要保证 `docker-entrypoint.sh` 有可执行权限。

### Caddy

Web Server 选择基本就是 Nginx 了，但是本文使用 Caddy :doge:，考虑到 Caddy 的插件是在编译时引入的，那么通过自定义 Dockerfile 来构建 Caddy 镜像是有必要的。

```Dockerfile
# docker/caddy/Dockerfile
FROM alpine:latest as builder
RUN apk add --no-cache curl bash gnupg

ARG plugins="http.cache,http.cors,http.expires,http.realip,http.git"
RUN curl https://getcaddy.com | bash -s personal ${plugins}

FROM alpine:latest

RUN apk add --no-cache openssh-client ca-certificates git

COPY --from=builder ["/usr/local/bin/caddy","/usr/local/bin/"]

ENV CADDYPATH="/caddy"

WORKDIR /caddy

RUN mkdir -p "etc" "www" "logs"
VOLUME ["/caddy"]
EXPOSE 80 443 2015
CMD ["caddy","-agree","-conf","etc/Caddyfile"]
```

* Caddyfile

```Caddyfile
flask.exmaple.com {
    root /caddy/www/flask
    tls flask@example.com
    proxy / flask:3031 {
        except /static /robots.txt
        transparent
    }
}
```

## Docker Compose

最后通过 docker-compose.yaml 编排文件来把所有的应用组合起来。

```yaml
# docker-compose.yaml
version: "3.7"
services:
  flask:
    build:
      context: .
      dockerfile: docker/Dockerfile
    restart: on-failure
    networks:
      - caddy-network
      - flask-network
    depends_on:
      - caddy
      - postgres
    volumes:
      - type: volume
        source: flask-static-data
        target: /app/static
    env_file:
      - .env
    logging: &logging
      driver: "json-file"
      options:
        max-size: "20m"
        max-file: "10"
  postgres:
    image: postgres:11.3-alpine
    environment:
      POSTGRES_USER: ${DATABASE_USER}
      POSTGRES_PASSWORD: ${DATABASE_PASSWORD}
      POSTGRES_DB: ${DATABASE_NAME}
    networks:
      flask-network:
        aliases:
          - ${DATABASE_HOST}
    volumes:
      - type: volume
        source: postgres-data
        target: /var/lib/postgresql/data
    logging:
      <<: *logging
  caddy:
    build:
      context: docker/caddy
    restart: unless-stopped
    networks:
      - caddy-network
    volumes:
      - type: volume
        source: flask-static-data
        target: /caddy/www/flask/static
        read_only: true
      - type: volume
        source: caddy-data
        target: /caddy
      - type: bind
        source: ./docker/caddy/Caddyfile
        target: /caddy/etc/Caddyfile
    ports:
      - target: 80
        published: 80
        protocol: tcp
        mode: host
      - target: 443
        published: 443
        protocol: tcp
        mode: host
    logging:
      <<: *logging

networks:
  caddy-network:
    name: caddy-network
  flask-network:
    name: flask-network

volumes:
  flask-static-data:
  postgres-data:
  caddy-data:
```

通过加载 `.env` 文件对容器进行配置，需要注意，开发环境下如果使用了 pipenv ，激活虚拟环境时也会自动加载 `.env` 配置环境变量。

注意：docker-compose 自动加载 `.env` 文件只对 `docker-compose.yaml` 内容产生影响，并不会传入容器内部，可以使用 `docker-compose config` 查看完整内容。如果想加载到容器内部，要么使用 `env_file` 加载，要么在 `environment` 再指定一次变量。

```shell
# .env
# Flask Config

FLASK_APP=app

# Database Config

DATABASE_DRIVER=postgresql+psycopg2
DATABASE_HOST=postgres
DATABASE_PORT=5432
DATABASE_NAME=flask
DATABASE_USER=flask
DATABASE_PASSWORD=P@ssw0rd

# uWSGI Config

UWSGI_WORKERS=4
UWSGI_THREADS=2
```

这样只需要编排文件，加上 .env 配置，就能在不同配置下运行了。如果情况允许，可以通过将 Caddyfile 添加到 Caddy 镜像中，并通过 Caddy 支持的变量方式来配置。

```
{$CADDY_DOMAIN} {
    root /caddy/www/{$CADDY_DOMAIN}
    gzip
    tls {$CADDY_TLS_EMAIL}
    proxy / flask:3031 {
        except /static /robots.txt
        transparent
    }
}
import /caddy/etc/*.Caddyfile
```