# Mindmap 整理文档

> 来源：`Image/Mindmap.png`

## 1. 总体结构
Mindmap 的主线为：

`Star project` → `Systems`

`Systems` 下分为四个主要系统分支：
1. Core loop
2. Quest system
3. Needs
4. Inventory

---

## 2. Core loop（核心循环）
核心循环展示了按“天”推进的基础玩法：

1. **Start a day**（开始新的一天）
2. **Do Star related quest**（执行与 Star 相关的任务）
3. **Reach X point on Star needs**（使 Star needs 达到某个点数/阈值）
4. 返回 **Start a day**，形成循环

说明：该分支体现了游戏日常推进节奏，属于最核心的时间与任务闭环。

---

## 3. Quest system（任务系统）
任务系统下包含教程任务与任务链设计：

- **Tutorial quest**（教程任务）
  - 图中出现了两个 Tutorial quest 节点，推测为教程任务的不同阶段或分支。

- 任务链示例：
  1. **Quest1: Collect 1 goo...**（收集类任务，原图文字末尾被截断）
  2. **Quest: Photo**（拍照/图片相关任务）
  3. **Quest design**（任务设计）

- 奖励相关：
  - **Quest reward: Start needs...**（任务奖励与 Start/Star needs 相关，原图文字末尾被截断）

说明：该分支强调“教程引导 → 任务目标 → 奖励回流需求系统”的设计思路。

---

## 4. Needs（需求系统）
Needs 分为三类：

1. **Study needs**（学习需求）
   - **Homeworks**（作业）

2. **Living needs**（生活需求）
   - **Stamina**（体力）
   - **Hungry**（饥饿）

3. **Star need**（Star 需求）
   - 作为独立需求项存在，并与核心循环目标关联。

说明：需求系统承担状态管理作用，既包含学习/生活等基础状态，也包含与主线 Star 相关的专项状态。

---

## 5. Inventory（背包系统）
Inventory 分支目前可见：

- **Phone**
- 一个未命名空白节点（可能预留给后续道具类型）

说明：背包系统处于初步设计阶段，已明确至少包含手机类道具。

---

## 6. 可落地的产品拆分建议（基于当前图）
为便于后续开发，可按以下顺序实现：

1. **MVP 1：核心循环 + 基础需求值**
   - 实现“开始一天 → 做任务 → 需求值变化 → 结算/进入下一天”。

2. **MVP 2：教程任务与任务链**
   - 完成 Tutorial quest；
   - 增加收集类与拍照类任务；
   - 打通任务奖励对需求系统的影响。

3. **MVP 3：背包与道具扩展**
   - 实现 Phone 的获取/使用；
   - 补全第二个道具槽位与道具分类。

4. **MVP 4：系统平衡与任务设计完善**
   - 调整需求衰减与任务奖励；
   - 完善 Quest design 文档与配置。

---

## 7. 图中可疑/待确认项
由于原图分辨率与节点文字可见范围限制，以下内容建议在源文件中二次确认：

- `Collect 1 goo...` 的完整任务描述；
- `Quest reward: Start needs...` 的完整奖励字段；
- `Star need` 与 `Start needs` 命名是否一致（是否存在拼写或概念区分）；
- Inventory 下空白节点的目标定义。

