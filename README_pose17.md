# pose17 骨骼标注说明 — 小贾巴里·史密斯

兼容 NBAcv-py-tools 的 `nba.pose17.v1` 规范（COCO17 关键点体系）。

## 文件
| 文件 | 内容 |
|---|---|
| `jabari_pose17.json` | **主交付**。逐帧 17 关键点：`kpts`（原图 1920×1080 坐标）、`scores`（模型置信度 0~1）、`visibility`（COCO v 值）、`match_iou`（与人员标注框的匹配度）、`n_visible` |
| `labelme/frame_XXXXXX.skeleton.json` | 每帧一个 LabelMe JSON（单 shape 17 点，flags 里带 visibility），供 skeleton_labeler.parse_labelme 直接读取 |
| `pose17_normalized.json` | 由 `run_cv_tools.py skeleton` 规范化的输出（0 invalid） |
| `pose17_gated.json` | 时序门禁结果（`detect_temporal_jumps`，max_displacement=50） |

## 关键点顺序（COCO17）
nose, left_eye, right_eye, left_ear, right_ear, left_shoulder, right_shoulder, left_elbow, right_elbow, left_wrist, right_wrist, left_hip, right_hip, left_knee, right_knee, left_ankle, right_ankle

## 可见性 v 值映射（由模型分数映射，非人工逐点确认）
- score ≥ 0.6 → `2`（可见）
- 0.2 ≤ score < 0.6 → `1`（已标注但遮挡/不确定）
- score < 0.2 → `0`（未标注）

## 来源与准确性验证
- 模型：yolo11x-pose.pt（ultralytics 8.4.171，GPU 推理，imgsz=960，conf=0.15）
- 关联：姿态检测框与人员标注框（frame_XXXXXX.json 中的多边形）IoU ≥ 0.25 才收录
- 抽检：12 帧随机可视化 + 1 帧关节级放大 + 最低 IoU 6 帧专项核查，骨架均落在标注框主体（史密斯本人）身上
- 覆盖：586 帧 / 9,962 关键点（其余帧为特写/重度遮挡，无法给出可靠的全身 17 点，**宁缺勿错，未输出**）

## 门禁 review 说明
`pose17_gated.json` 中 416 帧带 review 标记：本数据集为 2fps 抽帧，篮球跑动、镜头摇摄和切镜会使相邻帧关键点平均位移天然超过 50px 固定阈值。review ≠ 错误，按工具链约定进入人工复核队列；切镜两侧（plan.json 的镜头边界）的 review 可直接忽略。
