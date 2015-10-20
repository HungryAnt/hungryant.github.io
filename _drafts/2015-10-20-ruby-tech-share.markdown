# ruby语言

## Ruby简介

### Ruby is...

- 动态编程语言
- 简单易用，开发效率
- 优雅的语法，易于阅读与编写

### Ruby的诞生

在20世纪90年代由日本人松本行弘(Yukihiro Matsumoto "Matz")开发

灵感与特性来自于 Perl、Smalltalk、Eiffel、Ada以及 Lisp 语言

> He has often said that he is “trying to make Ruby natural, not simple,” in a way that mirrors life.

> Ruby is simple in appearance, but is very complex inside, just like our human body.

###　开发者对Ruby的评价

开发者们认为Ruby是一个美丽精巧的语言，同时，也是个灵活实用的语言

对于Ruby有意思的评价

> 现在如果只使用Java，就感觉像拿着一根香蕉参加一场决斗，而我的对手挥舞着一把半人长的日本刀。

Ruby就是这把日本刀


## Ruby语法

###　官方首页面中的语法示例

Hello World

	# The famous Hello World
	# Program is trivial in
	# Ruby. Superfluous:
	#
	# * A "main" method
	# * Newline
	# * Semicolons
	#
	# Here is the Code:

	puts "Hello World!"

没有main函数，没有换行，没有`;`

puts实际是一个方法，方法调用时可以省略掉括号

	# Ruby knows what you
	# mean, even if you
	# want to do math on
	# an entire Array
	cities  = %w[ London
	              Oslo
	              Paris
	              Amsterdam
	              Berlin ]
	visited = %w[Berlin Oslo]

	puts "I still need " +
	     "to visit the " +
	     "following cities:",
	     cities - visited

数组执行算术运算符

	# Output "I love Ruby"
	say = "I love Ruby"
	puts say

	# Output "I *LOVE* RUBY"
	say['love'] = "*love*"
	puts say.upcase

	# Output "I *love* Ruby"
	# five times
	5.times { puts say }

直接字符串替换
完全面向对象


	# The Greeter class
	class Greeter
	  def initialize(name)
	    @name = name.capitalize
	  end

	  def salute
	    puts "Hello #{@name}!"
	  end
	end

	# Create a new object
	g = Greeter.new("world")

	# Output "Hello World!"
	g.salute


## 实例介绍，结合其他语言进行介绍

## Duck Typing

是动态类型的一种风格。在这种风格中，一个对象有效的语义，不是由继承自特定的类或实现特定的接口，而是由当前方法和属性的集合决定。

这点类似于其他脚本语言





> Duck typing常常被引用的一个批评是：它要求程序员在任何时候都必须很好地理解他/她正在编写的代码。

## 可变字符串类型

不同于Java Python等语言

ruby中字符串类型为可变类型

## Mixin

Ruby同Java类似，进支持单一继承，但可以使用Mixin机制

通用的功能定义为模板module，一个class可以包含多个module，module中定义的所有方法都可以在class中使用

## Ruby-game



