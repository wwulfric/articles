---
title: Python 运行时观测的三条边界：frame、greenlet 与 task
date: 2026-05-27 00:00
categories: [技术]
tags: [python, cpython, gevent, asyncio, debug]
---

一段 Python 程序跑起来以后，源码里的函数名、行号、局部变量、异常传播并没有消失。它们变成了运行时里的 frame、code object、字节码位置和调用栈。调试器要停在某一行，coverage 要知道哪段代码被执行过，profiler 要记录时间花在什么函数里，线上诊断工具要在进程继续服务时看见这些信息。

日志只能覆盖提前想到的路径。真正难查的问题常常发生在没有埋点的地方：某个分支被意外走到，某个 coroutine 长时间挂起，某个 gevent 服务看起来像阻塞，实际已经在多个 greenlet 之间来回切换。运行时观测接口的价值就在这里：工具可以跟着解释器的执行节奏，记录函数调用、行号变化、异常、yield、resume、调度切换等事件。

`sys.settrace()` 和 `sys.monitoring` 处理的是这类问题：解释器执行 Python 代码时，外部工具能在哪些位置插进来。`sys.monitoring` 来自 PEP 669，目标是给调试器、覆盖率工具、profiler 提供一套低影响的执行事件接口。[^sys-settrace][^sys-monitoring][^pep-669]

这个问题看起来很底层，放进真实程序之后会变得微妙。因为 Python 程序里的“正在运行”有几种形态：

普通函数沿着 CPython frame 一层层调用；gevent 在同一个 OS 线程里切换多组 greenlet 栈；asyncio 把 coroutine 包成 Task，再交给 event loop 调度。观测工具选中的边界不同，看到的世界也不同。

这也是很多追踪问题的根源。事件本身还在，事件归属却可能已经换了一套语义。

---

## 从 frame 开始

frame 可以理解成 Python 层的栈帧。CPython 执行 Python 函数时，会为这次调用准备一份执行现场：正在跑哪段代码、局部变量是什么、全局变量从哪里取、下一条字节码在哪里、上一层调用是谁。这份执行现场暴露到 Python 层，就是 frame object。

比如一段调用链：

```text
main() -> handle_request() -> query_db()
```

当 `query_db()` 正在执行时，当前 frame 属于 `query_db()`；它的上一层 frame 指向 `handle_request()`；再往上能回到 `main()`。异常 traceback 也是沿着这串 frame 展开，所以我们能看到错误从哪一层函数传出来。

frame 里最常用的几块信息包括：

- `f_code`：当前 frame 正在执行的 code object，里面有函数名、文件名、字节码等信息
- `f_locals`：当前局部变量
- `f_globals`：当前全局变量
- `f_lineno`：当前源码行号
- `f_back`：上一层 frame

调试器能停在某一行，coverage 能知道哪一行跑过，profiler 能记录函数进入和退出，底层入口都绕不开 frame 或 code object。

`sys.settrace(tracefunc)` 是老接口。它给当前线程安装一个 trace function。新作用域进入时会触发 `call` 事件；trace function 返回的本地 trace function 会继续接收 `line`、`return`、`exception`，以及可选的 `opcode` 事件。[^sys-settrace]

一个最小追踪器长这样：

```python
import sys


def trace(frame, event, arg):
    if event == "call":
        code = frame.f_code
        print("call", code.co_name, code.co_filename, frame.f_lineno)
    return trace


sys.settrace(trace)
```

这里有两个很重要的细节。

第一，`sys.settrace()` 是线程相关的。它影响当前线程；多线程调试要给每个线程设置 trace function，或者用 `threading.settrace()`。Python 3.12 又加了 `threading.settrace_all_threads()`，可以覆盖已经在执行的 Python 线程。[^threading-trace]

第二，它的成本来自回调频率。`line` 事件会在大量源码行前触发，`opcode` 更细。每次事件都要调用 Python 函数，工具越想看得细，对程序执行节奏的扰动就越明显。调试器和 coverage 可以接受这种成本，常驻生产 profiler 往往要谨慎得多。

`sys.settrace()` 提供的是 frame 级叙事：这个线程进入了哪个 frame，这个 frame 执行到哪一行，什么时候返回或抛异常。只要“一个线程里的调用栈”就是你要分析的执行上下文，它就很好用。

---

## sys.monitoring 把事件做细

`sys.monitoring` 是 Python 3.12 之后的新接口。PEP 669 给它的定位很明确：降低调试、覆盖率、profiling 这类工具的常态成本。[^sys-monitoring][^pep-669]

它的设计比 `sys.settrace()` 更像一套事件总线：

- 工具先用 `sys.monitoring.use_tool_id(tool_id, name)` 占用一个 tool id
- 再用 `register_callback(tool_id, event, func)` 注册事件回调
- 用 `set_events(tool_id, event_set)` 打开全局事件
- 或者用 `set_local_events(tool_id, code, event_set)` 只打开某个 code object 上的事件

事件粒度也更丰富。比如：

- `PY_START`：Python 函数开始执行
- `PY_RETURN`：Python 函数返回
- `PY_YIELD`：Python 函数 yield
- `PY_RESUME`：generator 或 coroutine 恢复执行
- `CALL`：Python 代码里发生一次调用
- `LINE`：即将执行不同源码行对应的指令
- `INSTRUCTION`：即将执行 VM 指令
- `JUMP`、`BRANCH`：控制流跳转
- `RAISE`、`EXCEPTION_HANDLED`、`PY_UNWIND`：异常相关事件

代码形态大概是这样：

```python
import sys

tool_id = sys.monitoring.PROFILER_ID
events = sys.monitoring.events

sys.monitoring.use_tool_id(tool_id, "demo-profiler")


def on_start(code, instruction_offset):
    print("start", code.co_name, instruction_offset)


sys.monitoring.register_callback(tool_id, events.PY_START, on_start)
sys.monitoring.set_events(tool_id, events.PY_START)
```

`sys.monitoring` 改进的是插桩方式；观测对象仍然是 CPython 正在执行的 code object、instruction offset 和函数事件。它可以用 `DISABLE` 关闭某个代码位置上的后续事件，也能按 code object 控制事件集合。断点、单步、覆盖率、调用统计都可以建立在这些能力上。

这个接口降低了常态成本，但细粒度事件依然昂贵。PEP 669 也提醒过，`LINE` 事件会显著影响性能，精确 profiler 本身会扭曲结果；如果目标是长期线上画像，统计采样经常更稳。

---

## gevent 改变了栈的连续性

gevent 的核心模型来自 greenlet。gevent 文档把自己描述成基于 greenlet 的 coroutine 网络库；greenlet 文档则把 greenlet 说成一组独立的小栈。每个 greenlet 有自己的 frame 栈，执行可以在这些栈之间显式跳转。[^gevent-intro][^greenlet-concepts]

普通同步代码里，一个线程的 frame 栈大致有清晰的进出顺序：

```text
main -> handle_request -> query_db -> return -> return
```

工具收到 `call` 和 `return` 后，用一个线程本地的栈就能配对。

gevent 下，同一个 OS 线程里可能跑着很多请求。某个 greenlet 进入 `query_db()`，里面调用了被 monkey patch 过的 socket API。这个 API 看起来像同步阻塞调用，实际会把当前 greenlet 切到 Hub，让别的 greenlet 继续跑：

```text
greenlet A: handle_request -> query_db -> socket recv
                                      switch to Hub
greenlet B: handle_request -> render -> ...
```

从 `sys.settrace()` 的视角看，还是同一个 OS 线程在发出事件。问题出在归属关系：A 的 `query_db()` 没有返回，B 的 `handle_request()` 已经开始执行。一个按线程维护调用栈的追踪器，会把 A 和 B 的 frame 混在一起。

gevent 下常说 `sys.settrace()` 失效，问题主要在这里：它仍然能收到 CPython frame 事件，事件序列里却没有 greenlet 调度边界。调试某个具体 frame 时，它有价值；构建请求级调用树、耗时归因、阻塞点归因时，单靠它很容易得出错乱结果。

`sys.monitoring` 也会遇到同类边界。它比 `sys.settrace()` 事件更细，开销更可控，还能看到 coroutine/generator 的 `PY_YIELD` 和 `PY_RESUME`。可 greenlet switch 是 greenlet 运行时的栈转移，标准 monitoring 事件不会直接告诉你“刚才从哪个 greenlet 切到了哪个 greenlet”。缺少这条边，工具就很难把 frame 事件重新挂回正确的逻辑执行流。

monkey patch 会让这种错位更隐蔽。业务代码写的是 `time.sleep()`、`socket.recv()`、数据库驱动调用；运行时实际发生的是一次让出控制权。源码层面的同步形态还在，调度层面的连续性已经变了。

---

## greenlet trace 补上调度边界

greenlet 为这个问题提供了自己的追踪接口：`greenlet.settrace()`。greenlet 文档也专门提到，标准 Python tracing/profiling 在 greenlet 下无法按预期工作，因为栈和 frame 切换发生在同一个 Python 线程里。[^greenlet-tracing]

`greenlet.settrace()` 安装的回调签名是：

```python
callback(event: str, args: object)
```

当前定义的事件主要有两类：

- `switch`：一次普通 greenlet 切换
- `throw`：通过 `greenlet.throw()` 切到目标 greenlet，并在目标里抛异常

这两个事件的 `args` 都是 `(origin, target)`，也就是从哪个 greenlet 切到哪个 greenlet。

一个最小例子：

```python
import greenlet


def on_greenlet_event(event, args):
    if event in ("switch", "throw"):
        origin, target = args
        print(event, origin, "->", target)


old_trace = greenlet.settrace(on_greenlet_event)

try:
    ...
finally:
    greenlet.settrace(old_trace)
```

这套接口补上的正是 `sys.settrace()` 和 `sys.monitoring` 缺少的调度边界。它不会告诉你每一行代码执行了什么，也不会替代 frame 事件；它告诉你逻辑执行流什么时候从一组 frame 栈跳到另一组 frame 栈。

gevent 的监控能力也会用到这条边。gevent 的配置文档提到，监控线程检测 event loop 被阻塞的功能依赖 `greenlet.settrace()`；如果用户覆盖 trace function，需要把前一个 trace function 串起来，否则 gevent 自己的监控也会受影响。[^gevent-monitoring]

做 gevent 观测时，比较可靠的模型通常是组合式的：

- CPython frame 事件负责函数、行号、异常、返回值
- greenlet trace 负责 `origin -> target` 的调度边
- gevent 自己的 Hub、monitor thread、`gevent.util.print_run_info()` 负责运行时状态、阻塞报告和 greenlet 栈信息

单独看任何一层，都容易把调度事实压扁。

---

## asyncio 的边界在 Task

asyncio 的情况要温和一些。它同样是协作式并发，默认也在一个线程里跑很多任务；但它的调度单位是 coroutine 和 Task，暂停点写在 `await` 上，恢复路径由 event loop 管理。

asyncio 的调度规则是：event loop 一次运行一个 Task；正在运行的 Task 遇到 `await` 后暂停，event loop 再运行下一个 Task。[^asyncio-task]

`sys.settrace()` 和 `sys.monitoring` 在 asyncio 代码里仍然可用。coroutine 执行时还是 Python frame，进入函数、执行行、返回、抛异常，这些事件都能被看到。Python 3.12 的 `sys.monitoring` 还给了 `PY_YIELD`、`PY_RESUME` 这类事件，对 generator/coroutine 的暂停和恢复更友好。

这里缺的是 Task 身份。一个 event loop 线程里可能有很多 Task 轮流执行；frame 事件告诉你当前跑到了哪段代码，Task API 告诉你这段代码属于哪个调度单元。

标准库已经给了几组常用工具：[^asyncio-task]

- `asyncio.current_task()`：拿到当前正在运行的 Task
- `asyncio.all_tasks()`：列出 loop 里尚未完成的 Task
- `Task.get_stack()` / `Task.print_stack()`：查看挂起 Task 的栈或 traceback
- `Task.get_coro()`：拿到 Task 包裹的 coroutine
- `Task.get_name()`：拿到 Task 名称，适合做日志和诊断标签
- `Task.get_context()`：Python 3.12 后可取出 Task 对应的 `contextvars.Context`

`contextvars` 是 asyncio 观测里很关键的一层。请求 ID、trace ID、租户 ID 这类“逻辑上下文”，放在线程局部变量里会在并发任务之间互相污染；ContextVar 会随 Task 上下文传播，更适合异步框架。[^contextvars]

调试层面，asyncio 还提供了开发模式工具。可以通过 `PYTHONASYNCIODEBUG=1`、`asyncio.run(..., debug=True)` 或 `loop.set_debug(True)` 打开 debug mode。它会帮助发现从未 await 的 coroutine、从未取回的 Task 异常，并能在 debug 输出里带上对象创建位置。`sys.set_coroutine_origin_tracking_depth()` 也可以打开 coroutine 创建位置记录，让 coroutine 对象的 `cr_origin` 保存创建栈摘要。[^asyncio-dev][^sys-coroutine-origin]

asyncio 下的观测路径通常是：

1. 用 `sys.settrace()` 或 `sys.monitoring` 拿到 frame / code object 级事件
2. 在事件发生时用 `asyncio.current_task()` 补上 Task 身份
3. 用 `contextvars` 绑定请求级上下文
4. 用 debug mode、Task stack、coroutine origin 查创建点和挂起点

这里的难点不在事件能不能收到，而在事件如何归入正确的 Task。

---

## 工具设计里的边界

`sys.settrace()`、`sys.monitoring`、`greenlet.settrace()`、asyncio Task API 看起来像几套互相重叠的追踪接口。拆开看，它们各自站在不同边界上。

`sys.settrace()` 站在 frame 上。它适合 debugger、coverage、教学工具，也适合短时间抓取调用细节。它直接、通用，代价也直接。

`sys.monitoring` 站在 code object 和解释器事件上。它把低层事件做成可组合接口，给调试器、coverage、profiler 留出了更低成本的实现空间。它解决的是 CPython 事件插桩成本和协作问题。

`greenlet.settrace()` 站在 greenlet 调度边界上。它记录从哪个 greenlet 切到哪个 greenlet，适合 gevent 这类基于 greenlet 的运行时。它解决的是同一 OS 线程里多组栈互相切换后的归属问题。

asyncio 的 Task API 站在 coroutine 调度边界上。它把当前 Task、全部 Task、挂起栈、创建上下文暴露出来。它解决的是 event loop 内部多个 coroutine 轮流执行时的身份问题。

做运行时观测时，把“线程”当成唯一执行上下文，很容易埋下误判。在线程模型里，这个假设大多够用；进入 gevent 或 asyncio 后，一个线程里会出现多个逻辑执行流。frame 事件依然真实，调用树、耗时归因、请求归属却需要额外的调度信息。

判断接口之前，先看观测对象：

如果对象是源码行、函数进入退出、异常传播，frame 或 monitoring 事件就够接近。  
如果对象是 gevent 里的请求、连接、任务切换，就要接上 greenlet trace。  
如果对象是 asyncio 里的请求、后台任务、取消传播，就要接上 Task 和 contextvars。

选对边界以后，追踪接口才会变成工具。边界选错时，越精细的事件反而越容易制造假象。

---

[^sys-settrace]: Python 文档：`sys.settrace()`，https://docs.python.org/3/library/sys.html#sys.settrace
[^sys-monitoring]: Python 文档：`sys.monitoring`，https://docs.python.org/3/library/sys.monitoring.html
[^pep-669]: PEP 669: Low Impact Monitoring for CPython，https://peps.python.org/pep-0669/
[^threading-trace]: Python 文档：`threading.settrace()`，https://docs.python.org/3/library/threading.html#threading.settrace；`threading.settrace_all_threads()`，https://docs.python.org/3/library/threading.html#threading.settrace_all_threads
[^greenlet-tracing]: greenlet 文档：Tracing And Profiling，https://greenlet.readthedocs.io/en/stable/tracing.html
[^greenlet-concepts]: greenlet 文档：greenlet Concepts，https://greenlet.readthedocs.io/en/latest/greenlet.html
[^gevent-intro]: gevent 文档：Introduction，https://docs.gevent.org/intro.html
[^gevent-monitoring]: gevent 文档：Monitoring and Debugging gevent Applications，https://www.gevent.org/monitoring.html
[^asyncio-dev]: Python 文档：Developing with asyncio，https://docs.python.org/3/library/asyncio-dev.html
[^asyncio-task]: Python 文档：Coroutines and Tasks，https://docs.python.org/3/library/asyncio-task.html
[^contextvars]: Python 文档：`contextvars`，https://docs.python.org/3/library/contextvars.html
[^sys-coroutine-origin]: Python 文档：`sys.set_coroutine_origin_tracking_depth()`，https://docs.python.org/3/library/sys.html#sys.set_coroutine_origin_tracking_depth
