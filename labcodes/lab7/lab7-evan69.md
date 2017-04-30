# 操作系统lab7实验报告

### 计45 侯禺凡 2014011433

## 练习0：填写已有实验

本实验依赖实验1/2/3/4/5/6。请把你做的实验1/2/3/4/5/6的代码填入本实验中代码中有“LAB1”/“LAB2”/“LAB3”/“LAB4”/“LAB5”/“LAB6”的注释相应部分。并确保编译通过。注意：为了能够正确执行lab7的测试应用程序，可能需对已完成的实验1/2/3/4/5/6的代码进行进一步改进。

答：我使用了Ubuntu下的meld工具，将lab5中有"LAB1","LAB2","LAB3","LAB4","LAB5"及"LAB6"注释的相应部分代码合并到lab7里。由于需要支持lab7，我在lab7中注释提示的帮助下，对在前面6个实验所写的代码进行了修改，说明如下。

修改kern/trap/trap.c的trap_dispatch函数如下：

	/* LAB7 2014011433 */
    /* you should upate you lab6 code
     * IMPORTANT FUNCTIONS:
     * run_timer_list
     */
    ticks++;
    run_timer_list();
    break;

即根据提示，增加调用run_timer_list函数，而run_timer_list函数是已经被定义在sched.c文件里的：

	void
	run_timer_list(void) {
	    bool intr_flag;
	    local_intr_save(intr_flag);
	    {
	        list_entry_t *le = list_next(&timer_list);
	        if (le != &timer_list) {
	            timer_t *timer = le2timer(le, timer_link);
	            assert(timer->expires != 0);
	            timer->expires --;
	            while (timer->expires == 0) {
	                le = list_next(le);
	                struct proc_struct *proc = timer->proc;
	                if (proc->wait_state != 0) {
	                    assert(proc->wait_state & WT_INTERRUPTED);
	                }
	                else {
	                    warn("process %d's wait_state == 0.\n", proc->pid);
	                }
	                wakeup_proc(proc);
	                del_timer(timer);
	                if (le == &timer_list) {
	                    break;
	                }
	                timer = le2timer(le, timer_link);
	            }
	        }
	        sched_class_proc_tick(current);
	    }
	    local_intr_restore(intr_flag);
	}

作用是调用调度器来来更新tick相关信息，检测是否有timer到时，若有，则唤醒timer对应进程。

### 本练习与答案比较：

答：对于trap.c的trap_dispatch函数，实现的不同之处在于我没有像答案一样调用

	assert(current != NULL);
	
也即没有对current指针是否为空进行检查。经过测试不影响结果。

## 练习1：理解内核级信号量的实现和基于内核级信号量的哲学家就餐问题（不需要编码）
完成练习0后，建议大家比较一下（可用meld等文件diff比较软件）个人完成的lab6和练习0完成后的刚修改的lab7之间的区别，分析了解lab7采用信号量的执行过程。执行make grade，大部分测试用例应该通过。

### 问题1：给出内核级信号量的设计描述，并说其大致执行流流程。请在实验报告中给出内核级信号量的设计描述，并说其大致执行流流程。

答：ucore中，信号量定义在kern/sync/sem.h中：

	typedef struct {
	    int value;
	    wait_queue_t wait_queue;
	} semaphore_t;
	
	void sem_init(semaphore_t *sem, int value);
	void up(semaphore_t *sem);
	void down(semaphore_t *sem);
	bool try_down(semaphore_t *sem);

信号量中，value用于表示信号量中资源的整数值，wait_queue表示等待队列。 对于信号量存在以下几种操作：

- void sem_init(semaphore_t *sem, int value): 初始化信号量的value和wait_queue；
- void up(semaphore_t *sem): 信号量的V操作；
- void down(semaphore_t *sem): 信号量的P操作；
- bool try_down(semaphore_t *sem): 非阻塞的P操作

上面四个函数实现用到了__up()和__down()函数，它们的实现如下：

	static __noinline void __up(semaphore_t *sem, uint32_t wait_state) {
	    bool intr_flag;
	    local_intr_save(intr_flag);
	    {
	        wait_t *wait;
	        if ((wait = wait_queue_first(&(sem->wait_queue))) == NULL) {
	            sem->value ++;
	        }
	        else {
	            assert(wait->proc->wait_state == wait_state);
	            wakeup_wait(&(sem->wait_queue), wait, wait_state, 1);
	        }
	    }
	    local_intr_restore(intr_flag);
	}
	
	static __noinline uint32_t __down(semaphore_t *sem, uint32_t wait_state) {
	    bool intr_flag;
	    local_intr_save(intr_flag);
	    if (sem->value > 0) {
	        sem->value --;
	        local_intr_restore(intr_flag);
	        return 0;
	    }
	    wait_t __wait, *wait = &__wait;
	    wait_current_set(&(sem->wait_queue), wait, wait_state);
	    local_intr_restore(intr_flag);
	
	    schedule();
	
	    local_intr_save(intr_flag);
	    wait_current_del(&(sem->wait_queue), wait);
	    local_intr_restore(intr_flag);
	
	    if (wait->wakeup_flags != wait_state) {
	        return wait->wakeup_flags;
	    }
	    return 0;
	}

由上面代码实现可见，__up()函数的执行过程如下：

- 关中断
- 判断等待队列是否为空。若为空，将value值加一；否则，调用wake_up将睡眠的进程唤醒
- 开中断

__down()函数的执行过程如下：

- 关中断
- 判断信号量的value是否大于0
	- 如果是，则将value减一后，开中断返回
	- 否则，将当前进程加入到等待队列
- 开中断
- 执行schedule函数进行调度
- 关中断
- 从等待队列中移除此进程
- 开中断
- 若唤醒的标志变量和等待状态变量不一样，则返回标志变量以供调用者进行分析，否则返回0

### 问题2：请在实验报告中给出给用户态进程/线程提供信号量机制的设计方案，并比较说明给内核级提供信号量机制的异同。

答：由于实现信号量机制需要包含开关中断的操作，而这些都是特权指令，在用户态无法直接执行。因此，我的一个设计想法是，操作系统为用户提供实现信号量机制所要用到的一些系统调用，比如SYS_SEMINIT,SYS_UP,SYS_DOWN（分别与ucore中的sem_init()、up()、down()函数对应）等。比较而言，异同如下：

- 相同点：实现机制相同
- 不同点：用户态需要系统调用才能实现，实现过程中有特权级的转换

## 练习2: 完成内核级条件变量和基于内核级条件变量的哲学家就餐问题（需要编码）

首先掌握管程机制，然后基于信号量实现完成条件变量实现，然后用管程机制实现哲学家就餐问题的解决方案（基于条件变量）。

### 问题1：请在实验报告中给出内核级条件变量的设计描述，并说其大致执行流流程。

答：

ucore中管程数据结构monitor_t的定义如下：

	typedef struct monitor{
	    semaphore_t mutex;      // the mutex lock for going into the routines in monitor, should be initialized to 1
	    semaphore_t next;       // the next semaphore is used to down the signaling proc itself, and the other OR wakeuped waiting proc should wake up the sleeped signaling proc.
	    int next_count;         // the number of of sleeped signaling proc
	    condvar_t *cv;          // the condvars in monitor
	} monitor_t;

其成员变量的含义及功能为：

- mutex: 一个二值信号量，实现每次只允许一个进程进入管程的信号量，确保了互斥访问。
- cv：一个条件变量。cv通过执行wait_cv，会使得等待某个条件C为真的进程能够离开管程并睡眠，且让其他进程进入管程继续执行；而进入管程的某进程设置条件C为真并执行signal_cv时，能够让等待某个条件C为真的睡眠进程被唤醒，从而继续进入管程中执行。
- next: 一个信号量。发出signal_cv的进程A会唤醒睡眠进程B，进程B执行会导致进程A睡眠，直到进程B离开管程，进程A才能继续执行。信号量next负责保证这个同步过程。
- next_count：一个整型变量，表示由于发出singal_cv而睡眠的进程个数。

条件变量的数据结构condvar_t定义如下：

	typedef struct condvar{
	    semaphore_t sem;        // the sem semaphore  is used to down the waiting proc, and the signaling proc should up the waiting proc
	    int count;              // the number of waiters on condvar
	    monitor_t * owner;      // the owner(monitor) of this condvar
	} condvar_t;

其成员变量的含义及功能为：

- sem: 一个信号量，用于让发出wait_cv操作的等待某个条件C为真的进程睡眠，而让发出signal_cv操作的进程通过这个sem来唤醒睡眠的进程。
- count: 表示等在这个条件变量上的睡眠进程的个数。
- owner: 表示此条件变量的宿主管程。由于很多函数以条件变量为参数，owner属性便于通过条件变量查找对应的管程。

对于管程的两个重要的函数为cond_wait和cond_signal，也即本练习需要我们实现的地方，**cond_signal实现如下**：

	void 
	cond_signal (condvar_t *cvp) {
	   //LAB7 EXERCISE1: 2014011433
	   cprintf("cond_signal begin: cvp %x, cvp->count %d, cvp->owner->next_count %d\n", cvp, cvp->count, cvp->owner->next_count);  
	  /*
	   *      cond_signal(cv) {
	   *          if(cv.count>0) {
	   *             mt.next_count ++;
	   *             signal(cv.sem);
	   *             wait(mt.next);
	   *             mt.next_count--;
	   *          }
	   *       }
	   */
	   if(cvp->count > 0)
	   {
	      cvp->owner->next_count++;
	      up(&(cvp->sem));
	      down(&(cvp->owner->next));
	      cvp->owner->next_count--;
	   }
	   cprintf("cond_signal end: cvp %x, cvp->count %d, cvp->owner->next_count %d\n", cvp, cvp->count, cvp->owner->next_count);
	}

由上述代码可见cond_signal函数的执行过程如下：

- 通过cvp->count的计数，判断是否存在等待此条件的进程
- 如果有，则：
	- 自增next_count
	- 唤醒这个等待条件的进程
	- 将本进程睡眠在cvp->owner->next上，等待其他进程将本进程再次唤醒
	- 自减next_count
- 否则，不作任何操作

**cond_wait函数实现如下**：
	
	void
	cond_wait (condvar_t *cvp) {
	    //LAB7 EXERCISE1: 2014011433
	    cprintf("cond_wait begin:  cvp %x, cvp->count %d, cvp->owner->next_count %d\n", cvp, cvp->count, cvp->owner->next_count);
	   /*
	    *         cv.count ++;
	    *         if(mt.next_count>0)
	    *            signal(mt.next)
	    *         else
	    *            signal(mt.mutex);
	    *         wait(cv.sem);
	    *         cv.count --;
	    */
	    cvp->count++;
	    if(cvp->owner->next_count > 0)
	    {
	        up(&(cvp->owner->next));
	    }
	    else
	    {
	        up(&(cvp->owner->mutex));
	    }
	    down(&(cvp->sem));
	    cvp->count--;
	    cprintf("cond_wait end:  cvp %x, cvp->count %d, cvp->owner->next_count %d\n", cvp, cvp->count, cvp->owner->next_count);
	}

由上述代码可见cond_wait函数的执行过程如下：

- 条件变量的count自增
- 如果monitor.next_count大于0，表示有进程执行cond_signal函数睡眠了，这些进程构成一个链表，需要唤醒链表中的一个进程。
- 否则，需要唤醒由于互斥而不能进入管程的进程链表中的一个进程
- 对条件变量的信号量执行P操作，请求访问资源
- 条件变量的count自减

**为了实现哲学家就餐问题，实现phi_take_forks_condvar和phi_put_forks_condvar函数如下**：

	void phi_take_forks_condvar(int i) {
	     down(&(mtp->mutex));
	//--------into routine in monitor--------------
	     // LAB7 EXERCISE1: 2014011433
	     state_condvar[i] = HUNGRY;
	     // I am hungry
	     phi_test_condvar(i);
	     if(state_condvar[i] != EATING)
	     {
	        cond_wait(&(mtp->cv[i]));
	     }
	     // try to get fork
	
	//--------leave routine in monitor--------------
	      if(mtp->next_count>0)
	         up(&(mtp->next));
	      else
	         up(&(mtp->mutex));
	}
	
	void phi_put_forks_condvar(int i) {
	     down(&(mtp->mutex));
	
	//--------into routine in monitor--------------
	     // LAB7 EXERCISE1: 2014011433
	     state_condvar[i] = THINKING;
	      // I ate over
	     phi_test_condvar((N+i-1) % N);
	     phi_test_condvar((i+1) % N);
	     // test left and right neighbors
	//--------leave routine in monitor--------------
	     if(mtp->next_count>0)
	        up(&(mtp->next));
	     else
	        up(&(mtp->mutex));
	}

对于phi_take_forks_condvar，先通过P操作尝试进入管程，之后将对应的哲学家设为饥饿状态，调用phi_test_condvar函数测试自己能否拿到叉子开始用餐，若不能，等待对应的条件变量，最后通过V操作离开管程。

对于phi_put_forks_condvar，先通过P操作尝试进入管程，之后将自己设置为思考也即用餐完毕状态，调用phi_test_condvar函数测试自己旁边的两个哲学家能否开始用餐，最后通过V操作离开管程。

### 问题2：请在实验报告中给出给用户态进程/线程提供条件变量机制的设计方案，并比较说明给内核级提供条件变量机制的异同。

答：思路和练习1的问题2一致，可以让操作系统提供系统调用给用户使用。

### 问题3：请在实验报告中回答：能否不用基于信号量机制来完成条件变量？如果不能，请给出理由，如果能，请给出设计说明和具体实现。

答：我认为可以不用基于信号量机制来完成。大致思路如下：

	Class Condition
	{
		int numWaiting = 0;
		WaitQueue q;
	}
	
	Condition::Wait(lock)
	{
		numWaiting++;
		Add thread t to q;
		release(lock);
		schedule();
		require(lock);
	}
	
	Condition::Signal()
	{
		if(numWaiting > 0)
		{
			remove a thread t from q;
			wakeup(t);
			numWaiting--;
		}
	}

### 本练习与答案比较：

答：在phi_take_forks_condvar函数中，我有这样一段代码：

	 if(state_condvar[i] != EATING)
     {
        cond_wait(&(mtp->cv[i]));
     }

而答案的实现为：

	while (state_condvar[i] != EATING) {
		cond_wait(mtp->cv + i);
	}

我认为我的更加合理，并且也能通过make grade测试。

## LAB7运行结果

在LAB7目录下执行

	make grade

输出为：

![](http://i.imgur.com/QH5Pgpr.png)

说明结果正确

## 其他
### 列出你认为本实验中重要的知识点，以及与对应的OS原理中的知识点，并简要说明你对二者的含义，关系，差异等方面的理解（也可能出现实验中的知识点没有对应的原理知识点）

答：本实验中重要的知识点我认为有如下几个：

- 同步互斥的基本概念
- 信号量的原理与实现
- 管程、条件变量的原理与实现
- 哲学家就餐问题

### 列出你认为OS原理中很重要，但在实验中没有对应上的知识点

答：我认为有如下几点，在OS原理中很重要，但在实验中没有对应：

- 其他同步互斥类问题，如读者-写者问题