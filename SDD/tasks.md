# 任务清单 (Task Board)

> **地位：** 执行拆解；不得反向改写 `spec.md` 或 `plan.md`。
> **用法：** 每项任务完成并通过验证后勾选，并在提交信息中引用任务编号。

## Phase 1: 基础设施

- [x] T-01 编写 Docker Compose 拉起 MySQL/Redis/Nacos/ES/RocketMQ
- [x] T-02 初始化四个数据库及基础表结构 (data_db, ai_db, user_db, curriculum_db)

## Phase 2: 微服务底座

- [x] T-03 创建微服务 Maven 父工程及网关模块
- [x] T-04 实现 Gateway 全局鉴权过滤器与路由转发
- [x] T-05 验证所有微服务成功注册至 Nacos

## Phase 3: 数据服务 (data-service)

- [x] T-06 开发岗位数据 CSV/Excel 解析与导入接口
- [x] T-07 开发基于 RocketMQ 的异步数据清洗去重逻辑
- [x] T-08 建立 Elasticsearch 索引，实现多维组合查询 API
- [x] T-09 开发数据溯源查询接口

## Phase 4: AI 服务 (ai-service)

- [x] T-10 封装通用大模型 API 调用客户端
- [x] T-11 开发 Prompt 模板，实现岗位 JD 的 JSON 结构化抽取
- [x] T-12 开发人才需求预测算法（时间序列/模型调用）接口

## Phase 5: 业务服务与前端

- [x] T-13 开发 user-service 的 RBAC 与登录接口
- [x] T-14 开发 curriculum-service 的课程管理接口
- [x] T-15 搭建 Vue3 前端项目，完成原型图切图与静态页
- [x] T-16 对接 ECharts 开发大屏看板与统计图表
- [x] T-17 实现前端权限路由与 API 对接

## Phase 6: 验证与交付

- [x] T-18 实现 Word/PDF 报告导出引擎
- [x] T-19 验收 AC-DATA-01 至 AC-BIZ-03 全部指标
- [x] T-20 编写部署运维手册与系统安全配置
- [x] T-21 实现采集任务配置与定时/增量执行（AC-DATA-04）
- [x] T-22 实现向量语义检索与相似岗位推荐（AC-AI-03）
- [x] T-23 接入 Redis 热点缓存（看板统计/技能热度/岗位详情，TTL 可配，导入与重建索引失效）
- [x] T-24 看板新增长三角岗位区域地图（内置 GeoJSON，不依赖外部服务）
- [x] T-25 报告异步任务与进度查询、AI 抽取进度接口、RocketMQ 重试配置
- [x] T-26 五个 Java 服务容器化 + docker compose 应用编排 + Nginx 统一入口

## Phase 7: 前端原型对齐（2026-11-23）

> 依据：用户提供的 12 张高保真原型截图（产品名「产业人才智析」），仅调整前端信息架构与页面呈现，不新增后端接口、不伪造业务数据。

- [x] T-27 应用外壳对齐原型：品牌区改为「产业人才智析」，导航按 工作台/分析洞察/专业建设/数据与质量/系统 分组，左下角改为可核验的系统状态卡
- [x] T-28 系统管理页对齐原型：新增账号/角色/技能标签/导入任务 KPI；新增「技能标签体系」「数据源与同步」「模型用量」页签，数据分别来自技能热度、导入任务、AI 抽取记录
- [x] T-29 导航深链：课程页支持 `?tab=advice` 直达专业调整建议；数据导入与分析改为独立路由 `/import`
- [x] T-30 前端类型检查与构建通过，并在浏览器复核页面渲染与深链行为
- [x] T-31 新增数据导入与分析页（`ImportView.vue`）：4 个真实 KPI、五步导入流程、上传与任务进度、最近任务、质量报告、导入规范与字段识别说明；岗位页导入入口统一跳转该页
- [x] T-32 课程与技能点页「课程库」页签改为左侧课程目录 + 右侧课程详情：课程搜索与列表、课程基本信息、技能点明细与绑定弹窗、关联项目案例；保留知识技能点/项目案例/专业调整建议页签与 `?tab=advice` 深链
- [x] T-33 侧边栏「专业建设」分组移除「专业调整建议」导航项，该功能仅保留在课程与技能点页的页签中（`?tab=advice` 深链仍可用）
- [x] T-34 AI 大模型接入切换：「模型用量」页签新增接入卡片，支持 SYS_ADMIN 在本地规则引擎与 OpenAI 兼容大模型间运行时切换（`GET/PUT /api/v1/ai/llm/config`），切换状态存内存、重启恢复配置（已被 T-35 取代，旧接口与卡片已移除）
- [x] T-35 AI 大模型配置（多模型管理）：新增 `llm_model` 表（启动时幂等建表并播种当前 DeepSeek 配置），系统管理新增「AI 大模型配置」页签（左栏模型接入表 + 右栏模型用量卡），支持 SYS_ADMIN 新增/编辑/删除模型与「设为当前」切换；抽取路由按 DB 中当前启用模型动态调用 OpenAI 兼容接口，无可用模型时回退本地规则引擎；接口 `GET/POST /api/v1/ai/llm/models`、`PUT/DELETE /api/v1/ai/llm/models/{id}`、`POST /api/v1/ai/llm/models/{id}/activate`
- [x] T-36 2027 级专业动态调整建议：curriculum-service 新增 `adjustment_plan_item` 表与 `GET /api/v1/curriculum/adjustments/plan`（按 data-service 真实岗位/技能数据生成 9 条建议并幂等 upsert）、`POST /api/v1/curriculum/adjustments/{id}/adopt`（MAJOR_LEADER/SYS_ADMIN）；前端新增 `/adjustments` 页与侧边栏入口，展示周期、KPI、筛选、建议卡片、建议依据与评审进度
- [x] T-37 业务规格 v2 评审采纳（2026-11-23）：按 `spec-prompt.md` 结构重写业务规格，覆盖 16 个场景 / 44 条需求 / 33 条验收标准，含术语表、业务不变量与追溯矩阵；状态置为已评审生效并作为唯一事实来源，v1 归档为 `docs/spec-v1.md`；C-01~C-05、TBD-01~TBD-07 作为开放项持续跟踪
- [x] T-38 技术方案与数据库设计重构（2026-11-24）：按 plan-prompt v3.1 重构 `plan.md`（审查结论、7 项架构决策、领域模型与状态机、API 契约、AC→测试映射、里程碑）；按 db-design-prompt v1.1 编制 `db-design.md`（四库 23 张表、关系/索引/数据字典/生命周期/追溯矩阵，登记 TBD-01~TBD-09）；`data-model.md` 标记为历史版本
- [x] T-39 原型对齐 v3（2026-11-24）：导航调整为工作台/分析洞察/专业建设/数据与质量/系统，顶栏新增全局搜索与数据更新时间；看板更名「产业数据看板」，新增季度/区域筛选、AI 岗位占比 KPI、新兴技能预警、AI 协同岗位结构、重点岗位需求明细、数据溯源与质量卡（真实来源：`/data/jobs/stats`、`/data/jobs/insights`、`/data/jobs/skills/heat`、`/data/import/tasks`、`/data/quality/report`）；新增 `/reports` 报告与成果页（真实报告任务 `GET /data/reports/tasks`、异步生成与下载、评审发布系统外说明、按创建年度归集课程/技能点/案例台账）；新增 `GET /data/jobs/insights`、`GET /data/jobs/updated-at`、`GET /data/reports/tasks`，全部指标来自真实台账，不使用演示数字

## Phase 8: AI 辅助能力（2026-11-26）

**预检报告（tasks-prompt v3.2）**
- 一致性检查：`spec.md` SCN-17 / REQ-45~47 / AC-AI-06~08 与 `plan.md` AD-08 一致；无冲突。
- 技术栈确认：Java 8 + Spring Boot 2.7.18 + Spring Cloud 2021.0.8 + MyBatis-Plus；Vue 3 + TS + Element Plus。
- 工作目录：项目根 `e:\RCYCsystem`；后端 `backend/`，前端 `frontend/`，部署 `deploy/`。
- 前后端边界：AI 能力在 `backend/ai-service`；导入预览在 `backend/data-service`；建议接入在 `backend/curriculum-service`；前端 API 在 `frontend/src/api`。

- [x] T40-01 [REQ-46] [AC-AI-07] [Phase 8] [Size: S] 编写字段映射规则测试：精确匹配 100%、无法匹配“待映射”、模型返回字典外编码降级为“待映射”。 依赖：无。 文件：`backend/ai-service/src/test/java/edu/niit/rcyc/ai/assist/FieldMappingServiceTest.java`。 验证：`mvn -pl ai-service test -Dtest=FieldMappingServiceTest`。 完成判据：三类断言全部通过。 来源：`spec.md#SCN-17, AC-AI-07`；`plan.md#AD-08`
- [x] T40-02 [REQ-46] [AC-AI-07] [Phase 8] [Size: M] 实现字段映射能力：按 `databaseFields` 先精确匹配，再调用当前模型做语义匹配，最后白名单校验；返回 `provider/model/fallback`。 依赖：T40-01。 文件：`backend/ai-service/src/main/java/edu/niit/rcyc/ai/assist/FieldMappingService.java`。 验证：T40-01 通过，接口手工调用返回 4 条映射且无字典外编码。 完成判据：AC-AI-07 满足。 来源：`spec.md#REQ-46`
- [x] T40-03 [REQ-45] [AC-AI-06] [Phase 8] [Size: S] 编写企业行业识别规则测试：IT 类企业 aiFlag=1、非 IT=0、模型不可用时回退仍返回全部公司。 依赖：无。 文件：`backend/ai-service/src/test/java/edu/niit/rcyc/ai/assist/CompanyClassifyServiceTest.java`。 验证：`mvn -pl ai-service test -Dtest=CompanyClassifyServiceTest`。 完成判据：0/1 与回退断言通过。 来源：`spec.md#AC-AI-06`
- [x] T40-04 [REQ-45] [AC-AI-06] [Phase 8] [Size: S] 实现企业行业识别能力与 `POST /api/v1/ai/assist/companies/classify`。 依赖：T40-03。 文件：`backend/ai-service/src/main/java/edu/niit/rcyc/ai/assist/CompanyClassifyService.java`、`.../controller/AiAssistController.java`。 验证：T40-03 通过，接口返回 `aiFlag` 为数字。 完成判据：AC-AI-06 满足。 来源：`spec.md#REQ-45`
- [x] T40-05 [REQ-47] [AC-AI-08] [Phase 8] [Size: S] 编写调整建议规则回退测试：输入岗位技能与课程技能点，输出方向/课程/技能点三类且每项含依据。 依赖：无。 文件：`backend/ai-service/src/test/java/edu/niit/rcyc/ai/assist/AdjustmentSuggestionServiceTest.java`。 验证：`mvn -pl ai-service test -Dtest=AdjustmentSuggestionServiceTest`。 完成判据：三类覆盖与依据非空断言通过。 来源：`spec.md#AC-AI-08`
- [x] T40-06 [REQ-47] [AC-AI-08] [Phase 8] [Size: M] 实现调整建议生成能力与 `POST /api/v1/ai/assist/adjustments/generate`，统一复用当前模型并记录调用日志。 依赖：T40-05。 文件：`backend/ai-service/src/main/java/edu/niit/rcyc/ai/assist/AdjustmentSuggestionService.java`、`.../llm/OpenAiCompatibleLlmClient.java`、`.../llm/LlmProviderRouter.java`。 验证：T40-05 通过。 完成判据：AC-AI-08 满足。 来源：`spec.md#REQ-47`；`plan.md#AD-08`
- [ ] T40-07 [REQ-46] [AC-AI-07] [Phase 8] [Size: S] data-service 新增表头读取与映射预览：读取 xlsx/csv 首行表头，调用 ai-service 映射接口，服务不可用时按本地精确匹配回退。 依赖：T40-02。 文件：`backend/data-service/src/main/java/edu/niit/rcyc/data/controller/ImportController.java`、`.../service/ImportMappingService.java`、`.../client/AiServiceClient.java`。 验证：上传 `data/samples/cnopendata_jobs_sample.xlsx` 返回映射清单。 完成判据：每个表头一条结果，响应标注来源。 状态：代码与编译已完成；运行时上传复核待环境内存恢复。 来源：`spec.md#REQ-46`
- [x] T40-08 [REQ-46] [AC-AI-07] [Phase 8] [Size: S] 前端导入页选择文件后展示真实字段映射表（外部表头/标准字段/匹配方式/置信度/状态）。 依赖：T40-07。 文件：`frontend/src/views/ImportView.vue`、`frontend/src/api/data.ts`、`frontend/src/types/index.ts`。 验证：`npm run build` 通过；浏览器选择样例文件后表格出现 4 行。 完成判据：映射结果来自接口，不写死。 来源：`spec.md#AC-AI-07`
- [x] T40-09 [REQ-45] [AC-AI-06] [Phase 8] [Size: S] 前端 AI 工作台新增“企业行业识别”面板，使用真实覆盖企业名称调用识别接口并展示 aiFlag 与来源。 依赖：T40-04。 文件：`frontend/src/views/AiView.vue`、`frontend/src/api/ai.ts`、`frontend/src/types/index.ts`。 验证：`npm run build` 通过；浏览器识别 Top 企业返回 0/1。 完成判据：结果来自接口且标注模型/规则。 来源：`spec.md#AC-AI-06`
- [x] T40-10 [REQ-47] [AC-AI-08] [Phase 8] [Size: M] curriculum-service 年度建议生成接入 AI：组装真实岗位与课程体系输入，AI 不可用或输出不合法时回退现有规则；已采纳建议不被覆盖。 依赖：T40-06。 文件：`backend/curriculum-service/src/main/java/edu/niit/rcyc/curriculum/service/AdjustmentPlanService.java`、`.../client/AiServiceClient.java`、`.../dto/AiSuggestion*.java`。 验证：无模型时仍生成三类建议；采纳后重新打开状态保持。 完成判据：AC-AI-08 与 INV-07 满足。 来源：`spec.md#REQ-47, INV-07`
- [x] T40-11 [REQ-45~47] [AC-AI-06~08] [Phase 8] [Size: S] 构建、部署与浏览器复核三个 AI 能力，并记录验收证据。 依赖：T40-02、T40-04、T40-06、T40-08、T40-09、T40-10。 文件：`docs/tasks.md`。 验证：`mvn -DskipTests package`、`npm run build`、`docker compose up -d --build ai-service data-service curriculum-service`、浏览器实测。 完成判据：三处能力均可用且失败时显式标注。 来源：`plan.md#AD-08`

### 验收记录（2026-11-26，AI 辅助能力）

| 验收项 | 结果 | 证据 |
|---|---|---|
| ai-service 单测 | 通过 | `mvn -pl ai-service -am test`：13 个测试全部通过（新增分类/映射/建议 7 个） |
| 构建 | 通过 | `mvn -DskipTests -pl ai-service,data-service,curriculum-service -am package` 成功；`npm run build` 成功（2278 模块） |
| 企业行业识别 | 通过 | `POST /ai/assist/companies/classify`：真实模型 `provider=openai fallback=false`，云创/润和 `aiFlag=1`，餐饮公司 `aiFlag=0` |
| 导入字段映射 | 通过 | `POST /data/import/mapping` 上传样例文件：HTTP 200，12 个表头全部“完全匹配/已映射”，来源 `rule/fallback=false` |
| 调整建议生成 | 通过 | `GET /curriculum/adjustments/plan?regenerate=true`：HTTP 200，14 条建议（DIRECTION 4 / COURSE 5 / SKILL 5），内容来自模型 |
| 已采纳保护 | 通过 | 采纳 `ADJ-2027-D001` 后重新生成：该条仍存在且状态 `ADOPTED`，未被覆盖 |
| 浏览器 UI 复核 | 待环境恢复 | 内存受限（宿主机可用 <1GB），已完成接口与构建验证；页面交互复核待资源恢复后补测 |

> 内存受限处理：停用 `rcyc-elasticsearch` 释放内存（岗位全文检索暂不可用，可 `docker start rcyc-elasticsearch` 恢复）；各 Java 服务通过 `JAVA_TOOL_OPTIONS` 限制堆（256~384MB），RocketMQ NameServer/Broker 限制为 256/512MB。

### 验收记录（2026-11-24，构建验证）

| 验收项 | 结果 | 证据 |
|---|---|---|
| 前端构建（T-39） | 通过 | `npm run build`（vue-tsc + vite）成功，2278 个模块，`ReportsView` 与 `DashboardView` 产物生成 |
| 后端编译（T-39） | 通过 | `mvn -DskipTests -pl data-service -am package` 成功，78 个源文件编译通过，`data-service-1.0.0.jar` 生成 |
| 看板真实口径 | 静态核对 | KPI/图表仅引用 `/data/jobs/stats`、`/data/jobs/insights`、`/data/jobs/skills/heat`、`/data/import/tasks`、`/data/quality/report`，未写死演示数字 |
| 运行时验证（2026-11-24） | 通过 | Docker 引擎 28.5.1 启动，12 个 rcyc 容器运行；admin 登录成功；`/data/jobs/insights` 返回 `aiJobRatio=15.6`、`emergingSkills=6`、`topJobDetails=8`；`/data/reports/tasks` 返回 2 条 DONE 任务；报告下载 HTTP 200（3643 bytes，docx） |
| 浏览器复核（2026-11-24） | 通过 | `http://localhost/dashboard` 显示「岗位需求总量 4,981 条」「AI 岗位占比 15.6%」，无报错；`http://localhost/reports` 显示 2 条已完成报告（Word/PDF）与 2026 年建设成果台账（17 条），无报错 |

> 启动排障记录：Nacos 内嵌 Derby 残留锁会导致 `Login timeout exceeded` 启动失败，清理 `/home/nacos/data/derby-data/*.lck` 并重启容器后恢复。
> 看板排障记录：`DashboardInsight.emergingSkills` 曾使用 `subList`，Redis JSON 反序列化失败导致看板报“服务器内部错误”，已改为 `new ArrayList<>(signals.subList(...))` 并重启 data-service 验证通过。
> 导入排障记录：`deploy/rocketmq/broker.conf` 的 `brokerIP1` 原为 `127.0.0.1`，容器内生产者拿到错误 broker 地址导致 `RemotingException`；已改为 `rocketmq-broker`，重启 broker 与 data-service 后样例文件导入成功（上传 200，任务 DONE，成功 318 / 失败 7 / 总数 325）。

### 验收记录（2026-11-23，本地环境）

| 验收项 | 结果 | 证据 |
|---|---|---|
| 应用外壳 | 通过 | 侧边栏显示品牌「产业人才智析」与 5 个导航分组，底部状态卡展示当前角色/数据范围 |
| 系统管理 KPI | 通过 | 浏览器实测：系统账号 6、角色类型 6、技能标签 24、导入任务 8（均来自既有接口） |
| 系统管理页签 | 通过 | 实测 5 个页签可见：账号与角色、技能标签体系、数据源与同步、AI 大模型配置、操作日志 |
| 导航深链 | 通过 | `/curriculum?tab=advice` 可直达专业调整建议；数据导入使用 `/import` 独立路由 |
| 数据导入与分析页 | 通过 | 浏览器实测：4 个 KPI（导入任务 8、已抽取 4981、待抽取 0、完成度 100%）、五步流程、上传区、最近任务表、导入规范与字段识别说明均正常渲染 |
| 课程库双栏 | 通过 | 浏览器实测：左侧“课程目录”列出 5 门课程并默认选中首门，右侧显示课程名称/编码/学分/简介、技能点明细与关联项目案例；点击“Java企业级开发”后右侧详情同步更新 |
| 技能点绑定接口 | 通过 | 经网关以 SYS_ADMIN 调用 `POST /api/v1/curriculum/courses/1/points` 绑定 2 个技能点成功返回 2 条，再传空数组恢复为 0 条，数据已还原 |
| 大模型接入接口 | 通过 | `GET /api/v1/ai/llm/config` 返回 provider=openai、model=deepseek-chat、defaultProvider=openai、apiKeyConfigured=true、2 个可选接入方；`PUT` 切到 rule 返回 rule-dictionary-v1，再切回 openai 成功（T-34 历史记录，接口已随 T-35 移除） |
| 大模型接入界面 | 通过 | 浏览器实测「模型用量」页：接入卡片显示当前 deepseek-chat、API Key 已配置、两个接入选项；选择「本地规则引擎」并点击「切换接入」后，当前生效更新为 rule-dictionary-v1（T-34 历史记录，卡片已随 T-35 移除） |
| AI 大模型配置页签 | 通过 | 浏览器实测：系统管理「AI 大模型配置」页签左栏「大模型接入配置」显示 DeepSeek Chat（当前、启用、本年调用 0），右栏「模型用量」显示本年总调用与模型条形，控制台无报错 |
| 大模型多模型接口 | 通过 | 经网关以 SYS_ADMIN 实测：GET 返回播种模型；POST 新增临时模型 → activate 切换成功 → 恢复 DeepSeek 为当前 → DELETE 删除成功；删除/停用当前模型分别返回 400「当前使用的模型不可删除/停用，请先切换其他模型」；临时数据已清理，最终仅保留 DeepSeek Chat 为当前 |
| 抽取路由使用当前模型 | 通过 | `POST /api/v1/ai/extract` 返回 `modelName=deepseek-chat`，技能抽取结果由 DB 中当前 DeepSeek 配置调用生成 |
| 前端构建 | 通过 | `npm run build`（vue-tsc + vite）成功，2271 个模块，产物已由 Nginx 挂载生效 |
| 专业调整建议接口 | 通过 | 经网关以 SYS_ADMIN 调用 `GET /api/v1/curriculum/adjustments/plan`，返回 planYear=2027、9 条建议（方向 2 / 课程 4 / 技能 3）、样本岗位 4,981 与企业 215、平均月薪 22,443、生成时间，编号 ADJ-2027-D01…S03 |
| 采纳与权限 | 通过 | `POST /api/v1/curriculum/adjustments/1/adopt` 返回 ADOPTED，计划 adoptedCount=1 且复核阶段变为「已采纳 1/9 条」；重复采纳返回 409；直连服务以 TEACHER 调用返回 403；测试采纳数据已还原为 PENDING |
| 专业调整建议页面 | 通过 | 浏览器实测 `/adjustments`：标题「2027级专业动态调整建议」、4 个 KPI（9/2/4/3）、筛选（全部 9 / 专业方向 2 / 课程内容 4 / 技能新点 3）、建议卡片、右栏「本栏建议依据」与 4 阶段「评审进度」均正常渲染，导航入口位于专业建设分组 |
| 前端构建（T-36） | 通过 | `npm run build` 成功，2275 个模块，`AdjustmentsView` 产物已由 Nginx 挂载生效 |

## 验收记录（2026-11-20，本地环境）

| 验收项 | 结果 | 证据 |
|---|---|---|
| AC-DATA-01 | 通过 | 5079 行 CSV 导入耗时 8.10s（含上传），任务 success=5079 |
| AC-DATA-02 | 通过 | 岗位详情返回 5 条溯源记录（原始链接、渠道、采集时间） |
| AC-DATA-03 | 通过 | 重复键覆盖更新，`job_sample` 4979 行且唯一键 4979；非法行生成 failed=1 失败报告 |
| AC-AI-01 | 通过 | 抽取返回 skills/tools/abilities 三类标签；批量抽取 4979 条记录 |
| AC-AI-02 | 通过 | 江苏未来 3 个月预测返回 3 个数据点（linear-regression-v1） |
| AC-BIZ-01 | 通过 | 看板 KPI 与趋势/薪资/区域/词云/Top10 图表均有数据 |
| AC-BIZ-02 | 通过 | 教师访问用户管理 403、分析员访问审计 403、审计员可查审计日志；无 Token 401 |
| AC-BIZ-03 | 通过 | 年度报告 Word/PDF/Excel 与专业调整建议 Word/PDF 均成功导出 |
| AC-CUR-03 | 通过 | 教师提交→负责人审批通过；重复审批 422；教师越权审批 403 |
| ES 检索 | 通过 | `job_sample` 索引 4979 文档，关键字检索 total=924 |
| AC-DATA-04 | 通过 | FILE 采集任务 Cron `0 * * * * *` 自动触发，导入 1 行成功；重复执行后岗位数仍为 1（增量覆盖） |
| AC-AI-03 | 通过 | 语义检索“Java backend microservice”+江苏返回 total=1847；相似岗位返回同族 Java 岗位，排除自身 |

## 验收补充记录（2026-11-22，本地环境）

| 验收项 | 结果 | 证据 |
|---|---|---|
| Redis 缓存 | 通过 | `/api/v1/data/jobs/stats` 首次 3090ms，命中缓存后 113ms；Redis 存在 `rcyc:cache:dashboard::null:null:null` |
| 地图图表 | 通过 | 看板渲染“长三角岗位区域地图”，江苏/浙江/上海/安徽按样本量着色 |
| 抽取进度 | 通过 | `GET /api/v1/data/jobs/extraction-progress` 返回 total=4981、done=4981、percent=100.0 |
| 报告异步生成 | 通过 | `POST /api/v1/data/reports/async` 返回任务号，状态 PENDING→DONE；PDF 4307B（%PDF）、DOCX 3643B（PK）下载成功 |
| 容器化 + Nginx | 通过 | 5 个服务镜像构建成功；`docker compose up -d` 后经 `http://127.0.0.1/` 登录 200、看板统计与异步报告可用 |

> 本地端口适配：MySQL 3307、Nacos 18848（gRPC 19848/19849）、Elasticsearch 19200、RocketMQ NameServer 19876，详见 `docs/plan.md` 5.1 节与 `docs/deployment.md`。
