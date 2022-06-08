# Python常用工具

IPython
-------

通过`%quickref`可以查看功能导览，通过`%lsmagic`可以列出ipython中支持的魔法函数:
```sh
In [9]: %lsmagic
Out[9]:
Available line magics:
%alias  %alias_magic  %autoawait  %autocall  %autoindent  %automagic  %bookmark  %cat  %cd  %clear
%colors  %conda  %config  %cp  %cpaste  %debug  %dhist  %dirs  %doctest_mode  %ed  %edit  %env  %gui  %hist  %history  %killbgscripts  %ldir  %less  %lf  %lk  %ll  %load  %load_ext  %loadpy  %logoff
%logon  %logstart  %logstate  %logstop  %ls  %lsmagic  %lx  %macro  %magic  %man  %matplotlib  %mkdir  %more  %mv  %notebook  %page  %paste  %pastebin  %pdb  %pdef  %pdoc  %pfile  %pinfo  %pinfo2  %pip  %popd  %pprint  %precision  %prun  %psearch  %psource  %pushd  %pwd  %pycat  %pylab  %quickref  %recall  %rehashx  %reload_ext  %rep  %rerun  %reset  %reset_selective  %rm  %rmdir  %run  %save  %sc  %set_env  %store  %sx  %system  %tb  %time  %timeit  %unalias  %unload_ext  %who  %who_ls  %whos
%xdel  %xmode

Available cell magics:
%%!  %%HTML  %%SVG  %%bash  %%capture  %%debug  %%file  %%html  %%javascript  %%js  %%latex  %%markdown  %%perl  %%prun  %%pypy  %%python  %%python2  %%python3  %%ruby  %%script  %%sh  %%svg  %%sx  %%system  %%time  %%timeit  %%writefile
```
接着可通过给某个魔法函数前加问号的方式例如`?alias`来查看它的具体使用方法。单个`%`表示执行当前行的内容，`%%`用于支持把多行内容作用于该魔法函数。

### 自省
在ipython中，通常使用`?<object>`的方式查看一个对象是什么，当输入某个对象后可以按下tab键查看其成员，通过`%pdoc`可查看其docstring，`%psource`可查看某个目标对象的源码(如果对象是module，需要用`%pfile`)，`%who`和`%whos`查看当前环境中的对象列表。
```sh
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
```

### 搜索
ipython支持搜索可用的函数，以及当前环境下的对象:
```sh
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
```

### 历史
通过`%history -n`可用查看在ipython中全部输入的历史，也可以通过`_i`查看上一个，`_ii`和`_iii`查看上两个和三个，`_i<n>`查看上n个:
```sh
In [6]: _i2
Out[6]: '?sys.get*'

In [7]: %history -n
   1: import sys
   2: ?sys.get*
   3: ?sys.get*of
   4: ?s *
   5: ?s*
   6: _i2
   7: %history -n

In [8]: _i, _ii, _iii
Out[8]: ('%history -n', '_i2', '?s*')
```

还可以把历史命令中的某些行编写为一个宏，通过宏可以重复调用，下例中把第9至第10行、外加第12行编写入了一个宏:
```sh
In [9]: x,y=1,2

In [10]: print(x+y)
3

In [11]: y=10

In [12]: print(x+y)
11

In [13]: %macro tes 9-10 12
Macro `tes` created. To execute, type its name (without quotes).
=== Macro contents: ===
x,y=1,2
print(x+y)
print(x+y)

In [14]: tes
3
3
```

还可以使用`%save`把宏或者某些行保存在一个文件中:
```sh
In [15]: %save test.py tes
The following commands were written to file `test.py`:
x,y=1,2
print(x+y)
print(x+y)

In [16]: %save test.py 9-10
File `test.py` exists. Overwrite (y/[N])?  y
The following commands were written to file `test.py`:
x,y=1,2
print(x+y)
```

### 命令
可以通过`!`执行shell中的命令，命令执行结果可以直接赋值给python变量，也可以将python变量传入shell:
```sh
In [26]: !ls
6.824        basic                 dump.rdb         get-pip.py         my_git       test.py
MIT-6.824    cliu8siteprivate.key  fastapi-vue-cms  helloworld.key     practise-go
__pycache__  demo.py               flask-shop       hjlarry.github.io  practise-py

In [27]: a = !ls

In [28]: a
Out[28]:
['6.824',
 'MIT-6.824',
 '__pycache__',
 ...]

In [31]: x=100

In [32]: !echo "python.x={x+20}"
python.x=120
```


内置模块
-------

### 查看包路径
```sh
~/projects » python3 -m site                                                                         
sys.path = [
    '/Users/hejl/projects',
    '/usr/local/Cellar/python@3.8/3.8.4/Frameworks/Python.framework/Versions/3.8/lib/python38.zip',
    '/usr/local/Cellar/python@3.8/3.8.4/Frameworks/Python.framework/Versions/3.8/lib/python3.8',
    '/usr/local/Cellar/python@3.8/3.8.4/Frameworks/Python.framework/Versions/3.8/lib/python3.8/lib-dynload',
    '/usr/local/lib/python3.8/site-packages',
    '/usr/local/Cellar/protobuf/3.12.3/libexec/lib/python3.8/site-packages',
    '/usr/local/Cellar/python@3.8/3.8.4/Frameworks/Python.framework/Versions/3.8/lib/python3.8/site-packages',
]
USER_BASE: '/Users/hejl/Library/Python/3.8' (doesn't exist)
USER_SITE: '/Users/hejl/Library/Python/3.8/lib/python/site-packages' (doesn't exist)
ENABLE_USER_SITE: True
```

### 格式化json
```sh
~/projects » cat test.json                  
{"_id":"5f12d319624e57e27d1291fe","index":0,"name":"MasseySaunders","gender":"male","company":"TALAE","email":"masseysaunders@talae.com","phone":"+1(853)508-3237","address":"246IndianaPlace,Glenbrook,Iowa,3896","registered":"2017-02-06T06:42:20-08:00","latitude":-10.269827,"longitude":-103.12419,"friends":[{"id":0,"name":"DorotheaShields"},{"id":1,"name":"AnnaRosales"},{"id":2,"name":"GravesBryant"}],"greeting":"Hello,MasseySaunders!Youhave8unreadmessages.","favoriteFruit":"apple"}
------------------------------------------------------------
~/projects » python3 -m json.tool test.json                             
{
    "_id": "5f12d319624e57e27d1291fe",
    "index": 0,
    "name": "MasseySaunders",
    "gender": "male",
    "company": "TALAE",
    "email": "masseysaunders@talae.com",
    "phone": "+1(853)508-3237",
    "address": "246IndianaPlace,Glenbrook,Iowa,3896",
    "registered": "2017-02-06T06:42:20-08:00",
    "latitude": -10.269827,
    "longitude": -103.12419,
    "friends": [
        {
            "id": 0,
            "name": "DorotheaShields"
        },
        {
            "id": 1,
            "name": "AnnaRosales"
        },
        {
            "id": 2,
            "name": "GravesBryant"
        }
    ],
    "greeting": "Hello,MasseySaunders!Youhave8unreadmessages.",
    "favoriteFruit": "apple"
}
```

### 搭建http服务器
```sh
~/projects » python3 -m http.server 8002                                                        
Serving HTTP on :: port 8002 (http://[::]:8002/) ...
```

### 打开py文档
```sh
~/projects » python3 -m pydoc -p 8002                                                            
Server ready at http://localhost:8002/
Server commands: [b]rowser, [q]uit
server>
```

### 直接进入pdb
```sh
~/projects » python3 -m pdb test.py                                                            
> /Users/hejl/projects/test.py(1)<module>()
-> print("hello")
(Pdb) 
```

### 安装包
```sh
$ python3.8 -m pip install requests
```

### 虚拟环境
```sh
~/projects » python3 -m venv .venv 
```