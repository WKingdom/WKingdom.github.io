---
title: jni无法访问so提示not accessible
date: 2020-04-04 14:49:37
categories: 
- Android
- 系统源码
tag: 
- Android
- 系统源码
---
问题：app需要引用系统的so库，当install形式的时候安装时，打开app需要使用/system/lib64目录下的so库时，提示不能访问。
原因分析：放到system/app下的app是可以找到的，但是普通的安装形式的app是没有权限访问的，所以需要需要声明公有so库才能使用。

```
library "/system/lib64/libserialport.so" ("/system/lib64/libhqbindcs.so") needed or
dlopened by"/system/lib64/libhqbindcs.so" is not accessible for the namespace
"classloader-namespace"
```
解决方法：  
修改system/core/rootdir/etc/public.libraries.txt  
添加你要使用的的so到此问题件中。

```
cat public.libraries.txt 
....
libz.so
libhqbindcs.so
```

也可以直接不编译在板子上修改，对应目录/system/etc  

参考：  
Framework基础：Android N 公共so库怎么定义呢？  
https://www.jianshu.com/p/4be3d1dafbec

[Google]原生库的命名空间  
https://source.android.com/devices/tech/config/namespaces_libraries