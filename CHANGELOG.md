# 修改记录

## 2026-06-19

### 新增可配置的子任务时间间隔（方案二扩展）

- **文件**:
  - `app/src/main/java/io/github/lazyimmortal/sesame/model/normal/base/BaseModel.java`
  - `app/src/main/java/io/github/lazyimmortal/sesame/model/task/antOcean/AntOcean.java`
  - `app/src/main/java/io/github/lazyimmortal/sesame/model/task/antFarm/AntFarm.java`
  - `app/src/main/java/io/github/lazyimmortal/sesame/model/task/antStall/AntStall.java`
  - `app/src/main/java/io/github/lazyimmortal/sesame/model/task/antOrchard/AntOrchard.java`
  - `app/src/main/java/io/github/lazyimmortal/sesame/model/task/antSports/AntSports.java`
  - `app/src/main/java/io/github/lazyimmortal/sesame/model/task/protectEcology/ProtectEcology.java`
- **改动**:
  - `BaseModel` 新增 `taskInterval` 配置字段（`IntegerModelField`，范围 0~10000ms，默认 1000ms）及静态辅助方法 `sleepTaskInterval()`；Web 界面显示名称为"子任务时间间隔"
  - **神奇海洋**：`cleanOcean()`、`collectEnergy()`、`collectReplicaAsset()`、`queryReplicaTaskList()` 各循环迭代间增加可配置间隔
  - **蚂蚁庄园**：`rewardFriend()`、`sendBackAnimal()`、`feedFriend()` 各好友循环迭代间增加可配置间隔
  - **蚂蚁新村**：`inviteRegister()` 各好友循环迭代间增加可配置间隔
  - **芭芭农场**：`triggerTbTask()` 各任务奖励领取间增加可配置间隔
  - **支付宝运动**：`userTaskGroupQuery()`、`userTaskRightsReceive()` 各任务迭代间增加可配置间隔
  - **生态保护**：`cooperateWater()`、`protectBeach()` 各保护项目迭代间增加可配置间隔
- **原因**: 上述模块子任务内部循环密集发出 RPC 请求时缺乏间隔，容易触发支付宝限流。新增统一可配置的 `taskInterval` 字段（在基础设置中调整），默认为 1 秒，允许用户按需关闭（设为 0）或调大间隔以规避限流。

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
