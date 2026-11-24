# 部署与运维手册

> 适用于本地开发与演示环境。生产部署需替换所有默认口令、JWT 密钥与内网令牌，并关闭开发期性能参数。

## 1. 环境要求

| 组件 | 版本 | 说明 |
|---|---|---|
| JDK | 8+（实测 17） | 源码级别 Java 8 |
| Maven | 3.8+ | 父工程统一管理依赖 |
| Node.js | 18+ | 前端构建 |
| Docker Desktop | 24+ | 基础设施容器 |

## 2. 基础设施

```powershell
cd deploy
docker compose up -d
docker ps
```

端口映射（本机适配，详见 `docs/plan.md` 5.1）：

| 组件 | 宿主端口 → 容器端口 |
|---|---|
| MySQL | 3307 → 3306 |
| Redis | 6379 → 6379 |
| Nacos HTTP | 18848 → 8848 |
| Nacos gRPC | 19848 → 9848、19849 → 9849 |
| RocketMQ NameServer | 19876 → 9876 |
| RocketMQ Broker | 10911 → 10911 |
| Elasticsearch | 19200 → 9200 |

首次启动会执行 `deploy/mysql/init/01-schema.sql`，创建四个库、表结构与种子数据。

### 2.1 预创建 RocketMQ Topic

首次部署或清空 Broker 数据后执行，避免消费者首次订阅等待路由：

```powershell
docker exec rcyc-rocketmq-broker sh -c "/home/rocketmq/rocketmq-4.9.7/bin/mqadmin updateTopic -n rocketmq-namesrv:9876 -c DefaultCluster -t job-data-import-topic -r 4 -w 4"
docker exec rcyc-rocketmq-broker sh -c "/home/rocketmq/rocketmq-4.9.7/bin/mqadmin updateTopic -n rocketmq-namesrv:9876 -c DefaultCluster -t job-extract-topic -r 4 -w 4"
```

### 2.2 镜像拉取

Docker Hub 不可达时，可用国内镜像源拉取后重打标签：

```powershell
docker pull docker.m.daocloud.io/library/mysql:8.0.36
docker tag docker.m.daocloud.io/library/mysql:8.0.36 mysql:8.0.36
# 其余镜像同理：redis:7.2-alpine、nacos/nacos-server:v2.2.3、
# apache/rocketmq:4.9.7、elasticsearch:7.17.15
```

### 2.3 应用容器化（可选）

5 个 Java 服务均提供 `Dockerfile`（基础镜像 `eclipse-temurin:17-jre`，构建上下文为对应模块目录）。先构建 JAR，再随 compose 一起构建镜像：

```powershell
cd backend
mvn -DskipTests package
cd ..\frontend
npm run build
cd ..\deploy
docker compose up -d --build
```

compose 会额外拉起 `user-service`、`data-service`、`ai-service`、`curriculum-service`、`gateway-service` 与 `nginx`：

- 容器内通过服务名访问基础设施：`mysql:3306`、`nacos:8848`、`redis:6379`、`elasticsearch:9200`、`rocketmq-namesrv:9876`（由 compose 环境变量覆盖 application.yml 中的 127.0.0.1 配置）。
- `Nacos` 注册地址由 `SPRING_CLOUD_NACOS_DISCOVERY_IP` 置空，容器自动上报自身 IP，网关可正确路由。
- 仅暴露 `gateway-service:8080` 与 `nginx:80`，其余服务只在 compose 网络内可见。
- `nginx` 托管 `frontend/dist` 并将 `/api/` 反向代理到 `gateway-service:8080`，入口为 `http://localhost/`。
- `data-service` 的导入与报告文件保存在命名卷 `data-service-files`（容器内 `/app/data`）。

> **RocketMQ 广播地址说明：** `deploy/rocketmq/broker.conf` 默认 `brokerIP1=127.0.0.1`，适用于宿主机进程直连。若需要在容器内使用导入/抽取异步链路，请把 `brokerIP1` 改为宿主机局域网 IP（如 `10.21.139.3`），并确认 10911 端口已映射、容器可访问该地址，然后重启 broker。登录、查询类接口与报告异步生成不经过 MQ，不受此限制。

## 3. 后端

```powershell
cd backend
mvn -DskipTests package
```

按顺序启动（每个 JAR 独立进程）：

```powershell
java -jar user-service/target/user-service-1.0.0.jar
java -jar data-service/target/data-service-1.0.0.jar
java -jar ai-service/target/ai-service-1.0.0.jar
java -jar curriculum-service/target/curriculum-service-1.0.0.jar
java -jar gateway-service/target/gateway-service-1.0.0.jar
```

服务端口：网关 8080、用户 8081、数据 8082、AI 8083、课程 8084。健康检查：`GET /actuator/health`。

## 4. 前端

```powershell
cd frontend
npm install
npm run dev      # http://127.0.0.1:5173，代理 /api → 8080
npm run build    # 产物在 dist/，可由部署目录 Nginx 托管
```

Nginx 配置见 `deploy/nginx/nginx.conf`，静态资源根目录指向 `frontend/dist`，`/api/` 反向代理到 `gateway-service:8080`（compose 内）。若在本机直接运行 Nginx（非容器），需把代理目标改为 `127.0.0.1:8080`。

### 4.1 局域网访问（其他设备登录）

1. 前端 dev server 已监听 `0.0.0.0:5173`，`/api` 由该进程代理到本机 `127.0.0.1:8080`，其他设备无需直连后端。
2. 放行入站端口（管理员 PowerShell，限本地子网）：

```powershell
New-NetFirewallRule -DisplayName 'RCYC Frontend 5173' -Direction Inbound -Action Allow `
  -Protocol TCP -LocalPort 5173 -Profile Any -RemoteAddress LocalSubnet
```

3. 查看本机局域网 IP 后，其他设备访问 `http://<本机IP>:5173`（当前环境为 `10.21.139.3`，校园网 DHCP 可能变化）。验证：`Invoke-WebRequest http://<本机IP>:5173/` 返回 200。
4. 若其他设备打不开：确认两端在同一网段、校园网未开启客户端隔离/AP 隔离；跨网段需改用 Nginx 统一入口并开通对应端口。
5. 生产环境不要直接暴露 5173，应使用 `npm run build` + Nginx 80/443 统一入口。

## 5. 演示账号

| 账号 | 角色 | 密码 |
|---|---|---|
| admin | SYS_ADMIN | Admin@123 |
| security | SEC_ADMIN | Admin@123 |
| auditor | AUDITOR | Admin@123 |
| leader | MAJOR_LEADER | Admin@123 |
| teacher | TEACHER | Admin@123 |
| analyst | ANALYST | Admin@123 |

## 6. 关键配置

| 配置项 | 位置 | 说明 |
|---|---|---|
| `rcyc.jwt.secret` | 各服务 `application.yml` | HS256 密钥，生产必须更换 |
| `rcyc.internal-token` | 各服务 `application.yml` | 服务间调用 `X-Internal-Token` |
| `rcyc.ai.llm.provider` | `ai-service/application.yml` | `rule`（本地词典）或 `openai`（OpenAI 兼容协议） |
| `rcyc.ai.llm.base-url` | `ai-service/application.yml` | OpenAI 兼容接口地址 |
| `rcyc.ai.llm.model` | `ai-service/application.yml` | 模型名，如 `deepseek-chat` |
| `rcyc.ai.llm.api-key` | `ai-service/application.yml` | `provider=openai` 时必填，为空则启动失败（Fail-Fast） |
| `rcyc.import.dir` | `data-service/application.yml` | 上传文件落盘目录，默认 `./data/import` |
| `rcyc.report.dir` | `data-service/application.yml` | 异步报告落盘目录，默认 `./data/report` |
| `rcyc.cache.ttl-seconds` | `data-service/application.yml` | Redis 热点缓存 TTL，默认 60 秒 |
| `rocketmq.producer.retry-times-when-send-failed` | `data-service/application.yml` | 消息发送失败重试次数，默认 3 |
| `rcyc.embedding.provider` | `data-service/application.yml` | `local`（默认，256 维离线向量）或 `openai`（兼容 `/embeddings`），切换后须重建索引 |
| `rcyc.collect.scan-ms` | `data-service/application.yml` | 采集任务扫描间隔，默认 60000ms |

当前接入：DeepSeek（`provider=openai`、`base-url=https://api.deepseek.com/v1`、`model=deepseek-chat`）。密钥内联在 `application.yml`，仅适用于本地演示；生产必须改为环境变量或 Nacos 配置中心，并在上线前轮换密钥。调用记录写入 `ai_db.llm_call_log`（含 token 与耗时）。切回本地词典只需把 `provider` 改为 `rule` 并重启 ai-service。

## 7. 安全与审计

- 网关统一校验 JWT，并注入 `X-User-Id`、`X-Username`、`X-Roles`；外部请求携带的同名头会被移除，防止伪造。
- `/internal/` 路径经网关直接返回 403；服务间调用必须携带内部令牌。
- 三员分立：系统管理员、安全管理员、审计员；关键操作写入 `sys_audit_log`，审计员只读。
- 生产环境务必更换数据库口令、JWT 密钥、内部令牌，并启用 HTTPS。

## 8. 开发期性能参数

`docker-compose.yml` 中 MySQL 使用 `--innodb-flush-log-at-trx-commit=0 --sync-binlog=0 --skip-log-bin --innodb-doublewrite=0`，用于满足 AC-DATA-01 的导入性能。**生产环境不得沿用**，应按数据安全要求恢复默认刷盘与双写。

## 9. 常见问题

1. **端口被 Windows 保留段占用**：`netsh interface ipv4 show excludedportrange protocol=tcp` 查看。9816–10015 可能覆盖 9848/9876/9200，本项目已改用 18848/19876/19200 映射。
2. **Nacos 客户端报 `Client not connected`**：确认 `server-addr=127.0.0.1:18848` 且宿主 19848 已映射；客户端按 +1000 推导 gRPC 端口。
3. **导入任务长时间 PENDING**：检查 `job-data-import-topic` 是否已创建、Broker 10911 是否可从宿主访问。
4. **全文检索 503**：检查 `http://127.0.0.1:19200/_cluster/health`，并确认索引已通过 `/api/v1/data/jobs/reindex` 重建。
5. **Docker 引擎未启动**：先启动 Docker Desktop，待 `docker info` 返回 ServerVersion 后再执行 compose。
