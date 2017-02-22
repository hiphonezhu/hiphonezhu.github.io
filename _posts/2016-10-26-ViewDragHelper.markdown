---
layout:     post
title:      "Android自定义ViewGroup神器-ViewDragHelper"
subtitle:   ""
date:       2016-10-26 12:00:00
author:     "zhuhf"
header-img: ""
header-mask: 0.3
catalog:    true
tags:
    - Android 7.0
    - FileProvider
---



## 一、概述
>ViewDragHelper is a utility class for writing custom ViewGroups. It offers a number of useful operations and state tracking for allowing a user to drag and reposition views within their parent ViewGroup.

这是官方的解释：在自定义ViewGroup时，ViewDragHelper可以用来拖拽和设置子View的位置（在ViewGroup范围内）。另外，还提供了一系列的方法和状态跟踪。

可见，在自定义ViewGroup时，ViewDragHelper一般用来处理子View的位置移动。

## 二、入门示例

![demo1.gif](http://upload-images.jianshu.io/upload_images/1787010-9197bc3e11623ba6.gif?imageMogr2/auto-orient/strip)
效果很简单，屏幕中间有两个TextView，位置随着我们的手指不断移动。

传统方式实现：一般需要重写``onInterceptTouchEvent``和``onTouchEvent``这两个方法，写好这两个方法不是一件容易的事情，需要自己去处理：事件冲突、加速检测等。

ViewDragHelper简化了很多工作，让我们更加关注“业务”的需求，实现步骤如下：
1. 创建ViewDragHelper实例
2. 处理ViewGroup的触摸事件
3. ViewDragHelper.Callback的编写

#### (一)  自定义ViewGroup

    public class VDHLinearLayout extends LinearLayout {
      ViewDragHelper dragHelper;

      public VDHLinearLayout(Context context, AttributeSet attrs) {
          super(context, attrs);
          dragHelper = ViewDragHelper.create(this, 1.0f, new ViewDragHelper.Callback() {
              @Override
              public boolean tryCaptureView(View child, int pointerId) {
                  return true;
              }

              @Override
              public int clampViewPositionVertical(View child, int top, int dy) {
                  return top;
              }

              @Override
              public int clampViewPositionHorizontal(View child, int left, int dx) {
                  return left;
              }
          });
      }

      @Override
      public boolean onInterceptTouchEvent(MotionEvent ev) {
          return dragHelper.shouldInterceptTouchEvent(ev);
      }

      @Override
      public boolean onTouchEvent(MotionEvent event) {
          dragHelper.processTouchEvent(event);
          return true;
      }
    }

VDHLinearLayout的代码还是非常简单的，主要是分为以下三个步骤：

1. 创建ViewDragHelper实例
```
dragHelper = ViewDragHelper.create(this, 1.0f, new ViewDragHelper.Callback() {});
```
创建需要三个参数，第一个为当前的ViewGroup，第二个为``sensitivity``，主要用于设置``touchSlop``：
```
helper.mTouchSlop = (int) (helper.mTouchSlop * (1 / sensitivity));
```
传入越大，``touchSlop``就越小。第三个参数就是ViewDragHelper.Callback，触摸过程中会回调相关方法。

2. 实现ViewDragHelper.Callback相关方法
```
new ViewDragHelper.Callback() {
      @Override
      public boolean tryCaptureView(View child, int pointerId) {
          return true;
      }

      @Override
      public int clampViewPositionVertical(View child, int top, int dy) {
          return top;
      }

      @Override
      public int clampViewPositionHorizontal(View child, int left, int dx) {
          return left;
      }
}
```

    * tryCaptureView：如果返回true表示捕获相关View，你可以根据第一个参数child决定捕获哪个View。
    * clampViewPositionVertical：计算child垂直方向的位置，top表示y轴坐标（相对于ViewGroup），默认返回0（如果不复写该方法）。这里，你可以控制垂直方向可移动的范围。
    * clampViewPositionHorizontal：与clampViewPositionVertical类似，只不过是控制水平方向的位置。

      > 比如效果图中，**“拖拽2”**明显超过屏幕范围了，你可以这样控制：

      ```
      @Override
      public int clampViewPositionHorizontal(View child, int left, int dx) {
        if (left > getWidth() - child.getMeasuredWidth()) // 右侧边界
        {
            left = getWidth() - child.getMeasuredWidth();
        }
        else if (left < 0) // 左侧边界
        {
            left = 0;
        }
        return left;
      }
      ```
3. 处理ViewGroup触摸事件
```
   @Override
   public boolean onInterceptTouchEvent(MotionEvent ev) {
       return dragHelper.shouldInterceptTouchEvent(ev);
   }

   @Override
   public boolean onTouchEvent(MotionEvent event) {
       dragHelper.processTouchEvent(event);
       return true;
   }
```

    ``onInterceptTouchEvent``直接交给``dragHelper.shouldInterceptTouchEvent``去处理，``onTouchEvent``通过``dragHelper.processTouchEvent``来处理。
    >如果你希望拖拽的子View是不可点击的，可以不重写onInterceptTouchEvent方法，后面我们会介绍为什么。

#### (二) 布局文件

    <?xml version="1.0" encoding="utf-8"?>
    <android.drag.viewdraghelperdemo.VDHLinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
        android:id="@+id/activity_main"
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        android:gravity="center"
        android:orientation="vertical">
        <TextView
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:padding="20dp"
            android:background="@color/colorPrimaryDark"
            android:textColor="@android:color/white"
            android:text="拖拽1"/>
        <TextView
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:padding="20dp"
            android:layout_marginTop="10dp"
            android:background="@color/colorPrimaryDark"
            android:textColor="@android:color/white"
            android:text="拖拽2"/>
    </android.drag.viewdraghelperdemo.VDHLinearLayout>

布局很简单，自定义的ViewGroup包含两个TextView。

## 三、更多用法

ViewDragHelper不仅仅能够让子View跟随我们的手指移动，还能实现以下功能：
* 边界触摸检测
* Drag释放回调
* 移动到某个指定位置

我么改造下上面的例子，效果图如下：

![demo2.gif](http://upload-images.jianshu.io/upload_images/1787010-09fcedbea00e4134.gif?imageMogr2/auto-orient/strip)
第一个View，可以随意被拖动位置
第二个View，只能从ViewGroup左侧拖动
第三个View，拖动释放之后会回到原始位置

修改后的ViewGroup代码如下：

    public class VDHLinearLayout extends LinearLayout {
      ViewDragHelper dragHelper;

      public VDHLinearLayout(Context context, AttributeSet attrs) {
          super(context, attrs);
          dragHelper = ViewDragHelper.create(this, 1.0f, new ViewDragHelper.Callback() {
              @Override
              public boolean tryCaptureView(View child, int pointerId) {
                  return child == dragView || child == autoBackView;
              }

              @Override
              public int clampViewPositionVertical(View child, int top, int dy) {
                  return top;
              }

              @Override
              public int clampViewPositionHorizontal(View child, int left, int dx) {
                  return left;
              }

              // 当前被捕获的View释放之后回调
              @Override
              public void onViewReleased(View releasedChild, float xvel, float yvel) {
                  if (releasedChild == autoBackView)
                  {
                      dragHelper.settleCapturedViewAt(autoBackViewOriginLeft, autoBackViewOriginTop);
                      invalidate();
                  }
              }

              @Override
              public void onEdgeDragStarted(int edgeFlags, int pointerId) {
                  dragHelper.captureChildView(edgeDragView, pointerId);
              }
          });
          // 设置左边缘可以被Drag
          dragHelper.setEdgeTrackingEnabled(ViewDragHelper.EDGE_LEFT);
      }

      @Override
      public boolean onInterceptTouchEvent(MotionEvent ev) {
          return dragHelper.shouldInterceptTouchEvent(ev);
      }

      @Override
      public boolean onTouchEvent(MotionEvent event) {
          dragHelper.processTouchEvent(event);
          return true;
      }

      @Override
      public void computeScroll() {
          if (dragHelper.continueSettling(true))
          {
              invalidate();
          }
      }

      View dragView;
      View edgeDragView;
      View autoBackView;
      @Override
      protected void onFinishInflate() {
          super.onFinishInflate();
          dragView = findViewById(R.id.dragView);
          edgeDragView = findViewById(R.id.edgeDragView);
          autoBackView = findViewById(R.id.autoBackView);
      }

      int autoBackViewOriginLeft;
      int autoBackViewOriginTop;
      @Override
      protected void onLayout(boolean changed, int l, int t, int r, int b) {
          super.onLayout(changed, l, t, r, b);
          autoBackViewOriginLeft = autoBackView.getLeft();
          autoBackViewOriginTop = autoBackView.getTop();
      }
    }

1. ``tryCaptureView``方法，我们只捕获第一个和第三个View，分别是``dragView``和``autoBackView``。

2. 使用``dragHelper.setEdgeTrackingEnabled(ViewDragHelper.EDGE_LEFT)``设置ViewGroup左边缘可以被拖拽，同时在ViewDragHelper.Callback的``onEdgeDragStarted``方法中，使用``dragHelper.captureChildView``主动去捕获第二个View：``edgeDragView``。


    虽然在``tryCaptureView``方法中我们并未捕获``edgeDragView``，但``dragHelper.captureChildView``可以绕过该方法，详见官方解释：
    >Capture a specific child view for dragging within the parent. The callback will be notified but {@link Callback#tryCaptureView(android.view.View, int)} will not be asked permission to capture this view.

3. ``onViewReleased``方法会在被捕获的子View释放之后调用，我们判断释放的View：``releasedChild``是``autoBackView``，使用``dragHelper.settleCapturedViewAt``方法设置``autoBackView``的位置为它的初始位置。

    注意，此方法内部是通过**Scroller**实现的，所以我们需要使用``invalidate``来刷新，同时需要重写``computeScroll``方法：

       @Override
       public void computeScroll() {
          if (dragHelper.continueSettling(true))
          {
             invalidate();
          }
       }

    ``dragHelper.continueSettling``方法是用来判断当前被捕获的子View是否还需要继续移动，类似**Scroller**的``computeScrollOffset``方法一样，我们需要在返回true的时候使用``invalidate``刷新。

----
至此，我么已经介绍了ViewDragHelper以及ViewDragHelper.Callback的多数用法。

还记得前面我们留下的一个问题吗？
>“如果你希望拖拽的子View是不可点击的，可以不重写onInterceptTouchEvent方法，后面我们会介绍为什么。”

我们尝试将TextView设置成``clickable=true``，你会发现原本可以被拖拽的View都不动了。我们思考下，这是为什么呢？

原因在于：
> 由于子View是可被点击的，那么会触发ViewGroup的``onInterceptTouchEvent``方法。默认情况下，事件会被子View消耗掉，这显然是有问题的，因为这样ViewGroup的``onTouch``方法就不会被调用，而``onTouch``方法中正是我们的关键方法：``dragHelper.processTouchEvent``。

既然我们找到原因了，有人说：你不能在``onInterceptTouchEvent``直接返回true吗？为啥还要用``dragHelper.shouldInterceptTouchEvent(ev) ``的返回值啊？？？

确实，如果你直接返回true，会发现一切都能正常工作了。

这里我们需要解释下：
>打个比方，如果你的ViewGroup中有另外一个Button（或者任何可点击的View），但是它不在ViewDragHelper的处理范围内，你可能需要监听它的``onClick``事件，如果直接返回true，你会发现``onClick``事件不会被触发了。

纳尼，为啥呢？因为ViewGroup拦截了它的事件了啊。。。好吧，我们还是老实这样写吧：

    @Override
    public boolean onInterceptTouchEvent(MotionEvent ev) {
        return dragHelper.shouldInterceptTouchEvent(ev);
    }

你迫不及待的运行修改之后的代码。咦？为啥还是不能拖拽。。。
此时，遇到这种情况，我一般是查看下``dragHelper.shouldInterceptTouchEvent``的源码（此处省略了部分不相关的代码）：

    public boolean shouldInterceptTouchEvent(MotionEvent ev) {
        final int action = MotionEventCompat.getActionMasked(ev);
        switch (action) {
            case MotionEvent.ACTION_MOVE: {          
                final int pointerCount = ev.getPointerCount();
                for (int i = 0; i < pointerCount; i++) {           
                    final int horizontalDragRange = mCallback.getViewHorizontalDragRange(
                                toCapture);
                    final int verticalDragRange = mCallback.getViewVerticalDragRange(toCapture);
                    // 如果getViewHorizontalDragRange和getViewVerticalDragRange的返回值都为0，则break
                    if (horizontalDragRange == 0 && verticalDragRange == 0) {
                        break;
                    }

                    // tryCaptureViewForDrag方法中会设置mDragState=STATE_DRAGGING
                    if (pastSlop && tryCaptureViewForDrag(toCapture, pointerId)) {
                        break;
                    }
                }
                break;
            }
        }
        return mDragState == STATE_DRAGGING;
    }

``shouldInterceptTouchEvent``返回true的条件是``mDragState == STATE_DRAGGING``，然而``mDragState``是在``tryCaptureViewForDrag``方法中被设置为``STATE_DRAGGING``的。

所以，如果``horizontalDragRange == 0 && verticalDragRange == 0``这个条件一直为true的话，``tryCaptureViewForDrag``方法就得不到调用了。

而``horizontalDragRange``和``verticalDragRange``分别是Callback的``getViewHorizontalDragRange``和``getViewVerticalDragRange``方法返回的值，这两个方法默认情况下都返回0。

*  getViewHorizontalDragRange，返回子View水平方向可以被拖拽的范围
*  getViewVerticalDragRange，返回子View垂直方向可以被拖拽的范围

我们尝试重写这两个方法：

    @Override
    public int getViewVerticalDragRange(View child) {
       return getMeasuredHeight() - child.getMeasuredHeight();
    }

    @Override
    public int getViewHorizontalDragRange(View child) {
       return getMeasuredWidth() - child.getMeasuredWidth();
    }

再次运行下，你会发现TextView设置``clickable=true``之后也可以被拖拽了。

----

至此，ViewDragHelper的基本使用方式我们已经介绍完了。详细的代码可以查看文章最后的源码，另外，源码中还实现了一个比较常用的效果：

![demo3.gif](http://upload-images.jianshu.io/upload_images/1787010-6529f34b23b681f3.gif?imageMogr2/auto-orient/strip)

[本文源码](https://github.com/hiphonezhu/Android-Demos/tree/master/ViewDragHelperDemo)
