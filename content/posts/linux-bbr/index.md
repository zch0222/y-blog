---
date: '2026-02-14T23:16:43+08:00'
draft: false
title: 'Linux 开启BBR'
summary: "Linux 开启BBR教程"
---

# Linux 服务器开启 BBR 加速教程

Google BBR (Bottleneck Bandwidth and RTT) 是一种 TCP 拥塞控制算法，能显著降低网络延迟并提高吞吐量。

### 前置条件

* **虚拟化架构**：KVM, Xen, VMWare, Hyper-V 等（OpenVZ 通常不支持）。
* **内核版本**：Linux Kernel **4.9** 及以上。

---

## 第一步：检查内核版本

BBR 需要 Linux 内核版本 4.9+。在终端输入以下命令检查：

```bash
uname -r

```

* 如果显示版本号 `>= 4.9`（例如 `5.4.0-xxx`），请直接进行下一步。
* 如果版本号 `< 4.9`，请先升级内核（CentOS 7 或旧版 Ubuntu 需先升级）。

---

## 第二步：开启 BBR

执行以下命令，将 BBR 配置写入系统参数并生效：

```bash
# 1. 修改系统配置文件
echo "net.core.default_qdisc=fq" >> /etc/sysctl.conf
echo "net.ipv4.tcp_congestion_control=bbr" >> /etc/sysctl.conf

# 2.重新加载配置
sysctl -p

```

---

## 第三步：验证开启状态

依次执行以下命令进行验证：

**1. 验证模块是否加载**

```bash
lsmod | grep bbr

```

* **成功标准**：输出包含 `tcp_bbr` 字样。

**2. 验证拥塞控制算法**

```bash
sysctl net.ipv4.tcp_congestion_control

```

* **成功标准**：输出为 `net.ipv4.tcp_congestion_control = bbr`。

**3. 验证队列调度算法**

```bash
sysctl net.core.default_qdisc

```

* **成功标准**：输出为 `net.core.default_qdisc = fq`。

---

### 常见问题

* **重启后失效？**
如果重启后失效，请检查 `/etc/sysctl.conf` 中是否有重复或冲突的配置。
* **OpenVZ 无法开启？**
OpenVZ 架构通常无法直接更换内核，建议使用服务商提供的特定 BBR 开启方案或更换 KVM 架构 VPS。
