---
layout:     post
title:      "通过调用C语言的库函数与在C代码中使用内联汇编两种方式来使用同一个系统调用来分析系统调用的工作机制"
subtitle:   "网易云课堂`Linux内核分析`课程的第四周作业"
date:       2015-03-25 13:00::00
author:     "MarkWoo"
header-img: "img/home-bg.jpg"
---

#通过调用C语言的库函数与在C代码中使用内联汇编两种方式来使用同一个系统调用来分析系统调用的工作机制

##前言说明
本篇为网易云课堂Linux内核分析课程的第四周作业，我将通过调用C语言的库函数与在C代码中使用内联汇编两种方式来使用同一个系统调用来分析系统调用的工作机制,本篇中，我将分别使用两个典型的系统调用（`getpid`,`open`）来进行实例分析，意图通过这三个不同的系统调用来阐述Linux中的系统调用的工作方式。

**运行环境：**

- Ubuntu 14.04 LTS x64
- gcc 4.9.2
- gdb 7.8
- vim 7.4 with vundle
##分析过程
系统调用（System Call）是操作系统为在用户态运行的进程与硬件设备（如CPU、磁盘、打印机等）进行交互提供的一组接口。当用户进程需要发生系统调用时，CPU 通过软中断切换到内核态开始执行内核系统调用函数，一般而言,在Linux中会有三种系统调用:

- 通过标准C库（libc）的API
- 通过Syscall直接调用
- int 0x80中断向量指令陷入

###getpid
`getpid`的函数原型为：

```c
pid getpid(void)
```

其功能为返回一个进程的进程ID，该函数没有参数

```c
#include <stdio.h>
#include <unistd.h>

int main(void) {
	pid_t process_id;

#if 1
	process_id = getpid();

#else
	asm volatile (
			"movl $0, %%ebx\n\t"
			"movl $0x14, %%eax\n\t" //getpid的系统调用号是0x14
			"int $0x80\n\t" //中断向量陷入指令
			"movl %%eax, %0\n\t"
			:"=r"(process_id)
			);
#endif

	printf("process is %u\n", process_id);
	return 0;
}
```

- 首先，从C代码分析，从内联汇编可以看出，当进行系统调用时，首先应该把系统调用号放入`eax`寄存器中，然后通过`int 0x80`中断向量指令来使用户态进程陷入内核态，参数的传递是通过寄存器，`eax`传递的是系统调用号，`ebx`,`ecx`,`edx`,`exi`,`edi`来传递其他参数，同时`eax`也负责保存系统调用后的返回值

```
//调用C库的API
main:
	leal	4(%esp), %ecx
	andl	$-16, %esp
	pushl	-4(%ecx)
	pushl	%ebp
	movl	%esp, %ebp
	pushl	%ecx
	subl	$20, %esp
	call	getpid
	movl	%eax, -12(%ebp)
	subl	$8, %esp
	pushl	-12(%ebp)
	pushl	$.LC0
	call	printf
	addl	$16, %esp
	movl	$0, %eax
	movl	-4(%ebp), %ecx
	leave
	leal	-4(%ecx), %esp
	ret

//内联汇编方式
main:
	leal	4(%esp), %ecx
	andl	$-16, %esp
	pushl	-4(%ecx)
	pushl	%ebp
	movl	%esp, %ebp
	pushl	%ecx
	subl	$20, %esp
#APP
# 14 "getpid.c" 1
	movl $0, %ebx
	movl $0x14, %eax
	int $0x80
	movl %eax, %eax
# 0 "" 2
#NO_APP
	movl	%eax, -12(%ebp)
	subl	$8, %esp
	pushl	-12(%ebp)
	pushl	$.LC0
	call	printf
	addl	$16, %esp
	movl	$0, %eax
	movl	-4(%ebp), %ecx
	leave
	leal	-4(%ecx), %esp
	ret
```

- 从以上代码可以看出除了内联汇编与`call getpid`外，其他部分完全相同，这里需要指出的一点是：

```c
	movl $0, %ebx
	movl $0x14, %eax
	int $0x80
	movl %eax, %eax
```

这里由于我是选择了在输出时，使用`'=r'`约束条件，即输出的值可以保存在任何可用的全局的寄存器中，所以这里会有一步`movl %eax, %eax`

---
###open
`open`的函数原型为：

```c
int open(const char *pathname, int flags, mode_t mode)
```

其功能是打开一个特定路径下的文件，返回一个文件描述符

```c
int Open(const char *pathname, int flags, mode_t mode) {
	int fd_t;

	asm volatile (
			"movl S0, %%ebi\n\t"
			"movl $3, %%edx\n\t"
			"movl $2, %%ecx\n\t"
			"movl $1, %%ebx\n\t"
			"movl $0x5, %%eax\n\t"
			"int $0x80\n\t"
			"movl %%eax, $0\n\t"
			:"=r"(fd_t)
			:"b"(pathname), "c"(flags), "d"(mode)
			);

	return fd_t;
}
``` 

- 这里我封装了一个Open系统调用，与第一个实例中相同的部分这里就不再赘述了，仅仅说一下不同的地方，`:"b"(pathname), "c"(flags), "d"(mode)`，这里是输入参数，表示`pathname`，`flags`，`mode`，这三个参数分别传递到`ebx`，`ecx`，`edx`三个寄存器中

```c
Open:
	pushl	%ebp
	movl	%esp, %ebp
	pushl	%ebx
	subl	$16, %esp
	movl	8(%ebp), %eax
	movl	12(%ebp), %ecx
	movl	16(%ebp), %edx
	movl	%eax, %ebx
#APP
# 15 "open.c" 1
	movl S0, %ebi
	movl $1, %ebx
	movl $2, %ecx
	movl $3, %edx
	movl $0x5, %eax
	int $0x80
	movl %eax, $0
# 0 "" 2
#NO_APP
	movl	%eax, -8(%ebp)
	movl	-8(%ebp), %eax
	addl	$16, %esp
	popl	%ebx
	popl	%ebp
	ret
```

---
##我的总结
通过这一周的学习，我更加熟悉了系统调用的本质，也更加熟悉了内联汇编，系统调用是用户进程与操作系统进行交互的核心，通过`int 0x80`中断向量指令实现从用户态进入内核态，系统调用过程中，`eax`寄存器负责传递**系统调用号**,`ebx`,`ecx`等其他寄存器负责传递**其他参数**。

---
##参考资料
- Understanding The Linux Kernel, the 3rd edtion

---
##署名信息
吴欣伟 原创作品转载请注明出处 《Linux内核分析》MOOC课程：http://mooc.study.163.com/course/USTC-1000029000
