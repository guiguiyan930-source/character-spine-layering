---
name: character-spine-layering
description: 按 Spine 骨骼动画需求拆解角色/游戏人物立绘。先由动作清单确定骨骼树，再由骨骼推导插槽、附件和切图，最终输出可直接导入 Spine 4.2 的骨骼 JSON、切图和带 Spine 标签的分层 PSD。部件与原图同坐标、同比例；眼睛、眉毛、鼻子、嘴巴、手、武器、衣服都是语义完整的单个物体，不是抠图或裁切碎片；遮挡、出画、误删处自动补全；关节重叠量按动作旋转角计算，衣服先与身体分离补全再按骨骼合并，动作中不露底。适用场景：Spine 动画拆图、角色骨骼绑定素材、需要拆到哪一层、眨眼/口型/手型附件、按动作清单拆分、遮挡补全、衣服与身体分层或合并。
---

# character-spine-layering v7

**骨骼优先**：先决定角色怎么动，再决定怎么切。所有拆分、重叠、合并和附件都由 `rig_plan.json` 驱动。

---

## 一、核心原则

1. **动作决定拆分**：没有动作需要的拆分一律不做；动作需要的拆分一个不漏。
2. **四条拆分规则**（优先级从上到下）：

   | 需求 | 做法 |
   |---|---|
   | 部件之间会穿插，前后关系会变 | 拆成前后两个插槽（`_back` / `_front`） |
   | 关节旋转 > 30° 且有遮挡 | 拆层，每段一根骨骼 |
   | 需要换形状（眨眼、口型、手型） | 同一插槽放多个附件 |
   | 软变形、呼吸、飘动、小幅弯曲 | 不拆，整图标记为 Mesh 候选 |

3. **原图坐标锁定**：所有切图与原图同尺寸、同坐标，不缩放、不变形；五官和手的位置、尺寸以原图实测为准。
4. **完整物体**：左右眼（眼白 + 虹膜 + 瞳孔 + 高光 + 眼线 + 睫毛）、左右眉、鼻、嘴、手、武器、每件衣服都是完整的单个物体；禁止出现直线切口、半只眼、断眉、断指、缺角弓梢。
5. **先分离，后合并**：衣服与身体一律先分开分割、分别补全，最后按骨骼分段合并。合并版和分离版来自同一份完整部件。
6. **动起来不露底**：关节重叠量由最大旋转角计算；每根骨骼按动作的角度范围做露底测试。
7. **不编造**：补全只基于可见部分和结构先验；换图附件属于新绘内容，单独标注，交给人工复核。

---

## 二、总体逻辑

```
动作清单 ──► 骨骼树 ──► 插槽表（绘制顺序）──► 附件表 ──► 切图计划
   │            │             │                  │
   │            │             │                  └─ 每个附件 = 一张完整部件图
   │            │             └─ 每个插槽 = 一个 PSD 图层 / 图层组
   │            └─ 每根骨骼给出 pivot、父骨骼、旋转范围 → 决定关节重叠量
   └─ 可以直接读取《角色动作设计 Skill》输出的动作卡片
```

语义分割、遮挡补全和衣服分离都只是**生产完整附件图**的手段，交付物以 Spine 需要什么为准。

---

## 三、骨骼树与级别

### 3.1 standard 骨骼树（默认，持弓角色示例）

```
root
└─ hip ─────────────────────────── pivot: 骨盆中心
   ├─ pelvis                       腰胯 + 下装上半
   │  ├─ thigh_l ─ calf_l ─ foot_l
   │  ├─ thigh_r ─ calf_r ─ foot_r
   │  └─ skirt_front_1..2 / skirt_back_1..2   （物理链）
   └─ spine
      └─ chest                     胸 + 肩 + 上衣上半（Mesh 呼吸）
         ├─ neck ─ head
         │         ├─ face  （face_base / eye_l / eye_r / eyebrow_l / eyebrow_r / mouth）
         │         ├─ front_hair_1..3 （物理链）
         │         ├─ side_hair_l / side_hair_r （物理链）
         │         └─ back_hair_1..3  （物理链）
         ├─ upper_arm_l ─ forearm_l ─ hand_l
         ├─ upper_arm_r ─ forearm_r ─ hand_r
         ├─ quiver
         └─ cape_1..3（物理链，可选）
bow（父：握弓手） ─ bow_tip_upper / bow_tip_lower（Mesh 弯曲权重）
bow_string（父：bow，中点约束到拉弦手）
arrow（父：拉弦手，射出后切换到 root）
```

### 3.2 三个级别

| 级别 | 骨骼数 | 插槽数 | 附件数 | 适合 | 与 standard 的差异 |
|---|---|---|---|---|---|
| `lite` | 12–18 | 20–25 | 2–4 | 卡牌立绘、小尺寸 Q 版、待机 | 躯干不拆胸胯；手臂、腿各一段；头发前后各 1 片；只有眨眼附件 |
| **`standard`** | 30–45 | 40–50 | 10–15 | 横版 / 俯视战斗角色 | 以 3.1 为准 |
| `fine` | 50–80 | 60–100 | 20+ | 大尺寸立绘、剧情特写 | 衣服与身体分开（换装）；袖口、鞋独立；手指分为拇指 / 四指两组；头发 4–5 束；裙摆 4 片；鼻子独立；5 个口型；远侧肢体另出一套（用于转身） |

### 3.3 动作 → 必需拆分

| 动作 | 必需骨骼 / 插槽 / 附件 | 关键旋转范围 |
|---|---|---|
| `idle` | chest Mesh、头发物理链 | 胸 ±3°，头 ±5° |
| `blink` | eye 附件 `open/half/closed` | — |
| `talk` | mouth 附件 3–5 个 | — |
| `walk` | 大腿 / 小腿 / 脚、上臂 / 前臂 | 髋 ±30°，膝 0–60°，肩 ±25° |
| `run` | 同 walk，加 hip 骨骼 | 髋 ±50°，膝 0–100°，肩 ±45°，肘 0–90° |
| `slash` / `thrust` | 胸胯分开、武器插槽、握持手前后层 | 腰 ±20°，肩 -90°～+60°，肘 0–120° |
| `shoot_bow` | 弓 Mesh、`bow_string` 独立、`arrow` 独立、拉弦手附件 `draw/release` | 肩 -90°～0°，肘 0–150° |
| `hit` / `down` | 头、颈、胸、胯分开 | 腰 ±30°，颈 ±25° |
| `turn` | 远侧肢体额外插槽（仅 fine） | — |

指定 `--actions` 后，系统在所选级别的基础上**自动补齐**缺失的骨骼、插槽和附件，并把每根骨骼的旋转范围取为所有动作的并集。

### 3.4 不拆清单

逐根手指（用手型附件代替）、眼白 / 虹膜 / 高光分层（Spine 用眨眼附件代替）、衣服褶皱（用 Mesh 代替）、鼻子单独骨骼（fine 做转头视差时除外）、不换装时上衣与躯干分开。

---

## 四、插槽绘制顺序（自底向上，standard）

```
back_hair_1..3
cape_1..3 / cape_inner
quiver
arm_far: upper_arm_far / forearm_far / hand_far      ← 远侧手臂（侧身时由 Step 4 判定）
leg: thigh_l / calf_l / foot_l / thigh_r / calf_r / foot_r
skirt_back_1..2
pelvis
chest
skirt_front_1..2
neck
upper_arm_near / forearm_near
hand_x_back                                          ← 握持手掌心
bow
bow_string
arrow
hand_x_front / hand_near                             ← 握持手环绕指段 / 非握持手
face_base
mouth
eye_l / eye_r
eyebrow_l / eyebrow_r
side_hair_l / side_hair_r
front_hair_1..3                                       ← 最顶层
shadow_* 紧贴被投影插槽之上，blend: multiply
```

- 这个顺序是初始值，Step 4 根据原图推断调整：透发眉移到前发之上、塞衣时 pelvis 在 chest 之上、远侧手臂放到躯干之下。
- 动作中需要临时改变前后关系（例如挥砍时手臂从身后甩到身前）的，在 `rig_plan.json` 中标记 `draw_order_keys`，由动画师在 Spine 中做 Draw Order 关键帧。

---

## 五、衣服与身体（`garment_mode`）

| 模式 | 行为 | 默认用于 |
|---|---|---|
| `merged` | 衣服按骨骼分段并入身体插槽（`chest` 含上衣上半，`upper_arm` 含袖子，`pelvis` 含下装上半） | lite、standard |
| `separate` | 身体与每件衣服（包括袖子、裤腿、鞋）各自成为插槽，同一骨骼下衣服插槽在身体插槽之上 | fine、换装 |
| `auto` | 有换装需求或设置了 `--skins` 时用 `separate`，否则按级别默认 | 默认值 |

**固定处理流程**（与模式无关）：

1. Step 3 分割时，衣服和身体始终分开。
2. Step 6 分别补全：
   - **身体**：被衣服遮住的区域补成**素体**（皮肤色或内衬色，保留大致体型轮廓），换装后不露洞，也不残留原衣服
   - **衣服**：领口、袖口、下摆、腰头等结构闭合，花纹延续
3. Step 7 按骨骼切分后：`merged` 模式在每个骨骼分段内合并“素体 + 衣服”；`separate` 模式保留独立插槽。
4. 两种结果可以同时导出：`--export-both` 会把分离版输出为 Spine 皮肤 `skin_nude_base`，供后续换装。

---

## 六、一键执行

```bash
python3 scripts/run_all.py <角色图> --out <输出目录> \
  [--level lite|standard|fine] \
  [--actions idle,blink,walk,run,shoot_bow,hit,talk | --action-cards cards.yaml] \
  [--garment-mode auto|merged|separate] [--skins default,nude_base] \
  [--style auto|chibi|anime|realistic] \
  [--no-attachments] [--export spine,psd,atlas] \
  [--config cfg.json]
```

### 产物

| 文件 | 用途 |
|---|---|
| `spine/skeleton.json` | Spine 4.2 骨骼 JSON：骨骼、插槽、皮肤、附件、物理约束初稿 |
| `spine/images/<slot>/<attachment>.png` | 按插槽 / 附件命名、裁到 bbox 的切图（带 2px padding） |
| `spine/character.atlas` + `.png` | 可选打包图集（`--export atlas`） |
| `character_spine.psd` | 带 Spine 标签的 PSD，可用官方 PhotoshopToSpine 脚本一键导入 |
| `rig_plan.json` | 骨骼树、旋转范围、插槽表、附件表、合并规则（整个管线的“单一事实来源”） |
| `parts_full/*.png` | 每个语义部件的完整图（衣服与身体分离，与原图同尺寸） |
| `parts_hidden_mask/*.png` | 补全区域掩码 |
| `attachments_sheet.png` | 新绘附件与原部件并排对比 |
| `reveal_test/*.gif` | 每根骨骼按动作角度旋转的露底测试 |
| `action_coverage.json` | 每个动作的骨骼、插槽、附件满足情况 |
| `report.json` | 置信度、验证结果、待复核清单（含全部新绘附件与推断补全） |

---

## 七、管线步骤

### Step 0 预处理 `00_prepare.py`

- 记录原图尺寸 `W×H` 和原图 alpha；所有坐标以原图坐标系为准。
- 风格识别（头身比）只用于合理性校验，不参与定位：

  | 风格 | 头身比 | 眼宽/脸宽 | 手长/头高 |
  |---|---|---|---|
  | `chibi` | 2–3 | 0.28–0.38 | 0.25–0.40 |
  | `anime` | 5–7 | 0.20–0.28 | 0.55–0.75 |
  | `realistic` | 7–8.5 | 0.14–0.20 | 0.70–0.85 |

- 短边 < 1500px 时内部 2× 超分，输出前精确缩回 `W×H`。
- 抠图：优先 mediakit-cli（`remove-image-background --scene general`），失败时回退颜色抠图。
- 清理标注：Zhang-Suen 细化加骨架支线剪枝；脸框内不做剪枝。

### Step 1 骨骼规划 `01_plan.py` → `rig_plan.json`

1. 读取 `--level` 和 `--actions`（或动作卡片），合并得到每根骨骼的旋转范围。
2. 从骨骼模板生成骨骼树，按第三节补齐缺失骨骼。
3. 生成插槽表（第四节顺序）和附件表。
4. 按 `garment_mode` 生成合并规则。
5. 计算每个关节的**重叠量**（见 Step 6.2）。

```json
{
  "level": "standard",
  "garment_mode": "merged",
  "bones": {
    "forearm_l": {"parent": "upper_arm_l", "pivot_kp": "elbow_l",
                  "rot_range": [0, 150], "length_kp": ["elbow_l", "wrist_l"]}
  },
  "slots": [
    {"name": "forearm_l", "bone": "forearm_l",
     "sources": ["arm_l_skin:forearm", "sleeve_l:forearm"],
     "attachments": ["default"], "mesh_hint": false}
  ],
  "attachments": {
    "eye_l": [{"name": "open", "from": "source"},
              {"name": "half", "from": "generate"},
              {"name": "closed", "from": "generate"}]
  },
  "draw_order_keys": [{"action": "slash", "slot": "upper_arm_far", "move_above": "chest"}]
}
```

### Step 2 关键点与骨骼定位 `02_keypoints.py`

- 身体：DWPose / anime-pose 给出颈、肩、肘、腕、髋、膝、踝；胸椎和骨盆中心由肩、髋中点推算。
- 手：21 点；脸：眉 5 点 ×2、眼（内外眼角、上下眼线、虹膜中心和半径）×2、鼻尖、嘴角、上下唇中点、下颌线。
- 头发：发根线和发束方向场；武器：弓梢、握把、弦端点、箭头、箭羽、剑尖、柄头。
- **把关键点写入骨骼**：每根骨骼的 pivot = 对应关键点，长度和方向 = 到子关节的向量，写回 `rig_plan.json`。
- 置信度低的面部点用脸框内深色线条连通域回退；实测比例超出风格先验时只标注，不修改。

### Step 3 语义分割 `03_segment.py` → `parts_visible/`

按**物体**分割，不按骨骼分割（骨骼切分在 Step 7 进行）。顺序：五官 → 手 → 武器 → 衣服 → 身体 → 头发。先分出的部件从后面的部件中剔除。

| 部件 | 方法 | 归属规则 |
|---|---|---|
| `eye_l` / `eye_r` | 眼眶多边形外扩到睫毛尖端，作为 SAM2 提示 | 眼白、虹膜、瞳孔、高光、上下眼线、睫毛、双眼皮线都归眼睛 |
| `eyebrow_l` / `eyebrow_r` | 眉头、眉峰、眉尾细长框 | 与睫毛相连时沿最窄处分开 |
| `nose` / `mouth` | 鼻尖小框（脸宽 8–12%）/ 嘴角与唇中点轮廓 | 鼻线、鼻孔、高光归鼻子；口缝、唇、口腔、牙齿归嘴 |
| `face_base` | 脸轮廓内剩余皮肤、轮廓线、耳朵 | — |
| `hand_l` / `hand_r` | 腕点 + 指尖点 | 完整手掌和全部可见手指 |
| 武器 | 端点 + 几何拟合（弓弧、箭线、剑轴） | 从其他部件中剔除 |
| 衣服 | `topwear`、`sleeve_l/r`、`bottomwear`、`skirt`、`cape`、`shoes`、配饰 | **袖子、鞋独立分割**，不并入手臂和脚 |
| 身体 | `torso_skin`、`arm_x_skin`、`leg_x_skin`、`neck` | 只含皮肤和素体 |
| 头发 | `front_hair`、`side_hair_x`、`back_hair` | 发束在 Step 7 切分 |

- 主路径用 SAM2 加关键点提示；模型不可用或置信度 < 0.7 时，回退到 HSV 语义分类、基于关键点的分区和几何拟合。
- 每个部件记录 `source` 和 `confidence`。

### Step 4 前后关系推断 `04_order.py`

- 在相邻部件交界处判断：轮廓完整且带描线的一方在前，被截断的一方在后；建立有向图后做拓扑排序，结果回写插槽顺序。
- 重点判定：透发眉、刘海遮眼、塞衣、远侧手臂、握持关系（`hand_x_back < 武器 < hand_x_front`）、弦在拉弦手指之下、发束内外。
- 出现环时拆成 `_front` / `_back` 两个插槽；证据不足时沿用默认顺序。

### Step 5 去投影 `05_deshadow.py`

- 在上层轮廓外侧窄带内找明显偏暗、且与轮廓平行的像素，恢复为本部件固有色，投影输出为 `shadow_<投射者>` 插槽（正片叠底，挂在投射者的骨骼上）。
- 眼白上缘的睫毛阴影不剥离；不确定的暗部只标注。

### Step 6 完整度补全 `06_complete/` → `parts_full/`

#### 6.1 缺失分类

```
missing = 预期完整形状 − 可见区域
  occluded   被上层遮挡       → 补全后被盖住，合成不变
  truncated  超出画布         → 扩展画布补全，原图区域坐标不变
  damaged    抠图/去标注误删   → 与原图 alpha 比对后补
预期完整形状 = 结构先验（原图实测尺寸）∪ 关节重叠区（6.2）
```

生成方法按顺序尝试：L1 几何重建 → L2 对称 / 模板 → L3 条件重绘（以结构线稿作为 ControlNet 条件，Seedream 局部重绘，回退 `cv2.inpaint`）。生成后统一做色板量化、明暗阶数对齐和线宽匹配。

#### 6.2 关节重叠量（由旋转角计算）

子段在关节处必须伸入父段之下，重叠量随最大旋转角增加：

```
overlap = clamp( k × W_joint × sin(θ_max / 2) + base , min, max )
  W_joint  关节处截面宽度（实测）
  θ_max    rig_plan 中该骨骼的最大旋转角
  k = 1.2，base = 0.1 × W_joint
  min = 0.15 × W_joint，max = 0.6 × W_joint
```

| 关节 | 典型 θ_max | 约等于 |
|---|---|---|
| 肩（run / slash） | 90° | 0.95 W → 取上限 0.6 W |
| 肘（shoot_bow） | 150° | 取上限 0.6 W |
| 膝（walk） | 60° | 0.7 W → 0.6 W |
| 腰（idle） | 3° | 取下限 0.15 W |
| 颈（hit） | 25° | 0.36 W |

- 关节处还要补出**圆形关节盖**：父段下缘和子段上缘各补成以 pivot 为圆心、半径 W/2 的半圆，保证任意角度下都有像素覆盖。
- 发束根部伸入头皮 10–20%；服装分片之间互相伸入 10–15%。

#### 6.3 专项补全器

| 补全器 | 对象 | 要点 |
|---|---|---|
| `face.py` | 眼、眉、鼻、嘴、脸底 | 眼睛：眼眶拟合，或以脸中轴镜像另一只眼（只借形状，位置取本侧实测点）；眉：沿眉头—眉峰—眉尾做样条外推；脸底在五官位置补肤色、在前发下补额头；不改变可见五官的大小、间距和角度 |
| `hand.py` | 手 | 21 点骨架加握姿模板（`grip_bow` / `grip_sword` / `draw_string` / `fist` / `open` / `point`），模板按实测掌宽缩放；握持手拆为 back / front，共享 pivot |
| `weapon.py` | 弓、箭、箭囊、剑、杖 | 按子件补全；弓弦取出后，弓层中的弦位置补为透明；装饰沿轴按周期延续 |
| `garment.py` | 上衣、袖、下装、裙、披风、鞋 | 结构开口闭合、花纹按周期平铺、褶皱沿走向场延伸；袖口伸入手腕之下、裤腿伸入鞋口之下 |
| `body.py` | 素体 | 衣服下的皮肤补成素体（按关键点构建体型轮廓，填肤色或内衬色）；关节盖 |
| `hair.py` | 头发 | 按方向场延伸，发尖不凭空延长；每束发根补全 |

硬性规则：补全像素只写入本层；可见像素不变；补全区不画交界描线；`missing / 完整形状 > 0.6` 时不补，标注 `unreliable`；完全不可见时保留空层，标注 `fully_occluded`；不编造原图中没有的设计元素。

### Step 7 按骨骼切分与合并 `07_rig_cut.py`

把完整语义部件切成骨骼插槽图：

| 切分 | 依据 | 规则 |
|---|---|---|
| 上臂 / 前臂、大腿 / 小腿 | 肘、膝 pivot | 垂直于肢体轴线切开，两侧各保留 6.2 的重叠区和关节盖 |
| chest / pelvis | 腰部天然分界线（腰带、衣摆），没有时过胸椎—骨盆中点 | 同上 |
| 袖子、裤腿 | 跟随所在肢体段切分 | 袖子被肘切成上下两段，分别归入 upper_arm / forearm |
| 发束 | 发束间的深色分隔线或凹口 | 每束保留完整发尖 |
| 裙摆 / 披风 | 褶皱深谷 | 每片保留完整下摆 |

- 切分线与原图中的任何可见边界都不能重合；切口处必须落在重叠区内（被上层盖住），合成时看不到。
- 按 `garment_mode` 合并：`merged` 模式在同一骨骼分段内把“素体 + 衣服”合成一张附件图；`separate` 模式保留独立插槽。
- `lite` / `standard` 模式下 `nose` 并入 `face_base`。

### Step 8 换图附件 `08_attachments.py`

| 附件 | 生成方式 | 对齐基准 |
|---|---|---|
| 眼 `half` / `closed`（fine 加 `smile`） | 以原上眼线为基准向下闭合；`closed` 为一条沿眼眶弧度的睫毛线 | 内外眼角 |
| 嘴 `closed` / `open_s` / `open_l`（fine 加 `a i u e o`） | 以实测嘴角为端点张开，补口腔和牙齿 | 左右嘴角 |
| 手 `fist` / `open` / `point` / `grip` | 握姿模板按实测掌宽生成 | 腕点与袖口 |
| 拉弦手 `draw` / `release` | 勾弦与松弦 | 弦接触点 |

- 附件与原部件使用同一骨骼、同一 pivot，bbox 不超过原部件的 120%；生成后执行风格统一。
- 全部标注 `generated: true`，列入待复核清单；`--no-attachments` 可以跳过。

### Step 9 边缘处理 `09_edges.py`

- 交界处描线只保留在前层插槽；半透明边缘做颜色去污，去除白边和灰边。
- 去除 alpha < 8 的孤立像素；五官细线不做开运算。
- Mesh 候选附件（chest、头发、裙摆、披风、弓、弦）的边缘不能有 ≥ 3px 的硬直线切口。

### Step 10 导出 `10_export.py`

**Spine 骨骼 JSON（4.2）**
- `bones`：来自 `rig_plan`，坐标从原图换算为 Spine 坐标：`x = x_img − root_x`，`y = root_y − y_img`；子骨骼使用相对父骨骼的局部坐标和角度。
- `slots`：按推断后的绘制顺序；阴影插槽使用 `blend: multiply`。
- `skins.default`：每个插槽的 region 附件，偏移 = 切图 bbox 中心相对骨骼的局部坐标；换图附件放在同一插槽下；`--skins nude_base` 时额外输出分离版皮肤。
- `physics`：头发、裙摆、披风骨骼链的物理约束初稿。
- Mesh 候选只以 region 导出，并在 `rig_plan.json` 中标注 `mesh_hint`，由动画师在 Spine 中转换并刷权重（自动生成网格不可靠，不做）。

**带标签的 PSD**（兼容官方 PhotoshopToSpine 脚本；标签语法以所用脚本版本为准）
```
[bone:root]
  [bone:hip]
    [bone:chest]
      [slot:chest] chest
      [bone:head]
        [slot:eye_l] 组
          open
          half   （隐藏）
          closed （隐藏）
        [slot:front_hair_1] front_hair_1 [mesh]
[skin:nude_base] ...（仅 separate / export-both）
[ignore] _reference（原图参考层）
```
- 每个图层保留原图坐标（layer top / left），不要裁到 bbox 后放在 (0,0)。
- 同时输出 `spine/images/`，文件路径与附件名一一对应。

### Step 11 验证 `11_verify.py`

| 类别 | 检查项 | 合格标准 |
|---|---|---|
| 原图对齐 | 画布尺寸、逐层位置、五官关键点 | 与原图一致；位移 = 0；五官偏差 ≤ 1px |
| 合成一致 | 默认附件按绘制顺序合成 vs 原图 | IoU ≥ 0.95，ΔE 中位数 < 3 |
| 完整物体 | 五官独立、单连通、眼含眼白虹膜睫毛、眉头到眉尾连续、口缝到嘴角 | 全部满足 |
| | 手指数、握持包裹、武器子件、服装闭合 | 符合风格要求 |
| | 切口检测 | 与边界重合的直线段 ≤ 物体尺寸 8% |
| **骨骼露底** | 每根骨骼按 `rot_range` 两端角度旋转（含父子链联动） | 露底 ≤ 角色面积 0.3% |
| | 关节盖 | 旋转到 θ_max 时关节处无透明缝 |
| 衣服 | separate 模式隐藏衣服插槽后 | 素体无空洞、无原衣服残影 |
| 附件 | 与默认附件叠放时基准点偏差 | ≤ 2px |
| 动作覆盖 | `action_coverage.json` | 每个动作的必需项全部满足 |
| 导出 | skeleton.json 结构校验、附件名与切图对应、PSD 回读标签 | 全部通过 |

- 露底集中在某个关节：加大该关节的 k 值，重跑 Step 6 → 7 → 11。
- 动作覆盖不满足：按提示补充 `rig_plan` 后，从 Step 1 起重跑受影响的步骤。
- 其他问题放宽参数自动重跑一次；仍不合格的写入待复核清单。

---

## 八、人工修正

```bash
--fix <part> --points x1,y1,x2,y2        # 给部件加点提示后重新分割
--face-keypoints face_fix.json           # 覆盖五官关键点
--bone-override "elbow_l=610,1180"       # 修正骨骼 pivot（同时改变切分位置）
--rot-override "forearm_l=0,160"         # 修正旋转范围（重新计算重叠量）
--order-override "pelvis>chest"          # 调整插槽顺序
--grip hand_r=draw_string                # 指定握姿
--regen-attachment mouth:open_l          # 只重新生成某个附件
--hair-strands front=4,back=3
```

- 手动修改过的 `parts_full/`、`spine/images/` 文件（按哈希识别）重跑时不覆盖。
- 所有修正写入 `fixes.json`，可以复现。

---

## 九、配置（`--config`）

```json
{
  "level": "standard",
  "actions": ["idle", "blink", "walk", "run", "shoot_bow", "hit", "talk"],
  "garment_mode": "auto",
  "skins": ["default"],
  "style": "auto",
  "geometry": {"lock_source_size": true, "allow_rescale": false, "extend_canvas": true},
  "upscale": {"min_short_side": 1500, "restore_exact": true},
  "segmentation": {"backend": "sam2", "keypoints": "dwpose", "hand_keypoints": true,
                   "face_keypoints": "anime-face-landmark", "min_confidence": 0.7},
  "face": {"eyebrow_over_hair": "auto", "prior_usage": "validate_only",
           "mirror_shape_only": true, "merge_nose": "auto"},
  "rig": {
    "hair_strands": {"front": 3, "side": 2, "back": 3},
    "cloth_panels": {"skirt": ["front", "back"], "cape": 3},
    "overlap": {"k": 1.2, "base": 0.1, "min": 0.15, "max": 0.6},
    "joint_cap": true,
    "physics_chain_len": 3
  },
  "weapons": [
    {"name": "bow", "parts": ["limb_upper", "limb_lower", "grip", "tip_upper", "tip_lower"],
     "string_slot": true, "mesh_hint": true},
    {"name": "arrow", "parts": ["head", "shaft", "fletching", "nock"], "world_space_on_release": true},
    {"name": "quiver", "parts": ["body", "mouth", "strap", "arrows"]}
  ],
  "attachments": {
    "generate": true,
    "eye": ["open", "half", "closed"],
    "mouth": ["closed", "open_s", "open_l"],
    "hand": ["fist", "open", "grip"],
    "max_bbox_ratio": 1.2
  },
  "completion": {"order": ["geometric", "template", "inpaint"], "inpaint_backend": "seedream",
                 "controlnet": ["lineart", "pose"], "max_hidden_ratio": 0.6,
                 "harmonize": {"palette_size": 12, "match_line_width": true}},
  "edges": {"decontaminate": true, "line_owner": "front", "mesh_edge_max_cut_px": 3},
  "verify": {"iou_min": 0.95, "face_kp_tolerance_px": 1, "reveal_max_ratio": 0.003,
             "attachment_align_px": 2},
  "export": {"spine_version": "4.2", "formats": ["spine", "psd"], "image_padding": 2,
             "psd_tags": true}
}
```

参考文档：`references/rig-templates.md`（三级骨骼模板、动作 → 骨骼映射）、`references/completion-spec.md`（五官关键点、握姿模板、武器子件、服装模板）、`references/spine-export.md`（坐标换算、JSON 字段、PSD 标签）。

---

## 十、目录结构

```
doubao-character-spine-layering/
├── SKILL.md
├── scripts/
│   ├── run_all.py
│   ├── 00_prepare.py        # 原图基准、风格校验、超分、抠图、去标注
│   ├── 01_plan.py           # 动作 → 骨骼树 → 插槽 → 附件 → rig_plan.json
│   ├── 02_keypoints.py      # 关键点 → 骨骼 pivot
│   ├── 03_segment.py        # 语义分割（衣服与身体分离）
│   ├── 04_order.py          # 前后关系推断 → 插槽顺序
│   ├── 05_deshadow.py       # 投影 → 阴影插槽
│   ├── 06_complete/         # face / hand / weapon / garment / body / hair / core
│   ├── 07_rig_cut.py        # 按骨骼切分、关节盖、按 garment_mode 合并
│   ├── 08_attachments.py    # 眨眼 / 口型 / 手型
│   ├── 09_edges.py
│   ├── 10_export.py         # skeleton.json、images、atlas、带标签的 PSD
│   └── 11_verify.py         # 对齐、完整度、骨骼露底、附件、导出校验
├── references/
│   ├── rig-templates.md
│   ├── completion-spec.md
│   └── spine-export.md
└── tests/
    ├── samples/             # Q版/动漫/写实 × 正脸/3/4侧 × 持弓/持剑/双持 × 裙/披风/塞衣
    ├── baseline.json
    └── run_regression.py    # IoU、五官偏差、露底率、动作覆盖率、附件对齐；退化 > 2% 即失败
```

---

## 十一、已知限制（交付时说明）

- 补全和素体都是推断的，请用 `parts_hidden_mask/` 复核。
- 换图附件是新绘内容，必须人工复核；重要角色建议由原画师补画后替换。
- 不自动生成 Mesh 网格和权重；`skeleton.json` 是初稿，骨骼位置和物理参数需要动画师微调；物理约束需要 Spine 4.2 及以上。
- 大角度旋转（> 120°）时，关节盖只能保证不露底，形体不一定正确，建议为这类关节准备额外的弯曲附件。
- 转身所需的远侧肢体只能推断，质量有限，统一标注为 `unreliable`。
- 极简画风的鼻、嘴可能漏检（用 `--face-keypoints` 修正）；透发眉可能误判（用 `--eyebrow-over-hair` 指定）。
- 非人形角色需要自定义骨骼模板（`references/rig-templates.md`）。

---

## 十二、环境与已知坑

- pytoshop 1.2.1：`compression` 使用 `enums.Compression.raw`；`write` 传入 `open(path, "wb")`；alpha 通道 id 必须是 `enums.ChannelId.transparency`(-1)。
- psd-tools：合成时用 `composite(force=True)`；图层名带尾随 `\x00`，比对前要清洗；验证合成时跳过隐藏附件和 `[ignore]` 层。
- PSD 图层必须保留原图坐标；Spine 切图裁到 bbox 后，附件偏移按 bbox 中心计算。
- Spine 坐标 y 轴向上：`y_spine = root_y − y_img`；子骨骼使用局部坐标和相对角度，导出时逐级换算。
- 超分缩回必须恰好还原 `W×H`，所有坐标四舍五入到整数像素。