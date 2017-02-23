---
layout:     post
title:      "RecyclerView Divider完美解决方案"
subtitle:   ""
date:       2016-10-09 12:00:00
author:     "zhuhf"
header-img: ""
header-mask: 0.3
catalog:    true
tags:
    - Android
    - RecyclerView
    - Divider
---

关于RecyclerView的使用，不是本文介绍的重点，还不清楚的同学可以参考这篇文章： [Android RecyclerView 使用完全解析 体验艺术般的控件](http://blog.csdn.net/lmj623565791/article/details/45059587)。

RecyclerView替代ListView势在必行，唯一比较遗憾的是官方没有内置几个好用的ItemDecoration，这使得很多人觉得使用起来比较麻烦。

有幸站在巨人的肩膀上，github上有大神实现了一个：[RecyclerView-FlexibleDivider](https://github.com/yqritc/RecyclerView-FlexibleDivider)，经过一番使用，我发现对于GridLayoutManager(不支持SpanSizeLookup)和StaggeredGridLayoutManager的支持并不好。

RecyclerView-FlexibleDivider的实现思路和网上多数方案是一致的：
* 从左到右绘制横向divider
* 从上至下绘制垂直divider

这种实现的方式弊端很明显，就是遇到GridLayoutManager(使用了SpanSizeLookup)和StaggeredGridLayoutManager的时候，就没法正常工作了，因为它们不是“规则”的占据一个“单元”。所以，本文的实现思路是这样的
* 横向divider绘制在每个item的下方
* 垂直divider绘制在每个item的右侧

----
在开始之前，我们有必要了解下**ItemDecoration**的几个方法
* onDraw(Canvas c, ...)：在绘制item views**之前**绘制decorations，一般我们都用这个方法
* onDrawOver(Canvas c, ...)：与上面的方法相反，在绘制item views**之后**绘制decorations
* getItemOffsets(Rect outRect, ...)：为outRect设置left,top和right,bottom，简单的理解就是设置item view的margin（marginLeft、marginTop、marginRight、marginBottom）

绘制decorations的方法就是使用onDraw方法中的参数Canvas，比如
* 绘制一条线：canvas.drawLine
* 绘制图片：drawable.draw(canvas)
* 等等...

不管哪种绘制都需要的一个参数，那就是绘制的范围：Rect；另外针对横向和垂直divider，getItemOffsets方法中的outRect的设置，应该也是不一样的。所以抽象出一个父类FlexibleDividerDecoration，继承自RecyclerView.ItemDecoration。

**FlexibleDividerDecoration**

    public abstract class FlexibleDividerDecoration extends RecyclerView.ItemDecoration {

        protected abstract Rect getDividerBound(int position, RecyclerView parent, View child);

        protected abstract void setItemOffsets(Rect outRect, int position, RecyclerView parent);

        @Override
        public void onDraw(Canvas c, RecyclerView parent, RecyclerView.State state) {
            int validChildCount = parent.getChildCount();

            for (int i = 0; i < validChildCount; i++) {
                View child = parent.getChildAt(i);
                int childPosition = parent.getChildAdapterPosition(child);

                // 当前item是否需要divider（通常最后一个不需要）
                if (!hasDivider(parent, childPosition)) {
                    continue;
                }

                // 绘制divider的区域，由子类具体实现
                Rect bounds = getDividerBound(childPosition, parent, child);

                switch (mDividerType) {
                    case DRAWABLE: // 图片类型的divider
                        Drawable drawable = mDrawableProvider.drawableProvider(childPosition, parent);
                        drawable.setBounds(bounds);
                        drawable.draw(c);
                        break;
                }
            }
        }

        @Override
        public void getItemOffsets(Rect rect, View v, RecyclerView parent, RecyclerView.State state) {
            int position = parent.getChildAdapterPosition(v);
            if (!hasDivider(parent, position)) {
                return;
            }
            setItemOffsets(rect, position, parent);
        }

        /**
         * Whether child has divider
         *
         * @param parent
         * @param childPosition
         * @return true if child has divider
         */
        public boolean hasDivider(RecyclerView parent, int childPosition) {
            if (this instanceof VerticalDividerItemDecoration) { // 垂直方向是否有divider
                return hasVerticalDivider(parent, childPosition);
            } else if (this instanceof HorizontalDividerItemDecoration) { // 水平方向是否有divider
                return hasHorizontalDivider(parent, childPosition);
            }
            return false;
        }
    }

上面的代码看注释应该比较好理解（注意：实际代码会比文章中的复杂些许）。具体实现的子类有两个，分别是**HorizontalDividerItemDecoration**和**VerticalDividerItemDecoration**。
我们看下**HorizontalDividerItemDecoration**：

    public class HorizontalDividerItemDecoration extends FlexibleDividerDecoration {
      protected HorizontalDividerItemDecoration(Builder builder) {
          super(builder);
      }

      @Override
      protected Rect getDividerBound(int position, RecyclerView parent, View child) {
          Rect bounds = new Rect(0, 0, 0, 0);
          RecyclerView.LayoutParams params = (RecyclerView.LayoutParams) child.getLayoutParams();
          bounds.left = child.getLeft();
          bounds.right = child.getRight();

          // divider大小
          int dividerSize = getDividerSize(position, parent);

          bounds.top = child.getBottom() + params.bottomMargin;
          bounds.bottom = bounds.top + dividerSize;

          return bounds;
      }

      @Override
      protected void setItemOffsets(Rect outRect, int position, RecyclerView parent) {
          outRect.set(0, 0, 0, getDividerSize(position, parent));
      }
    }

这里只贴出了关键代码，因为实际情况，我们可能还需要考虑更多的问题，比如：
1. 水平divider的marginLeft和marginRight，还需要判断item是否是最边上的，因为中间的divider是不需要margin的
2. 垂直divider的marginTop和marginBottom，还需要判断item是否是最边上的，因为中间的divider是不需要margin的
3. 水平和垂直交叉位置的空白问题
4. getReverseLayout为true和false时，不同的处理

为了支持不同的LayoutManager，以上的判断策略还不尽相同，因为篇幅限制，具体的代码请查看文章最后的源码，最后我们来看下最终实现的效果吧~


![divider.gif](http://upload-images.jianshu.io/upload_images/1787010-803f6ecd4ef49d29.gif?imageMogr2/auto-orient/strip)
![divider2.gif](http://upload-images.jianshu.io/upload_images/1787010-289153cef6779e28.gif?imageMogr2/auto-orient/strip)

本文源码地址：[https://github.com/hiphonezhu/RecyclerView-FlexibleDivider](https://github.com/hiphonezhu/RecyclerView-FlexibleDivider)（由于改动较大，fork过来的代码没有pull给作者，如需使用请下载本文的源码，而不是通过Gradle引入）
