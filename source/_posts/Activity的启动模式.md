---
title: Activity的启动模式
date: 2018-05-02 21:26:43
categories: 
- Android
- 基础
tag: 
- Android
- 基础
---

#### LaunchMode

四种启动模式：standard、singleTop、singleTask、singleInstance

##### standard

标准模式，系统默认的启动模式。每次启动一个Activity都会重新创建一个新的实例。在这种模式下，谁启动这个Activity,那么这个Activity就运行在启动它的那个Activity所在的栈中。

##### singleTop

栈顶复用模式，在这种模式下，如果新的Activity已经位于任务栈的栈顶，那么此Activity不会被重新创建，同时它的onNewIntent方法会被回调，通过此方法的参数可以取出当前的请求信息。如果新的Activity实例已存在但不是位于栈顶，那么新的Activity仍然会被重新创建。

##### singleTask

栈内复用模式，在这种模式下，只要Activity在一个栈中存在，那么多次启动此Activity都不会重新创建实例，和singleTop一样，系统也会回调onNewIntent。

##### singleInstance

单实例模式，这是一种加强的singleTask模式，具备此模式的Activity只能单独位于一个任务栈中。

#### TaskAffinity

这个参数标识一个Activity所需要的任务栈的名字，默认情况下所有Activity所需的任务栈的名字是应用的包名。

TaskAffinity主要和singleTask启动模式或者allowTaskReparenting属性配对使用。

#### 给Activity指定启动模式

1、通过AndroidMenifest为Activity指定启动模式；

2、通过在Intent中设置标志位。

这两种启动模式第二种的优先级高于第一种，当两种同时存在，以第二种为准；另外，两种方式在限定范围上有所不同，第一种无法直接为Activity设定FLAG_ACTIVITY_CLEAR_TOP标识，第二种不能指定singleInstance模式。

#### Activity的Flags

##### FLAG_ACTIVITY_NEW_TASK

为Activity指定singleTask启动模式，效果和XML中指定该启动模式相同。

##### FLAG_ACTIVITY_SINGLE_TOP

为Activity指定singleTop模式

##### FLAG_ACTIVITY_CLEAR_TOP

具有此标记的Activity,当它启动时，在同一任务栈中的所以位于它上面的Activity都要出栈，这种模式一般和FLAG_ACTIVITY_NEW_TASK配合使用。

#### IntentFilter的匹配规则

IntentFilter的过滤信息有action、category、data，action的匹配规则只要有一个action成功匹配就可以，category每个都要和过滤规则中的相同。

data由两部分最初，mineType和URI,mineType指媒体类型，如image/jpeg、video/*等，

URI结构：

```
<scheme>://<host>:<port>/[<path>|[pathPrefix]|[pathPattern]]
```

过滤规则中没有红的URI时默认content和file,也就是说，虽然没有指定URI,但是Intent的URI部分的scheme必须为content或者file才能匹配。