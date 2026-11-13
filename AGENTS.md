# AGENTS.md — starcat-recommend-api

> **唯一协作规范源**：本仓库根目录 `AGENTS.md` 是本项目协作规范的唯一正文维护源。
> 开工前还必须阅读并遵守上级 [`../AGENTS.md`](../AGENTS.md) 的跨仓规则。

## 项目概述

Starcat 统一自研推荐后端：v1 推荐兼容入口与 v2 共用 `starcat-recsys-trainer` 发布的 ServingBundle，不再调用第三方推荐。双渠道新客户端只用 v2；下一次包含该切换的 App 版本上线后删除 v1 推荐兼容层，见 `TODO.md`。商业版才经 `starcat-api` 聚合；本工作区运行实例见「当前部署」。

## 当前部署（开源实例，2026-11-13）

本工作区正在运行的 trending、weekly、wiki、recommend、discovery 是开源独立服务，不是商业版 `starcat-api` 聚合网关。排查、调用和改配置时，不要假设流量走 `https://starcat-api.fly.dev` 或请求头 `X-SC-Svc`。`starcat-api` 只适用于商业版。

实例在 SSH 主机 `ten1027` 的 `/home/ubuntu/starcat`，Compose 项目名 `starcat`，五个容器各跑各的进程。端口只绑定服务器 `127.0.0.1`，本机通过 `ssh ten1027` 已配置的 LocalForward 访问：

- trending：`http://127.0.0.1:5002`，数据 `data/trending/trending.db`
- weekly：`http://127.0.0.1:5003`，数据 `data/weekly/weekly.db` 与 `data/weekly/repo`
- wiki：`http://127.0.0.1:5004`，数据 `data/wiki/`（部署时为空库，由进程自己创建）
- recommend：`http://127.0.0.1:5005`，数据 `data/recommend/model-registry`（尚无 ServingBundle，服务在线但没有推荐结果）
- discovery：`http://127.0.0.1:5006`，数据 `data/discovery/discovery.db`；Linux.do 采集和 Star History 保持关闭

密钥在服务器 `/home/ubuntu/starcat/env/*.env`，权限 `600`。禁止把这些文件、API Key 或 GitHub Token 写入 Git、日志或回复。本地编排副本是与各仓库同级的 `starcat-stack/docker-compose.yml`，该目录不属于任何一个 Git 仓库。服务器 `src/` 只是构建副本，其中 Dockerfile 额外设置了 `GOPROXY=https://goproxy.cn,direct`；各仓库里的 Dockerfile 没有这条。

Starcat 客户端要在设置里把对应服务的 baseURL 指到上述地址，Key 用各仓库 `.env` 里已有的 `API_KEYS`。设置未改时，客户端默认仍会请求聚合网关。跨仓细节见上级 [`../AGENTS.md`](../AGENTS.md)。

## 技术栈

- Go 1.26.10 · `net/http`
- `modernc.org/sqlite`（metrics 与 model registry）
- `github.com/starcat-app/starcat-api-kit` v0.3.0
- `github.com/joho/godotenv`

## 关键目录

```
cmd/server/
server/               # 可导出装配
internal/handler/     # v1 recommend + v2 query
internal/provider/    # trained、content、bundle_database_pool
internal/serving/     # Bundle registry 与 stats
Makefile              # VERSION := 0.1.0
```

## 开发与测试命令

```bash
cp .env.example .env          # API_KEYS 必填；推荐需要已激活的 Bundle
go mod tidy && make run       # PORT=5005
make build
make check                    # fmt-check + vet + test（PR 前）
make docker-build
```

CI（`.github/workflows/go.yml`）：gofmt · vet · govulncheck · docker build · test -race · build。

Smoke test（README）：
```bash
curl http://127.0.0.1:5005/healthz
curl -H "Authorization: Bearer $API_KEY" http://127.0.0.1:5005/api/v1/ping
```

环境变量见 `.env.example`：`API_KEYS`、`MODEL_PUBLISH_KEYS`、`MODEL_REGISTRY_DIR`、`METRICS_STORE_FILE`、`GITHUB_TOKENS`、`MAX_BUNDLE_BYTES`（默认 512 MiB）。Registry 和 metrics 路径必须持久化；无活动 Bundle 时不回退第三方。

## 代码与架构约束

- **密钥隔离**：`API_KEYS`（客户端）与 `MODEL_PUBLISH_KEYS`（Trainer 发布）必须分离；`GITHUB_TOKENS` 仅服务端。
- **v2 ETag**：单仓推荐根据模型版本与分页返回 `ETag`；支持 `If-None-Match` → 304。
- **内部端点**：`POST /internal/v1/model-bundles/{version}` 用 `MODEL_PUBLISH_KEYS`；`GET /internal/stats`、`/internal/metrics/*` 用 `API_KEYS`。
- **响应契约**：envelope + Bearer 鉴权；`/healthz` 公开。
- 改 provider 逻辑后跑 `make check`，注意 `server/trained_test.go` 等集成测试。

## 安全与数据边界

- 禁止入库：`.env`、`data/model-registry/` 运行时内容、`bin/`、`coverage.out`。
- 禁止向 macOS 客户端或日志泄露 `GITHUB_TOKENS`、`MODEL_PUBLISH_KEYS`。
- Bundle 文件为 Trainer 发布产物，非 git 管理。

## 部署与发布禁令

未经 dong4j 明确授权，禁止：`make release`、`scripts/deploy.sh`、`fly deploy`、`git push`/`git tag`、向生产 registry 写入 Bundle。商业版 Fly 生产才经 `starcat-api`。本工作区开源实例在 `ten1027` 的 Docker Compose，见「当前部署」；不要为了访问这些服务去部署或调用聚合网关。
