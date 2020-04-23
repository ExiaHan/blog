---
title: "IDF.CN Practice 01"
date: 2015-12-30T22:12:08+08:00
categories: ['CTF']
tags: ['idf.cn']
---

## 0x1 Preparation

既然决定要做，那就加油，写在开头，打好基础，加油吧。
Exia，斩获未来。。

## 0x2 Start

### 一、牛刀小试

#### 被改错的密码

从前有一个熊孩子入侵了一个网站的数据库，找到了管理员密码，手一抖在数据库中修改了一下，现在的密码变成了 cca9cc444e64c8116a30la00559c042b4，那个熊孩子其实就是我！肿么办求解！在线等，挺急的。。

*看起来挺像md5,直接拿去在线破，结果提示有问题，再一看，长度有问题，md5是32位哈希，这玩意33位，用python生成去掉其中某个字符的序列，然后再一个个在线破，发现是**cca9cc444e64c8116a30a00559c042b4**，解出来是idf，所以flag是wctf{idf}*

---

#### 啥

题目就是一张图片，bless打开看hex，在底部找到flag：wctf{mianwubiaoqing__}

---

#### ASCII码而已

明显一看是unicode，找个在线转换，flag为wctf{moremore_weibo_fans}

---

#### 摩斯密码

一段morsecode
--  ---  .-.  ...  . -.-.  ---  -..  .
在线解一下，flag为wctf{morsecode}

---

#### 聪明的小羊

*一只小羊跳过了栅栏，两只小样跳过了栅栏，一坨小羊跳过了栅栏...
tn c0afsiwal kes,hwit1r  g,npt  ttessfu}ua u  hmqik e {m,  n huiouosarwCniibecesnren.*

看题目就想到古典加密，既然是栅栏，那就栅栏密码吧，正好带学一下栅栏密码，就写了个python的加解码脚本，跑一下看到flag:wctf{C01umnar}

### 二、包罗万象 MISC

#### 图片里的英语

给了一张小李，binwalk一下。看到有个rar，dd提取出来，解压，得到一长flag，恩，没错，就是那张赵本山的图片，may the force be with you，然后首字母大写，wctf{Mtfbwy}

---

#### 抓到一只苍蝇

给了一个pcapng网络dump包，根据他说的内容，和包名带有fly，搜索包内容，找下字符串fly，成功找到，发现是上传了一个附件fly.rar,分成了5个包上传，第一个post指出fly.rar大小为525701，接着5个包都是传向ftn开头服务器，所以猜测是附件内容，前个都是131436大小，最后一个1777，（131436×4 + 1777 - 525701） / 5 = 364，所以每个包开头364部分为分包后的包头，dd导出内容，skip掉头364字节，然后cat成fly.rar，比较下post里给的md5,一样。

解压工具打开，提示文件损坏，看上去因该不是正常的加密，而且也根本没有其他密码的信息，推测可能改了文件头，bless修改加密位（第24个字节改为0x80）：

解压得到一个flag.txt，打开发现不行，binwalk一看是个pe。。。不过里面有png，而且很多60x60的，但有一个280x280的，dd解出来，打开是一张二维码，扫一下，flag为:flag{m1Sc_oxO2_Fly}，不过说没改格式，额，所以应该是wctf{m1Sc_oxO2_Fly}

### 三、初探乾坤 PPC

#### 简单编程-字符统计

给了一串字符串，让统计个数，妥妥的python，count函数直接搞定。直接提交，发现有坑，仔细看了下题目是要求2秒内搞定，那就只能自动接受内容然后发送post请求了

写个python脚本跑一下

```python
#!/bin/env python3
#coding:utf-8

import requests
import re

#url and regex for get string to count
url = 'http://ctf.idf.cn/game/pro/37/index.php'
regex = r'<hr />(.+)<hr />'

#get the string and remember the cookie
def requestString():
    respon = requests.get(url);
    strTarget = respon.text
    strTarget = strTarget.replace("\n", "")
    strTarget = strTarget.replace("\r", "")
    strTarget = strTarget.replace("\t", "")
    webCookie = respon.cookies
    strTarget = re.findall(regex, strTarget)
    return strTarget, webCookie

#count the words num
def genAnwser(strTarget):
    w, o, l, d, y = 0, 0, 0, 0, 0

    for c in strTarget:
        if c == "w":
            w += 1
        elif c == "o":
            o += 1
        elif c == "l":
            l += 1
        elif c == "d":
            d += 1
        elif c == "y":
            y += 1
    strAnwser = str(w) + str(o) + str(l) + str(d) + str(y)
    return strAnwser

#post the anwser with the cookie we get via requestString()
def postAnwser(strAnwser, webCookie):
    data = {}
    data['anwser'] = strAnwser
    respon = requests.post(url, data = data, cookies = webCookie)
    print (respon.text)

if __name__ == '__main__':
    result = requestString()
    strTarget = result[0][0]
    webCookie = result[1]
    strAnwser = genAnwser(strTarget)
    postAnwser(strAnwser, webCookie)
```

---

#### 谁是卧底

给了一个大文本串，打开看基本上都是乱字符，根据题意，大部分人都市文盲，卧底有点姿势，所以找个字典去统计下出现单词的地方，然后打印出最密集出现的部分，在里面找到

```
bananjpywlqclassifyubcjesqtqyjhazbornndomhfchvlc
what will you see if you throw the butter out the window
wzqmtwmyjutipvqetrsshyosypzydevelopponaxoezspdespairkuoqi

```
第二行是一句话，字面意思，丟块黄油你会看到啥，当然黄油飞啊，butter fly。。。就是butterfly，所以是wctf{butterfly}

---

#### Fuck your brain

Brain Fuck编码，直接找个在线解析器解析一下OK。
WCTF{Br31nF4ck}

### 倒行逆施

---

#### .Net逆向第一题

如题目说，应该是个简单的.Net逆向，用ILSpy查看代码，是个典型的des加密，key = "wctf{wol}"
iv = "dy_crack}"，然后加密后的结果在base64编码下和已有字符串**fOCPTVF0diO+B0IMXntkPoRJDUj5CCsT**进行比较。
根据这个过程逆推即可，写个python脚本跑下：
```python
#!/bin/env python3
#coding:utf-8

from Crypto.Cipher import DES
import base64

key64 = 'wctf{wol'
iv64 = 'dy_crack'
encryptedString = 'fOCPTVF0diO+B0IMXntkPoRJDUj5CCsT'

def myDecrypt(key, iv, arg):
    k = bytes(key.encode("ASCII"))
    i = bytes(iv.encode("ASCII"))
    mydes = DES.new(k, DES.MODE_CBC, i)
    rarg = base64.b64decode(arg.encode("ASCII"))
    for i in rarg:
        print('0x%x' %i, end= ' ')
    print(end='\n')
    strRes = mydes.decrypt(rarg)
    print(strRes)

if __name__ == '__main__':
    myDecrypt(key64, iv64, encryptedString)
```

最终结果是**wctf{dotnet_crackme1}**， 没怎么写过C#，这里有个坑是C#的createEncrypter()可以接受byte[] iv不为8 bytes，但是python里除了DES.MODE_ECB外都不可以，刚开始用ECB解出来不对，换回ECB会提示iv应该是8 bytes，就猜可能C#有做截断处理？于是把上面代码里的iv64最后的那个'}'去掉，再跑一下果然OK。。当然C#不熟，不知道理解的对不对。。。

---

#### 简单的PE文件逆向

和题目说的一样，一道简单的PE逆向，首先运行下，命令行输入flag
丢ida， shift f12 查看string，有个
`swfxc{gdv}fwfctslydRddoepsckaNDMSRITPNsmr1_=2cdsef66246087138`，是个字符数组

![pe1](/images/IDFCNPractice/pe1.jpg)

查找引用，在**sub_4113A0**里,tab查看反汇编代码，主体是个循环，和输入进行了17次判断，然后再继续用一个if判断5次，只要有一个不同就让v13为1,继而输出**Wrong**

![pe2](/images/IDFCNPractice/pe2.jpg)

继续看循环内如何从字符串数组取值，index通过
```CPP
*(&v14 + i)
```
来取值，再看函数开头，从v14到v35,刚好有超过17个连续的变量可用，再看前面的代码，有从v14到v35的初始化：

![pe3](/images/IDFCNPractice/pe3.jpg)

这里把v14到v30的值保存，为
**(0x1, 0x4, 0xe, 0xa, 0x5, 0x24, 0x17, 0x2a, 0x0d, 0x13, 0x1c, 0x0d, 0x1b, 0x27, 0x30, 0x29, 0x2a)**，
因为向下在if里还有5次比较，把这五次的值也保存，**(49, 48, 50, 52, 125)**。
接着动态运行确认下，在如下图里的位置下断：

![pe4](/images/IDFCNPractice/pe4.jpg)

运行几次，查看eax的值，和上面列出的相同，可知上述序列确实是用来从那个字符数组里取值的，所以最后的5次比较也是字符，因为接着v37后是四个chr类型，所以其实真正应该是一个v37[22]的数组。如此依赖用脚本跑一把。

```python
#!/bin/env python3
#! -*- coding:utf-8 -*-

strSrc = 'swfxc{gdv}fwfctslydRddoepsckaNDMSRITPNsmr1_=2cdsef66246087138'

numTuple = (0x1, 0x4, 0xe, 0xa, 0x5, 0x24, 0x17, 0x2a, 0x0d, 0x13, 0x1c, 0x0d, 0x1b, 0x27, 0x30, 0x29, 0x2a)

numChar = (49, 48, 50, 52, 125)

def doTheGenerate():
    for i in numTuple:
        print(strSrc[i], end='')
    for i in numChar:
        print(chr(i), end='')
    print('\n')

if __name__ == '__main__':
    doTheGenerate()

```

得到结果wctf{Pe_cRackme1_1024}

---

#### 简单的ELF逆向

题目给的是一个x86-64的ELF文件，跑一下和简单PE类似，要输入正确的flag，错误就打印"u r wrong。"
丢IDA反一下，ida对x64支持不太好，pseudo code显示还有点不爽（--！），不过幸好不太复杂，对着汇编还是能看的:

![elf1](/images/IDFCNPractice/elf1.jpg)
![elf2](/images/IDFCNPractice/elf2.jpg)

从上图看到v8到v15是8个int64_t的类型，v16是个int32_t，对应汇编用两次mov dword ptr指令初始化每个变量的低4bytes和个高4bytes:

![elf3](/images/IDFCNPractice/elf3.jpg)
![elf4](/images/IDFCNPractice/elf4.jpg)

接着是程序主体，接受输入，进行判断，如果和要求不符合，v24为1，打印 u r wrong，从伪码可以看出来，输入需要是个长度为22的串，否则会把v24设为1。

![elf5](/images/IDFCNPractice/elf5.jpg)

接下来看判断部分，和PE的类似，分两部分判定，for循环里判定前17个字符，用于进行判定比较的取值方法如下：

![elf6](/images/IDFCNPractice/elf6.jpg)

```CPP
j from 0 to 16
(*((DWORD *)&v8 + j) - 1) / 2
```

按照上面方法一次和输入v17[j]里的每个字符比较，这里可以看到已经把从v8开始的这些int64_t的变量看成了一个int32_t数组，刚好v8到v15有8个int64_t，刚好是16个int32_t，在加上最后的v16刚好是17个值，所以，根据上面的截图，这17的用来计算比较值的数据分别为**(0xEF, 0xC7, 0xE9, 0xCD, 0xF7, 0x8B, 0xD9, 0x8D, 0xBF, 0xD9, 0xDD, 0xB1, 0xBF, 0x87, 0xD7, 0xDB, 0xBF)**
紧接着是在if里进行5次最后的比较，用于比较的值是**(48, 56, 50, 51, 125)**
综上已经知道flag计算过程，写个脚本跑出来:

```python
#!/bin/env python3
#! -*- coding:utf-8 -*-

numForCalc = (0xEF, 0xC7, 0xE9, 0xCD, 0xF7, 0x8B, 0xD9, 0x8D, 0xBF, 0xD9, 0xDD, 0xB1, 0xBF, 0x87, 0xD7, 0xDB, 0xBF)
numForLastFive = (48, 56, 50, 51, 125)

def genTheFlag():
    resultLst = []
    for i in numForCalc:
        resultLst.append(chr((i - 1) // 2)) #落地除
    for i in numForLastFive:
        resultLst.append(chr(i))
    print(str('').join(resultLst))

if __name__ == '__main__':
    genTheFlag()

```

结果是**wctf{ElF_lnX_Ckm_0823}**

- - -

### python ByteCode

看题目就知道，工具用上，**uncompyle2**（如果是exe可能还要用**unpy2exe**），得到下面的py脚本代码：
```python
# 2015.12.31 18:17:37 CST
#Embedded file name: d:/idf.py


def encrypt(key, seed, string):
    rst = []
    for v in string:
        rst.append((ord(v) + seed ^ ord(key[seed])) % 255)
        seed = (seed + 1) % len(key)

    return rst


if __name__ == '__main__':
    print "Welcome to idf's python crackme"
    flag = input('Enter the Flag: ')
    KEY1 = 'Maybe you are good at decryptint Byte Code, have a try!'
    KEY2 = [124,
     48,
     52,
     59,
     164,
     50,
     37,
     62,
     67,
     52,
     48,
     6,
     1,
     122,
     3,
     22,
     72,
     1,
     1,
     14,
     46,
     27,
     232]
    en_out = encrypt(KEY1, 5, flag)
    if KEY2 == en_out:
        print 'You Win'
    else:
        print 'Try Again !'
+++ okay decompyling crackme.pyc
# decompiled 1 files: 1 okay, 0 failed, 0 verify failed
# 2015.12.31 18:17:37 CST
```

把输入存到flag， 然后通过encrypt(KEY1, 5, flag)加密，再和KEY2进行比较，相同即可，从encrypt()函数可以知道输入和KEY2长度相同，再按照encrypt()的思路反过来算一遍即可。。。
结果发现，尼码想简单了。。根本不行。。。怎么算带乱码。。只能爆破了。。。
```python
#!/bin/env python3
#! -*- coding:utf-8 -*-

KEY1 = 'Maybe you are good at decryptint Byte Code, have a try!'
KEY2 = (124, 48, 52, 59, 164, 50, 37, 62, 67, 52, 48, 6, 1, 122, 3, 22, 72, 1, 1, 14, 46, 27, 232)
SEED = 5

def deCrack(KEY1, KEY2, SEED):
    resultList = []
    resultStr = ''
    tmp = 0
    for v in KEY2:
        #尽量缩小范围吧，可见字符是从33到126
        for i in range(33, 127):
            if v == ((i + SEED ^ ord(KEY1[SEED])) % 255):
                resultList.append(chr(i))
        SEED = (SEED + 1) % len(KEY1)
    resultStr = resultStr.join(resultList)
    print()
    return resultStr

if __name__ == '__main__':
    rst = deCrack(KEY1, KEY2, SEED)
    print(rst)

```
拿到flag，**WCTF{ILOVEPYTHONSOMUCH}**

### BreakPoint

给了一个ELF32的文件，行为和之前的一样，输入flag，正确即可。
丢IDA，程序逻辑很明显：

+ 打印提示字符
+ 接收输入
+ 利用提示字符的地址和main的地址计算了一个值放v0和v4
+ 在一个if里进行比较，比较的值都是写死在程序里

关键逻辑和对应数据如图：

![bk1](/images/IDFCNPractice/bk1.jpg)
![bk2](/images/IDFCNPractice/bk2.jpg)
![bk3](/images/IDFCNPractice/bk3.jpg)

ida看下main地址就知道v0的值肯定会在第一个if的do-while里算出来，需要注意的是，这里不能gdb去调试，因为gdb下断是用 int $0x3，软中断的机制是在断点处把代码替换成int $0x3，但是计算的时候会把整个程序从main（**0x80483B0**）到字符串（**0x804872A**）的数据拿来用于计算v0,所以只能把这些数据抠出来单独跑v0的值，跑到后下面就OK了。

```CPP
#include <stdio.h>
#include <sys/types.h>

u_int8_t v1[] = {0x55, 0x89, 0xE5, 0x57, 0x56, 0x53, 0x83, 0xE4, 0xF0, 0x83, 0xEC, 0x20, 0xA1, 0x00, 0x99, 0x04, 0x08, 0xC7, 0x04, 0x24, 0xF0, 0x86, 0x04, 0x08, 0x89, 0x44, 0x24, 0x04, 0xE8, 0x8F, 0xFF, 0xFF, 0xFF, 0xC7, 0x44, 0x24, 0x04, 0x08, 0x99, 0x04, 0x08, 0xC7, 0x04, 0x24, 0x0B, 0x87, 0x04, 0x08, 0xE8, 0xBB, 0xFF, 0xFF, 0xFF, 0x8B, 0x1D, 0x00, 0x99, 0x04, 0x08, 0x89, 0xD8, 0x2D, 0xB0, 0x83, 0x04, 0x08, 0x81, 0xFB, 0xB0, 0x83, 0x04, 0x08, 0x76, 0x1C, 0xBA, 0xB0, 0x83, 0x04, 0x08, 0x90, 0x89, 0xC1, 0xC1, 0xF9, 0x1B, 0xC1, 0xE0, 0x05, 0x31, 0xC8, 0x0F, 0xB6, 0x0A, 0x83, 0xC2, 0x01, 0x31, 0xC8, 0x39, 0xDA, 0x75, 0xEA, 0x89, 0xC2, 0x32, 0x15, 0x08, 0x99, 0x04, 0x08, 0x3A, 0x15, 0xF0, 0x98, 0x04, 0x08, 0x89, 0x44, 0x24, 0x1C, 0x0F, 0x85, 0x38, 0x01, 0x00, 0x00, 0x0F, 0xB6, 0x54, 0x24, 0x1D, 0x89, 0xD1, 0x32, 0x0D, 0x09, 0x99, 0x04, 0x08, 0x3A, 0x0D, 0xF1, 0x98, 0x04, 0x08, 0x0F, 0x85, 0x1F, 0x01, 0x00, 0x00, 0x0F, 0xB6, 0x4C, 0x24, 0x1E, 0x89, 0xCB, 0x32, 0x1D, 0x0A, 0x99, 0x04, 0x08, 0x3A, 0x1D, 0xF2, 0x98, 0x04, 0x08, 0x0F, 0x85, 0x06, 0x01, 0x00, 0x00, 0x0F, 0xB6, 0x7C, 0x24, 0x1F, 0x0F, 0xB6, 0x1D, 0x0B, 0x99, 0x04, 0x08, 0x31, 0xFB, 0x3A, 0x1D, 0xF3, 0x98, 0x04, 0x08, 0x0F, 0x85, 0xEC, 0x00, 0x00, 0x00, 0x0F, 0xB6, 0x1D, 0x0C, 0x99, 0x04, 0x08, 0x31, 0xC3, 0x3A, 0x1D, 0xF4, 0x98, 0x04, 0x08, 0x0F, 0x85, 0xD7, 0x00, 0x00, 0x00, 0x0F, 0xB6, 0x1D, 0x0D, 0x99, 0x04, 0x08, 0x31, 0xD3, 0x3A, 0x1D, 0xF5, 0x98, 0x04, 0x08, 0x0F, 0x85, 0xC2, 0x00, 0x00, 0x00, 0x0F, 0xB6, 0x1D, 0x0E, 0x99, 0x04, 0x08, 0x31, 0xCB, 0x3A, 0x1D, 0xF6, 0x98, 0x04, 0x08, 0x0F, 0x85, 0xAD, 0x00, 0x00, 0x00, 0x0F, 0xB6, 0x1D, 0x0F, 0x99, 0x04, 0x08, 0x31, 0xFB, 0x3A, 0x1D, 0xF7, 0x98, 0x04, 0x08, 0x0F, 0x85, 0x98, 0x00, 0x00, 0x00, 0x0F, 0xB6, 0x1D, 0x10, 0x99, 0x04, 0x08, 0x31, 0xC3, 0x3A, 0x1D, 0xF8, 0x98, 0x04, 0x08, 0x0F, 0x85, 0x83, 0x00, 0x00, 0x00, 0x0F, 0xB6, 0x1D, 0x11, 0x99, 0x04, 0x08, 0x31, 0xD3, 0x3A, 0x1D, 0xF9, 0x98, 0x04, 0x08, 0x75, 0x72, 0x0F, 0xB6, 0x1D, 0x12, 0x99, 0x04, 0x08, 0x31, 0xCB, 0x3A, 0x1D, 0xFA, 0x98, 0x04, 0x08, 0x75, 0x61, 0x0F, 0xB6, 0x1D, 0x13, 0x99, 0x04, 0x08, 0x31, 0xFB, 0x3A, 0x1D, 0xFB, 0x98, 0x04, 0x08, 0x75, 0x50, 0x33, 0x05, 0x14, 0x99, 0x04, 0x08, 0x3A, 0x05, 0xFC, 0x98, 0x04, 0x08, 0x75, 0x42, 0x32, 0x15, 0x15, 0x99, 0x04, 0x08, 0x3A, 0x15, 0xFD, 0x98, 0x04, 0x08, 0x75, 0x34, 0x32, 0x0D, 0x16, 0x99, 0x04, 0x08, 0x3A, 0x0D, 0xFE, 0x98, 0x04, 0x08, 0x75, 0x26, 0x89, 0xFB, 0x32, 0x1D, 0x17, 0x99, 0x04, 0x08, 0x3A, 0x1D, 0xFF, 0x98, 0x04, 0x08, 0x75, 0x16, 0xC7, 0x04, 0x24, 0x0E, 0x87, 0x04, 0x08, 0xE8, 0x14, 0xFE, 0xFF, 0xFF, 0x8D, 0x65, 0xF4, 0x31, 0xC0, 0x5B, 0x5E, 0x5F, 0x5D, 0xC3, 0xC7, 0x04, 0x24, 0x1C, 0x87, 0x04, 0x08, 0xE8, 0xFE, 0xFD, 0xFF, 0xFF, 0xEB, 0xE8, 0x31, 0xED, 0x5E, 0x89, 0xE1, 0x83, 0xE4, 0xF0, 0x50, 0x54, 0x52, 0x68, 0x60, 0x86, 0x04, 0x08, 0x68, 0x70, 0x86, 0x04, 0x08, 0x51, 0x56, 0x68, 0xB0, 0x83, 0x04, 0x08, 0xE8, 0xFB, 0xFD, 0xFF, 0xFF, 0xF4, 0x90, 0x90, 0x90, 0x90, 0x90, 0x90, 0x90, 0x90, 0x90, 0x90, 0xB8, 0x07, 0x99, 0x04, 0x08, 0x2D, 0x04, 0x99, 0x04, 0x08, 0x83, 0xF8, 0x06, 0x77, 0x02, 0xF3, 0xC3, 0xB8, 0x00, 0x00, 0x00, 0x00, 0x85, 0xC0, 0x74, 0xF5, 0x55, 0x89, 0xE5, 0x83, 0xEC, 0x18, 0xC7, 0x04, 0x24, 0x04, 0x99, 0x04, 0x08, 0xFF, 0xD0, 0xC9, 0xC3, 0x90, 0x8D, 0x74, 0x26, 0x00, 0xB8, 0x04, 0x99, 0x04, 0x08, 0x2D, 0x04, 0x99, 0x04, 0x08, 0xC1, 0xF8, 0x02, 0x89, 0xC2, 0xC1, 0xEA, 0x1F, 0x01, 0xD0, 0xD1, 0xF8, 0x75, 0x02, 0xF3, 0xC3, 0xBA, 0x00, 0x00, 0x00, 0x00, 0x85, 0xD2, 0x74, 0xF5, 0x55, 0x89, 0xE5, 0x83, 0xEC, 0x18, 0x89, 0x44, 0x24, 0x04, 0xC7, 0x04, 0x24, 0x04, 0x99, 0x04, 0x08, 0xFF, 0xD2, 0xC9, 0xC3, 0x90, 0x8D, 0xB4, 0x26, 0x00, 0x00, 0x00, 0x00, 0x80, 0x3D, 0x04, 0x99, 0x04, 0x08, 0x00, 0x75, 0x13, 0x55, 0x89, 0xE5, 0x83, 0xEC, 0x08, 0xE8, 0x7C, 0xFF, 0xFF, 0xFF, 0xC6, 0x05, 0x04, 0x99, 0x04, 0x08, 0x01, 0xC9, 0xF3, 0xC3, 0x66, 0x90, 0xA1, 0xD0, 0x97, 0x04, 0x08, 0x85, 0xC0, 0x74, 0x1E, 0xB8, 0x00, 0x00, 0x00, 0x00, 0x85, 0xC0, 0x74, 0x15, 0x55, 0x89, 0xE5, 0x83, 0xEC, 0x18, 0xC7, 0x04, 0x24, 0xD0, 0x97, 0x04, 0x08, 0xFF, 0xD0, 0xC9, 0xE9, 0x79, 0xFF, 0xFF, 0xFF, 0xE9, 0x74, 0xFF, 0xFF, 0xFF, 0x90, 0x90, 0x90, 0x90, 0x55, 0x89, 0xE5, 0x5D, 0xC3, 0x8D, 0x74, 0x26, 0x00, 0x8D, 0xBC, 0x27, 0x00, 0x00, 0x00, 0x00, 0x55, 0x89, 0xE5, 0x57, 0x56, 0x53, 0xE8, 0x4F, 0x00, 0x00, 0x00, 0x81, 0xC3, 0x4D, 0x12, 0x00, 0x00, 0x83, 0xEC, 0x1C, 0xE8, 0x9B, 0xFC, 0xFF, 0xFF, 0x8D, 0xBB, 0x04, 0xFF, 0xFF, 0xFF, 0x8D, 0x83, 0x00, 0xFF, 0xFF, 0xFF, 0x29, 0xC7, 0xC1, 0xFF, 0x02, 0x85, 0xFF, 0x74, 0x24, 0x31, 0xF6, 0x8B, 0x45, 0x10, 0x89, 0x44, 0x24, 0x08, 0x8B, 0x45, 0x0C, 0x89, 0x44, 0x24, 0x04, 0x8B, 0x45, 0x08, 0x89, 0x04, 0x24, 0xFF, 0x94, 0xB3, 0x00, 0xFF, 0xFF, 0xFF, 0x83, 0xC6, 0x01, 0x39, 0xFE, 0x72, 0xDE, 0x83, 0xC4, 0x1C, 0x5B, 0x5E, 0x5F, 0x5D, 0xC3, 0x8B, 0x1C, 0x24, 0xC3, 0x90, 0x90, 0x55, 0x89, 0xE5, 0x53, 0x83, 0xEC, 0x04, 0xE8, 0x00, 0x00, 0x00, 0x00, 0x5B, 0x81, 0xC3, 0xEC, 0x11, 0x00, 0x00, 0x59, 0x5B, 0xC9, 0xC3, 0x00, 0x03, 0x00, 0x00, 0x00, 0x01, 0x00, 0x02, 0x00, 0x25, 0x73, 0x0A, 0x50, 0x6C, 0x65, 0x61, 0x73, 0x65, 0x20, 0x69, 0x6E, 0x70, 0x75, 0x74, 0x20, 0x79, 0x6F, 0x75, 0x72, 0x20, 0x66, 0x6C, 0x61, 0x67, 0x3A, 0x00, 0x25, 0x73, 0x00, 0x59, 0x6F, 0x75, 0x20, 0x61, 0x72, 0x65, 0x20, 0x72, 0x69, 0x67, 0x68, 0x74, 0x00, 0x59, 0x6F, 0x75, 0x20, 0x61, 0x72, 0x65, 0x20, 0x77, 0x72, 0x6F, 0x6E, 0x67, 0x00, 0x7E};

int v0 = 0x37A;

int main(int argc, char **argv)
{
    int v2 = 0;
    for (int i = 0; i < 0x37A; i++){
        v2 = v1[i];
        v0 = v2 ^ (v0 >> 27) ^ 32 * v0;
    }
    printf("0x%08X\n", v0);
    return 0;
}

```

最终拿到
```CPP
v4 = v0 = 0x7EEB184F
```
但是用这个值去计算的话发现不对，多跑了几次，结果好像断点数和输入不一样这个值都会变，再看看题目的意思，估计断点不能乱下了。。。那就暴力点，直接patch二进制文件，让程序把值打出来。。。

接下来就是被霍霍成一大坨的if大判定了。。。其实都是写比较，几个函数：

+ __PAIR__(word, word) 比较两个16bit的值
+ BYTE1()～BYTE3() 分别取一个32bit值的第1～3个byte（从0开始，一个32bit的value有0,1,2,3四个byte

用来比较的值也知道了，见上面的截图，这里单独按序列出来
```python
valueForCmp = (0x5B18, 0xBF, 0x38, 0x34, 0x5A, 0x99, 0x4D, 0x2E, 0x73, 0xBB, 0x4E, 0x23, 0x9F76, 0x3)
```

用来比较的值从地址**0x08049908到0x08049914**，而且0x08049914指向的是个dword，所以实际上是0x14-0x8 + 0x1 + 0x3 = 0x10，即我们要输入的是16个字符

如上，if里的14次比较都都分析的差不多了，下面就好办了，直接脚本跑出来，要注意的就是比较时候的字节序问题了，可以上下两个**valueForCmp**有什么不同
```python
#!/bin/env python3
#! -*- coding:utf-8 -*-

valueForCmp = (0x18, 0x5B, 0xBF, 0x38, 0x34, 0x5A, 0x99, 0x4D, 0x2E, 0x73, 0xBB, 0x4E, 0x23, 0x76, 0x9F, 0x3)
value = (0x4F, 0x18, 0xEB, 0x7e)

def deBreakPoint():
    resList = []
    resStr = ''
    TmpValue = 0
    for i in range(0, 16):
        TmpValue = (valueForCmp[i] ^ value[i%4])
        resList.append(chr(TmpValue))
    resStr = resStr.join(resList)
    return resStr

if __name__ == '__main__':
    res = deBreakPoint()
    print(res)

```