---
title: Android.mk打包apk
date: 2020-03-18 22:55:00
categories: 
- Android
- 打包编译
tag: 
- Android
- 打包编译
---

一、新建Android.mk文件 放在app/src/main 目录下

二、代码中引用的第三方库通过aar包存放在最外层libs目录下，所有app共享这些库文件。

如何有自己用的第三方开源库，[查找aar的方法](https://www.jianshu.com/p/59efa895589e) 

电脑上rxandroid aar的目录：
C:\Users\Lenovo\.gradle\caches\modules-2\files-2.1\io.reactivex\rxandroid（各个电脑目录不同）

Android.mk编写规则  
（系统设置例子）

```
#代表mk当前文档路径
LOCAL_PATH := $(call my-dir) 

include $(CLEAR_VARS)

LOCAL_PACKAGE_NAME := HQ_DevSet

#指该模块在所有版本下都编译
LOCAL_MODULE_TAGS := optional

#混淆配置
LOCAL_PROGUARD_ENABLED := full obfuscation
LOCAL_PROGUARD_FLAG_FILES := ../../proguard-rules.pro

#设置不打odex包
LOCAL_DEX_PREOPT := false
DONT_DEXPREOPT_PREBUILTS := true  

#apk路径
LOCAL_MODULE_PATH := $(TARGET_OUT)/app

#签名配置
LOCAL_CERTIFICATE := platform

LOCAL_MULTILIB := 32

#源码文件
#引用当前app的资源
LOCAL_RESOURCE_DIR += $(LOCAL_PATH)/res

#声明当前app的代码目录
src_dirs := java/

#引用当前app的代码
LOCAL_SRC_FILES := $(call all-java-files-under, $(src_dirs))

#添加aidl源码文件
LOCAL_SRC_FILES += \
src/xx/xx/xx/XxxOne.aidl \
src/xx/xx/xx/XxxTwo.aidl

# jar包
#声明多个 jar 包的位置   ----其中\代表换行，后面不能跟任何字符，后面的一样
LOCAL_PREBUILT_STATIC_JAVA_LIBRARIES := \
 hqlib:../../../libs/hqlib.jar \
 hqapi:../../../libs/hqapi.jar \
 hqapi_sq:../../../libs/hqapi_sq.jar\
 rxjava:../../../libs/rxjava-1.3.7.jar
																			
#引用我们声明的多个 jar 包的变量
LOCAL_STATIC_JAVA_LIBRARIES +=  hqlib \
                                hqapi \
                                hqapi_sq \
                                rxjava \
                                android-support-v4
#aar包
#声明aar包
LOCAL_PREBUILT_STATIC_JAVA_LIBRARIES += PublicTitle2:../../../libs/PublicTitle2.aar\
rxandroid:../../../libs/rxandroid-1.2.1.aar

#引用aar包
LOCAL_STATIC_JAVA_AAR_LIBRARIES += PublicTitle2 \
								 Rxandroid

#引入使用的资源 添加包名
LOCAL_AAPT_FLAGS += --auto-add-overlay \
					--extra-packages com.hopechart.sq.publictitle \
					--extra-packages rx.android 

#设置version版本
#设置版本号和名字，如果不写会默认使用系统api的版本号25 --7.1.1
version_code = 24
version_name := 4.0.0.1
						
LOCAL_AAPT_FLAGS += --version-code $(version_code)
LOCAL_AAPT_FLAGS += --version-name $(version_name)    

#打出来的minSDK和targetSDK version都是19 
LOCAL_SDK_VERSION := 19  
#如果有特殊情况可在清单文件中添加
# 【
#  AndroidManifest.xml清单文件添加sdk版本，优先使用清单文件中
# 的版本号
#  <uses-sdk android:minSdkVersion="19"
#         android:targetSdkVersion="25"/>
# 】

#打apk包
include $(BUILD_PACKAGE)

#调用子目录的mk文件
include $(call all-makefiles-under,$(LOCAL_PATH))
```


#Android.mk END

#如果使用的系统的包，需要引入他们使用的资源文件，否则会提示编译资源找不到的错误

```
#RecyclerView例子
LOCAL_RESOURCE_DIR += frameworks/support/v7/recyclerview/res
LOCAL_STATIC_JAVA_LIBRARIES += android-support-v7-recyclerview
LOCAL_AAPT_FLAGS += --auto-add-overlay \
--extra-packages android.support.v7.recyclerview
```

代码编译
1、在源码任意目录下创建一个目录，把改好的代码上传到该目录；  
2、切换到源码顶层目录，执行

```
source build/envsetup.sh;
```
    
3、编译单个apk , 执行

```
mmm  [apk mk文件的目录]
```

需要在全编的情况下才能单独编译

```
make -j32
```
编译所有的模块编译所有的模块需要在每个模块的父级目录添加Android.mk文件

内容【

```
LOCAL_PATH := $(call my-dir)
include $(CLEAR_VARS)
include $(call all-makefiles-under,$(LOCAL_PATH))
```

】 
这样就可以执行子目录中的Android.mk文件。
如下目录执行的命令就是: mmm 代码目录/SQ10Inch,然后就可以在输出目录看到所有打包出来的apk文件。



参考链接：  
[Android源码编译第三方app如何写Android.mk](https://blog.csdn.net/jsn_ze/article/details/72790401)    
[Android.mk引用jar包、so库、aar包系统签名](https://www.jianshu.com/p/e19e0d3bf13a)