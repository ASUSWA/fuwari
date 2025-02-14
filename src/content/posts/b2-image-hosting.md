---
title: b2-image-hosting
published: 2022-12-23
description: '简单记录如何将 B2 当作图床使用'
image: 'https://blog-img-b2.asuswa.top/img/2c7c747dd9816c14d90dfde758f68250.webp'
tags: [对象存储, 图床, S3, B2]
category: '笔记'
draft: false 
---

# 前言

Backblaze B2 是一种 IaaS（Infrastructure as a Service，基础设施即服务），与 Amazon S3，Google Cloud Storage 等对象存储服务运作方式相似，且兼容 S3 API。

B2 提供一定的免费额度，包括了 10 GB 永久免费的存储容量、每日 1 GB 的下载流量，以及每日 2500 次的 B 类操作（与数据下载有关）。由于 Backblaze 加入了 Cloudflare 的 Bandwidth Alliance（带宽联盟），故 Cloudflare CDN 到源站的回源流量不计入 B2 的下载流量，可以极大地减少流量消耗。

与 Cloudflare R2 不同， 获取 Backblaze B2 的免费额度不需要绑定支付信息，注册一个 Backblaze 账号即可。

**Backblaze B2 Storge Free Level**

> ***Storage***
> 
> - The first 10GB of storage is free of charge.
> - Above 10GB, we charge $0.005/GB/month, around a quarter of the cost of other leading cloud object stores (cough, S3, cough).
> - Storage cost is calculated hourly, with no minimum retention requirement, and billed monthly.
> 
> ***Downloaded Data***
> 
> - The first 1GB of data downloaded each day is free.
> - Above 1GB, we charge $0.01/GB, but…
> - Downloads through our CDN and compute partners, of which Cloudflare is one, are free.
> 
> ***Transactions***
> 
> - Each download operation counts as one class B transaction.
> - The first 2,500 class B transactions each day are free.
>   Beyond 2,500 class B transactions, they are charged at a rate of $0.004 per 10,000.

# 配置 Backblade B2 Storage

## 创建 Bucket（存储桶）

登录到 [Backblaze B2](https://www.backblaze.com/b2/cloud-storage.html)

创建一个 Bucket

![Create Bucket](https://blog-img-b2.asuswa.top/img/058e40061810a511304babe26a7d9ad1.png#vwid=555&vhei=758)

为了尽可能避免盗链，Bucket Unique Name 应设为他人难以猜到的字符串

Files in Bucket are 设为 `Public`，设为 `Private` 时桶内对象无法公开访问

余下两项为安全相关设置，本次保持默认选项即可

## 上传第一个文件

创建 Bucket 后，上传任意一个文件至该 Bucket，以获取该 Bucket 的主机名

查看文件的详细信息

![Check File Detail](https://blog-img-b2.asuswa.top/img/83ef27541c18de365adaf8735a745644.webp#vwid=691&vhei=59)

可在 Friendly URL 项获取该 Bucket 的主机名，记录下来备用

![Get Bucket Hostname](https://blog-img-b2.asuswa.top/img/ffa461e8719a59f99cee9412b3336b2c.webp#vwid=300&vhei=32)

## 修改 Bucket Settings

点击 Bucket Settings，在 Bucket Info 项中填入 `{"cache-control":"max-age=86400"}`

即 86400 秒（24 小时）内 CDN 不会返回源站重新获取信息，避免无法命中缓存导致回源次数过多，影响访问速度

![Bucket Settings](https://blog-img-b2.asuswa.top/img/a1810b50bdd3a7a8eb5488e746b18f02.webp#vwid=594&vhei=679)

## 配置 CORS Rules

CORS Rules 设置为 `Share everything in this bucket with all HTTPS origins` 以便在不同的站点访问文件（仅限 Https）

第二项设置为 `Both`，将该规则应用于两种 API

![CORS Rules](https://blog-img-b2.asuswa.top/img/0ff816a20588761142d0b25710e41649.webp#vwid=517&vhei=616)

## 添加 App Key

在 B2 Storage 控制台左侧选择 App Keys，添加一个新的 Application Key 用于后面配置 PicGo

![Create App Key](https://blog-img-b2.asuswa.top/img/b76914d92c62e7b5af5abf1b84e3aba6.webp#vwid=581&vhei=590)

App Key 在退出该界面时将被隐藏并且无法再被查询，需将 App Key 保存好备用

# 配置 Cloudflare

## 为 Bucket 设置子域名

登录 [Cloudflare](https://www.cloudflare.com)

***(Optional) 这次使用的域名接入了萌精灵的 Cloudflare Partner，且该域名 DNS 在 DNSPod 进行管理，所以先登陆[萌精灵 Cloudflare Partner 面板](https://cdn.moeelf.com/)***

***添加一条 CNAME 记录，主机名自定义，记录值为上文取得 Bucket Hostname，并启用 Proxy***

![Setting CNAME on Cloudflare Partner Panel](https://blog-img-b2.asuswa.top/img/3c8f41bb12c4a19f7458097755a325a7.webp#vwid=900&vhei=68)

***复制 CNAME Setup 内对应记录的 CNAME，登录 [DNSPod](https://www.dnspod.cn/)，添加一条 CNAME 记录，主机名沿用，记录值为上述 CNAME***

![Setting CNAME on DNSPod](https://blog-img-b2.asuswa.top/img/c69a2ca93976f4ff2553f590e97c85dc.webp#vwid=1278&vhei=65)

若不使用 Cloudflare Partner，则直接在 Cloudflare 面板中添加 CNAME 记录

此时可以使用该子域名访问 bucket 中的对象：`https://<hostname>/file/<bucke tname>/<filename>`

## 配置 URL Rewrite Rules

直接使用刚刚配置好的子域名会暴露 bucket 信息，这里用 Cloudflare Rules 设定重写规则来隐藏它们

![Rewrite Rules](https://blog-img-b2.asuswa.top/img/b2storage-rewrite-rules-7bf9301698d8f6f94212e84a752425f5.webp#vwid=1190&vhei=1255)

从左侧面板进入 Rules / Transform Rules，并创建一个 Transform Rule，编辑表达式（Edit expression）：

`http.host eq "https://<hostname>" and not http.request.full_uri contains "https://<hostname>/file/<bucketname>/"`

该表达式匹配一个主机名为 `<hostname>` 且 url 中不含 `https://<hostname>/file/<bucketname>/` 的请求，第二部分表达式避免了重复匹配被重写之后的 url

因为每个不同的对象对应了不同的文件名，每次的请求也就不相同，所以在 Then… Rewrite 中选择 `Dynamic`，并填入 `concat("/file/<bucketname>", http.request.uri.path)`

## 配置 Modify Response Header Rules

某些响应头会暴露 bucket 信息，配置一个规则来去除它

创建一个 Modify Response Header，并在填入该表达式：`http.host eq "https:/<hostname>/"`

随后，在下方设置需要去除的 headers

- `Expires`
- `X-Bz-Upload-Timestamp`
- `x-bz-content-sha1`
- `x-bz-file-id`
- `x-bz-file-name`
- `x-bz-info-src_last_modified_millis`

![Modify Response Header Rules](https://blog-img-b2.asuswa.top/img/b2storage-modify-response-header-rules-69bb5a9ddaa6c9b0d374b08cb14ec56a.webp#vwid=1161&vhei=1295)

之后可以使用

```bash
$ curl --head URL
```

来验证是否起效

## （Optional）配置防盗链规则

本次配置的图床仅用于个人博客，使用 WAF 来实现防盗链功能

从左侧面板进入 Security / WAF 页面，添加一条 WAF 规则

![Anti-Leech](https://blog-img-b2.asuswa.top/img/b2storage-anti-leech-8a0e5f64b6c551c2d265c1641347b234.webp#vwid=912&vhei=666)

填入表达式：`(not http.referer contains "<blog_hostname>" and http.host eq "hostname")`

该表达式匹配来自博客之外的请求

随后设置行为为 `Block`

## （Optional）配置定时缓存规则

****未验证该方案有效性***

从左侧面板进入 Rules 页面，添加一条 Page Rule

![Auto Cache](https://blog-img-b2.asuswa.top/img/b2storage-auto-cache-24488691c1789b0020421e399d6ce2de.webp#vwid=815&vhei=584)

在 `URL` 中填入 `<hostname>/*`

并按如图设置缓存策略

理想情况下，Cloudflare 会每个月缓存源站所有文件

# 使用 PicGo 上传图片

PicGo 是一款快速上传图片并获取图片 URL 的工具，支持多种 URL 方式，支持多个图床平台以及 Markdown、HTML、URL、UBB 和 Custom 五种格式的链接

## 获取 PicGo

```powershell
#Windows
scoop install picgo
```

```bash
#AUR
$ yay -S picgo-appimage
```

## 配置图床

在 PicGo 左侧选择 `插件设置` 搜索 S3 插件并安装（需要 Node.js）

左侧进入 Amazon S3 图床设置，输入 [#配置 Backblade B2 Storage](#toc_4) 章节中生成的密钥，并根据 bucket 信息填入 bucket name, path, endpoint

> 注意，B2 仅允许 https 请求，故 endpoint 应以 `https` 开头

![PicGo S3 Settings](https://blog-img-b2.asuswa.top/img/b2storage-picgo-s3-settings-369b5a02b09c48e75b40f709913bf4c1.webp#vwid=795&vhei=445)

[文件路径支持 payload](https://github.com/wayjam/picgo-plugin-s3 "picgo-plugin-s3")，按需设置

> | payload      | 描述            |
> | ------------ | ------------- |
> | `{year}`     | 当前日期 - 年      |
> | `{month}`    | 当前日期 - 月      |
> | `{day}`      | 当前日期 - 日      |
> | `{fullName}` | 完整文件名（含扩展名）   |
> | `{fileName}` | 文件名（不含扩展名）    |
> | `{extName}`  | 扩展名（不含`.`）    |
> | `{md5}`      | 图片 MD5 计算值    |
> | `{sha1}`     | 图片 SHA1 计算值   |
> | `{sha256}`   | 图片 SHA256 计算值 |

确定保存后即可使用

