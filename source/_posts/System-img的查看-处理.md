---
title: System.img的查看-处理
date: 2020-06-04 14:49:04
categories: 
- Android
- 系统源码
tag: 
- Android
- 系统源码
---
在服务器上编译完源码之后，有时候需要想要查看打出img的内容是不是自己想要的，比如说添加了一个新的apk文件，如果重新烧写到手机会浪费很多时间，想在服务器上直接查看，可以参考如下方法。
[System.img的查看/处理](https://blog.csdn.net/sir_zeng/article/details/51983432)


```
file out/target/product/px3/system.img

out/host/linux-x86/bin/simg2img out/target/product/px3/system.img system.img
file system.img
sudo mount -o loop system.img /mnt
ls -lh /mnt/
```
解包system.img
simg2img  
simg2img用于把压缩过/hash过的img文件还原为raw的img文件  
它通常在编译输出的out/host/linux-x86/bin/simg2img中  
查询原始system.img文件类型,可以看到是data  

```
file out/target/product/msm8909/system.img
out/target/product/msm8909/system.img: data
转换命令如下

Usage: simg2img <sparse_image_files> <raw_image_file>
out/host/linux-x86/bin/simg2img out/target/product/msm8909/system.img out/target/product/msm8909/systemraw.img
```

查询下转换后的systemraw.img文件类型  

```
file out/target/product/msm8909/systemraw.img
out/target/product/msm8909/systemraw.img: Linux rev 1.0 ext4 filesystem data, UUID=57f8f4bc-abf4-655f-bf67-946fc0f9f25b (extents) (large files)
```

[编辑] mount systemraw.img  
这个systemraw.img就可以任意我们处理了,最好的处理方法是直接mount它,然后进去看/处理内容,如果直接解开,很容易丢失了软链接,甚至会到是selinux的权限错乱


```
mkdir aaa
sudo mount -t ext4 -o loop out/target/product/msm8909/systemraw.img aaa
```

[编辑] 处理mount的img  
[编辑] 查看占用的空间  
df -h  
输出类似  


```
Filesystem      Size  Used Avail Use% Mounted on
/dev/loop0      2.0G  450M  1.5G  23% /media/work_r/szgit/8909/LINUX/android/aaa
```

[编辑] 其他处理  
其他的诸如 chmod rm cp mv   chown想怎样就可以怎样了,随便修改  

[编辑] 保存处理mount的img 
umount就ok,自动保存了  

sudo umount aaa  
[编辑] 打包为system.img  
[编辑] img2simg  
img2simg用于把img文件打包  
它通常在编译输出的out/host/linux-x86/bin/img2simg中  
转换命令如下  


```
Usage: img2simg <raw_image_file> <sparse_image_file> [<block_size>]
out/host/linux-x86/bin/img2simg out/target/product/msm8909/systemraw.img out/target/product/msm8909/system.img
```

转换出来 system.img和之前编译的没有啥区别,直接用fastboot就可以刷机  

[编辑] 其他  
如果你找不到img2simg/simg2img,自己编译下,或者从其他项目中copy过来  


```
mmma system/core/libsparse/   
或者  
mmm system/core/libsparse/
```
