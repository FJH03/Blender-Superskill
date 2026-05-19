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

## 外部模型适配工作流 —— 双模型对照法

将任何外部来源的模型适配到 CS:CZ 标准时，**严禁跳过对照直接改**。无论模型类型（角色、手臂），统一走以下流程。

### 核心原则

| 原则 | 说明 |
|------|------|
| **限定范围** | 只改需要改的，不要无节制导出或批量操作。一次聚焦一个模型和它的标准对照物 |
| **先对照再替换** | 必须同时导入源模型（A）和该类模型的标准模板（B），逐项对比差异后再动手 |
| **多余部件做附件** | 不属于标准骨骼蒙皮范围的部分（装饰、附加布料、配件等），作为 bodygroup 处理，不参与骨骼网格操作 |

### 对照诊断清单

导入源模型和标准模板后，先完成以下对照再决定路线：

- 骨骼数量：源 vs 标准，差多少？谁多谁少？
- 骨骼命名：源是否使用目标命名规范？是否可建立一一映射？
- 骨骼层级：父子关系是否一致？根骨骼是否相同？
- 多余骨骼：源有哪些标准没有的骨骼？这些骨骼上有多少顶点权重？
- 缺失骨骼：标准有哪些源没有的骨骼？这些骨骼是否必需（如根骨骼、动画引用骨骼）？
- 空间位置：世界空间中对应骨骼的位置差多少？比例尺是否一致？
- 顶点组绑定：源 mesh 实际使用了哪些骨骼？有没有空组？

### 两条适配路线

根据对照诊断结果选择：

| 判断条件 | 路线 | 核心操作 |
|----------|------|----------|
| 骨骼层级结构可建立一一映射（即使命名不同） | **路线一：权重映射** | 把源 mesh 的蒙皮权重按骨骼对应关系搬运到模板骨骼上，保留原始刷权重成果 |
| 骨骼结构完全不同、无法建立映射 | **路线二：骨骼移植 + 重刷** | 用模板骨骼替换源骨骼，重新刷蒙皮权重；多余部件分离为 bodygroup |

路线一内部再分两种情况：
- 命名不同但层级一致 → 先建映射表，再搬运权重，最后删旧骨骼
- 命名相同但 rest pose 不同 → 只微调 rest pose 差异，权重不动

### 通用执行流程

1. **双导入**：源模型和标准模板同时放入 Blender，同框对比
2. **诊断**：逐项过对照清单，记录所有差异
3. **定路线**：根据诊断结果选择路线一或路线二
4. **执行**：
   - 路线一：建映射 → 搬运权重（含多余骨骼权重合并到最近等价骨骼）→ 补齐缺失骨骼 → 删旧骨骼 → 验证形态
   - 路线二：删源骨骼 → 导入模板骨架 → 父子化 mesh → 重刷权重 → 分离附件 → 验证形态
5. **附件分离**：不属于标准骨骼蒙皮的 mesh 部件拆为独立对象，在 QC 中用 `$bodygroup` 声明
6. **编译验证**：导出、编译为 `.mdl`，游戏内验证

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

## SourceIO —— Blender 插件参考（Source 引擎资产导入/导出）

SourceIO（GitHub: [REDxEYE/SourceIO](https://github.com/REDxEYE/SourceIO)，当前版本 v5.5.4）是一个 Blender 4.0+ 插件，用于导入/导出 Source 引擎的模型、贴图、材质和地图。MIT 协议开源。

### 对 CS:CZS 项目最实用的功能

| 功能 | 用途 | 与项目的关系 |
|---|---|---|
| **MDL 导入**（Source 1） | 将编译后的 `.mdl` 模型导入 Blender，含骨骼、蒙皮、flex（表情形变）、bodygroup（身体部件切换）、phy 碰撞体、动画 | 可直接读取 `game_res/` 中的编译模型作为参考，对比标准模型的骨骼结构和蒙皮 |
| **VTF 导入** | 将 `.vtf` 贴图导入 Blender | 读取参考贴图的实际尺寸、格式、mipmap 层级 |
| **VMT 导入** | 将 `.vmt` 材质导入 Blender，支持 BVLG（BlenderVertexLitGeneric）着色器节点 | 还原材质的着色器参数、纹理引用链 |
| **VTF 导出** | 从 Blender 导出 `.vtf` 贴图，支持 RGBA8888/DXT1/DXT5 等格式、mipmap 生成、分辨率限制、法线贴图绿色通道翻转 | 制作新贴图时的编译输出（替代 VTFCmd） |
| **BSP 导入**（Source 1） | 导入编译后的 `.bsp` 地图，含几何体和实体占位符（可延迟加载） | 查看地图布局、实体位置作为场景参考 |
| **GoldSrc BSP 导入** | 导入 GoldSrc `.bsp` 地图 | 兼容旧格式地图 |

### 游戏自动检测与内容管理

SourceIO 通过扫描 `gameinfo.txt` 自动识别游戏并挂载资源路径：

- **检测器链**：`Source1Detector` 会 backwalk 查找 `platform/` 和 `bin/` 目录，然后解析 `gameinfo.txt` 获取 VPK 和松散文件路径
- **内容提供器**：支持 `Source1GameInfoProvider`（松散文件 + VPK）、`VPKContentProvider`（直接读取 VPK 包）
- **已支持游戏**：HL2 及章节、CSGO、TF2、GMod、L4D2、Portal 1/2、BlackMesa、SFM、Titanfall 等
- **手动挂载资源**：可在 Scene 属性 → SourceIO 面板中添加任意路径作为资源目录

对 CS:CZS 项目，将 `game_res/` 路径挂载为资源目录后，导入 MDL 时即可自动解析材质引用的 VMT/VTF。

### BSP 实体系统

导入 BSP 后，实体（prop、NPC 等）显示为十字占位符：

- 选中实体后，右侧边栏出现 **SourceIO 面板**，显示实体属性
- **Load Entity** 按钮：批量加载选中实体的实际模型
- 加载选项：
  - `Use BVLG`：使用仿 TF2/HL2 风格的 EEVEE 着色器节点（关闭则为标准 PBR 近似）
  - `Use Instances`：用 Collection Instance 替代真实几何体，提升性能
  - `Replace entity`：用模型替换占位符（否则模型父子化到占位符下）

### VTF 导出选项

导出 VTF 时可选参数（与 VTFCmd 对标）：

| 参数 | 说明 |
|---|---|
| `img_format` | 纹理格式：RGBA8888、DXT1、DXT5、RGB888 等 |
| `generate_mipmaps` | 是否生成 mipmap |
| `mip_filter` | Mipmap 滤波：Catrom、Gaussian、Mitchell 等 |
| `flip_green` | 翻转绿色通道（法线贴图用） |
| `resize_to_pow2` | 强制缩放到 2 的幂 |
| `limit_resolution` | 限制最大分辨率，可保持宽高比 |

### 节点化导出系统（WIP）

SourceIO 有一个实验性的节点树导出系统（`SourceIOModelDefinition`），用于在 Blender 中用节点定义模型导出结构：

- **模型节点**：`SourceIOModelNode`（输出模型定义，设 `$modelname`）
- **材质节点**：`SourceIOMaterialNode`、`SourceIOVertexLitGenericNode`
- **组织结构节点**：`SourceIOBodygroupNode`、`SourceIOSkinNode`、`SourceIOSkingroupNode`
- **贴图处理节点**：VTF 转换、通道操作、法线处理、色彩空间转换、贴图预览
- **对象引用节点**：`SourceIOObjectNode` 引用场景中的网格对象

此系统目前标记为 WIP，可关注其发展用于未来的自动化模型导出管线。

### SourceIO 与 Source Tools 的分工

| 场景 | 用什么 | 原因 |
|---|---|---|
| 导入编译后的游戏资产做参考 | **SourceIO** | 直接读取 `.mdl`/`.vtf`/`.vmt`/`.bsp` |
| 导出 `.smd` 网格和动画源文件 | **Source Tools**（`io_scene_valvesource`） | 项目编译管线需要的源格式 |
| 导出 `.vtf` 贴图 | **SourceIO** | 格式丰富、参数可控 |
| 查看地图布局 | **SourceIO** | BSP 导入含实体信息 |
| 还原材质着色器 | **SourceIO** | VMT 导入可还原 BVLG 节点 |
