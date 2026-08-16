# ryuchan-content

RyuChan 博客的内容仓库。这里只放**内容**：文章、结构化数据、站点配置、文章配图、分析脚本。前端代码（Astro 模板、组件、样式、构建配置）在仓库 `RyuChan` 中，可选的可视化 CMS 管理端在仓库 `RyuChan-CMS` 中。

拆分的目的是让写文章和改代码互不干扰：内容仓的提交历史干净，可以被 RyuChan-CMS 管理端直接编辑，也可以单独设为私有。

## 系统组成

| 仓库 | 职责 | 地址 |
|------|------|------|
| **RyuChan**（前端仓） | Astro 静态博客展示 | [kobaridev/RyuChan](https://github.com/kobaridev/RyuChan) |
| **RyuChan-Content**（内容仓） | 本文档所在仓库，存储所有内容 | 当前仓库 |
| **RyuChan-CMS**（管理端·可选） | React 可视化内容管理后台（可选） | [kobaridev/RyuChan-CMS](https://github.com/kobaridev/RyuChan-CMS) |

## 目录结构

```
ryuchan-content/
├── src/content/
│   ├── footer/
│   │   ├── config.yaml                      # 页脚：社交链接与备案信息
│   ├── site/config.yaml                   # 站点通用配置：标题/描述/主题/favicon/备案/banner
│   ├── user/config.yaml                   # 用户信息：头像/描述/社交链接/收款码
│   ├── blog/
│   │   ├── config.yaml                      # 博客模块标题/副标题 + 每页文章条数 pageSize
│   │   └── src/                             # 文章，一条集录一篇文章
│   │       ├── adding-comment-systems.md
│   │       ├── markdown-style-guide.md
│   │       ├── mathematics-examples.md
│   │       ├── ryuchan-mdx.mdx
│   │       └── using-mdx.mdx
│   ├── about/
│   │   ├── config.yaml                      # 关于页标题/副标题
│   │   └── src/                             # 关于页正文
│   │       └── index.md
│   ├── album/
│   │   ├── config.yaml                      # 相册页标题/副标题
│   │   └── categories/                      # 一个相册一个文件（一组照片）
│   │       ├── 01.yaml
│   │       └── ...
│   ├── friends/
│   │   ├── config.yaml
│   │   └── list/                            # 一条友链一个文件
│   │       ├── 01.yaml
│   │       └── ...
│   ├── project/
│   │   ├── config.yaml
│   │   └── src/                             # 一个项目一个文件
│   │       ├── 01.yaml
│   │       └── ...
│   ├── navigation/
│   │   ├── config.yaml
│   │   └── categories/                      # 一个分组一个文件
│   │       ├── 01.yaml
│   │       └── ...
│   ├── analysis/
│   │   ├── config.yaml                      # 分析脚本开关与 provider
│   │   └── provider/
│   │       ├── umami.head.html              # 统计脚本（umami）
│   │       └── claity.head.html             # 统计脚本（claity）
│   ├── anime/
│   │   ├── config.yaml                      # 追番页标题/副标题 + 数据源列表
│   │   └── provider/
│   │       ├── bilibili.yaml                # Bilibili uid
│   │       └── tmdb.yaml                    # TMDB apiKey + listId
│   ├── music/
│   │   ├── config.yaml                      # 音乐页标题/副标题 + Meting API 基址
│   │   ├── list/                              # 播放列表索引，一个文件一个歌单
│   │   └── custom/                            # 自定义歌单数据，文件名格式 ID.yaml，含完整歌曲信息
│   │       ├── 01.yaml
│   │       └── ...
│   └── comments/
│       ├── config.yaml                      # 评论开关与当前选用的 provider
│       └── provider/
│           ├── twikoo.yaml                  # Twikoo envId
│           ├── waline.yaml                  # Waline serverURL + lang
│           └── giscus.yaml                  # Giscus GitHub Discussions 配置
├── assets/
│   ├── media/                               # 文章配图，构建时映射到 public/image/
│   └── brand/                               # 站点级图片，构建时展开到 public/ 根
│       ├── favicon.ico
│       ├── logo.png
│       ├── profile.png
│       ├── home.webp
│       └── qrcode/
│           ├── Alipay.jpg
│           └── WeChat.jpg
└── ryucms.schema.json                       # RyuChan-CMS 管理端的 collection 定义
```

## 前端仓如何消费

前端仓在构建前把本仓库 clone 到工作区，`prebuild` 脚本把内容同步到 Astro 期望的位置。约定的映射关系是：

| 内容仓 | 前端仓 |
|---|---|
| `src/content/blog/src/` | `src/content/blog/` |
| `src/content/friends/list/*.yaml` | 合成 → `src/data/friends.yaml` |
| `src/content/project/src/*.yaml` | 合成 → `src/data/projects.yaml` |
| `src/content/navigation/categories/*.yaml` | 合成 → `src/data/navigation.yaml` |
| `src/content/album/categories/*.yaml` | 合成 → `src/data/albums.json` |
| `src/content/site/config.yaml` | `config.site.*`（除 pages/menu 外） |
| `src/content/user/config.yaml` | `config.user.*` |
| `src/content/blog/config.yaml` | `config.site.pages.home.*` + `config.site.blog` |
| `src/content/music/list/*.yaml` | 合成 → `ryuchan.config.yaml`（playlists 配置） |
| `src/content/music/custom/*.yaml` | 自定义歌单数据，由 `fetch-music-duration.mjs` 读取并注入 `music.json` |
| `src/content/footer/config.yaml` | 注入前端页脚（社交链接 + 备案信息） |
| `src/content/comments/provider/*.yaml` | Astro 构建时读取 |
| `assets/media/` | `public/image/`（注意：目录名 media，引用路径 `/image/`） |
| `assets/brand/` | `public/`（根，**展开**，`brand/qrcode/` 下文件也展开到根） |
| `src/content/analysis/provider/*.html` | 注入 `<head>`（按 provider 配置） |
| `src/content/anime/provider/*.yaml` | Astro 构建时读取 |

`assets/brand` **展开**到 `public/` 根目录（含 `brand/qrcode/` 下的二维码文件），让 `/logo.png`、`/profile.png`、`/favicon.ico`、`/Alipay.jpg`、`/WeChat.jpg` 等绝对路径引用继续有效——配置文件和文章正文里的路径不需要改。

## 写作约定

### 文章

文章 frontmatter 必填 `title`、`description`、`pubDate`，可选 `updated`、`image`、`badge`、`draft`、`categories`、`tags`。

`pubDate` 在 schema 里是自由字符串而不是 `date` 格式，因为仓库里历史文章存在 `Jul 01 2024`、`07 12 2024`、`2025-04-15T00:00` 三种写法，Astro 用 `z.coerce.date()` 全部容忍。新文章建议统一写 ISO 形式 `2026-08-14T00:00`，但不要给 schema 加 `format: date` 去强制，否则旧文章会报错。

配图文件放在 `assets/media/`，但**引用路径统一写 `/image/`**（因为构建时 `assets/media/` 会同步到前端仓的 `public/image/`）。例如文件 `assets/media/image1.webp`，frontmatter 写 `image: /image/image1.webp`，正文写 `![封面](/image/image1.webp)`。

`.mdx` 文章可以使用前端仓 `src/components/mdx/` 下的组件（如 `<Diff l="/image/l.webp" r="/image/r.webp" />`），这类组件的可用性由前端仓决定，改动组件签名时两个仓库要一起调整。

### 友链、项目、相册、分组

每个条目单独一个文件（`01.yaml`、`02.yaml` ...）。可手动维护，也可通过 RyuChan 内置的 `/write`、`/config` 等页面或 RyuChan-CMS 管理端编辑。

相册 `categories/01.yaml` 是一个相册的元数据 + 照片列表，不是单个照片。照片在 `photos` 数组里，字段 `src` / `title` / `description` / `variant`（`1x1`/`4x3`/`4x5`/`9x16`）。导航里 `navigations` 是数组，每条 `name` / `url` / `avatar` / `description` / `badge` 等。

### 站点配置

各模块的 `config.yaml` 对应原 `ryuchan.config.yaml` 的不同层级：
- `site/` → `site` 下通用字段（标题、主题、favicon、备案、banner）
- `user/` → `user` 下个人信息与社交链接
- `blog/`、`album/` 等 → `site.pages.*` 对应页面标题/副标题

本地资源引用（`/logo.png`、`/profile.png`、`/Alipay.jpg` 等）保持不变，图片文件在 `assets/brand/` 里。

> GitHub App 认证凭据（`appId`、`encryptKey` 等）不在内容仓管理，请通过前端仓的环境变量配置：`PUBLIC_GITHUB_APP_ID`、`PUBLIC_GITHUB_ENCRYPT_KEY`。

## 不属于这里的东西

以下文件刻意留在前端仓，因为它们是生成物或不该手工编辑：

- `src/data/music.json`（`prefetch:music` 脚本产出）
- `public/anime-list.json`（TMDB 抓取产出）
- `src/i18n/translations.yaml`（多语言字典）
- `public/pagefind/`（搜索索引）

Meting API 统一处理音乐数据请求，`music/config.yaml` 中的 `api` 字段提供 API 基址。`list/` 下一个文件对应一个播放列表，`songs` 数组里每条用 `index`（歌单 ID 或自定义标识）+ `provider`（`netease` / `tencent` / `custom`）标识来源。`custom/` 目录下存放自定义歌单的完整歌曲数据（title、artist、cover、url、lrc、duration），`provider: custom` 的 `index` 指向 `custom/` 下的文件名前缀（不含 `.yaml` 后缀）。构建时 `fetch-music-duration.mjs` 会将自定义数据注入 `music.json`，无需在线拉取。`provider/` 目录已删除——平台参数由 Meting API 内部处理，不需要本地配置。

## 构建后内容仓的状态

内容仓**永远不会**包含以下目录（由 prebuild 在 gitignore 层排除）：
- `dist/`、`.astro/`、`node_modules/`（这是前端仓的东西）
- `public/pagefind/`、`src/data/music.json`、`public/anime-list.json`（这是前端仓生成物）

内容仓只包含上面 `## 目录结构` 列出的文件，保持干净。
