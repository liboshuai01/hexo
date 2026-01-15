---
title: Nginx 开启 HTTPS：SSL 证书配置指南
abbrlink: a549c8a6
date: 2025-07-28 05:43:33
tags:
  - 运维手册
categories:
  - 运维手册
toc: true
---


在当今的互联网环境中，为你的网站启用 HTTPS 不再是“可选项”，而是“必选项”。HTTPS 不仅能保障数据传输安全、防止内容被篡改，还能提升用户信任度和搜索引擎排名。本文将为你提供一份清晰、精简的 Nginx 配置 SSL 证书的操作指南，让你能快速为自己的网站部署 HTTPS。

<!-- more -->

## 准备工作

在开始配置之前，请确保你已准备好以下内容：

1.  **一台已安装 Nginx 的服务器**：本文所有操作基于 Nginx。
2.  **一个域名**：例如 `your_domain.com`，并已解析到你的服务器 IP。
3.  **SSL 证书文件**：
    *   证书文件（通常是 `.pem` 或 `.crt` 格式），这是公钥证书。
    *   私钥文件（通常是 `.key` 格式），**请务必妥善保管此文件**。

你可以从 Let's Encrypt 获取免费证书，或从其他商业证书颁发机构（CA）购买。

---

## 操作步骤

### 第一步：上传证书文件到服务器

为了方便管理，我们建议将证书和私钥文件存放在一个统一的目录中。一个常见的做法是在 Nginx 配置目录下创建一个 `ssl` 文件夹。

```bash
# 创建 SSL 证书存放目录
sudo mkdir -p /etc/nginx/ssl

# 将你的证书和私钥文件上传或复制到该目录
# 例如，命名为 your_domain.com.pem 和 your_domain.com.key
sudo cp /path/to/your/certificate.pem /etc/nginx/ssl/your_domain.com.pem
sudo cp /path/to/your/private.key /etc/nginx/ssl/your_domain.com.key

# （重要）设置权限，保护私钥文件不被其他用户读取
sudo chmod 600 /etc/nginx/ssl/your_domain.com.key
```

### 第二步：创建 Nginx 站点配置文件

Nginx 的一个最佳实践是为每个网站创建一个独立的配置文件，并存放在 `/etc/nginx/conf.d/` 目录下。这样做便于管理，且不会污染主配置文件 `nginx.conf`。

我们创建一个新的配置文件，例如 `your_domain.com.conf`。

```bash
sudo touch /etc/nginx/conf.d/your_domain.com.conf
```

现在，使用你喜欢的编辑器（如 `vim` 或 `nano`）打开这个文件，并写入以下内容。配置分为两个核心部分：**HTTP 强制跳转** 和 **HTTPS 服务**。

```nginx
# /etc/nginx/conf.d/your_domain.com.conf

# 第 1 部分：HTTP (80端口) 服务器块
# 作用：将所有来自 http://your_domain.com 的请求永久重定向到 https 版本
server {
    listen 80;
    listen [::]:80; # 兼容 IPv6
    server_name your_domain.com; # 替换成你的域名

    # 301 永久重定向到 HTTPS
    return 301 https://$host$request_uri;
}

# 第 2 部分：HTTPS (443端口) 服务器块
# 作用：处理所有 HTTPS 请求
server {
    listen 443 ssl http2; # 开启 ssl 和 http2
    server_name your_domain.com; # 替换成你的域名

    # --- SSL 核心配置 ---
    # 证书文件路径
    ssl_certificate /etc/nginx/ssl/your_domain.com.pem;
    # 私钥文件路径
    ssl_certificate_key /etc/nginx/ssl/your_domain.com.key;

    # --- SSL 性能与安全优化 (推荐) ---
    ssl_session_cache   shared:SSL:1m;
    ssl_session_timeout   5m;
    ssl_ciphers HIGH:!aNULL:!MD5;
    ssl_prefer_server_ciphers   on;
    
    # 允许上传最大 100 MB 的文件，按需调整
    client_max_body_size 100m;

    # --- 网站内容配置 (根据你的需求二选一) ---

    # 场景一：作为静态网站服务器 (例如博客、文档)
    location / {
        root   /var/www/your_project; # 你的网站文件根目录
        index  index.html index.htm;
        try_files $uri $uri/ /index.html; # 适用于单页应用 (SPA)
    }
    
    # 场景二：作为反向代理 (例如 Spring Boot, Node.js, Python后端应用)
    # location / {
    #     proxy_pass http://127.0.0.1:13000; # 代理到你后端应用的地址
    #     proxy_buffering off;                  # 禁用缓冲，并发大的场景慎用
    #
    #     proxy_set_header   X-Real-IP        $remote_addr;
    #     proxy_set_header   Host             $http_host;
    #     proxy_set_header   X-Forwarded-For  $proxy_add_x_forwarded_for;
    #     proxy_set_header   X-Forwarded-Proto  $scheme;
    #     proxy_set_header   Upgrade            $http_upgrade;
    #     proxy_set_header   Connection         "upgrade";
    #
    # }

    # --- 错误页面配置 ---
    error_page 500 502 503 504 /50x.html;
    location = /50x.html {
        root /usr/share/nginx/html;
    }
}
```

**配置重点解读**：

1.  **HTTP to HTTPS 跳转**：第一个 `server` 块监听 80 端口，它的唯一作用就是通过 `return 301` 将所有 HTTP 请求永久重定向到对应的 HTTPS 地址。这是实现全站 HTTPS 的标准做法。
2.  **`listen 443 ssl http2`**：这是 HTTPS 服务的核心指令。`443` 是 HTTPS 的标准端口，`ssl` 指令用于启用 SSL/TLS，`http2` 则启用更高效的 HTTP/2 协议。
3.  **`ssl_certificate` 和 `ssl_certificate_key`**：这是两个**必须正确配置**的指令，分别指向你的证书公钥和私钥文件。
4.  **安全优化**：通过 `ssl_protocols` 和 `ssl_ciphers` 限制只使用安全的协议和加密算法，可以避免你的网站受到已知的漏洞攻击。
5.  **网站内容配置**：根据你的实际用途，选择配置 `location /` 块。
    *   **静态网站**：使用 `root` 指定网站文件的根目录。
    *   **反向代理**：使用 `proxy_pass` 将请求转发给在本地或其他机器上运行的后端服务。

### 第三步：检查配置并重载 Nginx

在应用新配置之前，务必先检查语法是否有误，这可以避免因配置错误导致 Nginx 服务中断。

```bash
# 检查 Nginx 配置文件语法是否正确
sudo nginx -t
```

如果看到如下输出，说明配置无误：

```
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
```

确认无误后，平滑地重载 Nginx 服务以应用新的配置。

```bash
# 平滑重载 Nginx，不会中断现有连接
sudo systemctl reload nginx
```

至此，你的网站已经成功启用了 HTTPS！现在访问 `http://your_domain.com`，浏览器应该会自动跳转到 `https://your_domain.com`，并且地址栏会显示一把安全锁标志。

## 总结

为 Nginx 配置 SSL 证书的核心流程非常清晰：

1.  **准备**：获取域名和证书文件。
2.  **上传**：将证书和私钥放到服务器指定目录。
3.  **配置**：创建站点配置文件，包含一个**HTTP 跳转块**和一个**HTTPS 服务块**。
4.  **生效**：检查并重载 Nginx。

掌握这个流程后，你就可以轻松地为任何网站配置 HTTPS，为你的应用和用户提供一层坚实的安全保障。