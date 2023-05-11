---
categories: []
date: "2023-05-11T15:29:50+08:00"
description: 准备工作
image: ""
slug: ru-he-li-yong-hugokuai-su-chuang-jian-ge-jing-tai-bo-ke-wang-zhan
tags: []
title: 如何利用hugo快速创建一个静态博客网站
---
#### 在 Github 中新建一个 public 的repositery

准备工作

1. 在github上创建一个新的repo，比如：`mysite`
2. 电脑上安装好hugo工具和 git 工具

#### 实际步骤

1. 运行hugo新建一个目录

```bash
hugo new site mysite
```

2. 进入目录

```bash
cd mysite
```

3. 绑定 GitHub 地址

```
git init
git remote add origin git@github.com:xxxx/mysite.git
```

4. 选择一个主题，官方主题 [列表](https://themes.gohugo.io/)，比如，我选中的 [stack](https://themes.gohugo.io/themes/hugo-theme-stack/).
   将主题拉下来，然后放到theme目录中

```bash
git submodule add https://github.com/CaiJimmy/hugo-theme-stack themes/stack
echo "theme = 'stack'" >> config.toml
```

#### 添加文章

```bash
 hugo new posts/如何利用hugo快速创建一个静态博客网站.md
```

#### 部署网站

1. 自动部署到 [GitHub Page](https://gohugo.io/hosting-and-deployment/hosting-on-github/)。

2. 手动部署

```bash
hugo server
```

其他的部署方式[列表](https://gohugo.io/hosting-and-deployment/)。

