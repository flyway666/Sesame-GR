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

## 2026-06-19

### 海洋模块新增可配置操作间隔

- **文件**:
  - `app/src/main/java/io/github/lazyimmortal/sesame/model/task/antOcean/AntOcean.java`
  - `[jim]-config_v2.json`
- **改动**:
  - `AntOcean` 新增 `operateInterval` 配置字段（`IntegerModelField`，范围 0~10000ms，默认 1200ms，Web 界面显示名称为"操作间隔(毫秒)"）
  - 新增静态辅助方法 `sleepOperateInterval()`
  - 以下方法的循环迭代间增加可配置间隔（共 16 处）：
    - 收取能量球：`collectEnergy()`
    - 清理海域：`cleanOcean()`
    - 检查奖励：`checkReward()`
    - 收集潘多拉能量：`collectReplicaAsset()`
    - 潘多拉任务奖励：`queryReplicaTaskList()`
    - 海域/鱼群处理：`querySeaAreaDetailList()`
    - 好友海域清理（3 段）：`queryUserRanking()`、`cleanFriendOcean()`
    - 日常任务领取：`queryTaskList()`、`receiveTaskAward()`
    - 万能拼图（3 处）：`exchangeUniversalPiece()`、`useUniversalPiece()`
  - 将原有的硬编码 `TimeUtil.sleep(500)` 和 `TimeUtil.sleep(1000)` 替换为可配置的 `sleepOperateInterval()`
  - `[jim]-config_v2.json`：AntOcean 段添加 `operateInterval` 配置项；修复 BaseModel 段 `toastOffsetY` 后缺失的逗号
- **原因**: 海洋模块内部循环密集发出 RPC 请求时缺乏统一可配置间隔，容易触发支付宝限流。在海洋模块自身设置页面增加"操作间隔"字段（非基础设置），默认 1.2 秒，允许用户按需关闭（设为 0）或调大间隔。

## 2026-06-20

### 支付宝运动/生态保护/蚂蚁会员 新增可配置操作间隔

- **文件**:
  - `app/src/main/java/io/github/lazyimmortal/sesame/model/task/antSports/AntSports.java`
  - `app/src/main/java/io/github/lazyimmortal/sesame/model/task/protectEcology/ProtectEcology.java`
  - `app/src/main/java/io/github/lazyimmortal/sesame/model/task/antMember/AntMember.java`
- **改动**:
  - 三个模块各自新增 `operateInterval` 配置字段（`IntegerModelField`，范围 0~10000ms，默认 1200ms，Web 界面显示名称为"操作间隔(毫秒)"）
  - 各自新增静态辅助方法 `sleepOperateInterval()`
  - **支付宝运动（AntSports）**：以下方法的循环迭代间增加可配置间隔（共 22 处）：
    - `userTaskGroupQuery()`、`userTaskRightsReceive()` — 文体任务/奖励循环
    - `sportsTasks()` — 运动任务列表循环
    - `receiveCoinAsset()` — 收取运动币气泡循环
    - `queryClubHome()` — 抢好友多个循环（买卖/训练/购买）
    - `walk()` — 行走路线 do-while 循环
    - `completeTask()` — 任务完成延迟
    - `queryMemberPriceRanking()` — 抢购好友 RPC 间隔
    - `openTreasureBox()` — 宝箱领取循环
    - `walkGrid()`、`build()` — 能量泵 while 循环
    - `queryAndProcessBubbleTasks()` — 气泡任务循环
    - `processTaskCenter()` — 任务中心循环
    - `processBrowseTasks()` — 浏览任务循环
  - **生态保护（ProtectEcology）**：以下 7 处 `TimeUtil.sleep(300)` 全部替换为可配置间隔：
    - `queryCooperatePlant()` — 合种浇水
    - `protectTree()` — 植树循环
    - `protectReserve()` — 保护地循环
    - `protectAnimal()` — 护林员循环
    - `protectReserveMinNum()` — 最少保护循环
    - `protectBeachMinNum()` — 海滩最少保护循环
    - `protectBeach()` — 海滩保护循环
  - **蚂蚁会员（AntMember）**：以下方法的循环迭代间增加可配置间隔（共 18 处）：
    - `initMemberTaskListMap()` — 任务列表分页 do-while
    - `signPageTaskList()` — 签到任务页 do-while
    - `queryPointCert()` — 积分领取 for 循环
    - `doBrowseTask()` — 浏览任务 for 循环（2 处）
    - `collectSesame()` — 芝麻粒领取多层 for 循环（4 处）
    - `RecommendTask()` — 推荐任务 for 循环（2 处）
    - `OrdinaryTask()` — 普通任务 for 循环
    - `queryAllStatusTaskList()` — 递归调用前
    - `handleGrowthGuideTasks()` — 成长引导 for 循环
    - `memberPointExchangeBenefit()` — 权益兑换 for 循环
    - `queryAndProcessTaskList()` — 任务模块嵌套 for 循环
- **原因**: 上述三个模块内部循环密集发出 RPC 请求时缺乏统一可配置间隔，容易触发支付宝限流。在每个模块自身设置页面增加"操作间隔"字段，默认 1.2 秒，允许用户按需关闭（设为 0）或调大间隔。

