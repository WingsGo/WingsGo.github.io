---
title: Windows10+Ubuntu16.04双系统安装
key: 20181026
tags:
- Windows10 Ubuntu 双系统
---

## 一、前言
<!--more-->
&nbsp;&nbsp;&nbsp;&nbsp;作为一个开发人员，Linux下的开发是一项必备的技能，但一些日常娱乐、办公还是在Windows下更加方便，为了解决这个问题，笔者查阅了一些博客，成功配置Windows10+Ubuntu双系统，在此做一个记录与分享，细节部分可查阅参考文献，本文仅对资料做一个整合，将自己遇到的一些坑记录下来。

## 二、准备知识
&nbsp;&nbsp;&nbsp;&nbsp;首先，我们要对磁盘分区及系统启动有个基本的了解。磁盘的分区格式有MBR与GPT分区，MBR分区是比较老的分区方式，其中第一个扇区含有主要开机区及分区表，主要开机区占446Bytes,分区表占64Bytes,这决定了它支持的磁盘容量不超过2TB，而GPT分区是一种较新的分区格式，它支持的磁盘容量也远大于2TB，具体关于GPT与MBR的与别可以参考《鸟哥的Linux私房菜-基础学习篇》第七章。

&nbsp;&nbsp;&nbsp;&nbsp;目前来说，主要的两种操作系统的启动方式有BIOS+MBR与UEFI+GPT，在安装系统时要确定你自己的硬盘属于哪种分区，开机模式是属于哪一种，否则很有可能进入不了系统，并且在安装Ubuntu时也会遇到问题。

## 三、安装步骤
&nbsp;&nbsp;&nbsp;&nbsp;首先根据[实用教程：PC实现Win10/Ubuntu双系统](https://www.ithome.com/html/win10/303077.htm)将硬盘进行分区，安装Ubuntu,并在BIOS中设置好选项。同时在U盘写入Ubuntu的iso镜像。根据步骤一步一步进行即可，关于磁盘分区需要注意两点，一个就是主分区的数量最多为4个，所以分区的时候需要注意逻辑分区与主分区的选择，如果遇到无法分区的情况就要考虑是不是主分区已经达到了四个。另一个就是在安装过程中会提示是否在强制在UEFI模式下运行，如果你的分区格式为MBR，那么就点击后退，继续安装。

## 四、开机选项设置
&nbsp;&nbsp;&nbsp;&nbsp;在安装好Ubuntu后，系统开机会进入Ubuntu的开机选项，此时选项中没有Windows系统，是因为ubuntu的开机程序grub中没有找到Windows的开机选项，我们需要修改一下配置文件，具体步骤参考[解決windows10和Ubuntu16.04双系统后windows10不能正常启动](https://blog.csdn.net/oreooo/article/details/60141864)。此时双系统就安装好了，我们开机时可以选择Ubuntu还是Windows10开机。


## 参考文献
1. [实用教程：PC实现Win10/Ubuntu双系统](https://www.ithome.com/html/win10/303077.htm)
2. [win10+ubuntu双系统配置](https://blog.csdn.net/qq_31192383/article/details/78876905)
3. [解決windows10和Ubuntu16.04双系统后windows10不能正常启动](https://blog.csdn.net/oreooo/article/details/60141864)