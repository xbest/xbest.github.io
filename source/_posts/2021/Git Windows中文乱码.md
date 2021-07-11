---
title: Git Windows中文乱码
date: 2021/07/12
categories:
- [Git]
- [常用工具]
  urlName: Git-Windows-ZH-CN-Messy-Code
---

```shell
$ git config --global core.quotepath false # 解决 git status 中文件路径的编码问题
$ git config --global gui.encoding utf-8 # 图形Git GUI界面编码
$ git config --global i18n.commit.encoding utf-8 # 提交信息编码
$ git config --global i18n.logoutputencoding utf-8 # 输出 log 编码
```