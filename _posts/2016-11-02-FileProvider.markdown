---
layout:     post
title:      "Android 7.0适配-应用之间共享文件(FileProvider)"
subtitle:   ""
date:       2016-11-02 12:00:00
author:     "zhuhf"
header-img: ""
header-mask: 0.3
catalog:    true
tags:
    - Android 7.0
    - FileProvider
---



## 一、前言
Android 7.0强制启用了被称作 [StrictMode](https://developer.android.com/reference/android/os/StrictMode.html)的策略，带来的影响就是你的App对外无法暴露``file://``类型的**URI**了。

如果你使用**Intent**携带这样的**URI**去打开外部App(比如：打开系统相机拍照)，那么会抛出**FileUriExposedException**异常。

官方给出解决这个问题的方案，就是使用**FileProvider**：

![FileProvider.png](http://upload-images.jianshu.io/upload_images/1787010-151a156af9a127f4.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

我们来看一段代码：

    String cachePath = getApplicationContext().getExternalCacheDir().getPath();
    File picFile = new File(cachePath, "test.jpg");
    Uri picUri = Uri.fromFile(picFile);
    Intent intent = new Intent(MediaStore.ACTION_IMAGE_CAPTURE);
    intent.putExtra(MediaStore.EXTRA_OUTPUT, picUri);
    startActivityForResult(intent, 100);

这是常见的打开系统相机拍照的代码，拍照成功后，照片会存储在``picFile``文件中。

这段代码在Android 7.0之前是没有任何问题的(奇葩情况忽略~)，但是如果你尝试在7.0的系统上运行(可以用模拟器测试，我也没真机~)，会抛出文章开头提到的**FileUriExposedException**异常。

既然官方推荐使用**FileProvider**来解决此问题，我们就来看下如何使用吧~

## 二、使用FileProvider
**FileProvider**使用大概分为以下几个步骤：
1. manifest中申明FileProvider
2. res/xml中定义对外暴露的文件夹路径
3. 生成content://类型的Uri
4. 给Uri授予临时权限
5. 使用Intent传递Uri

我们分别看下这几个步骤的具体实现吧

**1.manifest中申明FileProvider：**

    <manifest>
      ...
      <application>
        ...
        <provider
            android:name="android.support.v4.content.FileProvider"
            android:authorities="com.mydomain.fileprovider"
            android:exported="false"
            android:grantUriPermissions="true">
            ...
        </provider>
        ...
      </application>
    </manifest>

**android:name：**``provider``你可以使用``v4``包提供的FileProvider，或者自定义您自己的，只需要在``name``申明就好了，一般使用系统的就足够了。
**android:authorities：**类似``schema``，命名空间之类，后面会用到。
**android:exported：**``false``表示我们的``provider``不需要对外开放。
**android:grantUriPermissions：**申明为``true``，你才能获取临时共享权限。

**2. res/xml中定义对外暴露的文件夹路径：**
新建``file_paths.xml``，文件名随便起，后面会引用到。

    <paths xmlns:android="http://schemas.android.com/apk/res/android">
      <files-path name="my_images" path="images"/>
    </paths>

**name：**一个引用字符串。
**path：**文件夹“相对路径”，完整路径取决于当前的标签类型。
>path可以为空，表示指定目录下的所有文件、文件夹都可以被共享。



``<paths>``这个元素内可以包含以下**一个或多个**，具体如下：

    <files-path name="name" path="path" />

物理路径相当于[Context.getFilesDir()](https://developer.android.com/reference/android/content/Context.html#getFilesDir()) + /path/。

    <cache-path name="name" path="path" />

物理路径相当于[Context.getCacheDir()](https://developer.android.com/reference/android/content/Context.html#getFilesDir()) + /path/。

    <external-path name="name" path="path" />

物理路径相当于[Environment.getExternalStorageDirectory()](https://developer.android.com/reference/android/os/Environment.html#getExternalStorageDirectory()) + /path/。

    <external-files-path name="name" path="path" />

物理路径相当于**Context.getExternalFilesDir(String) **+ /path/。

    <external-cache-path name="name" path="path" />

物理路径相当于[Context.getExternalCacheDir()](https://developer.android.com/reference/android/content/Context.html#getExternalCacheDir()) + /path/。
> 注意：external-cache-path在support-v4:24.0.0这个版本并未支持，直到support-v4:25.0.0才支持，最近适配才发现这个坑!!!

**番外：**以上是官方提供的几种path类型，不过如果你想使用外置**SD**卡，可以用这个：

    <root-path name="name" path="path" />

物理路径相当于/path/。

这个官方文档并没有给出，我们查看源码可以发现：

![root.png](http://upload-images.jianshu.io/upload_images/1787010-d09c0c1c55daa5f6.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

编写好``file_paths.xml``，我们在``manifest``中的``provider``这样使用：

    <provider
      android:name="android.support.v4.content.FileProvider"
      android:authorities="com.mydomain.fileprovider"
      android:exported="false"
      android:grantUriPermissions="true">
      <meta-data
          android:name="android.support.FILE_PROVIDER_PATHS"
          android:resource="@xml/file_paths" />
    </provider>

**3.生成content://类型的Uri**
我们通常通过**File**生成**Uri**的代码是这样：

    File picFile = xxx;
    Uri picUri = Uri.fromFile(picFile);

这样生成的**Uri**，路径格式为``file://xxx``。前面我们也说了这种**Uri**是无法在App之间共享的，我们需要生成``content://xxx``类型的**Uri**，方法就是通过``Context.getUriForFile``来实现：

    File imagePath = new File(Context.getFilesDir(), "images");
    File newFile = new File(imagePath, "default_image.jpg");
    Uri contentUri = getUriForFile(getContext(),
                     "com.mydomain.fileprovider", newFile);

**imagePath：**使用的路径需要和你在``file_paths.xml``申明的其中一个符合(**或者子文件夹："images/work"**)。当然，你可以申明**N**个你需要共享的路径：

    <paths xmlns:android="http://schemas.android.com/apk/res/android">   
      <files-path name="my_images" path="images"/>   
      <files-path name="my_docs" path="docs"/>
      <external-files-path name="my_video" path="video" />
      //...
    </paths>

**getUriForFile：**第一个参数是**Context**；第二个参数，就是我们之前在``manifest#provider``中定义的``android:authorities``属性的值；第三个参数是**File**。

**4.给Uri授予临时权限**

    intent.addFlags(Intent.FLAG_GRANT_READ_URI_PERMISSION
                   | Intent.FLAG_GRANT_WRITE_URI_PERMISSION);

**FLAG_GRANT_READ_URI_PERMISSION：**表示读取权限；
**FLAG_GRANT_WRITE_URI_PERMISSION：**表示写入权限。

你可以同时或单独使用这两个权限，视你的需求而定。

**5.使用Intent传递Uri**
以开头的拍照代码作为示例，需要这样改写：

    // 重新构造Uri：content://
    File imagePath = new File(Context.getFilesDir(), "images");
    if (!imagePath.exists()){imagePath.mkdirs();}
    File newFile = new File(imagePath, "default_image.jpg");
    Uri contentUri = getUriForFile(getContext(),
                     "com.mydomain.fileprovider", newFile);
    Intent intent = new Intent(MediaStore.ACTION_IMAGE_CAPTURE);
    intent.putExtra(MediaStore.EXTRA_OUTPUT, contentUri);
    // 授予目录临时共享权限
    intent.addFlags(Intent.FLAG_GRANT_READ_URI_PERMISSION
                   | Intent.FLAG_GRANT_WRITE_URI_PERMISSION);
    startActivityForResult(intent, 100);
    
当然，6.0以上系统需要动态申请权限，这个不在本篇文章讨论的范围。

[本文源码](https://github.com/hiphonezhu/Android-Demos/tree/master/FileProviderDemo)
