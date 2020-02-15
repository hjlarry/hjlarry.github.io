# CPython解释器

字节码
-------
Python中的字节码并不能被CPU执行，而是由栈式虚拟机执行，每条指令的背后都对应了一大堆C实现的机器指令。

字节码被存储在代码对象(即`__code__`)的`co_code`中，以一个函数为例:
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
指令所对应的源码行这个信息其实保存在代码对象的两个相关属性中，`co_firstlineno`用来存储该段代码起始的行号，`co_lnotab`由每两个数字一组组成，前一个为字节码偏移的位置，后一个为相对前一组行号的增量。每条字节码指令代表的意义可通过官方文档[此处](https://docs.python.org/3/library/dis.html)查询到。


GIL
-------
全局解释器锁机制使得解释器在同一时刻仅有一个线程可以被调度执行，某个线程若想要执行，就先要拿到GIL，但在每个Python进程中，只有一个GIL。它的存在使得解释器本身的实现更简单一些，更容易实现对象的安全访问，便于进行内存管理和编写扩展。但在多核环境下无法实现并行，对于以多线程为基础的并发应用就是一个灾难。

对于IO密集型任务，线程是在发生阻塞时主动释放GIL的，让其他线程得以执行。而对于CPU密集型任务，采取超时策略。

当GIL被其他线程占用时，等待线程会阻塞一段时间。如果超时（默认为0.005秒）后，依然无法获取锁，则发出请求。这种请求设计的很轻巧，就是一个全局条件变量设置。正在执行的线程在解释循环内会检查该标记，然后释放锁，切换线程执行，其自身进入等待状态。属于典型的协作机制。相关源码:
{{< highlight c>}}
<!-- cpython/Python/ceval.c -->
main_loop:
    for (;;) {
        <!-- ... -->
        if (_Py_atomic_load_relaxed(eval_breaker)) {
            opcode = _Py_OPCODE(*next_instr);
            if (_Py_atomic_load_relaxed(&ceval->gil_drop_request)) {
                /* Give another thread a chance */
                drop_gil(ceval, tstate);

                /* Other threads may run now */
                take_gil(ceval, tstate);

                /* Check if we should make a quick exit. */
                exit_thread_if_finalizing(tstate);

                if (_PyThreadState_Swap(&runtime->gilstate, tstate) != NULL) {
                    Py_FatalError("ceval: orphan tstate");
                }
            }
        }
        <!-- ... -->
        switch (opcode) {
          case TARGET(NOP):
          case TARGET(LOAD_FAST):
          <!-- ... -->
        }
    }
{{< /highlight >}}
CPython使用系统线程，且没有实现线程调度。所以，具体哪个等待线程被切换执行，由操作系统决定。甚至，发出请求和被切换执行的未必就是同一个线程。

对于CPU密集型任务，除了使用多进程架构绕开，也可以使用C来编写多线程的扩展也能绕开GIL限制。


执行过程
-------

### 入口
{{< highlight c>}}
<!-- cpython/Programs/python.c -->
int
main(int argc, char **argv)
{
    // unix平台是_Py_UnixMain，windows平台是Py_Main
    return _Py_UnixMain(argc, argv);
}
{{< /highlight >}}

然后是选择执行模式:
{{< highlight c>}}
<!-- cpython/Modules/main.c -->
int
_Py_UnixMain(int argc, char **argv)
{
    return pymain_main(&pymain);
}
static int
pymain_main(_PyMain *pymain)
{
    
    pymain_init(pymain);

    int res = pymain_cmdline(pymain);
    if (res < 0) {
        _Py_FatalInitError(pymain->err);
    }
    if (res == 1) {
        goto done;
    }

    pymain_init_stdio(pymain);
    // 初始化
    pymain->err = _Py_InitializeCore(&pymain->config);

    // 执行逻辑
    pymain_run_python(pymain);

    if (Py_FinalizeEx() < 0) {
        pymain->status = 120;
    }

done:
    // 退出清理
    pymain_free(pymain);

    return pymain->status;
}
static void
pymain_run_python(_PyMain *pymain)
{
    PyCompilerFlags cf = {.cf_flags = 0};

    pymain_header(pymain);
    pymain_import_readline(pymain);

    // 执行模式选择
    if (pymain->command) {
        // 命令行模式 -c
        pymain->status = pymain_run_command(pymain->command, &cf);
    }
    else if (pymain->module) {
        // 模块模式 -m
        pymain->status = (pymain_run_module(pymain->module, 1) != 0);
    }
    else {
        // 入口文件模式
        pymain_run_filename(pymain, &cf);
    }

    pymain_repl(pymain, &cf);
}
{{< /highlight >}}

### 初始化
主要是初始化内置类型，以及创建buildins、sys模块，并初始化sys.modules，sys.path等运行所需的环境配置。
{{< highlight c>}}
<!-- cpython/Python/pylifecycle.c -->
_PyInitError
_Py_InitializeCore(const _PyCoreConfig *core_config)
{
    // 创建解释器状态实例
    interp = PyInterpreterState_New();

    // 创建主线程状态实例
    tstate = PyThreadState_New(interp);
    (void) PyThreadState_Swap(tstate);

    /* Auto-thread-state API */
    _PyGILState_Init(interp, tstate);

    /* Create the GIL */
    PyEval_InitThreads();

    // 创建并初始化内置类型
    _Py_ReadyTypes();

    // 初始化带有对象缓存的内置类型
    if (!_PyLong_Init())
        return _Py_INIT_ERR("can't init longs");

    if (!PyByteArray_Init())
        return _Py_INIT_ERR("can't init bytearray");

    if (!_PyFloat_Init())
        return _Py_INIT_ERR("can't init float");

    PyObject *modules = PyDict_New();

    // 创建sys.modules,存储运行期被导入的模块
    interp->modules = modules;

    // 初始化sys模块
    err = _PySys_BeginInit(&sysmod);
    interp->sysdict = PyModule_GetDict(sysmod);
    Py_INCREF(interp->sysdict);
    PyDict_SetItemString(interp->sysdict, "modules", modules);
    _PyImport_FixupBuiltin(sysmod, "sys", modules);

    // 初始化 __buildin__ 模块
    bimod = _PyBuiltin_Init();
    _PyImport_FixupBuiltin(bimod, "builtins", modules);
    interp->builtins = PyModule_GetDict(bimod);
    Py_INCREF(interp->builtins);

    // 初始化内置异常类型
    _PyExc_Init(bimod);

    // 初始化导入机制
    err = _PyImport_Init(interp);
    err = _PyImportHooks_Init();

    /* Initialize _warnings. */
    if (_PyWarnings_Init() == NULL) {
        return _Py_INIT_ERR("can't initialize warnings");
    }

    if (!_PyContext_Init())
        return _Py_INIT_ERR("can't init context");

    /* Only when we get here is the runtime core fully initialized */
    _PyRuntime.core_initialized = 1;
    return _Py_INIT_OK();
}
{{< /highlight >}}

### 执行

### 终止