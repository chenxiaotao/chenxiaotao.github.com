---
layout: post
title: ruby元编程读书笔记--对象模型
subtitle:   "Ruby中的对象是什么样的呢?"
date:       2015-09-12 12:00:00
author:     "Sheldon"
header-img: "img/post-bg-ruby-metaprogramming-1.jpg"
tags:       
  - ruby
---

#### 对象内省(introspection)：
* 询问对象的类： `obj.class`
* 询问对象的类的实例方法：`obj.class.instance_methods`
* 不包括继承而来的方法：`obj.class.instance_methods(false)`
* 询问对象的实例变量：`obj.instance_variables`

#### 打开类(open class):
可以重新打开已经存在的类并对之进行动态修改，即使像String或者Array这样标准库的类也不例外。这种行为方式称之为打开类

#### 猴子补丁(monkeypath):
如果你粗心地为某个类添加了新功能，同时覆盖了类原来的功能，进而影响到其他部分的代码，这样的patch称之为猴子补丁

#### 类与模块(class and module)：
* Ruby的class关键字更像是一个作用域操作符，而不是类型声明语句。class关键字的核心任务是把你带到类的上下文中，让你可以在里面定义方法
* 每个类都是一个模块，类就是带有三个方法（new，allocate，superclass）的增强模块，通过这三个方法可以组织类的继承结构，并创建对象
* Ruby中的类和模块的概念十分接近，完全可以将二者相互替代，之所以同时保留二者的原因是为了保持代码的清晰性，让代码意图更加明确。使用原则:**希望把自己代码包含(include)到别的代码中，应该使用模块；希望某段代码被实例化或被继承，应该使用类**
* 模块机制可以用来实现类似其它语言中的命名空间(Namespace)概念，如： `Rake::Task` ，Rake就是用来充当常量容器的模块

#### 常量(constants)：
* Ruby中常量的路径(作用域)，类似与文件系统中的目录，通过::进行分割和访问，默认直接以::开头(例: :: Y)表示变量路径的根位置
* `Module#constants` (实例方法) 返回当前范围内的所有常量
* `Module.constants` (类方法) 返回当前程序中所有顶层常量(包括类名)
* `Module.nesting`  获取当前代码所在路径

