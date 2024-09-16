---
title:  "Welcome to Jekyll!"
date:   2024-09-15 15:58:54 +0800
categories: jekyll
---

### 文档

- [官方文档][jekyll-docs]
- [minimal-mistakes主题文档][minimal-mistakes]

### 坑点
1. 本地安装后，_layout _sass assert _includes 文件不在本目录，需要通过
`bundle info minimal-mistakes-jekyll` 查找到, 
2. _config 配置 主题和皮肤。 github上的ruby 版本问题，导致需要降 _layout 文件复制过来，然后不能指定主题
3. 本地运行的时候报错  ` Invalid US-ASCII character "\xE2" on ` [issues], github上不需要配置，本地需要
```shell
export LC_ALL="C.UTF-8"
export LANG="en_US.UTF-8"
export LANGUAGE="en_US.UTF-8"
```

4. 本地运行命令：
```shell
bundle exec jekyll clean
bundle exec jekyll server --port 8080
```

5. 我这个主要的修改的地方，其他基本没加
- _pages/categories.md 自定义了一个分类的列表
- assert/css/main.scss 修改了文章和toc宽度

### 最后
好了，东西都配置好了，可以开始写东西了~


[jekyll-docs]: https://jekyllrb.com/docs/home
[minimal-mistakes]: https://mmistakes.github.io/minimal-mistakes/docs/quick-start-guide/
[issues]: https://github.com/jekyll/jekyll/issues/4268