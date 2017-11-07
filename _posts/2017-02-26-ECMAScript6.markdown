---
layout:     post
title:      ECMAScript 6
subtitle:   "let + const，Iterator(迭代器) + for of，模板字符串，集合(set + map)，symbols"
date:       2017-02-26
author:     "Sheldon"
header-img: "img/pic/post/new-nodejs.jpg"
tags:       
  - javascript
  - ES6
  - nodejs
---

### let + const

let 和 const与var一样都是进行变量声明，其中const用来声明常量，但是与var声明的方式又有很多的区别：

* 形成块级作用域
* 变量不会提升
* 暂时性死区
* 声明的全局变量不是全局对象的属性
* 不能对已定义过的变量进行重新定义
* const在声明变量时就必须赋值

~~~javascript
// eg1
function f1() {
  let n = 5;
  if (true) {
    let n = 10;
  }
  console.log(n); // 5
}
~~~

上面的函数有两个代码块，都声明了变量n，运行后输出5。这表示外层代码块不受内层代码块的影响，如果使用var定义变量n，最后输出的值就是10

~~~javascript
// eg2
console.log(foo); // 输出undefined
console.log(bar); // 报错ReferenceError

var foo = 2;
let bar = 2;
~~~

上面代码中，变量foo用var命令声明，会发生变量提升，即脚本开始运行时，变量foo已经存在了，但是没有值，所以会输出undefined。
变量bar用let命令声明，不会发生变量提升，这表示在声明它之前，变量bar是不存在的，这时如果用到它，就会抛出一个错误

~~~javascript
// eg3
var tmp = 123;

if (true) {
  tmp = 'abc'; // ReferenceError
  let tmp;
}
~~~

上面代码中，存在全局变量tmp，但是块级作用域内let又声明了一个局部变量tmp，导致后者绑定这个块级作用域，所以在let声明变量前，对tmp赋值会报错。
ES6明确规定，如果区块中存在let和const命令，这个区块对这些命令声明的变量，从一开始就形成了封闭作用域。凡是在声明之前就使用这些变量，就会报错。
总之，在代码块内，使用let命令声明变量之前，该变量都是不可用的。这在语法上，称为“暂时性死区”(temporal dead zone，简称TDZ)

~~~javascript
// eg4
var a = 1;
window.a // 1

let b = 1;
window.b // undefined
~~~

上面代码中，全局变量a由var命令声明，所以它是全局对象的属性；全局变量b由let命令声明，所以它不是全局对象的属性，返回undefined

~~~javascript
// eg5
// 报错
function () {
  let a = 10;
  var a = 1;
}

// 报错
function () {
  let a = 10;
  let a = 1;
}

//报错
function s() {
    let a = 10;
    if (true) {
        var a = 12; //如果这里用let声明就不会报错
    }
}
~~~

let不允许在相同作用域内，重复声明同一个变量

~~~javascript
// eg6
'use strict';
const PI = 3.1415;
PI // 3.1415

PI = 3;
// TypeError: "PI" is read-only
~~~

严格模式下，const的常量一旦声明，常量的值就不能改变

~~~javascript
'use strict';
const foo;
// SyntaxError: missing = in const declaration
~~~

const声明的变量不得改变值，这意味着，const一旦声明变量，就必须立即初始化，不能留到以后赋值

~~~javascript
const foo = {};
foo.prop = 123;

foo.prop
// 123

foo = {} // TypeError: "foo" is read-only
~~~

常量foo储存的是一个地址，这个地址指向一个对象。不可变的只是这个地址，即不能把foo指向另一个地址，但对象本身是可变的，所以依然可以为其添加新属性

es7 块级作用域的使用场景

~~~javascript
var a = [];
for (let i = 0; i < 10; i++) {
  a[i] = function () {
    console.log(i);
  };
}
a[6](); // 6

// 形成一个私有作用域
(function () {
  var tmp = ...;
  ...
}());

// 块级作用域写法
{
  let tmp = ...;
  ...
}
~~~

在for循环中，每次使用的i都是不同的值，如果是用var声明的i，则i的值都会是10

### Iterator(迭代器) + for of
JavaScript原有的表示“集合”的数据结构，主要是数组（Array）和对象（Object），ES6又添加了Map和Set。
这样就有了四种数据集合，用户还可以组合使用它们，定义自己的数据结构，比如数组的成员是Map，Map的成员是对象。这样就需要一种统一的接口机制，来处理所有不同的数据结构。
遍历器（Iterator）就是这样一种机制。它是一种接口，为各种不同的数据结构提供统一的访问机制。任何数据结构只要部署Iterator接口，就可以完成遍历操作
（即依次处理该数据结构的所有成员）

Iterator的作用有三个：

* 一是为各种数据结构，提供一个统一的、简便的访问接口；
* 二是使得数据结构的成员能够按某种次序排列；
* 三是ES6创造了一种新的遍历命令for…of循环，Iterator接口主要供for…of消费

ES6规定，默认的Iterator接口部署在数据结构的`Symbol.iterator`属性，或者说，一个数据结构只要具有Symbol.iterator属性，就可以认为是“可遍历的”（iterable）。
调用`Symbol.iterator`方法，就会得到当前数据结构默认的遍历器生成函数。`Symbol.iterator`本身是一个表达式，返回Symbol对象的iterator属性，
这是一个预定义好的、类型为Symbol的特殊值，所以要放在方括号内（请参考Symbol一章）

在ES6中，有三类数据结构原生具备Iterator接口：数组、某些类似数组的对象、Set和Map结构。

~~~javascript
// eg1
let arr = ['a', 'b', 'c'];
let iter = arr[Symbol.iterator]();

iter.next() // { value: 'a', done: false }
iter.next() // { value: 'b', done: false }
iter.next() // { value: 'c', done: false }
iter.next() // { value: undefined, done: true }
~~~

上面代码中，变量arr是一个数组，原生就具有遍历器接口，部署在arr的Symbol.iterator属性上面。所以，调用这个属性，就得到遍历器对象

对数组和Set结构进行解构赋值时，会默认调用Symbol.iterator方法

~~~javascript
// eg2

let set = new Set().add('a').add('b').add('c');

let [x,y] = set;
// x='a'; y='b'

let [first, ...rest] = set;
// first='a'; rest=['b','c'];
~~~ 

扩展运算符（...）也会调用默认的iterator接口,只要某个数据结构部署了Iterator接口，就可以对它使用扩展运算符，将其转为数组

~~~javascript
eg3
// 例一
var str = 'hello';
[...str] //  ['h','e','l','l','o']

// 例二
let arr = ['b', 'c'];
['a', ...arr, 'd']
// ['a', 'b', 'c', 'd']

let arr = [...iterable];
~~~
yield*后面跟的是一个可遍历的结构，它会调用该结构的遍历器接口

~~~javascript
// eg4

let generator = function* () {
  yield 1;
  yield* [2,3,4];
  yield 5;
};

var iterator = generator();

iterator.next() // { value: 1, done: false }
iterator.next() // { value: 2, done: false }
iterator.next() // { value: 3, done: false }
iterator.next() // { value: 4, done: false }
iterator.next() // { value: 5, done: false }
iterator.next() // { value: undefined, done: true }
~~~

for…of循环可以使用的范围包括数组、Set和Map结构、某些类似数组的对象（比如arguments对象、DOM NodeList对象）、Generator对象，以及字符串

~~~javascript
// eg1
const arr = ['red', 'green', 'blue'];
let iterator  = arr[Symbol.iterator]();

for(let v of arr) {
  console.log(v); // red green blue
}

for(let v of iterator) {
  console.log(v); // red green blue
}

// eg2
var engines = new Set(["Gecko", "Trident", "Webkit", "Webkit"]);
for (var e of engines) {
  console.log(e);
}
// Gecko
// Trident
// Webkit
~~~

遍历的顺序是按照各个成员被添加进数据结构的顺序。其次，Set结构遍历时，返回的是一个值，而Map结构遍历时，返回的是一个数组，
该数组的两个成员分别为当前Map成员的键名和键值

~~~javascript
let map = new Map().set('a', 1).set('b', 2);

for (let pair of map) {
  console.log(pair);
}
// ['a', 1]
// ['b', 2]

for (let [key, value] of map) {
  console.log(key + ' : ' + value);
}
// a : 1
// b : 2
~~~

### 模板字符串

ES6引入了一种新型的字符串字面量语法，称之为模板字符串，它为JavaScript提供了简单的字符串插值功能，你可以通过一种更加美观、更加方便的方式向字符串中插值。

现在我们一般插入模板是这样的：

~~~javascript
$("#result").append(
  "There are <b>" + basket.count + "</b> " +
  "items in your basket, " +
  "<em>" + basket.onSale +
  "</em> are on sale!"
);
~~~

这种写法很繁琐，而且不太美观。
有了es6的模板字符串功能，我们就可以这样写：

~~~javascript
$("#result").append(`
  There are <b>${basket.count}</b> items
   in your basket, <em>${basket.onSale}</em>
  are on sale!
`);
~~~

**一些特点**

* 用反引号（`）标识
* 所有的空格和缩进都会被保留在输出之中
* 字符串中嵌入变量，需要将变量名写在${}之中, 大括号中就相当于一个js表达式
* 模板字符串中能调用函数

~~~javascript
function fn() {
  return "Hello World";
}

`foo ${fn()} bar`
// foo Hello World bar
~~~

### 集合(set + map)

ES6提供了新的数据结构Set。它类似于数组，但是成员的值都是唯一的，没有重复的值。
Set本身是一个构造函数，用来生成Set数据结构

~~~javascript
var s = new Set();

[2,3,5,4,5,2,2].map(x => s.add(x))

for (let i of s) {console.log(i)}
// 2 3 5 4
~~~

**set的特点：**

* Set函数可以接受一个数组（或类似数组的对象）作为参数，用来初始化
* 向Set加入值的时候，不会发生类型转换，所以5和"5"是两个不同的值
* 在Set内部，两个NaN是相等
* 两个对象总是不相等的：s.add({})

**Set实例的属性和方法**：

* `Set.prototype.size`返回Set实例的成员总数
* `add(value)`添加某个值，返回Set结构本身
* `delete(value)`删除某个值，返回一个布尔值，表示删除是否成功
* `has(value)`返回一个布尔值，表示该值是否为Set的成员
* `clear()`清除所有成员，没有返回值
* `keys()`返回一个键名的遍历器
* `values()`返回一个键值的遍历器
* `entries()`返回一个键值对的遍历器
* `forEach()`使用回调函数遍历每个成员

~~~javascript
// eg1
s.add(1).add(2).add(2);
// 注意2被加入了两次

s.size // 2

s.has(1) // true
s.has(2) // true
s.has(3) // false

s.delete(2);
s.has(2) // false

// eg2
// Array.from方法可以将Set结构转为数组，这样提供了一种去除数组的重复元素的方法
function dedupe(array) {
  return Array.from(new Set(array));
}

dedupe([1,1,2,3]) // [1, 2, 3]

// 更简洁的方法
var set = new Set(['red', 'green','green', 'blue', 'blue']);
var arr = [...set];
console.log(arr);
// ['red', 'green', 'blue']

// eg3
let set = new Set(['red', 'green', 'blue']);

for ( let item of set.keys() ){
  console.log(item);
}
// red
// green
// blue

for ( let item of set.values() ){
  console.log(item);
}
// red
// green
// blue

for ( let item of set.entries() ){
  console.log(item);
}
// ["red", "red"]
// ["green", "green"]
// ["blue", "blue"]
~~~

### Map
ES6提供了Map数据结构，它类似于对象，也是键值对的集合，但是“键”的范围不限于字符串，各种类型的值（包括对象）都可以当作键。
也就是说，Object结构提供了“字符串—值”的对应，Map结构提供了“值—值”的对应，是一种更完善的Hash结构实现。如果你需要“键值对”的数据结构，Map比Object更合适。

**Map的特点：**

* “键”的范围不限于字符串，各种类型的值（包括对象）都可以当作键
* new Map时，可以接受一个数组作为参数
* 如果对同一个键多次赋值，后面的值将覆盖前面的值
* 如果读取一个未知的键，则返回undefined
* 只有对同一个对象的引用，Map结构才将其视为同一个键
* set方法返回的是Map本身，因此可以采用链式写法

~~~javascript
// eg1
var m = new Map();
var o = {p: "Hello World"};

m.set(o, "content")
m.get(o) // "content"

m.has(o) // true
m.delete(o) // true
m.has(o) // false
~~~

上面代码使用set方法，将对象o当作m的一个键，然后又使用get方法读取这个键，接着使用delete方法删除了这个键

~~~javascript
// eg2
var map = new Map([["name", "张三"], ["title", "Author"]]);

map.size // 2
map.has("name") // true
map.get("name") // "张三"
map.has("title") // true
map.get("title") // "Author"

var map = new Map();

map.set(['a'], 555);
map.get(['a']) // undefined

let map = new Map()
  .set(1, 'a')
  .set(2, 'b')
  .set(3, 'c');

~~~

Map原生提供三个遍历器生成函数和一个遍历方法

* `keys()`返回键名的遍历器。
* `values()`返回键值的遍历器。
* `entries()`返回所有成员的遍历器。
* `forEach()`遍历Map的所有成员。

~~~javascript
// eg3
let map = new Map([
  ['F', 'no'],
  ['T',  'yes'],
]);

for (let key of map.keys()) {
  console.log(key);
}
// "F"
// "T"

for (let value of map.values()) {
  console.log(value);
}
// "no"
// "yes"

for (let item of map.entries()) {
  console.log(item[0], item[1]);
}
// "F" "no"
// "T" "yes"

// 或者
for (let [key, value] of map.entries()) {
  console.log(key, value);
}

// 等同于使用map.entries()
for (let [key, value] of map) {
  console.log(key, value);
}
~~~

### WeakSet

WeakSet结构与Set类似，也是不重复的值的集合。它与Set有两个区别：

* 首先，WeakSet的成员只能是对象，而不能是其他类型的值。
* 其次，WeakSet中的对象都是弱引用，即垃圾回收机制不考虑WeakSet对该对象的引用，也就是说，如果其他对象都不再引用该对象，
那么垃圾回收机制会自动回收该对象所占用的内存，不考虑该对象还存在于WeakSet之中。这个特点意味着，无法引用WeakSet的成员，因此WeakSet是不可遍历的

**特点:**
* 成员只能是对象
* 成员都是弱引用，可以被GC回收
* 无法引用WeakSet的成员，WeakSet集合是不可遍历的
* WeakSet的一个用处，是储存DOM节点，而不用担心这些节点从文档移除时，会引发内存泄漏

~~~javascript
// eg
var ws = new WeakSet();
ws.add(1)
// TypeError: Invalid value used in weak set
ws.add(Symbol())
// TypeError: invalid value used in weak set

var a = [[1,2], [3,4]];
var ws = new WeakSet(a);

var ws = new WeakSet();
var obj = {};
var foo = {};

ws.add(window);
ws.add(obj);

ws.has(window); // true
ws.has(foo);    // false

ws.delete(window);
ws.has(window);    // false

ws.size // undefined
ws.forEach // undefined

ws.forEach(function(item){ console.log('WeakSet has ' + item)})
// TypeError: undefined is not a function
~~~

### WeakMap

WeakMap结构与Map结构基本类似，唯一的区别是它只接受对象作为键名（null除外），不接受其他类型的值作为键名，而且键名所指向的对象，不计入垃圾回收机制。

~~~javascript
// eg
var map = new WeakMap()
map.set(1, 2)
// TypeError: 1 is not an object!
map.set(Symbol(), 2)
// TypeError: Invalid value used as weak map key

var wm = new WeakMap();
var element = document.querySelector(".element");

wm.set(element, "Original");
wm.get(element) // "Original"

element.parentNode.removeChild(element);
element = null;
wm.get(element) // undefined
~~~

WeakMap应用的典型场合就是DOM节点作为键名

~~~javascript
let myElement = document.getElementById('logo');
let myWeakmap = new WeakMap();

myWeakmap.set(myElement, {timesClicked: 0});

myElement.addEventListener('click', function() {
  let logoData = myWeakmap.get(myElement);
  logoData.timesClicked++;
  myWeakmap.set(myElement, logoData);
}, false);
~~~

### symbols

ES5的对象属性名都是字符串，这容易造成属性名的冲突。比如，你使用了一个他人提供的对象，但又想为这个对象添加新的方法，新方法的名字就有可能与现有方法产生冲突。
如果有一种机制，保证每个属性的名字都是独一无二的就好了，这样就从根本上防止属性名的冲突。这就是ES6引入Symbol的原因

ES6引入了一种新的原始数据类型Symbol，表示独一无二的值。它是JavaScript语言的第七种数据类型，
前六种是：Undefined、Null、布尔值（Boolean）、字符串（String）、数值（Number）、对象（Object）

Symbol值通过Symbol函数生成，这就是说，对象的属性名现在可以有两种类型，一种是原来就有的字符串，另一种就是新增的Symbol类型。
凡是属性名属于Symbol类型，就都是独一无二的，可以保证不会与其他属性名产生冲突。

**一些特点：**

* 生成一个独一无二的值
* 不能通过`new Symbol()`的方式去生成symbol，否则会报错
* 生成的Symbol是一个原始类型的值，不是对象，所以不能添加属性，它是一种类似于字符串的数据类型
* Symbol函数的参数只是表示对当前Symbol值的描述，因此相同参数的Symbol函数的返回值是不相等的
* Symbol值不能与其他类型的值进行运算，会报错
* Symbol值可以显式转为字符串，也可以转为布尔值，但是不能转为数值
* Symbol值作为对象属性名时，不能用点运算符，必须放在方括号之中
* Symbol作为属性名，该属性不会出现在for…in、for…of循环中，也不会被`Object.keys()``Object.getOwnPropertyNames()`返回。
但是，它也不是私有属性，有一个`Object.getOwnPropertySymbols`方法，可以获取指定对象的所有Symbol属性名

~~~javascript
// eg1
// 没有参数的情况
var s1 = Symbol();
var s2 = Symbol();

s1 === s2 // false

// 有参数的情况
var s1 = Symbol("foo");
var s2 = Symbol("foo");

s1 === s2 // false

var sym = Symbol('My symbol');

"your symbol is " + sym
// TypeError: can't convert symbol to string
~~~

上面代码中，s1和s2都是Symbol函数的返回值，而且参数相同，但是它们是不相等的。

~~~javascript
// eg2
var mySymbol = Symbol();

// 第一种写法
var a = {};
a[mySymbol] = 'Hello!';

// 第二种写法
var a = {
  [mySymbol]: 'Hello!'
};

// 第三种写法
var a = {};
Object.defineProperty(a, mySymbol, { value: 'Hello!' });

// 以上写法都得到同样结果
a[mySymbol] // "Hello!"
~~~

由于每一个Symbol值都是不相等的，这意味着Symbol值可以作为标识符，用于对象的属性名，就能保证不会出现同名的属性。
这对于一个对象由多个模块构成的情况非常有用，能防止某一个键被不小心改写或覆盖。

使用Symbol值定义属性时，Symbol值必须放在方括号之中

~~~javascript
// eg3
var mySymbol = Symbol();
var a = {};

a.mySymbol = 'Hello!';
a[mySymbol] // undefined
a['mySymbol'] // "Hello!"

let s = Symbol();

let obj = {
  [s]: function (arg) { ... }
};

obj[s](123);
~~~

Symbol的其他方法: `Symbol.for()` `Symbol.keyFor()`

~~~javascript
Symbol.for("bar") === Symbol.for("bar")
// true

Symbol("bar") === Symbol("bar")
// false

var s1 = Symbol.for("foo");
Symbol.keyFor(s1) // "foo"

var s2 = Symbol("foo");
Symbol.keyFor(s2) // undefined
~~~

### 参考资料

> * [ECMAScript 6 入门](http://es6.ruanyifeng.com/)
