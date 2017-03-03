---
layout:     post
title:      "[React Native]动画-LayoutAnimation"
subtitle:   ""
date:       2016-07-05 12:00:00
author:     "zhuhf"
header-img: ""
header-mask: 0.3
catalog:    true
tags:
    - RN
    - LayoutAnimation
---

**实现动画的几种方式：**
* requestAnimationFrame
* setNativeProps
* LayoutAnimation
* Animated

本篇文章我们会介绍前三种方式，以及它们的区别。下一篇文章会介绍高级API—**Animated**的用法。

**requestAnimationFrame**
如果不使用任何动画API，我会想到一种简单粗暴的方式来实现动画效果—通过修改```state```不断改变视图的样式。

我们来个简单的示例：

    class Demo9 extends Component {
      constructor(props) {
          super(props);
          this.state = {
              width: 100,
              height: 100
          };
      }
      startAnimation() {
          var count = 0;
          while (++count < 50) {
              requestAnimationFrame(() = >{
                  this.setState({
                      width: this.state.width + 1,
                      height: this.state.height + 1
                  });
              });
          }
      }
      render() {
          return (
             <View style = {styles.container}>
               <Image source = { require('./icon.jpg') }
                  style={{width: this.state.width, height: this.state.height}}/>
               <TouchableOpacity style={styles.instructions} onPress={()=>this.startAnimation()}>
	              <Text style={{alignSelf: 'center', color: '#FFFFFF'}}>Press me!</Text>
               </TouchableOpacity>
              </View>
          );
      }
    }

效果图如下：


![preview1.gif](http://upload-images.jianshu.io/upload_images/1787010-5554e3677bf82271.gif?imageMogr2/auto-orient/strip)

这个方式实现的动画有几个问题：

1、实现方式是通过不断销毁、创建视图来完成，一方面如果你的视图的数据是动态获取的，那么就需要以合适的方式恢复数据；另外一方面，这种方式必然造成性能和内存开销的问题。

2、如果需要刷新的View的层级比较深，那么这种方式会带来严重的性能问题。

3、```requestAnimationFrame```毕竟是```web```上```css```的用法，在手机上，动画的效果比较生硬，如果需要‘弹性动画’，‘淡入淡出’等效果，则是比较难以实现的（需要辅助各种函数）。


**setNativeProps**
如果执意使用修改state的方式，觉得这种方式更符合当前需求对动画的控制，那么则应当使用原生组件的```setNativeProps```方法来做对应实现，它会直接修改组件底层特性，不会重绘组件，因此性能也远胜动态修改组件内联style的方法。

我们稍微修改下```startAnimation```方法：

    startAnimation() {
      var count = 0;
      while (++count < 50) {
          requestAnimationFrame(() = >{
              this.refs.image.setNativeProps({
                  style: {
                      width: this.state.width++,
                      height: this.state.height++
                  }
              });
          });
      }
    }

其中```this.refs.image```指向的是Image视图，效果图如下（比上一种方式流畅多了~）：

![preview2.gif](http://upload-images.jianshu.io/upload_images/1787010-c1e733378562e824.gif?imageMogr2/auto-orient/strip)
***优点：***

```setNativeProps```直接修改组件底层特性，不会重绘组件，因此性能也远胜动态修改组件内联style的方法。
***缺点：***

1、```setNativeProps```属于原生视图的方法，如果我们使用一个动画，单纯只是为了跟踪它的值，那么这么方法有点不合时宜。

2、还是和上面一种方式一样，如果需要实现‘弹性动画’，‘淡入淡出’等效果，则还是比较麻烦的。

---
***重点***，本篇文章的主角要登场了(此处有掌声~)

**LayoutAnimation**
当布局变化时，自动将视图运动到它们新的位置上。
一个常用的调用此API的办法是调用```LayoutAnimation.configureNext(config)```，然后调用```setState```。
> 建议大家阅读的时候打开```LayoutAnimation```源码，路径为：
```xxx/node_modules/react-native/Libraries/LayoutAnimation/LayoutAnimation.js```，其中‘xxx’为当前项目文件夹。

一个标准的```config```格式如下：(```create```、```update```和```delete```，分别表示视图```创建```、```更新```和```删除```时候的动画)


![Config.png](http://upload-images.jianshu.io/upload_images/1787010-f9555764f11614a8.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
示例：

    {   
      duration: 700,   //持续时间   
      create: {   // 视图创建         
        type: LayoutAnimation.Types.linear,      
        property: LayoutAnimation.Properties.scaleXY // opacity、scaleXY   
      },   
      update: { // 视图更新      
        type: LayoutAnimation.Types.spring,      
        springDamping: 0.4   
      },
    }

其中```create```、```update```和```delete```的```Anim```格式如下：


![Anim.png](http://upload-images.jianshu.io/upload_images/1787010-4f3647279984c8d3.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

*delay：*延迟指定时间（单位：毫秒）
*springDamping：*弹跳动画阻尼系数（配合```spring```使用）
*initialVelocity：*初始速度
*type：*类型定义在```LayoutAnimation.Types```中：
* spring：弹跳
* linear：线性
* easeInEaseOut：缓入缓出
* easeIn：缓入
* easeOut：缓出

*property：*类型定义在```LayoutAnimation.Properties```中：
* opacity：透明度
* scaleXY：缩放

让我们来看一段示例代码，同样是修改```startAnimation```方法：

    startAnimation() {
      LayoutAnimation.configureNext({
        duration: 700, //持续时间
        create: { // 视图创建
            type: LayoutAnimation.Types.spring,
            property: LayoutAnimation.Properties.scaleXY,// opacity、scaleXY
        },
        update: { // 视图更新
            type: LayoutAnimation.Types.spring,
        },
      });
      this.setState({width: this.state.width + 10, height: this.state.height + 10});
    }

效果图如下：
![config1.gif](http://upload-images.jianshu.io/upload_images/1787010-422fdf87dff4215f.gif?imageMogr2/auto-orient/strip)

观察下这个动画：
视图创建的时候是缩放动画，点击‘Press Me!’，图片也会有个缩放动画。


我们还可以通过```LayoutAnimation.create```这个函数更简单的创建一个```config```，同样可以实现和上图一样的效果。

    startAnimation() {
      LayoutAnimation.configureNext(LayoutAnimation.create(700,
                 LayoutAnimation.Types.spring,
                 LayoutAnimation.Properties.scaleXY));
      this.setState({width: this.state.width + 10, height: this.state.height + 10});
    }

```create```函数接受三个参数：
* duration：动画持续时间。
* type：```create```和```update```时的动画类型，定义在* ```LayoutAnimation.Types```。
* creationProp：```create```时的动画属性，定义在```LayoutAnimation.Properties```。

附```create```的源码：

![create.png](http://upload-images.jianshu.io/upload_images/1787010-0b21ad84c0367634.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
效果图和上图一样，这里不在贴出来了。

实际上，系统已经为我们提供了3个默认的动画，定义在```LayoutAnimation.Presets```中：
* easeInEaseOut：缓入缓出
* linear：线性
* spring：弹性


一个简单的示例：

    startAnimation() {
      LayoutAnimation.configureNext(LayoutAnimation.Presets.linear);
      this.setState({width: this.state.width + 10, height: this.state.height + 10});
    }

具体参数配置，请查看```Presets```源码：

![Presets.png](http://upload-images.jianshu.io/upload_images/1787010-acf189b7816e54e2.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

我们来分析下***LayoutAnimation***的优缺点：

***优点：***

1、效果非常的流畅，而且动画的过程很柔和，丝毫没有生硬的感觉。

2、可以灵活的配置动画参数，基本能满足单个动画的需求。

***缺点：***

1、如果要实现**‘组合顺序’**动画，比如先缩小50%、再向左平移100像素，那么就比较麻烦了，需要监听上一个动画的结束事件，再进行下一个。那么问题来了，```configureNext```第二个参数是可以监听动画结束的，但是只在IOS平台有效！

2、如果动画的效果更为复杂，比如同时执行动画，再顺序执行，对于编程来讲，需要做的事情很多，复杂度也大大提升。

那么如何定制更灵活丰富的动画效果呢，这就需要使用到高级动画API **Animated**。下一篇文章我们会介绍高级动画API—**Animated**的使用，感兴趣的朋友请继续阅读[[React Native]动画-Animated](http://zhuhf.tech/2016/07/07/RN-Animated/)。

**本文的源码地址**：[Demo9](https://github.com/hiphonezhu/RN-Demos/tree/master/Demo9)
