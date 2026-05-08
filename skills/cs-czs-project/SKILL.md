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
| `models/` | 读写 | 所有 `.blend` 源文件、参考图和制作中模型的工作区。玩家角色放在 `models/cs_player/` 下。 |

## 资产管线规则

1. **创建任何新资产前**，先翻看 `game_res/czero/models/` 和 `game_res/czero/materials/` 下同类型现有资产，搞清楚已有规范。
2. **源 `.blend` 文件**放在 `models/` 下（玩家角色放 `models/cs_player/`）。
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
