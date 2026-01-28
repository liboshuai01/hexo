---
title: Git的.gitignore模板和失效修复
abbrlink: d98067fd
date: 2026-01-20 03:38:10
tags:
  - 杂货小铺
categories:
  - 杂货小铺
toc: true
---


> **核心用途**：一键复制通用的忽略规则；解决配置了忽略文件但不生效的“鬼打墙”问题。

## 1. 最佳实践模板

在项目根目录新建 `.gitignore` 文件，直接粘贴以下内容（涵盖 IDEA, Eclipse, VSCode, Mac 及 Maven）：

```gitignore
target/
!.mvn/wrapper/maven-wrapper.jar
!**/src/main/**/target/
!**/src/test/**/target/
.kotlin

### IntelliJ IDEA ###
.idea/*
!.idea/vcs.xml
!.idea/icon.png
!.idea/bookmarks.json
!.idea/code-remark.xml
*.iws
*.iml
*.ipr

### Eclipse ###
.apt_generated
.classpath
.factorypath
.project
.settings
.springBeans
.sts4-cache

### NetBeans ###
/nbproject/private/
/nbbuild/
/dist/
/nbdist/
/.nb-gradle/
build/
!**/src/main/**/build/
!**/src/test/**/build/

### VS Code ###
.vscode/

### Mac OS ###
.DS_Store

```

---

## 2. 忽略不生效？急救方案！

**现象**：明明已经把 `.idea/` 或 `target/` 加到了上面的文件中，但 `git status` 依然显示它们被修改了。
**原因**：这些文件在配置 `.gitignore` 之前就已经被 Git **缓存（Tracked）** 过了。
**解决**：必须先从 Git 缓存中移除它们（不会删除本地文件）。

请按顺序执行以下命令：

```bash
# 1. 清除所有文件的缓存（这一步最关键，让 Git 重新读取忽略规则）
git rm -r --cached .

# 2. 重新添加所有文件（此时 Git 会自动过滤掉忽略文件）
git add .

# 3. 提交更改
git commit -m "fix: update .gitignore configuration"

```

执行完上述三步，世界就清静了。