# 接口契约

> **地位：** 服务间与前后端接口的约定；变更需同步 `spec.md` 验收标准。
> **通用约定：** 所有接口经 Gateway 暴露，统一前缀 `/api/v1`；响应体 `{ code, message, data }`；鉴权使用 `Authorization: Bearer <JWT>`。

## 数据服务 (data-service)

### POST /api/v1/data/import/jobs

- **描述**：上传岗位样本文件
- **请求**：`multipart/form-data`，file (Excel/CSV)
- **响应**：

```json
{
  "code": 200,
  "message": "文件已接收，正在后台处理",
  "data": { "taskId": "IMP-20231024-001" }
}
```

### GET /api/v1/data/import/tasks/{taskId}

- **描述**：查询导入任务进度与成功/失败报告（AC-DATA-03）
- **响应**：

```json
{
  "code": 200,
  "data": {
    "taskId": "IMP-20231024-001",
    "status": "DONE",
    "total": 5000,
    "success": 4870,
    "failed": 130,
    "errors": [{ "row": 42, "reason": "企业名称为空" }]
  }
}
```

### GET /api/v1/data/jobs/stats

- **描述**：多维度统计查询
- **请求参数**：`region` (江苏)，`skill` (Java)，`timeRange` (2023-01~2023-10)
- **响应**：

```json
{
  "code": 200,
  "data": {
    "totalCount": 15000,
    "salaryAvg": 12500,
    "trend": [{ "month": "2023-09", "count": 1200 }]
  }
}
```

### GET /api/v1/data/jobs/extraction-progress

- **描述**：AI 抽取进度反馈（异步任务进度查询）
- **响应**：

```json
{ "code": 200, "data": { "total": 4981, "done": 4800, "pending": 181, "percent": 96.4 } }
```

- `done` 统计 `job_sample.data_status=2`（已完成抽取）的记录，`percent` 保留 1 位小数。

### GET /api/v1/data/jobs/{id}/trace

- **描述**：查询单条岗位样本的来源溯源信息（AC-DATA-02）
- **响应**：

```json
{
  "code": 200,
  "data": {
    "jobId": 1001,
    "originalUrl": "https://example.com/job/1001",
    "channel": "IMPORT",
    "collectedAt": "2023-10-24 10:00:00"
  }
}
```

### POST /api/v1/data/jobs/semantic-search

- **描述**：自然语言语义检索岗位（AC-AI-03），按向量余弦相似度排序
- **请求体**：

```json
{ "query": "Java 后端 微服务", "region": "江苏", "size": 10 }
```

- **响应**：`code=200`，`data.total` 为命中总数，`data.list` 为岗位文档（`jobId/jobTitle/companyName/region/skills` 等）

### GET /api/v1/data/jobs/{id}/similar

- **描述**：相似岗位推荐（AC-AI-03），排除自身
- **请求参数**：`size`，默认 5，最大 50
- **响应**：同语义检索结构

### POST /api/v1/data/jobs/reindex

- **描述**：重建 ES 索引（含语义向量）；切换 `rcyc.embedding.provider` 或维度后必须调用
- **响应**：`{"code":200,"data":{"indexed":4981}}`

### 报告导出（同步与异步）

同步（保留）：

- `GET /api/v1/data/reports/annual?format=docx|pdf&year=2026`
- `GET /api/v1/data/reports/annual/excel?year=2026`

异步（推荐）：

- `POST /api/v1/data/reports/async?format=docx|pdf|excel&year=2026`：创建报告任务，立即返回 `taskId` 与 `status=PENDING`
- `GET /api/v1/data/reports/async/{taskId}`：查询进度，`status` 为 `PENDING/RUNNING/DONE/FAILED`，失败时 `message` 给出原因
- `GET /api/v1/data/reports/async/{taskId}/download`：任务为 `DONE` 时下载文件；未完成返回 409

报告文件落盘目录由 `rcyc.report.dir` 配置（默认 `./data/report`），任务记录写入 `data_db.report_task`。

### 采集任务（AC-DATA-04）

- `GET /api/v1/data/collection-tasks?page=1&size=10`：分页查询
- `POST /api/v1/data/collection-tasks`：创建，请求体 `{"taskName","sourceType":"FILE|HTTP","sourceUrl","cronExpr":"0 0 2 * * *","status":1}`；Cron 为 6 段格式，非法返回 422
- `PUT /api/v1/data/collection-tasks/{id}`：更新
- `POST /api/v1/data/collection-tasks/{id}/run`：立即执行一次，返回含 `lastStatus/lastMessage/lastRunAt`
- `DELETE /api/v1/data/collection-tasks/{id}`：删除

## AI 服务 (ai-service)

### POST /api/v1/ai/extract

- **描述**：触发岗位文本 AI 抽取
- **请求体**：

```json
{ "jobDescription": "熟悉Java, Spring Boot, 有大模型开发经验优先..." }
```

- **响应**：

```json
{
  "code": 200,
  "data": {
    "skills": ["Java", "Spring Boot"],
    "tools": ["Git", "Maven"],
    "abilities": ["大模型开发"]
  }
}
```

### GET /api/v1/ai/forecast

- **描述**：人才需求预测（AC-AI-02）
- **请求参数**：`region` (江苏)，`months` (3)
- **响应**：

```json
{
  "code": 200,
  "data": {
    "region": "江苏",
    "model": "prophet",
    "points": [{ "month": "2023-11", "demand": 1320 }]
  }
}
```

## 用户服务 (user-service)

### POST /api/v1/auth/login

- **描述**：账号登录，返回 JWT
- **请求体**：

```json
{ "username": "teacher01", "password": "******" }
```

- **响应**：

```json
{
  "code": 200,
  "data": {
    "token": "<JWT>",
    "user": { "id": 3, "username": "teacher01", "roles": ["TEACHER"] }
  }
}
```

### GET /api/v1/audit/logs

- **描述**：审计员查询操作日志，仅只读（AC-BIZ-02）
- **请求参数**：`page`，`size`，`operator`，`timeRange`
- **响应**：分页日志列表（含操作人、动作、对象、时间、结果）。

## 课程服务 (curriculum-service)

### POST /api/v1/curriculum/advice

- **描述**：教师提交课程更新建议
- **请求体**：

```json
{
  "courseId": 101,
  "title": "将大模型应用开发纳入实训",
  "content": "建议新增 Spring AI 实训项目...",
  "evidence": ["AC-AI-01"]
}
```

- **响应**：返回建议单号与待审批状态。

### POST /api/v1/curriculum/advice/{id}/approve

- **描述**：专业负责人审批课程更新建议（AC-CUR-03）
- **请求体**：

```json
{ "decision": "APPROVED", "comment": "同意，纳入 2025 级培养方案" }
```
