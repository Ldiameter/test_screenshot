# 《灵魂回响》Demo 项目结构与实现总结

> 文档基线：本地源码提交 `f28d1b7`（2026-09-02）。本文描述第一至第六里程碑、其后的试玩体验改进，以及当前代码的实际状态。

## 1. 项目概况

本项目是使用 Godot 4.7.2 和 GDScript 实现的 3D 核心玩法 Demo。目标不是制作完整关卡、成长或剧情系统，而是验证以下玩法命题：

- 战士在实时横版战斗中处理“物理层”；
- 游侠在战术俯视模式中处理“空间层”；
- 法师通过灵魂视界修改已有状态的“规则层”；
- 三种模式操作的是同一个战场、同一组实体和同一份状态，而不是三套互相同步的世界；
- 玩家通过跨模式利用状态获得灵魂能量（SE），再用 SE 改写战斗规则。

当前 Demo 已完成第一至第六里程碑，并在此基础上加入开始/暂停菜单、角色头顶信息、命中反馈、弹体、死亡退场、远程索敌、战斗音效和中英文切换。

## 2. 总体技术方案

### 2.1 单一权威战场

`Battlefield` 是运行时的权威上下文，持有：

- 全部 `BattleEntity`；
- 陷阱等环境对象；
- 当前模式与战斗时间；
- 战术暂停和菜单暂停状态；
- 最近战斗事件与最近跨模式连锁；
- `SoulEnergySystem`、`TelemetryRecorder`、`GameLocalization` 和 `CombatAudio`。

横版与俯视模式只切换摄像机、输入解释和时间状态。实体始终保留原来的 `global_position`、生命、架势和状态，因此不存在“横版坐标”和“战棋坐标”两份数据。

### 2.2 三种模式对应三层操作

| 模式 | 表现 | 时间 | 主要职责 |
| --- | --- | --- | --- |
| 战士 / Physical | 横版摄像机 | 实时运行 | 移动、攻击、格挡、完美格挡、盾击和击退 |
| 游侠 / Spatial | 战术俯视摄像机 | 战术暂停 | 选择目标、技能射击、放置陷阱和空间规划 |
| 法师 / Rule | 灵魂视界叠色 + 俯视摄像机 | 战术暂停 | Echo、Extend、Transfer，修改已有状态规则 |

`ModeManager` 负责模式切换。进入游侠或法师模式时，`Battlefield.simulation_paused` 为 `true`；敌人 AI、弹体和状态倒计时停止。切回战士模式后继续模拟。游侠和法师完成一次动作后保持在当前模式，等待玩家自行选择下一步。

### 2.3 通用实体与状态模型

`BattleEntity` 是战士、游侠和敌人的共同基类，统一处理：

- 队伍、生命、架势、轨道、朝向和当前动作；
- 伤害、死亡、状态添加/消耗/过期；
- 战斗事件及遥测上报；
- 头顶名称、生命和状态显示；
- 攻击/受击颜色反馈；
- 敌人死亡后的碰撞关闭与退场动画；
- 战术模式下的鼠标点击信号。

状态由 `StatusComponent` 管理，具体实例是 `StatusEffect`。状态除了类型和持续时间，还记录来源实体、来源模式、标签、元数据及 Echo 触发倍率。当前状态类型为：

| 状态 | 作用来源或用途 |
| --- | --- |
| `OFF_BALANCE` | 战士完美格挡制造，供游侠转换 |
| `PINNED` | 游侠利用失衡后制造，限制敌人并可被法师修改 |
| `BOUND` | 敌人被推进游侠陷阱后产生 |
| `IMPACT` | 高速碰撞产生的物理状态 |
| `GUARD_BROKEN` | 架势归零后产生，可让游侠精准射击获得额外伤害 |

### 2.4 跨模式连锁与 SE

状态记录其 `source_mode`。当前模式利用了其他模式制造的状态时，`Battlefield.record_cross_mode_interaction()` 会记录连锁，并按规则给予 1 点 SE，最大值为 3。

典型数据流为：

```text
战士完美格挡
  → 敌人获得 OFF_BALANCE（来源：Physical）
  → 游侠技能射击消费 OFF_BALANCE
  → 敌人获得 PINNED（来源：Spatial）
  → 记录 Physical → Spatial 连锁并获得 SE
  → 法师消耗 SE，对 PINNED 执行 Echo / Extend / Transfer
```

### 2.5 事件、遥测与显示层

核心逻辑记录稳定的英文事件文本，HUD 显示时再由 `GameLocalization` 翻译。这使切换语言后，当前事件和跨模式连锁可以立即以新语言重绘，而不改变战斗数据。

`TelemetryRecorder` 记录模式停留、切换次数、状态创建与利用、跨模式连锁、SE 获取、法师施法、玩家受伤和敌人击杀，并可输出 `user://soul_echoes_telemetry.csv`。重点指标是跨模式状态利用率和最大连锁深度，而不是单纯鼓励频繁切换模式。

## 3. 代码目录结构

```text
mark_two/
├─ project.godot                     Godot 项目配置与主场景入口
├─ export_presets.cfg                Windows Desktop 导出预设
├─ IMPLEMENTATION_SPEC.md            原始玩法与里程碑规范
├─ GODOT_QUICKSTART.md               本机 Godot 调试经验
├─ levels/
│  └─ prototype_arena.tscn           唯一原型战场，组装全部系统和实体
├─ battle/
│  ├─ battlefield.gd                 权威战场、时间、注册、事件和系统入口
│  ├─ battle_entity.gd               通用战斗实体、伤害、状态、反馈和退场
│  ├─ battle_lane.gd                 TOP/MIDDLE/BOTTOM 与 Z 坐标映射
│  ├─ combat_audio.gd                程序化战斗音效生成与播放
│  ├─ soul/soul_energy_system.gd     SE 获取、上限和消耗
│  ├─ status/                        状态类型、状态实例和状态容器
│  └─ telemetry/telemetry_recorder.gd 遥测统计与 CSV 输出
├─ mode/
│  └─ mode_manager.gd                三模式切换及暂停规则
├─ camera/
│  └─ camera_manager.gd              横版/战术摄像机与轨道指示切换
├─ characters/
│  ├─ warrior/warrior_controller.gd  战士实时移动、攻击、格挡和盾击
│  ├─ ranger/ranger_controller.gd    游侠自动战斗、战术操作与陷阱
│  ├─ ranger/ranger_projectile.gd    游侠弹体和命中回调
│  └─ mage/mage_controller.gd        灵魂视界和三种规则法术
├─ enemies/
│  ├─ hound/hound.gd                 接近、预警、冲锋、碰撞和恢复 AI
│  ├─ shield_guard/shield_guard.gd   正面减伤、架势和近战 AI
│  ├─ archer/archer.gd               射程、索敌、瞄准、射击和装填 AI
│  └─ archer/archer_projectile.gd    可被完美格挡反射的敌方弹体
├─ environment/
│  ├─ ranger_trap.tscn
│  └─ ranger_trap.gd                 陷阱检测、BOUND 与反向连锁
├─ ui/
│  ├─ pause_menu.gd                  开始、暂停、继续、重试和语言按钮
│  ├─ milestone_hud.gd               精简调试 HUD、模式提示和遥测
│  └─ game_localization.gd           英文/中文文本与动态事件翻译
├─ tests/                             六个里程碑、UI 和综合功能自动化测试
└─ docs/screenshots/                  里程碑截图与 2 秒间隔演示 GIF
```

`prototype_arena.tscn` 是组合根：它实例化战士、游侠、两只猎犬、盾卫、弓手、两套摄像机、三条轨道、HUD、菜单以及所有战场服务。角色并未拆成独立场景的部分仍直接配置在该原型场景中，这符合当前 Demo 优先验证玩法、避免过早资源化的原则。

## 4. 第一至第六里程碑实现情况

### 4.1 第一里程碑：统一战场

目标是建立 `Battlefield + Warrior + 1 Hound + Side Camera` 的最小实时战斗。

已实现：

- Godot 项目、原型场景和权威 `Battlefield`；
- `BattleEntity` 通用实体基类；
- 战士左右移动、普通攻击、格挡和完美格挡；
- 猎犬“接近 → 冲锋预警 → 冲锋 → 恢复”的状态机；
- 攻击前摇、命中时刻、距离判定和架势伤害；
- 完美格挡窗口及 `OFF_BALANCE` 状态；
- 横版摄像机与基础 Debug HUD；
- `tests/milestone1_smoke.gd` 自动验收。

当前版本在此基础上把普通长按格挡改为减伤：近战保留 25% 伤害，远程保留 40% 伤害；完美格挡仍完全免伤。因此里程碑 1 测试也已更新为验证当前减伤规则。

结论：**已完成并通过回归测试。**

### 4.2 第二里程碑：空间连续

目标是加入战术摄像机、游侠和三轨道，同时保证模式切换不复制世界。

已实现：

- `CameraManager` 在 Side 与 Tactical 两套摄像机间切换；
- `BattleLane` 将上、中、下三条轨道映射到唯一的 3D Z 坐标；
- 游侠实体、战术暂停、目标选择和战术射击入口；
- 切换摄像机前后对比同一实体实例和 `global_position`；
- `tests/milestone2_spatial_continuity.gd` 自动验收及两视角截图。

结论：**已完成，所有模式共享同一份实体和空间状态。**

### 4.3 第三里程碑：第一条跨模式链

目标链路：

```text
Perfect Block → OFF_BALANCE → Ranger Shot → PINNED
```

已实现：

- 战士在完美格挡窗口内拦截猎犬攻击；
- 攻击者获得来源为战士模式的 `OFF_BALANCE`；
- 游侠技能射击选中该实体并在弹体命中时结算；
- 射击消费 `OFF_BALANCE`，创建 `PINNED`；
- 记录 `PHYSICAL -> SPATIAL | OFF_BALANCE -> PINNED`；
- 连锁成功后获得 1 SE；
- `tests/milestone3_cross_mode_chain.gd` 验证完整因果链。

结论：**已完成，这是当前 Demo 的第一条核心跨模式玩法链。**

### 4.4 第四里程碑：反向空间链

目标链路：

```text
Ranger Trap → Warrior Shield Bash → Enemy enters Trap → BOUND
```

已实现：

- 游侠在战术暂停中沿 X 轴规划陷阱位置和目标轨道；
- 陷阱作为同一战场里的 `Area3D` 环境对象存在；
- 战士盾击造成小额伤害并施加实际位移冲量；
- 敌人被战士推进陷阱后获得 `BOUND`；
- 陷阱根据最近冲量来源识别反向连锁；
- 记录 `SPATIAL -> PHYSICAL | SHIELD BASH -> TRAP -> BOUND` 并给予 SE；
- `tests/milestone4_reverse_spatial_chain.gd` 验证空间规划到物理执行。

结论：**已完成，验证了战术布置可以改变后续实时物理结果。**

### 4.5 第五里程碑：法师规则层

目标是加入 SE、Soul Vision、Echo、Extend 和 Transfer。

已实现：

- `SoulEnergySystem`：SE 上限 3，每次法术消耗 1；
- 灵魂视界枚举战场上所有仍可操作的状态；
- `Extend`：将选中状态延长 3 秒；
- `Echo`：武装一次双倍触发，下一次相关利用获得 x2 效果；
- `Transfer`：把状态以剩余时间的 50% 复制给最近同队实体，并记录原始来源；
- 法术不会直接凭空创建战斗状态，只修改已由物理层或空间层产生的状态；
- 规则层操作写入跨模式事件和遥测，但不重复奖励 SE；
- `tests/milestone5_mage_rule_chain.gd` 覆盖 SE、三种法术和三层连锁。

结论：**已完成并通过回归测试。**

### 4.6 第六里程碑：综合遭遇

目标是加入三类敌人、完整战场和遥测，让玩家自由组合既有系统。

当前综合战场包含：

- 两只 `Hound`：以明显预警发动高速冲锋；
- 一名 `ShieldGuard`：正面承伤降至 25%，拥有独立架势条；架势归零后进入 `GUARD_BROKEN`，游侠可消费该状态完成精准射击并追加伤害；
- 一名 `Archer`：寻找最近目标、保持攻击距离、瞄准、发射弹体并装填；
- 战士和游侠两个玩家阵营实体，以及可动态生成的陷阱；
- 模式时间、状态利用、连锁深度、SE、法术、玩家受伤和击杀参与统计；
- `tests/milestone6_comprehensive_encounter.gd` 综合验证敌人阵容、目标选择、盾卫破防、深度三连锁、弓手攻击和 CSV 遥测。

结论：**已完成并通过回归测试。** 玩法范围仍严格停留在核心 Demo，没有提前扩展 Roguelike、成长、Boss、剧情或正式关卡。

## 5. 第六里程碑之后的修改

### 5.1 第一轮试玩 UI 与反馈优化

第六里程碑完成后，首先围绕“方便玩家测试”进行了以下改动：

1. **开始/暂停/重试菜单**
   - 启动后先显示菜单，战场作为背景但不立即开战；
   - 游戏中按 Esc 可暂停并重新打开菜单；
   - 支持继续和重新开始；
   - 菜单暂停与战术暂停分开管理。

2. **精简 Debug HUD**
   - HUD 不再集中列出所有角色生命和状态；
   - 每个角色头顶显示名称、生命值和当前状态；
   - HUD 保留模式、时间、SE、最近事件、最近连锁、遥测和当前模式操作提示。

3. **模式动作完成后不自动切换**
   - 游侠射击/放置陷阱后继续停留在游侠模式；
   - 法师施法后继续停留在灵魂视界；
   - 玩家通过数字键自行决定后续模式。

4. **战斗视觉反馈**
   - 攻击、受击和格挡使用短暂颜色变化；
   - 材质在运行时复制，避免多个实体共享材质导致同时变色；
   - 游侠射击改为可见弹体；
   - 伤害和状态转换在弹体命中后结算，而不是按键时立即结算。

5. **UI 流程验证**
   - 新增 `tests/playtest_ui_flow.gd`；
   - 新增 `tests/capture_playtest_ui_gif.gd`，输出每帧停留 2 秒的演示 GIF。

### 5.2 Git、截图和 Windows 分发

- 项目初始化为 Git 仓库；
- 本地源码仓库绑定私有源码远端 `Ldiameter/mark_two`；
- 截图和演示 GIF 上传至公开仓库 `Ldiameter/test_screenshot`，以便 PC 和远程手机通过 HTTPS 查看；
- 增加 Windows Desktop 导出预设；
- 导出的单文件 EXE 约 36 MiB，压缩包约 37.45 MiB，曾上传至公开截图仓库供试玩；
- 建立约定：之后每次涉及代码修改的任务完成后创建 Git 提交。

注意：公开仓库中的现有 Windows 压缩包生成于最新一轮战斗与本地化改动之前。若要测试本文描述的全部最新功能，需要重新导出并替换该压缩包。

### 5.3 第二轮战斗可用性改进

1. **敌人死亡退场**
   - 敌人死亡瞬间将碰撞层和碰撞掩码设为 0，并禁用碰撞体；
   - 隐藏头顶信息和选择标记；
   - 在 0.85 秒内缩小、下沉并倾倒，随后 `queue_free()`；
   - 离开场景树时从 `Battlefield.entities` 注销，尸体不再阻挡战士。

2. **长按格挡与远程反弹**
   - 普通长按格挡：近战伤害倍率 0.25，远程伤害倍率 0.40；
   - 完美格挡：完全免伤并让攻击来源进入 `OFF_BALANCE`；
   - 敌方远程攻击改为 `ArcherProjectile`；
   - 完美格挡远程弹体时执行回程弹道，命中原弓手后再计算伤害。

3. **游侠实时 AI 与战术点选**
   - 横版实时模式下，游侠持续选择最近敌方单位；
   - 超出 10 单位攻击范围时以 2 单位/秒自动接近；
   - 进入范围后每 1.35 秒进行一次普通射击；
   - 自动普通攻击只造成基础伤害，不消费状态、不生成 `PINNED`、不自动释放技能；
   - 战术模式中可直接用鼠标点击敌人，也保留 Q/E 循环选择；
   - 战术技能射击射程为 18 单位。

4. **远程单位射程与索敌**
   - 游侠与弓手均使用 `Battlefield.find_closest_opponent()`；
   - 目标超出射程时进入移动索敌状态；
   - 弓手进入范围后依次执行瞄准、发射弹体和装填；
   - 目标死亡或离开射程时重新索敌，不再隔着全场立即结算伤害。

5. **战斗音效**
   - `CombatAudio` 在运行时生成并缓存短促的 `AudioStreamWAV`，当前不依赖外部音频素材；
   - 已覆盖近战攻击、远程攻击、技能、普通受击、格挡受击、完美格挡和死亡；
   - 无头测试仍记录 cue 次数，但不创建播放器，避免测试节点泄漏。

6. **中英文切换**
   - 菜单按钮和 F1 均可切换语言；
   - 覆盖菜单、Debug HUD、控制提示、角色名称、生命、状态、动作、模式、轨道、陷阱、灵魂视界选择和战斗事件；
   - 内部状态枚举与事件仍保持稳定英文标识，中文只属于显示层；
   - 当前语言是运行期状态，重启游戏后恢复默认英文。

7. **新增测试和演示**
   - `tests/combat_usability_features.gd` 验证死亡退场、近/远程减伤、弹体反射、游侠自动移动/攻击、鼠标选敌、全部音效 cue 与中文 UI；
   - `tests/capture_combat_update_gif.gd` 捕获 7 个关键画面；
   - `docs/screenshots/combat_usability_update.gif` 为 768×432、每帧 2 秒的演示；
   - 原有测试和截图脚本在运行时关闭游侠自主 AI，保证旧里程碑验收可重复且不受后台射击干扰。

## 6. 当前输入与测试入口

### 6.1 默认输入

| 输入 | 功能 |
| --- | --- |
| `1` | 战士横版实时模式 |
| `2` | 游侠战术模式 |
| `3` | 法师灵魂视界 |
| `A / D` | 战士移动；战术模式下移动陷阱光标或选择法师状态 |
| `Q / E` | 游侠循环选择目标 |
| 鼠标左键 | 游侠战术模式直接选择敌人 |
| `J` | 战士攻击 / 游侠技能射击 / 法师 Echo |
| `K` | 战士格挡 / 法师 Extend |
| `L` | 战士盾击 / 游侠陷阱 / 法师 Transfer |
| `Esc` | 打开或关闭暂停菜单 |
| `F1` | 中英文切换 |

### 6.2 自动测试

| 测试脚本 | 覆盖范围 |
| --- | --- |
| `tests/milestone1_smoke.gd` | 统一战场、敌人时序、格挡和攻击 |
| `tests/milestone2_spatial_continuity.gd` | 两摄像机下的实体身份和位置连续性 |
| `tests/milestone3_cross_mode_chain.gd` | 完美格挡到 PINNED 的第一条连锁 |
| `tests/milestone4_reverse_spatial_chain.gd` | 陷阱、盾击、BOUND 反向连锁 |
| `tests/milestone5_mage_rule_chain.gd` | SE、灵魂视界与三种法师能力 |
| `tests/milestone6_comprehensive_encounter.gd` | 综合敌人、破防、弓手和遥测 |
| `tests/playtest_ui_flow.gd` | 开始、暂停、继续和重试菜单流程 |
| `tests/combat_usability_features.gd` | 第六里程碑后的全部战斗可用性改动 |

本机命令行测试若需要写入遥测 CSV，应把 `APPDATA` 和 `LOCALAPPDATA` 指向项目内可写目录；普通 Godot 编辑器运行不需要这一处理。详细调试方式见 `GODOT_QUICKSTART.md`。

## 7. 当前完成度与边界

当前代码已经形成一个可运行、可自动验证的核心战斗原型，六个里程碑和后续可用性需求均有对应实现。项目仍有意保留以下原型边界：

- 只有一个综合原型场景，没有正式关卡流程；
- 模型、动画和音效仍是低成本原型表现，音效为程序化占位音；
- 本地化使用代码字典，没有外置翻译表，也未持久化语言选项；
- Debug HUD 和遥测仍面向开发测试，不是最终玩家 UI；
- 未实现 Roguelike、成长、卡牌、完整回合制、Boss、剧情、存档和设置系统；
- 最新源码尚未重新导出 Windows 包；
- 是否进入更大范围开发，仍应以真人试玩能否稳定理解“战士=物理、游侠=空间、法师=规则”为判断标准。

## 8. Git 实现时间线

| 提交 | 内容 |
| --- | --- |
| `3666bfb` | 初始化项目并实现里程碑 1–4 |
| `dffffda` | 添加可通过远程地址查看的里程碑截图 |
| `519b2ea` | 完成里程碑 5–6、试玩 UI 优化、测试 GIF 和 Windows 导出配置 |
| `f28d1b7` | 完成死亡退场、格挡反弹、游侠 AI、射程、音效和本地化 |

设计里程碑与 Git 提交并非一一对应：里程碑 1–4 在首个提交中一起落库，里程碑 5–6 与第一轮试玩 UI 改进在同一提交中落库；代码和测试仍按各里程碑独立组织。
