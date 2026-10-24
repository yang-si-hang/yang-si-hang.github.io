# al-folio 升级迁移方案

## 目标与原则

将本网站从当前传统 al-folio 模板迁移到上游当前稳定版本，同时保留个人内容、网站地址和 GitHub Pages 发布方式。迁移完成并通过检查前，现有 `main` 和线上站点继续作为可用版本。

- 在独立分支完成迁移，不直接在 `main` 上开发。
- 不覆盖、移动或清理当前未跟踪的用户文件。
- 只迁移用户确认正在使用的个人资料、论文、项目和博客；排除模板与 Einstein 示例内容。
- 只有检查通过后才合入 `main`；回退使用 `git revert`，不重写历史。

## 当前基线

- 迁移开始时的 `main` 提交：`dfe2e69b80ac192152d5f40894e97e949caddc69`。
- 本地仅配置个人仓库 `origin`，没有配置 al-folio 上游 remote。
- 当前站点使用传统 Jekyll、Bootstrap/SCSS 和插件 Gemfile；`Gemfile.lock` 锁定 Jekyll 4.4.1。
- 已识别需要重点适配的本地改动：`_config.yml`、`_pages/`、`_bibliography/papers.bib`、`_layouts/bib.liquid`、`_sass/_publications.scss`、`_projects/`、`_posts/` 与相关媒体。
- 迁移目标已锁定为 al-folio `v1.2`，tag commit 为 `b95d6d61a1b0663529094b1b5f4fbe8aa41f8a04`（2026-08-09 发布）。
- 当前工作区有未跟踪文件：`AGENTS.md`、`assets/pi05.drawio` 和四段项目视频。迁移中保留这些原件；仅在确认新版需要时复制媒体。
- v1.2 中将 `repo_trophies.enabled` 保持为 `false`：上游在 v1.2 中默认关闭公共奖杯服务，因为该实例返回 HTTP 402。若需要，可自托管服务后再启用。
- 用户确认不迁移 Einstein 示例 CV；之后取得真实简历资料时再单独添加 CV 页面。

## 阶段与验收点

### 0. 建立隔离与回退基线（已完成）

- [x] 记录迁移前 `main` 提交号。
- [x] 从该提交建立独立迁移分支 `codex/al-folio-migration-plan`。
- [x] 保留现有未跟踪文件，不执行清理或强制重置。

**验收：** `main` 仍指向基线提交；迁移分支未推送，不会触发线上发布。

### 1. 核实上游并建立新版基线（已完成）

- [x] 在本地 Git 配置中添加 `upstream`，指向 `https://github.com/alshedivat/al-folio.git`。
- [x] 确认最新稳定 tag 为 `v1.2`，获取上游提交和官方旧站迁移指南。
- [x] 在独立 worktree `/home/ubuntu/ysh/Code/yang-si-hang.github.io-v1.2` 建立干净新版基线。
- [x] 在迁移计划中记录目标 tag/commit；迁移配置保留现有站点 URL、个人信息和 GitHub Pages 发布目标。
- [x] 保留新版 `Gemfile`、锁文件、插件配置与部署 workflow。

**本地预审发现：** `_pages/about.md`、`_pages/projects.md`、`_pages/repositories.md` 和多个项目详情页含 Bootstrap 类；自定义论文布局也包含依赖 jQuery 的作者展开交互。这些会作为新版兼容与重做的重点。

**验收：** 目标版本明确，干净模板构建成功，迁移来源可追溯。

### 2. 迁移个人内容（已按用户确认收窄）

- [x] 保留个人简介、邮箱、Google Scholar ID、3 篇真实论文、3 个当前项目和 3 篇 2026 博客。
- [x] 保留上述项目和文章实际引用的图片、视频与真实论文 PDF。
- [x] 删除 Einstein 示例 CV、照片、简历 PDF、相对论页面及示例 PDF；移除指向示例 PDF 的错误论文链接。
- [x] 排除 al-folio 演示文章、项目、新闻、书评、课程、示例仓库和插件展示页。
- [x] 暂不迁移 CV 页面；工作区原有未跟踪视频和 drawio 原件仍保留。
- [x] 保留的原始 drawio/视频文件不进入生成站点；仅网站内容实际引用的媒体会发布。

迁移内容时从新模板保留了插件目录与运行时资源；只保留用户内容及其引用资源，没有搬入旧版 `assets/js`、`assets/css` 和字体等运行时代码。Bootstrap 内容兼容选项已开启。

**验收：** 首页、论文、项目和博客内容齐全，内部链接与媒体路径有效；不发布 Einstein 或模板示例资料。

### 3. 适配本地模板定制（进行中）

- [x] 基于 v1.2 核心布局重建 `_layouts/bib.liquid` 覆盖，保留 Google Scholar 引用徽标并修正 scholar ID 读取方式；保留新版的无 jQuery 作者展开和视频预览实现。
- [x] 基于 v1.2 核心样式重建 `_sass/_publications.scss` 覆盖，只保留引用徽标所需样式。
- [x] 修复 Bootstrap 兼容层令桌面导航不可见的问题：在 `_sass/_navbar.scss` 桌面断点恢复折叠导航的可见性。
- [x] 修复深色模式固定页脚颜色：背景跟随页面底色，文字和链接跟随主题文字色。
- [x] 审查 Bootstrap 页面标记和 SCSS 覆盖；开启官方兼容模式，后续升级时再移除旧标记。
- [x] 按新版配置契约迁移 `_config.yml`，保留站点 URL、社交、论文和页面设置。
- [x] 运行上游升级审计和覆盖审计，检查差异后登记四个本地覆盖（论文布局、论文样式、导航和页脚）。

清理后审计为 **0 个阻塞项、1 个非阻塞项**。唯一非阻塞项是论文布局为旧 Bootstrap 提示框保留的 `data-toggle` 标记，兼容模式已启用。

**验收：** 配置审计无未解决的阻塞项；保留的本地覆盖都有用途说明。

### 4. 构建与发布检查

- [x] 保留 v1.2 的 Gemfile / lockfile 与 GitHub Actions 构建步骤，沿用 GitHub Pages 发布目标。
- [x] Docker 生产环境 Jekyll build 成功。
- [x] 按用户确认移除示例内容后，Docker 生产构建再次成功。
- [x] 本地 HTTP 检查首页、博客索引及 3 篇博客、项目索引及 3 个项目详情、论文页；均返回 HTTP 200。
- [x] 生成站点中没有 Einstein/CV 示例或迁移计划文件；项目原始 drawio/视频被配置为不发布。
- [x] 清除模板默认的 Disqus `al-folio` shortname；Giscus 未配置，因此不启用评论。
- 在 Pull Request 中运行与个人站点相关的检查；合并前不得将迁移版本推送到生产发布分支。

本地构建有 Rails 时区弃用提示，未导致构建失败。PR 检查和部署切换尚未进行。

**验收：** 构建成功，关键页面和资源无明显回归，发布配置仍指向预期站点。

### 5. 切换与回退

- 全部验收通过后才将迁移分支合入 `main`，由现有部署流程发布。
- 合并前失败：保留或删除迁移分支，旧 `main` 和线上站点不变。
- 发布后发现问题：对迁移合并提交执行 `git revert` 并推送，等待 GitHub Actions 重新发布旧版。
- 不使用 `reset --hard`、强制推送或删除 `gh-pages` 作为回退手段。

**验收：** 新版线上站点确认正常；回退提交和迁移分支保留，便于审计或后续修复。

## 当前执行进度

阶段 0 至阶段 4 的本地工作和内容范围确认已完成。用户确认只保留个人简介/联系方式、真实论文、3 个项目及引用媒体、3 篇 2026 博客，并暂不迁移 CV；所有 Einstein 与模板示例内容已从迁移候选移除。修复了 Bootstrap 兼容层令桌面导航不可见的问题及深色模式固定页脚的颜色问题，并确认四个本地覆盖均已登记。清理后的生产构建成功，保留页面路由均返回 HTTP 200；升级审计为 0 个阻塞项、1 个 Bootstrap 兼容提示。PR #2（`main` <- `codex/al-folio-migration-plan`）已创建。首轮 CI 指出 Prettier 格式问题，以及两项只适用于 al-folio 官方 starter 的检查在个人站点仓库中不适用；格式已修正，starter 专用检查已限定为官方仓库运行。待将这些修正推送到 PR 分支后查看新一轮 CI。PR 工作流的部署任务已确认仅构建而未执行 Pages 发布；`main` 仍指向基线提交 `dfe2e69b80ac192152d5f40894e97e949caddc69`。PR 尚未合并。
