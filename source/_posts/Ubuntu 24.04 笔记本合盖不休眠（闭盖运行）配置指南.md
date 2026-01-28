---
title: Ubuntu 24.04 笔记本合盖不休眠（闭盖运行）配置指南
abbrlink: 56c2e1d2
date: 2026-01-19 00:06:11
tags:
  - 杂货小铺
categories:
  - 杂货小铺
toc: true
---


## 背景

在从旧版本升级到 **Ubuntu 24.04 LTS (GNOME 46)** 后，许多用户发现常用的 **GNOME Tweaks（优化）** 工具中，“通用”标签页下的“合盖不挂起”开关消失了。

对于开发者而言，我们经常需要将笔记本作为本地服务器使用（例如运行 Java 后端服务、Flink 任务或下载大型文件），此时需要**合上盖子但保持系统运行**。既然图形界面入口已取消，最稳妥且持久的方法是直接修改 `systemd-logind` 配置文件。

## 解决方案：修改 logind.conf

此方法属于系统级配置，无论是否登录桌面环境均有效。

### 1. 编辑配置文件

打开终端，使用 `vim` 编辑 `/etc/systemd/logind.conf` 文件：

```bash
sudo vim /etc/systemd/logind.conf
```

### 2. 修改关键参数

在文件中找到 `HandleLidSwitch` 这一行（通常被 `#` 注释掉了）。

* 去掉行首的 `#` 号。
* 将值修改为 `ignore`。

修改前：

```ini
#HandleLidSwitch=suspend
```

**修改后：**

```ini
HandleLidSwitch=ignore
```

> **参数解释：**
> * `ignore`: 忽略合盖动作，系统继续运行（我们需要的）。
> * `suspend`: 挂起/睡眠（系统默认）。
> * `hibernate`: 休眠（需交换分区支持）。
> * `poweroff`: 关机。
>
>

### 3. 应用更改

保存文件（在 vim 中输入 `:wq` 保存并退出），然后重启登录服务：

```bash
sudo systemctl restart systemd-logind
```

> 注意: 执行上述重启服务的命令会导致**当前的图形界面会话立即注销**，所有已打开的应用程序都会关闭。请务必在执行前保存所有未完成的工作！

**或者，您也可以直接重启电脑使配置生效。**

---

## 如何恢复（开启合盖挂起）

如果后续想恢复“合盖自动睡眠”的功能，只需反向操作：

1. 再次编辑配置文件：`sudo nano /etc/systemd/logind.conf`
2. 将 `HandleLidSwitch=ignore` 改回 `HandleLidSwitch=suspend`。
3. 或者直接在行首加上 `#` 注释掉该行（恢复系统默认值）。
4. 执行 `sudo systemctl restart systemd-logind` 或重启电脑。

---

## 进阶配置：仅在接电源时生效

如果你希望**“插电时合盖运行，没插电时合盖省电休眠”**，可以使用 `HandleLidSwitchExternalPower` 参数。

在 `logind.conf` 中进行如下配置：

```ini
# 电池供电时：合盖挂起（默认行为，或者显式设置为 suspend）
HandleLidSwitch=suspend

# 接电源时：合盖忽略（不挂起）
HandleLidSwitchExternalPower=ignore
```

这样可以避免忘记拔电源合盖导致笔记本在背包里发热耗电的问题。
