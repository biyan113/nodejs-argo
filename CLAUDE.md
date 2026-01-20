# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 项目概述

nodejs-argo 是一个 Argo 隧道部署工具，专为 PaaS 平台设计。项目使用 Cloudflare Argo 隧道创建代理服务，支持多种代理协议（VLESS、VMess、Trojan），并集成了哪吒探针监控功能。

**注意**：本项目根据 LICENSE 文件采用 GPL 3.0 开源协议，仅供个人使用。

---

## 开发命令

### 运行开发服务器

```bash
# 使用 npm
npm run dev

# 或直接运行
node index.js

# 全局安装后运行
npm install -g .
nodejs-argo
```

### Docker 部署

```bash
# 构建镜像
docker build -t nodejs-argo .

# 运行容器
docker run -p 3000:3000 -e PORT=3000 nodejs-argo
```

### 后台运行

```bash
# 使用 PM2
pm2 start nodejs-argo --name "argo-service"

# 使用 screen
screen -S argo
nodejs-argo
# Ctrl+A 然后 D 分离

# 使用 systemd
sudo systemctl start nodejs-argo
```

---

## 核心架构

### 单文件结构

整个项目采用**单文件架构**，所有逻辑位于 `index.js` 中。主要模块包括：

| 模块 | 职责 |
|------|------|
| **Express 服务器** | HTTP 服务（默认 3000 端口），提供 `/` 和 `/sub` 路由 |
| **配置生成** | 生成 xr-ay 配置文件 `config.json`，支持 VLESS/VMess/Trojan 协议 |
| **二进制下载** | 根据系统架构（amd/arm）下载外部依赖文件（web、bot、nezha） |
| **Argo 隧道** | 通过 cloudflared 创建临时或固定隧道 |
| **哪吒探针** | 支持 v0 和 v1 两个版本的哪吒监控 Agent |
| **订阅上传** | 可选上传节点到 Merge-sub 订阅合并服务 |

### 程序启动流程

```
startserver()
    │
    ├── argoType()              # 处理固定隧道配置
    ├── deleteNodes()           # 删除订阅器中的历史节点
    ├── cleanupOldFiles()       # 清理运行目录历史文件
    ├── generateConfig()        # 生成 xr-ay config.json
    ├── downloadFilesAndRun()   # 下载并启动二进制文件
    │     ├── 下载 web (xr-ay)
    │     ├── 下载 bot (cloudflared)
    │     ├── 下载 nezha agent (可选)
    │     ├── 运行哪吒
    │     ├── 运行 xr-ay
    │     └── 运行 cloudflared
    ├── extractDomains()        # 从 boot.log 提取 Argo 域名
    │     └── generateLinks()   # 生成订阅链接
    └── AddVisitTask()          # 添加自动访问保活任务
```

### 二进制依赖

项目运行时会从远程下载以下二进制文件到 `FILE_PATH` 目录（默认 `./tmp`）：

| 文件 | 来源 URL | 用途 |
|------|---------|------|
| `web` | `https://amd64.ssss.nyc.mn/web` 或 `https://arm64.ssss.nyc.mn/web` | xr-ay 核心 |
| `bot` | `https://amd64.ssss.nyc.mn/bot` 或 `https://arm64.ssss.nyc.mn/bot` | cloudflared 隧道 |
| `agent` | `https://amd64.ssss.nyc.mn/agent` 或 `https://arm64.ssss.nyc.mn/agent` | 哪吒 v1 |
| `v1` | `https://amd64.ssss.nyc.mn/v1` 或 `https://arm64.ssss.nyc.mn/v1` | 哪吒 v0 |

这些二进制文件会在 90 秒后自动删除。

---

## 环境变量

### 核心变量

| 变量 | 默认值 | 说明 |
|------|--------|------|
| `PORT` | 3000 | HTTP 服务端口 |
| `SERVER_PORT` | 3000 | PORT 的别名（优先级更高） |
| `ARGO_PORT` | 8001 | Argo 隧道本地端口 |
| `UUID` | 自动生成 | 用户标识（所有协议共用） |

### Argo 隧道配置

| 变量 | 默认值 | 说明 |
|------|--------|------|
| `ARGO_DOMAIN` | - | 固定隧道域名，留空使用临时隧道 |
| `ARGO_AUTH` | - | 固定隧道 token 或 JSON 配置 |

### 哪吒探针配置

| 变量 | 默认值 | 说明 |
|------|--------|------|
| `NEZHA_SERVER` | - | 哪吒面板地址 |
| `NEZHA_PORT` | - | 哪吒端口（有值则用 v1，留空用 v0） |
| `NEZHA_KEY` | - | 哪吒密钥 |

**注意**：当哪吒端口为 `{443,8443,2096,2087,2083,2053}` 之一时，自动开启 TLS。

### 订阅与上传配置

| 变量 | 默认值 | 说明 |
|------|--------|------|
| `UPLOAD_URL` | - | Merge-sub 项目地址 |
| `PROJECT_URL` | https://www.google.com | 项目分配的域名 |
| `SUB_PATH` | sub | 订阅路径 |
| `AUTO_ACCESS` | false | 自动访问保活 |

### 节点配置

| 变量 | 默认值 | 说明 |
|------|--------|------|
| `CFIP` | cdns.doon.eu.org | 优选域名或 IP |
| `CFPORT` | 443 | 优选端口 |
| `NAME` | - | 节点名称前缀 |
| `FILE_PATH` | ./tmp | 运行目录 |

---

## 关键设计点

### 哪吒版本选择逻辑

```javascript
if (NEZHA_PORT) {
    // 使用哪吒 v1 (npm agent)
    // TLS 自动检测
} else if (NEZHA_SERVER && NEZHA_KEY) {
    // 使用哪吒 v0 (php agent)
}
```

### Argo 隧道模式

1. **临时隧道**：`ARGO_DOMAIN` 和 `ARGO_AUTH` 均留空
2. **Token 隧道**：`ARGO_AUTH` 为 token 格式（120-250 字符）
3. **JSON 隧道**：`ARGO_AUTH` 包含 `TunnelSecret`

### 订阅生成

- ISP 信息通过 `ipapi.co` 或 `ip-api.com` 获取
- 节点名称格式：`{NAME}-{ISP}` 或仅 `{ISP}`
- 支持协议：VLESS、VMess、Trojan

---

## 部署平台适配

### Node.js PaaS 平台

只需上传 `index.js` 和 `package.json`

### Docker 平台

使用 `Dockerfile` 构建，基于 `node:alpine3.20`

### 游戏玩具平台

参考 README 中的具体部署指南

---

## 文件清理机制

- **启动时**：清理 `FILE_PATH` 目录下所有历史文件
- **运行后 90 秒**：删除 `boot.log`、`config.json`、二进制文件
- **订阅节点删除**：如果 `UPLOAD_URL` 已设置，会删除订阅器中的历史节点

---

## 订阅地址

- 标准端口：`https://your-domain.com/sub`
- 非标端口：`http://your-domain.com:port/sub`

---

## 外部依赖

- `axios` - HTTP 请求
- `express` - Web 服务器
- `node` 内置模块：`fs`、`path`、`os`、`child_process`
