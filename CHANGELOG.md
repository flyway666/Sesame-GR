# 修改记录

## 2026-06-19

### 任务串行执行改造（防限流）

- **文件**: `app/src/main/java/io/github/lazyimmortal/sesame/data/task/ModelTask.java`
- **改动**:
  - 新增导入：`Future`、`TimeoutException`、`ExecutionException`
  - 新增 `startTaskAndWait(Boolean force, long timeoutMs)` 方法：提交任务到线程池后用 `Future.get(timeout, unit)` 等待完成，超时则 `cancel(true)` 中断并 `stopTask()` 清理
  - 新增 `startTaskAndWait(Boolean force)` 重载，默认超时 5 分钟
  - 修改 `startAllTask()`：将 `startTask(force)` 替换为 `startTaskAndWait(force)`，实现任务串行执行
- **原因**: 原架构下所有任务通过 `MAIN_THREAD_POOL` 并发执行（`corePoolSize = 模型总数`），短时间内密集发出大量 RPC 请求，易触发支付宝限流。改为串行执行 + 超时保护后，每个任务执行完或超时才启动下一个，降低限流风险。
