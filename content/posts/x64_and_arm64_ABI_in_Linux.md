---
title: "x64 and arm64 ABI in Linux"
date: 2018-06-01T22:10:20+08:00
categories: ['Security']
tags: ['ABI', 'Security']
---

## x64 and arm64 ABI in Linux

### TL;DR

Ok, it has been a few months since Arch Linux announced that it will only provide x64 image in the future. Android also has been support x64 and arm64 for a long time.  Seems that it's true that x86 is on the way of out of time. So it's time to learn some thing about the ABI in x64 and arm64.

**PS:** If you've read the ABI manual about Linux on X64/Arm64, there is no need to read this article. It's just for beginners and lazy guys who don't want to read manual.

## Language we used to study

As we know that C/C++ is the best language for us to research the ABI because they cover both "**Procedure Oriented**" and "**Object Oriented**" language. So we will use simple C/C++ console application to dig the details about ABI. Now let's rock.

## x64

First we will analysis the x64 architecture. Please view the C source code below.

```c
#include <stdio.h>

void exputs(const char *s, const char *s1, const char*s2, const char *s3, const char *s4, const char *s5, const char *s6)
{
  puts(s);
  puts(s1);
  puts(s2);
  puts(s3);
  puts(s4);
  puts(s5);
  puts(s6);
}

int main(int argc, char **argv)
{
  exputs("Zero\n", "One\n", "Two\n", "Three\n", "Four\n", "Five\n", "Six\n");
  return 0;
}

```
