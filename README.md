# TrainPlatform · 轻眸识刻业务后端与训练平台

> **组织归属**：`QingMouFoodDate` GitHub 组织  
> **仓库性质**：轻眸识刻核心 Web 原型系统及多端前台的**轻量业务 API 后端与模型推理调度枢纽**  

---

## 一、 核心定位与职责边界

本仓库负责为轻眸识刻全系统（以 Web 原型系统为核心，兼顾后续移动端延伸）提供**稳定、标准、轻量的业务逻辑支撑与模型推理服务**。

### 1. 职责范围
- **多端统一 API 服务**：为 `WebClient`（网页前台）、`WxClient`（微信小程序）、`MobileClient`（采图 App）提供高可用 RESTful 接口；
- **核心推理转发与日期推断**：接收多端上传的包装图片，调度算法推理，并结合保质期时长推算剩余天数（计算安全/临期/过期状态）；
- **样本资产管理与防重拦截**：管理样本流转，在接收移动端上传时集成二进制 SHA-256 与 pHash 汉明距离防重算法，拒绝劣质或换皮图片；
- **轻量训练任务调起**：以单机优先原则，调起 `ModelTrain` 的算法训练脚本并记录模型产物版本。

### 2. 架构演进与关注重点
- **单机轻量优先**：现阶段以保障核心业务链路走通与单机高可用为主，推荐使用轻量简洁的架构方案；复杂的分布式集群管理可视后续实际承载需求平滑演进。
- **契约驱动解耦**：深度学习网络结构与参数训练调优聚焦于 ModelTrain，本仓重点依据统一契约（`contracts/模型交付规约.md`）集成导出模型，完成业务推理调度与规则计算。

---

## 二、 推荐架构分层

为保证代码可读性与团队协作流畅度，**推荐采用清晰的领域分层架构**。可根据实际选用框架适当调整，保持业务逻辑与接口路由解耦：

```text
TrainPlatform/ (推荐目录结构参考)
├── README.md                      # 本说明文档
├── requirements.txt / go.mod      # 依赖包声明
├── .env.example                   # 环境变量配置模板 (PORT, DATA_DIR, MODEL_PATH)
├── .gitignore                     # 忽略项配置
└── app/                           # 核心业务代码目录
    ├── api/                       # 【路由接入层】定义 HTTP 路由端点，只负责接收参数与返回结果
    │   └── v1/                    # v1 版本路由 (/ocr, /samples, /models)
    ├── schemas/                   # 【契约数据层】强类型请求/响应校验模型 (对齐 ProjectPRD 契约)
    ├── services/                  # 【业务逻辑层】核心业务处理（调用推理、保质期规则引擎、防重检测）
    └── storage/                   # 【数据访问层】对接 ../data/raw/ 样本与数据库持久化
```

---

## 三、 跨仓契约绑定

无论本仓采用何种技术栈实现，**对外输出的 HTTP 接口必须 100% 对齐组织中枢协议**：

1. **接口规范**：严格遵循 [`ProjectPRD/contracts/API接口协议.md`](https://github.com/QingMouFoodDate/ProjectPRD/blob/main/contracts/API接口协议.md)；
   - `POST /api/v1/ocr/predict`：上传图片，返回检测框坐标 `bbox`、识别文本 `text`、标准化日期与临期状态；
   - `POST /api/v1/samples/feedback`：接收前台提交的难例纠错数据；
   - `POST /api/v1/samples/batch-upload`：接收移动端采集批量上传；
2. **数据源挂载**：直接对接同级独立暂存库 `../data/raw/`（包含 197 张黄金样本及 `manifest.jsonl`），避免大文件进 Git；
3. **质量防线**：遵循 [`ProjectPRD/standards/数据质量规范.md`](https://github.com/QingMouFoodDate/ProjectPRD/blob/main/standards/数据质量规范.md)。

---

## 四、 技术选型参考

结合项目实际需求与工程轻量原则，推荐以下实现路线：
- **路线 A (推荐 · 敏捷高效)**：`Python (FastAPI + Uvicorn)`
  - *优势*：与算法导出的 ONNX Runtime 天然无缝衔接，开发速度快，生态成熟，配合 Pydantic 可秒级对齐 API 契约；
- **路线 B (高并发演进)**：`Go (Gin)`
  - *优势*：编译为单二进制，内存占用低、高并发能力强。
- **配置与数据**：推荐优先使用 `.env` 管理路径与端口；数据库初期推荐 `SQLite`（零运维）或轻量 `PostgreSQL`。

---

## 五、 阶段目标

- [ ] **阶段 1：Mock 接口联调**：服务能够启动，并在 `/api/v1/ocr/predict` 上返回符合契约格式的 Mock 模拟数据，供 `WebClient` 前端立即联调；
- [ ] **阶段 2：真实推理对接**：服务能够载入 `ModelTrain` 交付的 ONNX 模型，对上传的原图完成真实打码圈框与日期推断；
- [ ] **阶段 3：样本流转管理**：实现采集样本入库与难例纠错记录。
