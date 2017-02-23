---
layout:     post
title:      "刨根问底-论Android“沉浸式”"
subtitle:   ""
date:       2016-09-11 12:00:00
author:     "zhuhf"
header-img: ""
header-mask: 0.3
catalog:    true
tags:
    - Android
    - NestedScrolling
---

网上谈论“沉浸式”的文章多的不可胜数，有人把**“沉浸式”**叫做**“沉浸式状态栏”**，还有人称作为**“透明状态栏”**、**“变色状态栏”**等。

这里我们先给出**“沉浸式”**的直观感受，引用“知乎”网友的答案
> 使用带虚拟键的手机才能明显感觉到沉浸式所带来的变化：状态栏、导航栏隐藏。
而对于使用实体按键的手机的用户来说，**“沉浸式”**所带来的变化仅仅是状态栏隐藏，事实上，状态栏隐藏在之前也很常见，各种国产应用启动时都会隐藏状态栏。

出处来自：[为什么在国内会有很多用户把 ｢透明栏｣（Translucent Bars）称作 ｢沉浸式顶栏｣？](http://www.zhihu.com/question/27040217)感兴趣的朋友可以去阅读下。

把上面的文字翻译成图片的话，标准的**“沉浸式”**大致是这个效果。

![标准沉浸式.gif](http://upload-images.jianshu.io/upload_images/1787010-f5290e1a2916677d.gif?imageMogr2/auto-orient/strip)

App默认是全屏的，用户可以从顶部或者底部“滑出”状态栏和导航栏，一段时间后状态栏和导航栏会自动消失。

嗯，没错，这个才是标准的**“沉浸式”**。

有了以上的定义，下面这两种常见的效果，只能说改变了状态栏的“样式”（Translucent Bar倒是比较贴近）。但是，他们不是**“沉浸式”**。


![非沉浸式.png](http://upload-images.jianshu.io/upload_images/1787010-fb160a536ecc511d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

接下来，将进入本篇文章的主题，我们会从两个方面去**“论沉浸式”**。
* 4.4之前，谷歌是如何提供类似**“沉浸式”**的体验的
* 如何以最简单的方式实现**“状态栏”**与App**“合二为一”**的效果（真个真的不叫**沉浸式**啦~）

**setSystemUiVisibility**
4.0之后（3.0也支持，这里暂时只讨论手机平台），官方提供了这个方法，可以改变系统UI的可见性，使用方法如下：

    int flag = View.SYSTEM_UI_FLAG_FULLSCREEN;
    getWindow().getDecorView().setSystemUiVisibility(flag);

多个值可使用“|”操作符，比如：

    int flag = View.SYSTEM_UI_FLAG_FULLSCREEN
               | View.SYSTEM_UI_FLAG_LAYOUT_FULLSCREEN;
    getWindow().getDecorView().setSystemUiVisibility(flag);

系统提供了很多类似View.SYSTEM_UI_FLAG_xxx的常量，具体有下面几种：

* SYSTEM_UI_FLAG_FULLSCREEN(4.1+)：隐藏状态栏，手指在屏幕顶部往下拖动，状态栏会再次出现且不会消失，另外activity界面会重新调整大小，直观感觉就是activity高度有个变小的过程。

![SYSTEM_UI_FLAG_FULLSCREEN.gif](http://upload-images.jianshu.io/upload_images/1787010-8e4c3d8282c39997.gif?imageMogr2/auto-orient/strip)

* SYSTEM_UI_FLAG_LAYOUT_FULLSCREEN(4.1+)：配合SYSTEM_UI_FLAG_LAYOUT_FULLSCREEN一起使用，效果使得状态栏出现的时候不会挤压activity高度，状态栏会覆盖在activity之上。

![SYSTEM_UI_FLAG_LAYOUT_FULLSCREEN.gif](http://upload-images.jianshu.io/upload_images/1787010-4ccb87bc968fe191.gif?imageMogr2/auto-orient/strip)

* SYSTEM_UI_FLAG_HIDE_NAVIGATION(4.0+)
：会使得虚拟导航栏隐藏，但同样用户可以从屏幕下边缘“拖出”且不会再次消失，同时activity界面会被挤压。

![SYSTEM_UI_FLAG_HIDE_NAVIGATION.gif](http://upload-images.jianshu.io/upload_images/1787010-0f7a0871cb0fa1a6.gif?imageMogr2/auto-orient/strip)

* SYSTEM_UI_FLAG_LAYOUT_HIDE_NAVIGATION(4.1+)：配合 SYSTEM_UI_FLAG_HIDE_NAVIGATION 一起使用，效果使得导航栏出现的时候不会挤压activity高度，导航栏会覆盖在activity之上。


![SYSTEM_UI_FLAG_LAYOUT_HIDE_NAVIGATION.gif](http://upload-images.jianshu.io/upload_images/1787010-7a96d237d4f3b047.gif?imageMogr2/auto-orient/strip)

* SYSTEM_UI_FLAG_LAYOUT_STABLE(4.1+)：使用以上四个属性，可以达到activity占据屏幕所有空间，同时状态栏和导航栏可以悬浮在activity之上的效果。但是此时activity的内容也会（比如顶部和底部各有一个TextView）状态栏和导航栏之下，当状态栏和导航栏出现的时候，看起来会这样：

![1.gif](http://upload-images.jianshu.io/upload_images/1787010-a27f209613c4a7a3.gif?imageMogr2/auto-orient/strip)

显然，文字被遮盖了我们是不能接受的，此时我们需要另外一个属性，```android:fitsSystemWindows=“true”```，这个属性表示系统UI（状态栏、导航栏）可见的时候，会给我们的布局加上padding（paddingTop、paddingBottom）属性，这样内容就不会被盖住了。我们在activity的根布局加上这个属性，效果如下

![fitsSystemWindows.gif](http://upload-images.jianshu.io/upload_images/1787010-6873c7b3e8af14e0.gif?imageMogr2/auto-orient/strip)

我们发现，当状态栏和导航栏出现的时候，内容会被挤压一下，这个体验也不是很理想，那么怎么解决这个问题呢？

其实，我们的内容本就不该占据状态栏和导航栏的位置（游戏一般才会全屏，普通app的状态栏一般都是透明而不是让内容去占据这个空间，后面会介绍怎么实现状态栏“透明效果”），我们需要加上SYSTEM_UI_FLAG_LAYOUT_STABLE这个属性来解决这个问题
，效果如下


![SYSTEM_UI_FLAG_LAYOUT_STABLE.gif](http://upload-images.jianshu.io/upload_images/1787010-9500c3098726d320.gif?imageMogr2/auto-orient/strip)

以上都是4.1(除了SYSTEM_UI_FLAG_HIDE_NAVIGATION
)的属性，观察之后我们发现，不管是那种属性，状态栏和导航栏总是会“遮挡”activity，为了解决这个问题，4.4引入了**“全屏沉浸模式”**这个概念。

* SYSTEM_UI_FLAG_IMMERSIVE(4.4+)：这个属性是用来实现**“沉浸式”**效果的，官方称作**“Immersive full-screen mode”**。
> To provide your app with a layout that fills the entire screen, the new ***SYSTEM_UI_FLAG_IMMERSIVE***
 flag for ***setSystemUiVisibility()***
 (when combined with ***SYSTEM_UI_FLAG_HIDE_NAVIGATION***
) enables a new *immersive* full-screen mode. 

大意是SYSTEM_UI_FLAG_IMMERSIVE和 SYSTEM_UI_FLAG_HIDE_NAVIGATION一起使用，会带来一个全新的**“全屏沉浸模式”**。我们使用这两个属性看下效果~

![immersive1.gif](http://upload-images.jianshu.io/upload_images/1787010-0fcabd8001ed942c.gif?imageMogr2/auto-orient/strip)

纳尼？显然和官方描述的占据整个屏幕有点不符合，状态栏一直占据着空间（不知道为何官方这么描述~）。我们再加一个属性SYSTEM_UI_FLAG_FULLSCREEN，我们再来看下效果

![immersive2.gif](http://upload-images.jianshu.io/upload_images/1787010-0ad0281615c13916.gif?imageMogr2/auto-orient/strip)

挤压的效果如果你不满意，加上SYSTEM_UI_FLAG_LAYOUT_FULLSCREEN和SYSTEM_UI_FLAG_LAYOUT_HIDE_NAVIGATION，这会让状态栏和导航栏“悬浮”在activity之上，这个前面提到过，不做更多解释，效果如下

![immersive3.gif](http://upload-images.jianshu.io/upload_images/1787010-aaeb684da3ba6ecf.gif?imageMogr2/auto-orient/strip)

* SYSTEM_UI_FLAG_IMMERSIVE_STICKY：和SYSTEM_UI_FLAG_IMMERSIVE相似，它被称作“粘性”的沉浸模式，这个模式会在状态栏和导航栏显示一段时间后，自动隐藏（你可以点击一下屏幕，立即隐藏）。同时需要重点说明的是，这种模式下，状态栏和导航栏出现的时候是“半透明”状态，效果如下

![immersive4.gif](http://upload-images.jianshu.io/upload_images/1787010-d101b65f5e7523ca.gif?imageMogr2/auto-orient/strip)

我觉得官方要表达的**“Immersive full-screen mode”**，应该是这个意思才对吧？

慎用**“沉浸式”**，除非你真的需要这样做。比如做一款游戏或者绘图应用就很合适。

**“Translucent Bar”**
网上所谓的**“沉浸式状态栏”**，这个概念官方从未提及过。我看了就大多数文章，最终表达的效果基本是这样

![预览.png](http://upload-images.jianshu.io/upload_images/1787010-b05990571f4bda80.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

为了对比，贴出了不同Android版本的效果（某Q也是如此），其实就是为了让状态栏看上起和我们应用的标题栏（当然，也可能是和整个背景）“合二为一”，不至于和之前一样，看到的是“黑乎乎”的状态栏，网上也有很多第三方库来实现这样的效果，比如：[SystemBarTint](https://github.com/jgilfelt/SystemBarTint)。

其实这个效果4.4以上实现很简单，大概需要用到以下两个属性：
* windowTranslucentNavigation：application的主题加上这个属性，表示状态栏半透明，另外，会使得状态栏会悬浮在activity之上(此时，activity布局会扩展到状态栏底部（Z轴方向）)：

      <style name="AppTheme" parent="Theme.AppCompat.Light.NoActionBar">                                
          <item name="android:windowTranslucentStatus">true</item>
      </style>

为了不遮挡activity内容，需要配合另外一个属性
* android:fitsSystemWindows：使用这个属性的View，系统会在View顶部添加padding（大小为状态栏高度）：

      <?xml version="1.0" encoding="utf-8"?>
      <LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
          android:layout_width="match_parent"
          android:layout_height="match_parent"
          android:orientation="vertical">
          <LinearLayout
              android:layout_width="match_parent"
              android:layout_height="wrap_content"
              android:fitsSystemWindows="true"
              android:background="@color/colorPrimary">
              <!--嵌套的原因是因为LinearLayout会被加上paddingTop，导致TextView无法居中-->
              <RelativeLayout
                  android:layout_width="match_parent"
                  android:layout_height="50dp">
                  <TextView
                      android:layout_width="wrap_content"
                      android:layout_height="wrap_content"
                      android:layout_centerInParent="true"
                      android:textColor="@android:color/white"
                      android:textSize="20sp"
                      android:text="这是标题"/>
              </RelativeLayout>
          </LinearLayout>
      </LinearLayout>

有些应用没有标题，希望将背景和标题融为一体，比如顶部是“轮播”栏：

![预览2.png](http://upload-images.jianshu.io/upload_images/1787010-8ace1ccc46307b3e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

其实也很简单，布局稍微调整下（```android:fitsSystemWindows="true"```不需要设置。如果你是对一个ViewGroup设置背景，你可能会用到这个属性，因为你不想ViewGroup的子布局被状态栏“遮盖”）：

    <?xml version="1.0" encoding="utf-8"?>
    <LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        android:orientation="vertical">
        <ImageView
            android:layout_width="match_parent"
            android:layout_height="150dp"
            android:scaleType="centerCrop"
            android:src="@drawable/bg"/>
        <TextView
            android:layout_width="match_parent"
            android:layout_height="match_parent"
            android:gravity="center"
            android:text="内容区域"/>
    </LinearLayout>

总之一个原则是，如果使用以下属性，activity布局会扩展到状态栏

    <item name="android:windowTranslucentStatus">true</item>

如果你希望扩展的区域，不被状态栏盖住内容，那就加上

    android:fitsSystemWindows="true"

我们注意到，**状态栏**在4.4-5.0之间的效果是全透明，5.0+是半透明，颜色的差别QQ等App也没有特意去做处理，我也觉得没必要，你觉得呢？

好了，希望本文能帮助你真正理解什么是**“沉浸式”**。
