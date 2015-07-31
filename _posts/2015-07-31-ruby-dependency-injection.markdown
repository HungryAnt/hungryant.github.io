---
layout: post
title:  "20行ruby代码实现依赖注入框架"
date:   2015-7-31 23:36:56
categories: Ruby
---

## 我需要依赖注入

[业余时间开发的娱乐项目](/yecai/){:target="_blank"} (为了练习使用ruby语言)

遵循SRP原则，业务逻辑拆分由各个service类型提供，假设存在如下几个类型

- GameService 封装主要游戏业务逻辑
- UserService 维护用户信息
- PlayerService 封装当前玩家信息及业务逻辑
- ChatService 封装聊天业务逻辑

Service之间存在依赖关系，例如
PlayerService 依赖 UserService中的方法
PlayerService 依赖 ChatService中的方法
GameService 又依赖其他三个Service中的方法

最佳解决方案，实现依赖注入

## Ruby实现依赖注入 第一版 快速实现

	def init_dependency_injection
	  map = {}

	  Kernel.send :define_method, :get_instance do |clazz|
	    instance = map[clazz]
	    if instance.nil?
	      instance = clazz.new
	      map[clazz] = instance
	    end
	    instance
	  end
	end

init_dependency_injection方法在程序入口处调用

代码解析

初始化ioc容器 `map = {}`

向内核类型中添加扩展方法`get_instance`(所谓的`monkey hook`)，

get_instance方法接受参数clazz，表示一个类型，方法中创建该类型对象并添加对象值ico容器中

PlayerService中如何使用UserService，代码如下

	class PlayerService
	  def initialize
	    user_service = get_instance(UserService)
	    user_service.do_sth
	  end
	end


## Ruby实现依赖注入 第二版 改进

我觉得使用上还可以更方便些，类似spring mvc中通过注解进行依赖注入，向Kernal中补充autowired方法

	def init_dependency_injection
	  map = {}

	  Kernel.send :define_method, :get_instance do |clazz|
	    instance = map[clazz]
	    if instance.nil?
	      instance = clazz.new
	      map[clazz] = instance
	    end
	    instance
	  end

	  Kernel.send :define_method, :autowired do |*classes|
	    classes.each do |clazz|
	      underscore_class_name = clazz.name.to_s.gsub(/(.)([A-Z])/, '\1_\2').downcase
	      instance_variable_set("@#{underscore_class_name}", get_instance(clazz))
	    end
	  end
	end

使用方式如下

	class PlayerService
	  def initialize
	    autowired(UserService)
	  end

	  def x
	    @user_service.do_sth
	  end
	end

对于GameService依赖多个Services，同样简单，如下

	class GameService
	  def initialize
	    autowired(UserService, PlayerService, ChatService)
	  end
	  
	  def x
	    @user_service.do_sth
	    @player_service.do_sth
	    @chat_service.do_sth
	  end
	end


程序中任何类型中，都可以使用autowired方法注入所需类型对象，自动定义实例变量，便于使用

autowired方法代码解析

1. 获取类型集合，遍历之
2. 获取类名，由CamelCase转换为under_score，作为实例变量名
3. 通过`instance_variable_set`设置实例变量，值通过get_instance获取

perfect！