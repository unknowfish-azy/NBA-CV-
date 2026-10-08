# NBA CV 数据审计与 Agent 编排设计

日期：2026-10-08

状态：待用户审阅

## 目标

建立一个可复现的第一阶段工具链，盘点本地 E 盘和 GitHub 数据源，生成数据交接手册、标注质量报告、主 Agent/Subagent 编排 prompts，并在本地完成最小训练 smoke test。第一阶段不自动启动 AWS 长时间训练，不伪造人工标注，不把远程仓库在网络不可用时当成已同步数据。

## 数据范围

### 已发现的本地来源

- `E:/AWS-NBA-Workspace/NBA-AWS-CV-Agent`：现有项目，包含 `agent/`、`cv/`、`aws/`、`schemas/`、`annotation_spec.md`、`HANDOFF.md`、训练与评估脚本。
- `E:/NBA_Workspace`：NBA 工作目录，需由审计脚本统计文件类型和入口文档。
- `E:/NBAHackathon-Team41-Handoff-20261007`：交接资料目录，需纳入来源索引。
- `E:/2026-02-01_独行侠vs火箭_CCTV_1080P_frames` 及球员标注目录：本地帧与 JSON 标注候选来源。
- `E:/篮球标注`、`E:/篮球场地标注`、`E:/杜兰特人物标注`、`E:/阿门汤普森`、`E:/塔里伊森球员标记`、`E:/小贾巴里史密斯`、`E:/阿尔佩伦申京标注`：按目录登记，先做文件级和 schema 级审计。

### 外部来源

- `https://github.com/unknowfish-azy/NBA-CV-.git`：登记为外部候选来源。工具必须记录 clone 时间、commit SHA、许可证文件、数据文件清单和同步错误；远程不可达时输出 `UNSYNCED`，禁止静默使用旧缓存。

## 统一数据事实源

沿用现有 `annotation_spec.md` 的原则：Annotation JSON 是唯一事实源，YOLO 导出是投影视图。审计器至少识别以下字段：`video_id`、`frame_id`、`camera`、`class`、`bbox`、`keypoints`、`track_id`、`source`、`annotator`、`split`、`schema_version`。

类别审计包含：侧全景、单人特写、回放、广告、导播转场；检测类别包含现有规范中的 player、referee、ball、hoop。两套标签必须分栏记录，不能混成一个类别表。

## Agent 编排

### 主 Agent：NBA Project Orchestrator

职责：读取审计结果，生成任务 DAG，阻止未经门禁的云端动作，汇总报告。

输入：`dataset_manifest.json`、`annotation_audit.json`、`repo_sources.json`、本地 smoke test 结果。

输出：`handoff_manual.md`、`agent_runbook.md`、`cloud_preflight.md`、阶段状态 JSON。

门禁：数据来源完整、标注 schema 可解析、split 无泄漏、smoke test 通过后，才允许生成云端训练命令；生成命令不等于执行命令。

### Subagent A：数据审计

扫描文件而不读取私钥或上传数据；统计路径、大小、哈希、格式、视频时长/分辨率（可用工具存在时）、帧数和来源关系。

### Subagent B：标注与分割审计

解析 JSON、YOLO TXT、COCO 等格式；检查类别、坐标范围、空标签、重复帧、跨 split 视频泄漏；输出 500–700 张人工复核候选清单。人工确认字段必须保持 `human_pending`，不能由模型自动填充为人工结果。

### Subagent C：训练工程

读取现有依赖和训练入口；安装前生成锁定清单；运行小样本、少步数 smoke test；记录设备、显存、吞吐量、loss 是否有限和 checkpoint 是否可加载。

### Subagent D：球员与空间数据

建立球员统计资料的来源、时间、字段和许可证表；把身高、体重、臂展、弹跳、位置、投篮概率、战术特征作为可选特征输入，缺失时显式为空，不填猜测值。

### Subagent E：AWS 预检

使用 `--profile nba --region us-east-1` 做只读身份、EC2、EBS、S3、ECR、SageMaker 检查；输出 GPU/磁盘/网络评估。只生成上传和训练计划，不自动创建资源或启动长训。

### Subagent F：评估与呈现

生成数据分布、标注质量、smoke test、失败样例和可复现命令报告；报告中区分真实测量、推断和待人工确认。

## 执行顺序

1. 扫描本地目录和已有文档，生成来源清单。
2. 尝试同步 GitHub 仓库，记录 commit、许可证和失败原因。
3. 解析标注并检查 schema、类别、分割和泄漏。
4. 生成 500–700 张人工复核候选，不改变原始标注。
5. 检查依赖和训练入口，运行最小 smoke test。
6. 生成数据交接手册、Agent prompts、AWS 预检报告。
7. 用户审阅报告后，另行批准云端上传和训练。

## Prompt 规范

每个 Agent prompt 必须包含：目标、输入路径、输出文件、允许修改范围、禁止事项、停止条件、验收命令。模型基座可配置为 GLM5.3 或 GLM5.3-flash；工具不假设本机存在这两个模型，也不把模型名称写死在训练脚本中。

## 验收标准

- 来源清单覆盖本地已发现目录，并明确每个来源的用途、格式、许可证/授权状态和审计状态。
- GitHub 来源有 commit SHA；网络失败时有可见错误，不产生伪成功记录。
- JSON/YOLO/COCO 标注的有效率、空标签、越界框和重复样本均有数字。
- train、Valid、test 按视频或比赛段隔离，泄漏检查通过。
- 人工复核候选数量在 500–700 之间，并保留抽样种子和路径。
- smoke test 在本地完成一个可加载 checkpoint 的最小训练/验证循环。
- 报告包含设备、显存、吞吐量、预计 epoch 时间和误差限制。
- 任何 AWS 命令默认只读；创建、上传、训练、删除动作都必须单独列出并等待明确批准。

## 风险与边界

- 远程 GitHub 当前无法连接，外部来源不能宣称已合并。
- 训练视频和模型权重可能占用数十 GB，工具先统计再复制。
- 私钥、凭证、SSO 缓存、`.pem` 文件不进入清单内容、不上传、不打印。
- “明天 9 点全部清空”属于不可逆操作；第一阶段不创建删除计划。
