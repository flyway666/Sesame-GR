# 修改记录

## 2026-06-19

### 任务调度统一重构（方案三 + 方案五融合）

- **文件**: `app/src/main/java/io/github/lazyimmortal/sesame/data/task/ModelTask.java`
- **改动**:
  - 将 `startTask()` 重构为 4 个重载的统一方法族，覆盖全部执行模式：
    - `startTask()` / `startTask(Boolean force)` — 原行为，向后兼容
    - `startTask(Boolean force, Boolean sync)` — **方案三**：调用方控制同步/异步
    - `startTask(Boolean force, Boolean sync, long timeoutMs)` — **融合核心**：sync=true 同步执行；sync=false 且 timeoutMs>0 时提交线程池 + `Future.get(timeout)` 超时保护；sync=false 且 timeoutMs≤0 时原异步行为
  - 移除之前独立的 `startTaskAndWait()` 方法，统一由 `startTask(force, false, 300_000L)` 替代
  - `startAllTask()` 改用 `startTask(force, false, 300_000L)` 串行执行，每任务超时 5 分钟
- **原因**: 方案三提供灵活的 sync 参数控制；方案五提供超时保护防止任务卡死。融合后 API 统一、无重复代码，同时保留完整向后兼容性。
