---
title: Android Studio安装panltUML
date: 2017-04-03 22:25:41
categories: 
- Java
- UML
tag: 
- Java
- UML
---
### 安装panltUML插件  
1.File->Settings->Plugins->Browse repositories  
2.在搜索框输入plantUML  
3.导入插件Install plugin  
如果以上步骤正确的完成，重启AndroidStudio 右键->new 的时候你会发现多了这么一堆东西，如果出现了这些说明plantUML已经正确的安装了。
### 安装Graphviz
设置plantUML  
1.点击右上角的设置按钮或进入File->Settings->Other Settings ->PlantUML  
2.将文件路径填写为刚刚Graphviz的目录下bin目录中dot.exe文件。
(我的为：D:/Program/Graphviz/bin/dot.exe)    
3.点击OK 刷新一下界面就能看到这个了