# character-psd-layering

> 角色立绘 → **完整物体 + 前后遮挡 + 可骨架动画**的分层 PSD、Spine 4.2 骨骼与换图附件，一条命令跑通。

把任意角色/游戏人物立绘自动拆解为**美术可编辑的分层 PSD**、**Spine 可直接导入的骨骼初稿**和**闭眼/口型/手型换图附件**。核心主张：每个部件是语义完整的单个物体（完整眼睛、完整手、整张弓），不是抠图碎片；被遮挡处按真实遮挡关系自动补全图层；拆分深度由**动画动作需求**决定（Spine lite / standard / fine 三档）。

---

## ✨ 特性

- 🧩 **完整物体拆解**：眼睛=眼白+虹膜+眼线睫毛+眼妆整块；手=完整拳形（握持时拆掌心/指段两层，union 还原整手）；弓/箭/箭囊各自成体
- 🎭 **按动作拆分**：`--spine-actions idle,blink,walk,shoot_bow,...` 声明动画需求，自动反推需要拆到哪一层，输出 `action_coverage.json` 校验
- 🦴 **Spine 4.2 骨骼初稿**：`root→pelvis→spine→chest→neck→head` 主链 + 部件骨 + 发丝物理链 + `spine_images/` 裁切导出
- 🔄 **换图附件**：闭眼（half/closed/smile）、口型（open_s/open_l）、手型（fist/open/point）12+ 个状态附件，同 pivot 可直接替换
- 🛡️ **遮挡补全**：被挡住的图层按 occluded/truncated/damaged 分类补全，补全像素量化到可见色板（ΔE<5），不编造原图没有的设计
- ✅ **自动化验证**：合成 IoU≥0.95、露底=0、五官独立层、Spine 专项（骨骼闭环/动作覆盖/附件对齐≤2px/导入校验）
- 🧹 **脏图也能用**：带标注引线/棋盘格背景的示意图自动抠图 + 骨架剪枝清理

## 🖼️ 效果

| 原图（透明底输入） | 分层 PSD 合成预览 | 部件平铺 |
|---|---|---|
| ![原图](assets/example/character_ref.png) | ![PSD 合成](assets/example/psd_preview.png) | ![部件平铺](assets/example/parts_preview.png) |

| 五官独立层检查 | 换图附件总览 | 关节旋转露底验证 |
|---|---|---|
| ![五官检查](assets/example/face_check.png) | ![附件总览](assets/example/attachments_sheet.png) | ![露底验证](assets/example/reveal_preview.gif) |

> 示例产物（v6 spine standard，Q 版古风女弓箭手）完整文件见 `assets/example/`。

## 🚀 快速开始

```bash
# 1. 安装依赖
pip install -r requirements.txt

# 2. Spine standard 全流程（推荐：分层 PSD + Spine 骨骼 + 换图附件）
python3 scripts/run_all.py 角色立绘.png --out out/ \
    --granularity spine --spine-level standard \
    --spine-actions idle,blink,walk,shoot_bow,hit,talk \
    --export psd,rig,spine

# 3. 最简：只出分层 PSD
python3 scripts/run_all.py 角色立绘.png --out out/ --granularity basic

# 4. 人工修正
python3 scripts/run_all.py 角色立绘.png --out out/ \
    --fix hand_l --points 320,400,340,420 --grip hand_r=bow \
    --split-override "elbow_l=610,1180" --regen-attachment mouth:open_l
```

> 原图含标注引线/背景时管线自动抠图清理；透明底素材加 `--skip-clean --no-upscale` 可跳过。

## 🎛️ 三档拆分（Spine）

| 档位 | 规模 | 拆分内容 | 典型动作 |
|---|---|---|---|
| **lite** | 20–25 层 | 臂腿上下段 + 发束 2–3 束，胸胯合一 | idle / walk |
| **standard**（默认） | 35–50 层 | 胸胯拆分 + 发束 3 束 + 臂腿 + 握持前后 + 换图附件 | idle / blink / walk / run / slash / shoot_bow / hit / talk |
| **fine** | 60–100+ 层 | standard + 更多发束 + smile 附件 + 远侧肢体 | turn / 表情细节 |

**动作 → 拆分映射**（`--spine-actions` 声明后自动校验）：

| 动作 | 必需拆分 |
|---|---|
| idle | chest + 发束 |
| blink | 眼睛附件（half/closed） |
| walk / run | leg_upper+lower / arm_upper+fore |
| slash / thrust | chest+pelvis + 武器 + 握持前后 |
| shoot_bow | bow + arrow + 握持前后 |
| hit / down | head+neck+chest+pelvis |
| talk | 嘴附件（open_s/open_l） |
| turn | 远侧肢体（fine） |

## 📦 产物清单

| 产物 | 说明 |
|---|---|
| `character.psd` | 分组分层 PSD（后发→背挂→身体组→颈→头组；含隐藏「附件 attachments/」组） |
| `spine_skeleton.json` + `spine_images/` | Spine 4.2 骨骼（bones/slots/skins/physics）+ 每插槽裁切 PNG |
| `attachments_manifest.json` + `attachments_sheet.png` | 换图附件清单与总览（generated:true 待复核） |
| `action_coverage.json` | 动作覆盖校验（每动作 required/satisfied/missing） |
| `layer_plan.json` | 拆分计划（level/actions/mesh/physics/attachments/split_overrides） |
| `rig.json` | 部件 bbox/pivot/parent/bone/mesh/physics/attachments/actions |
| `face_layout.json` + `face_check.png` | 五官实测关键点 + 逐层对比 |
| `completion.json` + `completion_compare.png` | 缺失分类（occluded/truncated/damaged）+ 三联图 |
| `parts/*.png` 等 | 补全后部件 / 原始部件 / 补全区掩膜 / 阴影层 |
| `psd_preview.png` / `parts_preview.png` / `reveal_preview.gif` | 合成预览 / 部件平铺 / 关节露底动画 |

## 📁 目录结构

```
doubao-character-psd-layering/
├── SKILL.md                    # Skill 入口（七原则/遮挡顺序/三档/动作映射/管线/验证）
├── README.md                   # 本文件（GitHub 首页）
├── LICENSE / .gitignore / requirements.txt
├── scripts/
│   ├── 00_prepare.py           # 抠图→剪枝清理→超分并精确缩回→source_alpha
│   ├── 01_keypoints.py         # 风格识别 + 关键点 + face_layout（五官实测）
│   ├── 02_segment.py           # 语义分割→完整眼睛（嘴保护）/手/武器→五官拆分互斥→握持拆层
│   ├── split_spine.py          # Spine 分级拆层：胸胯/发束/弓弦/肘膝切分校正
│   ├── 03_order.py             # 遮挡顺序推断（透发眉/塞衣/握持硬约束）
│   ├── 04_deshadow.py          # 投影剥离（独立 multiply 层，睫毛阴影保护）
│   ├── 05_complete/            # 补全模块包（core/face/hand/weapon/garment/generic）
│   │   └── attachments.py      # 换图附件生成（闭眼/口型/手型 + manifest + sheet）
│   ├── 06_edges.py             # 描线归属 / 去污 / 微小残片清理
│   ├── 07_build.py             # 分组 PSD + 附件隐藏组 + rig.json + Spine 4.2 导出
│   ├── 08_verify.py            # IoU/露底/五官独立层/配色 + Spine 专项 + 预览
│   └── run_all.py              # 一键串联 + 人工修正闭环
├── references/
│   ├── segmentation-guide.md   # 分割规则与调参（颜色分类/分区几何/五官拆分）
│   ├── completion-spec.md      # 补全规格（缺失分类/方法/验证指标）
│   ├── spine-spec.md           # Spine 分级图层表/动作-拆分映射/导出字段
│   └── models.md               # 可选模型接入路径（SAM2/DWPose/Seedream）
├── assets/example/             # 示例角色完整产物（v6 spine standard）
└── tests/
    ├── samples/                # 回归样本
    ├── baseline.json           # v6 五变体基准
    └── run_regression.py       # 回归（退化 >2% 判失败）
```

## 🧪 回归

```bash
python3 tests/run_regression.py                    # 全部变体
python3 tests/run_regression.py --variants basic,spine_standard
```

v6 基线（Q 版古风女弓箭手，matte 透明底，`--skip-clean --no-upscale`）：

| 变体 | 部件数 | IoU | 附件 | 结果 |
|---|---|---|---|---|
| basic | 23 | 0.9590 | — | PASS |
| live2d | 38 | 0.9594 | — | PASS |
| spine_lite | 31 | 0.9593 | 0（--no-attachments） | PASS |
| spine_standard | 31 | 0.9603 | 12 | PASS |
| spine_fine | 35 | 0.9603 | 14 | PASS |

## 🛠️ 常见问题

- **弓弦没拆出来？** 素材弓弦与弓臂同色一体时按设计不拆（action_coverage 提示），可人工补弦或换带独立弦的素材。
- **换图附件画风不贴合？** 附件是几何新绘内容（标注 generated:true），用同名 PNG 覆盖 `parts/attachments/<slot>/<name>.png` 即可替换。
- **颜色/分区不匹配？** 用 `--config cfg.json` 覆盖规则与分区，参数语义见 `references/segmentation-guide.md`。
- **依赖装不上？** 必需 numpy/opencv/pillow/pytoshop/psd-tools；可选模型（rembg/Real-ESRGAN/SAM2/DWPose/Seedream）未装自动回退启发式。

## 📜 License

MIT，见 [LICENSE](LICENSE)。
