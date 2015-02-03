# 进程管理与调度

“进程”（process）是20世纪60年代初首先由MIT的MULTICS系统和IBM公司的CTSS/360系统率先引入的概念。简单地说，进程是一个正在运行的程序。但传统的程序本身是一组指令的集合，是一个静态的概念。程序在一方面无法描述程序在内存中的动态执行情况，即无法从程序的字面上看出它何时执行，何时结束；另一方面，在内存中可存在多个程序，这些程序分时复用一个CPU，但无法清楚地表达程序间关系（比如父子关系、同步互斥关系等）。因此，程序这个静态概念已不能如实反映多程序并发执行过程的特征。

为了从根本上描述程序动态执行过程的性质，计算机科学家引入了“进程（Process）”概念。在计算机系统中，由于CPU的速度非常快（现在的通用CPU主频达到2GHz是很平常的事情），只让它做一件事情无法充分发挥其能力。我们可以“同时”运行多个程序，这个“同时”，其实是操作系统给用户造成的一个“错觉”。大家知道，CPU是计算机系统中的硬件资源。为了提高CPU的利用率，在内存中的多个程序可分时复用CPU，即如果一个程序因某个事件而不能继续执行时，就可把CPU占用权转交给另一个可运行程序。为了刻画多各程序的并发执行的过程，就引入了“进程”的概念。从操作系统原理上看，一个进程是一个具有一定独立功能的程序在一个数据集合上的一次动态执行过程。操作系统中的进程管理需要协调多道程序之间的关系，解决对处理器分配调度策略、分配实施和回收等问题，从而使得处理器资源得到最充分的利用。

操作系统需要管理这些进程，使得这些进程能够公平、合理、高效地分时使用CPU，这需要操作系统完成如下主要任务：

- 进程生命周期管理：创建进程、让进程占用CPU执行、让进程放弃CPU执行、销毁进程；
- 进程分派（dispatch）与调度（scheduling）：设定进程占用/放弃CPU的时机、根据某种策略和算法选择将占用的CPU（这就是调度），完成进程切换；
- 进程内存空间保护：给每个进程一个受到保护的地址空间，确保进程有独立的地址空间，不会被其他进程非法访问；
- 进程内存空间等资源共享：提供内存等资源共享机制，可以使得不同进程可共享内存等资源；
- 系统调用机制：给用户进程提供访问操作系统功能的接口，即系统调用接口。

本章内容主要涉及操作系统的进程管理与调度，并能够利用lab2的虚存管理功能实现高效的进程中的内存管理。读者通过阅读本章的内容并动手实践相关的lab3和lab4种的9个project实验：

- Proj10：创建进程控制块和内核线程。
- Proj10.1：实现用户进程、读和加载ELF格式执行程序、一个简单的调度器，以及提供创建（fork）/execve（执行）/ 放弃对CPU的占用（yield）等系统调用实现。
- Proj10.2：完成等待子进程结束（wait），杀死进程（kill），进程自己退出（exit）等系统调用， 从而完善了进程的生命周期管理。
- Proj10.3：实现sys\_brk系统调用和相应的用户进程内存管理，从而与lab2的内存管理进一步联合在一起。
- Proj10.4：让进程可以睡眠和被唤醒。
- Proj11：创建kswapd内核线程来专门处理内存页的换入和换出。 
- Proj12：基于进程间内存共享（proj9.1）实现父子进程数据共享，并实现了用户态的线程机制。
- Proj13：设计实现了通用的调度器框架
- Proj13.1/Proj13.2：在通用调度器框架下实现了轮转（RoundRobin， RR）调度器/多级反馈队列（Multi Level Feedback Queue， MLFQ）调度器

可以掌握如下知识：

- 与操作系统原理相关
- 进程管理：进程状态和进程状态转换 
- 进程管理：进程创建、进程删除、进程阻塞、进程唤醒
- 进程管理：父子进程的关系和区别 
- 进程管理：进程中的内存管理 
- 进程管理：用户进程、内核进程、用户线程、内核线程的关系区别
- 进程管理：线程的特征和实现机制
- 进程调度：进程调度算法 
- 操作系统原理之外
- 页面换入换出的内核线程实现技术
- 父子进程数据共享实现
- 通用的调度器框架
- 进程切换的实现细节

本章内容中主要涉及进程管理的重要功能主要有两个：

- 进程生命周期的管理：如何高效地创建进程、切换进程、删除进程和管理进程对资源的需求（内存和CPU）涉及一系列的动态管理机制。线程的加入使得整个系统的执行效率更高，这需要根据线程的特点设计与进程不同的线程组织和管理机制。
- 进程调度算法：进程调度（部分教科书也称为处理器调度）算法主要是选择响应时间决定应该由哪个进程占用CPU来执行。这里需要确保通过调度来提高整个系统的吞吐量和减少响应时间。 

为了让读者能够从实践上来理解进程管理和调度的基本原理，我们设计了上述实验，主要的功能逐步实现如下所示：

- 首先需要能够对运行的程序进行管理，这需要一个“档案”，进程控制块（Process Control Block）；
- 有了进程控制块，我们就可以实现不同特点的进程或线程，这里首先实现了相对简单的内核线程；
- 为了能够执行应用程序，还需通过进程控制块实现用户进程的管理，能够创建/删除/阻塞/唤醒进程，从而能够对用户进程的整个生命周期进行全程管理； 
- 由于在内存中有多个进程，但只有一个CPU，所以需要设计合理的调度器，让不同进程能够分时复用CPU，从而提高整个系统的吞吐量和减少响应时间。

