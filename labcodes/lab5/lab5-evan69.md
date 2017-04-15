# 操作系统lab5实验报告

### 计45 侯禺凡 2014011433

## 练习0：填写已有实验

本实验依赖实验1/2/3/4。请把你做的实验1/2/3/4的代码填入本实验中代码中有“LAB1”/“LAB2”/“LAB3”/“LAB4”的注释相应部分。注意：为了能够正确执行lab5的测试应用程序，可能需对已完成的实验1/2/3/4的代码进行进一步改进。

答：我使用了Ubuntu下的meld工具，将lab4中有"LAB1","LAB2","LAB3"及"LAB4"注释的相应部分代码合并到lab5里。由于需要支持lab5，我在lab5中注释提示的帮助下，对在前面四个实验所写的代码进行了修改，说明如下。

首先是对kern/trap/trap.c的idt_init函数做修改：

    SETGATE(idt[T_SWITCH_TOK], 0, GD_KTEXT, __vectors[T_SWITCH_TOK], DPL_USER);
    //add this line :
    SETGATE(idt[T_SYSCALL], 1, GD_KTEXT, __vectors[T_SYSCALL], DPL_USER);
    lidt(&idt_pd);

为了支持用户进程对系统调用的访问，这里在idt_init函数里设置一个特定中断号的中断门，专门用于用户进程访问系统调用。

其次，修改kern/trap/trap.c的trap_dispatch函数：

    ticks++;
    if (ticks % TICK_NUM == 0)
    {
        //print_ticks();
        //add two lines:
        assert(current != NULL);
        current->need_resched = 1;
    }
    break;

此处即是将当前进程设为可以被调度。

接下来，修改kern/process/proc.c的do_fork函数：

    proc->parent =  current;
    //add this line :
    assert(current->wait_state == 0);
    //step 1 : call alloc_proc to allocate a proc_struct
    int kstack_ret = setup_kstack(proc);
    if(kstack_ret != 0)
      goto bad_fork_cleanup_proc;
    //......
    local_intr_save(tmp);
    proc->pid = get_pid();
    hash_proc(proc);
    //list_add(&proc_list,&proc->list_link);
    //nr_process++;
    //add this line :
    set_links(proc);

    local_intr_restore(tmp);
    //......

这里主要是确保当前进程的wait_state为0以及为当前进程设置relation links。

最后，根据提示还要在kern/process/proc.c的alloc_proc函数里增加初始化lab5要用到的几个成员变量：
		
		//......
		memset(proc->name,0,PROC_NAME_LEN + 1);
	
		//add two lines :
		proc->wait_state = 0;
		proc->cptr = proc->optr = proc->yptr = NULL;
	}
	return proc;

### 本练习与答案比较：

答：由于注释中给出的提示比较详细完整，我在练习0的实现上和答案基本相同，主要有一点区别：

在kern/trap/trap.c的idt_init函数中，我相比lab1增加了一条和系统调用相关的设置语句，而答案是将lab1中的另一条设置语句替换为这个和系统调用相关的设置语句。

## 练习1：加载应用程序并执行（需要编码）
do_execv函数调用load_icode（位于kern/process/proc.c中）来加载并解析一个处于内存中的ELF执行文件格式的应用程序，建立相应的用户内存空间来放置应用程序的代码段、数据段等，且要设置好proc_struct结构中的成员变量trapframe中的内容，确保在执行此进程后，能够从应用程序设定的起始执行地址开始执行。需设置正确的trapframe内容。请在实验报告中简要说明你的设计实现过程。

答：在kern/process/proc.c的load_icode函数中补充下面代码：

    tf->tf_cs = USER_CS;
    tf->tf_ds = tf->tf_es = tf->tf_ss = USER_DS;
    tf->tf_esp = USTACKTOP;
    tf->tf_eip = elf->e_entry;
    //tf->tf_eflags = 1;
    tf->tf_eflags = FL_IF;

这段代码所做的初始化工作有：

- 设置tf_cs为用户态代码段
- 设置tf_ds、tf_es、tf_ss为用户态数据段
- 设置tf_esp为用户栈栈顶
- 设置tf_eip为可执行程序的入口点
- 设置tf_eflags以使能中断

### 问题：描述当创建一个用户态进程并加载了应用程序后，CPU是如何让这个应用程序最终在用户态执行起来的。即这个用户态进程被ucore选择占用CPU执行（RUNNING态）到具体执行应用程序第一条指令的整个经过。

答：整个经过如下：

1. 通过schedule找到下一个需要被执行的进程
2. 调用proc_run，切换栈和页表，调用switch_to函数切换上下文
3. switch_to函数返回至forkret，进而执行forkrets函数
4. 设置栈指针，弹出段寄存器，执行tf->tf_eip所指向的地址处的指令
5. 由于tf->tf_eip被设置为了elf->e_entry，从而开始执行用户程序第一条指令

### 本练习与答案比较：

答：根据答案的提示，需要设置eflags以使能中断。最开始我不知道FL_IF宏的存在，便设置eflags为1，发现能够通过测试。后来和答案对比时才发现答案将eflags设置为了FL_IF宏。**考虑到在这一点上答案更加合理，我也修改为了答案的实现方式**。

## 练习2：父进程复制自己的内存空间给子进程（需要编码）
创建子进程的函数do_fork在执行中将拷贝当前进程（即父进程）的用户内存地址空间中的合法内容到新进程中（子进程），完成内存资源的复制。具体是通过copy_range函数（位于kern/mm/pmm.c中）实现的，请补充copy_range的实现，确保能够正确执行。请在实验报告中简要说明如何设计实现”Copy on Write 机制“，给出概要设计，鼓励给出详细设计。

答：在kern/mm/pmm.c的copy_range函数里增加以下代码：
	
	void* src_kvadddr = page2kva(page);
	void* dst_kvadddr = page2kva(npage);
	memcpy(dst_kvadddr,src_kvadddr,PGSIZE);
	ret = page_insert(to, npage, start, perm);

这段代码所做的主要工作为：

- 使用page2kva函数获得要拷贝的源页面和目标页面地址
- 用memcpy对它们进行拷贝
- 设置与物理页的映射关系

### 问题：请在实验报告中简要说明如何设计实现”Copy on Write 机制“，给出概要设计，鼓励给出详细设计。

答：在copy_range函数中，不直接将父进程拷贝一份给子进程，而是用一个标记为表示此页被多个使用者访问。当进行读操作时，可以共享资源。当进行写操作时，先判断此页是否共享，再进行拷贝的操作。

### 本练习与答案比较：

答：本练习有注释的提示，共分四步每步仅有一条语句，实现较为简单，因此和答案除了变量命名外没有不同。

## 练习3： 阅读分析源代码，理解进程执行 fork/exec/wait/exit 的实现，以及系统调用的实现（不需要编码）
答：

- fork: 通过do_fork函数实现。首先分配并初始化进程控制块，分配并初始化内核栈，根据clone_flag标志复制或共享进程内存管理结构。接下来，设置进程在内核(将来也包括用户态)正常运行和调度所需的中断帧和执行上下文，把设置好的进程控制块放入hash_list和proc_list两个全局进程链表中。自此,进程已经准备好执行了，最后把进程状态设置为“就绪”态，设置返回值为子进程的pid。

- exec: 通过do_execve函数实现。首先为加载新的执行码做好用户态内存空间清空准备。如果mm不为NULL，则设置页表为内核空间页表，且进一步判断mm的引用计数减1后是否为0，如果为0，则表明没有进程再需要此进程所占用的内存空间，为此将根据mm中的记录，释放进程所占用户空间内存和进程页表本身所占空间。最后把当前进程的mm内存管理指针为空。接下来的一步是加载应用程序执行码到当前进程的新创建的用户态虚拟空间中。这里涉及到读ELF格式的文件，申请内存空间，建立用户态虚存空间，加载应用程序执行码等。load_icode函数完成了整个复杂的工作。

- wait: 通过do_wait函数实现。如果pid不为0，表示只找一个进程id号为pid的退出状态的子进程，否则找任意一个处于退出状态的子进程；如果此子进程的执行状态不为PROC_ZOMBIE，表明此子进程还没有退出，则当前进程设置自己的执行状态为PROC_SLEEPING，进入等待状态，调用schedule()函数选择新的进程执行；如果此子进程的执行状态为PROC_ZOMBIE，表明此子进程处于退出状态，需要当前进程（即子进程的父进程）完成对子进程的最终回收工作，即首先把子进程控制块从两个进程队列proc_list和hash_list中删除，并释放子进程的内核堆栈和进程控制块。自此，子进程才彻底地结束了它的执行过程。

- exit: 通过do_exit函数实现。如果current->mm != NULL，表示是用户进程，则开始回收此用户进程所占用的用户态虚拟内存空间，包括切换页表和回收资源。这时，设置当前进程的执行状态current->state=PROC_ZOMBIE，当前进程的退出码current->exit_code=error_code。此时当前进程已经不能被调度了，需要此进程的父进程来做最后的回收工作，让父进程帮助自己完成最后的资源回收。如果当前进程还有子进程，则需要把这些子进程的父进程指针设置为内核线程initproc，且各个子进程指针需要插入到initproc的子进程链表中。如果某个子进程的执行状态是PROC_ZOMBIE，则需要唤醒initproc来完成对此子进程的最后回收工作。最后执行schedule()函数，选择新的进程执行。

- 系统调用：首先初始化系统调用的中断描述符，设置特权级为DPL_USER；然后建立系统调用的用户库准备；在用户进行系统调用时，根据系统调用编号，跳转到相应的例程入口进行处理。

### 问题一：请分析fork/exec/wait/exit在实现中是如何影响进程的执行状态的？

答：

- fork: fork产生出子进程，即子进程进入**就绪状态**
- exec: exec以新的进程代替原进程，即新进程进入**就绪状态**
- wait: 当前进程等待子进程的结束，即进入**等待状态**
- exit: 当前进程作为某个进程的子进程，成为僵尸进程，即进入**僵尸状态**

### 问题二：请给出ucore中一个用户态进程的执行状态生命周期图（包执行状态，执行状态之间的变换关系，以及产生变换的事件或函数调用）。（字符方式画即可）

答：
                        
	  alloc_proc                                 RUNNING
	      |                                   +--<----<--+
	      |                                   + proc_run +
	      V                                   +-->---->--+
	PROC_UNINIT -- proc_init/wakeup_proc --> PROC_RUNNABLE -- try_free_pages/do_wait/do_sleep --> PROC_SLEEPING --+
	                                            |      ^                                                          |      
	                                            |      |                                                          |
		            PROC_ZOMBIE <-- do_exit ----+      |                                                          |
	                                                   +-----------------------wakeup_proc------------------------+


## LAB5运行结果

在LAB5目录下执行

	make grade

输出为：

![](http://i.imgur.com/qlhIlks.png)

说明结果正确

## 其他
### 列出你认为本实验中重要的知识点，以及与对应的OS原理中的知识点，并简要说明你对二者的含义，关系，差异等方面的理解（也可能出现实验中的知识点没有对应的原理知识点）

答：本实验中重要的知识点我认为有如下几个：

- 用户进程的加载
- 用户进程的执行
- 父进程复制内存给子进程

### 列出你认为OS原理中很重要，但在实验中没有对应上的知识点

答：我认为有如下几点，在OS原理中很重要，但在实验中没有对应：

- 进程控制的三状态模型及其转换
- 进程的挂起状态
