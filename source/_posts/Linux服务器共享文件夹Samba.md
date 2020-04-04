---
title: Linux服务器共享文件夹Samba
date: 2020-03-21 14:24:15
categories: Linux
tag: Linux
---
[Windows访问Linux服务器共享文件夹--Samba](https://blog.csdn.net/Utotao/article/details/100848930)

#### 1、Linux安装Samba及配置
##### 安装Samba：

```
sudo apt-get install samba
```
##### 配置Samba：
```
sudo vi /etc/samba/smb.conf
```
##### 配置内容：


```
[share]
    #comment = Ubuntu File Server Share
    path = /home/ubuntu
    valid users = root
    available = yes
    browsable = yes
    writable = yes
```
配置用户名密码：（windows访问share文件夹时候使用）

```
sudo smbpasswd -a root -> 注意此处的用户名要和上面的valid users保持一致
```

回车后会让你输入密码  
设置共享文件夹权限：

```
sudo chmod 777 /home/ubuntu
```

重启：


```
sudo /etc/init.d/samb restart
```
### 2、windows访问共享文件夹
网络—>右键选择映射网络驱动器，填写IP+Samba配置文件中的[share]之[]里面的字样