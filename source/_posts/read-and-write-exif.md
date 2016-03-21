title:  Android读写图片Exif信息

date: 2016-3-1 20:10:30

tags: Android

---

需求场景：需要将原始图片缩放之后依旧保留原始图片Exif信息

解决方案：在原始数据中读取Exif信息，图片缩放之后重新写入

最简单易用的EXIF信息处理包是[metadata-extractor](https://github.com/drewnoakes/metadata-extractor/)，只可惜只能读，不能写。

然后发现老旧的[MediaUtil](http://mediachest.sourceforge.net/mediautil/)，这个可以读可以写，看看[AndroidMediaUtil](https://github.com/bkhall/AndroidMediaUtil)，不是那么好用。

最后搜到了最佳方案：
[Android-Exif-Extended](https://github.com/sephiroth74/Android-Exif-Extended/)

不得不说搜索也是一门功夫，特地去查了搜索路径，一般来说用中文搜索是无果的。




