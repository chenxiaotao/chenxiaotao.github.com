---
layout:     post
title:      《Ruby元编程》 读书笔记2
subtitle:   "Ruby的方法"
date:       2015-09-26
author:     "Sheldon"
header-img: "img/pic/post/ruby-metaprogramming.jpg"
tags:       
  - ruby
  - 元编程
  - 面向对象
  - 读书笔记
---
#### 动态方法

##### 动态调用方法
在Ruby中通过`Object#send`方法可以代替点标识调用对象的指定实例方法

~~~ruby
class MyClass
  def my_method(my_arg)
    my_arg * 2
  end
end

obj = MyClass.new
obj.my_method(3)        #=> 6
obj.send(:my_method, 3) #=> 6
~~~

上面代码通过直接调用和使用send方法调用得到的结果是一样的，使用send的好处是可以在编码中动态的决定方法调用。
这个技巧在元编程中被称为**动态派发**

另外需要指出的地方是通过`Object#send`不仅可以调用公共方法，也可以调用对象的私有方法。
如果想保留对象的封装特性，不向外暴露私有方法可以使用`Object#public_send`方法
(不过一般情况下使用send就是为了调用私有方法)

##### 动态定义方法
除了方法的动态调用之外，Ruby还通过`Module#define_method`方法和代码块提供了动态方法定义方式

~~~ruby
class MyClass
  define_method :my_method do |my_arg|
    my_arg * 3
  do
end

obj = MyClass.new
obj.my_method(2)  #=> 6
~~~

上面代码通过`define_method`方法取代了关键词def，其本质上都是相同的，只是在定义方式上，
`Module#define_method`的方式更加灵活一些，可以通过在编码中通过推导，完成函数的定义，增加了实现的灵活性

##### 动态方法查找
由于使用了动态定义方法，很多同学喜欢在项目里使用grep搜索，或者是vim的跳转是找不到方法定义的。这里可以使用
一个`source_location`的方法查看方法定义的文件以及行数，使用方式如：
实例方法：`obj.method(:method_name).source_location`，类方法：`class.method(:method_name).source_location`

#### method_missing方法
method_missing利用的机制是，当一个对象进行某个方法调用的时候，会到其对应的类的实例方法中进行查找，
如果没有找到，则顺着祖先链向上查找，直到找到BasicObject类为止。
如果都没有则会最终调用一个`BasicObject#method_missing`抛出NoMethodError异常

严格意义上将method_missing方法，并不算是明确的定义(不会出现在methods列表中)，
其本质是通过方法查找的机制来截获调用信息进而合理的给出相应方法的回应。
有点类似与异常处理中的抛出异常，一层一层的往外抛。

当我们需要定义很多相似的方法时候，可以通过重写method_missing方法，对相似的方法进行统一做出回应，
这样一来其行为就类似与调用定义过的方法一样。

~~~ruby
class Roulette
  def method_missing(name, *args)
    person = name.to_s.capitalize
    super unless %w[Bob Frank Bill Honda Eric].include? person
    number = 0
    3.times do
      number = rand(10) + 1
      puts "#{number}..."
    end
    "#{person} got a #{number}"
  end
end

number_of = Roulette.new
puts number_of.bob
puts number_of.kitty
~~~

#### 动态代理
对一些封装过的对象，通过method_missing方法收集调用，并把这些调用转发到被封装的对象，这一过程称为**动态代理**，
其中method_missing体现了动态，转发体现了代理

#### const_missing方法
与method_missing类似，还有关于常量的const_missing方法，当引用一个不存在的常量时，Ruby会把这个常量名作为一个符号传递给const_missing方法。

#### 白板类(blank slates)
拥有极少方法的类称为白板类，通过继承BasicObject类，可以迅速的得到一个白板类。除了这种方法以外，还可以通过删除方法来将一个普通类变为白板类。

#### 删除方法
* `Module#undef_method` 会删除所有(包括继承而来的)方法
* `Module#remove_method` 只删除接受者自己的方法，而保留继承来的方法

#### 动态方法与Method_missing的使用原则
当可以使用动态方法时候，尽量使用动态方法。除非必须使用method_missing方法(方法特别多的情况)，否则尽量少使用它。

