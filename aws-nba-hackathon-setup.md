---
name: aws-nba-hackathon-setup
description: NBA Hackathon AWS 环境：nba/nba-sso 两个 profile、S3 桶名、数据已上传、SSO 每12小时需重登
metadata:
  node_type: memory
  type: project
  originSessionId: sess_5bac5f34-c645-4ef3-a5b6-c8f6cf01314b
---

NBA Hackathon AWS 环境（2026-10-06 配置完成）：
- AWS CLI v2.37.9 已装（winget，路径 `C:\Program Files\Amazon\AWSCLIV2`；另有早前解压的副本在 `E:\AWSCLI\extract\...`，两份都能用。Git Bash 新开 shell 若找不到 aws，手动 `export PATH="$PATH:/c/Program Files/Amazon/AWSCLIV2"`）。
- `~/.aws/config` 有两个 profile：`nba-sso`（登录用，SSO start URL d-92674d9b56.awsapps.com）和 `nba`（chained role `NBAHackathonParticipantRole`，账号 766815611718）。日常命令一律 `--profile nba --region us-east-1`。
- `aws sso login --profile nba-sso` 每 12 小时要重新执行一次（会弹浏览器）。
- S3 桶：`s3://nba-court-annotations-766815611718`（us-east-1），已上传 `2026-02-01_独行侠vs火箭_CCTV_1080P_frames/` 全部 27681 个文件 / 3.9 GB（原始帧+LabelMe json + `_yolo/` 下的 YOLO v4 数据集与训练脚本）。
- YOLO 项目：ultralytics 分割，单类 `basketball_court`，数据集配置在 `_yolo/court_v4.yaml`（path 指向本地 E: 盘，AWS 上训练时需改）。
- 注意：`--max-concurrent-requests` 这类参数在 Git Bash 会被拼接成 `flag,value` 报错，改用 `aws configure set default.s3.xxx` 设置。
- 训练用 GPU 实例（EC2/SageMaker）尚未启动，待用户决定。相关：[[roboflow-basketball-dataset-training]]
