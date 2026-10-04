## 这篇文章

- 标题：
- 文件路径：`content/posts/…`
- 标签：

## 发布前检查

- [ ] frontmatter 四个字段齐全：`title` / `date` / `tags` / `author`
- [ ] `date` 不是未来时间——Hugo 默认不构建未来日期的文章，页面会静默消失（CI 不会报错）
- [ ] 本地 `hugo --minify --gc` 无报错，或等 CI 的 **Build site** 变绿
- [ ] 标题、标签与正文内容相符，没有编造的链接或数据
- [ ] 正文通读一遍，去掉 AI 腔

## 合并后

合并进 `main` 即触发构建与部署，约 1 分钟。部署完成后确认 <https://mywebtestmail.github.io/hermes-blog/> 上能看到这篇文章，然后删掉 `post/<slug>` 分支。
