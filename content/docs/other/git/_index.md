---
title: "Git"
draft: false
---

# Git原理和技巧

Git本质上是一个内容寻址(content-addressable)文件系统，并在此之上提供了一个版本控制系统的用户界面。了解底层原理，可以更好的帮助我们理清思路，知道真正在操作什么，而不会迷失在大量的指令和参数上。

原理
-------

### 区域
工作区(Working Directory)，就是我们在电脑里看到的目录，我们对文件进行增、改、删都发生在这个目录中。

在这里目录中，有一个隐藏目录`.git`就是git的版本库(Repository)。

版本库中存了很多东西，其中还有一片区域，我们称为stage(或index)暂存区。

再加上远程仓库，例如github上的repo，我们平时做的大量操作都是把文件在这四个区域之间相互移动。
![area](./images/area.jpg)

### 对象
在git中，除了`init`、`clone`等我们常用的高层命令(被称为porcelain)之外，还有一些底层命令(被称为plumbing)。例如`git hash-object`把一个文件转换为一个git对象，`git cat-file`可以打印出一个git对象。

那么git对象到底是什么呢？因为Git的核心是一个内容寻址文件系统，可以理解为一个简单的键值对数据库。当我们向它插入一个任意类型的内容时，它会返回一个键值对，通过键值对能够在任意时刻再次检索到该内容。

当我们初始化时，`.git`中会有一个文件夹`objects`专门存储git对象:
{{< highlight sh>}}
➜  testgit git init
Initialized empty Git repository in /home/hejl/testgit/.git/

➜  testgit git:(master) ls .git
HEAD  branches  config  description  hooks  info  objects  refs
{{< /highlight >}}

当我们添加一项内容的时候，这个objects文件夹就会多出一个文件:
{{< highlight sh>}}
➜  testgit git:(master) echo 'abc' > test.txt
➜  testgit git:(master) ✗ git add .
➜  testgit git:(master) ✗ tree .git/objects/
.git/objects/
├── 8b
│   └── aef1b4abc478178b004d62031cf7fe6db6f903
├── info
└── pack

3 directories, 1 file
{{< /highlight >}}
它的过程实际上是通过`sha1`算法计算出文件内容也就是`abc`的hash值，是一个40位的16进制的字符串，字符串的前两位作为一个文件夹名称，后38位作为文件名称。因为大多数的文件系统都讨厌单个目录中包含太多的文件，这会减慢其爬取的速度，Git用这个方法使得第一级目录最多256个。文件本身就是Git object，它的头部定义了类型，紧跟着是一个ASCII的空格(0x20)，然后是对象的大小多少字节，紧跟着一个null(0x00)，最后是对象本身的内容。整个文件通过`zlib`压缩存储。

有四种类型的对象，它们分别是`blob`,`commit`,`tree`,`tag`。

#### blob
该种类型的对象存储的原始内容，例如刚才新增的对象就是这种类型:
{{< highlight sh>}}
➜  testgit git:(master) ✗ git cat-file -t 8bae
blob
➜  testgit git:(master) ✗ git cat-file -p 8bae
abc
{{< /highlight >}}
但它也只存储内容，例如工作区中该文件的路径、文件名称、创建时间等都不存储在这种类型的git对象中。

#### commit
当我们进行一次提交操作之后，就会产生两种类型的对象，一个是tree对象稍后再说，另一个就是commit对象。它的内容包括:
{{< highlight sh>}}
➜  testgit git:(master) ✗ git commit -m "first commit"
[master (root-commit) 37b766f] first commit
 1 file changed, 1 insertion(+)
 create mode 100644 test.txt
➜  testgit git:(master) tree .git/objects/
.git/objects/
├── 37
│   └── b766ff8084b9ea2e9a159bdb35b286a3406d3e
├── 8b
│   └── aef1b4abc478178b004d62031cf7fe6db6f903
├── 93
│   └── 4a610d6024005decae12f6ec4c242caa2a4de4
├── info
└── pack

5 directories, 3 files
➜  testgit git:(master) git cat-file -p 37b7
tree 934a610d6024005decae12f6ec4c242caa2a4de4
author hjlarry <hjlarry@163.com> 1582182460 +0800
committer hjlarry <hjlarry@163.com> 1582182460 +0800

first commit
{{< /highlight >}}
tree表示本次提交产生的另一个tree类型git对象的路径地址。author和committer大多数情况相同，但早期一些项目通过电子邮件来进行git，author和committer可以是不同的人，包含了它们的username、email(所以刚使用git时要求在gitconfig中设置用户名和邮箱)、时间戳。紧跟着是提交的message。

实际上完整的commit对象的内容正是邮件的一种简单的格式，源于[RFC2822](https://www.ietf.org/rfc/rfc2822.txt)。还包括一个parent，指向本次提交的父commit对象的路径地址，因为现在是第一次提交，没有parent。以及一个gpgsig，通过PGP签名了一下这个对象，通过`cat-file`看不到。

也就是说我们通过`git log`查看提交日志的信息，就是一层层的递归查看父对象拿到的。

#### tree
树对象解决的是文件名保存的问题，并能够将多个文件组织到一起。我们新建一次提交，添加更多内容为例:
{{< highlight sh>}}
➜  testgit git:(master) echo 'abcdef' > test.txt
➜  testgit git:(master) ✗ mkdir adir
➜  testgit git:(master) ✗ echo '123' > adir/tt.txt
➜  testgit git:(master) ✗ git add .
➜  testgit git:(master) ✗ git commit -m "third commit"
[master 79380b0] third commit
 2 files changed, 2 insertions(+), 1 deletion(-)
 create mode 100644 adir/tt.txt
➜  testgit git:(master) git cat-file -p 1bc4
040000 tree f1af0a5ebfe47de9d2d6db753088462b797b2075    adir
100644 blob 0373d9336f8c8ee90faff225de842888e884a48b    test.txt
➜  testgit git:(master) git cat-file -p f1af
100644 blob 190a18037c64c43e6b11489df4bf0b9eb6d2c9bf    tt.txt
{{< /highlight >}}
它会把当前的目录结构打一个快照，相当于以表格的形式记录该目录下每一个目标的权限、类型(通常是文件blob，或者是文件夹变成一个新的tree对象)、哈希值(也就相当于文件路径)、文件名。

暂存区有一个index，当我们通过`git add *`时会把工作区文件的文件内容生成一个blob object，把文件名和blob的sha1等其他信息更新到这个index中；当我们通过`git commit -m`时，git会根据暂存区的index信息生成一个tree object，并创建一个新的commit object关联到这个tree。commit object的parent指向了上一个commit，当前分支的指针也会移动到新的commit结点。

#### tag
git中还有一种对象类型就是tag，它通常和commit差不多。

### 引用


常用指令
-------

FAQ
-------

相关链接
-------

* [图解git](https://marklodato.github.io/visual-git-guide/index-zh-cn.html)
* [git可视化](http://onlywei.github.io/explain-git-with-d3/#commit)
* [Pro git V2](https://git-scm.com/book/zh/v2)
* [Write yourself a Git!](https://wyag.thb.lt/)
* [Dulwich---Pure-Python Git implementation](https://github.com/dulwich/dulwich)
* [Oh Shit, Git!?!](https://ohshitgit.com/)