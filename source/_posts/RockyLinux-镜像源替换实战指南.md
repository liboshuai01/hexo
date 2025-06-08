---
title: RockyLinux 镜像源替换实战指南
tags:
  - Linux
  - RockyLinux
categories:
  - 运维手册
cover: 'https://lbs-images.oss-cn-shanghai.aliyuncs.com/20250608095948532.png'
toc: true
abbrlink: 8120de54
date: 2025-06-08 09:58:31
---

对于每一位构建和维护后端系统的工程师而言，一个稳定、高速的软件包环境是保障开发与运维效率的生命线。当您在使用企业级的 Rocky Linux 系统时，其默认的官方软件包镜像源位于海外，国内访问时常因网络延迟或国际线路抖动，导致 `dnf` / `yum` 操作变得异常缓慢，甚至超时失败。这无疑会拖慢新环境的部署、阻碍日常的系统维护与安全更新。

本指南将提供一套完整、安全、可回滚的镜像源替换方案，帮助您将 Rocky Linux 的软件源切换至国内，从而彻底解决速度瓶颈。我们将遵循严谨的运维操作流程：**备份 -> 替换 -> 清理 -> 验证**，并一并处理对于后端服务至关重要的 EPEL (Extra Packages for Enterprise Linux) 镜像源。

<!-- more -->

## 准备工作

在开始之前，请确保您满足以下条件：
1.  一台正在运行的 Rocky Linux 8.x 或 9.x 系统。
2.  拥有 `root` 权限或具备 `sudo` 执行权限的用户。
3.  服务器已连接到互联网。

## 第 1 步：选择合适的国内镜像源

国内有众多高校和云服务商提供了高质量的 Rocky Linux 镜像服务。选择一个合适的镜像源是加速的第一步。

**选择建议：**
一般建议选择**地理位置**或**网络服务商**（例如，您在使用腾讯云服务器，可优先考虑腾讯云镜像）与您服务器最接近的镜像源，以获得最佳网络速度。

以下是国内一些主流、稳定的 Rocky Linux 镜像源地址，可供您选择：

```text
# 云服务商
阿里云镜像：https://mirrors.aliyun.com/rockylinux/
腾讯云镜像：https://mirrors.cloud.tencent.com/rocky/

# 高校
中科大镜像：https://mirrors.ustc.edu.cn/rocky/
上海交大镜像：https://mirrors.sjtug.sjtu.edu.cn/rocky/
西安交大镜像：https://mirrors.xjtu.edu.cn/rocky/
浙江大学镜像：https://mirrors.zju.edu.cn/rocky/
南京大学镜像：https://mirrors.nju.edu.cn/rocky/
山东大学镜像：https://mirrors.sdu.edu.cn/rocky/
兰州大学镜像：https://mirror.lzu.edu.cn/rocky/
大连东软镜像：https://mirrors.neusoft.edu.cn/rocky/
```

**在本篇指南的后续步骤中，我们将以【阿里云镜像】为例进行演示。如果您选择了其他镜像源，请在执行命令时将其中的 URL 替换为您选择的地址。**

## 第 2 步：安全第一，备份原始镜像源配置

在对系统关键配置文件进行任何修改之前，备份是不可或缺的黄金法则。这将为我们提供一个快速回滚的“安全网”。

1.  **创建备份目录**：
    为了保持 `/etc/yum.repos.d/` 目录的整洁，我们先创建一个专门的备份文件夹。

    ```bash
    sudo mkdir -p /etc/yum.repos.d/backup
    ```

2.  **备份所有 `.repo` 文件**：
    将当前所有有效的仓库配置文件移动到备份目录中。

    ```bash
    sudo mv /etc/yum.repos.d/*.repo /etc/yum.repos.d/backup/
    ```

> **为何使用 `mv` 而不是 `cp`？**
> 将原始文件移走可以确保它们不会干扰后续 dnf 的操作，避免因新旧配置文件共存而引发潜在的仓库冲突。

## 第 3 步：核心操作，替换 Rocky Linux 基础镜像源

我们将下载阿里云提供的新的 `.repo` 配置文件。这种方法比使用 `sed` 修改更简单、更不容易出错。

1.  **根据您的 Rocky Linux 版本，下载新的 repo 文件**：
    首先，检查您的系统版本：
    ```bash
    cat /etc/os-release | grep VERSION_ID
    ```
    您会看到类似 `VERSION_ID="8.10"` 或 `VERSION_ID="9.4"` 的输出。

2.  **下载对应的配置文件**

    *   **如果您的系统是 Rocky Linux 8.x**：
        ```bash
        sudo curl -o /etc/yum.repos.d/Rocky-Base.repo https://mirrors.aliyun.com/repo/Rocky-8.repo
        ```

    *   **如果您的系统是 Rocky Linux 9.x**：
        ```bash
        sudo curl -o /etc/yum.repos.d/Rocky-Base.repo https://mirrors.aliyun.com/repo/Rocky-9.repo
        ```
    > **提示**：如果您选择了其他镜像源，请访问该镜像站点的帮助页面，通常会提供一键替换的脚本或直接的 `.repo` 文件下载地址。上述 `curl` 方式是云厂商镜像站点的通用实践。

## 第 4 步：锦上添花，配置 EPEL 镜像源

EPEL 源为企业版 Linux 提供了大量高质量的额外软件包（如 `nginx`, `redis`, `htop`），是我们后端开发和运维的常用工具库。

1.  **安装 EPEL (如果尚未安装)**
    ```bash
    sudo dnf install -y epel-release
    ```
    安装后，系统会生成 `/etc/yum.repos.d/epel.repo` 等相关文件。

2.  **备份并替换 EPEL 镜像源**
    同样，我们先备份原始的 EPEL 配置文件。
    ```bash
    sudo mv /etc/yum.repos.d/epel*.repo /etc/yum.repos.d/backup/
    ```
    然后下载阿里云的 EPEL 配置文件。
    *   **如果您的系统是 Rocky Linux 8.x / RHEL 8.x 系列**：
        ```bash
        sudo curl -o /etc/yum.repos.d/epel.repo https://mirrors.aliyun.com/repo/epel-8.repo
        ```
    *   **如果您的系统是 Rocky Linux 9.x / RHEL 9.x 系列**：
        ```bash
        sudo curl -o /etc/yum.repos.d/epel.repo https://mirrors.aliyun.com/repo/epel-9.repo
        ```

## 第 5 步：焕然一新，清理并重建缓存

为了让系统丢弃旧的元数据，并从新的阿里云镜像源重新获取信息，我们需要执行清理和重建缓存的操作。

```bash
# 清理所有当前已启用的仓库缓存
sudo dnf clean all

# 生成新的缓存，dnf 将会从阿里云服务器拉取元数据
sudo dnf makecache
```
执行 `dnf makecache` 时，您应该能看到终端输出的 URL 地址中包含了 `mirrors.aliyun.com`，并且该过程的速度相比之前应该会有质的飞跃。

## 第 6 步：验证成果，检查镜像源是否生效

现在是检验成果的时刻。

1.  **列出已启用的仓库**：
    执行以下命令，仔细检查输出结果。

    ```bash
    dnf repolist enabled
    ```

    您应该会看到类似下面的输出，所有仓库的 URL 都已指向阿里云：
    ```
    repo id           repo name
    appstream         Rocky Linux 8 - AppStream
    baseos            Rocky Linux 8 - BaseOS
    epel              Extra Packages for Enterprise Linux 8 - x86_64
    extras            Rocky Linux 8 - Extras
    ```
    虽然 `repo name` 可能没有显示 URL，但上一步 `makecache` 的过程已经证明了来源的正确性。

2.  **尝试安装软件包**：
    通过一次实际的安装来感受速度的提升并最终确认一切正常。

    ```bash
    sudo dnf install -y htop
    ```
    如果软件包能够快速下载并成功安装，恭喜您，镜像源替换已圆满完成！

## 有备无患：紧急回滚方案

如果在验证步骤中遇到任何问题（例如，下载的 repo 文件版本错误、网络不通等），我们可以利用第二步的备份快速恢复到原始状态。

执行以下命令即可回滚：

```bash
# 1. 删除当前错误的配置文件
sudo rm -f /etc/yum.repos.d/Rocky-Base.repo /etc/yum.repos.d/epel.repo

# 2. 从备份目录中恢复原始文件
sudo mv /etc/yum.repos.d/backup/*.repo /etc/yum.repos.d/

# 3. 再次清理并重建缓存
sudo dnf clean all
sudo dnf makecache
```

执行后，您的系统将恢复使用官方镜像源。之后您可以重新检查并选择正确的镜像源，再按照本文流程进行操作。

## 总结

通过遵循 **选择 -> 备份 -> 替换 -> 清理 -> 验证** 的严谨流程，我们成功地将 Rocky Linux 及其 EPEL 仓库的镜像源切换到了国内高速镜像。这不仅能大幅提升软件包的下载和更新速度，也为我们构建稳定、高效的后端服务环境扫除了一个关键障碍。掌握这项技能，意味着您可以根据服务器所在网络环境，灵活选择最优的软件源，时刻保持高效。