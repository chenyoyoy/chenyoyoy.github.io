> 效果图

![Paste_Image.png](http://upload-images.jianshu.io/upload_images/1239900-c37a6fb5d5eb552d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

> 相关接口

<code>
public final void setText(CharSequence text)</code>
TextView 设置显示内容的接口；

<code>public static Spanned fromHtml(String source, ImageGetter imageGetter, TagHandler tagHandler)</code>
Html 类中用于解析html资源的接口，source 表示html内容，ImageGetter 是图片下载器，TagHandler 是标签解析器，返回值Spanned 是一个接口，继承自CharSequence；

<code>public static Spanned fromHtml(String source)</code>
实际上是调用的上面的接口，ImageGetter 和 TagHandler 都传值为null 。这种情况下就只能支持默认的html标签解析以及无法进行图片加载，这种情况下，demo的显示会变成
![Paste_Image.png](http://upload-images.jianshu.io/upload_images/1239900-3e0018eecdbe9009.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

Android TextView 本身支持的 html标签类型：
![Paste_Image.png](http://upload-images.jianshu.io/upload_images/1239900-aa4ddeb3218dac72.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

> 图片加载器的设置

 ImageGetter 实际上是一个接口，提供了一个方法，
<code>public Drawable getDrawable(String source);</code>
它的调用发生在解析到img标签的时候，会去同步的加载图片资源

![Paste_Image.png](http://upload-images.jianshu.io/upload_images/1239900-2245e5e9db2f85c2.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

明显的是，网络图片的下载以及加载时不能够通过同步接口完成的，必须通过异步线程实现下载，成功后回调显示

![图片下载器的实现](http://upload-images.jianshu.io/upload_images/1239900-b26bd029f00cba07.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

> 标签解析器的扩展

TagHandler 同样是一个接口，提供方法
<code>public void handleTag(boolean opening, String tag,                         Editable output, XMLReader xmlReader);</code>
需要额外解析一些标签的时候，实现对应的方法即可

![解析显示列表](http://upload-images.jianshu.io/upload_images/1239900-f0a5826321692d68.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

>显示内容点击事件的获取

TextView 提供方法 setMovementMethod（MovementMethod movement），设置MovementMethod 对象，重写其onTouchEvent（）方法，就可以获取到显示内容的点击事件加以处理 。以下是图片点击事件和超链接点击事件的获取示例：

![点击事件获取](http://upload-images.jianshu.io/upload_images/1239900-3dcb0cb7bc691db0.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![不同点击事件的监听](http://upload-images.jianshu.io/upload_images/1239900-862e59dec9573c7e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

通过以上几个步骤的处理，就可以得到开始的显示效果，同时支持图片超链接的点击事件了，完整demo地址：HtmlTest
https://github.com/binye33333/android.git

>发现的问题

- 内存问题
图片数据是一次性加载进去的，不是根据显示需要从缓存中获取。这里就会暴露一个问题，在大量图片加载的时候，页面会大量的占用内存资源，导致内存溢出
内存不可控的问题导致了这种方案完全无法被采纳，因为我们不知道一篇文章到底会有多大的图片数据，除非给与约束；

- 超链接的点击背景
![Paste_Image.png](http://upload-images.jianshu.io/upload_images/1239900-280b8fedad09007f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
超链接点击后，内容背景色没有恢复
