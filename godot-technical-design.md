# 基于 Godot Engine 的制作技术文档（技术方案）

> 依据：`mindmap整理文档.md` 中的系统拆分（Core loop / Quest system / Needs / Inventory）进行工程化落地。
> 目标：给出可实施的 Godot 技术架构、模块边界、数据流、实现细节与迭代路线。

---

## 1. 技术目标与约束

### 1.1 技术目标
- 以 **Godot 4.x** 构建 2D 项目，优先保证快速迭代与玩法验证。
- 将思维导图中的四大系统（核心循环、任务、需求、背包）模块化，降低后续功能扩展成本。
- 支持“配置驱动”的任务/数值迭代，避免频繁改代码。

### 1.2 约束与假设
- 单机优先，联网能力（云存档/排行榜）作为可插拔能力后续接入。
- 先实现可玩 MVP，再逐步扩展表现层和内容量。
- 团队默认使用 GDScript（必要时可混合 C#）。

---

## 2. 版本与工具链

### 2.1 引擎与语言
- **引擎**：Godot 4.2+（建议锁定小版本，避免升级破坏）。
- **脚本**：GDScript 2.0（开发效率高，协作门槛低）。
- **可选**：C# 用于复杂工具链或重计算模块（非 MVP 必需）。

### 2.2 项目结构建议

```text
res://
  scenes/
    main/
      main.tscn
    ui/
      hud.tscn
      quest_panel.tscn
      inventory_panel.tscn
  systems/
    game_flow/
      game_flow_manager.gd
      day_cycle_manager.gd
    quest/
      quest_manager.gd
      quest_runtime.gd
    needs/
      needs_manager.gd
      need_model.gd
    inventory/
      inventory_manager.gd
      item_model.gd
    save/
      save_manager.gd
  data/
    quests/
      tutorial_quests.json
      daily_quests.json
    items/
      items.json
    balance/
      needs_balance.json
  resources/
    quest/
      quest_def.gd
    item/
      item_def.gd
  autoload/
    app.gd
    event_bus.gd
    config_service.gd
```

### 2.3 开发插件建议
- Godot 内置 Git 插件（或外部 Git 工具）用于版本管理。
- EditorExport 配置多平台导出模板（Windows/macOS/Web）。
- 可选：使用自定义 EditorPlugin 做任务配置可视化编辑（中后期）。

---

## 3. 总体架构设计

### 3.1 分层模型
- **表现层（Scene/UI）**：场景节点、动画、HUD、面板交互。
- **业务层（Systems）**：Core loop、Quest、Needs、Inventory 的规则实现。
- **数据层（Config + Save）**：静态配置（JSON/Resource）+ 动态存档（Save）。
- **事件层（Event Bus）**：系统间解耦通信。

### 3.2 关键设计原则
1. **系统独立**：每个系统有独立 Manager，避免循环依赖。
2. **事件驱动**：通过信号或 EventBus 传递变化（任务完成、需求变更等）。
3. **配置驱动**：任务目标、奖励、需求衰减由配置文件控制。
4. **可序列化**：运行态对象必须可存档/恢复。

### 3.3 建议的 Autoload 单例
- `App`：启动流程、全局状态引用。
- `EventBus`：统一信号注册与广播。
- `ConfigService`：加载并缓存配置数据。
- `SaveManager`：读写存档。

---

## 4. 四大系统实现方案

### 4.1 Core Loop（按天循环）

### 职责
- 管理 `Start a day -> Do quest -> Reach star needs threshold -> Next day` 的主流程。

### 核心状态机
建议使用枚举状态：
- `DAY_START`
- `IN_PROGRESS`
- `DAY_REVIEW`
- `DAY_END`

### 关键流程
1. Day 开始：重置当日任务、刷新需求衰减参数。
2. 玩家执行任务：Quest 系统推进。
3. Needs 系统实时变更：体力/饥饿/学习/Star need。
4. 达成阈值判定：满足目标后进入 Day End。
5. 结算：发放奖励、记录统计、进入下一天。

### 关键接口（示意）
- `start_day(day_index: int)`
- `update_day_progress(delta: float)`
- `try_finish_day() -> bool`

---

### 4.2 Quest System（任务系统）

### 数据结构
任务定义建议最小字段：
- `quest_id`
- `type`（collect/photo/tutorial/custom）
- `target`
- `progress`
- `reward`（需求值、道具、货币）
- `prerequisites`
- `state`（locked/active/completed/claimed）

### 技术方案
- `QuestManager` 管理任务生命周期。
- `QuestRuntime` 保存运行态进度，避免直接污染静态配置。
- 任务监听 EventBus 事件（如 `on_item_collected`、`on_photo_taken`）自动推进。

### 教程任务（Tutorial quest）落地
- 使用任务链 `tutorial_01 -> tutorial_02 -> tutorial_03`。
- 每步教程绑定 UI 引导（手指提示、遮罩高亮、按钮锁定）。

### 奖励回流
- 完成任务后调用 `NeedsManager.apply_reward()` 与 `InventoryManager.add_item()`。
- 奖励领取采用“可重复调用幂等校验”，避免重复发放。

---

### 4.3 Needs System（需求系统）

### 需求模型
至少包含：
- `study_needs`
- `living_stamina`
- `living_hungry`
- `star_need`

统一取值建议：`0 ~ 100`。

### 数值更新策略
- **被动衰减**：按时间衰减（每 N 秒 -x）。
- **主动变化**：任务/道具/行为触发加减。
- **阈值事件**：低于阈值触发 debuff 或 UI 警报。

### 实现要点
- 使用 `Timer` 或 `_process(delta)` 做周期更新。
- 通过信号 `need_changed(name, old, new)` 驱动 HUD 实时刷新。
- 配置化衰减参数：`needs_balance.json` 管理衰减速度与上下限。

---

### 4.4 Inventory（背包系统）

### 最小功能
- 道具增删改查（增量、消耗、堆叠上限）。
- 道具分类（如 `tool`, `consumable`, `key_item`）。
- 与任务系统联动（收集目标、提交目标）。

### 数据结构建议
- `item_id`
- `count`
- `meta`（可选扩展字段，如耐久、品质）

### 与 Mindmap 对齐
- 首批保证 `Phone` 作为关键道具可获取/可展示。
- 预留空白节点对应的第二道具类型扩展能力。

---

## 5. 数据驱动与配置体系

#### 5.1 配置文件格式
推荐 JSON（便于策划维护），运行时转内存模型：
- `tutorial_quests.json`
- `daily_quests.json`
- `needs_balance.json`
- `items.json`

#### 5.2 配置热更新（单机模式下）
- 开发期：启动时自动重载配置。
- 正式版：仅在版本更新时替换配置。
- 后续联网：通过远端配置 + 本地缓存校验版本号。

#### 5.3 配置校验
增加简单校验器：
- ID 唯一性。
- 前置任务是否存在。
- 奖励字段是否完整。
- 数值上下限是否合法。

---

## 6. 存档与状态恢复

#### 6.1 存档内容
- 当前天数与阶段。
- 任务运行态（进度、状态、领奖标记）。
- 需求值。
- 背包内容。
- 教程引导状态。

#### 6.2 存档策略
- 手动存档 + 关键节点自动存档（每日结束、任务完成后）。
- 双存档槽（主存档 + 备份存档）防止损坏。
- 版本号字段支持迁移：`save_version`。

#### 6.3 存档格式
- MVP 用 `ConfigFile` 或 JSON。
- 中后期改二进制压缩（减小体积、防篡改）。

---

## 7. UI 与交互技术方案

#### 7.1 UI 场景拆分
- `HUD`：天数、需求条、任务简报。
- `QuestPanel`：任务详情、领奖。
- `InventoryPanel`：道具列表与使用。

#### 7.2 数据绑定策略
- UI 不直接读写业务数据。
- 通过事件/接口更新：
  - `EventBus.emit("quest_updated", payload)`
  - `EventBus.emit("need_changed", payload)`

#### 7.3 引导系统
- 独立 `GuideManager` 控制教程步骤。
- 每步配置绑定目标节点路径（NodePath）与触发条件。

---

## 8. 性能与工程质量

#### 8.1 性能优化
- 控制 `_process` 逻辑量，业务更新尽量事件化。
- 面板按需实例化（懒加载）减少首屏开销。
- 减少频繁创建临时对象，复用数据结构。

#### 8.2 日志与调试
- 定义日志等级（INFO/WARN/ERROR）。
- 提供 Debug Overlay：实时显示天数、任务进度、需求值。

#### 8.3 自动化检查（建议）
- GDScript Lint（如有 CI 环境可接入）。
- 配置校验脚本在 CI 上执行，阻断非法配置入库。

---

## 9. 分阶段实施计划（Godot 版）

### P0（1~2 周）：框架搭建
- 完成项目目录、Autoload、事件总线、基础场景切换。
- 建立 Needs + Day Cycle 最小可运行链路。

### P1（2~4 周）：核心玩法 MVP
- 完成 Tutorial quest 与基础任务链。
- 打通任务奖励对 Needs 与 Inventory 的回流。
- 完成本地存档。

### P2（4~6 周）：可运营能力
- 增加每日任务池、数值配置化。
- 完善引导系统与 UI 反馈。
- 加入配置校验与调试面板。

### P3（6+ 周）：扩展与上线准备
- 增强道具系统（更多类型与效果）。
- 导出多平台构建。
- 评估并接入云存档/排行榜。

---

## 10. 风险与规避

1. **系统耦合过高**
   - 规避：统一事件总线 + Manager 边界约束。

2. **配置与代码不一致**
   - 规避：配置校验器 + 启动时断言。

3. **迭代导致存档不兼容**
   - 规避：存档版本迁移函数与回滚策略。

4. **教程体验割裂**
   - 规避：教程状态独立管理，不与业务逻辑硬编码混写。

---

## 11. 结论

基于当前思维导图，采用 **Godot 4 + GDScript + 事件驱动 + 配置驱动** 的技术方案可以在较低复杂度下快速实现可玩版本，并支持任务、需求、背包三大系统的持续扩展。建议优先完成 Core loop 与 Tutorial 任务链，随后再扩展配置化和运营化能力。
