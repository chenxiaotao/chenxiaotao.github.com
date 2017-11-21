---
layout:     post
title:      Ruby中的一些常用逻辑
subtitle:   "持续更新ruby中的常用逻辑"
date:       2016-03-19
author:     "Sheldon"
header-img: "img/pic/post/ruby-effective.jpg"
tags:       
  - ruby
  - rails
  - 日常积累
---

### Proc和lambda的区别

* return不同：lambda中的return和普通方法的一样只返回到调用的地方，可以向下执行；而Proc中的return直接就作为调用处的返回，不能向下执行了
* lambda本质上是匿名函数，调用的时候会检查参数,参数不匹配抛ArgumentError，而proc调用时候会传递并绑定参数，但是不会检查

### and/or 和 &&/||

~~~ruby
surprise = true and false # => surprise 的值为 true
surprise = true && false  # => surprise 的值为 false

# 实际运算过程
(surprise = true) and false # => surprise is true
surprise = (true && false)  # => surprise is false
~~~

由上述例子可以看出：

* `and / or` 的优先级比 `=` 低，而 `&& / ||` 的优先级比 `=` 高
* 故`and / or` 运算符的优先级比 `&& / ||` 低
* `and` 和 `or` 的优先级相同，而 `&&` 的优先级比 `||` 高

也有这样的说法`and/or`用于流程控制，而`&&/||`用于布尔型运算。但是控制流程使用`if/unless`不是更好么

最佳实践: **只使用 `==` 运算符**，[延伸阅读](https://stackoverflow.com/questions/2083112/difference-between-or-and-in-ruby)

### 自定义异常不能继承Exception
~~~ruby
class MyException < Exception
end

begin
  raise MyException
rescue
  puts 'Caught it!'
end

# MyException: MyException
#       from (irb):17
#       from /Users/karol/.rbenv/versions/2.1.0/bin/irb:11:in `<main>'
~~~

上述代码中不会捕捉到 MyException，也不会显示 'Caught it!' 的消息

原因分析与使用建议：

* 当使用空的`rescue`语句时，它会捕捉所有继承自`StandardError`的异常，而不是`Exception`
* 如果你使用了`rescue Exception`，问题就更严重了，你会捕捉到你无法恢复的错误（比如内存溢出错误）
而且，你会捕捉到 SIGTERM 这样的系统信号，导致你无法使用 CTRL+C 来中止你的脚本，只是使用`kill -9`
* 自定义异常类时，继承`StandardError`或任何其后代子类（越精确越好）永远不要直接继承`Exception`
* 不要轻易尝试`rescue Exception`如果你想要大范围捕捉异常，直接使用空的`rescue`语句，
或者使用`rescue => e`来获取错误对象

[延伸阅读](https://stackoverflow.com/questions/10048173/why-is-it-bad-style-to-rescue-exception-e-in-ruby)

### 创建私有类方法
~~~ruby
class Foo

  private
  def self.bar
    puts 'Not-so-private class method called'
  end
end

Foo.bar # => "Not-so-private class method called"
~~~

如果这个方法真的是私有方法，那么 Foo.bar 就会抛出`NoMethodError`，但是上面正确打印了，说明调用的并不是私有方法

~~~ruby
class Foo

  # 方法一
  class << self
    private    
    def bar
      puts 'Class method called'
    end    
  end

  # 方法二
  def self.baz
    puts 'Another class method called'
  end
  private_class_method :baz

end

Foo.bar # => NoMethodError: private method `bar' called for Foo:Class
Foo.baz # => NoMethodError: private method `baz' called for Foo:Class
~~~

* 把私有类方法放到`class << self`后的块中
* 使用`private_class_method :method_name`注明

### rails中save过程逻辑
update：

validate -> before_save -> before_update -> after_update -> after_save -> after_commit

create：

validate -> before_save -> before_create -> after_create -> after_save -> after_commit

