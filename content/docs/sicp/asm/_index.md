# 汇编语言


基础知识
-------

编程语言是给人看的，CPU看不懂汇编语言，CPU能看懂的只有二进制指令。早期通过纸带打孔的方式输入指令，后来逐渐发展出了助记符，也就是例如跳转、循环之类的指令，再由编译器(汇编器)把这些助记符还原为二进制的指令。所以汇编语言可以理解为机器语言的一种方言，方便人类去记忆它，汇编语言和机器语言就存在着一定程度上的一一对应的关系。

汇编是面向机器编程的语言，不同的硬件的指令可能是不同的，它能直接访问硬件的存储和端口，最大程度发挥出硬件的能力。它还有一些其他的使用场景，例如优化代码追求极致效率、直接调试和修改没有源码的程序、诊断恶意软件、进行逆向分析等。汇编的缺点也显而易见，对多数人而言，汇编代码难懂，不易维护，难于调试，不易移植，开发效率低。

汇编语言的风格分为两种，Intel的和AT&T的，相当于用不同的方言描述机器语言。AT&T风格主要是GNU使用的，汇编器主要是GAS/as;Intel风格主要是windows使用的，汇编器有MASM、NASM、TASM、FASM。它们的主要区别在于AT&T会在寄存器名字前加%，在立即操作数前加$前缀，且源和目标操作数的顺序和Intel相反。详细区别可参考[wiki](https://zh.wikipedia.org/wiki/%E6%B1%87%E7%BC%96%E8%AF%AD%E8%A8%80#%E7%B5%84%E8%AD%AF%E9%A2%A8%E6%A0%BC)，以及[这篇文章](https://developer.ibm.com/articles/l-gas-nasm/)。

本文使用NASM汇编器，它采用Intel语法风格，支持很多种格式，包括ELF、COFF、Mach-O、Win32、Win64等，可使用`nasm -hf`查看其所有支持的格式。


处理过程
-------

Hello world示例程序:
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
{{< /highlight >}}

### 编译
把每一个源码文件(可能是`.c`,`.s`,`.go`等格式)翻译为一个目标文件(可能是`.o`)，它和可执行文件很像，也是由一个个表构成，但问题是A文件若调用B文件的函数，A是不知道B在哪的，编译器就会把这个B的位置空下来，并把这个信息写在表中某个位置等待之后进行重定位。

我们使用编译语句`nasm -g -F dwarf -f elf64 -o hello.o hello.s`来编译之前的helloworld程序。其中`-o hello.o`表示输出的目标文件名称为hello.o，`-f elf64`表示目标文件的格式采取elf64的，`-g`表示要生成调试信息，`-F dwarf`用来指定生成的调试信息的格式。此外可以通过`-O`指定不同的优化级别，可能有O0、O1、O2等，`-E`表示只做预处理。

日常说编译的时候往往指的是编译和链接两个过程。

### 链接
链接就是把编译后的目标文件合并在一起，通过起始地址就能计算出各个目标文件的偏移量，也就知道了编译时需要重定位的那些函数的地址，再把它填进去。

我们使用GNU通用的链接器来链接，也可以对比出目标文件和链接后的可执行文件的区别:
{{< highlight sh "hl_lines=12 15 25 28">}}
[ubuntu] ~/.mac/assem $ nasm -g -f elf64 -o hello.o hello.s
[ubuntu] ~/.mac/assem $ ld -o hello hello.o

[ubuntu] ~/.mac/assem $ file hello.o
hello.o: ELF 64-bit LSB relocatable, x86-64, version 1 (SYSV), not stripped
[ubuntu] ~/.mac/assem $ file hello
hello: ELF 64-bit LSB executable, x86-64, version 1 (SYSV), statically linked, not stripped

[ubuntu] ~/.mac/assem $ objdump -d -M intel hello.o
hello.o:     file format elf64-x86-64
Disassembly of section .text:
0000000000000000 <_start>:
   0:	b8 01 00 00 00       	mov    eax,0x1
   5:	bf 01 00 00 00       	mov    edi,0x1
   a:	48 be 00 00 00 00 00 	movabs rsi,0x0
  11:	00 00 00
  14:	ba 0e 00 00 00       	mov    edx,0xe
  19:	0f 05                	syscall
  1b:	b8 3c 00 00 00       	mov    eax,0x3c
  20:	48 31 ff             	xor    rdi,rdi
  23:	0f 05                	syscall
[ubuntu] ~/.mac/assem $ objdump -d -M intel hello
hello:     file format elf64-x86-64
Disassembly of section .text:
00000000004000b0 <_start>:
  4000b0:	b8 01 00 00 00       	mov    eax,0x1
  4000b5:	bf 01 00 00 00       	mov    edi,0x1
  4000ba:	48 be d8 00 60 00 00 	movabs rsi,0x6000d8
  4000c1:	00 00 00
  4000c4:	ba 0e 00 00 00       	mov    edx,0xe
  4000c9:	0f 05                	syscall
  4000cb:	b8 3c 00 00 00       	mov    eax,0x3c
  4000d0:	48 31 ff             	xor    rdi,rdi
  4000d3:	0f 05                	syscall
{{< /highlight >}}

通过反编译发现目标文件的起始地址_start是0，所以此时的`movabs rsi,0x0`也表示不知道hello的地址，而链接之后这些地址都有了。链接器也支持一些常见的参数，例如`-e`去指定一个非默认的入口标签，`-s`去移除所有的符号信息，`-S`仅移除调试信息。

我们再来看看Go的编译和链接过程:
{{< highlight sh "hl_lines=9 23">}}
[ubuntu] ~/.mac/gocode $ go build -x main.go
WORK=/tmp/go-build182029561
mkdir -p $WORK/b001/
cat >$WORK/b001/importcfg << 'EOF' # internal
# import config
packagefile runtime=/usr/local/go/pkg/linux_amd64/runtime.a
EOF
cd /root/.mac/gocode
/usr/local/go/pkg/tool/linux_amd64/compile -o $WORK/b001/_pkg_.a -trimpath "$WORK/b001=>" -p main -complete -buildid dR1ZsbI6brd59SPrnOhX/dR1ZsbI6brd59SPrnOhX -goversion go1.13 -D _/root/.mac/gocode -importcfg $WORK/b001/importcfg -pack -c=2 ./main.go
/usr/local/go/pkg/tool/linux_amd64/buildid -w $WORK/b001/_pkg_.a # internal
cp $WORK/b001/_pkg_.a /root/.cache/go-build/dd/ddfadd34424d01a7a08b5c1cad7d09610caf9d21c8372338aed2d3331c8802cd-d # internal
cat >$WORK/b001/importcfg.link << 'EOF' # internal
packagefile command-line-arguments=$WORK/b001/_pkg_.a
packagefile runtime=/usr/local/go/pkg/linux_amd64/runtime.a
packagefile internal/bytealg=/usr/local/go/pkg/linux_amd64/internal/bytealg.a
packagefile internal/cpu=/usr/local/go/pkg/linux_amd64/internal/cpu.a
packagefile runtime/internal/atomic=/usr/local/go/pkg/linux_amd64/runtime/internal/atomic.a
packagefile runtime/internal/math=/usr/local/go/pkg/linux_amd64/runtime/internal/math.a
packagefile runtime/internal/sys=/usr/local/go/pkg/linux_amd64/runtime/internal/sys.a
EOF
mkdir -p $WORK/b001/exe/
cd .
/usr/local/go/pkg/tool/linux_amd64/link -o $WORK/b001/exe/a.out -importcfg $WORK/b001/importcfg.link -buildmode=exe -buildid=Nk0qisr5lCLaTQS25eF-/dR1ZsbI6brd59SPrnOhX/NtfWFt1P1m_70TCcgRz3/Nk0qisr5lCLaTQS25eF- -extld=gcc $WORK/b001/_pkg_.a
/usr/local/go/pkg/tool/linux_amd64/buildid -w $WORK/b001/exe/a.out # internal
cp $WORK/b001/exe/a.out main
rm -r $WORK/b001/
{{< /highlight >}}

它也有自己的编译器和链接器，只是它的目标文件通过`-pack`参数进行了打包，打包为一个个`.a`格式再进行链接。


源码结构
-------

汇编语言都是对内存的处理，其本身没有函数的概念，为了便于维护和写跳转之类的语句，就有了标签代表相应的内存地址，放在不同的section中。

### 标签
{{< highlight asm>}}
global _start 

section .text 
    main:
        mov rax, 60
        xor rdi, rdi
        syscall
    _start:
        jmp main 
{{< /highlight >}}

上述源码中`main`与`_start`称为标签，可以使用`jmp`跳转至某个标签，或者`call`调用某个标签的内容。

标签会被编译器翻译为内存地址，我们通过符号表也能看到它。使用global可以把这个标签声明为全局的标签。


### 本地标签
{{< highlight asm>}}
section .text 
    main:
        jmp .hello
    .hello:
        mov rax, 60
        xor rdi, rdi
        syscall
{{< /highlight >}}

以`.`开头的称为本地标签，例如上面的`.hello`，编译器会把它翻译为其前一个标签+本地标签，即`main.hello`，这样就形成了一种类似于名字空间的效果。让大段的逻辑分成多个片段，不需要担心命名上的冲突，属于汇编这门语言提供的一个功能。它和本地符号无关，`main.hello`也可以是一个全局的符号，代表的一种身份，而`.hello`依然是一个本地标签，相当于一个称谓。


### 入口标签
GNU的链接器默认使用一个特殊符号`_start`当做程序的入口，如果我们没定义这样的名字或者把它改为一个其他名字，那么链接器链接的时候就会用一个默认的地址，这个地址可能是程序`.text`段的第一行，同时链接器会提示:`ld: warning: cannot find entry symbol _start; defaulting to 00000000004000b0`。链接器允许使用`ld -e`自行指定一个入口标签，当然这个标签必须是全局的。


### 段标签与内存地址
可以使用`$`表示当前这行指令的内存地址，`$$`表示当前section的起始地址，同时我们也可以通过反汇编观察到跳转时标签和内存地址一一对应的关系:
{{< highlight asm>}}
global _start 
section .text 
    main:
        mov rax, main 
        mov rbx, .exit
        mov rcx, $
        mov rdx, $$
    .exit:    
        mov rax, 60
        xor rdi, rdi
        syscall
    _start:
        jmp main
{{< /highlight >}}

{{< highlight sh>}}
[ubuntu] ~/.mac/assem $ nasm -g -F dwarf -f elf64 -o hello.o hello.s
[ubuntu] ~/.mac/assem $ ld -o hello hello.o
[ubuntu] ~/.mac/assem $ objdump -d -M intel hello
hello:     file format elf64-x86-64
Disassembly of section .text:
0000000000400080 <main>:
  400080:	48 b8 80 00 40 00 00 	movabs rax,0x400080
  400087:	00 00 00
  40008a:	48 bb a8 00 40 00 00 	movabs rbx,0x4000a8
  400091:	00 00 00
  400094:	48 b9 94 00 40 00 00 	movabs rcx,0x400094
  40009b:	00 00 00
  40009e:	48 ba 80 00 40 00 00 	movabs rdx,0x400080
  4000a5:	00 00 00
00000000004000a8 <main.exit>:
  4000a8:	b8 3c 00 00 00       	mov    eax,0x3c
  4000ad:	48 31 ff             	xor    rdi,rdi
  4000b0:	0f 05                	syscall
00000000004000b2 <_start>:
  4000b2:	eb cc                	jmp    400080 <main>
{{< /highlight >}}


语言规范
-------

### 寻址方式
寻址方式就是通过什么方式确定目标所在的地址。我们把寄存器当成一个个的盒子，有以下几种寻址方式:

* 立即寻址，`mov rax, 0x100`，直接把值0x100放在一个盒子中
* 寄存器寻址，`mov rax, rbx`，把一个盒子中的值放在另一个盒子中
* 直接寻址，`mov rax, [5]`，把5#盒子中的值放在另一个盒子中，5代表内存地址
* 寄存器间接寻址，`mov rax,[rbx]`，rbx中存着内存地址，把这个地址对应的值放在盒子中
* 寄存器相对寻址，`mov rax,[8+rbx]`，先算出rbx+8，得到一个内存地址而已
* 基址变址，`lea rax, [rbx+rcx]`，把地址拿出来放到盒子里

`mov rax, 0x100`表示给rax赋值为0x100，`mov rax, [0x100]`表示给rax赋值内存0x100位置的值，`lea rax, [0x100]`表示直接去找[0x100]的地址即0x100，赋值给rax。`mov`是对于内容的操作，`lea`是对于地址的操作。`mov`是不能直接在两块内存之间进行复制的。`[addr]`这种表达方式属于NASM编译器对内存操作必须这么做，即便是一个变量名称（代表一个内存地址），例如`x=100`，也得用`[x]`才取的到它的值100。

### 变量
可以在`.data`和`.bss`段中定义变量，定义示例:
{{< highlight asm>}}
section .data
    x dq 0x8070605040302010    
    y db 1,2,3
    z times 3 dw 1,2                  
    leny equ $-y                ;$为当前地址，所以减去y的地址，就是y的长度

    s db "abcd", "e", "f"       ; db*6
    lens equ $-s                ; equ定义的是常量，不会存储在.data中

section .bss
    xx resq 2 
    yy times 2 resb 8
{{< /highlight >}}
`x`是变量名称，代表符号的地址，变量的起始地址；`dq`代表它的长度；`0x8070605040302010`代表它的初始化值。

字符和它代表的长度如下表所示:

.data中|长度(bytes)|.bss中
---|---|---
db|1|resb
dw|2|resw
dd|4|resd
dq|8|resq
dt|10|rest
do|16|reso
dy|32|resy
dz|64|resz

`y db 1,2,3`就表示从y这个内存地址开始，后面逗号分隔的有三块内存，每块内存都是db大小且值为对应的1、2、3。汇编语言中是没有字符串的，像`"abcd"`就是拿它的ASCII值把它当做字节序列。`s db "abcd", "e", "f"`中db放不下整个`"abcd"`，就用了4个db来放。`equ`是定义常量的方式，见后文描述。另外，数字有大小端和进制的问题。

在`.data`中是定义，而在`.bss`中是预留。`xx resq 2`就表示从`xx`开始，预留`resq`大小的空间，预留`2`组。

`times`类似于语法糖，用于重复定义数据或指令。`yy times 2 resb 8`就表示`yy resb 8`重复两次。

### 常量
常量是不会存在`.data`段中的，尽管我们在那里定义，但它会被编译器展开，变为指令的一部分，保存在`.text`段中。

汇编语言中常量有四种，即整数、浮点数、字符、字符串。整数常量的值有不同的写法:

* 十进制的，100, 0100, 100d, 0100d, 0d100
* 十六进制的，0h64, 0x64, $064, 64h
* 二进制的，0b101, 101b

字符代表的是其ASCII的常量值，支持转义，支持单双和反引号。字符串会被解析为ASCII的序列，从左依次排列。

整型常量通过`equ`定义，例如`leny equ $-y `，它常用来和`$`、`$$`配合计算长度，但不支持用它来定义浮点数。

另一种常见的定义方式是使用`%assign name value`，它可以在任意位置定义，而且可以重复定义，使用时以最后一次的定义为准。这种定义方式属于使用宏，它是编译器在对代码预处理阶段就展开的。