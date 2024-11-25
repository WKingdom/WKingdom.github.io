---
title: 添加系统服务及安全机制
date: 2021-9-22 23:26:43
categories: 
- Android
- 系统源码
tag: 
- Android
- 系统源码
---

常见的AMS、PWS、WMS等都是系统服务，运行于system_server进程，并且向ServiceManager进程注册其Binder以便其他进程获取binder与对应的服务进行通信。为了新增自定义系统服务，我们可以参考AMS等原生系统 服务，如新增 TestManagerService ： 

1. ITestManager.aidl 文件：生成Binder类，其中Stub即为Binder的服务端；
2. TestManagerService：系统服务类，继承自Stub； 
3. TestManager：封装了AIDL接口方法的类，相当于Binder客户端（Proxy），其他进程通过此类完成与系统服务的通信。

### 源码中添加自定义系统服务

##### 1、ITestManager.aidl

在frameworks/base/core/java/android/app中编写 ITestManager.aidl

```java
// ITestManager.aidl
package android.app;

/**
 * System private API for talking with the activity manager service.  This
 * provides calls from the application back to the activity manager.
 *
 * {@hide}
 */
interface ITestManager {
    String request(String msg);
}
```

##### 2、TestManager.java

在frameworks/base/core/java/android/app 下编写TestManager.java

```java
package android.app;

import android.annotation.SystemService;
import android.compat.annotation.UnsupportedAppUsage;
import android.content.Context;
import android.os.IBinder;
import android.os.RemoteException;
import android.annotation.Nullable;
import android.os.ServiceManager;
import android.util.Singleton;

@SystemService(Context.LANCE_SERVICE)
public class TestManager{
    private Context mContext;

    /**
     * @hide
     */
    public TestManager() {

    }
    /**
     * @hide
     */
    public static ITestManager getService() {
        return ILanceManagerSingleton.get();
    }

    @UnsupportedAppUsage
    private static final Singleton<ITestManager> ITestManagerSingleton =
            new Singleton<ITestManager>() {
                @Override
                protected ITestManager create() {
                    final IBinder b = ServiceManager.getService(Context.TEST_SERVICE);
                    final ITestManager im = ITestManager.Stub.asInterface(b);
                    return im;
                }
            };

    @Nullable
    public String request( @Nullable String msg) {
        try {
            return getService().request(msg);
        } catch (RemoteException e) {
            throw e.rethrowFromSystemServer();
        }
    }
}
```

##### 3、修改Context 

在frameworks/base/core/java/android/content/Context.java中加入常量：

```java
/** @hide */
    @StringDef(suffix = { "_SERVICE" }, value = {
        //......
        ACTIVITY_SERVICE,
        TEST_SERVICE, 
        //......          
    })
    @Retention(RetentionPolicy.SOURCE)
    public @interface ServiceName {}
public static final String TEST_SERVICE="test";
```

##### 4、TestManagerService.java

在frameworks/base/services/core/java/com/android/service/test中编写TestManagerService.java

```java
package com.android.server.test;

import android.annotation.Nullable;
import android.app.ITestManager;
import android.os.RemoteException;

public class TestManagerService extends ITestManager.Stub {
    @Override
    public String request(String msg) throws RemoteException {
        return "TestManagerService接收数据:" + msg;
    }
}
```

##### 5、ServiceManager注册

在frameworks/base/services/java/com/android/server/SystemServer.java中注册系统服务

```java
import com.android.server.test.TestManagerService;
private void startOtherServices(){
    //......
    ServiceManager.addService(Context.TEST_SERVICE,new TestManagerService());
    //......
}
```

##### 6、SystemServiceRegistry注册

在frameworks/base/core/java/android/app/SystemServiceRegistry.java注册服务获取器：

```java
import android.app.TestManager;
import android.app.ITestManager;
static{
    registerService(Context.TEST_SERVICE, TestManager.class,
                new CachedServiceFetcher<TestManager>() {
            @Override
            public TestManager createService(ContextImpl ctx) throws ServiceNotFoundException {
                return new TestManager();
            }});
}
```

##### 7、配置SELinux权限 

在system/sepolicy/prebuilts/api/31.0/private/与system/sepolicy/private/目录下，分别修改：

```
注意：两个目录下文件需要一致，否则报错，如：
Files system/sepolicy/prebuilts/api/31.0/private/untrusted_app_all.te and
system/sepolicy/private/untrusted_app_all.te diﬀer
ninja failed with: exit status 1
```

**service_contexts**

```
activity                                  u:object_r:activity_service:s0                
#配置自定义服务selinux角色      
test                                     u:object_r:test_service:s0
```

service.te:

```
#配置自定义服务类型
type test_service, app_api_service, ephemeral_app_api_service, system_server_service, 
service_manager_type;
```

untrusted_app_all.te :

```
#允许所有app使用自定义服务
allow untrusted_app_all test_service:service_manager find;
```

8、更新并编译

```
# 更新api
make update-api
# 编译
make
```

编译成功后，启动模拟器或者烧写设备后命令行查看服务是否：

```
adb shell service list| grep test
#输出： 表示成功加入自定义服务
106 test: [android.app.ITestManager]
```

##### 使用自定义服务

1、双亲委托机制，在需要使用自定义服务的app中编写TestManager（包名与framework中一致），方法名空实现。app中的
TestManager仅仅只是为了编译成功编写的空壳。

2、修改app使用的sdk，可以通过make sdk 将SDK完整编译出来，在AS中使用自编译出的SDK即可，也可以将原生SDK中的android.jar替换。

### 安全机制

```
SELinux: avc: denied { add } for service=cvnavi_storage pid=3933 uid=1000 scontext=u:r:system_app:s0 tcontext=u:object_r:default_android_service:s0 tclass=service_manager permissive=0
```
系统进程添加一个服务到系统进程调用ServiceManager.addService(name, service);系统中抛出上面的错误。

通过错误信息看出可能是没有添加到系统服务的权限，SELinux拒绝添加。然后通过命令行验证是否是SELinux安全策略导致的这个问题。

```
setenforce 0 （1：Enforcing 0：Permissive）临时禁用掉SELinux，5.1版本默认强制开启；

getenforce （得到结果为Permissive）
```

禁用掉SELinux后发现功能可以正常使用，就是这个问题导致的，直接关掉是不可能的，容易出问题，需要想办法解决。

上面的这个属于Android中SEAndroid安全机制（MAC）强制访问控制。

Android中自主访问控制是通过Linux UID/GID实现，而强制访问控制则是使用的SEAndroid。

在Android中SEAndroid安全机制（MAC）与传统的Linux UID/GID安全机制（DAC）是并存关系的，也就是说，它们同时用来约束进程的权限。

**当一个进程访问一个文件的时候，首先要通过基于UID/GID的DAC安全检查，接着才有资格进入到基于SEAndroid的MAC安全检查。只要其中的一个检查不通过，那么进程访问文件的请求就会被拒绝。**



#### DAC自主访问控制

自主访问控制，正式的英文名称为Discretionary Access Control，简称为DAC。

比如通过ls -l /system，可以查看到该目录下存在一个manifest.xml文件，其输出为

```java
-rwxr-x---   1 root root   2544 2021-09-22 22:02 manifest.xml
```

表示 manifest.xml是root用户组的root用户拥有，对于root用户来说，是rwx（可读可写可执行）；而对于root用户组其他用户来说，是可读可执行；对于其他用户则没有任何权限；也就是750权限。

Android是一个基于Linux内核的系统，但是它不像传统的Linux系统，需要用户登录之后才能使用。然而，Android系统又像传统的Linux系统一样有用户的概念。只不过这些用户不需要登录，也可以使用Android系统，这是因为Android系统将每一个安装在系统的APK都映射为一个不同的Linux用户。也就是说，每一个APK都有一个对应的UID和GID。这些UID和GID是在APK安装的时候由系统安装服务PMS分配的：

```java
//framework/base/services/core/java/com/android/server/pm/PackageManagerService.java
private PackageParser.Package scanPackageDirtyLI(PackageParser.Package pkg,
                                                 final int policyFlags, final int 
scanFlags, long currentTime, @Nullable UserHandle user)
        throws PackageManagerException {
    ......
 
        if (pkgSetting == null) {
            .......................
            mSettings.addUserToSettingLPw(pkgSetting);
        }
    ......
    return pkg;
}
//framework/base/services/core/java/com/android/server/pm/Settings.java
void addUserToSettingLPw(PackageSetting p) throws PackageManagerException {
    if (p.appId == 0) {
        // 分配uid
        p.appId = newUserIdLPw(p);
    } 
    ......
}

private int newUserIdLPw(Object obj) {
    final int N = mUserIds.size();
    //从0开始，找到第一个未使用的ID，此处对应之前有应用被移除的情况，复用之前的ID
    for (int i = mFirstAvailableUid; i < N; i++) {
        if (mUserIds.get(i) == null) {
            mUserIds.set(i, obj);
            return Process.FIRST_APPLICATION_UID + i;
        }
    }

    //最多只能安装 9999 个应用
    if (N > (Process.LAST_APPLICATION_UID-Process.FIRST_APPLICATION_UID)) {
        return -1;
            }

    mUserIds.add(obj);
    return Process.FIRST_APPLICATION_UID + N;
}
```

在完成安装并运行程序后，可以通过ps -A | grep PACKAGENAME 查看程序uid

```
ps -A | grep xxx
u0_a72        7124   562 2402748 170132 SyS_epoll_wait 78511081d4 S xxx
```

得到当前程序进程ID为7124，然后通过cat /proc/7124/status查看

```
cat /proc/7124/status
#输出
Name:   xxxxx
State:  S (sleeping)
Tgid:   7124
Pid:    7124
PPid:   562
TracerPid:      0
Uid:    10072   10072   10072   10072
Gid:    10072   10072   10072   10072
FDSize: 64
Groups: 9997 20072 50072
```

可以看到当前程序UID为10072，通过这种方式，就可以保证每一个APK进程都以不同的身份来运行，从而保证了相互之间不会受到干扰。这就是所谓的沙箱了，这完全是建立在Linux的UID和GID基础上的。

root的UID/GID可以通过下面方式查看：

```c++
//system/core/include/private/android_ﬁlesystem_conﬁg.h
#define AID_ROOT 0 /* traditional unix root user */
#define AID_SYSTEM 1000 /* system server */
......
#define AID_APP 10000       /* TODO: switch users over to AID_APP_START */
#define AID_APP_START 10000 /* first app user */
#define AID_APP_END 19999   /* last app user */
```

可以看到，普通APP的UID从10000开始分配，最大到19999，而root用户的id为0。

##### 应用权限与DAC的关系

如何才能让我们的进程能通过Linux UID/GID的拦截呢？如果是一个Android APP若让其具备网络权限，只需要在AndroidManifest.xml中配置:

```java
<uses-permission android:name="android.permission.INTERNET"/>
```

APP的UID和GID是在安装时候就由PMS分配好了的。为什么这样一个配置就能够让程序通过Linux UID/GID的拦截？

这是因为PMS在安装APK时，从Manifest文件中把App信息和权限存到/data/system/packages.xml和/data/system/packages.list文件中。打开packages.list会存在下面的记录：

```
com.baidu.homework 10051 0 /data/user/0/com.baidu.homework default:targetSdkVersion=26 3002,3003,3001
```

其中3002,3003,3001代表的就是用户组，通过/system/core/include/private/android_filesystem_config.h 查看可知

```c++
#define AID_NET_BT_ADMIN 3001 /* bluetooth: create any socket */
#define AID_NET_BT 3002       /* bluetooth: create sco, rfcomm or l2cap sockets */
#define AID_INET 3003         /* can create AF_INET and AF_INET6 sockets */
```

3001与3002代表了具备蓝牙相关权限的用户组，而3003则表示具备网络权限的用户组。PMS会将APK加入到相应的某个Linux用户组去，这样APK才能够具备对应的权限。

为了通过DAC的权限检查具备网络权限，以FDBus为例，有网络权限，需要将name-server指定到3003用户组。

在/framework/base/data/etc/platform.xml 中可以看到：

```java
<permission name="android.permission.NET_TUNNELING" >
    <group gid="vpn" />
</permission>

<permission name="android.permission.INTERNET" >
    <group gid="inet" />
</permission>

<permission name="android.permission.READ_LOGS" >
    <group gid="log" />
</permission>

<permission name="android.permission.WRITE_MEDIA_STORAGE" >
    <group gid="media_rw" />
    <group gid="sdcard_rw" />
</permission>
```

inet 就是网络权限组，即：inet就是3003。因此，我们配置init.rc内容如下：

```
service name-server /system/bin/name-server -u tcp://104.129.181.88:60002 -n android
    #指定进程类属，类属的作用是方便操作多个服务同时启动或停止
    class core
    # 进程 用户组为inet （也可以直接给system，system组也具备网络权限）
    group inet
    # 进程id 写入到tasks
    writepid /dev/cpuset/system-background/tasks
```

在理想情况下，DAC机制是没有问题的。如果将某个系统文件的权限改为777，只有DAC机制那么任何用户都能具备对该文件的读写以及执行权限，可能造成严重的安全问题。为了解决这个问题，Android中还使用了一种更为强有力的安全机制来保证系统的安全，这种机制就是MAC。



#### MAC强制访问控制

Android中使用的MAC机制就是SEAndroid。SELinux(Security-Enhanced Linux) 是美国国家安全局（NSA）在Linux社区的帮助下设计的一个Linux历史上最杰出的安全系统，是一种MAC机制（Mandatory Access Control，强制访问控制）。在这种访问控制体系的限制下，进程只能访问那些在他的任务中所需要文件。由于Android系统有着独特的用户空间运行时，因此SELinux不能完全适用于Android系统。为此，NSA针对Android系统，在SELinux基础上开发了SEAndroid。

#### SEAndroid权限配置

SEAndroid安全机制又称为是基于TE（Type Enforcement）策略的安全机制。在/system/sepolicy目录中，所有以.te为后缀的文件均为策略配置文件。

##### 安全上下文

SEAndroid安全机制中的安全策略是在安全上下文的基础上进行描述的，也就是说，它通过主体和客体的安全上下文，定义主体是否有权限访问客体。主体通常就是进程，而客体就是指进程所要访问的资源，例如文件、系统属性等。

由于我们的服务程序可执行为/system/bin/name-server，首先我们需要在/system/sepolicy/private/ﬁle_contexts中声明该执行文件的SEAndroid系统文件的安全上下文

```
/system/bin/name-server     u:object_r:name-server_exec:s0
```

安全上下文实际上就是一个附加在对象上的标签（Tag）。这个标签实际上就是一个字符串，它由四部分内容组成，
**分别是SELinux用户、SELinux角色、类型、安全级别，每一个部分都通过一个冒号来分隔，格式为"user:role:type:sensitivity"**

在安全上下文中，只有类型（Type）才是最重要的，SELinux用户、SELinux角色和安全级别都几乎可以忽略不计的。正因为如此，SEAndroid安全机制又称为是基于TE（Type Enforcement）策略的安全机制。

##### 用户与角色

在/system/sepolicy/private/users中声明了SELinux用户u，它可用的SELinux角色为r，它的默认安全级别为s0，可用的安全级别范围为s0 - mls_systemhigh：

```
user u roles { r } level s0 range s0 - mls_systemhigh;
```

mls_systemhigh为系统定义的最高安全级别

在/system/sepolicy/public/roles中声明了SELinux角色r与类型domain关联：

```
role r types domain;
```

在SEAndroid中，只定义了唯一一个用户u，两个角色r（适用于主题，如进程）与object_r(适用于对象，如文件)，这意味着只有u、r/object_r和domain可以组合在一起形成一个合法的安全上下文，那么ps -Z查看进程应为u:r:domain:s0 ，ls -Z查看文件则为： u:object_r:domain:s0，而其它形式的安全上下文定义均是非法的。

以init进程为例，执行 adb shell ps -Z | grep zygote，可以看到输出为：

```
u:r:zygote:s0 root 646     1 1577904  69500 poll_schedule_timeout eb7e743c S zygote
```

安全上下文u:r:zygote:s0，按照上面的分析，这不是应该是一个不合法的上下文吗？原因是在/system/sepolicy/public/zygote.te 中通过type声明了类型zygote并且将domain设置为类型zygote的属性：

```
type zygote, domain;
type zygote_tmpfs, file_type;
type zygote_exec, system_file_type, exec_type, file_type;
```

因此它就可以像domain一样，可以和SELinux用户u和SELinux角色组合在一起形成合法的安全上下文。

##### 类型

在SEAndroid中，每一个用来描述文件安全上下文的类型都将ﬁle_type设置为其属性，每一个用于进程安全上下文的类型都将domain设置为其属性。

##### 安全策略

SEAndroid安全机制主要是使用对象安全上下文中的类型来定义安全策略，这种安全策略就称TypeEnforcement，简称TE，.te文件即为安全策略配置文件。

system/sepolicy/public：公共策略配置，将此目录视为相应平台的已导出政策 API，包括供应商特定策略。
system/sepolicy/private：系统正常运行所必需（但供应商映像政策应该不知道）的策略。

对于权限的配置不建议直接修改以上目录，应该在/device/manufacturer/device-name/sepolicy 目录进行自己设备的专用策略配置。
如Pixel5手机搭载AAOS，则应该在：/device/google_car/redfin_car/sepolicy目录下配置。同时修改或添加政策文件和上下文的描述文件后，需要修改 /device/manufacturer/device-name/BoardConfig.mk 以引用 sepolicy 子目录和每个新的政策文件。

```java
BOARD_SEPOLICY_DIRS += \
     <root>/device/manufacturer/device-name/sepolicy

BOARD_SEPOLICY_UNION += \
     genfs_contexts \
     file_contexts \
     sepolicy.te
```

以FDBUS name-server为例，在/system/sepolicy/private目录下创建一个name-server.te作为该进程的策略文件。文件内容如下：

```java
# 声明name-server类型，并将domain属性关联到该类型 （进程）
type name-server, domain; 
# 声明name-server_exec类型（可执行文件，类型要与file_contexts中的相同）
type name-server_exec, exec_type, file_type;

#allow语句表示允许的权限。
allow name-server self:tcp_socket { read write getattr getopt setopt shutdown create bind 
connect name_connect };
allow name-server self:netlink_route_socket {create write read nlmsg_readpriv nlmsg_read};
allow name-server fwmarkd_socket:sock_file {write};
allow name-server port:tcp_socket {name_connect};
allow name-server netd:unix_stream_socket {connectto};
allow name-server self:capability {net_raw};
allow name-server node:tcp_socket {node_bind};

#init_daemon_domain：system/sepolicy/public/te_macros中定义的宏（函数） 进行默认的一些配置
#domain_auto_trans（init,name-server_exec,name-server）
#   domain_trans(init,name-server_exec,name-server)
#       allow init name-server_exec:file { getattr open read execute };
#       allow init name-server:process transition;
#       allow name-server name-server_exec:file { entrypoint open read execute getattr };
#   type_transition init name-server_exec:process name-server;
# ......
#声明 name-server 是从 init 衍生而来的，并且可以与其通信
init_daemon_domain(name-server)
```

##### Allow规则 

allow表示开放权限，当某个进程执行，如果该进程不具备对应的权限，则可以在logcat 或者在执行adb root 后执行 adb shell dmesg查看。

```java
avc: denied { bind } for pid=417 comm="name-server" scontext=u:r:name-server:s0 
tcontext=u:r:name-server:s0 tclass=tcp_socket permissive=0
    
avc: denied { connectto } for pid=417 comm="name-server" scontext=u:r:name-server:s0 
tcontext=u:r:netd:s0 tclass=unix_stream_socket permissive=0   
```

对于上面日志的分析步骤如下：

- 缺少什么权限 ：denied { bind } 
- 谁缺少权限：scontext=u:r:name-server:s0 
- 对谁缺少权限 ：tcontext=u:r:name-server:s0
- 什么类型的权限：tclass=tcp_socket

需要添加对应的权限，在TE文件中声明：

```java
#allow [谁缺少权限]  [对谁缺少权限]:[什么类型的权限] [缺少什么权限]
#当sconext与tcontext都是自己时候，可以将 [对谁缺少权限]写为：self
allow name-server self:tcp_socket {bind}
allow name-server netd:unix_stream_socket {connectto};
```

##### 宏函数

在system/sepolicy/public/te_macros中定义了很多宏函数，其中init_daemon_domain表示声明 name-server是从 init 衍生而来的，并且可以与其通信。
另外还有其他宏如：net_domain，其定义为：

```
#####################################
# net_domain(domain)
# Allow a base set of permissions required for network access.
define(`net_domain', `
typeattribute $1 netdomain;
')
```

如果调用该宏：net_domain(name-server)表示将name-server赋予netdomain类型。而netdomain类型可以在
system/sepolicy/public/net.te 中看到，存在如下规则：

```
#......
allow netdomain self:tcp_socket create_stream_socket_perms;
#......
```

那么使用该宏则表示，将name-server赋予tcp_socket对应的create_stream_socket_perms权限。整体而言net_domain函数表示允许使用 net 域中的常用网络功能，如读取和写入 TCP 数据包、通过套接字进行通信，以及执行 DNS 请求等。也就是说，可以通过对应的宏函数调用完成对某些类型权限的统一allow。

##### name-server编译报错

out目录下error.log

```java
libsepol.report_failure: neverallow on line 959 of system/sepolicy/public/domain.te (or line 12961 of policy.conf) violated by allow name-server name-server_exec:file { entrypoint };
```

在system/sepolicy/public/domain.te中第959行

```java
full_treble_only(`
    # Do not allow coredomain to access entrypoint for files other
    # than system_file_type and postinstall_file
       
    neverallow coredomain {
        file_type  # 禁止coredomain对file_type类型的执行权限
        -system_file_type  # 排除system_file_type
        -postinstall_file
    }:file entrypoint;
    # Do not allow domains other than coredomain to access entrypoint
    # for anything but vendor_file_type and init_exec for vendor_init.
    neverallow { domain -coredomain } {
        file_type
        -vendor_file_type
        -init_exec
    }:file entrypoint;
')
```

在以上配置中，可以理解为neverallow（禁止）了coredomain类型与domain类型对于ﬁle_type类型的访问权限。为解决该问题，可以在name-server.te中，修改name-server_exec赋予system_ﬁle_type属性。

```
type name-server_exec, system_file_type, exec_type, file_type;
```

另外，在system/sepolicy/public/domain.te中第300行：

```java
#不允许coredomain类型对domain类型进行unix_stream_socket
full_treble_only(`
  neverallow_establish_socket_comms({
    coredomain
    -init
    -adbd
  }, {
    domain
    -coredomain
    -socket_between_core_and_vendor_violators
  });
')
```

此时system/sepolicy/private/untrusted_app.te 中配置为：

```java
typeattribute untrusted_app coredomain;

app_domain(untrusted_app)
untrusted_app_domain(untrusted_app)
net_domain(untrusted_app)
bluetooth_domain(untrusted_app)
```

即第三方app类型为coredomain类型，而我们的name-server又是domain，即不允许APP通过Socket连接系统服务name-server。解决办法就是将name-server赋于其他类型（如cordomain）作为属性！

```
typeattribute name-server coredomain;
```

同时在system/sepolicy/private/mls中存在如下限制：

```
mlsconstrain unix_stream_socket { connectto }
             (l1 eq l2 or t1 == mlstrustedsubject or t2 == mlstrustedsubject);
```

必须满足3 个条件之一，才能授予 unix_stream_socket 的 connectto 权限：
1. l1 == l2，l1代表untrusted_app，l2代表name-server，不满足；
2. t1 == mlstrustedsubject，t1为 untrusted_app，不属于 mlstrustedsubject；
3. t2 == mlstrustedsubject，t2为 name-server，不属于 mlstrustedsubject。

以上3 个条件都不满足。因此app还是无法访问name-server的socket服务。

综合上述分析，可以修改name-server.te，为name-server配置mlstrustedsubject属性即可。

```java
typeattribute name-server coredomain;
typeattribute name-server mlstrustedsubject;
```

untrusted_app与untrusted_app_all的关系：

untrusted_app表示第三方app，在untrusted_app.te中存在untrusted_app_domain(untrusted_app)配置。untrusted_app_domain 定义于：system/sepolicy/public/te_macros

```java
#####################################
# untrusted_app_domain(domain)
# Allow a base set of permissions required for all untrusted apps.
define(`untrusted_app_domain', `
typeattribute $1 untrusted_app_all;
')
```

表示为untrusted_app赋于untrusted_app_all属性，即untrusted_app就是untrusted_app_all。
