---
title: 在onResume刷新View？
date: 2016-03-28 19:24:54 +08:00
tags: Android Activity Lifecircle
category: Android Dev
---

前几天Fix了一个问题，是在执行onActivityResult的时候某些元素缺失导致的NPE。
<!-- more -->
重现该问题很容易，就是进入开发者模式打开“不保留活动”，然后进入应用。

后来发现是在onResume里面做了类似刷新View的操作，而刷新View的函数里还有一些初始化的操作。

而在Activity被销毁后重建的过程中，生命周期函数执行的顺序是：

onCreate -〉onRestoreSavedInstanceState -〉onActivityResult -〉onResume 

所以在onResume里做刷新初始化等等的操作是偷懒欠考虑的。




