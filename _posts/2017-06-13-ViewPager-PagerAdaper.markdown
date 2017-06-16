---
layout:     post
title:      "探究 ViewPager  使用 Fragment 无法刷新的原因"
subtitle:   ""
date:       2017-06-13 16:36:00
author:     "zhuhf"
header-img: ""
header-mask: 0.3
catalog:    true
tags:
    - ViewPager
    - FragmentPagerAdapter
    - FragmentStatePagerAdapter
---



> 本文将从源码角度探究 ViewPager 使用 FragmentPagerAdapter、FragmentStatePagerAdapter 无法刷新的原因，以及对应的解决方案。

让我们先从一个简单的例子入手，请看下面这一段代码：

    public class FragmentStatePagerItemAdapter extends FragmentStatePagerAdapter {
        FragmentManager fm;
        private List<Fragment> fragments;

        public FragmentStatePagerItemAdapter(FragmentManager fm, List<Fragment> fragments) {
            this.fm = fm;
            this.fragments = fragments;
        }

        public void setDataSource(List<Fragment> fragments) {
            this.fragments = fragments;
        }

        @Override
        public Fragment getItem(int position) {
            return fragments.get(position);
        }

        @Override
        public int getCount() {
            return (fragments != null) ? fragments.size() : 0;
        }
    }

这是一段很常见的 FragmentStatePagerAdapter 的实现，使用的方法也很简单：

    private void bindValue(List<Fragment> fragments) {
        if (myAdapter == null) {
            myAdapter = new FragmentStatePagerItemAdapter(getSupportFragmentManager(), fragments);
            viewPager.setAdapter(myAdapter);
        } else {
            myAdapter.setDataSource(fragments);
        }
        myAdapter.notifyDataSetChanged();
    }

我们只需要提供 fragments 数据源，调用此方法就可以刷新 ViewPager 展示的内容了。
![此处输入图片的描述][1]
确实，如果真是这样，那么本文就没有写的必要了。

有过实践经验的同学应该都知道，上面的做法是无法刷新 ViewPager 的（这里指的是第二次调用此方法）。

解决方案一般是在 FragmentStatePagerItemAdapter ，加上下面这个方法：

    public class FragmentStatePagerItemAdapter extends FragmentStatePagerAdapter {
        ...
        @Override
        public int getItemPosition(Object object) {
            return POSITION_NONE;
        }
        ...
    }

![此处输入图片的描述][2]



那么，这么做背后的原理是什么呢？让我们来一探究竟~

POSITION_NONE
=============

我们知道，所有 Adapter（和 ListView、RecyclerView、ViewPager 一起使用等） 刷新的流程都是：

 1. 改变数据源（List data）；
 2. 调用 adapter.notifyDataSetChanged() 方法。

我们有理由怀疑，问题应该出在 notifyDataSetChanged() 这个方法，看一下它的实现：

    public abstract class PagerAdapter {
        ...
        public void notifyDataSetChanged() {
            synchronized (this) {
                if (mViewPagerObserver != null) {
                   mViewPagerObserver.onChanged();
                }
            }
            ...
        }
        
        // 这里为 mViewPagerObserver 赋值
        void setViewPagerObserver(DataSetObserver observer) {
            synchronized (this) {
                mViewPagerObserver = observer;
            }
        }
        ...
    }

我们只看下关键的几个方法，notifyDataSetChanged() 内部是调用 mViewPagerObserver 的 onChanged() 方法。

而 mViewPagerObserver 是通过 setViewPagerObserver() 方法赋值的，调用 setViewPagerObserver() 方法的地方在 ViewPager 的 setAdaper() 方法中：

    public class ViewPager extends ViewGroup {
        ...
        public void setAdapter(PagerAdapter adapter) {
            ...
            if (mAdapter != null) {
                if (mObserver == null) {
                    mObserver = new PagerObserver();
                }
                mAdapter.setViewPagerObserver(mObserver);
            }
            ...
        }
        ...
    }

这样就明显了，其实最终调用的是 PagerObserver 的 onChanged() 方法。PagerObserver 是 ViewPager 的内部类，它的实现也很简单：

    private class PagerObserver extends DataSetObserver {
        PagerObserver() {
        }
        @Override
        public void onChanged() {
            dataSetChanged();
        }
        @Override
        public void onInvalidated() {
            dataSetChanged();
        }
    }

可以看到，onChanged() 内部调用的又是 ViewPager 的 dataSetChanged() 的方法。

我们有理由相信，文章开始的 FragmentStatePagerItemAdapter 重写的 getItem() 方法会在这个方法内部被调用，让我们继续一探究竟~

    void dataSetChanged() {
        ...
        boolean needPopulate = mItems.size() < mOffscreenPageLimit * 2 + 1
                && mItems.size() < adapterCount;
        ...
        for (int i = 0; i < mItems.size(); i++) {
            final ItemInfo ii = mItems.get(i);
            final int newPos = mAdapter.getItemPosition(ii.object);

            if (newPos == PagerAdapter.POSITION_UNCHANGED) {
                continue;
            }

            if (newPos == PagerAdapter.POSITION_NONE) {
                ...
                mAdapter.destroyItem(this, ii.position, ii.object);
                needPopulate = true;
                ...
            }
        }
        ...
        if (needPopulate) {
            ...
            setCurrentItemInternal(newCurrItem, false, true);
            requestLayout();
        }
    }

needPopulate：是否需要重新布局，也就是刷新的意思。

needPopulate 为 true 的条件是 newPos == PagerAdapter.POSITION_NONE，而 newPos 是通过 getItemPosition() 方法得到的：

    public int getItemPosition(Object object) {
        return POSITION_UNCHANGED;
    }

这个方法在默认情况下，返回是常量 POSITION_UNCHANGED，它的含义是未发生变化。

这就解释了为何返回 POSITION_NONE，ViewPager 就能够正常刷新了。

经过一系列调用，最终会调用 PagerAdapter 的 instantiateItem(...) 方法。
> 调用链：ViewPager.dataSetChanged() -> ViewPager.setCurrentItemInternal(...) -> ViewPager.populate(...) -> ViewPager.addNewItem(...) -> PagerAdapter.instantiateItem(...)

而PagerAdapter 的实现有两个：FragmentStatePagerAdapter 和 FragmentPagerAdapter，所以我们分两种情况来分析下~

instantiateItem() & destroyItem()
===============
在分析 instantiateItem() 之前，需要提及另一个与之对应的方法：destroyItem()，这个方法表示销毁一个 Item。

对应的，instantiateItem() 表示创建一个 Item。
> 在上文 dataSetChanged() 方法中，POSITION_NONE 情况下，会去调用 destroyItem() 方法。

这两个方法非常重要，它们共同决定了 Fragment 是“新建”还是从“缓存”读取。

FragmentStatePagerAdapter
-------------------------
    ...
    @Override
    public Object instantiateItem(ViewGroup container, int position) {
        if (mFragments.size() > position) {
            Fragment f = mFragments.get(position);
            if (f != null) {
                return f;
            }
        }
        if (mCurTransaction == null) {
            mCurTransaction = mFragmentManager.beginTransaction();
        }

        Fragment fragment = getItem(position);
        ...
        mCurTransaction.add(container.getId(), fragment);

        return fragment;
    }
    
    @Override
    public void destroyItem(ViewGroup container, int position, Object object) {
        Fragment fragment = (Fragment) object;

        if (mCurTransaction == null) {
            mCurTransaction = mFragmentManager.beginTransaction();
        }
        ...
        mFragments.set(position, null);

        mCurTransaction.remove(fragment);
    }
    ...

 mFragments 是用来缓存 Fragment 的集合，在 destroyItem() 方法中，把对应 position 位置的元素设置为 null。
 
这样使得 instantiateItem() 从缓存中获取的 f == null，然后会调用我们重写的 getItem() 方法。

分析完 FragmentStatePagerAdapter，开始我觉得 FragmentPagerAdapter 的使用应该也是如此吧？

于是有了下面这段代码：

    public class FragmentPagerItemAdapter extends FragmentPagerAdapter {
        FragmentManager fm;
        private List<Fragment> fragments;

        public FragmentStatePagerItemAdapter(FragmentManager fm, List<Fragment> fragments) {
             this.fm = fm;
             this.fragments = fragments;
        }

        public void setDataSource(List<Fragment> fragments) {
             this.fragments = fragments;
        }

        @Override
        public Fragment getItem(int position) {
            return fragments.get(position);
        }

        @Override
        public int getCount() {
            return (fragments != null) ? fragments.size() : 0;
        }
    
        @Override
        public int getItemPosition(Object object) {
            return POSITION_NONE;
        }
    }

然后测试了一下，发现居然不会刷新，纳尼？？？
![此处输入图片的描述][3]

根据上面分析的，十有八九原因出在 instantiateItem() 和 destroyItem() 这两个方法的实现。

FragmentPagerAdapter
--------------------

    @Override
    public Object instantiateItem(ViewGroup container, int position) {
        if (mCurTransaction == null) {
            mCurTransaction = mFragmentManager.beginTransaction();
        }

        final long itemId = getItemId(position);

        // 根据 Tag 查找 Fragment
        String name = makeFragmentName(container.getId(), itemId);
        Fragment fragment = mFragmentManager.findFragmentByTag(name);
        if (fragment != null) {
            mCurTransaction.attach(fragment);
        } else {
            // 不存在
            fragment = getItem(position);
            mCurTransaction.add(container.getId(), fragment,
                    makeFragmentName(container.getId(), itemId));
        }
        ...

        return fragment;
    }

    @Override
    public void destroyItem(ViewGroup container, int position, Object object) {
        if (mCurTransaction == null) {
            mCurTransaction = mFragmentManager.beginTransaction();
        }
        // detach Fragment
        mCurTransaction.detach((Fragment)object);
    }

区别就在于 destroyItem() 对 Fragment 的“销毁”机制不一样。

FragmentPagerAdapter.destroyItem() 方法对于 Fragment 的处理使用的是 detach() 操作，而 FragmentStatePagerAdapter.destroyItem() 使用的是 remove() 操作，区别如下：

- detach() 只会把 Fragment 从 View hierarchy 中移除，但是它的“状态”还在 FragmentManager 中存在，这就意味着通过 mFragmentManager.findFragmentByTag() 还是可以找到对应的 Fragment。
- remove() 同时会从 View hierarchy 和 FragmentManager 中移除 Fragment。

分析到这里，你是不是有解决思路了？
![此处输入图片的描述][4]

没错，就是通过重写 FragmentPagerAdapter 的 destroyItem() 方法，使用 FragmentManager.remove() 将 Fragment 移除。

    public class FragmentPagerItemAdapter extends FragmentPagerAdapter {
        ...
        @Override
        public void destroyItem(ViewGroup container, int position, Object object) {
            super.destroyItem(container, position, object);
            if (mCurTransaction == null) {
                mCurTransaction = fm.beginTransaction();
            }
            mCurTransaction.remove((Fragment)object);
            mCurTransaction.commitNowAllowingStateLoss();
        }
        ...
    }


----------


over ~

  [1]: http://qq.yh31.com/tp/zjbq/201706081719583453.jpg
  [2]: http://qq.yh31.com/tp/zjbq/201706081719585140.jpg
  [3]: http://qq.yh31.com/tp/zjbq/201706081719588140.jpg
  [4]: http://qq.yh31.com/tp/zjbq/201706081719588531.jpg