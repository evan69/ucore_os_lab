## 操作系统lab1实验报告

###计45 侯禺凡 2014011433

### 练习1

#### 问题1：操作系统镜像文件ucore.img是如何一步一步生成的？(需要比较详细地解释Makefile中每一条相关命令和命令参数的含义，以及说明命令导致的结果)
答：

- 这类命令
<pre>
gcc -Iboot/ -fno-builtin -Wall -ggdb -m32 -gstabs -nostdinc  -fno-stack-protector -Ilibs/ -Os -nostdinc -c boot/bootasm.S -o obj/boot/bootasm.o
</pre>
将调用gcc工具，把C代码编译为.o的目标文件，用到的参数说明如下：

	-ggdb  生成可供gdb使用的调试信息。这样才能用qemu+gdb来调试bootloader or ucore。
	-m32  生成适用于32位环境的代码。我们用的模拟硬件是32bit的80386，所以ucore也要是32位的软件。
	-gstabs  生成stabs格式的调试信息。这样要ucore的monitor可以显示出便于开发者阅读的函数调用栈信息
	-nostdinc  不使用标准库。标准库是给应用程序用的，我们是编译ucore内核，OS内核是提供服务的，所以所有的服务要自给自足。
	-fno-stack-protector  不生成用于检测缓冲区溢出的代码。这是for 应用程序的，我们是编译内核，ucore内核好像还用不到此功能。
	-Os  为减小代码大小而进行优化。根据硬件spec，主引导扇区只有512字节，我们写的简单bootloader的最终大小不能大于510字节。
	-I<dir>  添加搜索头文件的路径
	-fno-builtin  除非用__builtin_前缀，否则不进行builtin函数的优化

- 命令
<pre>
ld -m elf_i386 -nostdlib -N -e start -Ttext 0x7C00 obj/boot/bootasm.o obj/boot/bootmain.o -o obj/bootblock.o
</pre>
将目标文件转化为执行程序（如bootloader执行程序），用到的参数说明如下：

	-m <emulation>  模拟为i386上的连接器
	-nostdlib  不使用标准库
	-N  设置代码段和数据段均可读写
	-e <entry>  指定入口
	-Ttext  制定代码段开始位置

- 命令
<pre>
dd if=/dev/zero of=bin/ucore.img count=10000
</pre>
生成一个有10000个块的文件，每个块默认512字节，用0填充

#### 问题2：一个被系统认为是符合规范的硬盘主引导扇区的特征是什么？
答：在tools/sign.c中有如下代码片段

	buf[510] = 0x55;
	buf[511] = 0xAA;
	FILE *ofp = fopen(argv[2], "wb+");
	size = fwrite(buf, 1, 512, ofp);

由此可见，一个符合规范的硬盘主引导扇区的应具备以下特征：

- 磁盘主引导扇区只有512字节 
- 磁盘最后两个字节为0x55AA 
- 由不超过466字节的启动代码和不超过64字节的硬盘分区表加上两个字节的结束符组成

### 练习2：为了熟悉使用qemu和gdb进行的调试工作，我们进行如下的小练习：

#### 1.从CPU加电后执行的第一条指令开始，单步跟踪BIOS的执行。
修改 lab1/tools/gdbinit

	set architecture i8086
	target remote :1234

然后于lab1目录下执行

	make debug

gdb启动后，查看pc寄存器和后2条指令的情况：

	(gdb) i r pc
	pc             0xfff0   0xfff0
	(gdb) x /2i 0xffff0 
	0xffff0:     ljmp   $0xf000,$0xe05b
	0xffff5:     xor    %dh,0x322f

![](http://i.imgur.com/TXLM5SI.png)
	
说明正确停在了第一条指令处

	(gdb) x /5i 0xfe05b
	0xfe05b:     cmpl   $0x0,%cs:0x6574
	0xfe062:     jne    0xfd2b6
	0xfe066:     xor    %ax,%ax
	0xfe068:     mov    %ax,%ss
	0xfe06a:     mov    $0x7000,%esp

![](http://i.imgur.com/c9W1xOL.png)

使用si可以单步：

	(gdb) si
	0x0000e05b in ?? ()
	(gdb) si
	0x0000e062 in ?? ()

![](http://i.imgur.com/m0aZ1ng.png)

#### 2.在初始化位置0x7c00设置实地址断点,测试断点正常。
在0x7c00设置断点

	(gdb) b *0x7c00
	Breakpoint 1 at 0x7c00
	(gdb) continue
	Continuing.
	
	Breakpoint 1, 0x00007c00 in ?? ()

![](http://i.imgur.com/8X3sIl8.png)

测试断点

	(gdb) i r
	eax            0xaa55   43605
	ecx            0x0      0
	edx            0x80     128
	ebx            0x0      0
	esp            0x6f2c   0x6f2c
	ebp            0x0      0x0
	esi            0x0      0
	edi            0x0      0
	eip            0x7c00   0x7c00
	eflags         0x202    [ IF ]
	cs             0x0      0
	ss             0x0      0
	ds             0x0      0
	es             0x0      0
	fs             0x0      0
	gs             0x0      0
	(gdb) x /2i $pc
	=> 0x7c00:      cli    
	   0x7c01:      cld    

![](http://i.imgur.com/CtxaOl4.png)

#### 3.从0x7c00开始跟踪代码运行,将单步跟踪反汇编得到的代码与bootasm.S和 bootblock.asm进行比较。

使用命令x /10i $pc以便比较，执行结果如下：

![](http://i.imgur.com/bZIsqB3.png)

![](http://i.imgur.com/rmPb8KR.png)

对比bootasm.S文件里的对应代码如下：

		cli                                             # Disable interrupts
	    cld                                             # String operations increment
	
	    # Set up the important data segment registers (DS, ES, SS).
	    xorw %ax, %ax                                   # Segment number zero
	    movw %ax, %ds                                   # -> Data Segment
	    movw %ax, %es                                   # -> Extra Segment
	    movw %ax, %ss                                   # -> Stack Segment
	
	    # Enable A20:
	    #  For backwards compatibility with the earliest PCs, physical
	    #  address line 20 is tied low, so that addresses higher than
	    #  1MB wrap around to zero by default. This code undoes this.
	seta20.1:
	    inb $0x64, %al                                  # Wait for not busy(8042 input buffer empty).
	    testb $0x2, %al
	    jnz seta20.1
	
	    movb $0xd1, %al                                 # 0xd1 -> port 0x64

可以发现运行结果和代码能一一对应上。单步之后再次进行反汇编，结果依然一致。

#### 4.自己找一个bootloader或内核中的代码位置，设置断点并进行测试。

我使用GDB在内核中的函数gdt_init设置了断点，结果如下：

	(gdb) b gdt_init
	Breakpoint 2 at 0x1025cc: file kern/mm/pmm.c, line 80.
	(gdb) continue
	Continuing.
	
	Breakpoint 2, gdt_init () at kern/mm/pmm.c:80
	(gdb) 

![](http://i.imgur.com/D578Ko7.png)

测试断点正常。

### 练习3：分析bootloader进入保护模式的过程。
1.关闭中断，初始化寄存器，代码片段如下

    cli                                             # Disable interrupts
    cld                                             # String operations increment

    # Set up the important data segment registers (DS, ES, SS).
    xorw %ax, %ax                                   # Segment number zero
    movw %ax, %ds                                   # -> Data Segment
    movw %ax, %es                                   # -> Extra Segment
    movw %ax, %ss                                   # -> Stack Segment

2.开启A20。出于保护模式时，为了正常访问全部内存空间，必须打开A20。对应的代码片段如下：

	# Enable A20:
	#  For backwards compatibility with the earliest PCs, physical
	#  address line 20 is tied low, so that addresses higher than
	#  1MB wrap around to zero by default. This code undoes this.
	seta20.1:
		inb $0x64, %al                       # Wait for not busy(8042 input buffer empty).
		testb $0x2, %al
		jnz seta20.1
		
		movb $0xd1, %al                      # 0xd1 -> port 0x64
		outb %al, $0x64                      # 0xd1 means: write data to 8042's P2 port
	
	seta20.2:
		inb $0x64, %al                       # Wait for not busy(8042 input buffer empty).
		testb $0x2, %al
		jnz seta20.2

3.加载全局描述符表GDT：

	lgdt gdtdesc

4.置cr0寄存器的最低位为1：

    movl %cr0, %eax
    orl $CR0_PE_ON, %eax
    movl %eax, %cr0

5.跳转到32位代码处开始执行

	ljmp $PROT_MODE_CSEG, $protcseg

6.初始化各个寄存器

	# Set up the protected-mode data segment registers
    movw $PROT_MODE_DSEG, %ax                       # Our data segment selector
    movw %ax, %ds                                   # -> DS: Data Segment
    movw %ax, %es                                   # -> ES: Extra Segment
    movw %ax, %fs                                   # -> FS
    movw %ax, %gs                                   # -> GS
    movw %ax, %ss                                   # -> SS: Stack Segment

    # Set up the stack pointer and call into C. The stack region is from 0--start(0x7c00)
    movl $0x0, %ebp
    movl $start, %esp

7.完成对保护模式的切换，进入bootmain：

	call bootmain

### 练习4：分析bootloader加载ELF格式的OS的过程。（要求在报告中写出分析）
#### 问题1：bootloader如何读取硬盘扇区的？
在bootmain.c文件中，定义了readsect函数用以读取磁盘扇区：（省略号处的代码留作之后拆开逐一分析）

	/* readsect - read a single sector at @secno into @dst */
	static void
	readsect(void *dst, uint32_t secno) {
		// wait for disk to be ready
    	waitdisk();
	    ......
	}

由此可见，readsect函数首先调用了waitdisk函数，waitdisc函数定义如下：

	/* waitdisk - wait for disk ready */
	static void
	waitdisk(void) {
	    while ((inb(0x1F7) & 0xC0) != 0x40)
	        /* do nothing */;
	}

waitdisc函数用来等待磁盘扇区就绪，方法是一直查看0x1F7处的数据，直到其高两位为01就认为已经就绪。接下来，readsect函数向磁盘扇区里写入一些特定数据：

    outb(0x1F2, 1);                         // count = 1
    outb(0x1F3, secno & 0xFF);
    outb(0x1F4, (secno >> 8) & 0xFF);
    outb(0x1F5, (secno >> 16) & 0xFF);
    outb(0x1F6, ((secno >> 24) & 0xF) | 0xE0);
    outb(0x1F7, 0x20);                      // cmd 0x20 - read sectors

这些写操作主要是为了完成发出读取磁盘请求等任务。最后，readsect函数将再次等待磁盘就绪，并从磁盘读取一个扇区：

	// wait for disk to be ready
    waitdisk();

    // read a sector
    insl(0x1F0, dst, SECTSIZE / 4);

#### 问题2：bootloader是如何加载ELF格式的OS？
在bootmain函数中，以下代码和加载ELF格式的OS有关。将其拆分成几部分依次分析：

    // is this a valid ELF?
    if (ELFHDR->e_magic != ELF_MAGIC) {
        goto bad;
    }

上面这部分代码首先判断该文件是不是合法的ELF格式，如果不是，就跳过加载过程。

    struct proghdr *ph, *eph;

    // load each program segment (ignores ph flags)
    ph = (struct proghdr *)((uintptr_t)ELFHDR + ELFHDR->e_phoff);
    eph = ph + ELFHDR->e_phnum;

上面这部分代码将读取ELF文件里的描述表等信息读到变量里，并计算其他需要的数据，以便后续的读取等处理。

    for (; ph < eph; ph ++) {
        readseg(ph->p_va & 0xFFFFFF, ph->p_memsz, ph->p_offset);
    }

上面这部分代码根据前一阶段获取到的信息，调用readseg函数，将ELF文件加载到内存中

    // call the entry point from the ELF header
    // note: does not return
    ((void (*)(void))(ELFHDR->e_entry & 0xFFFFFF))();

最后，根据ELF头表中的入口信息，找到内核的入口并开始运行。

### 练习5：实现函数调用堆栈跟踪函数 （需要编程）

在lab1里我已经实现函数调用堆栈跟踪函数print_stackframe，经测试结果正常，截图如下：

![](http://i.imgur.com/X82gQOv.png)

其中最后一行为

    ebp:0x00007bf8 eip:0x00007d68 args:0xc031fcfa 0xc08ed88e 0x64e4d08e 0xfa7502a8 
    <unknow>: -- 0x00007d67 --

这一行代表堆栈最深一层的情况，也即第一个使用堆栈的函数，这个函数是bootmain.c中的bootmain。这句话的意思是，bootloader设置的堆栈从0x7c00开始，使用"call bootmain"指令压栈和调用bootmain函数，bootmain中ebp为0x7bf8。而0xc031fcfa，0xc08ed88e，0x64e4d08e，0xfa7502a8为其使用的4个参数。

### 练习6：完善中断初始化和处理 （需要编程）

问题1回答为：

中断向量表一个表项占用8字节；该表项的2-3字节是段选择子，0-1字节和6-7字节拼成位移；中断处理程序的入口地址由两者联合构成。

### 扩展练习Challenge1与2：见代码
