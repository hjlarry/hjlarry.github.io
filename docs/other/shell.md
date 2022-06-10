# Shell

什么是shell
-------

现代计算机有越来越多的GUI程序、语音输入程序、甚至VR、AR程序，这些交互方式能解决我们80%的使用场景，但它们还是从根本上限制了我们使用计算机的方式，你没法点击一个不存在的按钮。为了充分利用计算机的能力，我们不得不回到最根本的方式，即​**shell**。

所以，shell也是一种交互方式，有很多种类型的shell，例如系统自带的Windows的powershell、Linux的bash(Bourne-Again SHell)，还有开发者编写的zsh(Z Shell)、fish等都比较常见。


基础使用
-------

### 基础技巧
`$`表示当前不是root用户，`#`表示当前是root用户。

除了内置的函数，shell通过递归`$PATH`中的目录找到合适的执行程序:
```sh
[ubuntu] / $ echo $PATH
/usr/local/go/bin:/root/go/bin:/usr/local/go/bin:/root/go/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
[ubuntu] / $ which echo
/bin/echo
[ubuntu] / $ /bin/echo $PATH
/usr/local/go/bin:/root/go/bin:/usr/local/go/bin:/root/go/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
```

当传递的参数包含空格时，应该使用单引号或双引号将其包起来，或者使用`\`转义一下，否则会被认为是两个参数:
```sh
[ubuntu] ~ $ mkdir My Photo
[ubuntu] ~ $ ls
My  Photo  go1.13.linux-amd64.tar.gz
[ubuntu] ~ $ mkdir My\ Photo
[ubuntu] ~ $ ls
 My  'My Photo'   Photo   go1.13.linux-amd64.tar.gz
```

`/`表示根目录，`~`表示home目录，`.`表示当前目录，`..`表示上级目录，还可以使用`-`来切换目录:
```sh
[ubuntu] ~ $ cd /tmp
[ubuntu] /tmp $ cd -
/root
[ubuntu] ~ $ cd -
/tmp
```

### 程序间的连接
在shell中，程序都有两个主要的流，即输入流和输出流。当程序需要读取数据时就是从输入流读取，这通常是你的键盘；当程序需要打印信息时，它会将信息放在输出流中，这通常是你的屏幕。然而我们也可以修改默认的输入输出流。

例如，通过`> file`我们可以把输出流的内容放在一个文件中，通过`< file`我们可以从一个文件中读取内容作为输入流:
```sh
[ubuntu] /tmp $ echo 'hello world' > hello.txt
[ubuntu] /tmp $ cat < hello.txt > hello2.txt
[ubuntu] /tmp $ cat hello2.txt
hello world
```
也可以通过`>> file`把输出流的内容追加到文件的尾部，通过`|`把前面一个程序的输出作为后一个程序的输入:
```sh
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
```

### root
在大多数类Unix系统中，`root`用户都是很特殊的用户，它几乎可以不受限制的创建、读取、修改和删除系统中的任意文件。所以我们为避免对系统造成一定的破坏，通常不会使用root用户去登录，取而代之的是在需要的时候使用`sudo`命令。

例如，Linux系统的笔记本的屏幕亮度是写在`/sys/class/backlight`文件中，我们通过修改它可以修改屏幕亮度，这就需要`sudo`，我们可能做这样的尝试:
```sh
$ cd /sys/class/backlight/thinkpad_screen
$ sudo echo 3 > brightness
An error occurred while redirecting file 'brightness'
open: Permission denied
```
我们依然会得到错误的没有权限的提示，这是因为shell中`|`,`>`,`<`，各个具体的程序是不知道他们的存在的，echo只知道把内容写入到某个输入流中，具体哪个它不知道，所以我们应该使用:
```sh
$ echo 3 | sudo tee brightness
```
这样打开brightness的是tee这个程序，且这个程序是有root权限的，才能正确的修改。


编写脚本
-------

大多数shell都有自己的一套脚本语言，包括变量、控制流和自己的语法。本文主要使用`bash`脚本，它是目前应用最广泛的脚本。

### 变量
可以使用`foo=bar`来定义变量foo，但要注意空格。输出变量值时，单引号和双引号也有区别:
```sh
[ubuntu] /tmp/missing $ foo = bar
bash: foo: command not found
[ubuntu] /tmp/missing $ foo=bar
[ubuntu] /tmp/missing $ echo 'this is $foo'
this is $foo
[ubuntu] /tmp/missing $ echo "this is $foo"
this is bar
```

同时，bash中也提供了一些特殊的变量:
* `$0`，脚本名
* `$1`至`$9`，脚本的第几个参数
* `$@`，脚本的所有参数
* `$#`，参数的个数
* `$?`，前一个命令的返回值
* `$$`，返回当前脚本的进程ID(PID)
* `!!`，表示上一条命令，提示权限不足时可以使用`sudo !!`再次执行
* `$_`，上一条命令的最后一个参数

执行一条命令后，通常使用STDOUT返回输出的值，STDERR返回错误及错误码，正常执行的错误码是0，也就是`$?`得到的值。

### 函数
可以在一个脚本中编写函数:
```sh
mcd () {
    mkdir -p "$1"
    cd "$1"
}
```

然后使用source导入就可以使用该函数:
```sh
[ubuntu] /tmp/missing $ source ./test.sh
[ubuntu] /tmp/missing $ mcd tt
[ubuntu] /tmp/missing/tt $
```

### 命令替换
可以通过`$(CMD)`的方式先来执行CMD，执行得到的结果替换掉$(CMD)。例如，`for file in $(ls)`使shell首先调用ls，然后遍历得到的这些返回值。此外，还可以通过`<(CMD)`去先执行CMD，然后将结果输出到一个临时文件中，并将<(CMD)替换成临时文件名。
```sh
[ubuntu] /tmp $ echo "Starting program at $(date)"
Starting program at Wed Jun 17 16:13:34 CST 2020
[ubuntu] /tmp $ diff <(ls go) <(ls missing)
1,18c1,4
< AUTHORS
< CONTRIBUTING.md
...
---
> 3
> semester
> test.sh
> tt
```

### 控制流
我们先来看一个例子:
```sh
#!/bin/bash

echo "Starting program at $(date)" # Date will be substituted

echo "Running program $0 with $# arguments with pid $$"

for file in "$@"; do
    grep foobar "$file" > /dev/null 2> /dev/null
    # When pattern is not found, grep has exit status 1
    # We redirect STDOUT and STDERR to a null register since we do not care about them
    if [[ $? -ne 0 ]]; then
        echo "File $file does not have any foobar, adding one"
        echo "# foobar" >> "$file"
    fi
done
```
例如，运行`./test.sh a b c`，则遍历a、b、c三个文件，分别去寻找`foobar`字符串，如果没找到，错误返回值就是1。接着使用`-ne`来做比较`$?`不等于0，则把`# foobar`追加到这个文件中。

注意，做比较时尽量使用双方括号`[[]]`，而非单方括号，可以降低犯错的几率。具体的原因和其他比较条件可参考[此文章](http://mywiki.wooledge.org/BashFAQ/031){:target="_blank"}。

### 通配符
在shell中，可能经常需要输入一批类似的参数，例如`mkdir foo1 foo2 foo3`等等，如果挨个输入会很不方便，因此有了通配符。

可以分别使用`?`和`*`来匹配一个或任意个字符，花括号`{}`中可以包含一系列的指令。
```sh
[ubuntu] /tmp/missing/tt $ touch foo{1..20}.txt
[ubuntu] /tmp/missing/tt $ ls
foo1.txt   foo11.txt  foo13.txt  foo15.txt  foo17.txt  foo19.txt  foo20.txt  foo4.txt  foo6.txt  foo8.txt
foo10.txt  foo12.txt  foo14.txt  foo16.txt  foo18.txt  foo2.txt   foo3.txt   foo5.txt  foo7.txt  foo9.txt
[ubuntu] /tmp/missing/tt $ rm foo?
rm: cannot remove 'foo?': No such file or directory
[ubuntu] /tmp/missing/tt $ rm foo?.txt
[ubuntu] /tmp/missing/tt $ ls
foo10.txt  foo11.txt  foo12.txt  foo13.txt  foo14.txt  foo15.txt  foo16.txt  foo17.txt  foo18.txt  foo19.txt  foo20.txt
[ubuntu] /tmp/missing/tt $ rm foo*
[ubuntu] /tmp/missing/tt $ ls
[ubuntu] /tmp/missing/tt $ mkdir {foo,bar}123
[ubuntu] /tmp/missing/tt $ ls
bar123  foo123
[ubuntu] /tmp/missing/tt $ touch {foo,bar}123/{a..h}
[ubuntu] /tmp/missing/tt $ ls bar123
a  b  c  d  e  f  g  h
```

### 脚本
脚本也可以使用其他语言编写，只要标记好`shebang`，例如:
```python
#!/usr/local/bin/python
import sys
for arg in reversed(sys.argv[1:]):
    print(arg)
```

写完以后要赋予它权限，才能正确执行:
```sh
[ubuntu] /tmp/missing/tt $ ./test.py
bash: ./test.py: Permission denied
[ubuntu] /tmp/missing/tt $ chmod a+x test.py
[ubuntu] /tmp/missing/tt $ ./test.py a n cs
cs
n
a
```

但是其他电脑的python程序并不一定位于`/usr/local/bin/python`，如何让这个脚本通用呢？我们可以利用环境变量中的程序来解析该脚本，将shebang信息改为`#!/usr/bin/env python`即可。

此外，shell函数和脚本是有一些不同点的:
* 函数只能用shell语言，脚本可以用任意语言，只要脚本中包含shebang就行
* 函数仅在定义时被加载，脚本是在每次执行时加载的。所以函数的加载略快，但每次修改得自己手动重新加载
* 函数会在当前的shell环境中执行，脚本会在单独的进程中执行
* 函数可以对环境变量直接进行修改，脚本则只能使用`export`将环境变量导出

**那么使用`source script.sh`和`./script.sh`执行脚本有什么区别呢？**  
实际上使用source命令是在当前shell的session中执行脚本，该脚本中执行的任何对当前环境的更改都会留存在当前会话中；而如果直接执行./script.sh的话，当前的shell会新建一个新的session，并在该session中执行命令，执行完成后对原session没有影响。


任务控制
-------
shell使用UNIX提供的信号机制来执行进程间的通信。当一个进程接收到信号时，它会停止执行、处理该信号并基于信号传递的信息来改变其执行，所以信号相当于是软件中断。

例如当我们在键盘上输入`Ctrl-C`时，shell会发送一个`SIGINT`信号到进程，来终止这个程序。当然，进程中是可以捕获信号去做别的处理的，例如:
```python title="sigint.py"
#!/usr/bin/env python
import signal, time

def handler(signum, time):
    print("\nI got a SIGINT, but I am not stopping")

signal.signal(signal.SIGINT, handler)
i = 0
while True:
    time.sleep(.1)
    print("\r{}".format(i), end="")
    i += 1
```
此时，我们通过`Ctrl-C`就已经无法终止该程序了:
```sh
$ python sigint.py
24^C
I got a SIGINT, but I am not stopping
26^C
I got a SIGINT, but I am not stopping
30^\[1]    39913 quit       python sigint.py
```
我们可以通过发送其他中断信号来停止该程序，例如`SIGQUIT`或`SIGTERM`等。那么如果一个程序捕获了所有的信号，是不是它就无法被外部停止了？实际上有一个特殊的信号`SIGKILL`是无法被捕获的，也就是我们经常使用的`kill -9 <pid>`发出的信号。

通过`man signal`能查看到信号的更多详细信息，`kill -l`显示出信号列表，然后`kill -<信号名称> <pid>`发送某个信号。

我们可以使用命令加一个后缀`&`来让该命令在后台运行，然后通过`jobs`命令能列出当前shell会话中未完成的全部任务。但后台进程仍然是当前终端会话进程的子进程，一旦关闭终端就会发送一个信号`SIGHUP`导致这些后台的进程被终止掉，命令前加`nohup`就是用来防止这种情况的。
```sh
$ sleep 1000
^Z      # 使用ctrl+Z发送了一个SIGTSTP信号从而暂停了任务
[1]  + 18653 suspended  sleep 1000

$ nohup sleep 2000 &
[2] 18745
appending output to nohup.out

$ jobs
[1]  + suspended  sleep 1000
[2]  - running    nohup sleep 2000

$ bg %1     # 后台接着启动暂停中的1号任务，也可以使用fg来前台启动
[1]  - 18653 continued  sleep 1000

$ jobs
[1]  - running    sleep 1000
[2]  + running    nohup sleep 2000

$ kill -SIGHUP %1   # 也可以使用%任务编号的形式，不一定要用PID
[1]  + 18653 hangup     sleep 1000

$ jobs
[2]  + running    nohup sleep 2000
```


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

```sh
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
```

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

```sh
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
```