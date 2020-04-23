---
title: "Study ELF File Format"
date: 2015-07-31T16:30:21+08:00
categories: ['Linux']
tags: ['ELF']
---

## 前言

为了对Hook， Relocation， 以及一些程序保护技术有更深的了解，正在学习Linux ELF文件格式，所以找到了ELF_Format的文档，这里把里面的一些内容做一下翻译同时记录下。如有错误还请指出，共同学习一起进步啊鲁～ლ(╹◡╹ლ)

## 正文

### 简介

ELF是Executable and Linking Format的简称，作为Application Binary Interface(ABI)的一部分，是Unix System Laboratories(USL)推出和制定的，是Unix、Linux以及一些类Unix系统使用的一套可执行文件结构标准，Linux下常见的.o,.so以及可执行文件都是ELF文件格式。

ELF文件格式出现的目的是为不同架构的机器提供一个较为统一的规范，减少重新编码、编译的工作量。

ELF文件在Linux中有三种类型：

+ Relocatable File: 可重定位文件，承载代码(说成指令更准确)和数据，可以同其他的**.o**文件一起经由链接器生成Executable File(可执行文件)或者Share Object file(共享对象文件，或者叫动态链接库)。
+ Executable File：可执行文件，可以被执行，文件中的包含了要被执行的指令和数据等，以及指明了如何创建一个程序的进程镜像。
+ Shared Object File：共享对象文件，类似于Windows下的Dynamic Link Library(动态链接库)，共享对象文件可以有如下两种Link环境:
	- 同其他的Relocatable File或者Shared Object File组成一个新的Object File文件
	- 通过**动态链接**的方式，同其他的Shared Object File以及Executable File组成进程镜像。

### File Format(文件结构)

ELF的Object File参与了程序的Link和执行，不同的阶段，其文件中的结构略有不同，下表给出了组织结构：

| Link View | Execution View |
|:---:|:---:|
| ELF Header | ELF Header |
| Program header table(optional)  | Program header table |
| Section 1 | Segment 1 |
| ... | ... |
| Section n | Segment n |
| ... | ... |
| Section header table | Section header table(optional) |

- - -

+ ELF Header 包含整个文件的路线图，描述了整个的文件的组成结构
+ Section 包含了说有Link需要的数据，如指令(instruction)，数据(data)，符号表(Symbol Table)，重定位信息(Relocation Infomation)
+ Program header table 告诉系统如何创建程序的进程镜像。*用于创建进程镜像的文件必有Program header table，相反，重定位文件可以没有*
+ Section header table 包含了描述文件中各个section的信息，每个Section在表中都有一个对应的条目，其中包含了Section Name，Size等信息。*linking过程中文件必须提此表，其他情形，如已经被链接一个可执行文件可以不需要。*

-----

**（需要注意的是，ELF文件的结构并不一定是上面表中所列的顺序，事实上除了ELF Header一定在文件头部外，其他部分的顺序都是可变的）**

-----

### Data Representation(数据表述)

- - -

为了达到适应不同平台的目的，Object File已一种平台无关的格式存放一些控制数据，从而达到用一种统一的方式定义和翻译Object File，余下的部分数据使用目标处理器(即平台架构)编码方式进行编码。
下面是ELF文件定义用到的32Bit-data-types:

| Name | Size | Alignment | Purpose |
|:--:|:--:|:--:|:--:|
| Elf32_Addr | 4 | 4 | Unsigned program address |
| Elf32_Half | 2 | 2 | Unsigned medium integer |
| Elf32_Off | 4 | 4 | Unsigned file ofset |
| Elf32_Sword | 4 | 4 | Signed large integer |
| Elf32_Word | 4 | 4 | Unsigned large integer |
| unsigned char | 1 | 1 | Unsigned small integer |

Object File中定义的数据结构都遵从“自然”的大小和对齐方式(Po主理解为平台对应)，如果有需要，可以显示指定对齐方式，以保证如4 Bytes的对象一定是4 Bytes对齐的，即该对象所占存储大小是4 Bytes的倍数。举例来说，一个包含Elf32_Addr的成员的数据结构将被4 Bytes对齐。

为了可移植性(Portability)，ELF没有位字段(bit-field)。

- - -

### ELF Header(文件头)

有些Object File的控制结构(Control Structures)可以增长，因为ELF Header包含了它们的实际大小。如果Object File的文件格式改变，程序可能会遇到控制结构与期望的大小不相符的情况。在这种情况下程序可能会忽略掉“多余”(因为比期望的大小大，可能会多出一些描述数据)的信息；对于“遗失”(因为比期望的大小小)的信息的处理方法依赖于上下文，并且如果定义extensions的话，将在其中被规定相关处理方式。

**ELF Header**

```cpp
#define EI_NIDENT 16
typedef struct{
	unsignedchar  	e_ident[EI_NIDENT];
	Elf32_Half		e_type;
	Elf32_Half		e_machine;
	Elf32_Word		e_version;
	Elf32_Addr		e_entry;
	Elf32_Off	 	e_phoff;
	Elf32_Off	 	e_shoff;
	Elf32_Word		e_flags;
	Elf32_Half		e_ehsize;
	Elf32_Half		e_phentsize;
	Elf32_Half		e_phnum;
	Elf32_Half		e_shentsize;
	Elf32_Half		e_shnum;
	Elf32_Half		e_shstrndx;
}Elf32_Ehdr;
```

+ e_ident 标记整个文件为Object File并且提供平台无关的数据用来解码和翻译整个文件内容。
+ e_type 说明具体是哪一种Object File：

```cpp
enum Elf32_Type{
	ET_NONE = 0x0;
	ET_REL = 0x1;
	ET_EXEC = 0x2;
	ET_DYN = 0x3;
	ET_CORE = 0x4;
	ET_LOPROC = 0xff00;
	ET_HIPROC = 0xffff;
};
```
Core File的内容没有被制定，ET_CORE作为保留字存在。ET_LOPROC/HIPROC两者作为处理器指定语义的保留字，两者可兼或(ET_LOPROC | ET_HIPROC)，其他的一些没有用到的类型同样作为保留字，将来可能用来关联一些新的必要的文件类型。

+ e_machine 指定平台架构：

```cpp
enum Elf32_Machine{
	EM_NONE = 0x0;
	EM_M32 = 0x1;
	EM_SPARC = 0x2;
	EM_386 = 0x3;
	EM_68K = 0x4;
	EM_88K = 0x5;
	EM_860 = 0x7;
	EM_MIPS = 0x8;
};
```

其他的值作为保留字来关联将来可能出现的新平台架构，处理器定制(Processor-specific)ELF名使用机器平台名来区分它们，举例来说：有一个在平台EM_386上的 e_flag(会以EF_开头) 为 WIDGET，那么，这个flag的将会写成 EF_386_WINDGET。

+ e_version 指定ELF的文件版本，现在只有两个值：

```cpp
enum Elf32_Version{
	EV_NONE = 0x0;
	EV_CURRENT = 0x1;
};
```

目前一般情况下值是1，但一些特殊情况下也会是一些其他的数值(如有扩展的ELF会用一个大数，或者当需要用不同的值来反应当前的版本时)。

+ e_entry 给出system first transfers control需要的用来启动进程的虚拟地址，如果没有入口，那么值是0

***(bytes)表示字节为单位***

+ e_phoff 给出program header table的offet(bytes)，如果没有，那么是0
+ e_shoff 给出section header table的offset(bytes)，如果没有，那么是0
+ e_ehsize ELF Header的size(bytes)，如果没有，那么是0
+ e_phentsize Program Header Table中每个条目(entry)的size(bytes)，所有的条目大小相同
+ e_phnum Program Header Table中条目(entry)的数目，因此phentsize × phnum即为整个表的大小(bytes)；如果没有Program Header Table，那么值为0
+ e_shentsize Section Header Table中每个条目(entry)的size(bytes)，所有条目大小相同
+ e_shnum Section Header Table中条目(entry)的数目，因此shentsize × shnum即为整个表的大小(bytes)；如果没有Section Header Table，那么值为0
+ e_shstrndx 存放一个与Section name string table相关的索引(index)，如果没有section name table，那么值为SHN_UNDEF

####ELF Identification(ELF 定义)

如前所说，ELF 提供了不同种类的文件框架来适应不同的处理器、数据编码以及不同类型的机器。为了支持整个Object File家族，ELF文件起始处的一连串字节(**elf32_header.e_ident[]**)指出了如何做到平台无关的解析文件(即不用在意是在哪种处理器上做的探询，同时也和文件剩余内容无关)。

ELF头部(Object File同样)起始的几个字节与e_ident[]的成员一致：

*e_ident[] Identification Index(e_ident[]数组中标识的索引值)：*

|索引名|值|用途|
|:-:|:-:|:-:|
|EI_MAG0|0x0|文件标识|
|EI_MAG1|0x1|文件标识|
|EI_MAG2|0x2|文件标识|
|EI_MAG3|0x3|文件标识|
|EI_CLASS|0x4|文件类型|
|EI_DATA|0x5|数据编码方式|
|EI_VERSION|0x6|文件版本|
|EI_PAD|0x7|填充字节起始处|
|EI_NIDENT|0x10|e_ident[]的大小|

上面列出的是e_ident[index]中每个index对应的值，与索引对应位的值定义如下：

+ EI_MAG0～EI_MAG3 文件魔数，标识文件为ELF文件，值依次为**0x7F、'E'、'L'、'F'**
+ EI_CLASS 标识文件类型

	+ ELFCLASSNONE &emsp;0x0 &emsp; 无效文件
	+ ELFCLASS32 &emsp;&emsp;&emsp;0x1 &emsp; 32bit文件
	+ ELFCLASS64 &emsp;&emsp;&emsp;0x2 &emsp; 64bit文件

ELF文件格式被定义为对于各种不同大小(这里是说如果机器cpu地址线等大小的不同)的机器来说都是便携的，而不用把大型机的文件形式强加于小型机上。ELFCLASS32类型可以支持虚拟地址最大到4GiB(2^32 Bytes),它使用的其他定义和之前已经描述到的相同。

ELFCLASS64是为64位架构保留的，出现在这里仅仅是为了体现未来可能的变化方向，但是64位的定义还没被定义。其他类型将按需被定义，以适应不同的基本类型和大小的Object File。

+ EI_DATA 指定编码方式，现在有如下三种：

	+ ELFDATANONE &emsp; 0x0 &emsp; 无效数据编码
	+ ELFDATA2LSB &emsp; 0x1 &emsp; 小端(低位在低地址)
	+ ELFDATA2MSB &emsp; 0x2 &emsp; 大端(低位在高地址)

+ EI_VERSION 指定ELF版本号，现在必须是EV_CURRENT(0x1)
+ EI_PAD 标志着e_ident[]中未使用字节的起始索引，现阶段作为保留字节全为0x0，以后可能会有其他用途。
+ EI_NIDENT 表示e_ident[]的结束位置，这也是固定的，因为e_ident[]长度是16，所以为0x16

一个文件的编码方式制定了如何解析文件中的基本对象。就像上面说到的，ELFCLASS32类型的文件使用的对象占用的字节数(bytes)有1Byte，2Bytes和4Bytes。在这种顶一下，数据对象的表示方法如下所示：

+ ELFDATA2LSB

ELFDATA2LSB编码方式指定了使用补码，并且最低位有效字节(the least significant byte)放在最低的地址上(小端，Intel主机序)


|字节编号|0x0|0x1|0x2|0x3|
|-:|:-:|:-:|:-:|
|0x01|0x1|
|0x0102|0x02|0x01|
|0x01020304|0x04|0x03|0x02|0x01|

+ELFDATA2MSB

ELFDATA2MSB编码方式指定了使用补码，并且最高位有效字节(the least significant byte)放在最低的地址上(大端，网络序)


|字节编号|0x0|0x1|0x2|0x3|
|-:|:-:|:-:|:-:|
|0x01|0x1|
|0x0102|0x01|0x02|
|0x01020304|0x01|0x02|0x03|0x04|

####Machine Infomation(机器信息)

**32Bit Intel Archtecture**

根据e_ident[]中的标识符定义，Intel IA32架构要求如下的值

|Position|Value|
|:-|:-|
|e_ident[EI_CLASS]|ELFCLASS32|
|e_ident[EI_DATA]|ELFDATA2LSB|

驻留在ELF Header中用来标识处理器的e_machine必须是**EM_386**

e_flags成员存放与文件相关的按位标识的flags，32Bit Intel Archtecture没有定义flags，所以为0。

####Sections(节)

一个Object File使用一个Section Header Table(节区头表)定位文件中所有的Sections(节)。节区头表是一个由Elf32_Shdr结构体组成的数组。一个节区头表索引(section header table index)相当于这个数组的下标。Elf32_Ehdr.e_shoff给出了从文件起始到节区头表的偏移；Elf32_Ehdr.e_shnum指出节区头表中一共有多少个条目；Elf32_Ehdr.shentzise指出每个条目的大小(其实每个大小都一样)。

有一些节区头表索引值是保留的，一个Object File不会有与这些索引对应的节：

|Name|Value|
|:-|-:|
|SHN_UNDEF|0x0|
|SHN_LORESERVE|0xFF00|
|SHN_LOPROC|0xFF00|
|SHN_HIPROC|0xFF1F|
|SHN_ABS|0xFFF1|
|SHN_COMMON|0xFFF2|
|SHN_HIRESERVE|0xFFFF|

+ SHN_UNDEF &emsp; 用来标记一个未定义的、丢失的、无关的、或者其他无意义的节引用(Section Reference)。如一个符号(symbol) “defined” 和 索引Section Number SHN_UNDEF相关，那么它是一个未定义的符号
+ SHN_LORESERVE &emsp; 保留索引的下界
+ SHN_LOPROC &emsp;
+ SHN_HIPROC &emsp; 从SHN_LOPROC到SHN_HIPROC是保留给CPU 的特殊的语义。
+ SHN_ABS &emsp; 对应引用量的绝对取值，这些值不受重定位影响(*Po主理解Virtual Address写死*)。举例来说，一个与SHN_ABS节有关的符号(symbol)拥有不受重定位影响的值。
+ SHN_COMMON &emsp; 与此节相关的是一些公共符号(common symbol)，如Fortran的COMMOM或者未分配的C外部变量
+ SHN_HIRESERVE &emsp; 保留索引的上界。系统保留了从SHN_LOREVERSE到SHN_HIREVERSE的索引值。这些索引值出现在Section Header Table中，即Section Header Table中不会包含与保留索引值对应的条目。

Sections包含了一个Object File除ELF Header、Program Header Table、Section Header Table以外所有的信息。不仅如此，Object File的Sections还需要满足一些条件：

+ 每个Section只有一个Section Header来描述它。也有可能有只有Section Header但没有Section的情况。
+ 每个Section占据Object File一段连续的字节空间(也可能为空)
+ Section之间不能覆盖。不存在一个Section中的直接位于另一个Section中的情况。
+ 一个Object File中可能有无效(inactive)空间(原因在于对齐，硬盘上是512B，内存上是4K)，即文件中各个Headers和Sections可能不会使用到文件的每个字节空间。无效数据所表示的内容是未定义的。

**Section Header(节头)**

Section Header 的结构化定义如下：

```cpp
typedef struct {
	Elf32_Word	sh_name;
	Elf32_Word	sh_type;
	Elf32_Word	sh_flags;
	Elf32_Addr	sh_addr;
	Elf32_Off 	sh_offset;
	Elf32_Word	sh_size;
	Elf32_Word	sh_link;
	Elf32_Word	sh_info;
	Elf32_Word	sh_addralign;
	Elf32_Word	sh_entsize;
}Elf32_Shdr;
```

+ sh_name 指出其所在Section的名字，其值是一个index，指向Section Header String Table中的对应条目(一个以'\0'结尾的字符串，C风格字符串)
+ sh_type 归类总结本段内容和语义
+ sh_flags 按bit标出一些属性杂项
+ sh_addr 如果所在Section将出现在一个进程镜像中，则本字段指出本Section第一个Byte应占据的Address，否则为0
+ sh_offset 从文件起始到这个Section第一个Byte的偏移，如果节类型是SHT_NOBITS，那么此种节不占空间，此时sh_offset仅是一种概念性的表示
+ sh_size 指出所在Section的大小(bytes)，除非节类型是SHT_NOBITS，否则节在文件中占据sh_size描述的字节大小。一个SHT_NOBITS的sh_size可能不是0，但是缺不会在文件中占据字节空间
+ sh_link 指出节区头表链接，具体解释依赖具体节区类型
+ sh_info 给出一些额外信息，具体解释依赖具体节区类型
+ sh_addralign 许多节区都有地址对齐约束(address alignment constraints)，举例来说：如果一个节存放的是双字(double word)，那么系统必须保证整个节区都是按照Double Word方式对齐。因此，**sh_addr对sh_addralign取模结果必为0**。现阶段仅有0，和2的整数幂是允许的数值，如果值为0或1，则表示没有对齐限制
+ sh_entsize 一些节区存放一张条目大小固定的表(a table of fixed-size entries)，如符号表。对于这种表，sh_entsize给出条目的大小。如果没有此种表，则为0

*sh_type的类型如下*

|Name|Value|
|:-|-:|
|SHT_NULL|0x0|
|SHT_PROGBITS|0x1|
|SHT_SYMTAB|0x2|
|SHT_STRTAB|0x3|
|SHT_RELA|0x4|
|SHT_HASH|0x5|
|SHT_DYNAMIC|0x6|
|SHT_NOTE|0x7|
|SHT_NOBITS|0x8|
|SHT_REL|0x9|
|SHT_SHLIB|0xA|
|SHT_DYNSYM|0xB|
|SHT_LOPROC|0x70000000|
|SHT_HIPROC|0x7FFFFFFF|
|SHT_LOUSER|0x80000000|
|SHT_HIUSER|0xFFFFFFFF|

+ SHT_NULL 标记此Section Header为非活动(inactive)头，不与任何Section关联，此Section Header中其他字段值无定义
+ SHT_PROGBITS 包含由程序定义的信息，其格式和意义都由程序唯一指定
+ SHT_SYMTAB 存放一张符号表，现阶段一个Object File每种类型的Section仅能有一个，这个限制在未来可能被解除。SHT_SYMTAB提供用于链接(link, ld)的符号，不过也能用于动态链接。
+ SHT_STRTAB 存放一张String 表，一个Object File可能有多个string table sections
+ SHT_RELA 存放用于重定位的条目，并且**明确附加数(with explicit addends)**，如用于32bit Object File的Elf32_Rela重定位类型。**同样，一个Object File可能有多个重定位Section**
+ SHT_HASH 存放符号哈希表(symbol hash table)，所有参与动态链接的对象都必须包含一张符号哈习表，现阶段一个Object File只能包含一张symbol hash table，以后限制可能会解除
+ SHT_DYNAMIC 存放用于动态链接的信息，现阶段一个Object File只能包含一个SHT_DYNAMIC节，以后可能解除此限制
+ SHT_NOTE 存放以某种方式标记文件的信息
+ SHT_NOBITS 不占用文件空间，其他类似SHT_PROGBITSS，尽管不占用文件空间，但是其Section Header中的sh_offset成员放有抽象的文件偏移
+ SHT_REL 存放用于重定位的条目，但没有**明确附加数(without explicit addends)**，如用于32Bit Object File的Elf32_Rel重定位类型，一个Object File可能含有多个SHT_REL节
+ SHT_SHLIB 保留的，但是没有指明其意义。带有此节的程序可能没有遵守ABI(Application Binary Interface)
+ SHT_DYNSYM SHT_SYMTAB标识的Section作为一个完整的符号表，可能包含很对动态链接不需要的符号。因此有些Object File可能还会包含一个SHT_DYNSYM节，用来存放一个动态链接符号(dynamic linking symbols)的最小集合，以节省空间
+ SHT_LOPROC 从SHT_LOPROC到SHT_HIPROC作为保留的处理器定义语义节(Processor-Specifier Semantics)
+ SHT_HIPROC 从SHT_LOPROC到SHT_HIPROC作为保留的处理器定义语义节(Processor-Specifier Semantics)
+ SHT_LOUSER 指出为应用程序保留的Index的下界
+ SHT_HIUSER 指出为应用程序保留的Index的上界，SHT_LOUSER到SHT_HIUSER之间的Section Types可以被程序使用，而且不会和现在或者将来系统定义的节产生冲突

其他的一些Section Type值作为保留字，就像之前提到的，索引为SHN_UNDEF(0x0)的Section Header可以存在，即使SHN_UNDEF标记未定义的节区关系(Section Reference)，SHN_UNDEF标记的Section，其在Section Header Table里对应的Elf32_Shdr结构体条目中所有成员的值均为0x0:

|Name|Value|Note|
|:-|:-:|:-|
|sh_name|0x0|No name|
|sh_type|SHT_NULL|Inactive|
|sh_flags|0x0|No flags|
|sh_addr|0x0|No address|
|sh_offset|0x0|No file offset|
|sh_size|0x0|No size|
|sh_link|SHN_UNDEF|No link information|
|sh_info|0x0|No auxiliary informatioon|
|sh_addralign|0x0|No alignment|
|sh_entsize|0x0|No entries|

*sh_flags类型如下*

|Name|Value|
|:-|-:|
|SHF_WRITE|0x1|
|SHF_ALLOC|0x2|
|SHF_EXECINSTR|0x4|
|SHF_MASKPROC|0xF0000000|

这些flags通过对应的bit是否置1控制，为1即打开，0为关闭，所有未定义属性皆为0

+ SHF_WRITE 所在Section运行时可写
+ SHF_ALLOC 所在节运行时占据进程内存空间(因为有些控制节运行时不占据进程空间，所以这一位对于这些节来说是关闭)
+ SHF_EXECINSTR 所在节包含可执行的机器指令
+ SHF_MASKPROC 为处理器特定语义保留

*sh_link和sh_info给出的信息依赖于所在节*

|sh_type|sh_link|sh_info|
|:-|:-|:-|
|SHT_DYNAMIC|The section header index of the string table used by entries in the section.|0x0|
|SHT_HASH|The section header index of the string table to which the hash table applies.|0x0|
|SHT_REL </br>SHT_RELA|The section header index of the associated symbol table.|The section header index of the section to which the relocation applies.|
|SHT_SYMTAB </br>SHT_DYNSYM|The section header index of the associated string table.|One greater than the symbol table index of the last local symbol (binding STB_LOCAL).(最后一个局部符号的符号表索引值加一)|
|Other|SHN_UNDEF|0x0|

### 特殊节(Special Sections)