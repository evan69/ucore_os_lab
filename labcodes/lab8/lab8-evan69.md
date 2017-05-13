# 操作系统lab8实验报告

### 计45 侯禺凡 2014011433

## 练习0：填写已有实验

本实验依赖实验1/2/3/4/5/6/7。请把你做的实验1/2/3/4/5/6/7的代码填入本实验中代码中有“LAB1”/“LAB2”/“LAB3”/“LAB4”/“LAB5”/“LAB6” /“LAB7”的注释相应部分。并确保编译通过。注意：为了能够正确执行lab8的测试应用程序，可能需对已完成的实验1/2/3/4/5/6/7的代码进行进一步改进。

答：我使用了Ubuntu下的meld工具，将lab5中有"LAB1","LAB2","LAB3","LAB4","LAB5","LAB6"及"LAB7"注释的相应部分代码合并到ab8里。由于需要支持lab8，我在lab8中注释提示的帮助下，对在前面7个实验所写的代码进行了修改，说明如下。

修改kern/process/proc.c的alloc_proc函数，增加代码如下：

	//LAB8:EXERCISE2 2014011433 HINT:need add some code to init fs in proc_struct, ...
	proc->filesp = NULL;

作用是为进程附带的文件系统指针filesp进行初始化。修改同一文件的do_fork函数，增加代码如下：

	int copy_file_ret = copy_files(clone_flags, proc);
	if(copy_file_ret != 0)
	  goto bad_fork_cleanup_fs;
	//added step for lab8 : copy file

作用是调用提供的copy_files函数进行文件的拷贝。

### 本练习与答案比较：

答：这部分主要是修改lab1-7以使lab8正确，代码较为简单，与答案实现相同。

## 练习1：完成读文件操作的实现（需要编码）

首先了解打开文件的处理流程，然后参考本实验后续的文件读写操作的过程分析，编写在sfs_inode.c中sfs_io_nolock读文件中数据的实现代码。

请在实验报告中给出设计实现”UNIX的PIPE机制“的概要设方案，鼓励给出详细设计方案

答：在sfs_inode.c中sfs_io_nolock函数中增加代码：

    blkoff = offset % SFS_BLKSIZE;

    if (blkoff != 0)
    {
        size = (nblks != 0) ? (SFS_BLKSIZE - blkoff) : (endpos - offset);
        if ((ret = sfs_bmap_load_nolock(sfs, sin, blkno, &ino)) != 0)
        {
            goto out;
        }
        if ((ret = sfs_buf_op(sfs, buf, size, ino, blkoff)) != 0)
        {
            goto out;
        }
        alen += size;
        if (nblks == 0)
        {
            goto out;
        }
        buf += size;
        blkno++;
        nblks--;
    }

    blkoff = 1;
    size = SFS_BLKSIZE;
    while (nblks > 0)
    {
        if ((ret = sfs_bmap_load_nolock(sfs, sin, blkno, &ino)) != 0)
        {
            goto out;
        }
        if ((ret = sfs_block_op(sfs, buf, ino, blkoff)) != 0)
        {
            goto out;
        }
        alen += size;
        buf += size;
        blkno ++;
        nblks --;
    }

    blkoff = 0;
    size = endpos % SFS_BLKSIZE;

    if (size > 0)
    {
        if ((ret = sfs_bmap_load_nolock(sfs, sin, blkno, &ino)) != 0)
        {
            goto out;
        }
        if ((ret = sfs_buf_op(sfs, buf, size, ino, blkoff)) != 0)
        {
            goto out;
        }
        alen += size;
    }

代码的执行步骤为：

1. 首先处理起始地址所在的块。先计算块内偏移blkoff，据此判断起始地址的offset是否对齐。如果不对齐，则计算大小size，调用sfs_bmap_load_nolock获取ino，再调用sfs_buf_op完成读写操作。完成后，需要更新已经完成的大小alen，缓冲区指针buf，以及blkno和nblks这几个变量；
2. 处理中间完整的块。遍历每一个块，对这个块调用sfs_bmap_load_nolock获取ino，再调用sfs_buf_op完成实际的读写操作。最后效仿步骤一更新各个变量；
3. 处理最后的没有对齐的块。这时，blkoff应当设为0，再次计算size。之后调用sfs_bmap_load_nolock获取ino，再调用sfs_buf_op完成读写操作，最后更新alen变量。

### 问题：给出设计实现“UNIX的PIPE机制”的概要设方案，鼓励给出详细设计方案

答：联想到已经学习过的“读者-写者问题”，我认为可以在PIPE中设置一个双向链表维护缓冲区的数据。发送方向缓冲区里面写，写的数据插入链表尾部；接收方从缓冲区里读，读出的数据从链表头部删除。需要注意的是，缓冲区满时，发送方需要被阻塞；当缓冲区空时，接收方需要被阻塞。链表的读写以及空和满需要同步互斥的支持，这可以使用lab7中用到的信号量或者管程来实现。

### 本练习与答案比较：

答：大体实现思路和答案相同，有较小区别如下：

- 答案是

		while (nblks != 0) {
	        //...
	    }

	而我的实现是

		while (nblks > 0)
	    {
	        //...
	    }

	执行效果一样

- 答案是

		if ((size = endpos % SFS_BLKSIZE) != 0) {
	        //...
	    }

	而我的实现是

		if (size > 0)
	    {
	        //...
	    }

	执行效果一样


## 练习2: 完成基于文件系统的执行程序机制的实现（需要编码）

改写proc.c中的load_icode函数和其他相关函数，实现基于文件系统的执行程序机制。执行：make qemu。如果能看看到sh用户程序的执行界面，则基本成功了。如果在sh用户界面上可以执行”ls”,”hello”等其他放置在sfs文件系统中的其他执行程序，则可以认为本实验基本成功。

请在实验报告中给出设计实现基于”UNIX的硬链接和软链接机制“的概要设方案，鼓励给出详细设计方案

答：在proc.c中的load_icode函数中增加代码，思路以代码中注释的形式体现：

	assert(argc >= 0 && argc <= EXEC_MAX_ARG_NUM);

    //新建mm
    if (current->mm != NULL)
    {
        panic("load_icode: current->mm must be empty.\n");
    }

    int ret = -E_NO_MEM;
    struct mm_struct *mm;
    if ((mm = mm_create()) == NULL)
    {
        goto bad_mm;
    }

    //建立页目录表
    if (setup_pgdir(mm) != 0)
    {
        goto bad_pgdir_cleanup_mm;
    }

    struct Page *page;

    //将文件从硬盘加载到内存
    struct elfhdr __elf, *elf = &__elf;

    //读elf文件头
    if ((ret = load_icode_read(fd, elf, sizeof(struct elfhdr), 0)) != 0)
    {
        goto bad_elf_cleanup_pgdir;
    }

    if (elf->e_magic != ELF_MAGIC)
    {
        ret = -E_INVAL_ELF;
        goto bad_elf_cleanup_pgdir;
    }

    //读elf文件头中的所有程序头
    struct proghdr __ph, *ph = &__ph;
    uint32_t vm_flags, perm, phnum;
    for (phnum = 0; phnum < elf->e_phnum; phnum ++)
    {
        off_t phoff = elf->e_phoff + sizeof(struct proghdr) * phnum;//当前读到的程序头在文件中的偏移量

        if ((ret = load_icode_read(fd, ph, sizeof(struct proghdr), phoff)) != 0)
        {
            //读程序头
            goto bad_cleanup_mmap;
        }
        if (ph->p_type != ELF_PT_LOAD)
        {
            //判断该程序头描述的段是否可装载
            continue ;
        }
        if (ph->p_filesz > ph->p_memsz)
        {
            ret = -E_INVAL_ELF;
            goto bad_cleanup_mmap;
        }
        if (ph->p_filesz == 0)
        {
            //若该程序头描述的段的大小为0，继续读下一个程序头
            continue ;
        }

        //建立程序头的地址映射
        vm_flags = 0, perm = PTE_U;
        //设置标志位
        if (ph->p_flags & ELF_PF_X) vm_flags |= VM_EXEC;
        if (ph->p_flags & ELF_PF_W) vm_flags |= VM_WRITE;
        if (ph->p_flags & ELF_PF_R) vm_flags |= VM_READ;
        if (vm_flags & VM_WRITE) perm |= PTE_W;
        if ((ret = mm_map(mm, ph->p_va, ph->p_memsz, vm_flags, NULL)) != 0)
        {
            goto bad_cleanup_mmap;
        }
        off_t offset = ph->p_offset;
        size_t off, size;
        uintptr_t start = ph->p_va, end, la = ROUNDDOWN(start, PGSIZE);
        //la为start的PGSIZE对齐处

        //复制程序头对应的段
        ret = -E_NO_MEM;

        //复制p_filesz（text/data）
        end = ph->p_va + ph->p_filesz;
        while (start < end)
        {
            if ((page = pgdir_alloc_page(mm->pgdir, la, perm)) == NULL)
            {
                ret = -E_NO_MEM;
                goto bad_cleanup_mmap;
            }
            off = start - la, size = PGSIZE - off, la += PGSIZE;
            if (end < la)
            {
                size -= la - end;
            }
            if ((ret = load_icode_read(fd, page2kva(page) + off, size, offset)) != 0)
            {
                goto bad_cleanup_mmap;
            }
            start += size, offset += size;
        }

        //p_memsz（bss）开始时未对齐部分初始化
        end = ph->p_va + ph->p_memsz;
        if (start < la)
        {
            if (start == end)
            {
                continue ;
            }
            off = start + PGSIZE - la, size = PGSIZE - off;
            if (end < la)
            {
                size -= la - end;
            }
            memset(page2kva(page) + off, 0, size);
            start += size;
            assert((end < la && start == end) || (end >= la && start == la));
        }

        //p_memsz（bss）对齐部分初始化
        while (start < end)
        {
            if ((page = pgdir_alloc_page(mm->pgdir, la, perm)) == NULL)
            {
                ret = -E_NO_MEM;
                goto bad_cleanup_mmap;
            }
            off = start - la, size = PGSIZE - off, la += PGSIZE;
            if (end < la)
            {
                size -= la - end;
            }
            memset(page2kva(page) + off, 0, size);
            start += size;
        }
    }

    sysfile_close(fd);
    //关闭文件


    //建立用户栈的内存映射
    vm_flags = VM_READ | VM_WRITE | VM_STACK;
    if ((ret = mm_map(mm, USTACKTOP - USTACKSIZE, USTACKSIZE, vm_flags, NULL)) != 0)
    {
        goto bad_cleanup_mmap;
    }
    assert(pgdir_alloc_page(mm->pgdir, USTACKTOP-PGSIZE , PTE_USER) != NULL);
    assert(pgdir_alloc_page(mm->pgdir, USTACKTOP-2*PGSIZE , PTE_USER) != NULL);
    assert(pgdir_alloc_page(mm->pgdir, USTACKTOP-3*PGSIZE , PTE_USER) != NULL);
    assert(pgdir_alloc_page(mm->pgdir, USTACKTOP-4*PGSIZE , PTE_USER) != NULL);
    
    //设置当前进程的mm、cr3（页目录表基址）
    mm_count_inc(mm);
    current->mm = mm;
    current->cr3 = PADDR(mm->pgdir);
    lcr3(PADDR(mm->pgdir));

    uint32_t argv_size=0, i;
    //计算所有参数的长度和
    for (i = 0; i < argc; i ++)
    {
        argv_size += strnlen(kargv[i],EXEC_MAX_ARG_LEN + 1)+1;
    }

    uintptr_t stacktop = USTACKTOP - (argv_size/sizeof(long)+1)*sizeof(long);
    char** uargv=(char **)(stacktop  - argc * sizeof(char *));
    
    argv_size = 0;
    //将传进的参数依次放入用户栈
    for (i = 0; i < argc; i ++) 
    {
        uargv[i] = strcpy((char *)(stacktop + argv_size ), kargv[i]);
        argv_size +=  strnlen(kargv[i],EXEC_MAX_ARG_LEN + 1)+1;
    }
    
    stacktop = (uintptr_t)uargv - sizeof(int);
    *(int *)stacktop = argc;
    
    //设置中断帧，这样从中断返回时即可开始执行从elf文件中载入的内容
    struct trapframe *tf = current->tf;
    memset(tf, 0, sizeof(struct trapframe));
    tf->tf_cs = USER_CS;
    tf->tf_ds = tf->tf_es = tf->tf_ss = USER_DS;
    tf->tf_esp = stacktop;
    tf->tf_eip = elf->e_entry;//第一条指令
    tf->tf_eflags = FL_IF;
    ret = 0;

    //错误处理，恢复之前的状态
	out:
	    return ret;
	bad_cleanup_mmap:
	    exit_mmap(mm);
	bad_elf_cleanup_pgdir:
	    put_pgdir(mm);
	bad_pgdir_cleanup_mm:
	    mm_destroy(mm);
	bad_mm:
	    goto out;

### 问题：给出设计实现基于”UNIX的硬链接和软链接机制“的概要设方案，鼓励给出详细设计方案

答：在文件控制块中增加标识符标识是普通文件/硬链接/软链接，对于这三种情况：

- 普通文件：正常读取文件
- 硬链接：多个硬链接的文件共用同一个inode。访问硬链接文件时，可通过指针获取这个inode，进而对这个文件进行访问和读写。另外，在inode中使用ref_count变量记录文件的引用计数。当创建文件的硬链接时，文件对应的inode的引用计数增加1。当删除文件时，将inode的引用计数减1。若引用计数为0，则删除文件与其对应的inode。
- 软链接：文件控制块中存有真实文件的逻辑名称等信息

### 本练习与答案比较：

答：本练习实现参考了已给出的提示性注释，与答案的实现思路基本相同。

## LAB8运行结果

在LAB8目录下执行

	make grade

输出为：

![](http://i.imgur.com/Yr0DVE3.png)

说明结果正确。另外，执行

	make qemu

待哲学家就餐问题（lab7）输出完成之后，按一下回车按键，可以看见shell中显示了$符号。此时，ucore可以正确执行”ls”,”hello”等放置在sfs文件系统中的执行程序，测试结果见下图：

![](http://i.imgur.com/VV7mOUm.png)

说明实验基本完成。

## 其他
### 列出你认为本实验中重要的知识点，以及与对应的OS原理中的知识点，并简要说明你对二者的含义，关系，差异等方面的理解（也可能出现实验中的知识点没有对应的原理知识点）

答：本实验中重要的知识点我认为有如下几个：

- 文件系统整体架构                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                        
- VFS的设计与实现
- SFS的设计与实现

### 列出你认为OS原理中很重要，但在实验中没有对应上的知识点

答：我认为有如下几点，在OS原理中很重要，但在实验中没有对应：

- 文件系统中的文件分配方法
- 文件系统中的空闲空间组织方式
- 多磁盘管理：RAID
