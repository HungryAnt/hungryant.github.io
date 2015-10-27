---
layout: post
title:  "《Ruby元编程》读书笔记-代码块"
date:   2015-10-25 00:00:00
categories: ruby
---

###Block

前面章节的内容跟普通的面向对象技术并没有太多差别。块则是继承自“函数式编程语言”的世界

代码块实例

	def a_method(a, b)
	  a + yield(a, b)
	end

	puts a_method(1, 2) {|x, y| (x + y) * 3}  # => 10

块定义可以放在大括号中（通常用于当行块），也可以放在do...end关键字中（通常用于多行的块）

可以通过#Kernal#block_given?()方法判断当前的方法是调用是否包含块

	def a_method
	  yield(a, b) if block_given?
	  'no block'
	end

### using关键字

来自C#中的非常漂亮的特性

设想你在写C#程序，它会连接一个远程服务器，并有一个对象表示这个连接

	RemoteConnection conn = new RemoteConnection("my_server");
	String stuff = conn.readStuff();
	conn.dispose(); // 关闭连接以避免内存泄露

这段代码在使用连接后，释放连接。然而，它并没有处理异常。如果readStuff()方法抛出异常，则conn对先过奖永远不会得到释放。

C#提供了一个叫做using的关键字，能够帮助处理整个过程

	using(RemoteConnection conn = new RemteConnection("my_server"))
	{
		String stuff = conn.readStuff();
		...
	}

尝试自己用实现一个Ruby版本的using

	require 'minitest/autorun'

	class TestUsing < MiniTest::Test
	  class Resource
	    def dispose
	      @disposed = true
	    end

	    def disposed?
	      @disposed
	    end
	  end

	  def test_disposes_of_resources
	    r = Resource.new
	    using(r) {}
	    assert r.disposed?
	  end

	  def test_disposes_of_resources_in_case_of_exception
	    r = Resource.new
	    assert_raises(Exception) {
	      using(r) {
	        raise Exception
	      }
	    }
	    assert r.disposed?
	  end
	end

实现如下

	def using(obj)
	  begin
	    yield if block_given?
	  ensure
	    obj.dispose
	  end
	end


### 闭包

当代码运行时，它需要一个执行环境：局部变量、实例变量、self...这些对象都有绑定在对象的名字上，简称绑定。块是完整的，可以立即运行，块既包含代码，也包含一组绑定

当块被定义时，他会获得当时环境中的绑定

	def my_method
	  x = 123
	  yield
	end

	x = 'Hello'
	my_method { puts x }

输出为 Hello

虽然在方法中定义了变量x，但块看到的x是在块定义时绑定的x，方法中的x对这个块来说是不可见的

这个特性成为闭包(closure)。 这意味着一个快可以获得局部绑定，并一直带着它们

### 作用域

作用域可见不同的绑定，局部变量、对象的方法与实例变量、常量树、全局变量

#### 切换作用域

作用域门

- 类定义
- 模块定义
- 方法

#### 扁平化作用域

怎样让绑定穿越一个作用域

	my_var = 'Success'

	class MyClass
	  # 你希望在这里打印my_var

	  def my_method
	    # ..还有这里
	  end
	end

作用域门是一道难以翻越的藩篱

使用Class.new替换class定义

	my_var = 'Success'

	MyClass = Class.new do
	  # 现在可以在这里打印my_var了
	  puts "#{my_var} in the class definition!"

	  def my_method
	    # ... 但是在这里怎样把它打印出来呢?
	  end
	end

使用Module#define_method()方法替代def

	my_var = 'Success'

	MyClass = Class.new do
	  # 现在可以在这里打印my_var了
	  puts "#{my_var} in the class definition!"

	  define_method :my_method do
	    puts "#{my_var} in the method!"
	  end
	end

	MyClass.new.my_method

=>
	
	Success in the class definition!
	Success in the method!

#### 共享作用域

假定你想在一组方法之间共享一个变量，但是又不希望其他方法也能访问这个变量，可以将这些方法定义在哪个变量所在的扁平作用域中

	def define_methods
	  share = 0

	  Kernel.send :define_method, :counter do
	    share
	  end

	  Kernel.send :define_method, :inc do |x|
	    share += x
	  end
	end

	define_methods

	puts counter
	inc 4
	puts counter


### instance_evel

以下程序演示了Object#instance_evel()方法，它在一个对象的上下文中执行一个快：

	class MyClass
	  def initialize
	    @v = 1
	  end
	end

	obj = MyClass.new
	obj.instance_eval do
	  puts self  # => #<MyClass:0x0000000285ec40>
	  puts @v  # => 1
	end

在运行时，改块的接受者会成为self，因此它可以访问接受者的私有方法和实例变量，例如@v。instance_eval()方法可以直接修改self对象，如下

	v = 2
	obj.instance_eval { @v = v }
	puts obj.instance_eval { @v }  # => 2

### 洁净室

有时，你会创建一个对象，仅仅是为了在其中执行块。这样的对象成为**洁净室(Clean Room)**

	class CleanRoom
	  def complex_calculation
	    # ...
	  end

	  def do_something
	    # ...
	  end
	end

	clean_room = CleanRoom.new
	clean_room.instance_eval do
	  if complex_calculation > 10
	    do_something
	  end
	end

### 可调用对象

从底层看，使用块需要分两步。第一步，将代码打包备用；第二步，调用块（通过yield语句）来执行代码。这种“先打包代码，以后调用”的机制并不是块的专利。在Ruby中，至少还有以下三种方法可以用来打包代码。

- 使用porc
- 使用lambda
- 使用方法

	a = Proc.new {puts 'Proc.new'}
	b = proc {puts 'proc'}
	c = lambda {puts 'lambda'}

	a.call
	b.call
	c.call

#### &操作符

&操作符可将块转换为Proc

	def my_method(&the_proc)
	  the_proc
	end

	p = my_method {|name| "Hello, #{name}!"}
	puts p.class  # => Proc
	puts p.call('Bill')  # => Hello, Bill!

&操作符又可以将Proc转换为块

	def my_method(greeting)
	  puts "#{greeting}, #{yield}!"
	end

	my_proc = proc { 'Bill' }
	my_method('Hello', &my_proc)

#### proc与lambda的对比

- 区别一 return

- 区别二 参数检验

在lambda中，return仅仅表示从这个lambda中返回

在proc中，return从定义proc的作用域中返回

### 领域专属语言

去除全局变量后的版本

	lambda {
	  setups = []
	  events = {}

	  Kernel.send :define_method, :event do |name, &block|
	    events[name] = block
	  end

	  Kernel.send :define_method, :setup do |&block|
	    setups << block
	  end

	  Kernel.send :define_method, :each_event do |&block|
	    events.each_pair do |name, event|
	      block.call name, event
	    end
	  end

	  Kernel.send :define_method, :each_setup do |&block|
	    setups.each do |setup|
	      block.call setup
	    end
	  end

	}.call

	Dir.glob('./events/*events.rb').each do |file|
	  load file
	  each_event do |name, event|
	    env = Object.new
	    each_setup do |setup|
	      env.instance_eval &setup
	    end
	    puts "ALERT: #{name}" if env.instance_eval &event
	  end
	end

by Ant

通过load file加载的代码，其根作用域为main对象（可以通过再file文件第一行puts self查看），因此event与setup两个方法必须定义为全局可访问的方法

案例中的each_event、each_setup两个方法意义并不大，可以去除，稍作调整如下

	lambda {
	  setups = []
	  events = {}

	  Kernel.send :define_method, :event do |name, &block|
	    events[name] = block
	  end

	  Kernel.send :define_method, :setup do |&block|
	    setups << block
	  end

	  Dir.glob('./events/*events.rb').each do |file|
	    load file
	    events.each_pair do |name, event|
	      env = Object.new
	      setups.each do |setup|
	        env.instance_eval &setup
	      end
	      puts "ALERT: #{name}" if env.instance_eval &event
	    end
	  end

	}.call

