# hermes-blog

个人博客的源码仓库。[Hugo](https://gohugo.io/) 静态站点 + GitHub Pages 部署，主题用 [PaperMod](https://github.com/adityatelange/hugo-PaperMod)。

- 线上地址：<https://mywebtestmail.github.io/hermes-blog/>
- 发布分支：`main`（合并即自动部署）
- 文章目录：`content/posts/`

## 目录结构

```
content/posts/        文章（Markdown）
content/search.md     搜索页（PaperMod 要求）
archetypes/posts.md   新文章模板，含 title/date/tags/author 四个字段
hugo.yaml             站点配置
themes/PaperMod/      主题（git submodule）
.github/workflows/    hugo.yml：构建 + 部署到 Pages
```

## 写一篇新文章

1. 拉最新 `main`：`git checkout main && git pull --ff-only`
2. 开分支：`git checkout -b post/<slug>`
3. 新建文件（推荐用模板，自动带上四个字段）：
   `hugo new content/posts/<slug>.md`
   或者手动创建 `content/posts/<slug>.md`，frontmatter 必须是：

   ```markdown
   ---
   title: "文章标题"
   date: 2026-09-10T10:00:00+08:00
   tags: ["标签一", "标签二"]
   author: "mywebtestmail"
   ---
   ```

4. 本地预览（可选）：`hugo server -D`，浏览器打开 <http://localhost:1313/hermes-blog/>
5. 提交并推送：`git add content/posts/<slug>.md && git commit -m "post: <标题>" && git push -u origin post/<slug>`
6. 开 PR 指向 `main`，等 CI 的 Build site 变成绿色
7. 合并 PR → `main` 触发构建与部署
8. 合并后删掉 `post/<slug>` 分支

> PR 阶段只跑构建检查、不部署；只有合并进 `main` 才会发布。

## 本地环境

```bash
# Hugo extended（PaperMod 需要 extended 才能编译 SCSS）
hugo version   # 本机为 v0.166.0+extended
# 首次克隆记得带上主题子模块
git clone --recurse-submodules git@github.com:mywebtestmail/hermes-blog.git
# 已有克隆则补一次
git submodule update --init --recursive
```

## 部署

GitHub Actions 工作流 `.github/workflows/hugo.yml`：安装 Hugo extended → `hugo --minify --gc` → 上传 `public/` → 部署到 GitHub Pages（`build_type: workflow`）。Hugo 版本锁在 workflow 的 `HUGO_VERSION` 里。

## 版本

| 组件 | 版本 |
|---|---|
| Hugo | 0.166.0 extended |
| PaperMod | `d376885`（master，2026-08-02）|
| actions/checkout | v7 |
| actions/upload-pages-artifact | v5 |
| actions/deploy-pages | v5 |
