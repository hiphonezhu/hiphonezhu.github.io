---
layout:     post
title:      "[React Native]react-native-scrollable-tab-view(入门篇)"
subtitle:   ""
date:       2016-06-23 12:00:00
author:     "zhuhf"
header-img: ""
header-mask: 0.3
catalog:    true
tags:
    - RN
    - react-native-scrollable-tab-view
---

官方为我们提供的Tab控制器有两种：
[TabBarIOS](http://reactnative.cn/docs/0.27/tabbarios.html#content)，仅适用于IOS平台
[ViewPagerAndroid](http://reactnative.cn/docs/0.27/viewpagerandroid.html#content)，仅适用于Android平台（严格来讲并不算，因为我们还需要自己实现Tab）

如果我们需要一个更通用的Tab控制器，那么就要借助开源的力量，本篇文章教你如何使用
[react-native-scrollable-tab-view](https://github.com/skv-headless/react-native-scrollable-tab-view)，这是官方Demo的效果
![demo.gif](http://upload-images.jianshu.io/upload_images/1787010-84dcc34642976efd.gif?imageMogr2/auto-orient/strip)![demo-fb.gif](http://upload-images.jianshu.io/upload_images/1787010-f7aa1bc8275bfb3e.gif?imageMogr2/auto-orient/strip)

## 一、准备工作

1. 新建一个项目

        react-native init Demo6

2. 添加react-native-scrollable-tab-view

        npm install react-native-scrollable-tab-view --save


## 二、Props介绍

#### renderTabBar*(Function:ReactComponent)

TabBar的样式，系统提供了两种默认的，分别是```DefaultTabBar```和```ScrollableTabBar```。当然，我们也可以自定义一个，我们会在下篇文章重点讲解如何去自定义TabBar样式。

    render() {
      return (
        <ScrollableTabView
          renderTabBar={() => <DefaultTabBar/>}>
          <Text tabLabel='Tab1'/>
          <Text tabLabel='Tab2'/>
          <Text tabLabel='Tab3'/>
          <Text tabLabel='Tab4'/>
          <Text tabLabel='Tab5'/>
          <Text tabLabel='Tab6'/>
        </ScrollableTabView>
      );
    }

注意：每个被包含的子视图需要使用**tabLabel**属性，表示对应Tab显示的文字

**DefaultTabBar**：Tab会平分在水平方向的空间

![default.png](http://upload-images.jianshu.io/upload_images/1787010-a9b6daddbf9fe647.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
**ScrollableTabBar**：Tab可以超过屏幕范围，滚动可以显示
![scrollable.png](http://upload-images.jianshu.io/upload_images/1787010-e4a297e713728e62.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

#### tabBarPosition(String，默认值'top')

    render() {
      return (
        <ScrollableTabView
          tabBarPosition='top'
          renderTabBar={() => <DefaultTabBar/>}>
          ...
        </ScrollableTabView>
      );
    }

**top**：位于屏幕顶部
![default.png](http://upload-images.jianshu.io/upload_images/1787010-23d94ebfb5cc8183.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
**bottom**：位于屏幕底部
![bottom.png](http://upload-images.jianshu.io/upload_images/1787010-1ec1da432a332143.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
**overlayTop**：位于屏幕顶部，悬浮在内容视图之上（看颜色区分：视图有颜色，Tab栏没有颜色）
![overlayTop.png](http://upload-images.jianshu.io/upload_images/1787010-ddb9d119e4ca47bd.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
**overlayBottom**：位于屏幕底部，悬浮在内容视图之上（看颜色区分：视图有颜色，Tab栏没有颜色）
![overlayBottom.png](http://upload-images.jianshu.io/upload_images/1787010-731d8b172cd42cfb.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

#### onChangeTab(Function)

Tab切换之后会触发此方法，包含一个参数（Object类型），这个对象有两个参数
**i**：被选中的Tab的下标（从0开始）
**ref**：被选中的Tab对象（基本用不到）

    render() {
      return (
        <ScrollableTabView
          renderTabBar={() => <DefaultTabBar/>}
          onChangeTab={(obj) => {
              console.log('index:' + obj.i);
            }
          }>
          ...
        </ScrollableTabView>
      );
    }

#### onScroll(Function)

视图正在滑动的时候触发此方法，包含一个Float类型的数字，范围是[0, tab数量-1]

    render() {
      return (
        <ScrollableTabView
          renderTabBar={() => <DefaultTabBar/>}
          onScroll={(postion) => {  
              // float类型 [0, tab数量-1]  
              console.log('scroll position:' + postion);
            }
          }>
          ...
        </ScrollableTabView>
      );
    }

#### locked(Bool，默认为false)

表示手指是否能拖动视图，默认为false（表示可以拖动）。设为true的话，我们只能**“点击”**Tab来切换视图。

    render() {
      return (
        <ScrollableTabView
          locked={false}
          renderTabBar={() => <DefaultTabBar/>}>
          ...
        </ScrollableTabView>
      );
    }

#### initialPage(Integer)

初始化时被选中的Tab下标，默认是0（即第一页）

    render() {
      return (
        <ScrollableTabView
          initialPage={1}
          renderTabBar={() => <DefaultTabBar/>}>
          ...
        </ScrollableTabView>
      );
    }

> 2016.12.12更新：react-native-scrollable-tab-view最新版本0.7.0，此属性在Android平台无效，具体表现为页面不会被“渲染”，但是iOS平台是没问题的。建议大家暂时使用0.6.0，作者表示已经准备修复此问题，详见：https://github.com/skv-headless/react-native-scrollable-tab-view/issues/483

#### page(Integer)

设置选中指定的Tab（然而测试下来并没有任何效果，知道原因的同学麻烦留言下 ~）
>写于2016.06.29：跟作者沟通下来下个版本打算废弃这个属性，原话为‘It is a legacy I would like to remove it but someone might use it. Probably in next version I'll remove it.’参考[issue#320](https://github.com/skv-headless/react-native-scrollable-tab-view/issues/320#issuecomment-229266981)

#### children(ReactComponents)

表示所有子视图的数组，比如下面的代码，children则是一个长度为6的数组，元素类型为Text

    render() {
      return (
        <ScrollableTabView
          renderTabBar={() => <DefaultTabBar/>}>
          <Text tabLabel='Tab1'/>
          <Text tabLabel='Tab2'/>
          <Text tabLabel='Tab3'/>
          <Text tabLabel='Tab4'/>
          <Text tabLabel='Tab5'/>
          <Text tabLabel='Tab6'/>
        </ScrollableTabView>
      );
    }

#### tabBarUnderlineStyle(style)

设置```DefaultTabBar```和```ScrollableTabBar```Tab选中时下方横线的颜色
#### tabBarBackgroundColor(String)

设置整个Tab这一栏的背景颜色
#### tabBarActiveTextColor(String)

设置选中Tab的文字颜色
#### tabBarInactiveTextColor(String)

设置未选中Tab的文字颜色
#### tabBarTextStyle(Object)

设置Tab文字的样式，比如字号、字体等
**上面这5个样式示例如下**：

    render() {
      return (
        <ScrollableTabView
          renderTabBar={() => <DefaultTabBar />}
          tabBarUnderlineColor='#FF0000'
          tabBarBackgroundColor='#FFFFFF'
          tabBarActiveTextColor='#9B30FF'
          tabBarInactiveTextColor='#7A67EE'
          tabBarTextStyle={{fontSize: 18}}
        >
        ...
        </ScrollableTabView>
      );
    }

  ![top5.png](http://upload-images.jianshu.io/upload_images/1787010-821e05f2f6d9084a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

#### style([View.propTypes.style](https://facebook.github.io/react-native/docs/view.html#style))

系统View都拥有的属性，基本不会涉及到。

#### contentProps(Object)

这里要稍微说下[react-native-scrollable-tab-view](https://github.com/skv-headless/react-native-scrollable-tab-view)的实现，其实在Android平台底层用的是```ViewPagerAndroid```，iOS平台用的是```ScrollView```。这个属性的意义是：比如我们设置了某个属性，最后这个属性会被应用在ScrollView/ViewPagerAndroid，这样会覆盖库里面默认的，通常官方不建议我们去使用。

#### scrollWithoutAnimation(Bool，默认为false)

设置**“点击”**Tab时，视图切换是否有动画，默认为false（即：有动画效果）。

    render() {
      return (
        <ScrollableTabView
          scrollWithoutAnimation={true}
          renderTabBar={() => <DefaultTabBar/>}>
          ...
        </ScrollableTabView>
      );
    }

  **注意：这个属性的设置对滑动视图切换的动画效果没有影响**，仅仅对**“点击”**Tab效果有作用。看下下面动态图的对比（今天录得动态图不知道为啥抽疯了，调了好几遍都不行，凑合看吧~）
![click.gif](http://upload-images.jianshu.io/upload_images/1787010-58e35b12fcb78868.gif?imageMogr2/auto-orient/strip)
![slide.gif](http://upload-images.jianshu.io/upload_images/1787010-a765b9dd5bac1759.gif?imageMogr2/auto-orient/strip)

好了，感谢大家耐心看到最后，下篇文章会介绍[react-native-scrollable-tab-view](https://github.com/skv-headless/react-native-scrollable-tab-view)如何去自定义一个TabBar样式。感兴趣的朋友请继续阅读[[React Native]react-native-scrollable-tab-view(进阶篇)](http://zhuhf.tech/2016/06/28/RN-react-native-scrollable-tab-view(2)/)。
