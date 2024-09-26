---
title: 安卓系统签名转keystore
date: 2019-10-18 22:55:00
categories: 
- Android
- 系统源码
tag: 
- Android
- 系统源码
---
 单独签名解决方案


```
找到平台签名文件“platform.pk8”和“platform.x509.pem”

文件位置 android/build/target/product/security/
```




```
签名工具“signapk.jar”

位置：android/prebuilts/sdk/tools/lib
```




```
签名证书“platform.pk8 ”“platform.x509.pem ”，签名工具“signapk.jar ”放置在同一个文件夹；
```

 下载 keytool-importkeypair 工具，使用sdk的security文件生成对应平台的key：


```
./keytool-importkeypair -k [jks文件名] -p [jks的密码] -pk8 platform.pk8 -cert platform.x509.pem -alias [jks的别名]

如：
./keytool-importkeypair -k ./SignDemo.jks -p 123456 -pk8 platform.pk8 -cert platform.x509.pem -alias SignDemo
```
