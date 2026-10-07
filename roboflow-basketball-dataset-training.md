---
name: roboflow-basketball-dataset-training
description: 篮球/YOLOv11 训练管线:Roboflow 下载器在 E:\roboflow_datasets(Key 仍无效),NBA
  Hackathon AWS 环境已配置好待 SSO 授权后训练
metadata:
  node_type: memory
  type: project
  originSessionId: sess_fa846db9-bd79-4be7-8453-bdfeb6fa0490
---

2026-10-06:用户要从 universe.roboflow.com 下载篮球检测数据集(YOLOv11 格式)做模型训练。已在 E:\roboflow_datasets 建好管线:config.json(API Key 占位、搜索词 "basketball player detection"、format=yolov11、max_datasets=3)+ download_datasets.py(纯标准库,兜底数据集 roboflow-universe-projects/basketball-player-detection-3,3267 张图)。用户粘贴的 Roboflow API Key 被服务器拒绝(401),`api_key` 仍为空,等有效 Private API Key。API Key 是私密凭据,不要写入记忆。

2026-10-06(同日晚):用户转入 **NBA Hackathon AWS 环境**做云端训练,目标:在 AWS 上用 Roboflow 篮球数据训练、再对 NBA 视频推理。关键环境事实:
- AWS CLI:系统安装曾损坏(msiexec 1603),当时解压到 `E:\AWSCLI\extract\...`;后经 winget 正常装好 v2.37.9 到 `C:\Program Files\Amazon\AWSCLIV2`(两份并存,都能用)。详见 [[aws-nba-hackathon-setup]]
- `~/.aws/config` 已配好双 profile:`nba-sso`(SSO 入口 d-92674d9b56.awsapps.com/start,账号 405496568869,角色 NBAHackathon-Team41)→ 链式 `nba`(role_arn arn:aws:iam::766815611718:role/NBAHackathonParticipantRole);**必须用 --profile nba --region us-east-1**(其他 region 被封),每 12 小时需 `aws sso login --profile nba-sso`
- SSO 授权已于 2026-10-06 完成,`sts get-caller-identity --profile nba` 验证通过
- **场地标注已上传**:E:\篮球场地标注\2026-02-01_独行侠vs火箭_CCTV_1080P_frames(27681 文件/3.9GB,原始帧+LabelMe json+YOLO v4 数据集)已完整传到 s3://nba-court-annotations-766815611718
- Roboflow 数据仍未下载成功(API Key 无效);E 盘另有大量 NBA 比赛视频素材(E:\NBA_*)可做推理素材
