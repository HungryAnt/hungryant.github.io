---
layout: post
title:  "ruby实战-mysql数据库共享连接池实现"
date:   2015-11-26 15:00:00
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

本次实现是在第一个版本的基础上进行了升级，ruby-mysql提供的api，本身不保证线程安全性,，原版本在多线程同时进行数据库操作时存安全隐患

[第一版 ruby实战-操作mysql数据库](/ruby/2015/10/07/ruby-mysql-in-action.html){:target="_blank"}

升级版连接池操作逻辑如下

1. 初始创建一批连接，加入队列
2. 数据库操作前从队列中取得一个连接
3. 连接数达上限后稍作等待，超时则失败抛出异常
4. 获取连接后测试连接有效性，失效则重新获取连接
5. 数据库操作完成后立即归还连接至队列中

技术点说明

- 线程池对外仅暴露`execute`方法
- 数据库操作全部封装在代码块中执行

源码如下

	class DbConnectionPool
	  INIT_POOL_SIZE = 20

	  def initialize
	    @pool = []
	    0.upto(INIT_POOL_SIZE - 1) do
	      @pool << mysql_connect
	    end
	    @mutex = Mutex.new
	    @actual_pool_size = INIT_POOL_SIZE
	  end

	  def execute
	    return unless block_given?
	    begin
	      conn = get_conn
	      test conn
	      yield conn
	      add_to_pool conn
	    rescue Exception => e
	      @mutex.synchronize {
	        @actual_pool_size -= 1
	      }
	      conn.close
	      LogUtil.error 'DbConnectionPool execute'
	      LogUtil.error e.backtrace.inspect
	      raise e
	    end
	    LogUtil.info "actual_pool_size: #{@actual_pool_size}"
	  end

	  private
	  def mysql_connect
	    Mysql.connect(DatabaseConfig::HOST, 'root', 'ant', 'yecai', 3306,
	                  Mysql::OPT_CONNECT_TIMEOUT=>1000,
	                  Mysql::OPT_READ_TIMEOUT=>1000,
	                  Mysql::OPT_WRITE_TIMEOUT=>1000)
	  end

	  def get_conn
	    Timeout.timeout(3) do
	      while @actual_pool_size >= 30
	        sleep 2
	      end
	    end

	    @mutex.synchronize {
	      if @pool.size > 0
	        return @pool.pop
	      else
	        @actual_pool_size += 1
	        return mysql_connect
	      end
	    }
	  end

	  def add_to_pool(conn)
	    @mutex.synchronize {
	      @pool.push conn
	    }
	  end

	  def test(conn)
	    begin
	      conn.query('select 1')
	    rescue Exception => e
	      LogUtil.error 'reconnect'
	      conn.close
	      conn = mysql_connect
	    end
	    conn
	  end
	end

### 使用方式

通常创建一个封装数据库操作的基类 `DaoBase`

	class DaoBase
	  def initialize
	    autowired(DbConnectionPool)
	  end

	  def execute(&action)
	    @db_connection_pool.execute &action
	  end
	end

通过依赖注入的方式引入`DbConnectionPool`实例，提供方法`execute`

构造器中完成对数据库连接池对象的注入，关于使用`autowaired`方法实现依赖注入请参考我的另一篇博文[20行ruby代码实现依赖注入框架](/ruby/2015/07/31/ruby-dependency-injection.html){:target="_blank"}

### 上层Dao实现举例

**用户数据表**

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

每一个用户数据为该表一条记录，程序逻辑需要实现用户创建，用户查询操作，确保易维护性，数据库操作封装到一个独立的Dao类`UserDataDao`中，实现如下

	class UserDataDao < DaoBase
	  def create_user(user_id, user_name)
	    execute do |conn|
	      stmt = conn.prepare('insert into v1_users(user_id, user_name) values(?, ?)')
	      stmt.execute user_id, user_name
	    end
	  end

	  def has_user?(user_id)
	    users = nil
	    execute do |conn|
	      users = conn.prepare('select user_id from v1_users where user_id = ?').execute(user_id).fetch
	    end
	    !users.nil? && users.size == 1
	  end

	  def update_user_name(user_id, user_name)
	    execute do |conn|
	      conn.prepare('update v1_users set user_name = ? where user_id = ?')
	          .execute(user_name, user_id)
	    end
	  end

	  def get_user_lv(user_id)
	    r = nil
	    execute do |conn|
	      r = conn.prepare('select lv, exp from v1_users where user_id = ?').execute(user_id).fetch
	    end
	    return r[0].to_i, r[1].to_i
	  end
	end