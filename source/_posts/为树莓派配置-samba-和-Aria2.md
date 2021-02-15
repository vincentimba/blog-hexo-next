---
title: 为树莓派配置 Aria2 和 Samba
categories: 
- IT 业余票友
- 树莓派
tags:
  - 树莓派
  - Linux
abbrlink: 27be0ddf
date: 2021-02-10 17:00:12
---

Aria2 可以使树莓派进行 24 小时远程下载。再加上 smb (Samba) 服务，即可在局域网内的各种设备上直接远程查看下载到的文件。 除此之外，由于 Linux 系统对 NTFS 格式硬盘的访问需要额外的插件和较大的算力占用，还需将外接的储存设备格式化为 ext4 格式。

<!--more-->

### 换源

查阅过一些文章，很多都会先对原来的文件进行备份。但我觉得其实只需要把原来的部分注释掉即可，不用那么麻烦。以下是需要添加的内容。

```yml /etc/apt/sources.list
deb http://mirrors.tuna.tsinghua.edu.cn/raspbian/raspbian/ buster main non-free contrib
deb-src http://mirrors.tuna.tsinghua.edu.cn/raspbian/raspbian/ buster main non-free contrib
```

```yml /etc/apt/sources.list.d/raspi.list
deb http://mirrors.tuna.tsinghua.edu.cn/raspberrypi/ buster main ui
```

需要注意的是在换源之后还要进行 apt-get 的更新和升级。

```bash
$ sudo apt-get update
$ sudo apt-get upgrade
```

在清华开源镜像站上也有针对树莓派的[帮助文档](https://mirrors.tuna.tsinghua.edu.cn/help/raspbian/)可以参考。

### 格式化硬盘

为了更好的在 Linux 上使用硬盘，我们首先将其格式化为 ext4 格式。和 NTFS 一样，ext4 也是一种日志文件管理格式，这就使得其具有相对 exFAT 更高的安全性。

首先我们将硬盘插入树莓派，并查看系统的硬盘挂载情况，`sudo fdisk -l`。

```
[……]
Device     Boot Start        End    Sectors   Size Id Type
/dev/sda1        2048 1953521663 1953519616 931.5G  7 HPFS/NTFS/exFAT
```

可以看到最后一项便是新接入的移动硬盘，大小为 1TB， 格式为 `HPFS/NTFS/exFAT`，这里用 `sudo mkfs.ext4 /dev/sda` 直接将它格式化（**请提前备份好其中的数据）**，接着再进行硬盘分区。

```
$ sudo fdisk /dev/sda
```

```
# 这里创建了两个分区，各 500GB
# 输入 n 来新建第一个分区
Command (m for help): n
Partition type
   p   primary (0 primary, 0 extended, 4 free)
   e   extended (container for logical partitions)
# 使用默认的 p ，表示新建的是一个主分区，而非扩展分区（p）
Select (default p): p
Partition number (1-4, default 1): 1
First sector (2048-1953525166, default 2048):
Last sector, +/-sectors or +/-size{K,M,G,T,P} (2048-1953525166, default 1953525166): 1,048,576,0^[[A^[[A^[[A^[[A^[[A^[^[^[^[^[^[^[^[^[^[^[^[^[^[^[^[^[^[^[^[^[^[^[^[^[^[^[^[^[^[^[^[^[^[^[^[^[^[^[^[^[^[
Last sector, +/-sectors or +/-size{K,M,G,T,P} (2048-1953525166, default 1953525166): 1048576000

Created a new partition 1 of type 'Linux' and of size 500 GiB.

# 第一个 500GB 的分区创建完毕，再次输入 n 创建第二个分区
Command (m for help): n
Partition type
   p   primary (1 primary, 0 extended, 3 free)
   e   extended (container for logical partitions)
Select (default p): p
Partition number (2-4, default 2): 2
First sector (1048576001-1953525166, default 1048578048):
Last sector, +/-sectors or +/-size{K,M,G,T,P} (1048578048-1953525166, default 1953525166):

Created a new partition 2 of type 'Linux' and of size 431.5 GiB.
# 这样两个分区就都创建好了
```

再次查看一下分区的情况，可以看到两个分区都创建好了，名称分别为 `/dev/sda1 `和 `/dev/sda2`。

```
Device     Boot      Start        End    Sectors   Size Id Type
/dev/sda1             2048 1048576000 1048573953   500G 83 Linux
/dev/sda2       1048578048 1953525166  904947119 431.5G 83 Linux
```

再次格式化两个分区，使用 ext4 格式。

```bash
$ sudo mkfs.ext4 /dev/sda1
$ sudo mkfs.ext4 /dev/sda2
```

### 挂载硬盘分区

新建用来挂载硬盘的文件，我选择创建在用户主目录下。

```bash
$ mkdir st_disk
$ mkdir st_disk_2
```

接下来将上文中创建的硬盘分区分别挂载到这两个目录上。

```bash
$ sudo mount /dev/sda1 /home/pi/st_disk/
$ sudo mount /dev/sda2 /home/pi/st_disk_2/
```

使用 `df -T`命令查看设备目录，可以看到两个分区已经成功挂载。

```
文件系统       类型         1K-块    已用      可用   已用% 挂载点
/dev/root      ext4      15022248 1598292  12781416   12% /
devtmpfs       devtmpfs   1827800       0   1827800    0% /dev
tmpfs          tmpfs      1959896       0   1959896    0% /dev/shm
tmpfs          tmpfs      1959896    8652   1951244    1% /run
tmpfs          tmpfs         5120       4      5116    1% /run/lock
tmpfs          tmpfs      1959896       0   1959896    0% /sys/fs/cgroup
/dev/mmcblk0p1 vfat        258095   55050    203046   22% /boot
tmpfs          tmpfs       391976       0    391976    0% /run/user/1000
/dev/sda1      ext4     515009792   73756 488705304    1% /home/pi/st_disk
/dev/sda2      ext4     444321652   73756 421607836    1% /home/pi/st_disk_2
```

配置一下硬盘的开机自动挂载，这样就不需要在每次树莓派重启后手动挂载硬盘了。输入 `sudo blkid` 来获取分区的唯一身份标识，也就是 `UUID`。

```
/dev/sda1: UUID="9102ec87-e386-xxxxxxxxxx" TYPE="ext4" PARTUUID="7dxxxxxx-01"
/dev/sda2: UUID="2ac3b729-59f7-xxxxxxxxxx" TYPE="ext4" PARTUUID="7dxxxxxx-02"
```

打开`/etc/fatab` 文件，各个设备的挂载信息都是在这里进行配置的。在其中添加两行。

```
PARTUUID=7dxxxxxx-01  /home/pi/st_disk/  ext4  defaults  0  0
PARTUUID=7dxxxxxx-02  /home/pi/st_disk_2/  ext4  defaults  0  0
```

文件信息共有五列，每一项的意思都不同：

- `PARTUUID`即上面获取的 `UUID` 。当然这里也可以使用硬盘的相对地址 `/dev/sda1`，但如果同时接入多个外置设备可能会造成挂载的混乱，所以最好还是使用 `UUID` 。
- `/home/pi/st_disk/`即硬盘挂载到的目录。
- `ext4` 为硬盘的格式，根据自己的配置信息来填写。
- `defaults` 这一项是挂载配置选项，使用默认即可。
- 第一个 `0` 指转储频率：0-不做备份；1-每天转储；2-每隔一天转储。
- 第二个 `0` 指自建次序：0-不自检；1-首先自检。

最后更改一下两个新建文件夹的用户，使用默认的用户 `pi`。

```bash
$ sudo chown pi:pi st_disk
$ sudo chown pi:pi st_disk_2
```

### 配置 Aria2

安装 Aria2 并创建配置文件。

```bash
$ sudo apt-get install aria2  # 最好在安装前先对树莓派进行换源并更新 apt-get
$ sudo mkdir /etc/aria2
$ sudo touch /etc/aria2/aria2.conf aria2.session
```

这里新建了两个文件，`aria2.session`是会话记录保存文件，用于保存信息，所以无需改动。`aria2.conf` 是具体的配置文件，打开并添加以下内容。

```yml
#文件保存目录
dir=/home/pi/st_disk/
disk-cache=32M
continue=true
#NTFS建议使用falloc, EXT3/4建议trunc, MAC 下需要注释此项
file-allocation=trunc

#下载连接相关
max-concurrent-downloads=10
max-connection-per-server=15
split=10

#进度保存相关
input-file=/etc/aria2/aria2.session
save-session=/etc/aria2/aria2.session
save-session-interval=60

#RPC相关设置
enable-rpc=true
rpc-allow-origin-all=true
rpc-listen-all=true

#BT/PT下载相关
peer-id-prefix=-TR2770-
user-agent=Transmission/2.77
bt-seed-unverified=true
bt-save-metadata=true
```

使用配置文件打开 Aria2 ，测试一下是否配置成功`sudo aria2c --conf-path=/etc/aria2/aria2.conf` ，显示正在监听，表明启动成功。

```
02/09 03:36:13 [NOTICE] IPv4 RPC：正在监听 TCP 端口 6800
02/09 03:36:13 [NOTICE] IPv6 RPC：正在监听 TCP 端口 6800
```

接下来配置 Aria2 的开机启动。创建文件 `/etc/init.d/aria2c` 并添加以下内容：

```
### BEGIN INIT INFO
# Provides: aria2c
# Required-Start:    $network $local_fs $remote_fs
# Required-Stop:     $network $local_fs $remote_fs
# Default-Start:     2 3 4 5
# Default-Stop:      0 1 6
# Short-Description: aria2c RPC init script.
# Description: Starts and stops aria2 RPC services.
### END INIT INFO
###这里的用户名需要注意
USER=pi
RETVAL=0

case "$1" in
    start)
        echo "Starting service Aria2..."
        aria2c --conf-path=/etc/aria2/aria2.conf -D
        echo "Start service done."
    ;;
    stop)
        echo "Stoping service Aria2..."
        killall aria2c
        echo "Stop service done."
    ;;
esac

exit $RETVAL
```

更改文件的权限与同目录内的其他文件一致，键入 `sudo chmod 755 /etc/init.d/aria2c`。

### 配置 AriaNG 和 Nginx

AriaNG 是 Aria2 的 WebUI，让 Aria2 可以通过浏览器的图形界面直接进行管理。

首先安装 Nginx 作为 Web 服务器，并设置其的开机启动。

```
$ sudo apt-get install nginx-full
$ sudo update-rc.d nginx defaults
```

现在直接在浏览器输入树莓派的内网 ip 已经可以看到 Nginx 的提示。

接下来安装 AriaNG ，并将其解压到 `/var/www/html`供 Nginx 使用。

```
$ cd /tmp
$ wget https://github.com/mayswind/AriaNg/releases/download/1.1.7/AriaNg-1.1.7.zip
$ sudo unzip AriaNg-1.1.7.zip -d /var/www/html/
```

再次启动 Aria2 ，`sudo aria2c --conf-path=/etc/aria2/aria2.conf` 。

顺利的话，输入 ip 地址就可以顺利看见 Aria2 的图形界面后台了。

### 配置 Samba

首先进行安装。

```
$ sudo apt-get install samba 
$ sudo apt-get install samba-common
```

修改 Samba 的配置文件，打开`/etc/samba/smb.conf`，找到`[printers]`这一行，在其后添加以下内容：

```yml
[public]
   comment = public path
   path = /mnt/usb/
   guest ok = yes
   browseable = yes
   writeable = yes
   create mask = 0777
   directory mask = 0777
```

为 Samba 创建一个用户，这个用户必须已经存在于系统中。更确切的说，这个用户名必须存在于`/etc/passpwd`之中。

```
sudo smbpasswd -a pi # 设置密码
sudo service smbd restart # 重启 Samba 服务
```

现在即可在各种设备是访问 smb 服务了。Windows 可以在资源管理器中添加映射网络驱动器，iOS 设备可以使用第三方的文件管理器，实测 ES 文件浏览器可以正常访问。安卓同理。