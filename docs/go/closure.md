# 闭包


原理
-------

我们先来看这样一个示例:
```go title="closure.go"
func test() func() {
	x := 100
	return func() {
		println(x)
	}
}
func main() {
	closure := test()
	closure()
}
```
声明了一个test函数，它会返回一个匿名函数，这个匿名函数引用了它的局部变量x。那么当执行完`closure := test()`时，test的栈帧会被销毁，局部变量x应该失效，按照常理，匿名函数应该找不到x而无法打印，但实际上运行之后是能够正常打印的。那么这个x在哪？只有一种可能，它逃逸到堆上去了。既然是在堆上，x就会有一个地址，问题就变为匿名函数是怎么知道x的地址的？

我们把这种现象叫做闭包，当一个匿名函数离开了它的宿主时，它依然持有它所引用的环境变量。所以闭包是包含两个部分的，其一是一个指针指向匿名函数的地址，其二是所拥有的环境变量，这两个东西打包合起来才称为一个闭包。

可以理解为test函数的返回用伪码来表达应该是:
```
return closure struct{
    f : func(){println(x)}
    v : malloc(x)
}
```
只不过Go的语法上让我们可以简写为示例代码中的样子，调用的时候编译器也是隐式的使用`closure.f(closure.v)`这样的方式来调用。


实现
-------

下面我们观察它的编译过程来证明这件事。使用`go build -gcflags "-S" 2>a.txt closure.go`把编译过程放到一个文本文件里，从中能找到一些相关线索:
```asm linenums="1" hl_lines="3 4 12 14 15 18"
"".test STEXT size=126 args=0x8 locals=0x28
	0x0026 00038 (/root/.mac/gocode/closure.go:4)	MOVQ	$100, "".x+16(SP)
	0x002f 00047 (/root/.mac/gocode/closure.go:5)	LEAQ	type.noalg.struct { F uintptr; "".x int }(SB), AX
	0x003a 00058 (/root/.mac/gocode/closure.go:5)	CALL	runtime.newobject(SB)  
	0x0049 00073 (/root/.mac/gocode/closure.go:5)	LEAQ	"".test.func1(SB), CX  
	0x0050 00080 (/root/.mac/gocode/closure.go:5)	MOVQ	CX, (AX)
	0x005a 00090 (/root/.mac/gocode/closure.go:5)	MOVQ	"".x+16(SP), CX
	0x005f 00095 (/root/.mac/gocode/closure.go:5)	MOVQ	CX, 8(AX)			   
	0x0068 00104 (/root/.mac/gocode/closure.go:5)	MOVQ	AX, "".~r0+48(SP)	   

"".main STEXT size=65 args=0x0 locals=0x18
	0x0022 00034 (/root/.mac/gocode/closure.go:11)	MOVQ	(SP), DX               
	0x0026 00038 (/root/.mac/gocode/closure.go:11)	MOVQ	DX, "".closure+8(SP)
	0x002b 00043 (/root/.mac/gocode/closure.go:12)	MOVQ	(DX), AX
	0x002e 00046 (/root/.mac/gocode/closure.go:12)	CALL	AX                     

"".test.func1 STEXT size=84 args=0x0 locals=0x18
	0x001d 00029 (/root/.mac/gocode/closure.go:5)	MOVQ	8(DX), AX
	0x0021 00033 (/root/.mac/gocode/closure.go:5)	MOVQ	AX, "".x+8(SP)
	0x0026 00038 (/root/.mac/gocode/closure.go:6)	CALL	runtime.printlock(SB)
	0x002b 00043 (/root/.mac/gocode/closure.go:6)	MOVQ	"".x+8(SP), AX
	0x0030 00048 (/root/.mac/gocode/closure.go:6)	MOVQ	AX, (SP)
	0x0034 00052 (/root/.mac/gocode/closure.go:6)	CALL	runtime.printint(SB)
```
第3行说明编译期就会根据元数据和类型信息创建好一个结构体，这个结构体就代表了一个闭包，现在在运行期只是把这个早已创建好的结构体的内存地址放到了AX寄存器中；接下来通过第4行依照之前的类型信息在堆上创建了一个对象；接着把函数的地址通过CX转到AX存的地址中，之前已经把x的值100放在`"".x+16(SP)`的位置现在也转到了AX中存的地址+8的位置，也就是说现在在给之前的结构体填充数据。最终把AX中的内容当做返回值返回，`~r0`通常表示返回值，伪码表示test函数实际上返回的就是`return &struct{F:test.func1, x:100}`。

那么在main函数中也就会接收到这样一个地址的返回值。因为test函数没有参数，SP偏移量为0的地址就是其返回值的地址，第12行就说明把test的返回放在了DX中；接着通过第14行把test返回值的地址也就是`test.func1`返回给AX寄存器，并在第15行调用它。

在`test.func1`中，通过第18行从DX中存的地址偏移量+8也就是x的值100放在了AX寄存器中，接着放到栈上执行print。

DX寄存器是整个闭包实现的核心，因为结构体是在堆上运行期动态分配的，它的地址在编译期不知道，闭包中的函数也是编译期就被编译为指令的，那么函数执行时如何知道它的本地变量的内存地址就是问题。DX寄存器是一个名字，编译期、运行期都一样，Go把动态的地址写进去，通过这个静态的名字加上偏移量来交互实现闭包。

应用场景
-------

### 计数器
每次调用返回一个自增的值，以前可能通过类或者全局变量，用函数实现会比较麻烦，有了闭包会比较简单:
```go
func getCounter() func() int {
	x := 0
	return func() int {
		x++
		return x
	}
}

func main() {
	c := getCounter()
	println(c())
	println(c())
	println(c())
}
```
有点类似于C语言中的静态局部变量，可以把数据绑定到函数身上，也就有点像类了。

### 封闭区间
也可以把一份数据绑定到多个函数中去，如下例:
```go
func test() (func(), func(), func()) {
	x := 100
	fmt.Printf("%v\n", &x)
	return func() {
			fmt.Printf("f1.x:%v\n", &x)
		}, func() {
			fmt.Printf("f2.x:%v\n", &x)
		}, func() {
			fmt.Printf("f3.x:%v\n", &x)
		}
}
func main() {
	f1, f2, f3 := test()
	f1()
	f2()
	f3()
}
```
x像是把全局变量的作用域缩小到test函数中，使得里面的三个不同逻辑共享它，构成了一个相对封闭的区间。

### 绑定数据和逻辑
有时候需要A把方法和不同数据都传给B，直接传看上去不优雅，把它们打包为一个闭包看上去就像只传了一个方法，B只是接收一套逻辑，这个逻辑具体是怎么运行B不需要知道。
```go
func hello(db Database) func(http.ResponseWriter, *http.Request) {
	return func(w http.ResponseWriter, r *http.Request) {
		fmt.Fprintln(w, db.Url)
	}
}
func main() {
	db := NewDatabase("localhost:5432")
	http.HandleFunc("/hello", hello(db))
	http.ListenAndServe(":3000", nil)
}
```

### 改变函数签名
例如把一个函数变成偏函数(partial)
```go
func test(x int) {
	println(x)
}

func partial(f func(int), x int) func() {
	return func() {
		f(x)
	}
}

func main() {
	f := partial(test, 100)
	f()
}
```

总结
-------

闭包虽然改善了一些代码上的设计，让代码看上去更干净整洁，可以实现一些设计模式。但它延长了变量的生命周期，必然会导致逃逸行为，其内部的一些交互过程也会使执行性能降低。所以对于闭包也不能滥用。