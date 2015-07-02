---
layout: post
title:  "windows环境安装jekyll"
date:   2015-06-29 23:00:00
categories: ruby
tags: [jekyll]
---

步骤整理如下

1. 下载RubyInstaller

	http://rubyinstaller.org/downloads/

	Baidu网盘地址: http://pan.baidu.com/s/1bnjlCmz

	下载 `rubyinstaller-2.2.2-x64.exe`

2. 运行RubyInstaller，完成ruby安装

3. 设置系统环境变量

	假设安装至目录 `D:\Ruby22-x64`

	修改Path，添加 `D:\Ruby22-x64\bin;`

4. 设置gem源

	cmd中执行如下命令，设置taobao源

		gem sources --remove https://rubygems.org/
		gem sources -a http://ruby.taobao.org/
		gem sources -l

5. 安装DevKit

	部分ruby gem包由C/C++开发，DevKit在windows上提供一套编译环境

	网盘地址: http://pan.baidu.com/s/1kT9tKwB

	下载完成得到 `DevKit-mingw64-64-4.7.2-20130224-1432-sfx.exe`

	解压到`D:\DevKit`

	进入解压后的目录，执行命令`ruby dk.rb init`，将在此目录中生产config.yml，打开此文件，确认此文件末尾包含了ruby所在目录，如末尾包含

		---
		- D:/Ruby22-x64

	执行安装

		ruby dk.rb install

	> 参考 https://github.com/oneclick/rubyinstaller/wiki/Development-Kit

5. 安装jekyll

	使用gem下载安装jekyll及其所有依赖库

		gem install jekyll


PS

- 运行jekyll serve若报python错误
	
	注意 Python 3 is unsupported by the pygments.rb gem

	下载安装python2
	
	下载地址: http://pan.baidu.com/s/1nth5vCX

	

