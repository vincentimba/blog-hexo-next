---
title: 在 Windows 上配置 SQLite
categories:
  - IT 业余票友
  - 数据库
tags:
  - SQLite
abbrlink: '8e517981'
date: 2021-02-22 15:34:15
---

SQLite 这种轻量易用数据库仿佛就是为自己这种业余票友量身定做。

<!--more-->

进入 SQLite 的[官方网站](https://www.sqlite.org/download.html)，下载两个预编译的二进制文件。

![image-20210222153831094](https://squidzh-1304890557.cos.ap-nanjing.myqcloud.com/blog_pic_bed/20210222153842.png)

因为下载速度非常的慢，自己下载了很多次才成功，所以这里也提供一下这两个文件的下载，地址放在文末。

创建目录 `C:\sqlite`，将两个压缩包中的内容解压进去，如下图所示。

![image-20210222163846129](https://squidzh-1304890557.cos.ap-nanjing.myqcloud.com/blog_pic_bed/20210222163850.png)

将 `C:\sqlite` 添加进系统的环境变量，过程不表。

最后在 powershell 或 cmd 中输入 `sqlite3` ，显示图下结果即为成功。

<img src="https://squidzh-1304890557.cos.ap-nanjing.myqcloud.com/blog_pic_bed/20210222164032.png" alt="image-20210222164024129" style="zoom:50%;" />

推荐使用 [SQLiteStudio](https://sqlitestudio.pl/) 作为管理工具。

相关文件下载地址：

- [sqlite-dll-win64-x64-3340100.zip](https://squidzh-1304890557.cos.ap-nanjing.myqcloud.com/blog_pic_bed/sqlite-dll-win64-x64-3340100.zip)
- [sqlite-tools-win32-x86-3340100.zip](https://squidzh-1304890557.cos.ap-nanjing.myqcloud.com/blog_pic_bed/sqlite-tools-win32-x86-3340100.zip)

