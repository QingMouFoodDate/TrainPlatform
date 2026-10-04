# TrainPlatform · 轻眸识刻集成调度训练平台

> 面向内部数据治理、标注质量审核、单机训练任务调度和模型登记发布的设计仓库

TrainPlatform 定位为模型生产与治理平台：管理样本、冻结数据集及标注、调度 `ModelTrain` 执行入口，登记评测与模型血缘，负责 `register / validate / release`。面向客户端的 API、会话、在线前向推理与日期决策归 `InferPlatform`，不属于本仓。

## 当前状态与 M0 边界

现有 8 仓（`.github`、`ProjectPRD`、`TrainPlatform`、`InferPlatform`、`ModelTrain`、`WebClient`、`MobileClient`、`WxClient`）保持不新增、不合并、不重命名，统一处于**设计阶段，M0 待实现**。本 README 描述职责和验收约束，不能证明已有平台、训练结果、模型包或发布能力；实际能力以代码、测试、实验产物和发布记录为准。

M0 可先用 `ModelTrain` 的 CLI 单机入口训练、评测和导出候选包，再手工审核、登记并留存治理记录；**完整 Train 管理平台不是 M0 前置条件**。容器、Kubernetes、全部研究实验、多端和端侧模型也不是 M0 前置条件。手工流程仍须满足冻结、防泄漏、血缘与发布门禁。M0 是 C0 单图切片，不等于完整 C0；多图与完整纠偏审核回流保留为后续 C0，MobileClient/WxClient 暂停实现但保留原仓。

## 一、职责、核心模块与实体

| 模块 | 设计职责 |
| :--- | :--- |
| 样本与资产 | 内部入库、质量门禁、来源与许可核验、文件登记和删除保护 |
| 数据集 | 版本成员、分组切分、防泄漏校验，冻结数据、标注和 split |
| 标注与审核 | 标注任务、修订历史、审核与返工；审核通过才可晋为训练候选 |
| 训练任务 | 单机排队、资源检查、调用 ModelTrain、状态与 attempt 跟踪、受控重试 |
| 评测登记 | 登记 ModelTrain 产生的 EvalRecord、配置、指标、环境和实验关联 |
| 模型治理 | ModelBundle 登记、不可变血缘、validate 与 release 审核留痕 |

不自行组网、实现训练算法或执行在线前向推理；算法执行归 `ModelTrain`，在线 Runtime 是 `InferPlatform` 内部模块。不提供客户端会话、Evidence、Decision 或日期规则实现，不把线上纠偏直接变为训练标签，不自动训练或自动激活模型。

以下保留内部领域实体；字段、枚举和序列化格式统一由上位 [DATA](https://github.com/QingMouFoodDate/ProjectPRD/blob/main/contracts/DATA.md)、[TRAINING](https://github.com/QingMouFoodDate/ProjectPRD/blob/main/contracts/TRAINING.md) 与 [MODEL](https://github.com/QingMouFoodDate/ProjectPRD/blob/main/contracts/MODEL.md) 定义，本仓不另建 DTO 规格。

| 实体 | 职责 |
| :--- | :--- |
| `SampleAsset` | 受控样本及 source、许可、哈希、质量和标注引用，不等同于在线 Image |
| `DatasetVersion` | 冻结后不可变的数据集快照，保留 ParentDataset 血缘 |
| `DatasetVersionItem` | 版本成员关系，关联样本、split 和 group_key |
| `LabelTask` | 指定样本范围与标注类型的标注作业 |
| `Annotation` | 标注结果，保留标注人、时间和修订历史 |
| `LabelReview` | 训练质量审核记录与返工原因，不等于线上业务纠偏确认 |
| `TrainJob` | 单机执行任务、配置引用、状态、attempt 与产出引用 |
| `EvalRecord` | 评测数据版本、模型、指标、配置和环境记录 |
| `ModelBundle` | 候选交付包、评测引用及 ParentBundle 血缘，登记不等于发布或 active |
| `ManagedFile` | 受控文件唯一登记与引用保护，不暴露物理路径 |

## 二、数据、标注与冻结约束

- **来源与许可**：每条样本保留 source、获取方式、许可/授权与可训练用途；许可不明、未获授权或质量不合格的样本拒绝准入。许可证明与敏感原图仅留于受控环境。
- **标注审核**：按上位 DATA 与数据质量规范创建 LabelTask，执行标注、审核、打回返工。Annotation 留存人、时间和修订号，修订不覆盖历史；未经 LabelReview 通过的标注不得用于训练。
- **纠偏回流**：Infer 收集原始结果、Correction 和待审核记录；Train 另行审核来源许可、图像质量、标注正确性及分组。通过训练质量审核才晋为训练候选，再显式纳入新 DatasetVersion；不触发训练自动化。
- **冻结范围**：数据资产及清单、标注修订引用、split 分配共同冻结，只读消费。任何增补、清洗、标注修正或重新切分都产生新版本，不原地改写冻结快照。
- **ParentDataset**：新版本保留父数据集引用（如 parent_version）及变更原因；父子关系不能替代每版独立快照与完整校验。
- **防泄漏**：以 group_key 整组分配 train / val / test。同一商品、包装批次、拍摄会话、连拍与近重复簇不得跨 split；有关联的分组须合并处理，不能仅靠随机按图片切分。分组完整性优先于比例精确，测试集不参与调参。
- **冻结门禁**：来源许可、图像质量、审核标注覆盖、清单与哈希一致性、训练/验证集非空、group 跨 split 冲突均须检查；任一不通过拒绝冻结并记录原因。

ManagedFile 按文件内容 SHA-256 与大小校验复用，同一物理文件只登记一次，其他实体引用它；复用不能绕过来源许可或访问控制。被冻结版本、训练任务或模型血缘引用的文件不得直接删除，清理前须校验全部引用。

## 三、TrainJob 单机调度与状态

平台规划只负责组装冻结版本、训练配置、代码版本及受控执行环境，调用 ModelTrain 的任务级入口，不依赖其内部网络结构，也不要求算法入口反向调用平台内部 API。超出单机并发容量的任务排队；启动前检查 GPU/内存/磁盘，不满足资源或冻结门禁时不启动 worker。

任务状态限定为 `PENDING`、`RUNNING`、`SUCCEEDED`、`FAILED`、`RETRYING`、`CANCELLED`，迁移须白名单化。以下为待实现的迁移约束，具体机器契约归 TRAINING：

| 当前状态 | 允许的后继状态及条件 |
| :--- | :--- |
| `PENDING` | `RUNNING`；启动前校验失败到 `FAILED`；取消到 `CANCELLED` |
| `RUNNING` | 产出成功到 `SUCCEEDED`；执行失败到 `FAILED`；确认 worker 停止后到 `CANCELLED` |
| `FAILED` | 仅符合重试策略、未超上限且旧 worker 已终止/隔离时到 `RETRYING`；最终失败不再自动迁移 |
| `RETRYING` | 新 attempt 启动到 `RUNNING`；准备失败到 `FAILED`；取消到 `CANCELLED` |
| `SUCCEEDED` / `CANCELLED` | 终态，不允许重复执行或通过回调回退状态 |

- 重试复用同一 TrainJob，仅递增 `attempt`；产出登记以 `(train_job_id, attempt)` 幂等，重复回调不得重复登记。
- watchdog 检测心跳/超时后，必须**先终止旧 worker，或完成资源与写入权限隔离，再允许重试**；不能只改状态就启动新 worker。无法确认终止/隔离时阻止重试并转人工处置。
- 旧 attempt 的迟到回调不得覆盖当前状态或登记为有效产出；隔离其输出并留审计记录。当前 attempt 的失败或未完整写入产物不能当成成功包登记。
- 超过重试上限保留最终 FAILED 与原因。续训仅可使用校验通过且与冻结数据、配置和父模型相容的 checkpoint；否则明确失败，不静默更换输入。

## 四、执行输入输出与异常

| 环节 | 输入 | 输出与失败处理 |
| :--- | :--- | :--- |
| 数据准入与冻结 | 内部受控资产、source/许可、标注、group 与 split | 冻结 DatasetVersion 或可追溯拒绝原因；禁止输出本地路径 |
| 任务执行 | TrainJob/attempt、冻结版本引用、配置与代码版本、受控输出位置 | 状态、实验/评测记录、候选包引用；超时、OOM、异常退出按状态和重试策略处理 |
| 评测与模型登记 | ModelTrain 产生的记录、候选包和血缘 | EvalRecord / ModelBundle 登记；不完整、哈希或版本不一致时拒绝有效登记 |
| validate / release | 已登记包、评测证据、兼容性与 smoke 检查、审核意见 | 校验及发布记录；门禁失败保持未发布，不通知 Infer 激活 |

文件位置由执行环境注入并仅用于可信 worker，管理接口与跨仓登记传递受控资产标识及版本引用，不向调用方暴露本地路径、数据库结构或部署细节。异常使用上位契约的错误码与关联标识，不在 README 自造另一套响应格式。

## 五、模型登记、血缘与发布边界

登记时固化数据版本、配置哈希、代码提交、运行环境、EvalRecord、文件清单/校验和及 ParentBundle（如 parent_bundle_id）；续训或微调必须可追溯其父包。上述事实与文件不可覆盖，审核和状态变化另行追加审计记录。

发布职责链为：

```text
ModelTrain 候选包与评测记录
  -> TrainPlatform register -> validate -> release
  -> InferPlatform activate / rollback
  -> InferPlatform 内部 Runtime 加载与执行
```

候选包统一遵循 [MODEL 的必需文件清单](https://github.com/QingMouFoodDate/ProjectPRD/blob/main/contracts/MODEL.md)，本仓不复制第二棵交付目录树。validate 应检查文件完整性与哈希、字典/标签及预后处理一致性、张量规格、独立评测和 smoke 结果；通过后再经授权审核 release。

`candidate / validated / released / deprecated` 等治理状态以 MODEL 为准，**released 不等于 Infer 的 active**。Train 不修改线上 active 指针；Infer 切换失败须保留原 active。授权且仍符合兼容性与安全要求的 N-1 旧版可回滚，不能因新版发布就删除旧包，也不能把已撤销授权的包当作回滚目标。

## 六、安全与观测

内部入口须限制数据、审核、执行及发布权限；审核、版本冻结、任务迁移、重试、产出登记、发布与删除均留审计记录。日志只保留必要的关联标识、版本、状态、耗时和脱敏错误，不记录原始图片、完整敏感转录、令牌或本地路径。资产和模型权重不提交到代码仓库；敏感存储位置由部署配置决定。

## 七、提议工程结构与候选技术

以下为 Python 路线的**提议目录**，不代表已有对应文件或实现：

```text
TrainPlatform/                  # 内部数据与训练治理仓库
├── README.md                   # 职责、状态与跨仓约束说明
├── requirements.txt            # Python 候选依赖清单
├── .env.example                # 无秘密的配置示例
└── app/                        # 平台实现的提议入口
    ├── api/                    # 仅内部管理接口与权限检查
    ├── samples/                # 来源许可、样本质量与准入
    ├── datasets/               # 版本、分组切分和冻结校验
    ├── labeling/               # 标注任务、修订与质量审核
    ├── jobs/                   # 单机调度、attempt 与 watchdog
    ├── evaluation/             # 评测记录登记与追溯
    ├── models/                 # 模型血缘、校验与发布治理
    ├── storage/                # 文件登记、受控引用与删除保护
    └── core/                   # 配置、异常与观测审计
```

候选路线为 Python FastAPI + SQLAlchemy + Pydantic，或 Go + Gin/GORM；元数据可选 SQLite 或 PostgreSQL。这里只保留候选，不承诺实现、容器化或集群化，语言选择不改变冻结、调度与交付契约。

## 八、上位文档与访问权限

- [需求](https://github.com/QingMouFoodDate/ProjectPRD/blob/main/REQUIREMENTS.md)、[架构](https://github.com/QingMouFoodDate/ProjectPRD/blob/main/ARCHITECTURE.md)、[推进路线](https://github.com/QingMouFoodDate/ProjectPRD/blob/main/ROADMAP.md)
- [API](https://github.com/QingMouFoodDate/ProjectPRD/blob/main/contracts/API.md)、[DATA](https://github.com/QingMouFoodDate/ProjectPRD/blob/main/contracts/DATA.md)、[DATE_RULES](https://github.com/QingMouFoodDate/ProjectPRD/blob/main/contracts/DATE_RULES.md)、[MODEL](https://github.com/QingMouFoodDate/ProjectPRD/blob/main/contracts/MODEL.md)、[TRAINING](https://github.com/QingMouFoodDate/ProjectPRD/blob/main/contracts/TRAINING.md)
- [研究唯一源](https://github.com/QingMouFoodDate/ProjectPRD/blob/main/RESEARCH.md)、[数据质量](https://github.com/QingMouFoodDate/ProjectPRD/blob/main/standards/DATA_QUALITY.md)、[工程规范](https://github.com/QingMouFoodDate/ProjectPRD/blob/main/standards/ENGINEERING.md)
- [ADR-001 八仓职责与在线离线边界](https://github.com/QingMouFoodDate/ProjectPRD/blob/main/decisions/ADR-001-REPOSITORY_BOUNDARIES.md)

日期规则与 DTO 仅引用上位契约，不在本仓重新定义。ADR-001 已记录批准的职责设计；具体契约仍待评审与实现，不能据此认定生产字段已冻结。DATE_RULES、TRAINING 与 MODEL 的规格和完整清单由上位文档统一维护；候选引用不代表已实现接口、训练产物或验收成果。

当前仓库为私有仓库；访问 README、上位文档及 [组织主页](https://github.com/QingMouFoodDate) 的内容取决于登录、组织/仓库授权与主页可见性设置。GitHub 绝对链接不保证公共可读；未获授权时链接可能不可见，不通过公开复制原图或敏感资产解决访问问题。
�。
