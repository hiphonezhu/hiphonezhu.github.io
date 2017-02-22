---
layout:     post
title:      "Android轻量级路由框架LiteRouter"
subtitle:   ""
date:       2016-10-24 12:00:00
author:     "zhuhf"
header-img: ""
header-mask: 0.3
catalog:    true
tags:
    - Android
    - Router
---


## 前言
开始之前，我们介绍一下什么是**“路由”**？

路由这个概念来自于Web前端开发，引用知乎网友的[解答](https://www.zhihu.com/question/46767015?sort=created)：
> 不同的请求地址会交给路由处理来转发给相应的控制器处理，所以说路由就可以在转发前修改转发地址，你可以在这上面大作文章。

简单的概括：路由是一个框架，可以控制、转发对页面的跳转，并在跳转之前做任何你想要的处理。

那么，Android中为何要引入一个Web中才有的路由概念？

如果你用过一些路由框架，比如[Router](https://github.com/yjfnypeu/Router/blob/master/README-CN.md)、[AndRoute](https://github.com/campusappcn/AndRouter)、[ActivityRouter](https://github.com/mzule/ActivityRouter)，它们和Web中的路由框架强调的思想很类似，注重动态跳转（比如服务器下发跳转路径）、统一转发等。

而**LiteRouter**关注的有些许不同：
> 我们在团队的开发过程中，可能会遇到一些“航母级”的App，涉及到很多业务线，N个团队的合作开发，这时候如果还是“单”App的开发模式，会遇到各个业务代码冗杂、编译时间几何级上涨、发布版本需要全量测试等问题。

为了解决这个问题，出现了很多关于**组件化开发**的思想：
>各个业务线团队专注自己的开发，在开发期间可以看做一个单独的App（独立开发，独立测试），发布时又会作为了个library，被引入到最终的App中。

这里会涉及到很多问题，比如公共资源、公共库的设计、跨业务的界面跳转等。关于组件化开发，可以参考这个文章：[Android业务组件化开发实践](https://github.com/yjfnypeu/Router/blob/master/README-CN.md)。

**LiteRouter**关注的正是如何在各个业务独立的情况下，实现跨业务界面跳转这一问题：
> Android中通常的界面跳转指的是Activity控制器的处理，我们知道一般情况下需要知道要跳转的目标Activity，以及传递的参数内容等，这个与组件化开发思想中，业务线独立开发显然是违背的。

那么我们如何才能解决此问题呢？

前几天写[Retrofit2源码分析](http://www.jianshu.com/p/084137ce2066)这篇文章的时候，突然给了我灵感，App端的开发和后台的开发不就是独立的？那么，它们是如何互相“独立”最终又互相配合的呢？

显然是依靠我们的接口设计规约~ 那么如果我们也能像Retrofit那样，定义好Activity跳转的接口方法，然后调用此方法就能实现跳转，这岂不是就能解决我们的问题，于是**LiteRouter**诞生了~

## LiteRouter基本使用

#### Step1：定义Activity跳转的接口

    public interface IntentService {
      @ClassName("com.hiphonezhu.test.demo.ActivityDemo2")
      @RequestCode(100)
      void intent2ActivityDemo2(@Key("platform") String platform, @Key("year") int year);
    }

注解介绍：
* @ClassName：要跳转的``Activity``的完整路径。
* @RequestCode：需要返回值，即``startActivityForResult``，如果不添加，则使用``startActivity``。
* @Key：``Intent``传递的参数的``key``（支持``Intent``可传递的所有格式）。

#### Step2：创建服务对象

    LiteRouter liteRouter = new LiteRouter.Builder().build();
    IntentService intentService = liteRouter.create(IntentService.class, ActivityDemo4.this);

``ActivityDemo4.this``为当前的``Activity``。

#### Step3：Activity跳转

    intentService.intent2ActivityDemo2("android", 2016);

有了这样一个约束，就好比一份接口设计文档，各个业务方之间可以根据需求，协商好跨业务之间的``Activity``跳转以及参数传递的规范问题。

不管对于调用方，还是最终跳转方，都可以根据这个接口定义，保持业务线的独立开发。

这份接口定义可以放在公共的库，便于业务需求的随时变更，只要接口定义方及时通知调用方即可，这和App与后台接口的交互的开发方式非常类似。

## 更多用法
* Activity Flag设置
* 转场动画
* 其他原生的Intent或Activity的用法

  Step1：定义方法的返回类型为**IntentWrapper**

      public interface IntentService {
        @ClassName("com.hiphonezhu.test.demo.ActivityDemo2")     
        @RequestCode(100)
        IntentWrapper intent2ActivityDemo2Raw(@Key("platform") String platform, @Key("year") int year);
      }

  Step2：

      IntentWrapper intentWrapper = intentService.intent2ActivityDemo2Raw("android", 2016);
      // Intent intent = intentWrapper.getIntent();  // 原始intent，可以做任何处理
      // 添加flags
      intentWrapper.addFlags(Intent.FLAG_ACTIVITY_CLEAR_TASK | Intent.FLAG_ACTIVITY_NEW_TASK);
      // 启动Activity
      intentWrapper.start();

* 拦截器支持
```
  LiteRouter liteRouter = new LiteRouter.Builder().interceptor(new Interceptor() {
     @Override
     public boolean intercept(IntentWrapper intentWrapper) {
         return false;
     }
  }).build();
```
**intercept**方法返回false表示不做拦截，true表示拦截跳转。
>这里，可以做全局统一的处理，比如用户未登录，你可以使用``intentWrapper#setClassName``方法，修改为登录的Activity，强制用户去登陆。


**LiteRouter**的原理和**Retrofit**非常一致，最重要的都是通过动态代理来实现接口的方法，这里不做过多介绍，感兴趣的同学可以看下源码（代码行很少~）。

源码：[LiteRouter](https://github.com/hiphonezhu/Android-Demos/tree/master/LiteRouter)
