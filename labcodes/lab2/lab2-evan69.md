# 操作系统lab2实验报告

###计45 侯禺凡 2014011433

## 练习0：填写已有实验

本实验依赖实验1。请把你做的实验1的代码填入本实验中代码中有“LAB1”的注释相应部分。提示：可采用diff和patch工具进行半自动的合并（merge），也可用一些图形化的比较/merge工具来手动合并，比如meld，eclipse中的diff/merge工具，understand中的diff/merge工具等。

答：

我使用了Ubuntu下的meld工具，将lab1中有"LAB1"注释的相应部分代码合并到lab2里。主要在以下三个代码文件：

- kern/debug/kdebug.c
- kern/trap/trap.c
- kern/init/init.c

## 练习1：实现 first-fit 连续物理内存分配算法（需要编程）

在实现first fit 内存分配算法的回收函数时，要考虑地址连续的空闲块之间的合并操作。提示:在建立空闲页块链表时，需要按照空闲页块起始地址来排序，形成一个有序的链表。可能会修改default\_pmm.c中的default\_init，default\_init\_memmap，default\_alloc\_pages， default\_free\_pages等相关函数。请仔细查看和理解default\_pmm.c中的注释。

答：

#### 对于default\_init函数：

依照注释里的提示，该函数作用是初始化，且不需要修改直接使用给出的缺省实现即可。

#### 对于default\_init\_memmap函数：

首先检查输入的合法性：

	assert(n > 0);

然后，从base开始连续n个页，进行初始化工作，依次设置属性，并加到free\_list上去，表示空闲的页面：

	struct Page *p = base;
    for (; p != base + n; p ++) {
        assert(PageReserved(p));
        p->flags = p->property = 0;
        SetPageProperty(p);
        set_page_ref(p, 0);
        list_add_before(&free_list, &(p->page_link));
    }

最后，设置nr_free等属性，记录总的空闲页面数目为n：

	base->property = n;
    //SetPageProperty(base);
    nr_free += n;
    //list_add(&free_list, &(base->page_link));

#### 对于default\_alloc\_pages函数：

根据课上所讲的first fit算法，空闲分区列按地址顺序排序，分配时顺次搜索一个合适的分区。我的编程思路如下（代码中也有对应语句的注释说明）：

首先，要进行输入参数的合法性检查，以及检查总空闲页面数。若数目不足n个，直接返回失败，从而节约时间：

    assert(n > 0);
    if (n > nr_free) {
        return NULL;
    } //there are no more than n pages -> fail

根据注释里面的提示，用一个while循环，依地址顺序遍历检查所有页面，查看是否有连续n个空闲页面。若有，在if语句内部对它进行处理：

    struct Page *ret = NULL;
    list_entry_t *le = &free_list;
    while((le=list_next(le)) != &free_list) 
    {
        //check every pages
        struct Page *p = le2page(le, page_link);
        //cprintf("0x%08x \n", p);
        if(p->property >= n)
        // find a page with more than n free pages after it
        {
            //.......
        }
    }
    return ret;

在if语句内部，即找到了连续n个以上的连续空闲页面，则首先要对这n个页面要依次设置属性值，并将其从链表中去除，表明这些页面已不再空闲：
	
    int i = 0;
    while(i < n)
    {
        //init every page after finding more than n pages
        struct Page *pp = le2page(le, page_link);
        SetPageReserved(pp);
        ClearPageProperty(pp);
        //set and clear
        list_del(le);
        //delete from list
        le = list_next(le);
        //move to next page
        i++;
    }

最后，还需要更新各个变量和属性值，包括连续空闲页的头页面的空闲页面数(le\_page->property)和总的空闲页面数(nr\_free)等：

    if(p->property > n)
    {
        struct Page* le_page = (le2page(le,page_link));
        le_page->property = p->property - n;
        // set property
    }
    nr_free -= n;
    // update nr_free
    ret = p;
    // set return variable
    break;

最后函数返回页面p作为分配页面的结果：

    return ret;

#### 对于default\_free\_pages函数：

在释放页面时，根据课上所讲，需要检查是否可以与邻近的空闲分区合并。我的编程思路如下（代码中也有对应语句的注释说明）：

首先，要进行输入参数的合法性检查，包括释放的页面数以及保留位是否正确：

    assert(n > 0);
    assert(PageReserved(base));

为了维护空闲页面链表的地址的从小到大特性，需要寻找到链表中连续的两项，使得要释放的页面介乎这两项之间，以便之后进行链表项的插入操作。我使用while循环来寻找，并用变量记录下这两项：

    list_entry_t *le = &free_list;
    struct Page * p_last = NULL,* p_next = NULL;
    //p_next : page with higher address than base
    //p_last : page with  lower address than base
    while((le=list_next(le)) != &free_list) 
    {
        p_next = le2page(le, page_link);
        if(p_next > base)
        {
            p_last = le2page(list_prev(le),page_link);
            break;
        }
    }// use while loop to find p_next and p_last

找到了要插入的位置之后，需要将要释放的n项依次插入到链表中去：

    struct Page* p = p_next;
    for(p = base;p < base+n;p++)
    {
        list_add_before(le, &(p->page_link));
    }// add n pages into the list (between p_next and p_last)

并设置页面的属性：

    base->flags = 0;
    set_page_ref(base, 0);
    ClearPageProperty(base);
    SetPageProperty(base);
    base->property = n;
    // set attributes

接下来需要对相邻页面进行可能的合并操作。我选择首先合并所释放的空间地址之前的若干连续空闲页。在合并的时候还要遍历找到这些空闲页的开头，进而设置property等属性：

    p = base;
    if(base - 1 == p_last)
    //merge base and p_last
    {
        base->property = 0;
        // base changes to a non-head page
        le = &(p_last->page_link);
        p = p_last;
        while(le!=&free_list)
        {
            //find the head of p_last 
            if(p->property > 0)
            {
                p->property += n;
                // update property of the head
                break;
            }
            le = list_prev(le);
            p = le2page(le,page_link);
        }
    }

之后合并所释放的空间地址之后的若干连续空闲页：

    //p is the head lower than p_next at this time

    if(base + n == p_next)
    //merge p and p_next
    {
        p->property += p_next->property;
        p_next->property = 0;
    }

最后更新总空闲页面数：

    nr_free += n;
    // update nr_free

### Q:你的first fit算法是否有进一步的改进空间?

答：还有可以改进空间。在与所释放的空间地址之前的若干连续空闲页合并的时候，可以省去遍历过程。实现这一点需要在之前寻找插入位置的时候增加一个变量struct Page * last\_head记录空闲页开头，以下是一种可能的实现：
	
	while((le=list_next(le)) != &free_list) 
    {
        p_next = le2page(le, page_link);

		if(p_next + p_next->property == base)
		{
			last_head = p_next;
		}// code added for recording head of continuous free pages 

        if(p_next > base)
        {
            p_last = le2page(list_prev(le),page_link);
            break;
        }
    }

这样，在判断需要与所释放的空间地址之前的若干连续空闲页进行合并后，对last\_head的property属性进行更新即可，无需再用循环寻找。

### 本练习与答案比较：

答：我的实现思路和给出的答案主要有以下几个不同：

- 遍历链表实现的方式有所不同
- 在函数default\_alloc\_pages()里，答案中有如下片段
		
		ClearPageProperty(p);
		SetPageReserved(p);
		
	而我没有这两个语句。经我思考，我认为这一区别对于功能的实现没有不同。
- 在函数default\_free\_pages中，我是先与地址更低的连续空闲页合并，再和地址高的合并，而答案的实现恰恰相反

## 练习2：实现寻找虚拟地址对应的页表项（需要编程）

通过设置页表和对应的页表项，可建立虚拟内存地址和物理内存地址的对应关系。其中的get_pte函数是设置页表项环节中的一个重要步骤。此函数找到一个虚地址对应的二级页表项的内核虚地址，如果此二级页表项不存在，则分配一个包含此项的二级页表。本练习需要补全get\_pte函数 in kern/mm/pmm.c，实现其功能。请仔细查看和理解get\_pte函数中的注释。get\_pte函数的调用关系图如下所示：（图略）

答：由于二级页表的结构，为了获取页表项我们需要先利用线性地址的高10位计算PDE中偏移量，结合PDE基址从PDE中获取PTE基址，然后利用线性地址中间10位计算PTE中偏移量，加上基址来计算二级页表项的内核虚地址。我的编程思路如下（代码中也有对应语句的注释说明）：

首先用线性地址高10位计算PDE中的偏移量：

    uint32_t pde_index = PDX(la);
    //get offset

加上基址，得到PDE中的某一项，其中一部分为PTE基址：

    pde_t* pde_entry_addr = pgdir + pde_index;
    //get pde entry by base addr and offset
    pde_t pde_entry_data = *pde_entry_addr;

判断是否需要建立二级页表项。若需要，尝试建立，并判断是否创建成功。else语句内处理创建成功二级页表项的情况：

    if(!(pde_entry_data & PTE_P))
    //if not present
    {
        struct Page *page = alloc_page();
        //try to alloc page for PTE
        if (!create || page == NULL) 
        //fail or forbidden by the parameter
        {
            return NULL;
        }
        else
        {
            //......
        }
    }

else语句内部，创建页表项后，需要设置该页表项的存在位、读写位和访问位，并更新页的引用数：

	uintptr_t pa = page2pa(page);
	//get pa of the page
	*pde_entry_addr = pa | PTE_U | PTE_W | PTE_P;
	//set bit of PDE
	set_page_ref(page, 1);
	//set page ref

获得PTE基址后，利用线性地址中间10位计算PTE中偏移量，利用基址和偏移访问PTE中对应项，进而获得二级页表项的内核虚地址：

    pde_entry_data = *pde_entry_addr;
    //update pde entry data
    pte_t* pte_base = PDE_ADDR(pde_entry_data);
    //get base of pte from pde entry
    uint32_t pte_index = PTX(la);
    //get pte offset
    return (pte_t *)KADDR(pte_base) + pte_index;

#### 问题1：请描述页目录项（Pag Director Entry）和页表（Page Table Entry）中每个组成部分的含义和以及对ucore而言的潜在用处。

答：

#### 问题2：如果ucore执行过程中访问内存，出现了页访问异常，请问硬件要做哪些事情？

答：

## 练习3：释放某虚地址所在的页并取消对应二级页表项的映射（需要编程）

当释放一个包含某虚地址的物理内存页时，需要让对应此物理内存页的管理数据结构Page做相关的清除处理，使得此物理内存页成为空闲；另外还需把表示虚地址与物理地址对应关系的二级页表项清除。请仔细查看和理解page\_remove\_pte函数中的注释。为此，需要补全在 kern/mm/pmm.c中的page\_remove\_pte函数。page\_remove\_pte函数的调用关系图如下所示：（图略）

答：我的编程思路如下（代码中也有对应语句的注释说明）：

首先判断要取消对应的二级页表项是否存在。不存在则直接返回：

    if(!*ptep & PTE_P)
    //check if this page table entry is present
        return;

下面，找到对应于ptep的page，若其引用次数为1，则直接释放该页面。还要释放二级页表的表项：

    struct Page *page = pte2page(*ptep);
    //find corresponding page to pte
    uint32_t page_ref = page_ref_dec(page);
    //decrease page reference
    if (page_ref == 0)
    //free this page when page reference reachs 0
    {
        free_pages(page,1);
    }
	*ptep = NULL;

最后，清除TLB：

    tlb_invalidate(pgdir,la);
    //clear tlb

#### 问题1：数据结构Page的全局变量（其实是一个数组）的每一项与页表中的页目录项和页表项有无对应关系？如果有，其对应关系是啥？

#### 问题2：如果希望虚拟地址与物理地址相等，则需要如何修改lab2，完成此事？ 鼓励通过编程来具体完成这个问题

## LAB2运行结果

在LAB2目录下执行

	make grade

截图如下：

![](http://i.imgur.com/KxCLpvx.png)

说明结果正确

## 其他
### 列出你认为本实验中重要的知识点，以及与对应的OS原理中的知识点，并简要说明你对二者的含义，关系，差异等方面的理解（也可能出现实验中的知识点没有对应的原理知识点）

答：

### 列出你认为OS原理中很重要，但在实验中没有对应上的知识点

答：