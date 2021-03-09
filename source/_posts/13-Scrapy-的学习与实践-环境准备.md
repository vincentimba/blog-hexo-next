---
title: Scrapy 的学习与实践（一）——环境准备
categories:
  - IT 业余票友
  - Python
tags:
  - Python
  - 爬虫
abbrlink: 31f71550
date: 2021-02-16 19:22:32
---

Scrapy 是一套基于 Twisted 的异步处理框架，也是纯 python 实现的爬虫框架，用户只需要定制开发几个模块就可以轻松的实现一个爬虫，用来抓取网页内容以及各种图片。之前自己写过一些基础的基于 requests 和 Beautiful Soup 库的小项目，这次不玩轮子了，直接试驾一下 Scrapy 这辆功能全面的小跑车。

<!--more-->

### 安装 Anaconda

由于 Scrapy 是一个安装起来比较麻烦的框架，需要依赖非常多的扩展库，所以这次选择了在 Anaconda 进行开发。

> Anaconda 是一个用于科学计算的 Python 发行版，支持 Linux, Mac, Windows, 包含了众多流行的科学计算、数据分析的 Python 包。

安装和下载很简单。可以在 Anaconda 的[官网](https://www.anaconda.com/)下载，也可以通过 tuna 提供的[镜像](https://mirrors.tuna.tsinghua.edu.cn/anaconda/archive/)下载。

![image-20210216135128514](https://squidzh-1304890557.cos.ap-nanjing.myqcloud.com/blog_pic_bed/20210216192554.png)

这里有比较特别的一个点，Anaconda 会建议你不要将其添加入系统的环境变量里，直接从开始菜单进入其专用的 bash ，如下图所示。另外第二个选项中，Anaconda 也会将其自己设为默认的 Python ，这样各种 IDE 就可以找到它了。

![image-20210216135731256](https://squidzh-1304890557.cos.ap-nanjing.myqcloud.com/blog_pic_bed/20210216192557.png)



### 配置 Pycharm

打开 Pycharm 菜单栏的 File-Settings ，在搜索框中输入 `interpreter` ，接着在右边选择添加新的编译器。

![image-20210216140521858](https://squidzh-1304890557.cos.ap-nanjing.myqcloud.com/blog_pic_bed/20210216192600.png)

之后选择下面的使用现有环境 `Existing environment` ， 设置 Anaconda 的安装目录，勾选对所有项目生效。

![image-20210216140749136](https://squidzh-1304890557.cos.ap-nanjing.myqcloud.com/blog_pic_bed/20210216192604.png)

现在可以看到，Anaconda 的编译器已经被正确地配置到了 Pycharm 里。

![image-20210216141155753](https://squidzh-1304890557.cos.ap-nanjing.myqcloud.com/blog_pic_bed/20210216192607.png)



### 安装 Scrapy 爬虫框架

安装了 Anaconda 后，Scrapy 的安装就会简单很多。为了更快更稳定的下载速度，首先对 Anaconda 进行换源。必须要感谢 [tuna](https://mirrors.tuna.tsinghua.edu.cn/anaconda/cloud/) 做出的贡献，其也给出了官方的[中文帮助文档](https://mirror.tuna.tsinghua.edu.cn/help/anaconda/)。

由于 Windows 用户无法直接创建名为 `.condarc` 的文件，可先执行 `conda config --set show_channel_urls yes` 生成该文件之后再修改。

```yml .condarc
channels:
  - defaults
show_channel_urls: true
default_channels:
  - https://mirrors.tuna.tsinghua.edu.cn/anaconda/pkgs/main
  - https://mirrors.tuna.tsinghua.edu.cn/anaconda/pkgs/r
  - https://mirrors.tuna.tsinghua.edu.cn/anaconda/pkgs/msys2
custom_channels:
  conda-forge: https://mirrors.tuna.tsinghua.edu.cn/anaconda/cloud
  msys2: https://mirrors.tuna.tsinghua.edu.cn/anaconda/cloud
  bioconda: https://mirrors.tuna.tsinghua.edu.cn/anaconda/cloud
  menpo: https://mirrors.tuna.tsinghua.edu.cn/anaconda/cloud
  pytorch: https://mirrors.tuna.tsinghua.edu.cn/anaconda/cloud
  simpleitk: https://mirrors.tuna.tsinghua.edu.cn/anaconda/cloud
```

接着运行 `conda clean -i` 清除索引缓存，换源完毕。

现在正式安装 Scrapy 。在 bash 中输入 `conda install scrapy` ，略作等待。之后输入 `scrapy` 查看是否安装成功。

![image-20210216191036718](https://squidzh-1304890557.cos.ap-nanjing.myqcloud.com/blog_pic_bed/20210216192611.png)

显示这样的界面就说明安装成功了。