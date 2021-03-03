---
title: Scrapy 的学习与实践（三）——抓取图片
categories:
  - IT 业余票友
  - Python
tags:
  - Python
  - 爬虫
abbrlink: fdc64a3b
date: 2021-02-23 23:47:48
---

想要使用 Scrapy 爬取图片，可以借助其内部的 Item Pipeline（项目管道）来实现。但在默认的配置下，无法在图片的存储过程中进行详细的定制化，这就要求我们对于 Scrapy 中一些封装类的方法进行重写。

<!--more-->

这次的~~受害者~~目标网站为[动物世界](http://www.iltaw.com/animal/all)。

http://www.iltaw.com/animal/all

可以看出这个网站已经较长时间没有维护，反爬的设置也很少，网页以静态为主，正好适合爬虫的练习。

这次的目的就是爬取网站上所有动物的照片和介绍文字，并将其按名字分别存储到各自的文件夹里。

### 创建项目&爬虫文件

在创建好的项目文件夹中使用执行命令：

```
# 创建项目 animalSpider
scrapy startproject animalSpider
cd animalSpider
# 创建爬虫 animal
scrapy genspider aninmal iltaw.com
# 创建存放图片的目录
mkdir animal_imgs
```

这里除了创建爬虫外，还建立了文件夹 `animal_imgs` ，用来放置接下来爬取的图片等内容。创建好的文件结构如下所示：

<img src="https://squidzh-1304890557.cos.ap-nanjing.myqcloud.com/blog_pic_bed/20210224202255.png" alt="image-20210224202241146" style="zoom:67%;" />



接下来就正式开始写代码。

### 编写项目文件 item.py

首先需要定义爬虫的 `Item` 类，关于 `Item` 的说明可以参考[我的另一篇文章](/post/7e9e6e60/#%E5%B0%8F%E8%AF%95%E7%89%9B%E5%88%80)。打开文件 `items.py`，输入以下代码：

```python items.py
import scrapy

class AnimalspiderItem(scrapy.Item):
    # 动物的名字
    name = scrapy.Field()
    # 动物图片的下载链接
    img_url = scrapy.Field()
    # 动物的介绍文字
    animal_info = scrapy.Field()
```

需要注意的是，为了让 Scrapy 可以正确提取到 Item 中的图片 url ，我们必须将存放图片下载链接的键名设置为 `image_urls` 。但在上面设置的值并不是 `image_urls` ， 而是 `img_url` ，这是因为在后续 `setting.py` 文件的设置中重新定义了这个键名。

### 编写爬虫文件 animal.py

`animal.py` 即爬虫的配置文件，是整个爬虫项目中最核心的配置文件。

首先编写爬虫文件的第一部分：

```python animal.py
import scrapy
from ..items import AnimalspiderItem
import re

class AninmalSpider(scrapy.Spider):
    name = 'animal'
    allowed_domains = ['www.iltaw.com']
    
    def start_requests(self):
        url = 'http://www.iltaw.com/animal/all?page='
        page_num = 1
        page_url = url + str(page_num)
        yield scrapy.Request(page_url,
                             callback=self.parse,
                             meta={'page_num': page_num})
	"""
	在代码调试过程中，出现了不能正确回调函数的问题
	在排查了 allowed_domains 的设置和过滤开关的设置后
	最终确定是 Request 优先级的问题
	Request 的优先级用 priority 的数字参数来定义
	数字越大，优先级越高，例如下面一行：
	scrapy.Request(page_url, callback=self.parse, priority=10)
	但我这里没有使用这个方法，而是选择更改整个爬虫的结构来从根本上改变优先级的问题
	"""
```

在方法 `self.start_requests` 中，构造了需要爬取的第一个页面的 url。值得注意的是，这里定义了一个参数 `page_num` ，它用来定义爬取网页的页数，并相应地辅助构造每一页的 url 。

那么如何将 `page_num` 传递到下一层的方法中呢？这里使用到了 Request 中的 `meta` 参数。在发起 Request 时，我们可以构造一个字典，并将需要传递的内容导入字典的值，之后将整个字典赋给 `meta` 。在下一层通过回调函数处理 response 时，就可以直接使用 `response.meta` 来获得这个字典中的内容。这个方法真的太巧妙了！

接下来编写 `animal.py` 的第二部分，这一部分用来获取每一页目录上各个动物页面的 url。

```python animal.py
    def parse(self, response):
        # 先从 Request 中的 meta 字典中取出相应的页数
        page_num = response.meta['page_num']
        # 获取这一页内的动物的链接，并将其调入 parse_animal 方法中处理
        print('#####开始处理第{}页的内容#####'.format(page_num))
        print('#######[{}]#######'.format(response.url))
        animal_urls = response.xpath("//li[@class='clearfix']/div[1]/a/@href").extract()
        for url in animal_urls:
            print(url)
            yield scrapy.Request(url=url, callback=self.parse_animal)
        # 处理完毕后，定义下一页的 url
        next_page_num = page_num + 1
        if next_page_num <= 80:
            next_page_url = 'http://www.iltaw.com/animal/all?page=' + str(next_page_num)
            yield scrapy.Request(next_page_url,
                                 callback=self.parse,
                                 meta={'page_num': next_page_num})
```

这个方法结合了递归的思想，在函数的最后又调用了它本身，从而做到周而复始，一页一页地将整个目录全部爬取到。用来计数的参数就是前一个方法中传递来的 `meta` 字典中的 `page_num`，每爬取一个页面，便将它的值增加 1 ，确定它不大于 80 后（网站目录一共有 80 页），再将它又一次导入  `meta` 中为下一次 Request 所用，如同俄罗斯套娃一般。

现在是 `animal.py` 的最后一部分，这里编写的方法用来在每一个动物自己的页面中提取需要的各项信息并进行封装，供 Item Pipeline 使用。

```python animal.py
    def parse_animal(self, response):
        # 提取需要的各项信息
        name = response.xpath("/html/body/div[1]/div/div[2]/div/div[2]/h3/text()").get()
        img_url = response.xpath('//div[@class="img"]/img/@data-url').get()
        # 这里的图片 url 信息也可以使用了正则表达式来抓取，如下
        # img_url = re.findall('"bdPic":"(.*?)","bdStyle', response.body.decode('utf-8'), re.S)[0]
        # 用 join 函数将分散的文字连接到一起
        animal_info = "".join(response.xpath('/html/body/div[1]/div/div[4]/div/div[2]/text()').extract())
        # 调用自己继承创建的 Item，并将相应的内容导入
        animal_item = AnimalspiderItem()
        animal_item['name'] = name
        animal_item['img_url'] = img_url
        animal_item['animal_info'] = animal_info.strip()
        # 将封装好信息的实例抛出
        yield animal_item
```

根据我自己的理解，在 Scrapy 中，只要通过 `yield` 函数将封装好的 `Item` 实例抛出，Item Pipeline 就可以自己将其拾取，根据 `pipelines.py` 中的配置解析它，最终实现文件或者图片或者其他信息的处理。这一点应该是使用 Scrapy 过程中较难理解的地方了。

### 编写管道文件 pipelines.py

Scrapy 中管道模块接收到的信息都是由 spider 传递过来的。对于简单的使用，我们无需对这个文件做任何设置，只需要简单地从 `object` 继承出一个类，并在 `setting.py` 中将其设置为开启，Scrapy 便可自动的进行工作。但为了更加定制化的对要爬取的内容进行处理，我们就需要重写一些方法。

对于一个 item pipeline 类，最少需要存在一个方法 `self.process_item`，这个方法在每次 spider 抛出 `Item` 时都会被调用。当然也有一些其他的方法可以重写，从而提供更多额外的功能，这里按过不表，具体可以查看 Scrapy 的源码或[官方文档](https://docs.scrapy.org/en/latest/topics/item-pipeline.html)的相关章节。

```python pipelines.py
import os
from scrapy.pipelines.images import ImagesPipeline
from scrapy.spiders import Request


class AnimalSpiderPipeline(object):
    # 先创建一个用来生成目录的方法
    def create_dir(self, path):
        # 去除首尾的空格
        path = path.strip()
        # 去除尾部 \ 符号
        path = path.rstrip("\\")
        # 判断路径是否存在
        isExists = os.path.exists(path)
        if not isExists:
            # 如果不存在则创建目录
            # 创建目录操作函数
            os.makedirs(path)
            print(path + ' 创建成功')
            return True
        else:
            # 如果目录存在则不创建，并提示目录已存在
            print(path + ' 目录已存在')
            return False

    def process_item(self, item, spider):
        """
        这个方法将调用上面创建的方法来创建目录，
        并将不同的动物介绍信息保存到不同的文件夹中
        写法要参考爬虫文件中存入 item 时使用的名称
        """
        # 定义存放照片的文件夹名称
        animal_img_path = './animal_imgs/' + item['name']
        # 存放动物信息的 txt 文件的目录位置
        animal_txt = animal_img_path + "/" + item['name'] + '.txt'
        # 调用上面配置的方法来创建目录
        self.create_dir(animal_img_path)
        # 写入各个动物的介绍信息
        with open(animal_txt, 'wb') as f:
            f.write((item['name'] + item['animal_info'] + '\n').encode('utf-8'))
        return item
    
    
class ImagesSpiderPipeline(ImagesPipeline):
    
    def get_media_requests(self, item, info):
        # 这个方法用来获取图片的 url，并通过 Request 方法来保存图片
        # 这里将 item 加入到 Request 的 meta 字典中，继续传入下一个方法
        # 需要注意的是，在源代码中这里返回的时一个列表，所以这里模仿源代码也使用列表
        return [Request(item['img_url'], meta={'item': item})]

    def file_path(self, request, response=None, info=None, *, item=None):
        # 先从 Request 的 meta 字典中取出图片的信息
        item = request.meta['item']
        # 根据取出的信息定义好图片保存的路径
        path = "./" + item["name"]
        img_name = item["name"]
        img_path = path + '/' + img_name + '.jpg'
        print('######将使用路径 {} 保存图片##########'.format(img_path))
        # 这个方法最终返回的是一个字符串格式的路径
        return img_path
```

在类 `AnimalSpiderPipeline` 的继承中，除了重写 `self.create_dir` 方法，还额外添加了一个 `self.create_dir` 方法。后者根据爬取的动物信息创建存储照片和 txt 文件的文件夹，并在前者中被调用。

在类 `ImagesSpiderPipeline` 的继承中，重写了 `self.get_media_requests` 和 `self.file_path` 两个方法。`self.get_media_requests` 是用来下载图片的方法，它会提取自己接收到的 `Item` 实例中的待下载图片 url ，接着在输出中对这个 url 中发起 Request。

既然这里存在着 Request 请求，我们就可以和上面 `animal.py` 中一样，利用 `meta` 来传递信息。在上面的代码中，这里将整个 item 作为 `meta` 的值传递到了下一层 `self.file_path` 方法中。`self.file_path` 方法得到请求的结果后，即可从  `request.meta` 取出之前注入的 item ，再具体的根据 item 中的信息生成最终图片储存的路径 `img_path`，最终将其返回（`return`）。

这里再附上提到的两个方法的源代码，可以更好地帮助自己模仿构建。

```python site-packages\scrapy\pipelines\images.py
def get_media_requests(self, item, info):
    urls = ItemAdapter(item).get(self.images_urls_field, [])
    return [Request(u) for u in urls]

def file_path(self, request, response=None, info=None, *, item=None):
    image_guid = hashlib.sha1(to_bytes(request.url)).hexdigest()
    return f'full/{image_guid}.jpg'
```

### 编写配置文件 settings.py

前面的相关文件配置好后，最后一步就是检查设置文件 `settings.py` ，这里只罗列出了变动的部分。

```python settings.py
BOT_NAME = 'animalSpider'
SPIDER_MODULES = ['animalSpider.spiders']
NEWSPIDER_MODULE = 'animalSpider.spiders'
# 是否遵循爬虫的 robot.txt 协议
ROBOTSTXT_OBEY = False
# 下载的延迟
DOWNLOAD_DELAY = 2
# 自定义请求标头
DEFAULT_REQUEST_HEADERS = {
  'Accept': 'text/html,application/xhtml+xml'
  'Accept-Language': 'zh-CN,zh;q=0.9,en;q=0.8,en-GB;q=0.7',
  'User-Agent': 'Mozilla/5.0 (Windows NT 10.0; Win64; x64)'
}
# 需要使用的 Item Pipelines，和 pipelines.py 中的类名称对应
# 后面的数字代表管道的优先级，越小的数字代表越高的优先级
# 由于需要先创建好文件夹，再进行文件和图片的储存，所以两个通道的数字需要满足：
# AnimalspiderPipeline < ImagesspiderPipeline
ITEM_PIPELINES = {
   'animalSpider.pipelines.AnimalspiderPipeline': 100,
   'animalspider.pipelines.ImagesspiderPipeline': 200,
}
# 定义图片的保存路径
IMAGES_STORE = 'animal_imgs'
# 定义接受图片的变量，默认为 'image_urls'
IMAGES_URLS_FIELD = 'img_url'
# 在输出中只显示错误相关的信息
LOG_LEVEL = 'ERROR'
```

### 启动爬虫

执行命令 `scrapy crawl animal ` ，可以看到不同动物的图片和简介都被存放到了以名字命名的文件夹中。

<img src="https://squidzh-1304890557.cos.ap-nanjing.myqcloud.com/blog_pic_bed/20210224220446.png" alt="image-20210224214357068" style="zoom:50%;" />

### 一些补充

在后续其他项目中，又不断发现了许多问题，尤其一些是之前撰写本文时没有考虑到的。

#### cookies 的设置

默认情况下，Scrapy 在 `settings.py` 文件中提供了一个全局性的 cookies 设置，如下所示。但这个选项在不同的情况下有着不同的效果，根据自己实践中的经验，这里做一个具体的记录。

```python settings.py
# Disable cookies (enabled by default)
COOKIES_ENABLED = True
```

- 当 `COOKIES_ENABLED` 这一行被注释掉时，Scrapy 会默认其开启 ，即值为 `True` 。
- 当 `COOKIES_ENABLED` 值为 `True` 时，Scrapy 就会启动相关的中间件，自动的进行 cookies 的获取和迭代工作，即使我们一开始并未设置请求头。如果我们手动设置了 cookies，例如 `Request(url, cookirs={'foo': 'bar'})`  ，那么 Scrapy 就会使用它，并不断地在请求过程中更新它（这里不能使用自定义 headers 的方法来定义 cookies ）。
- 当 `COOKIES_ENABLED` 值为 `False` 时，类似 `Request(url, cookies={'foo': 'bar'})` 这样设置的 cookies 就会失效，因为 Scrapy 中用于处理 cookies 的中间件在这时不会开启。这种情况下只能通过给请求自定义 headers 的方法来自定义 cookies ，例如 `Request(url, headers=my_headers)`。


**总而言之：**

- 完全不想使用 cookies ，就将 `COOKIES_ENABLED` 值设为 `False` ；
- 想要让 Scrapy 自动地管理 cookies ，就将其设置为 `True` ，且最好在第一次请求中手动加入初始的 cookies （这里不能使用自定义 headers 的方法来定义 cookies ）；
- 想在每一个请求中都使用自己自定义的 cookies （或在每一次请求中使用同样的 cookies ）可以将其设置为 `False` 。
- 对于 `settings.py` 中请求头中的 cookies 信息，有[文章](https://www.pianshen.com/article/9983265604/)表明当 `COOKIES_ENABLED` 没有注释设置为 `False` 的时候 Scrapy 便会默认使用它，这一点我没有考证。但可以确定的是，使用这个方法时不能重写爬虫文件中的 `start_requests` 方法。所以我个人还是不推荐通过 `setting.py` 文件来定义 cookies 。

#### 图片管道的设置

在使用图片管道时，需要先定义图片保存的根目录。

```python settings.py
IMAGES_STORE = 'IMAGES'  # 图片保存的根路径
```

在管道类的方法重写中，实测只需要继承一个类 `ImagesPipeline` 就可以正常保存图片。且不需要手动添加创建文件夹的方法，Scrapy 便会自动地在保存过程中创建文件夹。为了防止在文件保存过程中，文件名中含有非法导致保存失败，我们还可以手动添加一个替换非法字符的方法 `check_valid_title` 。下面是一个例子：

```python pipelines.py
from scrapy.pipelines.images import ImagesPipeline
from scrapy import Request
import re

class RmImageDownload(ImagesPipeline):
    
    def check_valid_title(self, title):  # 这个方法用来检查文件夹的名字是否合法
        rstr = r"[\/\\\:\*\?\"\<\>\|]"  # '/ \ : * ? " < > |'
        new_title = re.sub(rstr, "_", title)  # 将不合法字符替换为下划线
        return new_title

    def get_media_requests(self, item, info):
        img_url = item['image_url']
        return Request(img_url, headers=self.headers, meta={'item': item})

    def file_path(self, request, response=None, info=None, *, item=None):
        img_name = item['img_name']
        img_tiezi_name = self.check_valid_title(item['tiezi_name'])
        print(f'{img_name}将被保存到 - {img_tiezi_name}/')
        return f'{img_tiezi_name}/{img_name}'
```

`get_media_requests` 和 `file_path` 这两个方法在写的过程中比较痛苦，需要仔细对比源代码，模仿来写。

#### 日志信息的设置

Scrapy 在项目运行的过程中会在终端输出详细的日志，下面整理一些日志的设置方法。

```python settings.py
LOG_ENABLED = True  # 启用logging，默认 True
LOG_ENCODING = 'utf-8'  # logging 使用的编码， 默认为 utf-8
LOG_FILE = 'log.log'  # 日志文件保存的位置和名字
LOG_LEVEL = 'DEBUG'  # 定义显示在终端上的日志的级别，默认为 DEBUG
"""
Scrapy 提供 5 层 logging 级别供 LOG_LEVEL 定义使用:
CRITICAL - 严重错误(critical)
ERROR - 一般错误(regular errors)
WARNING - 警告信息(warning messages)
INFO - 一般信息(informational messages)
DEBUG - 调试信息(debugging messages)
"""
```

如果想要手动的在运行过程中输出一些日志，可以自行在爬虫或者管道文件中定义：

```python
import logging
logging.info('一般信息')
logging.warning('警告信息')
logging.debug('调试信息')
logging.error('错误信息')
logging.critical('严重错误信息')
# 使用此方法而不使用 print 的好处在于可以将自定义的日志内容记录到日志文件中
# 在打开日志记录时，终端就不再显示相关调试信息，这时可以将 print 和 logging 一起使用。
```

#### 另一种定制配置信息的方法（覆盖 settings.py）

我们可以在爬虫类或管道类内引入配置信息，这样就可以覆盖（跳过）`settings.py` 文件的设置，更加客制化的定制不同的爬虫或者管道。注意，`custom_settings` 必须是一个字典，其内部的参数也是字典。

```python spider.py
class ASpider(scrapy.Spider):
    name = 'spider'
    allowed_domains = ['xxx.com']
    custom_settings = {
        'ITEM_PIPELINES': { 
            'autospider.pipelines.APipeline': 300,
        },
        'LOG_LEVEL': 'DEBUG',
        'LOG_FILE': 'log.log'
        'DOWNLOAD_DELAY': 0,
    }
```

### 相关链接

本文代码：[Github/vincentimba/learnscrapy](https://github.com/vincentimba/learnscrapy)

> **参考文章：**
>
> [scrapy爬取图片并保存在不同的文件夹下](https://blog.csdn.net/qq_41963640/article/details/83904351)	
>
> [Scrapy官方文档](https://docs.scrapy.org/en/latest/intro/overview.html)
>
> [scrapy 中 COOKIES_ENABLED 设置](https://blog.csdn.net/u013444182/article/details/105282050/)
>
> [scrapy设置headers，cookies](https://www.pianshen.com/article/9983265604/)
>
> [爬虫scrapy框架--log日志输出配置及使用](https://blog.csdn.net/weixin_41666747/article/details/82716688)

