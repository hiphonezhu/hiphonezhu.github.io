---
layout:     post
title:      "[React Native]弹性布局Flexbox"
subtitle:   ""
date:       2016-06-20 12:00:00
author:     "zhuhf"
header-img: ""
header-mask: 0.3
catalog:    true
tags:
    - RN
    - Flexbox
---

## 什么是Flexbox？

简单来说Flexbox是属于web前端领域CSS的一种布局方案，是2009年W3C提出了一种新的布局方案，可以简便、完整、响应式地实现各种页面布局。你可以简单的理解为Flexbox是CSS领域类似Android中 **LinearLayout**的一种布局，但是要比 **LinearLayout**要强大的多。

## ReactNative中的Flexbox

总所周知，移动端在使用和操作习惯与PC端有着截然的不同，这就注定了它和WEB端在布局方式以及复杂度方面有着巨大的差别。所以在ReactNative中，官方对Flexbox做了一些阉割，以用来适应移动端的布局方式。

关于完整的Flexbox布局教程，可以参考阮一峰的[Flex 布局教程：语法篇](http://www.ruanyifeng.com/blog/2015/07/flex-grammar.html)。

## ReactNative中Flexbox常用的几个属性

#### 容器属性

flexDirection、justifyContent、alignItems、flexWrap

#### 元素属性

alignSelf、flex、position

>本文只介绍重点的几个属性，其他类似**marginLeft、padding**等，无论你之前是做网页开发，还是做原生开发，应该都非常熟悉，所以这里不做过多说明，大家稍微尝试看一下效果就知道了。另外，更多属性支持请查看[官方文档](http://reactnative.cn/docs/0.27/flexbox.html#content)。

在我们介绍这些属性之前，有一个概念，需要跟大家讲解下，那就是**主轴**和**交叉轴**。上面的多数属性和这个概念有直接关系，起初学习Flexbox可能比较困惑，有可能就是没理解清楚这个概念。

**主轴**和**交叉轴**是由```flexDirection```这个属性来决定的，让我们首先来看下```flexDirection```

#### flexDirection

这个属性决定了子View排列的方向，类似Android中LinearLayout的orientation属性，它有两个值：

**row**：横向排列，**主轴**为水平方向，**交叉轴**为垂直方向；

**column**：竖直排列，**主轴**为垂直方向，**交叉轴**为水平方向。
其中，默认值是**column**。

![flexDirection.png](http://upload-images.jianshu.io/upload_images/1787010-1a71e040a5135fae.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

#### justifyContent

设置子布局在主轴方向位置

![justifyContent.png](http://upload-images.jianshu.io/upload_images/1787010-98908935704e6eea.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

#### alignItems

设置子布局在交叉轴方向位置

![alignItems.png](http://upload-images.jianshu.io/upload_images/1787010-956079d832321201.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

#### flexWrap

水平或垂直布局时：如果子View放不下，则自动换行, 默认为'nowrap'(不换行)

![flexWrap.png](http://upload-images.jianshu.io/upload_images/1787010-505afef93a3c06fb.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

#### alignSelf

设置子布局在交叉轴方向位置

![alignSelf.png](http://upload-images.jianshu.io/upload_images/1787010-fc82fbcd74094cbb.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
#### flex

权重，类似Android中weight

![flex.png](http://upload-images.jianshu.io/upload_images/1787010-c3d793fdf8c6f132.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

#### position

定位模式，分为absolute和relative两种

#### absolute

绝对定位，相对于原点（左上角）

    <Text style=\{\{
      fontSize: 20,
      textAlign: 'center',
      backgroundColor: '#0000FF',
      color: '#FFFFFF',
      position: 'absolute',
      left: 100,
      top: 100
      \}\}>
      B
    </Text>

![absolute.png](http://upload-images.jianshu.io/upload_images/1787010-2939ee541cbe83fa.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

#### relative

相对定位，相对于当前位置

    <Text style=\{\{
      fontSize: 20,
      textAlign: 'center',
      backgroundColor: '#0000FF',
      color: '#FFFFFF',
      position: 'relative',
      left: 100,
      top: 100
      \}\}>
      B
    </Text>

![relative.png](http://upload-images.jianshu.io/upload_images/1787010-0272dd96bf12b5ea.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
