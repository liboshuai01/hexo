---
title: GNOME 桌面环境必备软件与插件推荐
abbrlink: 9ca32b82
date: 2026-01-15 02:59:05
tags:
  - 环境搭建
categories:
  - 环境搭建
toc: true
---


## 前言

在升级到 Ubuntu 24.04 LTS 并使用 GNOME 桌面环境后，为了打造一个既美观又高效的开发环境，我整理了一份必备软件和 GNOME 插件清单。

这份清单涵盖了从日常通讯、开发工具到系统美化的各个方面。如果你也是一名追求效率的开发者，这份列表或许能给你一些参考。

## 🛠️ 第一部分：常用软件清单

这部分涵盖了日常开发和办公的基础设施。对于大多数商业软件，推荐直接去官网下载 `.deb` 包进行安装，以获得最佳的兼容性。对于开发环境管理工具（如 NVM, SDKMan），则推荐使用官方脚本安装。

| 软件分类 | 软件名称 | 推荐理由 / 主要用途 | 安装方式 |
| --- | --- | --- | --- |
| **通讯社交** | **Linux QQ** | 官方重构版，Electron 架构，功能基本完善，满足日常沟通。 | 官网下载 (.deb) |
|   | **WeChat (微信)** | 官方原生 Linux 版（基于 QT 或 Electron），解决办公刚需。 | 官网下载 (.deb) |
|   | **Telegram** | 强大的即时通讯工具，Linux 客户端体验极佳。 | 官网下载 (官网包) |
| **网络工具** | **Clash Verge** | 优秀的网络代理客户端，支持 Clash Meta 内核，UI 现代化。 | 官网/GitHub (.deb/AppImage) |
| **开发工具** | **VS Code** | 轻量级代码编辑器，插件生态丰富。 | 官网下载 (.deb) |
|   | **IntelliJ IDEA** | Java 开发神器，不仅是 IDE，更是生产力标准。 | 官网下载 (tar.gz 解压) |
|   | **WindTerm** | 高颜值、高性能的 SSH/Sftp 客户端，自带自动补全。 | GitHub/官网下载 (便携版) |
|   | **Guake** | 下拉式终端（类似 Quake 游戏风格），F12 随时呼出，极客必备。 | `sudo apt install guake` |
| **环境管理** | **NVM** | Node.js 版本管理工具，前端开发必备。 | 官方 Install Script |
|   | **SDKMan** | Java/JVM 生态版本管理工具（Java, Gradle, Maven 等）。 | 官方 Install Script |
| **办公/效率** | **WPS Office** | Linux 下最接近 MS Office 体验的办公套件。 | 官网下载 (.deb) |
|   | **Pot** | 跨平台划词翻译软件，聚合多接口 + OCR，阅读英文文档神器。 | Flatpak |
|   | **Flameshot** | 火焰截图，功能强大的截图与标注工具。 | `sudo apt install flameshot` |
| **系统工具** | **Fcitx5** | 小企鹅输入法 5，比 IBus 更流畅，配合雾凇拼音体验极佳。 | `apt` 安装 + 配置 |
|   | **Timeshift** | 系统备份还原工具，折腾系统的“后悔药”。 | `apt` 或 PPA 安装 |
|   | **GNOME Tweaks** | 优化微调工具，用于管理字体、开机启动项等。 | `sudo apt install gnome-tweaks` |
|   | **Extension Manager** | 专门管理 GNOME 插件的工具，支持搜索和浏览，比浏览器安装更稳。 | Flatpak 或 `apt` 安装 |

---

## 🧩 第二部分：GNOME Shell 插件清单

GNOME 的强大在于其可扩展性。通过 **Extension Manager**，我们可以将朴素的桌面定制成符合个人习惯的工作流。

**前置准备**：请确保已安装 `Extension Manager`。

| 插件名称 (Extension) | 功能简介 | 配置/备注 |
| --- | --- | --- |
| **User Themes** | 允许从用户目录加载 Shell 主题 | 必须开启，美化基础 |
| **Blur My Shell** | 为概览、面板、应用列表添加毛玻璃模糊效果 | 提升颜值神级插件，消除廉价感 |
| **Dash to Dock** | 将原本隐藏的 Dock 栏固定在桌面，高度可定制 | 取代 Ubuntu 默认 Dock，功能更强 |
| **Ubuntu Dock** | Ubuntu 系统自带的 Dock 扩展 | **建议关闭** (与 Dash to Dock 冲突) |
| **Clipboard Indicator** | 剪贴板历史记录管理，支持搜索和固定条目 | 生产力神器，必装 |
| **Vitals** | 在顶栏显示 CPU、内存、温度、网速等实时数据 | 简洁直观，时刻掌握系统状态 |
| **Input Method Panel** | 修复或优化输入法候选框在 GNOME 下的显示 | 配合 Fcitx5 使用效果更佳 |
| **GSConnect** | KDE Connect 的 GNOME 实现，手机与电脑互传文件、同步剪贴板 | 安卓用户必备 |
| **Grand Theft Focus** | 移除窗口焦点被抢占时的恼人弹窗（"xxx is ready"） | 直接将焦点赋予新窗口，不再手动点击 |
| **Lilypad** | 快速便签/记事本，随时记录灵感 | 轻量级，随手记 |
| **Desktop Icons NG (DING)** | 允许在桌面上显示图标和文件 | 找回传统的桌面图标习惯 |
| **Ubuntu Tiling Assistant** | 增强窗口平铺功能，支持类似 Windows 的四角拖拽 | Ubuntu 24.04 自带增强版 |
| **Ubuntu Appindicators** | 托盘图标支持，让后台软件（如微信、QQ）图标常驻顶栏 | 系统默认集成，建议保留 |

---

## 💡 结语

通过以上软件的武装，Ubuntu 24.04 不仅拥有了强大的开发能力，在日常使用的便利性和视觉美观度上也完全不输 Windows 或 macOS。

如果你有其他好用的 Linux 软件推荐，欢迎在评论区留言交流！