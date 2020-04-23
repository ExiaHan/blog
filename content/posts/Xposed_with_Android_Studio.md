---
title: "Xposed with Android Studio"
date: 2015-09-02T16:06:55+08:00
categories: ["Android"]
tags: ["Android", "Android-Studio", "Xposed"]
---

## 0x1 介绍
Xposed不用说，估计是很多人都知道的神器，通过替换系统文件在zygote(Android应用的孵化器)阶段进行hook，给予了xposed相当于root的权限，使用Xposed来修改修改和定制系统，或者在某些需要的情况下想从一些应用里套点数据，Xposed都是十分在行。
本文简单介绍使用Xposed来套取一个示例Android应用中某个函数运行时传入的参数。

## 0x2 开工
**这篇文章也是作为Xposed插桩的练习记录**

### 引入Xposed框架
不得不说，即使有[rovo89](https://github.com/rovo89)的[Xposed Tutorial](https://github.com/rovo89/XposedBridge/wiki/Development-tutorial)，Android-Studio到目前为止也并不是很让人能轻松上手，特别是eclipse用惯以后，这里要说明一下如何在Android-Stuio中引入Xposed框架包。

+ 下载xposed框架jar包
+ 创建你的xposed module工程，如"HelloXposedHook"
+ 在app/src下创建文件夹xposedLib


- - -
之所以要在这里分开说最后一步，是因为目前我知道的有两种方法引入框架而不在生成时包含

+ **方法一：**
	+ 把下载好的xposed-bridgeAPI.jar包拖进去，右键，选择add as library
	+ 在Project视图下右键项目名，选择open module settings，把xposed-bridgeAPI那一项有compile改成proivde

如图：
![xposed provide](/images/XposedWithAndroidStudio/xposedprovide.jpg)

+ **方法二：**
	+ 把下载好的xposed-bridgeAPI.jar包拖进去
	+ 在app/xposedLib/路径里创建build.gradle文件，内容如下

```xml
apply plugin: 'java'
dependencies {
    compile project(":lib")
    provided fileTree(dir: 'lib', include: ['*.jar'])
}
```




>根据xposed tutorial的提示，jar包不能放app/libs，否则会自动包含
>xposed的jar包要求仅被引用，而不能包含
>所以放xposedLib
>并且在module settings里把相关项改成provide



**到这里xposed框架的引入即可成功**

### 创建你的Xposed Module

+ 修改AndroidManifest.xml，加入xposed的meta元素（**具体如何参见rovo89的Tutorial**）

如下所示：
```xml
<application
        android:allowBackup="true"
        android:icon="@mipmap/ic_launcher"
        android:label="@string/app_name"
        android:theme="@style/AppTheme" >
        <meta-data
            android:name="xposedmodule"
            android:value="true" />
        <meta-data
            android:name="xposeddescription"
            android:value="My First Xposed Module for Hook" />
        <meta-data
            android:name="xposedminversion"
            android:value="30" />
		<activity
            android:name=".MainActivity"
            android:label="@string/app_name" >
......
```

+ 创建你的使用xposed插件的函数，如MyXposedModule
+ 添加文件夹/app/src/main/assets/xposed_init，在里面写上你的xposed module类全称（包含包名
	+ 如**com.xxx.helloxposedhook.MyXposedModule**

这里我要做的是Hook自己写的另一个示例小程序的一个java方法，在log里打印出传给他的参数

功能代码如下：
```java
public class MyXposedModule implements IXposedHookLoadPackage{

    @Override
    public void handleLoadPackage(XC_LoadPackage.LoadPackageParam lpparam) throws Throwable {
        //TODO Auto-generated method stub
        //filter the package


        if (!lpparam.packageName.equals("com.xxx.remotecontrol"))
            return;

        Log.e("Package", "\n" + lpparam.packageName + "\n");

        //if has a custom class param, use reflact to get the class as type
        //Class<?> XXXJni = null;
        //XXXJni = lpparam.classLoader.loadClass("com.xxx.remotecontrol");
        findAndHookMethod("com.xxx.remotecontrol.mainActivity", lpparam.classLoader, "forHook", String.class, new XC_MethodHook() {
            @Override
            protected void beforeHookedMethod(MethodHookParam param) throws Throwable {
                // this will be called before the clock was updated by the original method
                String parg = (String) param.args[0];
                Log.e("Result:", "\nHi\n");
                Log.e("CMD:", parg);
            }

            @Override
            protected void afterHookedMethod(MethodHookParam param) throws Throwable {
                // this will be called after the clock was updated by the original method
            }
        });
    }
}
```

Hook成功后首先打印Hi，然后打印参数，即CMD后的parg

结果如图：
![result](/images/XposedWithAndroidStudio/result.jpg)

**到这里也即完成了xposed module的创建**

## 0x3 结束
这里仅仅作为一个例子. 想继续深入研究的朋友可以去github找一些较大的xposed插件工程源码阅读:\).
