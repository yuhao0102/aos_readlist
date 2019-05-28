# Report for 'Faults in Linux: Ten Years Later'
Chou等人的文章指出drivers文件夹代码中的错误是其他文件夹代码中的7倍。本文通过开发新的错误检查工具发现，虽然Linux系统的规模扩大了两倍，但是其代码的错误率（每行代码的错误个数）却下降了。

## Summary of major innovations
1. 提供了一系列在Linux中可复现的代码检测和错误发现工具，且这个工具是基于开源工具的。
2. 对Chou等人的工作进行检验，证明其提出的错误种类仍然在Linux中存在，并出现了很多针对这些错误的补丁。
3. 证明了虽然错误引入率上升了，但是错误修正率也上升了，这使得Linux内核更加可靠；Linux中在运行时有可见影响的错误生存周期更短。Linux的代码质量不断提升，且driver不再是最大的bug源（从bug率上）。
4. 文章定义了一些新的错误检查工具，Block、Null、Range、Lock等，相比于Chou等人的工作，本文提到的检测工具能够进行更精确的的函数别名分析。
5. 使用了基于控制流的模式检测工具Coccinelle和在版本之间进行错误关联对的Herodotos，引入了SQL，使用SQL对获得的数据进行分析查询。
6. 本文的统计数据不仅仅基于x86架构，而是针对内核的全部代码，这比Chou等人的工作更加细致。
7. 文章通过对Linux2.4.1和Linux2.6分别进行检查，得出结论，Linux的代码量攀升，但是代码的错误率下降，arch文件夹中的代码是错误率最高的代码。当然，文中也针对不同种类的错误进行了比较，并有详细的图表说明结果。通过对代码中错误的生命周期的比较，指出平均寿命大致与发现和修复bug的困难程度相关，且造成明显错误的bug更容易被修复。
8. 使用了三种评价指标来对代码质量进行预测。

## What the problems the paper mentioned?
1. 2001年Chou等人的工作对OS代码中的错误进行了研究，并提供了从大量代码中自动收集错误信息的方法，但是经过多年发展，需要重新评估其工作
2. 对Linux进行大规模代码检测需要实现一个自动检测工具，同时确定确定错误检测的标准和定义。
3. 针对代码分析，确定哪些部分的代码需要被分析，分析得到的bug数据如何进行处理分析，bug的分布情况、生命周期等一系列与代码质量相关的数据。
4. 如何对代码的质量进行预测，需要那几个指标更好地预测，其原理何在。
5. 通过对代码的分析能够得到什么结论，此项结论能够对日后Linux的开发带来哪些帮助。

## How about the important related works/papers?
1. 文中提及的Chou等人的工作是An empirical study of operating system errors，本文是在此项工作的基础上重新开发工具进行检测。
2. 本文中开发的检测工具是基于两个开源工具Coccinelle(v0.2.2)和Herodotos(v0.6.0)。Coccinelle是一个程序匹配和转换引擎，它提供语言SmPL（Semantic Patch Language），用于指定C代码中的所需匹配和转换。Coccinelle也成功地用于查找和修复系统代码中的错误。之前也有很多工作是使用这个工具对Linux和其他OS进行错误分析，如Palix等人。这个工具可以从 http://coccinelle.lip6.fr/ 下载最新版，执行：
    ```
    ./configure 
    make
    ```
3. 文中提到的三个评估预测代码质量的指标，其中之一的code churn是由Munson等人的文中中提出的，另两个指标Chou等人的研究中也有提及。
4. 文中提到的Flawfinder是一款开源的关于C/C++静态扫描分析工具，其根据内部字典数据库进行静态搜索，匹配简单的缺陷与漏洞，flawfinder工具不需要编译C/C++代码，可以直接进行扫描分析。简单快速，不需要编译。在服务器上对这个工具进行了测试。

## What are some intriguing aspects of the paper?
1. 首先，这篇文章中的方法是可重复的，也是基于开源软件的，这样其他研究者就可以很方便的重复其工作。
2. 文章从方法、工具和结果等方面与Chou等人的工作进行了对比，既体现了本文工作的创新性，也可以对比Linux不同版本之间代码bug的分布情况和变化趋势，研究者可以更有针对性的进行修复和开发工作。
3. 文中提到了几个检测工具，Block工具通过对函数指针的分析和对调用图进行分析自动检查Linux中新增的阻塞函数；针对可能返回或者被引用的NULL指针，使用Null和Inull进行检查等。这些工具是工作在源码级别的，因此对架构相关的数据类型不做关心，这提高了代码检测的覆盖度。
4. 文章发现两个版本的代码都是driver文件夹中存在的错误最多，但是driver已经不是错误率最高的。同时发现提交代码数、作者数与系统代码的完善程度相关，如arch和fs的作者数下降表明了其与新架构和新文件系统发展的不匹配。
5. 在文中不仅分析了Chou的结果与本文结果的差别，也对两个版本的Liunx进行了分析，从而得到结论，Linux系统的代码质量在不断提高。
6. 比较了内核中用不同架构下的配置进行编译的源代码文件中的错误数，因为Linux是面向多种架构的，因此这种比较对不同架构下的稳定性具有重要意义，且其确实存在较大的差异。sound文件夹中的代码变化较少，且其中的bug被修复的时间周期也明显较长。
7. 对Linux中一种轻量级同步机制RCU进行了分析。

## How to test/compare/analyze the results?
首先，本文与Chou的工作进行了对比，指明了本文的工作比Chou的工作更加全面，在分析工具和分析统计上更加精确。其次，Linux为开源软件，可以在本地对其工作进行分析，重现其结果。

## If you write this paper, then how would you do?
首先，介绍本文的背景，比如当前Linux内核源码中错误率和错误检测工具的发展，其次，分析前人工作的不足与优点。之后介绍工作中用到或开发的错误检测工具及其原理、对代码中的错误进行分析的方法。最后介绍在哪个Linux版本源码中进行了测试及其测试结果，对结果进行合理分析，指出其对Linux后续开发的指导意义。

## Give the survey paper list in the same area.
1. A. Chou, J. Yang, B. Chelf, S. Hallem, and D. Engler. An empirical study of operating systems errors。这篇文章是本文工作的基础和参考
2. A. Israeli and D. G. Feitelson. The Linux kernel as a case study in
software evolution.
3. J. C. Munson and S. G. Elbaum. Code churn: A measure for estimating
the impact of code change.这篇文章中提出的一个评价代码的指标被用在了本文中。
4. N. Palix, J. L. Lawall, and G. Muller. Herodotos: A tool to expose bugs’ lives. 本文中的工具是基于Coccinelle和Herodotos开发的。
5. N. Palix, S. Saha, G. Thomas, C. Calvès, J. Lawall, and G. Muller.
Database of Faults in Linux: Ten Years Later. 这篇文章是对数据库进行了分析。