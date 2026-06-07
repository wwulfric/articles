---
title: gevent monkey patch 对 peeka 工具的影响
date: 2026-05-08 00:00
categories: [技术]
tags: [python, gevent, debug]
---

上一篇文章中，我们提到了几个常见的 python 进程诊断工具。其中提到的 [peeka](https://github.com/peeka-project/peeka) 就属于 attach agent 这一类。用户在外部选择一个正在运行的 Python 进程，peeka 把一段 agent 代码送进去；agent 在目标进程里建立命令通道，后续 `watch`、`trace`、`top` 等命令再通过这个通道执行。它更接近运行时诊断工具，而不是离线性能分析器。

运行时诊断工具可以分成控制面和数据面两层：

- 控制面：工具为了启动、attach、通信、接收命令、返回结果而维护的基础设施。
- 数据面：工具负责观察程序的路径，比如函数 wrapper、调用栈采样、tracing backend、内存分配 hook。

不同工具的控制面和数据面位置不一样。`cProfile` 基本都在目标进程内部；`py-spy` 的控制面在外部，数据采样也尽量从外部完成；memray 会把采集逻辑放进目标进程内部，再把数据写到外部可分析的文件；peeka 的控制面横跨内外两侧，外部 CLI 负责发命令，目标进程里的 agent 负责接命令和执行观测。

这种差异平时不明显。一旦目标进程使用 gevent，问题就会浮现出来。gevent 通过 monkey patch 把 `socket`、`threading`、`time`、`select` 等标准库对象替换成协程版本。对业务代码来说，使用 gevent 的目的就是使用其协程能力；对诊断工具来说，它可能会破坏控制面和数据面的正常工作。

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

具体工具可能会把这个超时写成等待某个文件、notify socket 或 IPC 消息超时。传递方式不同，但外部控制端等的都是同一件事：agent 发出「我已经启动完成」的就绪信号。

这个启动超时容易被误读成「注入失败」或者「等待时间太短」。实际更常见的情况是：agent 代码已经在目标进程里开始执行，但在发出就绪信号之前阻塞在启动路径上，或者抛出异常中断；外部控制端只能继续等待，最后表现为初始化超时。

---

## 1. 问题背景：gevent 会影响 attach agent 的控制面

gevent 的 monkey patch 会把 `socket`、`threading`、`time`、`select` 等标准库接口替换成协作式实现，让业务 I/O 在等待时主动让出执行权；如果 attach agent 的启动路径也直接使用这些模块，控制面就可能在建 socket、起线程或等待就绪信号时出问题。

### 1.1 attach agent 的启动链路：就绪信号只代表控制面初始化

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
        # 1. 创建本地 socket。
        self.server = socket.socket(socket.AF_UNIX, socket.SOCK_STREAM)
        self.server.bind(self.sock_path)
        self.server.listen(5)

        # 2. 启动后台 accept loop。
        accept_started = threading.Event()
        thread = threading.Thread(
            target=self._accept_loop,
            args=(accept_started,),
            daemon=True,
        )
        thread.start()
        accept_started.wait(timeout=10)

        # 3. 发出就绪信号。
        notify_started()

    def _accept_loop(self, accept_started):
        accept_started.set()
        while True:
            # 4. 等外部 client 连上来，接收后续命令。
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

### 1.2 gevent monkey patch 会让 agent 卡在就绪信号之前

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

常见错误是 `BlockingSwitchOutError`[^blocking-switch-out]：

```text
gevent.exceptions.BlockingSwitchOutError:
Impossible to call blocking function in the event loop callback
```

这个异常和 GDB、LLDB、PEP 768[^pep-768] 这些注入方式没有直接关系。agent 已经进入目标进程，只是在执行过程中撞到了 gevent 对阻塞切换的限制。

外部最终只等到：

```text
Agent initialization timeout
```

这时排查重点在就绪信号之前的启动路径：socket 创建、线程启动、同步等待这些对象，是否已经来自 gevent monkey patch 后的实现。

---

## 2. 控制面优化：启动和通信需要避开 monkey patch

控制面要尽量减少对 gevent 协作调度的依赖。socket、后台线程、就绪通知这些基础设施属于诊断工具自身的启动和通信路径，适合使用稳定的底层原语；目标应用已经改过的 `socket` 和 `threading` 不适合作为这条路径的基础。这一层先保证外部 client 能连上 agent；诊断结果能看到多少，再放到数据面里说明。

### 2.1 用原生 socket 和线程入口建立命令通道

控制面的第一目标是让 agent 先跑起来。

「跑起来」不涉及诊断结果是否准确，只要求 agent 至少完成几件事：

1. 在目标进程里启动；
2. 建好命令 socket；
3. 发出就绪信号；
4. 响应外部 client 的第一次连接。

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
        # 1. 创建原生 Unix socket。
        self.server = _socket.socket(_socket.AF_UNIX, _socket.SOCK_STREAM)

        # 2. bind 到本地路径。
        self.server.bind(self.sock_path)

        # 3. listen 返回后，外部连接可以先进入 backlog。
        self.server.listen(5)

        # 4. 启动原生线程跑 accept loop。
        start_thread = native_start_new_thread()
        start_thread(self._accept_loop, ())

        # 5. 发出就绪信号；不再等待 Python 层 Event。
        notify_started()

    def _accept_loop(self):
        start_thread = native_start_new_thread()
        while True:
            # 第 6 步由外部 client 发起；这里负责 accept 并派发处理。
            conn, _ = self.server.accept()
            start_thread(self._handle_client, (conn,))

    def _handle_client(self, conn):
        # hello 探测用的最小响应。
        conn.sendall(b"hello\n")
        conn.close()
```

改动看起来不大，但依赖的对象已经换了：

- socket 改用底层 `_socket.socket`，不再通过 `socket.socket` 建命令通道；
- 线程改用原始 `_thread.start_new_thread`，不再通过 `threading.Thread` 启动控制线程；
- 就绪同步交给 `listen()` 后的 socket backlog 和外部 hello 探测，不再用 `threading.Event.wait()` 作为发出就绪信号前的 barrier。这部分在下一个小节会涉及；
- agent 自己的命令通道整体退到 `_socket` / `_thread` 这组底层原语上，不再把控制线程和控制 socket 建在 `socket` / `threading` 这层可能被替换的 API 上。

如果目标进程已经加载了 `gevent.monkey`，`monkey.get_original()` 可以拿到 monkey patch 之前的对象。`native_start_new_thread()` 借这个接口，在 gevent 已经 patch 过 `_thread` 时仍然取回原始线程入口。

> 控制面越依赖目标进程的高层 runtime，越容易被业务框架的 monkey patch 影响。

---

### 2.2 listen 之后发信号，再用 hello 探测确认可用

上面的代码里少了一步：启动 accept loop 后，没有等它显式报告「我已经跑起来了」。这一步可以不等。

对 socket server 来说，更关键的边界是 `listen()`。`listen()` 返回后，socket 已经开始监听，连接可以进入 backlog。即使 accept loop 还没执行到 `accept()`，外部 client 的连接也可以先排队。

控制面可以按这个顺序走：

1. 创建原生 Unix socket；
2. bind；
3. listen；
4. 启动原生线程跑 accept loop；
5. 发出就绪信号；
6. 外部 client 连接后，再做一次 hello 探测。

第 1 到第 5 步对应上节 agent 代码里的注释。第 6 步发生在外部控制端，用来确认 agent 的 accept loop 和 client handler 至少能处理一次连接并写回响应。外部控制端不在目标 gevent 进程里，这里用普通 `socket` 即可：

```python
import socket


def probe_agent(sock_path, timeout=1.0):
    with socket.socket(socket.AF_UNIX, socket.SOCK_STREAM) as client:
        client.settimeout(timeout)
        client.connect(sock_path)
        return client.recv(len(b"hello\n")) == b"hello\n"
```

就绪信号只表示「agent 已完成基本初始化」。写文件、回连 notify socket、发送 IPC 消息都只是传递方式；agent 是否真的能接收连接并返回响应，交给后续 hello 探测确认。

这样一来，就绪同步由内核 socket backlog 和外部 hello 探测承接，发出就绪信号之前不需要 Python 层 event。

---

## 3. 数据面：命令能执行，不代表观测语义完全相同

命令通道稳定以后，问题还没有结束。agent 能接收命令，只说明 socket、后台线程和 hello 探测这些控制面路径已经可用；真正执行 `watch`、`trace`、`top` 时，代码会进入函数 wrapper、tracing backend、frame sampling 这些数据面路径。它们依赖的观测机制不同，受 gevent 影响的方式也不同，所以还要继续看数据面：命令能返回结果是一回事，结果能解释到什么程度是另一回事。

### 3.1 不同命令在 gevent 下能看到的范围不同

`watch`、`monitor`、`stack`、`trace`、`top` 看起来都在「观察目标进程」，但它们依赖的机制不同：

| 命令 | 主要机制 | gevent 下需要标明的边界 |
|---|---|---|
| `watch` | wrapper 包住目标函数 | 观察函数入口、返回值、异常和耗时 |
| `monitor` | wrapper 统计调用 | 统计函数调用次数、成功率和响应时间 |
| `stack` | 在函数入口捕获 stack | 记录进入目标函数时的调用栈 |
| `trace` | tracing backend 展开调用树 | 可能只能保留根调用，不能承诺完整子调用树 |
| `top` | 周期性采样 frame | 可能只能看到当前活跃 greenlet，不能代表所有挂起 greenlet |

前几类 wrapper 观测仍然比较清楚：它们关注的是目标函数边界。只要 wrapper 真的包住了目标函数，就可以说清楚这次调用的参数、返回、异常和耗时。

`trace` 和 `top` 更复杂，因为它们试图观察更深的执行过程。

这时输出仍然可以给，但应该把 backend、观测范围和降级原因一起带出来，避免调用方把降级结果当成完整结果。

---

### 3.2 trace 只保留根调用，不承诺完整调用树

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

### 3.3 top 的 frame sampling 看不见所有挂起 greenlet

`top` 的普通做法是 frame sampling：定期读取各 OS thread 当前正在执行的 frame，再按函数聚合热点。

这套采样模型在普通线程程序里比较直观。但 gevent 使用 greenlet 调度，多个 greenlet 共享同一个 OS thread。

`sys._current_frames()` 返回的是每个 OS thread 当前正在执行的 frame。它天然更容易看到「现在正在跑的 greenlet」，看不到所有挂起的 greenlet。

所以在 gevent 下，`top` 的结果仍然有用，但覆盖范围有限。它适合说明当前活跃执行路径的热点，不适合被解释成「所有 greenlet 的完整 CPU 画像」。

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

## 4. 总结：控制面先保活，数据面说明边界

gevent 带来的问题不是单点故障。它先可能让 attach agent 的控制面卡在 socket、线程、就绪信号这些启动路径上；控制面修好以后，数据面的结果也不能直接按普通线程模型解释。

处理顺序可以简单分成两步：

1. 控制面先退到底层原语，用 `_socket`、原始 `_thread.start_new_thread` 和 hello 探测保证 agent 能启动、能通信；
2. 数据面再按命令说明观测范围，`watch`、`monitor`、`stack` 保持函数边界观测，`trace` 和 `top` 在 gevent 下要标出降级和盲区。

外部最先看到的可能只是 `Agent initialization timeout`，但它只说明控制面没有完成就绪流程。继续往下排查时，要分清 agent 是不是已经进入目标进程、命令通道能不能建立，以及命令返回的数据还能代表什么。

[^blocking-switch-out]: gevent API 文档：[`BlockingSwitchOutError`](https://www.gevent.org/api/gevent.hub.html#gevent.hub.BlockingSwitchOutError)。
[^pep-768]: [PEP 768: Safe external debugger interface for CPython](https://peps.python.org/pep-0768/)。
