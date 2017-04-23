# 操作系统lab6实验报告

### 计45 侯禺凡 2014011433

## 练习0：填写已有实验

本实验依赖实验1/2/3/4/5。请把你做的实验1/2/3/4/5的代码填入本实验中代码中有“LAB1”/“LAB2”/“LAB3”/“LAB4”“LAB5”的注释相应部分。并确保编译通过。注意：为了能够正确执行lab6的测试应用程序，可能需对已完成的实验1/2/3/4/5的代码进行进一步改进。

答：我使用了Ubuntu下的meld工具，将lab5中有"LAB1","LAB2","LAB3","LAB4"及"LAB5"注释的相应部分代码合并到lab6里。由于需要支持lab6，我在lab6中注释提示的帮助下，对在前面5个实验所写的代码进行了修改，说明如下。

首先是对kern/process/proc.c的alloc_proc函数做修改：

	//add code for lab6
	proc->rq = NULL;
	list_init(&(proc->run_link));
	proc->time_slice = 0;
	proc->lab6_run_pool.left = proc->lab6_run_pool.right = proc->lab6_run_pool.parent = NULL;
	proc->lab6_stride = 0;
	proc->lab6_priority = 0;

主要是为了lab6而初始化新增加的一些进程控制相关成员变量。其次，修改kern/trap/trap.c的trap_dispatch函数：

	/* LAB6 2014011433 */
	/* you should upate you lab5 code
	 * IMPORTANT FUNCTIONS:
	 * sched_class_proc_tick
	 */
	ticks++;
	run_timer_list(current);
	break;

而run_timer_list函数是我新定义在sched.c文件里的：

	void
	run_timer_list(struct proc_struct *proc) {
	    sched_class_proc_tick(proc);
	}

作用是调用调度算法里的sched_class_proc_tick函数进行对时钟节拍进行处理。

### 本练习与答案比较：

答：对于练习0的实现，我在补充proc.c的alloc_proc函数时和答案基本相同，而对于trap.c的trap_dispatch函数，则有以下不同之处：

- 我没有像答案一样调用

		assert(current != NULL);
	
	也即没有对current指针是否为空进行检查。经过测试不影响结果。

- 我受到注释的提示，调用了`run_timer_list()`函数来调用`sched_class_proc_tick()`函数，而答案中并没有此处的调用。

## 练习1：使用 Round Robin 调度算法（不需要编码）
完成练习0后，建议大家比较一下（可用kdiff3等文件比较软件）个人完成的lab5和练习0完成后的刚修改的lab6之间的区别，分析了解lab6采用RR调度算法后的执行过程。执行make grade，大部分测试用例应该通过。但执行priority.c应该过不去。

### 问题一：请理解并分析sched_class中各个函数指针的用法，并接合Round Robin调度算法描ucore的调度执行过程

答：sched_class的定义在kern/schedule/sched.h中：

	struct sched_class {
	    // the name of sched_class
	    const char *name;
	    // Init the run queue
	    void (*init)(struct run_queue *rq);
	    // put the proc into runqueue, and this function must be called with rq_lock
	    void (*enqueue)(struct run_queue *rq, struct proc_struct *proc);
	    // get the proc out runqueue, and this function must be called with rq_lock
	    void (*dequeue)(struct run_queue *rq, struct proc_struct *proc);
	    // choose the next runnable task
	    struct proc_struct *(*pick_next)(struct run_queue *rq);
	    // dealer of the time-tick
	    void (*proc_tick)(struct run_queue *rq, struct proc_struct *proc);
	    /* for SMP support in the future
	     *  load_balance
	     *     void (*load_balance)(struct rq* rq);
	     *  get some proc from this rq, used in load_balance,
	     *  return value is the num of gotten proc
	     *  int (*get_proc)(struct rq* rq, struct proc* procs_moved[]);
	     */
	}

在sched.c中，实例化一个sched_class对象为`sched_class = &default_sched_class`，并进行初始化。在后续进行进程选择和切换的时候，分别调用sched_class_enqueue、sched_class_pick_next、 sched_class_dequeue等sched_class类中的函数指针，以执行相应的功能。

在Round Robin调度算法中，通过将default_sched_class指向为相应的实现，即可使用RR调度算法。

	struct sched_class default_sched_class = {
	    .name = "RR_scheduler",
	    .init = RR_init,
	    .enqueue = RR_enqueue,
	    .dequeue = RR_dequeue,
	    .pick_next = RR_pick_next,
	    .proc_tick = RR_proc_tick,
	};

### 问题二：请在实验报告中简要说明如何设计实现”多级反馈队列调度算法“，给出概要设计，鼓励给出详细设计

答：实现要点描述如下：

1. 首先维护多个链表，每个链表保存优先级不同的就绪队列
2. 进程在初始化时首先插入优先级最高的队列等待
3. 在调度时，从优先级最高的队列中开始寻找进程，若为空则进入下一优先级队列寻找
4. 对于某个进程，如果在规定的时间片内没有完成，则将其插入下一优先级队列的链表中

## 练习2：实现 Stride Scheduling 调度算法（需要编码）

答：首先需要换掉RR调度器的实现，然后注释对Stride度器的相关描述，完成Stride调度算法的实现。本题我主要实现了kern/schedule/default_sched.c中的五个函数：

- stride_init函数：

		static void
		stride_init(struct run_queue *rq) {
		     /* LAB6: 2014011433 
		      * (1) init the ready process list: rq->run_list
		      * (2) init the run pool: rq->lab6_run_pool
		      * (3) set number of process: rq->proc_num to 0       
		      */
		     list_init(&(rq->run_list));
		     rq->lab6_run_pool = NULL;
		     rq->proc_num = 0;
		}
	
	这个函数主要是进行初始化操作，初始化队列，同时将proc_num变量置零。

- stride_enqueue函数：

		static void
		stride_enqueue(struct run_queue *rq, struct proc_struct *proc) {
		     /* LAB6: 2014011433 
		      * (1) insert the proc into rq correctly
		      * NOTICE: you can use skew_heap or list. Important functions
		      *         skew_heap_insert: insert a entry into skew_heap
		      *         list_add_before: insert  a entry into the last of list   
		      * (2) recalculate proc->time_slice
		      * (3) set proc->rq pointer to rq
		      * (4) increase rq->proc_num
		      */
			rq->lab6_run_pool = skew_heap_insert(rq->lab6_run_pool,&(proc->lab6_run_pool),proc_stride_comp_f);
			if (proc->time_slice == 0 || proc->time_slice > rq->max_time_slice) {
				proc->time_slice = rq->max_time_slice;
			}
			proc->rq = rq;
			rq->proc_num++;
		}

	这里使用skew_heap的数据结构来维护进程队列。stride_enqueue函数调用skew_heap_insert函数将进程插入到heap里。另外，还要将proc->rq置为当前rq，同时将proc_num变量自增，表示进程队列里进程数量加1。

- stride_dequeue函数：

		static void
		stride_dequeue(struct run_queue *rq, struct proc_struct *proc) {
		     /* LAB6: 2014011433 
		      * (1) remove the proc from rq correctly
		      * NOTICE: you can use skew_heap or list. Important functions
		      *         skew_heap_remove: remove a entry from skew_heap
		      *         list_del_init: remove a entry from the  list
		      */
		     rq->lab6_run_pool = skew_heap_remove(rq->lab6_run_pool,&(proc->lab6_run_pool),proc_stride_comp_f);
		     rq->proc_num--;
		}

	和stride_enqueue函数对应，stride_dequeue函数用来把进程从heap里移除，方法依然是调用已提供的函数skew_heap_remove。另外，还要将proc_num变量自减，表示进程队列里进程数量减1。

- stride_pick_next函数：

		static struct proc_struct *
		stride_pick_next(struct run_queue *rq) {
		     /* LAB6: 2014011433 
		      * (1) get a  proc_struct pointer p  with the minimum value of stride
		             (1.1) If using skew_heap, we can use le2proc get the p from rq->lab6_run_poll
		             (1.2) If using list, we have to search list to find the p with minimum stride value
		      * (2) update p;s stride value: p->lab6_stride
		      * (3) return p
		      */
			 //added after comparing to lab6_result
		     if (rq->lab6_run_pool == NULL)
		      return NULL;
		     struct proc_struct* p = le2proc(rq->lab6_run_pool,lab6_run_pool);
		     p->lab6_stride += p->lab6_priority > 0 ? BIG_STRIDE / p->lab6_priority : BIG_STRIDE;
		     return p;
		}
	
	stride_pick_next函数用来选择下一个需要被调度的进程，并做必要的变量更新操作。首先在队列中找到下一个选择的进程。对于skew_heap而言，堆顶端的进程即为结果。然后根据算法将lab6_stride变量加上`p->lab6_priority > 0 ? BIG_STRIDE / p->lab6_priority : BIG_STRIDE`，即更新对应进程的stride值。

- stride_proc_tick函数：
		
		static void
		stride_proc_tick(struct run_queue *rq, struct proc_struct *proc) {
		     /* LAB6: 2014011433 */
		    if (proc->time_slice > 0) {
		        proc->time_slice --;
		    }
		    if (proc->time_slice == 0) {
		        proc->need_resched = 1;
		    }
		}

	stride_proc_tick函数完全类似于RR算法中的实现，作用是判断当前进程的时间片个数是否大于零，如果是则减一，否则将need_resched置1，使得在下次时钟中断时进行调度。

最后还要修改default_sched_class变量来让stride算法应用到ucore中：

	struct sched_class default_sched_class = {
	     .name = "stride_scheduler",
	     .init = stride_init,
	     .enqueue = stride_enqueue,
	     .dequeue = stride_dequeue,
	     .pick_next = stride_pick_next,
	     .proc_tick = stride_proc_tick,
	};

### 本练习与答案比较：

答：本练习与答案比较主要有以下差别：

- 答案用宏的方式实现了基于skew_heap和链表两种方式的stride算法，考虑到使用skew_heap数据结构时复杂度更低，我仅仅实现了基于skew_heap的stride算法；
- 答案在stride_pick_next函数中含有以下语句：

		if (rq->lab6_run_pool == NULL) return NULL;
	
	而我并未对rq->lab6_run_pool是否为空指针进行检查，经测试不影响通过make grade。但考虑到答案更为严谨，鲁棒性更好，我还是加上了这一步检查。

## LAB6运行结果

在LAB6目录下执行

	make grade

输出为：

![](http://i.imgur.com/W681Cay.png)

说明结果正确

## 其他
### 列出你认为本实验中重要的知识点，以及与对应的OS原理中的知识点，并简要说明你对二者的含义，关系，差异等方面的理解（也可能出现实验中的知识点没有对应的原理知识点）

答：本实验中重要的知识点我认为有如下几个：

- 操作系统中进程调度过程的掌握
- 对ucore中调度算法框架的掌握
- 对stride算法的理解和实现

### 列出你认为OS原理中很重要，但在实验中没有对应上的知识点

答：我认为有如下几点，在OS原理中很重要，但在实验中没有对应：

- 调度过程中进程的状态
- 比较调度算法的准则
- 实时调度、多处理器调度等高级话题
