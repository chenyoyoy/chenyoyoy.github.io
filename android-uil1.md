## 单张图片加载模型
---
![单张图片加载模型](http://upload-images.jianshu.io/upload_images/1239900-89b438f9963d28aa.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

>图片加载步骤：
- 根据uri 定位到文件，本地磁盘文件，Assert 文件，res 资源文件，network 网络文件等；
- 从图片文件中获取流数据到内存，这个过程可以叫做load；
- 对流数据进行解码，得到原始图像数据Bitmap，这个过程叫做解码，decode ；
- 将bitmap设置给view 显示出来，这个过程叫display ；

>注意事项：
- 图片解码过程中，单张图片解码出来的图片数据会达到很大，甚至导致应用出现oom的情况，需要谨慎对待；
- 图片显示过程中，View 的大小相对来说都是比较固定的，一张很大的图片显示在一个比较小的view 上面，显示效果不但得不到提升，而且还会消耗系统资源；

## 多张图片的缓存模型
---
![多张图片加载的缓存模型](http://upload-images.jianshu.io/upload_images/1239900-5d537c2dba9dacea.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

>场景：
   - 在界面的切换过程中，很多图片资源是共用的，我们的每一次图片显示都经过load，decode，display？
   - 一张网络图片，上次从服务器下载下来了，下次用到的时候，还需要从服务器download，然后load，decode，display？

>解决方案：
 - 对于已经显示过得图片，将它存在内存中，下次再用的时候直接从内存中拿取；不用走整个的图片加载流程;
 - 对于已经下载过得网络图片，保存在本地磁盘，再次加载的时候直接从磁盘读取，同时缓存到内存中，就不用再次从服务器下载了；

>注意事项：
- 图片缓存在内存中，内存的大小始终是有限的，需要控制总的内存占用；
- 图片缓存到磁盘，磁盘的空间是有限的，需要有效的控制磁盘占用；

## UIL 框架源码结构
———

![UIL代码结构](http://upload-images.jianshu.io/upload_images/1239900-8413535d1179578c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

>流程控制：
   ImageLoader 入口类；
   ImageLoaderEngine 提供线程池；
   XXXXXTask ，Runnable 接口实现类，具体的流程控制类，会被丢到线程池中执行 ；

> 具体的实现：
   downloader 目录： 文件加载，从网络，本地，res ，assert 加载图片数据到内存中；
   decode 目录：对加载进来的流数据进行解码，得到Bitmap 数据 ；
   display 目录：对需要显示的bitmap数据进行处理，比如显示倒影，圆角等；
   imageaware 目录： bitmap显示封装类，提供统一的图片数据显示方式；
   listener 目录：图片加载过程的回调接口，通知流程进度和事件；
   memory 目录：内存缓存模型，提供各种策略的缓存方式；
   disc 目录：磁盘缓存模型，提供各种磁盘缓存方式；

## 类结构与加载模型对应关系
---

![Paste_Image.png](http://upload-images.jianshu.io/upload_images/1239900-902ed3a7bf436f51.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

类与流程
---
UIL 流程控制：ImageLoader 中生成Task ，提交到ImageLoaderEngine 中的线程池中执行；
具体的执行过程：downloader -> disc ->decode ->memory->display->imageaware ；

![UIL类和流程.png](http://upload-images.jianshu.io/upload_images/1239900-6de350554b8e26a0.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
