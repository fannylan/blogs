---
title: AndFix接入逻辑分析
date: 2016-04-20 15:22:48 +08:00
tags: Hot Fix
category: Android Dev
---

首先先了解一下[AndFix](https://github.com/alibaba/AndFix)

> AndFix是一种为修复bug提供在线修复而不需要重新发布应用的解决方案。通俗地说就是fix bug不需要用户重装应用。通过生成以.aptch后缀结尾的固定格式补丁文件，从服务器端下发到客户端，运行时加载补丁以达到在线修复bug的功能。AndFix是一个Android Library Project. 在使用Gradle构建项目之前，项目中引用Android Library Project是通过apklib这种格式使android library project能够像引入jar包一样方便。然而在2013年的Google IO，发布了一种新的格式aar。而AndroidMavenPlugin和Gradle对这两种格式的支持程度不同：AndroidMavenPlugin在3.7.0之后才开始支持aar，而Gradle目前还不支持apklib。作为Android Library Project的打包格式，这两种格式的主要区别在于apklib是将java文件以及其他资源文件直接打包引入工程，而aar文件中class文件都被编译成.class和打包成classes.jar。

AndFix的原理是方法体替换：Method Replcaing。具体方式下面会说。

AndFix在项目中的基本配置及使用方法在github上写得很清楚。

<!-- more -->

这边写一下引入具体实际项目的方式和细节，以及一些要注意的问题：

**Gradle配置**

```
compile 'com.alipay.euler:andfix:0.4.0@aar'

```

**proguard-rules.pro配置**

```
-dontwarn android.annotation
-dontwarn com.alipay.euler.**
-keep class com.alipay.euler.** {*;}
-keep class * extends java.lang.annotation.Annotation
-keepclasseswithmembernames class * {
    native <methods>;
}

#-applymapping build\mapping.txt

```

这边配置是保持AndFix包里面的所有类和所有方法而不混淆。如果不混淆，可以不用配置这些。关于ProGuard的讨论，可以看看[为什么这么多商业Android开发者不混淆代码？] (http://www.zhihu.com/question/37446729)。


加入了混淆的情况下，在需要发补丁包的时候，把上次打Release包的mapping.txt文件拷贝出来放到build目录下（或者其他），同时打开 “-applymapping build\mapping.txt”这项配置，再打包。由此可以看出在打Release包的补丁中增加新的类是会出错的。


**用未加固的包生成patch文件**

实际测试过加固的包生成的patch文件加载后改动不生效。

把经过上述提示生成的.apatch文件修改后缀成*.zip，用[jadx](https://github.com/skylot/jadx)工具查看，因此可以看到混淆后发生改动的整个类的代码。在修改的函数前会有一句这样的代码：

```
@MethodReplace(clazz = "classPath/className", method = "methodName")

```

因此，如果一个类中有两个同名函数，比如基类有个private methodA(), 子类也有一个public methodA()。经过测试后发现其表现就是打了补丁之后触发改函数时会crash。

**加载patch逻辑分析**

AndFix Java接口相对简单。先看一下加载patch的关键代码，这个初始化的位置一般是在Application onCreate的时候执行，可以在Release包中才有这个逻辑，以下代码的主要逻辑是初始化PatchManager，加载已有的patches。

```
      PatchManager patchManager = new PatchManager(this);
      patchManager.init(appVersion);
      patchManager.loadPatch();
        
```

PatchManager的初始化，新的.apatch文件的应用内缓存目录是data/packagename/DIR，并且patches的内存缓存是有序的:

```
	public PatchManager(Context context) {
		mContext = context;
		mAndFixManager = new AndFixManager(mContext);
		mPatchDir = new File(mContext.getFilesDir(), DIR);
		mPatchs = new ConcurrentSkipListSet<Patch>();
		mLoaders = new ConcurrentHashMap<String, ClassLoader>();
	}


```

初始化PatchManager的时候传入了一个版本号，该版本号是当前应用版本号，即patches是该版本的补丁。
然后根据版本号判断对已存在patch的处理，初始化已经加载过的patches到PatchManager。
同一个代码版本号可以有多个patch。看PatchManager的源码可以看到patches的数据结构是SortedSet，而load patch的顺序是按照Patch文件的生成时间来排序的。因此最终生效的补丁包应该是最新的那个包。

```
	public void init(String appVersion) {
		if (!mPatchDir.exists() && !mPatchDir.mkdirs()) {// make directory fail
			Log.e(TAG, "patch dir create error.");
			return;
		} else if (!mPatchDir.isDirectory()) {// not directory
			mPatchDir.delete();
			return;
		}
		SharedPreferences sp = mContext.getSharedPreferences(SP_NAME,
				Context.MODE_PRIVATE);
		String ver = sp.getString(SP_VERSION, null);
		if (ver == null || !ver.equalsIgnoreCase(appVersion)) {
			cleanPatch();
			sp.edit().putString(SP_VERSION, appVersion).commit();
		} else {
			initPatchs();
		}
	}

```

加载新patch：

```
        try {
            String patchFileString = Environment.getExternalStorageDirectory()
                    .getAbsolutePath() + "/out.apatch";
            patchManager.addPatch(patchFileString);
        } catch (IOException e) {
            logger.error(e.toString(), e);
        }

```
看addPatch的逻辑可以看出加载patch的时候，是先拷贝到data目录下的patch文件夹，之后便不需要重复拷贝了。判断是否拷贝过的依据就是patch的文件名。

接口上还是传代码版本号获取该版本对应的最新的补丁，服务器上补丁文件名为appversion.apatch，但是PatchManassger里addPatch的逻辑是通过文件名判断该patch是否加载过。

所以最新的补丁文件名最好是appversion+patchMD5的组合。才能保证说补丁能够及时更新。

```
	public void addPatch(String path) throws IOException {
		File src = new File(path);
		File dest = new File(mPatchDir, src.getName());
		if(!src.exists()){
			throw new FileNotFoundException(path);
		}
		if (dest.exists()) {
			Log.d(TAG, "patch [" + path + "] has be loaded.");
			return;
		}
		FileUtil.copyFile(src, dest);// copy to patch's directory
		Patch patch = addPatch(dest);
		if (patch != null) {
			loadPatch(patch);
		}
	}

```

**打补丁接口设计**

该patch接口的设计需要满足

1.发布patch之后需要客户端对应的版本能够即时更新，可以通过定时检查的方式（最简单的方式）。

2.获取patch的接口可以有两个参数，一个是当前的代码版本号，另一个是本地最新已加载的patch文件的MD5，如果客户端传的和服务器上的patch不一致才返回新的，patch文件。




