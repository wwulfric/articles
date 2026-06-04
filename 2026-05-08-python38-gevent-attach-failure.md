---
title: gevent monkey patch 如何影响 attach agent：从启动超时到观测边界
date: 2026-05-08 00:00
categories: [技术]
tags: [python, gevent, debug]
---

线上 Python 服务偶尔卡住、CPU 飙高，或者某个接口突然变慢时，进程往往还在跑，不能为了加一行日志或者打开 `cProfile` 就重启。此时关心的是这个 PID 正在做什么：哪些函数在调用，栈停在哪里，内存分配是否异常，某个函数的入参、返回值和耗时到底是什么。

性能分析和运行时诊断工具做的就是这类事，只是进入目标进程的方式不同。`cProfile` 跟着应用一起跑，通过解释器事件记录函数调用和耗时；`py-spy` 或系统级 `perf` 更像旁路观察者，控制端留在目标进程外面，通过操作系统能力读取线程栈或采样 PC；memray、运行时 trace 工具、attach agent 则会把一部分采集逻辑放进目标进程内部，安装 hook、wrapper 或后台 agent，观察更细的运行时状态。

peeka 属于 attach agent 这一类。用户在外部选择一个正在运行的 Python 进程，peeka 把一段 agent 代码送进去；agent 在目标进程里建立命令通道，后续 `watch`、`trace`、`top` 等命令再通过这个通道执行。它更接近运行时诊断工具，而不是离线性能分析器。

先把问题分成控制面和数据面两层。

- 控制面：工具为了启动、attach、通信、接收命令、返回结果而维护的基础设施。
- 数据面：工具负责观察程序的路径，比如函数 wrapper、调用栈采样、tracing backend、内存分配 hook。

不同工具的控制面和数据面位置不一样。`cProfile` 基本都在目标进程内部；`py-spy` 的控制面在外部，数据采样也尽量从外部完成；memray 会把采集逻辑放进目标进程内部，再把数据写到外部可分析的文件；peeka 的控制面横跨内外两侧，外部 CLI 负责发命令，目标进程里的 agent 负责接命令和执行观测。

这种差异平时不明显。一旦目标进程使用 gevent，问题就会浮出来。gevent 通过 monkey patch 把 `socket`、`threading`、`time`、`select` 等标准库对象替换成协作式版本。对业务代码来说，这是 gevent 的能力来源；对诊断工具来说，它可能同时影响控制面和数据面。

控制面可能在启动 socket、线程或同步信号时卡住。数据面可能还能给出结果，但结果的含义会变化：函数 wrapper 仍然能观察入口调用，递归 trace 和线程 frame 采样却不一定还能代表完整执行过程。

attach agent 在 gevent 目标进程里的启动超时，就是这种影响最先暴露出来的地方。

很多 gevent 应用会在启动早期执行：

```python
from gevent import monkey
monkey.patch_all()
```

这行代码会把 `socket`、`threading`、`time`、`select` 等标准库对象替换成 gevent 版本。业务代码这么做是为了获得协作式 I/O；但 attach agent 如果也直接使用这些被替换后的对象，就可能在自己刚启动时卡住。

外部看到的错误通常很普通：

```text
Agent initialization timeout
```

具体工具可能会把这个超时写成等待某个文件、notify socket 或 IPC 消息超时。传递方式不同，但外部控制端等的都是同一件事：agent 发出“我已经启动完成”的就绪信号。

这个启动超时容易被误读成“注入失败”或者“等待时间太短”。实际更常见的情况是：agent 已经进去了，但它死在发出就绪信号之前。

启动超时先暴露控制面问题；agent 能连上以后，还要继续解释数据面结果的观测边界。

---

## 问题背景：gevent 会同时影响控制面和数据面

### attach agent 的启动链路：就绪信号只代表控制面初始化

一个 attach agent 的最小流程可以画成这样：

```text
诊断工具
    |
    | attach(pid)
    v
GDB / LLDB / dlopen / PEP 768
    |
    | execute agent script
    v
目标 Python 进程
    |
    | start agent
    v
/tmp/agent_<session>.sock  <---->  诊断工具 client
```

agent 在目标进程里通常要做几件事：

1. 创建本地 socket；
2. 启动后台 accept loop；
3. 发出就绪信号；
4. 等外部 client 连上来，接收后续命令。

如果目标是普通 Python 进程，下面这段代码看起来很自然：

```python
import socket
import threading


def notify_started():
    # 通知外部控制端：agent 已经完成基础初始化。
    pass


class MiniAgent:
    def __init__(self, session_id):
        self.sock_path = f"/tmp/agent_{session_id}.sock"
        self.server = None

    def start(self):
        self.server = socket.socket(socket.AF_UNIX, socket.SOCK_STREAM)
        self.server.bind(self.sock_path)
        self.server.listen(5)

        accept_started = threading.Event()
        thread = threading.Thread(
            target=self._accept_loop,
            args=(accept_started,),
            daemon=True,
        )
        thread.start()
        accept_started.wait(timeout=10)

        notify_started()

    def _accept_loop(self, accept_started):
        accept_started.set()
        while True:
            conn, _ = self.server.accept()
            threading.Thread(
                target=self._handle_client,
                args=(conn,),
                daemon=True,
            ).start()

    def _handle_client(self, conn):
        with conn:
            conn.sendall(b"hello\n")
```

这段写法放在普通 Python 进程里通常能工作；但 attach agent 落进的是目标进程自己的 runtime。目标进程如果已经做过 gevent monkey patch，这里的 `socket` 和 `threading` 就可能不是原始实现了。

---

### gevent monkey patch 会让 agent 卡在就绪信号之前

monkey patch 会改变后续 import 的结果：再 `import socket`、`import threading` 时，拿到的可能已经不是原始实现。

对业务代码来说，这是 gevent 的能力来源。对 attach agent 来说，这会影响自己的启动路径。

最明显的风险是：

```python
accept_started.wait(timeout=10)
```

更隐蔽的风险在：

```python
thread.start()
```

CPython 的 `Thread.start()` 不只是创建底层线程。它还会等待线程内部的 `_started` 事件：

```python
def start(self):
    _start_new_thread(self._bootstrap, ())
    self._started.wait()
```

如果 `threading` 已经被 gevent 改造，这个 wait 可能落到 gevent 的 event 语义里。在某些注入时机下，调用链会变成：

```text
agent.start()
  -> threading.Thread.start()
     -> self._started.wait()
        -> gevent.event.Event.wait()
           -> gevent.exceptions.BlockingSwitchOutError
```

常见错误是：

```text
gevent.exceptions.BlockingSwitchOutError:
Impossible to call blocking function in the event loop callback
```

这个异常和 GDB、LLDB、PEP 768 这些注入方式没有直接关系。agent 已经进入目标进程，只是在执行过程中撞到了 gevent 对阻塞切换的限制。

外部最终只等到：

```text
Agent initialization timeout
```

排查时先别急着改等待时间。更值得确认的是：agent 发出就绪信号之前，是否已经碰到了被 monkey patch 的控制面对象。

---

## 控制面：启动和通信要避开 monkey patch

### 用原生 socket 和线程入口建立命令通道

控制面的第一目标是让 agent 先跑起来。

“跑起来”还不涉及诊断结果是否准确，只要求 agent 至少完成几件事：

1. 在目标进程里启动；
2. 建好命令 socket；
3. 发出就绪信号；
4. 接收外部 client 的第一条命令。

agent 自己的命令通道要尽量避开目标应用可能 monkey patch 的高层 API。启动路径可以退到这几类原语上：

```python
import _socket
import _thread
import sys


def notify_started():
    # 通知外部控制端：agent 已经完成基础初始化。
    pass


def native_start_new_thread():
    monkey = sys.modules.get("gevent.monkey")
    if monkey is not None:
        return monkey.get_original("_thread", "start_new_thread")
    return _thread.start_new_thread


class MiniAgent:
    def __init__(self, session_id):
        self.sock_path = f"/tmp/agent_{session_id}.sock"
        self.server = None

    def start(self):
        self.server = _socket.socket(_socket.AF_UNIX, _socket.SOCK_STREAM)
        self.server.bind(self.sock_path)
        self.server.listen(5)

        start_thread = native_start_new_thread()
        start_thread(self._accept_loop, ())
        notify_started()

    def _accept_loop(self):
        start_thread = native_start_new_thread()
        while True:
            conn, _ = self.server.accept()
            start_thread(self._handle_client, (conn,))

    def _handle_client(self, conn):
        conn.sendall(b"hello\n")
        conn.close()
```

改动看起来不大，但依赖的对象已经换了：

- socket 改用底层 `_socket.socket`，不再通过 `socket.socket` 建命令通道；
- 线程改用原始 `_thread.start_new_thread`，不再通过 `threading.Thread` 启动控制线程；
- 就绪同步交给 `listen()` 后的 socket backlog 和外部 hello 探测，不再用 `threading.Event.wait()` 作为发出就绪信号前的 barrier；
- agent 自己的命令通道整体退到 `_socket` / `_thread` 这组底层原语上，不再把控制线程和控制 socket 建在 `socket` / `threading` 这层可能被替换的 API 上。

如果目标进程已经加载了 `gevent.monkey`，`get_original()` 可以拿到 monkey patch 之前的对象。`native_start_new_thread()` 借这个接口，在 gevent 已经 patch 过 `_thread` 时仍然取回原始线程入口。

> 控制面越依赖目标进程的高层 runtime，越容易被业务框架的 monkey patch 影响。

---

### listen 之后发信号，再用 hello 探测确认可用

上面的代码里少了一步：启动 accept loop 后，没有等它显式报告“我已经跑起来了”。这一步可以不等。

对 socket server 来说，更关键的边界是 `listen()`。`listen()` 返回后，socket 已经开始监听，连接可以进入 backlog。即使 accept loop 还没执行到 `accept()`，外部 client 的连接也可以先排队。

控制面可以按这个顺序走：

1. 创建原生 Unix socket；
2. bind；
3. listen；
4. 启动原生线程跑 accept loop；
5. 发出就绪信号；
6. 外部 client 连接后，再做一次 hello 探测。

就绪信号只表示“agent 已完成基本初始化”。写文件、回连 notify socket、发送 IPC 消息都只是传递方式；命令循环是否真的可用，交给后续 hello 探测确认。

这样一来，就绪同步由内核 socket backlog 和外部 hello 探测承接，发出就绪信号之前不需要 Python 层 event。

---

## 数据面：命令能执行，不代表观测语义完全相同

### 不同命令在 gevent 下能看到的范围不同

控制面修完，只能说明 agent 能活下来，命令通道能响应。它不自动保证所有诊断命令在 gevent 下都和普通线程模型完全等价。

数据面要处理的就是这些语义差异。

`watch`、`monitor`、`stack`、`trace`、`top` 看起来都在“观察目标进程”，但它们依赖的机制不同：

| 命令 | 主要机制 | gevent 下需要标明的边界 |
|---|---|---|
| `watch` | wrapper 包住目标函数 | 观察函数入口、返回值、异常和耗时 |
| `monitor` | wrapper 统计调用 | 统计函数调用次数、成功率和响应时间 |
| `stack` | 在函数入口捕获 stack | 记录进入目标函数时的调用栈 |
| `trace` | tracing backend 展开调用树 | 可能只能保留根调用，不能承诺完整子调用树 |
| `top` | 周期性采样 frame | 可能只能看到当前活跃 greenlet，不能代表所有挂起 greenlet |

前几类 wrapper 观测仍然比较清楚：它们关注的是目标函数边界。只要 wrapper 真的包住了目标函数，就可以说清楚这次调用的参数、返回、异常和耗时。

`trace` 和 `top` 更复杂，因为它们试图观察更深的执行过程。

这时输出仍然可以给，但要把 backend、观测范围和降级原因一起带出来，避免调用方把降级结果当成完整结果。

---

### trace 只保留根调用，不承诺完整调用树

完整 `trace` 通常依赖解释器的 tracing 机制。Python 3.12+ 可以用 `sys.monitoring`，更老版本通常用 `sys.settrace`。这类机制会进入 frame 事件流，用来展开函数内部的子调用。

在 gevent patched 或 active hub 状态下，递归 tracing 更容易碰到调度和 frame 栈边界。此时更稳的选择是降级成 `wrapper_only`：

```text
trace target()
  -> wrapper 记录 target() 总耗时
  -> 不展开 sys.settrace / sys.monitoring 子调用树
```

这样做之后，`trace` 仍然能回答几个问题：

1. 目标函数有没有被调用；
2. 这次调用总共花了多久；
3. 它有没有抛异常；
4. 入参和返回值是什么。

但它不再承诺完整的内部调用树。

输出里的 `meta` 可以直接标出这次降级：

```json
{
  "meta": {
    "gevent_state": "patched",
    "backend": "wrapper_only",
    "degraded_reason": "recursive tracing is disabled under gevent"
  }
}
```

宁愿给一个明确降级的根调用结果，也不要给一棵看似完整、实际不可靠的调用树。

---

### top 的 frame sampling 看不见所有挂起 greenlet

`top` 的普通做法是 frame sampling：定期读取各 OS thread 当前正在执行的 frame，再按函数聚合热点。

这套采样模型在普通线程程序里比较直观。但 gevent 使用 greenlet 调度，多个 greenlet 共享同一个 OS thread。

`sys._current_frames()` 返回的是每个 OS thread 当前正在执行的 frame。它天然更容易看到“现在正在跑的 greenlet”，看不到所有挂起的 greenlet。

所以在 gevent 下，`top` 的结果仍然有用，但覆盖范围有限。它适合说明当前活跃执行路径的热点，不适合被解释成“所有 greenlet 的完整 CPU 画像”。

如果工具接入了 greenlet 的 switch / throw trace 事件，可以多保留一些调度信息；但挂起 greenlet 的完整栈枚举仍然不能随口承诺。

输出里的 `meta` 也要标出采样盲区：

```json
{
  "meta": {
    "gevent_state": "patched",
    "backend": "greenlet_aware_sampling",
    "greenlet_blind": true,
    "degraded_reason": "frame sampling sees the active greenlet per OS thread, not every suspended greenlet"
  }
}
```

`greenlet_blind` 标出来以后，这份热点列表的边界也就清楚了：它可以参考，但不是完整世界。

---

## 排查与结论：先确认 runtime，再解释输出边界

### 先确认目标 runtime，再解释启动超时和降级输出

调这类问题时，别先把目标进程当成普通 Python 进程。

更可靠的顺序是：

1. 先确认目标进程是否已经做过 gevent monkey patch；
2. 再看 agent 控制面是否使用了原生 socket 和原生线程入口；
3. 最后解释 `trace`、`top` 这类数据面输出的边界。

在 peeka 里，这类信息可以通过类似 `patch-status` 的命令摆到台面上。它不直接修复问题，只负责把当前 runtime 长什么样说清楚：

- gevent 是否已经 import；
- gevent hub 是否 active；
- `socket.socket`、`threading.Event` 这类对象是否已经不是原始对象；
- 当前命令输出是否带有降级说明。

有了这些信息，`Agent initialization timeout` 就不再只是一个模糊的启动超时。它到底是注入路径失败，还是 agent 进入目标进程后踩到了被改造过的 runtime，可以继续往下分。

---

### 控制面先保活，数据面再标注观测范围

attach agent 运行在别人的进程里。那个进程的标准库、线程模型、socket 行为和 frame 调度模型，都可能已经被业务框架改造过。

处理时也要分成两层。

控制面先保证 agent 能活下来：

1. 用 `_socket.socket` 这类底层接口建立命令 socket，不再通过 `socket.socket`；
2. 用原始 `_thread.start_new_thread` 这类底层入口启动控制线程，不再通过 `threading.Thread`；
3. 用 `listen()` 后的 backlog 承接早到的 client 连接，把同步点从 Python 层 event 挪到 socket backlog；
4. 发出就绪信号后，再用外部 hello 探测确认命令循环可用，避免在发出就绪信号前进入 `threading.Event.wait()`。

数据面再说明诊断结果的覆盖范围：

1. `watch`、`monitor`、`stack` 可以保持函数边界观测；
2. `trace` 在 gevent 下可以降级到 `wrapper_only`；
3. `top` 可以采样当前活跃执行路径，但要承认 greenlet 盲区；
4. 受影响输出要带 `meta`，让调用方知道 backend 和降级原因。

`Agent initialization timeout` 只是外部看到的结果。agent 跨进了一个已经被改造过的 runtime，控制面要尽量退到底层原语，数据面要清楚声明自己还能看见什么。
