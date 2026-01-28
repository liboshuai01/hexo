---
title: Debian 13.3 GNOME 桌面环境软件与插件清单
abbrlink: fa66d41c
date: 2026-01-23 11:45:50
tags:
  - 杂货小铺
categories:
  - 杂货小铺
toc: true
---


## 前言

随着 Debian 13.3 (Stable) 的发布，系统的稳定性与兼容性达到了新的高度。作为一个追求极致稳定开发的程序员，我将工作流完全迁移到了这个版本。

Debian 13.3 搭载了“原汁原味”的 GNOME 桌面，没有任何预装的冗余组件。这种纯净带来了系统资源的极致低占用，但也意味着我们需要手动配置，才能将其打造成顺手的生产力工具。

本文整理了一份在 Debian 13.3 下经过验证的必备软件和 GNOME 插件清单，助你构建一个既稳固又高效的开发环境。

## 🛠️ 第一部分：常用软件清单

这部分涵盖了构建开发工作流的基础设施。Debian 13.3 的软件源策略稳健，对于工具类应用，Flatpak 往往是获取最新版本且不破坏系统依赖的最佳途径。

| 软件分类 | 软件名称 | 推荐理由 / 主要用途 | 安装方式 |
| --- | --- | --- | --- |
| **通讯社交** | **Linux QQ** | 官方重构版，Electron 架构，在 Debian 下运行稳定。 | 官网下载 (.deb) |
|  | **WeChat (微信)** | 官方原生 Linux 版，功能够用，解决办公刚需。 | 官网下载 (.deb) |
|  | **Telegram** | 强大的即时通讯工具，推荐使用官方包以获得自动更新能力。 | 官网下载 (官网包) |
| **网络工具** | **Clash Verge** | 优秀的网络代理客户端，支持 Clash Meta 内核。 | 官网/GitHub (.deb) |
| **开发工具** | **Git** | 分布式版本控制系统，现代软件协作开发的基石。 | apt |
|  | **VS Code** | 轻量级代码编辑器，Debian 下最流行的主力编辑器之一。 | 官网下载 (.deb) |
|  | **IntelliJ IDEA** | Java 开发神器。配合 Debian 稳定的 JDK 环境，开发体验极佳。 | 官网下载 (tar.gz) |
|  | **Apache Maven** | 经典的 Java 项目构建管理工具，手动安装便于精确控制配置。 | 官网下载 (tar.gz) |
|  | **WindTerm** | 高颜值、高性能的 SSH/Sftp 客户端，自带命令补全功能。 | GitHub/官网 (便携版) |
|  | **Guake** | 下拉式终端，F12 随时呼出，极客必备。 | apt |
| **环境管理** | **NVM** | Node.js 版本管理工具，避免直接操作系统的 Node 环境。 | 官方 Install Script |
|  | **SDKMan** | Java/JVM 生态版本管理工具（Java, Gradle, Maven 等）。 | 官方 Install Script |
| **办公/效率** | **WPS Office** | Linux 下文档兼容性最好的办公套件，替代 LibreOffice 的首选。 | 官网下载 (.deb) |
|  | **uTools** | 新一代效率工具平台，插件生态丰富，大幅提升操作效率。 | 官网下载 (.deb) |
|  | **Errands** | 极简主义待办清单，完全遵循 GNOME 设计规范。 | Flatpak |
|  | **Pot** | 跨平台划词翻译软件，聚合多接口 + OCR，阅读英文文档神器。 | Flatpak |
|  | **Flameshot** | 火焰截图，弥补了原生截图工具在标注功能上的不足。 | apt |
| **系统工具** | **trash-cli** | 命令行回收站工具，替代 `rm` 命令防止误删文件，安全必备。 | apt |
|  | **Fcitx5** | 小企鹅输入法 5。Debian 13.3 对 Wayland 的支持已非常成熟。 | apt |
|  | **Timeshift** | 系统备份还原工具，折腾系统时的“安全气囊”。 | apt |
|  | **GNOME Tweaks** | 优化微调工具，用于管理字体、窗口行为等基础设置。 | apt |
|  | **Extension Manager** | 专门管理 GNOME 插件的工具，无需依赖浏览器插件。 | apt |

---

## 🧩 第二部分：GNOME Shell 插件清单

原生的 GNOME 桌面非常“素”——没有 Dock 栏，没有托盘图标。通过 **Extension Manager** 安装插件，是补全 Debian 桌面体验最关键的一步。

**前置准备**：`sudo apt install gnome-shell-extension-manager`

| 插件名称 (Extension)         | 开发者 (Author) | 功能描述 (Description)                                                      |
| ---------------------------- | --------------- | --------------------------------------------------------------------------- |
| **User Themes**              | fmuellner       | **基础增强**：允许从用户目录加载 Shell 主题，是美化的基础依赖。             |
| **Blur My Shell**            | aunetx          | **视觉美化**：消除原生界面的灰暗感，为面板添加磨砂玻璃特效。                |
| **Dash to Dock**             | michele_g       | **核心交互**：原生 GNOME 只有活动视图才有 Dock，此插件可让 Dock 常驻。      |
| **Clipboard Indicator**      | Tudmotu         | **效率神器**：顶栏剪贴板历史记录，弥补 GNOME 无剪贴板管理的短板。           |
| **Vitals**                   | corecoding      | **系统监控**：在顶栏实时显示 CPU、内存及温度，时刻掌握负载。                |
| **Input Method Panel**       | csslayer        | **体验优化**：修复输入法候选框样式，Fcitx5 用户必备。                       |
| **GSConnect**                | dlandau         | **生态互联**：实现安卓手机与电脑的文件/剪贴板同步（KDE Connect 协议）。     |
| **Grand Theft Focus**        | zalckos         | **干扰移除**：移除烦人的“窗口已就绪”通知，直接跳转到新打开的应用。          |
| **Lilypad Top Bar Manager**  | shendrew        | **顶栏管理**：整理顶栏图标，隐藏不常用内容，保持界面清爽。                  |
| **Tiling Assistant**         | Leleat          | **窗口管理**：原生 GNOME 不支持四角分屏，此插件提供类似 Windows 的体验。    |
| **AppIndicator ... Support** | 3v1n0           | **功能补全**：Debian 默认不显示后台应用图标（如 QQ/微信），此插件**必装**。 |

---

## 💡 结语

Debian 13.3 提供了坚实的系统底座，而上述软件和插件则构建了舒适的上层建筑。

通过这套配置，我们既享受了 Debian Stable 带来的极致稳定性，又通过 GNOME 插件补全了现代桌面的交互体验。对于追求“长期稳定使用”的开发者来说，这无疑是一个理想的组合。

如果你在 Debian 13.3 的配置过程中有其他心得，欢迎在评论区分享！