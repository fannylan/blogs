---
title: Android apk 防止反编译：加壳保护方案
date: 2016-03-18 10:49:48 +08:00
tags: "Android"
---

主要讨论Android APK的加壳技术，主要是讨论加壳方式，原理，以及应用在实际工程中遇到的问题和解决方案。撇开其他第三方的Android  APK加固工具和加固方式，这里只讨论手动的加壳保护。

加壳保护是在不影响app的运行效率和app开发效率的的前提下，防止他人直接对APK进行反编译，提高Android app的安全性，保护开发成果的一种方式。原理就是当壳程序运行的时候，真正的源程序被解密并且动态加载到系统中。

**从最简单的加壳Demo开始入手：**

[Android APK加壳技术方案[1]](http://blog.csdn.net/androidsecurity/article/details/8678399)

[Android APK加壳技术方案[2]](http://blog.csdn.net/androidsecurity/article/details/8809542)


**一些准备知识：**

[DEX文件结构](http://blog.csdn.net/jiazhijun/article/details/8664778)

在Android 4.0源码Dalvik/docs目录下提供了一份文档dex-format.html，里面详细介绍了dex文件格式以及使用到的数据结构。但是之后版本该文档被移除了。

**结合实践和理解，撇开加密算法和加密方式加壳保护具体步骤如下：**

1.源程序：需要被保护的工程源代码，是Android Project

   build源程序，得倒source.apk
   
2.壳程序：也是一个Android Project，主要逻辑是将自身dex文件中的源程序apk解密出来再加载进内存中

   build壳程序，得倒shell.apk，打开将class.dex解压出来备用
   
3.加壳程序：将源程序编译出来的apk加密写入壳程序的dex文件中

  通过加壳程序将source.apk写入壳程序的class.dex，再把新合成的的class.dex放回shell.apk, 替换旧的class.dex，重新签名 。

**遇到的问题：**

最后一步不能正确签名导致包无法解析，或者无法安装APK，以debug key签名示例 , 正确的方式：
JDK 1.7要加  -digestalg SHA1 -sigalg MD5withRSA
```
jarsigner -verbose -keystore debug.keystore -sigfile CERT -digestalg SHA1 -sigalg MD5withRSA  -signedjar final.apk shell.apk androiddebugkey

输入密钥库的密码短语:

 正在更新: META-INF/MANIFEST.MF
 
  正在添加: META-INF/CERT.SF
   
  正在添加: META-INF/CERT.RSA
   
  正在签名: AndroidManifest.xml
  
  正在签名: classes.dex
  
jar 已签名。

警告:
未提供 -tsa 或 -tsacert, 此 jar 没有时间戳。如果没有时间戳, 则在签名者证书的到期
日期 (2045-11-25) 或以后的任何撤销日期之后, 用户可能无法验证此 jar。
```
```
验证是否正确的签名：
jarsigner -verify final.apk  

```
没有时间戳那个加上 -tsa https://timestamp.geotrust.com/  但是会出现java.net.UnknownHostException: timestamp.digicert.com

2. 壳程序将加密方式暴露了
反编译最终的加壳程序，可以看到脱壳代码和方式

3. 壳程序和源程序的Manifest定义的组件和变量要保持一致，否则会发生找不到组件的异常退出。
比如引用的string和theme等等，定义在哪里会是一个问题，如果两边都定义，不利于维护，如果一边定义，另外一边无法打包。

**Android动态加载技术**

上面的那个demo仅仅是作为一个入门示例，引入实际的工程是有难度的。但是加壳的思路就是动态加载组件提高反编译的难度，接着从之前那个demo遇到的问题入手了解了一下动态加载的东西：

[Android动态加载技术 简单易懂的介绍方式](https://segmentfault.com/a/1190000004062866)

[Android动态加载入门 简单加载模式](https://segmentfault.com/a/1190000004062952)

[Android动态加载进阶 代理Activity模式](https://segmentfault.com/a/1190000004062972)

一些对我们目前的需求有帮助我摘出来了：

>Android项目中，动态加载技术按照加载的可执行文件的不同大致可以分为两种：
1.动态加载so库；
2.动态加载dex/jar/apk文件（现在动态加载普遍说的是这种）；

>Android中NDK中其实就使用了动态加载，动态加载.so库并通过JNI调用其封装好的方法。后者一般是由C/C++编译而成，运行在Native层，效率会比执行在虚拟机层的Java代码高很多，所以Android中经常通过动态加载.so库来完成一些对性能比较有需求的工作（比如T9搜索、或者Bitmap的解码、图片高斯模糊处理等）。此外，由于so库是由C/C++编译而来的，只能被反编译成汇编代码，相比中dex文件反编译得到的Smali代码更难被破解，因此so库也可以被用于安全领域。

>简单的动态加载模式
理解ClassLoader的工作机制后，我们知道了Android应用在运行时使用ClassLoader动态加载外部的dex文件非常简单，不用覆盖安装新的APK，就可以更改APP的代码逻辑。但是Android却很难使用插件APK里的res资源，这意味着无法使用新的XML布局等资源，同时由于无法更改本地的Manifest清单文件，所以无法启动新的Activity等组件。

简单的动态加载的缺陷以及**so库也可以被用于安全领域。**所以把需要保护的模块编译成so库，也可以达到我们的要求。

**插件化编程**

**[携程Android App插件化和动态加载实践](http://www.infoq.com/cn/articles/ctrip-android-dynamic-loading?email=947091870@qq.com&isappinstalled=0)**

>从以上几点根本性需求可以看出，插件化动态加载架构方案会为我们带来多么巨大的收益，除此之外还有诸多好处：
编译速度提升

>工程被拆分为十来个子工程之后，Android Studio编译流程繁冗的缺点被迅速放大，在Win7机械硬盘开发机上编译时间曾突破1小时，令人发指的龟速编译让开发人员叫苦不迭（当然现在换成Mac+SSD快太多）。

>启动速度提升

>Google提供的MultiDex方案，会在主线程中执行所有dex的解压、dexopt、加载操作，这是一个非常漫长的过程，用户会明显的看到长久的黑屏，更容易造成主线程的ANR，导致首次启动初始化失败。

>A/B Testing

>可以独立开发AB版本的模块，而不是将AB版本代码写在同一个模块中。

>可选模块按需下载

​>例如用于调试功能的模块可以在需要时进行下载后进行加载，减少App Size


后来了解到了[Small](https://github.com/wequick/Small)，一个插件框架，使用该框架的工程就像一个壳程序（宿主程序），里面的插件模块就像一个个相互独立的源程序模块，在代码组织和管理上有很大的便利性。

*Small Sample代码解读*

先将源码编译出一个包来，安装，应用先进入启动页，然后首页。首页分两个tab ,可以滑动，还有页面的简单跳转。

![pic](https://d1zjcuqflbd5k.cloudfront.net/files/acc_467455/10Z61?response-content-disposition=inline;%20filename=S60321-110746.jpg&Expires=1458543065&Signature=g3-5kRG6r9TrPKF8rcQ~Sd0uLMPnr5xGNDEbdy~bGjq7Q0z-mvcHjf9QNw-OvqQvSnMyoEMiGwGsuMqHvkEbhSx-FHIjPyfbRBqAt6tGPj9eUxji~A55iFhWwYVcFifVNYa2TdmDKw~HdTSLTYAcnAjYujO2LXoMKXQraC9eHuw_&Key-Pair-Id=APKAJTEIOJM3LSMN33SA)

![pic](https://d1zjcuqflbd5k.cloudfront.net/files/acc_467455/1gGAw?response-content-disposition=inline;%20filename=S60321-110811.jpg&Expires=1458542898&Signature=FifVugG~FB2DsnersjxZVjG-WmMuDcQGOFOrol3zJDakuMlWnN1bc~M5XSaUd8h99epJ-s8n8qsowy5SQ-Uc8a6jWFzdBcEwWutZ5nM8KiOeQWpReGV33C8hS7S~y-2AMhtD~1tfYBu0QH5eWMHgyHBH3WJlmMPmh5AacfqBmAg_&Key-Pair-Id=APKAJTEIOJM3LSMN33SA)

看源码目录结构，可以看到有如下几个模块：
-app
-app.detail
-app.home
-app.main
-app.mine
-web.about

app就是壳程序（宿主程序），app中的逻辑很少，只有一个代理Activity，主要是启动main模块然后finish。

关键代码如下，BaseUri按照自己需要的设置，具体方式见[Small/Android](https://github.com/wequick/Small/tree/master/Android)。

```

	Small.setBaseUri("http://m.wequick.net/demo/");

```

```

        Small.setUp(this, new net.wequick.small.Small.OnCompleteListener() {
            @Override
            public void onComplete() {
                mContentView.postDelayed(new Runnable() {
                    @Override
                    public void run() {
                        Small.openUri("main", LaunchActivity.this);
                        finish();
                    }
                }, 2000);
            }
        });

```


接着看app.main模块，Tab上的Fragment也是动态加载的，

```
Fragment fragment = Small.createObject("fragment-v4", sUris[position], MainActivity.this);

```
打开网页

```
 Small.openUri("https://github.com/wequick/Small/issues", MainActivity.this);

```
引用AndroidLib，其中style utils都是以插件的方式，是独立模块

```

    compile project(':lib.style')
    compile project(':lib.utils')

```

其余几个模块类似。


**其他**

反编译apk的一些工具：
[ApkTool](http://ibotpeaches.github.io/Apktool/)
[dex2jar](https://github.com/pxb1988/dex2jar)
[jdgui](http://jd.benow.ca/)
[JADX]((https://github.com/skylot/jadx))

[android-security-awesome](https://github.com/ashishb/android-security-awesome)









