---
layout:     post
title:      "ViewDragHelper实战：APP内“悬浮球”"
subtitle:   ""
date:       2016-12-29 12:00:00
author:     "zhuhf"
header-img: ""
header-mask: 0.3
catalog:    true
tags:
    - Android
    - ViewDragHelper
---



>本文的理论知识是基于：[Android自定义ViewGroup神器-ViewDragHelper](http://www.jianshu.com/p/111a7bc76a0e)，如果你对ViewDragHelper的使用不熟悉，请先阅读这篇文章。

## 前言

**“悬浮球”**最初是iPhone手机上的一个虚拟按键，它会悬浮于所有APP之上，手指随意拖动，松开后会自动贴边显示。现在满大街都是iPhone手机，相信大家都用过或者看过这个效果，这里就不上图了~

当前，很多Android手机也都有了这个功能，并且很多第三方APP也实现了此功能，比如某垃圾清理软件。可能大家立马就会想到，这个不就是使用**WindowManager**实现的**悬浮窗**，然后在`onTouch`事件里面根据手指的移动来改变位置吗？

确实，如果你的**“悬浮球”**是在桌面，实现方案的确如此（也只能如此）。但是，本文需要实现的是**应用内“悬浮球”**，即：退出应用不需要显示，并且我们不希望使用`android.permission.SYSTEM_ALERT_WINDOW`这个权限，要知道Android M 6.0此权限属于危险权限，需要动态申请授权后才能使用，且使用**WindowManager**实现**悬浮窗** **“必须”**(`此处有引号~`)使用此权限。
>上文的**“必须”**加引号的原因：WindowManager特定情况是可以无权限显示悬浮框的，但这不是本文讨论的范畴，感兴趣的同学可以阅读这篇文章：[Android无需权限显示悬浮窗, 兼谈逆向分析app](http://www.jianshu.com/p/167fd5f47d5c)。总结来说，无权限的坑还是很多~

## 效果图

下面的效果图，是一款线上App新版即将发布的功能。

![demo](http://upload-images.jianshu.io/upload_images/1787010-2afbe65c0e58a5dd.gif?imageMogr2/auto-orient/strip)

可以看到，**“悬浮球”**在App内所有界面都**“独立”**显示，每个界面都支持拖动并**自动贴边**，且所有界面的**“悬浮球”**位置都保持一致。

## 实现步骤

我们将“悬浮球”实现步骤分解为以下几步：
1. 屏幕范围内任意位置拖动
2. 释放后自动贴边
3. 解决UI刷新，恢复到原始位置的问题
4. 提供统一入口给所有Activity
4. 所有Activity保持“实时”位置一致

下面，我们就每个步骤进行分别讲解：

### 一、屏幕范围内任意位置拖动

我们在[Android自定义ViewGroup神器-ViewDragHelper](http://www.jianshu.com/p/111a7bc76a0e)一文中已经做过详细的讲解，通过重写`ViewDragHelper.Callback`的以下方法实现：
1. `tryCaptureView`判断`View`是否是我们要拖动的

       @Override
       public boolean tryCaptureView(View child, int pointerId) {    
          return child == floatingBtn;
       }

2. `clampViewPositionHorizontal`和`clampViewPositionVertical`，返回水平和垂直方向可移动的范围

       @Override
       public int clampViewPositionVertical(View child, int top, int dy) {
          if (top > getHeight() - child.getMeasuredHeight()) {
              top = getHeight() - child.getMeasuredHeight();
          } else if (top < 0) {
              top = 0;
          }
          return top;
       }

       @Override
       public int clampViewPositionHorizontal(View child, int left, int dx) {
          if (left > getWidth() - child.getMeasuredWidth()) {
              left = getWidth() - child.getMeasuredWidth();
          } else if (left < 0) {
              left = 0;
          }
          return left;
       }

3. 如果可拖动的`View`是可点击的(Button or 其他)，`getViewHorizontalDragRange`和`getViewVerticalDragRange`需要返回水平和垂直可移动的范围

       @Override
       public int getViewVerticalDragRange(View child) {
          return getMeasuredHeight() - child.getMeasuredHeight();
       }

       @Override
       public int getViewHorizontalDragRange(View child) {
          return getMeasuredWidth() - child.getMeasuredWidth();
       }

### 二、释放后自动贴边

需要监听手指“释放”被拖拽`View`的事件，可以重写`ViewDragHelper.Callback`的`onViewReleased`方法。

我们观察下，自动贴边是根据当前`View`所在的区域，决定贴在哪一个方向。这个是和产品的需求有关，以下代码仅供参考：

    @Override
    public void onViewReleased(View releasedChild, float xvel, float yvel) {
       if (releasedChild == floatingBtn) {
           float x = floatingBtn.getX();
           float y = floatingBtn.getY();
           if (x < (getMeasuredWidth() / 2f - releasedChild.getMeasuredWidth() / 2f)) { // 0-x/2
               if (x < releasedChild.getMeasuredWidth() / 3f) {
                   x = 0;
               } else if (y < (releasedChild.getMeasuredHeight() * 3)) { // 0-y/3
                   y = 0;
               } else if (y > (getMeasuredHeight() - releasedChild.getMeasuredHeight() * 3)) { // 0-(y-y/3)
                   y = getMeasuredHeight() - releasedChild.getMeasuredHeight();
               } else {
                   x = 0;
               }
           } else { // x/2-x
               if (x > getMeasuredWidth() - releasedChild.getMeasuredWidth() / 3f - releasedChild.getMeasuredWidth()) {
                   x = getMeasuredWidth() - releasedChild.getMeasuredWidth();
               } else if (y < (releasedChild.getMeasuredHeight() * 3)) { // 0-y/3
                   y = 0;
               } else if (y > (getMeasuredHeight() - releasedChild.getMeasuredHeight() * 3)) { // 0-(y-y/3)
                   y = getMeasuredHeight() - releasedChild.getMeasuredHeight();
               } else {
                   x = getMeasuredWidth() - releasedChild.getMeasuredWidth();
               }
           }
           // 移动到x,y
           dragHelper.smoothSlideViewTo(releasedChild, (int) x, (int) y);
           invalidate();
       }
    }

根据你的产品的需求(上面模仿了iPhone的悬浮球)，计算好最终的`x`和`y`，然后使用`ViewDragHelper`的`smoothSlideViewTo`方法，将`View`移动到指定位置。

### 三、解决UI刷新，恢复到原始位置的问题

这个问题在做Demo的时候并没有遇到，但当集成到项目中的时候，就出现了这个问题，如下图：

![move](http://upload-images.jianshu.io/upload_images/1787010-8dc04ac31a9f7795.gif?imageMogr2/auto-orient/strip)

首页点击某个**Item**展开(`ExpandableListView`)或者切换底部**Tab**(`Fragment`显示与隐藏)，“悬浮球”会恢复到原始的位置，我们来分析下为什么？

我们先来简单分析下`ViewDragHelper`的部分源码实现。

从`smoothSlideViewTo`这个方法切入，该方法内部的实现如下：

![smoothSlideViewTo](http://upload-images.jianshu.io/upload_images/1787010-57031cb71fc992b0.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
545行，`forceSettleCapturedViewAt`方法

![forceSettleCapturedViewAt](http://upload-images.jianshu.io/upload_images/1787010-87f8078a6b35cf7d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
600行，使用`Scroller`来实现`View`的位置滑动，熟悉`Scroller`的同学应该都知道，需要在自定义`ViewGroup`的`computeScroll`方法做处理

    @Override
    public void computeScroll() {    
       if (dragHelper.continueSettling(true)) {        
           invalidate();    
       }
    }

关键代码在`if`语句的`continueSettling`方法：

![continueSettling](http://upload-images.jianshu.io/upload_images/1787010-e5e3b2ce1393906c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
733、736行，使用`offsetLeftAndRight`和`offsetTopAndBottom`来设置`View`的位置，这个方法与`View`的`setX`和`setY`方法有异曲同工之效。

通过这种方式，的确是真实改变了`View`的`x`和`y`坐标。但是，当UI刷新后，我们自定义的`ViewGroup`的`onMeasure`、`onLayout`等方法会被调用，我们都知道`onLayout`方法直接决定了子`View`的位置。

但是`onLayout`方法是不会根据子`View`的`x`和`y`来排列它的位置，而是根据`LayoutParams`来决定，关键源码如下：

    @Override
    protected void onLayout(boolean changed, int left, int top, int right, int bottom) {
        layoutChildren(left, top, right, bottom, false /* no force left gravity */);
    }

    void layoutChildren(int left, int top, int right, int bottom, boolean forceLeftGravity) {
        final int count = getChildCount();

        final int parentLeft = getPaddingLeftWithForeground();
        final int parentRight = right - left - getPaddingRightWithForeground();

        final int parentTop = getPaddingTopWithForeground();
        final int parentBottom = bottom - top - getPaddingBottomWithForeground();

        for (int i = 0; i < count; i++) {
            final View child = getChildAt(i);
            if (child.getVisibility() != GONE) {
                final LayoutParams lp = (LayoutParams) child.getLayoutParams();
                ...
                final int width = child.getMeasuredWidth();
                final int height = child.getMeasuredHeight();
                ...
                int childLeft;
                int childTop;
                ...
                childLeft = parentLeft + lp.leftMargin;
                ...
                childTop = parentTop + lp.topMargin;
                ...
                child.layout(childLeft, childTop, childLeft + width, childTop + height);
            }
        }
    }

所以，我们的解决方案很简单，就是重写`ViewGroup`的`onLayout`方法，设置被拖拽`View`的位置：

    @Override
    protected void onLayout(boolean changed, int left, int top, int right, int bottom) {
        super.onLayout(changed, left, top, right, bottom);
        restorePosition();
    }

    // 记录最后的位置
    float mLastX = -1;
    float mLastY = -1;
    public void restorePosition() {
        if (mLastX == -1 && mLastY == -1) { // 初始位置
            mLastX = getMeasuredWidth() - floatingBtn.getMeasuredWidth();
            mLastY = getMeasuredHeight() * 2 / 3;
        }
        floatingBtn.layout((int)mLastX, (int)mLastY,
                    (int)mLastX + floatingBtn.getMeasuredWidth(), (int)mLastY + floatingBtn.getMeasuredHeight());
    }

`mLastX`和`mLastY`是用来记录“悬浮球”最后的位置，需要在`ViewDragHelper.Callback`的`onViewPositionChanged`方法中处理

    @Override
    public void onViewPositionChanged(View changedView, int left, int top, int dx, int dy) {
        super.onViewPositionChanged(changedView, left, top, dx, dy);
        mLastX = changedView.getX();
        mLastY = changedView.getY();
    }

只要“悬浮球”的位置发生变化，就会回调这个方法。

### 四、提供统一入口给所有Activity

基本所有项目都会有一个`BaseActivity`(如果没有，只能呵呵了~)，重写`setContentView`方法，统一接入我们的“悬浮球”：

    public class BaseActivity extends AppCompatActivity{
      ...
      @Override
      public void setContentView(int layoutResID)
      {
          super.setContentView(new FloatingDragger(this, layoutResID).getView());
      }
      ...
    }

这样，所有`Activity`的代码可以保持不变，只要继承自`BaseActivity`，就会拥有“悬浮球”功能，所有业务全部封装在`FloatingDragger`这个类中。

### 五、所有Activity保持“实时”位置一致

`FloatingDragger`这个类，实际上是在`Activity`原有的布局`layoutResID`之上添加了一个`View`，也就是我们的“悬浮球”，所以每个`Activity`都拥有一个不同的`FloatingDragger`对象。

我们可以实时保存“悬浮球”的位置，这样每次重新打开APP，“悬浮球”总会在上次的位置。如果进入下一个`Activity2`，它的位置也总是和上一个`Activity1`一致。这个实现比较简单，将上文的`mLastX`和`mLastY`存储到配置文件

    @Override
    public void onViewPositionChanged(View changedView, int left, int top, int dx, int dy) {
        super.onViewPositionChanged(changedView, left, top, dx, dy);
        int x = changedView.getX();
        int y = changedView.getY();
        spdbHelper.putFloat(KEY_FLOATING_X, x);
        spdbHelper.putFloat(KEY_FLOATING_Y, y);
    }

然后位置从配置文件读取

    public void restorePosition() {
        float x = spdbHelper.getFloat(KEY_FLOATING_X, -1);
        float y = spdbHelper.getFloat(KEY_FLOATING_Y, -1);
        if (x == -1 && y == -1) { // 初始位置
            x = getMeasuredWidth() - floatingBtn.getMeasuredWidth();
            y = getMeasuredHeight() * 2 / 3;
        }
        floatingBtn.layout((int)x, (int)y,
                    (int)x + floatingBtn.getMeasuredWidth(), (int)y + floatingBtn.getMeasuredHeight());
    }

但是，如果你在`Activity2`改变了位置，怎么让`Activity1`“悬浮球”的位置也刷新呢？

这里有两种方案：
1. `BaseActivity`的`onResume`调用`FloatingDragger`对象的某个方法
2. `FloatingDragger`内部实现

方法1比较简单，这里不做演示。另外，显然方案2也更好一点，因为和`Activity`的耦合度更低，比较符合“封装”的思想。

我们思考下，`FloatingDragger`对所有“悬浮球”位置的改变感兴趣，似乎比较符合设计模式中的**观察者模式**，`FloatingDragger`是**观察者**，**被观察者**是一个单例`PositionObservable`，“悬浮球”位置发生变化后通过`PositionObservable`通知所有的`FloatingDragger`对象。

**被观察者：**

    public class PositionObservable extends Observable {
        public static PositionObservable sInstance;
        public static PositionObservable getInstance() {
            if (sInstance == null) {
                sInstance = new PositionObservable();
            }
            return sInstance;
        }

        /**
         * 通知观察者FloatingDragger
         */
        public void update() {
            setChanged();
            notifyObservers();
        }
    }

**观察者：**

    public class FloatingDragger implements Observer {
        PositionObservable observable = PositionObservable.getInstance();
        FloatingDraggedView floatingDraggedView;

        public FloatingDragger(Context context, @LayoutRes int layoutResID) {
            // 用户布局
            View contentView = LayoutInflater.from(context).inflate(layoutResID, null);
            // 悬浮球按钮
            View floatingView = LayoutInflater.from(context).inflate(R.layout.layout_floating_dragged, null);

            // ViewDragHelper的ViewGroup容器
            floatingDraggedView = new FloatingDraggedView(context);
            floatingDraggedView.addView(contentView, new FrameLayout.LayoutParams(ViewGroup.LayoutParams.MATCH_PARENT, ViewGroup.LayoutParams.MATCH_PARENT));
            floatingDraggedView.addView(floatingView, new FrameLayout.LayoutParams(APKUtil.dip2px(context, 45), APKUtil.dip2px(context, 40)));

            // 添加观察者
            observable.addObserver(this);
        }
        ....
        @Override
        public void update(Observable o, Object arg) {
            if (floatingDraggedView != null) {
                // 更新位置
                floatingDraggedView.restorePosition();
            }
        }

        public class FloatingDraggedView extends FrameLayout {
            ...
            public FloatingDraggedView(Context context) {
                super(context);
                init();
            }

            void init() {
                dragHelper = ViewDragHelper.create(FloatingDraggedView.this, 1.0f, new ViewDragHelper.Callback() {
                    @Override
                    public void onViewDragStateChanged(int state) {
                        super.onViewDragStateChanged(state);
                        if (state == ViewDragHelper.STATE_SETTLING) { // 拖拽结束，通知观察者
                            observable.update();
                        }
                    }
                    ...
                }
            }
            ...
       }
       ...
    }

`ViewDragHelper.Callback`的`onViewDragStateChanged`方法，在`View`被拖动的时候会回调三次，分别对应三个状态
* STATE_IDLE：空闲
* STATE_DRAGGING：正在拖拽
* STATE_SETTLING：拖拽结束，放置View

至此，我们已经实现了功能比较完善的“悬浮球”，感谢大家耐心看到最后，本文的源码在这里：[FloatingOval](https://github.com/hiphonezhu/Android-Demos/tree/master/FloatingOval)

## 写在最后

2016年转眼就要过去了，回忆这一年，自己从一家外包公司，跳槽到一家创业公司。以前在外包公司职责是移动端负责人（数十人的移动团队），外包公司项目的周期非常短，压力非常大，移动端一年至少8-10个项目，自己也是全程参与Android端的代码开发，同时还要负责业务以及和后台的API对接工作，另外还要管理iOS团队（因为精力问题，这点做的不是太合格，需要检讨）。

另外，经常还要和销售一起出去面对客户谈需求，不得不说外包公司虽然累了点，但是做为过来人，我还是要告诉大家，**“公司是别人的，学到东西才是自己的”**。所以，刚毕业的小伙伴，或者正在找工作的同学，没有必要太“歧视”或看不上外包公司，毕竟学习技术还是要靠自己，再好的公司，你如果是一颗螺丝钉还不如在小公司多负责、多做点东西，这样才能在工作中成长，在学习中进步。

2017年希望自己在新公司能够取得更大的进步，也希望公司能够更快、更好、更强的发展。

**不忘初心，方得始终——献给正在人生道路上奋斗的我们！！！**
