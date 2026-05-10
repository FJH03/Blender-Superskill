---
name: l4d2-to-gmod-v-arm-fix
description: 修复从 L4D2/GMod 来源移植到 CS:CZ 标准骨架的 viewmodel 手臂时，大拇指骨骼 rest pose 朝向错误导致游戏内拇指翻转的问题。使用此 skill 当从非标准来源导入手臂模型、拇指朝向异常、或需要对齐标准 v_arm_gign 骨架时。
---

# L4D2/GMod → CS:CZ Viewmodel 手臂拇指朝向修复

## 问题根因

从 L4D2 / GMod 等来源移植的手臂模型（`.smd`），其骨架 rest pose 中**大拇指骨骼 `Finger0` 的 X 轴旋转和 CS:CZ 标准骨架差了约 170°**。这导致游戏内播放动画时，拇指指甲朝向反方向（朝内而非朝外）。

表面现象：
- HLMV 里预览 `idle.smd` **看起来正常**（因为动画和骨架来自同一来源，自洽）
- 游戏内**拇指翻转**（因为动画或其他骨骼操作按标准骨架的本地坐标系计算）

本质原因：
- SMD 文件的 `skeleton` 段定义了每根骨骼在 `time 0` 的世界空间旋转值（Euler XYZ）
- Blender SMD 导入器会从世界空间数据反算正确的局部旋转，因此 Blender 视口里 rest pose 可能显示正确
- 但 **studiomdl 编译器直接读取 SMD 文本里的 raw 旋转值**作为 rest pose
- 如果 SMD 文件里的 raw 值和标准骨架不一致，编译出的 `.mdl` 就会有错误的骨骼朝向

## 对比标准骨架

CS:CZ 的标准 viewmodel 手臂在 `models/v_model/c_arm/standard/`：
- 参考骨架：`c_arms_cstrike_v2.smd`（43 骨骼，根为 `ValveBiped.Bip01_Spine4`）
- QC：`v_arm_gign.qc`
- 动画：`v_arm_gign_anims/idle.smd`

关键拇指骨骼标准 rest pose（SMD skeleton time 0 的旋转值，弧度）：

| 骨骼 | 标准旋转 (rx, ry, rz) rad |
|------|--------------------------|
| `ValveBiped.Bip01_L_Finger0` | `-1.223, -0.679, -0.789` |
| `ValveBiped.Bip01_L_Finger01` | `0, 0, 0.229` |
| `ValveBiped.Bip01_L_Finger02` | `0, 0, 0.363` |
| `ValveBiped.Bip01_R_Finger0` | `1.221, 0.674, -0.794` |
| `ValveBiped.Bip01_R_Finger01` | `0, 0, 0.229` |
| `ValveBiped.Bip01_R_Finger02` | `0, 0, 0.363` |

## 修复流程

### 第一步：Blender 导入 + 验证

1. 在 Blender 中清空场景
2. 导入待修复的 `arms.smd`（参考骨架 + 网格）
3. 进入 Edit Mode，检查 `Finger0` 骨骼相对 Hand 父骨骼的局部旋转是否匹配标准值

```python
# 检查局部旋转
import bpy, math
armature = bpy.data.objects['arms_skeleton']
bpy.ops.object.mode_set(mode='EDIT')
for name in ['ValveBiped.Bip01_L_Finger0', 'ValveBiped.Bip01_R_Finger0']:
    bone = armature.data.edit_bones[name]
    parent = armature.data.edit_bones[bone.parent.name]
    parent_mat = armature.matrix_world @ parent.matrix
    bone_mat = armature.matrix_world @ bone.matrix
    local = parent_mat.inverted() @ bone_mat
    e = local.decompose()[1].to_euler()
    print(f"{name}: ({math.degrees(e.x):.1f}, {math.degrees(e.y):.1f}, {math.degrees(e.z):.1f})")
```

目标值：L_Finger0 约 `(-70°, -39°, -45°)`，R_Finger0 约 `(70°, 39°, -45°)`

### 第二步：用 Source Tools 重新导出

Blender SMD 导入器通常已正确还原局部旋转，因此 Blender 里的 rest pose 往往是正确的。**直接用 Source Tools 重新导出即可修正 raw 值**。

导出 arms.smd（参考网格）：

```python
import bpy
scene = bpy.context.scene

# 1. 清空旧 export_list，创建导出 collection
scene.vs.export_list.clear()
export_col = bpy.data.collections.new('arms_export')
scene.collection.children.link(export_col)

# 2. 将 mesh 和 armature 移入 export collection
for obj_name in ['arms', 'arms_skeleton']:
    obj = bpy.data.objects[obj_name]
    for col in list(obj.users_collection):
        col.objects.unlink(obj)
    export_col.objects.link(obj)

# 3. 添加到 export_list
item = scene.vs.export_list.add()
item.name = "arms.smd"
item.collection = export_col
item.ob_type = 'COLLECTION'

# 4. 设置导出路径和格式
scene.vs.export_path = r"<输出目录>\\"
scene.vs.export_format = 'SMD'
scene.vs.smd_format = 'SOURCE'

# 5. 导出（必须传 export_scene=True）
bpy.ops.export_scene.smd(export_scene=True)
```

导出 idle.smd（动画）：

```python
# 先导入旧动画到骨架
bpy.ops.import_scene.smd(filepath=r"<旧idle.smd路径>")

# 设置 armature 的动画子目录
armature = bpy.data.objects['arms_skeleton']
armature.vs.subdir = "v_arm_gign_anims"

# 用相同的 export_list 导出（会同时生成参考 .smd 和动画 .smd）
bpy.ops.export_scene.smd(export_scene=True)
```

### 第三步：手工验证导出结果

导出后，检查新 SMD 文件中 `Finger0` 的旋转值：

```
# 在 skeleton time 0 段查找拇指骨骼行
# L_Finger0 应为: 位置值 位置值 位置值  -1.2227 -0.6792 -0.7888
# R_Finger0 应为: 位置值 位置值 位置值   1.2214  0.6736 -0.7936
```

### 第四步：覆盖并备份

```python
import shutil, os
dst = "<目标arms.smd>"
if not os.path.exists(dst + ".backup"):
    shutil.copy2(dst, dst + ".backup")
shutil.copy2(src, dst)
```

## 注意事项

1. **不要直接文本编辑 SMD**：只改骨骼旋转不改顶点蒙皮会撕裂网格。必须通过 Blender 重新导出。
2. **多出的骨骼不要急于删除**：L4D2/GMod 骨架可能包含 `thumbroot`、`zArmTwist`、`Sode*` 等额外骨骼。先保留，逐个验证是否需要。
3. **HLMV 正常 ≠ 游戏正常**：HLMV 只播放模型自带的动画，是自洽的。游戏里如果引用了共享动画库（`$includemodel`），就会暴露 rest pose 差异。
4. **同时更新 arms.smd 和 idle.smd**：参考骨架和动画必须一起重新导出，保证来源一致。
5. **QC 的 `$definebone` 值不需修改**：如果只改 SMD 不改骨骼层级，QC 的骨骼定义保持不变。
