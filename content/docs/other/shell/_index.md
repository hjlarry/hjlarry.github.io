---
title: "Shell"
draft: false
---


# Shell

什么是shell
-------

现代计算机有越来越多的GUI程序、语音输入程序、甚至VR、AR程序，这些交互方式能解决我们80%的使用场景，但它们还是从根本上限制了我们使用计算机的方式，你没法点击一个不存在的按钮。为了充分利用计算机的能力，我们不得不回到最根本的方式，即**shell**。

所以，shell也是一种交互方式，有很多种类型的shell，例如系统自带的Windows的powershell、Linux的bash(Bourne-Again SHell)，还有开发者编写的zsh(Z Shell)、fish等都比较常见。


基础使用
-------

### 基础技巧
`$`表示当前不是root用户，`#`表示当前是root用户。

除了内置的函数，shell通过递归`$PATH`中的目录找到合适的执行程序:
{{< highlight sh>}}
[ubuntu] / $ echo $PATH
/usr/local/go/bin:/root/go/bin:/usr/local/go/bin:/root/go/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
[ubuntu] / $ which echo
/bin/echo
[ubuntu] / $ /bin/echo $PATH
/usr/local/go/bin:/root/go/bin:/usr/local/go/bin:/root/go/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
{{< /highlight >}}

当传递的参数包含空格时，应该使用单引号或双引号将其包起来，或者使用`\`转义一下，否则会被认为是两个参数:
{{< highlight sh>}}
[ubuntu] ~ $ mkdir My Photo
[ubuntu] ~ $ ls
My  Photo  go1.13.linux-amd64.tar.gz
[ubuntu] ~ $ mkdir My\ Photo
[ubuntu] ~ $ ls
 My  'My Photo'   Photo   go1.13.linux-amd64.tar.gz
{{< /highlight >}}

`/`表示根目录，`~`表示home目录，`.`表示当前目录，`..`表示上级目录，还可以使用`-`来切换目录:
{{< highlight sh>}}
[ubuntu] ~ $ cd /tmp
[ubuntu] /tmp $ cd -
/root
[ubuntu] ~ $ cd -
/tmp
{{< /highlight >}}

### 程序间的连接
在shell中，程序都有两个主要的流，即输入流和输出流。当程序需要读取数据时就是从输入流读取，这通常是你的键盘；当程序需要打印信息时，它会将信息放在输出流中，这通常是你的屏幕。然而我们也可以修改默认的输入输出流。

例如，通过`> file`我们可以把输出流的内容放在一个文件中，通过`< file`我们可以从一个文件中读取内容作为输入流:
{{< highlight sh>}}
[ubuntu] /tmp $ echo 'hello world' > hello.txt
[ubuntu] /tmp $ cat < hello.txt > hello2.txt
[ubuntu] /tmp $ cat hello2.txt
hello world
{{< /highlight >}}
也可以通过`>> file`把输出流的内容追加到文件的尾部，通过`|`把前面一个程序的输出作为后一个程序的输入:
{{< highlight sh>}}
[ubuntu] /tmp $ echo 1234 >> hello2.txt
[ubuntu] /tmp $ cat hello2.txt
hello world
1234
[ubuntu] /tmp $ ls -l | tail -n1
drwx------  2 root root    4096 Dec  6  2019 tmux-0
[ubuntu] /tmp $ ls -l | tail -n3
-rw-r--r--  1 root root       1 Jun 15 10:13 hello3.txt
-rwxr-xr-x  1 root root 1148709 Dec  9  2019 test
drwx------  2 root root    4096 Dec  6  2019 tmux-0
{{< /highlight >}}

### root
在大多数类Unix系统中，`root`用户都是很特殊的用户，它几乎可以不受限制的创建、读取、修改和删除系统中的任意文件。所以我们为避免对系统造成一定的破坏，通常不会使用root用户去登录，取而代之的是在需要的时候使用`sudo`命令。

例如，Linux系统的笔记本的屏幕亮度是写在`/sys/class/backlight`文件中，我们通过修改它可以修改屏幕亮度，这就需要`sudo`，我们可能做这样的尝试:
{{< highlight sh>}}
$ cd /sys/class/backlight/thinkpad_screen
$ sudo echo 3 > brightness
An error occurred while redirecting file 'brightness'
open: Permission denied
{{< /highlight >}}
我们依然会得到错误的没有权限的提示，这是因为shell中`|`,`>`,`<`，各个具体的程序是不知道他们的存在的，echo只知道把内容写入到某个输入流中，具体哪个它不知道，所以我们应该使用:
{{< highlight sh>}}
$ echo 3 | sudo tee brightness
{{< /highlight >}}
这样打开brightness的是tee这个程序，且这个程序是有root权限的，才能正确的修改。


编写脚本
-------


常用工具
-------

可以使用`man`程序来查看一些其他shell程序的使用方法和参数，例如`man ls`。

常用的开发编译相关工具可参考[这篇文章](/docs/other/tools)。

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