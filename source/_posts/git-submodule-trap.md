---
title: 记一次 Hexo 部署 Vercel 白屏的排查过程
date: 2026-07-06
tags:
  - Hexo
  - Git
  - Vercel
  - 踩坑记录
categories:
  - 技术
---

## 现象

用 Hexo + Icarus 主题搭好博客，推送到 GitHub，Vercel 自动部署后访问页面——**完全白屏**，控制台没有任何报错。

## 排查过程

### 1. 本地构建正常

```bash
$ hexo generate
```

本地 `public/index.html` 生成正常，18KB，标题、样式、脚本都在。

### 2. Vercel 配置正确

Build Settings 三项都配了：

| 配置 | 值 |
|------|-----|
| Build Command | `hexo generate` |
| Output Directory | `public` |
| Install Command | `npm install` |

### 3. 找到真凶

用 `git ls-tree HEAD themes/icarus` 一看：

```
160000 commit 6f2d092... themes/icarus
```

`160000` 是 **gitlink（子模块）** 的模式码。这意味着 `themes/icarus` 在仓库里只是一个 commit hash 引用，不是真实文件。GitHub 上这文件夹是灰色图标，里面是空的。Vercel 拉下来，Hexo 没有主题可以渲染，自然白屏。

## 原因

`git add -A` 时，如果某个文件夹里藏了一个 `.git` 目录，Git 会自作聪明地把它标记为**子模块**，只存引用不存文件。

我的 Icarus 主题恰好是从 GitHub 直接 clone 到 `themes/icarus/` 的，天然自带 `.git`。于是第一次 `git add` 就掉进了这个坑。

后来即使删除了 `themes/icarus/.git`，Git 索引已经记住了「它是子模块」，再次 `git add` 不会自动纠正。

## 修复

两步搞定：

```bash
# 1. 把子模块引用从 Git 索引里删掉
git rm --cached themes/icarus

# 2. 以普通文件身份重新加入
git add themes/icarus/
```

提交推送后，129 个主题文件正常入库，Vercel 重新部署，页面终于出来了。

## 总结

- clone 下来的主题/依赖，进仓库前先 `rm -rf xxx/.git`，否则会被 Git 识别为子模块
- 怀疑文件没上传时，用 `git ls-tree HEAD <path>` 检查模式码，`160000` = 子模块，`100644` = 普通文件
- 推完代码上 GitHub 看一眼，灰色文件夹图标就说明没传上去
