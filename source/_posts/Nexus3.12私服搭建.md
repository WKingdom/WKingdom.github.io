---
title: Nexus3.12私服搭建
date: 2019-03-18 22:55:00
categories: 
- Android
- 打包编译
tag: 
- Android
- 打包编译
---
##### 下载安装  
（1）、[nexus最新版本下载](https://www.sonatype.com/download-oss-sonatype)（Nexus Repository Manager OSS 3.x - Windows）  
（2）、解压后，安装服务，启动服务  
```
    1）左shift打开CMD窗口nexus-3.12.1-01-win64\nexus-3.12.1-01\bin目录    
    2）nexus.exe /install  安装服务  
    3）nexus.exe /start    启动服务  
    4）nexus.exe /stop     停止服务
```
（3）、使用nexus软件
在浏览器中输入http://localhost:8081/，点击“login in"，输入admin/admin123即可。  
（4）、管理目前的仓库  
将central的远程地址修改如下（即改为aliyun的镜像，主要是速度快一些）：http://maven.aliyun.com/nexus/content/groups/public/  
（[参考链接](https://blog.csdn.net/ytfrdfiw/article/details/78476087)）

#### 多平台Nexus私服搭建
[Android Studio依赖管理与Nexus私服搭建](http://www.10tiao.com/html/227/201703/2650238831/1.html)  

创建自己的仓库

创建用户

首先使用管理员密码登陆到 Nexus私服 并添加用户：

。。。。

上传自己的Module到仓库

1、 在项目级别的 build.gradle 中的 allprojects 下 repositories节点 添加 mavenLocal()：  

```
allprojects {
    repositories {
        google()
        jcenter()
        mavenLocal()
        maven{
            url "http://localhost:8081/repository/PublicTitle/"
        }
    }
}
```


2、 在 Lib Module 级别的 build.gradle 中添加 maven 插件 apply plugin: 'maven'：

```
apply plugin: 'com.android.library'
apply plugin: 'maven'
```

3、 在 Lib Module 级别的 build.gradle 中 android节点 添加上传行为：

```
uploadArchives {
    repositories.mavenDeployer {
        repository(url: "http://localhost:8081/repository/PublicTitle/"){
            authentication(userName: "wz",password: "123456")
        }
        pom.groupId = 'com.hopechart'
        pom.artifactId = 'PublicTitle'
        pom.version = '1.0.1'
    }
}
```

4、使用Gradle插件上传aar到Maven私服，点击 uploadArchives 自动上传：


5、引用私服中的Module  

在需要依赖 Module 的 build.gradle 中添加如下节点,其中URL就是上文中创建仓库的ur
```
//项目级别的 build.gradle
 allprojects {
    repositories {
        ......
        maven{
            url "http://localhost:8081/repository/PublicTitle/"
        }
    }
}
```

```
//model级别的 build.gradle

 compile 'com.hopechart:PublicTitle:1.0.0'
```

