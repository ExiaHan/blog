---
title: "Dump Dex From IJiaMi"
date: 2015-10-07T23:20:29+08:00
categories: ['UnShell']
tags: ['Android','Reverse', 'Dex']
---

**内容仅供学习讨论**
**参考文档：** *[爱加密动态脱壳](http://www.sycode.cn/2015/06/27/%E7%88%B1%E5%8A%A0%E5%AF%86-%E5%8A%A8%E6%80%81%E8%84%B1%E5%A3%B3%E4%B9%8B%E3%80%90%E6%99%BA%E8%83%BD%E8%A7%86%E7%AA%97app%E3%80%91%E8%84%B1%E5%A3%B3%E4%BF%AE%E5%A4%8D/#0x001)，[爱加密动态脱壳法](http://www.cnblogs.com/2014asm/p/4112116.html)，[浅谈android逆向分析那些拦路虎](http://www.9hao.info/pages/2014/07/qian-tan-androidni-xiang-fen-xi-na-xie-lan-lu-hu)，[光哥博客](http://burningcodes.net/%E4%BB%8E%E6%BA%90%E7%A0%81%E4%B8%AD%E8%B7%9F%E8%B8%AAdex%E7%9A%84%E5%8A%A0%E8%BD%BD%E6%B5%81%E7%A8%8B/)*

## 0x1 说在开头

现在有很多软件都是有加壳的，作为安全狗肯定要研究研究，从网上找了点资料，然后找了个用IJiaMi加固的顺着做了一下。

## 0x2 实验环境

+ Android 真机，Version 4.1.2， with **ROOT & Xposed**
+ 某被IJiaMi加固的App
+ IDA调试器
+ adb

## 0x3 过程

下断点：

+ libc.so fgets fopen
+ libdvm.so _Z21dvmDexFileOpenPartialPKviPP6DvmDex

这里关于为何在fgets和fopen上下断点的原因几乎都有说明，是因为加固措施里有反调试，方法是使用子进程检测父进程是否被ptrace（即被调试），如果有则强制关闭说有进程，而检测方法就是读取
```bash
 |---/proc/[pid]/status
```

对于这个我也写了个，大概实现可以看这里：[getTracerPid](https://github.com/ExiaHan/exiahanLib/blob/master/exiahanLib.c)

文件里的TracerPid行，如果为0说明没有被调试，否则就是调试器的PID
所以要在fgets和fopen上下断，找到其打开后获取值的语句，改为0,patch掉反调试

*真正调试时发现其实不需要每次都patch掉，只要找到用于反调试的子进程，然后直接休眠掉即可，否则如果碰到变态的每隔几秒就检测一次的加固方式，那就只能一直人工patch了┗<(=｀o′=)>┓哼 ┏<(=｀○′=)>┛哼┏<(=｀o′=)>┓哈┗<(=｀O′=)>┛兮！！*

再下来是**_Z21dvmDexFileOpenPartialPKviPP6DvmDex**，位于libdvm.so，libdvm是library-dalvik-virtual-machine，即dalvik虚拟机的核心所在shared object，其中的*_Z21dvmDexFileOpenPartialPKviPP6DvmDex*即为运行一个dex前需要调用的**参与打开dex文件**的一个函数，为什么要在这里下断呢？

去源码dalvik文件夹下执行

```bash
grep -rn dvmDexFileOpen
```

可以找到dvmDexFileOpenPartial函数，其定义位于./dalvik/vm/DvmDex.h:84，如下：

```CPP
 72 /*
 73  * Given a file descriptor for an open "optimized" DEX file, map it into
 74  * memory and parse the contents.
 75  *
 76  * On success, returns 0 and sets "*ppDvmDex" to a newly-allocated DvmDex.
 77  * On failure, returns a meaningful error code [currently just -1].
 78  */
 79 int dvmDexFileOpenFromFd(int fd, DvmDex** ppDvmDex);
 80
 81 /*
 82  * Open a partial DEX file.  Only useful as part of the optimization process.
 83  */
 84 int dvmDexFileOpenPartial(const void* addr, int len, DvmDex** ppDvmDex);
 85
 86 /*
 87  * Free a DvmDex structure, along with any associated structures.
 88  */
 89 void dvmDexFileFree(DvmDex* pDvmDex);
```

可以看到三个和加载dex（包括odex）的函数都在这里，就像注释说明的，dvmDexFileOpenPartial函数是加载dex文件必须用到的一步，其前两个参数分别是dex文件在内存中的地址和其长度，知道这些后就可以尝试现在这里下断然后尝试dump出来dex文件啦。所以在其中下断，就能找到运行时所加载的真正dex文件，然后使用IDC脚本dump出来即可.

```CPP
auto fp, dexAddress;
fp = fopen("C:\\xx.dex","wb");
for (dexAddress = 0x3D4AE72C; dexAddress < 0x3DB9C098; dexAddress++)
   fputc(Byte(dexAddress), fp)
```

~~脱掉后dex2jar转成jar用jd-gui看下真正的入口Activity，然后baksmali一下，把manifest.xml里的入口改掉，改成真正入口Activity。~~
