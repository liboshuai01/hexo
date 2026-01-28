---
title: Debian 13 安装 Docker 与最佳配置
abbrlink: dde93a6a
date: 2026-01-27 12:55:35
tags:
  - 环境搭建
categories:
  - 环境搭建
toc: true
---


## 前言

Debian 13 (Trixie) 自带源里的 Docker 版本通常较老，且默认配置不适合高负载开发。本文记录了在 Debian 13 上安装官方 Docker CE 并进行生产级配置的全过程。

## 1. 安装官方 Docker CE

为了获取最新特性和安全补丁，推荐直接使用 Docker 官方仓库安装。

### 清理旧版本

首先移除系统可能预装的旧版本或非官方版本：

```bash
sudo apt-get remove docker.io docker-doc docker-compose podman-docker containerd runc
```

### 配置官方源

**1. 安装必要工具**

```bash
sudo apt-get update
sudo apt-get install -y ca-certificates curl gnupg
```

**2. 添加 GPG 密钥**

```bash
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL [https://download.docker.com/linux/debian/gpg](https://download.docker.com/linux/debian/gpg) | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg
```

**3. 写入源 (自动识别 Debian 版本)**

```bash
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] [https://download.docker.com/linux/debian](https://download.docker.com/linux/debian) \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

### 安装 Docker Engine

```bash
sudo apt-get update
sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

## 2. 免 Sudo 配置

将当前用户加入 `docker` 组，避免每次执行命令都要输密码，这对开发体验至关重要。

```bash
sudo usermod -aG docker $USER
```

> **注意**：执行完上述命令后，需要**注销并重新登录**，或者在当前终端执行 `newgrp docker` 才能立即生效。

## 3. 核心配置优化 (daemon.json)

这是最关键的一步。我们需要调整 Docker 的 Cgroup 驱动以匹配 Debian 的 Systemd，并限制日志大小防止磁盘爆满。

编辑配置文件：

```bash
sudo vim /etc/docker/daemon.json
```

写入以下配置：

```json
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m",
    "max-file": "3"
  },
  "live-restore": true,
  "max-concurrent-downloads": 10,
  "default-address-pools": [
    {
      "base": "10.0.0.0/8",
      "size": 24
    }
  ],
  "data-root": "/var/lib/docker"
}
```

### 配置详解

* **`native.cgroupdriver=systemd`**: **必须配置**。让 Docker 使用 systemd 管理资源（Cgroups），避免在高负载下与系统进程管理器冲突，造成不稳定。
* **`live-restore`**: 开启热重载。当 Docker 守护进程（Daemon）更新或重启时，运行中的容器**不会停止**。这对于生产环境升级和开发调试都非常友好。
* **`log-opts`**: 限制单个容器日志最大为 100MB，且只保留 3 个历史文件。防止某个容器疯狂报错导致服务器磁盘瞬间被撑爆。
* **`max-concurrent-downloads`**: 提高并行下载层数，加快 `docker pull` 拉取大镜像的速度。

## 4. 应用配置

配置完成后，重载配置并重启 Docker 服务：

```bash
sudo systemctl daemon-reload
sudo systemctl restart docker
```

**验证安装：**

```bash
docker run --rm hello-world
```

如果看到 "Hello from Docker!" 的欢迎语，说明安装与配置均已成功。
