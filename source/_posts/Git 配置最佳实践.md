---
title: Git 配置最佳实践
abbrlink: f85c7c38
date: 2026-01-20 00:32:06
tags:
  - 杂货小铺
categories:
  - 杂货小铺
toc: true
---


> **适用场景**：同时使用 Windows 和 Ubuntu 进行开发的 Java 程序员。
> **解决痛点**：换行符冲突 (`LF` vs `CRLF`)、中文文件名乱码、Java 长路径报错。

## 1. 通用配置 (所有系统必输)

无论在 Windows 还是 Linux，请首先执行以下命令打好基础。

```bash
# --- 身份认证 ---
git config --global user.name "Your Name"
git config --global user.email "your.email@example.com"

# --- 显示优化 ---
# 解决 git status/log 中文文件名显示为乱码数字问题
git config --global core.quotepath false

# --- 行为规范 ---
# 推荐：pull 时默认使用 rebase，保持提交历史为一条直线
git config --global pull.rebase true
# 推荐：新仓库默认分支设为 main
git config --global init.defaultBranch main

# --- 效率别名 (可选) ---
git config --global alias.st status
git config --global alias.co checkout
git config --global alias.ci commit
git config --global alias.br branch
# 强力推荐：可视化 Log 查看命令 (输入 git lg 使用)
git config --global alias.lg "log --color --graph --pretty=format:'%Cred%h%Creset -%C(yellow)%d%Creset %s %Cgreen(%cr) %C(bold blue)<%an>%Creset' --abbrev-commit"

```

---

## 2. Windows 专属配置

请打开 PowerShell 或 CMD 执行。

```bash
# --- 核心：换行符策略 (有借有还) ---
# 提交时转 LF，检出时转 CRLF (适配 Windows 编辑器)
git config --global core.autocrlf true

# --- 系统兼容 ---
# 开启长路径支持 (Java 项目必开，防止 Filename too long)
git config --global core.longpaths true
# 忽略文件权限位 (防止 NTFS/Linux 权限差异导致的文件修改误报)
git config --global core.filemode false

```

---

## 3. Linux (Ubuntu) 专属配置

请打开 Terminal 执行。

```bash
# --- 核心：换行符策略 (只进不出) ---
# 提交时转 LF，检出时不转换 (Linux 原生即 LF)
git config --global core.autocrlf input

# --- 系统兼容 ---
# 开启 NTFS 文件名保护 (防止在 Linux 创建 Windows 不合法的文件名)
git config --global core.protectNTFS true

```
