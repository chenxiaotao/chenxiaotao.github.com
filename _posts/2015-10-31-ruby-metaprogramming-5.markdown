---
layout:     post
title:      《Ruby元编程》 读书笔记5
subtitle:   "元编程终章"
date:       2015-10-31
author:     "Sheldon"
header-img: "img/pic/post/ruby-metaprogramming.jpg"
tags:       
  - ruby
---

#### Kernal#eval方法
`Kernal#eval`方法与之前的`BasicObject#instance_eval`和`Module#class_eval`一样，都属于eval家族，
都可以赋予程序在运行中进行动态变化的能力。与后两者想比Kernal#eval更加直接，不需要代码块、
直接就可以执行字符串代码(String of Code)

PS:`BasicObject#instance_eval`也是可以执行字符串代码的。

~~~ruby
array = [10, 20]
element = 30
eval("array << element")  #=> [10, 20, 30]
array.instance_eval "self << element"  #=> [10, 20, 30]
~~~

#### Here文档(Here documents)
以`<<`打头，后面跟一个 `结束序列标识`，之后就可以是正式的文档内容了，可以任意换行，直到遇到了独立出现的 结束序号标识

~~~ruby
puts <<GROCERY_LIST
Grocery list
------------
1. Salad mix.
2. Strawberries.*
3. Cereal.
4. Milk.*

* Organic
GROCERY_LIST

# 上面代码的输出格式如下:
Grocery list
------------
1. Salad mix.
2. Strawberries.*
3. Cereal.
4. Milk.*

* Organic
=> nil
~~~

#### 绑定对象
Binding是一个用对象标识的完整作用域(上下文环境)。可以通过创建Binding对象来捕获并带走当前的作用域。
之后，通过eval方法在这个Binding对象所携带的作用域内执行代码

~~~ruby
# 使用Kernel#binding方法可以用来创建Binding对象
class MyClass
  def my_method
    @x = 1
    binding
  end
end

b = MyClass.new.my_method
eval "@x", b #=> 1
~~~

上面代码，在MyClass类中定义了一个my_method方法来返回一个当前的绑定。
最后将这个返回的绑定，作为参数传递给eval方法。这样“@x” 就可以在返回的绑定作用域中执行了

关于绑定还有另外一个知识点，Ruby还提供了一个名为TOPLEVEL_BINDING的预定义常量，表示顶级作用域Binding对象。
该常量可以在程序的任何位置访问到。言外之意，你可以在程序的任何位置，通过Kernal#eval方法在顶级作用域中执行代码

~~~ruby
class AnotherClass
  def my_method
    eval "self", TOPLEVEL_BINDING
  end
end

AnotherClass.new.my_method  #=> main
~~~

#### 代码注入
Kernal#eval执行这样可以灵活执行字符串代码的特性，给编程带来了灵活性之外，
也带来了潜在的风险，如果字符串代码来源于不可信的用户输入，如果不做安全检查，
保不齐什么时候就会是一段破坏性的恶意代码

面对这样的风险，可以选择规避eval的使用，换用其它相对安全的方式代替，例如动态方法和动态派发。
此外，还可以通过在Ruby中通过修改$SAFE全局变量值，来控制程序的安全性级别，
具体就是在你要执行可信的字符串代码前，将安全级别降低，可以使用`Object#untaint`方法，
执行完之后在切换安全级别。这有点像操作系统中使用临界资源的步骤(请求锁，释放锁)
通过`Object#tainted?`方法可以判断一个对象是不是被污染了（是否来自一个不可信的输入源）

#### 钩子方法
钩子方法有些类似事件驱动装置，可以在特定的事件发生后执行特定的回调函数，
这个回调函数就是钩子方法(更形象的描述: 钩子方法可以像钩子一样，勾住一个特定的事件)
在Rails中before\after函数就是最常见的钩子方法

`Class#inherited`方法也是这样一个钩子方法，当一个类被继承时，Ruby会调用该方法。
默认情况下，`Class#inherited`什么都不做，但是通过继承，我们可以拦截该事件，对感兴趣的继承事件作出回应

~~~ruby
class String
  def self.inherited(subclass)
    puts "#{self} was inherited by #{subclass}"
  end
end
class MyString < String; end
# 输出
String was inherited by MyString
~~~

通过使用钩子方法，可以让我们在Ruby的类或模块的生命周期中进行干预，可以极大的提高编程的灵活性。
这些生命周期相关的钩子方法还有下面这些:

**类与模块相关**

* Class#inherited
* Module#include
* Module#prepended
* Module#extend_object
* Module#method_added
* Module#method_removed
* Module#method_undefined

**单件类相关**

* BasicObject#singleton_method_added
* BasicObject#singleton_method_removed
* BasicObject#singleton_method_undefined

~~~ruby
module M1
  def self.included(othermod)
    puts "M1 was included into #{othermod}"
  end
end

module M2
  def self.prepended(othermod)
    puts "M2 was prepended to #{othermod}"
  end
end

class C
  include M1
  prepend M2
end

# 输出
M1 was included into C
M2 was prepended to C

module M
  def self.method_added(method)
    puts "New method: M##{method}"
  end

  def my_method; end
end

# 输出
New method: M#my_method
~~~

除了上面列出来的一些方法外，也可以通过重写父类的某个方法，进行一些过滤操作后，
再通过调用super方法完成原函数的功能，从而实现类似钩子方法的功效，
如出一辙，**环绕别名**也可以作为一种钩子方法的替代实现


#### 书中示例
书中以迭代的形式，引导读者一步步实现一个名为attr_checked的类宏

任务：写一个操作方法类似attr_accessor的attr_checked的类宏，该类宏用来对属性值做检验，使用方法如下:

~~~ruby
class Person
  include CheckedAttributes

  attr_checked :age do |v|
    v >= 18
  end
end

me = Person.new
me.age = 39  #ok
me.age = 12  #抛出异常
~~~

实施计划
  1. 使用eval方法编写一个名为add_checked_attribute的内核方法，为指定类添加经过简单校验的属性
  2. 重构add_checked_attribute方法，去掉eval方法，改用其它手段实现
  3. 添加代码块校验功能
  4. 修改add_checked_attribute为要求的attr_checked,并使其对所有类都可用
  5. 通过引入模块的方式，只对引入该功能模块的类添加attr_checked方法

~~~ruby
# 第一步
def add_checked_attribute(klass, attribute)
  eval "
    class #{klass}
      def #{attribute}=(value)
        raise 'Invalid attribute' unless value
        @#{attribute} = value
      end
      def #{attribute}()
        @#{attribute}
      end
    end
  "
end

add_checked_attribute(String, :my_attr)
t = "hello,kitty"

t.my_attr = 100
puts t.my_attr

t.my_attr = false
puts t.my_attr
~~~

使用eval方法，用class和def关键词分别打开类，且定义了指定的属性的get和set方法，
其中的set方法会简单的判断值是否为空(nil 或 false)，如果是则抛出Invalid attribute异常

~~~ruby
# 第二步
def add_checked_attribute(klass, attribute)
  klass.class_eval do
    define_method "#{attribute}=" do |value|
      raise "Invaild attribute" unless value
      instance_variable_set("@#{attribute}", value)
    end

    define_method attribute do
      instance_variable_get "@#{attribute}"
    end
  end
end
~~~

更换掉了eval方法,同时也分别用class_eval和define_method方法替换了之前的class与def关键字，
实例变量的设置和获取分别改用了instance_variable_set和instance_variable_get方法，
使用上与第一步没有任何区别，只是一些内部实现的差异

~~~ruby
# 第三步
def add_checked_attribute(klass, attribute, &validation)
  klass.class_eval do
    define_method "#{attribute}=" do |value|
      raise "Invaild attribute" unless validation.call(value)
      instance_variable_set("@#{attribute}", value)
    end

    define_method attribute do
      instance_variable_get "@#{attribute}"
    end
  end
end

add_checked_attribute(String, :my_attr){|v| v >= 180 }
t = "hello,kitty"

t.my_attr = 100  #Invaild attribute (RuntimeError)
puts t.my_attr

t.my_attr = 200
puts t.my_attr  #200
~~~

通过代码块验证，增加了校验的灵活性，不再仅仅局限于nil和false之间

~~~ruby
# 第四步
class Class
  def attr_checked(attribute, &validation)
    define_method "#{attribute}=" do |value|
      raise "Invaild attribute" unless validation.call(value)
      instance_variable_set("@#{attribute}", value)
    end

    define_method attribute do
      instance_variable_get "@#{attribute}"
    end
  end
end

String.add_checked(:my_attr){|v| v >= 180 }
t = "hello,kitty"

t.my_attr = 100  #Invaild attribute (RuntimeError)
puts t.my_attr

t.my_attr = 200
puts t.my_attr  #200
~~~

这里我们把之前顶级作用域中方法名放到了Class中，由于所有对象都是Class的实例, 所以这里定义的实例方法，
也能被Ruby中的其它所有类访问到，同时在class定义中,self就是当前类，
所以也就省去了调用类这个参数和class_eval方法，并且我们把方法的名字也改成了attr_checked

~~~ruby
# 第五步
module CheckedAttributes
  def self.included(base)
    base.extend ClassMethods
  end
end

module ClassMethods
  def attr_checked(attribute, &validation)
    define_method "#{attribute}=" do |value|
      raise "Invaild attribute" unless validation.call(value)
      instance_variable_set("@#{attribute}", value)
    end

    define_method attribute do
      instance_variable_get "@#{attribute}"
    end
  end
end

class Person
  include CheckedAttributes

  attr_checked :age do |v|
    v >= 18
  end
end
~~~

通过钩子方法，在CheckedAttributes模块被引入后,对当前类通过被引入模块进行扩展，
从而使当前类支持引入后的方法调用，即这里的get与set方法组

我们已经得到了一个名为attr_checked，类似attr_accessor的类宏，通过它你可以对属性进行你想要的校验
