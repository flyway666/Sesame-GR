# Sesame-GR 任务执行架构分析与重构方案

> 分析日期：2026-06-19

---

## 一、背景

Sesame-GR（芝麻粒）是一个 Xposed 模块，通过 Hook 支付宝进程来自动执行蚂蚁森林、蚂蚁农场、蚂蚁新村等小程序的能量收集和任务操作。模块中包含大量**并发运行的自动化任务**，在频繁执行时容易触发支付宝服务端的**限流（Rate Limiting）**，导致任务失败、IP/账号被临时封禁。

本文档从源码层面分析当前任务执行架构，梳理并发模型、线程池和同步机制，并给出多种重构方案以支持**单线程串行执行任务**，降低被限流风险。

---

## 二、任务架构总览

### 2.1 三层调度架构

```
┌─────────────────────────────────────────────────────────────┐
│                      Xposed Hook 层                           │
│              ApplicationHook.java (Service.onCreate)         │
│                  ↑ 触发初始化 / 重启                            │
├─────────────────────────────────────────────────────────────┤
│                    主调度循环 (mainTask)                        │
│               BaseTask + Handler.postDelayed                  │
│               ↓ 每隔 checkInterval 毫秒调用                     │
├─────────────────────────────────────────────────────────────┤
│                   ModelTask.startAllTask()                    │
│               ↓ 遍历所有 Model，逐个启动                        │
├─────────────────────────────────────────────────────────────┤
│               MAIN_THREAD_POOL (线程池)                        │
│     ↓ 每个 ModelTask 的 mainRunnable 提交到池中并发执行          │
├─────────────────────────────────────────────────────────────┤
│   AntForest  │  AntFarm  │  AntStall  │  ...  共 17+ 个任务   │
│   (run())    │  (run())  │  (run())   │                        │
└─────────────────────────────────────────────────────────────┘
```

### 2.2 五个触发机制

| 机制 | 描述 | 代码位置 |
|------|------|----------|
| Handler 自调度循环 | 主循环执行完后通过 `postDelayed` 安排下一次执行 | `ApplicationHook.java` → `execDelayedHandler()` |
| AlarmManager 唤醒 | `setExactAndAllowWhileIdle` 在午夜和配置时间触发 | `ApplicationHook.java` → `setWakenAtTimeAlarm()` |
| 精确时间调度 | 检查 `execAtTimeList`，精准对齐某个时刻 | `ApplicationHook.java` → mainTask runnable 后半段 |
| Broadcast 广播 | `sesame.restart/execute/reLogin` 等自定义广播 | `AlipayBroadcastReceiver.onReceive()` |
| Activity onResume | 检测登录状态变化后重启 | LauncherActivity hook |

### 2.3 关键源码文件

| 文件 | 路径 | 职责 |
|------|------|------|
| `ApplicationHook.java` | `hook/ApplicationHook.java` | Xposed 入口、主循环、AlarmManager、广播接收 |
| `ModelTask.java` | `data/task/ModelTask.java` | 任务抽象基类、线程池管理、`startAllTask/stopAllTask` |
| `BaseTask.java` | `data/task/BaseTask.java` | 基础线程封装，主调度循环使用 |
| `Model.java` | `data/Model.java` | 模型注册表、`initAllModel/bootAllModel` |
| `ModelOrder.java` | `model/base/ModelOrder.java` | 所有模型类的注册顺序列表 |
| `ChildTaskExecutor.java` | `data/task/ChildTaskExecutor.java` | 子任务调度器接口 |
| `ProgramChildTaskExecutor.java` | `data/task/ProgramChildTaskExecutor.java` | 纯线程池的子任务实现 |
| `SystemChildTaskExecutor.java` | `data/task/SystemChildTaskExecutor.java` | Handler + 线程池的子任务实现 |

---

## 三、核心并发模型分析

### 3.1 MAIN_THREAD_POOL — 全局主线程池

```java
// ModelTask.java 第 28 行
private static final ThreadPoolExecutor MAIN_THREAD_POOL = new ThreadPoolExecutor(
    getModelArray().length,   // corePoolSize = 模型总数（~20+），允许所有任务同时运行
    Integer.MAX_VALUE,        // maxPoolSize = 无上限
    30L, TimeUnit.SECONDS,    // 空闲线程 30 秒回收
    new SynchronousQueue<>(), // SynchronousQueue：不排队，有线程立即运行
    new ThreadPoolExecutor.CallerRunsPolicy() // 饱和时由提交线程运行
);
```

**关键特性：**
- `corePoolSize = getModelArray().length`：核心线程数等于所有模型数量，默认即可容纳所有任务同时运行
- `SynchronousQueue`：零容量队列，**不会缓冲等待**，有空闲线程就立即执行
- `CallerRunsPolicy`：如果线程池饱和（几乎不会发生），提交者自己运行任务
- 结论：**所有任务几乎同时并发执行，没有排队机制**

### 3.2 startAllTask() — 任务启动入口

```java
// ModelTask.java 第 190 行
public static void startAllTask(Boolean force) {
    // 每日自动备份配置
    FileUtil.backupConfigV2WithRolling(UserIdMap.getCurrentUid());
    // 执行自定义 RPC 请求
    taskRpcRequest();
    // 遍历所有模型，逐个启动
    for (Model model : getModelArray()) {
        if (model != null && ModelType.TASK == model.getType()) {
            if (((ModelTask) model).startTask(force)) {
                try {
                    Thread.sleep(750);  // 仅启动间隔 750ms，不是执行间隔
                } catch (InterruptedException e) { ... }
            }
        }
    }
}
```

虽然启动时每个 task 之间间隔了 750ms，但由于每个 task 都提交给线程池异步执行，**这 750ms 间隔只决定"提交时间"，不决定"执行时序"**——所有 task 实际上几乎是同时开始运行的。

### 3.3 startTask() — 单个任务启动

```java
// ModelTask.java 第 148 行
public synchronized Boolean startTask(Boolean force) {
    if (MAIN_TASK_MAP.containsKey(this)) {
        if (!force) return false;
        stopTask();
    }
    if (isEnable() && check()) {
        if (isSync()) {
            mainRunnable.run();           // 同步运行（默认 false）
        } else {
            MAIN_THREAD_POOL.execute(mainRunnable);  // 异步提交到线程池
        }
        return true;
    }
    return false;
}
```

- `isSync()` 默认返回 `false`（第 78 行），因此所有任务默认走线程池异步路径
- `MAIN_TASK_MAP`（`ConcurrentHashMap`）跟踪正在运行的任务，防止重复执行
- `mainRunnable` 在 `finally` 块中从 `MAIN_TASK_MAP` 移除自身

### 3.4 子任务（ChildTask）执行器

每个 `ModelTask` 可以创建子任务，通过 `ChildTaskExecutor` 调度：

- **`ProgramChildTaskExecutor`**：每个 group 有独立线程池（`0, Integer.MAX_VALUE, SynchronousQueue`），非延迟任务立即执行
- **`SystemChildTaskExecutor`**：对延迟任务（>3s）使用 `Handler.postDelayed` 提前 2.5s 唤醒，剩余时间由 `Thread.sleep` 完成；非延迟任务同样提交到线程池

子任务的线程池同样是无容量限制的 `SynchronousQueue`，可以并发运行大量子任务。

### 3.5 同步机制总结

| 机制 | 使用场景 |
|------|----------|
| `ConcurrentHashMap` | `MAIN_TASK_MAP`、`childTaskMap`、各执行器的线程池映射 |
| `synchronized` 方法 | `BaseTask.startTask/stopTask`、`ModelTask.startTask/stopTask` |
| `synchronized` 块 | 低版本 Android（< N）的 `childTaskMap` 操作 |
| `volatile` 变量 | `Thread thread`、`lastExecTime`、`broadcastReceiverRegistered`、`offline` |
| `CallerRunsPolicy` | 线程池饱和时的背压机制 |
| `interrupt() + join()` | 任务停止时的线程清理 |

---

## 四、限流问题的根因

```
时间轴 →
Task1 提交 ──▶ Task1 运行 ──────────────────────────────────▶ Task1 结束
Task2 提交 ──▶ Task2 运行 ──────────────────▶ Task2 结束
Task3 提交 ──▶ Task3 运行 ─────▶ Task3 结束
              ↑ 几乎同一时段发出大量 RPC 请求，触发支付宝限流
```

每个 `ModelTask.run()` 内部会通过 RPC 桥调用支付宝接口（`requestString/requestObject`）。当 17+ 个任务同时运行，短时间内密集发出数百个 RPC 请求，支付宝服务端会判定为异常行为，触发限流甚至封禁。

---

## 五、重构方案

### 方案一：改线程池为单线程（最小改动）

修改 `MAIN_THREAD_POOL` 为单线程 + 有界队列，所有任务自然排队依次执行：

```java
private static final ThreadPoolExecutor MAIN_THREAD_POOL = new ThreadPoolExecutor(
    1,                          // corePoolSize = 1
    1,                          // maximumPoolSize = 1
    30L, TimeUnit.SECONDS,
    new LinkedBlockingQueue<>(),// 任务排队等待
    new ThreadPoolExecutor.CallerRunsPolicy()
);
```

**优点：**
- 只改一行关键参数，无侵入性
- 所有 `ModelTask` 自动排队，无需修改每个子类
- 线程池之外的代码完全不受影响

**缺点：**
- 所有任务（包括小任务）都在同一个队列里串行等待
- **一个任务卡住（如 RPC 超时等待）会堵住所有后续任务**
- 不能区分优先级

**风险等级：中** — 如果有任何 task 的 `run()` 中包含了长时间阻塞且无超时的操作，会导致整个模块"假死"。

---

### 方案二：在 startAllTask() 中同步执行

跳过线程池，在 `startAllTask()` 中直接调用 `mainRunnable.run()`：

```java
// 方式 A：修改 startAllTask()
public static void startAllTask(Boolean force) {
    // 每日备份配置
    if (!Status.hasFlagToday("Config::backup")) {
        FileUtil.backupConfigV2WithRolling(UserIdMap.getCurrentUid());
        Status.flagToday("Config::backup");
    }
    taskRpcRequest();

    for (Model model : getModelArray()) {
        if (model != null && ModelType.TASK == model.getType()) {
            ModelTask task = (ModelTask) model;
            if (task.isEnable() && task.check()) {
                task.mainRunnable.run();  // 直接同步运行，不经过线程池
                try {
                    Thread.sleep(750);    // 真正的执行间隔
                } catch (InterruptedException e) {
                    Log.printStackTrace(e);
                }
            }
        }
    }
}
```

**优点：**
- 改动集中，不改变线程池结构
- 750ms 间隔真正起到"任务之间停顿"的作用
- 保留 `MAIN_THREAD_POOL` 供子任务等其他用途使用
- 一个任务卡住不会影响其他任务（但会延迟后续任务的启动）

**缺点：**
- 如果某个任务 `run()` 耗时很长（如持续 1-2 分钟），整个循环的延迟会累积
- 直接调用 `mainRunnable.run()` 绕过了 `stopTask()` 的清理逻辑（如果 force=true 场景）

**风险等级：低** — 逻辑清晰，改动可控。

---

### 方案三：startTask() 增加同步参数

给 `startTask()` 增加一个参数来控制是否走线程池：

```java
public synchronized Boolean startTask(Boolean force, Boolean sync) {
    if (MAIN_TASK_MAP.containsKey(this)) {
        if (!force) return false;
        stopTask();
    }
    if (isEnable() && check()) {
        if (sync || isSync()) {
            mainRunnable.run();          // 同步
        } else {
            MAIN_THREAD_POOL.execute(mainRunnable);  // 异步
        }
        return true;
    }
    return false;
}

// startAllTask 中传 sync=true
public static void startAllTask(Boolean force) {
    // ...
    for (Model model : getModelArray()) {
        if (model != null && ModelType.TASK == model.getType()) {
            if (((ModelTask) model).startTask(force, true)) {  // 强制同步
                Thread.sleep(750);
            }
        }
    }
}
```

**优点：**
- 保留 `isSync()` 的设计意图，子类仍可复写控制默认行为
- `startAllTask` 强制同步，其他入口（如果有）可保持异步
- 向后兼容性好

**缺点：**
- 需要修改 `startTask()` 签名及其所有调用点

**风险等级：低**

---

### 方案四：可配置的任务级同步/异步（最灵活）

在 `BaseModel` 中增加配置字段，让用户为每个任务分别选择执行模式：

```java
// BaseModel 新增字段
public static final ModelField taskExecMode = new ModelField(...);
// 值：SEQUENTIAL（顺序执行） / CONCURRENT（并发）

// ModelTask.startTask() 根据配置决定
public synchronized Boolean startTask(Boolean force) {
    if (task.getExecMode() == ExecMode.SEQUENTIAL) {
        mainRunnable.run();  // 同步
    } else {
        MAIN_THREAD_POOL.execute(mainRunnable);  // 异步
    }
}
```

**优点：**
- 用户可按需配置：快速任务并发，重量级任务串行
- 最灵活，适应不同网络环境和账号状态

**缺点：**
- 改动量大：需要加配置定义、配置 UI、序列化等
- 过度设计：大多数用户并不需要细粒度控制

**风险等级：低（但工作量大）**

---

### 方案五：CountDownLatch 协调 + 超时兜底（推荐增强方案）

保留线程池并发能力，但在调度层增加协调机制：

```java
public static void startAllTask(Boolean force) {
    // ...
    for (Model model : getModelArray()) {
        if (model != null && ModelType.TASK == model.getType()) {
            ModelTask task = (ModelTask) model;
            if (task.isEnable() && task.check()) {
                final CountDownLatch latch = new CountDownLatch(1);
                // 包装 Runnable，完成后 countDown
                task.executeWithCallback(() -> {
                    try { task.run(); } 
                    finally { latch.countDown(); }
                });
                // 等待完成或超时（每个任务最多 5 分钟）
                boolean completed = latch.await(5, TimeUnit.MINUTES);
                if (!completed) {
                    Log.warn(task.getName() + " 执行超时，继续下一个任务");
                }
                Thread.sleep(750); // 任务间隔
            }
        }
    }
}
```

**优点：**
- 可设置**每任务超时时间**，避免单个任务卡死整个队列
- 保留线程池结构，不影响子任务执行器
- 750ms 间隔真正起效

**缺点：**
- 引入 `CountDownLatch`，代码复杂度略有增加
- 需要确定合理的超时时间

**风险等级：低**

---

## 六、各方案对比

| 维度 | 方案一（单线程池） | 方案二（同步调用） | 方案三（参数控制） | 方案四（可配置） | 方案五（Latch协调） |
|------|:---:|:---:|:---:|:---:|:---:|
| 改动量 | 极小（2行） | 小（~15行） | 中（~30行） | 大（多文件） | 中（~40行） |
| 代码侵入性 | 低 | 低 | 中 | 高 | 中 |
| 单任务卡住影响 | 阻塞所有 | 仅阻塞后续 | 仅阻塞后续 | 取决于配置 | 超时后跳过 |
| 子任务并发控制 | 无影响 | 无影响 | 无影响 | 需额外处理 | 无影响 |
| 灵活度 | 低 | 低 | 中 | 高 | 中 |
| 降限流效果 | 高 | 高 | 高 | 高 | 高 |
| 推荐度 | ★★★ | ★★★★★ | ★★★★ | ★★ | ★★★★ |

---

## 七、推荐实施路径

### 短期（立即见效）

采用**方案二 + 方案三结合**：

1. 给 `startTask()` 增加一个 `sync` 参数，默认 `false` 保持向后兼容
2. `startAllTask()` 传 `sync=true`，让所有主任务顺序执行
3. 保留 `MAIN_THREAD_POOL` 不变，供子任务和其他异步场景使用

```java
// 修改点在 ModelTask.java 两个方法，改动不超过 20 行
```

### 中期（增加稳定性）

在方案二基础上增加**超时保护**：

- 在 `startAllTask()` 的同步执行循环中，用 `FutureTask` + `get(timeout)` 包裹 `mainRunnable.run()`，设置合理的超时时间（如 5 分钟）
- 超时后标记该任务失败，继续执行下一个任务，避免单个任务卡死整轮执行

### 长期（可观测性）

- 记录每个任务的执行耗时、RPC 调用次数和成功率
- 实现动态间隔调整：如果检测到限流（RPC 返回特定错误码），自动增加任务间隔
- 在 Web UI 中展示任务执行状态和时间线

---

## 八、涉及代码位置

| 文件 | 关键行 | 说明 |
|------|--------|------|
| `data/task/ModelTask.java` | L28 | `MAIN_THREAD_POOL` 定义，核心线程池 |
| `data/task/ModelTask.java` | L78-80 | `isSync()` 默认返回 false |
| `data/task/ModelTask.java` | L148-168 | `startTask(Boolean force)` 任务启动方法 |
| `data/task/ModelTask.java` | L190-211 | `startAllTask(Boolean force)` 任务遍历启动 |
| `data/task/ModelTask.java` | L213-225 | `stopAllTask()` 停止所有任务 |
| `data/task/ProgramChildTaskExecutor.java` | - | 子任务执行器，内部线程池同样是 SynchronousQueue |
| `data/task/SystemChildTaskExecutor.java` | - | 系统子任务执行器 |
| `data/task/BaseTask.java` | - | 主调度循环使用的基类 |
| `hook/ApplicationHook.java` | L298-394 | 主调度循环 mainTask runnable |
