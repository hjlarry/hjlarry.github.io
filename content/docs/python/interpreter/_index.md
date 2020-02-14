# CPython解释器

字节码
-------
Python中的字节码并不能被CPU执行，而是由栈式虚拟机执行，每条指令的背后都对应了一大堆C实现的机器指令。

字节码被存储在代码对象的__code__.co_code中，以一个函数为例:
{{< highlight python>}}
>>> def add(x, y):                                                                                          
...     z = x + y
...     return z

>>> " ".join(str(b) for b in add.__code__.co_code)
'124 0 124 1 23 0 125 2 124 2 83 0'
{{< /highlight >}}
字节码中每两个数字为一组，第一个为指令，第二个为参数，指令对应的二进制数可以在CPython源码中找到:
{{< highlight c>}}
<!-- cpython/Include/opcode.h -->
#define BINARY_ADD               23
#define RETURN_VALUE             83
#define LOAD_FAST               124
#define STORE_FAST              125
{{< /highlight >}}
这就可以和dis的输出结果对应起来:
{{< highlight python>}}
>>> dis.dis(add)
<!-- 源码行    偏移量 指令           参数(目标对象)  -->
  2           0 LOAD_FAST                0 (x)
              2 LOAD_FAST                1 (y)
              4 BINARY_ADD
              6 STORE_FAST               2 (z)

  3           8 LOAD_FAST                2 (z)
             10 RETURN_VALUE
{{< /highlight >}}
指令所对应的源码行这个信息其实保存在__code__的两个相关属性中，co_firstlineno用来存储该段代码起始的行号，co_lnotab由每两个数字一组组成，前一个为字节码偏移的位置，后一个为相对前一组行号的增量。