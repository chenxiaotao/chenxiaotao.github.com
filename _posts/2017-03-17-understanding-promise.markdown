---
layout:     post
title:      Promise
subtitle:   "使用Promise对付JavaScript中的回调"
date:       2017-03-27
author:     "Sheldon"
header-img: "img/pic/post/new-nodejs.jpg"
tags:       
  - javascript
  - nodejs
---

### 一、什么是Promise
1、Promise即“承诺”的意思，它是异步编程的一种解决方案，或者说是一种规范，目的是让回调更为优雅，可以将复杂的异步处理轻松地进行模式化

2、有了Promise对象，就可以将异步操作以同步操作的流程表达出来，避免了层层嵌套的回调函数

3、此外，Promise 对象提供统一的接口，使得控制异步操作更加容易

4、当我们new一个Promise时，我们需要明确两点：

* 明确承诺的需要完成的事情
* 设置承诺是否实现的标准

国际惯例，来个helloworld：

~~~ javascript
function helloWorld(ready) {
    return new Promise(function (resolve, reject) {
        if (ready) {
            resolve("Hello World!");
        } else {
            reject("Good bye!");
        }
    });
}

helloWorld(true).then(function (message) {
    console.log(message); // 第一个参数对应于resolve
}, function (error) {
    console.log(error);   // 第二个参数对应于reject（可选）
});

// Hello World!
~~~

### 二、Promise的基本概念
1、Promise有三种状态：`Pending`（进行中），`Resolved`（已完成）和 `Rejected`（已失败）

2、对象的状态不受外界影响，只有异步操作的结果，可以决定当前是哪一种状态，任何其他操作都无法改变这个状态

3、Promise新建后立即执行，然后then方法指定的回调函数，将在当前脚本所有同步任务执行完才会执行(即Promise只能进行异步调用)，用专业点的话说就是，立即resolve的Promise对象，是在本轮“事件循环”（event loop）的结束时执行。

~~~ javascript
var promise = new Promise(function(resolve, reject) {
  console.log('Promise');
  resolve();
});

promise.then(function() {
  console.log('Resolved');
});

console.log('Hi!');
// Promise
// Hi!
// Resolved
~~~

4、一旦状态改变，就不会再变，任何时候都可以得到这个结果。Promise 对象的状态改变，只有两种可能：从 Pending 变为 Resolved 和从 Pending 变为 Rejected

~~~ javascript
//使用throw添加错误事件
var p = new Promise(function(resolve, reject) {
    resolve("ok");
    console.log("before error");
    throw new Error('error');
    console.log("after error");
});

p.then(function(value){
    console.log(value);
  })
 .catch(function(err){
    console.log(err);
  });

// before error
// ok
~~~

### 三、Promise的缺点
1、无法取消Promise，一旦新建它就会立即执行，无法中途取消

2、当处于Pending状态时，无法得知目前进展到哪一个阶段（刚刚开始还是即将完成）

3、如果不设置回调函数，Promise内部抛出的错误，不会反应到外部

~~~ javascript
var p = new Promise(function(resolve, reject) {
    throw new Error('error');
    resolve("ok");
});

p.then(function(value){
    console.log(value)
  });
~~~

### 四、Promise的基本用法
1、**`Promise#then`**

`.then()`是一个Promise的实例方法，then方法是定义在原型对象Promise.prototype上的，它的作用是为Promise实例添加状态改变时的回调函数。then方法返回的是一个新的Promise实例(注意，不是原来那个Promise实例)，它有两个参数: 

第一个参数对应resolve()，用于执行Resolved状态的回调函数

第二个参数(选填)参数对应reject()，是Rejected状态的回调函数

2、**`Promise#catch`**

`.catch()`是`.then(null, reject())`的简写，用于指定发生错误时的回调函数，即它就是`.then`的第二个参数。使用`.catch`是可以捕捉`then`方法执行中的错误，而且这种写法是语法更容易理解，所以推荐使用catch

~~~ javascript
var promise = new Promise(function(resolve, reject) {
  throw new Error('test');     // 和下面注释的代码相同
  // reject(new Error('test'));
});

promise.catch(function(error) {
  console.log(error);
});
// Error: test
~~~
可以发现reject方法的作用，等同于抛出错误

3、**Promise的链式写法**

上面提过`.then`返回的是一个新的Promise对象，所以在后面可以继续使用`.then`进行类似方法链的调用，即then方法后面再调用另一个then方法，将上一个then中的返回值作为参数继续往下传递，采用链式的then，可以指定一组按照次序调用的回调函数

~~~ javascript
Promise.resolve(1)
  .then(function(data){
    console.log(data);
    return data * 2;
  })
  .then(function(data){
    console.log(data);
    //throw new Error('error');
    return data * 2;
  })
  .then(function(data){
    console.log(data);
    return data * 2;
  })
  .catch(function(e){
    console.log(e);
  })
// 1
// 2
// 4
~~~

4、**`Promise.resolve()`** 

~~~ javascript
Promise.resolve('foo')
// 等价于
new Promise(function(resolve){
  resolve('foo')
})
~~~
上述例子说明了静态方法Promise.resolve(value) 可以认为是 new Promise() 方法的快捷方式，会返回一个状态为Resolved 的Promise实例

+ 不带参数将直接返回一个Resolved状态的Promise对象
+ 如果参数是Promise实例，那么Promise.resolve将不做任何修改、原封不动地返回这个实例

  ~~~ javascript
  var p = new Promise(function(resolve, reject) {
      resolve("ok");
  });
  console.log(Promise.resolve(p) === p);
  // true
  ~~~

+ 带有then方法的thenable对象，Promise.resolve方法会将这个对象转为Promise对象，然后就立即执行thenable对象的then方法

  ~~~ javascript
  var thenable = {
   then: function(resolve, reject) {
     resolve(42);
   }
  };

  Promise.resolve(thenable).then(function(value) {
   console.log(value);  // 42
  });
  ~~~
+ 如果参数是一个原始值，或者是一个不具有then方法的对象，则Promise.resolve方法返回一个新的Promise对象，状态为Resolved

  ~~~ javascript
  Promise.resolve(42).then(function (s){
    console.log(s); //42
  });
  ~~~

5、**`Promise.reject()`**

`Promise.reject()`方法也会返回一个新的 Promise 实例，该实例的状态为Rejected，等价于 `new Promise((resolve, reject) => reject('error'))`，值得注意的是`Promise.reject()`方法的参数，会原封不动地作为reject的理由，变成后续方法的参数。这一点与Promise.resolve方法不一致

~~~ javascript
const thenable = {
  then(resolve, reject) {
    reject('出错了');
  }
};

Promise.reject(thenable).catch(e => {
  console.log(e === thenable) // true
})
~~~

6、 **`Promise.all`**

用于将多个Promise实例，包装成一个新的Promise实例，参数是一个数组（可以不是数组，但必须具有Iterator接口，且返回的每个成员都是Promise实例），例如`var p = Promise.all([p1,p2,p3])`，该实例的状态是有p1, p2, p3共同决定，有两种情况：

+ 只有p1、p2、p3的状态都变成fulfilled，p的状态才会变成fulfilled，此时p1、p2、p3的返回值组成一个数组，传递给p的回调函数

  ~~~ javascript
  var p1 = Promise.resolve("去数据库拿数据");
  var p2 = Promise.resolve("去redis拿数据");
  Promise.all([p1,p2]).then(function([sql, redis]){
    formatData(sql, redis); // 格式化数据
  });
  ~~~

+ 只要p1、p2、p3之中有一个被rejected，p的状态就变成rejected，此时第一个被reject的实例的返回值，会传递给p的回调函数

7、**`Promise.race`**

`Promise.race`同样是将多个Promise实例，包装成一个新的Promise实例。与`Promise.all`不同的是，只要参数中有一个的状态改变，它的状态就会改变，并将率先改变的Promise的返回值传递给回调函数

### 五、Promise进阶--then
为了加深Promise关于then方法的理解，我们假设一个下面的情况来理解then，以下实例摘自[进击的promise](https://zhuanlan.zhihu.com/p/25198178)

首先我们假设Hello耗时1000ms，World耗时1500ms

~~~ javascript
function Hello() {
  return new Promise((resolve, reject) => {
    setTimeout(() => {
      resolve('Hello')
    }, 1000)
  })
}

function World() {
  return new Promise((resolve, reject) => {
    setTimeout(() => {
      resolve('World')
    }, 1500)
  })
}
~~~

#### 第一种情况(正常的Promise用法)：

~~~ javascript
console.time('case 1')
Hello().then(() => {
  return World()
}).then(function finalHandler(res) {
  console.log(res)
  console.timeEnd('case 1')
})

// World
// case 1: 2509ms

//执行流程
Hello()
|----------|
           World()
           |---------------|
                           finalHandler(somethingElse)
                           |->
~~~

#### 第二种情况(因为没有使用 return，Hello 在 World 执行完后异步执行的)：

~~~ javascript
console.time('case 2')
Hello().then(() => {
  World()
}).then(function finalHandler(res) {
  console.log(res)
  console.timeEnd('case 2')
})

// undefined
// case 2: 1009ms

//执行流程
Hello()
|----------|
           World()
           |---------------|
           finalHandler(somethingElse)
           |->
~~~

#### 第三种情况

~~~ javascript
console.time('case 3')
Hello().then(World())
  .then(function finalHandler(res) {
    console.log(res)
    console.timeEnd('case 3')
  })

// Hello
// case 3: 1008ms

//执行流程
doSomething()
|----------|
doSomethingElse()
|---------------|
           finalHandler(something)
           |->
~~~

解释：我们知道 then 需要接受一个函数，否则会值穿透，所以打印 Hello，上述代码等同于

~~~ javascript
console.time('case 3')
var doHello = Hello()
var doWorld = World()
doHello.then(doWorld)
  .then(function finalHandler(res) {
    console.log(res)
    console.timeEnd('case 3')
  })
~~~

#### 第四种情况

~~~ javascript
console.time('case 4')
Hello().then(World)
  .then(function finalHandler(res) {
    console.log(res)
    console.timeEnd('case 4')
  })

// World
// case 4: 2513ms

// 执行顺序同第一种情况
~~~
解释：World 作为 then 参数传入不会发生值穿透，并返回一个 promise，所以会顺序执行

### 六、参考资料
> * [Javascript Promise 迷你书](http://liubin.org/promises-book/)
> * [阮一峰ES6入门(promise对象)](http://es6.ruanyifeng.com/#docs/promise)
> * [深入promise](https://zhuanlan.zhihu.com/p/25178630)


