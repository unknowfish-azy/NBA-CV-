---
name: nba-aws-hackathon-training
description: NBA Hackathon AWS 环境(账号766815611718)的配置、SageMaker管线约定与court分割训练任务状态
metadata:
  node_type: memory
  type: project
  originSessionId: sess_fa846db9-bd79-4be7-8453-bdfeb6fa0490
---

2026-10-06:NBA Hackathon AWS 环境已配好并验证:`~/.aws/config` 有 nba-sso(签到,405496568869,NBAHackathon-Team41)和 nba(链式到 766815611718 的 NBAHackathonParticipantRole)两个 profile;所有命令必须带 `--profile nba --region us-east-1`。AWS CLI v2.37.9 位于 E:\AWSCLI\extract\Amazon\AWSCLIV2\aws.exe(系统级安装损坏,msiexec /a 提取的干净副本,已加用户 PATH)。

团队管线:SageMaker 训练任务约定为 pytorch-training:2.1.0-gpu 镜像 + File 模式,入口用环境变量 SAGEMAKER_PROGRAM=/opt/ml/input/data/code/<脚本>;代码在 s3://nba-hackathon-766815611718-team41/awsbatch/code-*/;g1_perceive.py 用 best.pt(单类球员模型)+ByteTrack 产出 tracks jsonl。GPU 配额:ml.g5.xlarge 与 ml.g6.xlarge 训练各 1 台,与队友任务(命名 nbaws-hou-*/g1-*/b1-*)争抢。

我方训练:dataset_v4(队友 19:28 传完,nba-court-annotations 桶,2521 train+629 val,单类 basketball_court YOLO-seg);我写的 E:\AWSCLI\inspect\train_court.py(yolo11n-seg 迁移,EPOCHS=100/IMGSZ=640/BATCH=16)已传 awsbatch/code-court/,启动器 E:\AWSCLI\inspect\run_training.sh 后台等槽位→发车 nbaws-court-v4-train-*→监控到结束,输出到 awsbatch/court-v4/。Roboflow 数据集([[roboflow-basketball-dataset-training]])仍因 Key 无效未下载。
