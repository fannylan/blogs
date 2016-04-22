title:  Android读写图片Exif信息

date: 2016-3-1 20:10:30

tags: Android
category: Android Dev
---

需求场景：需要将原始图片缩放之后依旧保留原始图片Exif信息

解决方案：在原始数据中读取Exif信息，图片缩放之后重新写入
<!-- more -->
最简单易用的EXIF信息处理包是[metadata-extractor](https://github.com/drewnoakes/metadata-extractor/)，只可惜只能读，不能写。

然后发现老旧的[MediaUtil](http://mediachest.sourceforge.net/mediautil/)，这个可以读可以写，看看[AndroidMediaUtil](https://github.com/bkhall/AndroidMediaUtil)，不是那么好用。

最后搜到了最佳方案：
[Android-Exif-Extended](https://github.com/sephiroth74/Android-Exif-Extended/)


关键代码
```
            //在存储中读取图片
            byte[] data = getImageBytes();
            ExifInterface exifInterface = new ExifInterface();

            //即将要写入的图片
            File imageFile = new File(saveDir + "/" + System.currentTimeMillis() + ".jpg");
            try {
                exifInterface.readExif(data, ExifInterface.Options.OPTION_ALL);

                FileOutputStream fileOutputStream = new FileOutputStream(imageFile);
                if (!needCompress) {
                    fileOutputStream.write(data);
                    fileOutputStream.flush();
                    return imageFile.getAbsolutePath();
                }

                //压缩图片
                compressImage(data, fileOutputStream);

                List<ExifTag> exifTags = exifInterface.getAllTags();
                ExifInterface newExif = new ExifInterface();
                newExif.setTags(exifTags);
                newExif.setCompressedThumbnail(exifInterface.getThumbnail());
                newExif.writeExif(imageFile.getAbsolutePath());
                return imageFile.getAbsolutePath();
            } catch (FileNotFoundException e) {
                logger.error("file not found!", e);
            } catch (IOException e) {
                logger.error("io error when write data!", e);
            } catch (Exception e) {
                logger.error("error when process image", e);
            }
            return null;
```


2016/3/24 更新： 最近发现了这个异常，在少量机型上，GN151/ GT I9500/SM G900F/ GT N7100/ HTC M8ST/ HTC X720D...都是2014年左右的机器。

```
Fatal Exception: java.lang.RuntimeException: An error occured while executing doInBackground()
       at android.os.AsyncTask$3.done(AsyncTask.java:299)
       at java.util.concurrent.FutureTask.finishCompletion(FutureTask.java:352)
       at java.util.concurrent.FutureTask.setException(FutureTask.java:219)
       at java.util.concurrent.FutureTask.run(FutureTask.java:239)
       at android.os.AsyncTask$SerialExecutor$1.run(AsyncTask.java:230)
       at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1080)
       at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:573)
       at java.lang.Thread.run(Thread.java:856)
Caused by java.lang.NullPointerException
       at it.sephiroth.android.library.exif2.ExifOutputStream.stripNullValueTags(ExifOutputStream.java:117)
       at it.sephiroth.android.library.exif2.ExifOutputStream.writeExifData(ExifOutputStream.java:78)
       at it.sephiroth.android.library.exif2.ExifInterface.writeExif_internal(ExifInterface.java:1157)
       at it.sephiroth.android.library.exif2.ExifInterface.writeExif(ExifInterface.java:1089)
       at it.sephiroth.android.library.exif2.ExifInterface.writeExif(ExifInterface.java:1062)
       at com.wumii.android.loan.ui.activity.CameraActivity$TakePictureTask.doInBackground(CameraActivity.java:182)
       at com.wumii.android.loan.ui.activity.CameraActivity$TakePictureTask.doInBackground(CameraActivity.java:144)
       at android.os.AsyncTask$2.call(AsyncTask.java:287)
       at java.util.concurrent.FutureTask.run(FutureTask.java:234)
       at android.os.AsyncTask$SerialExecutor$1.run(AsyncTask.java:230)
       at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1080)
       at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:573)
       at java.lang.Thread.run(Thread.java:856)
```

查阅ExifOutputStream的源码，发现应该是ExifData为null。

```
    private ArrayList<ExifTag> stripNullValueTags(ExifData data) {
        ArrayList nullTags = new ArrayList();
        Iterator i$ = data.getAllTags().iterator();

        while(i$.hasNext()) {
            ExifTag t = (ExifTag)i$.next();
            if(t.getValue() == null && !ExifInterface.isOffsetTag(t.getTagId())) {
                data.removeTag(t.getTagId(), t.getIfd());
                nullTags.add(t);
            }
        }

        return nullTags;
    }
```

接着将代码中的这一行注释掉，能够重现这个crash。

```
exifInterface.readExif(data, ExifInterface.Options.OPTION_ALL);
```

所以因该是在这些机器上拍照读不到Exif数据, 所以没读到的时候不需要存了。
```
                List<ExifTag> exifTags = exifInterface.getAllTags();
                if (exifTags != null) {
                	ExifInterface newExif = new ExifInterface();
                	newExif.setTags(exifTags);
                	newExif.setCompressedThumbnail(exifInterface.getThumbnail());
                	newExif.writeExif(imageFile.getAbsolutePath());
                }
```




