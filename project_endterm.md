# uCore 汇报

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
lab1主要是对环境的熟悉、基础代码的熟悉和对堆栈、中断等进行处理的基本操作。

1. 在lab1中，搭建了基于Linux和qemu的运行环境，并对Makefile进行了初步分析，可以使用gdb完成调试。
2. 了解了线性地址、物理地址、逻辑地址之间的关系。
    - 逻辑地址空间就是应用程序员编程所用到的地址空间。
    - 物理地址是指CPU提交到内存总线上用于访问计算机内存和外设的最终地址，CPU通过物理地址访问物理地址空间。
    - 线性地址又称虚拟地址，是进行逻辑地址转换后形成的地址索引，用于寻址线性地址空间。
    - 逻辑地址（由段选择子selector和段偏移offset组成）中的段选择子的内容作为段描述符表的索引，找到表中对应的段描述符，然后把段描述符中保存的段基址加上段偏移值，形成线性地址（Linear Address）。
    - 分页地址转换，这一步中把线性地址转换为物理地址。
    - 逻辑地址即是程序指令中使用的地址，物理地址是实际访问内存的地址。逻辑地址通过段式管理的地址映射可以得到线性地址，线性地址通过页式管理的地址映射得到物理地址。

3. 分析了bootloader从实模式进入保护模式的过程。bootloader被BIOS加载到内存的0x7c00处，同时实现了开启A20的过程：
    - 从0x64地址中中读取8042的状态，直到检测到读取到的字节第二位为0，即端口不忙；
    - 向0x64写入0xd1命令，修改8042的P2端口；
    - 继续等不忙，向0x60端口写入0xDF，表示将P2 port的第二个位（A20）置为1，开启A20；
    - 使用命令`lgdt gdtdesc`即可载入全局描述符表；
    - 接下来只需要将cr0寄存器的PE位置1，即可从实模式切换到保护模式。

    > 如果A20 Gate被打开，则当程序员给出100000H-10FFEFH之间的地址的时候，系统将真正访问这块内存区域；

    > 如果A20 Gate被禁止，则当程序员给出100000H-10FFEFH之间的地址的时候，系统仍然使用8086/8088的方式即取模方式（8086仿真）。绝大多数IBM PC兼容机默认的A20 Gate是被禁止的。现在许多新型PC上存在直接通过BIOS功能调用来控制A20 Gate的功能。

4. 加载ELF格式的过程比较简单，这里使用了一些函数，，涉及读取磁盘扇区的操作，如outb函数其实是封装了out汇编指令，下述的流程主要是向控制器的特定端口发送相关指令，实现数据读取。主要有：
    - 设置读取的扇区数量，写入0x1f2端口；
    - 设置起始LBA（logical block address）扇区号；
    - 向端口 0x1f7 写入 0x20请求硬盘读；
    - 等待读写操作完成，即等待 0x1f7 端口第3位置1；
    - 连续取出数据。

5. 实现函数调用堆栈跟踪函数
    - 使用read_ebp和read_eip函数获取当前stack frame的base pointer以及这条指令下一条指令的地址
    - 打印出当前栈帧对应的函数的参数
    - 查找当前函数的调用者的栈帧。调用者的栈帧的bp存放在被调用者的ebp指向的内存单元，将其更新到ebp临时变量中，同时将eip更新为调用当前函数的指令的下一条指令所在位置，存放在ebp+4所在的内存单元中；

6. 建立中断向量表，这个函数主要依靠了gatedesc结构体和SETGATE宏。
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

5. 对物理页进行管理的数据结构
    - `struct Page`是保存一个物理页的数据结构，其中的ref记录了映射此物理页的虚拟页个数，flags描述了物理页属性（是否被保留(reserved)）和双向链接各个Page结构的page_link双向链表。用到page_link的是每个空闲区域的地址最小的一个页。
    - 所有的连续内存空闲块可用一个双向链表`free_area_t`管理，它有一个list_entry结构的双向链表指针和记录当前空闲页的个数的无符号整型变量nr_free。

6. 练习1实现了first-fit连续物理内存分配算法，主要重写了`default_alloc_pages`和`default_free_pages`两个函数。
    - `default_alloc_pages` 从空闲页链表中查找n个空闲页。主要步骤是遍历空闲链表，一旦发现有大于等于n的连续空闲页块，便将这个Head Page从空闲页链表中取出，同时使用SetPageReserved和ClearPageProperty表示这n个页为使用状态。同时如果该连续页的数目大于n，则从第n+1开始截断，之后为截断的块重新计算相应的property的值。
    - `default_free_pages`将base为起始地址的n个页面放回到free_list中。主要步骤为：
        - 首先找到比base大的页面地址作为插入地址，逐个插入；
        - 对被插入的页需要清空他们的flag信息和引用信息，设置property
        - 如果base为首的n个页之前/后还有空页，需要合并

6. 练习2实现了寻找虚拟地址对应的页表项。
    - 首先明确虚拟地址、线性地址以及物理地址之间的映射关系：`virt addr = linear addr = phy addr + 0xC0000000`
    - 本练习补充了get_pte函数。给定page directory以及线性地址，查询出该线性地址（la）对应的PTE（page table entry），并且根据输入参数要求判断是否创建不存在的页表。
    - 用到了`KADDR(pa)`宏，主要是通过物理地址（pa）得到虚拟地址。
    - 主要过程是获取对应的pgdir，请求一个物理页存储新创建的页表，初始化并返回线性地址对应的页目录项。

7. 练习3实现了释放某虚地址所在的页。
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

4. `struct context context`的作用主要是存储除了eax之外的所有通用寄存器以及eip的数值，保存了线程运行的上下文信息；`struct trapframe *tf`是在构造出新的线程并将控制权交给这个线程时提供一个中断返回现场。

5. 本实验中总共创建了两个内核线程，分别为：
    - idleproc: 最初的内核线程，在完成新的内核线程的创建以及各种初始化工作之后，进入死循环，用于调度其他线程；
    - initproc: 被创建用于打印"Hello World"的线程；

6. `local_intr_save(intr_flag);....local_intr_restore(intr_flag)`主要实现了关闭中断，使得在这个语句块内的内容不会被中断打断，是一个原子操作；
在proc_run函数中，需要将current指向要切换到的线程，如果在这个时候出现中断打断这些操作，就会出现current中保存的并不是正在运行的线程的中断控制块，从而出现错误。

### lab5 
lab4的线程运行都在内核态，lab5创建了用户进程，让用户进程在用户态执行，且可以调用系统调用。uCore提供了用户态进程的创建和执行机制，在进程管理和内存管理部分需要深入考虑。

1. 本实验中第一个用户进程是由内核线程initproc通过把hello应用程序执行码覆盖到initproc的用户虚拟内存空间来创建的。
    - ld命令会在kernel中会把__user_xxxx.out的位置和大小记录在全局变量**`_binary_obj___user_xxxx_out_start`** 和**`_binary_obj___user_xxxx_out_size`**中，这样这个用户程序就能够和ucore内核一起被 bootloader 加载到内存里中，并且通过这两个全局变量定位用户程序执行码的起始位置和大小；
    - initproc实际上是执行了`init_main`函数，它执行宏`KERNEL_EXECVE`，最终调用`kernel_execve`来调用`SYS_exec`系统调用，kernel_execve把上述两个变量作为`SYS_exec`系统调用的参数，让ucore来创建此用户进程。当ucore收到此系统调用后，最终调用`do_execve`函数；
    - `do_execve`清空用户态内存空间，释放进程所占用户空间内存和进程页表本身所占空间，调用`load_icode`完成加载执行码到当前进程的用户态虚拟空间中的工作；

2. `load_icode`函数读入一个ELF可执行二进制文件并执行。
    - 在完成本步之前，要做一些准备工作;
        - 在LAB1的初始化IDT时，设置系统调用对应的中断描述符，使其能够在用户态下被调用；
        - 每经过n个时钟周期，当前进程需要被设置为可调度的，这样当前的线程可以被及时换出；
        - 在`proc_alloc`函数中，对`proc_struct`增加的几个成员变量进行初始化；
        - `do_fork`中使用`set_links`函数完成插入线程链表的操作，这个函数包括了一些额外操作，要注意不重复执行某些步骤（比如进程总数加1。。。）；
    - 在`load_icode`的前半部分我们需要补充的代码主要实现构造一个中断返回现场，使得中断返回的时候可以正确地切换到需要的执行程序入口处。在这个部分中需要对tf进行设置。
        - 把段寄存器初始化为用户态的代码段、数据段、堆栈段；
        - esp应当指向先前的步骤中创建的用户栈的栈顶USTACKTOP；
        - eip应当指向ELF可执行文件加载到内存之后的入口处；
        - eflags中应当初始化为中断使能；
		- 设置ret为0，表示正常返回；

3. 父进程需要复制自己的内存空间给子进程，主要涉及的函数是`copy_range`。
    - 首先，根据上文说的，在user_main中调用宏`KERNEL_EXECVE(exit)`，在宏中拼接了“exit”，相当于是把exit的入口告知；
    - 执行上述的操作，把exit加载进来；
    - 在exit中，父进程调用了`fork()`，交至syscall根据`SYS_fork`调用号来处理；
    - syscall根据调用号进一步调用sys_fork --> do_fork，调用copy_mm --> dup_mmap进行内存空间的复制；
    - 接下来调用copy_range函数，以页为单位从父进程的内存空间复制到子进程的内存空间；
        - 首先为其分配一个页；
        - 找到父进程指定的某一物理页对应的内核虚拟地址；
        - 找到子进程的对应物理页对应的内核虚拟地址；
        - 将前者的内容拷贝到后者中去（memcpy）；
        - 为子进程当前分配这一物理页映射上在子进程虚拟地址空间里的一个虚拟页；

4. 在执行了某个系统调用之后，会将控制权转移给syscall，之后根据系统调用号执行特定函数，进一步执行上文中的do_execve、do_fork等函数。调用顺序如下，其中T_SYSCALL=0x80=128：`vector128(vectors.S) --> 
__alltraps(trapentry.S) --> trap(trap.c) --> trap_dispatch(trap.c) --> syscall(syscall.c)`

### lab6
本实验在ucore的调度器框架的基础上了解了Round-Robin，实现了Stride Scheduling调度算法。

1. 进程的生命周期如下：
	- 调度器sched_class根据run_queue的内容来判断一个进程是否应该被运行，把处于runnable态的进程转换成running状态；
	- running态的进程通过wait等系统调用被阻塞，进入sleeping态；
	- sleeping态的进程被wakeup变成runnable态的进程；
	- running态的进程exit变成zombie态后由其父进程完成对其资源的最后释放；
	- 所有从runnable态变成其他状态的进程都要出运行队列，反之，被放入某个运行队列中。

2. lab6实现了一个调度框架，如果要实现一个调度器，需要完成一些函数，而这些函数在框架中又被封装成一下的函数：
	- sched_init函数，用于调度算法的初始化，主要是初始化run_queue，设置调度器；
	- sched_class_enqueue函数：将指定的进程的状态置成RUNNABLE，并且放入调用算法中的可执行队列中；
	- sched_class_dequeue函数：将某个在队列中的进程取出，即调度算法选择的进程需要等待的可执行的进程队列中取出并执行；
	- sched_class_pick_next函数：选择要执行的下个进程；
	- sched_class_proc_tick函数：每次时钟中断的时候应当调用的调度算法函数，如时间片减一；
	- trap中调用了trap_dispatch和schedule函数，能够在need_resched时及时调度。

3. RR调度算法，它让所有runnable态的进程分时轮流使用CPU时间，当前进程的时间片用完之后，调度器将当前进程放置到运行队列的尾部，再从其头部取出进程进行调度。
	- RR_enqueue：某进程的PCB放入到run_queue队列末尾，如果时间片已经0，则重置为max_time_slice；
	- RR_pick_next：选取就绪进程队列run_queue中的队头元素，并使用le2proc转换成PCB；
	- RR_dequeue：就绪进程队列run_queue的队列元素删除，并把表示就绪进程个数的proc_num减一；
	- RR_proc_tick：时钟中断时，当前执行进程的时间片time_slice减一。如果time_slice为零，则设置此进程成员变量need_resched标识为1。

4. Stride Scheduling。本stride调度算法的实现中使用了斜堆来实现优先队列。
	- init函数初始化当前的运行队列；
	- enqueue函数把进程加入run_pool，并设置时间片为rq->max_time_slice；
	- dequeue函数把进程移出run_pool；
	- 进程的stride表示该进程当前的调度权，也可以表示这个进程执行了多久了。pass表示对应进程在调度后stride需要进行的累加值；
	- stride_pick_next表示每次需要调度时，从状态为 runnable 态的进程中选择stride最小的进程调度，选出的进程P的stride加上其对应的步长pass；
	- 在一段固定的时间之后，重新调度当前stride最小的进程。
	- `P.pass =BigStride / P.priority`，其中`P.priority`表示进程的优先权（大于 1），而 BigStride 表示一个预先定义的大常数，则该调度方案为每个进程分配的时间将与其优先级成正比。

5. 斜堆的相关实现

### lab7
本实验实现了同步互斥的操作

1. 在ucore中提供的底层机制包括中断屏蔽/使能控制等。`kern/sync.c`有开关中断的控制函数`local_intr_save(x)`和`local_intr_restore(x)`，它们是基于`kern/driver`文件下的`intr_enable()`、`intr_disable()`函数实现的。具体调用关系为：
	- 关中断：`local_intr_save --> __intr_save --> intr_disable --> cli`
	- 开中断：`local_intr_restore --> __intr_restore --> intr_enable --> sti`
	- cli和sti是x86的机器指令，最终实现了关（屏蔽）中断和开（使能）中断，即设置了eflags寄存器中与中断相关的位。
	- 关闭中断，可以避免执行的程序被中断处理事件打断造成异常

2. 等待项wait结构和等待队列wait queue结构以及相关函数是实现ucore中的信号量机制和条件变量机制的基础，进入wait queue的进程会被设为等待状态（PROC_SLEEPING），直到他们被唤醒，主要有以下几个函数：
	- wait_init：初始化wait结构
	- wait_in_queue：检查wait是否在wait queue中
	- wait_queue_init：初始化wait_queue结构
	- wait_queue_add：设置当前等待项wait的等待队列，并把wait前插到wait queue中
	- wait_queue_del：从wait queue中删除wait
	- wait_queue_next：取得wait_queue中wait等待项的后一个链接指针
	- wakeup_wait：唤醒等待队列上的wait所关联的进程

3. 信号量是一种同步互斥机制的实现
	- P操作函数`down(semaphore_t *sem)`
		- 关中断，信号量的value>0可以获得信号量，打开中断返回即可；
		- value≤0，则无法获得信号量，进程加入到等待队列中，开中断；
		- 然后运行调度器选择另外一个进程执行；
		- 唤醒后把自身关联的wait从wait_queue中删除。
	- V操作函数`up(semaphore_t *sem)`
		- 关中断，信号量的wait queue若为空则直接把信号量的value++，开中断返回；
		- 如果有进程在等待则调用wakeup_wait函数将wait_queue中等待的第一个wait删除，且把此wait关联的进程唤醒，最后开中断返回。

4. 管程和条件变量。管程定义了一个局限在管程内部的数据结构，和一组管程内部的过程，这个数据结构只能被内部的过程所访问，管程之外的过程不能访问它。
	- ucore中的管程基于信号量和条件变量，它的数据结构中包含：
		- mutex，控制只有一个进程能够进入管程中执行操作；
		- cv，管程中的条件变量；
		- next是用来控制一个进程离开管程之后该唤醒哪个管程，维持进程间相互唤醒；
		- next_count则表示由于发出singal_cv而睡眠的进程个数。
	- 条件变量中的数据项包括：
		- 信号量sem，用于让等待某个条件Cond为真的进程睡眠，让发出signal_cv操作的进程通过这个sem来唤醒睡眠的进程；
		- count表示等在这个条件变量上的睡眠进程的个数；
		- owner表示此条件变量的宿主是哪个管程。
	- A执行了cond_wait函数：
		- 等待此条件的睡眠进程个数cv.count要加一。
		- monitor.next_count大于0表示有进程执行cond_signal函数且等待在next上，则唤醒进程B；
		- monitor.next_count小于等于0，唤醒的等待monitor的进程；
		- A在cv.sem上等待
	- A执行了cond_signal函数：
		- cv.count小于等于0，没有执行cond_wait而睡眠的进程，直接返回；
		- cv.count大于0，则有执行cond_wait而睡眠的进程，因此需要唤醒并且自己在next上等待，monitor.next_count++；

### lab8
本实验展示了ucore的文件系统
1. core的文件系统架构主要由四部分组成：
	- 通用文件系统访问接口层：文件系统的标准访问接口
	- 文件系统抽象层：提供一致的访问接口和抽象函数指针列表和数据结构；
	- Simple FS文件系统层：一个基于索引方式的简单文件系统实例；
	- 外设接口层：提供抽象访问接口屏蔽不同硬件细节，向下实现访问各种具体设备驱动的接口
2. SFS文件系统主要有以下几部分：
	- 超级块，包含了关于文件系统的所有关键参数，如魔数（0x2f8dbe2a）等；
	- root-dir的inode，用来记录根目录的相关信息；
	- unused_blocks：类似freemap，表示块是否被占用；
	- info：所有其他目录和文件的inode信息；
3. 练习一主要是实现了读取文件数据的函数：
	- sys_read函数设置了文件指针和读取长度、基地址等信息，调用了sysfile_read函数；
	- sysfile_read函数中，kmalloc创建了一个缓冲区；
	- 在sysfile_read中调用了file_read，调用fd2file获取这个文件描述符，查找到了对应inode信息；
	- 调用vop_read进行读取，通过函数指针转交给sfs_read函数；
	- 调用sfs_io函数，主要是找到代表这个文件系统的fs和对应inode；
	- 进一步调用了sfs_io_nolock函数；
		- 首先根据读或写为函数指针赋值；
		- 计算要读写的块数、长度、偏移；
		- 根据开始读的偏移是不是在一个块的起始判断是否要从一个块的中间部分开始读；
		- 读写中间的完整块；
		- 读写末尾的块（如果最末尾有不完整的块的话）；
		- 根据块是否完整调用sfs_buf_op或sfs_block_op；

4. 练习二重写了load_icode函数，调用栈为sys_exec --> do_execve，这个函数主要是为了加载ELF可执行文件。主要是以下几个部分：
	- 创建内存管理结构mm和页目录表PDT（mm_create和setup_pgdir）；
	- 将磁盘上的ELF文件的TEXT/DATA/BSS段正确地加载到用户空间中；
	- 调用load_icode_read-->sysfile_read读取elf文件的header；
	- 根据elfheader中elf.e_phnum，获取到磁盘上所有的program header；
	- 对于每一个program header：
		- 这里同样也是调用load_icode_read读取；
		- 为段创建vma并设置权限，同时建立物理页和虚拟页的映射关系；
		- 调用load_icode_read读取段，并且复制到用户内存空间上去；
		- 设置设置用户栈，首先根据传入参数的个数计算栈顶应该在哪，为栈分配分配物理页；
		- 把argc参数和argv参数压入栈；
		- 设置trapframe，其中栈顶位置为之前计算的到的栈顶位置；

5. PIPE机制可以看成是一个缓冲区，可以在磁盘上（或内存中？更快）保留一部分空间作为pipe机制的缓冲区。当两个进程之间要求建立pipe时，在两个进程的进程控制块上修改某些属性表明这个进程是管道数据的发送方还是接受方，这样就可以将stdin或stdout重定向到生成的临时文件里，在两个进程中打开这个临时文件。
    - 当进程A使用stdout写时，查询PCB中的相关变量，把这些stdout数据输出到临时文件中；
    - 当进程B使用stdin的时候，查询PCB中的信息，从临时文件中读取数据；

6. UNIX的软硬链接
	- 硬链接： 与普通文件没什么不同，inode 都指向同一个文件在硬盘中的区块
	- 软链接： 保存了其代表的文件的绝对路径，是另外一种文件，在硬盘上有独立的区块，访问时替换自身路径。
	- sfs_disk_inode结构体中有一个nlinks变量，如果要创建一个文件的软链接，这个软链接也要创建inode，只是它的类型是链接，找一个域设置它所指向的文件inode，如果文件是一个链接，就可以通过保存的inode位置进行操作；当删除一个软链接时，直接删掉inode即可；
	- 硬链接与文件是共享inode的，如果创建一个硬链接，需要将源文件中的被链接的计数加1；当删除一个硬链接的时候，除了需要删掉inode之外，还需要将硬链接指向的文件的被链接计数减1，如果减到了0，则需要将源文件删除掉；