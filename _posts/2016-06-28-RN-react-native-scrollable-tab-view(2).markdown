---
layout:     post
title:      "[React Native]react-native-scrollable-tab-view(进阶篇)"
subtitle:   ""
date:       2016-06-28 12:00:00
author:     "zhuhf"
header-img: ""
header-mask: 0.3
catalog:    true
tags:
    - RN
    - react-native-scrollable-tab-view
---

在上一篇文章当中，我们学习了[react-native-scrollable-tab-view](http://zhuhf.tech/2016/06/23/RN-react-native-scrollable-tab-view(1))的基本使用方式，包括基本**Props**的使用介绍等。我们知道官方为我们提供了两种基本的Tab控制器样式，**DefaultTabBar**和**ScrollableTabBar**。很多情况下，官方的样式并不能满足我们的需求（备注：官方的样式是文字+下划线的风格），那么此时就需要我们自己来实现特定的样式。

---
今天我们是react-native-scrollable-tab-view的下篇，本片文章的很多概念建立在上篇的基础之上，还没有阅读过的朋友可以先去参考[[React Native]react-native-scrollable-tab-view(入门篇)](http://www.jianshu.com/p/b7788c3d106e)。

本文要实现这样的效果：

![preview.gif](http://upload-images.jianshu.io/upload_images/1787010-6e6cb4165725b28e.gif?imageMogr2/auto-orient/strip)

## 一、准备工作

**1.新建一个项目**

    react-native init Demo7

**2.添加[react-native-scrollable-tab-view](https://github.com/skv-headless/react-native-scrollable-tab-view)**

    npm install react-native-scrollable-tab-view --save

**3.添加react-native-vector-icons**

    > npm install react-native-vector-icons --save
    > rnpm link

[rnpm](https://github.com/rnpm/rnpm)是一个React Native包管理器，我们也可以通过编辑```android/app/build.gradle``` 添加下面的行达到同样的目的:

    apply from: "../../node_modules/react-native-vector-icons/fonts.gradle"

**[react-native-vector-icons](https://github.com/oblador/react-native-vector-icons)介绍：**
>一个“图标”库，官方描述为**‘3000 Customizable Icons for React Native with support for NavBar/TabBar/ToolbarAndroid, image source and full stying.’** 可见，这个库为我们提供了很多图标，如果你不想花费时间去设计一些图标，不妨使用这个库来替代。

   有趣的是，这个库的图标来源有很多，下面大概列举了一些：
  * [Entypo
](http://entypo.com/)by Daniel Bruce (**411**icons)
  * [EvilIcons
](http://evil-icons.io/)by Alexander Madyankin & Roman Shamin (v1.8.0,**70**icons)
  * [FontAwesome
](http://fortawesome.github.io/Font-Awesome/icons/)by Dave Gandy (v4.6.3,**634**icons)
  * [Foundation
](http://zurb.com/playground/foundation-icon-fonts-3)by ZURB, Inc. (v3.0,**283**icons)
  * [Ionicons
](http://ionicframework.com/docs/v2/ionicons/)by Ben Sperry (v3.0.0,**859**icons)
  * [MaterialIcons
](https://www.google.com/design/icons/)by Google, Inc. (v2.2.3,**932**icons)
  * [Octicons
](http://octicons.github.com/)by Github, Inc. (v3.5.0,**166**icons)
  * [Zocial
](http://zocial.smcllns.com/)by Sam Collins (v1.0,**100**icons)

其中用的最多的是[Ionicons
](http://ionicframework.com/docs/v2/ionicons/)，所以本篇文章的图标来源也就选择它了。
  另外，react-native-vector-icons的用法非常的多，我们今天只会用到3个基本属性：


  属性| 描述| 默认值
  :----|:------|:----
  **size**| 设置图标的大小 | 12
  **name** | 设置使用的图标 | 无
  **color** | 设置图标的颜色 | 系统默认

 其中，```name```就是你要使用的图标名称，如果我们选择[Ionicons
](http://ionicframework.com/docs/v2/ionicons/)，让我们看下如何找到你需要的图标，进入[Ionicons
](http://ionicframework.com/docs/v2/ionicons/)，我们看到如下的界面

![ionicons.png](http://upload-images.jianshu.io/upload_images/1787010-cf2168805fb30d9d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
红色区域输入你想要的图标名称（英文哈~），比如我们输入```search```，结果页面如下
![search.png](http://upload-images.jianshu.io/upload_images/1787010-424db64ea903e3ec.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
每个图标，都提供了iOS、iOS-Outline、Material Design三种不同风格的样式，点击结果中的某一行数据，出现如下界面

![result.png](http://upload-images.jianshu.io/upload_images/1787010-301307c694affcc3.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
红色区域就是我们最终需要的图标的名称，即```name```的值。

---

## 二、开始工作

首先，自定义一个TabBar组件你需要知道以下几个点：

* 添加必要属性到组件中（**必选**）

      propTypes = {
		goToPage: React.PropTypes.func, // 跳转到对应tab的方法
		activeTab: React.PropTypes.number, // 当前被选中的tab下标
		tabs: React.PropTypes.array, // 所有tabs集合
	  }

* 实现```setAnimationValue```（**可选**，如果你需要在tab切换的时候有动画效果）

      setAnimationValue({value}) {

	  }

* ```render```方法需要返回一个组件作为TabBar
* 跟其他任何组件一样，你可以传递自己的```props```

好了，介绍完要点，我们就开始编写TabBar组件了。

**1.新建一个WeixinTabBar.js文件，导入[Ionicons
](http://ionicframework.com/docs/v2/ionicons/)。**

    import Icon from 'react-native-vector-icons/Ionicons';

**2.我们希望每个Tab的**图标**和**名称**都是外部组件通过```prop```传递进来，而不是内部写死，这样有利于扩展，所以我们添加两个```prop```：```tabNames```和```tabIconNames```**

    propTypes = {
	  ...
      tabNames: React.PropTypes.array, // 保存Tab名称
      tabIconNames: React.PropTypes.array, // 保存Tab图标
	}

**3.实现```render```方法**

    render() {
      return (
        <View style={styles.tabs}>
        	{this.props.tabs.map((tab, i) => this.renderTabOption(tab, i))}
        </View>
      );
    }

这个方法很简单，返回一个**容器View**，**容器View**内包含的**子View**是通过遍历```tabs```，调用```renderTabOption```方法来动态生成的。

**4.实现```renderTabOption```方法**

    renderTabOption(tab, i) {
      const color = this.props.activeTab == i? "#6B8E23" : "#ADADAD"; // 判断i是否是当前选中的tab，设置不同的颜色
      return (
        <TouchableOpacity onPress={()=>this.props.goToPage(i)} style={styles.tab}>
          <View style={styles.tabItem}>
            <Icon
              name={this.props.tabIconNames[i]}  // 图标
              size={30}
              color={color}/>
            <Text style={{color: color}}>
              {this.props.tabNames[i]}
            </Text>
          </View>
        </TouchableOpacity>
       );
    }

这个方法应该也比较简单，使用```TouchableOpacity ```包裹两个```View```：```Icon```和```Text```。

**代码分析：**  

-start-

首先，判断```i```是否是当前选中的```activeTab```，来使用不同的颜色，然后：
TouchableOpacity：点击触发onPress方法，使用goToPage跳转到对应的tab
Icon：设置name(图标，使用tabIconNames[i]获取)，size(图标大小)，color(图标颜色)
Text：设置文本（使用tabNames[i]获取），color(文字颜色)

-end-

**5.使用```WeixinTabBar```**

打开index.android.js文件，导入`ScrollableTabView` 和 `WeixinTabBar`

    > import ScrollableTabView from 'react-native-scrollable-tab-view'
    > import WeixinTabBar from './WeixinTabBar'

我们最终实现的效果图有4个```tab```，所以这里定义两个数组```tabNames```和```tabIconNames```，分别表示每个```tab```显示的文字和图片

    constructor(props) {
	  super(props);

	  this.state = {
	  	tabNames: ['Tab1', 'Tab2', 'Tab3', 'Tab4'],
	  	tabIconNames: ['ios-paper', 'ios-albums', 'ios-paper-plane', 'ios-person-add'],
	  };
	}

最后，实现```render```方法

    render() {
      let tabNames = this.state.tabNames;
      let tabIconNames = this.state.tabIconNames;
      return (
        <ScrollableTabView
          renderTabBar={() => <WeixinTabBar tabNames={tabNames} tabIconNames={tabIconNames}/>}
          tabBarPosition='bottom'>
          <View style={styles.content} tabLabel='key1'>
            <Text>#1</Text>
          </View>
          <View style={styles.content} tabLabel='key2'>
            <Text>#2</Text>
          </View>
          <View style={styles.content} tabLabel='key3'>
            <Text>#3</Text>
          </View>
          <View style={styles.content} tabLabel='key4'>
            <Text>#4</Text>
          </View>
        </ScrollableTabView>
      );
    }

其中```renderTabBar```使用我们自定义的```WeixinTabBar```。需要说明的是，即使我们不使用系统的```DefaultTabBar```和```ScrollableTabBar```，但```tabLabel```这个属性必须使用，且值不能重复。

最后，感谢大家耐心看完本篇文章~

**本文的源码地址**：[Demo7](https://github.com/hiphonezhu/RN-Demos/tree/master/Demo7)
