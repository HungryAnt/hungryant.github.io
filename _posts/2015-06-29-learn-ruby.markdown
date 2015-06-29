---
layout: post
title:  "Ruby 学习"
date:   2015-06-29 20:29:15
categories: ruby
---

### Hello World

基本输出

	puts "Hello Wold"

参数

	language = 'Ruby'
	puts "hello, #{language}"

`""`中字符串支持替换参数、支持`\`转义字符，`''`表示原始字符串

### 基本语句 判断 循环
	
**判断语句**

	x = 4
	puts 'This appears to be false.' unless x == 4
	puts 'This appears to be true.' if x == 4
	
	if x == 4
		puts 'This appears to be true.'
	end
	
	unless x == 4
		puts 'This appears to be false.'
	end

**循环语句**

	x = 0
	x = x + 1 while x < 10
	
	x = x - 1 until x == 1

	while x < 10
		x = x + 1
		puts x
	end

处理`nil`和`false`之外，其他值都代表`true`

逻辑运算符 `and && or || not !`

**鸭子类型**

	a = ['100', 100.0]
	i = 0
	while i < 2
		puts a[i].to_i
		i = i + 1
	end

**定义函数**

	def tell_the_truth
		true
	end

默认返回值为最后处理的表达式的值

**数组**

	animals = ['lions', 'tigers', 'bears']
	animals[0]  # 取数组元素
	animals[10]  # 返回nil
	animals[-1]  # 返回最后一个元素
	animals[-2]  # 返回倒数第二个元素
	animals[0..1]  # 0..1是个Range（区间）对象，表示从0到1（包括0和1）的所有数字

**散列表**

	numbers = {1 => 'one', 2 => 'tow'}
	stuff = {:array => [1, 2, 3], :string => 'Hi, mom!'}

符号是前面带有冒号的标识符，类似`:symbol`的形式。相同的服务是同一个物理对象

	'string'.object_id == 'string'.object_id  # false
	:string.object_id == :string.object_id  # true

使用散列表模拟命名参数

	def tell_the_truth(options = {})
		if options[:profession] == :lawyer
			true
		else
			false
		end
	end

	tell_the_truth  # false
	tell_the_truth :profession => :lawyer  # true

**代码块和yield**

	3.times {puts 'hello!'}

	animals = ['lions and ', 'tigers and ', 'bears', 'oh my']
	animals.each {|a| puts a}

自定义实现的times方法

	class Fixnum
		def my_times
			i = self
			while i > 0
				i = i - 1
				yield
			end
		end
	end

	3.my_times {puts 'ant'}

代码块作为函数参数传递

参数名之前加一个"&"，表示将代码块作为闭包传递给函数

	def call_block(&block)
		block.call
	end

	def pass_block(&block)
		call_block(&block)
	end

	pass_block {puts 'Hello, block'}

**类型**

	4.class  # Fixnum
	4.class.superclass  # Integer
	4.class.superclass.superclass  # Numeric
	4.class.superclass.superclass.superclass  # Object
	4.class.superclass.superclass.superclass.superclass  # BasicObject

	4.class.class  # Class
	4.class.class.superclass  # Module
	4.class.class.superclass.superclass  # Object

Ruby中的一切事物都有一个共同祖先——Object

自定义类型

	class Tree
		attr_accessor :children, :node_name
		
		def initialize(name, children = [])
			@children = children
			@node_name = name
		end
		
		def visit_all(&block)
			visit &block
			children.each {|c| c.visit_all &block}
		end
		
		def visit(&block)
			block.call self
		end
	end
	
	ruby_tree = Tree.new('Ruby', [Tree.new('Reia'), Tree.new('MacRuby')])
	
	puts 'Visiting a nod'
	ruby_tree.visit {|node| puts node.node_name}
	puts
	
	puts 'Visiting entire tree'
	ruby_tree.visit_all {|node| puts node.node_name}

类首字母一般大写，CamelCase

实例变量前必须加上`@`，类变量前必须加上`@@`

实例变量和方法名以小写字母开头，并采用下划线命名法，如underscore_style

常量采用全达到写形式，如ALL_CAPS

attr定义实例变量和访问变量的同名方法

attr_accessor定义实例变量、访问方法和设置方法


**Minxin**

	module ToFile
		def filename
			"object_#{self.object_id}.txt"
		end
		
		def to_f
			File.open(filename, 'w') {|f| f.write(to_s)}
		end
	end
	
	class Person
		include ToFile
		attr_accessor :name
		
		def initialize(name)
			@name = name
		end
		
		def to_s
			return @name
		end
	end
	
	Person.new('matz').to_f

**比较**

`<=>`被人们叫做太空船操作，同其他语言中的cmp

	100 <=> 200  # -1
	100 <=> 100  # 0
	200 <=> 100  # 1
	'AAA' <=> 'BBB'  # -1
	[1, 2, 3] <=> [1, 2, 3]  # 0

**集合操作**

	a = [5, 3, 4, 1]
	a.sort  # 返回新的数组 [5, 3, 4, 1]
	
	a.any? {|i| i > 6}  # false
	a.any? {|i| i > 4}  # true
	a.all? {|i| i > 4}  # false
	a.collect {|i| i * 2}  # [10, 6, 8, 2]
	a.select {|i| i % 2 == 0}  # [4]
	a.max  # 5
	a.member?(2)  # false
	a.find {|i| i < 5}  # 找到一个符合条件的元素，未找到则返回nil
	a.find_all {|i| i < 5}  # 同select

使用inject方法计算列表的和与积
	
	a = [5, 3, 4, 1]
	a.inject(100) {|sum, i| sum + i}  # 113
	a.inject {|sum, i| sum + i}  # 13
	a.inject {|product, i| product * i}  # 60

	a.inject do |sum, i|
		puts "sum: #{sum}  i: #{i}  sum + i: #{sum + i}"
		sum + i 
	end

**开放类**

来自Rails框架中的一个例子，它为NilClass、String各添加了一个方法

	class NilClass
		def blank?
			true
		end
	end
	
	class String
		def blank?
			self.size == 0
		end
	end
	
	["", "person", nil].each do |element|
		puts element unless element.blank?
	end


