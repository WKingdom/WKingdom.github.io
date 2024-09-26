---
title: system.img打包出错设备上没有空间
date: 2020-06-14 14:49:19
categories: 
- Android
- 系统源码
tag: 
- Android
- 系统源码
---
在打system.img的时候提示出错: 设备上没有空间

在现有android系统时，用adb工具把程序放入系统中

cmd 

cd 到adb的目录下

adb push xxx.xml etc/

当我们要制作系统镜像时，可以使用


```
mkdir system

sudo mount -o loop system.img system



cp -rf xxx.xml system/etc



sudo umount system
```




执行cp时候，提示设备上没有空间，这时候，可以


```
sudo apt-get install e2fsprogs

e2fsck -f system.img
resize2fs system.img 1000M
sudo mount -o loop system.img system


cp xxx



sudo umount system
e2fsck -f system.img
resize2fs -M system.img
```
最后需要修改内存分配配置。