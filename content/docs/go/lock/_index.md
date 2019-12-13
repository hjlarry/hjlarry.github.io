# 锁的原理

锁是怎么实现的？怎么做到独占的？实际上得从原子操作说起。


原子操作
-------

原子操作不是说一定要只能一个动作，而是说把它包装成一条指令或者一个函数的时候，它可以保证不会被侵入。在单核CPU上，一条指令肯定是原子的，但不见得一条指令就只做了一个操作，可能是两个操作，比如`CompareAndSwap`是比较并交换操作数，这在单核上没有问题，但在多核上如何保证不被打断呢？这就需要硬件来支持了。

我们在[intel IA-64手册](https://www.intel.com/content/dam/www/public/us/en/documents/manuals/64-ia-32-architectures-software-developer-instruction-set-reference-manual-325383.pdf)中，可以翻到这样一条指令前缀:
![](images/lock.jpg)

按文档描述，把这条前缀加到某些指令前时，它可以实现一个原子操作。在x86体系中，CPU里专门有一条引线叫HLOCK pin，当遇到有LOCK前缀的指令就拉低该引线电位，从而锁住了总线。那么在同一总线上的其他Core就无法通过总线访问内存了。所以原子操作的本质仍然是锁，只是这个锁是硬件层面的。也不存在软件层面上真正意义的无锁操作，底层仍然是锁。

我们在Go的源码中也能看到大量这样的代码:

{{< highlight asm>}}
// src/runtime/internal/atomic/asm_amd64.s
TEXT runtime∕internal∕atomic·Cas(SB),NOSPLIT,$0-17
	MOVQ	ptr+0(FP), BX
	MOVL	old+8(FP), AX
	MOVL	new+12(FP), CX
	LOCK
	CMPXCHGL	CX, 0(BX)
	SETEQ	ret+16(FP)
	RET
{{< / highlight >}}

随着现在CPU核数逐步增多，这种LOCK前缀锁总线的方式带来的性能问题就凸显了出来，所以现在的体系遇到LOCK前缀时，不是去锁整个总线，而是先检查一下要锁的内容是不是在cache中，如果在的话只锁那一行cacheline，只有同时访问这一个cacheline的其他core才会被锁住，就像数据库的表锁和行锁。这样一来，性能的影响没那么大了，但是原子操作还是会带来性能影响的，只是硬件层面的东西我们作为程序员改变不了什么也就很少去提。

那么原子操作和我们平时所说的互斥锁有什么区别呢？我们谈原子操作是只针对一个操作，而锁是往往是要包含一段逻辑、一个block的。


无锁结构
-------

也就是Lock Free，它的做法是这样的