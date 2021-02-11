---
title: 一些学习笔记
categories: IT 业余票友
tags:
  - Linux
abbrlink: e482d28e
date: 2021-02-02 00:41:30
---

《Linux命令行与shell脚本编程大全》 人民邮电出版社

<!--more-->

### 关于 Windows Terminal 的一些设置
`Windows Terminal`是 MS 新开发的一个极好的终端。可以通过更改其配置文件`settings.json`进行高度的定制。另一个文件`defaults.json`提供了详细的参数设置，这个文件没有运行，只是一个参考文件。

下面提到的参数配置都是在`settings.json`中进行的。在`profiles-list`一项下可以进行各种各样的定义：
```java
"guid": "{-}",  // guid 可以通过工具生成
"name": "Ubuntu_TencentService", // 自定义名称
"commandline": "ssh ubuntu@42.193.152.16 -p22", // 可以在打开时自动建立 ssh 连接
"icon" : "ms-appx:///ProfileIcons/{9acb9455-ca41-5af7-950f-6bca1bc9722f}.png", // 图标
"startingDirectory": "D:\\HEXO_git_repo", // 启动时打开的目录
"colorScheme": "One Half Dark",  // 主题模板名字
"padding": "10,10,10,10",
"hidden": false // 是否显示
```
### /etc/passwd 文件
```
Vincent:x:501:501:Vincent Bresnahan:/home/Vincent:/bin/bash
```
在这个样例条目中，用户 Vincent 使用`/bin/bash`作为自己的默认 shell 程序。这意味着当 Vincent 登录 Linux 系统后，bash shell 会自动启动。

尽管bash shell会在登录时自动启动，但是，是否会出现 shell 命令行界面（CLI）则依赖于所使用的登录方式。如果采用虚拟控制台终端登录， CLI 提示符会自动出现，你可以输入 shell 命令。但如果是通过图形化桌面环境登录 Linux 系统，你就需要启动一个图形化终端仿真器来访问 hellCLI 提示符。

### Linux 的文件系统

Linux 使用正斜线（/）而不是反斜线（\）在文件路径中划分目录，例如`/home/Rich/Documents/test.doc`。Linux 中没有分区盘符的概念，而是使用一整个的虚拟目录。这点和 Windows 中的很不一样。下面列出几个常见的目录和其作用。

|目录名称|作用|
|----|----|
|/etc|系统配置文件目录|
|/home|主目录，Linux在这里创建用户目录|
|/lib|库目录，存放系统和应用程序的库文件|
|/root|root用户的主目录|
|/var|可变目录，用以存放经常变化的文件，比如日志文件|

### 遍历目录

cd 命令用来切换目录。cd 命令可接受单个参数`destination`，用以指定想切换到的目录名。**如果没有为 cd 命令指定目标路径，它将切换到用户主目录**。

`pwd`命令可以显示出 shell 会话的当前目录，这个目录被称为当前工作目录。

**绝对文件路径**总是以正斜线（/）作为起始，指明虚拟文件系统的根目录，例如`christine@server01:~$ cd /usr/bin`。

**相对文件路径**允许用户指定一个基于当前位置的目标文件路径。相对文件路径不以代表根目录的正斜线（/）开头，而是以目录名（如果用户准备切换到当前工作目录下的一个目录）或是一个特殊字符开始。例如`christine@server01:~$ cd Documents`。

单点符`.`，表示当前目录；双点符`..`，表示当前目录的父目录。可以直接利用双点符来切换同级目录，例如`christine@server01:~/Documents$ cd ../Downloads`。

### 处理文件

可用`touch`命令轻松创建空文件。
```
$ touch test_one
$ ls -l test_one
-rw-rw-r-- 1 christine christine 0 May 21 14:17 test_one
```

可用`cp`命令复制文件，`cp source destination`。当`source`和`destination`参数都是文件名时，cp命令将源文件复制成一个新文件，并且以`destination`命名。也可以将一个文件复制到空目录中。
```
$ cp -i test_one /home/christine/Documents/
$ ls -l /home/christine/Documents
total 0
-rw-rw-r-- 1 christine christine 0 May 21 15:25 test_one
```
可以用单点符来简化命令，尤其是当前目录名字繁琐时。
```
$ cp -i /etc/NetworkManager/NetworkManager.conf .
```
`mv`命令可以将文件和目录移动到另一个位置或重新命名，两个功能也可以一步完成。

```
$ mv /home/christine/Pictures/fzll /home/christine/fall
```
bash shell 中删除文件的命令是`rm`，可以加入参数`-i`，来询问自己是否真的需要删除文件。
```
$ rm -i fall
```
`mkdir`命令专门用来建立目录，且可以通过加入参数`-p`来同时创建其中缺失的父目录。
```
$ mkdir -p New_Dir/Sub_Dir/Under_Dir
```
一口气删除目录及其所有内容的终极大法就是使用带有-r参数和-f参数的rm命令。`rm -rf`命令既没有警告信息，也没有声音提示。**这是一个极其危险的命令。**

### 检测程序

`ps`命令可以显示当前运行的进程，但默认情况下显示内容很少。我们可以增加一些参数，例如：
```
$ ps -ef
```
这样可以查看系统上运行的所有进程。如果要删除一个进程，可以使用`kill`命令。
```sh
$ kill 3940 # 注意，使用这个命令必须是进程的属主或登录为root用户。
$ kill -9 3940 # 这里的参数 -9 代表了无条件中止
```

### 硬盘管理

Linux 上用来挂载媒体的命令叫作`mount`。默认情况下，`mount`命令会输出当前系统上挂载的设备列表。直接使用它可以输出当前系统上挂载的设备列表。

```sh
$ mount -t type device directory
```
上述是手动挂载设备的指令。卸载命令更加简单。
```
umount [directory | device ]
# 这里只需要输入 挂载目录 或 设备名 之中的一个作为参数
```
`df -h`命令可以显示设备上还有多少磁盘空间，也可以查看各个设备的挂载情况。
```
$ df -h
Filesystem      Size  Used Avail Use% Mounted on
udev            885M     0  885M   0% /dev
tmpfs           184M  6.1M  178M   4% /run
/dev/vda1        40G  2.7G   35G   8% /
tmpfs           917M   24K  917M   1% /dev/shm
tmpfs           5.0M     0  5.0M   0% /run/lock
tmpfs           917M     0  917M   0% /sys/fs/cgroup
tmpfs           184M     0  184M   0% /run/user/1001
tmpfs           184M     0  184M   0% /run/user/500
```
### 数据归档

目前，Unix和Linux上最广泛使用的归档工具是`tar`命令。
```
$ tar function [options] object1 object2 ...
# 下面是示例
tar -cvf test.tar test/ test2/
# 上面的命令创建了名为test.tar的归档文件，含有test和test2目录内容。
tar -xvf test.tar
# 上面的命令从tar文件test.tar中提取内容。
```

### shell 
可以在`/etc/passwd`文件中查看每个用户的默认shell程序。
```
$ cat /etc/passwd
[...]
git:x:115:124:git version control by pi,,,:/home/git:/bin/bash
```
这里最后一行的内容即表明，用户`git`使用`/bin/bash`作为自己的默认 shell 程序。
```
$ ls -lF /bin/bash
-rwxr-xr-x 1 root root 925124 4月  18  2019 /bin/bash*
```
从可以看出，`/bin/bash`是一个可执行的程序。


### 内建命令

可以使用`history`来回溯之前使用过的命令，通常会保存1000条。这个命令对于记录的整理可以说非常有用零。例如：
```
$ history
    1  sudo rsapi-config
    2  sudo raspi-config
    3  pwd
    4  su git
    5  cd
    6  ls
    7  cd -al
    8  ls -al
    9  ls
   10  cd
   11  mkdir .ssh
   12  cd .ssh
   13  ls
   14  pwd
   15  vim authorized_keys
```
可以通过`![num]`来快捷使用之前使用过的命令。

```
$ alias li='ls -li'
$ li
total 36
529581 drwxr-xr-x. 2 Christine Christine 4096 May 19 18:17 Desktop
529585 drwxr-xr-x. 2 Christine Christine 4096 Apr 25 16:59 Documents
529582 drwxr-xr-x. 2 Christine Christine 4096 Apr 25 16:59 Downloads
529586 drwxr-xr-x. 2 Christine Christine 4096 Apr 25 16:59 Music
529587 drwxr-xr-x. 2 Christine Christine 4096 Apr 25 16:59 Pictures
529584 drwxr-xr-x. 2 Christine Christine 4096 Apr 25 16:59 Public
529583 drwxr-xr-x. 2 Christine Christine 4096 Apr 25 16:59 Templates
532891 -rwxrw-r--. 1 Christine Christine 36 May 30 07:21 test.sh
529588 drwxr-xr-x. 2 Christine Christine 4096 Apr 25 16:59 Videos
```
`alias`命令可以创建常用命令的缩写，减少摄入量。

### Linux 中的环境变量

```
$ printenv # 查看全局环境变量
$ printenv [大写变量名称] # 查看具体某一个环境变量
```
可以通过将指定的目录或程序添加到环境变量中，实现以在虚拟目录结构中的任何位置执行程序。
```
$ echo $PATH
/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:
/sbin:/bin:/usr/games:/usr/local/games
$ PATH=$PATH:/home/christine/Scripts
$ echo $PATH
/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/
games:/usr/local/games:/home/christine/Scripts
```
对PATH变量的修改只能持续到退出或重启系统。

### Linux 中的安全性文件

用户对系统中各种对象的访问权限取决于他们登录系统时用的账户。

`/etc/passwd`文件专门用来将用户的登录名匹配到对应的UID值。例如`root`用户账户是 Linux 系统的管理员，固定分配给它的UID是`0`。
```
$ cat /etc/passwd
root:x:0:0:root:/root:/bin/bash
bin:x:1:1:bin:/bin:/sbin/nologin
daemon:x:2:2:daemon:/sbin:/sbin/nologin
adm:x:3:4:adm:/var/adm:/sbin/nologin
lp:x:4:7:lp:/var/spool/lpd:/sbin/nologin
sync:x:5:0:sync:/sbin:/bin/sync
shutdown:x:6:0:shutdown:/sbin:/sbin/shutdown
halt:x:7:0:halt:/sbin:/sbin/halt
mail:x:8:12:mail:/var/spool/mail:/sbin/nologin
news:x:9:13:news:/etc/news:
uucp:x:10:14:uucp:/var/spool/uucp:/sbin/nologin
operator:x:11:0:operator:/root:/sbin/nologin
games:x:12:100:games:/usr/games:/sbin/nologin
[...]
katie:x:502:502:katie:/home/katie:/bin/bash
jessica:x:503:503:Jessica:/home/jessica:/bin/bash
mysql:x:27:27:MySQL Server:/var/lib/mysql:/bin/bash
git:x:115:124:git version control by pi,,,:/home/git:/bin/bash
# 以下是每一项的含义
[用户名]:[密码 这里用x代替]:[用户UID]:[用户组UID]:[用户的描述和备注],,,:[home目录位置]:[用户默认shell]
```
为什么要大费周章地创建这么多用户来完成不同的事情呢？系统安全是最大的原因。如果没有这样的限制，一旦一个非授权的用户攻陷了这些服务中的一个，他立刻就能作为 root 用户进入系统并为所欲为。

用户的密码存储在`/etc/shadow`之中，只有 root 用户才能访问`/etc/shadow`文件，这让它比起`/etc/passwd`安全许多。它为系统上的每个用户账户都保存了一条记录。记录就像下面这样：
```
rich:$1$.FfcK0ns$f1UgiyHQ25wrB/hykCn020:11627:0:99999:7:::
```

### 创建、删除和修改用户
```
# useradd -m test
```
默认情况下，`useradd`命令不会创建`/home/username`目录，但是`-m`参数选项会使其创建目录，并将`/etc/skel`中的内容作为模板复制过来。

默认情况下，`userdel`命令会只删除`/etc/passwd`文件中的用户信息，而不会删除系统中属于该账户的任何文件。如果加上`-r`参数，`userdel`会删除用户的HOME目录以及邮件目录。
```
# /usr/sbin/userdel -r pi
# ls -al /home/pi
ls: cannot access /home/pi: No such file or directory
```
改变用户密码的一个简便方法就是用`passwd`命令。
```
# passwd test
Changing password for user test.
New UNIX password:
Retype new UNIX password:
passwd: all authentication tokens updated successfully.
```
如果只用`passwd`命令，它会改你自己的密码。系统上的任何用户都能改自己的密码，但只有`root`用户才有权限改别人的密码。