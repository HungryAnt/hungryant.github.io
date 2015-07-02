---
layout: post
title:  "sublime学习笔记"
date:   2015-06-30 23:27:04
categories: sublime
tags: [sublime, tools]
---

## 快捷键

打开命令窗口 `ctrl`+`shift`+`p`

开闭侧边栏 `ctrl`+`k` `ctrl`+`b`

关闭当前tab `ctrl`+`w`

查看当前文档格式 `ctrl`+`alt`+`shift`+`p`

Goto Anything `ctrl`+`p`

当前文件查找字符串 `ctrl`+`f`

当前文件替换字符串 `ctrl`+`h`

选中当前单词 `ctrl`+`d`，继续按`ctrl`+`d`选中下一个相同的单词

右键文件夹可以选择 Find in Folder... `f4`选中第一个匹配项，继续`f4`切换到下一项，`shift`+`f4`切换到上一项

跳回至前一个编辑点 `alt`+`-`，反向`alt`+`shift`+`-`

## PacakgeControl

The Sublime Text package manager that makes it exceedingly simple to find, install and keep packages up-to-date.

安装方式

[https://packagecontrol.io/installation](https://packagecontrol.io/installation)

## 工具包

安装工具包方式

`ctrl`+`shift`+`p` 输入 install，执行 Package Control 包的 Install Package命令

### AdvancedNewFile

`ctrl`+`alt`+`n`: General keymap to create new files.

`ctrl`+`shift`+`alt`+`n`: In addition to creating the folders specified, new folders will also contain an __init__.py file.

[For more information](https://github.com/skuroda/Sublime-AdvancedNewFile)

### Git

command中执行

	git add
	
	git commit (编写完log，执行`ctrl`+`w`跳出)
	
	git push

### SyncedSideBar

侧边栏同步显示当前文件

