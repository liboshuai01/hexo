---
title: Centos部署Hexo个人博客实战指南
abbrlink: b3552560
date: 2025-08-01 01:35:19
tags:
  - 环境搭建
categories:
  - 环境搭建
toc: true
---


笔者一直以来都习惯于在[稀土掘金](https://juejin.cn/user/1878395516357111)平台进行技术博文的分享和记录。掘金作为一个独立运营的第三方平台，确实提供了优质的技术交流环境和丰富的资源，对技术人来说是一片很好的内容创作天地。然而，随着使用时间的增加，笔者开始意识到自己在数据安全方面的隐患以及平台依赖的问题。

首先，掘金目前并没有提供历史文章的导出功能，这意味着用户的所有创作内容都被深深绑定在平台之上，一旦平台无法正常运行或出现类似删库跑路这样的极端情况，用户可能就会面临失去所有文章的风险。此外，如果管理员误删数据、服务器故障导致数据丢失等问题出现，则笔者之前所有的心血可能会付之东流。这种“不受控制”的内容存储方式让我感到缺乏安全感和预见性。

因此，笔者回忆起自己之前曾研究过的个人博客搭建方式，并决定重新捡起基于 Hexo 的博客搭建工具，来构建一个完全属于自己的内容创作和存储空间。在拥有个人博客的基础上，不仅可以保持创作自由，同时也能确保数据的安全和持久性。接下来，本文将详细记录 Hexo 博客的搭建过程，希望对有相同需求的朋友有所帮助。

<!-- more -->

## 环境推荐

- 本地环境：Windows11
- 服务器环境：RockyLinux 8.10（Centos系列）

## 服务器环境搭建

> 为了方便我直接使用了root用户进行操作，如果想要使用其他用户进行搭建要主要权限问题哦！

### 安装git环境

```shell
sudo yum install -y git
```

### 安装nodejs与pnpm环境

打开[node.js-download](https://nodejs.org/en/download)这个链接，选择符合自己的系统环境复制命令到终端进行下载即可。

![](https://lbs-images.oss-cn-shanghai.aliyuncs.com/20250801013021964.png)

### 安装nginx环境

```shell
# 通过yum安装nginx
sudo yum install -y nginx
# 启动nginx开机自启
sudo systemctl enable nginx
# 启动nginx服务
sudo systemctl start nginx
# 查看nginx状态
sudo systemctl status nginx
```

### 创建博客目录与git钩子

> 创建git钩子是为了当本地环境执行`hexo d`时可以实时更新服务器的博文内容

```shell
# 创建博客目录
mkdir -p /root/document/blog
# 创建git钩子
cd /root/document && git init --bare hexo.git
sudo tee /root/document/hexo.git/hooks/post-receive <<'EOF'
git --work-tree=/root/document/blog --git-dir=/root/document/hexo.git checkout -f
EOF
# 基于钩子文件执行权限
sudo chmod +x /root/document/hexo.git/hooks/post-receive
```

### 配置nginx代理

1. 修改`/etc/nginx/nginx.conf`文件

   ```txt
   user root;
   worker_processes auto;
   error_log /var/log/nginx/error.log;
   pid /run/nginx.pid;
   
   # Load dynamic modules. See /usr/share/doc/nginx/README.dynamic.
   include /usr/share/nginx/modules/*.conf;
   
   events {
       worker_connections 1024;
   }
   
   http {
       log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                         '$status $body_bytes_sent "$http_referer" '
                         '"$http_user_agent" "$http_x_forwarded_for"';
   
       access_log  /var/log/nginx/access.log  main;
   
       sendfile            on;
       tcp_nopush          on;
       tcp_nodelay         on;
       keepalive_timeout   65;
       types_hash_max_size 2048;
   
       include             /etc/nginx/mime.types;
       default_type        application/octet-stream;
   
       include /etc/nginx/conf.d/*.conf;
   }
   ```

2. 新增`/etc/nginx/conf.d/blog.conf`配置文件

   > 如果仅有公网ip、没有域名的情况，配置文件内容如下

   ```txt
   server {
      listen        80;
      listen   [::]:80;
      # 配置为自己的公网IP
      server_name  106.14.19.58;
      
      location / {
        # 配置为服务器中个人博客的目录
        root   /root/document/blog;
        index  index.html index.htm;
      }
      
      error_page   500 502 503 504  /50x.html;
      location = /50x.html {
        root   /usr/share/nginx/html;
      }
   }
   ```

   > 如果有域名、且有SSL证书的情况，配置文件内容如下

   ```txt
   server {
       listen        80;
       listen   [::]:80;
       # 配置为自己的域名
       server_name  lbs.wiki;
   
       rewrite ^(.*)$ https://${server_name}$1 permanent;
   }
   
   server {
       listen  443 ssl;
       # 配置为自己的域名
       server_name  lbs.wiki;
   
       # 配置为自己ssl证书pem文件的路径
       ssl_certificate   /etc/nginx/ssl/lbs.wiki.pem;
       # 配置为自己ssl证书key文件的路径
       ssl_certificate_key   /etc/nginx/ssl/lbs.wiki.key;
   
       ssl_session_cache   shared:SSL:1m;
       ssl_session_timeout   5m;
   
       ssl_ciphers HIGH:!aNULL:!MD5;
       ssl_prefer_server_ciphers   on;
      
       location / {
           proxy_set_header   X-Real-IP        $remote_addr;
           proxy_set_header   Host             $http_host;
           proxy_set_header   X-Forwarded-For  $proxy_add_x_forwarded_for;
           # 配置为服务器中个人博客的目录
           root   /root/document/blog;
           index  index.html index.htm;
       }
   
       error_page   500 502 503 504  /50x.html;
       location = /50x.html {
           root   /usr/share/nginx/html;
       }
   }
   ```

3. 重新加载nginx配置

   ```shell
   # 检查nginx配置文件是否正确
   /usr/sbin/nginx -t
   # 热加载nginx配置文件
   /usr/sbin/nginx -s reload
   ```

## 本地环境搭建

### 安装git环境

打开[git-download](https://git-scm.com/downloads)链接，选择符合自己的系统点击进行下载即可，随后打开安装包一直进行下一步即可。

随后配置git全局变量

```shell
# 修改为自己的邮箱
git config --global user.email "liboshuai01@gmail.com"
# 修改为自己的英文名
git config --global user.name "BoShuai Li"
```

### 安装nodejs与pnpm环境

> 要有科学的魔法哦，不然会pnpm会安装失败的哦！

打开[node-v22.17.1-x64.msi](https://nodejs.org/dist/v22.17.1/node-v22.17.1-x64.msi)这个链接，会自动下载`node.js`的`v22.17.1`版本安装包。
打开安装程序，一路下一步即可，然后使用管理员权限打开`Git Bash`。执行以下命令：
```bash
# Verify the Node.js version:
node -v # Should print "v22.17.1".

# Download and install pnpm:
corepack enable pnpm

# Verify pnpm version:
pnpm -v
```

### 安装hexo环境

1. 新建一个目录用于存储个人博客数据，例如：`C:\Me\Project\other`。

2. 在新建的目录中打开`git bash`终端，执行下面的命令。

   > 一定要在`C:\Me\Project\other`目录下执行

    ```shell
    # 全局安装hexo-cli
    pnpm add -g hexo-cli
    # 创建并进入博客目录
    hexo init blog && cd blog
    # 安装项目依赖
    pnpm install
    # 启动Hexo本地服务
    hexo s
    ```

   > 下面是安装结束后的目录结构

   ![](https://lbs-images.oss-cn-shanghai.aliyuncs.com/20250408145801328.png)

   > 在浏览器打开`http://localhost:4000`查看页面

   ![](https://lbs-images.oss-cn-shanghai.aliyuncs.com/20250408144547941.png)

3. 安装butterfly主题

   > 一定要在`C:\Me\Project\other\blog`目录下执行

   ```shell
   # 安装主题依赖
   pnpm add hexo-renderer-pug hexo-renderer-stylus hexo-deployer-git hexo-abbrlink hexo-wordcount hexo-generator-searchdb
   # git clone主题
   git clone -b master https://github.com/jerryc127/hexo-theme-butterfly.git themes/butterfly
   
   # 修改`_config.yml`文件中的`theme`项为`butterfly`
   # 并复制`themes/butterfly/_config.yml`文件到博客根目录下，并重命名为`_config.butterfly.yml`
   
   # 启动Hexo本地服务
   hexo clean && hexo s --debug
   ```

   > 在浏览器打开`http://localhost:4000`查看页面

4. 配置远程部署地址，修改`_config.yml`文件中的`deploy`项

   > 注意：我的ssh端口为22222，而非22

   ```yaml
   deploy:
     type: 'git'
     repo: ssh://root@106.14.19.58:22222/root/document/hexo.git
     branch: master
   ```

5. 配置本地SSH免密登录服务器（使用git bash）

   ```shell
   # 生成密钥，随后一路回车
   ssh-keygen -t rsa
   # 这一步的目的是将本地生产的`id_rsa.pub`文件的内容拷贝到服务器上的`~/.ssh/authorized_keys`文件中
   # 当然也可以自己手动将生成的`id_rsa.pub`文件内容拷贝到服务器上的`~/.ssh/authorized_keys`文件中
   ssh -p 22222 root@106.14.19.58 "cat >> ~/.ssh/authorized_keys" < C:/Users/libos/.ssh/id_rsa.pub
   ```

   > 现在需要测试一下本地是否可以免密登录到服务器了

   ```shell
   ssh -p 22222 root@106.14.19.58
   ```

6. 发布博客内容到服务器

   ```shell
   hexo clean && hexo d -g
   ```

7. 浏览器打开`https://lbs.wiki`链接访问博客页面，进行查看

## 主题美化

## 创建标签页

**1. 前往你的 Hexo 的根目录**

**2. 输入 `hexo new page tags`**

**3. 你会找到 `source/tags/index.md` 这个文件**

**4. 修改这个文件：**

```markdown
---
title: 标签
date: 2022-01-01 00:00:00
type: 'tags'
orderby: random
order: 1
---

```

## 创建分类页

**1. 前往你的 Hexo 的根目录**

**2. 输入 `hexo new page categories`**

**3. 你会找到 `source/categories/index.md` 这个文件**

**4. 修改这个文件**

```markdown
---
title: 分类
date: 2022-01-01 00:00:00
type: 'categories'
---

```

## 创建关于页

**1. 前往你的 Hexo 的根目录**

**2. 输入 `hexo new page about`**

**3. 你会找到 `source/about/index.md` 这个文件**

**4. 修改这个文件**

```markdown
---
title: about
date: 2022-01-01 00:00:00
---

### 你好 👋

我的主要技术栈包括 Java、Flink、Spring Boot、Kubernetes (k8s) 等。我专注于实时流处理开发和 Web 应用开发。

<img width="50%" align="right" src="https://github-readme-stats.vercel.app/api?username=liboshuai01&show_icons=true&hide_border=true" />

#### 🌱 我正在进行的工作：
- 探索并实践基于 Flink 的实时流应用最佳实践。
- 深入研究 Flink 的内部架构设计和源代码实现。
- 持续学习并应用基于容器的 DevOps 实践以及结合 Kubernetes 的 CI/CD 自动化。

```

## 配置`_config.yml`

> `_config.yml`文件内容如下

```yaml
# Hexo Configuration
## Docs: https://hexo.io/docs/configuration.html
## Source: https://github.com/hexojs/hexo/

# Site
title: '技术博客'
subtitle: '李博帅'
description:
keywords:
author: '李博帅'
language: 'zh-CN'
timezone: 'Asia/Shanghai'

# URL
## Set your site url here. For example, if you use GitHub Page, set url as 'https://username.github.io/project'
url: https://lbs.wiki
#permalink: :year/:month/:day/:title/
permalink: '/pages/:abbrlink/'
permalink_defaults:
pretty_urls:
   trailing_index: true # Set to false to remove trailing 'index.html' from permalinks
   trailing_html: true # Set to false to remove trailing '.html' from permalinks
# abbrlink config
abbrlink:
   alg: crc32      # Algorithm used to calc abbrlink. Support crc16(default) and crc32
   rep: hex        # Representation of abbrlink in URLs. Support dec(default) and hex
   drafts: false   # Whether to generate abbrlink for drafts. (false in default)
   force: false    # Enable force mode. In this mode, the plugin will ignore the cache, and calc the abbrlink for every post even it already had an abbrlink. (false in default)
   writeback: true # Whether to write changes to front-matters back to the actual markdown files. (true in default)

# Directory
source_dir: source
public_dir: public
tag_dir: tags
archive_dir: archives
category_dir: categories
code_dir: downloads/code
i18n_dir: :lang
skip_render:

# Writing
new_post_name: :title.md # File name of new posts
default_layout: post
titlecase: false # Transform title into titlecase
external_link:
   enable: true # Open external links in new tab
   field: site # Apply to the whole site
   exclude: ''
filename_case: 0
render_drafts: false
post_asset_folder: false
relative_link: false
future: true
syntax_highlighter: highlight.js
highlight:
   line_number: true
   auto_detect: false
   tab_replace: ''
   wrap: true
   hljs: false
prismjs:
   preprocess: true
   line_number: true
   tab_replace: ''

# Home page setting
# path: Root path for your blogs index page. (default = '')
# per_page: Posts displayed per page. (0 = disable pagination)
# order_by: Posts order. (Order by date descending by default)
index_generator:
   path: ''
   per_page: 10
   order_by: -date

# Category & Tag
default_category: uncategorized
category_map:
tag_map:

# Metadata elements
## https://developer.mozilla.org/en-US/docs/Web/HTML/Element/meta
meta_generator: true

# Date / Time format
## Hexo uses Moment.js to parse and display date
## You can customize the date format as defined in
## http://momentjs.com/docs/#/displaying/format/
date_format: YYYY-MM-DD
time_format: HH:mm:ss
## updated_option supports 'mtime', 'date', 'empty'
updated_option: 'mtime'

# Pagination
## Set per_page to 0 to disable pagination
per_page: 10
pagination_dir: page

# Include / Exclude file(s)
## include:/exclude: options only apply to the 'source/' folder
include:
exclude:
ignore:

## Themes: https://hexo.io/themes/
theme: butterfly
# Extensions
## Plugins: https://hexo.io/plugins/

# Deployment
## Docs: https://hexo.io/docs/one-command-deployment
deploy:
   type: 'git'
   repo: ssh://root@lbs.wiki:22222/root/document/hexo.git
   branch: master
```

## 配置`_config.butterfly.yml`

> 参考: [Butterfly 文档(三) 主题配置](https://butterfly.js.org/posts/4aa8abbe/)

> `_config.butterfly.yml`配置文件内容如下

```yaml
# --------------------------------------
# Hexo Butterfly Theme Configuration
# If you have any questions, please refer to the documentation
# Chinese: https://butterfly.js.org/
# English: https://butterfly.js.org/en/
# --------------------------------------

# --------------------------------------
# Navigation Settings
# --------------------------------------

nav:
  # Navigation bar logo image
  logo: https://lbs-images.oss-cn-shanghai.aliyuncs.com/202503181853491.svg
  display_title: true
  # Whether to fix navigation bar
  fixed: false

menu:
  首页: / || fas fa-home
  归档: /archives/ || fas fa-archive
  标签: /tags/ || fas fa-tags
  分类: /categories/ || fas fa-folder-open
  关于: /about/ || fas fa-heart

# --------------------------------------
# Code Blocks Settings
# --------------------------------------

code_blocks:
  # Code block theme: darker / pale night / light / ocean / false
  theme: pale night
  macStyle: true
  # Code block height limit (unit: px)
  height_limit: 500
  word_wrap: false

  # Toolbar
  copy: true
  language: true
  # true: shrink the code blocks | false: expand the code blocks | none: expand code blocks and hide the button
  shrink: false
  fullpage: true

# Social media links
# Formal:
#   icon: link || the description || color
social:
  # fab fa-github: https://github.com/xxxxx || Github || '#24292e'
  # fas fa-envelope: mailto:xxxxxx@gmail.com || Email || '#4a7dbe'
  fab fa-github: https://github.com/liboshuai01 || Github || "#hdhfbb"
  fab fa-git-alt: https://gitee.com/liboshuai01 || Gitee || "#C71D23"
  fas fa-envelope: mailto:liboshuai01@gmail.com || Email || "#000000"

# --------------------------------------
# Image Settings
# --------------------------------------

favicon: https://lbs-images.oss-cn-shanghai.aliyuncs.com/202503181853491.svg

avatar:
  img: https://lbs-images.oss-cn-shanghai.aliyuncs.com/20250607154130416.png
  effect: false

# Disable all banner images
disable_top_img: true

# If the banner of page not setting, it will show the default_top_img
default_top_img:

# The banner image of index page
index_img:

# The banner image of archive page
archive_img:

# Note: tag page, not tags page
tag_img:

# The banner image of tag page, you can set the banner image for each tag
# Format:
#  - tag name: xxxxx
tag_per_img:

# Note: category page, not categories page
category_img:

# The banner image of category page, you can set the banner image for each category
# Format:
#  - category name: xxxxx
category_per_img:

# The background image of footer
footer_img: false

# Website Background
# Can set it to color or image url
background:

cover:
  # Disable the cover or not
  index_enable: true
  aside_enable: true
  archives_enable: true
  # When cover is not set, the default cover is displayed
  default_cover:
    # - xxx.jpg

# Replace Broken Images
error_img:
  flink: /img/friend_404.gif
  post_page: /img/404.jpg

# A simple 404 page
error_404:
  enable: true
  subtitle: '页面没有找到'
  background: https://lbs-images.oss-cn-shanghai.aliyuncs.com/20250705092755902.png

post_meta:
  # Home Page
  page:
    # Choose: created / updated / both
    date_type: created
    # Choose: date / relative
    date_format: date
    categories: true
    tags: true
    label: true
  post:
    # Choose: left / center
    position: left
    # Choose: created / updated / both
    date_type: both
    # Choose: date / relative
    date_format: date
    categories: true
    tags: true
    label: true

# --------------------------------------
# Index page settings
# --------------------------------------

# The top_img settings of home page
# default: top img - full screen, site info - middle
# The position of site info, eg: 300px/300em/300rem/10%
index_site_info_top:
# The height of top_img, eg: 300px/300em/300rem
index_top_img_height:

# The subtitle on homepage
subtitle:
  enable: false
  # Typewriter Effect
  effect: true
  # Customize typed.js
  # https://github.com/mattboldt/typed.js/#customization
  typed_option:
  # Source - Call the third-party service API (Chinese only)
  # It will show the source first, then show the content of sub
  # Choose: false/1/2/3
  # false - disable the function
  # 1 - hitokoto.cn
  # 2 - https://api.aa1.cn/doc/yiyan.html
  # 3 - jinrishici.com
  source: false
  # If you close the typewriter effect, the subtitle will only show the first line of sub
  sub:

# Article layout on the homepage
# 1: Cover on the left, info on the right
# 2: Cover on the right, info on the left
# 3: Cover and info alternate between left and right
# 4: Cover on top, info on the bottom
# 5: Info displayed on the cover
# 6: Masonry layout - Cover on top, info on the bottom
# 7: Masonry layout - Info displayed on the cover
index_layout: 2

# Display the article introduction on homepage
# 1: description
# 2: both (if the description exists, it will show description, or show the auto_excerpt)
# 3: auto_excerpt (default)
# false: do not show the article introduction
index_post_content:
  method: 2
  # If you set method to 2 or 3, the length need to config
  length: 500

# --------------------------------------
# Post Settings
# --------------------------------------

toc:
  post: true
  page: false
  number: true
  expand: true
  # Only for post
  style_simple: false
  scroll_percent: true

post_copyright:
  enable: true
  decode: false
  author_href:
  license: CC BY-NC-SA 4.0
  license_url: https://creativecommons.org/licenses/by-nc-sa/4.0/

# Sponsor/reward
reward:
  enable: true
  text:
  QR_code:
    - img: https://lbs-images.oss-cn-shanghai.aliyuncs.com/20250704175000920.jpg
      link:
      text: 微信
    - img: https://lbs-images.oss-cn-shanghai.aliyuncs.com/20250704175729483.jpg
      link:
      text: 支付宝

# Post edit
# Easily browse and edit blog source code online.
post_edit:
  enable: false
  # url: https://github.com/user-name/repo-name/edit/branch-name/subdirectory-name/
  # For example: https://github.com/jerryc127/butterfly.js.org/edit/main/source/
  url:

# Related Articles
related_post:
  enable: true
  # Number of posts displayed
  limit: 6
  # Choose: created / updated
  date_type: created

# Choose: 1 / 2 / false
# 1: The 'next post' will link to old post
# 2: The 'next post' will link to new post
# false: disable pagination
post_pagination: 1

# Displays outdated notice for a post
noticeOutdate:
  enable: false
  # Style: simple / flat
  style: flat
  # When will it be shown
  limit_day: 365
  # Position: top / bottom
  position: top
  message_prev: It has been
  message_next: days since the last update, the content of the article may be outdated.

# --------------------------------------
# Footer Settings
# --------------------------------------
footer:
  owner:
    enable: true
    since: 2022
  custom_text:
  # Copyright of theme and framework
  copyright: true

# --------------------------------------
# Aside Settings
# --------------------------------------

aside:
  enable: true
  hide: false
  # Show the button to hide the aside in bottom right button
  button: true
  mobile: true
  # Position: left / right
  position: left
  display:
    archive: true
    tag: true
    category: true
  card_author:
    enable: true
    description:
    button:
      enable: true
      icon: fab fa-github
      text: Follow Me
      link: https://github.com/liboshuai01
  card_announcement:
    enable: true
    content: 记录自己的学习成长
  card_recent_post:
    enable: true
    # If set 0 will show all
    limit: 5
    # Sort: date / updated
    sort: date
    sort_order:
  card_newest_comments:
    enable: false
    sort_order:
    limit: 6
    # Unit: mins, save data to localStorage
    storage: 10
    avatar: true
  card_categories:
    enable: true
    # If set 0 will show all
    limit: 8
    # Choose: none / true / false
    expand: none
    sort_order:
  card_tags:
    enable: true
    # If set 0 will show all
    limit: 40
    color: false
    # Order of tags, random/name/length
    orderby: random
    # Sort of order. 1, asc for ascending; -1, desc for descending
    order: 1
    sort_order:
  card_archives:
    enable: true
    # Type: monthly / yearly
    type: monthly
    # Eg: YYYY年MM月
    format: MMMM YYYY
    # Sort of order. 1, asc for ascending; -1, desc for descending
    order: -1
    # If set 0 will show all
    limit: 8
    sort_order:
  card_post_series:
    enable: true
    # The title shows the series name
    series_title: false
    # Order by title or date
    orderBy: 'date'
    # Sort of order. 1, asc for ascending; -1, desc for descending
    order: -1
  card_webinfo:
    enable: true
    post_count: true
    last_push_date: true
    sort_order:
    # Time difference between publish date and now
    # Formal: Month/Day/Year Time or Year/Month/Day Time
    # Leave it empty if you don't enable this feature
    runtime_date: 2022/01/01 00:00:00

# --------------------------------------
# Bottom right button
# --------------------------------------

# The distance between the bottom right button and the bottom (default unit: px)
rightside_bottom:

# Conversion between Traditional and Simplified Chinese
translate:
  enable: false
  # The text of a button
  default: 繁
  # the language of website (1 - Traditional Chinese/ 2 - Simplified Chinese）
  defaultEncoding: 2
  # Time delay
  translateDelay: 0
  # The text of the button when the language is Simplified Chinese
  msgToTraditionalChinese: '繁'
  # The text of the button when the language is Traditional Chinese
  msgToSimplifiedChinese: '簡'

# Read Mode
readmode: true

# Dark Mode
darkmode:
  enable: true
  # Toggle Button to switch dark/light mode
  button: true
  # Switch dark/light mode automatically
  # autoChangeMode: 1  Following System Settings, if the system doesn't support dark mode, it will switch dark mode between 6 pm to 6 am
  # autoChangeMode: 2  Switch dark mode between 6 pm to 6 am
  # autoChangeMode: false
  autoChangeMode: false
  # Set the light mode time. The value is between 0 and 24. If not set, the default value is 6 and 18
  start:
  end:

# Show scroll percent in scroll-to-top button
rightside_scroll_percent: true

# Don't modify the following settings unless you know how they work
# Choose: readmode,translate,darkmode,hideAside,toc,chat,comment
# Don't repeat the same value
rightside_item_order:
  enable: false
  # Default: readmode,translate,darkmode,hideAside
  hide:
  # Default: toc,chat,comment
  show:

# --------------------------------------
# Global Settings
# --------------------------------------

anchor:
  # When you scroll, the URL will update according to header id.
  auto_update: false
  # Click the headline to scroll and update the anchor
  click_to_scroll: false

photofigcaption: false

copy:
  enable: true
  # Add the copyright information after copied content
  copyright:
    enable: false
    limit_count: 150

# Need to install the hexo-wordcount plugin
wordcount:
  enable: true
  # Display the word count of the article in post meta
  post_wordcount: true
  # Display the time to read the article in post meta
  min2read: true
  # Display the total word count of the website in aside's webinfo
  total_wordcount: true

# Busuanzi count for PV / UV in site
busuanzi:
  site_uv: false
  site_pv: false
  page_pv: true

# --------------------------------------
# Math
# --------------------------------------

# About the per_page
# if you set it to true, it will load mathjax/katex script in each page
# if you set it to false, it will load mathjax/katex script according to your setting (add the 'mathjax: true' or 'katex: true' in page's front-matter)
math:
  # Choose: mathjax, katex
  # Leave it empty if you don't need math
  use:
  per_page: true
  hide_scrollbar: false

  mathjax:
    # Enable the contextual menu
    enableMenu: true
    # Choose: all / ams / none, This controls whether equations are numbered and how
    tags: none

  katex:
    # Enable the copy KaTeX formula
    copy_tex: false

# --------------------------------------
# Search
# --------------------------------------

search:
  # Choose: algolia_search / local_search / docsearch
  # leave it empty if you don't need search
  use: local_search
  placeholder:

  # Algolia Search
  algolia_search:
    # Number of search results per page
    hitsPerPage: 6

  # Local Search
  local_search:
    # Preload the search data when the page loads.
    preload: true
    # Show top n results per article, show all results by setting to -1
    top_n_per_article: -1
    # Unescape html strings to the readable one.
    unescape: true
    CDN:

  # Docsearch
  # https://docsearch.algolia.com/
  docsearch:
    appId:
    apiKey:
    indexName:
    option:

# --------------------------------------
# Share System
# --------------------------------------

share:
  # Choose: sharejs / addtoany
  # Leave it empty if you don't need share
  use:

  # Share.js
  # https://github.com/overtrue/share.js
  sharejs:
    sites: facebook,twitter,wechat,weibo,qq

  # AddToAny
  # https://www.addtoany.com/
  addtoany:
    item: facebook,twitter,wechat,sina_weibo,facebook_messenger,email,copy_link

# --------------------------------------
# Comments System
# --------------------------------------

comments:
  # Up to two comments system, the first will be shown as default
  # Leave it empty if you don't need comments
  # Choose: Disqus/Disqusjs/Livere/Gitalk/Valine/Waline/Utterances/Facebook Comments/Twikoo/Giscus/Remark42/Artalk
  # Format of two comments system : Disqus,Waline
  use:
  # Display the comment name next to the button
  text: true
  # Lazyload: The comment system will be load when comment element enters the browser's viewport.
  # If you set it to true, the comment count will be invalid
  lazyload: false
  # Display comment count in post's top_img
  count: false
  # Display comment count in Home Page
  card_post_count: false

# Disqus
# https://disqus.com/
disqus:
  shortname:
  # For newest comments widget
  apikey:

# Alternative Disqus - Render comments with Disqus API
# https://github.com/SukkaW/DisqusJS
disqusjs:
  shortname:
  apikey:
  option:

# Livere
# https://www.livere.com/
livere:
  uid:

# Gitalk
# https://github.com/gitalk/gitalk
gitalk:
  client_id:
  client_secret:
  repo:
  owner:
  admin:
  option:

# Valine
# https://valine.js.org
valine:
  appId:
  appKey:
  avatar: monsterid
  # This configuration is suitable for domestic custom domain name users, overseas version will be automatically detected (no need to manually fill in)
  serverURLs:
  bg:
  # Use Valine visitor count as the page view count
  visitor: false
  option:

# Waline - A simple comment system with backend support fork from Valine
# https://waline.js.org/
waline:
  serverURL:
  bg:
  # Use Waline pageview count as the page view count
  pageview: false
  option:

# Utterances
# https://utteranc.es/
utterances:
  repo:
  # Issue Mapping: pathname/url/title/og:title
  issue_term: pathname
  # Theme: github-light/github-dark/github-dark-orange/icy-dark/dark-blue/photon-dark
  light_theme: github-light
  dark_theme: photon-dark
  js:
  option:

# Facebook Comments Plugin
# https://developers.facebook.com/docs/plugins/comments/
facebook_comments:
  app_id:
  # optional
  user_id:
  pageSize: 10
  # Choose: social / time / reverse_time
  order_by: social
  lang: en_US

# Twikoo
# https://github.com/imaegoo/twikoo
twikoo:
  envId:
  region:
  # Use Twikoo visitor count as the page view count
  visitor: false
  option:

# Giscus
# https://giscus.app/
giscus:
  repo:
  repo_id:
  category_id:
  light_theme: light
  dark_theme: dark
  js:
  option:

# Remark42
# https://remark42.com/docs/configuration/frontend/
remark42:
  host:
  siteId:
  option:

# Artalk
# https://artalk.js.org/guide/frontend/config.html
artalk:
  server:
  site:
  # Use Artalk visitor count as the page view count
  visitor: false
  option:

# --------------------------------------
# Chat Services
# --------------------------------------

chat:
  # Choose: chatra/tidio/crisp
  # Leave it empty if you don't need chat
  use:
  # Chat Button [recommend]
  # It will create a button in the bottom right corner of website, and hide the origin button
  rightside_button: false
  # The origin chat button is displayed when scrolling up, and the button is hidden when scrolling down
  button_hide_show: false

# https://chatra.io/
chatra:
  id:

# https://www.tidio.com/
tidio:
  public_key:

# https://crisp.chat/en/
crisp:
  website_id:

# --------------------------------------
# Analysis
# --------------------------------------

# https://tongji.baidu.com/web/welcome/login
baidu_analytics:

# https://analytics.google.com/analytics/web/
google_analytics:

# https://www.cloudflare.com/zh-tw/web-analytics/
cloudflare_analytics:

# https://clarity.microsoft.com/
microsoft_clarity:

# https://umami.is/
umami_analytics:
  enable: false
  # For self-hosted setups, configure the hostname of the Umami instance
  serverURL:
  website_id:
  option:
  UV_PV:
    site_uv: false
    site_pv: false
    page_pv: false
    # Umami Cloud (API key) / self-hosted Umami (token)
    token:

# --------------------------------------
# Advertisement
# --------------------------------------

# Google Adsense
google_adsense:
  enable: false
  auto_ads: true
  js: https://pagead2.googlesyndication.com/pagead/js/adsbygoogle.js
  client:
  enable_page_level_ads: true

# Insert ads manually
# Leave it empty if you don't need ads
ad:
  # Insert ads in the index (every three posts)
  index:
  # Insert ads in aside
  aside:
  # Insert ads in the post (before pagination)
  post:

# --------------------------------------
# Verification
# --------------------------------------

site_verification:
  # - name: google-site-verification
  #   content: xxxxxx
  # - name: baidu-site-verification
  #   content: xxxxxxx

# --------------------------------------
# Beautify / Effect
# --------------------------------------

# Theme color for customize
# Notice: color value must in double quotes like "#000" or may cause error!

# theme_color:
#   enable: true
#   main: "#49B1F5"
#   paginator: "#00c4b6"
#   button_hover: "#FF7242"
#   text_selection: "#00c4b6"
#   link_color: "#99a9bf"
#   meta_color: "#858585"
#   hr_color: "#A4D8FA"
#   code_foreground: "#F47466"
#   code_background: "rgba(27, 31, 35, .05)"
#   toc_color: "#00c4b6"
#   blockquote_padding_color: "#49b1f5"
#   blockquote_background_color: "#49b1f5"
#   scrollbar_color: "#49b1f5"
#   meta_theme_color_light: "ffffff"
#   meta_theme_color_dark: "#0d0d0d"

theme_color:
   enable: true
   # 主要强调色：一个专业且不过于鲜艳的蓝色，用于按钮、链接活跃状态等
   main: "#367CFF" # 专业蓝，比原先的更沉稳
   # 分页器颜色：与主色调保持一致，增强统一性
   paginator: "#367CFF"
   # 按钮悬停颜色：使用深灰色，显得内敛而有质感
   button_hover: "#4A4A4A"
   # 文本选中颜色：使用主色调，直观且与整体风格协调
   text_selection: "#367CFF"
   # 链接颜色：采用一个略带蓝色的中性灰，突出但不跳跃
   link_color: "#5A6B80" # 接近蓝色的深灰，有别于正文但又很稳重
   # 元信息颜色（例如日期、作者等）：使用中等灰色，易读且不抢眼
   meta_color: "#6B6B6B" # 比原先的略深，对比度更好
   # 分隔线颜色：非常浅的灰色，提供分隔感而不突兀
   hr_color: "#E0E0E0"
   # 代码块前景颜色：保持原有的橙红色，因为这是代码高亮的常见做法，能有效区分
   code_foreground: "#F47466"
   # 代码块背景颜色：保持不变，半透明的深色是很好的选择
   code_background: "rgba(27, 31, 35, .05)"
   # TOC（目录）颜色：与主色调保持一致，用于可点击的目录项
   toc_color: "#367CFF"
   # 引用块（blockquote）左边条颜色：使用非常浅的灰色或一个微弱的强调色，避免过于醒目
   blockquote_padding_color: "#D3DCE6" # 柔和的浅蓝色灰色，替代了醒目的主色
   # 引用块背景颜色：使用非常浅的灰色，让引用块更内敛
   blockquote_background_color: "#F8F8F8" # 接近白色，提供轻微的区分
   # 滚动条颜色：与主色调保持一致，提供统一的用户体验
   scrollbar_color: "#367CFF"
   # 针对浅色主题的 meta 颜色：白色背景
   meta_theme_color_light: "ffffff"
   # 针对深色主题的 meta 颜色：极深灰色背景
   meta_theme_color_dark: "#0d0d0d"

# The user interface setting of category and tag page
# Choose: index - same as Homepage UI / default - same as archives UI
# leave it empty or index
category_ui:
tag_ui:

# Rounded corners for UI elements
rounded_corners_ui: true

# Stretches the lines so that each line has equal width
text_align_justify: false

# Add a mask to the header and footer
mask:
  header: true
  footer: true

# Loading Animation
preloader:
  enable: false
  # source
  # 1. fullpage-loading
  # 2. pace (progress bar)
  source: 1
  # pace theme (see https://codebyzach.github.io/pace/)
  pace_css_url:

# Page Transition
enter_transitions: true

# Default display mode - light (default) / dark
display_mode: light

# Configuration for beautifying the content of the article
beautify:
  enable: true
  # Specify the field to beautify (site or post)
  field: site
  # Specify the icon to be used as a prefix for the title, such as '\f0c1'
  title_prefix_icon: '\f0c1'
  # Specify the color of the title prefix icon, such as '#F47466'
  title_prefix_icon_color: '#F47466'

# Global font settings
# Don't modify the following settings unless you know how they work
font:
  global_font_size:
  code_font_size:
  font_family:
  code_font_family:

# Font settings for the site title and site subtitle
blog_title_font:
  font_link:
  font_family:

# The setting of divider icon
hr_icon:
  enable: true
  # The unicode value of Font Awesome icon, such as '\3423'
  icon:
  icon_top:

# Typewriter Effect
# https://github.com/disjukr/activate-power-mode
activate_power_mode:
  enable: false
  colorful: true
  shake: true
  mobile: false

# Background effects
# --------------------------------------

# canvas_ribbon
# See: https://github.com/hustcc/ribbon.js
canvas_ribbon:
  enable: false
  # The size of ribbon
  size: 150
  # The opacity of ribbon (0 ~ 1)
  alpha: 0.6
  zIndex: -1
  click_to_change: false
  mobile: false

# Fluttering Ribbon
canvas_fluttering_ribbon:
  enable: false
  mobile: false

# canvas_nest
# https://github.com/hustcc/canvas-nest.js
canvas_nest:
  enable: false
  # Color of lines, default: '0,0,0'; RGB values: (R,G,B).(note: use ',' to separate.)
  color: '0,0,255'
  # The opacity of line (0~1)
  opacity: 0.7
  # The z-index property of the background
  zIndex: -1
  # The number of lines
  count: 99
  mobile: false

# Mouse click effects: fireworks
fireworks:
  enable: false
  zIndex: 9999
  mobile: false

# Mouse click effects: Heart symbol
click_heart:
  enable: false
  mobile: false

# Mouse click effects: words
clickShowText:
  enable: false
  text:
    # - I
    # - LOVE
    # - YOU
  fontSize: 15px
  random: false
  mobile: false

# --------------------------------------
# Lightbox Settings
# --------------------------------------

# Choose: fancybox / medium_zoom
# https://github.com/francoischalifour/medium-zoom
# https://fancyapps.com/fancybox/
# Leave it empty if you don't need lightbox
lightbox: fancybox

# --------------------------------------
# Tag Plugins settings
# --------------------------------------

# Series
series:
  enable: false
  # Order by title or date
  orderBy: 'title'
  # Sort of order. 1, asc for ascending; -1, desc for descending
  order: 1
  number: true

# ABCJS - The ABC Music Notation Plugin
# https://github.com/paulrosen/abcjs
abcjs:
  enable: false
  per_page: true

# Mermaid
# https://github.com/mermaid-js/mermaid
mermaid:
  enable: false
  # Write Mermaid diagrams using code blocks
  code_write: false
  # built-in themes: default / forest / dark / neutral
  theme:
    light: default
    dark: dark

# chartjs
# see https://www.chartjs.org/docs/latest/
chartjs:
  enable: false
  # Do not modify unless you understand how they work.
  # The default settings are only used when the MD syntax is not specified.
  # General font color for the chart
  fontColor:
    light: 'rgba(0, 0, 0, 0.8)'
    dark: 'rgba(255, 255, 255, 0.8)'
  # General border color for the chart
  borderColor:
    light: 'rgba(0, 0, 0, 0.1)'
    dark: 'rgba(255, 255, 255, 0.2)'
  # Background color for scale labels on radar and polar area charts
  scale_ticks_backdropColor:
    light: 'transparent'
    dark: 'transparent'

# Note - Bootstrap Callout
note:
  # Note tag style values:
  #  - simple    bs-callout old alert style. Default.
  #  - modern    bs-callout new (v2-v3) alert style.
  #  - flat      flat callout style with background, like on Mozilla or StackOverflow.
  #  - disabled  disable all CSS styles import of note tag.
  style: flat
  icons: true
  border_radius: 3
  # Offset lighter of background in % for modern and flat styles (modern: -12 | 12; flat: -18 | 6).
  # Offset also applied to label tag variables. This option can work with disabled note tag.
  light_bg_offset: 0

# --------------------------------------
# Other Settings
# --------------------------------------

# https://github.com/MoOx/pjax
pjax:
  enable: false
  # Exclude the specified pages from pjax, such as '/music/'
  exclude:
    # - /xxxxxx/

# Inject the css and script (aplayer/meting)
aplayerInject:
  enable: false
  per_page: true

# Snackbar - Toast Notification
# https://github.com/polonel/SnackBar
# position: top-left / top-center / top-right / bottom-left / bottom-center / bottom-right
snackbar:
  enable: false
  position: bottom-left
  # The background color of Toast Notification in light mode and dark mode
  bg_light: '#49b1f5'
  bg_dark: '#1f1f1f'

# Instant.page
# https://instant.page/
instantpage: true

# Lazyload
# https://github.com/verlok/vanilla-lazyload
lazyload:
  enable: false
  # Specify the field to use lazyload (site or post)
  field: site
  placeholder:
  blur: false

# PWA
# See https://github.com/JLHwung/hexo-offline
# ---------------
pwa:
  enable: false
  manifest:
  apple_touch_icon:
  favicon_32_32:
  favicon_16_16:
  mask_icon:

# Open graph meta tags
# https://hexo.io/docs/helpers#open-graph
Open_Graph_meta:
  enable: true
  option:
    # twitter_card:
    # twitter_image:
    # twitter_id:
    # twitter_site:
    # google_plus:
    # fb_admins:
    # fb_app_id:

# Structured Data
# https://developers.google.com/search/docs/guides/intro-structured-data
structured_data: true

# Add the vendor prefixes to ensure compatibility
css_prefix: true

# Inject
# Insert the code to head (before '</head>' tag) and the bottom (before '</body>' tag)
inject:
  head:
    # - <link rel="stylesheet" href="/xxx.css">
  bottom:
    # - <script src="xxxx"></script>

# CDN Settings
# Don't modify the following settings unless you know how they work
CDN:
  # The CDN provider for internal and third-party scripts
  # Options for both: local/jsdelivr/unpkg/cdnjs/custom
  # Note: Dev version can only use 'local' for internal scripts
  # Note: When setting third-party scripts to 'local', you need to install hexo-butterfly-extjs
  internal_provider: local
  third_party_provider: jsdelivr

  # Add version number to url, true or false
  version: false

  # Custom format
  # For example: https://cdn.staticfile.org/${cdnjs_name}/${version}/${min_cdnjs_file}
  custom_format:

  option:
    # abcjs_basic_js:
    # activate_power_mode:
    # algolia_js:
    # algolia_search:
    # aplayer_css:
    # aplayer_js:
    # artalk_css:
    # artalk_js:
    # blueimp_md5:
    # busuanzi:
    # canvas_fluttering_ribbon:
    # canvas_nest:
    # canvas_ribbon:
    # chartjs:
    # click_heart:
    # clickShowText:
    # disqusjs:
    # disqusjs_css:
    # docsearch_css:
    # docsearch_js:
    # egjs_infinitegrid:
    # fancybox:
    # fancybox_css:
    # fireworks:
    # fontawesome:
    # gitalk:
    # gitalk_css:
    # giscus:
    # instantpage:
    # instantsearch:
    # katex:
    # katex_copytex:
    # lazyload:
    # local_search:
    # main:
    # main_css:
    # mathjax:
    # medium_zoom:
    # mermaid:
    # meting_js:
    # prismjs_autoloader:
    # prismjs_js:
    # prismjs_lineNumber_js:
    # pjax:
    # sharejs:
    # sharejs_css:
    # snackbar:
    # snackbar_css:
    # translate:
    # twikoo:
    # typed:
    # utils:
    # valine:
    # waline_css:
    # waline_js:
```

## hexo常用命令

```shell
# 初始化站点，生成一个简单网站所需的各种文件。
hexo init

# 清除缓存 网页正常情况下可以忽略此条命令（简写 hexo c）
hexo clean

# 新建文章
hexo new "postName" 

# 新建页面
hexo new page "pageName" 

# 生成静态页面至public目录 简写：hexo g
hexo generate 

# 开启预览访问端口（默认端口4000，'ctrl + c'关闭server） 简写：hexo s，可用--debug
hexo server 

# 将.deploy目录部署到GitHub 简写：hexo d
hexo deploy 

# 组合命令，先生成页面，后部署到服务器
hexo d -g 
```

## 结语

这样就可以在本地编写博文，然后快速部署到服务器中。

> 参考: https://www.cnblogs.com/cheyaoyao/p/17836522.html