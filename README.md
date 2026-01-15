# 李博帅的技术博客 (Hexo Blog)

这是一个基于 [Hexo](https://hexo.io/) 静态博客框架构建的个人技术博客项目。项目使用了功能强大的 [Butterfly](https://github.com/jenkins-zh/hexo-theme-butterfly) 主题，致力于分享技术心得与生活感悟。

👉 **博客地址**: [https://wiki.liboshuai.cn](https://wiki.liboshuai.cn)  
📚 **搭建教程**: [Centos部署Hexo个人博客实战指南](https://lbs.wiki/pages/b3552560/)

## 🛠 技术栈与特性

本项目采用以下核心技术与配置：

* **核心框架**: [Hexo](https://hexo.io/) (v7.3.0) - 快速、简洁且高效的博客框架。
* **博客主题**: [Butterfly](https://butterfly.js.org/) - 一款追求简约、功能丰富且高度可定制的 Hexo 主题。
* **包管理器**: [pnpm](https://pnpm.io/) (v10.14.0) - 相比 npm/yarn 更快、更节省磁盘空间的包管理工具。
* **渲染引擎**: 
    * `pug` (HTML 模板)
    * `stylus` (CSS 预处理)
* **主要插件**:
    * `hexo-abbrlink`: 生成唯一且简短的文章永久链接 (CRC32算法)。
    * `hexo-generator-searchdb`: 支持本地搜索功能。
    * `hexo-wordcount`: 文章字数统计与阅读时长预计。
    * `hexo-deployer-git`: 自动化 Git 部署。

## 🚀 快速开始 (本地开发)

### 1. 环境准备
确保您的本地环境已安装以下工具：
* [Node.js](https://nodejs.org/) (推荐 LTS 版本)
* [Git](https://git-scm.com/)
* **pnpm**: 全局安装 pnpm (如果尚未安装)
    ```bash
    npm install -g pnpm
    ```

### 2. 获取代码
```bash
git clone <您的仓库地址>
cd <您的仓库目录>

```

### 3. 安装依赖

本项目指定使用 `pnpm` 进行依赖管理：

```bash
pnpm install

```

### 4. 运行本地服务器

启动开发服务器，支持热重载（文件修改后自动刷新）。由于未全局安装 Hexo，我们需要使用 `pnpm exec` 来调用本地依赖：

```bash
# 方式一：使用 npm script (推荐，简短方便)
pnpm run server

# 方式二：使用 pnpm exec 调用本地 hexo
pnpm exec hexo clean && pnpm exec hexo s

```

启动后，访问 `http://localhost:4000` 即可预览博客。

## 📝 常用命令

| 任务 | 命令 | 说明 |
| --- | --- | --- |
| **新建文章** | `pnpm exec hexo new "文章标题"` | 在 `source/_posts/` 下生成 Markdown 文件 |
| **清理缓存** | `pnpm exec hexo clean` | 清理 `public` 目录和数据库缓存 (解决样式异常等问题) |
| **生成静态页** | `pnpm exec hexo g` | 将 Markdown 渲染为 HTML 静态文件至 `public/` 目录 |
| **本地预览** | `pnpm exec hexo s` | 启动本地服务进行预览 |
| **部署发布** | `pnpm exec hexo d` | 将生成的静态文件推送到远程仓库 |

> **提示**: 您也可以在 `package.json` 中配置 script 脚本（如 `pnpm run build`），以进一步简化命令。

## 📂 目录结构说明

```text
.
├── .deploy_git/        # 自动生成的部署目录 (Git 仓库)
├── node_modules/       # 项目依赖文件
├── public/             # 最终生成的静态网页文件 (执行 generate 后生成)
├── scaffolds/          # 文章模板 (Hexo new 使用)
├── source/             # 原始内容目录
│   ├── _posts/         # 您的 Markdown 文章存放于此
│   └── ...             # 其他页面 (如 about, categories 等)
├── themes/             # 主题目录
│   └── butterfly/      # 当前使用的 Butterfly 主题源码
├── _config.yml         # 博客核心配置文件 (站点信息、插件配置等)
├── _config.butterfly.yml # Butterfly 主题配置文件 (样式、菜单等)
├── package.json        # 项目脚本与依赖列表
└── pnpm-lock.yaml      # 依赖版本锁定文件

```

## ⚙️ 部署配置

项目配置了 Git 自动化部署。在执行 `pnpm exec hexo deploy` 时，静态文件将被推送到以下仓库：

* **Type**: git
* **Repo**: `ssh://lbs@101.126.20.73:22/home/lbs/document/hexo.git`
* **Branch**: `master`

请确保您的本地 SSH Key 已配置并有权访问该服务器地址。

## 📄 许可证

本项目源码遵循开源协议（请查看项目根目录下的 [LICENSE](https://www.google.com/search?q=LICENSE) 文件）。
文章内容版权归作者 **李博帅** 所有。
