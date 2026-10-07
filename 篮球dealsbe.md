# Roboflow Universe 数据集下载区

用途:从 [Roboflow Universe](https://universe.roboflow.com/) 下载篮球检测相关数据集(YOLOv11 格式),供模型训练使用。

## 目录结构

| 文件/目录 | 说明 |
|---|---|
| `config.json` | 下载配置(API Key、搜索词、数量、格式) |
| `download_datasets.py` | 一键下载脚本(纯标准库,无需安装依赖) |
| `datasets/` | 下载后自动创建,每个数据集一个子目录 |
| `数据集清单.md` | 下载完成后自动生成(类别、划分、训练命令) |

## 使用步骤

1. 获取 Key:登录 [app.roboflow.com](https://app.roboflow.com) → 左下角 **Settings** → **Roboflow API Key** → 复制 **Private API Key**
   (免费账户即可下载所有公开数据集)
2. 把 Key 填入 `config.json` 的 `api_key` 字段
3. 运行:

```bash
python E:\roboflow_datasets\download_datasets.py
```

脚本会实时搜索 Universe、按图片数取前 3 个候选、逐个下载并解压到 `datasets\`,最后生成 `数据集清单.md`。

## config.json 字段说明

```jsonc
{
  "api_key": "你的 Private API Key",
  "search_query": "basketball player detection",  // 搜索主题
  "max_datasets": 3,          // 最多下载几个
  "format": "yolov11",        // 也可改 yolov8 / coco / voc
  "datasets": [],             // 非空则只下载这些(格式: 工作区/项目名),不再搜索
  "fallback_datasets": [      // 搜索无结果时的兜底
    "roboflow-universe-projects/basketball-player-detection-3"
  ]
}
```

## 已确认的候选数据集

- **basketball-player-detection-3**(工作区 `roboflow-universe-projects`)— 搜索页显示 3267 张图,Universe 篮球检测的旗舰数据集,含球员/球/篮筐等标注(以解压后 `data.yaml` 为准)。

## 训练(Ultralytics YOLOv11)

```bash
pip install ultralytics
yolo detect train data=E:\roboflow_datasets\datasets\basketball-player-detection-3\data.yaml model=yolo11n.pt epochs=100 imgsz=640
```

## 注意

- API Key 是私密凭据,不要把填好 Key 的 `config.json` 上传到公开仓库或发给别人;泄露可在 Roboflow 设置页随时吊销重建。
- 若提示 `This API key does not exist (or has been revoked)`,说明 Key 复制错了或已失效,重新复制 **Private API Key**(注意不是 Publishable Key,Publishable 的是 `rf_` 开头、只能用于推理)。
