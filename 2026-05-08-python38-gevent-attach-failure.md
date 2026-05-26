---
title: Python attach agent 遇到 gevent monkey patch：控制面超时与数据面降级
date: 2026-05-08 00:00
categories: [技术]
tags: [python, gevent, debug]
---

上一篇《理解 Python 进程注入：从 ptrace 到 PEP 768》写了 Python 进程注入的几种方式：GDB 直接调用 Python C API、`dlopen + pthread`、以及 Python 3.14 之后的 PEP 768。

这些方案解决的是“如何把一段代码送进一个正在运行的 Python 进程”。但代码被送进去之后，还有另一个问题：**这段代码会运行在目标进程自己的 Python runtime 里**。

如果目标进程是一个普通 Python 程序，这通常没什么特别。但如果它在启动早期执行过 gevent monkey patch，`socket`、`threading`、`time` 这些标准库模块就可能已经被替换。一个 attach agent 如果把自己的控制通道建立在这些高层 API 上，就可能在启动阶段出现很难从外部理解的 timeout；即使 agent 启动成功，后续诊断命令的观测语义也可能被 gevent 的调度模型改变。

本文用一个最小模型解释这个问题，再对照 peeka 0.1.13 之后的实现看两层处理：

1. 控制面：agent 自己如何创建 socket、线程、锁、ready 信号；
2. 数据面：`watch`、`trace`、`top` 这类命令如何在 gevent runtime 下保持可靠输出。

前者决定 agent 能不能稳定活下来，后者决定诊断结果会不会被误读。

---

## 最小 attach 模型

先把进程注入抽象成一张图：

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

不管底层 attach 是怎么做的，最终都会来到同一个点：一段 Python agent 代码开始在目标解释器里运行。

这段 agent 通常会做几件事：

- 创建一个 Unix socket，用来接收后续诊断命令；
- 启动后台循环，持续 `accept()` client 连接；
- 写一个 ready 文件，告诉外部工具“agent 已经准备好了”；
- 后续命令都通过这个 socket 进来。

ready 文件只是一个同步信号。例如：

```text
/tmp/agent_<session>.ready
```

它的作用是避免外部工具过早连接 socket。外部工具通常会做类似这样的等待：

```python
deadline = time.time() + 10
while time.time() < deadline:
    if ready_path.exists():
        break
    time.sleep(0.05)
else:
    raise TimeoutError("Agent initialization timeout (ready file)")
```

如果 ready 文件迟迟没有出现，外部只能报 timeout。但 timeout 本身不说明根因。它只说明 agent 没有走到“写 ready”这一步。

---

## 一个看起来正常的 agent

先写一个最小 agent。它不做任何复杂诊断，只创建 socket、启动 accept loop、写 ready 文件。

```python
import os
import socket
import threading
from pathlib import Path


class MiniAgent:
    def __init__(self, session_id):
        self.session_id = session_id
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

这段代码在一个干净的 Python 进程里没有问题：

1. `socket.socket()` 创建 Unix socket；
2. `listen()` 开始监听；
3. `threading.Thread()` 启动后台 accept loop；
4. `threading.Event()` 用来确认 accept loop 已经进入工作状态；
5. ready 文件写出；
6. 外部 client 再连接 socket。

如果你写的是一个普通服务端程序，这段代码非常自然。

但 attach agent 有一个特殊点：它不是运行在自己启动的干净进程里，而是运行在别人的进程里。

---

## 被 gevent 改造过的目标进程

很多 gevent 程序会在入口处执行 monkey patch：

```python
from gevent import monkey
monkey.patch_all()

from my_web_app import app

app.serve_forever()
```

`monkey.patch_all()` 会替换标准库中的一批阻塞 API，让它们进入 gevent 的协程调度语义。常见受影响对象包括：

- `socket.socket`
- `time.sleep`
- `select`
- `threading` 里的部分同步原语

业务代码仍然写：

```python
data = sock.recv(4096)
time.sleep(1)
```

但底层行为已经不再是普通阻塞调用，而是可能让出到 gevent hub，由协程调度器安排。

这对业务程序是好事。问题在于，attach agent 注入进去以后，看到的也是这个已经被 patch 过的世界。

也就是说，agent 里的这几行：

```python
import socket
import threading

socket.socket(...)
threading.Event()
threading.Thread(...)
```

拿到的不一定是 CPython 启动时的原始对象，而可能是 gevent patch 之后的对象。

---

## 失败是怎么发生的

前面的 agent 启动代码里，最危险的是这几行：

```python
accept_ready = threading.Event()
thread = threading.Thread(
    target=self._accept_loop,
    args=(accept_ready,),
    daemon=True,
)
thread.start()
accept_ready.wait(timeout=10)
```

第一眼看，风险似乎在最后一行 `accept_ready.wait()`。如果 gevent patch 了 `threading.Event`，这个 wait 就可能落入 gevent 的 event 实现。

但还有一个更隐蔽的位置：`threading.Thread.start()` 内部也会等待线程真正启动。

CPython 的 `Thread.start()` 大致会做这样的事：

```python
def start(self):
    _start_new_thread(self._bootstrap, ())
    self._started.wait()
```

这里的 `_started` 也是一个 event。正常情况下，这只是一个很短的同步等待。但如果 `threading` 已经被 gevent monkey patch，这个 wait 也可能变成 gevent event wait。

于是调用链会变成这样：

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

这类错误的意思不是“不能创建线程”，也不是“GDB attach 失败”。它更接近于：

> 当前执行点不允许做这种会切换 gevent hub 的阻塞等待。

对于 attach agent 来说，结果就是：

1. agent 代码确实被执行了；
2. socket 可能已经创建；
3. log 可能已经写出；
4. 但 agent 在写 ready 文件之前异常退出；
5. 外部工具最后只能看到 ready file timeout。

所以 `Agent initialization timeout (ready file)` 往往只是表象。真正的错误在目标进程内部的 agent log 里。

---

## 一个最小复现模型

真实 attach 需要 GDB、LLDB 或 PEP 768，不适合放在最小示例里。我们可以用伪代码模拟关键条件。

目标进程：

```python
# target.py
from gevent import monkey
monkey.patch_all()

import gevent


def business_task():
    while True:
        gevent.sleep(1)


gevent.spawn(business_task)
gevent.wait()
```

外部 attach 工具最终会让目标进程执行一段 agent 代码。把这个动作抽象成：

```python
# injected by debugger / remote exec / native injector
exec(agent_script)
MiniAgent(session_id).start()
```

如果 `MiniAgent.start()` 使用的是前面那版 `threading.Thread + threading.Event`，它就不再运行在“干净 Python”的假设下，而是运行在 gevent 改造过的 runtime 里。

真正的问题不是某一行 attach 命令，也不是 Python 3.8 本身，而是这条边界：

```text
诊断 agent 的控制面
    建立在
目标应用可能 monkey patch 的标准库 API
```

这条边界一旦混在一起，agent 的行为就会受目标应用运行模型影响。

---

## 先分清控制面和数据面

gevent 影响 attach agent 时，至少有两个层面：

| 层面 | 典型对象 | 主要风险 |
|---|---|---|
| 控制面 | agent socket、accept loop、client handler、ready/notify 信号 | agent 起不来，或者命令通道不响应 |
| 数据面 | `watch`、`trace`、`top`、`stack`、`monitor` | agent 能响应，但观测语义发生变化 |

控制面追求的是“agent 自己不要被目标 runtime 拖死”。数据面追求的是“命令输出不要假装自己比实际更完整”。这两层都要处理，不能只修 ready timeout。

---

## 控制面修复：隔离 runtime 原语

解决方向不是“换一种 attach 方法”。不同 attach 方法只决定 agent 如何进入目标进程；只要最终要在目标解释器里执行 Python agent，就仍然会遇到目标 runtime 已经被改造的问题。

更核心的原则是：

> attach agent 的控制面要尽量使用底层、可预测、少依赖的原语。

对这个例子来说，至少要避开这些对象：

- 被 patch 的 `socket.socket`
- 被 patch 的 `threading.Thread`
- 被 patch 的 `threading.Event`
- 控制面里的高层计时函数，例如 `time.time`

gevent 提供了一个有用的能力：如果当前进程已经加载了 `gevent.monkey`，可以通过 `get_original()` 拿回 monkey patch 之前的原始对象。

agent 不需要主动 import gevent，也不应该把 gevent 变成依赖。peeka 0.1.13 之后把这层抽成 Runtime Primitive Layer，大致是：

```python
import _socket
import _thread
import socket
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

这里有几个细节：

- 不主动 `import gevent`；
- 不要求目标进程一定使用 gevent；
- gevent 存在时尽量拿原始对象；
- socket 直接使用 `_socket.socket`，避开 `socket` 模块层的替换；
- 拿不到时回到当前 runtime 里的对象；
- 只影响 agent 自己的控制面，不修改业务代码。

---

## 修复后的最小 agent

基于这些原语，可以把 agent 改成这样：

```python
import os
import socket
from pathlib import Path


# 沿用上文定义的 NativeSocket、start_new_thread、allocate_lock。
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
        self.session_id = session_id
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

变化很少，但性质不同：

- socket 使用原始 `_socket.socket`；
- accept 也收口到 native helper；
- 线程使用 `_thread.start_new_thread`；
- 不再创建 `threading.Event`；
- 不再调用 `threading.Thread.start()`；
- 不再做 `Event.wait()` 这样的 Python 层 barrier。

这样就切断了原来的失败路径：

```text
threading.Thread.start()
  -> self._started.wait()
  -> gevent.event.Event.wait()
  -> BlockingSwitchOutError
```

这里用 `_thread` 是一个有意识的取舍。它比 `threading.Thread` 更低层，没有漂亮的对象封装，也没有 join、name、daemon 这些高级能力。但 agent 的控制面通常只需要启动一个长期运行的 accept loop 和若干短生命周期 client handler。为了避免踩到目标进程改写过的 `threading`，这个取舍是合理的。

---

## ready 文件还需要等待 accept loop 吗

前面的脆弱 agent 使用了一个 `accept_ready` event：

```python
thread.start()
accept_ready.wait(timeout=10)
Path(self.ready_path).touch()
```

它的意图是：确认 accept loop 已经启动，再告诉外部工具可以连接。

但这个等待并不是必须的。对于 socket server 来说，更关键的是：

```python
self.server.bind(self.sock_path)
self.server.listen(5)
```

`listen()` 返回后，socket 已经开始监听，连接可以进入 backlog。即使 accept loop 还没执行到 `accept()`，client 的连接请求也不需要依赖一个 Python 层 event 才能安全到达。

所以更稳的顺序是：

1. 创建原始 Unix socket；
2. bind；
3. listen；
4. 启动原始线程跑 accept loop；
5. 写 ready 文件；
6. 外部 client 连接后再做一次 hello 探测。

最后一步很重要。ready 文件只能说明 agent 走到了某个初始化阶段，不代表命令循环一定健康。外部工具在进入正式交互前，应该再连一次 socket，发一个最小 hello：

```python
def verify_agent(sock_path):
    client = socket.socket(socket.AF_UNIX, socket.SOCK_STREAM)
    client.settimeout(2)
    client.connect(sock_path)
    data = client.recv(1024)
    if data != b"hello\n":
        raise RuntimeError("agent hello failed")
```

这比在 agent 内部依赖 `threading.Event.wait()` 更适合 attach 场景：

- ready 文件负责进程间粗同步；
- hello 负责协议级健康检查；
- agent 内部少依赖目标 runtime 的高层同步原语。

---

## 这和 attach 方法有什么关系

上一篇文章里提到，Python 3.8 到 3.13 常见的较安全 attach 方案是 `dlopen + pthread`：

```text
GDB/LLDB attach
  -> dlopen(native injector)
  -> native injector 创建 pthread
  -> pthread 获取 GIL
  -> 执行 Python agent
```

这个方案解决的是“不要在 ptrace 暂停点直接执行复杂 Python 代码”。它能降低死锁、崩溃、时机不安全的风险。

但它不自动解决本文的问题。

原因很简单：不管 agent 是通过 GDB legacy、`dlopen + pthread`，还是 PEP 768 进入目标进程，只要最后执行的是 Python agent，它看到的就是目标进程当前的 Python runtime。

如果目标进程已经做了：

```python
from gevent import monkey
monkey.patch_all()
```

那么 agent 的 Python 代码就必须假设：

- `socket` 可能不是原始 socket；
- `threading.Event` 可能不是原始 event；
- `threading.Thread.start()` 内部可能走到 gevent wait；
- 阻塞等待可能不允许出现在当前执行点。

所以 attach 方法和 agent runtime 适配是两个层面：

| 层面 | 解决的问题 |
|---|---|
| attach 方法 | 怎么进入目标进程，什么时候执行代码 |
| agent 控制面 | 进入之后用什么 runtime 原语保持 socket、线程、ready 信号稳定 |
| agent 数据面 | 诊断命令在目标 runtime 里还能提供什么语义 |

控制面修复以后，agent 不再轻易被 monkey patch 拖死。但这还不是终点，诊断命令本身也要看 gevent 状态。

---

## 数据面：按命令声明兼容性

数据面和控制面不同。控制面可以尽量退到底层原语；数据面要观察业务函数、调用栈、采样热点，天然会碰到目标进程的调度模型。

peeka 0.1.13 之后的源码没有把“gevent 下所有命令都完全等价”作为目标，而是做了三件事：

1. 零副作用探测 gevent 状态：只看 `sys.modules` 里已有的 `gevent.monkey`、`gevent.hub`，不主动 import gevent；
2. 建一个命令兼容矩阵：同一个命令在 `none`、`imported`、`patched`、`active_hub` 下选择不同 backend；
3. 对会选择 backend 或发生降级的命令，把结果放进 JSONL `meta`：`gevent_state`、`backend`、`greenlet_blind`、`degraded_reason`。

源码中的兼容矩阵核心意思是：

| 命令 | gevent patched / active hub 下的策略 |
|---|---|
| `watch` | 保持 wrapper 观测，记录入参、返回值、异常和耗时 |
| `monitor` | 保持 wrapper 统计，记录调用数、成功率和响应时间 |
| `stack` | 保持函数入口处的 stack 捕获 |
| `trace` | 降级为 `wrapper_only`，只保留根调用耗时，不再展开递归调用树 |
| `top` | 使用 `greenlet_aware_sampling`，同时标记 `greenlet_blind` |

这里的重点不是“所有命令都安全无损”，而是每个命令都要明确自己在 gevent 下还能承诺什么。

### `trace` 为什么要降级

`trace` 的完整调用树依赖全局 tracing backend。Python 3.12+ 可以走 `sys.monitoring`，更老版本通常走 `sys.settrace`。这类机制会进入解释器的 frame 事件流。

在 gevent patched 或 active hub 状态下，源码选择避开递归 tracing，强制使用 `wrapper_only`：

```text
trace target()
  -> wrapper 记录 target() 总耗时
  -> 不再用 sys.settrace / sys.monitoring 展开子调用
```

因此 `trace` 仍然能告诉你“这个入口调用花了多久、是否抛异常”，但不能再声称自己拿到了完整子调用树。输出里的 `degraded_reason` 会说明这个降级原因。

### `top` 为什么还要标记 greenlet_blind

`top` 的普通采样方式是 frame walk：周期性读取各 OS thread 的当前 frame，再按函数聚合热点。

gevent 的问题在于：很多 greenlet 共享一个 OS thread。`sys._current_frames()` 只能看到每个 OS thread 当前正在运行的 frame，看不到所有挂起的 greenlet 栈。于是普通 frame walk 在 gevent 下会天然漏掉一部分上下文。

当前源码做了一个折中：如果 `greenlet` 已经加载并提供 `gettrace` / `settrace`，`top` 会安装一个链式 greenlet tracer，记录 switch / throw 事件，并在退出时恢复之前的 tracer。这样可以多保留一些 greenlet 调度信息，但仍然不能完整枚举所有挂起 greenlet 的栈，所以输出继续标记：

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

这比沉默地给出一份看似完整的热点列表更诚实。

### `patch-status` 的作用

还有一个辅助命令值得单独看：`patch-status`。它不修复任何东西，只报告目标 runtime 当前是什么状态：

- gevent / eventlet 是否已经 import 或 active；
- 当前 `socket.socket`、`threading.Event` 等对象是否仍等于 RPL 捕获的 native 对象；
- RPL 自检是否通过。

这个命令的价值在于把“agent 看到的世界”显式暴露出来。遇到 gevent 目标进程时，先看 `patch-status`，再读 `trace` / `top` 输出里的 `meta`，比只盯着 timeout 更快。

---

## 总结

表面错误可能只是：

```text
Agent initialization timeout (ready file)
```

但在 gevent 目标进程里，真正的根因可能是：

```text
attach agent 使用 threading.Thread / threading.Event
  -> 这些对象被 gevent monkey patch 影响
  -> Thread.start() 或 Event.wait() 落入 gevent wait
  -> 当前执行点不允许 blocking switch
  -> agent 在写 ready 文件前退出
```

控制面修复方向不是把某个 timeout 调长，也不是单纯更换 attach 路径，而是把 agent 控制面和目标应用 runtime 解耦：

1. 不主动依赖被 monkey patch 的 `socket` 和 `threading`；
2. 如果 gevent 已经存在，使用 `gevent.monkey.get_original()` 取原始对象；
3. 用 `_thread.start_new_thread` 替代 `threading.Thread` 启动控制线程；
4. 去掉 `threading.Event.wait()` 这种 ready barrier；
5. 用 `listen()` 后的 socket backlog 加外部 hello 探测完成同步；
6. 把 ready timeout 当作表象，优先查看目标进程内部 agent log。

但控制面稳定不代表数据面无损。gevent 下还要按命令处理观测语义：

1. `watch`、`monitor`、`stack` 在兼容矩阵里保持 safe，但它们不声称展开 gevent 的内部调度；
2. `trace` 在 patched / active hub 下应降级到 `wrapper_only`，避免递归 tracing API；
3. `top` 可以接入 greenlet trace 事件，但仍要标记 `greenlet_blind`；
4. `trace` / `top` 这类受影响输出要带 `meta`，让调用方知道当前 backend 和降级原因。

attach 工具运行在别人的进程里。那个进程的标准库、线程模型、socket 行为、frame 调度模型都可能已经被业务框架改造过。控制面要尽量退到底层原语；数据面要诚实声明自己还能看见什么。
