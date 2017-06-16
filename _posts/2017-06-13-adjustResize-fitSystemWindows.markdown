---
layout:     post
title:      "谈谈“adjustResize”在沉浸式状态栏下的失效问题"
subtitle:   ""
date:       2017-06-16 16:10:00
author:     "zhuhf"
header-img: ""
header-mask: 0.3
catalog:    true
tags:
    - 沉浸式状态栏 
    - adjustResize
---



前言
--

关于“沉浸式”的介绍，请看另外一篇文章：[刨根问底-论Android“沉浸式”](http://www.jianshu.com/p/38c2239dd0d4)，文章中详细介绍了“沉浸式”的相关知识，最后给出了Android 4.4 及以上“**状态栏着色**”的适配方案。

这里简单介绍下，一共分为两步：

 1. 在主题中增加以下属性，这会使得“状态栏”透明。
 *values-v19/styles.xml：*
`<item name="android:windowTranslucentStatus">true</item>`

 2. 上一步会使得 Activity 的布局“陷入”到状态栏中，所以需要配合另外一个属性：fitSystemWindows

        <?xml version="1.0" encoding="utf-8"?>
        <LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
            android:layout_width="match_parent"
            android:layout_height="match_parent"
            android:orientation="vertical">
            
            <android.support.v7.widget.Toolbar
                android:id="@+id/toolbar"
                android:layout_width="match_parent"
                android:layout_height="wrap_content"
                android:background="@color/colorPrimary"
                android:fitsSystemWindows="true"/>

            <RelativeLayout
                android:layout_width="match_parent"
                android:layout_height="match_parent">

                <EditText
                    android:layout_width="match_parent"
                    android:layout_height="wrap_content"
                    android:layout_alignParentBottom="true" />
            </RelativeLayout>
        </LinearLayout>
        
> 注意：这里将 **fitSystemWindows** 设置到了需要加上 **paddingTop** 的 View（不太明白？那就先看下[刨根问底-论Android“沉浸式”](http://www.jianshu.com/p/38c2239dd0d4)），而不是 xml 布局的根元素！！！

这里解释下，为何不建议将 **fitSystemWindows** 设置到 xml 布局的根元素，基于几下几个可能出现的问题：

- 必须将背景色设置到 xml 布局的根元素，而不是在标题栏设置。
 为何这么说呢？如果不设置根元素的背景色，你会看到状态栏的颜色和根布局的背景一致，而不是和我们的标题栏一致（**终极原因是因为根布局被系统加上 paddingTop 了。。。**）。
- 如果背景色设置到了根元素，那么布局内容的颜色（图中的白色，或其他颜色）还需要再设置一遍（这也是必须的），不然整个布局都是蓝色的，这个肯定不行。
- 其他，还没想到~

效果图如下：
![此处输入图片的描述][1]
正题
--
点击输入框（Activity 的键盘模式设置为 adjustResize），你会发现 ToolBar 被拉伸了。
![此处输入图片的描述][2]
> 网上有些文章说到 ToolBar 文字被顶上去，但是高度没有变化。区别在于：他们设置了 ToolBar 的具体高度，而不是使用的 `wrap_content` 属性。一般做法都是通过反射拿到状态栏的高度，再加上一个标题栏的高度作为 ToolBar 的实际高度。个人觉得这并不是一种好的做法，毕竟反射获取状态栏高度不是一个“合法渠道”。

这里需要解释下，为何会出现“**拉伸**”的情况？

我们都知道 adjustResize 模式，会使用界面“压缩”来腾出底部空间，显示键盘。

如果你的 View 使用了 `android:fitsSystemWindows="true"`，并且它的高度是 `wrap_content` 的。那么，当键盘出现的时候，系统会设置一个 `padingBottom` 值到此 View，它的数值正好等于键盘的高度。

因为 ToolBar 设置了 `android:fitsSystemWindows="true"` 且高度是 `wrap_content`，所以导致 ToolBar 被系统加上了 `paddingBottom`。

此时，我想到的第一个思路就是能不能不让系统去设置这个 `paddingBottom`，这样不就解决了“**拉伸**”的问题？

解决思路：
在 `View#onMeasure()` 方法最后，强制设置 `paddingBottom = 0`。

为了不入侵 ToolBar，另外也不是所有项目都是用了 ToolBar，很多项目都有自己的“**TitleBar**”，所以我们将上面的 xml 布局改造如下：

    ...    
    <com.example.hiphonezhu.immersivedemo.ImmerseGroup
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:background="@color/colorPrimary"
        android:fitsSystemWindows="true">
        <!-- 可以是其他 TitleBar -->
        <android.support.v7.widget.Toolbar
            android:id="@+id/toolbar"
            android:layout_width="match_parent"
            android:layout_height="wrap_content" />
    </com.example.hiphonezhu.immersivedemo.ImmerseGroup>
    ...

用一个 ImmerseGroup 去包裹 ToolBar，将`android:background`、`android:fitsSystemWindows`两个属性从 ToolBar 移到 ImmerseGroup。

ImmerseGroup 的实现也很简单：

    public class ImmerseGroup extends FrameLayout {
        public ImmerseGroup(Context context) {
            super(context);
        }

        public ImmerseGroup(Context context, @Nullable AttributeSet attrs) {
            super(context, attrs);
        }

        public ImmerseGroup(Context context, @Nullable AttributeSet attrs, int defStyleAttr) {
            super(context, attrs, defStyleAttr);
        }

        @Override
        protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
            setPadding(getPaddingLeft(), getPaddingTop(), getPaddingRight(), 0);
            super.onMeasure(widthMeasureSpec, heightMeasureSpec);
        }
    }

重点就是重写 `onMeasure()` 方法，设置 `paddingBottom = 0`。

![此处输入图片的描述][3]
重新运行，点击输入框，ToolBar 没有再被“拉伸”了。

但是，又出现了另外一个问题：**输入框不见了**，其实是输入框没有被顶起来。
> 这个问题就是网上很多文章描述的 adjustResize 失效问题，解决方案基本都一致：在跟布局加上 `android:fitsSystemWindows="true"`。这个方案前面已经讨论过，不是一个好的方案。

> 问题参考：http://blog.163.com/ittfxin@126/blog/static/11067486320162210549679/


----------


我们最终的解决方案是：通过监听`android.R.id.content` 布局的变化，动态去改变它的高度，来为键盘腾出空间。

这里可以直接使用了一个工具类：AndroidBug5497Workaround

    public class AndroidBug5497Workaround {
        public static void assistActivity(View content) {
            new AndroidBug5497Workaround(content);
        }

        private View mChildOfContent;
        private int usableHeightPrevious;
        private ViewGroup.LayoutParams frameLayoutParams;

        private AndroidBug5497Workaround(View content) {
           if (content != null) {
              mChildOfContent = content;
              mChildOfContent.getViewTreeObserver().addOnGlobalLayoutListener(new ViewTreeObserver.OnGlobalLayoutListener() {
                   public void onGlobalLayout() {
                       possiblyResizeChildOfContent();
                        }
             });
             frameLayoutParams = mChildOfContent.getLayoutParams();
           }
        }

        private void possiblyResizeChildOfContent() {
            int usableHeightNow = computeUsableHeight();
            if (usableHeightNow != usableHeightPrevious) {
                //如果两次高度不一致
                //将计算的可视高度设置成视图的高度
                frameLayoutParams.height = usableHeightNow;
                mChildOfContent.requestLayout();//请求重新布局
                usableHeightPrevious = usableHeightNow;
            }
        }

        private int computeUsableHeight() {
            //计算视图可视高度
            Rect r = new Rect();
            mChildOfContent.getWindowVisibleDisplayFrame(r);
            return (r.bottom);
        }
    }
在需要适配的 Activity 调用如下方法：

    AndroidBug5497Workaround.assistActivity(findViewById(android.R.id.content));

最后的效果如下：
![此处输入图片的描述][4]

示例代码参考：[ImmersiveDemo](https://github.com/hiphonezhu/Android-Demos/tree/master/ImmersiveDemo)

  [1]: http://upload-images.jianshu.io/upload_images/1787010-0dc32da35af42984.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240
  [2]: http://upload-images.jianshu.io/upload_images/1787010-299a0735baa48fdf.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240
  [3]: http://upload-images.jianshu.io/upload_images/1787010-a2978c81ac61a8e7.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240
  [4]: http://upload-images.jianshu.io/upload_images/1787010-0eb59ab182f7f21c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240