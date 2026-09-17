# XHiaoe 的博客 · 使用手册

> 站点地址：<https://xhiaoe.github.io>
> 源码仓库：<https://github.com/XHiaoe/XHiaoe.github.io>
> 定位：网安新手的学习记录（AI / CTF）

**技术栈**：Hexo 8.1.2 + Butterfly 5.7.0 + GitHub Actions 自动部署

---

## 目录

- [1. 日常速查](#1-日常速查)
- [2. 发布一篇文章](#2-发布一篇文章)
- [3. 删除一篇文章](#3-删除一篇文章)
- [4. 置顶文章](#4-置顶文章)
- [5. 文件与目录作用](#5-文件与目录作用)
- [6. 三个配置文件怎么分工](#6-三个配置文件怎么分工)
- [7. 本地调试](#7-本地调试)
- [8. 上线流程与自查清单](#8-上线流程与自查清单)
- [9. 图片资源清单](#9-图片资源清单)
- [10. 疑难速查](#10-疑难速查)
- [11. 还没做的事](#11-还没做的事)

> **所有命令都在 Git Bash 里执行，工作目录是项目根目录。**
>
> ```bash
> cd /e/blog/Git/my-blog
> ```
>
> 提示符结尾显示 `(main)` 就说明位置正确。

---

## 1. 日常速查

| 想做什么 | 命令 |
|---|---|
| 本地预览（改完立刻看效果） | `npx hexo server` → 浏览器打开 http://localhost:4000 |
| 新建一篇文章 | `npx hexo new "文章标题"` |
| 彻底重新生成 | `npx hexo clean && npx hexo generate` |
| **发布上线** | `git add -A && git commit -m "说明" && git push` |
| 看构建进度 | <https://github.com/XHiaoe/XHiaoe.github.io/actions> |

**关于 `npx hexo` 这个前缀**：`hexo` 命令装在项目的 `node_modules` 里，`npx` 会自动找到本地这份，最稳。以下三种写法等价，任选：

```bash
npx hexo generate                    # 推荐
./node_modules/.bin/hexo generate    # 不用 npx 的写法
npm run build                        # package.json 里预置的快捷脚本
```

> ⚠️ **不要用 `npm run deploy`**。它等价于 `hexo deploy`，而本项目走的是 GitHub Actions 自动构建，用 `hexo deploy` 会把源码 force push 覆盖掉。这个项目不需要 `hexo d`。

---

## 2. 发布一篇文章

### 第 1 步：新建文章

```bash
npx hexo new "XSS 入门笔记"
```

会在 `source/_posts/` 下生成 `XSS 入门笔记.md`，文件名取自标题。

> **注意**：新建的文章**立刻就是"已发布"状态**，不是草稿。如果只想先记点东西、暂时不公开，用草稿模式：
>
> ```bash
> npx hexo new draft "临时想法"      # 存到 source/_drafts/，不发布
> npx hexo publish "临时想法"        # 想发布了再执行这句
> ```
>
> 草稿在本地预览里默认也看不到，要看得加参数：`npx hexo server --drafts`

### 第 2 步：填 front-matter

打开新生成的文件，开头 `---` 包裹的部分就是 front-matter。新建出来的文件长这样：

```yaml
---
title: XSS 入门笔记
date: 2026-09-17 10:00:00
tags:
categories:
description:
cover:
sticky:
---
```

这些字段来自模板文件 `scaffolds/post.md`，想增删字段就编辑它（改完对以后新建的文章生效）。用不到的字段**留空即可**，不用删。

字段详解：

| 字段 | 作用 | 说明 |
|---|---|---|
| `title` | 文章标题 | 决定文件名和 URL |
| `date` | 发布日期 | 决定排序，也是 URL 的一部分（`/2026/09/17/标题/`） |
| `tags` | 标签 | 多个写成 `tags: [CTF, Web]`，或用 `-` 分行列出 |
| `categories` | 分类 | 写法同上；不填会自动归到 `uncategorized` |
| `description` | 摘要 | 用于首页卡片和搜索引擎描述 |
| `cover` | 封面图 | 图片放在 `source/img/`，这里写 `/img/cover.jpg` |
| `sticky` | **置顶** | 见[第 4 章](#4-置顶文章) |
| `published` | 是否发布 | 写 `false` 就隐藏这篇文章，但文件保留 |
| `updated` | 更新时间 | **不写**则自动用文件的修改时间 |

> ⚠️ 还有一个 `top` 字段，网上教程常见。它**只给首页卡片加一个图钉图标，不会真的排到前面**，要置顶请用 `sticky`。

### 第 3 步：写正文

正文就是 Markdown。两个实用技巧：

**① 控制首页摘要的截断位置**——在你想截断的地方单独占一行，写一个 HTML 注释形式的 more 标记：

```markdown
前面这段会显示在首页卡片上。

<!-- more -->

后面这段只在点进文章后才看得到。
```

**② 插入图片**——把图片放进 `source/img/`，然后：

```markdown
![图片说明](/img/图片名.png)
```

### 第 4 步：本地看一眼

```bash
npx hexo server
```

浏览器打开 http://localhost:4000，确认排版没问题后 `Ctrl + C` 停掉服务。

### 第 5 步：发布

```bash
git add -A
git status                        # 核对一下改了哪些文件
git commit -m "新增文章：XSS 入门笔记"
git push
```

推上去之后 GitHub Actions 会自动构建并发布，大约一分钟后刷新 https://xhiaoe.github.io 就能看到。

> **关于中文标题的 URL**：现在 `permalink` 是 `:year/:month/:day/:title/`，中文标题会让地址变成一长串百分号编码（如 `/2026/09/17/XSS%20%E5%85%A5%E9%97%A8%E7%AC%94%E8%AE%B0/`）。能用，但不好看、不好分享。介意的话以后可以装 `hexo-abbrlink` 插件把每篇文章的地址变成短哈希，这是可选项，不影响使用。

---

## 3. 删除一篇文章

### 情况一：彻底删掉

```bash
rm source/_posts/文章名.md
npx hexo clean && npx hexo generate     # 本地验证，确认构建不报错
git add -A
git commit -m "删除文章：文章名"
git push
```

**为什么用 `git add -A` 而不是 `git add .`**：这样能确保"文件被删除"这个动作也被记录下来。可以顺手 `git status` 看有没有 `deleted:` 行。

**删除的副作用**（想清楚再删）：

- 这篇文章的网址变成 **404**，别人收藏的链接、搜索引擎里的旧链接都会失效，GitHub Pages 不会做重定向
- 如果有其他文章用 Markdown 链接引用了它，那些链接也会失效
- 首页卡片、归档页、标签页、搜索索引都会在下次构建时自动更新，这部分不用手动处理

### 情况二：只是暂时不想让人看到（推荐）

保留文件，在 front-matter 里加一行：

```yaml
published: false
```

文章就不会出现在任何列表里，但文件还在、Git 历史也还在。想恢复就把它改回 `true` 或删掉这一行。

### 情况三：挪进草稿箱

```bash
mv source/_posts/文章名.md source/_drafts/
git add -A && git commit -m "移入草稿：文章名" && git push
```

`source/_drafts/` 目录现在还不存在，第一次用 `npx hexo new draft "标题"` 时会自动创建。

> **三种方式怎么选**：写错了、不想要了 → 情况一；还在改、先下线 → 情况二；想以后接着写 → 情况三。

---

## 4. 置顶文章

### 怎么用

在文章的 front-matter 里加一行：

```yaml
sticky: 1
```

就这么简单。效果有两个：

1. 这篇文章排到**首页最前面**（在时间排序之上）
2. 首页卡片标题旁出现一个**图钉图标**

### 多篇文章置顶时的顺序

`sticky` 的值是**数字，越大越靠前**：

```yaml
sticky: 3     # 排第一
sticky: 2     # 排第二
sticky: 1     # 排第三
```

`sticky` 值相同的文章之间，仍按发布日期从新到旧排。

### 取消置顶

把那行删掉，或者写成 `sticky: 0`。图钉图标也会一起消失。

### 两个容易踩的点

- **`sticky` 只影响首页。** 归档页、标签页、分类页仍然老老实实按时间排，这是正常设计（`hexo-generator-index` 只管首页那一份列表）。
- **别用 `top: true` 代替。** 主题里判断图钉图标的条件是"有 `top` 或 `sticky > 0`"，但排序只看 `sticky`。所以写 `top: true` 会出现"有图钉、但没排到前面"的诡异效果。

---

## 5. 文件与目录作用

### 根目录

| 名称 | 是什么 | 要不要提交到 Git |
|---|---|---|
| `_config.yml` | **站点主配置**：标题、网址、时区、分页、各插件的站点级参数 | ✅ 必须 |
| `_config.butterfly.yml` | **主题配置**：导航栏、头像、背景图、搜索、字数统计、深色模式 | ✅ 必须 |
| `_config.landscape.yml` | `hexo init` 留下的默认主题配置。当前用的是 butterfly，这个文件**没用**，留着无害 | ✅ 会一起提交 |
| `package.json` | 项目信息和依赖清单（hexo、主题、各插件） | ✅ 必须 |
| `package-lock.json` | 依赖的精确版本锁定，保证本地和 GitHub 构建出一样的结果 | ✅ 必须 |
| `.gitignore` | 告诉 Git 哪些文件不要提交 | ✅ 必须 |
| `.gitattributes` | 统一换行符（仓库内外都用 LF），顺带消除 Git 的 LF/CRLF 警告 | ✅ 必须 |
| `README.md` | 本文档 | ✅ 会一起提交 |
| `scaffolds/` | **新建文件的模板**。`post.md` 管文章，`page.md` 管独立页面，`draft.md` 管草稿 | ✅ 必须 |
| `source/` | **你真正的写作目录**，站点内容都从这里生成 | ✅ 必须 |
| `themes/` | 传统主题目录。里面只有一个 `.gitkeep`，这是**正常的**——主题从 `node_modules` 解析 | ✅ 会一起提交 |
| `node_modules/` | 依赖包，几百 MB | ❌ 已在 `.gitignore` 里 |
| `public/` | **构建产物**（真正的 HTML/CSS/JS）。每次构建重新生成，是它在被部署到线上 | ❌ 已在 `.gitignore` 里 |
| `db.json` | Hexo 的缓存文件，删了会自动重建 | ❌ 已在 `.gitignore` 里 |
| `.github/` | GitHub 相关配置，见下 | ✅ 必须 |

### `source/` 内部

| 名称 | 作用 |
|---|---|
| `source/_posts/` | **所有文章**。这是你平时打交道最多的目录 |
| `source/_drafts/` | 草稿，不会被发布（还没创建，用 `hexo new draft` 时会自动生成） |
| `source/img/` | 图片。里面放的东西会被**原样复制**到站点根目录的 `/img/` 下 |
| `source/images/` | 同上，对应 `/images/`。两个目录只是历史遗留的两种命名习惯 |
| `source/tags/`、`source/categories/` | 标签页和分类页，**目前还没建**，见[第 11 章](#11-还没做的事) |

> `source/` 下任何非下划线开头的文件或目录（如图片、PDF、`CNAME`）都会原样复制到站点里。所以图片路径直接写 `/img/xxx.png` 就能访问。

### `.github/` 内部

| 名称 | 作用 |
|---|---|
| `workflows/deploy.yml` | **自动部署流程**。你 push 到 `main` 分支后，GitHub 会自动跑：安装依赖 → `hexo generate` → 把 `public/` 发布到 Pages |
| `dependabot.yml` | `hexo init` 自带的依赖升级机器人，会偶尔提 PR 提醒有新版本，无害 |

**这套流程的关键好处**：仓库里存的是**源码**，`public/` 不进仓库。所以历史记录干净、diff 可读，也不会出现"产物和源码打架"的情况。

---

## 6. 三个配置文件怎么分工

这三个文件容易混，一句话记住：

| 文件 | 谁在用 | 管什么 |
|---|---|---|
| `_config.yml` | Hexo 本体 + 所有插件 | 站点标题、网址、时区、分页、URL 格式，以及 `search` / `sitemap` / `feed` 这些**插件参数** |
| `_config.butterfly.yml` | Butterfly 主题 | 导航栏、头像、背景图、代码高亮样式、深色模式、搜索框、字数统计 |
| `_config.landscape.yml` | 没人用 | 默认主题的配置，当前主题是 butterfly，忽略它 |

### 一个容易困惑的点

打开 `_config.butterfly.yml` 会发现它只有 68 行，但你的博客明显有文章目录、版权卡片这些功能——**这些东西的配置并不在这个文件里**。

原因：Butterfly 在生成页面之前，会把它内置的**默认配置**和你的 `_config.butterfly.yml` 合并（后者覆盖前者）。所以：

> **`_config.butterfly.yml` 是"增量覆盖"，只写你想改的项就行，没写的都用主题默认值。**

主题的完整默认值放在：

```
node_modules/hexo-theme-butterfly/scripts/common/default_config.js
```

想知道某个功能有哪些可选项，去这个文件里搜关键词就行（比如搜 `toc`、`comments`、`reward`）。

### 搜索功能的配置为什么分两处

这是之前踩过的坑，记一下免得再犯：

- `_config.yml` 里的 `search:` 块 → **插件的**参数（生成索引文件 `search.xml` 时的规则）
- `_config.butterfly.yml` 里的 `search.use: local_search` → **主题的**开关（决定导航栏要不要画搜索按钮）

**两处都配齐搜索才会生效**，少一处就会出现"插件装好了但按钮不见"或"按钮点了没反应"。

---

## 7. 本地调试

### 三个命令的区别

| 命令 | 作用 | 什么时候用 |
|---|---|---|
| `npx hexo clean` | 删除 `public/` 和 `db.json`（缓存） | 改完配置、或结果诡异时 |
| `npx hexo generate` | 生成静态文件到 `public/` | 验证"能不能构建成功" |
| `npx hexo server` | 启动本地服务器预览 | 写文章时开着看效果 |

`generate` 和 `clean` 可以连起来：`npx hexo clean && npx hexo generate`。

### 什么时候必须 clean

- 改了 `_config.yml` 或 `_config.butterfly.yml`（配置有缓存）
- 删掉了文章，但首页还显示着
- 页面样式莫名其妙错乱

改**文章正文**不用 clean，`generate` 就够；开着 `hexo server` 时甚至连 generate 都不用，它会自动监听并重新生成，浏览器刷新一下即可。

### 看构建有没有出错

构建正常时最后一行是：

```
INFO  20 files generated in 1.87 s
```

出现 `ERROR` 或 `WARN` 才需要处理。想确认全站能不能正常工作，跑一次 `npx hexo clean && npx hexo generate` 看有没有红色报错，这是**上线前最划算的一次检查**。

---

## 8. 上线流程与自查清单

### 发布是怎么跑起来的

```
你 git push 到 main
      ↓
GitHub Actions 读取 .github/workflows/deploy.yml
      ↓
装 Node 22 → npm ci 装依赖 → npx hexo generate 生成 public/
      ↓
把 public/ 作为 Pages 的产物发布
      ↓
https://xhiaoe.github.io 更新（约 1 分钟）
```

所以你**只需要管源码**，不用手动生成、不用 `hexo d`。`public/` 交给 GitHub 去算。

### 上线自查清单

- [ ] **仓库 Settings → Pages → Source 必须选 `GitHub Actions`**（默认是 "Deploy from a branch"，不改的话构建跑成功但网页不更新）
- [ ] **仓库必须是 Public**（免费账号的用户站点仓库不能是私有）
- [ ] 建议把仓库名改成全小写的 **`xhiaoe.github.io`**（GitHub 官方文档要求：用户名含大写字母时仓库名必须小写）。改名后本地要同步：
  ```bash
  git remote set-url origin https://github.com/XHiaoe/xhiaoe.github.io.git
  git remote -v      # 确认两条地址都变了
  ```
- [ ] push 后去 Actions 页面确认流程变成绿色 ✓

---

## 9. 图片资源清单

`_config.butterfly.yml` 里引用了 4 张图，但 `source/` 下目前一张都没有，所以首页有裂图。按下面的路径放进去即可（目录已建好，是空的）：

| 配置项 | 需要的文件 | 建议 |
|---|---|---|
| 网站图标 `favicon` | `source/img/favicon.png` | 32×32 或 64×64 的小图标 |
| 导航栏 logo | `source/img/favicon.png` | 同上，共用一张 |
| 首页顶部横幅 `index_img` | `source/img/banner.jpg` | 宽 1920，高 300～400 左右 |
| 归档/标签/分类页顶部 | `source/img/bg.jpg` | 同上 |
| 全站背景 `background` | `source/img/bg.jpg` | 同上，共用一张 |
| 侧栏头像 `avatar` | `source/images/avatar.jpg` | 正方形，300×300 左右 |

放好图片后 `npx hexo clean && npx hexo generate` 重新构建，再 push 就生效了。

> 不想用图也可以：把 `_config.butterfly.yml` 里对应的行**注释掉**（行首加 `#`），按钮和卡片就不会再去找这些文件。现在先留着，等你准备图的时候一次搞定。

---

## 10. 疑难速查

### `git push` 报 `Recv failure: Connection was reset`

国内直连 `github.com:443` 会被重置。你这台机器上有一个可用的本地代理（Clash 系列，端口 7897），但 **Git 不读 Windows 的系统代理设置**，要显式告诉它：

```bash
git config --global http.proxy http://127.0.0.1:7897
git config --global https.proxy http://127.0.0.1:7897
```

（这两条是全局配置，配一次就够。以后不想要了用 `git config --global --unset http.proxy`）

还有个隐患：如果 Git Bash 里存在 `HTTPS_PROXY` 环境变量指向别的端口，会干扰推送，用 `env | grep -i proxy` 查一下。

### `warning: LF will be replaced by CRLF`

Windows 换行是 CRLF、Linux 是 LF，Git 转换时的一句提示，**不是错误**，不影响任何功能。项目根目录已经加了 `.gitattributes` 来统一这件事，以后应该不会再刷。

### `npm warn install-scripts`

新版本 npm 的脚本白名单机制拦截了依赖包的安装脚本，**只警告不报错**，安装其实成功了。Hexo 主体功能不受影响，可以直接无视。

### 首页有裂图

见[第 9 章](#9-图片资源清单)，是 4 张图没放。

### 搜索按钮不显示 / 点了没反应

检查两个地方都配了没：`_config.yml` 的 `search:` 块 + `_config.butterfly.yml` 的 `search.use`。详见[第 6 章](#6-三个配置文件怎么分工)。

### 推送成功但线上没变化

1. 去 Actions 页面看流程是否跑成功（绿色 ✓）
2. 检查仓库 Settings → Pages → Source 是不是 `GitHub Actions`
3. 浏览器强刷（`Ctrl + F5`），Pages 有 CDN 缓存

### 本地预览能看到新文章，线上没有

九成是**忘了 `git commit` 或 `git push`**。在 Git Bash 里 `git status` 看一眼有没有未提交的改动。

---

## 11. 还没做的事

按优先级排列，都是**可选**的，不影响博客正常访问：

### ① 导航栏没有任何菜单项

现在站点顶部只有一个站名和搜索按钮，**访客没有办法进入归档页、标签页、分类页**——因为 `_config.butterfly.yml` 里没有 `menu:` 配置。

想加的话分两步。先在 `_config.butterfly.yml` 加：

```yaml
menu:
  首页: / || fas fa-home
  归档: /archives/ || fas fa-archive
  标签: /tags/ || fas fa-tags
  分类: /categories/ || fas fa-folder-open
```

然后创建标签页和分类页（`/archives/` 是自动生成的，不用建）：

```bash
npx hexo new page tags
npx hexo new page categories
```

打开生成的 `source/tags/index.md`，把 front-matter 改成：

```yaml
---
title: 标签
date: 2026-09-17 10:00:00
type: tags
---
```

`source/categories/index.md` 同理，写 `type: categories`。改完重新构建推送即可。

### ② 4 张图片还没放

见[第 9 章](#9-图片资源清单)。

### ③ 仓库名建议改成小写

`XHiaoe.github.io` → `xhiaoe.github.io`，理由见[第 8 章](#8-上线流程与自查清单)。

### ④ 评论功能还没开

`comments.use` 是空的，文章底部没有评论区。想加的话推荐 **Giscus**（免费，基于 GitHub Discussions，不需要备案、不需要服务器），需要你在 GitHub 上给仓库开启 Discussions 权限。

### ⑤ 默认示例文章还在

`source/_posts/hello-world.md` 是 `hexo init` 生成的英文示例，内容对访客没什么用。可以删掉，或者改写成一篇自己的开篇说明。

---

*最后更新：2026-09-17*
