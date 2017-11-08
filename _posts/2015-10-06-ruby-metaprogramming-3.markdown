---
layout:     post
title:      《Ruby元编程》 读书笔记3
subtitle:   "Ruby的代码块"
date:       2015-10-06
author:     "Sheldon"
header-img: "img/pic/post/ruby-metaprogramming.jpg"
tags:       
  - ruby
  - 元编程
  - 面向对象
  - 读书笔记
---

#### 代码块
* 代码块的定义方式有{}花括号与do...end关键字定义两种，程序员们习惯单行用花括号，多行用do...end
* 代码块只有在方法调用的时候才可以定义，块会被直接传递给这个方法，判断某个方法调用中是否包含代码块，
可以通过`Kernel#block_given?`
* 代码块不仅可以有自己的参数，也会有返回值，往往没有代码块中的最后一行执行结果会被作为返回值返回
* 代码块之所以可以执行，是因为其不仅包含代码，同时也涵盖一组相应绑定，即执行环境，也可以称之为上下文环境。
代码块可以携带这个上下文环境，到任何一个代码块可以达到的地方。
也就可以说，一个代码块是一个闭包，当定义一个代码块时，它会捕获当前环境中的绑定，并带它们四处流动

#### 作用域
Ruby中不具备嵌套作用域(即在内部作用域，可以看到外部作用域的)的特点，它的作用域是截然分开的，
一旦进入一个新的作用域，原先的绑定会被替换为一组新的绑定

程序会在三个地方关闭前一个作用域，同时打开一个新的作用域，它们是:

* 类定义class
* 模块定义 module
* 方法定义 def

上面三个关键字，每个关键字对应一个作用域门(进入)，相应的end则对应离开这道门

#### 扁平化作用域

从一个作用域进入另一个作用域的时候，局部变量会立即失效，为了让局部变量持续有效，
可以通过规避关键字的方式，使用方法调用来代替作用域门，让一个作用域看到另一个作用域里的变量，从而达到目的。
具体做法是，通过`Class.new`替代class，`Module#define_method`代替def，`Module.new`代替module。
这种做法称为扁平作用域，表示两个作用域挤压到一起

~~~ruby
# 示例代码(Wrong)
my_var = "Success"
class MyClass
  puts my_var  #这里无法正确打印"Success"
  def my_method
    puts my_var  #这里无法正确打印"Success"
  end
end

# 示例代码(Right)
my_var = "Success"
MyClass = Class.new do
  puts "#{my_var} in  the class definition"
  define_method :my_method do
    "#{my_var} in the method"
  end
end
~~~

#### 共享作用域
将一组方法定义到，某个变量的扁平作用域中，可以保证变量仅被有限的几个方法所共享。这种方式称为共享作用域

#### instance_eval方法
这个`BasicObject#instance_eval`有点类似JS中的bind方法，不同的时，bind是将this传入到对象中，
而instance_eval则是将代码块(上下文探针Context Probe)传入到指定的对象中，一个是传对象，一个是传执行体。
通过这种方式就可以在instance_eval中的代码块里访问到调用者对象中的变量

~~~ruby
class MyClass
  def initialize
    @v = 1
  end
end
obj = MyClass.new

obj.instance_eval do
  self    #=> #<MyClass:0x33333 @v=1>
  @v      #=> 1  
end

v = 2
obj.instance_eval { @v = v }
obj.instance_eval { @v }   # => 2
~~~

此外，instance_eval方法还有一个双胞胎兄弟：instance_exec方法。相比前者后者更加灵活，允许对代码块传入参数

~~~ruby
class C
  def initialize
    @x = 1
  end
end

class D
  def twisted_method
    @y = 2
    #C.new.instance_eval { "@x: #{@x}, @y>: #{y}" }
    C.new.instance_exec(@y) { |y| "@x: #{@x}, @y: #{y}" }
  end
end
#D.new.twisted_method   # => "@x: 1, @y: "
D.new.twisted_method   # => "@x: 1, @y: 2"
~~~

因为调用instance_eval后，将调用者作为了当前的self，所以作用域更换到了class C中，之前的作用域就不生效了。
这时如果还想访问到之前@y变量，就需要通过参数打包上@y一起随instance_eval转义，
但因为instance_eval不能携带参数，所以使用其同胞兄弟instance_exec方法

#### 洁净室
一个只为在其中执行块的对象，称为洁净室，其只是一个用来执行块的环境。
可以使用BasicObject作为洁净室是个不错的选择，因为它是白板类，内部的变量和方法都比较有限，不会引起冲突

#### Proc对象
Proc是由块转换来的对象。创建一个Proc共有四种方法,分别是:

~~~ruby
# 法一
inc = Proc.new { | x | x + 1}
inc.call(2)  #=> 3

# 法二
inc = lambda {| x | x + 1 }
inc.call(2)  #=> 3

# 法三
inc = ->(x) { x + 1}
inc.call(2) #=> 3

# 法四
inc = proc {|x| x + 1 }
inc.call(2) #=> 3
~~~

除了上面的四种之外，还有一种通过&操作符的方式，将代码块与Proc对象进行转换。
如果需要将某个代码块作为参数传递给方法，需要通过为这个参数添加&符号，并且其位置必须是在参数的最后一个
&符号的含义是：这是一个Proc对象，我想把它当成代码块来使用。去掉&符号，将能再次得到一个Proc对象

~~~ruby
def my_method(&the_proc)
  the_proc
end

p = my_method {|name| "Hello, #{name} !"}
p.class   #=> Proc
p.call("Bill")   #=> "Hello,Bill"

def my_method(greeting)
  "#{greeting}, #{yield}!"
end

my_proc = proc { "Bill" }
my_method("Hello", &my_proc)
~~~

#### Proc与Lambda对比
使用Lambda方法创建的Proc与其它方式创建的Proc是有一些差别的，用lambda方法创建的Proc称为lambda，
而用其他方式创建的则称为proc。通过`Proc#lambda?`可以检测Proc是不是lambda

二者之间主要的差异有以下两点:

* Proc、Lambda的return含义不同; lambda中的return表示的仅仅是从lambda中返回。而proc中，return的行为则不同，其并不是从proc中返回，而是从定义proc的作用域中返回。即相当与在你的业务代码处返回。

  ~~~ruby
  def double(callable_object)
    p = Proc.new { return 10 }
    result = p.call   
    return result * 2 # 不可达的代码
  end

  double #=> 10
  ~~~

* Proc、Lambda的return参数检查方式不同；Proc的参数检查要比Lambda参数检查要更宽松一些，
如果传入Proc中的参数数量不匹配其不会发生报错，会自行进行一定的调整到期望参数的样子，
但是对于lambda则不同，如果出现参数不匹配的情况，其往往会报ArgumentError异常，中断程序

  ~~~ruby
  p = Proc.new { |a, b|  [a,b] }
  p.call(1,2,3) #=> [1, 2]
  p.call(1) #=> [1, nil]
  ~~~

#### Proc与Lambda之间的选择
lambda更直观，更像是一个方法，参数要求更加严格，return也更像是方法定义中的return，
往往Ruby程序员会将lambda作为第一选择

* 代码块(不是真正的对象，但是它们是"可调用的") 在定义它们的作用域中执行
* proc: Proc类的对象与代码块一样，也在定义自身的作用域中执行
* lambda: 同样是Proc类的对象，但跟普通的proc有细微差别。与代码块与proc一样都是闭包，
也在定义自身的作用域中执行
* 方法: 绑定于一个对象，在所绑定对象的作用域中执行。他们也可以与这个作用域解除绑定，
然后再重新绑定到另一个对象的作用域中

#### Method 对象
通过Kernel#method方法，可以获得一个用Method对象表示的方法，在之后可以用Method#call方法对其进行调用。
同样也可以用Kernel#singleton_method方法把单件方法名转换为Method对象

~~~ruby
class MyClass
  def initialize(value)
    @x = value
  end

  def my_method
    @x
  end
end

obj = MyClass.new(1)
m = obj.method :my_method
m.call   #=> 1
~~~

#### 自由方法
与普通方法类似，不同的是它会从最初定义它的类或模块中脱离出来(即脱离之前的作用域)，
通过`Method#unbind`方法，可以将一个方法变为自由方法。同时，
你也可以通过调用`Module#instance_method`获得一个自由方法

~~~ruby
module MyClass
  def my_method
    42
  end
end

unbound = MyModule.instance_method(:my_method)
unbound.class  #=> UnboundMethod
~~~

上面代码中的UnboundMethod对象，不能够直接用来调用，不过可以将一个UnboundMethod方法绑定到一个对象上，
使其再次成为一个Method对象，然后再被调用执行

上面的绑定过程可以通过`UnboundMethod#bind`方法把UnboundMethod对象绑定到一个对象上，
从某个类中分离出来的UnboundMethod对象只能绑定在该类及其子类对象上，
不过从模块中分离出来的UnboundMethod对象在Ruby2.0后就不在有此限制了

此外还可以将UnboundMethod对象传递给`Module#define_method`方法，从而实现绑定

~~~ruby
String.send :define_method, :another_method, unbound
"abc".anther_method  #=> 42
~~~
