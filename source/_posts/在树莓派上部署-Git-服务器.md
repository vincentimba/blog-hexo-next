---
title: 在树莓派上部署 Git 服务器
categories: IT 业余票友
tags:
  - Git
  - 树莓派
  - Linux
abbrlink: c9172194
date: 2021-02-01 00:01:38
---
拿到树莓派的第一件事便是创建一个 Git 私有服务器。
<!--more-->

## 安装 Git
由于树莓派本身的源服务器在英国，因此速度比较慢，所以已经更换了国内的清华源，过程比较简单。

比较尴尬的是换源后发现，树莓派内部已经自带了最新版本的 Git。
```
sudo apt-get insatll git
```
之后新建一个新的用户`git`。
```
$ sudo adduser --system --shell /bin/bash --gecos 'git version control by pi' --group --home /home/git git
# 上面这句命令很厉害，一句话完成了很多工作。
$ sudo passwd git  # 设置用户密码
```
## 在 Pi 上初始化仓库
接下来在 git 用户根目录，即`/home/git/`创建一个新文件夹，并将其初始化为一个`--bare`仓库。
```
$ mkdir test_repo.git
$ cd test_repo.git/
$ git --bare init
# 已初始化空的 Git 仓库于 /home/git/test_repo.git/
```
`--bare`参数使得仓库为一个共享库，可以被多个终端进行`push`和`pull`操作。

## 在 PC 上提交代码
首先在电脑上创建一个新仓库（当然也可以使用已有的），更改路径到这个目录。打开`Git Bash`。
```
git remote add pi git@[your IP]:/home/git/test_repo.git
```
这里的`git remote add`作用为添加远程版本库，即树莓派上新建的`test_repo.git`。ip 地址即为局域网内地址。接下来就可以愉快地提交代码了~

```
git add .
git commit -am "Initial"
git push pi master
```
现在 PC 上新提交的代码已经被push到了树莓派上。

## 从树莓派拉取代码
为了测试代码的拉取，选择一个新的目录，打开`Git Bash`。
```
$ git clone git@192.168.0.221:~/test_repo.git
```
这里需要注意的是目录的写法`~/test_repo.git`。可以看到，树莓派中的代码已经成功地被拉取到了本地。

需要注意的是，在后续的`pull`中需要在命令中加上`branch`的名字。因为只有一个分支，所以一般为`master`，如下所示。
```
$ git pull pi master
```