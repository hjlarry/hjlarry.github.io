---
title: "Git原理和技巧"
draft: false
---

# Git原理和技巧

Git本质上是一个内容寻址(content-addressable)文件系统，并在此之上提供了一个版本控制系统的用户界面。了解底层原理，可以更好的帮助我们理清思路，知道真正在操作什么，而不会迷失在大量的指令和参数上。

原理
-------

### 区域
工作区(Working Directory)，就是我们在电脑里看到的目录，我们对文件进行增、改、删都发生在这个目录中。

在这里目录中，有一个隐藏目录`.git`就是git的版本库(Repository)。

版本库中存了很多东西，其中还有一片区域，一般为`.git/index`文件，我们称为stage(或index)暂存区。

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
tree表示本次提交产生的另一个tree类型git对象的路径地址。author和committer大多数情况相同，但有时比如说我们把别人分支的一个commit通过`cherry-pick`加入我们的分支，这时committer是我们自己，author仍然是别人。这里author需要包含username、email(所以刚使用git时要求在gitconfig中设置用户名和邮箱)、时间戳。紧跟着是提交的message。

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

也就是说，当我们通过`git add *`时会把工作区文件的文件内容生成一个blob object，并把文件名和blob的sha1等其他信息更新到暂存区的index文件中；当我们通过`git commit -m`时，git会根据index信息生成一个tree object，并创建一个新的commit object关联到这个tree。commit object的parent指向了上一个commit，当前分支的指针也会移动到新的commit结点。

#### tag
git中还有一种对象类型就是tag，它通常和commit差不多。见下文[标签引用](#标签引用)。

### 引用
引用本质上就是指向某个commit的指针，它被放在`.gits/refs`目录中。
{{< highlight sh>}}
➜  testgit git:(master) tree .git/refs/
.git/refs/
├── heads
│   └── master
└── tags

2 directories, 1 file
➜  testgit git:(master) cat .git/refs/heads/master
79380b0625150b1b71940cfac0b2a501793eac0c
{{< /highlight >}}

#### 分支引用
当我们新建一个分支时，就会在`.git/refs/heads`文件夹下多一个名称为分支名称的文件，内容就是一个commit的sha。当我们提交一次commit时，当前分支的引用的commit就会自动发生改变。
{{< highlight sh>}}
➜  testgit git:(6c27f42) git branch testbranch
➜  testgit git:(testbranch) cat .git/refs/heads/testbranch
6c27f425aae198e6c1e5098c13d352b3f27edfca
{{< /highlight >}}

#### HEAD引用
它是一个符号引用，指向当前所在的分支。它位于`.git/HEAD`文件。
{{< highlight sh>}}
➜  testgit git:(master) cat .git/HEAD
ref: refs/heads/master
➜  testgit git:(master) git checkout 6c27
Note: checking out '6c27'.
You are in 'detached HEAD' state.
➜  testgit git:(6c27f42) cat .git/HEAD
6c27f425aae198e6c1e5098c13d352b3f27edfca
{{< /highlight >}}
当我们checkout某个commit时，HEAD会指向commit，这时候称为分离头指针(Detached HEAD)状态。

#### 标签引用
我们通常创建的都是轻量级标签，它只是一个引用，位于`.gits/refs/tags`文件夹下。但还可以创建另一种附注标签，它会额外创建一个tag object。
{{< highlight sh>}}
➜  testgit git:(testbranch) git tag testmytag
➜  testgit git:(testbranch) cat .git/refs/tags/testmytag
6c27f425aae198e6c1e5098c13d352b3f27edfca
➜  testgit git:(testbranch) git tag -a v1.0 -m "test my tag"
➜  testgit git:(testbranch) cat .git/refs/tags/v1.0
0ee17848727b21f3086760c6bbc38cc9a12b5e08
➜  testgit git:(testbranch) git cat-file -t 0ee1
tag
➜  testgit git:(testbranch) git cat-file -p 0ee1
object 6c27f425aae198e6c1e5098c13d352b3f27edfca
type commit
tag v1.0
tagger hjlarry <hjlarry@163.com> 1582267013 +0800

test my tag
{{< /highlight >}}
标签和分支的主要区别是:

* 标签可以指向任意object，而分支只能指向commit object。
* 分支在每次提交更新时会自动更新，而标签不会。

#### 远程引用
当我们添加一个远程仓库，或者是从远程库clone过来时，就会有远程引用，它位于`.git/refs/remotes`文件夹下。
{{< highlight sh>}}
➜  testgit git:(testbranch) tree ~/my_git/.git/refs/remotes
/home/hejl/my_git/.git/refs/remotes
└── origin
    ├── HEAD
    └── master

1 directory, 2 files
➜  testgit git:(testbranch) cat ~/my_git/.git/refs/remotes/origin/HEAD
ref: refs/remotes/origin/master
➜  testgit git:(testbranch) cat ~/my_git/.git/refs/remotes/origin/master
94985c6535a5493d493ce89631c7e417f5e7ecaa
{{< /highlight >}}
远程引用和分支之间最主要的区别在于远程引用是只读的。虽然可以checkout到某个远程引用，但是Git并不会将HEAD引用指向该远程引用，这种情况仍然是分离头指针状态。 

常用操作
-------

### 变基

#### 交互式变基
通过`git rebase -i <commit_id>`就能进入一个交互式界面:
{{< highlight sh>}}
pick 2c70e0b second commit a
pick c351a72 third commit

# Rebase 37b766f..c351a72 onto 37b766f (2 commands)
#
# Commands:
# p, pick = use commit
# r, reword = use commit, but edit the commit message
# e, edit = use commit, but stop for amending
# s, squash = use commit, but meld into previous commit
# f, fixup = like "squash", but discard this commit's log message
# x, exec = run command (the rest of the line) using shell
# d, drop = remove commit
#
# These lines can be re-ordered; they are executed from top to bottom.
#
# If you remove a line here THAT COMMIT WILL BE LOST.
#
# However, if you remove everything, the rebase will be aborted.
#
# Note that empty commits are commented out
{{< /highlight >}}
它会列出这个`commit_id`(可以用commit的hash或者`HEAD~3`这样的形式)的child一直到当前分支的最后一个commit，我们可以在这个界面中编辑想对每一个commit做的操作：

* p，对这条commit不做任何变更
* r，使用这条commit，但修改它的commit message
* e，先暂停rebase，把这条commit修改编辑以后再继续，继续时使用`git rebase --continue`即可
* s，使用这条commit，但把它合并入前一条commit
* f, 和s相同，但丢弃掉它的commit message
* x, 在rebase过程中执行一些命令，例如`npm test`之类确保修改不会产生破坏性内容
* d, 移除这条commit

我们在一些场景，例如修改老旧commit的msg、把连续或间隔的多个commit整理为1个等使用交互式变基都会比较方便。

这种方式看起来没有办法修改最祖先的那条commit，实际上有这种场景时我们可以直接把祖先commit添加到头部即可。在交互式界面中也可以手动调整commit的顺序。有些操作例如修改commit msg，就会产生一个新的commit object，当然也会影响到它的所有child object，因为sha值变了，就得一层层的修改下去。


### 合并

### 常用命令

命令|意义
---|---
`git diff` | 比较工作区和暂存区所含文件的差异
`git diff HEAD` | 比较工作区和HEAD所含文件的差异
`git diff --cached` | 比较暂存区和HEAD所含文件的差异
`git diff -- <file>` | 比较工作区和HEAD某个文件的差异
`git reset HEAD` | 暂存区的所有文件恢复的和HEAD一样
`git reset HEAD <file>` | 暂存区某个文件恢复为HEAD那个文件
`git checkout -- <file>` | 工作区某个文件恢复为暂存区那个文件
`git reset --hard <commit>` | 工作区恢复为该commit，回滚和未来均可
`git stash` | 暂存工作现场
`git stash apply` | 恢复工作现场
`git stash apply <index>` | 恢复到某个工作现场
`git branch -av` | 查看所有本地和远程的分支及其对应的commit
`git checkout -b <name> <commit>` | 创建一个分支并切换过去，commit可省略
`git branch --set-upstream A origin/A` | 将本地A分支和远程A分支关联


FAQ
-------
#### 每次commit，Git储存的是全新的文件快照还是储存文件的变更部分？
Git储存的是全新的文件快照，即使你只修改了文件的一行，也会产生一个新的blob对象。这种储存方式存储的对象格式被称为松散(loose)格式。

这样势必会造成大量的空间浪费。但是，Git会时不时的把这些松散格式的文件打包成一个称为包文件(packfile)的二进制文件以节省空间和提升效率。手动执行`git gc`也可以达到这种效果。这也是`./git/objects/info`和`./git/objects/pack`两个文件夹的用途。

#### 为什么要有暂存区的概念，不能直接存储么？
因为提交是需要原子性的，即一次提交下的文件，要么全部成功，要么全部失败。

但在Git的命令行下，我们很难像SVN那样有一个图形界面，勾选要提交的文件，填入提交信息，点个按钮就能完成提交。需要`git add`这个命令帮助我们选择要提交的文件。

相关链接
-------

* [图解git](https://marklodato.github.io/visual-git-guide/index-zh-cn.html)
* [git可视化](http://onlywei.github.io/explain-git-with-d3/#commit)
* [Pro git V2](https://git-scm.com/book/zh/v2)
* [Write yourself a Git!](https://wyag.thb.lt/)
* [Dulwich---Pure-Python Git implementation](https://github.com/dulwich/dulwich)
* [Oh Shit, Git!?!](https://ohshitgit.com/)