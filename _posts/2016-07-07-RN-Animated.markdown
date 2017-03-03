---
layout:     post
title:      "[React Native]动画-Animated"
subtitle:   ""
date:       2016-07-07 12:00:00
author:     "zhuhf"
header-img: ""
header-mask: 0.3
catalog:    true
tags:
    - RN
    - Animated
---

在上一篇文章中，我们学习了React Native实现动画的几种方式，其中重点介绍了**LayoutAnimation**。文章的末尾也提到，如果你需要更强大的动画功能，就需要使用高级API—**Animated**。

如果你还不了解**LayoutAnimation**，建议先阅读下上一篇文章[[React Native]动画-LayoutAnimation](http://zhuhf.tech/2016/07/05/RN-LayoutAnimation/)，其中的一些概念能让你更好的理解本篇文章的内容。

本篇文章会一步步介绍**Animated**的用法，如果有误之处，欢迎指正~

---
**动画类型：**
* spring：基础的单次弹跳物理模型
* timing：从时间范围映射到渐变的值
* decay：以一个初始速度开始并且逐渐减慢停止

创建动画的参数：
* value：AnimatedValue、AnimatedValueXY（X轴**或**Y轴、X轴**和**Y轴）
* config：SpringAnimationConfig、TimingAnimationConfig、DecayAnimationConfig（动画的参数配置）

**组件类型：**
* Animated.Text
* Animated.Image
* Animated.View：可以用来包裹任意视图
* Animated.createAnimatedComponent()：其它组件（**较少用，用Animated.View包裹可以达到同样的效果**）

让我们来看一个示例：图片透明度2秒内从不透明到全透明，线性变化。

    class Demo8 extends Component {
      // 构造
      constructor(props) {
          super(props);
          // 初始状态
          this.state = {
              fadeOutOpacity: new Animated.Value(0),
          };
      }
      render() {
          return (
            <Animated.View // 可选的基本组件类型: Image, Text, View(可以包裹任意子View)
                style = \{\{flex: 1,alignItems: 'center',justifyContent: 'center',
                        opacity: this.state.fadeOutOpacity\}\}>
                <Image source = \{\{uri: 'http://i.imgur.com/XMKOH81.jpg'\}\}
                    style = \{\{width: 400,height: 400\}\}/>
			</Animated.View >
          );
      }
      startAnimation() {
          this.state.fadeOutOpacity.setValue(1);
          Animated.timing(this.state.fadeOutOpacity, {
              toValue: 0,
              duration: 2000,
              easing: Easing.linear,// 线性的渐变函数
          }).start();
      }
      componentDidMount() {
          this.startAnimation();
      }
    }
    AppRegistry.registerComponent('Demo8', () = >Demo8);

效果图如下：

![opacity.gif](http://upload-images.jianshu.io/upload_images/1787010-6801a13548059008.gif?imageMogr2/auto-orient/strip)

**值类型：**
* AnimatedValue：单个值
* AnimatedValueXY：向量值

多数情况下，**AnimatedValue**可以满足需求（上面的示例），但有些情况下我们可能会需要**AnimatedValueXY**。

比如：我们需要图片沿着X轴和Y轴交叉方向，向右下角移动一小段距离。

    class Demo8 extends Component {
      // 构造
      constructor(props) {
          super(props);
          // 初始状态
          this.state = {
              translateValue: new Animated.ValueXY({x:0, y:0}), // 二维坐标
          };
      }
      render() {
          return (
            <Animated.View // 可选的基本组件类型: Image, Text, View(可以包裹任意子View)
                style = \{\{flex: 1,alignItems:'center',justifyContent: 'center',
                      transform: [  
		                {translateX: this.state.translateValue.x}, // x轴移动
		                {translateY: this.state.translateValue.y} // y轴移动
		              ]
                      \}\}>
                <Image source = \{\{uri: 'http://i.imgur.com/XMKOH81.jpg'\}\}
                    style = \{\{width: 400,height: 400\}\}/>
			</Animated.View >
          );
      }
      startAnimation() {
          this.state.translateValue.setValue({x:0, y:0});
          Animated.decay( // 以一个初始速度开始并且逐渐减慢停止。
			  this.state.translateValue,
			  {
			      velocity: 10, // 起始速度，必填参数。
			      deceleration: 0.8, // 速度衰减比例，默认为0.997。
			  }
          ).start();
      }
      componentDidMount() {
          this.startAnimation();
      }
    }
    AppRegistry.registerComponent('Demo8', () = >Demo8);


其中，```transform```是一个变换数组，常用的有```scale, scaleX, scaleY, translateX, translateY, rotate, rotateX, rotateY, rotateZ```：
```
    ...
    transform: [  // scale, scaleX, scaleY, translateX, translateY, rotate, rotateX, rotateY, rotateZ
	    {scale: this.state.bounceValue},  // 缩放
	    {rotate: this.state.rotateValue.interpolate({ // 旋转，使用插值函数做值映射
		    inputRange: [0, 1],
		    outputRange: ['0deg', '360deg']})},
	    {translateX: this.state.translateValue.x}, // x轴移动
	    {translateY: this.state.translateValue.y}, // y轴移动
	],
    ...
```
**插值函数：**
将输入值范围转换为输出值范围，如下：将```0-1```数值转换为```0deg-360deg```角度，旋转```View```时你会用到

    this.state.rotateValue.interpolate({ // 旋转，使用插值函数做值映射
		    inputRange: [0, 1],
		    outputRange: ['0deg', '360deg']})

**组合动画：**
* parallel：同时执行
* sequence：顺序执行
* stagger：错峰，其实就是插入了delay的parrllel
* delay：组合动画之间的延迟方法，严格来讲，不算是组合动画

让我们来看一个示例：图片首先缩小80%，2秒之后，旋转360度，之后沿着X轴与Y轴交叉方向向右下角移动一段距离，最后消失变成全透明

    startAnimation() {
      this.state.bounceValue.setValue(1.5); // 设置一个较大的初始值
      this.state.rotateValue.setValue(0);
      this.state.translateValue.setValue({x: 0,y: 0});
      this.state.fadeOutOpacity.setValue(1);

      Animated.sequence([
          Animated.sequence([ //  组合动画 parallel（同时执行）、sequence（顺序执行）、stagger（错峰，其实就是插入了delay的parrllel）和delay（延迟）
            Animated.spring( //  基础的单次弹跳物理模型
              this.state.bounceValue, {
                toValue: 0.8,
                friction: 1,// 摩擦力，默认为7.
                tension: 40,// 张力，默认40。
              }),
            Animated.delay(2000), // 配合sequence，延迟2秒
            Animated.timing( // 从时间范围映射到渐变的值。
              this.state.rotateValue, {
                toValue: 1,
                duration: 800,// 动画持续的时间（单位是毫秒），默认为500
                easing: Easing.out(Easing.quad),// 一个用于定义曲线的渐变函数
                delay: 0,// 在一段时间之后开始动画（单位是毫秒），默认为0。
              }),
            Animated.decay( // 以一个初始速度开始并且逐渐减慢停止。  S=vt-（at^2）/2   v=v - at
              this.state.translateValue, {
                velocity: 10,// 起始速度，必填参数。
                deceleration: 0.8,// 速度衰减比例，默认为0.997。
            }),
          ]),
          Animated.timing(this.state.fadeOutOpacity, {
            toValue: 0,
            duration: 2000,
            easing: Easing.linear,// 线性的渐变函数
          })
      ]).start();
    }

效果图如下：

![compose.gif](http://upload-images.jianshu.io/upload_images/1787010-7558a218b271409a.gif?imageMogr2/auto-orient/strip)

**循环执行动画：**
```start```方法可以接受一个函数，通过监听动画结束，再调用```startAnimation```可以重复执行动画，例如：

    startAnimation() {
       this.state.translateValue.setValue({x:0, y:0});
       Animated.decay( // 以一个初始速度开始并且逐渐减慢停止。  S=vt-（at^2）/2   v=v - at
		  this.state.translateValue,
		  {
		     velocity: 10, // 起始速度，必填参数。
		     deceleration: 0.8, // 速度衰减比例，默认为0.997。
	      }
       ).start(() => this.startAnimation());
    }

**监听当前的动画值：**
* addListener(callback)：动画执行过程中的值
* stopAnimation(callback)：动画执行结束时的值

监听**AnimatedValueXY**类型```translateValue```的值变化：

    this.state.translateValue.addListener((value) => {   
        console.log("translateValue=>x:" + value.x + " y:" + value.y);
    });
    this.state.translateValue.stopAnimation((value) => {   
        console.log("translateValue=>x:" + value.x + " y:" + value.y);
    });

监听**AnimatedValue**类型```rotateValue```的值变化：

    this.state.rotateValue.addListener((state) => {   
        console.log("rotateValue=>" + state.value);
    });
    this.state.rotateValue.stopAnimation((state) => {   
        console.log("rotateValue=>" + state.value);
    });

好了，到这里我们把**Animated**的常用方法都介绍了，也给出了代码示例（略微有点多~）。建议大家动手尝试下每个效果，这样，可以理解的更加深刻~

**本文的源码地址**：[Demo8](https://github.com/hiphonezhu/RN-Demos/tree/master/Demo8)
