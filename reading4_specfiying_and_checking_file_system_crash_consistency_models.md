# Report for 'Specifying and checking file system crash-consistency models'
POSIX文件系统并未定义系统崩溃后文件系统的接口行为，本文提出了“崩溃一致性模型”(crash-consistency model)，用于描述文件系统在系统崩溃后的行为，同时提出了一个框架和一个工具“FERRITE”，根据实际文件系统实现验证这些模型。

## Summary of major innovations
现有的文件系统在系统崩溃后的行为并没有明确定义，用于文件系统性能优化的各种技术和文件系统的分层也为精确描述其行为带来了困难。本文提出的“崩溃一致性模型”(crash-consistency model)可以用来描述其行为。

主要有两个部分，其一是“litmus test”，它是用于显示（表明）文件系统崩溃时的行为。通过检查某些测试是否成立来记录文件系统在不同情况下的行为，有三个部分，初始化、程序体和检查，在程序体中程序会在特定时刻中断，同时提供了mark函数表明某事是否发生，在检查中对事件发生和文件状态进行检查，达到检测文件系统行为的目的。本文中对单/多文件操作、文件夹操作等做了多组测试，验证了不同文件系统对几种行为的支持。

其二是形式化规范“formal specification”，它是使用状态机描述形式化公理和崩溃一致性行为。本文提出了一套用于表示崩溃一致性协议的形式化框架。

主要包括两部分，其一是公理模型（axiomatic），使用一组公理和有序关系描述了崩溃后的合法行为。它包括了一组规则和形式化定义，明确定义了在某个文件系统中事件发生的顺序及对应行为，同时利用形式化语言对某些文件系统的行为进行了描述。其二是操作规范，以非确定性状态机的形式使用理想化元组对文件系统进行抽象，描述其运行行为及状态的改变。

最后描述了FERRITE，它多次运行特定测试以获取所有可能的指令序列，并生成符合其所抽象出的硬盘模型行为的指令重排序列，这样可以方便地模拟在系统崩溃情况先的程序行为。在FERRITE基础上，构建了一个合成器，能够自动在程序中插入fsync并优化。

## What the problems the paper mentioned?
首先，由于IO系统的多层复杂的特性，和IO系统基于效率考虑对指令序列进行的优化，如何判断指令执行顺序和系统崩溃时文件的状态是一个重要问题，以及采用何种方法对IO系统中指令执行顺序进行判断。

其次，在此基础上采用何种方法对文件系统的“崩溃一致性模型”、文件系统的行为和状态进行定义，本文中使用了形式化语言的方法。

最后，如何利用工具对文件系统进行测试，生成合理的指令重排序列并判断其行为，对应用程序和IO调用栈进行改进。

## How about the important related works/papers?
本文中以内存一致性模型进行类比，对内存一致性的研究主要有：
- Memory models: A case for rethinking parallel languages and hardware.本文主要介绍了在并行编程环境下内存读写一致性问题。
- Litmus: Running tests against hardware. 这篇文章介绍了文中使用的Litmus测试，将程序file.litmus转换为C源文件，并将程序封装为测试工具中的内联汇编。，然后，通过gcc将C文件编译为可执行文件，可以在机器上运行以执行检查。本文中也正是使用了Litmus测试对文件系统的特性进行了测试。
- Using Crash Hoare Logic for Certifying the FSCQ File System. FSCQ是第一个具有机器可检查证明（machine-checkable proof ）的文件系统，其实现符合崩溃后系统行为规范。

## What are some intriguing aspects of the paper?
提出了当前文件系统难以进行分析和崩溃后行为描述的原因，即文件系统内部指令重排和IO栈的建立。

使用Litmus test进行测试，在检测主程序中插入了mark函数，可以通过文件内容和marked函数检测文件系统在执行某项操作时崩溃的行为。

不仅使用了形式化语言对文件系统的操作进行了描述和建模，也将文件系统的细节进行了抽象，并描述了其一致性。

基于提出的FERRITE，建立一个合成器，能够在确保要求的前提下插入新的sync调用，保证了安全性。在每次写操作之后插入fsync，确保操作的原子性。

对一个测试多次执行，生成所有符合文件系统模型的指令重排序列，并据此生成所有对应的磁盘状态镜像，在此处结合了QEMU。

使用Dafny建立一个验证框架，用来验证崩溃后的安全性，Dafny是用来更好的写出正确代码的语言，用于验证程序的功能正确性。

对更高层面上的系统支持进行了讨论，包括提供文件系统特性的接口，改进IO栈的构造等。
## How to test/compare/analyze the results?
搭建本文的测试环境需要安装以下工具：

[ferrite的github地址](https://github.com/uwplse/ferrite)，此外需要安装Racket，也是一门语言，可以在Ubuntu上用apt-get直接安装，也可以通过源码安装，在本组服务器上已经安装成功。

此外，安装Dafny，它的github地址：(https://github.com/Microsoft/dafny) ，它是基于mono的，首先需要安装mono

还需要安装QEMU，因为环境不同，需要从源码开始安装。

## How can the research be improved?
其他文章中提到fsync的延迟很大，所以文中的工具在程序中插入fsync可能会造成一定的性能损失；文中对检查崩溃行为及重现崩溃后文件系统的镜像做了描述，在此基础上，可以尝试发展崩溃后自动恢复、修复的工具，完善文件系统在面临故障时的健壮性。

## If you write this paper, then how would you do?
该文从文件系统面临的问题、测试方法、形式化验证表示和文件系统实例测试等几个方面对文件系统崩溃后行为的监视、复现进行了描述，同时在操作系统层面提出了几项支持应用程序利用文件系统崩溃后一致性行为进行优化的建议。更加侧重于形式化语言进行系统建模部分。

## Give the survey paper list in the same area.
Consistency without ordering：针对大部分文件系统使用ordering-point保持系统崩溃时的一致性造成的性能较低问题，采用backpointer技术保持一致性，并证明NoFS在系统崩溃时提供数据一致性; 实验证明NoFS对于某些情况下的崩溃是健壮的，并且可以在一系列工作负载中提供出色的性能。

Generating Litmus Tests for Contrasting Memory Consistency Models：也是通过公理和操作模型来比较内存行为模型的。

Dafny: An Automatic Program Verifier for Functional Correctness：本文介绍了Dafny。Dafny是一种内置规范结构的编程语言。 Dafny静态程序验证程序可用于验证程序的功能正确性。这个网站是Dafny的主页，[Dafny的学习网站](https://www.microsoft.com/en-us/research/project/dafny-a-language-and-program-verifier-for-functional-correctness/?from=http%3A%2F%2Fresearch.microsoft.com%2Fdafny)

All File Systems Are Not Created Equal: On the Complexity of Crafting Crash-Consistent Applications：本文提出了一个在文件系统上构建的应用级崩溃一致性协议，建立了一个名为ALICE的框架，能够分析应用程序更新协议并发现崩溃的漏洞，并使用ALICE分析了11个广泛使用的系统（包括数据库，键值存储，版本控制系统，分布式系统和虚拟化软件）。

Nemos: a framework for axiomatic and executable specifications of memory consistency models：针对内存一致性进行了分析并开发了一个框架Nemo，在文中被称作是“非操作但可执行的内存序列规范”。