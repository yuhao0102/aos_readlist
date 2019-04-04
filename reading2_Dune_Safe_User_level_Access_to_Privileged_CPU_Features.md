# Dune: Safe User-level Access to Privileged CPU Features
## Summary of major innovations
本文介绍了Dune，它为用户的应用程序提供了**直接**且**方便**的访问硬件特性的能力，文中提到的主要有特权控制、访问页表和访问TLB等，它是通过利用现代处理器的虚拟化硬件实现的，以库函数的形式提供了一套能够在普通状态下直接执行特权指令机制。文中提到Dune是以一个进程（process）的形式提供服务，而不是一个机器的抽象。同时，没有像之前的类似工具一样对Linux内核进行大量修改，这保证了操作系统的稳定性和兼容性。Dune利用了VT-x为用户态应用程序提供了对x86保护模式硬件的访问，主要包括：异常、虚存和特权级访问。
## What the problems the paper mentioned?
现在的操作系统一般分为用户态和内核态，用户的应用程序执行在用户态，用户态应用程序如果需要调用某些在内核态才能使用的资源，这时应通过中断或系统调用切换到内核态，系统处于内核态时能够对资源进行有效管理，用户的应用程序一般是不被信任的，这种机制能够有效保护操作系统的安全性。但是这种机制带来的问题是在进行内核态的切换时系统开销巨大，之前的做法有：修改内核，这种行为被证明非常不安全且兼容性不好；将应用放入虚拟镜像，但是不仅难以整合，且Linux一类系统非常复杂，难以修改，文中提到代价太大（**not be worth the hassle**）。

## How about the important related works/papers?
在本文基础上，阅读了相关领域的论文，如：
- KVM: the Linux Virtual Machine Monitor.
- Extensibility Safety and Performance in the SPIN Operating System.

## What are some intriguing aspects of the paper?
- 首先，文中提及Dune的代码量较少，libDune现在只有5898行代码。
- Dune扩展了内核，将内核置于VMX root模式，配置和管理虚拟化硬件，与内核其余部分集成。使用Dune的进程即可在VMX non-root模式下直接访问特权模式。它提供了libDune作为在用户态管理和配置特权指令的接口，libDune的组件包括页表管理、ELF加载、页分配器、管理异常和系统调用机制等，但它不被内核所信任。
- 进程通过初始化一个transition启用Dune，其子进程不必强制处于Dune态，这使进程可以灵活选择是否需要处于Dune态。不仅如此，Dune提供了特权分离，同时不影响原有的安全性。
- 对于内存管理，文中划分了三种页地址转换：
 > host-virtual to host-physical，发生在内核页表中，host-virtual就是我们常说的虚拟地址，不过它只在内核和普通进程中用到；  
 > guest-virtual to guest-physical，发生在Dune进程通过用户页表进行地址转换的过程中；  
 > guest-physical to host-physical，这一步是内核通过EPT(extended page table)做的。
针对某些问题，Dune通过查询内核的内存映射，并手动更新用户的内存地址映射，实现了对内存的管理。并且对内存映射做了一定的限制，只允许特定地址范围映射。

- Dune提供了对某些硬件的访问，比如对分页或浮点硬件的控制，同时限制了一些寄存器的访问。
- 对系统调用，Dune修改了信号处理机制，使用更高效的硬件机制。文中举例了对不可信代码的自动特权级转换。
- 通过Dune实现了沙盒机制，限制运行代码可访问的内存和可调用的系统接口。通过设置监督位、libDune接收异常比in个跳转至异常处理实现了过滤和限制不受信任库的行为。同时也可对系统调用进行过滤， 实现系统调用的认证和参数检查。
- 文中对GC的讨论使我了解了系统内是如何进行GC的，本文也实现了一个基于Dune的GC模型，使用Dune可以方便的查询页表和页的某些位，更准确的判断脏页，以及刷新TLB等。

## How to test/compare/analyze the results?
文中描述了如何对Dune进行测试，包括沙盒测试，特权分离系统Wedge测试和GC。首先，虽然VT-x增加了进出内核态的开销，EPT的使用也使TLB缺失的代价加大，但Dune增加的开销是非常小的。文章通过对三个应用的测试证明了这一点。

在沙盒测试中，通过运行SPEC2000系统和lighthttp的IO性能。SPEC2000在Dune中只比Linux慢了2.9%。针对EPT带来的性能损失，调整策略后Dune沙盒与Linux的性能相差仅为0.1%。对lighthttp的测试也证明由于Dune较高的系统调用开销带来的性能损失而导致运行速度的降低仅为2%。

对Wedge分别测试了线程创建和上下文切换的速度。对比传统的linux下系统，Dune系统在线程创建中提供了40倍的加速，在上下文切换中提供了3倍的性能加速；而Wedge在线程创建中只能提供了12倍的性能加速，并且没有提高上下文切换的速度。在代码量上Dune也有明显优势，只有750LOC。

垃圾回收测试中，文章使用Dune修改了Boehm GC实现了垃圾回收器，共有三个版本，simple，Dune TLB和Dune Dirty三个不同版本，使用了GCbench，Linked list，Hash Map，XMLparser四种基准测试。在大部分测试中Dune都有性能提升，但是在XML测试中性能下降，这可能是因为EPT导致的。

## How can the research be improved?
Dune使用了英特尔VT-x虚拟化架构，提供了应用程序级的对异常、虚存、硬件和特权级的访问，保证了对Linux的兼容，且性能具有较大提升，但是仍有提升空间。

- 文中提到，仍有一些硬件特性尚未涉及，如cache和更多的寄存器，对IO设备的访问，如IOMMU，或者提供DMA的支持以减少IO开销。
- 对目前的超大规模计算和多核计算，Dune仍需不断开发以实现对多核架构的支持。
- Dune支持intel处理器，对广泛应用于移动设备的ARM处理器进行虚拟化扩展是本文作者的未来目标之一。
- 文中提到Dune支持了一些系统调用，对其余系统调用仍需支持，同时也可对某些系统调用进行封装。

## If you write this paper, then how would you do?
本文对Dune的实现基础介绍的非常详细，也介绍了Dune的应用和对应用的测试结果。但是，希望能够增加一些对实现细节的描述，因为在操作系统领域更多的关注底层原理的实现，但这些领域的代码非常抽象，难以阅读。在文中可以引用部分相关代码，并加以解释，使读者更深刻的理解。

## What’s your test Results about the paper?
非常遗憾，使用`git clone https://github.com/project-dune/dune.git`获得了Dune的代码，但是因为机器所限，没有实际运行这些示例，但是阅读了Dune的代码，了解了其内部代码构成。
