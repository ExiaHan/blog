---
title: "SubsTrate with AndroidStudio"
date: 2015-10-5T23:37:08+08:00
categories: ["Android"]
tags: ["Android", "Android-Studio", "SubsTrate"]
---

**参考文档：** *[看雪](http://bbs.pediy.com/showthread.php?t=199671)，[光哥博客](http://burningcodes.net/cydia-substrate-android-so-hook/)，[乌云知识库](http://drops.wooyun.org/tips/8084)*

## 0x1 介绍

在家木事，学了下substrate用法，这里记录下～

**[Substrate](http://www.cydiasubstrate.com/)**是大名鼎鼎的Cydia团队开发的一套框架工具，支持iOS和Android，相比于Xposed而言，Substrate不仅能提供java层函数的Hook，更能对Native层的C函数进行Hook，而且使用上也比Xposed要方便一些，而且官方提供的***[文档](http://www.cydiasubstrate.com/api/c/)***也比较全，查阅也比较方便，本文将实现一个简单的C函数Hook例子。

## 0x2 准备工作

+ 确保设备已经root，注意，测试发现目前substrate暂不支持Android Lollipop及其更新版本
+ 按照官方说明，下载substrate框架apk并安装，重启设备
+ 下载sdk到本地并解压，官方提供了两种方式
	- [直接下载](http://asdk.cydiasubstrate.com/zips/cydia_substrate-r2.zip)
	- 通过Android SDK Manager管理下载：[说明页面](http://www.cydiasubstrate.com/id/73e45fe5-4525-4de7-ac14-6016652cc1b8/)

## 0x3 Substrate with Android Studio


首先还是要吐槽下Android Studio至今对NDK的支持依然不敢恭维，而且一个版本一个config文件夹的本地文件夹命名方式也实在让人实在难受。。
言归正传：

+ 新建一个工程，名字随意，比如HelloSubstrate
+ 由于是个C函数Hook的例子，所以不需要有activity（其实连java层的源码都不需要
+ 在Project的app文件夹下创建子路径**./lib/armabi/**，把sdk里对应的文件夹下的.so文件丢进去
+ 在src/main下创建子路径**./jni**，创建一个**xxx.cy.cpp**文件用来存放c层源码，同时把substrate.h文件丢进去（注意这里文件名结尾必须是cy.cpp，substrate文档里要求这么做，用于识别---[文档](http://www.cydiasubstrate.com/inject/android/)
+ 在manifest文件里加上权限声明，否则无法正常工作
	- ***`<uses-permission android:name="cydia.permission.SUBSTRATE"/>`***

### 额外的准备工作

又要吐槽Android Studio了，ndk的整合上十分混乱，到现在为止已经有三种使用ndk的方法了，使用以前的的方法会提示gradle警告，但能通过，不过为了紧跟潮流，这里试用最新的方法，步骤如下：

+ 确保你的gradle是最新的
+ 修改./Project/build.gradle的classpath为
	- ***`classpath 'com.android.tools.build:gradle-experimental:0.2.0'`***
+ 修改./Project/app/build.gradle为如下样式：**注意其中的不同**
	- apply plugin的模块变了
	- 所有变量都要用 ***"="*** 来赋值，以前可以是*"变量名 空格 值"*的形式

```XML
apply plugin: 'com.android.model.application'

model {
    android {
        compileSdkVersion = 23
        buildToolsVersion = "23.0.2"
        defaultConfig.with {
            applicationId = "com.exiahan.hellosubstrate"
            minSdkVersion.apiLevel = 15
            targetSdkVersion.apiLevel = 23
            versionCode = 1
            versionName = "1.0"
        }
        tasks.withType(JavaCompile) {
            //指定编译JDK版本
            sourceCompatibility = JavaVersion.VERSION_1_7
            targetCompatibility = JavaVersion.VERSION_1_7
        }
    }
    android.buildTypes {
        release {
            minifyEnabled = false
            proguardFiles += file('proguard-rules.pro')
        }
    }

    android.ndk {
        moduleName = "cydiaSubstrateHook.cy"
        ldFlags += "-L./lib/armeabi"
        ldLibs += "log"
        ldLibs += "substrate"
        ldLibs += "substrate-dvm"
        abiFilters += "armeabi"
    }
}
dependencies {
    compile fileTree(include: ['*.jar'], dir: 'libs')
    compile 'com.android.support:appcompat-v7:23.0.0'
}

```

由于我们生成的hook用apk会在运行时使用substrate框架提供的库函数和log相关函数，所以需要加上动态链接的一些选项，同时按照sdk要求，生成的modulename必须是.cy.so结尾

一切就绪，就可以开始编写小例子了。

在刚刚的jni文件夹里创建cydiaSubstrateHook.cy.cpp文件，内容如下：

```CPP
//
// Created by exiahan on 15-10-5.
//

#include <stdio.h>
#include <android/log.h>
#include <unistd.h>
#include "substrate.h"
#define TAG "CydiaSubstrate"
#define LOGI(...) __android_log_print(ANDROID_LOG_INFO,TAG,__VA_ARGS__)//define LOGI Marco

#define GETLR(store_lr)  \
  __asm__ __volatile__(  \
    "mov %0, lr\n\t"  \
    :  "=r"(store_lr)  \
  )

//Specify the lib to hook
MSConfig(MSFilterLibrary, "/system/lib/libc.so")

int (* oldfopen)(const char* path, const char* mode);

int newfopen(const char* path, const char* mode) {
	unsigned lr;
    LOGI("[I]:newfopen called! -- [%d]", getpid());
    GETLR(lr);
    LOGI("[I]:BackTrace -- [0x%x]\n", lr);
    LOGI("[I]:File Path -- [%s]", path);
    return oldfopen(path, mode);
}

/*
 * Substrate entry point
 */
MSInitialize
{
    // Let the user know that the extension has been
    // extension has been registered
    LOGI( "Substrate initialized.");
    MSImageRef image;

    image = MSGetImageByName("/system/lib/libc.so");

    if (image != NULL)
    {
        void * hookfopen=MSFindSymbol(image,"fopen");
        if(hookfopen==NULL)
        {
            LOGI("error find fopen ");
        }
        else {
            MSHookFunction(hookfopen,(void*)&newfopen,(void **)&oldfopen);
        }
    }
    else {
        LOGI("ERROR FIND LIBC");
    }

}
```

阅读源码，可知Hook过程如下：

+ 确定我们要Hook的函数，为其重写一个我们自己的版本，这里选择的是fopen，也是脱壳时通常会注意的函数
+ 使用MSConfig宏指定fopen所在的库libc.so为要hook的库
+ 在MSInitialize初始化函数里执行如下流程：
	- MSGetImageByName得到libc.so句柄
	- MSFindSymbol找到即将被hook的函数fopen的函数指针并存储，以保证在hook后依旧可以正常完成相关功能
	- MSHookFunction开始hook函数，这时会把原本的fopen的调用地址放到oldopen里，这样在newfopen里调用oldfopen完成真正的fopen功能

运行结果如图：

![result](/images/SubsTrateWithAndroidStudio/result.jpg)

## 0x4 总结

+ 到现在为止接触了两种用于Hook的框架，相对于Xposed，Substrate在代码注入上更容易操作，而且提供了native层的注入选项，但是相对于xposed现在已经支持Lolipop而言，substrate还停留在dalvik的时代，期待支持lollipop的substrate尽快出现
+ 框架为我们提供了方便，但是阅读他们的文档就能发现归根结底的话都是在对rom做动态修改，最近看到了看雪上的**[DexHunter](http://bbs.pediy.com/showthread.php?p=1389779#post1389779)**，十分厉害，大概思路是定制ROM实现脱壳，这样apk做的一些反调试和防注入而言（*一些加固方案会自己通过.init方法在本地hook一些libc关键函数*）基本上都是小儿科了，准备下个月有空了买个Nexus好好研究下。
