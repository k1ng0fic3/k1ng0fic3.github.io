---
date: 2020-04-01 12:47:44
tags:
- 随笔
---

# 最友好のWin+Linux双系统安装

本文介绍一下装第二系统时踩的坑和最终解决的过程，想参考文章安装系统的朋友直接向下翻找到教程那儿即可。
<!-- more -->
起因：换新电脑之后，本想把旧电脑刷成Linux。后来想了想，还是做成双系统比较方便，于是开始了踩坑之路。

最开始的想法无非是，用iso镜像做一个U盘启动安装，但是马上就遇到了问题。

既然是双系统，肯定会考虑如何做好开机引导，后面看到了两篇文章，本以为找到了救星，结果不小心踩了坑……（主要原因大概还是我自己菜，大佬的文章还是很好的）

### 坑文

给出了[单磁盘多系统](https://blog.csdn.net/lidawei0124/article/details/80383469)和[多磁盘多系统](https://blog.csdn.net/u014113983/article/details/83049102)的两种方案。

其他博主写的这种方法的文章也有很多，可是照着实现的时候总是会出现引导错误等问题，安装不成功。

所以无论单磁盘还是多磁盘，用上面的办法都太冒险了，装不好系统事小，万一哪里没有注意，一个错误把原来的系统也损坏就亏大了。

![avatar](https://k1ng0fic3.github.io/images/xitong1.png)
### 教程

[救星](https://blog.csdn.net/qingsong3333/article/details/80863293)出现！

#### 准备工作

原材料：
>Windows X
EasyBCD 2.3
ubuntu-18.04.3-desktop-amd64.iso or [其他版本](https://ubuntu.com/download)

#### 可用空间

首先，win+r，输入`diskmgmt.msc`进行空间管理，删除某个分区空出一块地方来。

![avatar](https://k1ng0fic3.github.io/images/xitong2.png)

![avatar](https://k1ng0fic3.github.io/images/xitong3.png)

#### 配置引导

划重点！要将.iso中的`initrd.lz`、`vmlinuz.efi`解压出来，与iso一起，这三个文件一同放在C盘或D盘根目录（必须根目录）下：（我这里的两个文件没有后缀）

![avatar](https://k1ng0fic3.github.io/images/xitong4.png)

打开EasyBCD，依次进行：添加新条目 → NeoGrub → 安装：

![avatar](https://k1ng0fic3.github.io/images/xitong5.png)

再进行配置，我这里是这样的：

![avatar](https://k1ng0fic3.github.io/images/xitong6.png)

>title Install Ubuntu 
root (hd0,4) 
kernel (hd0,4)/vmlinuz boot=casper iso-scan/filename=/ubuntu-18.04.3-desktop-amd64.iso ro quiet splash locale=zh_CN.UTF-8 
initrd (hd0,4)/initrd

#### 磁盘编号

![avatar](https://k1ng0fic3.github.io/images/xitong7.png)

我这里把三个文件放到了D盘下，如上图，在硬盘中位于第一逻辑分区，所以定位在`(hd0,4)`。
主分区与逻辑分区的编号规则如下：

![avatar](https://k1ng0fic3.github.io/images/xitong8.png)

保存后查看引导菜单，一定记得保存设置！：

![avatar](https://k1ng0fic3.github.io/images/xitong9.png)

#### 进入引导

重启计算机，进入引导界面，选择NeoGrub引导加载器，开始安装Ubuntu

![avatar](https://k1ng0fic3.github.io/images/xitong10.jpg)

进入后，使用ctrl + alt + T打开终端，输入以下命令卸载分区：
>sudo umount -l /isodevice

![avatar](https://k1ng0fic3.github.io/images/xitong14.png)

之后运行桌面上的系统安装程序

![avatar](https://k1ng0fic3.github.io/images/xitong15.jpg)

具体过程略，要注意一下磁盘的分区安排，我的空间是200G多一些，如下：

目录 | 描述 | 大小 | 格式
-|-|-|-
/ | 为根目录，逻辑分区 | 96G+ | ext4
swap | 交换空间，一般为物理内存的两倍，逻辑分区 | 16G | swap
/home | 用户工作目录，含个人配置文件，逻辑分区 | 96G+ | ext4
/boot | 空间起始位置，应大于400M或1G，主分区 | 4G | ext4

![avatar](https://k1ng0fic3.github.io/images/xitong10.png)

安装启动引导器的设备指定`/boot`所在分区，之后一步一步安装即可

![avatar](https://k1ng0fic3.github.io/images/xitong11.png)

#### 清理引导

安装完之后，重启进入Windows系统，打开EasyBCD，添加Ubuntu启动项，（我的是GRUB(Legacy)）

![avatar](https://k1ng0fic3.github.io/images/xitong12.png)

并删除NeoGrub引导项，删除根目录下的iso等三个文件，预期效果：

![avatar](https://k1ng0fic3.github.io/images/xitong13.png)

![avatar](https://k1ng0fic3.github.io/images/xitong14.jpg)

当时选择的是中文安装Ubuntu，但是因为在终端操作时，中文的路径非常之难用，可在语言支持中把English拖动到中文之上，注销登出一下即可，然后再update一下就OK了~