---
layout:     post
title:      "Tomcat7 Maven Plugin 小记"
subtitle:   ""
date:       2017-05-09 16:36:00
author:     "zhuhf"
header-img: ""
header-mask: 0.3
catalog:    true
tags:
    - Tomcat
    - Maven
---



本文记录的是在 Mac 下使用 IntelliJ IDEA， 集成 tomcat7-maven-plugin 插件实现一键部署的过程。

### 安装 Tomcat
你可以在这里下载到 Tomcat 的各个版本，http://tomcat.apache.org/。

这里选择 **8.5.14 zip** 免安装版，它可以在 Windows、Linux、Mac 上使用。

![Tomcat 8.5.14.png](http://upload-images.jianshu.io/upload_images/1787010-7bc61c1c741335c3.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


解压缩 zip 包到磁盘任意位置，打开 bin 目录：

- Windows 用户使用 startup.bat、shutdown.bat 来启动和关闭 Tomcat；
- Mac、Linux 用户使用 startup.sh、shutdown.sh 来启动和关闭 Tomcat；
sh 脚本执行需要权限，否则会提示你“Permission denied”字样。
![Permission denied.png](http://upload-images.jianshu.io/upload_images/1787010-4e2df737e9029263.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
解决办法：
cd 进入 bin 目录，使用 chmod a+x *.sh 赋予所有 sh 脚本执行权限。

然后进入 conf 目录，打开 tomcat-users.xml 文件，新建一个用户并赋予权限：

    <role rolename="manager-gui”/>
    <role rolename="manager-script"/>
    <user username="admin" password="123456" roles="manager-gui,manager-script”/>

启动 Tomcat，浏览器打开 http://localhost:8080/manager/html，输入**用户名/密码**，确认是否配置成功。

### 配置 Maven（可省略）
Maven 安装步骤参考：http://www.jianshu.com/p/191685a33786

IntelliJ IDEA 内置了 Maven，确认下当前使用的是内置 Maven，还是本地安装的 Maven。

打开 IntelliJ IDEA -> Preferences -> Build, Execution, Deployment -> Build Tools -> Maven
![maven.png](http://upload-images.jianshu.io/upload_images/1787010-7e4777c19ca2e96d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


重点关注 **User settings file** 这一项，打开 **setting.xml** 文件，给 Maven 增加访问 Tomcat 的权限，在 **servers** 节点下增加：

    <server>  
      <id>tomcat</id>
      <!— Tomcat 账户信息 —>
      <username>admin</username>  
      <password>123456</password>  
    </server>
    
- id 可以随便填写，后面配置 pom.xml 会用到；
- username 与 password 填写的是 tomcat-users.xml 配置的用户。
- 如果使用的是内置的 Maven，setting.xml 文件可能不存在，可以直接新建此文件，格式如下：

        <?xml version="1.0" encoding="UTF-8"?>
        <settings xmlns="http://maven.apache.org/SETTINGS/1.0.0"
          xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
          xsi:schemaLocation="http://maven.apache.org/SETTINGS/1.0.0 http://maven.apache.org/xsd/settings-1.0.0.xsd">
         <servers>
            <server>
               <id>tomcat</id>
               <username>admin</username>
               <password>123456</password>
            </server>
         </servers>
        </settings>

### 配置 pom.xml

    <plugin>
        <groupId>org.apache.tomcat.maven</groupId>
        <artifactId>tomcat7-maven-plugin</artifactId>
        <version>${tomcat7-maven-plugin.version}</version>
        <configuration>
            <server>tomcat</server>
            <path>/sample-api</path>
            <port>8081</port>
        </configuration>
    </plugin>
**server** 其实就是**"配置 Maven"** 这一步 setting.xml 中的 **id**。

前面说过，**"配置 Maven"**这一步可以省略，修改 pom.xml 配置如下：

    <plugin>
        <groupId>org.apache.tomcat.maven</groupId>
        <artifactId>tomcat7-maven-plugin</artifactId>
        <version>${tomcat7-maven-plugin.version}</version>
        <configuration>
            <username>admin</username>
            <password>123456</password>
            <path>/sample-api</path>
            <port>8081</port>
        </configuration>
    </plugin>
   
   其实 server（id） 就是替代 username 和 password 的，也没什么好多说的。

正常情况下，点击 tomcat7:deploy 就可以直接部署到 Tomcat 了。

![tomcat7-deploy.png](http://upload-images.jianshu.io/upload_images/1787010-49794777da54b586.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
