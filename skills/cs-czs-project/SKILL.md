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

## Blender Source Tools 导出工作流（MCP 自动化）

通过 Blender MCP 调用 `io_scene_valvesource`（Source Tools）导出 `.smd` 时，以下为已验证的坑和正确做法：

### 关键发现

1. **操作符不继承 `ExportHelper`**：`EXPORT_SCENE_OT_smd` 没有 `filepath` 参数。传入 `filepath=` 会报 `keyword "filepath" unrecognized`。导出路径通过场景属性控制。

2. **必须传 `export_scene=True`**：不传此参数时，导出器走"选中对象"路径（`getSelectedExportables()`），可能找不到有效对象报 `Found no valid objects for export`。

3. **导出路径是目录而非文件**：`scene.vs.export_path` 设置为目标**目录**（带尾部 `\\`），文件名由 export_list 中的 `name` 决定。

4. **基于 Collection 而非 Selection**：导出器通过 `scene.vs.export_list` 中的 collection 条目决定导出哪些对象，不是通过 Blender 选中状态。

5. **动画导出通过 armature 的 `vs.subdir`**：设置 `armature.vs.subdir` 为动画子目录名（如 `"v_arm_gign_anims"`），导出时会在此子目录下生成动画 `.smd`。

### 完整导出模板

```python
import bpy
scene = bpy.context.scene

# 1. 清空并重建 export_list
scene.vs.export_list.clear()
col = bpy.data.collections.new('export_col')
scene.collection.children.link(col)

# 2. 将目标对象移入 collection（先从其原有 collection 中移除）
for obj_name in ['mesh_obj', 'armature_obj']:
    obj = bpy.data.objects[obj_name]
    for c in list(obj.users_collection):
        c.objects.unlink(obj)
    col.objects.link(obj)

# 3. 添加导出条目
item = scene.vs.export_list.add()
item.name = "output.smd"      # 输出文件名
item.collection = col          # 用 .collection 而非 .item（只有 getter）
item.ob_type = 'COLLECTION'

# 4. 设置场景导出属性
scene.vs.export_path = r"E:\target\dir\\"   # 目录，必须带尾部 \\
scene.vs.export_format = 'SMD'
scene.vs.smd_format = 'SOURCE'

# 5. 如需导出动画，设置 armature 子目录
armature = bpy.data.objects['armature_obj']
armature.vs.subdir = "anims_subdir"

# 6. 执行导出
bpy.ops.export_scene.smd(export_scene=True)
```

### 常见错误速查

| 错误 | 原因 | 解决 |
|------|------|------|
| `keyword "filepath" unrecognized` | Source Tools 不用 filepath 参数 | 用 `scene.vs.export_path` 代替 |
| `Found no valid objects for export` | 未传 `export_scene=True`，或 export_list 为空 | 传 `export_scene=True` 并正确配置 export_list |
| `property 'item' has no setter` | `export_list[].item` 是只读属性 | 用 `.collection` 设置集合引用 |
| `Operator bpy.ops.object.select_all.poll() failed` | 不在 OBJECT 模式 | 先执行 `bpy.ops.object.mode_set(mode='OBJECT')` |
| MCP 连接断开（`WinError 10038`） | 执行了 `read_factory_settings` 会卸载所有插件 | **严禁在 MCP 会话中调用** `read_factory_settings`；如需重置场景，逐个删除对象 |
| 导入第二个 SMD 时骨骼合并 | SMD 导入器会把同名骨骼合并到已有骨架 | 这是正常行为；如需独立对比，先另存或导出骨架矩阵数值 |

### 导入 SMD 注意事项

- SMD 导入器会创建 `smd_bone_vis` 辅助网格，导出前应删除它以免干扰
- 导入动画时，导入器自动匹配已有骨架的骨骼名（Validate 模式）
- 导入后检查 `armature.animation_data.action` 确认动画已加载
