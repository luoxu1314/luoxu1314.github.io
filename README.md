# 落絮的博客

https://luoxu1314.github.io ，用 [Astro](https://astro.build) 搭的。

## 写文章

在 `src/content/blog/` 下新建一个 `.md`，开头写好：

```md
---
title: '标题'
description: '一句话简介'
pubDate: 2026-10-12
tags: ['标签']
---
```

带图片的文章建一个文件夹，正文放 `index.md`，图片放旁边的 `images/`，用 `./images/xxx.png` 引用。要给人下载的文件放 `public/` 下面。

## 本地预览

```sh
npm install
npm run dev
```

源码在 `main` 分支。推到 `main` 后 GitHub Actions 会自动构建，把生成的网页推到 `master` 分支，GitHub Pages 从 `master` 发布。旧的 Hexo 站点留在 `backup-2022` 分支。

## 搜索引擎

`public/robots.txt` 指向站点地图。`public/f9ebc6aae527967b6817e39a11388680.txt` 是 IndexNow 的密钥文件（Bing 用），别删。发了新文章想让 Bing 快点收录，可以运行：

```sh
curl -s -X POST https://api.indexnow.org/indexnow -H 'Content-Type: application/json' -d '{"host":"luoxu1314.github.io","key":"f9ebc6aae527967b6817e39a11388680","urlList":["https://luoxu1314.github.io/blog/文章名/"]}'
```
