# 常用工具整理

体检工具
-------

生产环境不同于开发环境，生产环境往往是一个很干净的、只保留了相关依赖的环境，而且需要运行很长的时间。而开发环境属于实验室性质的，类库比较齐全，只跑个单元测试或者benchmark之类的，运行时间很短。所以开发环境中很小的问题比如内存泄漏几KB，在生产环境随着时间的累积或者成千上万的访问，可能会把垃圾回收器搞崩溃。这种情况如果我们不擅于使用工具可能找不到任何问题。

另外，开发环境我们可以使用100%的资源，但生产环境一定要预留一些资源用来做调度、预警之类的。

### dstat

它能实时查看一些基本信息，默认情况下它会包括:

* CPU基本信息（用户使用时间、系统使用时间、空闲时间等）
* 磁盘基本信息（读、写）
* 网络基本信息（接受、发送）
* 换入换出信息
* 系统基本信息（中断的次数、上下文切换的次数）

如下所示:
{{< highlight sh>}}
[ubuntu] ~/.mac $ dstat
You did not select any stats, using -cdngy by default.
--total-cpu-usage-- -dsk/total- -net/total- ---paging-- ---system--
usr sys idl wai stl| read  writ| recv  send|  in   out | int   csw
  0   0 100   0   0|2797B  597B|   0     0 |   0     0 |  90   236
  0   0 100   0   0|   0     0 |   0     0 |   0     0 | 140   323
  0   0 100   0   0|   0     0 |   0     0 |   0     0 | 163   376
  0   1 100   0   0|   0     0 |   0     0 |   0     0 | 169   391
{{< / highlight >}}

生产环境下我们程序出错的时候，应该先从大的方面入手定位到具体是哪个方面的问题，看看当前的系统环境有没有问题，是不是我们的程序在当前的系统环境下水土不服。比如程序是IO密集型的，系统中有另一个程序在和它抢磁盘资源等。可以通过`dstat --list`查看它能对哪些细分项目查看其情况。[查看更多使用示例](http://dag.wiee.rs/home-made/dstat/)


### sysstat

通过定位了系统中哪个方面的问题之后，接着我们应该去定位我们自己程序哪个方面有问题。sysstat可以根据单个进程去检查它的各个方面，比如说我们定位到是内存方面的问题，接着去看自己程序中内存的问题:
{{< highlight sh>}}
[ubuntu] ~/.mac $ pidstat -r -p `pidof tmux` 2
Linux 4.9.184-linuxkit (cabd4e519687) 	12/09/19 	_x86_64_	(2 CPU)

10:02:38      UID       PID  minflt/s  majflt/s     VSZ     RSS   %MEM  Command
10:02:40        0        35      0.00      0.00   27424    4000   0.20  tmux: server
10:02:42        0        35      0.00      0.00   27424    4000   0.20  tmux: server
10:02:44        0        35      0.00      0.00   27424    4000   0.20  tmux: server
10:02:46        0        35      0.00      0.00   27424    4000   0.20  tmux: server
{{< / highlight >}}
我们把tmux当做是自己写的某个程序，每2秒输出一次信息。minflt/s就代表一些小范围的缺页异常，可能是一些数据不需要了只要补上相应的物理内存即可。而majflt/s代表了需要从硬盘换入内存，这可能就意味着我们程序是不是有的任务优先级太低被操作系统换出到硬盘上了，我们可能需要通过系统调用告诉操作系统把某段内存锁死。

更详细的使用方法，请参考[官方文档](http://sebastien.godard.pagesperso-orange.fr/documentation.html)。

### strace

接着我们可以深入到自己程序的逻辑中去，比如使用strace检查我们程序的系统调用。可能只是一个简单的程序:
{{< highlight go>}}
package main

func main() {
	println("hello world!")
}
{{< / highlight >}}

却涉及到大量的系统调用:
{{< highlight sh>}}
[ubuntu] ~/.mac $ go build test.go
[ubuntu] ~/.mac $ strace ~/.mac/test
execve("/root/.mac/test", ["/root/.mac/test"], 0x7ffec1a0b740 /* 14 vars */) = 0
arch_prctl(ARCH_SET_FS, 0x4c5650)       = 0
sched_getaffinity(0, 8192, [0, 1])      = 16
openat(AT_FDCWD, "/sys/kernel/mm/transparent_hugepage/hpage_pmd_size", O_RDONLY) = -1 ENOENT (No such file or directory)
mmap(NULL, 262144, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0) = 0x7fd246a01000
mmap(0xc000000000, 67108864, PROT_NONE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0) = 0xc000000000
mmap(0xc000000000, 67108864, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_FIXED|MAP_ANONYMOUS, -1, 0) = 0xc000000000
mmap(NULL, 33554432, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0) = 0x7fd244a01000
mmap(NULL, 2164736, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0) = 0x7fd2447f0000
mmap(NULL, 65536, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0) = 0x7fd2447e0000
mmap(NULL, 65536, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0) = 0x7fd2447d0000
rt_sigprocmask(SIG_SETMASK, NULL, [], 8) = 0
sigaltstack(NULL, {ss_sp=NULL, ss_flags=SS_DISABLE, ss_size=0}) = 0
sigaltstack({ss_sp=0xc000002000, ss_flags=0, ss_size=32768}, NULL) = 0
rt_sigprocmask(SIG_SETMASK, [], NULL, 8) = 0
gettid()                                = 394
rt_sigaction(SIGHUP, NULL, {sa_handler=SIG_DFL, sa_mask=[], sa_flags=0}, 8) = 0
rt_sigaction(SIGHUP, {sa_handler=0x44d8f0, sa_mask=~[], sa_flags=SA_RESTORER|SA_ONSTACK|SA_RESTART|SA_SIGINFO, sa_restorer=0x44da20}, NULL, 8) = 0
rt_sigaction(SIGINT, NULL, {sa_handler=SIG_DFL, sa_mask=[], sa_flags=0}, 8) = 0
rt_sigaction(SIGINT, {sa_handler=0x44d8f0, sa_mask=~[], sa_flags=SA_RESTORER|SA_ONSTACK|SA_RESTART|SA_SIGINFO, sa_restorer=0x44da20}, NULL, 8) = 0

......

{{< / highlight >}}

打印输入输出必然会涉及系统调用，但如果我们使用一些第三方库时发现系统调用仍然很多，就可以去查找有没有优化替代的方案。

一般，我们查看一些摘要信息即可:
{{< highlight sh>}}
[ubuntu] ~/.mac $ strace -c ~/.mac/test
hello world!
% time     seconds  usecs/call     calls    errors syscall
------ ----------- ----------- --------- --------- ----------------
  0.00    0.000000           0         1           write
  0.00    0.000000           0         7           mmap
  0.00    0.000000           0       114           rt_sigaction
  0.00    0.000000           0         6           rt_sigprocmask
  0.00    0.000000           0         2           clone
  0.00    0.000000           0         1           execve
  0.00    0.000000           0         2           sigaltstack
  0.00    0.000000           0         1           arch_prctl
  0.00    0.000000           0         1           gettid
  0.00    0.000000           0         3           futex
  0.00    0.000000           0         1           sched_getaffinity
  0.00    0.000000           0         1         1 openat
------ ----------- ----------- --------- --------- ----------------
100.00    0.000000                   140         1 total
{{< / highlight >}}

开发工具
-------

GNU通用的开发工具，也叫**binutils**，是一个标准，属于随身带的瑞士军刀，任何语言编写的程序都适用，但可能与语言自带的工具链细节上有些出入。[官方网站](https://www.gnu.org/software/binutils/)

它主要包括:

* readelf : 查看ELF文件信息。
* objdump : 查看⽬标文件 (ELF/COFF) 信息。 
* addr2line : 将地址转换为文件行号信息。
* nm : 查看符号表。
* strip : 删除符号表。
* objcopy : 拷⻉数据到⽬标⽂文件。
* strings : 输出字符串。
* size : 查看各段⼤小。

当我们不熟悉一门语言写的程序时，我们可以把它翻译成汇编语言，在把汇编语言翻译成C语言，也许这个C语言无法运行，但方便了我们阅读和理解程序的思路。

调试工具
-------

GNU通用的调试工具，也叫**gdb**。binutils属于静态的观察，而gdb就可以动态的观察到每一个汇编指令的执行。[官方网站](https://www.gnu.org/software/gdb/)

也可以使用一些带图形界面的类似工具，例如[gdbgui](https://www.gdbgui.com/)。还有些[cheatsheet](https://kapeli.com/cheat_sheets/GDB.docset/Contents/Resources/Documents/index)很好用。


构建工具
-------

GNU的自动构建工具为**make**，有些像脚本语言，把一堆命令放一起批量执行。使用它做编译属于增量编译，即通过对比修改时间来判断是否需要重新执行。[官方网站](https://www.gnu.org/software/make/)

{{< highlight make>}}
hello: hello.s
    nasm -g -f elf64 -o hello.o hello.s
    ld -o $@ hello.o
    
clean:
    -rm *.o
    -rm hello
    
.PHONY: clean
{{< / highlight >}}

对于这段构建代码，`hello:`后的部分就是告诉它要去检查哪些文件的修改时间；`$@`就表示当前这段的目标`hello`，`$<`表示这段的第一个依赖项，`$^`表示这段的所有依赖项；命令前加`-`表示该命令如有错误则忽略；`PHONY`表示没有目标，没有依赖，总是执行规则，在该例中，若恰好文件夹内有一个名为`clean`的文件，没有`PHONY`时则`make clean`不会执行；另外，由于历史原因makefile只能使用tab来缩进，不能使用空格。