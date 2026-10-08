# 篮球比赛帧标注数据集

## 概览

| 项目 | 值 |
|---|---|
| 来源视频 | 2026-02-01 独行侠 @ 火箭（CCTV 转播, 1080P） |
| 总帧数 | 10,420（帧号 000001 ~ 010420） |
| 分辨率 | 1920 x 1080 |
| 标注对象 | 篮球（单一目标，每帧最多 1 个标注） |
| 已标注帧 | 2,150（1,106 人工 + 1,044 视觉模型自动标注+逐帧人工审核） |
| 确认无球帧 | 8,270（球被遮挡/出画/特写镜头，有效负样本） |
| 帧图位置 | `../2026-02-01_独行侠vs火箭_CCTV_1080P_frames/frame_XXXXXX.jpg` |
| 标注 JSON | 同目录 `frame_XXXXXX.json`（LabelMe 4.1.0 兼容，权威数据） |

## 标注格式

LabelMe/XemiX 4.1.0 circle 格式，每个 JSON 结构：

```json
{
  "version": "4.1.0",
  "flags": {},
  "checked": false,
  "shapes": [
    {
      "label": "篮球",
      "score": null,
      "points": [[cx, cy], [cx + r, cy]],
      "group_id": null,
      "description": "",
      "difficult": false,
      "shape_type": "circle",
      "flags": {},
      "attributes": {},
      "kie_linking": []
    }
  ],
  "imagePath": "frame_XXXXXX.jpg",
  "imageData": null,
  "imageHeight": 1080,
  "imageWidth": 1920
}
```

- `points[0]` = 球心 `(cx, cy)`，像素坐标，原点在左上
- 半径 `r = |points[1] - points[0]|`（本数据集统一写作 `[cx+r, cy]`）
- 转检测框：`bbox = [cx-r, cy-r, cx+r, cy+r]`

## 数据质量

| 指标 | 人工标注 (manual) | 自动+视觉审核 (auto) |
|---|---|---|
| 帧数 | 1,106 | 1,044 |
| 半径中位数 | 70 px | 17 px |

- auto 产出流程：轨迹跟踪生成候选 → 视觉模型逐帧裁决（球在/不在/坐标修正）→ 全量覆盖
  9,314 个未标注帧 → 随机抽检 24/24 合格、跨审核代理冲突 0
- **已知噪声（重要）**：人工标注中约半数圆心偏离真实球心 0.3~1.0 个半径，个别完全错标
  （例：frame_000004 标在球员头部而非球上）。若人工部分用于训练，建议先做一轮校正
- 确认无球帧是有效的负样本，可用于难度样本挖掘或评估虚警

## 本目录文件

| 文件 | 内容 |
|---|---|
| `annotations.md` | 全部 2,150 条标注明细（帧号, 球心, 半径, 来源） |
| `frame_map.md` | 全帧状态图（M=人工标注 / A=自动标注 / N=确认无球，按闭区间压缩） |

## 使用建议（面向 agent）

1. 需要标注坐标：解析 `annotations.md` 中的 tsv 代码块；**权威数据始终是帧同目录的 JSON**
2. 需要负样本或有效帧范围：解析 `frame_map.md`
3. 帧号不连续属正常：保留原始抽帧编号，无 jpg 的帧号不存在（约 100 个空洞帧号）
4. source 字段：`manual` = 人工原始标注；`auto` = 本管线产出（视觉模型逐帧裁决）
