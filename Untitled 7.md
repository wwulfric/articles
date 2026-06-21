# 当 Gevent 遇上 Python Profiler：一场静默的 CPU 风暴

## TL;DR

在 Gevent 协程环境中启用 `sys.setprofile`、`sys.settrace` 或 `sys.monitoring`（Python 3.12+）会导致 CPU 使用率随时间线性飙升，最终可能触发进程 OOM 或被容器平台杀死。根本原因是：**Gevent 的 greenlet 切换会高频触发 profiler callback，而 profiler callback 中的阻塞/同步操作又会反向加剧 greenlet 调度，形成正反馈循环**。

---

## 一、背景：两个看似无关的机制

### 1.1 Gevent 的协程调度

Gevent 通过 monkey-patch 标准库，将阻塞 I/O 转化为基于 libev/libuv 的事件循环调度。其核心是 **greenlet**——用户态协程，切换成本远低于内核线程，但仍涉及完整的 Python 调用栈保存/恢复。

```python
import gevent
from gevent import monkey
monkey.patch_all()

def task(n):
    gevent.sleep(0.01)  # 触发 greenlet 切换
    return n * 2

greenlets = [gevent.spawn(task, i) for i in range(100)]
gevent.joinall(greenlets)
```

在上述代码中，`gevent.sleep(0.01)` 会主动让出执行权，事件循环调度另一个 greenlet 运行。对于 100 个 greenlet，每 10ms 就会发生一次切换。

### 1.2 Python 的执行时钩子

Python 提供了三种在运行时拦截函数调用/返回的机制：

| API              | 引入版本 | 触发频率                       | 典型用途                  |
| ---------------- | -------- | ------------------------------ | ------------------------- |
| `sys.settrace`   | 2.0+     | 每条字节码、函数调用/返回      | 调试器（pdb）、覆盖率工具 |
| `sys.setprofile` | 2.0+     | 函数调用/返回、C 函数调用      | 性能分析器（cProfile）    |
| `sys.monitoring` | 3.12+    | 可选事件（call/return/line等） | 低开销调试/分析           |

三者的共同点：**注册一个全局回调函数，在特定执行事件发生时由解释器调用**。

```python
import sys

def profile_callback(frame, event, arg):
    print(f"[{event}] {frame.f_code.co_name}")
    return profile_callback

sys.setprofile(profile_callback)
```

---

## 二、冲突场景：什么时候会出问题

### 2.1 触发条件（需同时满足）

1. **进程运行在 Gevent 环境中**（`monkey.patch_all()` 或部分 patch）
2. **启用了 `sys.setprofile`、`sys.settrace` 或 `sys.monitoring`**
3. **Profiler callback 不是纯 CPU 计算**（包含锁、I/O、字典操作等）
4. **Greenlet 切换频率足够高**（典型阈值：>100 次/秒）

### 2.2 典型场景

- **协程密集型服务**：大量并发 greenlet 等待 I/O，事件循环频繁调度
- **诊断工具动态注入**：在运行中的 Gevent 进程上启动 trace/profiler
- **FaaS / 短生命周期容器**：高并发请求触发大量 greenlet，同时开启性能诊断
- **测试环境**：压力测试时同时启用覆盖率或性能分析

---

## 三、症状：如何识别这个问题

| 症状                     | 说明                                           |
| ------------------------ | ---------------------------------------------- |
| CPU 随时间线性增长       | 不是瞬间 100%，而是运行越久越高                |
| 即使业务空闲也飙高       | 没有请求时，greenlet 调度本身仍在触发 callback |
| 内存缓慢泄漏             | Profiler callback 创建的对象可能累积           |
| 进程被平台杀死           | CPU/内存超限后，被 K8s/Lambda/容器平台 OOMKill |
| 关闭 profiler 后恢复正常 | 这是最关键的确认信号                           |

---

## 四、原因分析：为什么它们会冲突

### 4.1 Greenlet 切换 = 大量函数调用事件

Gevent 的事件循环和 greenlet 切换完全由 Python 代码实现。每次切换涉及：

1. 当前 greenlet 的 `switch()` 被调用
2. 底层 C 代码保存当前栈帧
3. 恢复目标 greenlet 的栈帧
4. 目标 greenlet 从上次挂起点继续执行

这个过程在 C 层面和 Python 层面都会产生大量函数调用。`sys.setprofile` 会捕获其中所有 **C 调用** 和 **Python 调用/返回**。

### 4.2 Profiler Callback 被高频触发

假设一个 Gevent 服务有 **1000 并发 greenlet**，平均每个 greenlet 每 10ms 让出一次。那么每秒产生：

- **100,000 次 greenlet 切换**
- **每次切换触发 3-5 个 profile 事件**（C call / C return / Python call / Python return）
- **总计：30-50 万次 profiler callback/秒**

### 4.3 Callback 中的"重"操作引发二次调度

这是冲突的核心。**如果 profiler callback 包含以下任意操作，就可能触发二次 greenlet 调度**：

- **线程锁**（`threading.Lock`）：在 Gevent 中，标准 `threading.Lock` 仍然会阻塞 OS 线程。如果锁被其他 greenlet 持有（注意：是 greenlet，不是线程），当前 greenlet 无法继续，但 OS 线程仍可能执行——这会导致调度混乱
- **字典/列表操作**：可能触发内存分配，进而触发 GC，GC 可能释放 GIL
- **Socket/File I/O**：已被 monkey-patch，会触发事件循环的 hub 切换
- **Greenlet 切换相关操作**：任何涉及 `gevent.getcurrent()`、`greenlet.switch` 的调用

### 4.4 正反馈循环

```
+---------------+      +---------------------+
| Gevent 调度器  +----->+ Greenlet A -> B 切换 |
| (Event Loop)  |      | (C/Python 函数调用)  |
+---------------+      +---------------------+
        ^                        |
        |                        v
        |             +---------------------+
        |             | sys.setprofile      |
        |             | 触发 callback       |
        |             +---------------------+
        |                        |
        |                        v
        |             +---------------------+
        |             | callback 做 I/O/锁  |
        |             | -> 释放 GIL/触发 hub |
        |             +---------------------+
        |                        |
        +------------------------+
              (更多 greenlet 切换)
```

**循环链条**：

1. Gevent 切换 greenlet A → B
2. `sys.setprofile` 捕获 `c_call` / `call` 事件
3. Callback 执行（如：构造 observation → dict 操作 → 可能触发 GC）
4. GC 或 I/O 释放 GIL / 触发 hub 检查
5. Hub 发现其他 greenlet 就绪，执行额外调度
6. 额外调度产生更多 profile 事件
7. **循环回到步骤 2**











==========================================================================







