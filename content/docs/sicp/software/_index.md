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
```
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
```

相比于call指令， int属于中断指令，需要栈切换并进行相关检查，开销更大一些。