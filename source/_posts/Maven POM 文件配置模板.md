---
title: Maven POM 文件配置模板
abbrlink: 531040bd
date: 2026-01-22 12:13:11
tags:
  - java
categories:
  - java
toc: true
---


## 前言

在日常开发、写 Demo 验证想法，或者刷算法题时，我们往往需要一个干净但功能齐全的 `pom.xml`。每次都要手动去 Maven Repository 找依赖版本非常影响效率。

本文整理了两套我自用的 Maven 配置模板（分别对应 **JDK 8** 和 **JDK 17**）。相比于基础版本，这次更新集成了开发中最常用的“瑞士军刀”级依赖，并新增了**单元测试**与**Fat Jar 打包**支持，真正做到“开箱即用”。

建议收藏或直接复制使用，省去繁琐的配置时间。

## 核心集成说明

这两套模板均包含以下全栈基础库与插件，满足绝大多数“快速原型”开发的需要：

### 1. 开发利器

* **Lombok**: 消除 Getter/Setter/Builder 等样板代码。
* **Guava**: Google 出品的 Java 核心增强库（集合、缓存、并发工具等）。
* **Commons-Lang3**: Apache 的 Java 基础工具类增强。
* **Gson**: 轻量且高效的 JSON 序列化/反序列化工具。
* **Logback**: 简单易用的日志实现。

### 2. 测试与构建

* **JUnit 5 (Jupiter) & Mockito**: 现代化的单元测试与 Mock 框架，方便验证算法逻辑。
* **Maven Shade Plugin**: 配置了 Fat Jar 打包，生成的 Jar 包包含所有依赖，可直接通过 `java -jar` 运行。
* **Maven Compiler Plugin**: 明确指定了源码与目标版本，避免编码告警。

---

## 1. JDK 1.8 通用模板

适用于维护旧项目或在该版本下验证基础语法的场景。依赖版本经过挑选，稳定兼容 Java 8。

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>cn.liboshuai.example</groupId>
    <artifactId>example-jdk8</artifactId>
    <version>1.0-SNAPSHOT</version>

    <properties>
        <maven.compiler.source>1.8</maven.compiler.source>
        <maven.compiler.target>1.8</maven.compiler.target>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
        <project.reporting.outputEncoding>UTF-8</project.reporting.outputEncoding>

        <logback-version>1.2.12</logback-version>
        <lombok-version>1.18.30</lombok-version>
        <commons-lang3.version>3.12.0</commons-lang3.version>
        <guava.version>31.1-jre</guava.version>
        <gson.version>2.9.1</gson.version>
        <jupiter.version>5.8.2</jupiter.version>
        <mockito.version>4.5.1</mockito.version>
        <maven-compiler-plugin.version>3.10.1</maven-compiler-plugin.version>
        <maven-surefire-plugin.version>2.22.2</maven-surefire-plugin.version>
        <maven-shade-plugin.version>3.3.0</maven-shade-plugin.version>
    </properties>

    <dependencies>
        <dependency>
            <groupId>ch.qos.logback</groupId>
            <artifactId>logback-classic</artifactId>
            <version>${logback-version}</version>
        </dependency>
        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <version>${lombok-version}</version>
            <scope>provided</scope>
        </dependency>
        <dependency>
            <groupId>org.apache.commons</groupId>
            <artifactId>commons-lang3</artifactId>
            <version>${commons-lang3.version}</version>
        </dependency>
        <dependency>
            <groupId>com.google.guava</groupId>
            <artifactId>guava</artifactId>
            <version>${guava.version}</version>
        </dependency>
        <dependency>
            <groupId>com.google.code.gson</groupId>
            <artifactId>gson</artifactId>
            <version>${gson.version}</version>
        </dependency>
        <dependency>
            <groupId>org.junit.jupiter</groupId>
            <artifactId>junit-jupiter</artifactId>
            <version>${jupiter.version}</version>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>org.mockito</groupId>
            <artifactId>mockito-core</artifactId>
            <version>${mockito.version}</version>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>org.mockito</groupId>
            <artifactId>mockito-junit-jupiter</artifactId>
            <version>${mockito.version}</version>
            <scope>test</scope>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-compiler-plugin</artifactId>
                <version>${maven-compiler-plugin.version}</version>
                <configuration>
                    <source>${maven.compiler.source}</source>
                    <target>${maven.compiler.target}</target>
                    <encoding>${project.build.sourceEncoding}</encoding>
                </configuration>
            </plugin>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-surefire-plugin</artifactId>
                <version>${maven-surefire-plugin.version}</version>
            </plugin>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-shade-plugin</artifactId>
                <version>${maven-shade-plugin.version}</version>
                <configuration>
                    <createDependencyReducedPom>false</createDependencyReducedPom>
                </configuration>
                <executions>
                    <execution>
                        <phase>package</phase>
                        <goals>
                            <goal>shade</goal>
                        </goals>
                        <configuration>
                            <transformers>
                                <transformer implementation="org.apache.maven.plugins.shade.resource.ManifestResourceTransformer">
                                </transformer>
                            </transformers>

                            <filters>
                                <filter>
                                    <artifact>*:*</artifact>
                                    <excludes>
                                        <exclude>META-INF/*.SF</exclude>
                                        <exclude>META-INF/*.DSA</exclude>
                                        <exclude>META-INF/*.RSA</exclude>
                                    </excludes>
                                </filter>
                            </filters>
                        </configuration>
                    </execution>
                </executions>
            </plugin>
        </plugins>
    </build>

</project>
```

---

## 2. JDK 17 进阶模板

适用于新特性学习或新项目起手。在此版本中，我对部分依赖进行了全面升级（如 Guava 33.x，JUnit 5.12.x），以确保与新版 JDK 的最佳兼容性。

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    
    <groupId>cn.liboshuai.example</groupId>
    <artifactId>example-jdk17</artifactId>
    <version>1.0-SNAPSHOT</version>

    <properties>
        <maven.compiler.source>17</maven.compiler.source>
        <maven.compiler.target>17</maven.compiler.target>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
        <project.reporting.outputEncoding>UTF-8</project.reporting.outputEncoding>

        <logback-version>1.2.12</logback-version>
        <lombok-version>1.18.30</lombok-version>
        <commons-lang3.version>3.17.0</commons-lang3.version>
        <guava.version>33.4.8-jre</guava.version>
        <gson.version>2.13.2</gson.version>
        <junit-jupiter.version>5.12.2</junit-jupiter.version>
        <mockito.version>5.17.0</mockito.version>
        <maven-compiler-plugin.version>3.14.1</maven-compiler-plugin.version>
        <maven-surefire-plugin.version>3.5.4</maven-surefire-plugin.version>
        <maven-shade-plugin.version>3.6.1</maven-shade-plugin.version>
    </properties>

    <dependencies>
        <dependency>
            <groupId>ch.qos.logback</groupId>
            <artifactId>logback-classic</artifactId>
            <version>${logback-version}</version>
        </dependency>
        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <version>${lombok-version}</version>
            <scope>provided</scope>
        </dependency>
        <dependency>
            <groupId>org.apache.commons</groupId>
            <artifactId>commons-lang3</artifactId>
            <version>${commons-lang3.version}</version>
        </dependency>
        <dependency>
            <groupId>com.google.guava</groupId>
            <artifactId>guava</artifactId>
            <version>${guava.version}</version>
        </dependency>
        <dependency>
            <groupId>com.google.code.gson</groupId>
            <artifactId>gson</artifactId>
            <version>${gson.version}</version>
        </dependency>
        <dependency>
            <groupId>org.junit.jupiter</groupId>
            <artifactId>junit-jupiter</artifactId>
            <version>${junit-jupiter.version}</version>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>org.mockito</groupId>
            <artifactId>mockito-core</artifactId>
            <version>${mockito.version}</version>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>org.mockito</groupId>
            <artifactId>mockito-junit-jupiter</artifactId>
            <version>${mockito.version}</version>
            <scope>test</scope>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-compiler-plugin</artifactId>
                <version>${maven-compiler-plugin.version}</version>
                <configuration>
                    <source>${maven.compiler.source}</source>
                    <target>${maven.compiler.target}</target>
                    <encoding>${project.build.sourceEncoding}</encoding>
                </configuration>
            </plugin>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-surefire-plugin</artifactId>
                <version>${maven-surefire-plugin.version}</version>
            </plugin>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-shade-plugin</artifactId>
                <version>${maven-shade-plugin.version}</version>
                <configuration>
                    <createDependencyReducedPom>false</createDependencyReducedPom>
                </configuration>
                <executions>
                    <execution>
                        <phase>package</phase>
                        <goals>
                            <goal>shade</goal>
                        </goals>
                        <configuration>
                            <transformers>
                                <transformer implementation="org.apache.maven.plugins.shade.resource.ManifestResourceTransformer">
                                </transformer>
                            </transformers>

                            <filters>
                                <filter>
                                    <artifact>*:*</artifact>
                                    <excludes>
                                        <exclude>META-INF/*.SF</exclude>
                                        <exclude>META-INF/*.DSA</exclude>
                                        <exclude>META-INF/*.RSA</exclude>
                                    </excludes>
                                </filter>
                            </filters>
                        </configuration>
                    </execution>
                </executions>
            </plugin>
        </plugins>
    </build>

</project>
```

---

## 小贴士

1. **Guava 版本差异**：请注意 JDK 17 模板中使用了 `33.x` 版本，而 JDK 8 模板保留在 `31.x`。这是为了避免在新版 Java 中出现模块化访问限制的警告。
2. **关于打包**：新增的 `maven-shade-plugin` 会将所有依赖打入一个 Fat Jar 中。执行 `mvn package` 后，你可以直接使用 `java -jar target/xxx.jar` 运行你的程序，方便部署测试。
3. **切勿照搬 GAV**： 请不要直接复制 `<groupId>`、`<artifactId>` 等元信息，记得替换成你自己的项目坐标。

希望这两段升级版的配置能为你的“开荒”工作提速！