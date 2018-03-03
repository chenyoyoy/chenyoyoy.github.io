> ViewPager 和 PagerAdapter 的关键方法

![关联方法](http://upload-images.jianshu.io/upload_images/1239900-087563a1339d5a5a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

ViewPager：

    setAdapter() 设置适配器 ；
    dataSetChanged() Adapter中数据变化时候的监听回调处理方法；
    populate() ViewPager中填充页面item时候的处理方法

PagerAdapter：

    startUpdate()  Viewpager显示的页面数据有所改变的回调
    finishUpdate() 页面数据改变的处理结束后的回调方法
    instantiateItem() 初始化一个item数据的时候的回调
    destroyItem() 销毁一个item数据的时候会回调
    setPrimaryItem()设置好当前显示item后的回调
    isViewFromObject()  View 是否和 Object有关联关系
    getItemPosition() 获取当前数据对应的位置
    getPageTitle() 获取当前页面对应的标题
    getCount() 获取总的item数量
    getPageWidth() 获取item页面相对于ViewPager宽度

> setAdapter中的Adapter 方法调用

    public void setAdapter(PagerAdapter adapter) {

    if (mAdapter != null) {
        //取消之前的adapter数据监听
        mAdapter.setViewPagerObserver(null);  
        //老的关联数据需要销毁，意味着有数据改变，所以回调startUpdate方法                 
        mAdapter.startUpdate(this); 
        //移除之前的数据
        for (int i = 0; i < mItems.size(); i++) {
            final ItemInfo ii = mItems.get(i);
            //销毁之前的item
            mAdapter.destroyItem(this, ii.position, ii.object);
        }
        //数据改变结束，回调finishUpdate方法
        mAdapter.finishUpdate(this);
        //清楚之前的一些缓存变量 省略。。。。
    }
    
    //重新设置初始化变量 。。。。。
     final PagerAdapter oldAdapter = mAdapter;
     。。。。。
    
    // 如果之前有保留状态，这里恢复
        if (mRestoredCurItem >= 0) {
            mAdapter.restoreState(mRestoredAdapterState, mRestoredClassLoader);
            setCurrentItemInternal(mRestoredCurItem, false, true);
            mRestoredCurItem = -1;
            mRestoredAdapterState = null;
            mRestoredClassLoader = null;
        } else if (!wasFirstLayout) {
           //重新填充每个item
            populate();
        } else {
            requestLayout();
        }
    }
}
</code>

> populate

简要的来说，populate方法就是在ViewPager的FrameLayout上面，填充上需要显示的item页面 。

![页面示例图](http://upload-images.jianshu.io/upload_images/1239900-5adca64fe16afc5b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


    //省略。。。。
    //viewpager 要填充item了，回调startUpdate方法
    mAdapter.startUpdate(this);
    //以下代码用来计算需要实例化的item位置及数量
    final int pageLimit = mOffscreenPageLimit;
    final int startPos = Math.max(0, mCurItem - pageLimit);
    final int N = mAdapter.getCount();
    final int endPos = Math.min(N-1, mCurItem + pageLimit);
    //省略。。。。
    //左边缓存页面的处理
    for (int pos = mCurItem - 1; pos >= 0; pos--) {
    //以前的左边缓存数量大于现在需要的左边缓存数量，需要考虑销毁
    if (extraWidthLeft >= leftWidthNeeded && pos < startPos) {
        if (ii == null) {
            break;
        }
        //之前的缓存的item没有在执行滚动动画，可以销毁
        if (pos == ii.position && !ii.scrolling) {
            //移除ViewPager中的item相关数据
            mItems.remove(itemIndex);
            //回调destroyItem方法，通知PagerAdapter处理相关的销毁动作
            mAdapter.destroyItem(this, pos, ii.object);
            //减小index，ii 重新赋值
            itemIndex--;
            curIndex--;
            ii = itemIndex >= 0 ? mItems.get(itemIndex) : null;
        }
    //填充左边的item的时候，发现之前有实例化的item可以放到这个位置上，直接将item的宽度加在左边
    } else if (ii != null && pos == ii.position) {
        extraWidthLeft += ii.widthFactor;
        itemIndex--;
        ii = itemIndex >= 0 ? mItems.get(itemIndex) : null;
    } else {
        //左边这个位置，之前没有过item实例化，需要根据位置实例化一个item与之关联，同时计算左边需要的宽度
        ii = addNewItem(pos, itemIndex + 1);
        extraWidthLeft += ii.widthFactor;
        curIndex++;
        ii = itemIndex >= 0 ? mItems.get(itemIndex) : null;
      }
    }
以下是addNewItem的源码

    ItemInfo addNewItem(int position, int index) {
    ItemInfo ii = new ItemInfo();
    //item对应的位置 。数据刷新的时候，可能数据本身未变，但是位置改变了，就需要调整这个position
    ii.position = position;
    //调用Adapter的 instantiateItem方法，根据位置初始化一个item，并返回一个Object与之关联，类似于一个key
    ii.object = mAdapter.instantiateItem(this, position);
    //获取宽度当前页面item宽度
    ii.widthFactor = mAdapter.getPageWidth(position);
    //数据保存到mItems中
    if (index < 0 || index >= mItems.size()) {
        mItems.add(ii);
    } else {
        mItems.add(index, ii);
    }
    return ii;
    }

>  dataSetChanged

    setAdapter的时候，Viewpager 会设置一个监听器到Adapter中，去监听数据改变，然后调用到dataSetChanged方法。

    boolean isUpdating = false;
    //遍历现有的缓存数据
    for (int i = 0; i < mItems.size();i++)
    {
    final ItemInfo ii = mItems.get(i);
    //获取每个item对应的position  。Viewpager里面缓存的时候老的Position，当数据发生改变的时候，
    或许itemInfo 对应的位置发生了改变，所以需要通过Adapter重新获取
    // 这里会调用Adapter 中的 getItemPosition方法
    final int newPos = mAdapter.getItemPosition(ii.object);
    //如果itemInfo对应的数据位置没有发生改变，继续处理其他的item
    if (newPos == PagerAdapter.POSITION_UNCHANGED){        
       continue;
    }
    //如果itemInfo对应的数据，在新的数据集合中没有了，需要销毁itemInfo
    if (newPos == PagerAdapter.POSITION_NONE) {
        mItems.remove(i);
        i--;
        //告诉Adapter有数据的改变，对应的Adapter要处理相关工作
        if (!isUpdating) {
            mAdapter.startUpdate(this);
            isUpdating = true;
        }
        //销毁一个位置的item
        mAdapter.destroyItem(this, ii.position, ii.object);
       //有数据改变，item的销毁，需要重新布局填充页面数据
        needPopulate = true;
        if (mCurItem == ii.position) {
            // Keep the current item in the valid range
            newCurrItem = Math.max(0, Math.min(mCurItem, adapterCount - 1));
            needPopulate = true;
        }
        continue;
    }
    //当前数据对应的位置发生了改变，给itemInfo重新赋值position，重新布置页面
    if (ii.position != newPos) {
        if (ii.position == mCurItem) {
            // Our current item changed position. Follow it.            
     newCurrItem = newPos;
        }
        ii.position = newPos;
        needPopulate = true;
     }
    }
    //如果之前有数据的改变，回调finishUpdate方法，表示处理结束
    if (isUpdating) {
    mAdapter.finishUpdate(this);
    }
    //按照位置，重新排序关联的itemInfo
    Collections.sort(mItems, COMPARATOR);

> PagerAdapter中 notifyDataSetChanged不起作用的问题

PagerAdapter中调用notifyDataSetChanged方法，最终会调用到
dataSetChanged() 。在代码执行过程中，会重新获取新的数据中的位置，调用getItemPosition方法，如果位置未发生改变，就不做处理。

      public int getItemPosition(Object object) 
      {
         return POSITION_UNCHANGED;
      }

然后，getItemPosition方法返回值是默认值，不做处理。所以新的数据结构中，如果item数据发生了改变，需要重写这个方法，调整新数据集合中，原有的ItemInfo 对应的数据位置。

> FragmentStatePagerAdapter 最官方的ViewPager示例

* instantiateItem(ViewGroup container, int position)
      if (mFragments.size() > position) {
      //Adapter 中保存了之前实例化过的Framgent，再次显示的时候，直接从缓存中获取
      Fragment f = mFragments.get(position);
      if (f != null) {
        return f;
      }}
      //开启事务
      if (mCurTransaction == null) {
      mCurTransaction = mFragmentManager.beginTransaction();
      }
      //根据位置初始化一个item ，之后就不会初始化了，会从缓存中获取
      Fragment fragment = getItem(position);
      //如果之前的fragment状态中有保存，恢复状态
      if (mSavedState.size() > position) {
      Fragment.SavedState fss = mSavedState.get(position);    if (fss != null) {
      fragment.setInitialSavedState(fss);
      }}
      //保证fragment状态数量和fragment数量一致，没有状态的设置null
      while (mFragments.size() <= position) {
      mFragments.add(null);
      }
      //设置不可见
      fragment.setMenuVisibility(false);
      fragment.setUserVisibleHint(false);
      mFragments.set(position, fragment);
      //将fragment添加到Manager中进行管理
      mCurTransaction.add(container.getId(), fragment);
      return fragment;

* destroyItem()

      //由于instantiateItem方法返回值是Fragment，所以这里可以强转
      Fragment fragment = (Fragment) object;
      if (mCurTransaction == null) {
      mCurTransaction = mFragmentManager.beginTransaction();
      }
      //mSavedState中填满null数据
      while (mSavedState.size() <= position) {            
      mSavedState.add(null);
      }
      //之前有add过的fragment状态保存到mSavedState中
      mSavedState.set(position, fragment.isAdded()?mFragmentManager.saveFragmentInstanceState(fragment) :null);
      //mFragments移除保存的fragment实例
      mFragments.set(position, null);
      //manager中接触对fragment的管理
      mCurTransaction.remove(fragment);

* setPrimaryItem() 

      Fragment fragment = (Fragment)object;
      if (fragment != mCurrentPrimaryItem) ;
      //之前的显示的fragment设置不可见
      if (mCurrentPrimaryItem != null) {
      mCurrentPrimaryItem.setMenuVisibility(false);         
      mCurrentPrimaryItem.setUserVisibleHint(false);
      }
      //当前fragment设置显示状态为可见
       if (fragment != null) {
        fragment.setMenuVisibility(true);           
        fragment.setUserVisibleHint(true);
       }
       mCurrentPrimaryItem = fragment;
       }

* finishUpdate

      if (mCurTransaction != null) 
      {//提交事务
      mCurTransaction.commitNowAllowingStateLoss();
      mCurTransaction = null;
      }

> 总结一下FragmentStatePagerAdapter中的调用逻辑

*  instantiateItem 或者 destroyItem 做 add 或者 remove操作 
*  finishUpdate 中做事务的提交
*  setPrimaryItem 中做Fragment的显示隐藏控制，标志当前显示fragment
* Viewpager 不做Fragment的管理，以及View的显示，这些都是通过FragmentManager的事务来处理的
* 如果要做fragment数据的刷新，位置改变，需要 重写getItemPosition方法，同时控制 instantiateItem中对于缓存的获取，mFragments.get(position) 自定义一套缓存获取规则。
