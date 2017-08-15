---
layout:     post
title:      "通过分析一个简化版时间片轮转多道程序内核代码来认识操作系统中的进程调度"
subtitle:   "网易云课堂Linux内核分析课程"
date:       2015-03-24 13:00::00
author:     "MarkWoo"
header-img: "img/home-bg.jpg"
---

# 通过分析一个简化版时间片轮转多道程序内核代码来认识操作系统中的进程调度
## 前言说明
本篇为网易云课堂`Linux内核分析`课程的第二周作业，我讲将围绕一个时间片轮转，进程切换的执行过程来分析该代码来认识理解**操作系统的进程工作情况**，文中的代码来自USTC孟宁老师的`Github`，地址为[mykernel][1]

## 本篇关键词：`内联汇编`，`进程调度`，`时间片`，`进程切换`

## 分析
## 分析说明
分析过程将把主要精力放在关键代码的分析上，代码分析的方式我是采用注释说明的方法，这样比较简洁直观，对于一些关键过程我会在代码后面采用图文说明的方式。

---

```c
#define MAX_TASK_NUM        4  //最大进程数
#define KERNEL_STACK_SIZE   1024*8 //内核堆栈空间大小

/* CPU-specific state of this task */
struct Thread {
    unsigned long		ip; //用于保存进程eip内容
    unsigned long		sp; //用于保存进程esp内容
};

typedef struct PCB{
    int pid; //进程号
    //进程运行状态
    volatile long state;	/* -1 unrunnable, 0 runnable, >0 stopped */ 
    char stack[KERNEL_STACK_SIZE];
    /* CPU-specific state of this task */
    struct Thread thread;
    //进程执行的入口地址
    unsigned long	task_entry;
    //指向下一个进程PCB
    struct PCB *next;
}tPCB;

//进程调度
void my_schedule(void);
```

- `linux/mykernel/mypcb.h`
- 以上这段代码定义了进程执行所需要的一些必要信息，比如`pid`(进程号)，`stack`(进程堆栈),`task_entry`(进程执行的入口地址)等，通过这个文件我们可以知道这些信息都保存在一个叫做`PCB`(进程控制块, process control block)的数据结构中，这个头文件还定义了线程的一些相关信息，如：ip，sp等,结合之前上的知识，和改头文件，我们可以认识到进程真正的执行单元是**线程**,线程中包含了执行是所需要的一些关键信息，堆栈顶指针sp,和指令指针ip


---

```c
#include "mypcb.h"
tPCB task[MAX_TASK_NUM];
tPCB * my_current_task = NULL; //当前进程
volatile int my_need_sched = 0; //是否需要调度

void my_process(void);

/* 启动内核 */
void __init my_start_kernel(void)
{
    int pid = 0;
    int i;
    /* Initialize process 0*/
    task[pid].pid = pid;
    task[pid].state = 0;/* -1 unrunnable, 0 runnable, >0 stopped */
    // 设置进程执行入口到my_process
    task[pid].task_entry = task[pid].thread.ip = (unsigned long)my_process;
    // 设置堆栈栈顶指针
    task[pid].thread.sp = (unsigned long)&task[pid].stack[KERNEL_STACK_SIZE-1];
    task[pid].next = &task[pid];
    /*fork more process */
    for(i=1;i<MAX_TASK_NUM;i++)
    {
        // 复制init process的地址空间给新的进程
        memcpy(&task[i],&task[0],sizeof(tPCB));
        task[i].pid = i;
        task[i].state = -1;
        task[i].thread.sp = (unsigned long)&task[i].stack[KERNEL_STACK_SIZE-1];
        task[i].next = task[i-1].next;
        task[i-1].next = &task[i];
    }
    /* start process 0 by task[0] */
    pid = 0;
    my_current_task = &task[pid];
    /*
	 * 下面的内联汇编的过程是task[pid]中的线程sp传递给系统堆栈中的esp,
	 * 线程的ip传递给给系统的eip
	 * pid为0的进程的ip和sp先输入cx和dx中暂存起来，以便调用
	 * 我的理解：
	 * 为了让进程在硬件内存空间中运行起来，必须将准备的的进程PCB信息注册给计算机中相应的内存
	 */
	asm volatile(
    	"movl %1,%%esp\n\t" 	/* set task[pid].thread.sp to esp */
    	"pushl %1\n\t" 	        /* push ebp */
    	"pushl %0\n\t" 	        /* push task[pid].thread.ip */
    	"ret\n\t" 	            /* pop task[pid].thread.ip to eip */
    	"popl %%ebp\n\t"
    	: 
    	: "c" (task[pid].thread.ip),"d" (task[pid].thread.sp)	/* input c or d mean %ecx/%edx*/
	);
}

//进程执行
void my_process(void)
{
    int i = 0;
    while(1)
    {
        i++;
        if(i%10000000 == 0)
        {
            printk(KERN_NOTICE "this is process %d -\n",my_current_task->pid);
            if(my_need_sched == 1)
            {
                my_need_sched = 0;
        	    my_schedule();
        	}
        	printk(KERN_NOTICE "this is process %d +\n",my_current_task->pid);
        }     
    }
}
```

- `Linux/mykernel/mymain.c`
以上这段代码包含了内核的启动过程，进程的创建和启动运行（这里sp指的是PCB中的sp，esp指硬件esp），这里我将会进行代码块的逐个分析：
    - `void __init my_start_kernel(void)`从这个函数的命名上可以看出，这个函数负责了内核初始化和内核的启动过程，我们都知道Linux内核最开始启动执行的进程叫做`init`进程,该进程的`PID`为`0`,从这个函数的代码中我们可以看出其他进程的创建都是通过`fork init`进程，而且各个进程启动顺序是一种线性的关系
    - `void my_process(void)` 该函数是进程的执行部分，可以从代码中看出，每个进程在每执行1000万次时进行一次进程调度（实质是进程切换，调用my_schedule()），剥夺当前进程获取CPU时间片的权力
    - 这里有一段开始启动`init`进程的关键内联汇编代码, 首先，将`init`进程的sp移入esp中，将sp放入堆栈栈底（即初始esp指向的位置），这里这样做的目的是为了最后那一步`popl %%ebp\n\t`时可以将sp也移入ebp中，然后将进程的ip放入堆栈中（存有sp的下一个位置），通过`ret`将ip放入eip中。这段内联汇编代码执行完后，`init`进程完成了初始化，可以内核开始启动运行

```c
asm volatile(
    	"movl %1,%%esp\n\t" 	/* 将输入的task[pid].thread.sp放入esp中 */ 
    	"pushl %1\n\t" 	        /* 将输入的task[pid].thread.sp放入栈顶位置中存储起来 */
    	"pushl %0\n\t" 	        /* 将输入的task[pid].thread.ip放入栈顶位置中存储起来 */
    	"ret\n\t" 	            /* 弹出栈顶的task[pid].thread.ip放入eip中 */
    	"popl %%ebp\n\t"        /* 初始化ebp */
    	: 
    	: "c" (task[pid].thread.ip),"d" (task[pid].thread.sp)	/* input c or d mean %ecx/%edx*/
	);
```

---

```c
#include "mypcb.h"

extern tPCB task[MAX_TASK_NUM];
extern tPCB * my_current_task;
extern volatile int my_need_sched;
volatile int time_count = 0;

/*
 * Called by timer interrupt.
 * it runs in the name of current running process,
 * so it use kernel stack of current running process
 */
//时钟中断
void my_timer_handler(void)
{
#if 1
    if(time_count%1000 == 0 && my_need_sched != 1) /* 这里判断是否调用了1000次时钟中断且还没有进行过进程调度 */
    {
        printk(KERN_NOTICE ">>>my_timer_handler here<<<\n");
        my_need_sched = 1;
    } 
    time_count ++ ;  
#endif
    return;  	
}

//进程调度
void my_schedule(void)
{
    tPCB * next;
    tPCB * prev;
    if(my_current_task == NULL 
        || my_current_task->next == NULL)
    {
    	return;
    }
    printk(KERN_NOTICE ">>>my_schedule<<<\n");
    /* schedule */
    next = my_current_task->next; //next指向当前进程的下一个进程
    prev = my_current_task; //prev指向当前进程
    if(next->state == 0)/* -1 unrunnable, 0 runnable, >0 stopped */
    {
        /* 以下内联汇编的代码含义：
		 * 适用正在运行的进程
		 * 首先保存当前进程的执行上下文,也就是堆栈信息(主要是ebp,esp,eip）放入当前进程的tPCB中，
		 * 恢复将要运行的进程的执行上下文，即堆栈信息（ebp,esp,eip）
		 */
    	/* switch to next process */
    	asm volatile(	
        	"pushl %%ebp\n\t" 	    /* save ebp */
        	"movl %%esp,%0\n\t" 	/* save esp */
        	"movl %2,%%esp\n\t"     /* restore  esp */
        	"movl $1f,%1\n\t"       /* save eip */	
        	"pushl %3\n\t" 
        	"ret\n\t" 	            /* restore  eip */
        	"1:\t"                  /* next process start here */
        	"popl %%ebp\n\t"
        	: "=m" (prev->thread.sp),"=m" (prev->thread.ip)
        	: "m" (next->thread.sp),"m" (next->thread.ip)
    	); 
    	my_current_task = next; //设置新进程给当前运行进程指针
    	printk(KERN_NOTICE ">>>switch %d to %d<<<\n",prev->pid,next->pid);   	
    }
    else
    {
        next->state = 0;
        my_current_task = next;
        printk(KERN_NOTICE ">>>switch %d to %d<<<\n",prev->pid,next->pid);
        
        /* 以下内联汇编的代码含义：
		 * 适用未运行的进程
		 * 首先保存当前进程的执行上下文,也就是堆栈信息(主要是ebp,esp,eip）放入当前进程的tPCB中，
		 * 之前的不同点是：由于未进程堆栈内是空的，所以esp和ebp都是指向同一个位置（即初始化位置）
		 */
    	/* switch to new process */
    	asm volatile(	
        	"pushl %%ebp\n\t" 	    /* save ebp */
        	"movl %%esp,%0\n\t" 	/* save esp */
        	"movl %2,%%esp\n\t"     /* restore  esp */
        	"movl %2,%%ebp\n\t"     /* restore  ebp */
        	"movl $1f,%1\n\t"       /* save eip */	
        	"pushl %3\n\t" 
        	"ret\n\t" 	            /* restore  eip */
        	: "=m" (prev->thread.sp),"=m" (prev->thread.ip)
        	: "m" (next->thread.sp),"m" (next->thread.ip)
    	);          
    }   
    return;	
}
```
- `linux/mykernel/myinterrupt`
- 以上这段代码包含了时钟中断,进程切换
    - `void my_timer_handler(void)` 该函数对已经进行了每1000次时钟中断，且没有进行过进程调度（运行权）的进程给予获取CPU时间片的权力
    - `void my_schedule(void)` 该函数是进行进程调度的代码，这个函数主要包含一个大的if判断，这里是为了区分是正在运行的进程还是没有运行的进程，2个条件分支的相同点在于：**都需要保存当前正在运行的进程上下文**,不同点在于中间的2段内联汇编代码,其中第一段代码表示已经执行过的进程切换，其过程是首先保存当前进程运行时上下文（主要是堆栈信息，ebp，esp，eip等），然后恢复将要执行进程的运行时上下文。第二段代码的过程大体相同，不同点在于，由于第二段代码中的要切换的进程从来没有运行过，所以其堆栈内是空的，所以esp和ebp都是指向同一个位置（即堆栈初始化状态）

```c
/* switch to next process */
//切换到正在运行的进程
asm volatile(   
    "pushl %%ebp\n\t"       /* 将当前ebp压入栈 */
    "movl %%esp,%0\n\t"     /* 保存当前进程的esp到prev->thread.sp中  */
    "movl %2,%%esp\n\t"     /* 将输入的next->thread.sp放入esp中 */
    "movl $1f,%1\n\t"       /* 保存当前进程的eip到prev->thread.ip中 */ 
    "pushl %3\n\t"          /* 压入输入next->thread.ip入栈 */
    "ret\n\t"               /* next->thread.ip放入eip中，也就是恢复要切换的进程的eip */
    "1:\t"                  /* 开始执行下一个进程 */
    "popl %%ebp\n\t"        /* 初始化ebp */
    : "=m" (prev->thread.sp),"=m" (prev->thread.ip)
    : "m" (next->thread.sp),"m" (next->thread.ip)
); 

/* switch to new process */
//切换到没有运行的新进程
asm volatile(   
    "pushl %%ebp\n\t"       
    "movl %%esp,%0\n\t"     
    "movl %2,%%esp\n\t"     /* 将输入的next->thread.sp放入esp中 */
    "movl %2,%%ebp\n\t"     /* 将输入的next->thread.sp放入ebp中，这样堆栈就是一个初始化状态，即esp与ebp指向同一位置 */
    "movl $1f,%1\n\t" 
    "pushl %3\n\t" 
    "ret\n\t"               /* restore  eip */
    : "=m" (prev->thread.sp),"=m" (prev->thread.ip)
    : "m" (next->thread.sp),"m" (next->thread.ip)
);
```

## 我的对操作系统以及进程一点认识
- 操作系统的运行实体应该线程，非init进程在Linux内核中初始化，其资源分配的初始化都是通过复制init（pid为0）的资源空间开始的。
- 进程在操作系统中并不是一直占着CPU不放，而是通过时钟中断以及时间片轮转算法给予分配CPU时间片轮转执行的，时钟中断可以控制CPU时间片的分配，这样我们可以给优先级高比较急迫的进程任务让其有更多机会获取时间片，让其有更多的机会运行。

## 署名信息
    吴欣伟 原创作品转载请注明出处 《Linux内核分析》MOOC课程: http://mooc.study.163.com/course/USTC-1000029000
  [1]: https://github.com/mengning/mykernel
