>ViewPager应用场景

* App 引导页
* 广告Banner
* 分类tab
* 图片浏览器

![ViewPager应用场景](http://upload-images.jianshu.io/upload_images/1239900-66027f655be9a7f8.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

>ViewPager+PagerAdapter 的简单实现

![显示效果.png](http://upload-images.jianshu.io/upload_images/1239900-b26103516091fb30.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/300)

整个页面视图分为三部分：
顶部的PAGE0,PAGE1....等tab，使用的是PagerSlidingTabStrip 控件；
底下的整个页面为Viewpager控件；
图片部分为Viewpager 的一个子页面；

> 代码解析

![Paste_Image.png](http://upload-images.jianshu.io/upload_images/1239900-9626b0b8cd8e87c2.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

Viewpager 是一个类似于Listview的集合数据展示控件，需要配合PagerAdapter一起使用，以下介绍最基本的四个方法：

public abstract int getCount();
返回Viewpager控件内部的子View数量；
返回集合数据的长度即可。

public Object instantiateItem(ViewGroup container, int position) 
初始化当前位置的item，类似于listview的getView方法，滑动到一个新的位置时，需要构建当前view 。
在baseAdapter 的getView方法中，我们只需要返回一个View 就可 ，Listview 会直接帮我们添加显示出来；而 pagerAdapter 中，我们需要自己add到容器布局里面；
另外，baseAdapter 的getView 返回的是View,而instantiateItem（）返回值为 Object ，并不一定是一个View ，说明 Viewpager 并不帮我们处理View 的显示 ，需要我们自己添加到视图中 。

public void destroyItem(ViewGroup container, int position, Object object)
销毁一个item 。
在有限的空间内展示为止数量的数据，必然要将一些视图范围外的资源销毁，不然会导致oom。

因为Viewpager并不帮忙处理View相关事物，所以我们在instantiateItem 中创建视图，需要在destroyItem中销毁视图 。
这里的Object 就是instantiateItem 方法返回的Obeject  ，上面的方法中，我们直接返回了当前位置的View ，所以我们可以remove.
如果instantiateItem的方法并不是返回的View,那么我们就需要通过position，或者Object 找到当前View 进行销毁。

public abstract boolean isViewFromObject(View view, Object object);
判断当前View是否和Object有关联。
ViewPager 通过instantiateItem方法只是拿到了一个Object ，ViewPager里面关联的数据就只有Object数据 ；而 视图的滑动，操作的是View ，Viewpager 需要通过 滑动位置计算得到 当前View，并且拿到 对应的Object ，从而获取保存在Viewpager 中该位置的相关信息。


> 总结

* Viewpager 和 PagerAdapter 需要配套使用
* PagerAdapter 通过getCount 方法返回item数量
* instantiateItem 用于创建当前位置的View，并返回与之关联的Object ；
* destroyItem 用于销毁当前View ;
* isViewFromObject 用于表示View 和 Object是否对应
