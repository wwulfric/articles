---
title: gevent monkey patch 下的 attach agent：先活下来，再诚实观测
date: 2026-05-08 00:00
categories: [技术]
tags: [python, gevent, debug]
---

上一篇《理解 Python 进程注入：从 ptrace 到 PEP 768》讲的是怎么把一段代码送进正在运行的 Python 进程。GDB 直接调用 Python C API、`dlopen + pthread`、PEP 768，解决的都是“怎么进去”。

进去之后还有另一件事：这段 agent 代码会直接落进目标进程自己的 Python runtime；那个 runtime 可能已经被业务框架改过。

这个区别平时不明显。直到目标进程在启动早期执行过：

```python
from gevent import monkey
monkey.patch_all()
```

这时 `socket`、`threading`、`time`、`select` 等标准库对象可能已经被替换。attach agent 如果直接拿这些高层 API 做自己的命令通道，就可能在启动阶段卡住，外部只看到一个很普通的错误：

```text
Agent initialization timeout (ready file)
```

这个 timeout 容易误导人。它看起来像 attach 没成功，或者 ready 文件等待时间太短。实际更常见的情况是：agent 已经被执行了，但它死在写 ready 文件之前。

这篇文章想讲清楚两个边界：

1. 控制面：agent 自己怎么创建 socket、线程、锁、ready 信号，先保证能活下来；
2. 数据面：agent 活下来以后，`watch`、`trace`、`top` 这些诊断命令在 gevent runtime 下还能承诺什么。

文章如果停在控制面，读者会知道 timeout 从哪来，却不知道后续诊断结果有哪些边界。

---

## timeout 发生在 ready 之前

一个 attach agent 的最小流程通常是这样：

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

agent 在目标进程里做几件事：

- 创建 Unix socket；
- 启动后台 accept loop；
- 写 ready 文件或回连 notify socket；
- 后续命令通过这个 socket 进入 agent。

如果是普通 Python 进程，下面这段代码看起来很自然：

```python
import os
import socket
import threading
from pathlib import Path


class MiniAgent:
    def __init__(self, session_id):
        self.sock_path = f"/tmp/agent_{session_id}.sock"
        self.ready_path = f"/tmp/agent_{session_id}.ready"
        self.server = None

    def start(self):
        try:
            os.unlink(self.sock_path)
        except FileNotFoundError:
            pass

        self.server = socket.socket(socket.AF_UNIX, socket.SOCK_STREAM)
        self.server.bind(self.sock_path)
        self.server.listen(5)

        accept_ready = threading.Event()
        thread = threading.Thread(
            target=self._accept_loop,
            args=(accept_ready,),
            daemon=True,
        )
        thread.start()
        accept_ready.wait(timeout=10)

        Path(self.ready_path).touch()

    def _accept_loop(self, accept_ready):
        accept_ready.set()
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

问题出在 attach agent 的运行位置。它落在业务进程里，沿用业务进程已经改过的 runtime。目标应用如果已经做过 gevent monkey patch，这里的 `socket.socket`、`threading.Thread`、`threading.Event` 可能已经换成 gevent 版本。

最明显的风险是 `accept_ready.wait()`。更隐蔽的风险在 `threading.Thread.start()` 内部。

CPython 的 `Thread.start()` 会启动底层线程，然后等待 `_started` 事件：

```python
def start(self):
    _start_new_thread(self._bootstrap, ())
    self._started.wait()
```

如果 `threading` 被 gevent 改造过，这个 wait 可能落到 gevent 的 event 语义里。在某些注入时机下，调用链会变成：

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

这个异常指向当前执行点里的 gevent hub 切换限制，和线程创建本身、GDB attach 无关。

最后外部收到的是：

```text
Agent initialization timeout (ready file)
```

ready timeout 是表象。真正的错误在目标进程内部的 agent log 里。

---

## 控制面先退到底层原语

attach 方法换了，运行时边界仍在。GDB、`dlopen + pthread`、PEP 768 决定 agent 怎么进入目标进程；最后执行 Python agent 时，它面对的仍然是目标进程当前的 runtime。

所以控制面的原则很简单：

> agent 自己的命令通道应该避开目标应用可能 monkey patch 的高层 API。

peeka 0.1.13 之后把这层抽成 Runtime Primitive Layer。核心做法是：避免主动 import gevent；目标进程已经加载 `gevent.monkey` 时，再尝试拿原始对象；socket 直接使用 `_socket.socket`，避开 `socket` 模块层替换。

简化后大致是：

```python
import _socket
import _thread
import sys
import threading
import time


def get_original_runtime_attr(module_name, attr_name, fallback):
    monkey = sys.modules.get("gevent.monkey")
    get_original = getattr(monkey, "get_original", None)
    if callable(get_original):
        try:
            return get_original(module_name, attr_name)
        except Exception:
            pass
    return fallback


NativeSocket = _socket.socket
start_new_thread = get_original_runtime_attr(
    "_thread",
    "start_new_thread",
    _thread.start_new_thread,
)
allocate_lock = get_original_runtime_attr(
    "_thread",
    "allocate_lock",
    _thread.allocate_lock,
)
NativeEvent = get_original_runtime_attr(
    "threading",
    "Event",
    threading.Event,
)
native_time = get_original_runtime_attr(
    "time",
    "time",
    time.time,
)
```

这样改以后，agent 的启动代码可以去掉 `threading.Thread` 和 `threading.Event`：

```python
import os
import socket
from pathlib import Path


def create_socket(family, type_):
    return NativeSocket(family, type_)


def start_thread(target, args=()):
    return start_new_thread(target, args)


def native_accept(server):
    accept = getattr(server, "accept", None)
    if callable(accept):
        return accept()

    fd, address = server._accept()
    conn = NativeSocket(server.family, server.type, server.proto, fileno=fd)
    return conn, address


class MiniAgent:
    def __init__(self, session_id):
        self.sock_path = f"/tmp/agent_{session_id}.sock"
        self.ready_path = f"/tmp/agent_{session_id}.ready"
        self.server = None
        self.closed = False
        self.lock = allocate_lock()

    def start(self):
        try:
            os.unlink(self.sock_path)
        except FileNotFoundError:
            pass

        self.server = create_socket(socket.AF_UNIX, socket.SOCK_STREAM)
        self.server.bind(self.sock_path)
        self.server.listen(5)

        start_thread(self._accept_loop)
        Path(self.ready_path).touch()

    def _accept_loop(self):
        while not self.closed:
            conn, _ = native_accept(self.server)
            start_thread(self._handle_client, (conn,))

    def _handle_client(self, conn):
        try:
            conn.sendall(b"hello\n")
        finally:
            conn.close()
```

变化不多，但性质变了：

- socket 使用原始 `_socket.socket`；
- accept 收口到 native helper；
- 线程使用 `_thread.start_new_thread`；
- 绕开 `threading.Thread.start()` 里的 `_started`；
- 去掉 `threading.Event.wait()` 这类 ready barrier。

这里少了一个等待：启动 accept loop 后没有等它显式报告“我已经启动”。这是刻意的取舍。

对 socket server 来说，关键点是 `listen()`。`listen()` 返回后，socket 已经开始监听，连接可以进入 backlog。即使 accept loop 还没执行到 `accept()`，client 连接仍然可以排队等待，无需依赖 Python 层 event。

更稳的顺序是：

1. 创建原始 Unix socket；
2. bind；
3. listen；
4. 启动原始线程跑 accept loop；
5. 写 ready 文件或发送 READY notify；
6. 外部 client 连接后再做一次 hello 探测。

ready 文件表达的是初始化进度；命令循环健康度交给 hello 探测确认。

---

## 控制面稳定以后，数据面仍然会变形

控制面修完，agent 能活下来，命令通道能响应；诊断命令在 gevent 下的语义还要单独处理。

这是第二条边界。

`watch`、`monitor`、`stack`、`trace`、`top` 看起来都是“观察目标进程”，但它们依赖的机制不同：

| 命令 | 主要机制 |
|---|---|
| `watch` | wrapper 包住目标函数，记录入参、返回值、异常和耗时 |
| `monitor` | wrapper 统计调用数、成功率和响应时间 |
| `stack` | 在函数入口处捕获当前调用栈 |
| `trace` | 用 tracing backend 展开调用树 |
| `top` | 周期性采样线程 frame，聚合热点函数 |

在 gevent 下，前几类 wrapper/入口观测仍然有明确语义：它们观察的是目标函数调用边界。`trace` 和 `top` 就麻烦得多。

peeka 0.1.13 之后引入了数据面兼容矩阵，放弃“gevent 下所有命令都完全等价”的假设。探测逻辑也保持零副作用：读取 `sys.modules` 里已有的 `gevent.monkey`、`gevent.hub`，避免主动 import gevent。

矩阵可以简化成这样：

| 命令 | gevent patched / active hub 下的策略 |
|---|---|
| `watch` | safe，继续 wrapper 观测 |
| `monitor` | safe，继续 wrapper 统计 |
| `stack` | safe，继续入口 stack 捕获 |
| `trace` | degraded，降级到 `wrapper_only` |
| `top` | degraded，使用 `greenlet_aware_sampling`，并标记 `greenlet_blind` |

这张表把重点放在观测范围：每个命令还能看见什么。

---

## trace 保留根调用

`trace` 的完整调用树依赖全局 tracing backend。Python 3.12+ 可以用 `sys.monitoring`，更老版本通常用 `sys.settrace`。这类机制会进入解释器的 frame 事件流。

gevent patched 或 active hub 状态下，peeka 选择避开递归 tracing，强制使用 `wrapper_only`：

```text
trace target()
  -> wrapper 记录 target() 总耗时
  -> 跳过 sys.settrace / sys.monitoring 子调用展开
```

这时 `trace` 仍然能告诉你入口调用花了多久、有没有抛异常，同时明确放弃完整子调用树。

输出里的 `meta` 会把这个说清楚：

```json
{
  "meta": {
    "gevent_state": "patched",
    "backend": "wrapper_only",
    "greenlet_blind": false,
    "degraded_reason": "sys.settrace under gevent can violate frame stack invariants; using wrapper-only tracing without recursive call tree"
  }
}
```

这个降级牺牲了完整性，换来更可靠的输出：一棵看似完整、实际不可靠的调用树更危险。

---

## top 需要承认 greenlet blind

`top` 的普通做法是 frame walk：定期读取各 OS thread 当前 frame，再按函数聚合热点。

gevent 的调度模型会让这个假设变弱。多个 greenlet 共享同一个 OS thread。`sys._current_frames()` 返回每个 OS thread 当前正在执行的 frame；挂起的 greenlet 栈没有进入这份样本。

因此，普通 frame walk 在 gevent 下天然会漏掉一部分上下文。

当前源码做了一个折中：如果 `greenlet` 已经加载并提供 `gettrace` / `settrace`，`top` 会安装一个链式 greenlet tracer，记录 switch / throw 事件，并在退出时恢复之前的 tracer。这样能多保留一些 greenlet 调度信息；挂起 greenlet 的完整栈枚举仍然超出这个 backend 的能力范围。

所以它继续标记：

```json
{
  "meta": {
    "gevent_state": "patched",
    "backend": "greenlet_aware_sampling",
    "greenlet_blind": true,
    "degraded_reason": "Frame sampling under gevent only sees the active greenlet per OS thread. Suspended greenlets are not represented."
  }
}
```

这里的 `greenlet_blind` 很重要。它提醒读者：这份热点列表有价值，覆盖范围也有限。

---

## patch-status 把目标 runtime 摆到台面上

调这类问题时，最怕把目标进程当成普通 Python 进程。`patch-status` 的价值就在这里。

它读取并报告 agent 看到的 runtime：

- gevent / eventlet 是否已经 import 或 active；
- 当前 `socket.socket`、`threading.Event` 等对象是否仍等于 RPL 捕获的 native 对象；
- RPL 自检是否通过。

遇到 gevent 目标进程时，先看 `patch-status`，再读 `trace` / `top` 输出里的 `meta`。这比反复盯着 ready timeout 更快。

---

## 结论

attach 工具运行在别人的进程里。那个进程的标准库、线程模型、socket 行为、frame 调度模型，都可能已经被业务框架改造过。

所以这类问题要分两层处理。

控制面要先活下来：

1. 避开被 monkey patch 的 `socket` 和 `threading`；
2. 如果 gevent 已经存在，使用 `gevent.monkey.get_original()` 取原始对象；
3. 用 `_thread.start_new_thread` 替代 `threading.Thread` 启动控制线程；
4. 去掉 `threading.Event.wait()` 这种 ready barrier；
5. 用 `listen()` 后的 socket backlog 加外部 hello 探测完成同步。

数据面要诚实：

1. `watch`、`monitor`、`stack` 可以保持函数边界观测；
2. `trace` 在 gevent patched / active hub 下使用 `wrapper_only`；
3. `top` 可以接入 greenlet trace 事件，但仍要标记 `greenlet_blind`；
4. 受影响输出要带 `meta`，让调用方知道 backend 和降级原因。

`Agent initialization timeout (ready file)` 是表象。真正的问题是 agent 跨进了一个已经被改造过的 runtime。控制面要尽量退到底层原语，数据面要清楚声明自己还能看见什么。
