---
name: l4d2-to-gmod-v-arm-fix
description: 修复从 L4D2/GMod 来源移植到 CS:CZ 的 viewmodel 手臂时，大拇指翻转及位置偏差。Pose Mode 只旋转拇指骨骼，保持位置和 Hand 不动，游戏内目视微调 Finger0 的 XZ 轴。
---

# L4D2/GMod → CS:CZ Viewmodel 手臂拇指修复

## 问题诊断

从 L4D2/GMod 移植到 CS:CZ 的 c_arm 模型，游戏内拇指翻转、位置不对。

**根因**：CS:CZ 引擎用 `EF_BONEMERGE` 把武器动画合并到手臂。武器动画假设标准 rest pose（`models/v_model/c_arm/standard/c_arms_cstrike_v2.smd`）。L4D2 模型的拇指 rest pose 与标准差了约 173°，bonemerge 后拇指朝向反了。

**为什么不能简单匹配标准值**：L4D2 模型的 Hand 骨骼旋转（~79°）与标准 CS:CZ（~90°）差约 11°。这导致即使拇指局部旋转匹配标准，世界空间中拇指朝向也不同。但 Hand 不能改——改了四个手指全歪。

## 修复策略

| 能不能 | 为什么 |
|--------|--------|
| ❌ 改 Hand 旋转 | 四指跟着变，整体手部变形，不可接受 |
| ❌ 改骨骼位置 | 顶点蒙皮撕裂，mesh 破损 |
| ✅ 只旋转拇指骨骼 | 通过 Apply Pose as Rest Pose，骨骼和顶点同步变换，mesh 完整 |
| ✅ Finger0 X/Z 微调 | 补偿 Hand 旋转差异，游戏内目视确认 |

## 核心原则

- **不改 Hand**：会破坏其他四指
- **不改骨骼位置**：会撕裂 mesh
- **只旋转拇指骨骼**（共 6 根，左右各 3 根）：
  - `ValveBiped.Bip01_L_Finger0` / `ValveBiped.Bip01_R_Finger0`
  - `ValveBiped.Bip01_L_Finger01` / `ValveBiped.Bip01_R_Finger01`
  - `ValveBiped.Bip01_L_Finger02` / `ValveBiped.Bip01_R_Finger02`
- **标准值只是起点**，Finger0 的 X/Z 必须在游戏内目视微调

## 修复流程（完整可执行脚本）

以下是一份完整的 Blender Python 脚本，可直接在 MCP 中逐步执行。需要修改的路径用 `< >` 标记。

### 步骤 1：导入原始模型 + 旋转拇指

**目的**：在 Pose Mode 中把拇指骨骼旋转到接近标准值，然后 Apply Pose as Rest Pose 固化。mesh 顶点会自动跟随骨骼变换，不会撕裂。

**注意**：脚本中的目标旋转值（`target` 字典）先用标准值。后续步骤 4 中如果拇指位置不对，改 Finger0 的 X 和 Z 值后重新执行此步骤。

```python
import bpy, math, os, shutil
from mathutils import Euler, Matrix

# === 清场景 ===
for obj in list(bpy.data.objects): bpy.data.objects.remove(obj)
for m in list(bpy.data.meshes): bpy.data.meshes.remove(m)
for a in list(bpy.data.armatures): bpy.data.armatures.remove(a)

# === 导入原始模型 ===
bpy.ops.import_scene.smd(filepath=r"<你的arms.smd路径>")
for obj in list(bpy.data.objects):
    if 'smd_bone_vis' in obj.name.lower():
        bpy.data.objects.remove(obj)

arm = [o for o in bpy.data.objects if o.type == 'ARMATURE'][0]
mesh = [o for o in bpy.data.objects if o.type == 'MESH'][0]
print(f"Armature: {arm.name}, Mesh: {mesh.name}")

# === 读取当前 rest pose 局部矩阵 ===
bpy.ops.object.mode_set(mode='EDIT')

thumb_bones = [
    'ValveBiped.Bip01_L_Finger0',  'ValveBiped.Bip01_R_Finger0',
    'ValveBiped.Bip01_L_Finger01', 'ValveBiped.Bip01_R_Finger01',
    'ValveBiped.Bip01_L_Finger02', 'ValveBiped.Bip01_R_Finger02',
]

current = {}
for name in thumb_bones:
    bone = arm.data.edit_bones.get(name)
    if bone and bone.parent:
        p = arm.matrix_world @ bone.parent.matrix
        b = arm.matrix_world @ bone.matrix
        current[name] = (p.inverted() @ b).copy()
bpy.ops.object.mode_set(mode='OBJECT')

# === 目标旋转（标准值，Finger0 的 X/Z 后续游戏内微调） ===
target = {
    'ValveBiped.Bip01_L_Finger0':  Euler((-1.223, -0.679, -0.789), 'XYZ'),  # X=-70°,Y=-39°,Z=-45°
    'ValveBiped.Bip01_R_Finger0':  Euler(( 1.221,  0.674, -0.794), 'XYZ'),  # X= 70°,Y= 39°,Z=-45°
    'ValveBiped.Bip01_L_Finger01': Euler((0, 0, 0.229), 'XYZ'),              # Z= 13°
    'ValveBiped.Bip01_R_Finger01': Euler((0, 0, 0.229), 'XYZ'),
    'ValveBiped.Bip01_L_Finger02': Euler((0, 0, 0.363), 'XYZ'),              # Z= 21°
    'ValveBiped.Bip01_R_Finger02': Euler((0, 0, 0.363), 'XYZ'),
}

# === Pose Mode: 只旋转，保持原位置 ===
bpy.ops.object.mode_set(mode='POSE')
for pb in arm.pose.bones:
    pb.location = (0,0,0)
    pb.rotation_mode = 'XYZ'
    pb.rotation_euler = (0,0,0)
    pb.scale = (1,1,1)

for name, tgt_rot in target.items():
    pb = arm.pose.bones.get(name)
    if pb is None: continue
    cur_loc = current[name].decompose()[0]  # 保持原始位置
    tgt_mat = Matrix.Translation(cur_loc) @ tgt_rot.to_matrix().to_4x4()
    pose_mat = current[name].inverted() @ tgt_mat
    pose_loc, pose_rot, _ = pose_mat.decompose()
    pb.location = pose_loc
    pb.rotation_euler = pose_rot.to_euler('XYZ')

# === Apply Pose as Rest Pose ===
bpy.ops.pose.select_all(action='SELECT')
bpy.ops.pose.armature_apply(selected=False)
print("拇指旋转已应用。下一步导出。")
```

### 步骤 2：导出 arms.smd + 更新 idle.smd

**目的**：导出修正后的参考骨架，并让 idle.smd 的 `skeleton time 0` 与之一致。标准模型的 idle 就是 rest pose（无动画位移），所以直接复制骨架段即可。

**为什么不用 Source Tools 导 idle 动画**：Source Tools 动画导出有 bug，会产生错误的旋转值（Finger01/02 值会翻倍）。文本复制骨架段是最可靠的方式。

```python
out = r"<输出目录>"  # 如 E:\modelproject\models\v_model\c_arm\workbench\dom

bpy.ops.object.mode_set(mode='OBJECT')
scene = bpy.context.scene

# --- 导出 arms.smd ---
scene.vs.export_list.clear()
col = bpy.data.collections.new('export_tmp')
scene.collection.children.link(col)
for obj in [mesh, arm]:
    for c in list(obj.users_collection): c.objects.unlink(obj)
    col.objects.link(obj)

item = scene.vs.export_list.add()
item.collection = col
item.ob_type = 'COLLECTION'
arm.vs.subdir = ""  # 不导出动画
scene.vs.export_path = out + "\\"
scene.vs.export_format = 'SMD'
scene.vs.smd_format = 'SOURCE'
bpy.ops.export_scene.smd(export_scene=True)

# 覆盖原 arms.smd（先备份）
arms_path = os.path.join(out, "yzallsb.smd")
exported = os.path.join(out, "export_tmp.smd")
if not os.path.exists(arms_path + ".backup"):
    shutil.copy2(arms_path, arms_path + ".backup")
shutil.copy2(exported, arms_path)
os.remove(exported)

# --- 更新 idle.smd（直接复制骨架段） ---
with open(arms_path) as f:
    arms_lines = f.readlines()
skel_start = next(i for i,l in enumerate(arms_lines) if l.strip()=='skeleton')
skel_end   = next(i for i,l in enumerate(arms_lines[skel_start:], skel_start) if l.strip()=='end') + 1

idle_path = os.path.join(out, "v_arm_guerilla_anims", "Idle.smd")
if not os.path.exists(idle_path + ".backup"):
    shutil.copy2(idle_path, idle_path + ".backup")
with open(idle_path) as f:
    idle_lines = f.readlines()
il_sk_start = next(i for i,l in enumerate(idle_lines) if l.strip()=='skeleton')
il_sk_end   = next(i for i,l in enumerate(idle_lines[il_sk_start:], il_sk_start) if l.strip()=='end') + 1

new_idle = idle_lines[:il_sk_start] + arms_lines[skel_start:skel_end] + idle_lines[il_sk_end:]
with open(idle_path, 'w') as f:
    f.writelines(new_idle)

# --- 清理 ---
for c in list(bpy.data.collections):
    if c.name != 'Collection':
        bpy.data.collections.remove(c)

print("导出完成。arms.smd 和 idle.smd 已更新。")
```

**验证**：打开 yzallsb.smd 和 Idle.smd，确认 skeleton time 0 中 Finger0 行的旋转值一致：
```
# L_Finger0 应为: ... -1.222 -0.679 -0.789  （或你微调后的值）
# R_Finger0 应为: ...  1.221  0.674 -0.794
```

### 步骤 3：更新 QC + 清理

**目的**：QC 中的 `$definebone` 旋转值必须与 SMD 一致，否则 studiomdl 编译器会用旧值覆盖。同时删除标准模型不需要的 `reference` 序列。

编辑 `<模型>.qc` 文件：

- 修改 6 根拇指骨骼的 `$definebone` 行，**只改旋转值，位置不动**。QC 格式是 Crowbar ZYX（`rz ry rx` 度）：
```
  L_Finger0:  rz=-38.916  ry=-45.196  rx=-70.054
  L_Finger01: rz= 0.000   ry= 13.093  rx=  0.000
  L_Finger02: rz= 0.000   ry= 20.790  rx=  0.000
  R_Finger0:  rz= 38.596  ry=-45.468  rx= 69.982
  R_Finger01: rz= 0.000   ry= 13.095  rx=  0.000
  R_Finger02: rz= 0.000   ry= 20.790  rx=  0.000
```
  **注意**：如果步骤 4 中微调了 Finger0 的 X 或 Z，QC 中的 `rx` 或 `rz` 也要对应更新。SMD 的 `rx` 对应 QC 的 `rx`（最后一位）。

- 删除 `$sequence "reference" { ... }` 整个块
- 删除 `v_arm_xxx_anims/reference.smd` 文件

**为什么删 reference.smd**：标准 CS:CZ 模型（`v_arm_gign`）只有 idle 一个序列，没有 reference。这是 L4D2 来源的残留物，保留它可能导致引擎播放错误的起始帧。

### 步骤 4：游戏内目视微调 Finger0 X/Z

**目的**：标准值只是数学上正确，但因为 L4D2 模型 Hand 旋转不同，视觉上拇指位置可能偏移。需要在游戏内目视确认并微调。

编译模型进游戏。观察拇指是否：
- **翻转**（指甲朝内）→ Finger0 X 差太远，需要大幅调整
- **不贴枪**（离枪太远）→ 调 Finger0 X（弯曲程度）
- **扭转方向不对**（拇指指甲朝上/下而非朝外）→ 调 Finger0 Z

| 参数 | 控制效果 | 微调范围 | 调整方向（L 侧） |
|------|---------|---------|-----------------|
| Finger0 X | 拇指弯曲程度 | -80° ~ -40° | 更负 = 拇指向手心弯 |
| Finger0 Z | 拇指旋转方向 | -45° ~ -15° | 更接近 0 = 拇指向手心转 |

规则：
- 每次只调**一个轴**
- 每次调整 **5-10°**
- 调完后**回到步骤 1** 重新执行（改 `target` 字典中 Finger0 的 Euler 值）
- R 侧的值取 L 侧的相反数（L=-50° → R=50°）

**注意**：Finger01 和 Finger02 的标准值（13° 和 21°）通常不需要调。只调 Finger0。

## 实测案例

某 L4D2→CS:CZ 模型（guerilla 手臂）的完整修复过程：

1. 原始 Finger0：`L=(103°, -29°, -6°)` `R=(-103°, 29°, -6°)`
2. 先用标准值：`L=(-70°, -39°, -45°)` `R=(70°, 39°, -45°)`
3. 游戏内发现拇指位置不对 → 调 Z：-45° → -30° → -25° → **-20°**
4. 调 X：-70° → -80°（反了）→ **-50°**
5. 最终值：`L=(-50°, -39°, -20°)` `R=(50°, 39°, -20°)`

```python
# 最终 target 字典
target = {
    'ValveBiped.Bip01_L_Finger0':  Euler((math.radians(-50), math.radians(-39), math.radians(-20)), 'XYZ'),
    'ValveBiped.Bip01_R_Finger0':  Euler((math.radians( 50), math.radians( 39), math.radians(-20)), 'XYZ'),
    # Finger01/02 用标准值不变
}
```

## 常见陷阱

| 陷阱 | 表现 | 解决 |
|------|------|------|
| 改了 Hand 旋转 | 四指全歪，整个手型不对 | 回退，只改拇指 |
| 改了骨骼位置 | mesh 撕裂，顶点飞掉 | 回退，用 Pose 保持原位置 |
| 用 Source Tools 导 idle 动画 | Finger01/02 值翻倍，idle 变形 | 改用文本复制骨架段 |
| QC `$definebone` 没更新 | studiomdl 用旧值覆盖 SMD | 同步更新 QC |
| 忘了删 reference.smd | 引擎播放错误的起始帧 | 删文件 + 删 QC 中的 `$sequence "reference"` |
| Finger0 调反了方向 | 拇指更歪了 | 另一个方向试，每次 5-10° |
| SMD 和 QC 旋转格式混淆 | 值对但效果不对 | SMD: Euler XYZ 弧度, QC: Crowbar ZYX 度 |

## 关键参考

- 标准模型路径：`models/v_model/c_arm/standard/c_arms_cstrike_v2.smd` + `v_arm_gign.qc`
- SMD 旋转 = 局部空间 Euler **XYZ**（弧度）
- QC `$definebone` 旋转 = Crowbar **ZYX** 格式（度），`rx` 是最后一位，对应 SMD 的 `rx`
- 修正公式：`pose = current_rest^(-1) @ target_rest`
- 使用 `bpy.ops.pose.armature_apply()` — **Apply Pose as Rest Pose**，不是 Apply Visual Transform
