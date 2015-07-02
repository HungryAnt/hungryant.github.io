---
layout: post
title:  "ruby on rails 学习笔记"
date:   2015-07-01 00:00:00
categories: ruby
tags: [ruby, ruby on rails]
---

安装rails

	gem install rails

新建blog项目

	rails new blog

修改淘宝源

使用bundle下载依赖包

	bundle

安装sqlite3

	gem install sqlite3
	gem install sqlite3-ruby

	若执行rails server 提示sqlite错误，参考如下解决方案

	[cannot load such file — sqlite3/sqlite3_native (LoadError) on ruby on rails](http://stackoverflow.com/questions/17643897/cannot-load-such-file-sqlite3-sqlite3-native-loaderror-on-ruby-on-rails)
	

	> Find your sqlite3 gemspec file. One example is /usr/local/share/gem/specifications/sqlite3-1.3.7.gemspec or 'C:\Ruby21\lib\ruby\gems\2.1.0\specifications'. You should adjust according with your Rubygem path and sqlite3 version. Edit the file above and look for the following line

	> 	s.require_paths=["lib"]
	> change it to

	> 	s.require_paths= ["lib/sqlite3_native"]