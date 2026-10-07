# NBA Hackathon 数据交接手册

盘点日期：2026-10-07  
AWS 区域：`us-east-1`  
S3 数据桶：`nba-hackathon-766815611718-team41`  
本地副本目标：`E:\NBAHackathon-Team41-Handoff-20261007`

## 1. 当前结论

当前可访问的数据已经覆盖人体姿态、球场关键点、球/脚/球员检测、赛事视频、抽帧和 AI 生成的场景标签。它们可以支撑目标检测、姿态估计和初版镜头理解，但还不能直接训练“球员真实视线方向”或“视线锥区域”：现有姿态标签是 17 点人体骨骼，不含眼球/视线目标；场景标签是模型生成的弱标签，不是人工审核的最终金标。

已有两项 SageMaker 姿态训练分别为 `nbaws-pose-n-r2-172937`（`ml.g5.xlarge`）和 `nbaws-pose-s-r2-172939`（`ml.g6.xlarge`）。它们读取相同的 `pose_subcoco` 与 `code-pose` 输入，分别写入自己的 `_out` 前缀。两项均于 2026-10-07 18:33（北京时间）进入 `Stopped`，运行约 3,684 秒；`FailureReason` 为空，但最终指标列表为空，每项只上传了一个约 33 MB 的 `output.tar.gz`。因此目前没有可报告的最终精度，不能把它们视作完成训练。本次没有停止、覆盖或写入共享 S3 数据，也没有提交重复任务。

## 2. 数据资产清单

| 资产 | S3 来源 | 已确认内容与标注 | 用途和实现 | 来源/质量状态 |
|---|---|---|---|---|
| 人体姿态 `pose_subcoco` | `awsbatch/datasets/pose_subcoco/` | `data.yaml` 标为 `person`、`kpt_shape: [17, 3]`；有 images、labels、train、val，以及两个约 108 MB 的 NDJSON 文件 | `code-pose/train_pose.py` 使用 Ultralytics YOLO11 pose，模型尺寸 `n/s`，默认 40 epochs、640 输入尺寸；适合先识别身体关键点 | 名称显示为 COCO 子集，但需核实原始来源、授权和 17 点定义；不含注视目标标签 |
| 球场关键点 `court_v3` | `awsbatch/datasets/court_v3/` | `data.yaml` 标为 `court`、33 个关键点，并有 train/valid/test | 用于球场几何或关键点估计；具体 33 点与球场坐标的对应关系需要查标注说明 | 图片文件名含 NBA 比赛信息；原始采集/授权和点位字典需补齐 |
| 球/脚/球员检测 `ball_foot_player_v7` | `awsbatch/datasets/ball_foot_player_v7/` | 三类为 `ball`、`foot`、`player`；有 `images_manifest.json`、YOLO labels、train/val/test；训练脚本注释称 582 张，其中 test 为 36 张 | `train_entry.py` 使用 YOLO11n 检测，默认 60 epochs、960 输入、AdamW、seed 0；保留 best 权重和测试集指标 | 类别数量、逐类样本数和来源授权要用本地 manifest 再核验 |
| 球场/检测数据包 `rf_6sets` | `awsbatch/datasets/rf_6sets/` | 有 `rf_6sets.tar.gz` 和 `rf_data.tar.gz` | 可作为附加检测训练来源，解包后先核对类别映射与重复图片 | `rf` 名称暗示 Roboflow 来源，但实际 dataset 版本、license 和标签 schema 尚未核实 |
| 球场检测数据包 `rf_det_pk` | `awsbatch/datasets/rf_det_pk/` | 有 `basketball-court-detection-2.zip`；当前路径没有 `data.yaml` | 解包后再确认任务类型、类别和 train/valid/test 划分 | 来源、类别定义、授权和数据质量都待核实 |
| 场景帧与弱标签 | `awsbatch/code-b1/`、`awsbatch/b1b/results/` | `b1_manifest.json` 记录 1,349 帧：boundary 564、shot 100、feats 53、golden 632。B1 v1 弱标签数为 wide 896、closeup 162、follow 60、other 147、scoreboard 46、bench 23、replay 11、off-court 4。B1 v2 有 10,420 条弱标签：wide 7,440、closeup 1,860、follow 425、bench 477、other 101、replay 22、off-court 88、未规范化特写 7。字段有 `shot_type`、`court_zone`、`action`、`offense`、`players_hint`、`excitement`、`scoreboard` | `b1_label.py` 用 Qwen3-VL 生成七字段 JSONL；少量 hero 文案另由 Claude 复核 | AI 标签属于弱标签，需人工复核；标签分布高度偏向 wide/closeup，replay 极少，广告和导播包装/转场没有专门类别 |
| 场景原始视频 | `awsbatch/media/` | 盘点为 52 个对象、约 41.86 GB；包括 `cctv_full.mp4`、`houdal_yt_*.mp4` 和 `commentary/*.mp4` | 作为抽帧、镜头分类和后续时序分析的源素材 | 需逐个核实视频来源、比赛、时间戳、可用于赛事展示/训练的授权；大文件本地同步可能较慢 |
| 抽帧 | `awsbatch/frames/` | 盘点到 10,724 个对象，约 3.71 GB；目录含 `0201`、`mixed`、`zchhf`、`commentary` 等组 | 可用于筛选人工标注候选帧、目标检测和镜头样本 | 命名和目录来源需要映射到比赛/视频 ID；禁止随机拆散相邻视频帧 |
| 分段视频 | `awsbatch/segs/` | 9 个对象，约 2.52 GB；含 `merged`、`merged_segs` | 用于视频段级评估、动作/转场时序复核 | 需要建立片段到源视频的对应清单 |

本地副本只包含源数据、标注、训练配置和相关入口脚本；同步排除了 `_out`、`_train_out`、`_eval_out`、`_fetch_out` 等运行产物目录。源 S3 桶未被修改。

## 3. 镜头分类人工标注规范

首轮建议抽取 600 帧作为人工金标，优先覆盖不同比赛、视频源、机位、时间段、球员距离、回放和转场。B1 v2 可以给 wide/closeup 候选，但它的 replay 只有 22 条弱标签，且没有 advertisement/broadcast_transition；这三类必须从完整比赛、回放段和广告/转场片段定向抽取，不能只靠现有帧池随机抽样。建议主标签如下：

| 标签 ID | 定义 |
|---|---|
| `wide_court` | 正常比赛全景，能看到较大范围球场和多名球员 |
| `player_closeup` | 单名球员或单人主体占据画面中心，面部/上半身明显 |
| `replay` | 回放画面，包括慢动作、重复播放或明确的回放标识 |
| `advertisement` | 广告/赞助内容成为主要画面，不是球场边常驻广告牌 |
| `broadcast_transition` | 导播包装、片头片尾、转场动画或切入/切出包装；普通机位硬切不算 |
| `other_uncertain` | 替补席、裁判、观众、记分牌、纯图文，或无法确定的画面 |

每帧只选一个主标签；可另加 `replay_overlay`、`scoreboard_visible`、`player_count`、`occluded` 等属性。记录 `source_video_id`、`timestamp_ms`、`frame_path`、`annotator`、`confidence`、`notes`。无法确定时必须标 `other_uncertain`，不要猜。

建议首轮约 100 帧/类；如果广告或转场稀少，应从更多完整比赛/广告段补素材，而不是复制相邻帧。至少 10%–20% 双人复标并仲裁分歧。AI 预测只能写入 `suggested_label`，人工确认后才写入 `gold_label`。600 帧是首轮可行性样本，不足以证明所有比赛、频道和机位都泛化良好；最终样本量应按每类召回率置信区间和新增视频表现扩充。

数据集拆分：`train` 用于更新模型参数；`valid` 用于选模型、调参和早停；`test` 只在方案定型后做一次最终评估。按比赛或完整视频分组后再拆分，例如 70/15/15，不能把同一片段相邻帧随机分进三个集合，否则会造成泄漏和虚高指标。

## 4. 视线锥与球员 Agent 所需数据

1. 人体：逐帧球员框、稳定 `track_id`、17 点骨骼、关键点可见度和遮挡状态。已有 `pose_subcoco` 可做通用初始化，需用 NBA 转播画面验证。
2. 头眼与注视：头部姿态、双眼/眼角可见性、注视方向向量或画面内注视目标、目标类别（球/队友/篮筐/对手/其他）、置信度和标注来源。仅凭 17 点骨骼不能得出真实眼球视线；广角远景里眼睛像素不足时应退化为“头部朝向估计”，并明确显示低置信度。
3. 球场坐标：篮筐、边线、罚球线、三分线等控制点；每种固定机位的相机标定/单应性矩阵；球员、球和视线锥最终都映射到同一平面坐标。
4. 动作与时序：球员/球轨迹、传球/运球/投篮/防守动作、时间戳、进攻回合和事件结果。训练/测试应按比赛拆分。
5. 球员资料：统一 `player_id`，赛季、位置、身高/体重/臂展/弹跳、官方评分和来源 URL、单位、抓取日期；投篮分区概率需记录样本数、赛季和定义，不能把小样本当确定能力。
6. Agent 状态：把球员看作受约束的决策策略，输入位置/速度/朝向、持球状态、队友/对手相对位置、战术角色和能力参数；输出可解释动作候选及概率。先在回放/仿真中评估，再讨论自主策略，不要把视觉估计包装成真实“意识”。
7. 高斯空间：先确认老师所说的模型是 3D Gaussian Splatting 场景重建还是其他控制模型。单个转播机位无法可靠恢复被遮挡的 3D 场景；需要多视角或已知相机参数。首个演示可先做 2D 球场投影和扇形区域，再升级 3D。

## 5. 云端训练纪律

- 两项既有 pose 任务已停止；结果仍分别保存在各自 `_out/nbaws-pose-*` 路径。先由负责人确认是否继续/重跑，再提交使用新 job name 和新输出前缀的任务。
- 每个新训练任务使用唯一 job name、唯一输出前缀、固定数据版本和代码版本；记录模型、参数、数据 manifest、随机种子、指标与费用。
- 每次训练设置明确 epoch/早停条件和验证指标。不要无限提交作业；按基线、一次改动、评估、保留/回退的实验循环推进。
- scene 分类首选宏平均 F1、每类召回率和混淆矩阵；姿态看关键点 AP/PCK；视线要按角度误差/目标命中率评估；最终还要按完整视频检查动画是否抖动、漂移。
- 先做数据校验和小规模 dry-run，再启动大模型/长训练。每个任务确认训练结束后及时确认资源已释放。

## 6. 当前项目与仓库情况

当前可访问的工作目录是 `D:\SmartOpsAgent`，分支 `main`，Git 状态有 516 项未提交变更；这是 SmartOps 运维 Agent 项目，不是 NBA 计算机视觉项目。本次没有修改该目录。要实施镜头分类、视线锥和球员 Agent，需要提供/确认 NBA GitHub 仓库的本地路径、分支和允许修改范围，避免把 NBA 功能写入无关仓库。

## 7. 接下来由团队补齐的事项

1. 核实所有视频/数据集的原始来源、授权、比赛 ID、赛季和可公开展示范围。
2. 建立数据清单：每项记录 S3 key、本地相对路径、文件大小、ETag/校验和、来源、标签 schema、split 和负责人。
3. 对 600 帧镜头分类候选做人工标注；新增广告与导播转场样本，并按比赛/视频拆分 train/valid/test。
4. 确认 `court_v3` 的 33 点定义，并检查 `code-court/train_court.py` 使用的是 `dataset_v4` 分割数据，不能直接假定它对应 `court_v3`。
5. 提供 NBA 播放视频上的球员追踪/骨骼/头眼/注视目标标注；若需要真实眼动，安排球员眼动设备采集或明确改成头部方向代理指标。
6. 提供球员官方数据表及来源、单位、赛季和球员 ID 映射；由动力学老师确定状态变量、空间坐标、运动约束和 Agent 决策指标。
7. 确认 NBA GitHub 仓库与验收画面：至少包含一个全景、一个单人特写、一次回放、一次广告/转场，显示置信度和球场投影。
8. 由项目负责人设定云端训练费用/时长上限和实验停止标准。

## 8. 给 Codex/GPT 的项目提示词

```text
我们参加 NBA Hackathon，目标是从 NBA 转播视频识别镜头类型、球员/篮球/球场，展示球员骨骼、头部方向和视线感知扇区，并探索结合球员数据的可解释多球员决策 Agent。

当前数据在 S3 的 nba-hackathon-766815611718-team41 桶，区域 us-east-1。已经盘点到 pose_subcoco（person，17 点人体骨骼）、court_v3（court，33 点关键点）、ball_foot_player_v7（ball/foot/player 三类）、rf_6sets、rf_det_pk、赛事媒体/抽帧/切片，以及 1,349 帧 Qwen 场景弱标注。两项 YOLO11 pose 任务正在运行。场景标签仍需人工审核，尚无广告/导播转场金标、真实眼动/注视目标标签和球员追踪 GT。

当前 Codex 工作目录 D:\SmartOpsAgent 是无关的 SmartOps 项目，main 分支有大量未提交改动；不要在该仓库改 NBA 功能。先检查我提供的 NBA 仓库，再给出基于现有代码和数据的 2-3 个方案。每次都指出更简单或更可靠的改进想法。首期只实现可演示的镜头分类 + 人体/球检测 + 2D 球场视线扇区，不宣称从骨骼直接得到真实眼动。给出需要的人工标注 schema、按比赛拆分的 train/valid/test、评估指标、训练预算和验收画面；实现前先展示方案供团队确认。
```

## 9. 本地副本核验补充（2026-10-07）

E 盘数据副本已完成。data/media/ 的 52 个对象经逐对象长度核验与 S3 完全一致：41,860,435,529 字节。当前本地统计：datasets 114,177 个文件、11,754,826,033 字节；frames 10,727 个文件、3,708,437,313 字节；media 52 个文件、41,860,435,529 字节；segments 9 个文件、2,522,936,384 字节。整个交接目录共 124,982 个文件、60,172,688,977 字节。metadata/local_file_manifest.csv 列出 data 下每个文件的相对路径、字节数和扩展名；metadata/local_data_summary.json 为分目录汇总。metadata/ 中的代码入口及弱标签也随交接包保存。

注意：本次恢复的运行环境无法读取 nba AWS profile，因此没有重新完成全 bucket dry-run；已确认的媒体对象精确匹配不应被扩大解释为其他前缀均完成逐对象校验。两项 SageMaker 任务已停止且没有最终指标；本次未发起新训练，也未改写共享数据。提交新云训练前需重新登录并核对 AWS profile，再按独立 job name/output prefix 运行。

