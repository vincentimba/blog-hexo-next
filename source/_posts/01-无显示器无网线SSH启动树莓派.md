---
title: 无头启动树莓派
categories: 
- IT 业余票友
- 树莓派
tags:
  - 树莓派
abbrlink: 69d30923
date: 2021-01-31 01:49:38
---
没有外接显示器、没有外接鼠标、没有外接键盘的启动方式被官方称作 `headless` ，粗暴翻译就是无头模式。这次启动 Raspberry Pi 4B 4GB 版本，本以为做好了功课，所以自信地没有加购hdmi线，但还是出了很多问题。整理流程以作记录。
<!--more-->

## 烧录镜像至SD卡
提前下载系统镜像文件，一共有三个。

`2020-12-02-raspios-buster-armhf-lite.zip`
`2020-12-02-raspios-buster-armhf.zip`
`2020-12-02-raspios-buster-armhf-full.zip`

三个镜像依次从小到大，安装了不同数量的预装，因为国内安装软件需换源，所以选择了最多预装的`full`版本。解压后得到`img`格式的镜像文件。

下载安装`SDFormatter`和`Win32DiskImager`两个软件。

前者进行格式化，后者进行烧写。SD卡在经烧写之后会在系统中产生两个盘符，无法直接格式化，必须使用`SDFormatter`。之后选择解压后的文件进行烧写，时间比较久。



## 配置ssh和WiFi ssid
由于没有hdmi线材和网线，必须通过WiFi进行远程ssh连接。

SD卡烧制完成后，会出现windows的格式化提示。**忽略提示，拔掉读卡器，再插入读卡器。**此时会出现名为`boot`的盘符，大小在几百M。向其中拖入两个文件，分别为`ssh`和`wpa_supplicant.conf`。

`ssh`为空白文件，没有后缀。`wpa_supplicant.conf`内容如下。

```python
country=CN
ctrl_interface=DIR=/var/run/wpa_supplicant GROUP=netdev
update_config=1

network={
	ssid="WiFi名称"  //注意！这里的引号必须是英文引号""
	psk="WiFi密码"   //不能是中文引号“”，血的教训 QAQ
	key_mgmt=WPA-PSK
}
```
两个文件复制入`boot`盘符后，即可将SD卡插入Raspberry Pi了。接通电源开机后，Pi会自动连接预设的WiFi。
## ssh连接树莓派
对于同属一个局域网内的设备，只要知道了对方的内网ip地址即可方便地通过ssh访问。树莓派的ip可以通过路由器后台来获取。注意，**如果在烧制SD卡之后没有放入空白ssh文件，则ssh服务不能使用。**

路由器后台网址不同品牌的路由器会有不同，一般为`192.168.0.1`。输入管理员密码后可通过DCHP的地址列表来查看ip地址，树莓派的设备名即为英文`raspberrypi`，很好区分。拿到ip地址之后事情就变得简单起来。直接打开命令提示符，输入`ssh pi@[局域网ip地址]`，之后会要求输入密码，默认为`raspberry`。

ssh连接成功后，输入`sudo raspi-config`，进入配置界面打开vnc服务。之后即可下载vnc桌面版软件，直接进入图形界面配置了。除了vnc服务之外，也可以对诸多其他选项进行配置，包括地区、屏幕分辨率、语言等等，在此略过。

## 为树莓派换源

总共需要改写两个文件。第一个，`sudo nano /etc/apt/sources.list`，将源文件中的网站地址注释掉，并添加新的两行。

```
deb http://mirrors.tuna.tsinghua.edu.cn/raspbian/raspbian/ buster main non-free contrib
deb-src http://mirrors.tuna.tsinghua.edu.cn/raspbian/raspbian/ buster main non-free contrib
```

第二个文件，`/etc/apt/sources.list.d/raspi.list`。

```
deb http://mirrors.tuna.tsinghua.edu.cn/raspberrypi/ buster main ui
```

接下来可以更新列表了，`sudo apt-get update`，可以发现速度快了很多。

## 补充
由于多数路由器都开启了DHCP功能，树莓派每次进行ssh连接时都可能会发生ip地址变化，非常不便。可以通过配置内部文件进行解决，但较为麻烦。推荐在路由器后台中将树莓派mac地址与ip地址进行绑定。