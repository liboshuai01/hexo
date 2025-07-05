---
title: Linux禁用密码使用密钥登录指南
date: 2025-07-06 06:06:11
tags:
  - Linux
categories:
  - 运维手册
cover: 'https://lbs-images.oss-cn-shanghai.aliyuncs.com/20250706060731912.png'
toc: true
---

还在担心密码被暴力破解吗？本文提供了一份清晰、直接的操作指南，将带你一步步为Linux服务器禁用不安全的密码认证，并配置SSH密钥登录，从根本上加固你的服务器安全防线。

<!-- more -->

## **核心流程**

1.  **本地**：生成密钥对（公钥+私钥）。
2.  **本地 -> 服务器**：上传公钥。
3.  **服务器**：测试密钥登录。
4.  **服务器**：禁用密码登录。

---

## **第一步：在本地计算机生成密钥对**

> **操作位置**：你的个人电脑（Windows/macOS/Linux）终端。

运行以下命令生成 `ed25519` 密钥。它比传统的RSA更安全、更高效。

```bash
ssh-keygen -t ed25519
```

系统会询问你：
1.  **保存位置**: 直接按回车，使用默认路径 (`~/.ssh/id_ed25519`)。
2.  **密钥密码 (Passphrase)**: **强烈建议设置！** 这是私钥的最后一道防线。输入时不可见。

完成后，你将在 `~/.ssh/` 目录下得到两个文件：
*   `id_ed25519`：**私钥**，绝对不能泄露。
*   `id_ed25519.pub`：**公钥**，将要上传到服务器。

---

## **第二步：将公钥上传至服务器**

> **操作位置**：你的个人电脑终端。

使用 `ssh-copy-id` 命令是最简单、最推荐的方式，它能自动处理好服务器上的文件权限。

```bash
# 将 username 替换为你的服务器用户名，your_server_ip 替换为服务器IP
ssh-copy-id username@your_server_ip
```
根据提示输入你的服务器**登录密码**（这可能是你最后一次使用它）。

**备用方案：手动上传**
如果本地没有 `ssh-copy-id`，可按以下步骤手动操作：

1.  **本地**：复制公钥内容。
    ```bash
    cat ~/.ssh/id_ed25519.pub
    # 复制屏幕上输出的 ssh-ed25519 开头的所有内容
    ```

2.  **服务器**：登录后，将公钥写入 `authorized_keys` 文件。
    ```bash
    # 确保 .ssh 目录存在且权限正确 (700)
    mkdir -p ~/.ssh && chmod 700 ~/.ssh

    # 将公钥追加到文件，并设置权限 (600)
    echo "在此处粘贴你复制的公钥字符串" >> ~/.ssh/authorized_keys
    chmod 600 ~/.ssh/authorized_keys
    ```

---

## **第三步：测试密钥登录**

> **操作位置**：你的个人电脑终端。

⚠️ **至关重要的一步！在禁用密码前，务必确认此步成功！**

打开一个 **新终端窗口**，尝试登录：

```bash
ssh username@your_server_ip
```

如果配置正确，系统会提示你输入**密钥密码 (passphrase)** 而不是服务器登录密码。成功登录后，再进行最后一步。

---

## **第四步：在服务器上禁用密码登录**

> **操作位置**：已通过密钥登录的服务器终端。

1.  编辑SSH服务配置文件。
    ```bash
    sudo nano /etc/ssh/sshd_config
    ```

2.  找到并修改以下三项，确保它们的值如下（如果被 `#` 注释，请去掉 `#`）：
    ```conf
    PubkeyAuthentication yes
    PasswordAuthentication no
    ChallengeResponseAuthentication no
    ```
    *   `PubkeyAuthentication yes`：确保密钥认证是开启的。
    *   `PasswordAuthentication no`：**禁用密码认证**。

3.  保存文件并重启SSH服务使配置生效。
    ```bash
    sudo systemctl restart sshd
    ```

---

## **完成！**

恭喜！你的服务器现在已经切换到更安全的密钥登录模式。从现在开始，只有持有对应私钥的设备才能登录，大大提升了服务器的安全性。

**小贴士：**
如果你本地有多个密钥，想用指定的密钥登录，可以使用 `-i` 参数：
```bash
ssh -i /path/to/your/private_key username@your_server_ip
```