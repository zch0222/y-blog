---
date: '2026-02-18T17:49:05+08:00'
draft: false
title: 'Docker DinD 模式 Jenkin 安装'
summary: "实战：宿主机 Nginx 安全加固官方 Docker DinD 模式 Jenkins"
---

本文介绍如何通过宿主机 Nginx 配置 SSL 与 Basic Auth，安全地暴露以官方推荐模式运行的 Jenkins 服务。

## 1. 启动容器环境 (官方标准命令)

按照官方文档，先启动 Docker 守护进程，再启动 Jenkins 容器。

### 启动 Docker 节点 (DinD)

```bash
docker run --name jenkins-docker --rm --detach \
  --privileged --network jenkins --network-alias docker \
  --env DOCKER_TLS_CERTDIR=/certs \
  --volume jenkins-docker-certs:/certs/client \
  --volume jenkins-data:/var/jenkins_home \
  --publish 2376:2376 \
  docker:dind --storage-driver overlay2

```

### 编写并构建定制镜像

官方镜像不含 Docker 客户端。创建 `Dockerfile`：

```dockerfile
FROM jenkins/jenkins:2.541.1-jdk21
USER root
RUN apt-get update && apt-get install -y lsb-release ca-certificates curl && \
    install -m 0755 -d /etc/apt/keyrings && \
    curl -fsSL https://download.docker.com/linux/debian/gpg -o /etc/apt/keyrings/docker.asc && \
    chmod a+r /etc/apt/keyrings/docker.asc && \
    echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] \
    https://download.docker.com/linux/debian $(. /etc/os-release && echo \"$VERSION_CODENAME\") stable" \
    | tee /etc/apt/sources.list.d/docker.list > /dev/null && \
    apt-get update && apt-get install -y docker-ce-cli && \
    apt-get clean && rm -rf /var/lib/apt/lists/*
USER jenkins
RUN jenkins-plugin-cli --plugins "blueocean docker-workflow json-path-api"

```

构建镜像：`docker build -t myjenkins-blueocean:2.541.1-1 .`

### 启动 Jenkins 容器

```bash
docker run --name jenkins-blueocean --restart=on-failure --detach \
  --network jenkins --env DOCKER_HOST=tcp://docker:2376 \
  --env DOCKER_CERT_PATH=/certs/client --env DOCKER_TLS_VERIFY=1 \
  --publish 8080:8080 --publish 50000:50000 \
  --volume jenkins-data:/var/jenkins_home \
  --volume jenkins-docker-certs:/certs/client:ro \
  myjenkins-blueocean:2.541.1-1

```

## 2. Nginx 安全配置 (宿主机)

### 生成加密密码文件

使用 `openssl` 生成兼容性最佳的 MD5 哈希，避免认证死循环：

```bash
# 替换 YXLM 为用户名，123456 为密码
htpasswd -c -m /etc/nginx/.jenkins-htpasswd YXLM
chmod 644 /etc/nginx/.jenkins-htpasswd

```

### 配置 Nginx 反向代理

编辑宿主机配置（如 `/etc/nginx/conf.d/jenkins.conf`）：

```nginx
map $http_upgrade $connection_upgrade {
    default upgrade;
    ''      close;
}

server {
    listen 80;
    server_name jenkins.yypan.cloud;
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl http2;
    server_name jenkins.yypan.cloud;

    ssl_certificate     /etc/letsencrypt/live/jenkins.yypan.cloud/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/jenkins.yypan.cloud/privkey.pem;

    client_max_body_size 100m;

    location / {
        auth_basic "Restricted Access";
        auth_basic_user_file /etc/nginx/.jenkins-htpasswd;

        # 【关键】通过 Nginx 验证后清空 Authorization 头
        # 防止认证信息透传给 Jenkins 导致内部 401 冲突
        proxy_set_header Authorization "";

        proxy_pass http://127.0.0.1:8080;
        
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        # WebSocket 支持
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection $connection_upgrade;
        
        proxy_redirect off;
        proxy_hide_header X-Jenkins;
    }

    # Webhook 免密放行
    location /github-webhook/ {
        auth_basic off;
        proxy_pass http://127.0.0.1:8080/github-webhook/;
        proxy_set_header Host $host;
    }
}

```

## 3. 获取初始密钥与测试

**获取解锁密码：**

```bash
docker exec jenkins-blueocean cat /var/jenkins_home/secrets/initialAdminPassword

```

**验证：**

1. 访问域名，通过 Nginx 弹窗认证。
2. 使用初始密钥进入 Jenkins。
3. 新建任务执行 `sh 'docker version'`，确认 DinD 环境正常。

