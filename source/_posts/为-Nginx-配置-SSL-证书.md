---
title: 为 Nginx 配置 SSL 证书
categories: IT 业余票友
tags:
  - Nginx
abbrlink: 22bbd643
date: 2021-02-04 14:11:58
---

完成了备案，小站终于可以支棱起来了。现在只差为网站配置 SSL 证书，之后便可以使用高端大气上档次的 https 协议了~

还有什么比折腾更让人快乐呢？

<!--more-->

### SSL 证书的获取

本站使用腾讯云的服务，因此可以直接在腾讯云上申请 SSL 证书。过程很简单，在控制台中找到域名解析的菜单，SSL 点击就送。之后将 SSL 相关配置文件下载下来，解压相关的文件。

找到`Nginx`中的两个文件`1_zhangbh.xyz_bundle.crt`和`2_zhangbh.xyz.key`，将其上传到服务器上。为了方便管理，我在 Nginx 的安装目录中新建了一个`cert`文件夹放置它们。

### Nginx 的配置

Nginx 的安装文件目录一般为`/etc/nginx/`，可以用`tree -d`命令看到其中的目录结构。

```
$ tree -d
.
├── cert/
├── conf.d/
├── fastcgi.conf
├── fastcgi_params
├── koi-utf
├── koi-win
├── mime.types
├── modules-available/
├── modules-enabled/
├── nginx.conf
├── proxy_params
├── scgi_params
├── sites-available/
├── sites-enabled/
├── snippets/
├── uwsgi_params
└── win-utf
```

Nginx 最重要的配置参数都储存在 `nginx.conf`之中。

```
$ cat nginx.conf
http {
        ##
        # Basic Settings
        ##
        include /etc/nginx/conf.d/*.conf;
        include /etc/nginx/sites-enabled/*;
}
```

可以看到在 `http`下还有一层 `server`，它们被`include`到了`/conf.d/*.conf`和`/sites-enabled/*`之中。所以不需要直接将自己的配置直接写入到`nginx.conf`中，以文件进行分来，写入到根目录下面的相关配置文件夹中即可。这样可以更方便的管理不同版本的配置。

在`/sites-enabled/`里新建一个文件 `zhangbh.xyz`，写入`server`的相关配置。Nginx 运行时这里的内容会被上级文件`nginx.conf`导入。

```
# 这里配置 http 访问的转发，将其转发到 https 的 443 接口上
server {
    listen 80;
    server_name zhangbh.xyz;
    rewrite ^(.*)$ https://zhangbh.xyz;
}
# 这里配置 https 访问的文件
server {
    listen 443; # 接口
    server_name zhangbh.xyz; # 网站地址
    ssl on;
    root /var/www/blog; # 网页文件存放目录
    index index.html index.htm;
    ssl_certificate  /etc/nginx/cert/1_zhangbh.xyz_bundle.crt; # SSL 文件
    ssl_certificate_key /etc/nginx/cert/2_zhangbh.xyz.key; # SSL 文件
    ssl_session_timeout 5m;
    ssl_ciphers ECDHE-RSA-AES128-GCM-SHA256:ECDHE:ECDH:AES:HIGH:!NULL:!aNULL:!MD5:!ADH:!RC4;
    ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
    ssl_prefer_server_ciphers on;

    location / {
        index index.html index.htm;
    }
}
```

需要注意的是，因为之前配置过网站的 http 协议，所以需要先将之前在 `/etc/nginx/sites-enabled/default`中配置的文件内容注释掉。

### 一切就绪

现在重启 Nginx 服务。

```
# /etc/init.d/nginx restart
[ ok ] Restarting nginx (via systemctl): nginx.service.
```

输入网址`https://zhangbh.xyz`可以看到，新的`https`协议已经生效了~

<img src="/images/https_demo.png" style="zoom:50%;" />

