---
layout:     post
title:      "Android O 新特性 - Background Execution Limits"
subtitle:   ""
date:       2017-03-29 12:00:00
author:     "zhuhf"
header-img: ""
header-mask: 0.3
catalog:    true
tags:
    - Android
    - Android O
---



为了节省系统资源（内存、电量、流量等），提升手机流畅度和用户体验，Android O 对程序“后台运行”的限制变得更加严格，具体体现在两个方面：

 - 限制后台服务
当我们的程序处于“空闲”状态，“后台服务”会被限制执行，但是这不影响“前台服务”。
 - 限制广播
程序不能在Manifest中注册“限制性”广播，但仍然可以动态的去注册。

> 注意：如果 targetSdkVersion <= 25，则不受以上两个限制。

限制后台服务
======

系统将一个 App 分为“前台”和“后台”两种状态。当满足下面任意一个条件，则认为是“前台” App：

 - 拥有可见的 Activity，状态为 started 或 paused
 - 拥有一个“foreground Service”
 - 有一个处于前台的 App “连接”到了当前 App，一般通过“bind service” 或 “content provider”

如果以上三个条件没有一个满足，则认为它是“后台” App。

当 App 在“前台”，它可以随意的创建和启动一个“前台”或者“后台” Service。当 App 切换到“后台”后，它将有一小段时间仍然可以使用 Service，在这之后 App 便会处于“空闲”状态。这个时候，系统会停止 App 的所有“后台” Service，这就好像 App 自己调用了 `Service.stopSelf()` 方法。

在一些特定情况下，“后台” App 在一小段时间内也可以没有限制的使用 Service。这种“特定”情况，包含以下几种：

 - 处理 FCM 消息
 这是谷歌自己的消息推送系统，不过在国内无法使用。
 - 接收到广播消息，比如 SMS 信息等
 - 执行 notification 的 PendingIntent

**那么，我们如何解决 App “后台”运行时的诸多限制呢？**

谷歌提供了两种建议的方案：

 - 使用 **JobScheduler** 替代 Service 来执行“周期性”任务
关于 JobScheduler 的使用请参考[这里](http://blog.csdn.net/bboyfeiyu/article/details/44809395)
- 使用“前台” Service
Android O 之前，创建“前台” Service 的步骤：

 1. 先使用 `startService()` 方法启动一个 Service；
 
 2. 然后使用 `Service.startForeground()` 方法将 Service 设置为“前台” 。

这样在通知栏就会显示一个 notification ，表明 Service 是“前台”服务。

然而，Android O 建议我们使用 `NotificationManager.startServiceInForeground()` 方法来创建一个“前台” Service。这样做的好处是，当 App 处于“后台”时，我们是无法使用 `startService()` 来创建 Service，但却可以在 **广播** 和 **JobScheduler** 中使用 `NotificationManager.startServiceInForeground()` 来创建 Service。

限制广播
====
在 Android 7.0 (API level 24)  之后，谷歌已经对广播做了很多限制，具体表现为：

 - targetSdkVersion >= 24，Manifest 中声明 `CONNECTIVITY_ACTION` 的广播将无法收到通知。但如果是在代码中注册，则不受此影响。
 - 不能发送和接收 `ACTION_NEW_PICTURE` 和 `ACTION_NEW_VIDEO` 这两种类型的广播，这个影响所有的 App，与声明的 targetSdkVersion 无关。
 
而 Android O 让广播的限制变得更加严格，它将广播分为“**implicit**”和“**explicit**”两种类型。

**举个例子：**

 -  如果有 App 安装了新版本，那么`ACTION_PACKAGE_REPLACED`将会发送给所有注册了此广播的 App，而不是某一个指定的 App，所以我们称它为 “implicit”。
 -  而 `ACTION_MY_PACKAGE_REPLACED`只会发送给指定的 App，所以称它为“explicit”。
 
**具体限制表现为：**

 - Android O 不允许在 Manifest 中注册“implicit”类型的广播，但可以注册“explicit”类型的广播。
 - 仍然可以在代码中使用 `Context.registerReceiver()` 注册“implicit”和“explicit”类型的广播。
 
**“explicit” 类型的广播目前有以下几种：**

 - ACTION_LOCKED_BOOT_COMPLETED, 
 ACTION_BOOT_COMPLETED
 - ACTION_USER_INITIALIZE
 - ACTION_TIMEZONE_CHANGED
 - CTION_LOCALE_CHANGED
 - ACTION_USB_ACCESSORY_ATTACHED, ACTION_USB_ACCESSORY_DETACHED, 
ACTION_USB_DEVICE_ATTACHED, 
ACTION_USB_DEVICE_DETACHED
 - ACTION_HEADSET_PLUG
 - ACTION_CONNECTION_STATE_CHANGED, ACTION_CONNECTION_STATE_CHANGED
 - ACTION_CARRIER_CONFIG_CHANGED
 - ACTION_DEVICE_STORAGE_LOW, 
 ACTION_DEVICE_STORAGE_OK
 - LOGIN_ACCOUNTS_CHANGED_ACTION
 - ACTION_PACKAGE_DATA_CLEARED
 - ACTION_PACKAGE_FULLY_REMOVED
 - ACTION_NEW_OUTGOING_CALL
 - ACTION_DEVICE_OWNER_CHANGED
 - ACTION_EVENT_REMINDER
 
> 更多详细信息请参考[这里](https://developer.android.google.cn/preview/features/background-broadcasts.html)。

**解决方案：**

比如你有一个应用在收到系统“**充电广播**”的时候执行一些清理动作， 然而 `ACTION_POWER_CONNECTED` 属于“implicit”类型广播，所以你无法在 Android O 中使用。

你可以使用以下两个建议方案：

 - 使用 `JobScheduler` 来替代“特定”的广播，比如手机充电广播： `ACTION_POWER_CONNECTED`
 - 使用 `Context.registerReceiver()` 注册“implicit”广播，而不是在 Manifest 中注册
 
以上就是 Android O 对“后台”运行诸多限制的介绍，同时提供了一些建议性解决方案，希望能够帮到大家~