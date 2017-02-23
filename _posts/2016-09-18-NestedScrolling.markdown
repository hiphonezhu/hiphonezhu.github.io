---
layout:     post
title:      "Android NestedScrolling机制"
subtitle:   ""
date:       2016-09-18 12:00:00
author:     "zhuhf"
header-img: ""
header-mask: 0.3
catalog:    true
tags:
    - Android
    - NestedScrolling
---

## 一、概述
这样一个效果图，我们思考下如何实现

![nestedscrolling.gif](http://upload-images.jianshu.io/upload_images/1787010-6696bdfa50eba96d.gif?imageMogr2/auto-orient/strip)
可以看到“Sticky View”滚动到顶部会“固定住”，列表下拉到第一条数据“Sticky View”又会一起往下滚动。

有人说，这个不就是View的事件分发吗？
>假设我们按照传统的事件分发去理解，我们滑动的是下面的内容区域View，但是滚动的却是外部的ViewGroup，那么肯定是ViewGroup拦截了子View的事件；但是，上面的效果图，当ViewGroup滑动到一定程度，子View又开始滑动了，而且中间的过程是没有间断的。从正常的事件分发机制来讲这个是不可能的，因为当ViewGroup拦截事件后，是没办法再次交还给子View去处理的（除非你手动干预了事件的分发），关于这一点如果有不清楚的同学，可以先去了解下Android的事件分发机制。

那么有没有其他方案去解决我们的问题呢？答案是，有。

Android在support.v4包中为我们引入两个重要的接口：
* NestedScrollingParent
* NestedScrollingChild

有了上面这两个类，我们就可以实现“NestedScrolling（嵌套滚动）”的无缝衔接。

## 二、实现
上述效果图，分为三个部分：顶部布局（ImageView），中间的“Sticky View”（TextView）和底部的列表（RecyclerView）。

RecyclerView已经实现了NestedScrollingChild接口，所以本文的重点是实现NestedScrollingParent接口。

(1) 布局

    <?xml version="1.0" encoding="utf-8"?>
    <FrameLayout xmlns:android="http://schemas.android.com/apk/res/android"
        android:layout_width="match_parent"
        android:layout_height="match_parent">
        <com.example.hiphonezhu.nestedscrolling.StickyLayout
            android:id="@+id/stickyNavLayout"
            android:layout_width="match_parent"
            android:layout_height="match_parent"
            android:orientation="vertical">
            <ImageView
                android:id="@+id/iv"
                android:layout_width="match_parent"
                android:layout_height="100dp"
               android:scaleType="centerCrop"
                android:src="@drawable/bg" />
            <TextView
                android:id="@+id/tv_sticky"
                android:layout_width="match_parent"
                android:layout_height="wrap_content"
                android:background="@android:color/holo_green_dark"
                android:gravity="center"
                android:paddingBottom="10dp"
                android:paddingTop="10dp"
                android:text="Sticky View"
                android:textColor="@android:color/white" />
            <android.support.v7.widget.RecyclerView
                android:id="@+id/rv"
                android:layout_width="match_parent"
                android:layout_height="match_parent" />
        </com.example.hiphonezhu.nestedscrolling.StickyLayout>
    </FrameLayout>

StickyLayout是直接继承自LinearLayout，并且实现了NestedScrollingParent接口。

(2) 实现NestedScrollingParent接口
在具体实现之前，我们先看下这个接口的几个方法。

    public interface NestedScrollingParent {
        public boolean onStartNestedScroll(View child, View target, int nestedScrollAxes);
        public void onNestedScrollAccepted(View child, View target, int nestedScrollAxes);
        public void onStopNestedScroll(View target);
        public void onNestedScroll(View target, int dxConsumed, int dyConsumed,        int dxUnconsumed, int dyUnconsumed);
        public void onNestedPreScroll(View target, int dx, int dy, int[] consumed);
        public boolean onNestedFling(View target, float velocityX, float velocityY, boolean consumed);
        public boolean onNestedPreFling(View target, float velocityX, float velocityY);
        public int getNestedScrollAxes();
    }

我们需要重点关注下面几个方法
* onStartNestedScroll该方法返回true，代表当前ViewGroup能接受内部View的滑动参数（这个内部View不一定是直接子View），一般情况下建议直接返回true，当然你可以根据nestedScrollAxes：判断垂直或水平方向才返回true。
* onNestedPreScroll该方法会传入内部View移动的dx与dy，当前ViewGroup可以消耗掉一定的dx与dy，然后通过最后一个参数consumed传回给子View。例如，当前ViewGroup消耗掉一半dx与dy

      scrollBy(dx/2, dy/2);
      consumed[0] = dx/2;
      consumed[1] = dy/2;

* onNestedPreFling你可以捕获对内部View的fling事件，返回true表示拦截掉内部View的事件

  我们看下具体的代码实现（仅是关键代码）：
```
  public class StickyLayout extends LinearLayout implements NestedScrollingParent {
      @Override
      public boolean onStartNestedScroll(View child, View target, int nestedScrollAxes)
      {
          return true;
      }
      @Override
      public void onNestedPreScroll(View target, int dx, int dy, int[] consumed)
      {
          // dy > 0表示子View向上滑动;

          // 子View向上滑动且父View的偏移量<ImageView高度
          boolean hiddenTop = dy > 0 && getScrollY() < maxScrollY;

          // 子View向下滑动(说明此时父View已经往上偏移了)且父View还在屏幕外面, 另外内部View不能在垂直方向往下移动了
          /**
           * ViewCompat.canScrollVertically(view, int)
           * 负数: 顶部是否可以滚动(官方描述: 能否往上滚动, 不太准确吧~)
           * 正数: 底部是否可以滚动
           */
          boolean showTop = dy < 0 && getScrollY() > 0 && !ViewCompat.canScrollVertically(target, -1);

          if (hiddenTop || showTop)
          {
              scrollBy(0, dy);
              consumed[1] = dy;
          }
      }
      @Override
      public boolean onNestedPreFling(View target, float velocityX, float velocityY)
      {
          if (velocityY > 0 && getScrollY() < maxScrollY) // 向上滑动, 且当前View还没滑到顶
          {
              fling((int) velocityY, maxScrollY);
              return true;
          }
          else if (velocityY < 0 && getScrollY() > 0) // 向下滑动, 且当前View部分在屏幕外
          {
              fling((int) velocityY, 0);
              return true;
          }
          return false;
      }
  }    
```
* onNestedPreScroll中，判断子View上滑（```dy>0```）并且```StickyLayout```滚动到屏幕外的距离（```getScrollY()```）< 最大滚动距离```maxScrollY```，则隐藏顶部布局（```ImageView```）；同理，如果子View下滑（```dy < 0```）且```StickyLayout```还在屏幕外面（```getScrollY() > 0```），同时内部View不能在垂直方向往下移动了（可以借助```ViewCompat.canScrollVertically```来实现）。
> ViewCompat.canScrollVertically(view, int) ，第二个int类型参数
负数: 顶部是否可以往下滚动
正数: 底部是否可以往上滚动

  官方描述：“Negative to check scrolling up, positive to check scrolling down”，我觉得有误人子弟的嫌疑。
* onNestedPreFling中，如果向上滑动（```velocityY > 0```）且```ImageView```没有完全隐藏（```getScrollY() < maxScrollY```），则使用fling方法，“尝试”（因为滑动距离取决于初始速度）将```ImageView```完全隐藏；同理，如果向下滑动（```velocityY < 0```）且```ImageView```部分在屏幕外(```getScrollY() > 0```)，则使用fling方法，“尝试”（因为滑动距离取决于初始速度）将```ImageView```完全显示。

对于fling方法，我们使用OverScroller的fling方法，另外边界检测，重写了scrollTo方法：

    public void fling(int velocityY, int maxY)
    {    
        mScroller.fling(0, getScrollY(), 0, velocityY, 0, 0, 0, maxY);          
        invalidate();
    }

    @Override
    public void scrollTo(int x, int y)
    {
        if (y < 0) // 不允许向下滑动
        {
            y = 0;
        }
        if (y > maxScrollY) // 防止向上滑动距离大于最大滑动距离
        {
            y = maxScrollY;
        }
        if (y != getScrollY())
        {
            super.scrollTo(x, y);
        }
    }
    @Override
    public void computeScroll()
    {
        if (mScroller.computeScrollOffset())
        {
            scrollTo(0, mScroller.getCurrY());
            invalidate();
        }      
    }

到这里，大家发现其实NestedScrolling机制其实并不复杂：

在滑动的时候，内部View会把滑动的距离（dx与dy）传入给NestedScrollingParent，NestedScrollingParent可以决定对其是否消耗，消耗的值通过consumed[]再传回给子View。

## 三、写在最后
由于本文的效果ImageView和Sticky View（TextView）与“状态栏”有融合的效果，所以具体源码会比这个略微复杂些~

主要思路是：
>布局中有一个一模一样的Sticky View（TextView），通过隐藏和显示它来达到最终的效果，如果你有更好的想法可以联系我。

具体请参考源码：[NestedScrolling](https://github.com/hiphonezhu/Android-Demos/tree/master/NestedScrolling)
