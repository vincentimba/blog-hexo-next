---
title: Windows 下通过串口启动连接树莓派
categories:
  - IT 业余票友
  - 树莓派
tags:
  - 树莓派
abbrlink: e21af42
date: 2021-03-12 23:39:59
---

这几天回了学校，因为宿舍的网络原因，用 ssh 连接树莓派都做不到了。不过天无绝人之路，还剩下了最后一个方法，那就是用串口来登录树莓派。这次可以说是真正的 `headless` 模式了。

<!--more-->

#### 准备工作

除了常规的树莓派机器、SD 卡、SD 卡格式化软件以外，还需要准备以下东西。

**硬件：**

- USB 转 TTL 连接线，网购的话不到 10 块钱。

**软件：**

- PuTTY 或 Xshell，作为串口连接的终端。
- 刷机线驱动，这里提供我自己使用的 `PL2303TA` 驱动[下载地址](https://squidzh-1304890557.cos.ap-nanjing.myqcloud.com/blog_pic_bed/PL2303_Prolific_GPS_1013_20090319.zip)。



#### 安装驱动

首先将USB 转 TTL 连接线插入电脑的 USB 接口。如果电脑上没有对应的驱动，可以在设备管理器里看到一个未知设备。

<img src="https://squidzh-1304890557.cos.ap-nanjing.myqcloud.com/blog_pic_bed/20210312221046.png" alt="image-20210312205236081" style="zoom:50%;" />



安装了连接线驱动后，右键打开属性界面，点击**更新驱动程序**->**浏览我的电脑以查找驱动程序**->**让我从计算机上的可用驱动程序列表中选取**。

<img src="https://squidzh-1304890557.cos.ap-nanjing.myqcloud.com/blog_pic_bed/20210312234939.png" alt="image-20210312205417891" style="zoom:50%;" />

在这里可以看到刚刚安装的驱动程序，点击**下一步**安装。

<img src="https://squidzh-1304890557.cos.ap-nanjing.myqcloud.com/blog_pic_bed/20210312234944.png" alt="image-20210312205441004" style="zoom:50%;" />



之后再打开设备管理器，就可以看到已经被电脑正确识别的连接线设备了，其中括号中的 `COM3` 指电脑的端口，后续会用到。

<img src="https://squidzh-1304890557.cos.ap-nanjing.myqcloud.com/blog_pic_bed/20210312234949.png" alt="image-20210312205459304" style="zoom:50%;" />



右键打开设备属性，设置端口的波特率为 `115200`。至此电脑系统部分的设置完毕。

<img src="https://squidzh-1304890557.cos.ap-nanjing.myqcloud.com/blog_pic_bed/20210312234955.png" alt="image-20210312205632462" style="zoom:50%;" />



#### 烧写树莓派系统

给树莓派烧写系统的方法和常规的流程基本相同，可以参考之前写的[文章](https://zhangbh.xyz/post/69d30923/)。但需要注意，**若使用串口来进行树莓派的首次登录，还对配置文件进行编辑**（这一点网上的很多教程都没有提及，导致自己一直连接失败）。

SD 卡烧写完成后，在 `boot` 盘符中会存在一个 `config.txt` 文件，需要在其最后添加一行。

```yml
enable_uart=1
```



<img src="https://squidzh-1304890557.cos.ap-nanjing.myqcloud.com/blog_pic_bed/20210312235001.png" alt="image-20210312213026713" style="zoom:50%;" />



在这个文件的起始部分提供了一个[地址](http://rpf.io/configtxt)，里面详细说明了各项配置的含义，我们可以在其中找到 `enable_uart=1` 的含义。

> `enable_uart` Enable the primary/console UART (ttyS0 on a Pi 3, ttyAMA0 otherwise - unless swapped with an overlay such as miniuart-bt). If the primary UART is ttyAMA0 then enable_uart defaults to 1 (enabled), otherwise it defaults to 0 (disabled). This is because it is necessary to stop the core frequency from changing which would make ttyS0 unusable, so `enable_uart=1` implies core_freq=250 (unless force_turbo=1). In some cases this is a performance hit, so it is off by default. More details on UARTs can be found [here](https://www.raspberrypi.org/documentation/configuration/uart.md).

以我的六级擦边过的英语水平阅读理解了一下，大意为只有当树莓派的默认串口为 `ttyAMA0` 时，`enable_uart` 的值会默认为 `1`，否则的话就是 `0`。这样的设定原因可以看这一篇[文章](https://blog.csdn.net/weixin_45437140/article/details/102971270)。

> 树莓派3/4b的外设一共包含两个串口，一个称之为硬件串口（/dev/ttyAMA0），一个称之为mini串口（/dev/ttyS0）。硬件串口由硬件实现，有单独的波特率时钟源，性能高、可靠，mini串口性能低，功能也简单，并且没有波特率专用的时钟源而是由CPU内核时钟提供，因此mini串口有个致命的弱点是：波特率受到内核时钟的影响。内核若在智能调整功耗降低主频时，相应的这个mini串口的波特率便受到牵连了，虽然你可以固定内核的时钟频率，但这显然不符合低碳、节能的口号。在所有的树莓派板卡中都通过排针将一个串口引出来了，目前除了树莓派3代以外 ，引出的串口默认是CPU的那个硬件串口。而在树莓派3代中，由于板载蓝牙模块，因此这个硬件串口被默认分配给与蓝牙模块通信了，而把那个mini串口默认分配给了排针引出的GPIO Tx Rx。

之后将 SD 卡插入树莓派，树莓派的配置就完成了。



#### 软件配置

我这里使用了 PuTTY 作为通过串口连接树莓派的终端（Xshell 的设置方法大同小异）。打开软件之后选择 `Serial` 作为连接方式，并如下图进行配置。要注意填写的端口号要和前面设备管理器中显示的一致。

<img src="https://squidzh-1304890557.cos.ap-nanjing.myqcloud.com/blog_pic_bed/20210312235006.png" alt="image-20210312210133898" style="zoom:50%;" />



接着在左侧选择 `Serial` 选项，将 `Flow control` 改为 `None`，否则会出现连接成功后键盘无法输入的情况。

<img src="https://squidzh-1304890557.cos.ap-nanjing.myqcloud.com/blog_pic_bed/20210312224453.png" alt="image-20210312224449588" style="zoom:50%;" />



全部完成之后，点击 `Open`。



#### 硬件连接

现在将连接线插到树莓派的引脚上，树莓派的 GPIO 引脚图如下所示。

<img src="https://squidzh-1304890557.cos.ap-nanjing.myqcloud.com/blog_pic_bed/20210312225113.png" alt="image-20210312225109411" style="zoom:50%;" />



我们要使用的就是被框选的三个引脚，连接方式如下：

| 树莓派 | 引脚号 |       对应连接线接口       |
| :----: | :----: | :------------------------: |
|  GND   |   6    |    GND（Ground，黑色）     |
|  TXD   |   8    | RXD（Receive Data，白色）  |
|  RXD   |   10   | TXD（Transmit Data，绿色） |



下图是我连接好的样子。

![image-20210312230358200](https://squidzh-1304890557.cos.ap-nanjing.myqcloud.com/blog_pic_bed/20210312230400.png)



最后检查一边连接线已经正确接入电脑，并且所有配置填写无误。接通树莓派的电源并稍作等待，PuTTY 显示下面的字样表明已经连接成功（如果没有反应，可以尝试断开树莓派电源重启）。

<img src="https://squidzh-1304890557.cos.ap-nanjing.myqcloud.com/blog_pic_bed/20210312232151.png" alt="image-20210312215035058" style="zoom:50%;" />



输入默认用户名 `pi`，默认密码 `raspberry`，即可成功登录。

<img src="https://squidzh-1304890557.cos.ap-nanjing.myqcloud.com/blog_pic_bed/20210312232205.png" alt="image-20210312215053655" style="zoom:50%;" />



登录成功后，就可以和之前使用 ssh 一样进行各种操作了。

值得注意的是，启动信息显示我们使用了树莓派的 `ttyS0` 接口进行登录，即使用了树莓派的 mini 串口。由于这个串口受到内核时钟频率的影响，内核的时钟频率会被固定为 250MHz（`core_freq=250`），而树莓派的内核频率本应该是动态的，最大可以达到 400MHz。也就是说，在现在使用 mini 串口的情况下，树莓派的性能会被限制。至于这个限制造成的影响究竟有多大，如何手动调整树莓派端口的分配（据我现在的初步了解，应该是通过禁用蓝牙模块来腾出端口），就留给后面自己有时间再去研究吧。




