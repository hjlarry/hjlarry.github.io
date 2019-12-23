---
title: "软件部分"
date: 2019-11-22T14:59:52+08:00
draft: false
---

# 计算机科学基础知识之软件部分

操作系统
-------

它是计算机体系的内核和基石，管理计算机硬件与软件资源。分为:

* 个人机: Windows, macOS, Linux/BSD
* 大型机: Linux, Unix
* 嵌入式: VxWorks
* 移动端: Android, Windows CE

因为Unix商业版权的原因，必须完全符合UNIX标准才能称为UNIX系统，其他的BSD/Linux只能称为UNIX-like。学术上的操作系统和我们日常口中说的操作系统不太一样，我们一般说的windows严格来说属于一个操作系统产品或者说一个操作系统的发行版，它是系统本身和一些软件（浏览器、扫雷）的打包，而学术上的操作系统指的是内核和一些必要的服务。

现代操作系统的基本功能包括:

* 内存管理（memory management）
* 进程管理（process management）
* 中断处理（interrupt handling）
* 文件系统（file system)
* 安全机制（protection and security）
* 进程通信（inter-process communications）
* 设备驱动（device driver）

内核
-------

微内核就是把最核心、最基础的部分单独作为内核，其他功能围绕它处理。包括地址空间、线程处理、进程通信等。这类东西它们需要以特权模式运行，其他基础服务当做普通应用程序独立运行。

在内核中运行的程序使用的内存，叫内核空间，也叫内核态。其他程序运行在用户空间，也叫用户态。用户态的代码需要在内核态运行时，并不是直接放在内核态中运行的，这样安全没法保证，首先会做权限检查，通过之后相当于提交了申请，内核运行完成以后返回结果唤醒用户态的程序，这就又涉及到了上下文切换和状态保存。所以用户态和内核态的切换会消耗大量的资源。

微内核的优点就是内核很小，裁剪起来方便，裁剪不同的基础服务就可以形成不同的版本面向不同的用户，另外就是其中某个服务崩溃的话不会影响到内核，内核可以重启改服务。当然缺点就是由于处于不同的地址空间，基础服务和内核通信时得使用类似于IPC（Inter-Process Communication）的方式通讯，效率相对会差一些。

微内核的典型代表是学术上的windows和macOS，而不是Linux，早期的Linux为了性能考虑采用了宏内核。

宏内核也叫单一内核，把核心和基础服务放在一个地址空间内均以特权模式运行，好处就是调用一些基础服务的时候相当于函数调用，不需要通讯，性能很高。缺点就是复杂度和耦合度很高，虽然代码是模块化的，但其中某个模块崩溃都可能导致整个系统崩溃，也不方便裁剪和移植。

![kernel](./images/kernel.png)
现代的操作系统往往是采用混合内核的，并不是泾渭分明的。


系统调用
-------
系统调用是内核对外的接口，内核态的一些内核函数。应用程序只要和硬件打交道都会涉及到系统调用，向操作系统申请并等待回复，应该尽量避免或考虑优化，比如有些场景可以使用在用户空间的带buffer的文件替代操作系统提供的文件读写API。

典型的系统调用汇编代码:
{{< highlight asm>}}
global _start 

section .data
    hello : db `hello, world!\n`
section .text 
    _start:
        mov rax, 1      ; system call number should be store in rax
        mov rdi, 1      ; argument #1 in rdi: where to write (descriptor)?
        mov rsi, hello  ; argument #2 in rsi: where does the string start?
        mov rdx, 14     ; argument #3 in rdx: how many bytes to write?
        syscall         ; this instruction invokes a system call
        
        mov rax, 60     ; 'exit' syscall number
        xor rdi, rdi
        syscall
{{< / highlight >}}

相比于call指令， int属于中断指令，需要栈切换并进行相关检查，开销更大一些。


进程
-------

### 基础概念

程序和进程有什么区别？

某个程序里包含了某个进程所需要的数据，但在某个时间点，进程中的内容未必和程序中的内容是一一对应的。我们开发一个程序，不仅包括可执行的代码，还有周边的素材等等，程序运行的时候，也未必会把全部内容都载入到内存。而进程指的是程序运行时，在内存当中受操作系统管理的部分。程序就像是蓝图，我们可以是照着同一份蓝图启动多个进程。

进程不是执行单位，而是一个资源边界。相当于在内存这个世界里面，进程圈了一块地，这块地有很多的属性，比如虚拟地址空间、PID等，而地上的工人、流水线才是执行单位，我们称为线程。所以每个进程都有一个主线程。

* 进程是程序的运行期实例（但不同操作系统对进程定义可能不同）
* 是系统动态执行基本单元
* 是系统资源分配单位
* 是线程的容器
* 是指令、数据及相关资源的集合
* 是程序运行过程的抽象

### 状态

* ready:除分配CPU外，其他准备就绪（按优先级排队调度）
* running:占用处理器资源，正在执行
* waiting:因IO blocking、sync等原因⽆无法继续执⾏

### 调度

采用抢占式调度，CPU分配时间片，时间片结束则强制性切换并保存上下文。另外一种调度模式叫协作式调度，一般是在用户空间实现的。

线程
-------

也称轻量级进程（LWP），是进程中的实际执行单位，由程序计数器、寄存器、栈内存组成。

* 进程拥有一到多个线程
* 线程是调度和时间片的分配单位，更多线程理论上会获得更多的CPU资源
* 线程共享进程的资源
* 同一程序的多个线程是可以在多核处理器上并行执行来提高执行吞吐率的，这个要看操作系统的调度策略

协程只是在用户空间的线程里实现的一种策略，本身和线程有很大的区别。

内核线程和用户线程是有区别的，我们一般使用的线程都是被包装过的，很少通过系统调用来使用线程。用户线程可能是系统线程的包装体，它们是一对一关系；也可能是多个用户线程对应一个系统线程，多个用户线程分享一个时间片，这是多对一关系；还有一种可能是多对多关系，类似于Go的机制，某个时刻一个用户线程必然对应一个系统线程，但总体来看用户线程可能比系统线程更多或者更少。


虚拟存储器
-------

### 基础概念
操作系统会有一个很大的地址空间，也叫虚拟地址空间。这个空间的前面一大段是操作系统用的，之后的每个进程也都会有一个独立的很大的地址空间。这样不同的程序可以使用相同的虚拟地址，当程序在运行时访问这个地址就是访问进程内的虚拟地址，通过内存管理单元（MMU）翻译映射到一个物理地址。这样的好处就是无论在程序内的地址空间怎么折腾都不会影响到其他的程序，对编译器也更友好更方便，可以提前分配。

我们在编译时看到的都是虚拟地址，物理地址在运行期才能看到。虚拟地址空间未必会全部映射到主存中，也会给各个设备保留一段空间。

虚拟存储器可以看成硬盘保存的一个字节数组，虚拟地址空间有256TB，而实际的物理内存可能只有8GB，有这么大的虚拟地址空间就不能阻止程序去使用，不够的物理存储体我们就通过硬盘上的交换分区来弥补。也有可能物理内存加上交换分区都不够程序使用，在Linux中就会引发OOM机制。

### 内存分配过程
假设某个程序需要内存分配器分配10MB的内存，那么操作系统会怎么做呢？首先会划分出一个虚拟地址范围，然后返回起始地址的指针。它此时没有必要通过MMU分配真实的物理内存，因为有可能这段内存后续根本没有读写发生。接下来若写入数据，也只会以页（8KB）为单位一点点的去写，每次写入的时候操作系统再去补物理内存，采用一种按需分配的机会主义原则。具体如何实现呢？当向一个虚拟地址写入数据时，就会去MMU找对应的物理地址，没有找到就说明没有建立映射关系还没有分配物理内存，就会引发一个缺页异常（page fault），操作系统内有专门的程序把这一页补上，来实现按需分配。

我们通过程序可以模拟这种虚拟内存和物理内存的关系:
{{< highlight c>}}
#include <stdio.h>
#include <stdlib.h>

// gcc -g -O0 -o test test.c
// ./test
int main()
{
    const size_t length = 1024 * 1024 * 100;
    const size_t pause = length / 10;

    unsigned char *p = malloc(length); // 分配100MB内存
    unsigned char *x = p;

    for (int i = 0; i < (int)length; i++) // 循环写入数据
    {
        *x = 1;
        x++;
        if (i % (int)pause == 0) // 隔一段时间暂停一下便于观察
        {
            printf("%d\n", i);
            getchar();          // 可以让用户在命令行中随时控制运行进度
        }
    }
    free(p);
    return EXIT_SUCCESS;
}
{{< / highlight >}}

观察运行结果:
{{< highlight sh>}}
[ubuntu] ~/.mac $ pidstat -r -p `pidof test` 1
│Linux 4.9.184-linuxkit (cabd4e519687)   12/06/19        _x86_64_        (2 CPU)│
│11:10:18      UID       PID  minflt/s  majflt/s     VSZ     RSS   %MEM  Command
│11:10:19        0       180      0.00      0.00  106908   21624   1.06  test
│11:10:20        0       180      0.00      0.00  106908   21624   1.06  test
│11:10:21        0       180      0.00      0.00  106908   21624   1.06  test
│11:10:22        0       180   2560.00      0.00  106908   31920   1.56  test
│11:10:23        0       180      0.00      0.00  106908   31920   1.56  test
│11:10:24        0       180   2534.65      0.00  106908   42216   2.06  test
│11:10:25        0       180      0.00      0.00  106908   42216   2.06  test
│11:10:26        0       180   2560.00      0.00  106908   52512   2.57  test
│11:10:27        0       180      0.00      0.00  106908   52512   2.57  test
│11:10:28        0       180      0.00      0.00  106908   52512   2.57  test
│11:10:29        0       180      0.00      0.00  106908   52512   2.57  test
│11:10:30        0       180   2560.00      0.00  106908   62544   3.06  test
{{< / highlight >}}
VSZ表示虚拟内存，RSS表示物理内存，可以看到随用户的控制，数据不断写入，物理内存增加。

### 换入换出
假设这10MB内存已经分配下来，该程序却长时间不用，操作系统就会把这10MB内存的数据保存到硬盘的交换分区上，并对这些内存页的状态做变更，MMU的映射地址做变更，这些内存就可以去给别的程序用，这就叫换出（swap out）。下次重新激活该程序时，会把硬盘上交换分区的数据重新放回某些空闲的页并重新建立映射，这就是换入（swap in）。我们可以通过监控工具观察到系统内的换入换出情况:
{{< highlight sh>}}
[ubuntu] ~ $ dstat
You did not select any stats, using -cdngy by default.
--total-cpu-usage-- -dsk/total- -net/total- ---paging-- ---system--
usr sys idl wai stl| read  writ| recv  send|  in   out | int   csw
  3   4  77  16   0|  12M  101k|   0     0 |   0     0 |1464  2562
  0   0 100   0   0|   0     0 |   0     0 |   0     0 | 175   408
  0   0 100   0   0|   0     0 |   0     0 |   0     0 | 148   345
  0   0 100   0   0|   0     0 |   0     0 |   0     0 | 143   341
  
  // paging的in和out即为当前换入换出大小,int表示当前有多少中断,csw表示当前有多少上下文切换
{{< / highlight >}}
当程序运行需要的内存大于机器的物理内存时，就可能造成频繁的换入换出，产生颠簸效应（thrashing）。

### 性能相关
如果程序需要性能更高、速度要有保障，就需要向操作系统申请锁死这段内存。同时，缺页异常属于内核级别的，有一定的开销，所以为了追求性能的极致，有的C程序会先进行一个初始化操作，缺页异常仍然会有只是会提前，在执行具体的算法时就会更高效不受缺页异常的影响，而一些高级语言可能会由于编译器的优化使得提前初始化写入被优化忽略掉。

物理内存分配还采用了写入时复制（copy-on-write）的机制，即A若引用一块内存，那么复制A到B的时候并不会复制这块内存，只有当去写入A或B的时候才会去复制这块内存，起到节约内存的目的。

对于服务器来讲，假设物理内存有8G，当前运行的程序只有4G，操作系统就会拿另外4G当做自己的cache用，比如缓存文件读写等，程序需要使用时再从cache里还回来。但对于桌面端用户来讲，GUI程序居多，占用内存也多，所以往往会有足够的内存来让用户载入一个程序的速度更快。服务器往往运行的程序数量和时间更长更稳定，所以不同操作系统的分配内存策略也不同。

某种角度上，假设所有数据都交换到硬盘上，我们可以认为所有的数据保存在硬盘上，内存上只保存活跃数据，内存可以看成是硬盘的缓存或者说L4，虚拟存储器就可以看成硬盘上一个巨大的数组。

可执行文件
-------

![ELF](./images/elf.png)  

一个可执行程序看上去像是单个文件的数据库，我们以ELF格式的可执行文件为例，它包括头部元数据、Section header table、Program header table和各个section段。

### Head

文件头部包含一些元数据，用于进程加载找入口地址等:

{{< highlight sh>}}
[ubuntu] ~/.mac/assem $ readelf -h hello
ELF Header:
  Magic:   7f 45 4c 46 02 01 01 00 00 00 00 00 00 00 00 00      // 魔法数，可以快速读取出来用于预判整个文件是不是一个合法的内容
  Class:                             ELF64          // ELF文件的格式
  Data:                              2's complement, little endian  // 大小端情况
  Version:                           1 (current)
  OS/ABI:                            UNIX - System V    // 哪个平台使用
  ABI Version:                       0
  Type:                              EXEC (Executable file)     // 哪种类型，可执行的还是需重定位的等
  Machine:                           Advanced Micro Devices X86-64
  Version:                           0x1
  Entry point address:               0x4000b0       //入口地址
  Start of program headers:          64 (bytes into file)
  Start of section headers:          736 (bytes into file)
  Flags:                             0x0
  Size of this header:               64 (bytes)
  Size of program headers:           56 (bytes)
  Number of program headers:         2
  Size of section headers:           64 (bytes)
  Number of section headers:         8
  Section header string table index: 7
{{< /highlight >}}

### Section

头部之后紧跟着各种各样的表，我们称之为Section(段)，各个section被计入可执行文件的Section header table中，我们可以这样查看:

{{< highlight sh>}}
[ubuntu] ~/.mac/assem $ readelf -S hello
There are 8 section headers, starting at offset 0x2e0:
Section Headers:
  [Nr] Name              Type             Address           Offset
       Size              EntSize          Flags  Link  Info  Align
  [ 0]                   NULL             0000000000000000  00000000
       0000000000000000  0000000000000000           0     0     0
  [ 1] .text             PROGBITS         00000000004000b0  000000b0
       0000000000000025  0000000000000000  AX       0     0     16
  [ 2] .data             PROGBITS         00000000006000d8  000000d8
       000000000000000e  0000000000000000  WA       0     0     4
  [ 3] .stab             PROGBITS         0000000000000000  000000e8
       0000000000000084  0000000000000014           4     0     4
  [ 4] .stabstr          STRTAB           0000000000000000  0000016c
       0000000000000009  0000000000000000           0     0     1
  [ 5] .symtab           SYMTAB           0000000000000000  00000178
       0000000000000108  0000000000000018           6     7     8
  [ 6] .strtab           STRTAB           0000000000000000  00000280
       0000000000000027  0000000000000000           0     0     1
  [ 7] .shstrtab         STRTAB           0000000000000000  000002a7
       0000000000000036  0000000000000000           0     0     1
Key to Flags:
  W (write), A (alloc), X (execute), M (merge), S (strings), I (info),
  L (link order), O (extra OS processing required), G (group), T (TLS),
  C (compressed), x (unknown), o (OS specific), E (exclude),
  l (large), p (processor specific)
{{< /highlight >}}

其中，`Address`表示这个section加载到进程中的虚拟内存地址，`Offset`表示这个section相对于文件头部的偏移量，有了`Offset`+`Size`进程才知道如何安排相应的虚拟地址即`Address`。`Flags`表示权限标志位，`A`表示这个section的内容需要在内存中载入，`X`表示可执行的权限，`W`表示可写入的权限。

`.text`用于存放二进制的执行代码（指令）。我们可以看到这个段的地址恰好是文件头中的Entry point address。

`.data`用于存放所有已经初始化过的全局变量的数据，`.bss`用于存放未初始化的全局变量数据，虽然他们在内存中都需要相应的起始地址和内存区域，但在可执行文件中没必要用一定区域去记下`.bss`中的零值。`.rodata`专门用于保存只读数据，例如字符串的字面量，在很多语言中它都是只读的。

`.symtab`表示符号表，源码中的字面量、全局变量、函数名、类型名、文件名等会作为符号记录在该表中，会记录它们的类型、作用域和内存地址。对于程序运行来讲，符号表没有什么用，CPU不管什么符号、类型，它只需要内存地址。符号就相当于是内存地址的助记符，主要是便于我们调试和反汇编使用的。


### Segment

执行的时候，我们需要告诉操作系统，有哪些section需要载入为内存中的segment，这个信息被链接器放在Program header table中，操作系统的载入器以此信息来安排程序在内存中的样子，我们可以这样查看:

{{< highlight sh>}}
[ubuntu] ~/.mac/assem $ readelf -l hello
Elf file type is EXEC (Executable file)
Entry point 0x4000b0
There are 2 program headers, starting at offset 64
Program Headers:
  Type           Offset             VirtAddr           PhysAddr
                 FileSiz            MemSiz              Flags  Align
  LOAD           0x0000000000000000 0x0000000000400000 0x0000000000400000
                 0x00000000000000d5 0x00000000000000d5  R E    0x200000
  LOAD           0x00000000000000d8 0x00000000006000d8 0x00000000006000d8
                 0x000000000000000e 0x000000000000000e  RW     0x200000
 Section to Segment mapping:
  Segment Sections...
   00     .text
   01     .data
{{< /highlight >}}

上面的section to segment mapping就表示把section中的`.text`段的内容放在内存中`00`号的segment上。在可执行文件中会把section分的很细，但在内存中可能会把一些section合并起来，忽略掉一些section，起到节约内存的目的。