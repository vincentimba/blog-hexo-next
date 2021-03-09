---
title: 基于 LeanCloud 的 Valine 评论服务设置
categories: 
- IT 业余票友
- 建站
tags:
  - Hexo
  - Valine
abbrlink: d5da9e71
date: 2021-02-04 23:26:00
---

如果想在静态网页中加入评论功能，就不得不借助第三方服务的帮助。考察了诸多以后，最终选择了 Valine 来实现。如你所见，每篇文章最后的评论部分正是 Valine 所提供的。它是一款基于 [LeanCloud](https://leancloud.cn/) 的快速、简洁且高效的无后端评论系统。

<!--more-->

如果只是想拥有单纯的评论功能，那么 Valine 配置非常简单，甚至 Hexo 的 NexT 主题还对其做了集成；但如果想进行进一步的管理并克服 LeanCloud 为我们设置的一些限制的的话，还有一些配置需要完成。

本文参考了 [@DesertsP](https://github.com/DesertsP) 的 [Github 说明文档](https://github.com/DesertsP/Valine-Admin) 。

### 基础功能

首先在需要在 `LeanCloud` 注册一个账号。虽然免费用户只能使用开发版，支持有限的请求数量，但对于访问量很小的我来说已经足够。之后创建应用，获取到用户的`APP ID`和`APP Key`，并将其写入 Hexo 的配置文件中。具体的位置有所不同，对于我使用的 NexT 主题，配置文件`_config.yml`位于 `\themes\next\`。

```
# Valine
# For more information: https://valine.js.org, https://github.com/xCss/Valine
valine:
  enable: true # 这里改为 true
  appid: # 填入 appid
  appkey:  # 填入 appkey
  notify: false # Mail notifier
  verify: false # Verification code
  placeholder: '聊天框中默认的语句' 
  avatar: mm # Gravatar style # 头像
  guest_info: nick,mail # Custom comment header # 用户留言需要的登记信息
  pageSize: 10 # Pagination size
  language: # Language, available values: en, zh-cn
  visitor: false # Article reading statistic
  comment_count: true # If false, comment count will only be displayed in post page, not in home page
  recordIP: false # Whether to record the commenter IP
  serverURLs: # When the custom domain name is enabled, fill it in here (it will be detected automatically by default, no need to fill in)
  #post_meta_order: 0
```

```
$ hexo clean ; hexo g ; hexo d
```

重新初始化 Hexo ，留言板已经出现在文章下方。现在评论功能已经可以正常使用了，但如果需要更高的要求，还需要进一步的配置。

### 评论数据管理

进入评论系统数据库所在的 LeanCloud 应用。进入「云引擎-WEB-设置」，设置环境变量以及云引擎域名。其中，我遇到最大问题的就是`ADMIN_URL`这一项，我将重进行说明。

| 变量          | 示例                    | 说明                                                         |
| ------------- | ----------------------- | ------------------------------------------------------------ |
| SITE_NAME     | Deserts                 | [必填]博客名称                                               |
| SITE_URL      | https://panjunwen.com   | [必填]首页地址                                               |
| SMTP_SERVICE  | QQ                      | [新版支持]邮件服务提供商，支持 QQ、163、126、Gmail 以及 [更多](https://nodemailer.com/smtp/well-known/#supported-services) |
| SMTP_USER     | xxxxxx@qq.com           | [必填]SMTP登录用户                                           |
| SMTP_PASS     | ccxxxxxxxxch            | [必填]SMTP登录密码（QQ邮箱需要获取独立密码）                 |
| SENDER_NAME   | Deserts                 | [必填]发件人                                                 |
| SENDER_EMAIL  | xxxxxx@qq.com           | [必填]发件邮箱                                               |
| **ADMIN_URL** | https://xxx.leanapp.cn/ | [建议]Web主机二级域名（云引擎域名），用于自动唤醒            |
| BLOGGER_EMAIL | xxxxx@gmail.com         | [可选]博主通知收件地址，默认使用SENDER_EMAIL                 |
| AKISMET_KEY   | xxxxxxxx                | [可选]Akismet Key 用于垃圾评论检测，设为MANUAL_REVIEW开启人工审核，留空不使用反垃圾 |

`ADMIN_URL`指的管理员管理网页地址，其域名最好使用站点的三级域名，并且对其在 DNS 服务器上进行解析。

对于域名`xxx.example.yyy`，其实其全称可以写为`xxx.example.yyy.root`，其中`.yyy`为顶级域名；`.example.yyy`为二级域名，也就是可以买到或注册的域名；`xxx.example.yyy`为三级域名。如果说购买了一个二级域名，也便会相应获得其对应的三、四级域名。对应的，`CNAME`就是用来解析三级域名的记录。

在这里，我们可以新解析一个 `leanapp.mydomainname.yyy`作为 Valine 的评论管理 web 服务地址。并且，也可以和二级域名一样为其添加 SSL 证书以保证 https 协议的使用。

### 云引擎的设置

> 未完待续