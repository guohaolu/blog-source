# Hexo 个人博客

这是我的个人技术博客，基于 Hexo 框架搭建，使用 Next 主题。

## 项目结构
blog-source/
├── .github/ # GitHub 相关配置
│ └── workflows/ # GitHub Actions 工作流配置
│ └── deploy.yml # 自动部署配置文件
├── node_modules/ # 依赖包目录
├── scaffolds/ # 模板文件夹
│ ├── draft.md # 草稿模板
│ ├── page.md # 页面模板
│ └── post.md # 文章模板
├── source/ # 源文件夹
│ ├── posts/ # 博客文章目录
│ │ └── 第一篇博客.md # 博客示例：Hexo博客搭建教程
│ ├── about/ # 关于页面
│ │ └── index.md
│ ├── categories/ # 分类页面
│ │ └── index.md
│ ├── tags/ # 标签页面
│ │ └── index.md
│ └── images/ # 图片资源目录
│ └── avatar.jpg # 博主头像
├── themes/ # 主题文件夹
│ └── next/ # Next 主题
├── .cursorrules # 项目规则配置
├── .gitignore # Git 忽略文件配置
├── config.yml # Hexo 主配置文件
├── config.next.yml # Next 主题配置文件
├── package.json # 项目依赖配置
└── README.md # 项目说明文档

## 博客文章

### 技术博客
- [第一篇博客](source/_posts/第一篇博客.md)
  - 发布时间：2024-03-19
  - 分类：技术博客
  - 标签：Hexo, 教程
  - 内容：博客搭建教程和经验分享

## 部署信息

- 博客地址：https://guohaolu.github.io
- 部署方式：GitHub Actions 自动部署
- 源码仓库：https://github.com/guohaolu/blog-source
- 部署仓库：https://github.com/guohaolu/guohaolu.github.io

## 主题配置

使用 Next 主题的 Gemini 方案，特点：
- 双栏布局（左侧边栏）
- 响应式设计
- 支持文章目录
- 支持文章分类和标签
- 集成搜索功能
- 访问统计
- 阅读进度条