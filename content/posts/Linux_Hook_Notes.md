---
title: "Linux Hook Notes"
date: 2015-07-29T09:59:25+08:00
categories: ['Linux']
tags: ['Linux', 'Hook', 'GDB', 'Ptrace']
---

## Foreword

For have read a article in WebSite [code project](http://www.codeproject.com/Articles/33340/Code-Injection-into-Running-Linux-Application), and learned lots of from it. So make a memo for later consulting.

## Start

First we will make some file for the lab.So we need three file, a .h file, two .c file, the code showed below.

```cpp
//file mylib.h
#ifndef _MYLIB_H_
#define _MYLIB_H_

extern void myprint(const char *str);
#endif
```


```cpp
//file mylib.c
#include <stdio.h>
#include <stdlib.h>
#include <sys/types.h>
#include <unistd.h>
#include "mylib.h"

void myprint()
{
   static unsigned counter = 0;
   counter++;
   printf("%d, pid (%d)", counter, getpid());
   return;
}
```

```cpp
//mymain.c
#include <stdio.h>
#include <stdlib.h>
#include <sys/types.h>
#include <unistd.h>
#include "mylib.h"


#define TRUE 1

int main(int argc, char **argv)
{
    while(TRUE) {
        myprint();
        printf("Going to sleep\n");
        sleep(3);
        printf("Wake up\n");
    }
    return 0;
}

```

```cpp
//file myinject.c
#include <stdlib.h>
#include "mylib.h"

void injection()
{
    myprint();
    system("date");
}
```


For in the above, the file mylib.h, mylib.c is used to generate our test app, and the file myinject.c is used to do our injdect, that is we want hook the myprint function, when myprint is invoke, the function injection will be invoked indeed.

create the three file, and compile like below.

```bash
$ gcc -g -Wall  -fPIC -shared mylib.c -o libmylib.so
$ gcc -g -lmylib -L ./ mymain.c -o app
$ gcc -Wall myinject.c -c -o myinject.o
$ export LD_LIBRARY_PATH=.:[your path where the libmylib.so in]
```

For Address-Independent, we add the option -fPIC when build mylib.c to libmylib.so

what you should care is the -lmylib is fixed to the libmylib, so remember to change it if you name your file another name.

## Let's Rock Our App

Now we can do with the elf file what we just generated.

#### 1.Start our app

Run our app:

```bash
$ ./app
```

Now we will see the PID via the output. Then we will use gdb to attach the app, via the pid:

```bash
$ ./app
1, pid (5832)Going to sleep
Wake up
2, pid (5832)Going to sleep
Wake up
3, pid (5832)Going to sleep
```

```bash
$ sudo gdb -p PID[5832]
```

After do this, we now attached to the app:

```bash
$ sudo gdb -p 5832
GNU gdb (GDB) 7.9.1
Copyright (C) 2015 Free Software Foundation, Inc.
License GPLv3+: GNU GPL version 3 or later
......
Attaching to process 6618
Reading symbols from /home/exiahan/Developments/Linux/hijack/app...done.
Reading symbols from ./libmylib.so...done.
Reading symbols from /usr/lib32/libc.so.6...(no debugging symbols found)...done.
Reading symbols from /lib/ld-linux.so.2...(no debugging symbols found)...done.
0xf7722c10 in __kernel_vsyscall ()
(gdb)
```

#### 2.Operate and Analysis our app/myinject.o
First we load our myinject.o into the app's process space.
```bash
(gdb)call open("myinject.o", 2)
(gdb)call mmap(0, Size, 1|2|4, 2, ReturnFrom-open, 0)
```

In the open ,the value 2 represent the O_RDWR(Read and Write),then we use mmap function, to put the myinject.o in to the app's process space, the first and second specify the size of myinject.o, if first is NULL(0), the os justify the start address of the file, the **Size** can be get via:

```bash
$ ls -l myinject.o
```

Then the 1|2|4 represent PROT_READ | PROT_WRITE | PROT_EXEC, so we can for we will write and execute it. The follow 1 represent the **MAP_SHARED** **[But caution, we can use 2, means MAP_PRIVATE, it also works, the MAP_SHARED  will cause the myinject.o in the file also be changed if we write something, yes, actually we will.][好吧，重要的地方说中文，flags最好设置成MAP_PRIIVATE，这样mmap文件到内存后，修改不会影响disk文件，按照英文的文档用SHARED的话，下次要重新生成myinject.o，不然文件被改，怎么做都不会对的，对，我就是蠢货，没好好看mmap的文档，倒腾一天]**. Also the ReturnFom-open is the return value after we call the open(), after do the above two steps, what we will see is like this:

```cpp
(gdb) call open("myinject.o", 2)
$1 = 3
(gdb) call mmap(0, 1044, 1|2|4, 2, 3, 0)
$2 = -143544320
(gdb)
```

- - -

Then, we will use readelf to analyse our the app and myinject.o

```bash
[exiahan@Veda hijack]$ readelf -r app

重定位节 '.rel.dyn' 位于偏移量 0x3c8 含有 1 个条目：
 Offset     Info    Type            Sym.Value  Sym. Name
08049844  00000506 R_386_GLOB_DAT    00000000   __gmon_start__

重定位节 '.rel.plt' 位于偏移量 0x3d0 含有 5 个条目：
 Offset     Info    Type            Sym.Value  Sym. Name
08049854  00000207 R_386_JUMP_SLOT   00000000   sleep
08049858  00000307 R_386_JUMP_SLOT   00000000   myprint
0804985c  00000407 R_386_JUMP_SLOT   00000000   puts
08049860  00000507 R_386_JUMP_SLOT   00000000   __gmon_start__
08049864  00000607 R_386_JUMP_SLOT   00000000   __libc_start_main
```

we can see that the offset of the function we **myprint** want to hook is **0x08049858**, it is the address where it location in the file. But when app be loaded into memory, the the address will change. So how the app still can invoke myprint via call *0x08049858, it is the plt(Procedure Linker Table) and the GOT(Global Offset Table), when app is going to invoke myprint, it will call the *0x08049858(in fact it's in the GOT), what in the address is a pointer that pointer to some procedure in plt, via the procedure, will found the real address of myprint, then the address will be put in 0x08049858, in the furture, when invoke myprint, simple call *0x08049858 will work.

So, when the app is running, we can change the value in 0x08049858, then we done the hook operation of the function **myprint**.

For do this, we first get the Base Address of myinject.o in app's process space:

```bash
$ cat /proc/5832/maps | grep myinject

f771b000-f771c000 rwxp 00000000 fe:03 20342761  [path]/myinject.o
```

Now we know that the base-addr is 0xf774f000, then use readelf to get other info we need, like the offset of function injection(), and others that need to be relocated.

```bash
$ readelf -s myinject.o

Symbol table '.symtab' contains 12 entries:
   Num:    Value  Size Type    Bind   Vis      Ndx Name
     0: 00000000     0 NOTYPE  LOCAL  DEFAULT  UND
     1: 00000000     0 FILE    LOCAL  DEFAULT  ABS myinject.c
     2: 00000000     0 SECTION LOCAL  DEFAULT    1
     3: 00000000     0 SECTION LOCAL  DEFAULT    3
     4: 00000000     0 SECTION LOCAL  DEFAULT    4
     5: 00000000     0 SECTION LOCAL  DEFAULT    5
     6: 00000000     0 SECTION LOCAL  DEFAULT    7
     7: 00000000     0 SECTION LOCAL  DEFAULT    8
     8: 00000000     0 SECTION LOCAL  DEFAULT    6
     9: 00000000    30 FUNC    GLOBAL DEFAULT    1 injection
    10: 00000000     0 NOTYPE  GLOBAL DEFAULT  UND myprint
    11: 00000000     0 NOTYPE  GLOBAL DEFAULT  UND system
```

We see that the offset of function injection() is 0x0(the value).

```bash
$ readelf -S myinject.o
共有 13 个节头，从偏移量 0x20c 开始：

节头：
  [Nr] Name              Type            Addr     Off    Size   ES Flg Lk Inf Al
  [ 0]                   NULL            00000000 000000 000000 00      0   0  0
  [ 1] .text             PROGBITS        00000000 000034 00001e 00  AX  0   0  1
  [ 2] .rel.text         REL             00000000 0001ec 000018 08   I 11   1  4
  [ 3] .data             PROGBITS        00000000 000052 000000 00  WA  0   0  1
  [ 4] .bss              NOBITS          00000000 000052 000000 00  WA  0   0  1
  [ 5] .rodata           PROGBITS        00000000 000052 000005 00   A  0   0  1
  [ 6] .comment          PROGBITS        00000000 000057 000012 01  MS  0   0  1
  [ 7] .note.GNU-stack   PROGBITS        00000000 000069 000000 00      0   0  1
  [ 8] .eh_frame         PROGBITS        00000000 00006c 000038 00   A  0   0  4
  [ 9] .rel.eh_frame     REL             00000000 000204 000008 08   I 11   8  4
  [10] .shstrtab         STRTAB          00000000 0000a4 00005f 00      0   0  1
  [11] .symtab           SYMTAB          00000000 000104 0000c0 10     12   9  4
  [12] .strtab           STRTAB          00000000 0001c4 000025 00      0   0  1
Key to Flags:
  W (write), A (alloc), X (execute), M (merge), S (strings)
  I (info), L (link order), G (group), T (TLS), E (exclude), x (unknown)
  O (extra OS processing required) o (OS specific), p (processor specific)
```

Now,we know that the offset .text about the file base is 0x34, the .rodata is 0x52, which the string "date" in.
So, the entry addr of injection() is:

```bash
0xf771b000 + 0x34 = 0xf771b034
```

So we change the value in 0x08049858 to 0xf771b034:

```bash
(gdb)set *0x08049858 = 0xf771b034
```

But, only change the address of myprint() to injection() is not enough, there are some other entries need to be relocated, below:

```bash
$ readelf -r myinject.o

重定位节 '.rel.text' 位于偏移量 0x1ec 含有 3 个条目：
 Offset     Info    Type            Sym.Value  Sym. Name
00000007  00000a02 R_386_PC32        00000000   myprint
0000000f  00000501 R_386_32          00000000   .rodata
00000014  00000b02 R_386_PC32        00000000   system

重定位节 '.rel.eh_frame' 位于偏移量 0x204 含有 1 个条目：
 Offset     Info    Type            Sym.Value  Sym. Name
00000020  00000202 R_386_PC32        00000000   .text
```

As we invoke myprint() and system() in injection(), we give a string arguement "date" that stored in .rodata.
Because the myinject.o is load in to mem manual, so we have to do the relocation as the linker.
From above we see that there two kinds of relocation:

+ R_386_PC32
+ R_386_32

The R_386_PC32 means that the relocation will related to the value of eip(so called the program counter), simply, the offset is the offset that the real address of entries to the following address of the current instruction address,let **S, A, P** represent the real address in runtime, the addend num and the address that will be affect, in intel x86, the A is store in P, the calucator is:

<center>**`S-P+A`**</center>

(尼码，英语实在说不明白了，大概意思就是计算的偏移和实际运行是的eip相关，计算方法大概是实际所在的地址相对于读到这条指令后，紧接着的指令的地址的偏移。计算方法就是**S、A、P**分别表示的事该条目运行时的**实际所在地址**， **附加数（有时候一个段，如.rodata要重定位的不止一个，一个个排的话肯定需要一个附加数，通常是2,4,8,结构体另说）**，**将被影响的地址，即此地址的值要被修改成真正的地址所在(通过readelf 和加载基地址联合重定位节.rel.text里的偏移得到)**).

The R_386_32 is the absolute offset, just calc via base-addr + offset + addend.
(直接加载后基地址加上所在段偏移和附加数就OK)。

So, we will relocation the myprint(), "date", system():

```bash
#relocate myprint()
(gdb) p *(0xf771b000 + 0x34 + 0x7)
$3 = -4
(gdb) p myprint
$4 = {void ()} 0xf771d550 <myprint>
(gdb) set *(0xf771b000 + 0x34 + 0x7) = 0xf771d550 - (0xf771b000 + 0x34 + 0x7) - 4

#relocate system()
(gdb) p *(0xf771b000 + 0x34 + 0x14)
$5 = -4
(gdb) p system
$6 = {<text variable, no debug info>} 0xf7542e00 <system>
(gdb) set *(0xf771b000 + 0x34 + 0x14) = 0xf7542e00 - (0xf771b000 + 0x34 + 0x14) - 4

#relocate "date", in .rodata section, the off of .rodata is 0x52, get from above readelf -S myinject.o
(gdb) p *(0xf771b000 + 0x34 + 0xf)
$7 = 0
(gdb) set *(0xf771b000 + 0x34 + 0xf) = 0xf771b000 + 0x52
(gdb)
```

#### 3.Final

After do this, we will get the result like below:

```bash
1, pid (5832)Going to sleep
Wake up
2, pid (5832)Going to sleep
Wake up
3, pid (5832)Going to sleep
2015年 07月 29日 星期三 21:17:17 CST
4, pid (5832)Going to sleep
Wake up
2015年 07月 29日 星期三 21:17:20 CST
5, pid (5832)Going to sleep
Wake up
2015年 07月 29日 星期三 21:17:23 CST
6, pid (5832)Going to sleep
Wake up
2015年 07月 29日 星期三 21:17:26 CST
7, pid (5832)Going to sleep
```

## Summary
So we use gdb(ptrace) to attach a running process, and manual pull a .o file into its process space, then we replace the myprint@got.plt to the address of injection(), then we do the relocation for myinject.o as a linker(yeah, cause when the .o file to a executable file, the linker do the relocation, so we have to do it manually). Then we get our goal.

And in this lab, what we should learn is:

+ ELF File Format
+ ELF Relocation
	* R_386_PC32
	* R_386_32
+ Usage of GDB [-p PID]
+ Usage of mmap [MAP_PRIVATE/SHARED] and open
+ Usage of readelf [-r -s -S]
+ Usage of objdump [-s -S]