---
layout: post
title:  "H2 database 对timestamp on update语法不支持"
date:   2015-06-30 00:00:00
categories: database
---

Mysql DDL

	DROP TABLE IF EXISTS `fund_records`;
	CREATE TABLE `fund_records` (
	  `id` bigint(20) NOT NULL AUTO_INCREMENT COMMENT '自增id',
	  `user_id` varchar(255) NOT NULL COMMENT '用户id',
	  `mtime` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '数据更新时间',
	  PRIMARY KEY (`id`)
	) ENGINE=InnoDB DEFAULT CHARSET=utf8 COMMENT='记录';


转换为 H2 database DDL时，ENGINE后的语句全部去除，同时还需要去除 timestamp列定义中 ON UPDATE CURRENT_TIMESTAMP语句，否则出错
DDL如下

	DROP TABLE IF EXISTS `fund_records`;
	CREATE TABLE `fund_records` (
	  `id` bigint(20) NOT NULL AUTO_INCREMENT COMMENT '自增id',
	  `user_id` varchar(255) NOT NULL COMMENT '用户id',
	  `mtime` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '数据更新时间',
	  PRIMARY KEY (`id`)
	) ENGINE=InnoDB DEFAULT CHARSET=utf8 COMMENT='记录';


参考

[H2 mode MySQL- ON UPDATE CURRENT_TIMESTAMP not supported](https://code.google.com/p/h2database/issues/detail?id=491)