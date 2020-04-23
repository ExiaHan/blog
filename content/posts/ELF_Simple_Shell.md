---
title: "ELF_Simple_Shell"
date: 2015-08-18T12:48:57+08:00
categories: ["Linux"]
tags: ["Linux", "ELF", "Shell"]
---

## 0x00 前言

之前看了ELF文件的文件格式PDF[文档](http://flint.cs.yale.edu/cs422/doc/ELF_Format.pdf)，又从[看雪论坛](http://bbs.pediy.com/showthread.php?t=191649)和[光哥](http://www.burningcodes.net/)那里看了点so加密的文章，就想试下直接对elf可执行文件加密。现做一点整理和记录，留待备用。

大概流程如下：

+ 核心代码放到指定节
+ .init section 加入解密函数decryptFunc
+ 加密程序读取编译好的elf文件，加密指定节
+ 运行时decryptFunc函数解密核心代码
+ 正确执行

## 0x01 准备工作

要对ELF可执行文件进行处理，首先需要了解ELF文件格式，具体可见上一篇Po主的渣翻译（**真的很渣，英语老师已气死**），这里对需要用到的地方再进行一次说明：

### ELF文件格式

ELF文件组织结构如下：

|Linker View|Execution View|
|:-:|:-:|
|ELF Header|ELF Header|
|......|......|
|Section Header Table|Section Header Table[Optional]|
|......|......|
|Program Header Table[Optional]|Program Header Table|
|......|......|
|Sections|Segments|
|......|......|

ELF文件ELF Header结构如下：

```cpp
......
#define EI_NIDENT 16

typedef u_int8_t u1;
typedef u_int16_t u2;
typedef u_int32_t u4;

typedef int8_t i1;
typedef int16_t i2;
typedef int32_t i4;

typedef struct {
    u1 e_ident[EI_NIDENT];
    u2 e_type;
    u2 e_machine;
    u4 e_version;
    u4 e_entry;
    u4 e_phoff;
    u4 e_shoff;
    u4 e_flags;
    u2 e_ehsize;
    u2 e_phentsize;
    u2 e_phnum;
    u2 e_shentsize;
    u2 e_shnum;
    u2 e_shstrndx;
}elf32_Header, *pElf32_Header;

#endif

```

根据文档，对于Linker，其对Section进行操作，而在装入内存执行时，进程内部数据是按照段(Segment)进行组织，所以对于非可执行的ELF文件，如gcc -c xxx.o生成的 xxx.o文件，其内部可以没有Program Header Table，但必须有Section Header Table；对于可执行文件(gcc -o xxx)，其内部可以没有Section Header Table，但是必须有Program Header Table。

因此，我们可以利用ELF Header中与section有关的字段，如e_shoff和e_shentsize，用来存放我们加解密需要的数据来供decryptFunc函数使用，如此不仅方便，还能有效避免可执行文件被IDA等静态工具分析。

##0x02 开工

-----

#### A、准备一个简单的C源程序：

+ 头文件elf32.h，包含对elf header结构的定义
+ example.c 示例文件，包含解密函数等定义

```cpp
#include "elf32.h"

#define MAXLEN 1000
#define PAGESIZE 4096

//The function to decrypt the segment
__attribute__((constructor)) void mydecrypt();

//The section that will be encrypted
__attribute((section("mysection"))) void mysecFunction();

__attribute((section("mysection_data"))) char strMySec[] = "NzTfdujpo";

unsigned long getAddress(char *strName);

int main(int argc, char **argv)
{
    printf("hello world\n");
    mysecFunction();
    return 0;
}

void mysecFunction()
{
    ......
	//deal data in section mysection_data
	//then print it
	......
    return;
}

void mydecrypt()
{
	......
	//code will decrypt data in section: mysection
	......
}

/****************************************************
* find a module's base via /proc/[pid]/maps,
* in this scenario, it will be the base addr of the
* executable elf file's base address
****************************************************/
unsigned long getAddress(char *strName);
{
    char buf[MAXLEN] = {0};
    char *sAddr = NULL;
    unsigned long uRet = 0;
    FILE *fp = NULL;

    sprintf(buf, "/proc/%d/maps", getpid());
    if (!(fp = fopen(buf, "r"))) {
        perror("Error when open file\n");
        return uRet;
    }

    while(fgets(buf, sizeof(buf), fp)) {
        if (strstr(buf, strName)) {
            sAddr = strtok(buf, "-");
            uRet = strtol(sAddr, NULL, 16);
            break;
        }
    }

    return uRet;
}

```

代码大概内容如下：

+ 一个mysection节，包含一个会被加密的函数mysecFunction()，函数会把mysectin_data里的字符数组进行处理（其实是加），打印出真正的字符串
+ 一个mysection_data节，包含一个字符数组，用于打印
+ 一个会在程序加载到内存后init阶段执行的解密函数mydecrypt()，函数
+ 一个getAddress函数，用来获取程序加载基地址(**其实这个函数在这里可以看成是多余的，因为和shared object不一样，可执行文件的程序地址是写死的，写这里是作为备忘，总归会碰到加密so的**)

对源码进行编译生成目标文件：

```bash
$ gcc -m32 -c -o example.o example.c
$ gcc -m32 -o example example.o -Wl,--section-start=mysection=0x08881000
```

这里需要注意的是加了编译选项-Wl,--section-start=mysection=0x08881000，原因在于：

+ so被加载进来后，通常其起始地址是按页对齐的，不同于so，elf可执行文件里代码位置是写死的，而且通常每个section里的代码在segment里并不会按页对齐，所以我们的mysection通常不会在一个页的起始处
+ 加密函数使用了mprotect活的相关内存区域的写入权限，mprotect函数修改内存权限是按页修改，所以如果不强制到某个页上，函数调用会失败

所以，指定对应节的起始地址为一个按4K对齐的地址。

#### B、加密生成的ELF可执行文件的指定区域

这里使用python脚本自动搜索完成
```bash
$ ./elfshell.py example
```
思路如下：

+ 读取ELF Header，取得Section Table，Section String Name Table
+ 遍历Section Table和Section Stringg Name Table，找到mysection节的offset[**磁盘上文件中位置**]，addr[**加载到内存后的位置**]，size等信息
+ 利用offset和size，加密mysection的内容
+ 把mysection的addr，size填入elf-header结构的e_shoff,e_shentsize中，供解密函数mydecrypt()使用[**前面已经说过，elf-header结构里与section相关的信息在执行时是不会是用的**]

#### C、执行

完成上述步骤后，使用IDA静态逆向elf文件example，会提示无法正确解析，但是可以正常执行，执行流程：

+ 首先，constructor属性的mydecrypt()会执行，调用getAddress得到example基址
+ mydecrypt()读取elf-header结构的e_shoff和e_shentsize，找到mysection
+ 解密mysection
+ 程序开始执行

代码：[github](https://github.com/ExiaHan/ELF_Shell)