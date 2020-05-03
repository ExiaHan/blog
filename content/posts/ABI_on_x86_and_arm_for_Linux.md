---
title: "ABI on x86 and arm for Linux"
date: 2018-06-01T22:10:20+08:00
categories: ['Security']
tags: ['ABI', 'Security']
---

It's very important to know the ABI or it may be very hard to do reversing engineering.
This article will give you a glimpse about the x86's and arm's ABI for Linux.

To investigate the ABI, we need some sample code. Here we use below C source code.

```C
#include <stdio.h>

int sum(int, int, int, int, int,
        int, int, int, int, int);

int main(int argc, char** argv) {
  int args[10] = {0};

  for (int i = 0; i < 10; i ++)
    args[i] = i;

  int total = sum(args[0], args[1], args[2], args[3],
	      args[4], args[5], args[6], args[7],
	      args
	      [8], args[9]);
  printf("Sum is %d\n", total);

  return 0;
}

int sum(int a, int b, int c, int d,
        int e, int f, int g, int h,
        int i, int j) {
  return (a + b + c + d + e + f + g + h + i + j);
}

```

Just save it as **abi.c**.

## 1. x86

Use below command to build it.

```bash
gcc -m32 -S abi.c -o abi_x86.s
```

Then remove unnecessary asm code, we get below part.

```c
.LC0:
	.string	"Sum is %d\n"
main:
	leal	4(%esp), %ecx
	andl	$-16, %esp
	pushl	-4(%ecx) // ret to libc
	pushl	%ebp
	movl	%esp, %ebp
	pushl	%edi
	pushl	%esi
	pushl	%ebx
	pushl	%ecx
	subl	$104, %esp
	...
	xorl	%eax, %eax
	leal	-68(%ebp), %edx // args
	movl	$0, %eax
	movl	$10, %ecx
	movl	%edx, %edi // args
	rep stosl
	movl	$0, -76(%ebp) // int i = 0
	jmp	.L2
.L3:
	movl	-76(%ebp), %eax
	movl	-76(%ebp), %edx
	movl	%edx, -68(%ebp,%eax,4)
	addl	$1, -76(%ebp) // i ++
.L2:
	cmpl	$9, -76(%ebp) // i < 10
	jle	.L3
	movl	-32(%ebp), %edx  // args[9]
	movl	-36(%ebp), %esi  // args[8]
	movl	-40(%ebp), %eax  // args[7]
	movl	%eax, -96(%ebp)  // args[7]
	movl	-44(%ebp), %edi  // args[6]
	movl	%edi, -100(%ebp) // args[6]
	movl	-48(%ebp), %ecx  // args[5]
	movl	%ecx, -104(%ebp) // args[5]
	movl	-52(%ebp), %eax  // args[4]
	movl	%eax, -108(%ebp) // args[4]
	movl	-56(%ebp), %edi  // args[3]
	movl	%edi, -112(%ebp) // args[3]
	movl	-60(%ebp), %edi  // args[2]
	movl	-64(%ebp), %ecx  // args[1]
	movl	-68(%ebp), %eax  // args[0]
	subl	$8, %esp
	pushl	%edx
	pushl	%esi
	pushl	-96(%ebp)
	pushl	-100(%ebp)
	pushl	-104(%ebp)
	pushl	-108(%ebp)
	pushl	-112(%ebp)
	pushl	%edi
	pushl	%ecx
	pushl	%eax
	call	sum
	addl	$48, %esp
	movl	%eax, -72(%ebp)
	subl	$8, %esp
	pushl	-72(%ebp)
	leal	.LC0@GOTOFF(%ebx), %eax
	pushl	%eax
	call	printf@PLT
	addl	$16, %esp
	movl	$0, %eax
	leal	-16(%ebp), %esp
	popl	%ecx
	popl	%ebx
	popl	%esi
	popl	%edi
	popl	%ebp
	leal	-4(%ecx), %esp
	ret
sum:
	pushl	%ebp
	movl	%esp, %ebp
	movl	8(%ebp), %edx
	movl	12(%ebp), %eax
	addl	%eax, %edx
	movl	16(%ebp), %eax
	addl	%eax, %edx
	movl	20(%ebp), %eax
	addl	%eax, %edx
	movl	24(%ebp), %eax
	addl	%eax, %edx
	movl	28(%ebp), %eax
	addl	%eax, %edx
	movl	32(%ebp), %eax
	addl	%eax, %edx
	movl	36(%ebp), %eax
	addl	%eax, %edx
	movl	40(%ebp), %eax
	addl	%eax, %edx
	movl	44(%ebp), %eax
	addl	%edx, %eax
	popl	%ebp
	ret
```

Here we can see that in **main**, the **L3** and **L2**'s first two line is the **for** loop, and  **args** starts from **\-68(%ebp)**, before call **sum**, it do 10 times push operation to put all the args to the stack, the order is from **int j** to **int a**, aka. from the tail to head. Then the **sum** is called, this will cause a **ret address** also be push to the stack, so the stack grows about **10 * 4 + 4 = 44 Bytes**, here don't care about why it sub 48 bytes after call sum, since before ret, it just use **ebp** to restore the stack, which will ensure the balance.

### 1.1 Conclusion

Here we know that in x86, all args will be put on **stack**.

## 2. x86\_64

```bash
gcc -S abi.c -o abi_x86.s
```

Then we get below asm code.

```c
.LC0:
	.string	"Sum is %d\n"
main:
	pushq	%rbp
	movq	%rsp, %rbp
	pushq	%rbx
	subq	$88, %rsp
	movl	%edi, -84(%rbp)
	movq	%rsi, -96(%rbp)
	movq	%fs:40, %rax // Stack Guard, Canary
	movq	%rax, -24(%rbp)
	xorl	%eax, %eax
	movq	$0, -64(%rbp) // args
	movq	$0, -56(%rbp)
	movq	$0, -48(%rbp)
	movq	$0, -40(%rbp)
	movq	$0, -32(%rbp)
	movl	$0, -72(%rbp) // int i = 0
	jmp	.L2
.L3:
	movl	-72(%rbp), %eax // i
	movl	-72(%rbp), %edx // i
	movl	%edx, -64(%rbp,%rax,4) // args[i] = i
	addl	$1, -72(%rbp) // i ++
.L2:
	cmpl	$9, -72(%rbp) // i < 10
	jle	.L3
	movl	-28(%rbp), %r10d // args[9]
	movl	-32(%rbp), %r9d  // args[8]
	movl	-36(%rbp), %r8d  // args[7]
	movl	-40(%rbp), %edi  // args[6]
	movl	-44(%rbp), %ebx  // args[5]
	movl	-48(%rbp), %r11d // args[4]
	movl	-52(%rbp), %ecx  // args[3]
	movl	-56(%rbp), %edx  // args[2]
	movl	-60(%rbp), %esi  // args[1]
	movl	-64(%rbp), %eax  // args[0]
	pushq	%r10
	pushq	%r9
	pushq	%r8
	pushq	%rdi
	movl	%ebx, %r9d
	movl	%r11d, %r8d
	movl	%eax, %edi
	call	sum
	addq	$32, %rsp
	movl	%eax, -68(%rbp)
	movl	-68(%rbp), %eax
	movl	%eax, %esi
	leaq	.LC0(%rip), %rdi
	movl	$0, %eax
	call	printf@PLT
	movl	$0, %eax
	movq	-8(%rbp), %rbx
	leave
	ret
sum:
	pushq	%rbp
	movq	%rsp, %rbp
	movl	%edi, -4(%rbp)
	movl	%esi, -8(%rbp)
	movl	%edx, -12(%rbp)
	movl	%ecx, -16(%rbp)
	movl	%r8d, -20(%rbp)
	movl	%r9d, -24(%rbp)
	movl	-4(%rbp), %edx
	movl	-8(%rbp), %eax
	addl	%eax, %edx
	movl	-12(%rbp), %eax
	addl	%eax, %edx
	movl	-16(%rbp), %eax
	addl	%eax, %edx
	movl	-20(%rbp), %eax
	addl	%eax, %edx
	movl	-24(%rbp), %eax
	addl	%eax, %edx
	movl	16(%rbp), %eax
	addl	%eax, %edx
	movl	24(%rbp), %eax
	addl	%eax, %edx
	movl	32(%rbp), %eax
	addl	%eax, %edx
	movl	40(%rbp), %eax
	addl	%edx, %eax
	popq	%rbp
	ret
```

Here we can see that the order also is from **int j** to **int a**. But the first four args\(**from right to left**\) will use register **edi/rdi**,  **esi/rsi**, **edx/rdx**, **ecx/rcx**, **r8d/r8**, **r9d/r9**. The reminds will be put on stack in order.

The ret value also put on **eax/rax**.

*And the return still in the stack, same as x86.*

## arm

```bash
arm-none-eabi-gcc -S abi.c -o abi_arm.s
```

Strip unnecessary part, we will get below asm code.

```c
	.cpu arm7tdmi
	.align	2
.LC0:
	.ascii	"Sum is %d\012\000"
	.text
	.align	2
	.type	main, %function
main:
	push	{r4, r5, r6, r7, fp, lr}
	add	fp, sp, #20
	sub	sp, sp, #80
	str	r0, [fp, #-72]
	str	r1, [fp, #-76]
	sub	r3, fp, #68 // args
	mov	r2, #40 // sizeof(args) = 40
	mov	r1, #0
	mov	r0, r3
	bl	memset // memset(args, 0, 40)
	mov	r3, #0 // r3 = 0
	str	r3, [fp, #-24] // int i = 0
	b	.L2
.L3:
	ldr	r3, [fp, #-24] // r3 = i
	lsl	r3, r3, #2 // i = i * 4
	sub	r2, fp, #20
	add	r3, r2, r3 // r3 = fp - 20 + i * 4
	ldr	r2, [fp, #-24] // r2 = i
	str	r2, [r3, #-48] // [r3 - 48 = fp - 68 + i * 4] = r2 = i
                       // [args_base + i*4] = args[i] = i
	ldr	r3, [fp, #-24] // r3 = i
	add	r3, r3, #1 // i ++
	str	r3, [fp, #-24] // args[i] = i
.L2:
	ldr	r3, [fp, #-24] // r3 = i
	cmp	r3, #9 // int i < 10
	ble	.L3
	ldr	r4, [fp, #-68] // args[0]
	ldr	r5, [fp, #-64] // args[1]
	ldr	r6, [fp, #-60] // args[2]
	ldr	r7, [fp, #-56] // args[3]
	ldr	r3, [fp, #-52] // args[4]
	ldr	r2, [fp, #-48] // args[5]
	ldr	r1, [fp, #-44] // args[6]
	ldr	r0, [fp, #-40] // args[7]
	ldr	ip, [fp, #-36] // args[8]
	ldr	lr, [fp, #-32] // args[9]
	str	lr, [sp, #20]   // [sp + 12] = args[9]
	str	ip, [sp, #16]   // [sp + 12] = args[8]
	str	r0, [sp, #12]   // [sp + 12] = args[7]
	str	r1, [sp, #8]    // [sp + 8] = args[6]
	str	r2, [sp, #4]    // [sp + 4] = args[5]
	str	r3, [sp]        // [sp] = args[4]
	mov	r3, r7          // r3 = args[3]
	mov	r2, r6          // r2 = args[2]
	mov	r1, r5          // r1 = args[1]
	mov	r0, r4          // r0 = args[0]
	bl	sum
	str	r0, [fp, #-28] // r0 = ret_value
	ldr	r1, [fp, #-28] // r1 = ret_value
	ldr	r0, .L5 // "Sum is %d\n"
	bl	printf
	mov	r3, #0
	mov	r0, r3
	sub	sp, fp, #20
	@ sp needed
	pop	{r4, r5, r6, r7, fp, lr}
	bx	lr
.L5:
	.word	.LC0
	.size	main, .-main
	.align	2
	.global	sum
	.syntax unified
	.arm
	.fpu softvfp
	.type	sum, %function
sum:
	str	fp, [sp, #-4]! // Desc SP then store fp. (fp like ebp)
	add	fp, sp, #0
	sub	sp, sp, #20
	str	r0, [fp, #-8]
	str	r1, [fp, #-12]
	str	r2, [fp, #-16]
	str	r3, [fp, #-20]
	ldr	r2, [fp, #-8]
	ldr	r3, [fp, #-12]
	add	r2, r2, r3
	ldr	r3, [fp, #-16]
	add	r2, r2, r3
	ldr	r3, [fp, #-20]
	add	r2, r2, r3
	ldr	r3, [fp, #4]
	add	r2, r2, r3
	ldr	r3, [fp, #8]
	add	r2, r2, r3
	ldr	r3, [fp, #12]
	add	r2, r2, r3
	ldr	r3, [fp, #16]
	add	r2, r2, r3
	ldr	r3, [fp, #20]
	add	r2, r2, r3
	ldr	r3, [fp, #24]
	add	r3, r2, r3
	mov	r0, r3
	add	sp, fp, #0
	@ sp needed
	ldr	fp, [sp], #4
	bx	lr
```

So we can see that for arm, from the left to right, the first four args will be put in **register** **r0, r1, r2, r3**, then the remains will be put on stack in order.
Here cause **sum** is called via instruction **bl sum**, so in **sum** it will use **bx lr** to return to **main**.

In arm, below registers have special usage:

+ r15: pc
+ r14: lr
+ r13: sp
+ r12: ip\(intra procedure call\)
+ r11: fp
+ r10: sl\(stack limit\)
+ r9:  sb\(stack base\)

## aarch64

```bash
aarch64-linux-gnu-gcc -S abi.c -o abi_aarch64.s
```

Use above command to get assembly for aarch64, then remove the unnecessary part, we get below code:

```c
.LC0:
	.string	"Sum is %d\n"
	.text
	.align	2
	.global	main
	.type	main, %function
main:
.LFB0:
	sub	sp, sp, #96
	stp	x29, x30, [sp, 16]
	add	x29, sp, 16
	str	w0, [sp, 44]
	str	x1, [sp, 32]
	stp	xzr, xzr, [sp, 48] // args
	stp	xzr, xzr, [sp, 64]
	str	xzr, [sp, 80]
	str	wzr, [sp, 92] // int i = 0
	b	.L2
.L3:
	ldrsw	x0, [sp, 92] // x0 = i
	lsl	x0, x0, 2 // x0 = i * 4
	add	x1, sp, 48 // x1 = sp + 48
	ldr	w2, [sp, 92] // w2 = i
	str	w2, [x1, x0] // [x1 + x0] = [sp * 48 + i * 4] = args[i]
                     // args[i] = w2 = i
	ldr	w0, [sp, 92] // w0 = i
	add	w0, w0, 1 // i ++
	str	w0, [sp, 92]
.L2:
	ldr	w0, [sp, 92]
	cmp	w0, 9 // i < 10
	ble	.L3
	ldr	w8, [sp, 48] // args[0]
	ldr	w9, [sp, 52] // args[1]
	ldr	w2, [sp, 56] // args[2]
	ldr	w3, [sp, 60] // args[3]
	ldr	w4, [sp, 64] // args[4]
	ldr	w5, [sp, 68] // args[5]
	ldr	w6, [sp, 72] // args[6]
	ldr	w7, [sp, 76] // args[7]
	ldr	w0, [sp, 80] // args[8]
	ldr	w1, [sp, 84] // args[9]
	str	w1, [sp, 8]  // [sp, 8] = args[9]
	str	w0, [sp]     // [sp] = args[8]
	mov	w1, w9       // args[0]
	mov	w0, w8       // args[1]
	bl	sum
	str	w0, [sp, 88]
	ldr	w1, [sp, 88]
	adrp	x0, .LC0
	add	x0, x0, :lo12:.LC0
	bl	printf
	mov	w0, 0
	ldp	x29, x30, [sp, 16]
	add	sp, sp, 96
	ret
.LFE0:
	.size	main, .-main
	.align	2
	.global	sum
	.type	sum, %function
sum:
.LFB1:
	sub	sp, sp, #32 // sub for local variable
	str	w0, [sp, 28]
	str	w1, [sp, 24]
	str	w2, [sp, 20]
	str	w3, [sp, 16]
	str	w4, [sp, 12]
	str	w5, [sp, 8]
	str	w6, [sp, 4]
	str	w7, [sp]
	ldr	w1, [sp, 28]
	ldr	w0, [sp, 24]
	add	w1, w1, w0
	ldr	w0, [sp, 20]
	add	w1, w1, w0
	ldr	w0, [sp, 16]
	add	w1, w1, w0
	ldr	w0, [sp, 12]
	add	w1, w1, w0
	ldr	w0, [sp, 8]
	add	w1, w1, w0
	ldr	w0, [sp, 4]
	add	w1, w1, w0
	ldr	w0, [sp]
	add	w1, w1, w0
	ldr	w0, [sp, 32]
	add	w1, w1, w0
	ldr	w0, [sp, 40]
	add	w0, w1, w0
	add	sp, sp, 32 // restore stack balance
	ret
```

As we can see, for aarch64, more registers can be used to store args than arm, for args from left to right, first eight args will be stored in **register** **w0, w1, w2, w3, w4, w5, w6, w7**, the remains will be put on stack in order.

And the return is different that arm casue here it use **ret**, which in format **ret {xm}**, when xm is omitted, it will use **x30**\(aka. lr\)

In aarch64, below registers have special usage:

+ x30: lr

The 31th register will have different usage:

+ sp/wsp:  stack pointer/32-bit stack pointer
+ xzr/wzr: zero register

Also, **In AArch64 state, the PC is not a general purpose register and you cannot access it by name.**

## Related Reading

+ https://developer.arm.com/docs/den0024/latest/an-introduction-to-the-armv8-instruction-sets/the-armv8-instruction-sets/registers
+ http://infocenter.arm.com/help/index.jsp?topic=/com.arm.doc.dui0801a/BABBGCAC.html
+ http://infocenter.arm.com/help/index.jsp?topic=/com.arm.doc.dui0801a/BABBDBAD.html
+ http://infocenter.arm.com/help/index.jsp?topic=/com.arm.doc.dui0068b/ch03s03s01.html
+ http://infocenter.arm.com/help/index.jsp?topic=/com.arm.doc.kui0060a/is51_ret.htm