---
layout: post
title:  "《Ruby元编程》读书笔记-方法"
date:   2015-10-21 00:00:00
categories: ruby
---

###静态类型检查

有些语言比如Java，对象间的每一次方法调用，编译器都会检查接受对象是否有一个匹配的方法，称之为静态类型检查(static type checking)，这类语言被称为静态语言(static language)

动态语言———比如 Python 和 Ruby，直到方法被真正执行，对象无法理解调用时才会报错

静态类型检查的好处：在代码运行前编译器就可以发现其中的一些错误
静态语言强迫你写很多误区和重复的代码

Ruby中，契约方法不再是问题


### 历史遗留系统 The Legacy System

程序需要加载的数据存储在一个历史遗留系统中

	class DS
	  def initialize # connect to data source ...
	  def get_mouse_info(workstation_id) ...
	  def get_mouse_price(workstation_id) ...
	  def get_keyboard_info(workstation_id) ...
	  def get_keyboard_price(workstation_id) ...
	  def get_cpu_info(workstation_id) ...
	  def get_cpu_price(workstation_id) ...
	  def get_display_info(workstation_id) ...
	  def get_display_price(workstation_id) ...
	  ...
	end

需要检测出计算机各配件的信息与价格，将价格高于99的设备信息打印出来

### 事不过三 Double, Treble...Trouble

抽象出一个类型Computer

	class Computer
	  def initialize(computer_id, data_source)
	    @id = computer_id
	    @data_source = data_source
	  end

	  def mouse
	    info = @data_source.get_mouse_info(@id)
	    price = @data_source.get_mouse_price(@id)
	    result = "Mouse: #{info} ($#{price})"
	    return " * #{result}" if price >= 100
	    result
	  end

	  def cpu
	    info = @data_source.get_cpu_info(@id)
	    price = @data_source.get_cpu_price(@id)
	    result = "Cpu: #{info} ($#{price})"
	    return " * #{result}" if price >= 100
	    result
	  end

	  def keyboard
	    info = @data_source.get_keyboard_info(@id)
	    price = @data_source.get_keyboard_price(@id)
	    result = "Keyboard: #{info} ($#{price})"
	    return " * #{result}" if price >= 100
	    result
	  end

	  ...
	end

写到这里，你发现自己陷入了不断拷贝、粘贴代码的泥潭。你不仅有一大堆方法要写，而且每个方法还要写单元测试，否则这些重复性代码就很容易出错。很快你就感到乏味了———更别提有多痛苦了。

我们怎么来重构它？

### 动态方法

那时我还是个年轻的开发者，正在学习C++，我的导师告诉我，当你调用一个方法时，实际上是给一个对象发送了一条消息。

	class MyClass
	  def my_mythod(my_arg)
	    my_arg * 2
	  end
	end

	obj = MyClass.new
	obj.my_method(3)  # => 6


使用Object#send()取代点标记符来调用MyClass#my_method()方法

	obj.send(:my_mehtod, 3)  # => 6

#### 来自Test::Unit的例子

Test::Unit标准库使用一个命名管理来判定哪些方法时测试方法

一个TestCase对象会查找自己的公开方法，并选择其中名字以test开头的方法:

	method_names = public_instance_methods(true)
	tests = method_names.delete_if {|mehtod_name| method_name !~ /^test./}

得到测试方法数组，使用send()方法来调用数组中的每个方法

#### 符号

符号与字符串没有关系，并且它们属于完全不同的类

不同的人有不同的答案。有些人会告诉你，符号和字符串的区别在于：符号是不可变的，而你可以修改字符串中的字符。另外，一些操作针对符号运行得更快些。

绝大多数情况下，符号用于表示事物的名字，尤其是跟元编程相关的名字，比如方法名

符号与字符串的转换

	# 字符串转符号
	String#to_sym() 或 String#intern()

	# 符号转字符串
	Symbol#to_s() 或 Symbol#id2name()

### 定义动态方法

Module#define_method()

	class MyClass
	  define_method :my_mythod do |my_arg|
	    my_arg * 2
	  end
	end

	obj = MyClass.new
	obj.my_method(3)  # => 6


### 重构

#### 第一步：添加动态派发

	class Computer
	  def initialize(computer_id, data_source)
	    @id = computer_id
	    @data_source = data_source
	  end

	  def mouse
	    component :mouse
	  end

	  def cpu
	    component :cpu
	  end

	  def keyboard
	    component :keyboard
	  end

	  def display
	    component :display
	  end

	  def component(name)
	    info = @data_source.send "get_#{name}_info", @id
	    price = @data_source.send "get_#{name}_price", @id
	    result = "#{name.to_s.capitalize}: #{info} ($#{price})"
	    return " * #{result}" if price >= 100
	    result
	  end
	end

#### 第二步：动态创建方法

	class Computer
	  def initialize(computer_id, data_source)
	    @id = computer_id
	    @data_source = data_source
	  end

	  def self.define_component(name)
	    define_method(name) do
	      info = @data_source.send "get_#{name}_info", @id
	      price = @data_source.send "get_#{name}_price", @id
	      result = "#{name.to_s.capitalize}: #{info} ($#{price})"
	      return " * #{result}" if price >= 100
	      result
	    end
	  end

	  define_component :mouse
	  define_component :cpu
	  define_component :keyboard
	  define_component :display
	end

#### 第三步：用内省(Introspection)方式缩减代码

	class Computer
	  def initialize(computer_id, data_source)
	    @id = computer_id
	    @data_source = data_source
	    data_source.methods.grep(/^get_(.+)_info$/) do
	      Computer.define_component $1
	    end
	  end

	  def self.define_component(name)
	    define_method(name) do
	      info = @data_source.send "get_#{name}_info", @id
	      price = @data_source.send "get_#{name}_price", @id
	      result = "#{name.to_s.capitalize}: #{info} ($#{price})"
	      return " * #{result}" if price >= 100
	      result
	    end
	  end
	end

### method_missing()方法

	class A
	end

	a = A.new
	a.hello

由于A类中不存在hello实例方法，报错如下

	in `<top (required)>': undefined method `hello' for #<A:0x0000000279f2a0> (NoMethodError)
		from -e:1:in `load'
		from -e:1:in `<main>'

沿着祖先链查找hello方法

A > Object > Kernel > BasicObject

找不到hello()方法，则调用method_missing()方法

覆写method_missing()方法

	class A
	  def method_missing(method, *args)
	    puts "You called: #{method}(#{args.join(', ')})"
	  end
	end

	a = A.new
	a.hello

输出

	You called: hello()

### 幽灵方法

#### 来自Ruport的例子

	require 'ruport'

	table = Ruport::Data::Table.new column_names: ['country', 'wine'],
	                                data: [['France', 'Bordeaux'],
	                                       ['Italy', 'Chianti'],
	                                       ['France', 'Chablis']]

	found = table.rows_with_country('France')
	found.each do |row|
	  puts row.to_csv
	end

输出

	France,Bordeaux
	France,Chablis

rows_with_country与to_csv是幽灵方法，Table中method_missing实现如下

	class Table
	  ...
	  def method_missing(id,*args,&block)
       return as($1.to_sym,*args,&block) if id.to_s =~ /^to_(.*)/ 
       return rows_with($1.to_sym => args[0]) if id.to_s =~ /^rows_with_(.*)/
       super
      end
      ...
    end

#### 来自OpenStruct的例子

