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

### 连续翻页

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



### 小试牛刀

经过了学历，现在开始实战，爬取豆瓣 Top250 的电影名单，网页地址为 [*豆瓣电影 Top 250*](https://movie.douban.com/top250)。

在写爬虫部分之前，我们首先需要定义一个 `DoubanMovie` 类，它从 `Item` 类继承而来，作为存放管理爬取到的数据的工具。打开目录中的 `items.py` 编辑其中的内容：

```python DoubanTop250/DoubanTop250/items.py
import scrapy

class DoubanMovie(scrapy.Item):   
    ranking = scrapy.Field()     # 排名  
    movie_name = scrapy.Field()  # 电影名称   
    score = scrapy.Field()       # 评分    
    score_num = scrapy.Field()   # 评论人数
```

在 Scrapy 的[官方文档](https://scrapy-chs.readthedocs.io/zh_CN/1.0/topics/items.html)中，也对 `Item` 这个类做了精确而简洁的阐述：

> 为了定义常用的输出数据，Scrapy提供了 [`Item`](https://scrapy-chs.readthedocs.io/zh_CN/1.0/topics/items.html#scrapy.item.Item) 类。 [`Item`](https://scrapy-chs.readthedocs.io/zh_CN/1.0/topics/items.html#scrapy.item.Item) 对象是种简单的容器，保存了爬取到得数据。 其提供了 [类似于词典(dictionary-like)](https://docs.python.org/library/stdtypes.html#dict) 的API以及用于声明可用字段的简单语法。

在创造 `Item` 对象（实例）的过程中，需为其赋予 `Item Fields`，如上面的代码所示。

> `Field`对象指明了每个字段的元数据(metadata)。

之后分析网页的 html 代码，编写 Xpath 语句。可以说这是整个爬虫环境最重要的环节之一。

```html https://movie.douban.com/top250
<ol class="grid_view">
        <li>
            <div class="item">
                <div class="pic">
                    <em class="">1</em>
                    <a href="https://movie.douban.com/subject/1292052/">
                        <img width="100" alt="肖申克的救赎" src="https://img2.doubanio.com/view/photo/s_ratio_poster/public/p480747492.webp" class="">
                    </a>
                </div>
                <div class="info">
                    <div class="hd">
                        <a href="https://movie.douban.com/subject/1292052/" class="">
                            <span class="title">肖申克的救赎</span>
                                    <span class="title">&nbsp;/&nbsp;The Shawshank Redemption</span>
                                <span class="other">&nbsp;/&nbsp;月黑高飞(港)  /  刺激1995(台)</span>
                        </a>


                            <span class="playable">[可播放]</span>
                    </div>
                    <div class="bd">
                        <p class="">
                            导演: 弗兰克·德拉邦特 Frank Darabont&nbsp;&nbsp;&nbsp;主演: 蒂姆·罗宾斯 Tim Robbins /...<br>
                            1994&nbsp;/&nbsp;美国&nbsp;/&nbsp;犯罪 剧情
                        </p>

                        
                        <div class="star">
                                <span class="rating5-t"></span>
                                <span class="rating_num" property="v:average">9.7</span>
                                <span property="v:best" content="10.0"></span>
                                <span>2279813人评价</span>
                        </div>

                            <p class="quote">
                                <span class="inq">希望让人自由。</span>
                            </p>
                    </div>
                </div>
            </div>
        </li>
        <li>
     		[...]
        </li>
        <li>
    		[...]
        </li>
        [...]
</ol>
```

可以看到在 `<ol class="grid_view">` 标签下，每一部电影对应一个 `<li>` 标签，需要获取的相关数据也都分布在其中。依此我们写出相应的 Xpath 语句。

```python DoubanTop250/DoubanTop250/spiders/doubanmovie.py
import scrapy
from ..items import DoubanMovie
# 这里我卡了很久，主要还是对 python 上级目录的引用方法不够熟悉


class DoubanmovieSpider(scrapy.Spider):
    name = 'doubanmovie'
    allowed_domains = ['douban.com']
    headerssss = {
        'User-Agent': 'Mozilla/5.0 (Windows NT 6.1; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) '
                      'Chrome/53.0.2785.143 Safari/537.36 ',
        'Accept-Language': 'zh-CN,zh;q=0.9,en;q=0.8,en-GB;q=0.7,en-US;q=0.6',
        'Accept': 'text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,image/apng,*/*;q=0.8,'
                  'application/signed-exchange;v=b3;q=0.9 '
    }

    def start_requests(self):
        url = 'https://movie.douban.com/top250'
        yield scrapy.Request(url, headers=self.headerssss)

    def parse(self, response):
        item = DoubanMovie()  # 这里就是上级目录中引用的类 DoubanMovie 中创造一个实例 item
        movies = response.xpath('//ol[@class="grid_view"]/li')
        # 这里的 Xpath 代码非常关键，它提取了所有下层的 li，从而生成一个迭代器
        for movie in movies:  # 这里的每一个 movie 也就是一个 li 了
            item['ranking'] = movie.xpath(".//div[@class='pic']/em/text()").get()
            item['movie_name'] = movie.xpath(".//div[@class='info']/div/a/span[1]/text()").get()
            item['score'] = movie.xpath(".//div[@class='star']/span[2]/text()").get()
            item['score_num'] = movie.xpath(".//div[@class='star']/span[4]/text()").get()[:-3]
            yield item  # 这里犯了很低级的语法错误，yield 的缩进要注意在 for 语句内部

        next_page = response.xpath("//span[@class='next']/a/@href").get()

        if next_page:
            next_url = response.urljoin(next_page)
            yield scrapy.Request(next_url, headers=self.headerssss)

```

这次的代码中，没有在 `settings.py` 中定义 requests 的 headers，而是在 `DoubanmovieSpider` 的继承过程中定义了 `start_requests()` 方法，并在这个方法中引入了要爬取的 url 和请求头（headers）。

> `start_requests()` 该方法必须返回一个可迭代对象(iterable)。该对象包含了 spider 用于爬取的第一个Request。当 spide r启动爬取并且未制定 URL 时，该方法被调用。 当指定了URL时，`make_requests_from_url()` 将被调用来创建Request对象。 该方法仅仅会被Scrapy调用一次，因此您可以将其实现为生成器。

接下来输入命令启动爬虫。

```
scrapy crawl doubanmovie -o douban.csv
```

这里添加了一个参数 `-o` ，这使得 Scrapy 会在运行结束后对结果进行输出，输出的文件名为 `douban.csv`。

运行结束后，打开生成的 `douban.csv` 文件，我们可以看到完整的 Top250 电影列表和信息。

![image-20210217144225814](https://squidzh-1304890557.cos.ap-nanjing.myqcloud.com/blog_pic_bed/20210217144237.png)

### 参考

- [Scrapy爬虫框架教程（一）-- Scrapy入门](https://zhuanlan.zhihu.com/p/24669128)
- [Python爬虫框架Scrapy入门, 学会可以直接无视80%的网站！](https://www.bilibili.com/video/BV1m441157FY)
- [python 使用scrapy运行爬虫时出现“ModuleNotFoundError: No module named ‘protego‘”](https://blog.csdn.net/qq513536189/article/details/110789420)
- [Anaconda换源](https://blog.csdn.net/li_k_y/article/details/105204253)

