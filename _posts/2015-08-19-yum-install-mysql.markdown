---
layout: post
title:  "CentOS 安装mysql步骤"
date:   2015-8-19 23:36:02
categories: mysql
---

对mysql安装步骤做个记录

1. yum安装mysql

	yum install mysql-server.x86_64

2. 配置修改

	设置默认字符集

		vim /etc/my.cnf

	[mysqld]中添加 `default-character-set=utf8`

3. 启动mysql	
	
	执行如下命令

		chown -R mysql.mysql /var/lib/mysql  # 改变mysql主目录的所有者与所属者
		chkconfig mysqld on                  # 设置MySQL服务开机自启动
		chkconfig --list mysqld              # 查看服务状态是否正确 2-5 打开状态
		service mysqld start                 # 重启mysql服务

4. 连接数据库、密码修改

	连数据库，默认无密码

		mysql -u root

	修改密码

		set password for root@localhost=password('password');

		或者

		update user set password=password('password') where user='root';

	连数据库，执行密码

		mysql -u root -pxxxx