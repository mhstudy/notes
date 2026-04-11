# Docsify 文档站点搭建指南

> docsify 官网：<https://docsify.js.org/>

## 1. 什么是 Docsify ⭐

Docsify 是一个**动态生成文档网站**的工具，不会将 `.md` 转成 `.html` 文件，所有转换工作都是在运行时完成。只需要创建一个 `index.html` 就可以开始文档之旅。

**核心特点**：
- 🚀 无需构建，写完 Markdown 直接发布
- 🔍 全文搜索插件
- 📱 响应式布局，支持移动端
- 🎨 多套主题可选
- 🔌 丰富的插件生态

## 2. 快速开始

### 2.1 安装 docsify-cli

```bash
# 全局安装 docsify 命令行工具
npm i docsify-cli -g
```

### 2.2 初始化项目

```bash
# 初始化项目
docsify init ./docs

# 生成的文件结构
# docs/
#   ├── index.html   # 入口文件
#   ├── README.md     # 首页内容
#   └── .nojekyll     # 阻止 GitHub Pages 忽略下划线开头的文件
```

### 2.3 本地预览

```bash
# 启动本地服务器，默认 http://localhost:3000
docsify serve docs
```

## 3. 核心配置（index.html）⭐

```html
<!DOCTYPE html>
<html lang="zh-CN">
<head>
  <meta charset="UTF-8">
  <title>我的文档</title>
  <meta name="viewport" content="width=device-width,initial-scale=1">
  <!-- 主题样式 -->
  <link rel="stylesheet" href="//cdn.jsdelivr.net/npm/docsify@4/lib/themes/vue.css">
</head>
<body>
  <div id="app">加载中...</div>
  <script>
    window.$docsify = {
      name: '文档标题',           // 侧边栏标题
      repo: 'https://github.com/xxx', // 右上角 GitHub 图标
      loadSidebar: true,         // 加载自定义侧边栏
      loadNavbar: true,          // 加载自定义导航栏
      coverpage: true,           // 启用封面页
      subMaxLevel: 3,            // 侧边栏自动生成目录的最大层级
      auto2top: true,            // 切换页面自动滚动到顶部
      search: {                  // 全文搜索配置
        placeholder: '搜索',
        noData: '没有结果',
        depth: 6
      }
    }
  </script>
  <!-- docsify 核心 -->
  <script src="//cdn.jsdelivr.net/npm/docsify@4"></script>
  <!-- 搜索插件 -->
  <script src="//cdn.jsdelivr.net/npm/docsify/lib/plugins/search.min.js"></script>
  <!-- 代码高亮 -->
  <script src="//cdn.jsdelivr.net/npm/prismjs@1/components/prism-java.min.js"></script>
  <script src="//cdn.jsdelivr.net/npm/prismjs@1/components/prism-python.min.js"></script>
  <script src="//cdn.jsdelivr.net/npm/prismjs@1/components/prism-sql.min.js"></script>
  <script src="//cdn.jsdelivr.net/npm/prismjs@1/components/prism-bash.min.js"></script>
  <script src="//cdn.jsdelivr.net/npm/prismjs@1/components/prism-scala.min.js"></script>
</body>
</html>
```

## 4. 侧边栏配置（_sidebar.md）🔥

在项目根目录创建 `_sidebar.md` 文件：

```markdown
* [首页](/)
* **分类一**
  * [文档1](path/to/doc1.md)
  * [文档2](path/to/doc2.md)
* **分类二**
  * [文档3](path/to/doc3.md)
```

**注意事项**：
- 需要在 `index.html` 中设置 `loadSidebar: true`
- 文件路径是相对于项目根目录的
- 路径中含空格需用 `%20` 替代
- 支持多层嵌套

## 5. 导航栏配置（_navbar.md）

```markdown
* [首页](/)
* [GitHub](https://github.com)
* 更多
  * [关于](about.md)
```

## 6. 封面页配置（_coverpage.md）

```markdown
![logo](logo.png)

# 项目名称

> 一句话描述

- 特性 1
- 特性 2

[GitHub](https://github.com/xxx)
[开始阅读](#首页)
```

## 7. 常用插件 ⭐

| 插件 | 用途 | CDN |
|------|------|-----|
| search | 全文搜索 | `docsify/lib/plugins/search.min.js` |
| zoom-image | 图片缩放 | `docsify/lib/plugins/zoom-image.min.js` |
| pagination | 上下页翻页 | `docsify-pagination/dist/docsify-pagination.min.js` |
| copy-code | 代码复制按钮 | `docsify-copy-code/dist/docsify-copy-code.min.js` |
| count | 字数统计 | `docsify-count/dist/countable.min.js` |
| tabs | 标签页 | `docsify-tabs/dist/docsify-tabs.min.js` |
| progress | 阅读进度条 | `docsify-progress` |

## 8. 部署到 GitHub Pages 🔥

### 8.1 推送到 GitHub

```bash
# 初始化 Git 仓库
git init
git add .
git commit -m "初始化文档"
git remote add origin https://github.com/username/repo.git
git push -u origin master
```

### 8.2 开启 GitHub Pages

1. 进入仓库 → **Settings** → **Pages**
2. **Source** 选择 `master` 分支，目录选 `/ (root)` 或 `/docs`
3. 点击 **Save**
4. 等待几分钟后访问 `https://username.github.io/repo/`

### 8.3 自定义域名（可选）

1. 在仓库根目录创建 `CNAME` 文件，写入你的域名
2. DNS 添加 CNAME 记录指向 `username.github.io`

## 9. PWA 离线访问

在 `index.html` 中添加 Service Worker 注册：

```javascript
// pwa.js
if (typeof navigator.serviceWorker !== 'undefined') {
  navigator.serviceWorker.register('pwa.js')
}
```

然后在 docsify 配置中添加：

```javascript
window.$docsify = {
  // ...其他配置
  serviceWorker: {
    cacheName: 'my-docs',
    cacheKey: 'v1'
  }
}
```

## 10. 常用 Markdown 增强语法

### 提示框

```markdown
> [!NOTE]
> 这是一个提示

> [!TIP]
> 这是一个小技巧

> [!WARNING]
> 这是一个警告
```

### 嵌入文件

```markdown
[filename](path/to/file.md ':include')
```

### 忽略编译

```markdown
[link](/demo ':ignore')
```

## 11. 常见问题

| 问题 | 解决方案 |
|------|----------|
| 侧边栏不显示 | 确认 `loadSidebar: true` 且 `_sidebar.md` 存在 |
| 中文路径404 | 路径空格用 `%20`，特殊字符用 URL 编码 |
| GitHub Pages 404 | 添加 `.nojekyll` 文件到根目录 |
| 搜索不生效 | 引入 search 插件且配置 `search` 选项 |
| 图片不显示 | 检查图片路径是否相对于当前 md 文件 |
