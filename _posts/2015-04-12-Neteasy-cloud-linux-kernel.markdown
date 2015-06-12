---
layout:     post
title:      "分析Linux内核创建一个新进程的过程"
subtitle:   "网易云课堂`Linux内核分析`课程"
date:       2015-04-12 13:00::00
author:     "MarkWoo"
header-img: "img/home-bg.jpg"
---

#分析Linux内核创建一个新进程的过程

#前言说明
本篇为网易云课堂Linux内核分析课程的第六周作业，本次作业我们将具体来分析`fork`系统调用，来分析Linux内核创建新进程的过程

---
##关键词：`fork`, `系统调用`，`进程`

---
*运行环境：**

- Ubuntu 14.04 LTS x64
- gcc 4.9.2
- gdb 7.8
- vim 7.4 with vundle

---
#分析
##分析方法说明
- `PCB`包含了一个进程的重要运行信息，所以我们将围绕在创建一个新进程时，如何来建立一个新的`PCB`的这一个过程来进行分析，在`Linux`系统中，`PCB`主要是存储在一个叫做`task_struct`这一个结构体中，创建新进程仅能通过`fork`,`clone`,`vfork`等系统调用的形式来进行
- 不管是`fork`，还是`clone`，`vfork`,他们都是通过`do_fork`来创建进程
- 接下来我将通过精简版的`do_fork`代码，和`do_fork`中关键的过程来进行分析说明

##do_fork()
```c
long do_fork(unsigned long clone_flags,
	      unsigned long stack_start,
	      unsigned long stack_size,
	      int __user *parent_tidptr,
	      int __user *child_tidptr)
{
	struct task_struct *p; //进程描述符结构体指针
	int trace = 0;
	long nr; //总的pid数量

	/*
	 * Determine whether and which event to report to ptracer.  When
	 * called from kernel_thread or CLONE_UNTRACED is explicitly
	 * requested, no event is reported; otherwise, report if the event
	 * for the type of forking is enabled.
	 */
	if (!(clone_flags & CLONE_UNTRACED)) {
		if (clone_flags & CLONE_VFORK)
			trace = PTRACE_EVENT_VFORK;
		else if ((clone_flags & CSIGNAL) != SIGCHLD)
			trace = PTRACE_EVENT_CLONE;
		else
			trace = PTRACE_EVENT_FORK;

		if (likely(!ptrace_event_enabled(current, trace)))
			trace = 0;
	}

	// 复制进程描述符，返回创建的task_struct的指针
	p = copy_process(clone_flags, stack_start, stack_size,
			 child_tidptr, NULL, trace);
	/*
	 * Do this prior waking up the new thread - the thread pointer
	 * might get invalid after that point, if the thread exits quickly.
	 */
	if (!IS_ERR(p)) {
		struct completion vfork;
		struct pid *pid;

		trace_sched_process_fork(current, p);

		// 取出task结构体内的pid
		pid = get_task_pid(p, PIDTYPE_PID);
		nr = pid_vnr(pid);

		if (clone_flags & CLONE_PARENT_SETTID)
			put_user(nr, parent_tidptr);

		// 如果使用的是vfork，那么必须采用某种完成机制，确保父进程后运行
		if (clone_flags & CLONE_VFORK) {
			p->vfork_done = &vfork;
			init_completion(&vfork);
			get_task_struct(p);
		}

		// 将子进程添加到调度器的队列，使得子进程有机会获得CPU
		wake_up_new_task(p);

		/* forking complete and child started to run, tell ptracer */
		if (unlikely(trace))
			ptrace_event_pid(trace, pid);

		// 如果设置了 CLONE_VFORK 则将父进程插入等待队列，并挂起父进程直到子进程释放自己的内存空间
		// 保证子进程优先于父进程运行
		if (clone_flags & CLONE_VFORK) {
			if (!wait_for_vfork_done(p, &vfork))
				ptrace_event_pid(PTRACE_EVENT_VFORK_DONE, pid);
		}

		put_pid(pid);
	} else {
		nr = PTR_ERR(p);
	}
	return nr;
}
```
- 对于`do_fork`中比较重要的过程，我已经注释加以说明，这里我抽象的加以把过程总结一边
    >* 通过`copy_process`来复制进程描述符，返回新创建的子进程的task_struct的指针(即PCB指针)
    >* 将新创建的子进程放入调度器的队列中，让其有机会获得CPU,并且要确保子进程要先于父进程运行，
    >* **这里为什么要确保子进程先于父进程运行呢？**，答案是在Linux系统中，有一个叫做copy_on_write技术（写时拷贝技术），该技术的作用是创建新进程时可以减少系统开销，具体该技术的细节请各位Google之，这里子进程先于父进程运行可以保证写时拷贝技术发挥其作用

- 这里有一个**重点的地方需要说明**,在使用`get_pid`系统调用时，返回的并不是进程的`pid`,而是线程的`tgid`,`tgid`指的是一个线程组当时领头的进程的pid
- 在`do_fork`中，`copy_process`函数是比较重要的，其作用是创建进程描述符以及子进程所需要的其他所有数据结构为子进程准备运行环境,下面我将深入`copy_process`中来详细分析

---
##copy_process
```c
/*
	创建进程描述符以及子进程所需要的其他所有数据结构
	为子进程准备运行环境
*/
static struct task_struct *copy_process(unsigned long clone_flags,
					unsigned long stack_start,
					unsigned long stack_size,
					int __user *child_tidptr,
					struct pid *pid,
					int trace)
{
    ...
	int retval;
	struct task_struct *p;

    ...
	// 分配一个新的task_struct，此时的p与当前进程的task，仅仅是stack地址不同
	p = dup_task_struct(current);
	if (!p)
		goto fork_out;

	···
	
	retval = -EAGAIN;
	// 检查该用户的进程数是否超过限制
	if (atomic_read(&p->real_cred->user->processes) >=
			task_rlimit(p, RLIMIT_NPROC)) {
		// 检查该用户是否具有相关权限，不一定是root
		if (p->real_cred->user != INIT_USER &&
		    !capable(CAP_SYS_RESOURCE) && !capable(CAP_SYS_ADMIN))
			goto bad_fork_free;
	}
	current->flags &= ~PF_NPROC_EXCEEDED;

	retval = copy_creds(p, clone_flags);
	if (retval < 0)
		goto bad_fork_free;

	/*
	 * If multiple threads are within copy_process(), then this check
	 * triggers too late. This doesn't hurt, the check is only there
	 * to stop root fork bombs.
	 */
	retval = -EAGAIN;
	// 检查进程数量是否超过 max_threads，后者取决于内存的大小
	if (nr_threads >= max_threads)
		goto bad_fork_cleanup_count;

	if (!try_module_get(task_thread_info(p)->exec_domain->module))
		goto bad_fork_cleanup_count;

	delayacct_tsk_init(p);	/* Must remain after dup_task_struct() */
	p->flags &= ~(PF_SUPERPRIV | PF_WQ_WORKER);
	// 表明子进程还没有调用exec系统调用
	p->flags |= PF_FORKNOEXEC;
	INIT_LIST_HEAD(&p->children);
	INIT_LIST_HEAD(&p->sibling);
	rcu_copy_process(p);
	p->vfork_done = NULL;

	// 初始化自旋锁
	spin_lock_init(&p->alloc_lock);

	// 初始化挂起信号
	init_sigpending(&p->pending);

	// 初始化定时器
	p->utime = p->stime = p->gtime = 0;
	p->utimescaled = p->stimescaled = 0;
#ifndef CONFIG_VIRT_CPU_ACCOUNTING_NATIVE
	p->prev_cputime.utime = p->prev_cputime.stime = 0;
#endif
#ifdef CONFIG_VIRT_CPU_ACCOUNTING_GEN
	seqlock_init(&p->vtime_seqlock);
	p->vtime_snap = 0;
	p->vtime_snap_whence = VTIME_SLEEPING;
#endif

    ...

#ifdef CONFIG_DEBUG_MUTEXES
	p->blocked_on = NULL; /* not blocked yet */
#endif
#ifdef CONFIG_BCACHE
	p->sequential_io	= 0;
	p->sequential_io_avg	= 0;
#endif

	/* Perform scheduler related setup. Assign this task to a CPU. */
	
	// 完成对新进程调度程序数据结构的初始化，并把新进程的状态设置为TASK_RUNNING
	// 同时将thread_info中得preempt_count置为1，禁止内核抢占
	retval = sched_fork(clone_flags, p);
	if (retval)
		goto bad_fork_cleanup_policy;

	retval = perf_event_init_task(p);
	if (retval)
		goto bad_fork_cleanup_policy;
	retval = audit_alloc(p);
	if (retval)
		goto bad_fork_cleanup_perf;
	/* copy all the process information */

	// 复制所有的进程信息
	shm_init_task(p);
	retval = copy_semundo(clone_flags, p);
	if (retval)
		goto bad_fork_cleanup_audit;
	retval = copy_files(clone_flags, p);
	if (retval)
		goto bad_fork_cleanup_semundo;
		
	...

	// 初始化子进程的内核栈
	retval = copy_thread(clone_flags, stack_start, stack_size, p);
	if (retval)
		goto bad_fork_cleanup_io;

	if (pid != &init_struct_pid) {
		retval = -ENOMEM;
		// 这里为子进程分配了新的pid号
		pid = alloc_pid(p->nsproxy->pid_ns_for_children);
		if (!pid)
			goto bad_fork_cleanup_io;
	}

	...
	
	/*
	 * sigaltstack should be cleared when sharing the same VM
	 */
	if ((clone_flags & (CLONE_VM|CLONE_VFORK)) == CLONE_VM)
		p->sas_ss_sp = p->sas_ss_size = 0;

	/*
	 * Syscall tracing and stepping should be turned off in the
	 * child regardless of CLONE_PTRACE.
	 */
	user_disable_single_step(p);
	
	// 清除子进程thread_info结构的 TIF_SYSCALL_TRACE，防止 ret_from_fork将系统调用消息通知给调试进程
	clear_tsk_thread_flag(p, TIF_SYSCALL_TRACE);
#ifdef TIF_SYSCALL_EMU
	clear_tsk_thread_flag(p, TIF_SYSCALL_EMU);
#endif
	clear_all_latency_tracing(p);

	/* ok, now we should be set up.. */
	
	// 设置子进程的pid
	p->pid = pid_nr(pid);
	
	// 如果是创建线程
	if (clone_flags & CLONE_THREAD) {
		p->exit_signal = -1;
		
		// 线程组的leader设置为当前线程的leader
		p->group_leader = current->group_leader;
		
		// tgid是当前线程组的id，也就是main进程的pid
		p->tgid = current->tgid;
	} else {
		if (clone_flags & CLONE_PARENT)
			p->exit_signal = current->group_leader->exit_signal;
		else
			p->exit_signal = (clone_flags & CSIGNAL);
			
		// 创建的是进程，自己是一个单独的线程组
		p->group_leader = p;
		
		// tgid和pid相同
		p->tgid = p->pid;
	}

    ...
    
	if (likely(p->pid)) {
		ptrace_init_task(p, (clone_flags & CLONE_PTRACE) || trace);

		init_task_pid(p, PIDTYPE_PID, pid);
		if (thread_group_leader(p)) {
			init_task_pid(p, PIDTYPE_PGID, task_pgrp(current));
			init_task_pid(p, PIDTYPE_SID, task_session(current));

			if (is_child_reaper(pid)) {
				ns_of_pid(pid)->child_reaper = p;
				p->signal->flags |= SIGNAL_UNKILLABLE;
			}

			p->signal->leader_pid = pid;
			p->signal->tty = tty_kref_get(current->signal->tty);
			list_add_tail(&p->sibling, &p->real_parent->children);
			list_add_tail_rcu(&p->tasks, &init_task.tasks);
			
			// 将pid加入散列表
			attach_pid(p, PIDTYPE_PGID);
			attach_pid(p, PIDTYPE_SID);
			__this_cpu_inc(process_counts);
		} else {
			current->signal->nr_threads++;
			atomic_inc(&current->signal->live);
			atomic_inc(&current->signal->sigcnt);
			list_add_tail_rcu(&p->thread_group,
					  &p->group_leader->thread_group);
			list_add_tail_rcu(&p->thread_node,
					  &p->signal->thread_head);
		}
		// 将pid加入PIDTYPE_PID这个散列表
		attach_pid(p, PIDTYPE_PID);
		// 递增 nr_threads的值
		nr_threads++;
	}

	total_forks++;
	spin_unlock(&current->sighand->siglock);
	syscall_tracepoint_update(p);
	write_unlock_irq(&tasklist_lock);

	...

	// 返回被创建的task结构体指针
	return p;
	
    ...
    
}
```

同样，这里关键部分我已经有了注释我针对几个关键问题进行说明：

- `copy_process`的参数与`do_fork`的参数类型一模一样，参数也是几乎相同，除了`struct pid`那里是空
- `dup_task_struct`这个函数，会分配一个新的`task_struct`给子进程，但是这个`task_struct`是未初始化的，下面我将具体说说这个`dup_task_struct`:
- `copy_process`中会进行各种各样的初始化和信息检查，比如初始化自旋锁，初始化堆栈信息等等，同时会把新创建的子进程运行状态置为`TASK_RUNNING`（这里应该是就绪态）
- `copy_process`中，会通过`copy_thread`来初始化子进程的内核栈,下面也会进行具体说明

###dup_task_struct
```c
static struct task_struct *dup_task_struct(struct task_struct *orig)
{
	struct task_struct *tsk;
	struct thread_info *ti;
	int node = tsk_fork_get_node(orig);
	int err;

	// 分配一个task_struct结点
	tsk = alloc_task_struct_node(node);
	if (!tsk)
		return NULL;

	// 分配一个thread_info结点，其实内部分配了一个union，包含进程的内核栈
	// 此时ti的值为栈底，在x86下为union的高地址处。
	ti = alloc_thread_info_node(tsk, node);
	if (!ti)
		goto free_tsk;

	err = arch_dup_task_struct(tsk, orig);
	if (err)
		goto free_ti;

	// 将栈底的值赋给新结点的stack
	tsk->stack = ti;
    
    ...

	/*
	 * One for us, one for whoever does the "release_task()" (usually
	 * parent)
	 */
	// 将进程描述符的使用计数器置为2
	atomic_set(&tsk->usage, 2);
#ifdef CONFIG_BLK_DEV_IO_TRACE
	tsk->btrace_seq = 0;
#endif
	tsk->splice_pipe = NULL;
	tsk->task_frag.page = NULL;

	account_kernel_stack(ti, 1);

	// 返回新申请的结点
	return tsk;

free_ti:
	free_thread_info(ti);
free_tsk:
	free_task_struct(tsk);
	return NULL;
}
```

>* 这其中有个比较重要的结构struct thread_info,但是在内部分配时，其实是一个union(联合体),这个union包括了一个内核堆栈,其结构如下图(图来自Understanding Linux kernel 3th)

###copy_thread
```c
// 初始化子进程的内核栈
int copy_thread(unsigned long clone_flags, unsigned long sp,
	unsigned long arg, struct task_struct *p)
{

	// 取出子进程的寄存器信息
	struct pt_regs *childregs = task_pt_regs(p);
	struct task_struct *tsk;
	int err;

	// 栈顶 空栈
	p->thread.sp = (unsigned long) childregs;
	p->thread.sp0 = (unsigned long) (childregs+1);
	memset(p->thread.ptrace_bps, 0, sizeof(p->thread.ptrace_bps));

	// 如果是创建的内核线程
	if (unlikely(p->flags & PF_KTHREAD)) {
		/* kernel thread */
		memset(childregs, 0, sizeof(struct pt_regs));
		// 内核线程开始执行的位置
		p->thread.ip = (unsigned long) ret_from_kernel_thread;
		task_user_gs(p) = __KERNEL_STACK_CANARY;
		childregs->ds = __USER_DS;
		childregs->es = __USER_DS;
		childregs->fs = __KERNEL_PERCPU;
		childregs->bx = sp;	/* function */
		childregs->bp = arg;
		childregs->orig_ax = -1;
		childregs->cs = __KERNEL_CS | get_kernel_rpl();
		childregs->flags = X86_EFLAGS_IF | X86_EFLAGS_FIXED;
		p->thread.io_bitmap_ptr = NULL;
		return 0;
	}

	// 将当前进程的寄存器信息复制给子进程
	*childregs = *current_pt_regs();
	// 子进程的eax置为0，所以fork的子进程返回值为0
	childregs->ax = 0;
	if (sp)
		childregs->sp = sp;

	// 子进程从ret_from_fork开始执行
	p->thread.ip = (unsigned long) ret_from_fork;
	task_user_gs(p) = get_user_gs(current_pt_regs());

	p->thread.io_bitmap_ptr = NULL;
	tsk = current;
	err = -ENOMEM;

	// 如果父进程使用IO权限位图，那么子进程获得该位图的一个拷贝
	if (unlikely(test_tsk_thread_flag(tsk, TIF_IO_BITMAP))) {
		p->thread.io_bitmap_ptr = kmemdup(tsk->thread.io_bitmap_ptr,
						IO_BITMAP_BYTES, GFP_KERNEL);
		if (!p->thread.io_bitmap_ptr) {
			p->thread.io_bitmap_max = 0;
			return -ENOMEM;
		}
		set_tsk_thread_flag(p, TIF_IO_BITMAP);
	}

	...
	
	return err;
}
```

- 这里的过程基本就是给新的进程的各种运行时状态进行初始化，比如寄存器信息（通过父进程的寄存器信息来初始化，但是eip会是个例外，eip将会取决于最后子进程将会从哪里开始执行），栈会被置空未初始化状态
- 在代码中，有两段这样的代码`p->thread.ip = (unsigned long) ret_from_kernel_thread;
p->thread.ip = (unsigned long) ret_from_fork;`，**这里表面了在`fork`完成之后，新进程将会在哪里开始执行**,如果新的创建的新的线程是内核线程，那么将会从`ret_from_kernel_thread`开始执行，但是如果是普通的用户态线程，则将会从`p->thread.ip = (unsigned long) ret_from_fork`开始执行.

#总结
通过实验和fork系统调用的分析，让我认识到除了Linux系统中最开始启动时，创建的第一个始祖进程外，从`init进程`开始，其他所有的进程的创建方式均是通过`fork`，`clone`,`vfork`的方式，而他们又能够有归结到`do_fork`,就想孟宁老师说得道生一，一生二，二生三这样.

---
##参考资料
- Understanding The Linux Kernel, the 3rd edtion
- Linux内核设计与实现，第三版，Robert Love, 机械工业出版社
