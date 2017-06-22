---
layout:     post
title:      "《从0到1：实现 Android 编译时注解》"
subtitle:   ""
date:       2017-06-22 09:41:00
author:     "zhuhf"
header-img: ""
header-mask: 0.3
catalog:    true
tags:
    - Annotation
---



前言
--

我们经常使用的一些第三方框架，比如：[butterknife](https://github.com/JakeWharton/butterknife)，通过一行注解就可以实现`View` 的“自动赋值”。

那么，这其中的原理是什么呢？

为了带大家更好的深入了解，本文将打造一个简单的 Demo，来说明这其中的原理。
>Demo 虽然简单，但是完全按照 [butterknife](https://github.com/JakeWharton/butterknife) 实现的方式和原理打造。

实现思路
--

我们先看 Demo 的效果：

    public class MainActivity extends AppCompatActivity {
        // 被注解的 View
        @BindView(R.id.tv)
        TextView tv;
        @Override
        protected void onCreate(Bundle savedInstanceState) {
            super.onCreate(savedInstanceState);
            setContentView(R.layout.activity_main);
            // 为 tv 赋值
            InjectHelper.inject(this);
            tv.setText("I am injected");
        }
    }
    
代码非常简单，分为两步：

1. 通过 `@BindView(R.id.tv)` 指定被注解 View 的 Id 值；
2. `InjectHelper.inject(this);`，具体为 tv 赋值的方法。

`@BindView` 注解没什么好解释的，我们重点看下 `InjectHelper.inject(this);` 的实现：

    public class InjectHelper {
        public static void inject(Activity host) {
                // 1、
            String classFullName = host.getClass().getName() + "$$ViewInjector";
            try {
                // 2、
                Class proxy = Class.forName(classFullName);
                // 3、
                Constructor constructor = proxy.getConstructor(host.getClass())
                // 4、
                constructor.newInstance(host);
            } catch (Exception e) {
                e.printStackTrace();
            }
        }
    }

 1. 获得 View 所在 Activity 的类路径，然后拼接一个字符串“$$ViewInjector”。这个是编译时动态生成的 Class 的完整路径，也就是我们需要实现的，同时也是最关键的部分；
 2. 根据 Class 路径，使用 `Class.forName(classFullName)` 生成 Class 对象；
 3. 得到 Class 的构造函数 constructor 对象；
 4. 使用 `constructor.newInstance(host)` new 出一个对象，这会执行对象的构造方法，方法内部是我们为 MainActivity 的 tv 赋值的地方。

我们先看一个生成好的 “XXX$$ViewInjector” 示例：

    public class MainActivity$$ViewInjector {
    public MainActivity$$ViewInjector(MainActivity activity) {
            activity.tv = (TextView)activity.findViewById(2131427422);
        }
    }
 
到这里，我们大概知道了最关键的地方，就是如何生成 “XXX$$ViewInjector” 这个类了。

APT 实现方案
--------

APT 是一种处理注解的工具，它对源代码文件进行检测找出其中的 Annotation，再根据注解自动生成代码。

实现方案，分为两种：

 - [android-apt](https://bitbucket.org/hvisser/android-apt)，个人开发者提供，现在已经停止维护，作者推荐大家使用官方提供的解决方案。
 - Android Gradle 插件：annotationProcessor，由官方提供支持。
 
>如何由 [android-apt](https://bitbucket.org/hvisser/android-apt) 切换到 annotationProcessor，可以参考[这里](https://bitbucket.org/hvisser/android-apt)。

annotationProcessor 配置起来比较简单，另外由于是官方支持的，所以我们选择第二种方案。

实现步骤
----

**第一步：定义注解`@BindView**

    @Target(ElementType.FIELD)
    @Retention(RetentionPolicy.CLASS)
    public @interface BindView {
        int value();
    }
    
没什么好解释的~

**第二步：实现 AbstractProcessor**

***1、新建一个 Java Library，引入两个第三方库：***

    dependencies {
        // ...
        compile 'com.google.auto.service:auto-service:1.0-rc2'
        compile 'com.squareup:javapoet:1.7.0'
    }
    
- [auto-service](https://github.com/google/auto/tree/master/service)：Google 公司出品，用于自动为 JAVA Processor 生成 META-INF 信息。

    如果你定义了一个 Processor：

        package foo.bar;
        
        import javax.annotation.processing.Processor;
        
        @AutoService(Processor.class)
        final class MyProcessor implements Processor {
            // …
        }
  [auto-service](https://github.com/google/auto/tree/master/service) 会在编译目录生成一个文件，路径是：META-INF/services/javax.annotation.processing.Processor，文件内容为：
  
        foo.bar.MyProcessor
        
- [javapoet](https://github.com/square/javapoet)：大名鼎鼎的 squareup 公司出品，封装了一套生成 .java 源文件的 API。

    以 HelloWorld 类为例:

        package com.example.helloworld;

        public final class HelloWorld {
            public static void main(String[] args) {
               System.out.println("Hello, JavaPoet!");
            }
        }
    
    上面的代码就是使用javapoet用下面的代码进行生成的：

        MethodSpec main = MethodSpec.methodBuilder("main")
            .addModifiers(Modifier.PUBLIC, Modifier.STATIC)
            .returns(void.class)
            .addParameter(String[].class, "args")
            .addStatement("$T.out.println($S)", System.class, "Hello, JavaPoet!")
            .build();

        TypeSpec helloWorld = TypeSpec.classBuilder("HelloWorld")
            .addModifiers(Modifier.PUBLIC, Modifier.FINAL)
            .addMethod(main)
            .build();

        JavaFile javaFile = JavaFile.builder("com.example.helloworld", helloWorld)
            .build();

        javaFile.writeTo(System.out);
    
    更多关于 [javapoet](https://github.com/square/javapoet) 的使用，可以参考[这里](http://blog.csdn.net/crazy1235/article/details/51876192)。
    
***2、继承 AbstractProcessor，配置相关信息：***

    @AutoService(Processor.class)
    @SupportedAnnotationTypes({"com.example.BindView"})
    @SupportedSourceVersion(SourceVersion.RELEASE_7)
    public class ViewInjectProcessor extends AbstractProcessor {
    
        @Override
        public boolean process(Set<? extends TypeElement> set, RoundEnvironment roundEnvironment) {
            //...
        }
    }   

- @AutoService(Processor.class)，生成 META-INF 信息；
- @SupportedAnnotationTypes({"com.example.BindView"})，声明 Processor 处理的注解，注意这是一个数组，表示可以处理多个注解；
- @SupportedSourceVersion(SourceVersion.RELEASE_7)，声明支持的源码版本

补充说明一下，@SupportedAnnotationTypes 和 @SupportedSourceVersion 必须声明，否则会报错。具体原因看大家看一下源码就明白了，这里不做过多解释。

除了注解方式，你也可以通过重写下面两个函数实现：

    @AutoService(Processor.class)
    public class ViewInjectProcessor extends AbstractProcessor {
    
        @Override
        public Set<String> getSupportedAnnotationTypes() {
            Set<String> annotationTypes = new HashSet<>();
            annotationTypes.add("com.example.BindView");
            return annotationTypes;
        }

        @Override
        public SourceVersion getSupportedSourceVersion() {
            return SourceVersion.RELEASE_7;
        }
    }

***3、实现 AbstractProcessor 的 `process()` 方法：***

    @Override
    public boolean process(Set<? extends TypeElement> set, RoundEnvironment roundEnvironment) {
        // 1、
        collectInfo(roundEnvironment);
        // 2、
        writeToFile();
        return true;
    }
    
`process()` 方法的实现，分为两个步骤：

 1. 收集 Class 内的所有被 `@BindView` 注解的成员变量；
 2. 根据上一步收集的内容，生成 .java 源文件。
 

为此，我们声明了两个 Map，用于保存 `collectInfo()` 收集的相关信息，Map 的 key 为类的全路径：

    // 存放同一个Class下的所有注解信息
    Map<String, List<VariableInfo>> classMap = new HashMap<>();
    // 存放Class对应的信息：TypeElement
    Map<String, TypeElement> classTypeElement = new HashMap<>();
    
VariableInfo 是一个简单的类，用于保存被注解 View 对应的一些信息：

    public class VariableInfo {
        // 被注解 View 的 Id 值
        int viewId;
        // 被注解 View 的信息：变量名称、类型
        VariableElement variableElement;
        
        // ...
    }
    
***4、实现 `collectInfo()` 方法：***

    void collectInfo(RoundEnvironment roundEnvironment) {
        classMap.clear();
        classTypeElement.clear();

        Set<? extends Element> elements = roundEnvironment.getElementsAnnotatedWith(BindView.class);
        for (Element element : elements) {
            // 获取 BindView 注解的值
            int viewId = element.getAnnotation(BindView.class).value();

            // 代表被注解的元素
            VariableElement variableElement = (VariableElement) element;

            // 备注解元素所在的Class
            TypeElement typeElement = (TypeElement) variableElement.getEnclosingElement();
            // Class的完整路径
            String classFullName = typeElement.getQualifiedName().toString();

            // 收集Class中所有被注解的元素
            List<VariableInfo> variableList = classMap.get(classFullName);
            if (variableList == null) {
                variableList = new ArrayList<>();
                classMap.put(classFullName, variableList);

                // 保存Class对应要素（名称、完整路径等）
                classTypeElement.put(classFullName, typeElement);
            }
            VariableInfo variableInfo = new VariableInfo();
            variableInfo.setVariableElement(variableElement);
            variableInfo.setViewId(viewId);
            variableList.add(variableInfo);
        }
    }
    
代码的注释已经很完整，这里不再说明了。

这里提一下 Element 这个元素，它的子类我们用到了以下两个：

    Element
    - VariableElement：代表变量
    - TypeElement：代表 class
    
***5、实现 `writeToFile()` 方法：***

    void writeToFile() {
        try {
            for (String classFullName : classMap.keySet()) {
                TypeElement typeElement = classTypeElement.get(classFullName);

                // 使用构造函数绑定数据
                MethodSpec.Builder constructor = MethodSpec.constructorBuilder()
                        .addModifiers(Modifier.PUBLIC)
                        .addParameter(ParameterSpec.builder(TypeName.get(typeElement.asType()), "activity").build());
                List<VariableInfo> variableList = classMap.get(classFullName);
                for (VariableInfo variableInfo : variableList) {
                    VariableElement variableElement = variableInfo.getVariableElement();
                    // 变量名称(比如：TextView tv 的 tv)
                    String variableName = variableElement.getSimpleName().toString();
                    // 变量类型的完整类路径（比如：android.widget.TextView）
                    String variableFullName = variableElement.asType().toString();
                    // 在构造方法中增加赋值语句，例如：activity.tv = (android.widget.TextView)activity.findViewById(215334);
                    constructor.addStatement("activity.$L=($L)activity.findViewById($L)", variableName, variableFullName, variableInfo.getViewId());
                }

                // 构建Class
                TypeSpec typeSpec = TypeSpec.classBuilder(typeElement.getSimpleName() + "$$ViewInjector")
                        .addModifiers(Modifier.PUBLIC)
                        .addMethod(constructor.build())
                        .build();

                // 与目标Class放在同一个包下，解决Class属性的可访问性
                String packageFullName = elementUtils.getPackageOf(typeElement).getQualifiedName().toString();
                JavaFile javaFile = JavaFile.builder(packageFullName, typeSpec)
                        .build();
                // 生成class文件
                javaFile.writeTo(filer);
            }
        } catch (Exception ex) {
            ex.printStackTrace();
        }
    }
    
>这段代码主要就是通过 [javapoet](https://github.com/square/javapoet) 来生成 .java 源文件，大家如果感觉陌生，建议先看一下 [javapoet](https://github.com/square/javapoet) 的使用，参考文章： [JavaPoet的基本使用](http://blog.csdn.net/crazy1235/article/details/51876192)。

>当然，你完全可以通过拼接字符串来生成 .java 源文件的内容。

还记得文章开头提到的 “XXX$$ViewInjector” 吗？`writeToFile()` 方法，就是为了生成这个 .java 源文件的。

**第三步：使用 annotationProcessor**

在 app 的 build.gradle 文件中，使用 APT：

    dependencies {
        // ...
        annotationProcessor project(':lib-compiler')
    }

lib-compiler：为第二步新建的 Java Library。

**第四步：Activity 中使用 @BindView**

文章开始已经演示过相关代码了，这里不再贴了。

一共分为以下两步：

 1. `@BindView` 注解相关 View；
 2. 调用 `InjectHelper.inject(this)` 方法。
 
然后，你尝试去编译项目，会发现 APT 为我们自动生成了 XXX$$ViewInjector.class 文件。
> 你可以在 app/build/intermediates/classes/debug(release、其他 buildType) 下对应的包中找到。

APT 开启 debug 模式
---

AS 中 debug Java 源代码很简单，但是如果你想调试 APT 代码（AbstractProcessor），就必须做一些配置了。

**第一步：配置 gradle.properties 文件**

首先找到本地电脑的 gradle  home，它一般在当前用户的目录下，比如我的 Mac 电脑位置在： ~/用户名/.gradle。

打开（如果没有，新建一个）gradle.properties 文件，增加下面两行：

    org.gradle.daemon=true
    org.gradle.jvmargs=-agentlib:jdwp=transport=dt_socket,server=y,suspend=n,address=5005

**第二步：增加 Remote 编译选项**

1、找到：Select Run/Debug Configration -> Edit Configrations...，如下图
![此处输入图片的描述][1]

2、左侧面板，点击“+”，选择 Remote，如下图
![此处输入图片的描述][2]

3、随便填入一个名字，比如 APT，点击 Apply、Ok，使用默认配置完成设置
![此处输入图片的描述][3]

4、切换到 APT，点击 Debug 按钮，建立到本地5005端口的连接
![此处输入图片的描述][4]
成功之后，会有如下提示

![此处输入图片的描述][5]
这样，我们已经可以开始调试 APT 代码了。

**第三步：开始调试**

设置好断点，然后选择 AS 菜单栏：Build -> Rebuild Project，
![此处输入图片的描述][6]

然后，开始调试你的 APT 代码吧~

> 因为 AS 启动一个项目的时候，默认也会去占用上面配置的 5005 端口。所以，你如果你发现 AS 提示你端口被占用，请先杀掉本地占用 5005 端口的进程。

> 另外，这一步使用 Rebuild Project 也是我尝试多次后得出最靠谱的方案。网上有说 `gradle clean assembleDebug` 命令也可以开启 debug APT，我这里发现不太稳定，有时候不可以，不知道原因出在哪里，知道的同学麻烦告知下~

本文示例代码：[CompilerAnnotation](https://github.com/hiphonezhu/Android-Demos/tree/master/CompilerAnnotation)

参考文章：

 - [如何debug自定义AbstractProcessor](http://www.jianshu.com/p/80a14bc35000)
 - [JavaPoet的基本使用](http://blog.csdn.net/crazy1235/article/details/51876192)
 - [ Android 如何编写基于编译时注解的项目](http://blog.csdn.net/lmj623565791/article/details/51931859)

  [1]: http://upload-images.jianshu.io/upload_images/1787010-894c90712fad2e92.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240
  [2]: http://upload-images.jianshu.io/upload_images/1787010-ee77646013860e51.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240
  [3]: http://upload-images.jianshu.io/upload_images/1787010-5573b0a767c3d1d3.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240
  [4]: http://upload-images.jianshu.io/upload_images/1787010-0670f23dc43e09d8.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240
  [5]: http://upload-images.jianshu.io/upload_images/1787010-da4f7ffe68caa139.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240
  [6]: http://upload-images.jianshu.io/upload_images/1787010-4cba6cb827c07366.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240