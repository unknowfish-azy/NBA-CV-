# 交接任务书 — 小贾巴里·史密斯 单类检测/分割数据集

> 生成时间：2026-10-08。本文件写给**下一阶段接手的队友（人类或 AI Agent）**。
> 读完本文即可独立完成：复核补标 → 数据集重建 → 训练。无需询问前序上下文。

## 1 · 这是什么数据

NBA 常规赛 2026-02-01 独行侠 @ 火箭，CCTV-5 转播 1080P，按 2fps 抽帧共 **10,420 帧**（`frame_000001.jpg` ~ `frame_010420.jpg`，1920×1080）。
标注目标：**火箭队 10 号 小贾巴里·史密斯（Jabari Smith Jr.）**，单类别实例分割。

目标外观铁证特征：白色火箭主场球衣、红字 10 号、瘦高（约 2.11m）、浅棕肤色、黑短发、**左臂白色护套（臂上有可见纹身）**、白鞋。

## 2 · 当前状态（已完成的工作，勿重做）

| 项 | 状态 |
|---|---|
| 逐帧判定 | ✅ 全部 10,420 帧：在画面/不在画面/拿不准，三态已定 |
| 高置信标注 | ✅ **869 帧**多边形（人工级关键帧锚点 + ≤10 帧短插值） |
| 拿不准转人工 | ⏳ **3,508 帧**，见 `待人工复核清单.md`（多为远景白衣无法读号，宁缺勿错） |
| 不在画面（负样本） | ✅ 9,043 帧空 `shapes`（替补席/特写他人/转场） |
| 质检 | ✅ 两轮（帧间跳变对比 + 逐帧身份放大复核 + 漏标抽检），171 帧错误已除名 |
| 训练实验 | AWS SageMaker 跑过一版（g5.2xlarge / yolo11s-seg / 80epochs），权重在 S3；本地训练可直接用本包 |

**已踩过的坑（标注/训练 agent 必读）**：
- 干扰项：独行侠深色 10 号（白头带 B. Williams）、Finney-Smith 背面"…SMITH"、独行侠 25 号（也有白护套）、火箭 20 号（白头带+白护套+蓝鞋）、15 号 Sheppard（净脸瘦高浅棕发，左臂无护套）、背面单数字 0 号、观众席 10 号黑衫球迷、7 号杜兰特（高瘦无护套）
- 模板跟踪在"站位/罚球/回防"时段会漂移到空地板（NCC 高分假阳性），已弃用跟踪框——**不要重新引入**
- 他左臂有纹身：勿把"有纹身"当排除依据

## 3 · 包内结构

```
jabari_dataset_handoff/
├── HANDOFF.md                    ← 本文件
├── dataset/                      ← 训练就绪数据集（ultralytics + COCO 双格式）
│   ├── images/{train,val}/       已标注帧原图（train/val 按镜头块划分，防相邻帧泄漏）
│   ├── labels/{train,val}/       YOLO-seg: "0 x1 y1 x2 y2 ..." 归一化多边形
│   ├── coco/annotations.json     COCO bbox+segmentation（含 attributes.difficult）
│   ├── dataset.yaml              ultralytics 配置（0 = jabari_smith）
│   ├── README.md / 数据集说明.md  自动生成的格式与统计说明
├── 待人工复核清单.md              ← 3,508 帧待补标/核对清单（按区间分组）
├── annotations_sample/           ← 若干原始 X-AnyLabeling JSON 样例（含空标注格式）
└── tools/                        ← 全部可复跑脚本（依赖：Python3 + PIL/numpy/cv2/pyyaml）
    ├── merge.py                  分块结果 → 全部 frame_*.json（幂等）
    ├── make_dataset.py           frame_*.json → dataset/ + COCO + 说明文档
    ├── track.py                  模板跟踪（已弃用其输出，仅存档）
    ├── crop.py / qc_render.py / qc2_render.py   质检渲染工具
    ├── TASK.md / plan.json / qc_demote.json     标注规范/镜头切分/质检除名清单
```

原始逐帧 JSON（含空标注）与全部 JPG 在帧目录：`E:\小贾巴里史密斯\2026-02-01_独行侠vs火箭_CCTV_1080P_frames\`

## 4 · 下一阶段任务（按顺序）

### 任务 1：人工补标 3,508 帧（唯一人工环节）
1. 用 X-AnyLabeling（v4 格式）打开帧目录 `E:\小贾巴里史密斯\2026-02-01_独行侠vs火箭_CCTV_1080P_frames`
2. 按 `待人工复核清单.md` 的区间逐段核对：
   - 他在画面 → 描全身多边形（8~16 点，头顶到脚）
   - 不在画面 → 保持空 `shapes`
   - 判定标准见 `tools/TASK.md`（含全部干扰项与外观特征）
3. 提示：这些帧 80% 是"远景白衣读不出号"，优先用左臂白护套+纹身+轨迹连贯性判断

### 任务 2：重建数据集
```bash
cd E:\小贾巴里史密斯\2026-02-01_独行侠vs火箭_CCTV_1080P_frames
python _work/merge.py          # 补标结果合并（幂等）
python _work/make_dataset.py   # 重新导出 dataset/（同时更新说明文档）
```

### 任务 3：训练
```bash
pip install ultralytics
yolo segment train data=dataset/dataset.yaml model=yolo11n-seg.pt imgsz=960 epochs=120 batch=16
```
- 类不平衡注意：正样本 : 负样本 ≈ 1 : 10（他大半场在替补席），建议过采样正样本帧或加 `fraction`/`foreground` 策略
- `difficult=true` 帧已在 COCO attributes 标记，建议复核后再入训

## 5 · 溯源
- 镜头切分：帧间直方图差异，阈值 0.012，257 个镜头（plan.json）
- 标注：33 个分块 × 逐关键帧视觉判定（每帧均读过原图，号码存疑时放大核验），断点续跑 + 增量保存
- 质检除名清单：`tools/qc_demote.json`（171 帧，含原因）
