---
layout: post
title:  "《Ruby元编程》读书笔记-对象模型"
date:   2015-10-24 00:00:00
categories: ruby
---

### 打开类

工具方法，去除字符串中的标点符号和特殊字符

```ruby
def to_alphanumeric(s)
  s.gsub /[^\w\s]/, ''
end
```

对应单元测试

```ruby
require 'minitest/autorun'

class AlphanumericTest < MiniTest::Test
  def test_strips_non_alphanumeric_characters
    assert_equal '3 the Magic Number', to_alphanumeric('#3, the *Magic, Number*?')
  end
end
```

能否将to_alphanumeric方法植入到String类型内部

打开String类型

```ruby
class String
  def to_alphanumeric
    gsub /[^\w\s]/, ''
  end
end
```

修改单测

```ruby
class StringExtensionsTest < MiniTest::Test
  def test_strips_non_alphanumeric_characters
    assert_equal '3 the Magic Number', '#3, the *Magic, Number*?'.to_alphanumeric
  end
end
```

书中示例适合使用此法。一般而言，在创建领域专属的方法时，如果会对Ruby的标准库造成“污染”，那么应该三思而后行。毕竟像String这样的类，本书已经有很多需要记住的方法了

### 类定义揭秘

可以在类定义中放置任何语句

```ruby
3.times do
  class C
    puts "Hello"
  end
end

# 输出
=> Hello
    Hello
    Hello
```

像执行其他代码一样，Ruby执行了这些在类中定义的代码

```ruby
class D
  def x; 'x'; end
end

class D
  def y; 'y'; end
end

obj = D.new
puts obj.x
puts obj.y
```

定义此遇到class D时，类还不存在，Ruby定义这个类，并定义x()方法。第二次渠道D类时，他已存在，Ruby不再定义它。而是重新打开已存在的类，并为之定义y()方法

你重视可以重新打开已经存在的类并对它进行动态修改，即使是想String或Array这样标准库中的类也不例外。这种技术，可以简单称之为**打开类**(Open Class)技术

### 对象中有什么

#### 实例变量

与java这类语言不同，Ruby中对象的类和它的实例变量没有关系，当给实例变量赋值时，它们才会生成。

```ruby
class MyClass
  def my_method
    @v = 1
  end
end

obj = MyClass.new

puts obj.instance_variables.display # => []

obj.my_method
puts obj.instance_variables.display # => [:@v]
```

对于同一个类，你可以创建具有不同实例变量的对象。可以将Ruby中实例变量的名字和值理解为哈希表中的键/值对，每一个对象的键/值对都有可能不同

#### 方法

可以通过Object#methos()方法获取对象的方法列表。绝大多数对象都从Object类中继承了一组方法，这个列表会很长

```ruby
puts (obj.methods.grep /^my/).display  # => [:my_method]
```

一个对象仅包含它的实例变量以及一个对自身类的引用

共享同一个类的对象也共享同样的方法，因此方法必须存放在类中

obj又一个叫my_method()的方法

MyClass有一个叫my_method()的实例方法

```ruby
puts (MyClass.instance_methods.grep /^my/).display  # => [:my_method]

puts obj.methods == MyClass.instance_methods  # => true
```

### 一切都是对象

不存在值类型，数字 1也是一个对象，类型为 Fixnum

类自身也是对象，和其他任何对象一样，也有自己的类型

irb

  1.class  # => Fixnum
  1.class.class  # => Class
  1.class.class.class  # => Class

同其他对象，类也有方法。一个对象的方法也是其类的实例方法。一个类的方法就是class的实例访法

  inherited = false
  Class.instance_methods inherited  # => [:allocate, :new, :superclass]

new()用来创建对象，allocate()方法是new()方法的支撑方法

所有类最终都继承自Object，Object继承自BasicObject，BasicObject是Ruby对象体系中的根节点

  Class.superclass  # => Module
  Module.superclass  # => Object

一个类只不过是一个增强的Module

#### Java中的类也是对象么

在Java和C#中，类也是名为Class的类的实例。C#甚至允许对已有的类添加新的方法

不过，在Java和C#中，类和对象有很大区别，类的功能也比较有限。例如，你不能再运行时创建一个类、修改一个类的方法等。从某种程度来说，Class对象更像是类的描述符，而非“真正的”类。类似于Java的File类不过是文件的描述符，而非真正的文件本身

#### 模块的优点是什么

同时拥有模块和类的主要原因在于清晰性

希望在别处被包含（include）时（或者当做**命名空间**时），应该选择使用模块；当希望被实例化或者继承时，应该选择使用类

### 常量

任何以大写字母开头的引用（包扩类名和模块名），都是常量

```ruby
module MyModule
  MyConstant = 'Outer constant'

  class MyClass
    MyConstant = 'Inner constant'
  end
end

MyModule::MyClass::MyConstant
```


### 方法查找

两个概念

接受者：调用方法所在的对象 例如： my_string.reverse()语句中，my_string就是接受者

祖先链：从一个类移动到他的超类，在移到超类的超类，知道达到BasicObject类。这个过程中所经历的类路径就是该类的祖先链

```ruby
class MyClass
  def my_method
    'my_method'
  end
end

class MySubclass < MyClass
end

obj = MySubClass.new
puts obj.my_method  # => my_method()
```

my_method方法调用，会沿着祖先链查找目标方法

查看祖先链

```ruby
MySubclass.ancestors.display  # => [MySubclass, MyClass, Object, Kernel, BasicObject]
```

#### 模块和方法查找

```ruby
module M
  def my_method
    'M#my_method'
  end
end

class C
  include M
end

class D < C; end

puts D.new.my_method  # => M#my_method
```

当在一个类中包含一个模块时，Ruby创建了一个封装该模块的匿名类，并将这个匿名类插入到祖先链中，其在链中的位置正好在类上方

```ruby
D.ancestors.display  # => [D, C, M, Object, Kernel, BasicObject]
```
