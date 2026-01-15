---
title: Windows 11 安装 WSL2 与 Ubuntu 24.04 LTS 终极指南（手动导入版）
abbrlink: 682b9c2e
date: '2025-06-17T07:57:23.000Z'
tags:
  - 运维手册
categories:
  - 运维手册
toc: true
---


对于在 Windows 系统上工作的开发者而言，能够在不离开熟悉环境的前提下，无缝使用强大的 Linux 工具链，无疑是一大福音。WSL (Windows Subsystem for Linux) 的出现，尤其是性能卓越的 WSL2，完美地解决了这一需求。

本指南将基于您提供的核心步骤，手把手带您在 Windows 11 上，通过手动导入的方式，安装最新版的 Ubuntu 24.04 LTS。相比于从 Microsoft Store 安装，手动导入方式能让您对安装位置有完全的控制权，更适合有特定磁盘管理需求的用户。

<!-- more -->

## 前期准备

- 一台运行 Windows 11 的电脑。
- 拥有管理员权限的账户。

## **第一步：开启 WSL 相关 Windows 功能**

要运行 WSL，首先需要确保 Windows 系统的相关功能已经启用。这包括 WSL 自身以及其所依赖的虚拟机平台。

1.  打开 **控制面板** -> **程序** -> **启动或关闭 Windows 功能**。
2.  在弹出的窗口中，找到并勾选以下三项：
    *   **Hyper-V**
    *   **适用于 Linux 的 Windows 子系统 (Windows Subsystem for Linux)**
    *   **虚拟机平台 (Virtual Machine Platform)**

    ![](https://lbs-images.oss-cn-shanghai.aliyuncs.com/20250617074214661.png)
    ![](https://lbs-images.oss-cn-shanghai.aliyuncs.com/20250617074338537.png)

3.  点击“确定”后，系统会安装所需文件。完成后，**请务必按照提示重启计算机**，以使设置生效。

## **第二步：安装 WSL2 内核更新包**

WSL2 使用了一个真正的 Linux 内核，以提供比 WSL1 更高的性能和更好的系统调用兼容性。我们需要手动下载并安装这个内核更新包。

1.  访问 WSL 的官方 GitHub Releases 页面：[microsoft/WSL/releases](https://github.com/microsoft/WSL/releases)
2.  找到最新的 "Latest" 版本，下载 `wsl_update_x64.msi` 安装包。
    ![](https://lbs-images.oss-cn-shanghai.aliyuncs.com/20250617074736551.png)
3.  下载完成后，双击运行该 `.msi` 文件，然后一路点击“下一步”直到安装完成。过程非常简单。

> **小提示**：在命令行中运行 `wsl --set-default-version 2` 可以将 WSL2 设置为未来安装所有新发行版的默认版本。

## **第三步：下载 Ubuntu 24.04 LTS 系统镜像**

接下来，我们需要获取 Ubuntu 24.04 的 WSL 专用系统镜像文件。这是一个包含了完整根文件系统的 `tar.gz` 压缩包。

1.  打开 Ubuntu 官方的 WSL 镜像下载地址：[ubuntu-wsl-24.04lts.rootfs.tar.gz](https://cloud-images.ubuntu.com/wsl/releases/24.04/20240423/ubuntu-noble-wsl-amd64-24.04lts.rootfs.tar.gz)
2.  浏览器将自动开始下载 `ubuntu-noble-wsl-amd64-24.04lts.rootfs.tar.gz` 文件。
3.  **最佳实践**：建议您在本地创建一个专门存放 WSL 相关文件的文件夹，例如 `C:\WSL_Downloads`，并将下载好的文件移动到此处，方便管理。

## **第四步：导入并安装 Ubuntu 发行版**

这是手动安装的核心步骤。我们将使用 `wsl` 命令行工具来导入刚刚下载的系统镜像。

1.  **以管理员身份** 打开 **PowerShell** 或 **Windows Terminal**。

2.  **创建安装目录**：决定您想将 Ubuntu 安装在哪个位置。这是一个非常重要的选择，因为它将存放整个 Linux 系统文件。例如，我们选择 `C:\ProgramData\Wsl\Ubuntu24.04`。
    > **注意**：`C:\ProgramData` 是一个隐藏的系统文件夹，适合存放应用数据。如果您希望安装在其他盘，比如 D 盘，也是完全可以的，例如 `D:\WslDistros\Ubuntu24.04`。请先手动创建好这个目标文件夹。

3.  执行导入命令。命令格式如下：
    ```powershell
    wsl --import <自定义发行版名称> <安装位置> <镜像压缩包路径>
    ```
    *   `<自定义发行版名称>`：您为这个 Ubuntu 系统起的名字，例如 `Ubuntu-24.04`。
    *   `<安装位置>`：上一步创建的文件夹路径。
    *   `<镜像压缩包路径>`：第三步下载的 `.tar.gz` 文件的完整路径。

    **示例命令如下：**
    ```powershell
    # 假设您的下载文件在 C:\Users\YourUser\Downloads 目录下
    # 并且您想将系统安装在 C:\ProgramData\Wsl\Ubuntu24.04
    wsl --import Ubuntu-24.04 C:\ProgramData\Wsl\Ubuntu24.04 C:\Users\YourUser\Downloads\ubuntu-noble-wsl-amd64-24.04lts.rootfs.tar.gz
    ```
    执行后，等待片刻，wsl 会自动完成解压和导入过程。

## **第五步：启动并配置新安装的 Ubuntu**

导入成功后，我们就可以启动并进行初次配置了。

1.  **启动 Ubuntu**：在 PowerShell 或 Windows Terminal 中，可以先查看已安装的发行版列表，然后启动它。

    ```powershell
    # 查看所有已安装的 WSL 发行版及其状态
    wsl -l -v
    
    # 启动我们刚刚安装的 Ubuntu-24.04
    wsl -d Ubuntu-24.04
    ```

2.  **创建默认用户（关键步骤）**：
    手动导入的系统默认以 `root` 用户登录。为了安全和方便，我们需要创建一个普通用户并将其设为默认登录用户。

    进入 WSL 后（此时您是 `root` 用户），依次执行以下命令：

    ```bash
    # 1. 为 root 用户设置密码
    passwd root

    # 2. 创建一个新用户（将 'your_username' 替换为您想要的用户名）
    useradd -m -s /bin/bash your_username
    
    # 3. 为新用户设置密码
    passwd your_username
    
    # 4. 将新用户添加到 sudo 用户组，使其拥有管理员权限
    usermod -aG sudo your_username
    
    # 5. 配置 WSL，使其默认使用新用户登录
    # 创建并编辑 wsl.conf 文件
    echo -e "[user]\ndefault = your_username" > /etc/wsl.conf
    
    # 6. 退出 WSL
    exit
    ```

3.  **重启 WSL 使配置生效**：
    回到 PowerShell，完全关闭 WSL 来让新的用户配置生效。

    ```powershell
    wsl --shutdown
    ```

4.  **重新登录**：
    再次启动你的 Ubuntu，这次它就会以你刚创建的普通用户身份登录了！

    ```powershell
    wsl -d Ubuntu-24.04
    ```

## **后续步骤与技巧**

恭喜您！现在您已经拥有一个功能完备的 Ubuntu 24.04 环境了。接下来，您可以：

*   **更新软件包**：进入 Ubuntu 后，首先更新一下软件包列表和已安装的软件。
    ```bash
    sudo apt update && sudo apt upgrade
    ```
*   **使用 Windows Terminal**：为了获得最佳的命令行体验（支持多标签、美化等），强烈建议使用 [Windows Terminal](https://aka.ms/terminal)。
*   **与 VS Code 集成**：安装 [VS Code 的 WSL 扩展](https://marketplace.visualstudio.com/items?itemName=ms-vscode-remote.remote-wsl)，可以直接在 Windows 中编辑 Linux 里的代码，体验无缝开发。
*   **文件互通**：
    *   在 WSL 中，您的 Windows 盘符被挂载在 `/mnt/`下，例如 `C:` 盘就是 `/mnt/c`。
    *   在 Windows 的文件资源管理器中，地址栏输入 `\\wsl$` 即可访问所有 WSL 发行版的文件系统。

## **总结**

通过以上步骤，我们成功地在 Windows 11 上手动安装并配置了 Ubuntu 24.04 on WSL2。这种方法不仅让我们对安装过程有了更深的理解，也赋予了我们自定义安装路径的灵活性。现在，开启您在 Windows 上的 Linux 之旅吧！