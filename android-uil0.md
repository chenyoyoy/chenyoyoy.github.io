## 1.  UIL 的初始化
---
   > 方法名：ImageLoader.init(ImageLoaderConfiguration configuration)
      使用方法：
<code>public synchronized void init(ImageLoaderConfiguration configuration) {
   if (configuration == null) {
      throw new IllegalArgumentException(ERROR_INIT_CONFIG_WITH_NULL); 
  }   if (this.configuration == null) {
      L.d(LOG_INIT_CONFIG);
      engine = new ImageLoaderEngine(configuration);
      this.configuration = configuration;
   } else {
      L.w(WARNING_RE_INIT_CONFIG);
   }}</code>

>1.1 ImageLoaderConfiguration 
- 说明：全局配置，因为UIL是面向接口的，所以很多东西都是可以配置的 ;
- ImageLoaderConfiguration 中比较重要的配置有：
- maxImageWidthForMemoryCache   内存缓存的最大图片宽度
- maxImageHeightForMemoryCache   内存缓存的最大图片高度；
- maxImageWidthForDiskCache 磁盘缓存的最大图片宽度；
- maxImageHeightForDiskCache 磁盘缓存的最大图片高度；
- threadPoolSize 线程池的大小；
- threadPriority 线程池中线程优先级；
- memoryCache 内存缓存策略；
- DiskCache 磁盘缓存策略；
- ImageDownloader 下载器，从文件中获取数据流；
- ImageDecoder 解码方式；
- DisplayImageOptions 显示选项；

>1.2 DefaultConfigurationFactory
- 说明：默认配置生成器，在没有进行任何配置的时候，产生默认配置。
- 默认线程池中的队列弹出方式，先进先出
- 默认线程池中线程优先级，3 ；
- 默认磁盘缓存方式，LruDiskCache ；
- 默认内存缓存方式，LruMemoryCache ，最大值为app 最大内存占用/8；
-默认内存最大图片宽度，屏幕显示宽度；
-默认内存最大图片高度，屏幕显示高度 ；
-默认下载器，BaseImageDownloader ；
-默认解码器，BaseImageDecoder ；
-默认显示处理器，SimpleBitmapDisplayer ；
-默认显示器，ImageViewAware ；

>1.3 ImageLoaderEngine
- 说明，图片加载引擎，含有线程池，每次的图片加载显示任务都会被提交到他内部，到线程池中异步执行；
