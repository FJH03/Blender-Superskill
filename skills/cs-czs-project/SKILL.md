---
name: cs-czs-project
description: CS:CZ Source project layout, directory conventions, asset pipeline rules, and coding patterns. Use when working anywhere in this project — modeling, coding, config editing, asset importing, or debugging. Always load alongside any domain-specific skill.
---

# CS:CZ Source — 项目知识库

## 项目是什么

这是一个独立运行的 **《反恐精英：零点行动 起源》(CS:CZS)** 游戏，基于自主魔改的起源引擎（源自 2018 年 TF2 泄露版，经由 nillerusr 移植）。**不是 Steam Mod**——有自己的独立服务端和客户端，使用定制引擎。

已实现的引擎级特性：CSGO 风格的瞄准镜模糊和移动晃动、每个玩家独立的 Addon 手臂模型、战队标签、死亡竞赛模式、实体发光、多材质脚步声、第一人称腿模、第三人称后坐力动画、PCF 特效缓存系统、HL2 NPC 移植到 cstrike 分支、大量控制台命令和 ConVar 改进。

## 目录地图

| 目录 | 权限 | 用途 |
|---|---|---|
| `game_src/` | 只读 | C++ 游戏引擎源码。完整的起源引擎魔改版。用于了解游戏行为、ConVar 标志位、渲染限制、引擎层面的约束。**严禁在此放模型或资产文件。** |
| `game_res/` | **只读参考** | 现有游戏资产库——已编译的模型（`.mdl`、`.vvd`、`.vtx`）、材质（`.vmt`、`.vtf`）、贴图、音效、配置文件。翻阅此目录来了解美术风格、面数预算、贴图尺寸、命名规范。**严禁写入或导出到此目录。** |
| `models/` | 读写 | 所有 `.blend` 源文件、参考图和制作中模型的工作区。按模型类型分子目录，每个类型目录内含 `standard/` 存放该类标准模型。 |
| `models/cs_player/` | 读写 | 玩家角色模型。`standard/` 子目录存放 GIGN（ct_gign）作为玩家模型标准——其 `.smd` 源文件、`.qc` 编译脚本、动画均为标杆，其他角色（SAS、GSG9、Spetsnaz 等）应对齐此标准。 |
| `models/v_model/` | 读写 | 第一人称 viewmodel 的总目录，含两个子类：`c_arm/standard/`（手臂，GIGN 的 v_arm_gign 为标杆）和 `viewmodel/<武器名>/standard/`（武器本体，目前有 ak47、elites、m4a1、usp 四把标准武器）。手臂和武器的标准模型配合使用，用于理解动画驱动关系。 |

## 模型知识链——读懂模型是怎么工作的

制作或修改任何模型前，**不要孤立地看一个文件**。项目里有一套从源码到运行时的完整链路可以追溯：

### 三层对照法

| 层 | 在哪看 | 能看到什么 |
|---|---|---|
| **源文件层** | `models/<类型>/standard/` | `.smd` 网格源文件、`.qc` 编译脚本——骨骼结构、attachment 挂点、碰撞体定义、动画序列声明、身体部位拆分逻辑 |
| **编译产物层** | `game_res/czero/models/` | `.mdl`（模型）、`.vvd`（顶点数据）、`.vtx`（优化网格）、`.phy`（物理碰撞）——编译后的最终形态、实际面数、材质引用路径 |
| **引擎逻辑层** | `game_src/` 源码 | C++ 代码里模型是如何被加载、渲染、附加、动画混合的——viewmodel 渲染逻辑、Addon 手臂替换机制、玩家模型与手臂的关联方式 |

### 例子一：搞懂玩家模型和手臂的关系

1. 去 `models/cs_player/standard/` 看 `ct_gign.qc`——搞清楚 body/head/leftarm/rightarm 是怎么拆分的，挂点在哪
2. 去 `models/v_model/c_arm/standard/` 看 `v_arm_gign.qc`——搞清楚手臂的独立编译逻辑，它和玩家身体的 arm 是什么关系
3. 去 `game_res/czero/models/player/` 看 `ct_gign.mdl` 编译结果——验证最终产物长什么样
4. 去 `game_src/` 搜 viewmodel、Addon、c_arm 相关的 C++ 代码——理解引擎怎么把玩家身体、独立手臂、武器串起来的

### 例子二：搞懂 viewmodel 动画是怎么驱动的

手（c_arm）和武器（viewmodel）是两套独立的模型，但动画是协同播放的。通过对照两者的标准源文件，可以理解动画驱动逻辑：

1. 去 `models/v_model/c_arm/standard/` 看手臂的 `.smd` 和 `.qc`——骨骼有哪些、attachment 挂点（如 weapon_bone）定义在哪
2. 去 `models/v_model/viewmodel/<武器>/standard/` 看对应武器的 `.smd` 和 `.qc`——武器的骨骼结构和挂点如何匹配手臂
3. 对照手臂动画序列（`v_arm_*_anims/`）和武器动画序列（`v_rif_*_anims/` 或 `v_pist_*_anims/`）——同一个动作（如开火、换弹）在手臂和武器上各有哪些帧、如何同步
4. 去 `game_src/` 搜 viewmodel 渲染、weapon attachment、animation layer 相关代码——理解引擎是如何在同一时刻对手臂和武器播放配对动画的

做新武器 viewmodel 时，**手臂动画和武器动画必须配对设计**，不要只做武器而不管手臂怎么握。

## 资产管线规则

1. **接手任何模型任务前，先走一遍上面的三层对照链**，把源文件、编译产物、引擎逻辑串起来理解。
2. **源 `.blend` 文件**放在 `models/` 下（玩家角色放 `models/cs_player/`，手臂放 `models/v_model/c_arm/`，武器 viewmodel 放 `models/v_model/viewmodel/<武器名>/`）。
3. **严禁覆盖或修改 `game_res/`**——它是已知正确资产的参考快照。
4. **模型命名**沿用已有模式：玩家用阵营前缀（`ct_`、`t_`），武器和道具用功能性命名。
5. `game_res/` 里看到的编译产出的模型文件（`.mdl`、`.vvd`、`.vtx`、`.phy`）是 Source 的 studiomdl 编译器从 `.smd` / `.dmx` 源文件生成的——不是手工放进去的。

## 引擎核心模块（game_src/ 下）

| 模块 | 职责 |
|---|---|
| `appframework/` | 程序入口、SDL 集成、平台窗口管理 |
| `engine/` | 核心引擎——渲染、网络、客户端/服务端框架 |
| `game/` | 游戏 DLL 逻辑（cstrike 分支，已移植 HL2 NPC） |
| `vguimatsurface/` | VGUI2 界面渲染层 |
| `materialsystem/` | 材质与着色器系统 |
| `studiorender/` | 模型渲染管线 |
| `vphysics/` | 物理系统（约束、布娃娃、碰撞） |
| `tier0/`、`tier1/`、`tier2/`、`tier3/` | 工具库（数学、线程、文件 IO、数据结构） |
| `public/` | 引擎公共头文件和接口 |
