---
title: "NSCTF2015 WriteUp"
date: 2015-09-25T20:56:06+08:00
categories: ['CTF']
tags: ['NSCTF2015', 'WriteUp']
---

**参加了NSCTF线上比赛，感觉自己水平还是有待提高啊，写一下做出来的题目的备忘**

## Reverse

### 0x1 简单的逆向

**题目地址：**[Reverse01](http://www.nsctf.net/static/uploads/74ec9621ac5f8573abc90b3fb9199e38/Reverse01.exe)

+ 运行程序，出现CLI程序窗口，提示输入密码，随意输入，提示错误。
+ 使用PEID查看，发现加了ASPACK2.12的壳，使用od加载，看到有pushad操作，使用esp方法脱壳
+ ~~不需要完整脱壳，在壳程序运行完，dump程序~~[此步不是必须，但是可以dump出来后查看源码]
+ 回到od，查找字符串，找到提示的那句**"please input ns-ctf password"**，跳转到引用文字，发现有strcmp比较，比较字符串固定，为**"nsF0cuS!x01"**
+ 输入上面的字符串**"nsF0cuS!x01"**
+ 单步跟踪，发现有个jle跳转，如果条件满足会跳过一个函数调用，直接printf出来一个有乱码的flag，推测可能之前还有处理
+ 修改jle跳转为改为**"jmp short Reverse0.00401150"**，即调用其本来会通过jle跳过的函数，如图：

![reverse01.0](/images/NSCTF2015WriteUp/re11.jpg)

+ 继续F9运行，程序吐出flag，如图：

![reverse01.1](/images/NSCTF2015WriteUp/re12.jpg)

### 0x2 较简单的逆向

**题目地址：**[Reverse02](http://www.nsctf.net/static/uploads/806b95d9497584c4a9d89118c8944424/Reverse02.exe)

*本题和第一题类似，只不过改成了窗口程序*

+ 运行，发现窗口程序
+ 使用OD或者IDA打开
+ 尝试搜索"Flag"，发现有好几个匹配，记下，同时猜测可能和第一题一样有对flag处理
+ 查看导入表，发现是个dialogbox，查找调用，找到**0x00401240**处的DialogBoxParamA调用，从其参数里找到对应回调处理函数入口为**0x00401180**
+ 转到**0x00401180**处，发现有个GetDlgItemTextA的调用，在其下有个**call 0x00401070**，猜测会在其中处理Flag，修改程序执行流，让其可以执行，跟踪进入此函数，如图：

![reverse02.0](/images/NSCTF2015WriteUp/re21.jpg)

+ 进入0x00401070后，可以看到上面有个0x00401000的函数，可以看到内部有调用MessageBoxA显示Flag，同时在0x00401070内发现有此函数调用，修改执行流让其可以执行，跟踪进入0x00401000，在MessageBoxA调用前下断，看到真正的flag，如图：

![reverse02.1](/images/NSCTF2015WriteUp/re22.jpg)

### 0x3 逆向

**题目地址：**[Reverse03](http://www.nsctf.net/static/uploads/877afdad88a2340eaefe8c1a87bb391e/Revesre03.exe)

*分析：使用python生成的程序，运行时在本地Temp文件夹里释放文件，通过CreateProcessA运行一个新进程来执行，没搞定。。。。。。。。囧。。*

## MISC

### 0x1 Twitter
**这题，额。。。啧啧。。一个md5,100块钱，不过有人抖了答案出来**

### 0x2 Wireshark

**题目地址：**[sniffer.pcapng](http://www.nsctf.net/static/uploads/16d8a763795dd8cc3cf5f599fbb5e5af/sniffer.pcapng)

从题目可以看出来是个抓包题，wireshark打开文件[也可以用Dshell，或者binwalk]

![misc02](/images/NSCTF2015WriteUp/misc1.jpg)

+ 题目说是下载，猜测在http里，表达式过滤http
+ 看到有个key.rar，服务器为192.168.52.1
+ dump出key.rar，解压，需要密码
+ 继续查找发现获取rar之前还有个从服务器获取的页面，dump内容保存成html文件，内容有提示密码为nsfocus+5个数字
+ 生成字典，爆破，解压密码为nsfocus56317,打开后获得flag


## WEB

### 0x1 Be Careful

使用chrome dev tools跟踪页面，发现有个301重定向，猜测可能有个默认的动态页面，尝试index.php，发现确实存在，使用wireshark抓包，看到flag在注释里。

![web01](/images/NSCTF2015WriteUp/web1.jpg)

###0x2 Decode

题目里给了个php的函数，接受传入的字符串

```php
function encode($str){
	$_o = strrev($str);
	for($_0=0;$_0<strlen($_o);$_0++){
		$_c = substr($_o,$_0,1);
		$__ = ord($_c)+1;
		$_c = chr($__);
		$_= $_.$_c;
	}
	return str_rot13(strrev(base64_encode($_)));
}
```

其作用如下：

+ 反转字符串
+ 每个字符对应值加1再转回字符串，即每个字符都变成其后一个字符
+ 得到的新字符串base64编码
+ 编码后再反转
+ 反转后rot13编码得到最终字符串，即题目的

```c
**a1zLbgQsCESEIqRLwuQAyMwLyq2L5VwBxqGA3RQAyumZ0tmMvSGM2ZwB4tws**
```

根据上面的分析逆向一次即可得到flag

## CRYPTO

### 0x1 神秘的字符串

给了一串字符串（写在比赛结束后，已经看不到了。。），感觉像base64，解一下，发现有Salted_字样的玩意，应该是AES加密，到网上找了个网站解密了一下，结果解出来是flag{字符串}的形式，因为字符串不是NSCTF开头，感觉还有加密，在密码机器网上跑了一下，发现是凯撒移位加密，得到最终flag是NSCTF_Rot_Encryption

### 0x2 神奇的图片

**题目地址：**[oddpic.jpg](http://www.nsctf.net/static/uploads/8041661a723dcc82a8a088163e2cd9ac/oddpic.jpg)

binwalk走一遍，里面有其他的jpeg，dd出来，得到flag

![crypt02](/images/NSCTF2015WriteUp/crypto2.jpg)

### 0x3 神秘的图片+10086

**题目地址：**[newnewnew.jpg](http://www.nsctf.net/static/uploads/50cf3d7ee75c2bbb7a91808dc811aa24/newnewnew.jpg)

binwalk走一下没结果，用stegsolv分析下，在blue plane 0通道看到一个颜色反转的二维码，保存出来，用ps等工具反转下颜色，手机扫描，得到flag

![crypt03](/images/NSCTF2015WriteUp/crypto3.jpg)


**Update:附上官方博客资料，里面有WriteUp，[绿盟博客](http://blog.nsfocus.net/nsctf-network-attack-defence-game-download/)**
