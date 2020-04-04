---
title: 使用wpa_supplicant, wpa_cli连接wifi
date: 2020-04-04 14:48:50
categories: 
- Android
- 系统源码
tag: 
- Android
- 系统源码
---
WIFI on:

adb shell svc wifi enable

WIFI off:

adb shell svc wifi disable


[使用wpa_supplicant, wpa_cli连接wifi](http://blog.sina.com.cn/s/blog_15e89db360102x7qv.html)

如果对应的命令没有需要从源码编译后，导入到/system/bin目录下，修改权限755  
dhcpcd目录 \external\dhcpcd-6.8.2  
wpa_cli  \external\wpa_supplicant_8 编译后会生成该文件  


```
wpa_supplicant.conf目录  
/data/misc/wifi/wpa_supplicant.conf
```

```
#/system/bin/sh
ssid=${1}
psk=${2}
svc wifi enable
sleep 6
wpa_cli scan
wpa_cli scan_result
addID=`wpa_cli add_net | sed -n '2p'`

echo $addID
wpa_cli set_net ${addID} ssid "\"${ssid}\""
wpa_cli set_net ${addID} psk "\"${psk}\""
wpa_cli select_net ${addID}
wpa_cli enable_net ${addID}

dhcpcd wlan0
```

debug WifiStateMachine 程序需要debug system process进程



