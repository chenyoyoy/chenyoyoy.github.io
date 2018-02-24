> 类似于ViewPager的滑动效果

![](http://upload-images.jianshu.io/upload_images/1239900-b4711a7c11e18c8c.gif?imageMogr2/auto-orient/strip)

  * 拖动页面，控件会跟着滑动
  * 放开页面，页面会自动滑动到最中间

> 代码分析

![](http://upload-images.jianshu.io/upload_images/1239900-12ee95f942b7fc4b.jpeg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

- LinearLayout中水平放置几个View
- onTouch方法中处理拖动事件，根据拖动距离调用Scroll方法，因为move事件是连续触发的，所以有了拖动的动画感觉
- 动作取消的时候，调用Scroller的start方法，平滑的移动页面到某个位置，计算当前时间点的滑动位置，调用Scroll方法，实现滑动
- 滑动未结束的时候，由invalidate方法不断的循环出发刷新，造成缓慢滑动的现象
- scroll方法的调用，会引起Canvas 的translate，从而影响显示效果

[源码地址](https://github.com/binye33333/android/tree/master/app/src/main/java/com/teach/yo/codeshop/view ) 

>滑动流程

![](http://upload-images.jianshu.io/upload_images/1239900-6702680d35e64574.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

Scroller 类，处理时间和坐标点的辅助类，协助完成自行滑动时间段内滑动，加速减速效果。
它本身并不提供动画效果，只提供计算，动画效果是由View 自身的invalidate递归调用scrollBy实现的。

重点方法：
Scroller(Context context) 构造函数，默认的会初始化一个减速插值器-ViscousFluidInterpolator对象，显示加速滑动效果；
startScroll(int startX, int startY, int dx, int dy) 开始滑动计算
computeScrollOffset() 手动计算滑动偏移量，滑动位置会保存在Scroller对象中，滑动完成会返回false 
getCurrX()/getCurrY() 获取计算所得的当前位置值，用于滑动
abortAnimation()  ，结束计算

> 动画流程

View的滑动有一种特殊动画效果，那么系统本身的动画框架Animation怎么实现的呢？

![](http://upload-images.jianshu.io/upload_images/1239900-626fee0ad1df4d94.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

原理：
如果动画没有结束，递归调用invalidate方法，不断触发页面的重绘，显示帧的切换
利用Interpolator做平滑时间流逝到非平滑时间过渡，造成加速减速效果
利用applyTransformation做矩阵变换，然后在View中canvas.concat(Matrix matrix)

以下是RotateAnimation的矩阵变化部分:
![](http://upload-images.jianshu.io/upload_images/1239900-37d72c5e0766fdc7.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

> 总结

Android 的动画运行在主线程中，都是通过invalidate方法循环调用触发
具体的单帧显示效果都是由View的canvas变换所得，所以说这些变动只是影响了显示效果，不影响layout和measure
