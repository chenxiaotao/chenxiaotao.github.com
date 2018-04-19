---
layout:     post
title:      《Ruby元编程》 读书笔记4
subtitle:   "类定义"
date:       2015-10-19
author:     "Sheldon"
header-img: "img/pic/post/ruby-metaprogramming.jpg"
tags:       
  - ruby
---

#### 类的返回值
像方法一样，类定义也会返回最后一条语句的值:

~~~ruby
result = class MyClass
  self
end
result #=> MyClass
~~~

#### 当前类
与当前对象self一样，同时还存在当前类（或模块）
Ruby中并没有类似当前对象self一样的明确引用，不过在追踪当前类的时候，可以遵循下面几条:

* 在程序顶层，当前类是Object，这是main对象所属的类
* 在一个方法中，当前类就是当前对象的类。（在一个方法中用def关键字定义另一个方法，
新定义的方法会定义在self所属的类中）

  ~~~ruby
  class C
    def m1
      def m2; end
    end
  end

  class D < C; end

  obj = D.new
  obj.m1

  C.instance_methods(false)   #=> [:m1, :m2]
  ~~~

* 当用class关键字打开一个类时（或module关键字打开模块），那个类称为当前类

#### class_eval方法
* `Module#class_eval`（别名是`module_eval`），会在一个已存在的类的上下文中执行一个块。使用该方法可以在不需要class关键字的前提下，打开类
* `Module#class_eval`与`Object#instance_eval`方法相比，`instance_eval`方法只能修改self，而`class_eval`方法可以同时修改self与当前类
* `class_eval`的另一个优势就是可以利用扁平作用域，规避class关键字的作用域门问题
* `instance_eval`与`class_eval`的选择：
通常使用`instance_eval`方法打开非类的对象，而用`class_eval`方法打开类定义，然后用def定义方法


#### 单件方法
Ruby允许给单个对象增加方法，这种只针对单个对象生效的方法，称为单件方法

~~~ruby
str = "just a regular string"

def str.title?
  self.upcase == self
end

str.title?  # => false
str.methods.grep(/title?/) # => [:title?]
str.singleton_methods   #=> [:title?]

str.class # => String
String.title?  #=>  NoMethodError
~~~

另外，除了上面使用的定义方法，还可以通过`Object#define_singleton_method`方法来定义单件方法

#### 单件方法与类方法
* Ruby中类也是对象，而类名只是常量，所以，在类上调用方法其实跟在对象上调用方法一样
* 类方法的实质是: 它是一个类的单件方法，实际上，如果比较单件方法的定义和类方法的定义，会发现其实二者是一样的

~~~ruby
def obj.a_singleton_method; end
def MyClass.another_class_method; end

#二者均使用了def关键词做定义
def object.method
  #方法主体
end
~~~

上面的object可以是对象的引用、常量类名或者self。

#### 类宏
Ruby对象没有属性，如果希望得到一些像属性的东西，需要分别定义一个读方法和写方法（也就是java中的set和get方法)，最直接的可以这样:

~~~ruby
class MyClass
  def my_attribute=(value)
    @my_attribute = value
  end

  def my_attribute
    @my_attribute
  end
end

obj = MyClass.new
obj.my_attribute = 'x'
obj.my_attribute    #=> ‘x’
~~~

但是上面这种写法，如果属性众多的话就会存在Repeat Yourself的地方，这时就可以用到下面三个类宏:

* `Module#attr_reader`生成一个读方法
* `Module#attr_writer`生成一个写方法
* `Module#attr_accessor`同时生成读方法和写方法

#### 单件类
我们知道Ruby中对象的方法的查找顺序是: 先向右，再向上，其含义就是先向右找到对象的类，先在类的实例方法中尝试查找，
如果没有找到，再继续顺着祖先链找。
单件方法是指那些只针对某个对象有效的方法，那么如果为一个对象定义了单件方法，那么这个单件方法的会在祖先链的什么位置呢

~~~ruby
class MyClass
  def my_method; end
end

obj = MyClass.new

def obj.my_singleton_method; end
~~~

首先，单件方法不会在obj中，因为obj不是一个类，其次它也不在MyClass中，那样的话所有的MyClass都应该能共享调用这个方法，也就构不成单件类了。
同理，单件方法也不能在祖先链的某个位置(类似superclass: Object)中。正确的位置是在单件类中，不同的是这类与普通的类还是有稍稍不同的。
它是一个对象特有的隐藏类，也可以称其为元类或本征类。

Ruby提供了两种方法获取单件类的引用，一种是通过传统的关键词class配合特殊的语法;
另一种是通过`Object#singleton_class`方法来获得单件类的引用

~~~ruby
# 法一
class << an_object
  # do something
end

obj = Object.new
singleton_class = class << obj
  self
end
singleton_class.class  # => Class

# 方法二
"abc".singleton_class   # => #<Class: #<String:0xxxxxx>>
~~~

单件类的特性

* 每个单件类只有一个实例（被称为单件类的原因），而且不能被继承
* 单件类是一个对象的单件方法的存活所在

#### 引入单件类后的方法查找
基于上面对单件类的基本认识，引入单件类后，Ruby的方法查找方式就不应该是先从其类(普通类)开始，
而是应该先从对象的单件类中开始查找，如果在单件类中没有找到想要的方法，它才会开始沿着类(普通类)开始，
再到祖先链上去找。这样从单件类之后开始，一切又回到了我们在没有引入单件类时候的次序

~~~ruby
class C
  def a_method
    'C#a_method()'
  end
end

class D < C; end

obj = D.new

# 打开单件类定义单件方法
class << obj
  def a_singleton_method
    'obj#a_singleton_method()'
  end
end

obj.singleton_class.superclass  #=> D
~~~
<img src="/assets/images/ruby_metaprogram/obj_singleton_class.jpg" />

#### Ruby对象模型的7条规则
  1. 只有一个对象 —— 要么是普通对象，要么是模块
  2. 只有一个模块 —— 可以是一个普通模块、一个类或者一个单件类
  3. 只有一个方法，它存在于一个模块中 —— 通常是一个类中
  4. 每个对象（包括类）都有自己的“真正的类”—— 要么是一个普通类，要么是一个单件类
  5. 除了BasicObject类没有超类外，每个类有且只有一个祖先(superclass) —— 要么是一个类，要么是一个模块。这意味着任何类只有一条向上的，直到BasicObject的祖先链
  6. 一个对象的单件类的超类是这个对象的类；一个类的单件类的超类是这个类的超类的单件类。(结合上图)
  7. 调用一个方法时，Ruby先向右迈出一步进入接收者真正的类(单件类)，然后向上进入祖先链。

#### 类方法
类方法其实质是生活在该类的单件类中的单件方法。其定义方法有三种，分别是:

~~~ruby
# 方法一
def MyClass.a_class_method; end

# 方法二
class MyClass
  def self.anther_class_method; end
end

# 方法三(表明类方法的实质)
class MyClass
  class << self
    def yet_another_class_method; end
  end
end
~~~

#### 类与对象拓展
当一个类包含(include)一个模块时，他获得的是该模块的实例方法，而不是类方法。类方法存在于模块的单件类中，类无法获取

**类扩展**通过向类的单件类中添加模块来定义类方法

~~~ruby
module MyModule
  def my_method; 'hello'; end
end

class MyClass
  class << self
    include MyModule
  end
end

MyClass.my_method
~~~

上面代码展示了具体类扩展的实现方式，将一个MyModule模块引入到MyClass类的单件类中，
因为my_method方法是MyClass的单件类的一个实例方法，这样，my_method方法也是MyClass的一个类方法

利用类方法是单件方法的特例，因此可以把类扩展这种技巧应用到任意对象上，这种技巧即为**对象扩展**

~~~ruby
# 法一: 打开单件类来扩展
module MyModule
  def my_method; 'hello'; end
end

obj = Object.new
class << obj
  include MyModule
end

obj.my_method           # => "hello"
obj.singleton_methods   # => [:my_method]

# 法二：Object#extend方法
module MyModule
  def my_method; 'hello'; end
end
obj = Object.new

# 对象扩展
obj.extend MyModule
obj.my_method   # => "hello"

# 类扩展
class MyClass
  extend MyModule
end
MyClass.my_method  # => "hello"
~~~

Object#extend是在接受者的单件类中包含模块的快键方式

**总结**：所以在定义一个类的时候，可以这么这么理解使用
* 在类定义中引入模块，使模块中的方法成为类的实例方法: include
* 在类定义中引入模块，使模块中的方法成为类的类方法: extend

另外还有一个`Module.included`的方法，可以让include将引入的方法作为类方法

~~~ruby
module MyModule
def self.included(othermod)
  def othermod.cc
    "class_method cc"
  end
end

class MyClass
  include MyModule
end

MyClass.cc  => "class_method cc"
~~~

#### 方法包装器（Method Wrapper）
方法包装器一般适合于，处理一些你不能直接触碰到的方法，或者需要给某个方法进行一些预处理工作，
这个概念有点类似与Python中的装饰器

Ruby中使用`Module#alias_method`方法和`alias`关键字为方法取别名

~~~ruby
class MyClass
  def my_method
    "my_method()"
  end
  alias_method :m, :my_method
end

obj = MyClass.new
obj.my_method    #=> "my_method()"
obj.m            #=> "my_method()"
~~~

需要注意的是，在顶级作用域中(main)中只能使用alias关键字来命名别名，
因为在那里调用不到`Module#alias_method`方法

此外这里，还需要指出一点关于方法重定义的理解误区，一般我们认为方法一旦被重定义了，
就找不回原来的方法了，其实在Ruby中的重定义只是，对方法名与方法实体绑定的一种重新绑定，
如果被重新定义的方法体还有一个其它别名，那么一样还是可以访问到老方法的

**1、环绕别名**

经过下面三个步骤可以实现环绕别名
  1. 给方法定义一个别名
  2. 重定义这个方法
  3. 在新的方法中调用老的方法

上面三步后，即可以保留老方法(第一步)，同时又为原方法增加了新功能。
但其本质上没有改变调用之间的函数名称依赖，是一种名副其实的“新瓶装老酒”的取巧办法。

不过环绕别名也有缺点，就是它污染了类的命名空间，添加了额外的方法名。
不过这一点其实可以通过Ruby中的private关键字把旧方法限定为私有方法。
(Ruby中的public和private实际上针对的是方法名，而非方法本身，言外之意为private方法取个别名，
也可以把私有方法暴露出来)

最后要注意的是不要尝试load两次环绕别名的实现代码，否则得到的执行结果就不是你预期中的了，
不信你可以试着连续走两遍上面的3个步骤。

**2、细化**

前面提到的细化，也可以作为一种方法包装器，在细化的方法中调用super方法，
可以调用会细化之前的方法。此外细化相比起环绕别名，细化的作用范围是文件末尾，而环绕别名则是作用在全局。

~~~ruby
module StringRefinement
  refine String do
    def length
      super > 5 ? 'long' : 'short'
    end
  end
end

using StringRefinement

puts "War and Peace".length  #=> “long”
~~~

**3、下包含包装器 (Module#prepend)**

在介绍祖先链的部分有涉及过include与prepend两个方法，前者是将模块插入到当前类的上方，
后者是插入到下方，而下方的位置，正好是方法查找时优先查找的位置，
利用这一优势，可以覆写当前类的同名方法，同时通过super关键字还可以调用到该类中的原始方法

~~~ruby
module ExplicitString
  def length
    super > 5 ? 'long' : 'short'
  end
end

String.class_eval do
  prepend ExplicitString
end

puts "War and Peace".length  #=> "long"
~~~
