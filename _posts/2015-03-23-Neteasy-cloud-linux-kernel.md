---
layout:     post
title:      "通过反汇编一个简单的C程序，分析汇编代码理解计算机是如何工作的"
subtitle:   "网易云课堂Linux内核分析课程"
date:       2015-03-20 13:00::00
author:     "MarkWoo"
header-img: "img/home-bg.jpg"
---

# 通过反汇编一个简单的C程序，分析汇编代码理解计算机是如何工作的
```c
int g(int x) {
	return x + 109;
}

int f(int x) {
	return g(x);
}

int main() {
	return f(122) + 3;
}
```

# 汇编代码的工作过程分析
```
g:
	pushl	%ebp
	movl	%esp, %ebp
	movl	8(%ebp), %eax
	addl	$109, %eax
	popl	%ebp
	ret
f:
	pushl	%ebp
	movl	%esp, %ebp
	pushl	8(%ebp)
	call	g
	addl	$4, %esp
	leave
	ret
main:
	pushl	%ebp
	movl	%esp, %ebp
	pushl	$122
	call	f
	addl	$4, %esp
	addl	$3, %eax
	leave
	ret
```

**为了写作方便，ebp,esp等扩展的寄存器在以下均写作为ep,sp等**
首先，main函数为该程序的开始入口，所以从main函数开始分析: 

>- 在line 17 ~ line 18是进入main函数(enter操作)，其过程是：
首先是 `pushl  %ebp` 操作：sp-4，然后将当前bp的值放入sp所指向的内存区块，然后是`movl  %esp, %ebp`：将esp的值赋值给ebp，这样bp和sp将指向同一个位置，就是重新指向了sp所指向的栈顶位置.
- line 19操作将立即数122入栈，做好准备，以便于进行加法操作时使用.

在line 20开始调用f函数，这里开始对f函数进行分析：

>- call f完毕后，此时堆栈情况：sp(指向ip，ip指向cs中的f函数执行段),bp(指向sp前一个位置)
- line 9 ~ line 10为enter操作，进入函数其操作过程同`main函数`的操作过程，经过完此时后状态将是，bp与sp指向同一个栈顶位置，此时sp中所指向的内容是bp在执行进入f函数的enter操作之前的bp的值(**注意，这里的bp值和main函数中的bp值不一样**).
- 执行到line 11时，将bp加上8的值（即122的值）放入sp所指向的被分配的内存区块，为函数g的调用做准备.

在line 12是开始调用g函数，这里开始对g函数进行分析：

>- line 2 ~ line 3 执行enter操作，同f函数.
- line 4 将bp+8的所指向的值放入ax中，即122，为下面的加法操作做准备.
- line 5 将立即数109在ax中的值做加法操作，然后结果放入ax中.
- line 6 弹出栈顶的ip，sp+4
- line 7 返回g函数，执行完后，弹出栈顶的内容放入ip中，此时堆栈回到了调用函数g之前的状态,得到g(122)

回到f函数中：

>- line 13 ~ 15 执行后，堆栈恢复到函数f调用之，得到f(122)

最后，回到main函数:

>- line 21 ~ line 22 执行，sp+4， sp指向bp值（此bp的值为指向栈底的值），`add  $3， $eax` ，ax中的存储的值+3
- line 23 ~ line 24，main函数执行完毕，堆栈回到初始状态(sp,bp均指向栈底)，返回计算值。

# 总结

通过分析这段C语言代码的汇编代码，可以得到计算机程序执行的几个特点：

> - 总是通过EIP取得下一段要执行的代码，然后执行该段代码，即总是**取指执行**
- 当进行函数调用时，堆栈会保存调用函数之前的程序状态，同时堆栈指针bp和sp会在一个`伪初始位置`
- 每次函数调用结束，堆栈指针bp和sp回复到调用之前的状态
