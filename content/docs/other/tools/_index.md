---
title: "常用工具整理"
draft: false
bookcomments: true
---


# 常用工具整理


体检工具
-------

生产环境不同于开发环境，生产环境往往是一个很干净的、只保留了相关依赖的环境，而且需要运行很长的时间。而开发环境属于实验室性质的，类库比较齐全，只跑个单元测试或者benchmark之类的，运行时间很短。所以开发环境中很小的问题比如内存泄漏几KB，在生产环境随着时间的累积或者成千上万的访问，可能会把垃圾回收器搞崩溃。这种情况如果我们不擅于使用工具可能找不到任何问题。

另外，开发环境我们可以使用100%的资源，但生产环境一定要预留一些资源用来做调度、预警之类的。

### dstat
它能实时查看一些基本信息，默认情况下它会包括:

* CPU基本信息(用户使用时间、系统使用时间、空闲时间等)
* 磁盘基本信息(读、写)
* 网络基本信息(接受、发送)
* 换入换出信息
* 系统基本信息(中断的次数、上下文切换的次数)

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
{{< /highlight >}}

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
{{< /highlight >}}
我们把tmux当做是自己写的某个程序，每2秒输出一次信息。minflt/s就代表一些小范围的缺页异常，可能是一些数据不需要了只要补上相应的物理内存即可。而majflt/s代表了需要从硬盘换入内存，这可能就意味着我们程序是不是有的任务优先级太低被操作系统换出到硬盘上了，我们可能需要通过系统调用告诉操作系统把某段内存锁死。

更详细的使用方法，请参考[官方文档](http://sebastien.godard.pagesperso-orange.fr/documentation.html)。

### strace
接着我们可以深入到自己程序的逻辑中去，比如使用strace检查我们程序的系统调用。可能只是一个简单的程序:
{{< highlight go>}}
package main

func main() {
	println("hello world!")
}
{{< /highlight >}}

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

{{< /highlight >}}

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
{{< /highlight >}}


开发工具
-------

GNU通用的开发工具，也叫**binutils**，是一个标准，属于随身带的瑞士军刀，任何语言编写的程序都适用，但可能与语言自带的工具链细节上有些出入。[官方网站](https://www.gnu.org/software/binutils/)

它主要包括:

* readelf : 查看ELF文件信息。
* objdump : 查看⽬标文件 (ELF/COFF) 信息。 
* addr2line : 将地址转换为文件行号信息。
* nm : 查看符号表。
* strip : 删除符号表。
* objcopy : 拷⻉数据到⽬标文件。
* strings : 输出字符串。
* size : 查看各段⼤小。

当我们不熟悉一门语言写的程序时，我们可以把它翻译成汇编语言，在把汇编语言翻译成C语言，也许这个C语言无法运行，但方便了我们阅读和理解程序的思路。

### readelf
查看elf文件的头部信息:
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

查看其执行时需要向内存(进程)中载入哪些信息:
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

查看其本身的section信息:
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

查看其section的内容，例如我们想看其`.data`段的，它是第2个段，可以使用16进制和字符串的方式:
{{< highlight sh>}}
[ubuntu] ~/.mac/assem $ readelf -x 2 hello
Hex dump of section '.data':
  0x006000d8 68656c6c 6f2c2077 6f726c64 210a     hello, world!.
[ubuntu] ~/.mac/assem $ readelf -p 2 hello
String dump of section '.data':
  [     0]  hello, world!
{{< /highlight >}}

查看其符号表信息:
{{< highlight sh>}}
[ubuntu] ~/.mac/assem $ readelf -s hello
Symbol table '.symtab' contains 11 entries:
   Num:    Value          Size Type    Bind   Vis      Ndx Name
     0: 0000000000000000     0 NOTYPE  LOCAL  DEFAULT  UND
     1: 00000000004000b0     0 SECTION LOCAL  DEFAULT    1
     2: 00000000006000d8     0 SECTION LOCAL  DEFAULT    2
     3: 0000000000000000     0 SECTION LOCAL  DEFAULT    3
     4: 0000000000000000     0 SECTION LOCAL  DEFAULT    4
     5: 0000000000000000     0 FILE    LOCAL  DEFAULT  ABS hello.s
     6: 00000000006000d8     1 OBJECT  LOCAL  DEFAULT    2 hello
     7: 00000000004000b0     0 NOTYPE  GLOBAL DEFAULT    1 _start
     8: 00000000006000e6     0 NOTYPE  GLOBAL DEFAULT    2 __bss_start
     9: 00000000006000e6     0 NOTYPE  GLOBAL DEFAULT    2 _edata
    10: 00000000006000e8     0 NOTYPE  GLOBAL DEFAULT    2 _end
{{< /highlight >}}
此外，`-e`代表同时加上`-h -l -S`三个参数。

### nm
使用nm也可查看其符号表信息:

{{< highlight sh>}}
[ubuntu] ~/.mac/assem $ nm hello
00000000006000e6 D __bss_start
00000000006000e6 D _edata
00000000006000e8 D _end
00000000004000b0 T _start
00000000006000d8 d hello
{{< /highlight >}}

其中，`00000000006000e6`表示链接器安排给这个符号的虚拟内存地址；`D`表示它的类型，大写是全局的、可以跨文件访问到的，小写表示本地的。

### strip
使用strip可以剔除其符号表，减少文件本身的大小。默认为`-s`剔除的符号，也可以`-S`仅剔除掉调试用的符号。
{{< highlight sh>}}
[ubuntu] ~/.mac/assem $ strip -S hello
[ubuntu] ~/.mac/assem $ readelf -s hello
Symbol table '.symtab' contains 8 entries:
   Num:    Value          Size Type    Bind   Vis      Ndx Name
     0: 0000000000000000     0 NOTYPE  LOCAL  DEFAULT  UND
     1: 00000000006000d8     1 OBJECT  LOCAL  DEFAULT    2 hello
     2: 00000000004000b0     0 SECTION LOCAL  DEFAULT    1
     3: 00000000006000d8     0 SECTION LOCAL  DEFAULT    2
     4: 00000000004000b0     0 NOTYPE  GLOBAL DEFAULT    1 _start
     5: 00000000006000e6     0 NOTYPE  GLOBAL DEFAULT    2 __bss_start
     6: 00000000006000e6     0 NOTYPE  GLOBAL DEFAULT    2 _edata
     7: 00000000006000e8     0 NOTYPE  GLOBAL DEFAULT    2 _end
[ubuntu] ~/.mac/assem $ strip -s hello
[ubuntu] ~/.mac/assem $ readelf -s hello
[ubuntu] ~/.mac/assem $ nm hello
nm: hello: no symbols
{{< /highlight >}}

### objcopy
可以在可执行文件中自己创建section，藏一些东西，比如背景mp3文件等。我们就可以用到objcopy工具:

{{< highlight sh>}}
[ubuntu] ~/.mac/assem $ objcopy --add-section .abc=addr.c --set-section-flags .abc=noload,readonly hello hello2
[ubuntu] ~/.mac/assem $ readelf -S hello2
There are 5 section headers, starting at offset 0x190:
Section Headers:
  [Nr] Name              Type             Address           Offset
       Size              EntSize          Flags  Link  Info  Align
  [ 0]                   NULL             0000000000000000  00000000
       0000000000000000  0000000000000000           0     0     0
  [ 1] .text             PROGBITS         00000000004000b0  000000b0
       0000000000000025  0000000000000000  AX       0     0     16
  [ 2] .data             PROGBITS         00000000006000d8  000000d8
       000000000000000e  0000000000000000  WA       0     0     4
  [ 3] .abc              PROGBITS         0000000000000000  000000e6
       0000000000000088  0000000000000000           0     0     1
  [ 4] .shstrtab         STRTAB           0000000000000000  0000016e
       000000000000001c  0000000000000000           0     0     1
[ubuntu] ~/.mac/assem $ readelf -p .abc hello2
String dump of section '.abc':
  [     0]  // 查看基址变址的寻址方式^J#include <stdio.h>^J^Jint main(){^J    int x[3];^J    for(int i=0;i<3;i++){^J        x[i]= 0x22;^J
{{< /highlight >}}

我们为可执行文件hello新增了一个`.abc`的段，它的内容读取自`addr.c`文件，它的权限是只读的且无需载入的，并且把它另存为了hello2。此外，我们也可以使用别的文件的内容来更新某个section，或者对section进行重命名、删除等操作:
{{< highlight sh>}}
[ubuntu] ~/.mac/assem $ objcopy --rename-section .abc=.demo hello2
[ubuntu] ~/.mac/assem $ objcopy --update-section .demo=makefile hello2
[ubuntu] ~/.mac/assem $ objcopy --remove-section .demo hello2
{{< /highlight >}}

### objdump
主要用来查看反汇编代码，使用`objdump -d <file>`反汇编某文件`.text`段的内容，默认为att格式，可使用`-M`指定其他格式。
{{< highlight sh>}}
[ubuntu] ~/.mac/assem $ objdump -M intel -d fr

fr:     file format elf64-x86-64


Disassembly of section .text:

00000000004000b0 <_start>:
  4000b0:	b8 01 00 00 00       	mov    eax,0x1
  4000b5:	48 85 c0             	test   rax,rax
  4000b8:	75 1b                	jne    4000d5 <_start.exit>

00000000004000ba <_start.hello>:
  4000ba:	b8 01 00 00 00       	mov    eax,0x1
  4000bf:	bf 01 00 00 00       	mov    edi,0x1
  4000c4:	48 be e0 00 60 00 00 	movabs rsi,0x6000e0
  4000cb:	00 00 00
  4000ce:	ba 0e 00 00 00       	mov    edx,0xe
  4000d3:	0f 05                	syscall

00000000004000d5 <_start.exit>:
  4000d5:	b8 3c 00 00 00       	mov    eax,0x3c
  4000da:	48 31 ff             	xor    rdi,rdi
  4000dd:	0f 05                	syscall
{{< /highlight >}}


调试工具
-------

GNU通用的调试工具，也叫**gdb**。binutils属于静态的观察，而gdb就可以动态的观察到每一个汇编指令的执行。[官方网站](https://www.gnu.org/software/gdb/)

也可以使用一些带图形界面的类似工具，例如[gdbgui](https://www.gdbgui.com/)。还有些[cheatsheet](https://kapeli.com/cheat_sheets/GDB.docset/Contents/Resources/Documents/index)很好用。

常用指令

指令|作用
---|---
info r | 查看寄存器
info f | 查看栈帧
info files | 查看文件的各个段的起始地址
info var | 查看全局变量
info locals | 查看局部变量
info breakpoints | 查看断点信息
bt | 显示程序的栈
set disassembly-flavor intel | 设置反汇编的风格为intel或att
disass | 查看反汇编
layout src | 显示源码窗口
display [var] | 每次停下来时都会输出变量var的值，var可以设为寄存器
l [n] | 查看第N行代码的上下五行
c | 运行至下一断点或程序结束
p/x $[var] | 以16进制输出变量var的值，也可输出表达式，不加`/x`为十进制
x/10xb [addr]| 查看内存地址addr开始的10组数据，16进制的，单位是字节


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
{{< /highlight >}}

对于这段构建代码，`hello:`后的部分就是告诉它要去检查哪些文件的修改时间；`$@`就表示当前这段的目标`hello`，`$<`表示这段的第一个依赖项，`$^`表示这段的所有依赖项；命令前加`-`表示该命令如有错误则忽略；`PHONY`表示没有目标，没有依赖，总是执行规则，在该例中，若恰好文件夹内有一个名为`clean`的文件，没有`PHONY`时则`make clean`不会执行；另外，由于历史原因makefile只能使用tab来缩进，不能使用空格。


日常工具
-------

### find
使用`find 起始目录 参数`可以方便的查找文件，常用的参数为:

* -name，按文件名，-iname也是按文件名但可忽略大小写
* -type，按文件类型，d为目录，f为文件
* -perm，按权限，例如0755
* -user，按所属用户查找
* -group，按所属用户组查找
* -mtime，按修改时间查找，+表示以前，-表示以内，在mac下，`-1d2h`表示1天2小时以内修改过的文件，在linux下只能跟一个数字，例如`-1`表示1天内，如果需要多少分钟内应该用-mmin
* -size，按大小查找，+表示大于，-表示小于，可以跟单位k、M、G、T
* -maxdepth，指定最大搜索深度
* -executable，可执行文件
* -exec，执行相关命令

{{< highlight sh>}}
# 文件名匹配ind*.html的文件
➜  hjlarry.github.io git:(master) ✗ find . -iname ind*.html
./tags/index.html
./docs/sicp/software-introduction/index.html
# 文件修改时间在2小时内的文件
➜  hjlarry.github.io git:(master) ✗ find . -type f -mmin -120 | xargs ls -lh
-rw-rw-rw- 1 hejl hejl   98 May 24 10:12 ./.git/FETCH_HEAD
-rw-r--r-- 1 hejl hejl   41 May 24 10:12 ./.git/ORIG_HEAD
# 文件大小大于2M的文件
➜  hjlarry.github.io git:(master) ✗ find . -type f -size +2M | xargs ls -lh
-r--r--r-- 1 hejl hejl 5.7M Apr  4 17:41 ./.git/modules/themes/book/objects/pack/pack-3a8d2ad7db7fe81a41409e60d27dce22b77b6c68.pack
-r--r--r-- 1 hejl hejl 3.0M Apr 20 14:36 ./.git/objects/pack/pack-472a07bf934d028116265bd13cbea79d367c5e90.pack
{{< /highlight >}}

### grep
使用该命令可以从文件或管道中查找具体的内容，其支持多文件名，支持通配符，内容匹配时支持正则表达式。常用的参数为:

* -r，递归搜索所有子目录
* -l，仅输出文件名
* -n，输出行号
* -c，按文件统计匹配次数
* -v，排除指定内容
* -i，忽略大小写
* -e，多模式匹配，命中其一即可
* -An，输出其后n行内容
* -Bn，输出其前n行内容
* -Cn，输出其前后n行内容

{{< highlight sh>}}
[ubuntu] ~/.mac $ grep -c "main" *
alpine.sh:0
grep: assem: Is a directory
test.c:1
test.go:2
tt:17
[ubuntu] ~/.mac $ grep -n -i "\bmain" *.go
1:package main
5:func main() {
[ubuntu] ~/.mac $ objdump -d -M intel test | grep -A3 "<main.main>"
0000000000452330 <main.main>:
  452330:	64 48 8b 0c 25 f8 ff 	mov    rcx,QWORD PTR fs:0xfffffffffffffff8
  452337:	ff ff
  452339:	48 3b 61 10          	cmp    rsp,QWORD PTR [rcx+0x10]
--
  4523ae:	eb 80                	jmp    452330 <main.main>
{{< /highlight >}}