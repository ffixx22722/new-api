# Omni API Relay — 定制层规范与项目记忆

> 本文件是 **omni-api-relay** 这个 fork 的定制层长期记忆与工作契约。
> 上游 New API 的通用规范见 [AGENTS.md](AGENTS.md) 与 [CLAUDE.md](CLAUDE.md)，务必一并遵守（尤其 new-api / QuantumNous 品牌标识受保护、数据库三库兼容、JSON 走 `common.*` 包装等）。
> 本文件只记录「我们在上游基础上叠加的东西」，不重复上游已有的内容。

## 项目愿景
内部自用的多模型聚合与 API 中转网关，基于 [New API](https://github.com/QuantumNous/new-api) 核心逻辑进行定制开发。

## 定制层红线
- **后端核心**：Go (Gin) / 契约优先（contract-first）/ 强类型安全（与上游一致）
- **部署运维**：**Railway + MySQL**（注意：上游 docker-compose 默认用 PostgreSQL，我们改用 MySQL，靠 `SQL_DSN` 的 MySQL 格式自动切换）
- **安全**：严禁 Secret（API Key、数据库密码、令牌等）出现在代码、注释或 Git 历史中。所有敏感值走环境变量 / Railway Variables，本地用 `.env`（已在上游 `.gitignore` 忽略）
- **协作规则**：先读后写 / 改完即验 / 失败自愈

## 常用开发命令
- 依赖对齐：`go mod tidy`
- 本地编译自检：`go build -o local_test_bin main.go`
- 前端（如需）：进 `web/default/`，用 `bun install` / `bun run build`

## Git 仓库结构
- `origin` → `https://github.com/ffixx22722/new-api.git`（你的 fork，推这里）
- `upstream` → `https://github.com/QuantumNous/new-api.git`（官方，拉更新：`git fetch upstream && git merge upstream/main`）
- 本地路径：`~/Cloud/omni-api-relay`

## 环境现状（2026-07-05 记录）
- ✅ 本机已装 Go（go1.26.4 darwin/arm64），`go build` / `go mod tidy` 可直接运行。
- ✅ Railway CLI 已安装（v5.4.2）。
- ✅ gh CLI 已登录账号 `ffixx22722`。
- ✅ git 全局身份已配置（ffixx22722 / ffixx22722@gmail.com）。

## Railway + MySQL 部署要点（调研结论）
- **镜像**：直接用官方预构建镜像 `calciumion/new-api:latest`，无需自己编译 Go。
- **数据库**：Railway 加 MySQL 服务后，`SQL_DSN` 要用 Go 驱动格式（**不是** Railway 给的 `mysql://` URL 格式），用引用变量手拼：
  ```
  SQL_DSN=${{MySQL.MYSQLUSER}}:${{MySQL.MYSQLPASSWORD}}@tcp(${{MySQL.RAILWAY_PRIVATE_DOMAIN}}:3306)/${{MySQL.MYSQLDATABASE}}
  ```
  连不上时退回 `@tcp(${{MySQL.MYSQLHOST}}:${{MySQL.MYSQLPORT}})`。
- **端口**：New API 原生读 `PORT` 环境变量，天然兼容 Railway。生成域名时 target port 留空即可（用 Railway 注入的 `PORT`）。
- **必设 Secret**：生产必设 `SESSION_SECRET`；若接 Redis 则额外必设 `CRYPTO_SECRET`。
- **持久化**：给 New API 服务挂一个 Volume，mount path = `/data`（保住上传文件/日志；用 MySQL 后 SQLite 不需要）。
- **Redis**：可选。MVP 阶段先不上，单实例内存缓存够用。
- **默认管理员**：首次登录 `root` / `123456`，**登录后立即改密码**。
- 详见 [docs/DEPLOY_RAILWAY.md](docs/DEPLOY_RAILWAY.md)。

## 工作纪律
1. **先读后写**：动手前先读相关代码 / 文档，理解现状再改。
2. **改完即验**：每次改动后本地编译自检（`go build`）通过才算完成，不空口声称「已修复」。
3. **失败自愈**：命令失败先自行排查修复，不把报错原样甩回。
