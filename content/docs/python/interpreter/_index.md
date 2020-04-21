---
title: "CPython解释器"
draft: false
---

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
        switch (opcode) {
          case TARGET(NOP):
          case TARGET(LOAD_FAST):
          ...
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
int main(int argc, char **argv)
{
    // unix平台是_Py_UnixMain，windows平台是Py_Main
    return _Py_UnixMain(argc, argv);
}
{{< /highlight >}}

然后是选择执行模式:
{{< highlight c>}}
<!-- cpython/Modules/main.c -->
int _Py_UnixMain(int argc, char **argv)
{
    return pymain_main(&pymain);
}

static int pymain_main(_PyMain *pymain)
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

static void pymain_run_python(_PyMain *pymain)
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
_PyInitError _Py_InitializeCore(const _PyCoreConfig *core_config)
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
完成初始化之后，就进入执行流程，这里以文件模式的执行为例:
{{< highlight c>}}
<!-- cpython/Modules/main.c -->
static int pymain_run_file(FILE *fp, const wchar_t *filename, PyCompilerFlags *p_cf)
{
    run = PyRun_AnyFileExFlags(fp, filename_str, filename != NULL, p_cf);
    return run != 0;
}
{{< /highlight >}}

{{< highlight c>}}
<!-- cpython/Python/pythonrun.c -->
/* Parse input from a file and execute it */
int PyRun_AnyFileExFlags(FILE *fp, const char *filename, int closeit, ...)
{
    return PyRun_SimpleFileExFlags(fp, filename, closeit, flags);
}

int PyRun_SimpleFileExFlags(FILE *fp, const char *filename, int closeit, ...)
{
    // 获取__main__.__dict__，添加__file__ 信息
    m = PyImport_AddModule("__main__");
    d = PyModule_GetDict(m);
    if (PyDict_GetItemString(d, "__file__") == NULL) {
        PyObject *f;
        f = PyUnicode_DecodeFSDefault(filename);
        if (f == NULL)
            goto done;
        if (PyDict_SetItemString(d, "__file__", f) < 0) {
            Py_DECREF(f);
            goto done;
        }
        if (PyDict_SetItemString(d, "__cached__", Py_None) < 0) {
            Py_DECREF(f);
            goto done;
        }
    }

    // 运行入口文件（检查pyc缓存），将__main__.__dict__做为名字空间传入
    if (maybe_pyc_file(fp, filename, ext, closeit)) {
        // 运行缓存文件
        v = run_pyc_file(pyc_fp, filename, d, d, flags);
        fclose(pyc_fp);
    } else {
        // 运行py文件
        v = PyRun_FileExFlags(fp, filename, Py_file_input, d, d,
                              closeit, flags);
    }
  done:
    if (set_file_name && PyDict_DelItemString(d, "__file__"))
        PyErr_Clear();
    Py_DECREF(m);
    return ret;
}

// 编译源文件，随后进入字节码执行流程
PyObject * PyRun_FileExFlags(FILE *fp, const char *filename_str, int start, ...)
{
    filename = PyUnicode_DecodeFSDefault(filename_str);
    arena = PyArena_New();

    // 编译源文件
    mod = PyParser_ASTFromFileObject(fp, filename, NULL, start, 0, 0,
                                     flags, NULL, arena);
    // 字节码执行流程
    ret = run_mod(mod, filename, globals, locals, flags, arena);
    return ret;
}

static PyObject * run_mod(mod_ty mod, PyObject *filename, ...)
{
    co = PyAST_CompileObject(mod, filename, flags, -1, arena);
    v = PyEval_EvalCode((PyObject*)co, globals, locals);
}
{{< /highlight >}}

之后创建执行所需的栈帧对象，并准备好参数等待执行数据。
{{< highlight c>}}
<!-- cpython/Python/ceval.c -->
PyObject * PyEval_EvalCode(PyObject *co, PyObject *globals, PyObject *locals)
{
    return PyEval_EvalCodeEx(co, globals, locals, ...);
}
PyObject * PyEval_EvalCodeEx(PyObject *_co, PyObject *globals, PyObject *locals, ...)
{
    return _PyEval_EvalCodeWithName(_co, globals, locals, ...);
}
PyObject * _PyEval_EvalCodeWithName(PyObject *_co, PyObject *globals, PyObject *locals, ...)
{

    // 创建栈帧
    tstate = PyThreadState_GET();
    f = _PyFrame_New_NoTrack(tstate, co, globals, locals);

    // 填充参数、自由变量（闭包）等数据
    retval = PyEval_EvalFrameEx(f,0);
}
PyObject * PyEval_EvalFrameEx(PyFrameObject *f, int throwflag)
{
    PyThreadState *tstate = PyThreadState_GET();
    return tstate->interp->eval_frame(f, throwflag);
}
PyObject* _Py_HOT_FUNCTION _PyEval_EvalFrameDefault(PyFrameObject *f, int throwflag)
{
    // 指令参数所需的相关名字列表
    co = f->f_code;
    names = co->co_names;
    consts = co->co_consts;
    fastlocals = f->f_localsplus;
    freevars = f->f_localsplus + co->co_nlocals;

    // 类似于SP、PC寄存器，下一条指令及栈帧顶位置
    first_instr = (_Py_CODEUNIT *) PyBytes_AS_STRING(co->co_code);
    next_instr = first_instr;
    stack_pointer = f->f_stacktop;

    // 解释循环
    main_loop:
    for (;;) {
        // 检查并处理GIL
        if (_Py_atomic_load_relaxed(&_PyRuntime.ceval.eval_breaker)) {
        }

    fast_next_opcode:
        // 下一条指令和参数
        NEXTOPARG();
    dispatch_opcode:

        // 指令执行，每条指令由C实现具体过程
        switch (opcode) {

        TARGET(NOP)
            FAST_DISPATCH();

        TARGET(LOAD_FAST) {
            PyObject *value = GETLOCAL(oparg);
            Py_INCREF(value);
            PUSH(value);
            FAST_DISPATCH();
        }
     }
     exit_eval_frame:
        Py_LeaveRecursiveCall();
        f->f_executing = 0;
        tstate->frame = f->f_back;
    return _Py_CheckFunctionResult(NULL, retval, "PyEval_EvalFrameEx");
}
{{< /highlight >}}

用户栈内存按地址从低向高分配，每次执行时会递增指令计数器。
{{< highlight c>}}
<!-- cpython/Python/ceval.c -->
#define NEXTOPARG()  do { \
        _Py_CODEUNIT word = *next_instr; \
        opcode = _Py_OPCODE(word); \
        oparg = _Py_OPARG(word); \
        next_instr++; \
    } while (0)
#define EMPTY()           (STACK_LEVEL() == 0)
#define TOP()             (stack_pointer[-1])
#define SECOND()          (stack_pointer[-2])
#define THIRD()           (stack_pointer[-3])
#define FOURTH()          (stack_pointer[-4])
#define PEEK(n)           (stack_pointer[-(n)])
#define SET_TOP(v)        (stack_pointer[-1] = (v))
#define SET_SECOND(v)     (stack_pointer[-2] = (v))
#define SET_THIRD(v)      (stack_pointer[-3] = (v))
#define SET_FOURTH(v)     (stack_pointer[-4] = (v))
#define SET_VALUE(n, v)   (stack_pointer[-(n)] = (v))
#define BASIC_STACKADJ(n) (stack_pointer += n)
#define BASIC_PUSH(v)     (*stack_pointer++ = (v)) // 地址从低向高进行
#define BASIC_POP()       (*--stack_pointer)
{{< /highlight >}}

### 终止
执行完之后，结束之前还要进行一系列的清理操作。
{{< highlight c>}}
<!-- cpython/Python/pylifecycle.c -->
int Py_FinalizeEx(void)
{
    // 等待前台线程结束
    wait_for_thread_shutdown();

    /* Get current thread state and interpreter pointer */
    tstate = PyThreadState_GET();
    interp = tstate->interp;

    // 调用 atexit 注册的退出函数
    call_py_exitfuncs(interp);

    /* Flush sys.stdout and sys.stderr */
    if (flush_std_files() < 0) {
        status = -1;
    }

    /* Disable signal handling */
    PyOS_FiniInterrupts();

    // 垃圾回收，执行析构方法
    _PyGC_CollectIfEnabled();

    // 释放导入的模块
    PyImport_Cleanup();

    // 执行相关结束函数
    _PyTraceMalloc_Fini();
    _PyImport_Fini();
    _PyType_Fini();
    _PyFaulthandler_Fini();
    _PyExc_Fini();

    // 执行内置类型的结束函数
    PyMethod_Fini();
    PyFrame_Fini();
    PyCFunction_Fini();
    PyTuple_Fini();
    PyList_Fini();
    PySet_Fini();
    PyBytes_Fini();
    PyByteArray_Fini();
    PyLong_Fini();
    PyFloat_Fini();
    PyDict_Fini();
    PySlice_Fini();
    _PyGC_Fini();
    _Py_HashRandomization_Fini();
    _PyArg_Fini();
    PyAsyncGen_Fini();
    _PyContext_Fini();

    /* Cleanup Unicode implementation */
    _PyUnicode_Fini();

    PyGrammar_RemoveAccelerators(&_PyParser_Grammar);

    /* Cleanup auto-thread-state */
    _PyGILState_Fini();

    /* Delete current thread. After this, many C API calls become crashy. */
    PyThreadState_Swap(NULL);
    // 清理解释器和主线程状态
    PyInterpreterState_Delete(interp);

    call_ll_exitfuncs();

    _PyRuntime_Finalize();

    return status;
}
{{< /highlight >}}