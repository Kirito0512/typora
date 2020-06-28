这个问题其实折腾我很久，后来找到简书上这个博客讲的还挺细致

[Android 使用浏览器打开APP](https://www.jianshu.com/p/5edbd0d1af54)

其中有一句，"在html中加入点击事件"，我对html不了解，搞了半天才明白怎么弄

其实很简单，创建一个.html文件，把下面的代码复制进去就行

```html
<!DOCTYPE HTML>
<html>
<body>

<div id="logo"><a href="myapp://jp.app/openwith?name=zhangsan&age=26">Visit W3School.com.cn!</a></div>

</body>
</html>

```

打开就是一个html页面，再点击里面的按钮，就能唤起app中的页面了

感兴趣的同学可以详细看看   [w3cshool HTML 5 <a> 标签的 href 属性](https://www.w3school.com.cn/html5/att_a_href.asp)

