---
title: Perfetto使用
date: 2023-8-24 22:26:43
categories: 
- Android
- 优化工具
tag: 
- Android
- 优化工具
---
 Perfetto是google从Android10开始引入的一个全新的平台级跟踪分析工具。提供了用于记录系统级和应用级活动的服务和库、低开销的native+java内存分析工具，可供SQL分析跟踪文件的库，以及一个基于Web用于将追踪文件可视化方便分析的Perfetto UI。相比systrace的优势:

- 其可记录任意长度的跟踪记录并导出到文件系统中.

- 更合理的可视化分析标记功能.

- 内建SQLite数据库,SQL查询的支持,数据后期处理非常灵活.


#### 命令行抓取方式

```
adb shell perfetto -o /data/misc/perfetto-traces/trace_file.perfetto-trace -t 10s sched freq idle am wm gfx view binder_driver hal dalvik camera input res memory
adb pull /data/misc/perfetto-traces/trace_file.perfetto-trace 
```
网站打开trace：https://ui.perfetto.dev/

**针对可能不支持perfetto版本可以，离线抓取atrace方式**

```
//开始抓取
adb shell "atrace -c  -t 10 sched freq idle am wm gfx view binder_driver hal dalvik camera input res gfx view wm am ss video camera hal res sync idle binder_driver binder_lock ss  --async_start"
//停止抓取
adb shell atrace --async_stop -z -c -o /sdcard/atrace_normal.atrace
```

**AOSP中自带的record_android_trace工具**

具体的record_android_trace路径是在aosp源码的目录aosp/external/perfetto/tools

```
./record_android_trace -o $(date +%Y%m%d_%H%M%S)_trace_file.perfetto-trace -t 5s -b 32mb sched freq idle am wm gfx view binder_driver hal dalvik camera input res memory gfx view wm am ss video camera hal res sync idle binder_driver binder_lock ss
```

抓取后会直接打开相关的chrome的浏览器。

第三方app打印trace

```java
Trace.beginSection(String sectionName)；//sectionName代表打印trace的名字，一般可以写成当前方法等
Trace.endSection()；//trace代表结束，与上面方法成对出现
```

加入参数-a 加包名
