---
title: 自建 Git 仓库搭建 Hexo
categories: 
- IT 业余票友
- 树莓派
tags:
  - Git
  - Linux
  - Next
  - Hexo
abbrlink: 7116caa7
date: 2021-01-31 18:57:46
---
最早尝试使用了`Wordpress`，但实际使用中还是觉得略显沉重。偶然发现了`Hexo`，了解到了静态网站的概念，觉得非常适合自己。虽然在形式上需要通过`Git`来完成文件的推送，但也带来了轻量、方便版本管理的优势。

由于自有云服务器，所以没有使用传统的`GitHub`服务进行网站托管。这样的好处在于不用在忍受`GitHub`的龟速，缺点是增加了`Git`的部署工作。

本文的重点就在于服务器端`Git`的相关设置上。

<!--more-->

## Hexo 的安装和部署

`Hexo`静态的特性决定了所有的网页元素都在本地生成，相关的原始文件也都储存在本地，`Git`服务器上只存在经过编译过的网页文件，通过不断地`push`来实现网站的更新。`Hexo`在本地的具体安装方法在[官方文档](https://hexo.io/zh-cn/docs/)中有详细的说明，这里简要说明一下。

对于和我一样的Windows用户，`Git`和`Node.js`都可以通过客户端来安装，非常的直觉和方便。安装完毕后，选择一个合适的文件夹，打开命令提示符。
```
$ npm install -g hexo-cli
$ hexo init
$ npm install
```
文件夹中会生成相关的本地文件，代表着安装已经完成。具体的`Hexo`操作方式这里不做阐述。

## Git 的安装和配置

> 这一章节的操作全部在服务器端进行。

我的服务器使用`Ubuntu 18.04.1 LTS`系统，理论上其他的`Linux OS`操作方法大同小异。

```
$ sudo passwd root
$ su root
# useradd git
```
为`root`超级用户设置密码并切换，后续可以免去反复输入`sudo`的麻烦。可以看到前面的代表普通用户的`$`变成了`#`。之后创建新用户`git`（这里的用户名字可以随意选择，但使用`git`会更加明确，代表这个用户专门负责`Git`的操作）。

需要说明的是，创建专门的用户来对`Git`仓库进行管理并不是必须的，但从安全及管理的角度十分推荐这么做。后续的操作与这个用户相关权限设置有关。

```
# vim /etc/sudoers
```
打开文件`/etc/sudoers`，为新创建的`git`用户设置权限。

```
# User privilege specification
root	ALL=(ALL:ALL) ALL
git     ALL=(ALL:ALL) ALL  # 此行为添加的内容
```
这里通过在`User privilege specification`下添加一行，赋予git和root一样的权限。
```
# vim /etc/passwd
```
打开`/etc/passwd`文件。
```
# 将
git:x:1000:1000::/home/git:/bin/bash
# 修改为
git:x:1000:1000:,,,:/home/git:/usr/bin/git-shell
```

修改`/etc/passwd`的最后一行，禁用`git`用户的 shell 登录权限，保证服务器的安全。修改过后`git`用户只能从而只能用`git clone`，`git push`等方法连接服务器。

```
# su git
$ cd /home/git
```
切换为`git`用户并打开`/home/git`文件夹。这里遇到了一个问题，提示如下：
```
bash: history: /home/git/.bash_history: cannot create: Permission denied
bash: history: /home/git/.bash_history: cannot create: Permission denied
bash: history: /home/git/.bash_history: cannot create: Permission denied
bash: history: /home/git/.bash_history: cannot create: Permission denied
bash: history: /home/git/.bash_history: cannot create: Permission denied
bash: history: /home/git/.bash_history: cannot create: Permission denied
bash: history: /home/git/.bash_history: cannot create: Permission denied
bash: history: /home/git/.bash_history: cannot create: Permission denied
bash: history: /home/git/.bash_history: cannot create: Permission denied
```
显示权限不足，访问被拒绝。

```
$ exit
# chmod -R 777 /home/git
# chown -R git:git /home/git
```
我们可以切换到`root`用户，修改这个文件夹`/home/git/.bash_history`的权限，并将其用户指定给`git`。之后就可以顺利使用`git`用户进入文件夹了。

```
# su git
$ cd /home/git
$ mkdir blog.git
```
切换到`git`用户，创建一个名为`blog.git`的文件夹，这个文件夹用来接受从本地推送来的`Hexo`网页文件。接下来将其初始化为`Git`仓库。
```
$ cd /home/git/blog.git/
$ git init --bare
# Initialized empty Git repository in /home/git/blog.git/
# 出现上述提示表示仓库建立成功
```
注意在初始化仓库是加入了`--bare`的参数，即创建了一个裸仓库。裸仓与普通仓库不同，用来进行各种共享操作，生成的`repo`文件夹一般以`.git`为后缀作为区别。

接下来要进行`hook`的设置。`hook`原意为钩子，在这里指自动化操作，通过检测`Git`仓库中发生的变化而自动进行一些预设好的操作。我们将通过`hook`把仓库中被上传到服务器的网页文件移动到预设的文件夹中，供 web 服务器调用处理。

```
$ mkdir -p /var/www/blog/ # 新建存放Hexo生成网页的文件夹
$ cd /home/git/blog.git/hooks # 打开 blog.git 目录内的 hooks 文件夹
$ vim post-receive # 编辑 post-receive 文件
```
`post-receive`即定义`Git`在收到文件后所作操作的文件。在其中填写以下内容。
```
#!/bin/bash

GIT_REPO=/home/git/blog.git # 定义Git裸仓的地址
TMP_GIT_CLONE=/tmp/blog # 定义临时存放地址
PUBLIC_WWW=/var/www/blog # 定义裸仓中的文件去往的地址
rm -rf ${TMP_GIT_CLONE}
git clone $GIT_REPO $TMP_GIT_CLONE地址
rm -rf $PUBLIC_WWW/* 
cp -rf $TMP_GIT_CLONE/* $PUBLIC_WWW
```
hook 配置结束。再设置一下`post-receive`文件权限。
```
$ chmod +x post-receive
```
至此，`Git`基本设置完毕。

## Nginx 的安装

> 这一章节的操作全部在服务器端进行。

Web 服务器选择大名鼎鼎的`Nginx`，安装方法非常简单，只需要一行代码。

```
$ exit # 退出git用户
# apt-get install nginx
```

耐心等待安装结束后，可使用`nginx -v`查看版本号，确定是否安装成功。之后进行`Nginx`的配置。

```
# vim /etc/nginx/sites-enabled/default 
```
打开配置文件`/etc/nginx/sites-enabled/default `。

```
server {
	listen 80 default_server;
	listen [::]:80 default_server; # 这里的ip地址可以使用默认

	………

	root /var/www/blog; # 将这里改为前文 hook 设置中的目标目录

	# Add index.php to the list if you are using PHP
	index index.html index.htm index.nginx-debian.html;

	server_name _;

	location / {
		# First attempt to serve request as file, then
		# as directory, then fall back to displaying a 4
		try_files $uri $uri/ =404;
	}


	# deny access to .htaccess files, if Apache's document r
	# concurs with nginx's one
	#
	#location ~ /\.ht {
	#	deny all;
	#}
}
```

保存文件并退出。输入`# /etc/init.d/nginx restart`重启`Nignx`服务。
```
[ ok ] Restarting nginx (via systemctl): nginx.service.
```
出现上述字样即代表`Nignx`重启成功。如果此时直接访问服务器地址，将可以看到`Nignx`的 403 界面。

## Git 和 ssh 的免密码登录设置

首先本地需要创建公钥和私钥。执行命令`ssh-keygen -t rsa`，对于密钥的使用密码可以忽略填写，连续按回车跳过。本地`~/.ssh`下会出现两个文件，分别是私钥`id_rsa`和公钥`id_rsa.pub`。

将公钥内容复制到服务器端`~/.ssh/authorized_keys`中。注意，**这里的`authorized_keys`文件必须放置在`git`用户的目录下，即`/home/git/.ssh`下**。

实际操作中，在完成上述步骤后还是出现了不能免密登录的问题。查阅各种方法后，通过更改`authorized_keys`文件权限解决了问题，步骤如下。

```
# 更改用户权限
chmod 700 /home/username
# 更改 .ssh 文件夹权限
chmod 700 ~/.ssh/
# 更改 ~/.ssh/authorized_keys 文件权限
chmod 600 ~/.ssh/authorized_keys
```

## 即将就绪

最后在本地进行一项`Hexo`的配置，将`Hexo`生成的网页文件指向服务器的`Git`仓库。

打开`Hexo`文件目录下的`_config.yml`文件，找到最下方的`Deployment`设置。
```
# Deployment
## Docs: https://hexo.io/docs/one-command-deployment
deploy:
  type: git
  branch: master # 由于没有分支所以填写 master
  repo: git@[ip_address]:blog.git # 填写服务器的公网 ip 地址
```

保存退出，之后在`Hexo`打开命令提示符。
```
$ hexo clean && hexo g
$ hexo d
```
不出意外的话，`Hexo`已经顺利地推送网页文件到服务器了。 ^_^

## 总结
此次搭建`Hexo`的过程过程是艰辛曲折但是又饶有趣味的。

过程中发现自己对`Linux`，尤其是对其多用户的特性还有很大的认知不足，这一点造成了巨大的麻烦。另外，自己对于`Git`的工作原理理解也不够深入。