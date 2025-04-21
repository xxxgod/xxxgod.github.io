

1. npm install -g hexo-cli 安装

2. hexo serve 运行 

3. 生成文章：

```bash
hexo new [layout] <文章标题>
```

- `[layout]`

  ：可选参数，表示文章的布局类型（默认是post）。常见的布局类型有：

  - `post`：普通文章（默认）。
  - `page`：页面（如关于页、标签页等）。

- **`<文章标题>`**：文章的标题，Hexo 会自动将其转换为文件名（空格会被替换为 `-`，并移除特殊字符）。

`4. hexo clean && hexo deploy` 部署博客到GitHub Pages

