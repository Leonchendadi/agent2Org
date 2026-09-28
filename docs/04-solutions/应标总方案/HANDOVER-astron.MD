# HANDOVER · Astron Agent 现状事实

> **用途**：作为新会话讨论「Astron Agent 与替代方案对比」的输入材料
> **原则**：**只记录已证实的事实**（有代码/日志/数据库/浏览器实测证据），不做主观评价
> **日期**：2026-09-17
> **环境**：`100.78.54.115`（客户线下本地部署）

---

## 0. 一句话定位

Astron Agent 是**讯飞开源的企业级 Agentic Workflow 开发平台**（Apache 2.0），提供：多租户组织 → 企业空间 → 智能体/工作流 三层模型，配套 Web 控制台（含可视化工作流编排）、OIDC 单点登录、应用市场、插件与知识库接入能力。

---

## 1. 项目本体

| 项 | 值 |
|---|---|
| 仓库 | `iflytek/astron-agent`（本机 `/root/project/astron-agent`） |
| 许可证 | **Apache License 2.0** |
| 当前版本 | `v1.1.2-3-g5e758547`（`v1.1.2` 之后 3 个提交） |
| 最后提交 | `5e758547` · 2026-09-10 · *fix(workflow): reduce trace payload size for long conversations (#1675)* |
| 分支 | `main`（含**未提交**的本地修改：`console/backend/commons/...messages_*.properties`、`console/backend/hub/Dockerfile`、`PlatformAccountType.java`、`PlatformAccountService.java`、`docker/astronAgent/docker-compose.yaml`、`console/frontend/src/styles/classes.scss`） |

### 代码规模（本仓库统计）

| 语言 | 文件数 | 行数 |
|---|---|---|
| TypeScript (.ts/.tsx) | 697 | ~151,700 |
| Java | 1,458 | ~172,500 |
| Python | 819 | ~139,900 |
| Go | 54 | ~12,800 |
| **合计** | | **≈ 48 万行** |

### 主要模块与技术栈

| 模块 | 路径 | 语言/框架 | 职责 |
|---|---|---|---|
| 控制台前端 | `console/frontend` | React 18 + TS 5 + Vite 5 + AntD 5 + Tailwind + ReactFlow + Zustand | Web UI、工作流可视化编排 |
| 控制台后端 | `console/backend/{hub,toolkit,commons}` | Java 21 + Spring Boot 3.5.4 + MyBatis-Plus + Spring Security + OAuth2 | REST/SSE API、认证、业务聚合 |
| Agent 服务 | `core/agent` | Python 3.11 + FastAPI + SQLAlchemy 2.0 | Agent 执行引擎（Chat/CoT/CoT Process） |
| Workflow 服务 | `core/workflow` | Python 3.11 + FastAPI + SQLModel + LangChain + Deno/Pyodide | 工作流编排引擎（Spark Flow）、代码沙箱 |
| Knowledge 服务 | `core/knowledge` | Python 3.11 + FastAPI + RAGFlow SDK | 知识库、文档向量化、RAG |
| Memory 服务 | `core/memory/database` | Python 3.11 | 对话历史与长短记忆 |
| Tenant 服务 | `core/tenant` | Go 1.23 + Gin | 多租户、空间隔离、配额 |
| 插件 | `core/plugin/{aitools,link,rpa}` | Python 3.11 + FastAPI | AI 工具 / Link(MCP) / RPA 插件 |
| 公共库 | `core/common` | Python | 认证、日志、可观测、DB/缓存/MQ/OSS 抽象 |

### 制品形态

- **Docker Compose**：`docker/astronAgent/docker-compose*.yaml`（本次部署方式）
- **Helm Chart**：`helm/astron-agent/`（K8s 交付路径，含 Deployment 11 / StatefulSet 5 / Service 17 / ConfigMap 11 / Secret 9 / Ingress 4）
- **自研镜像**：10 个（`core-agent`、`core-workflow`、`core-knowledge`、`core-database`、`core-tenant`、`core-link`、`core-rpa`、`core-aitools`、`console-hub`、`console-frontend`），CI 推 GHCR

---

## 2. 当前部署拓扑与运行状态

**部署方式**：Docker Compose（`docker-compose-with-auth.yaml` = 基础栈 + Casdoor 认证栈）
**容器数**：**19 个**（17 运行中 + 2 个 busybox 一次性初始化容器已 `Exited(0)`）

| 类别 | 容器 |
|---|---|
| 基础设施（7） | `postgres:14`、`mysql:8.4.6`、`redis:7`、`quay.io/minio/minio`、`nginx:1.25-alpine`、`casbin/casdoor:v2.67.0`、`casdoor-mysql(mysql:8.4.6)` |
| 一次性初始化（2） | `busybox:1.35` × 2（内部凭据初始化、MinIO 凭据校验） |
| 自研服务（10） | `console-frontend`、`console-hub`、`core-agent`、`core-workflow`、`core-knowledge`、`core-database`、`core-tenant`、`core-link`、`core-rpa`、`core-aitools` |

**访问入口**

| 入口 | 地址 | 说明 |
|---|---|---|
| 控制台 | `http://100.78.54.115/` | Nginx:80，唯一公网应用入口 |
| Casdoor 管理台 | `http://100.78.54.115:8001` | SSO/组织/应用/用户管理 |
| MinIO API | `100.78.54.115:18998` | 绑主机 IP（供浏览器预签名下载） |
| MinIO 控制台 | `127.0.0.1:18999` | **仅回环**，不对外 |

**镜像来源**

| 服务 | 使用的镜像 |
|---|---|
| 8 个 core-* / console-frontend | `ghcr.nju.edu.cn/iflytek/astron-agent/<name>:latest`（南大镜像站） |
| console-hub | **本地自建** `astron-agent/console-hub:neutral`（带部署定制文案） |
| console-frontend | **本地自建**（含 Markdown 深色表格修复，覆盖了 `latest` tag；原镜像备份为 `...:backup-20260917-135815`） |

**未部署组件**

| 组件 | 状态 |
|---|---|
| RAGFlow（知识库可选项，含 ES/OpenSearch/Infinity） | ❌ 未部署 |
| Kafka（`KAFKA_ENABLE=0`） | ❌ 未启用 |
| Elasticsearch / Kibana / Logstash | ❌ compose 中已注释 |
| OTLP / Langfuse 可观测 | ⚙️ 配置项存在，未启用 |

---

## 3. 已部署的业务资产（实测台账）

### 认证域（Casdoor）

| 对象 | 值 |
|---|---|
| 组织 | `built-in`、`example-org`、`organization_904zw9`(冗余)、**`新丰市人民医院`** |
| 应用 | `app-built-in`(built-in)、**`astron-agent-app`(新丰市人民医院)**、`example-app`(example-org) |
| 控制台绑定 | `client_id=astron-agent-client`、`app=astron-agent-app`、`org=新丰市人民医院`、`redirect=http://100.78.54.115/callback` |

### 企业 / 空间 / 成员

| 层级 | 事实 |
|---|---|
| 企业 | `新丰市人民医院`（id=1，service_type=2 ENTERPRISE）· **owner = 张三丰（OFFICER/1）** |
| 企业成员 | 张三丰（role=1）· 李管理（role=2 GOVERNOR）· 李四方（role=3 STAFF） |
| 空间 | `旅游业务`（id=1，**type=4 企业版**）· **owner = 李管理** |
| 空间成员 | 李管理（OWNER/1）· 张三丰（MEMBER/3）· 李四方（MEMBER/3） |

### 应用与模型

| 对象 | 值 |
|---|---|
| 智能体 | **`旅游导师`**（`chat_bot_base#5`）· uid=李管理 · `space_id=1` · `model_id=3` · 已发布 **MARKET** |
| 已发布版本 | `workflow_version#9`（v3.0，`publish_result=成功`，`deleted=0`） |
| 工作流 | `flow_id=7506188321326616576`（`astron_console.workflow#202037`）· 节点：开始 → 智能体交互(大模型) → 结束 |
| 模型 | `Qwen3.8-27B-AWQ-INT4`（`model#3`，`space_id=1`，provider=openai，指向本地 vLLM） |
| admin 名下资产 | 3 个流程 + 4 个应用（含 `E2E验证助手`，已发市场）—— **admin 已无法登录控制台**（见 §7.1） |

### 账号（全部可用）

| 姓名 | 手机号 | 密码 | 身份 |
|---|---|---|---|
| 张三丰 | `13800001111` | `123456` | 企业超管（OFFICER）、空间成员 |
| 李管理 | `13900002222` | `123456` | 企业管理员（GOVERNOR）、空间 OWNER |
| 李四方 | `14000003333` | `123456` | 企业成员（STAFF）、空间成员 |
| Casdoor 内置 | `admin` | `123` | Casdoor 全局管理员（**进不了控制台**） |

---

## 4. 本地模型服务（vLLM）

| 项 | 值 |
|---|---|
| 部署形态 | **宿主机 systemd 服务** `qwen38-27b.service`（非容器） |
| 启动命令 | `python -m vllm.entrypoints.openai.api_server --config /opt/vllm-config/Qwen3.8-27B-AWQ-INT4.yaml` |
| 模型 | `/mnt/models/cyankiwi/Qwen3.8-27B-AWQ-INT4` |
| 硬件 | 2 × RTX 2080Ti 22GB，`tensor_parallel_size=2` |
| 关键参数 | `max_model_len=262144`、**`max_num_seqs=1`（单并发）**、`gpu_memory_utilization=0.85`、`kv_cache_dtype=fp8_e4m3`、`speculative_config: mtp` |
| 思考模式 | **`default_chat_template_kwargs.enable_thinking: false`**（本次为兼容平台而关闭） |
| 接口 | `http://192.168.20.244:8000/v1`（OpenAI 兼容） |
| 启动耗时 | **12–17 分钟**（正常；期间会重复打印 `No available shared memory broadcast block found`，非卡死） |
| 实测延迟 | 简单问题 ~6s；含 2000+ token 系统提示的长请求，开思考时约 34s，关思考后显著下降 |

---

## 5. 已验证可用的能力（有实测证据）

| 能力 | 证据 |
|---|---|
| OIDC 单点登录（Casdoor → 控制台） | 浏览器实测：`/` → Casdoor 授权页 → 登录 → `/home`，`localStorage.accessToken` 写入（JWT，约 2.9 KB） |
| 组织/应用/用户三层建模 | Casdoor API + DB 核对 |
| 企业开通与角色体系 | `agent_enterprise` / `agent_enterprise_user`（OFFICER/GOVERNOR/STAFF） |
| 空间与成员管理 | `agent_space`(type=4) / `agent_space_user`（OWNER/ADMIN/MEMBER）+ 权限矩阵表 |
| 邀请制成员加入（**无需短信/邮箱验证码**） | `invite-record/enterprise-invite` + `accept-invite` 实测成功 |
| 可视化工作流编排（开始/大模型/结束节点） | 编排页可编辑、可保存、可发布 |
| 发布到应用市场 | `POST /publish/bots/5` 成功，生成版本快照 |
| **智能体对话（含多轮上下文）** | 实测：第 2 轮答出「王小明 / 重庆 / 3 天」三项全对 |
| 空间内模型接入（OpenAI 兼容） | `POST /api/model` 通过校验（API Key 需 RSA 加密） |
| 空间内创建智能体 | `POST /workflow/base-save`（带 `space-id` 头）落 `chat_bot_base.space_id` |
| 新用户上线闭环 | 新账号 → 邀请入企业 → 加入空间 → 可正常对话（实测通过） |

---

## 6. 未验证 / 待验证的能力

| 能力 | 状态 |
|---|---|
| 知识库 / RAG（core-knowledge + RAGFlow） | ⚠️ 服务在跑，**未接入 RAGFlow，未做端到端验证** |
| 代码节点沙箱（Pyodide/Deno） | ⚠️ 镜像内已预置 Deno 与 Pyodide 缓存，**未实测** |
| 决策/循环/迭代/变量聚合等节点 | ⚠️ 未实测 |
| Agent 节点（多智能体协作） | ⚠️ 未实测 |
| RPA 插件（core-rpa） | ⚠️ 服务在跑，**未接外部 RPA 执行器** |
| Link/MCP 插件 | ⚠️ 服务在跑，未见实际调用 |
| AI Tools 插件（讯飞能力） | ⚠️ 需配置讯飞开放平台账号（未配） |
| Kafka 事件流 | ⚠️ 未启用 |
| 可观测（OTLP / Langfuse） | ⚠️ 未启用 |
| 多租户隔离的极限场景 | ⚠️ 单企业单空间验证，未压测 |

---

## 7. 已暴露的限制与缺陷（27 条已归档，此处列关键项）

> 完整清单见 `/root/project/astron-agent-enterprise-delivery-playbook.md` 第三部分

### 7.1 认证架构强耦合（🔴）

- Casdoor 的**应用归属且只归属一个组织**；控制台只绑定一个应用 → **一个控制台只能服务一个组织的用户**
- **实证**：把 `astron-agent-app` 从 `built-in` 迁到 `新丰市人民医院` 后，Casdoor 全局管理员 `admin` **再也无法登录控制台**，其名下 3 个流程 + 4 个应用变成无人可管理的孤儿资产
- 控制台按 **OIDC `sub`（Casdoor user id）** 建档 → 换认证源等于换账号身份，历史数据不跟随
- `built-in` 组织**禁止新增用户**（该组织用户均为 Casdoor 全局管理员，需开"特权同意"才可绕过，不建议）
- 组织**必须至少有一个应用**才能在该组织下建用户（否则报「组织应拥有至少一个应用」，且报错文案显示的是组织 owner 恒为 `admin`，极具误导）

### 7.2 模型作用域与市场交付（🔴）

- 运行时模型解析按 **`model.space_id = 当前空间`**；模型必须落在 `uid` 或 `space_id` 作用域内，否则报 `Model does not exist` → 返回平台兜底话术
- **市场可见 ≠ 可用**：智能体发布到应用市场后，**未加入空间的新用户能看到、能打开，但一发消息就返回兜底话术**（实测：加入空间前失败、加入后正常）
- 创建空间内模型时 API Key 需**用 RSA 公钥加密后提交**，否则报「API Key 加载失败」

### 7.3 发布与生效机制（🔴）

- **已发布到市场的智能体，生产对话入口运行的是「已发布版本的 DSL 快照」**（`api/v1/chat/open.py` → `get_and_validate_published_flow`）
- **在编辑器里改配置必须重新点「发布」才生效**（实证：勾选「对话历史」后不发布无效，重新发布后多轮上下文立刻生效）
- 多轮上下文需节点级开关 `enableChatHistoryV2.isEnabled = true`（+ `rounds`，1–20）

### 7.4 模型接入兼容性（🔴）

- 平台流式解析只识别 `reasoning_content`，且用 `delta["content"]` **直接取键**；而 vLLM 默认开思考时把思考过程放在 **`reasoning`** 字段 → 遇到只含 `reasoning` 的片段会 `KeyError`，最终**输出为空**
- 规避方式：**在 vLLM 侧关闭思考模式**（`enable_thinking: false`）

### 7.5 其他已修/已知缺陷

| 项 | 状态 |
|---|---|
| `workflow_version.deleted` DDL 默认值为 `1`（代码按 `deleted=0` 查询）→ 发布报「工作流版本发布结果异常」且整体回滚 | ✅ 已修（`ALTER ... SET DEFAULT 0`） |
| 编排结束节点若绑定到大模型的 `REASONING_CONTENT`（思考过程）而非 `output` → 输出为空 | ✅ 已修（改绑 `output`） |
| `POST /workflow/base-save` 更新分支名称去重把自己判为重复 → 同名更新必失败 | ❌ 未修（绕过：改名更新或直接改库） |
| 前端 Markdown 表格在**深色模式**下「黑底黑字」 | ✅ 已修（`classes.scss` 固定浅色 token，已构建上线） |
| 平台用**兜底话术掩盖真实错误**（如模型缺失、空间越权），用户侧只看到"抱歉，我暂时无法回答" | ❌ 缺陷（排障成本极高） |
| 通过 API 创建的工作流是**空图且 `can_publish=0`**，与 UI 创建（带模板）行为不一致 | ❌ 缺陷 |
| 空间切换依赖前端 `sessionStorage`（`space-storage` / `spaceType='team'`），UI 切换器不稳定 | ❌ 体验问题 |
| Casdoor 管理台的应用卡片**不可点击**（纯展示，HTML 内无 `<a>`） | ❌ 体验问题 |

---

## 8. 评估替代方案时的关键耦合点

以下 8 点是**换平台时必须一并考虑的约束**（均为实测得出）：

1. **认证源即账号身份**：控制台账号与 OIDC `sub` 绑定；迁移认证 = 迁移用户体系
2. **认证应用与组织一对一**：多组织客户需要多控制台或多应用实例
3. **企业 → 空间 → 成员三级权限模型**：权限矩阵落在 `agent_space_permission` / `agent_enterprise_permission`，共 157 条种子权限
4. **模型作用域隔离**：模型必须归属用户或空间，跨空间不可复用 → 多空间需重复配置模型
5. **发布快照机制**：编辑态 ≠ 运行态，必须显式发布
6. **市场 vs 空间两条交付路径**：市场全局可见但受模型作用域限制；空间内交付才完整可用
7. **工作流 DSL 为自研格式**（节点/边 JSON，存 `astron_console.workflow.data`），非 BPMN / 非 LangGraph 原生；节点类型含 `node-start` / `spark-llm` / `node-end` / `knowledge` / `code` / `decision` / `iteration` / `loop` 等
8. **智能体 = `chat_bot_base`（元数据）+ workflow（DSL）绑定**，且发布后生成 `workflow_version` 快照 + `chat_bot_market` 上架记录

---

## 9. 运维事实

**常用命令**

```bash
cd /root/project/astron-agent/docker/astronAgent

# 状态 / 日志
docker compose -f docker-compose-with-auth.yaml ps
docker logs -f astron-agent-console-hub
docker logs -f astron-agent-casdoor

# 改 .env 后重建
docker compose -f docker-compose-with-auth.yaml up -d --force-recreate console-frontend console-hub

# 部署前契约预检（必做）
python3 scripts/verify_security_contract.py --compose-file docker-compose-with-auth.yaml

# 停止（保留数据）
docker compose -f docker-compose-with-auth.yaml stop
```

**红线**：禁止 `docker compose down -v`（会删除命名卷与全部业务数据）。常规升级须保留 `.env` 与命名卷。

**数据落点**

| 存储 | 内容 |
|---|---|
| `astron_console` (MySQL) | 业务主库：user_info / workflow / chat_bot_base / chat_bot_market / agent_space / agent_space_user / model / chat_* 等 |
| `sparkdb_manager` (PostgreSQL) | Memory DB（对话历史/记忆） |
| `casdoor` (MySQL) | 认证域：organization / application / user / session / token |
| MinIO | 文件与产物（bucket：`console-oss`、`workflow-artifacts` 等） |
| Redis | 缓存、会话、事件注册表 |

**资源占用**

- 2 × RTX 2080Ti 22GB（vLLM，TP=2，单并发）
- 17 个常驻容器（含 7 个中间件）
- 日志路径：`docker/astronAgent/config/<service>/logs/`

---

## 10. 相关文档索引（父目录 `/root/project/`）

| 文件 | 内容 |
|---|---|
| `astron-agent-images.md` | 全部镜像清单（自研 10 + 第三方 22），含技术栈与作用 |
| `astron-agent-compose-vs-kubernetes.md` | Docker Compose 与 Kubernetes/Helm 的关系、对比与本项目双制品映射 |
| `astron-agent-enterprise-delivery-playbook.md` | **企业版交付手册**：六阶段路线图 + **27 条踩坑** + 人机协作规范 + P5 验收清单（8 项）+ 接口速查 |
| `astron-agent-markdown-dark-table-fix.md` | Markdown 深色表格问题完整修复记录（含排查方法论） |

---

## 11. 供对比讨论的开放问题

1. **认证**：是否接受「一个控制台 = 一个组织」的约束？多组织/多法人客户如何交付？
2. **模型作用域**：模型按空间隔离是否合理？多空间复用模型的成本如何？
3. **发布机制**：编辑态与运行态分离（快照）对交付效率的影响？
4. **工作流 DSL**：自研 DSL 的可移植性/可迁移性如何？是否构成迁移锁定？
5. **可观测**：当前未启用 OTLP/Langfuse，替代方案在可观测性上的差距？
6. **错误可见性**：平台用兜底话术掩盖真实错误，对运维与客户支持成本的影响？
7. **资源基线**：单并发 vLLM + 17 容器的最低资源要求，是否满足目标客户环境？
8. **组件完整性**：知识库（RAGFlow 未部署）、RPA（无外部执行器）、Kafka（未启用）等能力实际可用比例？

---

## 附：事实来源说明

本文所有结论均来自本次交付的实测，证据类型包括：

- **代码走读**：`console/backend`、`core/workflow`、`console/frontend` 关键路径
- **数据库核对**：`astron_console` / `casdoor` 库的 `SELECT` 结果
- **服务端日志**：console-hub / core-workflow / vLLM journal 中的异常栈与请求日志
- **浏览器实测**：headless Chromium + CDP 真实点击、填写、发送消息，读取 DOM 与计算结果
- **接口实测**：Casdoor REST API、控制台 `/console-api/*` 的真实响应

**未经验证的内容已在 §6 单独标注，请勿当作既有能力。**
