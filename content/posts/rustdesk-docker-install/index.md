---
date: '2026-02-13T21:36:28+08:00'
draft: false
title: '部署私有 RustDesk 远程桌面服务器（Docker Compose + 强制密钥安全模式）'
summary: 'Docker 私有部署RustDesk'
---

本文将详细介绍如何在 **已安装 Docker 环境** 的 Linux 服务器上，部署一套**防白嫖、高安全**的 RustDesk 服务端。我们将开启强制密钥验证（`-k _`），防止未授权用户盗用您的带宽资源。

## 前置准备

* **服务器**：一台拥有公网 IP 的 Linux 服务器（Debian/Ubuntu/CentOS 均可）。
* **环境**：已安装 Docker 和 Docker Compose。
* **防火墙**：确保您有权限修改服务器的防火墙规则或云厂商的安全组。

---

## 一、 部署 RustDesk Server

我们将使用 Docker Compose 进行管理，并启用 `Host` 网络模式以提高 P2P 打洞成功率，避免复杂的端口映射问题。

### 1. 创建工作目录

```bash
mkdir -p ~/rustdesk
cd ~/rustdesk

```

### 2. 编写配置 `docker-compose.yml`

创建并编辑文件：

```bash
vim docker-compose.yml

```

**请务必将下方的 `你的公网IP` 替换为服务器实际 IP 地址：**

```yaml
services:
  hbbs:
    container_name: hbbs
    image: rustdesk/rustdesk-server:latest
    # 参数详解：
    # -r: 指定中转服务器地址（你的IP:21117）
    # -k _: 开启强制密钥验证。没有 Key 的客户端将被拒绝连接（防止白嫖）
    command: hbbs -r 你的公网IP:21117 -k _
    volumes:
      - ./data:/root
    network_mode: "host"
    depends_on:
      - hbbr
    restart: unless-stopped

  hbbr:
    container_name: hbbr
    image: rustdesk/rustdesk-server:latest
    # -k _: 同样开启密钥验证，确保中转链路安全
    command: hbbr -k _
    volumes:
      - ./data:/root
    network_mode: "host"
    restart: unless-stopped

```

### 3. 启动服务

```bash
docker compose up -d

```

启动后，您可以使用 `docker compose logs -f` 查看日志确认运行状态。

---

## 二、 安全组与防火墙配置（至关重要）

RustDesk 使用特定端口进行通信，**尤其是 UDP 端口，如果未放行将导致“无法连接网络”**。

### 1. 必需端口列表

| 端口 | 协议 | 必须? | 用途 | 说明 |
| --- | --- | --- | --- | --- |
| **21116** | **UDP** | **是** | **心跳/打洞** | **最核心端口，不通则全挂** |
| 21116 | TCP | 是 | TCP打洞 | 辅助连接 |
| 21115 | TCP | 是 | NAT测试 | 基础服务 |
| 21117 | TCP | 是 | 中转服务 | Relay Server |
| 21118 | TCP | 否 | Web支持 | 可选，用于网页版客户端 |
| 21119 | TCP | 否 | Web支持 | 可选，用于网页版客户端 |

### 2. 放行命令 (UFW 示例)

如果您开启了系统防火墙（如 Ubuntu 的 UFW）：

```bash
sudo ufw allow 21115:21119/tcp
sudo ufw allow 21116/udp
sudo ufw reload

```

### 3. 云厂商安全组注意

**重要提示**：如果是阿里云、腾讯云、AWS 等云服务器，务必去网页控制台的**安全组**中，添加入站规则，同时放行上述 TCP 和 UDP 端口。仅仅在服务器内部关闭防火墙是不够的。

---

## 三、 客户端配置指南

为了连接到我们开启了强制安全验证（`-k _`）的服务器，客户端必须填写正确的公钥。

### 1. 获取公钥 (Key)

服务启动后，会自动在挂载的目录下生成密钥对。请在服务器终端执行：

```bash
cat /opt/rustdesk/data/id_ed25519.pub

```

*您将看到一串以 `=` 结尾的字符串，请完整复制它。*

### 2. 配置客户端

打开 RustDesk 客户端（控制端和被控端都需要配置）：

1. 点击 ID 旁的菜单按钮（三个点） -> **网络** -> **ID/中转服务器**。
2. **ID 服务器**：填写 `你的公网IP`。
3. **中转服务器**：填写 `你的公网IP` （或 `你的公网IP:21117`）。
4. **Key**：粘贴刚才获取的公钥字符串。
5. 点击确认。

回到主界面，若底部显示 **“就绪”**，即代表私有化部署成功！

---

## 四、 常见问题排查

* **状态显示“未就绪”或“无法连接 ID 服务器”**：
* 99% 是因为 **UDP 21116** 端口没放行。请仔细检查云控制台安全组是否遗漏了 UDP 协议。


* **提示“Key 不匹配”**：
* 检查 `docker-compose.yml` 中 `hbbs` 和 `hbbr` 是否都挂载了相同的 `./data` 目录。
* 确保客户端填写的 Key 与服务器 `id_ed25519.pub` 文件内容完全一致（注意不要多复制空格）。


* **连接速度慢**：
* 如果在同一局域网内，客户端会自动尝试直连。
* 如果是跨网连接且无法 P2P 打洞，流量会走服务器中转。此时速度取决于您服务器的带宽（hbbr）。