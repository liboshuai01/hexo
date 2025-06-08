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

本指南将提供一套完整、安全、可回滚的镜像源替换方案，帮助您将 Rocky Linux 的软件源切换至国内，从而彻底解决速度瓶颈。我们将遵循更为严谨的运维操作流程：**备份 -> 替换 -> 清理 -> 验证**，并一并处理对于后端服务至关重要的 EPEL (Extra Packages for Enterprise Linux) 镜像源。

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

在对系统关键配置文件进行任何修改之前，备份是不可或缺的黄金法则。这将为我们提供一个快速回滚的“安全网”。我们将采用两种备份方式，提供双重保障。

1.  **整目录备份**：
    我们先将整个 `/etc/yum.repos.d` 目录完整地复制一份，作为最保险的还原点。

    ```bash
    sudo cp -r /etc/yum.repos.d /etc/yum.repos.d.bak
    ```
    此操作创建了一个名为 `yum.repos.d.bak` 的同级目录，包含了所有原始配置文件。

2.  **文件级自动备份**：
    在接下来的替换步骤中，我们将使用的 `sed` 命令自带了 `-i.bak` 参数。这个参数会在修改每个文件的同时，自动创建一个以 `.bak` 结尾的原始文件备份。这为我们提供了更细粒度的、针对单个文件的回滚能力。

> **为何采用 `cp` + `sed -i.bak` 策略？**
> `cp -r` 提供了宏观的、一键还原的能力，是面对灾难性错误的最终防线。而 `sed -i.bak` 则在执行修改的同一刻，为每个被触碰的文件创建了即时备份。这种“宏观+微观”的双重备份策略，是专业运维中追求极致稳定性的体现。

## 第 3 步：核心操作，替换 Rocky Linux 基础镜像源

我们将使用 `sed` 命令直接修改 Rocky Linux 官方的 `.repo` 文件，这种方法精准、高效，且无需担心网络下载问题。

执行以下命令，将官方源地址替换为阿里云镜像地址：
```bash
sudo sed -e 's|^mirrorlist=|#mirrorlist=|g' \
         -e 's|^#baseurl=http://dl.rockylinux.org/$contentdir|baseurl=https://mirrors.aliyun.com/rockylinux|g' \
         -i.bak \
         /etc/yum.repos.d/Rocky-*.repo
```

**命令解析：**
*   `sed -e '...' -e '...'`: `sed` 是一个流编辑器，`-e` 允许我们执行多个编辑指令。
*   `'s|^mirrorlist=|#mirrorlist=|g'`: 第一个指令是在所有以 `mirrorlist=` 开头的行前面加上 `#`，将其注释掉。`mirrorlist` 会动态选择镜像，可能会绕过我们指定的国内源，因此必须禁用。
*   `'s|^#baseurl=...|baseurl=...|g'`: 第二个指令是查找默认被注释掉的 `baseurl` 行，取消其注释 (`#`)，并将其地址替换为阿里云的镜像地址。`$contentdir` 是一个仓库变量，代表了版本号和架构，我们保留它以确保路径正确。
*   `-i.bak`: 这是关键。它指示 `sed` 直接在原始文件上进行修改（in-place），同时为每个被修改的文件创建一个 `.bak` 后缀的备份。例如，`Rocky-BaseOS.repo` 会被修改，并生成一个 `Rocky-BaseOS.repo.bak` 的备份。
*   `/etc/yum.repos.d/Rocky-*.repo`: 指定操作对象为 `/etc/yum.repos.d/` 目录下所有以 `Rocky-` 开头的 `.repo` 文件，一次性处理所有相关的基础仓库。

## 第 4 步：锦上添花，配置 EPEL 镜像源

EPEL 源为企业版 Linux 提供了大量高质量的额外软件包（如 `nginx`, `redis`, `htop`），是我们后端开发和运维的常用工具库。

1.  **安装 EPEL (如果尚未安装)**
    ```bash
    sudo dnf install -y epel-release
    ```
    安装后，系统会生成 `/etc/yum.repos.d/epel.repo` 等相关文件。

2.  **替换 EPEL 镜像源**
    同样，我们使用 `sed` 命令来替换 EPEL 的源。EPEL 的配置文件结构略有不同，它使用 `metalink` 代替了 `mirrorlist`。
    ```bash
    sudo sed -e 's|^metalink=|#metalink=|g' \
             -e 's|^#baseurl=https://download.fedoraproject.org/pub/epel/|baseurl=https://mirrors.aliyun.com/epel/|g' \
             -i.bak /etc/yum.repos.d/epel*.repo
    ```
    这条命令的逻辑与上一步完全相同：注释掉动态寻址的 `metalink`，启用并修改 `baseurl` 指向阿里云的 EPEL 镜像，并为 `epel.repo` 等文件创建 `.bak` 备份。

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
    您应该会看到类似下面的输出，所有仓库都已就绪：
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

如果在验证步骤中遇到任何问题，我们可以利用第二步创建的备份快速恢复到原始状态。

### 方案A：使用 `.bak` 文件进行细粒度回滚

这是首选的回滚方案，它只会还原被我们修改过的文件。

```bash
# 查找所有 .bak 文件，并将其恢复为原始的 .repo 文件 (会覆盖当前文件)
find /etc/yum.repos.d -type f -name "*.repo.bak" -exec sudo bash -c 'mv "$1" "${1%.bak}"' _ {} \;

# 再次清理并重建缓存
sudo dnf clean all
sudo dnf makecache
```

### 方案B：使用目录备份进行完全回滚

如果情况比较复杂，或您想彻底还原到操作前的状态，可以使用此方案。

```bash
# 1. 删除当前的配置目录
sudo rm -rf /etc/yum.repos.d

# 2. 将备份的目录恢复原位
sudo mv /etc/yum.repos.d.bak /etc/yum.repos.d

# 3. 再次清理并重建缓存
sudo dnf clean all
sudo dnf makecache
```
执行后，您的系统将精确地恢复到使用官方镜像源的状态。

## 总结

通过遵循 **备份 -> 替换 -> 清理 -> 验证** 的严谨流程，并采用 `sed` 原地替换这一专业、可靠的技术手段，我们成功地将 Rocky Linux 及其 EPEL 仓库的镜像源切换到了国内高速镜像。这不仅能大幅提升软件包的下载和更新速度，也为我们构建稳定、高效的后端服务环境扫除了一个关键障碍。掌握这项技能，意味着您可以根据服务器所在网络环境，灵活选择最优的软件源，时刻保持高效。