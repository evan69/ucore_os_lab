# 操作系统lab3实验报告

### 计45 侯禺凡 2014011433

## 练习0：填写已有实验

本实验依赖实验1/2。请把你做的实验1/2的代码填入本实验中代码中有“LAB1”,“LAB2”的注释相应部分。

答：我使用了Ubuntu下的meld工具，将lab2中有"LAB1"及"LAB2"注释的相应部分代码合并到lab3里。

## 练习1：给未被映射的地址映射上物理页（需要编程）

完成do\_pgfault（mm/vmm.c）函数，给未被映射的地址映射上物理页。设置访问权限 的时候需要参考页面所在 VMA的权限，同时需要注意映射物理页时需要操作内存控制 结构所指定的页表，而不是内核的页表。注意：在LAB2 EXERCISE 1处填写代码。执行

	make qemu

后，如果通过check\_pgfault函数的测试后，会有“check\_pgfault() succeeded!”的输出，表示练习1基本正确。

答：我在kern/mm/vmm.c中，给未被映射的地址映射物理页，添加如下代码：

	ptep = get_pte(mm->pgdir,addr,1);
    if(ptep ==  NULL)
    {
        cprintf("create pte failed");
        goto failed;
    }

    if(*ptep == 0)
    {
        struct Page * alloc_result = pgdir_alloc_page(mm->pgdir,addr,perm);
        if(alloc_result == NULL)
        {
            goto failed;
        }
    }
    else
    {
        //......
    }

整个过程大致分为以下几步

- 首先调用get\_pte函数，从虚拟地址获取PTE；
- 检查PTE，若为NULL，则跳转到fail；
- 检查PTE，若其保存的是空指针，则调用pgdir\_alloc\_page分配物理页面；
- 若分配失败，则跳转到fail

### 问题一：请描述页目录项（Pag Director Entry）和页表（Page Table Entry）中组成部分对ucore实现页替换算法的潜在用处。

答：PTE和PDE主要包含一些标志位和下一页表（或物理页）的基地址，它们可以提供虚拟地址到物理地址的转换，并可被用于判断相应的物理地址是否合法、物理页是否在内存中以及相应的访问权限等。

### 问题二：如果ucore的缺页服务例程在执行过程中访问内存，出现了页访问异常，请问硬件要做哪些事情？

答：硬件产生缺页异常嵌套，会再次调用缺页服务例程进行相应处理。

### 本练习与答案比较：

答：我的实现思路和给出的答案思路基本一样。

## 练习2：补充完成基于FIFO的页面替换算法（需要编程）

首先，根据代码注释里的提示，在do\_pgfault里增加代码：

    if(swap_init_ok)
    //can swap
    {
        struct Page* page = NULL;
        if(swap_in(mm,addr,&page))
        {
            goto failed;
        }//load data from disk to a page with phy addr
        if(page_insert(mm->pgdir,page,addr,perm))
        {
            goto failed;
        }//According to the mm, addr AND page, setup the map of phy addr <---> logical addr
        if(swap_map_swappable(mm,addr,page,1))
        {
            goto failed;
        }//make the page swappable.
        page->pra_vaddr = addr;
        //why this line necessary?
    }
    else
    {
        cprintf("no swap_init_ok but ptep is %x, failed\n",*ptep);
        goto failed;
    }

在这段代码里，主要完成以下工作：

- 检查是否能够和外存进行交换；
- 根据给出的物理地址，将页面和外存进行交换，并检查返回值；
- 建立物理地址和逻辑地址之间的映射关系，并检查返回值；
- 设置页面可以交换，并检查返回值；
- 设置page的pra\_vaddr属性

然后，在\_fifo\_map\_swappable函数里增加以下代码：

    list_add_before(head,entry);
    //link the most recent arrival page at the back of the pra_list_head qeueue.

主要作用是将entry加入到管理程序内存空间的环形链表中。最后，在\_fifo\_swap\_out\_victim函数里增加以下代码：
	
	list_entry_t *entry = list_next(head);
	//choose the victim
	list_del(entry);
	//unlink the earliest arrival page in front of pra_list_head qeueue
	*ptr_page = le2page(entry,pra_page_link);
	//set the addr of addr of this page to ptr_page(but not understand)

这主要是根据FIFO算法的思路，选择链表一端最先被放入的页面进行换出，删除对应的链表项，并设置ptr\_page变量的值。

### 问题1：如果要在ucore上实现"extended clock页替换算法"请给你的设计方案，现有的swap_manager框架是否足以支持在ucore中实现此算法？如果是，请给你的设计方案。如果不是，请给出你的新的扩展和基此扩展的设计方案。并需要回答如下问题

- 需要被换出的页的特征是什么？
- 在ucore中如何判断具有这样特征的页？
- 何时进行换入和换出操作？

答：现有的框架不足以支持extended clock替换算法。原因如下：现有的框架中没有动态修改访问页面的函数。需要在现有的框架中加入访问某页时的函数，动态地修改页面的访问位和修改位。

- extended clock算法的执行过程为：将被遍历的节点的访问位或修改位清零，直到找到全为0的页替换。被换出的页在相应的时间间隔中，没有被访问过或修改过。如果都被访问或修改过，则extended clock算法会遍历一遍后找到最开始的页
- 在ucore中，通过从对应位置遍历，判断相应页的标志位来找到这样的页
- 在缺页异常发生时进行换入换出

### 本练习与答案比较：

答：答案中是将新加入的entry加到环形链表头的后面，然后每次从头的前面取；而我的实现是将新加入的entry加到环形链表头的前面，然后每次从头的后面取。经过分析比较，这两种实现对于FIFO算法的实现是相同的效果。

## LAB3运行结果

在LAB3目录下执行

	make grade

截图如下：

![](http://i.imgur.com/xCSexOV.png)

说明结果正确

## 其他
### 列出你认为本实验中重要的知识点，以及与对应的OS原理中的知识点，并简要说明你对二者的含义，关系，差异等方面的理解（也可能出现实验中的知识点没有对应的原理知识点）

答：本实验中重要的知识点我认为有如下几个：

- 页访问异常的几种情况；
- 页访问异常的处理方式；
- FIFO页替换算法的实现

### 列出你认为OS原理中很重要，但在实验中没有对应上的知识点

答：我认为有如下几点，在OS原理中很重要，但在实验中没有对应：

- 除FIFO算法外的其他页置换算法；
- 对Belady现象的探讨和实验
- 虚拟存储概念
