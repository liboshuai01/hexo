---
title: Debian 13 开发环境系统优化指南
abbrlink: d2b1bb32
date: 2026-01-23 13:40:06
tags:
  - 环境搭建
categories:
  - 环境搭建
toc: true
---


## 1. 解除资源限制 (Limits)

默认的文件打开数和进程数限制较低，容易导致 `Too many open files` 或无法创建新线程。

编辑 `/etc/security/limits.conf`，在末尾添加：

```conf
# 提升最大打开文件数 (针对 Netty/Kafka/ES)
* soft nofile 65535
* hard nofile 65535

# 提升最大进程/线程数 (针对 Java/Flink)
* soft nproc 65535
* hard nproc 65535
```

> **注**：需注销重新登录后生效。

---

## 2. 内核参数调优 (Sysctl)

编辑 `/etc/sysctl.conf`，追加以下针对 Java 开发环境的优化配置：

```conf
# --- 内存管理 ---
# 降低 Swap 使用倾向 (保护 JVM GC 性能)
vm.swappiness = 10

# 增加虚拟内存映射区域 (Elasticsearch/Flink 启动必备)
vm.max_map_count = 262144

# --- 开发工具 ---
# 增加文件监听上限 (防止 IDEA 索引卡顿)
fs.inotify.max_user_watches = 524288

# --- 网络优化 ---
# 开启 BBR 拥塞控制
net.core.default_qdisc = fq
net.ipv4.tcp_congestion_control = bbr
```

使配置立即生效：

```bash
sudo sysctl -p
```

---

## 3. 禁用透明大页 (THP)

透明大页 (Transparent HugePages) 可能会导致 Java 应用出现延迟抖动，建议在 GRUB 中禁用。

**1. 编辑 GRUB 配置：**
```bash
sudo nano /etc/default/grub
```


**2. 修改 `GRUB_CMDLINE_LINUX_DEFAULT` 行，追加参数 `transparent_hugepage=never`：**
```bash
GRUB_CMDLINE_LINUX_DEFAULT="quiet splash transparent_hugepage=never"
```


**3. 更新 GRUB 并重启：**
```bash
sudo update-grub
sudo reboot
```

---

## 4. 双系统时间同步

解决 Windows 与 Debian 双系统切换后时间相差 8 小时的问题。让 Debian 使用本地时间存储硬件时钟。

```bash
timedatectl set-local-rtc 1 --adjust-system-clock
```

---

## 5. 常用补充工具

### 字体优化

安装 Google Noto CJK 字体，解决中文字体发虚的问题。

```bash
sudo apt install fonts-noto-cjk fonts-noto-cjk-extra
```

### 查找缺失库 (apt-file)

编译代码时如果不知道缺少的 `.h` 或 `.so` 文件属于哪个包，可以使用 `apt-file`。

```bash
sudo apt install apt-file
sudo apt-file update

# 查找示例: 
apt-file search libz.so
```
