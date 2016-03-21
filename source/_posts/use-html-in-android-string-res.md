---

title: Android strings.xml 添加html代码
date: 2016-03-09 20:20:44 +08:00
tags: Android

---
在string.xml里面写html需要转义，烦得很，而且不利于维护。

在遇到TextView需要些简单样式的时候可以这样做：

```
TextView.setText(Html.fromHtml(getString(R.string.text_name)));

```

```xml
    <string name="text_name">Text1<Data><![CDATA[<font color="#ff643c">Text2</font>]]></Data>Text3</string>
```

*CDATA* 
>CDATA 指的是不由 XML 解析器进行解析的文本数据。
>在标记CDATA下，所有的标记、实体引用都被忽略，而被XML处理程序一视同仁地当做字符数据看待，>CDATA的形式如下：
><![CDATA[文本内容]]>
>CDATA的文本内容中不能出现字符串“]]>”，另外，CDATA不能嵌套。



