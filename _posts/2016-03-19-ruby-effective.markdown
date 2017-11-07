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

### include和extend的区别

* 在类定义中引入模块，使模块中的方法成为类的实例方法: include
* 在类定义中引入模块，使模块中的方法成为类的类方法: extend
* 在类定义中引入模块，既希望引入实例方法，也希望引入类方法，这个时候需要使用 include，但是在模块中对类方法的定义有不同，定义出现在 方法 def self.included(c) ... end 中 

### rails中save过程逻辑
update：

validate -> before_save -> before_update -> after_update -> after_save -> after_commit

create：

validate -> before_save -> before_create -> after_create -> after_save -> after_commit
