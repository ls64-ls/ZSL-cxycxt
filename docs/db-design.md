# 数据库设计（db-design.md）

> **版本：** v1.0（2026-11-24，按 db-design-prompt v1.1 编制）
> **数据库：** MySQL 8.0（InnoDB / utf8mb4 / utf8mb4_general_ci）
> **上游依据：** `spec.md`（v2，REQ-01~44）、`plan.md`（v2）、`api-contracts.md`、`deploy/mysql/init/01-schema.sql`、运行期初始化器
> **地位：** 数据库逻辑与物理设计的事实来源；不包含应用代码、ORM 配置、接口与界面设计。

## 0. 文档信息与阻塞项检查

| 项目 | 内容 |
|---|---|
| 数据库类型 | 关系型（MySQL） |
| 引擎/版本 | InnoDB / MySQL 8.0 |
| 字符集 | utf8mb4 / utf8mb4_general_ci |
| 部署形态 | 容器内单实例；四库同实例，按服务隔离 |
| 设计范围 | user_db、data_db、ai_db、curriculum_db 共 23 张表 |
| 事实来源 | 初始化脚本（已落库结构）+ 运行期幂等建表（`llm_model`、`adjustment_plan_item`） |
| 阻塞性缺失 | 无（引擎、版本、库边界均已知） |
| 未决事项 | 数据保留期限、API Key 加密、物理外键策略、模型当前唯一性约束（见第 9 章） |

## 1. 总体设计决策

| 编号 | 决策 | 说明 | 状态 |
|---|---|---|---|
| D-01 | 一服务一库 | user-service→user_db；data-service→data_db；ai-service→ai_db；curriculum-service→curriculum_db；禁止跨库直连 | 【已确认】 |
| D-02 | 主键统一 `id BIGINT AUTO_INCREMENT` | 单库自增，简单稳定；不使用 UUID/雪花 | 【已确认】 |
| D-03 | 命名统一 snake_case，表名业务单数 | 如 `job_sample`、`major_adjustment_advice`；字段与 Java 驼峰映射 | 【已确认】 |
| D-04 | 时间统一 `DATETIME`，时区 Asia/Shanghai | 业务时间不使用 TIMESTAMP，避免时区隐式转换 | 【已确认】 |
| D-05 | 布尔语义统一 `TINYINT` 0/1 | `status`、`deleted`、`is_current` | 【已确认】 |
| D-06 | 枚举统一 `VARCHAR` 可读码 | 便于跨语言与排查；由应用与服务端校验合法性 | 【已确认】 |
| D-07 | 金额现状为 `INT`（单位：K/月） | `job_sample.salary_min/max` 以“千元/月”存储；不满足“最小货币单位 DECIMAL”默认规范 | 【现状/待确认】 |
| D-08 | 软删除仅用于岗位样本 | `job_sample.deleted`；其余表采用物理删除或状态位 | 【已确认】 |
| D-09 | 不建物理外键，采用逻辑外键 | 服务自治、初始化脚本历史、跨库引用无法建 FK；以唯一键、应用校验、定期核对替代 | 【现状/待确认】 |
| D-10 | 单租户，无 tenant_id | 面向单校部署，无多租户隔离需求 | 【已确认】 |
| D-11 | JSON 结构存 `TEXT` | 抽取标签、建议指标以 JSON 文本保存，由应用序列化；未使用 MySQL JSON 类型 | 【现状/待确认】 |
| D-12 | 业务编号独立于主键 | `task_id`、`suggestion_no` 为可读业务号，并建唯一键 | 【已确认】 |
| D-13 | 审计与调用日志只追加 | 不提供更新/删除入口；保留期限待定 | 【假设】 |
| D-14 | 运行期幂等建表 | `llm_model`、`adjustment_plan_item` 由服务启动时 `CREATE TABLE IF NOT EXISTS` | 【已确认】 |

### 1.1 关于“不建物理外键”的前提说明

本项目属于明确的服务自治前提：四个库分属不同服务，跨库引用（如 `ai_db.extraction_result.job_id → data_db.job_sample.id`）在物理上不可建 FK；同库内为避免初始化顺序、批量导入与历史数据迁移受阻，也统一不使用物理 FK。

**替代机制：**

1. 唯一键与联合唯一键承担幂等与重复约束（如岗位唯一键、用户角色唯一键）。
2. 应用层在写入前校验引用存在性（课程-技能点、用户-角色）。
3. 删除采用“先校验引用，再删除”或状态位；不做级联删除。
4. 定期核对脚本检查孤儿记录（待建设）。

**风险：** 应用校验遗漏会产生孤儿数据；跨库删除无级联保障。已登记为 TBD-08，需在后续版本决定是否对同库关系补 FK。

### 1.2 关于金额与 JSON 的规范偏差

- **金额：** 现状以 `INT` 存“K/月”，满足展示与统计，但不满足“金额以最小货币单位 DECIMAL 存储”的通用规范。若后续需要精确到元/分或跨币种，应迁移为 `DECIMAL(10,2)` 并保留单位说明。当前标记为【待确认】，不阻塞现有业务。
- **JSON：** 项目以 Java 8 与 TEXT 字段实现 JSON 序列化，避免驱动/ORM 对 JSON 类型的差异。代价是无法用数据库函数检索 JSON；当前无此查询需求。

## 2. 逻辑模型

### 2.1 实体清单

| 实体编号 | 实体 | 物理表 | 所属库 | 职责 | 对应需求 |
|---|---|---|---|---|---|
| ENTITY-01 | 用户 | `sys_user` | user_db | 登录账号与状态 | REQ-01、REQ-41 |
| ENTITY-02 | 角色 | `sys_role` | user_db | 角色定义 | REQ-02、REQ-41、REQ-42 |
| ENTITY-03 | 用户角色 | `sys_user_role` | user_db | 用户与角色多对多 | REQ-41 |
| ENTITY-04 | 审计日志 | `sys_audit_log` | user_db | 关键操作留痕 | REQ-42、REQ-43 |
| ENTITY-05 | 企业 | `company` | data_db | 企业主数据 | REQ-03、REQ-12 |
| ENTITY-06 | 岗位样本 | `job_sample` | data_db | 可追溯岗位记录 | REQ-03~REQ-15 |
| ENTITY-07 | 数据溯源 | `data_source_trace` | data_db | 来源网址与入库时间 | REQ-07 |
| ENTITY-08 | 技能标签 | `skill_tag` | data_db | 归一化标签 | REQ-13、REQ-17~19 |
| ENTITY-09 | 岗位技能关联 | `job_skill` | data_db | 岗位与标签多对多 | REQ-13、REQ-17~19 |
| ENTITY-10 | 导入任务 | `import_task` | data_db | 导入执行与失败报告 | REQ-03~REQ-06 |
| ENTITY-11 | 质量问题 | `data_quality_issue` | data_db | 数据质量问题明细 | REQ-08 |
| ENTITY-12 | 采集任务 | `collection_task` | data_db | 周期采集配置 | REQ-09~REQ-11 |
| ENTITY-13 | 报告任务 | `report_task` | data_db | 异步报告生成 | REQ-35、REQ-36 |
| ENTITY-14 | 抽取结果 | `extraction_result` | ai_db | 岗位三类标签 | REQ-17~REQ-19 |
| ENTITY-15 | 需求预测 | `demand_forecast` | ai_db | 区域/月份预测值 | REQ-20、REQ-21 |
| ENTITY-16 | 大模型调用日志 | `llm_call_log` | ai_db | 调用耗时与结果 | REQ-40 |
| ENTITY-17 | 大模型配置 | `llm_model` | ai_db | 模型接入与当前模型 | REQ-37~REQ-40 |
| ENTITY-18 | 课程 | `course` | curriculum_db | 课程主数据 | REQ-24、REQ-25 |
| ENTITY-19 | 知识技能点 | `knowledge_point` | curriculum_db | 知识点/技能点 | REQ-24、REQ-26 |
| ENTITY-20 | 课程技能点关联 | `course_knowledge_point` | curriculum_db | 课程与技能点多对多 | REQ-26 |
| ENTITY-21 | 项目案例 | `project_case` | curriculum_db | 与课程关联的案例 | REQ-27 |
| ENTITY-22 | 专业调整建议（审批） | `major_adjustment_advice` | curriculum_db | 提案与审批结论 | REQ-28~REQ-30 |
| ENTITY-23 | 年度调整建议 | `adjustment_plan_item` | curriculum_db | 分类建议与采纳状态 | REQ-31~REQ-34 |

### 2.2 关系清单

| 关系编号 | 关系 | 基数 | 关联字段 | 物理约束 | 对应需求 |
|---|---|---|---|---|---|
| REL-01 | 用户—角色 | N:1 | `sys_user_role.user_id → sys_user.id` | 无 FK；唯一键 `(user_id, role_id)` | REQ-41 |
| REL-02 | 用户角色—角色 | N:1 | `sys_user_role.role_id → sys_role.id` | 无 FK | REQ-41 |
| REL-03 | 岗位—企业 | N:1 | `job_sample.company_id → company.id`（可空） | 无 FK | REQ-03、REQ-12 |
| REL-04 | 岗位—溯源 | 1:N | `data_source_trace.job_id → job_sample.id` | 无 FK；索引 | REQ-07 |
| REL-05 | 岗位—技能关联 | 1:N | `job_skill.job_id → job_sample.id` | 无 FK；联合唯一键 | REQ-13 |
| REL-06 | 技能关联—标签 | N:1 | `job_skill.skill_tag_id → skill_tag.id` | 无 FK；索引 | REQ-13 |
| REL-07 | 质量问题—岗位 | N:1 | `data_quality_issue.job_id → job_sample.id`（可空） | 无 FK | REQ-08 |
| REL-08 | 质量问题—导入任务 | N:1 | `data_quality_issue.task_id → import_task.task_id` | 无 FK；字符串业务号 | REQ-08 |
| REL-09 | 岗位—导入任务 | N:1 | `job_sample.import_task_id → import_task.task_id` | 无 FK；索引 | REQ-03 |
| REL-10 | 抽取结果—岗位 | 1:1 | `ai_db.extraction_result.job_id → data_db.job_sample.id` | 跨库，无 FK；应用校验 | REQ-17~REQ-19 |
| REL-11 | 课程技能点—课程 | N:1 | `course_knowledge_point.course_id → course.id` | 无 FK；联合唯一键 | REQ-26 |
| REL-12 | 课程技能点—技能点 | N:1 | `course_knowledge_point.point_id → knowledge_point.id` | 无 FK | REQ-26 |
| REL-13 | 案例—课程 | N:1 | `project_case.course_id → course.id`（可空） | 无 FK | REQ-27 |
| REL-14 | 建议—提交人 | N:1 | `major_adjustment_advice.submitter_id → user_db.sys_user.id` | 跨库，无 FK | REQ-28 |
| REL-15 | 建议—审批人 | N:1 | `major_adjustment_advice.approver_id → user_db.sys_user.id`（可空） | 跨库，无 FK | REQ-29 |
| REL-16 | 调用日志—模型 | N:1 | `llm_call_log.model_name → llm_model.model_code` | 无 FK；弱关联（日志可保留已删模型） | REQ-40 |
| REL-17 | 导入任务—创建人 | N:1 | `import_task.created_by → sys_user.username` | 跨库，弱关联 | REQ-03 |
| REL-18 | 报告任务—创建人 | N:1 | `report_task.created_by → sys_user.username` | 跨库，弱关联 | REQ-36 |

## 3. 表设计（物理）

> 以下 DDL 与当前库结构一致；`llm_model`、`adjustment_plan_item` 由服务启动时幂等创建。所有表使用 InnoDB、utf8mb4。

### 3.1 user_db

```sql
CREATE TABLE sys_user (
  id BIGINT NOT NULL AUTO_INCREMENT COMMENT '主键',
  username VARCHAR(64) NOT NULL COMMENT '登录名',
  password_hash VARCHAR(100) NOT NULL COMMENT 'BCrypt 密码哈希',
  real_name VARCHAR(64) DEFAULT NULL COMMENT '姓名',
  email VARCHAR(128) DEFAULT NULL COMMENT '邮箱',
  status TINYINT NOT NULL DEFAULT 1 COMMENT '1启用 0禁用',
  created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
  updated_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  PRIMARY KEY (id),
  UNIQUE KEY uk_username (username)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='用户表';

CREATE TABLE sys_role (
  id BIGINT NOT NULL AUTO_INCREMENT,
  role_code VARCHAR(50) NOT NULL COMMENT '角色编码',
  role_name VARCHAR(50) NOT NULL COMMENT '角色名称',
  description VARCHAR(200) DEFAULT NULL,
  created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
  PRIMARY KEY (id),
  UNIQUE KEY uk_role_code (role_code)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='角色表';

CREATE TABLE sys_user_role (
  id BIGINT NOT NULL AUTO_INCREMENT,
  user_id BIGINT NOT NULL,
  role_id BIGINT NOT NULL,
  PRIMARY KEY (id),
  UNIQUE KEY uk_user_role (user_id, role_id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='用户角色表';

CREATE TABLE sys_audit_log (
  id BIGINT NOT NULL AUTO_INCREMENT,
  user_id BIGINT DEFAULT NULL,
  username VARCHAR(64) DEFAULT NULL,
  action VARCHAR(100) NOT NULL COMMENT '操作动作',
  target VARCHAR(200) DEFAULT NULL COMMENT '操作对象',
  result VARCHAR(20) NOT NULL COMMENT 'SUCCESS/FAILED',
  ip VARCHAR(64) DEFAULT NULL,
  detail TEXT,
  created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
  PRIMARY KEY (id),
  KEY idx_audit_created (created_at),
  KEY idx_audit_username (username)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='操作日志表';
```

### 3.2 data_db

```sql
CREATE TABLE company (
  id BIGINT NOT NULL AUTO_INCREMENT,
  company_name VARCHAR(200) NOT NULL COMMENT '企业名称',
  industry VARCHAR(100) DEFAULT NULL COMMENT '所属行业',
  scale VARCHAR(50) DEFAULT NULL COMMENT '企业规模',
  address VARCHAR(500) DEFAULT NULL COMMENT '企业地址',
  intro TEXT COMMENT '企业简介',
  created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
  updated_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  PRIMARY KEY (id),
  UNIQUE KEY uk_company_name (company_name)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='企业表';

CREATE TABLE job_sample (
  id BIGINT NOT NULL AUTO_INCREMENT,
  job_title VARCHAR(150) NOT NULL COMMENT '岗位名称',
  company_name VARCHAR(200) NOT NULL COMMENT '企业名称',
  company_id BIGINT DEFAULT NULL,
  industry VARCHAR(100) DEFAULT NULL,
  company_scale VARCHAR(50) DEFAULT NULL,
  company_address VARCHAR(500) DEFAULT NULL,
  region VARCHAR(50) NOT NULL DEFAULT '其他' COMMENT '省/直辖市',
  city VARCHAR(50) DEFAULT NULL COMMENT '城市',
  salary_min INT DEFAULT NULL COMMENT '最低月薪(K)',
  salary_max INT DEFAULT NULL COMMENT '最高月薪(K)',
  education VARCHAR(50) DEFAULT NULL COMMENT '学历要求',
  experience VARCHAR(50) DEFAULT NULL COMMENT '经验要求',
  headcount VARCHAR(50) DEFAULT NULL COMMENT '招聘人数',
  job_status VARCHAR(20) DEFAULT NULL COMMENT '职位状态',
  description TEXT COMMENT '职位描述',
  skills_required TEXT COMMENT '技能要求原文',
  source_url VARCHAR(500) DEFAULT NULL COMMENT '原始链接',
  source_channel VARCHAR(30) NOT NULL DEFAULT 'IMPORT' COMMENT 'IMPORT/API/CRAWLER',
  collected_at DATETIME NOT NULL COMMENT '采集/发布时间',
  import_task_id VARCHAR(64) DEFAULT NULL,
  quality_score DECIMAL(5,2) DEFAULT NULL COMMENT '质量评分 0-100',
  data_status TINYINT NOT NULL DEFAULT 0 COMMENT '0未清洗 1已清洗 2已抽取',
  deleted TINYINT NOT NULL DEFAULT 0,
  created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
  updated_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  PRIMARY KEY (id),
  UNIQUE KEY uk_job_identity (company_name, job_title, collected_at),
  KEY idx_job_region (region),
  KEY idx_job_collected (collected_at),
  KEY idx_job_task (import_task_id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='岗位样本表';

CREATE TABLE data_source_trace (
  id BIGINT NOT NULL AUTO_INCREMENT,
  job_id BIGINT DEFAULT NULL,
  original_url VARCHAR(500) NOT NULL COMMENT '来源网址',
  channel VARCHAR(50) DEFAULT NULL COMMENT '渠道（爬虫/接口/导入）',
  trace_time DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '入库时间',
  created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
  PRIMARY KEY (id),
  KEY idx_trace_job (job_id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='数据来源溯源表';

CREATE TABLE skill_tag (
  id BIGINT NOT NULL AUTO_INCREMENT,
  tag_name VARCHAR(100) NOT NULL COMMENT '标签名称',
  tag_type VARCHAR(20) NOT NULL COMMENT 'SKILL/TOOL/ABILITY',
  first_seen_at DATETIME DEFAULT NULL,
  created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
  PRIMARY KEY (id),
  UNIQUE KEY uk_tag_name (tag_name)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='技能标签表';

CREATE TABLE job_skill (
  id BIGINT NOT NULL AUTO_INCREMENT,
  job_id BIGINT NOT NULL,
  skill_tag_id BIGINT NOT NULL,
  PRIMARY KEY (id),
  UNIQUE KEY uk_job_skill (job_id, skill_tag_id),
  KEY idx_skill_tag (skill_tag_id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='岗位-技能关联表';

CREATE TABLE import_task (
  id BIGINT NOT NULL AUTO_INCREMENT,
  task_id VARCHAR(64) NOT NULL COMMENT '业务任务号',
  file_name VARCHAR(255) NOT NULL,
  status VARCHAR(20) NOT NULL DEFAULT 'PENDING' COMMENT 'PENDING/RUNNING/DONE/FAILED',
  total INT NOT NULL DEFAULT 0,
  success INT NOT NULL DEFAULT 0,
  failed INT NOT NULL DEFAULT 0,
  error_report LONGTEXT,
  created_by VARCHAR(64) DEFAULT NULL,
  created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
  finished_at DATETIME DEFAULT NULL,
  PRIMARY KEY (id),
  UNIQUE KEY uk_task_id (task_id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='导入任务表';

CREATE TABLE data_quality_issue (
  id BIGINT NOT NULL AUTO_INCREMENT,
  job_id BIGINT DEFAULT NULL,
  task_id VARCHAR(64) DEFAULT NULL,
  issue_type VARCHAR(50) NOT NULL COMMENT 'MISSING/DUPLICATE/ABNORMAL',
  detail VARCHAR(500) DEFAULT NULL,
  created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
  PRIMARY KEY (id),
  KEY idx_issue_task (task_id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='数据质量问题表';

CREATE TABLE collection_task (
  id BIGINT NOT NULL AUTO_INCREMENT,
  task_name VARCHAR(100) NOT NULL,
  source_type VARCHAR(30) NOT NULL DEFAULT 'FILE' COMMENT 'FILE/IMPORT/HTTP/API/CRAWLER',
  source_url VARCHAR(500) DEFAULT NULL,
  cron_expr VARCHAR(50) DEFAULT NULL,
  status TINYINT NOT NULL DEFAULT 1 COMMENT '1启用 0停用',
  last_run_at DATETIME DEFAULT NULL,
  last_status VARCHAR(20) DEFAULT NULL COMMENT 'SUCCESS/FAILED',
  last_message VARCHAR(500) DEFAULT NULL COMMENT '最近一次执行结果说明',
  created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
  updated_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  PRIMARY KEY (id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='采集任务配置表';

CREATE TABLE report_task (
  id BIGINT NOT NULL AUTO_INCREMENT,
  task_id VARCHAR(64) NOT NULL COMMENT '业务任务号',
  report_type VARCHAR(30) NOT NULL COMMENT 'ANNUAL_DOCX/ANNUAL_PDF/ANNUAL_EXCEL',
  report_year INT NOT NULL,
  status VARCHAR(20) NOT NULL DEFAULT 'PENDING' COMMENT 'PENDING/RUNNING/DONE/FAILED',
  file_name VARCHAR(255) DEFAULT NULL,
  file_size BIGINT NOT NULL DEFAULT 0,
  message VARCHAR(500) DEFAULT NULL,
  created_by VARCHAR(64) DEFAULT NULL,
  created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
  finished_at DATETIME DEFAULT NULL,
  PRIMARY KEY (id),
  UNIQUE KEY uk_report_task_id (task_id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='报告生成任务表';
```

### 3.3 ai_db

```sql
CREATE TABLE extraction_result (
  id BIGINT NOT NULL AUTO_INCREMENT,
  job_id BIGINT NOT NULL,
  skills TEXT COMMENT '技能 JSON 数组',
  tools TEXT COMMENT '工具 JSON 数组',
  abilities TEXT COMMENT '工程能力 JSON 数组',
  raw_response LONGTEXT,
  model_name VARCHAR(100) DEFAULT NULL,
  status VARCHAR(20) NOT NULL DEFAULT 'SUCCESS',
  created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
  updated_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  PRIMARY KEY (id),
  UNIQUE KEY uk_extract_job (job_id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='抽取结果表';

CREATE TABLE demand_forecast (
  id BIGINT NOT NULL AUTO_INCREMENT,
  region VARCHAR(50) NOT NULL,
  forecast_month VARCHAR(7) NOT NULL COMMENT 'YYYY-MM',
  demand_value INT NOT NULL,
  model_name VARCHAR(100) NOT NULL,
  created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
  PRIMARY KEY (id),
  UNIQUE KEY uk_forecast (region, forecast_month, model_name)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='预测结果表';

CREATE TABLE llm_call_log (
  id BIGINT NOT NULL AUTO_INCREMENT,
  request_id VARCHAR(64) NOT NULL,
  model_name VARCHAR(100) NOT NULL,
  prompt_tokens INT DEFAULT NULL,
  completion_tokens INT DEFAULT NULL,
  latency_ms INT DEFAULT NULL,
  status VARCHAR(20) NOT NULL,
  error_message VARCHAR(500) DEFAULT NULL,
  created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
  PRIMARY KEY (id),
  KEY idx_llm_created (created_at)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='大模型调用日志表';

-- 运行期由 ai-service 的 LlmModelInitializer 幂等创建
CREATE TABLE llm_model (
  id BIGINT NOT NULL AUTO_INCREMENT COMMENT '主键',
  model_name VARCHAR(100) NOT NULL COMMENT '展示名称',
  model_code VARCHAR(100) NOT NULL COMMENT 'API 模型标识',
  provider VARCHAR(100) NOT NULL DEFAULT '自定义' COMMENT '服务商',
  access_type VARCHAR(32) NOT NULL DEFAULT 'API' COMMENT '接入方式',
  base_url VARCHAR(255) NOT NULL COMMENT 'API Base URL',
  api_key VARCHAR(255) DEFAULT NULL COMMENT 'API Key',
  status TINYINT NOT NULL DEFAULT 1 COMMENT '1启用 0停用',
  is_current TINYINT NOT NULL DEFAULT 0 COMMENT '1当前使用',
  remark VARCHAR(255) DEFAULT NULL COMMENT '备注',
  created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
  updated_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  PRIMARY KEY (id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='大模型接入配置';
```

### 3.4 curriculum_db

```sql
CREATE TABLE course (
  id BIGINT NOT NULL AUTO_INCREMENT,
  course_code VARCHAR(50) NOT NULL,
  course_name VARCHAR(200) NOT NULL,
  credit DECIMAL(3,1) DEFAULT NULL,
  description TEXT,
  created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
  updated_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  PRIMARY KEY (id),
  UNIQUE KEY uk_course_code (course_code)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='课程表';

CREATE TABLE knowledge_point (
  id BIGINT NOT NULL AUTO_INCREMENT,
  point_name VARCHAR(200) NOT NULL,
  point_type VARCHAR(20) NOT NULL COMMENT 'KNOWLEDGE/SKILL',
  description TEXT,
  created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
  PRIMARY KEY (id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='知识技能点表';

CREATE TABLE course_knowledge_point (
  id BIGINT NOT NULL AUTO_INCREMENT,
  course_id BIGINT NOT NULL,
  point_id BIGINT NOT NULL,
  PRIMARY KEY (id),
  UNIQUE KEY uk_course_point (course_id, point_id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='课程-知识技能点关联表';

CREATE TABLE project_case (
  id BIGINT NOT NULL AUTO_INCREMENT,
  case_name VARCHAR(200) NOT NULL,
  course_id BIGINT DEFAULT NULL,
  difficulty VARCHAR(20) DEFAULT NULL COMMENT '基础/进阶/综合',
  resource_url VARCHAR(500) DEFAULT NULL,
  description TEXT,
  created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
  PRIMARY KEY (id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='项目案例表';

CREATE TABLE major_adjustment_advice (
  id BIGINT NOT NULL AUTO_INCREMENT,
  title VARCHAR(200) NOT NULL,
  content TEXT NOT NULL,
  submitter_id BIGINT NOT NULL,
  submitter_name VARCHAR(64) DEFAULT NULL,
  status VARCHAR(20) NOT NULL DEFAULT 'PENDING' COMMENT 'PENDING/APPROVED/REJECTED',
  approver_id BIGINT DEFAULT NULL,
  approver_name VARCHAR(64) DEFAULT NULL,
  comment VARCHAR(500) DEFAULT NULL,
  approved_at DATETIME DEFAULT NULL,
  created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
  updated_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  PRIMARY KEY (id),
  KEY idx_advice_status (status)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='专业调整建议表';

-- 运行期由 curriculum-service 的 AdjustmentPlanInitializer 幂等创建
CREATE TABLE adjustment_plan_item (
  id BIGINT NOT NULL AUTO_INCREMENT COMMENT '主键',
  plan_year INT NOT NULL COMMENT '计划年度',
  suggestion_no VARCHAR(32) NOT NULL COMMENT '建议编号',
  category VARCHAR(16) NOT NULL COMMENT 'DIRECTION/COURSE/SKILL',
  priority VARCHAR(16) NOT NULL COMMENT 'HIGH/MEDIUM/LOW',
  title VARCHAR(200) NOT NULL COMMENT '建议标题',
  basis VARCHAR(1000) NOT NULL COMMENT '建议依据',
  metrics VARCHAR(500) DEFAULT NULL COMMENT '指标 JSON',
  owner_unit VARCHAR(200) DEFAULT NULL COMMENT '责任单位',
  impact_scope VARCHAR(200) DEFAULT NULL COMMENT '影响范围',
  status VARCHAR(16) NOT NULL DEFAULT 'PENDING' COMMENT 'PENDING/ADOPTED',
  adopted_by VARCHAR(64) DEFAULT NULL COMMENT '采纳人',
  adopted_at DATETIME DEFAULT NULL COMMENT '采纳时间',
  generated_at DATETIME NOT NULL COMMENT '生成时间',
  created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
  updated_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  PRIMARY KEY (id),
  UNIQUE KEY uk_plan_suggestion (plan_year, suggestion_no)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='专业动态调整建议';
```

## 4. 数据字典（枚举与码值）

| 字段 | 允许值 | 含义 | 变更影响 |
|---|---|---|---|
| `sys_user.status` | 0 / 1 | 禁用 / 启用 | 登录校验 |
| `sys_audit_log.result` | SUCCESS / FAILED | 操作结果 | 审计查询 |
| `job_sample.source_channel` | IMPORT / API / CRAWLER | 导入 / 接口 / 采集 | 溯源统计 |
| `job_sample.data_status` | 0 / 1 / 2 | 未清洗 / 已清洗 / 已抽取 | 抽取进度 |
| `job_sample.deleted` | 0 / 1 | 正常 / 已删除（软删） | 查询过滤 |
| `skill_tag.tag_type` | SKILL / TOOL / ABILITY | 技能 / 工具 / 工程能力 | 标签统计与抽取 |
| `import_task.status` | PENDING / RUNNING / DONE / FAILED | 任务状态 | 前端轮询 |
| `data_quality_issue.issue_type` | MISSING / DUPLICATE / ABNORMAL | 缺失 / 重复 / 异常 | 质量报告 |
| `collection_task.source_type` | FILE / IMPORT / HTTP / API / CRAWLER | 来源类型 | 采集执行 |
| `collection_task.status` | 0 / 1 | 停用 / 启用 | 调度 |
| `collection_task.last_status` | SUCCESS / FAILED / NULL | 最近执行结果 | 运营排查 |
| `report_task.report_type` | ANNUAL_DOCX / ANNUAL_PDF / ANNUAL_EXCEL | 报告格式 | 生成与下载 |
| `report_task.status` | PENDING / RUNNING / DONE / FAILED | 任务状态 | 下载前置校验 |
| `extraction_result.status` | SUCCESS / FAILED | 抽取结果 | 进度与重试 |
| `llm_call_log.status` | SUCCESS / FAILED | 调用结果 | 用量与故障分析 |
| `llm_model.access_type` | API 等 | 接入方式 | 路由 |
| `llm_model.status` | 0 / 1 | 停用 / 启用 | 可选模型 |
| `llm_model.is_current` | 0 / 1 | 非当前 / 当前 | 抽取路由 |
| `knowledge_point.point_type` | KNOWLEDGE / SKILL | 知识点 / 技能点 | 课程绑定 |
| `major_adjustment_advice.status` | PENDING / APPROVED / REJECTED | 待审批 / 通过 / 驳回 | 状态机、不可逆 |
| `adjustment_plan_item.category` | DIRECTION / COURSE / SKILL | 专业方向 / 课程内容 / 技能新点 | 分类统计 |
| `adjustment_plan_item.priority` | HIGH / MEDIUM / LOW | 高 / 中 / 低 | 展示与筛选 |
| `adjustment_plan_item.status` | PENDING / ADOPTED | 待确认 / 已采纳 | 评审进度 |

## 5. 生命周期与保留策略

| 数据 | 创建 | 更新 | 删除/归档 | 保留期 |
|---|---|---|---|---|
| 岗位样本 | 导入/采集 | 重复导入覆盖；抽取状态回写 | 软删 `deleted=1` | 【待确认】TBD-06 |
| 溯源记录 | 入库时写入 | 不更新 | 随样本归档 | 【待确认】 |
| 导入/报告任务 | 任务创建 | 状态与计数更新 | 不主动删除 | 【待确认】 |
| 报告文件 | 任务完成落盘 | — | 手工清理（暂无策略） | 【待确认】 |
| 抽取结果 | 抽取成功/失败 | 重抽覆盖 | 随岗位归档 | 【待确认】 |
| 预测结果 | 预测生成 | 唯一键覆盖/追加 | 不主动删除 | 【待确认】 |
| 模型配置 | 管理员新增 | 管理员编辑/切换 | 物理删除（当前模型禁止） | 长期 |
| 审计日志 | 关键操作 | 不可更新 | 不可删除 | 【待确认】TBD-04 |
| 调用日志 | 每次模型调用 | 不可更新 | 不主动删除 | 【待确认】 |
| 年度建议 | 首次生成 | 重新生成更新（保留采纳） | 不主动删除 | 长期（按年度归档待定） |

## 6. 安全与敏感数据

| 数据 | 现状 | 风险 | 建议 |
|---|---|---|---|
| 密码 | BCrypt 哈希 | 低 | 保持；禁止可逆加密 |
| 模型 API Key | `llm_model.api_key` 明文；历史上 `application.yml` 存在明文 Key | 高 | 迁移环境变量/密钥管理；库内加密；接口只返回“已配置” |
| 用户邮箱/姓名 | 明文 | 中 | 仅授权角色可见；导出需审批 |
| 企业地址/联系人信息 | 明文 | 中 | 按数据授权范围使用 |
| 审计日志 | 只读 | 低 | 限制查询角色；保留期限待定 |
| 内部令牌 | 配置项 | 中 | 环境变量注入；定期轮换 |

## 7. 并发、约束与索引设计

### 7.1 关键并发控制

| 场景 | 机制 | 结果 |
|---|---|---|
| 建议审批 | `UPDATE ... WHERE id=? AND status='PENDING'` | 影响行数 0 → 409 |
| 年度建议采纳 | `UPDATE ... WHERE id=? AND status='PENDING'` | 影响行数 0 → 409 |
| 年度建议生成 | 唯一键 `(plan_year, suggestion_no)` + 先查后写 | 并发首次生成可能撞唯一键 → 需捕获并重试【建议】 |
| 岗位去重 | 唯一键 `(company_name, job_title, collected_at)` | 重复导入覆盖更新 |
| 用户角色 | 唯一键 `(user_id, role_id)` | 防止重复分配 |
| 课程技能点 | 唯一键 `(course_id, point_id)` | 防止重复绑定 |
| 当前模型 | 应用事务内先清除再设置 | 数据库未强制唯一，存在并发窗口【待确认】 |
| 报告下载 | 校验 `status='DONE'` | 未完成返回 409 |

### 7.2 索引现状与建议

| 表 | 现有索引 | 建议补充 | 依据 |
|---|---|---|---|
| `sys_user` | uk_username | — | 登录 |
| `sys_audit_log` | idx_created、idx_username | `(username, created_at)` 复合【建议】 | 组合条件查询 |
| `job_sample` | uk_identity、idx_region、idx_collected、idx_task | `(region, collected_at)`、`(deleted, collected_at)`【建议】 | 看板区域+时间、列表过滤 |
| `data_source_trace` | idx_job | — | 单岗位溯源 |
| `job_skill` | uk_job_skill、idx_skill_tag | — | 双向查询 |
| `import_task` | uk_task_id | `created_at`【建议】 | 任务历史分页 |
| `data_quality_issue` | idx_task | — | 按任务查询 |
| `report_task` | uk_task_id | `(created_at)`【建议】 | 任务列表 |
| `extraction_result` | uk_extract_job | `status`【建议】 | 进度统计（若按状态扫描） |
| `demand_forecast` | uk_forecast | — | 区域+月份 |
| `llm_call_log` | idx_created | — | 用量统计 |
| `llm_model` | 无 | `model_code` 唯一【建议】 | 防止重复模型标识 |
| `course` | uk_course_code | — | 编码唯一 |
| `course_knowledge_point` | uk_course_point | `point_id`【建议】 | 反查课程 |
| `project_case` | 无 | `course_id`【建议】 | 课程详情关联 |
| `major_adjustment_advice` | idx_status | `(status, created_at)`【建议】 | 状态分页 |
| `adjustment_plan_item` | uk_plan_suggestion | `(plan_year, category)`【建议】 | 按年度分类查询 |

> 标注【建议】的索引为查询模式推导，落地前需结合实际执行计划验证；未纳入当前已建结构。

### 7.3 容量与性能假设

- 当前验证基线约 5,000 条岗位样本、200+ 企业；设计按万级岗位、十万级关联记录评估。
- 统计接口以聚合查询为主，需避免全表扫描；建议在数据量增长后引入汇总表或缓存（属实现优化，不在本设计强制）。
- 文本大字段（`description`、`error_report`、`raw_response`）不参与常规索引；查询时避免 `SELECT *`。

## 8. 追溯矩阵

### 8.1 实体 → 需求

| 实体 | 需求 |
|---|---|
| ENTITY-01/02/03 | REQ-01、REQ-02、REQ-41 |
| ENTITY-04 | REQ-42、REQ-43 |
| ENTITY-05/06 | REQ-03~REQ-15 |
| ENTITY-07 | REQ-07 |
| ENTITY-08/09 | REQ-13、REQ-17~REQ-19 |
| ENTITY-10/11 | REQ-03~REQ-06、REQ-08 |
| ENTITY-12 | REQ-09~REQ-11 |
| ENTITY-13 | REQ-35、REQ-36 |
| ENTITY-14 | REQ-17~REQ-19 |
| ENTITY-15 | REQ-20、REQ-21 |
| ENTITY-16 | REQ-40 |
| ENTITY-17 | REQ-37~REQ-40 |
| ENTITY-18/19/20 | REQ-24~REQ-26 |
| ENTITY-21 | REQ-27 |
| ENTITY-22 | REQ-28~REQ-30 |
| ENTITY-23 | REQ-31~REQ-34 |

### 8.2 关系 → 需求

| 关系 | 需求 |
|---|---|
| REL-01/02 | REQ-41 |
| REL-03~REL-09 | REQ-03、REQ-07、REQ-08、REQ-12、REQ-13 |
| REL-10 | REQ-17~REQ-19 |
| REL-11~REL-13 | REQ-24、REQ-26、REQ-27 |
| REL-14/15 | REQ-28、REQ-29 |
| REL-16 | REQ-40 |
| REL-17/18 | REQ-03、REQ-36 |

### 8.3 约束/索引 → 依据

| 约束/索引 | 依据 |
|---|---|
| `uk_job_identity` | INV-02、AC-DATA-03 |
| `uk_task_id`、`uk_report_task_id` | REQ-03、REQ-36 |
| `uk_user_role` | REQ-41 |
| `uk_course_point` | REQ-26 |
| `uk_plan_suggestion` | INV-07、AC-ADJ-01/02 |
| `uk_forecast` | REQ-20 |
| `idx_audit_*` | REQ-42 |
| `uq_llm_model.model_code`【建议】 | REQ-38、INV-09 |
| `job_sample` 组合索引【建议】 | REQ-12、REQ-14、AC-BIZ-01 |

## 9. 待确认与设计风险

| 编号 | 事项 | 影响 | 建议 |
|---|---|---|---|
| TBD-01 | 同库关系是否补物理外键 | 数据完整性与迁移成本 | 先以应用校验+核对脚本运行，评估后再补 |
| TBD-02 | 金额 `INT(K)` 是否迁移 `DECIMAL(10,2)` | 报表精度与迁移改造 | 明确展示口径后决策 |
| TBD-03 | `llm_model.api_key` 加密方案 | 安全合规 | 环境变量 + 库内加密；接口永不回传 |
| TBD-04 | 审计/调用日志保留期 | 存储增长与合规 | 明确 6/12/24 个月策略 |
| TBD-05 | 当前模型唯一性的数据库约束 | 并发切换一致性 | 增加生成列唯一索引或切换锁 |
| TBD-06 | 岗位样本与报告文件保留期 | 容量与合规 | 制定归档/清理任务 |
| TBD-07 | JSON 字段是否改用 MySQL JSON 类型 | 查询能力与迁移 | 当前无 JSON 检索需求，暂不迁移 |
| TBD-08 | 年度建议并发首次生成的唯一键冲突 | 偶发 500 | 捕获 DuplicateKey 并重查返回 |
| TBD-09 | `job_sample` 唯一键含 `collected_at` 秒级时间 | 同日同岗不同时刻会重复 | 评估改用“企业+岗位+日期”唯一键 |
