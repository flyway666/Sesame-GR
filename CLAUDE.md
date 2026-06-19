# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

**Sesame-GR** (芝麻粒) — An Xposed module for Android that automates Alipay Ant Forest (蚂蚁森林) energy collection and related mini-program tasks. It hooks into the Alipay app process via Xposed, intercepts RPC calls, and automates tasks across Ant Forest, Ant Farm, Ant Orchard, Ant Sports, Ant Member, and other Alipay mini-programs.

- Language: Java (no Kotlin)
- Build: Android Gradle Plugin 8.7.0, Gradle
- Min SDK 21, Target SDK 34
- License: GPLv3
- Originated from [LazyImmortal/Sesame](https://github.com/LazyImmortal/Sesame), [TKaxv-7S/Sesame-TK](https://github.com/TKaxv-7S/Sesame-TK), [constanline/XQuickEnergy](https://github.com/constanline/XQuickEnergy)

## Build Commands

```bash
# Build all variants (normal + compatible)
./gradlew assembleRelease

# Build normal flavor only
./gradlew assembleNormalRelease

# Build compatible flavor (older Jackson for compatibility)
./gradlew assembleCompatibleRelease

# Clean build
./gradlew clean assembleRelease
```

The APK outputs are at `app/build/outputs/apk/` with names like `Sesame-Normal-{version}.apk`.

## Architecture

### Entry Point

`assets/xposed_init` → `io.github.lazyimmortal.sesame.hook.ApplicationHook` implements `IXposedHookLoadPackage`. It hooks into:
- **`android.app.Service.onCreate`** — main lifecycle; initializes modules, sets up RPC bridge, starts execution loop
- **`com.alipay.mobile.quinox.LauncherActivity.onResume`** — detects login state changes
- Various Alipay internal classes for background detection avoidance, RPC interception

### Module System

All modules are subclasses of `Model` or `ModelTask` (which extends `Model`):

- **`Model`** — base with `getName()`, `getGroup()`, `getFields()` abstract methods. Fields are auto-generated configuration UI.
- **`ModelTask`** — adds `check()`, `run()`, child task scheduling via `ChildTaskExecutor` (system AlarmManager or program timer).
- **Module registration** — `model/base/ModelOrder.java` lists all module classes in a static block.
- **Module groups** — `ModelGroup` enum: BASE, FOREST, FARM, STALL, ORCHARD, SPORTS, MEMBER, OTHER.

Modules are initialized via `Model.initAllModel()` → `Model.bootAllModel()` in a lifecycle tied to the Alipay service.

### Package Layout

```
hook/              — Xposed hook implementations (RPC interception, UI analysis, captcha)
rpc/
  bridge/          — RpcBridge interface with NewRpcBridge / OldRpcBridge implementations
  intervallimit/   — Rate limiting for RPC calls
data/
  Model.java       — Abstract module base class
  ModelConfig.java — Configuration wrapper per module
  ModelField.java  — Configuration field system
  modelFieldExt/   — Concrete field types (Boolean, Integer, Choice, List, Select, Text, etc.)
  task/            — Task execution: ModelTask, BaseTask, ChildTaskExecutor
  ConfigV2.java    — JSON config persistence (config_v2.json per user)
model/
  normal/base/     — BaseModel: core settings (interval, timing, RPC mode, debug)
  normal/answerAI/ — AI answer modules (GeminiAI, TongyiAI)
  task/            — Feature modules, each with a Model class + RpcCall class:
    antForest/     — Main feature: energy collection, ForestChouChouLe, WhackMole, Privilege
    antFarm/       — Ant Farm automation
    antStall/      — Ant Stall (新村)
    antOrchard/    — Ant Orchard (农场水果)
    antOcean/      — Ant Ocean
    antSports/     — Ant Sports (运动)
    antMember/     — Ant Member (会员), AntInsurance, MerchantService
    antDodo/       — Ant Dodo
    protectEcology/— Protect Ecology (保护地/海洋/树)
    ancientTree/   — Ancient Tree (古树)
    antBookRead/   — Book Reading
    antGame/       — Game Center tasks
    greenFinance/  — Green Finance
    readingDada/   — Reading Dada (嗒嗒读书)
    consumeGold/   — Consume Gold (消费金)
    omegakoiTown/  — Omegakoi Town (锦鲤小镇)
entity/            — POJO data models (RpcEntity, AlipayUser, CollectEnergyEntity, etc.)
ui/                — Android activities (MainActivity, SettingsActivity, etc.)
util/              — Utilities (Log, FileUtil, TimeUtil, JsonUtil, Status, Statistics)
util/idMap/        — ID mapping caches for Alipay entities
assets/web/        — Web-based settings UI (Vue3 + Vant framework)
```

### RPC Mechanism

- `ApplicationHook` provides static `requestString()`/`requestObject()` methods used by all modules.
- `RpcBridge` interface has two implementations:
  - **NewRpcBridge** — for Alipay ≥ v10.3.96.8100, hooks into `RpcBridgeExtension.rpc()`
  - **OldRpcBridge** — for older Alipay versions, hooks into `RpcExceptionUtil.rpc()`
- `RpcIntervalLimit` manages per-method rate limiting.
- All RPC calls carry retry logic (default 3 tries).

### Configuration

- Per-user JSON config: `{files_dir}/cfg/{userId}/config_v2.json`
- Configuration UI is auto-generated from `Model.getFields()` — each field definition creates a UI widget.
- Rolling backup system with configurable retention days (`backupConfigDays`).
- Default config auto-restore from backup if corruption detected.

### Key Dependencies

- Xposed Framework API 82 (compileOnly)
- Lombok 1.18.32 (code generation)
- Jackson 2.18.2 (normal) / 2.13.5 (compatible) — JSON serialization
- OkHttp 4.12.0 — HTTP client
- NanoHTTPD 2.3.1 — embedded HTTP server (port 8080)
- xLog 1.11.0 — logging
- Native library: `libsesame.so` (arm64-v8a, armeabi-v7a, x86, x86_64)

### Product Flavors

- **normal** — Jackson 2.18.2 (latest features, higher minimum Alipay version)
- **compatible** — Jackson 2.13.5 (broader Alipay version compatibility)
