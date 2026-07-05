# Railway + MySQL 部署 Runbook

> 目标：把本 fork 部署到 Railway，数据库用 MySQL。
> 本文档是 omni-api-relay 定制层文档，不属于上游 New API。

## 背景

上游 New API 官方 `docker-compose.yml` 默认用 **PostgreSQL**，官方 Railway 一键模板（`railway.com/deploy/new-api`）也是 PostgreSQL + Redis。
本项目按红线要求改用 **MySQL** —— New API 的 `model/main.go` 里 `chooseDB()` 会根据 `SQL_DSN` 前缀自动选择数据库驱动，所以只要把 `SQL_DSN` 写成 MySQL 的 Go 驱动格式即可，无需改代码。

## 关键事实

| 项 | 值 |
|---|---|
| 部署镜像 | `calciumion/new-api:latest`（官方预构建，无需自己编译） |
| 容器端口 | 3000（但 New API 原生读 `PORT` 环境变量，兼容 Railway 注入） |
| 数据持久化目录 | `/data`（上传文件、日志；用 MySQL 后 SQLite 文件不需要） |
| 首次登录 | `root` / `123456` → **登录后立即改密码** |
| MySQL DSN 格式 | `user:password@tcp(host:port)/dbname`（**不是** `mysql://` URL 式） |

## 部署步骤

### 1. 建项目
Railway → **New Project** → **Empty Project**。

### 2. 加 MySQL 服务
`+ New` → **Database** → **MySQL**。
它会自动注入：`MYSQLHOST` / `MYSQLPORT` / `MYSQLUSER` / `MYSQLPASSWORD` / `MYSQLDATABASE` / `MYSQL_URL` / `RAILWAY_PRIVATE_DOMAIN`。
记下服务名（默认就叫 `MySQL`）。

### 3. 加 New API 服务
`+ New` → **Empty Service** → Settings → Source → **Docker Image** → 填 `calciumion/new-api:latest`。

> 备选：Deploy from GitHub Repo，选你的 fork `ffixx22722/new-api`，Railway 会用仓库根目录的 `Dockerfile` 构建（约 15s vs 镜像 6s，但能带上你的定制代码）。

### 4. 配 New API 环境变量

在 New API 服务的 **Variables** 里加（用 Railway 引用变量语法 `${{服务名.变量名}}`）：

```
SQL_DSN=${{MySQL.MYSQLUSER}}:${{MySQL.MYSQLPASSWORD}}@tcp(${{MySQL.RAILWAY_PRIVATE_DOMAIN}}:3306)/${{MySQL.MYSQLDATABASE}}
SESSION_SECRET=<生成一段随机字符串>
TZ=Asia/Shanghai
```

- `SESSION_SECRET`：生产必设，避免重启后掉登录态。生成方法：`openssl rand -hex 32`。
- ⚠️ 若私有域名连不上，把 host 段改成 `@tcp(${{MySQL.MYSQLHOST}}:${{MySQL.MYSQLPORT}})`（走公网代理）。
- **不要手动设 `PORT`**，让 Railway 注入随机端口，New API 会自动读它。

### 5. 挂持久化 Volume
New API 服务 → **Volumes** → 新建 Volume，mount path 填 `/data`。
（保住上传文件和本地日志；每个服务通常只能挂一个 Volume。）

### 6. 生成公网域名
New API 服务 → Settings → **Networking** → **Generate Domain**。
⚠️ **target port 留空**（Railway 用注入的 `PORT`）。若坚持手动设了 `PORT=3000`，则这里也要填 3000，否则会报 `Application failed to respond`。

### 7. 部署并首登
等构建 + 启动完成，访问生成的域名 → 用 `root` / `123456` 登录 → **立即在后台改管理员密码**。

## 可选：加 Redis

MVP 阶段可跳过（单实例内存缓存够用）。若要加：
1. `+ New` → **Redis**。
2. New API 加环境变量：
   ```
   REDIS_CONN_STRING=${{Redis.REDIS_URL}}
   CRYPTO_SECRET=<生成一段随机字符串>
   ```
   ⚠️ **启用 Redis 时 `CRYPTO_SECRET` 必填**（用于加密存入 Redis 的内容），别漏。

## 安全红线（务必遵守）

- 以上所有 Secret（`SESSION_SECRET` / `CRYPTO_SECRET` / 数据库密码）**只填在 Railway Variables**，绝不写进代码、注释或 Git。
- 本地测试用 `.env`（已被 `.gitignore` 忽略），同样不入库。

## 需实操验证的点（首次部署时确认）

1. Railway MySQL 服务的实际名字（引用变量的 `MySQL.` 前缀是否正确）。
2. 私有网络下 MySQL 端口是否为 3306。
3. 生成域名时 target port 与 `PORT` 的对齐（否则 `Application failed to respond`）。
4. Redis 场景下 Railway Redis 的连接变量名（常见是 `REDIS_URL`）。

## 参考
- [New API 环境变量文档](https://doc.newapi.pro/en/installation/environment-variables/)
- [Railway 引用变量](https://docs.railway.com/variables)
- [Railway MySQL](https://docs.railway.com/databases/mysql)
- [官方 New API Railway 模板（PostgreSQL 版，参考）](https://railway.com/deploy/new-api)
