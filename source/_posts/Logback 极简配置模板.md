---
title: Logback 极简配置模板
abbrlink: e0281790
date: 2026-01-22 10:28:48
tags:
  - java
categories:
  - java
toc: true
---


## 前言

在搭建验证性项目（Demo/Scratch）或本地调试 Flink/JUC 代码时，我们需要一个既能看清报错，又不会输出太多杂音的日志配置。

这份 `logback.xml` 模板主打**控制台输出**和**彩色高亮**，非常适合本地开发环境。

## 配置文件模板

将以下内容保存为 `src/main/resources/logback.xml`：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration>

    <appender name="STDOUT" class="ch.qos.logback.core.ConsoleAppender">
        <encoder class="ch.qos.logback.classic.encoder.PatternLayoutEncoder">
            <pattern>%d{HH:mm:ss.SSS} [%thread] %highlight(%-5level) %cyan(%logger{36}) - %msg%n</pattern>
        </encoder>
    </appender>

    <logger name="cn.liboshuai" level="DEBUG" />

    <root level="INFO">
        <appender-ref ref="STDOUT" />
    </root>

</configuration>

```

## 配置要点解析

1. **视觉优化**：
   使用了 Logback 的颜色转换符 `%highlight()` 和 `%cyan()`。
* **Level**: 会根据日志级别自动变色（报错时一眼就能看到红色的 ERROR）。
* **Logger**: 使用青色显示类名，与白色的日志内容区分开，视觉干扰更小。


2. **日志降噪**：
* `root` 级别设为 `INFO`：屏蔽掉 Spring、Netty 或 Flink 内部大量的 DEBUG 信息。
* `logger` 单独设为 `DEBUG`：只对自己编写的代码包（如 `cn.liboshuai`）开启调试日志，精准定位问题。


3. **线程信息**：
   保留了 `[%thread]`，这在进行 JUC 并发编程或 Flink 多线程调试时至关重要。
