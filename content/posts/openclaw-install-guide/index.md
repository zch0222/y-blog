---
date: '2026-02-07T12:54:57+08:00'
draft: true
title: "OpenClaw 部署指南：从零开始搭建私有 AI 网关"
categories: ["Self-hosted", "AI", "DevOps"]
tags: ["OpenClaw", "Node.js", "Nginx", "Linux", "Systemd"]
description: "记录 OpenClaw 的手动部署全流程：解决 NVM 冲突、配置 Systemd 服务以及 Nginx WebSocket 反向代理。"
---

OpenClaw 是一个强大的 AI 网关工具。本文记录了在 Linux VPS 上从零部署 OpenClaw 的完整流程，特别解决了使用 NVM 导致的 Systemd 路径问题、Token 认证不匹配以及 Nginx WebSocket 连接断开等常见坑点。

## 前置准备

* 一台 Linux 服务器 (Debian 11/12 或 Ubuntu 22.04+)
* Root 权限
* 一个域名 (用于 Nginx 反代)

---

## 第一步：环境准备 (Node.js)

**关键点：** 生产环境部署服务时，**不要使用 NVM** (Node Version Manager)。Systemd 服务很难正确加载 NVM 的环境变量，会导致服务无法启动。我们需要安装**系统级**的 Node.js。

### 1. 清理旧环境 (如果有)
如果你之前装过 NVM 或旧版 Node，建议先清理：

```bash
# 停止旧服务
systemctl --user stop openclaw-gateway.service 2>/dev/null
# 移除 NVM (可选，防止干扰)
rm -rf ~/.nvm
unset NVM_DIR

```

### 2. 安装 Node.js 22 (LTS)

使用 NodeSource 官方源安装系统级 Node.js：

```bash
# 1. 安装基础工具
sudo apt-get update && sudo apt-get install -y curl gnupg

# 2. 添加 Node.js 22 源
curl -fsSL https://deb.nodesource.com/setup_22.x | sudo -E bash -

# 3. 安装 Node.js
sudo apt-get install -y nodejs

# 4. 验证 (应输出 /usr/bin/node)
which node
node -v

```

---

## 第二步：安装与初始化 OpenClaw

### 1. 全局安装

```bash
sudo npm install -g openclaw

```

### 2. 初始化配置 (关键)

为了防止启动时出现 `mode unset` 或 `token mismatch` 错误，我们需要在启动前手动写入核心配置。

```bash
# 设置运行模式
openclaw config set gateway.mode local

# 生成一个强 Token (用于客户端和服务端认证)
# 你也可以手动指定一个字符串，但务必保证下面两行一致
MY_TOKEN=$(openssl rand -hex 16)
echo "Your Token: $MY_TOKEN"

# 同时写入服务端锁(auth)和客户端钥匙(remote)
openclaw config set gateway.auth.token "$MY_TOKEN"
openclaw config set gateway.remote.token "$MY_TOKEN"

# 信任本地代理 (为 Nginx 做准备)
openclaw config set gateway.trustedProxies "127.0.0.1"

```

---

## 第三步：配置 Systemd 服务

为了让 OpenClaw 开机自启并后台运行，我们需要创建一个标准的 Systemd 服务文件。

### 1. 创建服务文件

创建文件 `/etc/systemd/system/openclaw-gateway.service`：

```bash
sudo tee /etc/systemd/system/openclaw-gateway.service <<EOF
[Unit]
Description=OpenClaw Gateway Service
Documentation=https://docs.openclaw.ai
After=network.target

[Service]
Type=simple
User=<USER>
# 建议手动填入绝对路径，防止 $(which) 在 sudo 环境下失效
ExecStart=$(which openclaw || echo "/usr/local/bin/openclaw") gateway --port 18789
Restart=always
RestartSec=3
Environment=NODE_ENV=production

[Install]
WantedBy=multi-user.target
EOF

```
把<USER>替换成当前用户

### 2. 启动服务

```bash
sudo systemctl daemon-reload
sudo systemctl enable openclaw-gateway.service
sudo systemctl start openclaw-gateway.service

```

### 3. 检查状态

```bash
openclaw status

```

*此时状态应显示绿色的 `running`，且不应有红色报错。*

---

## 第四步：配置 Nginx 反向代理

OpenClaw 使用 WebSocket 进行实时通信，Nginx 必须配置 `Upgrade` 头，否则会出现 `Disconnected (1006)` 错误。

### 1. 修改 Nginx 配置

编辑你的站点配置文件 (例如 `/etc/nginx/sites-available/your-domain.com`)。

**注意：** `map` 块必须放在 `server` 块之外（通常在 `http` 块内，或者文件最顶部）。

```nginx
# --- 1. 定义 WebSocket 升级变量 (放在 server 块外面) ---
map $http_upgrade $connection_upgrade {
    default upgrade;
    ''      close;
}

server {
    listen 80;
    server_name your-domain.com; # 替换你的域名

    location / {
        proxy_pass [http://127.0.0.1:18789](http://127.0.0.1:18789);

        # --- 2. 关键 WebSocket 配置 ---
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection $connection_upgrade;

        # --- 3. 传递真实 IP ---
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        
        # --- 4. 延长超时 (防止长连接断开) ---
        proxy_read_timeout 86400s;
    }
}

```

### 2. 启用 HTTPS (强烈建议)

```bash
apt install certbot python3-certbot-nginx
certbot --nginx -d your-domain.com

```

---

## 第五步：设备配对与授权

访问 `https://your-domain.com`，你可能会看到提示 `Disconnected (1008): pairing required`。这是因为 OpenClaw 默认开启了设备安全验证。

### 1. 获取浏览器请求 ID

保持浏览器打开，在服务器终端运行：

```bash
openclaw devices list

```

你会看到一行状态为 `pending` 的记录：

```text
ID                                     Hostname    State
9a95ec67-abe7-xxxx-xxxx-xxxxxxxxxxxx   Chrome      pending

```

### 2. 批准设备

复制上面的 ID，运行批准命令：

```bash
openclaw devices approve <你的设备ID>

```

回到浏览器，页面会自动刷新并连接成功。

---

## 常用管理命令

* **查看状态**：`openclaw status`
* **实时日志**：`openclaw logs --follow`
* **查看配置**：`cat ~/.openclaw/openclaw.json`
* **重启服务**：`systemctl restart openclaw-gateway.service`

至此，OpenClaw 已成功部署。你可以开始在 Web 面板中配置 OpenAI 或 Claude 模型并创建 Agent 了。