---
date: '2026-02-18T14:34:27+08:00'
draft: false
title: 'Linux 服务器添加 Swap (交换空间)'
summary: "Linux Swap管理教程"
---

# Linux 服务器添加 Swap (交换空间) 

本文介绍如何在 Linux 系统中通过创建 Swap 文件来扩展虚拟内存。适用于 Ubuntu, CentOS, Debian 等主流发行版。

## 1. 检查当前 Swap

在操作前，检查系统是否已经存在 Swap 分区。

```bash
free -h

```

如果 `Swap` 行全是 `0B`，说明当前没有 Swap。

## 2. 创建 Swap 文件

使用 `fallocate` 快速创建一个指定大小的文件（推荐）。例如创建 **4GB** 的 Swap：

```bash
sudo fallocate -l 4G /swapfile

```

> **注意**：如果 `fallocate` 提示失败，可以使用 `dd` 命令替代（速度较慢）：
> `sudo dd if=/dev/zero of=/swapfile bs=1G count=4`

## 3. 设置权限

出于安全考虑，Swap 文件只能由 root 用户读写。

```bash
sudo chmod 600 /swapfile

```

## 4. 格式化并启用 Swap

将文件格式化为 Swap 区域并启用。

```bash
# 格式化
sudo mkswap /swapfile

# 启用
sudo swapon /swapfile

```

再次验证是否生效：

```bash
free -h

```

## 5. 设置开机自启

当前启用仅在本次运行有效，重启后会失效。需修改 `/etc/fstab` 文件。

备份 fstab 文件：

```bash
sudo cp /etc/fstab /etc/fstab.bak

```

将配置追加到 fstab 末尾：

```bash
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab

```

## 6. 优化 Swappiness (可选)

`swappiness` 参数控制系统使用 Swap 的积极程度（0-100）。

* **默认值**：60（内存使用约 40% 时开始 Swap）。
* **推荐值**：10（尽量使用物理内存，仅在内存紧张时使用 Swap）。

**临时修改：**

```bash
sudo sysctl vm.swappiness=10

```

**永久修改：**
编辑 `/etc/sysctl.conf` 文件，在末尾添加：

```bash
vm.swappiness=10

```

保存后执行 `sudo sysctl -p` 生效。

---

## 附：如何删除 Swap

如果不再需要，按以下步骤回滚：

1. **关闭 Swap**：
```bash
sudo swapoff /swapfile

```


2. **删除配置**：
编辑 `/etc/fstab`，删除包含 `/swapfile` 的那一行。
3. **删除文件**：
```bash
sudo rm /swapfile

```