---
date: '2026-02-13T20:22:25+08:00'
draft: false
title: 'Linux Docker 安装和代理配置 (Ubuntu/Debian)'
summary: 'Ubuntu/Debian安装Docker并配置代理方案'
---

在 Linux 服务器上部署 Docker 是现代开发的必备技能。本文将整合最标准的安装流程，以及代理服务器配置。

## 第一部分：Docker 安装 (Ubuntu / Debian)

我们推荐使用 Docker 官方源进行安装，以确保获取最新、最稳定的版本。

### 1. 卸载旧版本

如果之前安装过旧版本的 Docker（如 `docker.io` 或 `docker-engine`），请先卸载，以防冲突。

```bash
sudo apt-get remove docker.docker-engine docker.io containerd runc

```

### 2. 更新并安装依赖

更新 `apt` 包索引并安装通过 HTTPS 使用存储库所需的软件包。

```bash
sudo apt-get update
sudo apt-get install \
    ca-certificates \
    curl \
    gnupg \
    lsb-release

```

### 3. 添加 Docker 官方 GPG 密钥

这一步是为了确保下载软件的合法性。

```bash
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg

```

> **注意：** 如果您使用的是 **Debian**，请将上述 URL 中的 `ubuntu` 替换为 `debian`。

### 4. 设置稳定存储库

将 Docker 仓库添加到系统的源列表中。

```bash
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

```

> **注意：** 同样，如果是 **Debian** 系统，请将 URL 中的 `ubuntu` 替换为 `debian`。

### 5. 安装 Docker Engine

更新源并安装 Docker 核心组件。

```bash
sudo apt-get update
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-compose-plugin

```

### 6. 验证安装

运行 `hello-world` 镜像来验证 Docker 是否安装成功。

```bash
sudo docker run hello-world

```

### 7. (可选) 免 Sudo 使用 Docker

默认情况下运行 Docker 命令需要 `sudo`。如果您希望以普通用户身份运行，请执行以下命令：

```bash
# 创建 docker 组（通常安装时已自动创建）
sudo groupadd docker

# 将当前用户加入 docker 组
sudo usermod -aG docker $USER

# 激活更改（或者退出终端重新登录）
newgrp docker

```

---

## 第二部分：Docker 代理配置详解

很多新手在配置代理时容易混淆。Docker 的网络环境主要分为三个场景，需要分别配置：

1. **Daemon 代理：** 解决 `docker pull` 拉取国外镜像慢/失败的问题。
2. **Build 代理：** 解决 `docker build` 过程中 `apt-get` 或 `pip` 慢的问题。
3. **Container 代理：** 解决容器运行后，内部应用访问外网的问题。

### 场景一：配置 Docker Daemon 代理 (解决 pull 问题)

由于 Docker Daemon 是由 systemd 管理的服务，它无法读取 Shell 的环境变量（如 `.bashrc`），必须修改 systemd 配置。

**步骤：**

1. **创建配置目录**
```bash
sudo mkdir -p /etc/systemd/system/docker.service.d

```


2. **创建代理配置文件**
```bash
sudo vim /etc/systemd/system/docker.service.d/http-proxy.conf

```


3. **写入配置**
将 `127.0.0.1:7890` 替换为您的实际代理地址。
```ini
[Service]
Environment="HTTP_PROXY=http://127.0.0.1:7890"
Environment="HTTPS_PROXY=http://127.0.0.1:7890"
Environment="NO_PROXY=localhost,127.0.0.1,docker-registry.yourcompany.com"

```


> **重要提示：** `NO_PROXY` 必须包含 `localhost` 和 `127.0.0.1`，否则 Docker 内部通信会出错。


4. **应用更改**
```bash
sudo systemctl daemon-reload
sudo systemctl restart docker

```


5. **验证**
```bash
docker info | grep Proxy

```



---

### 场景二：全局容器代理 (自动注入)

如果您希望以后 **新建** 的所有容器（无论是 run 还是 build）都自动走代理，可以修改 Docker 客户端配置文件。

**步骤：**

1. 编辑 `~/.docker/config.json` (如果没有则新建)：
```bash
vim ~/.docker/config.json

```


2. 添加 `proxies` 字段：
```json
{
 "proxies": {
   "default": {
     "httpProxy": "http://127.0.0.1:7890",
     "httpsProxy": "http://127.0.0.1:7890",
     "noProxy": "localhost,127.0.0.1"
   }
 }
}

```



此后，执行 `docker run` 或 `docker build` 时，Docker 会自动将这些环境变量注入到容器内。

---

### 场景三：临时指定代理 (最灵活)

如果不希望全局配置，可以在命令中临时指定。

#### 1. 构建镜像时 (`docker build`)

推荐使用 `--build-arg`，这样不会把代理配置写死在 Dockerfile 里，方便分享镜像。

```bash
docker build \
  --build-arg HTTP_PROXY=http://127.0.0.1:7890 \
  --build-arg HTTPS_PROXY=http://127.0.0.1:7890 \
  -t my-image .

```

#### 2. 运行容器时 (`docker run`)

```bash
docker run -d \
  -e HTTP_PROXY=http://127.0.0.1:7890 \
  -e HTTPS_PROXY=http://127.0.0.1:7890 \
  nginx

```

---

## 特别注意：关于 127.0.0.1 的陷阱

在 Docker 容器内部，`127.0.0.1` 指向的是**容器自身**，而不是宿主机。
如果您在宿主机上运行代理软件（如 Clash, V2Ray），直接在容器内填 `127.0.0.1:7890` 是无法连接的。

**解决方案：**

1. **使用宿主机局域网 IP：** 查看宿主机 IP (如 `192.168.1.5`)，配置 `HTTP_PROXY=http://192.168.1.5:7890`。
2. **允许代理软件监听局域网：** 确保您的代理软件设置中开启了 "Allow LAN" 或监听 `0.0.0.0`，而不仅仅是监听 `127.0.0.1`。
3. **使用 host 网络 (仅限 Linux)：** 运行容器时添加 `--network host`，此时容器共享宿主机网络栈，可以直接使用 `127.0.0.1`。