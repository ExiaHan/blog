---
title: "Dice Write Up"
date: 2015-09-18T20:01:27+08:00
categories: ['CTF']
tags: ['CTF', 'Reverse', 'SimpleXue']
---

## 0x1

题目链接：[Simple-Reverse-Dice](http://ctf5.simplexue.com/re/Dice.exe)

## 0x2

拿到程序后运行观察其行为，发现是一个Windows CLI程序，运行后提示是按其要求摇骰子，摇出其指定的点数才能进行到下一步，全部正确则吐出Flag，错误则结束。

运行时程序会有一些提示字符串，记下部分，用IDA打开静态分析。

## 0x3

使用IDA和Ollydbg进行分析

+ 在String Tab中找到对应的String，找到其关联的代码，转成C伪代码分析，可以看到程序是一个WinMain程序，查看其变量，发现有两个**time类型**，推测可能每次摇骰子是随机生成数值。
+ 继续向下，可以看到程序有判断是否有附加调试器：
	+ isDebugerPresent()函数，标记，使用Ollydbg时要记得修改其值，绕过调试器检测
+ 继续向下走的话，可以看到每次判断都是调用随机数生成函数后存入值到内存地址0x0022FE9C处，然后从此内存取值，依次判断是否是3-1-3-3-7，需要注意的是最后一个判断7的时候没有再生成随机数，所以要在判断前直接修改内存为7
+ ~~再继续，3-1-3-3-7完成后，在显示flag之前，还有一次比较，如果相等会跳到something wrong，所以也要做一次patch~~
+ 仅按照上述内容patch后发现打出的flag是乱码，猜测可能有其他坑，重新浏览代码发现每次判断roll点正确（即3-1-3-3-7）后，还会判断时间差，如果大于2，就会把用来计算flag的值做一次乘2,所以每次比较完后要在比较时间差处也做patch，防止被乘2
+ 最后发现上面最好patch后，加了横线的描述应该不会发生

#0x4
上述步骤完成，拿到flag，如图：

![flag](/images/DiceWriteUp/flag.jpg)

#0x5

昨晚后光哥给的思路是直接运行前静态改好，然后直接让他跑一下就把flag吐出来了。。。o<<(≧口≦)>>o，炸裂了。。我好蠢。。汪，就这样

顺便把我的暴力方法打的断点图备忘一下：

![break list](/images/DiceWriteUp/breaklist.jpg)
