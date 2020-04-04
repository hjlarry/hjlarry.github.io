---
title: "Python常用工具"
draft: false
---


# Python常用工具

IPython
-------

通过`%quickref`可以查看功能导览，通过`%lsmagic`可以列出ipython中支持的魔法函数:
{{< highlight sh>}}
In [9]: %lsmagic
Out[9]:
Available line magics:
%alias  %alias_magic  %autoawait  %autocall  %autoindent  %automagic  %bookmark  %cat  %cd  %clear
%colors  %conda  %config  %cp  %cpaste  %debug  %dhist  %dirs  %doctest_mode  %ed  %edit  %env  %gui  %hist  %history  %killbgscripts  %ldir  %less  %lf  %lk  %ll  %load  %load_ext  %loadpy  %logoff
%logon  %logstart  %logstate  %logstop  %ls  %lsmagic  %lx  %macro  %magic  %man  %matplotlib  %mkdir  %more  %mv  %notebook  %page  %paste  %pastebin  %pdb  %pdef  %pdoc  %pfile  %pinfo  %pinfo2  %pip  %popd  %pprint  %precision  %prun  %psearch  %psource  %pushd  %pwd  %pycat  %pylab  %quickref  %recall  %rehashx  %reload_ext  %rep  %rerun  %reset  %reset_selective  %rm  %rmdir  %run  %save  %sc  %set_env  %store  %sx  %system  %tb  %time  %timeit  %unalias  %unload_ext  %who  %who_ls  %whos
%xdel  %xmode

Available cell magics:
%%!  %%HTML  %%SVG  %%bash  %%capture  %%debug  %%file  %%html  %%javascript  %%js  %%latex  %%markdown  %%perl  %%prun  %%pypy  %%python  %%python2  %%python3  %%ruby  %%script  %%sh  %%svg  %%sx  %%system  %%time  %%timeit  %%writefile
{{< /highlight >}}
接着可通过给某个魔法函数前加问号的方式例如`?alias`来查看它的具体使用方法。

### 自省
在ipython中，通常使用`?<object>`的方式查看一个对象是什么，当输入某个对象后可以按下tab键查看其成员，通过`%pdoc`可查看其docstring，`%psource`可查看某个目标对象的源码(如果对象是module，需要用`%pfile`)，`%who`和`%whos`查看当前环境中的对象列表。
{{< highlight sh>}}
In [1]: import demo

In [2]: ?demo
Type:        module
String form: <module 'demo' from '/home/hejl/demo.py'>
File:        ~/demo.py
Docstring:   test for doc module

In [3]: %pdoc demo
Class docstring:
    test for doc module

In [4]: %psource demo.b
def b():
    print("helloworld!")

In [5]: %pfile demo
"""
test for doc module
"""

a = 1

def b():
    print("helloworld!")

In [6]: c = 2

In [7]: %who
c        demo

In [8]: %whos
Variable   Type      Data/Info
------------------------------
c          int       2
demo       module    <module 'demo' from '/home/hejl/demo.py'>
{{< /highlight >}}

### 搜索
ipython支持搜索可用的函数，以及当前环境下的对象:
{{< highlight sh>}}
In [1]: import sys

In [2]: ?sys.get*
sys.get_asyncgen_hooks
sys.get_coroutine_wrapper
sys.getallocatedblocks
sys.getcheckinterval
sys.getdefaultencoding
sys.getdlopenflags
sys.getfilesystemencodeerrors
sys.getfilesystemencoding
sys.getprofile
sys.getrecursionlimit
sys.getrefcount
sys.getsizeof
sys.getswitchinterval
sys.gettrace

In [3]: ?sys.get*of
sys.getsizeof

In [4]: ?s*
set
setattr
slice
sorted
staticmethod
str
sum
super
sys
{{< /highlight >}}

### 历史

### 命令