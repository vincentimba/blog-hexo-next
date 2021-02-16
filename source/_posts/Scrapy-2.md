---
title: Scrapy 的学习与实践（二）——初窥门径
categories:
  - IT 业余票友
  - Python
tags:
  - Python
  - 爬虫
abbrlink: 7e9e6e60
date: 2021-02-16 21:13:12
---

之前完成了框架的搭建，现在以爬取自己的网站为例初步学习 Scrapy 的使用。

<!--more-->

### 创建项目

首先创建一个新的 Scrapy 项目。在创建好的目录下运行命令，创建一个用来爬取自己博客的命令。

```bash
scrapy startproject zhangbhSpider
```

该命令创建的目录结构如下：

```
zhangbhSpider/
    scrapy.cfg
    scrapyspider/
        __init__.py
        items.py
        pipelines.py
        settings.py
        spiders/
            __init__.py
            ...
```

这些文件分别是：

> scrapy.cfg: 项目的配置文件。
>
> scrapyspider/: 该项目的python模块。之后您将在此加入代码。
>
> scrapyspider/items.py: 项目中的item文件。
>
> scrapyspider/pipelines.py: 项目中的pipelines文件。
>
> scrapyspider/settings.py: 项目的设置文件。
>
> scrapyspider/spiders/: 放置spider代码的目录。



### 编写爬虫

接下来打开上一个命令生成的目录 `zhangbhSpider/` ，运行命令：

```
C:py_project\scrapy\1-littletest\zhangbhSpider>scrapy genspider zhangbh zhangbh.com
# scrapy genspider [爬虫名] [域名]
```

要注意，这里的爬虫名不能和上面的目录名重名。之后会在 `\zhangbhSpider\zhangbhSpider\spiders\` 下产生一个新文件 `zhangbh.py` ，即名为 `zhangbh` 的爬虫的配置文件，里面已经根据上面的命令自动填写好了一些预设内容，如下所示：

```python zhangbhSpider/zhangbhSpider/spiders/zhangbh.py
import scrapy

class ZhangbhSpider(scrapy.Spider):  # 创建了一个类 ZhangbhSpider，从 scrapy.Spider 继承而来
    name = 'zhangbh'  # 爬虫名称
    allowed_domains = ['zhangbh.com']  # 域名，非必须
    start_urls = ['http://zhangbh.com/']  # 要爬取的网页地址

    def parse(self, response):  # 对于网站的解析方法，这里的 response 指获取到的网页源代码
        pass
    
```

下面引用的内容是对上述的参数的详细介绍。

> Spider是用户编写用于从单个网站(或者一些网站)爬取数据的类。
>
> 其包含了一个用于下载的初始URL，如何跟进网页中的链接以及如何分析页面中的内容， 提取生成 item 的方法。
>
> 为了创建一个Spider，您必须继承 scrapy.Spider 类， 且定义以下三个属性:
>
> - name: 用于区别Spider。 该名字必须是唯一的，您不可以为不同的Spider设定相同的名字。
> - start_urls: 包含了Spider在启动时进行爬取的url列表。 因此，第一个被获取到的页面将是其中之一。 后续的URL则从初始的URL获取到的数据中提取。
> - parse() 是spider的一个方法。 被调用时，每个初始URL完成下载后生成的 Response 对象将会作为唯一的参数传递给该函数。 该方法负责解析返回的数据(response data)，提取数据(生成item)以及生成需要进一步处理的URL的 Request 对象。



### 配置并运行爬虫

对于数据的具体处理这里使用 Xpath 表达式，可以参照[XPath 语法](https://www.w3school.com.cn/xpath/xpath_syntax.asp)。这里以我自己的博客为例，获取网站首页所有文章的 url 和标题。代码如下：

```python zhangbhSpider/zhangbhSpider/spiders/zhangbh.py
def parse(self, response):  # 对于网站的解析方法，这里的 response 指获取到的网页源代码
    selectors = response.xpath('//article/header/h2/a')
    for slector in selectors:
        url = slector.xpath('./@href')
        post_title = slector.xpath('./text()')
        print(url.get(), post_title.get())
        # for 语句得到的是名为 Slector 的对象
        # 所以需要在后面加上 .get() 方法来获取其中的元素
        # 否则得到的就是 <Selector xpath='./@href' data='/post/d5da9e71/'> ，即对象的描述
```

在目录 `zhangbhSpider/` 输入命令运行一下试试看。

```
scrapy crawl zhangbh
```

果然没有让我失望，第一次运行就报错了…

```
Unhandled error in Deferred:
2021-02-16 21:57:13 [twisted] CRITICAL: Unhandled error in Deferred:
[……]
No module named 'protego'
```

原来是缺少了一个库 `protego` ，在 Anaconda 中安装即可解决问题，`conda install protego`。再次运行，又报错了…

```
DEBUG: Retrying <GET http://zhangbh.com/robots.txt> (failed 1 times): DNS lookup
failed: no results for hostname lookup: zhangbh.com.
```

这里表示没有找到网站的 `robots.txt` 文件。确实我的网站没有设置这个文件，我们在配置文件中将其忽略。

```python zhangbhSpider/zhangbhSpider/settings.py
# Obey robots.txt rules
ROBOTSTXT_OBEY = False  # 将这里的 True 改为 False
```

另外再设置一下请求头。由于我的个人博客比较简陋，所以即便使用空请求头也可以顺利得到网页的源码。但对于正常的网站来说，设置请求头是必须的，这也是最基本的应对反爬的方法。

还是再同一个配置文件中，搜索 `headers` 找到代码位置，取消掉注释并添加一条。

```yml zhangbhSpider/zhangbhSpider/settings.py
DEFAULT_REQUEST_HEADERS = {
  'Accept': 'text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8',
  'Accept-Language': 'en',
  'User-Agent': 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) '
                'Chrome/88.0.4324.150 Safari/537.36 Edg/88.0.705.68',
}
```

再次运行爬虫。

```
[……]
/post/68bb9efb/ 对本站的一些优化
/post/27be0ddf/ 为树莓派配置 Aria2 和 Samba
/post/d5da9e71/ 基于 LeanCloud 的 Valine 评论服务设置
/post/22bbd643/ 为 Nginx 配置 SSL 证书
/post/4f523815/ 一些学习笔记（二）
[……]
```

非常好！得到了想要的数据。

### 连续翻页的设置

使用上述的代码的确可以得到首页的信息，即文章的链接和目录。但我的博客有多页，如何可以一页一页地将整个网站彻底遍历呢？

```
https://zhangbh.xyz/page/2/
```

上面是网站第二页的 url 地址，可以看到最后一个数字 2 指定了页面显示的博客页数。所以我们可以提前生成所有页的网页链接，接着分别对其爬取。但这样有一个问题，如果网页的页数是不固定的，或者是实时变化的，这样的方法无疑就显得过于繁复，因为每次运行前都要先手动查看网页的目录页数再将其配置到文件中。

为了解决这个问题，可以让爬虫自动获得下一页的网页 url ，一页一页地进行爬取。修改 `parse` 函数如下：

```python zhangbhSpider/zhangbhSpider/spiders/zhangbh.py
def parse(self, response):  # 对于网站的解析方法，这里的 response 指获取到的网页源代码
    selectors = response.xpath('//article/header/h2/a')
    for slector in selectors:
        url = slector.xpath('./@href')
        post_title = slector.xpath('./text()')
        print(url.get(), post_title.get())
        next_page = response.xpath("//a[@class='extend next']/@href").get()
    # 提取到每一页中下一页的 url
    if next_page:
        next_url = response.urljoin(next_page)
        # 对获取到的下一页 url 进行网址拼接
        yield scrapy.Request(next_url, callback=self.parse)
        # 对 next_url 进行请求
        # 利用回调函数，将得到的 response 再返回给自己来处理
```



[参考文章](https://www.zhihu.com/people/zhang-yu-ge-71)  [参考视频](https://www.bilibili.com/video/BV1m441157FY)