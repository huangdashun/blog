---
title: Android打包之多版本、多环境、多渠道、混淆
date: 2018-03-29 20:45:52
tags:
---

# 打包准备

## 签名

[不会创建签名请点我](https://www.cnblogs.com/details-666/p/keystore.html)

签名创建好之后,将签名文件keystrore放在app目录下面

在gradle.properties里配置如下


```
RELEASE_STORE_PASSWORD=4662655
RELEASE_KEY_ALIAS=android.keystore
RELEASE_KEY_PASSWORD=4662655
```

在app下的build.gradle下配置如下

![image](https://note.youdao.com/yws/api/personal/file/WEBa99fcfcf2dd5a418606e221e85a68b42?method=download&shareKey=1a2fbcc799834e828d887b94cad64f0f)

# 配置版本,环境,渠道。

## 多版本(基于buildTypes)
1. debug:调试版本,无混淆
2. release:发布版本,有混淆、压缩

在app下的build.gradle下配置如下

![image](https://note.youdao.com/yws/api/personal/file/WEB0f437f20f56359259158855edd708f61?method=download&shareKey=43bf9734298266fa5b0b98c23ccf0049)

## 多环境 (基于pdoductFlavors)

1. develop:开发环境，开发和自测时使用
2. check:测试环境，克隆一份生产环境的配置，在这里测试通过后，再发布到生产环境。
之所以没命名为test是因为在gradle编译时:ProductFlavor names cannot start with 'test'.
3. product:生产环境，正式提供服务的。

在app下的build.gradle下配置如下

![image](https://note.youdao.com/yws/api/personal/file/WEB014f3be303b92b35398f7cd6fd1024ac?method=download&shareKey=c82e5f2653f3c24438e71e5e0ffabf56)

## 多渠道

使用了美团封装的Walle库,具体请参考[Android打包之多版本、多环境、多渠道](http://note.youdao.com/)

# 打包多环境

执行gradle命令

![image](https://note.youdao.com/yws/api/personal/file/WEB3e12a6870071ae58c5f7d893b51a5557?method=download&shareKey=2779ce07bc5128f5d554a09e27aed297)

或者命令执行 gradle assemble

![image](https://note.youdao.com/yws/api/personal/file/WEB1b33a26c6767bb9567b251d61a3db440?method=download&shareKey=266d853c12f16b75778e32366bb39baf)

# 利用脚本一件打包上传

## 配置fir环境

[请参考github](https://github.com/FIRHQ/fir-cli)

[fir官网](https://fir.im/)

在app里创建一个sh文件，以打包上传debug包为例


```
./gradlew clean
./gradlew assembleDebug

// fir.im 测试环境  账号需要申请
fir login 9a44776c3563fc133178ddaeb40ce030
fir publish app/build/outputs/apk/app-dev-release.apk
```

# 混淆

## 1.代码混淆配置

![image](https://note.youdao.com/yws/api/personal/file/WEB13e7186ea1092f8f600ed90eec6d792f?method=download&shareKey=97a19a6be6027384fd6931baad83274d)


proguardFiles用于选定混淆配置文件，getDefaultProguardFile(‘proguard-android.txt’) 方法可从 Android SDK tools/proguard/ 文件夹获取默认的 ProGuard 设置

## 2.混淆规则

在Android SDK tools/proguard/ 文件夹下默认有三个文件：

1. proguard-android.txt-->默认的Proguard配置文件（未优化）  
2. proguard-android-optimize.txt-->默认的Proguard配置文件（已优化）
3. proguard-project.txt-->默认的用户定制Proguard配置文件。


[详细介绍点击查看](http://blog.csdn.net/guolin_blog/article/details/50451259)

