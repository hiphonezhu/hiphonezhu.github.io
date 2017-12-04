---
layout:     post
title:      "实现可在 RecyclerView 中展开和收缩的 TextView"
subtitle:   ""
date:       2017-12-04 09:41:00
author:     "zhuhf"
header-img: ""
header-mask: 0.3
catalog:    true
tags:
    - ExpandedTextView
---



##前言&常用做法
效果类似微信朋友圈 - 查看全文的**“展开”**和**“收缩”**效果，这里就不贴图了，相信大家都不会陌生。

一般情况下，第一个想到的做法是通过 `TextView#setMaxLines(int maxLines)` 来控制 TextView 显示的行数。

了解 View 模型的同学都知道，在 View 没有“呈现”之前，我们是无法获取到当前 TextView 显示的文字的具体行数。所以，有了下面的一种方法：

    ...
    boolean expandable = false;
    boolean expanded = false;
    final TextView tv = findViewById(R.id.tv);
    tv.getViewTreeObserver().addOnGlobalLayoutListener(new ViewTreeObserver.OnGlobalLayoutListener() {
        @Override
        public void onGlobalLayout() {
            tv.getViewTreeObserver().removeOnGlobalLayoutListener(this);
            int lineCount = tv.getLineCount();
            if (lineCount > 3) {
                tv.setMaxLines(3);
                expandable = true;
            }
        }
    });
    tv.setOnClickListener(new View.OnClickListener() {
        @Override
        public void onClick(View v) {
            if (expandable) {
                if (!expanded) {
                    tv.setMaxLines(Integer.MAX_VALUE);
                } else {
                    tv.setMaxLines(3);
                }
                expanded = !expanded;
            }
        }
    });
    ...

这段代码如果不在 RecyclerView 或者 ListView 中使用是没有太大问题的。我们都知道，RV 或者 LV 是会复用 View 的，所以这段代码有两个问题：

1. 如果某个 View 执行了 ViewTreeObserver.onGlobalLayoutListener 回调，在它再次被复用的时候，是不会再次触发这个回调了。
2. 在滚动的情况下，`TextView#getLineCount()` 这个方法返回的行数，可能不是你当前看到的实际行数。


基于以上两个原因，在 RV 或者 LV 中，通过 `TextView#setMaxLines(int maxLines)` 来控制的方法就行不通了。

##思考

换个思路，如果我们能**“测量”**出一段文字显示的行数，和每一行文字显示需要的**“高度”**，那么就可以通过改变 **TextView 的高度**，来让用户看到**“展开”**和**“收缩”**文字的效果了。

那么这个工具有吗？很庆幸，Android Api 给我们提供了一个用来测量文字大小、宽度等数据的工具，它就是：**StaticLayout**。

##StaticLayout

引用 [hencoder](http://hencoder.com/) 一篇文章中对 StaticLayout 的介绍：
>StaticLayout 的构造方法是 StaticLayout(CharSequence source, TextPaint paint, int width, Layout.Alignment align, float spacingmult, float spacingadd, boolean includepad)，其中参数里：
width 是文字区域的宽度，文字到达这个宽度后就会自动换行； 
align 是文字的对齐方向； 
spacingmult 是行间距的倍数，通常情况下填 1 就好； 
spacingadd 是行间距的额外增加值，通常情况下填 0 就好； 
includeadd 是指是否在文字上下添加额外的空间，来避免某些过高的字符的绘制出现越界。

通过 StaticLayout，这里实现了一个 ExpandTextView，代码不多，注释非常全：

    public class ExpandTextView extends android.support.v7.widget.AppCompatTextView {
      /**
       * true：展开，false：收起
       */
      boolean mExpanded;
      /**
       * 状态回调
       */
      Callback mCallback;
      /**
       * 源文字内容
       */
      String mText;
      /**
       * 最多展示的行数
       */
      final int maxLineCount = 3;
      /**
       * 省略文字
       */
      final String ellipsizeText = "...";
    
      public ExpandTextView(Context context, @Nullable AttributeSet attrs) {
          super(context, attrs);
      }
    
      @Override
      protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
          super.onMeasure(widthMeasureSpec, heightMeasureSpec);
          // 文字计算辅助工具
          StaticLayout sl = new StaticLayout(mText, getPaint(), getMeasuredWidth() - getPaddingLeft() - getPaddingRight()
                , Layout.Alignment.ALIGN_CENTER, 1, 0, true);
          // 总计行数
          int lineCount = sl.getLineCount();
          if (lineCount > maxLineCount) {
              if (mExpanded) {
                  setText(mText);
                  mCallback.onExpand();
              } else {
                  lineCount = maxLineCount;
                
                  // 省略文字的宽度
                  float dotWidth = getPaint().measureText(ellipsizeText);
                
                  // 找出第 showLineCount 行的文字
                  int start = sl.getLineStart(lineCount - 1);
                  int end = sl.getLineEnd(lineCount - 1);
                  String lineText = mText.substring(start, end);
                
                  // 将第 showLineCount 行最后的文字替换为 ellipsizeText
                  int endIndex = 0;
                  for (int i = lineText.length() - 1; i >= 0; i--) {
                      String str = lineText.substring(i, lineText.length());
                      // 找出文字宽度大于 ellipsizeText 的字符
                      if (getPaint().measureText(str) >= dotWidth) {
                          endIndex = i;
                          break;
                      }
                  }
                
                  // 新的第 showLineCount 的文字
                   String newEndLineText = lineText.substring(0, endIndex) + ellipsizeText;
                  // 最终显示的文字
                  setText(mText.substring(0, start) + newEndLineText);
                
                  mCallback.onCollapse();
              }
          } else {
              setText(mText);
              mCallback.onLoss();
          }
        
          // 重新计算高度
          int lineHeight = 0;
          for (int i = 0; i < lineCount; i++) {
              Rect lineBound = new Rect();
              sl.getLineBounds(i, lineBound);
              lineHeight += lineBound.height();
          }
          lineHeight += getPaddingTop() + getPaddingBottom();
          setMeasuredDimension(getMeasuredWidth(), lineHeight);
      }
    
      /**
       * 设置要显示的文字以及状态
       * @param text
       * @param expanded true：展开，false：收起
       * @param callback
       */
      public void setText(String text, boolean expanded, Callback callback) {
          mText = text;
          mExpanded = expanded;
          mCallback = callback;
        
          // 设置要显示的文字，这一行必须要，否则 onMeasure 宽度测量不正确
          setText(text);
      }
    
      /**
       * 展开收起状态变化
       * @param expanded
       */
      public void setChanged(boolean expanded) {
          mExpanded = expanded;
          requestLayout();
      }
    
      public interface Callback {
          /**
           * 展开状态
           */
          void onExpand();
        
          /**
           * 收起状态
           */
          void onCollapse();
        
          /**
           * 行数小于最小行数，不满足展开或者收起条件
           */
          void onLoss();
      }
    }

在 RecyclerView 中使用：

    ...
    public void onBindViewHolder(VH holder, int position) {
        ....
        
        /**
         * item.getText()：    显示的文本
         * item.isExpanded()： 保存的是当前行是否是展开状态
         */
        tvContent.setText(item.getText(), item.isExpanded(), new ExpandTextView.Callback() {
                @Override
                public void onExpand() {
                    // 展开状态，比如：显示“收起”按钮
                }

                @Override
                public void onCollapse() {
                    // 收缩状态，比如：显示“全文”按钮
                }

                @Override
                public void onLoss() {
                    // 不满足展开的条件，比如：隐藏“全文”按钮
                }
            });
        }
        tvContent.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                // 保存当前行的状态
                item.setExpanded(!item.setExpanded());
                // 切换状态
                tvContent.setChanged(item.isExpanded());
            }
        });
    }
    ...




