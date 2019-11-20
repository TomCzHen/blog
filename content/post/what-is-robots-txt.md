---
title: robots.txt 是什么？
date: 2019-11-18T14:00:00+08:00
categories:
    - Web
tags:
    - robots.txt
    - Python
---

> [robots.txt - 维基百科](https://zh.wikipedia.org/wiki/Robots.txt)  

robots.txt（统一小写）是一种存放于网站根目录下的ASCII编码的文本文件，它通常告诉网络搜索引擎的漫游器（又称网络蜘蛛），此网站中的哪些内容是不应被搜索引擎的漫游器获取的，哪些是可以被漫游器获取的。因为一些系统中的URL是大小写敏感的，所以robots.txt的文件名应统一为小写。robots.txt应放置于网站的根目录下。如果想单独定义搜索引擎的漫游器访问子目录时的行为，那么可以将自定的设置合并到根目录下的robots.txt，或者使用robots元数据（Metadata，又称元数据）。

**robots.txt协议并不是一个规范，只是约定协议，并不能保护网站的隐私。**

<!--more-->

## robots.txt 结构

> 参考资料:  
> [robots.txt 文件简介](https://support.google.com/webmasters/answer/6062608)  
> [The Web Robots Pages](https://www.robotstxt.org/)


`robots.txt` 需要在网站根路径下可以访问，并且只能是小写，通常是存放一个文本文件命名为 `robots.txt`，例如：`https://www.jd.com/robots.txt`。

`robots.txt` 结构也很简单，通过 `User-agent` 针对不同的爬虫设置，使用 `Disallow` 来指定不允许访问的路径。

```
User-agent: spider_name
Disallow: /disallow_uri_path
```

### Robots 扩展标准

> 参考资料:  
> [Robots exclusion standard](https://en.wikipedia.org/wiki/Robots_exclusion_standard#Nonstandard_extensions)

标准的 `robots.txt` 中 `User-agent` 虽然可以使用 `*` 表示所有爬虫，但是不支持通配符或正则（`*` 只是一个特定值），`Disallow` 同样如此，扩展标准对这些不足进行了完善。

**注意：扩展标准是否有效取决于爬虫是否支持扩展配置项目，以及爬虫本身的限制。**

#### `Crawl-delay`

`Crawl-delay` 用于限制爬虫对网站主机的访问频率，设置值定义的是爬虫的请求间隔，单位为秒。

```
User-agent: *
Crawl-delay: 10
```

#### `Allow`

`Allow` 可以定义允许爬虫访问的路径、资源，配合 `Disallow` 可以实现更精准的控制。

```
Allow: /disallow_uri_path/allow.html
Disallow: /disallow_uri_path/
```

#### `Sitemap`

用于设置网站的 sitemap 访问 url。

```
Sitemap: https://www.example.com/sitemap.xml
```

#### `Host`

用于配置可供爬虫访问的镜像站。

#### 通配符 `*` 支持

标准 `robots` 在 `Disallow` 中不支持使用通配符，扩展标准则允许。

### Robots Meta tag

> 参考资料:  
> [漫游器元标记、data-nosnippet 和 X-Robots-Tag 规范](https://developers.google.com/search/reference/robots_meta_tag)  
> [Robots Metatags](https://www.bing.com/webmaster/help/which-robots-metatags-does-bing-support-5198d240)

通过 `robots.txt` 可以允许或拒绝爬虫获取页面信息，但是仍然会将页面索引，同样会出现在搜索引擎中（但是页面内容不会）。可以通过 `<meta>` 标签，或者 `X-Robots-Tag` 响应头的方式，来避免页面被爬虫索引。

#### `<meta>` 标签

```html
<meta name="robots" content="noindex">
```
meta 标签可以针对页面进行设置，是否允许索引等。

##### `data-nosnippet` 属性

针对页面元素的设置则需要使用 `data-nosnippet` 实现，支持在 `<span>` `<div>` `<section>` 块中定义。但是，获取 `data-nosnippet` 配置的前提是该页面允许爬虫访问。

```html
<p>This text can be shown in a snippet
 <span data-nosnippet>and this part would not be shown</span>.</p>

<div data-nosnippet>not in snippet</div>
<div data-nosnippet="true">also not in snippet</div>

<div data-nosnippet>some text</html>
<!-- unclosed "div" will include all content afterwards -->

<mytag data-nosnippet>some text</mytag>
<!-- NOT VALID: not a span, div, or section -->
```

#### `X-Robots-Tag` 响应头

通过在页面响应头中添加 `X-Robots-Tag` 可以达到 meta 标签一样效果。

```
X-Robots-Tag: noindex
```

通过 Web Server 添加响应头的功能可以轻松配置：

```
# Nginx
location ~* \.pdf$ {
  add_header X-Robots-Tag "noindex, nofollow";
}

location ~* \.(png|jpe?g|gif)$ {
  add_header X-Robots-Tag "noindex";
}
```

### 解析 `robots.txt`

使用 Python 标准库 [urllib.robotparser — Parser for robots.txt](https://docs.python.org/3.7/library/urllib.robotparser.html) 可以解析标准协议格式的 `robots.txt`，如果有扩展协议的需求则可以考虑使用 [Robots Exclusion Protocol Parser for Python](https://github.com/seomoz/reppy)。

```python
from urllib.robotparser import RobotFileParser

rfp = RobotFileParser(url='https://www.jd.com/robots.txt')
rfp.read()
if rfp.rfp.can_fetch('HuihuiSpider',"/"):
    print('ok')
else:
    print('no')
```

## `robots.txt` 的局限性

> 参考资料：
> [How to Detect and Verify Search Engine Crawlers](https://www.onely.com/blog/detect-verify-crawlers/)  
> [验证 Googlebot](https://support.google.com/webmasters/answer/80553)

显然光靠一个 `.txt` 文本是无法真正阻止爬虫的，遵循 `robots.txt` 协议完全靠爬虫的“自觉”，因此还需要一些 “`.exe`” 的方式来阻止爬虫。

### 技术限制

在 Web Server 配置为根据阻止指定的 `User-agent` 可以阻止爬虫的访问，首先需要获取搜索引擎爬虫的 `User-agent` 值。

Google 爬虫可以在 [Google 抓取工具（用户代理）概览](https://support.google.com/webmasters/answer/1061943) 查看；百度爬虫可以在 [Baiduspider常见问题解答](https://help.baidu.com/question?prod_id=99&class=476&id=2996) 查看。

以 Nginx 为例，阻挡所有百度爬虫的访问：

```
if ($http_user_agent ~* (Baiduspider) ) {
    return 403;
}
```

由于 `User-Agent` 是可以被假冒的，因此需要对访问 IP 进行验证。

可以通过分析 Web Server 的日志来整理使用了爬虫 `User-Agent` 的来源 IP，在通过 `host` 命令对 IP 进行验证，或者通过搜索引擎提供的验证工具进行验证，比如 [必应 Bing - 验证 Bingbot 工具](https://www.bing.com/toolbox/verify-bingbot)

在 Nginx 上则可以使用以下两种方式：

* [Nginx HTTP rDNS module](https://github.com/flant/nginx-http-rdns)
* [Nginx Ultimate Bad Bot](https://github.com/mitchellkrogza/nginx-ultimate-bad-bot-blocker) 


Nginx HTTP rDNS module 是基于 Nginx module 的方式，通过验证声明为爬虫的 IP 反查 Host 判断，Nginx Ultimate Bad Bot 则是基于自动脚本，添加更新 Nginx 配置的方式，另外还支持基于防火墙层级的阻断。

但是，爬虫可以通过伪装为正常浏览器 `User-Agent` 的方式突破这些限制，对于这种方式的爬虫，只能选择基于行为去分析、限制，或者依赖前端反爬方案。

### 法律意义

*注：以下内容为个人解读，不代表专业分析或法律咨询。*

从国内相关爬虫案例看，是否遵循 `robots.txt` 并非关键。

爬取、保存公民个人隐私数据存在极高风险，即便 `robots.txt` 并未 `Disallow`，开发爬虫获取此信息或提供第三方使用，一旦牵扯到犯罪行为，开发者必然会被连带。

当爬取内容涉及版权、商业信息时，情况则更复杂一些。

以刑事相关计算机信息安全法规条款看，当爬虫涉及到破解、逆向以突破目标网站限制时（包括但不限于频率、权限、验证码等）。一旦使用这些获取的版权、商业信息引起纠纷、诉讼，有被认定为破坏网络安全的风险存在，最终结果如何完全取决于公司的能力。

最后，即便是遵循 `robots.txt`，爬取的也是公开数据，但是因为爬取行为造成目标主机故障的情况，也有被判定为网络攻击的风险，涉及刑事犯罪。