---
layout:     post
title:      "[React Native]导航器Navigator"
subtitle:   ""
date:       2016-07-11 12:00:00
author:     "zhuhf"
header-img: ""
header-mask: 0.3
catalog:    true
tags:
    - RN
    - Navigator
---
## 前言
官方提供了两种导航器：**Navigator**和**NavigatorIOS**，后者只能在iOS平台使用，而前者是可以兼容Android和iOS平台的，详细介绍请参考[官方介绍](http://reactnative.cn/docs/0.28/navigator-comparison.html#content)。

## 基本使用方式
* initialRoute：初始化路由，指定第一个页面
* configureScene：配置页面切换场景
* renderScene：渲染场景，返回需要显示的页面


常见的一个写法：  

    // ...
    import FirstPage from './FirstPage';
    class Demo10 extends Component {
	  render() {
		return (
			<Navigator
				style=\{\{flex: 1\}\}
                initialRoute= \{\{id: 'FirstPage', component: FirstPage\}\}
                configureScene= {this.configureScene}
                renderScene= {this.renderScene}
			/>
		);
	  }
      configureScene(route, routeStack) {
	    if (route.sceneConfig) { // 有设置场景
			return route.sceneConfig;
        }
        return Navigator.SceneConfigs.PushFromRight; // 默认，右侧弹出
      }
      renderScene(route, navigator) {
        return <route.component {...route.passProps} navigator= {navigator}/>;
	  }
    }
    // ...

```initialRoute、configureScene、renderScene```三个方法都和一个重要的参数```route```有关联。

**route**：表示路由对象的信息，我们看下它包含哪些参数

    export interface Route {
        component?: React.ComponentClass<ViewProperties> // 必要参数，目标页面
        id?: string  // 通常用来设置一个与component匹配的标识
        title?: string // 常用来设置component页面的标题参数
        passProps?: Object; // 用来传递参数

        //anything else
        [key: string]: any

        //Commonly found properties
        backButtonTitle?: string  // 标题栏左边按钮文字
        content?: string
        message?: string;
        index?: number
        onRightButtonPress?: () => void // 标题栏右边按钮点击事件
        rightButtonTitle?: string // 标题栏右边按钮文字
        sceneConfig?: SceneConfig // 切换场景的参数
        wrapperStyle?: any
    }

本篇文章我们会使用到```id、component、passProps、sceneConfig```这四个参数，其他类似于```backButtonTitle、onRightButtonPress```等，这些参数在使用**通用导航栏**的时候会用到。

>实际项目中，这种方式基本不会用，因为业务的复杂等原因会导致**通用导航栏**变得越来越臃肿，不可维护，如果想了解的同学，可以参考[这篇文章](http://www.jianshu.com/p/91fa0f34895e?utm_campaign=haruki&utm_content=note&utm_medium=reader_share&utm_source=qq)。

**Navigator**和其他平台的导航一样，也是使用**“栈”**的形式来维护页面。我们都知道**“栈”**是先进后出的原则，**Navigator**多数API是**“遵循”**这个原则的。

不同的是，**Navigator**还提供了更加灵活的API，使得我们可以从类似**“栈顶”**跳转到**“栈底”**，同时还能保持**“栈底”**以上的页面存在而不销毁，我觉得可以用**“伪栈”**来形容它。

## 常用API
* push(route) - 跳转到新的场景，并且将场景入栈，你可以稍后跳转过去
* pop() - 跳转回去并且卸载掉当前场景
* popToTop() - pop到栈中的第一个场景，卸载掉所有的其他场景。
* popToRoute(route) - pop到路由指定的场景，在整个路由栈中，处于指定场景之后的场景将会被卸载。

* jumpBack() - 跳回之前的路由，当然前提是保留现在的，还可以再跳回来，会给你保留原样。
* jumpForward() - 上一个方法不是调到之前的路由了么，用这个跳回来就好了。
* jumpTo(route) - 跳转到已有的场景并且不卸载。
***备注：与push&popxxx效果类似，只是不会卸载场景***

* replace(route) - 用一个新的路由替换掉当前场景
* replaceAtIndex(route, index) - 替换掉指定序列的路由场景
* replacePrevious(route) - 替换掉之前的场景
***备注：replacexxx：仅仅替换场景，并不会跳转***

* resetTo(route) - 跳转到新的场景，并且重置整个路由栈
* immediatelyResetRouteStack(routeStack) - 用新的路由数组来重置路由栈
* getCurrentRoutes() - 获取当前栈里的路由，也就是push进来，没有pop掉的那些。


为了演示以上API的使用，我们新建三个js文件，```FirstPage.js```、```SecondPage.js```和```ThirdPage.js```。

```FirstPage```作为导航控制器的根页面，请参考文章开头给出的代码示例。

**push&popxxx：**

1、FirstPage->**push**->SecondPage：传递参数***from***

    // FirstPage.js
    _push()
	{
		this.props.navigator.push({
			id: 'SecondPage',
			component: SecondPage,
			passProps: {
				from: 'First Page'
			},
			sceneConfig: Navigator.SceneConfigs.HorizontalSwipeJump
		});
	}

2、SecondPage->**push**->ThirdPage，代码类似就不贴了。然后，ThirdPage->**pop**->SecondPage，返回上一个页面

    // ThirdPage.js
    _pop()
	{
		this.props.navigator.pop();
	}

3、ThirdPage->**popToTop**->FirstPage，返回第一个页面

    // ThirdPage.js
    _popToTop() {
		this.props.navigator.popToTop();
	}

4、ThirdPage->**popToRoute**->FirstPage，返回第一个页面

    // ThirdPage.js
    _popToRouteById(id) {
		var destRoute;
		this.props.navigator.getCurrentRoutes().map((route, i) => {
			if (route.id === id) {
				destRoute = route;
			}
		});
		// 携带参数
		destRoute.passProps = {from: 'Third Page'};
		this.props.navigator.popToRoute(destRoute);
	}

效果图如下：
![push&popxxx.gif](http://upload-images.jianshu.io/upload_images/1787010-edd403c6d616b4e1.gif?imageMogr2/auto-orient/strip)

**jumpxxx：**

1、SecondPage->**jumpBack**->FirstPage：返回上一个场景，并且不卸载当前场景

    // SecondPage.js
    _jumpBack() {
		this.props.navigator.jumpBack();
	}

2、FirstPage->**jumpForward**->SecondPage：与jumpBack配套使用，返回触发jumpBack的场景

    // FirstPage.js
    _jumpForward()
	{
		this.props.navigator.jumpForward();
	}

效果图如下：

![jumpxxx.gif](http://upload-images.jianshu.io/upload_images/1787010-1f8c764063495503.gif?imageMogr2/auto-orient/strip)
我们打印下```FirstPage```和```SecondPage```的生命周期函数```componentDidMount```和```componentWillUnmount```。

    // FirstPage.js
    componentDidMount() {
		console.log("FirstPage: componentDidMount");
	}

	componentWillUnmount() {
		console.log("FirstPage: componentWillUnmount");
	}

    // SecondPage.js
    componentDidMount() {
		console.log("SecondPage: componentDidMount");
	}

	componentWillUnmount() {
		console.log("SecondPage: componentWillUnmount");
	}

![lifecircle.png](http://upload-images.jianshu.io/upload_images/1787010-df80992dc0213bb5.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
我们看到，在来回的跳转过程中```SecondPage```并没有被卸载。

更有趣的是，从```SecondPage``` ```jumpBack```之后。在```FirstPage```页面，如果点击```push```，你会发现旧的```SecondPage```会被卸载，然后会创建一个新的```SecondPage```。console如下：

![lifecircle2.png](http://upload-images.jianshu.io/upload_images/1787010-37dbc1d1ea3336c9.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
3、SecondPage->**jumpTo**->FirstPage：跳转到一个已存在的场景，并且不卸载当前场景

    // SecondPage.js
    _jumpToById(id) {
		var destRoute;
		this.props.navigator.getCurrentRoutes().map((route, i) => {
			if (route.id === id) {
				destRoute = route;
			}
		});
		// 携带参数
		destRoute.passProps = {from: 'Second Page'};
		this.props.navigator.jumpTo(destRoute);
	}

**replacexxx：**

新建一个场景```FourthPage.js```。当前“栈”里面的场景为```FirstPage->SecondPage->ThirdPage```。
1、ThirdPage ->***replacePrevious***：在```ThirdPage ```页面替换上一个场景（也就是```SecondPage```）为```FourthPage```。

    // ThirdPage.js
    _replacePrevious()
	{
		this.props.navigator.replacePrevious({
			id: 'FourthPage',
			component: FourthPage,
		});
	}

此时，“栈”里面的场景为```FirstPage->FourthPage->ThirdPage```。

2、```replace(route)```和```replaceAtIndex(route, index)```与```replacePrevious```非常类似，只是替换的场景不同而已，这里不做讲解，大家自己测试一下就知道效果了。
* replace(route) - 用一个新的路由替换掉当前场景
* replaceAtIndex(route, index) - 替换掉指定序列的路由场景

需要注意的是，***replacexxx：仅仅替换场景，并不会跳转***。

**resetTo：**

跳转到新的场景，并且重置整个路由栈。

想象这样一个场景，我们有个App，用户登录之后，可能已经进入三级页面，此时用户希望切换用户重新登录，这时候我们就需要跳转到登录页面，并且清空其他所有页面。

此时，就需要使用到***resetTo***。

我们在ThirdPage->***resetTo***->FourthPage，然后在```FourthPage```使用```pop```，看下是否还有其他页面存在。

    // ThirdPage.js
    _resetTo()
	{
		this.props.navigator.resetTo({
			id: 'FourthPage',
			component: FourthPage,
		});
	}
    // FourthPage.js
    _pop()
	{
		this.props.navigator.pop();
	}

效果图如下：

![resetTo ](http://upload-images.jianshu.io/upload_images/1787010-83b868d5fe13db94.gif?imageMogr2/auto-orient/strip)

我们看到整个路由**“栈”**被重新初始化，```FourthPage```成为**“栈”**的根页面。

**immediatelyResetRouteStack(routeStack)：**

用新的路由数组来重置路由栈，与**resetTo**类似，只不过**resetTo**参数为一个对象，而**immediatelyResetRouteStack(routeStack)**为一个数组，感兴趣的同学可以自己尝试一下效果。


**本文的源码地址**：[Demo10](https://github.com/hiphonezhu/RN-Demos/tree/master/Demo10)
