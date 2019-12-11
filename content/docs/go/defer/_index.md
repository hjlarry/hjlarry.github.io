---
title: "defer实现方式"
draft: false
bookToc: false
---

# Defer延迟调用

defer是GO语言中独有的特性，在其他语言中可能采用`try...finally`这样的方式来实现，两者还是有很大区别的。我们先来看一个简单的例子:

```
func test(a, b int) {
	println("test: ", a, b)
}
func main() {
	a, b := 0x11, 0x22
	defer test(a, b)
	a++
	b++
	println("main: ", a, b)
}
```

然后反汇编看看它的执行过程:
```
[ubuntu] ~/.mac $ go build test.go
[ubuntu] ~/.mac $ go tool objdump -s "main\.main" test
TEXT main.main(SB) /root/.mac/test.go
  test.go:6		0x4523b0		64488b0c25f8ffffff	MOVQ FS:0xfffffff8, CX
  test.go:6		0x4523b9		483b6110		CMPQ 0x10(CX), SP
  test.go:6		0x4523bd		0f86ad000000		JBE 0x452470
  test.go:6		0x4523c3		4883ec58		SUBQ $0x58, SP
  test.go:6		0x4523c7		48896c2450		MOVQ BP, 0x50(SP)
  test.go:6		0x4523cc		488d6c2450		LEAQ 0x50(SP), BP
  test.go:8		0x4523d1		c744241010000000	MOVL $0x10, 0x10(SP)
  test.go:8		0x4523d9		488d05705a0200		LEAQ 0x25a70(IP), AX
  test.go:8		0x4523e0		4889442428		MOVQ AX, 0x28(SP)
  test.go:8		0x4523e5		48c744244011000000	MOVQ $0x11, 0x40(SP)
  test.go:8		0x4523ee		48c744244822000000	MOVQ $0x22, 0x48(SP)
  test.go:8		0x4523f7		488d442410		LEAQ 0x10(SP), AX
  test.go:8		0x4523fc		48890424		MOVQ AX, 0(SP)
  test.go:8		0x452400		e87b16fdff		CALL runtime.deferprocStack(SB)
  test.go:8		0x452405		85c0			TESTL AX, AX
  test.go:8		0x452407		7557			JNE 0x452460
  test.go:11		0x452409		e8a230fdff		CALL runtime.printlock(SB)
  test.go:11		0x45240e		488d0583130200		LEAQ 0x21383(IP), AX
  test.go:11		0x452415		48890424		MOVQ AX, 0(SP)
  test.go:11		0x452419		48c744240807000000	MOVQ $0x7, 0x8(SP)
  test.go:11		0x452422		e8c939fdff		CALL runtime.printstring(SB)
  test.go:11		0x452427		48c7042412000000	MOVQ $0x12, 0(SP)
  test.go:11		0x45242f		e8fc37fdff		CALL runtime.printint(SB)
  test.go:11		0x452434		e8b732fdff		CALL runtime.printsp(SB)
  test.go:11		0x452439		48c7042423000000	MOVQ $0x23, 0(SP)
  test.go:11		0x452441		e8ea37fdff		CALL runtime.printint(SB)
  test.go:11		0x452446		e8f532fdff		CALL runtime.printnl(SB)
  test.go:11		0x45244b		e8e030fdff		CALL runtime.printunlock(SB)
  test.go:12		0x452450		90			NOPL
  test.go:12		0x452451		e82a1cfdff		CALL runtime.deferreturn(SB)
  test.go:12		0x452456		488b6c2450		MOVQ 0x50(SP), BP
  test.go:12		0x45245b		4883c458		ADDQ $0x58, SP
  test.go:12		0x45245f		c3			RET
  test.go:8		0x452460		90			NOPL
  test.go:8		0x452461		e81a1cfdff		CALL runtime.deferreturn(SB)
  test.go:8		0x452466		488b6c2450		MOVQ 0x50(SP), BP
  test.go:8		0x45246b		4883c458		ADDQ $0x58, SP
  test.go:8		0x45246f		c3			RET
  test.go:6		0x452470		e82b7affff		CALL runtime.morestack_noctxt(SB)
  test.go:6		0x452475		e936ffffff		JMP main.main(SB)
```

我们发现`defer test(a, b)`实际上会被转化为`runtime.deferproc(0x10, test, 0x11, 0x22)`，之后进行的`a++`已经不会影响到它，而在main函数执行完之后，才会去通过`CALL runtime.deferreturn(SB)`调用它。

我们再来看看`runtime.deferproc`干了些什么:

```
func deferproc(siz int32, fn *funcval) {  
	if getg().m.curg != getg() {
		throw("defer on system stack")
	}

	sp := getcallersp()
	argp := uintptr(unsafe.Pointer(&fn)) + unsafe.Sizeof(fn)
	callerpc := getcallerpc()

	d := newdefer(siz)
	if d._panic != nil {
		throw("deferproc: d.panic != nil after newdefer")
	}
	d.fn = fn
	d.pc = callerpc
	d.sp = sp
	switch siz {
	case 0:
		// Do nothing.
	case sys.PtrSize:
		*(*uintptr)(deferArgs(d)) = *(*uintptr)(unsafe.Pointer(argp))
	default:
		memmove(deferArgs(d), unsafe.Pointer(argp), uintptr(siz))
	}
	return0()
}
```