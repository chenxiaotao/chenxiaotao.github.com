---
layout:     post
title:      JavaScript执行顺序
subtitle:   "先同步后异步"
date:       2017-02-16
author:     "Sheldon"
header-img: "img/pic/post/new-nodejs.jpg"
tags:       
  - javascript
---

先上一段代码： 

~~~javascript
setTimeout(function() {
  console.log('settimeout out')
  setTimeout(function() {
    console.log('settimeout in')
  }, 0);

  process.nextTick(function() {
    console.log('process.nextTick');
  });
}, 0);

setTimeout(function() {
  console.log('settimeout');
},0);

process.nextTick(function() {
  console.log('process.nextTick');
});

console.log('main thread');

// 输出结果:
// main thread
// process.nextTick
// settimeout out
// settimeout
// process.nextTick
// settimeout in
~~~

分析以上代码，首先代码根据顺序从上往下，依次执行，遇到同步任务就立即执行，
故先输出 main thread；遇到异步任务，会把异步任务放入任务队列，
由于异步任务中 `process.nextTick()` 操作优先级最高，故会先被执行；
然后输出setTimeout的异步任务，由于我在上面的第一个异步任务中加入了另一个setTimeout，
这个异步任务的数据会在最后执行。由于上述的setTimeout的时间都是0，所有可以从上述代码的
输出结果看出代码入队列的顺序

总结：首先在第一个时间片内，程序会遍历整个代码，然后将异步任务放入队列。然后继续执行
主线程上的代码，第一次轮询完成之后，进入队列找已经执行完成的任务，并执行其回调；
如果没有执行完成，那么跳过继续后面的任务；如果一个异步任务的回调中又有异步任务，
会继续加入任务队列；然后如此轮询直到所有的任务都执行完毕
