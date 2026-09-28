# Octop 加固部署包

在官方镜像基础上补齐生产可用性。**未实测**——编写时本机 Docker daemon 未运行，首次构建请留意 Chromium 下载与 gosu 安装两步。

## 文件说明

| 文件 | 作用 |
|------|------|
| `Dockerfile` | 加固层：预装 Playwright Chromium + 建立非 root 用户 |
| `docker-entrypoint-root.sh` | 入口脚本：修正数据目录属主、降权运行、修复上游初始化判据缺陷 |
| `docker-compose.yml` | octop + PostgreSQL(pgvector)，非 root、capability 收敛、日志轮转、资源上限 |
| `postgres/init-vector.sql` | 实例级 pgvector 扩展初始化 |
| `.env.example` | 环境变量模板 |

## 相对官方部署修了什么

### 1. 上游缺陷：PostgreSQL 模式下容器会崩溃循环（P0）

官方 `docker/docker-entrypoint.sh` 的初始化判据是：

```bash
DB_FILE="${OCTOP_HOME}/octop.db"
if [ ! -f "$DB_FILE" ]; then   # 不存在 → 执行 octop init
```

但使用 PostgreSQL 后端时**永远不会生成 `octop.db`**（`src/octop/infra/db/factory.py` 中 PG 分支只返回 `PostgresPool`）。于是：

1. 首次启动：`~/.octop` 为空 → `octop init` 成功 → 凭据写入 `credential.txt`（看起来一切正常）
2. 之后每次重启：`octop.db` 仍不存在 → 再次执行 `octop init`
3. `octop init` 发现 `~/.octop` 非空，直接 `raise SystemExit(1)`（`src/octop/cli/commands/init.py`）
4. 入口脚本把这次失败当成"密码太弱"，换随机密码重试一次，**第二次失败没有 `if` 保护**，`set -euo pipefail` 直接终止脚本
5. 容器退出 → `restart: unless-stopped` 反复拉起 → **崩溃循环**

官方 compose 注释里明确宣传可以用 `OCTOP_DATABASE_DRIVER=postgresql` / `OCTOP_DATABASE_URL`，所以这条路径是被推荐的，但实际跑不通。已验证上游 issue 中无相关记录。

本包改用自建标记文件 `.container-initialized` 判断是否已初始化，SQLite 与 PostgreSQL 两种后端都正确；同时把"目录已有数据"识别为已初始化而非致命错误。

### 2. 非 root 运行

官方 Dockerfile 无 `USER` 指令，容器以 uid 0 运行，会改写宿主绑定目录属主；Agent 的 shell 能力在容器内即 root 权限。本包建立 uid/gid 10001 用户，入口脚本先修正属主再降权。

### 3. 预装 Playwright Chromium

官方镜像只装了 playwright 的 Python 包，未执行 `playwright install chromium`，浏览器自动化能力开箱不可用；且 `PLAYWRIGHT_BROWSERS_PATH` 写死在 `/root/.cache`，非 root 用户不可读。本包改装到 `/opt/ms-playwright` 并开放读权限。

## 使用

```bash
cp .env.example .env
vim .env                       # 填 OCTOP_DATA / POSTGRES_PASSWORD / OCTOP_DEFAULT_PASSWORD

sudo mkdir -p /srv/octop/data
sudo chown -R 10001:10001 /srv/octop/data

docker compose up -d --build
docker compose logs -f octop
cat /srv/octop/data/credential.txt    # 未设密码时看随机密码
curl -f http://127.0.0.1:8088/api/health
```

## 离线交付

```bash
# 有网机器
docker pull ghcr.io/tencentcloud/octop:1.0.0
docker build -t octop-hospital:1.0.0 .
docker save octop-hospital:1.0.0 pgvector/pgvector:pg16 | gzip > octop-offline.tar.gz

# 内网机器
gunzip -c octop-offline.tar.gz | docker load
docker compose up -d            # 去掉 --build，直接用已导入的镜像
```

## 硬约束

- **只能 1 副本**。单进程架构（进程内 APScheduler + 进程内 HarnessProcessor + IM 长连接 + 本地文件 workspace），多副本会导致定时任务重复触发、消息重复或丢失、workspace 状态分裂。K8s 部署须 `replicas: 1` + 更新策略 `Recreate` + PVC `ReadWriteOnce`。
- **无高可用**。故障即中断，靠容器重启恢复（重启可从控制面库重建状态）。
- **官方镜像仅 linux/amd64**。鲲鹏（ARM64 CPU）需自行 `buildx` 构建——依赖树已实测无阻塞（241 个包中 188 个纯 Python、41 个有 aarch64 wheel，其余为 macOS 专用或纯源码 sdist；Playwright 官方 CDN 有 `chromium-linux-arm64` 构建）。**昇腾 NPU 与 Octop 容器零耦合**：Octop 不含 CANN/Ascend 代码，模型接入走 OpenAI 兼容 HTTP，NPU 直通属于推理服务容器的事。
- **注意宿主容器引擎**。若宿主是 openEuler，默认引擎可能是 iSulad 而非 Docker，本 compose 不通用。
- **不要开**：`docker-compose.mobile.yml`（需 privileged + `/dev/binderfs`）、Agent 的 Docker 沙箱 backend（需挂 `/var/run/docker.sock`）。
