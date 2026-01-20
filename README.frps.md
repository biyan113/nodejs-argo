# FRPS Docker 镜像使用指南

## 快速开始

### 拉取镜像

```bash
docker pull ghcr.io/biyan113/frps:latest
```

### 使用默认配置运行

```bash
docker run -d \
  --name frps \
  -p 7000:7000 \
  -p 7500:7500 \
  -p 80:80 \
  -p 443:443 \
  --restart unless-stopped \
  ghcr.io/biyan113/frps:latest
```

### 自定义配置运行

```bash
docker run -d \
  --name frps \
  -p 7000:7000 \
  -p 7500:7500 \
  -p 80:80 \
  -p 443:443 \
  -e DASHBOARD_USER=myadmin \
  -e DASHBOARD_PWD=mypassword123 \
  -e TOKEN=my_secret_token_here \
  --restart unless-stopped \
  ghcr.io/biyan113/frps:latest
```

## 环境变量

| 变量 | 默认值 | 说明 |
|------|--------|------|
| `BIND_PORT` | 7000 | FRP 服务器端口 |
| `DASHBOARD_PORT` | 7500 | Dashboard 管理面板端口 |
| `DASHBOARD_USER` | admin | Dashboard 用户名 |
| `DASHBOARD_PWD` | admin123 | Dashboard 密码 |
| `TOKEN` | change_me_token | 客户端连接令牌 |

## 端口说明

| 端口 | 说明 |
|------|------|
| 7000 | FRP 服务端口（客户端连接） |
| 7500 | Dashboard 管理面板 |
| 80 | HTTP 代理端口 |
| 443 | HTTPS 代理端口 |

## 访问 Dashboard

启动后访问: `http://your-ip:7500`

- 用户名: `admin` (或自定义的 DASHBOARD_USER)
- 密码: `admin123` (或自定义的 DASHBOARD_PWD)

## 客户端配置示例

```ini
[common]
server_addr = your-server-ip
server_port = 7000
token = change_me_token

[ssh]
type = tcp
local_ip = 127.0.0.1
local_port = 22
remote_port = 6000
```

## 常用命令

```bash
# 查看日志
docker logs -f frps

# 停止服务
docker stop frps

# 启动服务
docker start frps

# 重启服务
docker restart frps

# 删除容器
docker rm -f frps
```
