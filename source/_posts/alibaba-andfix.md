---
title: AndFix
date: 2016-04-20 15:22:48 +08:00
tags: Hot Fix
category: Android Dev
---

[AndFix](https://github.com/alibaba/AndFix)

基本配置及使用方法在github上写得很清楚。
<!-- more -->
这边写一下引入具体实际项目的方式和细节，以及一些要注意的问题：

1. Gradle配置

```
compile 'com.alipay.euler:andfix:0.4.0@aar'

```

2.  proguard-rules.pro配置

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

在需要发补丁包的时候，把上次打包的mapping.txt文件拷贝出来放到build目录下（或者其他），同时打开 “-applymapping build\mapping.txt”这项配置，再打包。

3. 用未加固的包生成patch文件

   实际测试过加固的包生成的patch文件无法使用。

4. 把patch文件修改后缀*.zip，用jadx工具查看，因为没有加固生成的patch包，因此可以看到混淆后发生改动的整个类的代码。在修改的函数前会有一句这样的：

```
@MethodReplace(clazz = "classPath/className", method = "methodName")

```

因此，如果一个类中有两个同名函数，会出问题，比如基类有个private methodA(), 子类也有一个public methodA()。就会有问题，表现就是打了补丁之后会crash。

5. 打补丁接口设计

加载patch的关键代码
```
      PatchManager patchManager = new PatchManager(this);
      patchManager.init(appVersion);
      patchManager.loadPatch();
        
       
        // add patch at runtime
        try {
            String patchFileString = Environment.getExternalStorageDirectory()
                    .getAbsolutePath() + "/out.apatch";
            patchManager.addPatch(patchFileString);
        } catch (IOException e) {
            logger.error(e.toString(), e);
        }

```
先看PatchManager的初始化:

```
	public PatchManager(Context context) {
		mContext = context;
		mAndFixManager = new AndFixManager(mContext);
		mPatchDir = new File(mContext.getFilesDir(), DIR);
		mPatchs = new ConcurrentSkipListSet<Patch>();
		mLoaders = new ConcurrentHashMap<String, ClassLoader>();
	}


```

可以看出.apatch文件是加载在data/packagename/apatch目录下的。


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

可以看出init的时候传入了一个版本号，然后根据版本号判断对已有patch的处理，初始化已经加载过的patches到PatchManager。同一个版本可以有多个patch。

因此appversion应该传入的是代码版本号。即patches是是基于该版本的补丁。

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

在从服务器获取到patch后存在本地，拷贝到patch文件夹，之后便不需要重复下载了。

接口上还是传appversion获取该版本对应的最新的补丁，服务器上补丁文件名为appversion.apatch，但是PatchManassger里addPatch的逻辑是通过文件名判断该patch是否加载过。所以最新的补丁文件名最好是appversion+patchMD5的组合。才能保证说补丁能够及时更新。

为了不重复下载，在接口上可以多加一个参数。最好能判断当前运行的app版本是打了补丁的。



