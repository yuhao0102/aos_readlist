# uCore 中期汇报

## 实验目的
完成uCore的8个实验，并在实验基础上深入阅读代码，加深对uCore和操作系统底层实现原理的认识，掌握C编写底层代码的方法和技巧。

## 实验环境
- uCore
- Ubuntu
- Git 2.7.4
- gcc 5.4.0
- qemu

## 实验内容
uCore共分为8个实验，主要实现了对物理内存、虚拟内存的管理；线程、进程的创建和调度；同步互斥的实现；文件系统的实现几个部分。

### lab1
lab1主要是对环境的熟悉和基本操作。

1. 在lab1中，搭建了基于Linux和qemu的运行环境，并对Makefile进行了初步分析，使用gdb完成调试。
2. 分析了bootloader从实模式进入保护模式的过程。bootloader被BIOS加载到内存的0x7c00处，同时实现了开启A20的过程：
    - 从0x64地址中中读取8042的状态，直到检测到读取到的字节第二位为0，即端口不忙；
    - 向0x64写入0xd1命令，修改8042的P2端口；
    - 继续等不忙，向0x60端口写入0xDF，表示将P2 port的第二个位（A20）置为1，开启A20；
    - 使用命令`lgdt gdtdesc`即可载入全局描述符表；
    - 接下来只需要将cr0寄存器的PE位置1，即可从实模式切换到保护模式。

    > 如果A20 Gate被打开，则当程序员给出100000H-10FFEFH之间的地址的时候，系统将真正访问这块内存区域；

    > 如果A20 Gate被禁止，则当程序员给出100000H-10FFEFH之间的地址的时候，系统仍然使用8086/8088的方式即取模方式（8086仿真）。绝大多数IBM PC兼容机默认的A20 Gate是被禁止的。现在许多新型PC上存在直接通过BIOS功能调用来控制A20 Gate的功能。

3. 加载ELF格式的过程比较简单，这里使用了一些函数，涉及读取磁盘扇区的操作。主要有：
    - 设置读取的扇区数量，写入0x1f2端口；
    - 设置起始LBA（logical block address）扇区号；
    - 向端口 0x1f7 写入 0x20请求硬盘读；
    - 等待读写操作完成，即等待 0x1f7 端口第3位置1；
    - 连续取出数据。

4. 建立中断向量表，这个函数主要依靠了gatedesc 结构体和SETGATE宏。
    - 中断描述符表（IDT）把每个中断或异常编号和一个指向中断服务例程的描述符联系起来，对IDT中的每一项，调用SETGATE进行设置。
    - \_\_vectors[i]是在代码段中的偏移量，vectors[i]在kern/trap/vectors.S中定义，定义了255个中断服务例程的地址，这里才是入口，且都跳转到__alltraps。
    - \_\_alltraps调用了trap函数，在trap中调用了trap_dispatch，这样就根据传进来的中断号进行switch处理。
    - 在trap_dispatch中设置一个计数器，实现了每隔100个时钟周期输出一句话的操作。

### lab2
lab2主要是对物理内存的发现和初步管理。

1. 在做实验之前先熟悉了发现物理内存的部分代码。
    - 在bootloader进入保护模式前进行探测物理内存分布和大小，在实模式下完成。每探测到一块内存空间，对应的内存映射描述符被写入指定表；
    - 主要操作是调用int 0x15中断并向其传入e820h参数来探测物理内存空间的信息，返回一个内存段信息，保存在struct e820map结构体中。

2. 为了对Page等结构体进行有效的管理，定义了一种特殊的通用双向循环链表。
    - 具体来说，就是在链表中不包含数据域，只有prev和next指针；
    - 在数据节点（如Page结构体）中定义一个链表节点，这样在双向链表中仅保存了特定数据结构中的链表节点成员，实现了通用性。

3. 这带来一个问题，就是怎么从链表节点中找到我们想要的那个数据节点，uCore提供了宏（如le2page），它的实现很有技巧，因此在此详述。
    - `#define le2page(le, member) to_struct((le), struct Page, member)`中使用了`to_struct`宏；
    - `#define to_struct(ptr, type, member) ((type *)((char *)(ptr) - offsetof(type, member)))` 中的ptr是这个（Page+双向链表的两个指针）块的双向链表指针的开始地址。
    - `#define offsetof(type, member) ((size_t)(&((type *)0)->member))` `offset`宏算出了page_link在Page中的偏移值，ptr减去双向链表第一个指针的偏移量得到了这个Page的地址
    - 在`offset`宏中的0不代表具体的数据对象，而是“0地址”，把这个“0地址”转换成type类型在访问member成员，就得到了这个member成员相对于数据结构变量的偏移量。

4. 练习1实现了first-fit连续物理内存分配算法，主要重写了`default_alloc_pages`和`default_free_pages`两个函数。
    - `default_alloc_pages` 从空闲页链表中查找n个空闲页。主要步骤是遍历空闲链表，一旦发现有大于等于n的连续空闲页块，便将这个Head Page从空闲页链表中取出，同时使用SetPageReserved和ClearPageProperty表示这n个页为使用状态。同时如果该连续页的数目大于n，则从第n+1开始截断，之后为截断的块重新计算相应的property的值。
    - `default_free_pages`将base为起始地址的n个页面放回到free_list中。主要步骤为：
        - 首先找到比base大的页面地址作为插入地址，逐个插入；
        - 对被插入的页需要清空他们的flag信息和引用信息，设置property
        - 如果base为首的n个页之前/后还有空页，需要合并

5. 练习2实现了寻找虚拟地址对应的页表项。
    - 首先明确虚拟地址、线性地址以及物理地址之间的映射关系：`virt addr = linear addr = phy addr + 0xC0000000`
    - 本练习补充了get_pte函数。给定page directory以及线性地址，查询出该线性地址（la）对应的PTE（page table entry），并且根据输入参数要求判断是否创建不存在的页表。
    - 用到了`KADDR(pa)`宏，主要是通过物理地址（pa）得到虚拟地址。
    - 主要过程是获取对应的pgdir，请求一个物理页存储新创建的页表，初始化并返回线性地址对应的页目录项。

6. 练习3实现了释放某虚地址所在的页。
    - 本练习比较简单，主要是使用`pte2page(*ptep)`获取页表项对应的物理页的Page结构，并对已经没有引用映射的物理页调用`free_page`进行释放，设置`PTE_P`存在位为0表示该映射关系无效。
    - 最后调用了`tlb_invalidate`函数，它的作用是“刷新TLB，确保TLB的缓存中不会有错误的映射关系”，在做实验的时候忽略了这一点。


### lab3
lab3主要是实现了Page Fault异常处理和FIFO页替换算法。涉及了do_pgfault函数，它会申请一个空闲物理页，并建立好虚实映射关系；
ide_init函数，完成对用于页换入换出的初始化工作。另外涉及的数据结构有`mm_struct(mm)`和`vma_struct(vma)`。mm是具有相同PDT的连续虚拟内存区域集的内存管理器。 vma是一个连续的虚拟内存区域。

1. 在描述实验之前，先介绍kmalloc。在内核模块中动态开辟内存，不是用malloc，而是kmalloc，释放内存用的是kfree。kmalloc函数返回的是虚拟地址（线性地址）。kmalloc特殊之处在于它分配的内存在物理上连续。

2. 本实验用do_pgfault函数来处理Page Fault异常，当启动分页机制以后，如果一条指令或数据的虚拟地址所对应的物理页框不在内存中或者访问的类型有错误（比如写一个只读页或用户态程序访问内核态的数据等），就会发生页访问异常，CPU把引起页访问异常的线性地址装到寄存器CR2中。保存现场后，按照如下调用序列进行异常处理。
    ```
    trap --> trap_dispatch --> pgfault_handler --> do_pgfault
    ```

3. 在do_pgfault函数中，有如下操作：
    - 查询了mm_struct中的合法的虚拟地址，使用error code以及该线性地址的内存页判断是否合法；
    - 如果通过了权限检查，进行下边的操作；
    - 使用get_pte获取出错的线性地址所对应的虚拟页起始地址对应到的页表项，这里判断返回值；
    - 如果需要的物理页是没有分配而不是被换出到外存中，即返回了0，就分配一个页面并使用逻辑地址映射物理地址；
    - 否则的话说明对应的物理页被换出在外存中，需要被换进内存。这是练习2的内容，下述。

4. 如果需要将物理页换入内存，则首先需要知道，物理页在外存中的位置保存在了PTE中。主要过程如下：
    - 检查`swap_init_ok`判断当前的交换机制初始化完成；
    - 调用`swap_in`将物理页换入到内存中；在`swap_in`函数中，又调用了`alloc_pages(1)`函数进行分配物理页，如果没有足够的物理页，则会使用`swap_out`函数将某一页换出到外存中，该函数会进一步调用`swap_out_victim`函数来选择需要换出的“受害者”物理页，在这里对应`_fifo_swap_out_victim`函数；
    - 调用`page_insert`完成物理页与虚拟页映射关系的建立；
    - 调用`swap_map_swappable`，这里实际上调用的是`_fifo_swap_map_swappable`函数，把当前的物理页面插入到FIFO算法中表明可被交换出去的物理页面链表的末尾，也就是设置当前的物理页为可交换的，同时也体现了该链表中越接近链表头的物理页面在内存中的驻留时间越长；

### lab4
本实验涉及了内核线程的创建执行的管理过程以及内核线程的切换和基本调度过程。内核线程是一种特殊的进程，内核线程与用户进程的区别有两个：内核线程只运行在内核态、所有内核线程共用ucore内核内存空间。

1. 对第0个内核线程idleproc，其工作就是不停地查询，看是否有其他内核线程可以执行了，如果有，马上让调度器选择那个内核线程执行。使用kmalloc函数获得`proc_struct`结构的一块内存块，作为第0个PCB，并进行初始化。把`idleproc->need_resched`设置为1，表明如果idleproc在执行了，就马上就调用schedule函数调度切换其他进程开始执行。创建idleproc的调用在：
    ```
    kern_init --> proc_init --> alloc_proc
    ```
2. 接着就是调用kernel_thread函数中的do_fork来创建initproc内核线程。
    - kernel_thread函数的局部变量tf保存了内核线程的临时中断帧，在tf中设置了eip为kernel_thread_entry，指出了initproc内核线程从kernel_thread_entry开始执行；
    - `kernel_thread_entry`主要是把fn函数执行的参数压栈，并`call fn`，为fn打造了一个执行环境，为其提供参数后把函数返回值eax寄存器内容压栈，调用do_exit函数退出线程执行； 
    - 把中断帧的指针传递给do_fork函数，而do_fork函数在调用copy_thread时拷贝了在kernel_thread函数建立的临时中断帧tf的值。这相当于是把initproc需要执行的函数`init_main`及其参数告诉了do_fork；
    - do_fork是创建线程的主要函数，主要做了以下6件事情：
        - 分配并初始化进程控制块（alloc_proc函数）；
        - 分配并初始化内核栈（setup_stack函数）；
        - 根据clone_flag标志复制或共享进程内存管理结构（copy_mm函数）；
        - 设置进程在内核正常运行和调度所需的中断帧和执行上下文（copy_thread函数）；
        - 把设置好的进程控制块放入hash_list和proc_list两个全局进程链表中；
        - 进程已经准备好执行了，把进程状态设置为“就绪”态；设置返回码为子进程的id号。

3. idleproc通过执行cpu_idle函数让出CPU，给其它内核线程执行，具体过程如下：
    - 调用schedule函数找其他处于“就绪”态的进程执行，主要有以下步骤；
        - 当前进程的need_resched置为0；
        - 查找一个处于“就绪”态的线程或进程；
    - 找到后，就调用proc_run函数；
        - 修改当前的cr3寄存器成需要运行线程（进程）的页目录表；
        - 保存当前进程current的上下文，执行切换（内部的switch_to）。

4. struct context context的作用主要是存储除了eax之外的所有通用寄存器以及eip的数值，保存了线程运行的上下文信息；struct trapframe \*tf是在构造出新的线程并将控制权交给这个线程时提供一个中断返回现场。

5. 本实验中总共创建了两个内核线程，分别为：
    - idleproc: 最初的内核线程，在完成新的内核线程的创建以及各种初始化工作之后，进入死循环，用于调度其他线程；
    - initproc: 被创建用于打印"Hello World"的线程；

6. local_intr_save(intr_flag);....local_intr_restore(intr_flag)主要实现了关闭中断，使得在这个语句块内的内容不会被中断打断，是一个原子操作；
在proc_run函数中，需要将current指向要切换到的线程，如果在这个时候出现中断打断这些操作，就会出现current中保存的并不是正在运行的线程的中断控制块，从而出现错误。

### lab5 
lab4的线程运行都在内核态，lab5创建了用户进程，让用户进程在用户态执行，且可以调用系统调用。uCore提供了用户态进程的创建和执行机制，在进程管理和内存管理部分需要深入考虑。

正在通过mooc和指导书学习本部分实验原理。

## 实验收获
uCore实验实现了从简单到复杂的构建一个操作系统的工作。在实验中，更加深入地了解了操作系统底层是如何与硬件配合，完成对中断的管理，对物理内存和虚拟内存的分配、管理和映射，同时，对线程与进程的创建、执行过程有了更进一步的体会。

这也加深了我对OS底层的认识，相信对之后的工作也大有裨益。