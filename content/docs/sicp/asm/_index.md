# 汇编语言


基础知识
-------

编程语言是给人看的，CPU看不懂汇编语言，CPU能看懂的只有二进制指令。早期通过纸带打孔的方式输入指令，后来逐渐发展出了助记符，也就是例如跳转、循环之类的指令，再由编译器（汇编器）把这些助记符还原为二进制的指令。所以汇编语言可以理解为机器语言的一种方言，方便人类去记忆它，汇编语言和机器语言就存在着一定程度上的一一对应的关系。

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
{{< / highlight >}}

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