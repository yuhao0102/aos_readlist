# Report for 'The UNIX Time-Sharing System'
本篇文章主要对Unix的文件系统、进程及Shell的工作原理进行了介绍。正如评语中所说的，在操作系统趋向于复杂化的时代，Unix成为优雅和简洁的标志。在系统的设计过程中，主要有三点考量：
1. 从程序员的角度让Unix更利于开发、测试和运行程序。交互式的系统设计方便了使用，同时也可以通过脚本的形式实现批处理；
2. 对系统资源的严格大小限制。不仅体现在性能上，也体现了整个系统的优雅设计；
3. 自我迭代使得开发人员可以根据技术的发展和其他人的建议迅速修改系统，使其不断改进。  

## Summary of major innovations 
文章首先介绍了Unix的发展与应用，C语言的使用虽然使其比用汇编语言写成的操作系统更大，但是其具有更好的可读性和可理解性。其次介绍了Unix文件系统的设计和实现，对于不同类型的文件提供了不同的访问机制，同时提供了文件的权限控制和IO调用接口实现对文件的读写。之后介绍了进程的实现与进程的切换、进程间通信的机制等。最后介绍了shell的一些特性。其主要的创新点有：
1. 使用C编写系统内核，提高了可移植性和可维护性；
2. UNIX将所有设备都用文件表示，简化了设备管理，普通文件与设备可采用一套系统调用集访问。设备驱动只需实现对应的函数。与shell中的I/O重定向相结合，提供了强大的数据读写能力；
3. 提供了多任务和进程间通信机制。

## What the problems the paper mentioned?
这篇文章主要是对其提出的Unix操作系统中的部分组件（文件系统、进程和shell）进行介绍，Unix证明了一个强大的操作系统可以非常简洁，并且不需要太多的人力物力投入，快速的迭代使其能够不断改进。如何实现对文件的有效管理和高效存储，尽可能简化IO系统调用是首要问题；其次，之前的操作系统大多是批处理系统，难以实现有效的交互，在程序开发过程中对程序运行结果的实时反馈是非常重要的，因此需要提高系统的可操作性和交互的简便性。

## How about the important related works/papers?
对Unix系统的研究有很多，以下列出一部分。
1. Design of the Unix Operating System By Maurice Bach，主要介绍了内核架构、系统组件、系统调用、缓存、等相关内容。
2. A Fast File System for UNIX, MARSHALL K. MCKUSICK，主要描述了一种UNIX文件系统的重新实现。通过使用更灵活的分配策略，重新实现提供了更高的吞吐率，这些分配策略更好地利用了局部性，新文件系统提供两种块大小，以允许快速访问大型文件，同时不会浪费大量空间用于小文件。
3. The Evolution of the Unix Time-sharing System, Dennis M. Ritchie, Bell Laboratories，主要介绍了Unix的起源和发展，进程控制和shell的相关技术，如pipe等。
4. 《深入理解计算机系统》一书中也介绍了Unix的文件系统、进程管理和shell的实现和使用。

## What are some intriguing aspects of the paper?
Unix中最令人感兴趣的地方是其对文件系统的管理。
1. Unix把所有二进制文件、设备等抽象成文件进行管理，对一些特殊的image也可以调用mount进行挂载，使其成为文件系统的一部分进行管理。
2. 对于文件权限的控制和在特殊情况下权限的转移（set-user-ID）实现了安全且灵活的管理。
3. Unix也提供了一种高效的文件组织形式，使用i-node作为文件描述符，记录了文件的一系列信息；设计了一种文件存储形式，在i-node中分配13个地址，前10个作为文件存储块指针，第11个块作为指针数组的指针，以此类推，这样对于小文件可以实现尽量少地硬盘访问。同时实现了对大文件的支持。
4. 维护了一个缓存和预加载机制，提高了访问速度。
5. 当然，shell提供了各种功能，强大且方便的重定向和管道功能极大的提高了工作效率，使我们不必显式进行临时文件的读写；使用&可以在运行其他程序的同时进行其他工作，与nohup配合使用也会提高效率。
6. trap（陷阱）是操作系统的重要属性，在ucore中也有相关的实现，Unix中对trap保证了系统的平稳运行和适当的错误处理。
7. Unix系统初始化创建了一个进程并执行init，init的作用是为每个终端创建一个进程。fork、execute等程序为Unix建立了一套启动程序的机制。

## How to test/compare/analyze the results?
在文章发表的年代正是各种操作系统出现，技术快速更新的时候，对架构的评价可能仁者见仁智者见智，不过可以将Unix与其他操作系统进行对比，展现其易用性；在多核机器上测试可扩展性，对大文件的打开和IO速度进行测试等。

## How can the research be improved?
Unix非常简洁，在此基础上发明的Linux更是成为了科研必备的开发环境，可以不断测试并提高其在多核机器上的可扩展性，发挥机器的应用潜力。对文件系统的组织探索更好的分配调度算法，充分利用时空局部性。

## What’s your test Results about the paper?
本文中的Linux系统是比较早期的版本。ucore中的具体实现与Linux类似，通过进行ucore的实验对文件系统、作业调度、页面置换算法等问题有了更深入的了解。

## If you write this paper, then how would you do?
从标题来讲，首先，可能会对Unix介绍的更加深入，比如它的页面置换算法；可能更希望看到Unix内是如何管理时间片并对其进行调度的。对进程的管理和调度等。在最后添加一些实验过程，将Unix与其他操作系统进行对比实验，运行速度、运行稳定性和可扩展性等，其他的指标主要有是否支持大文件，对不同大小的单个文件的读写效率实验；因为涉及shell的部分各个系统的设计不同，所以暂时无法对比，不过可以通过易用性进行简单比较。

## Give the survey paper list in the same area.
Unix Implementation, Bell System Tech J  

Portability of C Programs and the Unix System, Johnson, S. C., Ritchie, D. M.  

The Evolution of the Unix Time-sharing System

深入理解Linux内核中也有对此的详细讲解