---
layout: post
title:  "ruby实战-操作mysql数据库"
date:   2015-10-07 00:00:00
categories: ruby
---

## 开源库ruby-mysql

ruby-mysql库是一个纯ruby版本的MsSQL connector

开源项目地址

[https://github.com/tmtm/ruby-mysql](https://github.com/tmtm/ruby-mysql){:target="_blank"}

下载依赖包

	gem install ruby-mysql

项目中引入

	require 'mysql'

## ruby实现数据库连接池

基于ruby-mysql，使用ruby开发一个简单的数据库连接池易如反掌，这里给出一个实现方案，应用于本人的游戏服务器端

- 初始时创建n个连接加入缓存
- 每次获取连接时，已当前线程hash值模n从缓存获取连接
- 先尝试执行`select 1`，验证连接状态，若已经断开连接，则重新建立连接，更新缓存
- 建立链接时，设置相关`timeout`参数，避免连接严重超时影响服务运行效率

代码如下

	class DbConnectionPool
	  POOL_SIZE = 10

	  def initialize
	    @pool = []
	    0.upto(POOL_SIZE - 1) do
	      @pool << mysql_connect
	    end
	  end

	  def get_conn
	    num = Thread.current.hash % POOL_SIZE
	    conn = @pool[num]
	    begin
	      conn.query('select 1')
	    rescue Exception => e
	      LogUtil.error 'DbConnectionPool get_conn:'
	      LogUtil.error e.backtrace.inspect
	      conn = mysql_connect
	      @pool[num] = conn
	    end
	    conn
	  end

	  private
	  def mysql_connect
	    Mysql.connect(DatabaseConfig::HOST, 'root', 'password', 'dbname', 3306,
	                  Mysql::OPT_CONNECT_TIMEOUT=>1000,
	                  Mysql::OPT_READ_TIMEOUT=>1000,
	                  Mysql::OPT_WRITE_TIMEOUT=>1000)
	  end
	end


## 实际案例

这里展示一个操作用户数据的案例

用户数据表

	CREATE TABLE `v1_users` (
	  `id` bigint(20) NOT NULL AUTO_INCREMENT,
	  `user_id` varchar(64) NOT NULL,
	  `user_name` varchar(64) NOT NULL DEFAULT '',
	  `lv` int(11) NOT NULL DEFAULT '1',
	  `exp` int(11) NOT NULL DEFAULT '0',
	  `mtime` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
	  PRIMARY KEY (`id`),
	  UNIQUE KEY `idx_user_id` (`user_id`)
	) ENGINE=InnoDB DEFAULT CHARSET=utf8;

每一个用户数据为该表一条记录，程序逻辑需要实现用户创建，用户查询操作，确保易维护性，数据库操作封装到一个独立的Dao类`UserDataDao`中

	class UserDataDao
	  def initialize
	    autowired(DbConnectionPool)
	  end

	  def get_my
	    @db_connection_pool.get_conn
	  end
	  ...

构造器中完成对数据库连接池对象的注入，关于使用`autowaired`方法实现依赖注入请参考我的另一篇博文[20行ruby代码实现依赖注入框架](/ruby/2015/07/31/ruby-dependency-injection.html){:target="_blank"}

如下展示账号数据表操作相关方法实现

	  def create_user(user_id, user_name)
	    stmt = get_my.prepare('insert into v1_users(user_id, user_name) values(?, ?)')
	    stmt.execute user_id, user_name
	  end

	  def has_user?(user_id)
	    users = get_my.prepare('select user_id from v1_users where user_id = ?').execute(user_id).fetch
	    !users.nil? && users.size == 1
	  end

	  def update_user_name(user_id, user_name)
	    get_my.prepare('update v1_users set user_name = ? where user_id = ?')
	        .execute(user_name, user_id)
	  end

	  def get_user_lv(user_id)
	    r = get_my.prepare('select lv, exp from v1_users where user_id = ?').execute(user_id).fetch
	    return r[0].to_i, r[1].to_i
	  end

