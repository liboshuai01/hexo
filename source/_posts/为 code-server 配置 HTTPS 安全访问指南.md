---
title: 为 code-server 配置 HTTPS 安全访问指南
abbrlink: 65b7d8fd
date: '2025-07-18T00:44:08.000Z'
tags:
  - 杂货小铺
categories:
  - 杂货小铺
toc: true
---


## 为什么需要 HTTPS？
默认情况下，code-server 通过 HTTP 协议提供服务，所有传输数据（包括密码、代码文件等敏感信息）均以明文传输，存在被中间人攻击窃取的风险。HTTPS 通过 SSL/TLS 加密通信，可有效防止数据泄露和篡改，是生产环境必备的安全措施。

---

## 完整配置步骤

### 1️⃣ 获取 SSL 证书
推荐使用阿里云免费 SSL 证书（每年 20 个免费额度）：
1. 登录 [阿里云 SSL 证书控制台](https://yundun.console.aliyun.com/?p=cas#/)
2. 申请免费 DV 证书，完成域名验证
3. 证书签发后，选择「下载」证书，类型选择 **Apache**

### 2️⃣ 准备证书文件
下载的压缩包包含三个关键文件：
```
your_domain.key          # 私钥文件
your_domain_public.crt   # 公钥证书
your_domain_chain.crt    # 证书链
```

### 3️⃣ 创建证书目录
在服务器上创建专用目录存放证书：
```bash
mkdir -p ~/.local/share/code-server/cert
```
将下载的三个证书文件上传至此目录，结构如下：
```
/root/.local/share/code-server/cert/
├── your_domain.key
├── your_domain_public.crt
└── your_domain_chain.crt
```

### 4️⃣ 合并证书文件
**关键步骤**：将公钥证书与证书链合并，确保浏览器信任：
```bash
cd ~/.local/share/code-server/cert
cat your_domain_public.crt your_domain_chain.crt > combined.crt
```

### 5️⃣ 修改 code-server 配置
编辑配置文件：
```bash
vi ~/.config/code-server/config.yaml
```
添加以下关键配置：
```yaml
bind-addr: 0.0.0.0:8080  # 监听端口
auth: password
password: your_strong_password  # 登录密码
cert: /root/.local/share/code-server/cert/combined.crt
cert-key: /root/.local/share/code-server/cert/your_domain.key
```

### 6️⃣ 重启 code-server
先终止旧进程：
```bash
pkill -f code-server
```
启动新服务（推荐后台运行）：
```bash
nohup code-server &> /var/log/code-server.log &
```

---

## 验证 HTTPS 访问
浏览器访问 `https://your-domain.com:8080`：
1. 地址栏显示 🔒 安全锁标志
2. 点击锁标志可查看证书详情
3. 使用配置的密码登录

> **故障排查**：若遇证书错误，检查：
> 1. 证书路径是否正确
> 2. 合并证书步骤是否执行（`combined.crt`）
> 3. 服务器防火墙是否开放端口

---

通过以上步骤，您的在线开发环境安全性将得到显著提升，保护代码资产免受窃取风险。