---
title: "Shell"
draft: false
bookToc: false
---


# Shell

什么是shell
-------

现代计算机有越来越多的GUI程序、语音输入程序、甚至VR、AR程序，这些交互方式能解决我们80%的使用场景，但它们还是从根本上限制了我们使用计算机的方式，你没法点击一个不存在的按钮。为了充分利用计算机的能力，我们不得不回到最根本的方式，即**shell**。

所以，shell也是一种交互方式，有很多种类型的shell，例如系统自带的Windows的powershell、Linux的bash(Bourne-Again SHell)，还有开发者编写的zsh(Z Shell)、fish等都比较常见。


使用
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

可以使用`man`程序来查看一些其他shell程序的使用方法和参数，例如`man ls`。

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
我们依然会得到错误的没有权限的提示，这是因为shell中`|`,`>`,`<`各个具体的程序是不知道他们的存在的，echo只知道把内容写入到某个输入流中，具体哪个它不知道，所以我们应该使用:
{{< highlight sh>}}
$ echo 3 | sudo tee brightness
{{< /highlight >}}
这样打开brightness的是tee这个程序，且这个程序是有root权限的，才能正确的修改。