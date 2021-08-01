---
title: UE5打包安卓环境搭建
date: 2021-07-30 07:30:46
tags:
- UE5
- 打包
- 安卓
categories: UE5
cover: https://img-blog.csdnimg.cn/20210730074129258.jpg
---

# 下载UE5
首先自己下载UE5，然后记得把Android选项勾上下载，默认好像是勾上了的，如果没勾上自己下载一下。   

![](https://img-blog.csdnimg.cn/20210730212849100.png)
![](https://img-blog.csdnimg.cn/20210730212858482.png)

# 下载jdk-8u77-windows-x64.exe   
[官网下载地址](https://www.oracle.com/java/technologies/javase/javase8-archive-downloads.html)

![](https://img-blog.csdnimg.cn/20210730213151369.png)
当然官网下载速度可能有点慢，如果实在是下载不了的话，就从后面我提供的链接里面下载。      
下载完以后直接安装，安装路径就保持默认，安装比较简单，直接一直下一步就可以了，这里就不废话了。安装完以后找到安装路径，默认为C:\Program Files\Java，如果你更改了路径，那你就找到你自己安装的路径就可以了。然后复制路径C:\Program Files\Java\jdk1.8.0_77，在电脑左下角搜索environment，选择编辑系统环境变量。如下图所示：   

![](https://img-blog.csdnimg.cn/20210730213735212.png)
点击环境变量 

![](https://img-blog.csdnimg.cn/20210730213842439.png)
在用户变量里面点击新建        

![](https://img-blog.csdnimg.cn/20210730213902952.png)
然后如下图设置，设置完以后点击确定。      

![](https://img-blog.csdnimg.cn/20210730213951521.png)
点击下方系统环境变量的新建         

![](https://img-blog.csdnimg.cn/2021073021405544.png)
然后如下图设置，设置完以后点击确定。     

![](https://img-blog.csdnimg.cn/20210730213951521.png)
这样环境变量就设置好了。

# 下载AndroidStudio 
[官网下载地址](https://developer.android.com/studio/archive)

![](https://img-blog.csdnimg.cn/2021073021441631.png)
当然官网下载速度可能有点慢，如果实在是下载不了的话，就从后面我提供的链接里面下载。     
下载完以后直接安装，取消Android Virtual Device勾选，没必要下载，当然你也可以选择下载。  

![](https://img-blog.csdnimg.cn/20210730214659346.jpg)
AndroidStudio安装位置自己选择，比如我安装在D:\AndroidStudio，安装完以后直接启动。        
选择SDK Manager。         

![](https://img-blog.csdnimg.cn/20210730220913881.png)
选择Android API 30（如果31勾选了的话取消了它）   

![](https://img-blog.csdnimg.cn/20210730220947851.png)
然后点击SDK Tools，按如下图进行选择，如果31.0.0勾选了的话取消它，然后点击Apply。 

![](https://img-blog.csdnimg.cn/20210730221522261.png)
安装位置的话我放在D:\sdk，你自己看着放，等待安装完。             

# UE5创建测试项目
打开UE5创建一个蓝图第三人称模板项目，如下图设置。   

![](https://img-blog.csdnimg.cn/20210730222309410.png) 
打开ProjectSettings       

![](https://img-blog.csdnimg.cn/20210730222355712.png)
然后如下图设置     

![](https://img-blog.csdnimg.cn/20210730222537679.png)
现在进入jdk安装目录复制C:\Program Files\Java\jdk1.8.0_77\bin 路径，然后打开cmd命令行窗口，记得以管理员身份运行。 

![](https://img-blog.csdnimg.cn/2021073022285233.png)
通过cd 进入我们刚刚复制的路径     

![](https://img-blog.csdnimg.cn/2021073022302776.png)
然后复制如下命令进去回车           
```(C++)
    keytool -genkey -v -keystore ExampleKey.keystore -alias MyKey -keyalg RSA -keysize 2048 -validity 10000  
```   
口令这个得记住后面有用，比如我这里设置为**666666**.其他的东西随便写，还有记得回复写**是**不能是**yes**，然后回车就可以了。  

![](https://img-blog.csdnimg.cn/20210730223211947.png)
这个时候在咱们的C:\Program Files\Java\jdk1.8.0_77\bin路径下生成了一个文件。  

![](https://img-blog.csdnimg.cn/20210730223508117.png)
把这玩意复制到刚刚创建的UE5项目的Build/Android路径下。       

![](https://img-blog.csdnimg.cn/2021073022432889.png) 
进入UE5编辑器的ProjectSettings里面，选择Android。    

![](https://img-blog.csdnimg.cn/20210730224537107.png)
![](https://img-blog.csdnimg.cn/20210730224828429.png)
到这里就全部设置完了，直接去导出就可以了。      

![](https://img-blog.csdnimg.cn/20210730224907127.png)
等待打包完，OK环境搭建结束了。

# 可能出现的问题
1. Android Studio报错unable to access android sdk add-on
![](https://img-blog.csdnimg.cn/2021073023401258.png)
> 解决办法：        
> * 在Android Studio的安装目录下，找到\bin\idea.properties   
> * 在尾行添加disable.android.first.run=true，表示初次启动不检测SDK
2. UE5打包失败    

![](https://img-blog.csdnimg.cn/2021073022520373.png)   
> 解决办法：   
> 安装dotnet-sdk-3.1.409-win-x64就可以了，[下载地址](https://dotnet.microsoft.com/download/dotnet/thank-you/sdk-3.1.409-windows-x64-installer)。   
> 下载安装完以后重新打包就可以了。  

# 下载
1. [jdk-8u77-windows-x64](https://pan.baidu.com/s/1XvOiuAs0W0OdkX06s3AlNg) 提取码: grgu   
2. [android-studio-ide-191.6010548-windows](https://pan.baidu.com/s/1cyyZUGysB5MwfR1LLnfXkw) 提取码: q7aq    
3. [dotnet-sdk-3.1.409-win-x64](https://pan.baidu.com/s/1Vto2VcM9AI7TSyaZcSPiyA)  提取码: bb5z   