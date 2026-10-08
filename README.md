# NBA Hackathon 数据集（Team 41）

- **打包日期**：2026-10-08
- **数据来源**：本地交接包 `E:\NBAHackathon-Team41-Handoff-20261007`（2026-10-07 盘点）+ 从 AWS 云端拉回的模型权重
- **状态**：AWS 账户已于 2026-10-08 关闭，本数据集为**本地已收集数据的最终整合**（云端仍有部分资产未能拉回，见文末"已知缺失"）

## 目录结构（共 68.00 GiB）

| 目录 | 内容 | 大小 |
|---|---|---|
| `models/` | 训练好的权重（YOLO）+ LoRA 适配器 + Qwen2.5-Omni-7B 基础模型（部分分片） | 11.95 GiB |
| `data/datasets/` | YOLO 格式数据集（court_v3 / pose_subcoco / ball_foot_player_v7 / rf_det_pk / rf_6sets + Roboflow zip） | ~11.75 GiB |
| `data/frames/` | 抽帧（0201 / mixed / zchhf） | ~3.71 GiB |
| `data/media/` | 赛事视频（cctv_full.mp4、houdal_yt_*、commentary/*） | ~41.86 GiB |
| `data/segments/` | 视频分段（g1_seg0–7、seg_smoke） | ~2.52 GiB |
| `metadata/` | 训练代码 + 场景弱标签 + manifests | 0.31 GiB |
| `docs/` | 数据交接手册 | — |
| `templates/` | 模板 | — |
| `scripts/` | 运维脚本（sync-7am.ps1 / wipe-9am.ps1） | — |

## 模型权重清单（models/）

训练产物（YOLO，可直接用 Ultralytics 加载）：
- `best.pt`、`court_best.pt`（+ `court_best_meta.json`）
- `ball_best.pt`、`ball_foot_v1/best.pt`、`court_v3/court_v3_best.pt`、`players_v25i_v1/best.pt`
- 基础权重：`yolo11n-pose.pt`、`yolo11n.pt`

LoRA 适配器（Qwen Omni 微调）：
- `omni_lora_v5/`（adapter_model.safetensors + adapter_config.json）
- `omni_lora_v7_commentary/final/`（adapter_model.safetensors + train_summary.json）

训练产出（output.tar.gz，含 best/last 权重与结果）：
- `pose_subcoco/nbaws-pose-n-121618/output/output.tar.gz`、`pose_subcoco/nbaws-pose-s-121618/output/output.tar.gz`
- `ball_foot_v1/nbaws-bfoot-v1-1007-100759/output/output.tar.gz`
- `players_v25i_v1/nbaws-p25i-v1-1007-100801/output/output.tar.gz`

大模型（Qwen2.5-Omni-7B，**部分**）：
- `qwen25_omni_7b/`：分片 `model-00001 / 00004 / 00005-of-00005.safetensors`（11.5 GiB）及 tokenizer/config 等。**缺分片 00002、00003**（见"已知缺失"）。

## 数据集详情（data/datasets/）

| 数据集 | 任务 | 类别/关键点 |
|---|---|---|
| `court_v3` | 球场关键点/分割 | `court`，33 个关键点 |
| `pose_subcoco` | 人体姿态估计 | `person`，17 点骨骼（kpt_shape [17,3]） |
| `ball_foot_player_v7` | 目标检测 | `ball` / `foot` / `player` |
| `rf_det_pk` | 球场/球员检测 | 待核实类别 |
| `rf_6sets` | 附加检测 | 待核实 |
| `*.zip` | Roboflow 原始包 | court detection v2、players v25i、player detect 654 等 |

## 训练情况（已确认，2026-10-08 早间）

- **RF-DETR**（`rfpk-rtrain-r1i`，`ml.g5.xlarge` / NVIDIA A10G）：**25/25 epochs 完成**，val `mAP50 ≈ 1.0`、`mAP50:95 ≈ 0.94`
- **YOLO11s-pose**（`nbaws-pose-s-r3b`，`ml.g6.xlarge` / NVIDIA L4）：**40/40 epochs 完成**，val `Pose mAP50 ≈ 0.856`、`Box mAP50 ≈ 0.935`
- 合计 **65 epochs**，均在 07:30 前完成

## 已知缺失（因 AWS 关闭未能拉回）

- Qwen2.5-Omni-7B 分片 `model-00002 / 00003-of-00005.safetensors`（约 9.2 GiB）——可从 HuggingFace 重下
- `amen_seg` 数据集（约 3 GiB）
- `jabari_train/dataset_v1.zip`（约 318 MB）
- 球场标注桶 `nba-court-annotations-766815611718`（约 25.8 GiB，含 69,887 帧原始抽帧）——可从本地 `cctv_full.mp4` 重新抽帧
- RF-DETR 最终训练产出 `_train_out/`

## 使用说明

- YOLO 数据集为 Ultralytics 标准格式（`data.yaml` + `images/` + `labels/`），训练脚本见 `metadata/code-pose/train_pose.py`、`metadata/code-court/train_court.py`。
- 场景弱标签：`metadata/scene-labels*/`、`metadata/scene-labels-b1c/` 下的 `*.jsonl`（Qwen3-VL 生成，**未经人工审核**，不可直接当金标）。
- 详细盘点与标注规范见 `docs/NBA_数据交接手册.md`。
