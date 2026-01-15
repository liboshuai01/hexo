---
title: RockyLinux 镜像源替换实战指南
abbrlink: 8120de54
date: '2025-06-08T09:58:31.000Z'
tags:
  - 运维手册
categories:
  - 运维手册
toc: true
---


本指南提供一套精简的 Rocky Linux 及 EPEL 镜像源替换命令，以**阿里云镜像**为例。所有命令均需 `root` 或 `sudo` 权限。

<!-- more -->

---

## 一、一键替换与刷新

直接复制并执行以下所有命令，即可完成从备份到生效的全过程。

```bash
# =================== 核心替换脚本 (以阿里云为例) ===================

# 1. 备份原始 repo 文件 (整目录备份，安全可靠)
sudo cp -r /etc/yum.repos.d /etc/yum.repos.d.bak

# 2. 替换 Rocky Linux 基础镜像源
# (注释 mirrorlist，启用并替换 baseurl，同时自动创建 .repo.bak 文件)
sudo sed -e 's|^mirrorlist=|#mirrorlist=|g' \
         -e 's|^#baseurl=http://dl.rockylinux.org/$contentdir|baseurl=https://mirrors.aliyun.com/rockylinux|g' \
         -i.bak \
         /etc/yum.repos.d/Rocky-*.repo

# 3. 安装并替换 EPEL 镜像源
sudo dnf install -y epel-release
sudo sed -e 's!^metalink=!#metalink=!g' \
         -e 's!^#baseurl=!baseurl=!g' \
         -e 's!https\?://download\.fedoraproject\.org/pub/epel!https://mirrors.aliyun.com/epel!g' \
         -e 's!https\?://download\.example/pub/epel!https://mirrors.aliyun.com/epel!g' \
         -i.bak /etc/yum.repos.d/epel{,-testing}.repo

# 4. 清理旧缓存并生成新缓存
sudo dnf clean all
sudo dnf makecache

echo "镜像源替换完成！"
```

> **提示**：如需使用其他镜像源，请将脚本中 `https://mirrors.aliyun.com/rockylinux` 和 `https://mirrors.aliyun.com/epel` 替换为您的目标地址。

---

## 二、验证操作

执行以下命令，分别检查 `baseos` 和 `epel` 仓库的详细配置信息。重点关注输出中的 `Repo-baseurl` 字段是否已更新为您设置的镜像地址。

```bash
# 检查 BaseOS 仓库信息
sudo dnf repoinfo baseos

# 检查 EPEL 仓库信息
sudo dnf repoinfo epel
```

---

## 三、紧急回滚方案

如果遇到问题，请选择以下任一方案快速恢复。

**方案 A：(推荐) 还原被修改的文件**
```bash
# 查找所有 .bak 文件并恢复
sudo find /etc/yum.repos.d -type f -name "*.repo.bak" -exec bash -c 'mv "$1" "${1%.bak}"' _ {} \;

# 刷新缓存
sudo dnf clean all && sudo dnf makecache
```

**方案 B：(彻底) 还原整个目录**
```bash
sudo rm -rf /etc/yum.repos.d
sudo mv /etc/yum.repos.d.bak /etc/yum.repos.d
sudo dnf clean all && sudo dnf makecache
```

---

## 附：国内常用镜像源参考

| 服务商 | Rocky Linux 地址                            | EPEL 地址                                  |
| :----- | :------------------------------------------ | :----------------------------------------- |
| **阿里云** | `https://mirrors.aliyun.com/rockylinux/`    | `https://mirrors.aliyun.com/epel/`         |
| **腾讯云** | `https://mirrors.cloud.tencent.com/rocky/` | `https://mirrors.cloud.tencent.com/epel/`  |
| **中科大** | `https://mirrors.ustc.edu.cn/rocky/`        | `https://mirrors.ustc.edu.cn/epel/`        |
| **上海交大**| `https://mirrors.sjtug.sjtu.edu.cn/rocky/`  | `https://mirrors.sjtug.sjtu.edu.cn/epel/`  |