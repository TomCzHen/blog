---
title: 在 Windows IIS 上部署 Django 项目
authors:
  - tomczhen
date: 2024-01-25T23:30:00+08:00
categories:
  - Python
tags:
  - Python
  - IIS
  - Windows
---

!!! quote "参考资料"

    * [为 Python Web 应用配置 IIS](https://learn.microsoft.com/zh-cn/visualstudio/python/configure-web-apps-for-iis-windows){:target="_blank"}
    * [HttpPlatformHandler 配置参考](https://learn.microsoft.com/zh-cn/iis/extensions/httpplatformhandler/httpplatformhandler-configuration-reference){:target="_blank"}

Django 官方文档中提供的生产部署方案并没有在 Windows 上部署的方案，但是拥抱开源的巨硬还是提供了 Python 在 IIS 上的部署方案，早前是使用 FastCGI 与 [WFastCGI](https://pypi.org/project/wfastcgi/){:target="_blank"} 一起使用。
而来到 2023 年，微软推荐使用 HttpPlatformHandler 的方式来托管 Python Web 应用。

诚然，Linux 或者容器的部署方案或许更常见一些，但是能在 Windows 下只获得可以接受的效果，也可以节省很多资源，毕竟不是人人都需要高并发。希望也能稍微改善一下直接 `runserver` 运行后吐槽 Django 性能不行的情况吧。

<!-- more -->

## 运行环境

正常安装 IIS 后还需要安装 [URL 重写模块](https://learn.microsoft.com/zh-cn/iis/extensions/url-rewrite-module/using-the-url-rewrite-module){:target="_blank"} 和 [应用程序请求路由模块](https://learn.microsoft.com/zh-cn/iis/extensions/planning-for-arr/using-the-application-request-routing-module){:target="_blank"} 还有 [HttpPlatformHandler](https://www.iis.net/downloads/microsoft/httpplatformhandler){:target="_blank"}。

按照微软文档中的例子 [为 Python Web 应用配置 IIS](https://learn.microsoft.com/zh-cn/visualstudio/python/configure-web-apps-for-iis-windows){:target="_blank"} ，部署 Django IIS 站点的示例配置如下

```xml title="web.config"
<?xml version="1.0" encoding="utf-8"?>
<configuration>
    <system.webServer>
        <handlers>
            <add name="PythonHandler" path="*" verb="*" modules="httpPlatformHandler" resourceType="Unspecified"/>
        </handlers>
        <httpPlatform processPath="c:\python36-32\python.exe"
                      arguments="c:\home\site\wwwroot\runserver.py --port %HTTP_PLATFORM_PORT%"
                      stdoutLogEnabled="true"
                      stdoutLogFile="c:\home\LogFiles\python.log"
                      startupTimeLimit="60"
                      processesPerApplication="16">
            <environmentVariables>
                <environmentVariable name="SERVER_PORT" value="%HTTP_PLATFORM_PORT%"/>
            </environmentVariables>
        </httpPlatform>
    </system.webServer>
</configuration>
```

可以看到，实际最终是以 `runserver.py` 的方式运行的，而在阅读 Django 文档中 [How to deploy Django](https://docs.djangoproject.com/en/5.0/howto/deployment/){:target="_blank"} 可以看到，部署方式是使用了 WSGI 或者 ASGI 与 Web Server 配合使用的。

考虑到需要使用到 [Django Channels](https://channels.readthedocs.io/en/latest/){:target="_blank"} 所以只能选择 ASGI。 那么在 Windows 下可以选择的 ASGI Server 有 [Daphne](https://github.com/django/daphne){:target="_blank"} 、 [Hypercorn](https://pgjones.gitlab.io/hypercorn/){:target="_blank"} 和 [Uvicorn](https://www.uvicorn.org/){:target="_blank"}


## 站点配置

配置项 `processesPerApplication` 可以决定最终启动的 Python 进程数量，可以根据负载和目标性能需求，在内存可以满足的情况下，配置 为 CPU 核心数量 2-4 倍。 

!!! tips

    IIS 新建站点时默认会创建一个应用，可以在`应用程序池`中看到。当有请求到对应站点时，对应应用名作为用户运行的 `w3wp.exe` 进程就会运行，然后根据 `web.config` 配置来拉起 Python 进程。 所以出现文件被占用或者修改配置等不急 IIS 重新拉起新进程时，可以手动结束对应的 `w3wp.exe` 进程。

    IIS 应用进程用户属于 `IIS_IUSRS` 用户组，因此如果代码需要对文件或者文件夹进行读写时，需要通过文件系统的安全配置，设置目录的读写权限，给予 `IIS_IUSRS` 用户组足够的权限。

=== "Daphne"

    ```xml title="web.config"
    <?xml version="1.0" encoding="utf-8"?>
    <configuration>
        <system.webServer>
            <httpPlatform
                    processPath="D:\Python\Scripts\daphne.exe"
                    arguments="-p %HTTP_PLATFORM_PORT% mysite.asgi:application"
                    stdoutLogEnabled="true"
                    stdoutLogFile=".\logs\stdout.log"
                    startupTimeLimit="60"
                    processesPerApplication="32"
            >
                <environmentVariables>
                    <environmentVariable name="DJANGO_SETTINGS_MODULE" value="myiste.settings"/>
                </environmentVariables>
            </httpPlatform>
            <handlers>
                <add name="PythonHandler" path="*" verb="*" modules="httpPlatformHandler" resourceType="Unspecified"/>
            </handlers>
        </system.webServer>
    </configuration>
    ```

=== "Hypercorn"

    ```xml title="web.config"
    <?xml version="1.0" encoding="utf-8"?>
    <configuration>
        <system.webServer>
            <httpPlatform
                    processPath="D:\Python\Scripts\hypercorn.exe"
                    arguments="--workers=1 --bind=127.0.0.1:%HTTP_PLATFORM_PORT% --backlog=1000 mysite.asgi:application"
                    stdoutLogEnabled="true"
                    stdoutLogFile=".\logs\stdout.log"
                    startupTimeLimit="60"
                    processesPerApplication="32"
            >
                <environmentVariables>
                    <environmentVariable name="DJANGO_SETTINGS_MODULE" value="myiste.settings"/>
                </environmentVariables>
            </httpPlatform>
            <handlers>
                <add name="PythonHandler" path="*" verb="*" modules="httpPlatformHandler" resourceType="Unspecified"/>
            </handlers>
        </system.webServer>
    </configuration>
    ```

=== "Uvicorn"

    ```xml title="web.config"
    <?xml version="1.0" encoding="utf-8"?>
    <configuration>
        <system.webServer>
            <httpPlatform 
                    processPath="D:\Python\Scripts\uvicorn.exe"
                    arguments="--port=%HTTP_PLATFORM_PORT% --backlog 256 --timeout-graceful-shutdown 10 --workers=1 mysite.asgi:application" 
                    stdoutLogEnabled="true"
                    stdoutLogFile=".\logs\stdout.log" 
                    startupTimeLimit="60" 
                    processesPerApplication="32"
            >
                <environmentVariables>
                    <environmentVariable name="DJANGO_SETTINGS_MODULE" value="myiste.settings"/>
                </environmentVariables>
            </httpPlatform>
            <handlers>
                <add name="PythonHandler" path="*" verb="*" modules="httpPlatformHandler" resourceType="Unspecified" />
            </handlers>
        </system.webServer>
    </configuration>
    ```

### 性能测试

```python title="mysite/urls.py"
from django.urls import path
from django.http import HttpResponse

def index(request):
    return HttpResponse("Hello, world.")


async def async_index(request):
    return HttpResponse("Hello, aysnc world.")


urlpatterns = [
    # path('admin/', admin.site.urls),
    path("sync/", index, name="index"),
    path("async/", async_index, name="async_index"),
]
```
!!! note

    测试环境为 Windows Server 2019、Python 3.12 与 Django 5.0，服务器硬件配置为 AMD Ryzen 5600G 虚拟机 8 核 16G 内存。结果仅供参考，用于对比不同部署配置方式下的性能差异，不作为平台或框架本身的性能参考。Uvicorn 在 Windows 下会使用 asyncio 替代 uvloop。

关闭了 Admin、DEBUG 和所有中间件后，使用 `wrk -c 64 -t 16 -d 30` 进行测试，测试结果如下:

|   ASGI Server    | 同步视图 - RPS | 同步视图 - 99% 延迟 | 异步视图 - RPS | 异步视图 - 99% 延迟 |
|:----------------:|:----------:|:-------------:|:----------:|:-------------:|
|   Daphne 4.0.0   |    3200    |      65       |    3400    |      65       |
| Hypercorn 0.16.0 |    3200    |      45       |    3500    |      45       |
|  Uvicorn 0.27.0  |    3300    |      45       |    3600    |      45       |

### 静态文件

配置 `STATIC_ROOT` 到对应站点目录下的文件夹，比如 `static` 目录，然后执行 `python manager.py collectstatic`，这样所有静态文件都会被收集到 `static` 目录下。

由于站点配置为 PythonHandler 是处理文件和文件夹的所有请求，所以还需要对 `static` 做一下配置，让站点 `static` 路径的请求是按静态文件资源进行处理。

在 IIS 的站点管理目录下，右键点击 `static` 目录，选择 `转换为应用程序`，然后可以对 `static` 这个应用进行配置，调整 `处理程序映射`，将 `StaticFile` 之外的映射删除即可。在 `static` 目录下的 `web.config` 文件如下:

```xml title="static/web.config"
<?xml version="1.0" encoding="UTF-8"?>
<configuration>
    <system.webServer>
        <handlers>
            <clear />
            <add name="StaticFile" path="*" verb="*" type="" 
                 modules="StaticFileModule,DefaultDocumentModule,DirectoryListingModule" 
                 scriptProcessor="" 
                 resourceType="Either" 
                 requireAccess="Read" 
                 allowPathInfo="false" 
                 preCondition="" 
                 responseBufferLimit="4194304" 
            />
        </handlers>
    </system.webServer>
</configuration>
```

### 前后端分离项目

前后端分离的项目一般会将首页入口在网站根目录下，而后端则在一个子路径下，例如：`/api`。在站点根目录下创建一个 api 目录，把前面的代码保存到该目录下，然后配合 Django 路由配置。

```python title="mysite/urls.py"
from django.contrib import admin
from django.urls import path, include
from django.http import HttpResponse


def index(request):
    return HttpResponse("Hello, world.")

async def async_index(request):
    return HttpResponse("Hello, aysnc world.")
    
urlpatterns = [
    path("admin/", admin.site.urls),
    path("sync/", index, name="index"),
    path("async/", async_index, name="async_index"),
]

urlpatterns = [
     path('api/', include(urlpatterns))
]
```

同样将 api 目录执行 `转换为应用程序`，按照前面的例子修改 `web.config` 即可。