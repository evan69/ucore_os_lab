# 操作系统lab4实验报告

### 计45 侯禺凡 2014011433

## 练习0：填写已有实验

本实验依赖实验1/2/3。请把你做的实验1/2/3的代码填入本实验中代码中有“LAB1”,“LAB2”,“LAB3”的注释相应部分。

答：我使用了Ubuntu下的meld工具，将lab3中有"LAB1","LAB2"及"LAB3"注释的相应部分代码合并到lab4里。

## 练习1：分配并初始化一个进程控制块（需要编码）

alloc_proc函数（位于kern/process/proc.c中）负责分配并返回一个新的struct proc_struct结构，用于存储新建立的内核线程的管理信息。ucore需要对这个结构进行最基本的初始化，你需要完成这个初始化过程。
> 【提示】在alloc_proc函数的实现中，需要初始化的proc_struct结构中的成员变量至少包括：state/pid/runs/kstack/need_resched/parent/mm/context/tf/cr3/flags/name。

答：根据题干以及课程gitbook里[**创建第0个内核线程idleproc**](https://chyyuu.gitbooks.io/ucore_os_docs/lab4/lab4_3_3_1_create_kthread_idleproc.html)这一章节的提示，我在kern/process/proc.c中的alloc_proc函数中增加了以下的初始化代码，用来初始化struct proc_struct结构：
	
	proc->state = PROC_UNINIT;
	proc->pid = -1;
	proc->runs = 0;
	proc->kstack = 0;
	proc->need_resched = 0;
	proc->parent = 0;
	proc->mm = 0;
	proc->tf = 0;
	proc->cr3 = boot_cr3;
	proc->flags = 0;
	
	memset(&proc->context,0,sizeof(struct context));
	//memset(proc->tf,0,sizeof(struct trapframe));
	memset(proc->name,0,PROC_NAME_LEN + 1);

这段代码对于struct proc_struct结构的state、pid、runs、kstack、need_resched、parent、mm、context、tf、cr3、flags、name等变量进行了初始化操作。有的直接初始化为0，对于部分变量还要使用memset函数进行内存分配。

### 问题：请说明proc_struct中`struct context context和struct trapframe *tf`成员变量含义和在本实验中的作用是啥？（提示通过看代码和编程调试可以判断出来）

答：struct context定义在kern/process/proc.h里：

	struct context {
		uint32_t eip;
		uint32_t esp;
		uint32_t ebx;
		uint32_t ecx;
		uint32_t edx;
		uint32_t esi;
		uint32_t edi;
		uint32_t ebp;
	};

根据变量的名字，容易知道这些变量代表着寄存器。进而可以推断，context的作用是在进行进程或线程的上下文切换的过程中，保存当前寄存器的值。

struct trapframe定义在kern/trap/trap.h里：

	struct trapframe {
	    struct pushregs tf_regs;
	    uint16_t tf_gs;
	    uint16_t tf_padding0;
	    uint16_t tf_fs;
	    uint16_t tf_padding1;
	    uint16_t tf_es;
	    uint16_t tf_padding2;
	    uint16_t tf_ds;
	    uint16_t tf_padding3;
	    uint32_t tf_trapno;
	    /* below here defined by x86 hardware */
	    uint32_t tf_err;
	    uintptr_t tf_eip;
	    uint16_t tf_cs;
	    uint16_t tf_padding4;
	    uint32_t tf_eflags;
	    /* below here only when crossing rings, such as from user to kernel */
	    uintptr_t tf_esp;
	    uint16_t tf_ss;
	    uint16_t tf_padding5;
	} __attribute__((packed));

由此可以推断出，trapframe *tf保存了前一个被中断或者异常打断的进程或者线程当前的状态。

### 本练习与答案比较：

答：本练习主要是对变量做初始化工作，与答案区别如下：

- 初始化变量的顺序有所不同
- 最初我对struct trapframe* tf变量也进行了memset函数的内存分配（和char[] name等变量一样）：
 
		memset(proc->tf,0,sizeof(struct trapframe));

	结果发现不能通过测试。直到我调试之后发现这种写法存在问题，才修改过来，而答案也并没有这条语句。

## 练习2：为新创建的内核线程分配资源（需要编码）
创建一个内核线程需要分配和设置好很多资源。kernel_thread函数通过调用do_fork函数完成具体内核线程的创建工作。do_kernel函数会调用alloc_proc函数来分配并初始化一个进程控制块，但alloc_proc只是找到了一小块内存用以记录进程的必要信息，并没有实际分配这些资源。ucore一般通过do_fork实际创建新的内核线程。do_fork的作用是，创建当前内核线程的一个副本，它们的执行上下文、代码、数据都一样，但是存储位置不同。在这个过程中，需要给新内核线程分配资源，并且复制原进程的状态。你需要完成在kern/process/proc.c中的do_fork函数中的处理过程。它的大致执行步骤包括：

- 调用alloc_proc，首先获得一块用户信息块。
- 为进程分配一个内核栈。
- 复制原进程的内存管理信息到新进程（但内核线程不必做此事）
- 复制原进程上下文到新进程
- 将新进程添加到进程列表
- 唤醒新进程
- 返回新进程号

答：为了完成对新创建的内核线程分配资源，我在kern/process/proc.c中的do_fork()函数里增加以下代码：

	proc = alloc_proc();
	if(proc == 0)
		goto fork_out;
	proc->parent =  current;
	//step 1 : call alloc_proc to allocate a proc_struct
	int kstack_ret = setup_kstack(proc);
	if(kstack_ret != 0)
		goto bad_fork_cleanup_proc;
	//step 2 : call setup_kstack to allocate a kernel stack for child process
	int copy_mm_ret = copy_mm(clone_flags,proc);
	if(copy_mm_ret != 0)
		goto bad_fork_cleanup_kstack;
	//step 3 : call copy_mm to dup OR share mm according clone_flag
	copy_thread(proc,stack,tf);
	//step 4 : call copy_thread to setup tf & context in proc_struct
	bool tmp;
	local_intr_save(tmp);
	proc->pid = get_pid();
	hash_proc(proc);
	list_add(&proc_list,&proc->list_link);
	nr_process++;
	local_intr_restore(tmp);
	//step 5 : insert proc_struct into hash_list && proc_list
	wakeup_proc(proc);
	//step 6 : call wakeup_proc to make the new child process RUNNABLE
	ret = proc->pid;
	//step 7 : set ret vaule using child proc's pid

主要参考了题干以及注释里的步骤提示，主要分了七个步骤完成对内核线程的资源分配：

- 调用alloc_proc，首先获得一块用户信息块。
- 为进程分配一个内核栈。
- 复制原进程的内存管理信息到新进程（但内核线程不必做此事）
- 复制原进程上下文到新进程
- 将新进程添加到进程列表
- 唤醒新进程
- 返回新进程号

分别对应了我在代码中的注释step1~step7

### 问题：请说明ucore是否做到给每个新fork的线程一个唯一的id？请说明你的分析和理由。

答：在do_fork函数中，通过get_pid()函数为新进程分配一个pid：

	proc->pid = get_pid();

我查看get_pid()的实现，发现它通过遍历进程链表，找到一个唯一的pid返回：

	repeat:
        le = list;
        while ((le = list_next(le)) != list) {
            proc = le2proc(le, list_link);
            if (proc->pid == last_pid) {
                if (++ last_pid >= next_safe) {
                    if (last_pid >= MAX_PID) {
                        last_pid = 1;
                    }
                    next_safe = MAX_PID;
                    goto repeat;
                }
            }
            else if (proc->pid > last_pid && next_safe > proc->pid) {
                next_safe = proc->pid;
            }
        }

因此，ucore做到了给每个新fork线程一个唯一的pid。

### 本练习与答案比较：

答：**我所写代码的最初版本能够通过make grade的测试，但在阅读答案之后，发现尚有没有考虑到的地方，所以参考答案加以了改正**。我**最初版本的实现与答案的区别**如下：

- 没有将新线程的parent变量设为current，也即没有

		proc->parent =  current;

	这行代码；

- 没有在fork过程中关中断和开中断，使在切换进程的过程中不被中断打断。也即没有执行

		local_intr_save(tmp);
		//....
		local_intr_restore(tmp);

经测试，上述两个问题不影响我的代码通过make grade。但在我比较了答案之后已改正。

## 练习3：阅读代码，理解 proc_run 函数和它调用的函数如何完成进程切换的。（无编码工作）
请在实验报告中简要说明你对proc_run函数的分析。并回答如下问题：

### 问题一：在本实验的执行过程中，创建且运行了几个内核线程？

答：两个。第一个为idleproc线程，第二个为initproc线程。

### 问题二：语句`local_intr_save(intr_flag);....local_intr_restore(intr_flag);`在这里有何作用?请说明理由

答：这两条语句分别用于关中断和开中断，作用是使在切换进程的过程中不被中断打断，避免操作系统出现问题。

## LAB4运行结果

在LAB4目录下执行

	make grade

输出为：

![](http://i.imgur.com/iMVtOAg.png)

执行

	make qemu

输出为：

![](http://i.imgur.com/8lgD8q8.png)

说明结果正确

## 其他
### 列出你认为本实验中重要的知识点，以及与对应的OS原理中的知识点，并简要说明你对二者的含义，关系，差异等方面的理解（也可能出现实验中的知识点没有对应的原理知识点）

答：本实验中重要的知识点我认为有如下几个：

- ucore对内核线程的实现机制
- 进程控制块的初始化过程
- 内核线程的创建和资源分配过程

### 列出你认为OS原理中很重要，但在实验中没有对应上的知识点

答：我认为有如下几点，在OS原理中很重要，但在实验中没有对应：

- 没有对用户级线程的涉及
- 没有对进程的涉及
- 没有对进程/线程调度算法的涉及
