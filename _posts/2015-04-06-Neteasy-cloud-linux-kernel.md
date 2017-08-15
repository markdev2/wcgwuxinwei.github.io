---
layout:     post
title:      "通过分析system_call中断处理过程来深入理解系统调用"
subtitle:   "网易云课堂Linux内核分析课程"
date:       2015-04-06 13:00::00
author:     "MarkWoo"
header-img: "img/home-bg.jpg"
---

# 通过分析system_call中断处理过程来深入理解系统调用

## 前言说明
本篇为网易云课堂Linux内核分析课程的第五周作业，我将通过上一次作业中分析的2个系统调用（`getpid`, `open`）作为分析实例来更加深入分析系统调用的过程，本篇中我将深入到system_call（汇编级别代码）中来分析其执行过程.

---
## 关键词：`system_call`, `系统调用`

---
*运行环境：*

- Ubuntu 14.04 LTS x64
- gcc 4.9.2
- gdb 7.8
- vim 7.4 with vundle

---
## 分析过程
从上一次课后，我们对于系统调用在执行过程中的一些基本情况有了一个比较抽象的认识，一个系统调用的基本过程是用户态程序通过`int 0x80`中断向量指令实现从用户态进入内核态，系统调用过程中，`eax`寄存器负责传递**系统调用号**,`ebx`,`ecx`等其他寄存器负责传递**其他参数**，但是对于执行完了`int 0x80`之后，在内核态时：

- 操作系统到底干了哪些具体的工作？
- 系统调用在内核态这一阶段的过程是什么？

### 基本概念的总结
1. 中断
    中断分为2种，
    - 可屏蔽中断： I/O设备发出的所有的中断请求（IRQ）都产生可屏蔽中断。可屏蔽中断产生两种状态：屏蔽的（masked）或非屏蔽的（unmasked）；当中断被屏蔽，则CPU控制单元就忽略它。
    - 非可屏蔽中断：总是由CPU辨认。只有几个危急事件引起非屏蔽中断。

2. 进程上下文
    一般来说，CPU在任何时刻都处于以下三种情况之一：
    >* 运行于用户空间，执行用户进程；
    >* 运行于内核空间，处于进程上下文；
    >* 运行于内核空间，处于中断上下文。
    
    应用程序通过系统调用陷入内核，此时处于进程上下文。现代几乎所有的CPU体系结构都支持中断。当外部设备产生中断，向CPU发送一个异步信号，CPU调用相应的中断处理程序来处理该中断，此时CPU处于中断上下文。
    在进程上下文中，可以通过current关联相应的任务。进程以进程上下文的形式运行在内核空间，可以发生睡眠，所以在进程上下文中，可以使作信号量(semaphore)。实际上，内核经常在进程上下文中使用信号量来完成任务之间的同步，当然也可以使用锁。
    中断上下文不属于任何进程，它与current没有任何关系(尽管此时current指向被中断的进程)。由于没有进程背景，在中断上下文中不能发生睡眠，否则又如何对它进行调度。所以在中断上下文中只能使用锁进行同步，正是因为这个原因，中断上下文也叫做原子上下文(atomic context)(关于同步以后再详细讨论)。在中断处理程序中，通常会禁止同一中断，甚至会禁止整个本地中断，所以中断处理程序应该尽可能迅速，所以又把中断处理分成上部和下部。
    
    　　相对于进程而言，就是进程执行时的环境。具体来说就是各个变量和数据，包括所有的寄存器变量、进程打开的文件、内存信息等。一个进程的上下文可以分为三个部分:用户级上下文、寄存器上下文以及系统级上下文。
    
    >*  用户级上下文: 正文、数据、用户堆栈以及共享存储区；
    >*  寄存器上下文: 通用寄存器、程序寄存器(IP)、处理器状态寄存器(EFLAGS)、栈指针(ESP)；
    >* 系统级上下文: 进程控制块task_struct、内存管理信息(mm_struct、vm_area_struct、pgd、pte)、内核栈。

3. 中断上下文与进程上下文
硬件通过触发信号，导致内核调用中断处理程序，进入内核空间。这个过程中，硬件的 一些变量和参数也要传递给内核，内核通过这些参数进行中断处理。所谓的“ 中断上下文”，其实也可以看作就是硬件传递过来的这些参数和内核需要保存的一些其他环境（主要是当前被打断执行的进程环境）。**中断时，内核不代表任何进程运行，它一般只访问系统空间，而不会访问进程空间，内核在中断上下文中执行时一般不会阻塞**.

---
## System_Call

```c
# system call handler stub
ENTRY(system_call)
	RING0_INT_FRAME			# can't unwind into user space anyway
	ASM_CLAC
	pushl_cfi %eax			# save orig_eax
	SAVE_ALL				// 保存系统寄存器信息
	GET_THREAD_INFO(%ebp)   // 获取thread_info结构的信息
					# system call tracing in operation / emulation
	testl $_TIF_WORK_SYSCALL_ENTRY,TI_flags(%ebp) # 测试是否有系统跟踪
	jnz syscall_trace_entry   // 如果有系统跟踪，先执行，然后再回来
	cmpl $(NR_syscalls), %eax // 比较eax中的系统调用号和最大syscall，超过则无效
	jae syscall_badsys  // 无效的系统调用 直接返回
syscall_call:
	call *sys_call_table(,%eax,4) // 调用实际的系统调用程序
syscall_after_call:
	movl %eax,PT_EAX(%esp)		// 将系统调用的返回值eax存储在栈中
syscall_exit:
	LOCKDEP_SYS_EXIT
	DISABLE_INTERRUPTS(CLBR_ANY)	# make sure we don't miss an interrupt
					# setting need_resched or sigpending
					# between sampling and the iret
	TRACE_IRQS_OFF
	movl TI_flags(%ebp), %ecx
	testl $_TIF_ALLWORK_MASK, %ecx	//检测是否所有工作已完成
	jne syscall_exit_work  			//工作已经完成，则去进行系统调用推出工作

restore_all:
	TRACE_IRQS_IRET			// iret 从系统调用返回
```

System_Call的基本处理流程为：
- 首先保存中断上下文（SAVE_ALL，也就是CPU状态，包括各个寄存器）,判断请求的系统调用是否有效
- 然后`call *sys_call_table(,%eax,4)`通过系统查询系统调用查到相应的系统调用程序地址，执行相应的系统调用
- 系统调用完后，返回系统调用的返回值
- 关闭中断响应，检测系统调用的所有工作是否已经完成，如果完成则进行`syscall_exit_work`(完成系统调用退出工作）
- 最后restore_all(恢复中断请求响应),返回用户态

接下来对于中间比较关键的片段代码进行重点分析

## System_Call中的关键部分

### syscall_exit_work

```c
syscall_exit_work:
	testl $_TIF_WORK_SYSCALL_EXIT, %ecx //测试syscall的工作完成
	jz work_pending
	TRACE_IRQS_ON  //切换中断请求响应追踪可用
	ENABLE_INTERRUPTS(CLBR_ANY)	# could let syscall_trace_leave() call
					//schedule() instead
	movl %esp, %eax
	call syscall_trace_leave //停止追踪系统调用
	jmp resume_userspace //返回用户空间,只需要检查need_resched
END(syscall_exit_work)
```

该过程为系统调用完成后如何退出调用的过程,其中比较重要的是`work_pending`,详见如下：

```c
work_pending:
	testb $_TIF_NEED_RESCHED, %cl  // 判断是否需要调度
	jz work_notifysig   // 不需要则跳转到work_notifysig
work_resched:
	call schedule   // 调度进程
	LOCKDEP_SYS_EXIT
	DISABLE_INTERRUPTS(CLBR_ANY)	# make sure we don't miss an interrupt
					# setting need_resched or sigpending
					# between sampling and the iret
	TRACE_IRQS_OFF
	movl TI_flags(%ebp), %ecx
	andl $_TIF_WORK_MASK, %ecx	// 是否所有工作都已经做完
	jz restore_all  			// 是则退出
	testb $_TIF_NEED_RESCHED, %cl // 测试是否需要调度
	jnz work_resched  			// 重新执行调度代码

work_notifysig:				// 处理未决信号集
#ifdef CONFIG_VM86
	testl $X86_EFLAGS_VM, PT_EFLAGS(%esp) // 判断是否在虚拟8086模式下
	movl %esp, %eax
	jne work_notifysig_v86		// 返回到内核空间
1:
#else
	movl %esp, %eax
#endif
	TRACE_IRQS_ON  #启动跟踪中断请求响应
	ENABLE_INTERRUPTS(CLBR_NONE)
	movb PT_CS(%esp), %bl
	andb $SEGMENT_RPL_MASK, %bl
	cmpb $USER_RPL, %bl
	jb resume_kernel        # 恢复内核空间
	xorl %edx, %edx
	call do_notify_resume  # 将信号投递到进程
	jmp resume_userspace  # 恢复用户空间

#ifdef CONFIG_VM86
	ALIGN
work_notifysig_v86:
	pushl_cfi %ecx			# save ti_flags for do_notify_resume
	call save_v86_state		# 保存VM86模式下的CPU信息
	popl_cfi %ecx
	movl %eax, %esp
	jmp 1b
#endif
END(work_pending)
```
首先是`work_pending`这段汇编逻辑：
- 检查是否需要进行调度
- 如果需要，进行进程调度，然后再次进行判断
- 如果无需调度，那就去执行`work_notifying`,处理信号

然后是`work_notifysig`的这段汇编逻辑:
- 先检查是否是虚拟8086模式,即8086保护模式
- 如果是，那么需要先保存虚模式下的状态信息
- 然后跳转到之前的代码继续执行
- 将信号投递到进程
- 恢复用户空间

最后返回系统调用

---
## 我的总结
系统调用中断本质上是一个保存当前工作状态，然后处理，最后返回并且恢复线程的过程.

---
## 参考资料
- Understanding The Linux Kernel, the 3rd edtion
- Linux内核设计与实现，第三版，Robert Love, 机械工业出版社
