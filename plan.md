# 技术方案与架构设计

> **地位：** 技术方案、模块边界与关键约束的主要事实来源。
> **上游依据：** `spec.md`（v2，2026-11-23 评审生效；v1 归档于 `spec-v1.md`）。
> **变更规则：** 任何偏离本文件的实现，必须先更新本文件并评审通过。

## 1. 技术栈选型

| 层次 | 选型 |
|---|---|
| 前端 | Vue 3 + TypeScript + Vite + Pinia + Element Plus + ECharts |
| 后端 | Java 8（源码级别）+ Spring Boot 2.7.x + Spring Cloud Alibaba |
| 存储 | MySQL 8.0（关系型）、Redis（缓存）、Elasticsearch（检索） |
| 中间件 | Nacos（注册/配置）、RocketMQ（消息队列）、XXL-Job（分布式调度） |
| AI 接入 | RestTemplate 对接 OpenAI 兼容接口；默认本地规则词典，当前接入 DeepSeek `deepseek-chat` |
| 部署 | Docker 容器化 + Nginx |

## 2. 微服务划分与数据库隔离

| 服务名 | 职责 | 数据库 | 依赖 |
|---|---|---|---|
| gateway-service | 统一路由、JWT 鉴权、限流 | 无 | Nacos |
| user-service | 用户、角色、权限、日志 | user_db | Nacos, MySQL |
| data-service | 采集、治理、主题库、统计查询 | data_db | Nacos, MySQL, ES, RocketMQ |
| ai-service | 大模型接入、文本抽取、预测 | ai_db | Nacos, MySQL, data-service (Feign) |
| curriculum-service | 课程、技能点、项目案例 | curriculum_db | Nacos, MySQL |

**约束：** 服务之间只通过 REST/Feign 或消息队列交互，禁止跨服务直连数据库。

## 3. 分层架构

自下而上五层，标准规范体系与安全运维体系贯穿各层：

1. **基础设施层：** 服务器、操作系统、数据库、中间件、网络与备份。
2. **数据资源层：** 企业/岗位/招聘/技能主题库、标签库、课程与知识库、预测结果库、元数据与数据质量/溯源库。
3. **AI 能力层：** 岗位文本抽取模型、技能标签引擎、人才需求预测模型、大模型服务/API、向量与语义分析、异步任务调度。
4. **业务应用层：** 产业数据库管理、岗位技能分析、人才需求预测、课程与项目资源、专业调整建议、年度报告、系统管理。
5. **用户展现层：** Web 门户、管理后台、可视化大屏/看板、报告导出。

## 4. 数据流说明

### 4.1 导入流

前端上传文件 → data-service 接收 → 发送消息到 RocketMQ → 异步消费清洗去重 → 入库 MySQL → 写入 Elasticsearch。

### 4.2 AI 流

data-service 发送待抽取文本到队列 → ai-service 消费 → 调用大模型 API → 结构化结果入库 ai_db。

### 4.3 预测流

前端提交区域/时间条件 → ai-service 拉取 data-service 清洗后数据（Feign）→ 运行预测模型 → 预测结果入库 ai_db → 返回趋势/预警。

### 4.4 报告流

前端发起导出 → 报告模块按模板渲染 → 输出 Word/PDF/Excel → 记录操作日志。

## 5. 关键设计约束

- **鉴权：** Gateway 统一校验 JWT；下游服务通过请求头传递用户身份与角色，业务服务二次校验数据权限。
- **异步优先：** 超过 5 秒的采集、抽取、预测训练、报告生成任务一律走消息队列 + 任务进度查询。
- **数据库隔离：** 4 个独立库，禁止跨库 JOIN；跨域统计由 data-service 提供接口。
- **缓存：** Redis 仅缓存热点统计与字典数据，不作为事实来源。
- **检索：** Elasticsearch 承载岗位/技能全文检索、多维筛选与向量语义检索（`embedding` 字段为 `dense_vector`），MySQL 为准。
- **可观测：** 统一日志格式、操作审计表、健康检查 `/actuator/health`。
- **配置：** 所有环境相关配置外置到 Nacos，本地使用 `application-local.yml`。

### 5.1 本地环境端口适配

开发机 3306 已被本机 mysqld 占用，compose 中 MySQL 宿主端口映射为 `3307:3306`，四个服务的 JDBC URL 统一使用 `127.0.0.1:3307`。Windows 保留了 9816–10015 端口段，Nacos 2.x 的 gRPC 端口 9848/9849 无法绑定，因此 Nacos 宿主端口映射为 `18848→8848`、`19848→9848`、`19849→9849`，各服务 `server-addr` 使用 `127.0.0.1:18848`（客户端按 +1000 派生 gRPC 端口，正好命中 19848）。Elasticsearch 的 9200 同样落在保留段，宿主映射为 `19200→9200`；RocketMQ NameServer 的 9876 也落在保留段，宿主映射为 `19876→9876`。网关保持 `8080`；若该端口被其他容器占用，需先释放或停止对应容器。其余端口：Redis 6379、RocketMQ Broker 10911。

### 5.2 定时采集、语义检索与异步任务实现

**定时采集（AC-DATA-04）：** `collection_task` 表保存来源地址、Spring 6 段 Cron（秒 分 时 日 月 周）与启停状态；`CollectionTaskScheduler` 每 60 秒扫描启用任务，按 `last_run_at` 推算下次触发时间，到期后由 `CollectionTaskRunner` 读取本地文件（FILE/IMPORT）或下载 HTTP/API 文件，经 `ImportService.submitPath` 复用导入链路；增量更新依赖唯一键（企业名+岗位名+采集日期）覆盖。执行结果写入 `last_status`/`last_message`，非法 Cron 自动停用。此处以 Spring `@Scheduled` + 数据库任务表替代 XXL-Job，减少一套中间件依赖，语义等价（单实例部署）。

**语义检索（AC-AI-03）：** data-service 通过 `EmbeddingClient` 生成向量，默认 `local`（中文二元组+词元特征哈希，256 维、离线确定），可切换 `openai` 调用 OpenAI 兼容 `/embeddings` 接口（需配置 base-url/api-key/model，维度必须一致）。向量写入 ES `job_sample.embedding`（`dense_vector`，cosine），检索使用 `script_score` + `cosineSimilarity`。切换 provider 或维度后必须调用 `POST /api/v1/data/jobs/reindex` 重建索引。

**Redis 缓存：** data-service 使用 `RedisCacheManager`，默认 TTL 60 秒（`rcyc.cache.ttl-seconds`），缓存 `dashboard`/`skillHeat`/`monthlyStats`/`jobDetail` 四类热点数据；Redis 只做加速，MySQL 为事实来源。导入完成（`ImportProcessor`）与重建索引（`JobService.reindex`）会清空上述缓存，避免读到旧统计。

**异步任务与重试：** 报告导出新增 `report_task` 任务表：`POST /api/v1/data/reports/async` 立即返回任务号，后台线程渲染 Word/PDF/Excel 并落盘，`GET .../async/{taskId}` 返回进度，完成后 `GET .../async/{taskId}/download` 下载；同步导出接口保留。AI 抽取进度通过 `GET /api/v1/data/jobs/extraction-progress` 统计 `data_status=2` 的占比。RocketMQ 生产端发送失败重试 3 次（`rocketmq.producer.retry-times-when-send-failed`），导入与抽取消费者 `maxReconsumeTimes=3`。

**地图展示：** 看板内置长三角省级 GeoJSON（`frontend/src/assets/geo/changsanjiao.json`，随构建产物打包，不依赖运行时外部服务），使用 ECharts `map` 按省份展示岗位样本量，与趋势、排行、关联图表共同满足技术选型表中的图表能力要求。

### 5.3 前端信息架构对齐（2026-11-23）

依据产品原型（品牌名「产业人才智析」），前端外壳调整为分组导航：工作台（岗位数据看板）、分析洞察（岗位与技能分析、AI 工作台）、专业建设（课程与技能点、专业调整建议）、数据与质量（数据导入与分析）、系统（系统管理）。原有「专业调整建议」审批工作流保留在课程与技能点页的页签内（`?tab=advice` 深链仍可用）；侧边栏的「专业调整建议」指向 2027 级专业动态调整建议页（见 5.4）。数据导入与分析独立为 `/import` 路由，页面包含五步导入流程、导入 KPI、任务进度与质量报告，KPI 与进度来自 `GET /api/v1/data/jobs/extraction-progress` 与导入任务接口。系统管理页在原有账号与审计基础上新增三类视图：技能标签体系读取 `GET /api/v1/data/jobs/skills/heat`，数据源与同步读取 `GET /api/v1/data/import/tasks`，模型用量与模型接入配置读取 `GET /api/v1/ai/llm/models`（调用次数按 `extraction_result.model_name` 聚合）。

课程与技能点页的「课程库」页签按原型改为左侧课程目录、右侧课程详情：左栏提供课程搜索与课程列表（课程名、编码、学分），右栏展示选中课程的基本信息、技能点明细与关联项目案例。技能点明细读取 `GET /api/v1/curriculum/courses/{id}/points`，绑定操作调用 `POST /api/v1/curriculum/courses/{id}/points`（仅专业负责人/系统管理员），关联案例读取 `GET /api/v1/curriculum/project-cases?courseId=`。知识技能点、项目案例、专业调整建议三个页签与 `?tab=advice` 深链保持不变。原则：前端只做既有接口的重新组织与展示，不新增假数据、不改变接口契约。

系统管理「AI 大模型配置」页签（原「模型用量」页签升级，`name=models`）承载多模型管理：左栏为模型接入配置表（模型名称/模型标识、服务商、本年调用、状态、当前标记，操作：设为当前/编辑/删除），右栏为模型用量卡（本年总调用、当前模型占比、各模型调用条形与集中度提示）。数据源为 `llm_model` 表：ai-service 启动时由 `LlmModelInitializer` 通过 JdbcTemplate 幂等建表，空表且配置了 `rcyc.ai.llm.api-key` 时播种一行当前模型（DeepSeek / `deepseek-chat`）。接口：`GET /api/v1/ai/llm/models`（附带按 `extraction_result.model_name` 统计的本年调用数，不回传 API Key，仅 `apiKeyConfigured`）、`POST /api/v1/ai/llm/models`、`PUT/DELETE /api/v1/ai/llm/models/{id}`、`POST /api/v1/ai/llm/models/{id}/activate`；写操作需 `X-Roles` 含 `SYS_ADMIN`，API Key 编辑时留空表示不修改，当前使用的模型不可删除。抽取侧由 `LlmProviderRouter`（@Primary）在每次请求时读取 DB 中 `status=1 AND is_current=1` 的模型，用其 base_url/api_key/model_code 调用 OpenAI 兼容接口并记录调用日志；无可用模型时回退 `RuleBasedLlmClient`，调用失败按 Fail-Fast 直接报错。上一轮的 `GET/PUT /api/v1/ai/llm/config` 与其内存切换卡片已移除。

### 5.4 专业动态调整建议（2026-11-23）

**定位：** 面向 2027 级人才培养方案调整，依据岗位与技能数据自动生成调整建议，展示评审周期、建议清单、建议依据与评审进度，并支持专业负责人/系统管理员采纳建议。与 5.3 中保留的课程页「专业调整建议」审批工作流（`major_adjustment_advice`）相互独立，后者不做改动。

**数据来源：** curriculum-service 通过 Feign 调用 data-service 的 `GET /api/v1/data/jobs/stats`（岗位总数、企业数、平均月薪、`topJobs` 前 10）与 `GET /api/v1/data/jobs/skills/heat?limit=30`（技能热度），不引入任何写死的统计数字。data-service 不可用时直接报错（HTTP 502），不做降级或兜底假数据。

**生成规则：** 计划年度取当前年份 + 1；专业方向 2 条取 `topJobs` 前 2，课程内容 4 条取技能热度第 4–7 名，技能新点 3 条取技能热度前 3；优先级按样本占比阈值划分（≥20% 高、≥10% 中、其余低）；编号分别为 `ADJ-{year}-D01…`、`-C01…`、`-S01…`。`GET /api/v1/curriculum/adjustments/plan` 每次请求重新拉取数据并按 `(plan_year, suggestion_no)` 幂等 upsert，已采纳记录保留 `status/adopted_by/adopted_at` 不变。

**存储：** `adjustment_plan_item` 表（curriculum_db），由 `AdjustmentPlanInitializer` 启动时通过 JdbcTemplate 幂等建表；字段含 plan_year、suggestion_no、category、priority、title、basis、metrics(JSON)、owner_unit、impact_scope、status、adopted_by、adopted_at、generated_at 及时间戳。

**采纳：** `POST /api/v1/curriculum/adjustments/{id}/adopt`，要求 `X-Roles` 含 `MAJOR_LEADER` 或 `SYS_ADMIN`，写入 `status=ADOPTED` 与采纳人/时间并写操作审计；重复采纳返回 409。计划查询与详情无需角色限制。

**评审进度：** 四阶段（建议生成 / 专业负责人复核 / 院系教学委员会评审 / 教务处备案）。建议生成固定为已完成（取生成时间）；专业负责人复核按已采纳条数展示「进行中 x/9」或未发起；后两阶段未发起，不虚构审批结果。

**前端：** 新增 `/adjustments` 路由与 `AdjustmentsView.vue`：顶部周期条与数据依据、4 个 KPI、左侧「全部/专业方向/课程内容/技能新点」筛选与建议卡片（分类、优先级、编号、标题、依据、指标、责任单位/影响范围、查看详情/采纳建议），右栏「本栏建议依据」与「评审进度」时间线。非 `MAJOR_LEADER`/`SYS_ADMIN` 不显示采纳按钮，仅可查看。

## 6. 目录结构约定

```
RCYCsystem/
├── docs/                    # 权威文档
├── deploy/                  # docker-compose、初始化 SQL、Nginx 配置
├── backend/                 # Maven 父工程
│   ├── gateway-service/
│   ├── user-service/
│   ├── data-service/
│   ├── ai-service/
│   └── curriculum-service/
└── frontend/                # Vue 3 单页应用
```
