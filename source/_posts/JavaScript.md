---
title: 前端基础（三）-- JavaScript
date: 2016-05-31 15:34:31
tags: [前端, JavaScript]
---

# JavaScript简介

``JavaScript``可以说是世界上最流行的脚本语言，这也正达到了作者给这种语言取名字的目的，由于当时``Java``很火，所以这哥们儿为了``JavaScript``也能火起来就取了个相似的名字，并希望``JavaScript``也能火起来，所以现在就有了很多程序员的段子，一脸黑线。

``JavaScript``是一种解释型的动态语言，我们知道用``JavaScript``开发web，然而自从有了``Node.js``，前端程序员瞬间变成了全栈，自从``facebook``开源了跨平台开发框架[react-native](https://github.com/facebook/react-native)，前端程序员瞬间能写源生客户端了，最近阿里也开源了更轻量级``Weex``，而且最近RoyLi的创业公司搞出来一个硬件产品``Ruff``，也可以用``JavaScript``开发，前端程序员瞬间成为了真正的全栈，``JavaScript``这是要大一统的节奏啊，想想也是醉了。所以，是时候学一波``JavaScript``了。

虽然之前基本没怎么用过``JavaScript``，但是也学过像``python``这种动态语言，说实话，学完``JavaScript``就有一种感觉这是tm什么玩意儿的感觉，可以说非常怪异，也可以说非常``magic``，用一个词形容感觉特别合适，喜欢NBA的童鞋应该深有感触，妖刀--GinoBili。真的是太妖了。幸好有最新的标准``ECMAScript 6``（以下简称ES6）写起来还能轻松点，不然真的坑太多了。下文会对比着说。

# 数据类型

``JavaScript``中一个有5种基本数据类型：

 1. 数字，包括整数和浮点和NaN(not a number)
 2. 字符串
 3. 布尔值
 4. undefined：未声明或者声明了却未赋值
 5. null：空值，与undefined的区别在于这是一个已经声明的变量

``JavaScript``，并且任何不属于基本数据类型的东西都是对象。
数组，``Map``什么的就不写了。

# 变量

说到变量我真是喷出一口老血，太特么容易坑了。
由于是动态语言，所以不需要指定变量的类型，可以在运行时绑定。声明变量用var(variable)。

## 作用域

为什么说容易坑呢，先看个列子
```javascript
var a = [];
for (var i = 0; i < 10; i++) {
  a[i] = function () {
    b = 'f**k';
    console.log(i);
  };
}
console.log(b);// f**k
a[6](); // 10
```
竟然打印了10。法克。
用``var``声明的变量的作用域是函数级别的，也就是说在函数内部有效，不同于``java``等静态语言变量声明是块级别的。由于for语句属于函数，所以变量``i``的作用域就是整块代码。
那为啥会打印出``f**k``？如果不用``var``声明变量，那么默认就是全局变量，也就是说``b``其实是个全局变量..醉了。``JavaScript``中有一种严格模式，在JavaScript代码的第一行写上：``'use strict';``，就会强制通过``var``声明变量，避免发生错误。
另外一种办法就是，``ES6``新增了``let``命令用来声明变量，该变量的作用域是块级别的，把上面的例子``var``改为``let``最终输出就是变成6。

## 变量提升

又为啥这么说坑呢，看例子
```javascript
var v='Hello World';
(function(){
    alert(v);
    var v='I love you';
})()
```
结果竟然是``undefined``，再法克。
上面函数的意思是匿名函数立即执行的写法。
``JavaScript``的函数定义有个特点，它会先扫描整个函数体的语句，把所有申明的变量“提升”到函数顶部。所以上面代码在``JavaScript``引擎看来是这样的：
```javascript
var v='Hello World';
(function(){
    var v;
    alert(v);
    v='I love you';
})()
```
那有什么办法解决呢？
用``let``，用``let``，用``let``。

# 坑爹的this

一般正常的面向对象的语言``this``就是指向对象本身，而``JavaScript``很坑爹，``this``指向视情况而定，用好了是指向对象本身，用不好就指向全局对象(非strict模式)或者指向``undefined``(strict模式)，全局对象是指``web``中是``window``，``Node.js``中是``global``。

 - 指向对象本身
    1. ``obj.fuc()``; 
    2. 由于``JavaScript``中函数也是对象，调用函数对象的``call()``、 ``apply()``，第一个参数传入要绑定的``this``对象。
 - 指向全局对象或者``undefined`` 
    1. 没有绑定对象
    2. 间接调用方法 ``var a = obj.func(), a();``
    3. 方法中返回闭包，闭包中使用了``this``
 
总之，``this``坑很多，能不用最好不用。
 
# 闭包

闭包(``closure``)是一种包含了外部函数的参数和局部变量的返回函数。换句话说，闭包就是携带状态的函数，并且它的状态可以完全对外 隐藏起来。

```javascript
function make_pow(n) {
    return function (x) {//这是一个闭包
        return Math.pow(x, n);
    }
}
// 创建两个新函数:
let pow2 = make_pow(2);
let pow3 = make_pow(3);
pow2(5); // 25
pow3(7); // 343
```

Android开发中常用的回调就是一种闭包，只不过是用对象方法的方式表达，而``JavaScript``中函数也是一种对象，所以无需多余的对象引用。
类似``Java``的``lambda``表达式，ES6中可以用箭头函数定义匿名函数：

```javascript
// 两个参数:
(x, y) => x * x + y * y

// 无参数:
() => 3.14

// 可变参数:
(x, y, ...rest) => {
    var i, sum = x + y;
    for (i=0; i<rest.length; i++) {
        sum += rest[i];
    }
    return sum;
}
```
 
# 面向对象
 
``JavaScript``是一种面向对象的语言，刚才已经说了除基本数据类型外，所有的东西都是对象，但是又跟正常的面向对象语言不一样。像``Java``、``C++``这种大多数面向对象语言，类和实例是面向对象的基础，而``JavaScript``不区分类和实例的概念，而是通过原型（``prototype``）来实现面向对象编程。

``JavaScript``对每个创建的对象都会设置一个原型，指向它的原型对象。并且有一个属性查找原则，当我们用``obj.xxx``访问一个对象的属性时，``JavaScript``引擎先在当前对象上查找该属性，如果没有找到，就到其原型对象上找，如果还没有找到，就一直上溯到``Object.prototype``对象，最后，如果还没有找到，就只能返回``undefined``。

## 创建对象

``JavaScript``面向对象基于原型实现，这就导致其用法比较灵活，也足够简单，缺点就是比较难理解，容易出错。下面是几种创建对象的方法。

### 直接用``{ ... }``创建一个对象
``Student``就是一个对象
```javascript
let Student = {
name: 'xiaoming',
age: 19,
run: function () {
    console.log(this.name + ' is running...');
    }
}
```
原型链是这样的：
```
Student ----> Object.prototype ----> null
```
``JavaScript``的原型链和``Java``的``Class``区别就在，它没有“Class”的概念，所有对象都是实例，所谓继承关系不过是把一个对象的原型指向另一个对象而已。

### ``Object.create()``
传入一个原型对象作为参数，并创建一个基于该原型的新对象，但是新对象什么属性都没有

```javascript
function createStudent(name) {
    // 基于Student原型创建一个新对象:
    var s = Object.create(Student);
    // 初始化新对象:
    s.name = name;
    return s;
}

var xiaoming = createStudent('小明');
xiaoming.run(); // 小明 is running...
xiaoming.__proto__ === Student; // true
```

### 构造函数
``JavaScript``的构造函数就是普通函数

```javascript
function Student(name) {
    this.name = name;
    this.hello = function () {
        console.log('Hello, ' + this.name + '!');
    }
}

let xiaoming = new Student('小明');
xiaoming.name; // '小明'
xiaoming.hello(); // Hello, 小明!
let xiaohong = new Student('小红');
xiaohong.name;//'小红'
xiaohong.hello();//Hello,小红!
```
原型链是这样的
```
xiaoming ↘
xiaohong -→ Student.prototype ----> Object.prototype ----> null
xiaojun  ↗
```
也就是说，``xiaoming``的原型指向函数``Student``的原型。验证一下
```javascript
xiaoming.constructor === Student.prototype.constructor; // true
Student.prototype.constructor === Student; // true
Object.getPrototypeOf(xiaoming) === Student.prototype; // true
xiaoming instanceof Student; // true
xiaoming.hello === xiaohong.hello; // false
```

### 封装构造函数，以对象作为初始化参数

```javacsript
function Student(props) {
    this.name = props.name || 'noname'; // 默认值为'noname'
    this.grade = props.grade || 1; // 默认值为1
}

Student.prototype.hello = function () {
    alert('Hello, ' + this.name + '!');
};

function createStudent(props) {
    return new Student(props || {})
}

let xiaoming = createStudent({
    name: '小明'
});
xiaoming.name; //小明
xiaoming.grade; // 1
```
这个``createStudent()``函数有几个巨大的优点：一是不需要new来调用，二是参数非常灵活，可以不传，也可以传一个对象。

## 原型继承

一张图看懂上面的关系
![原型继承](http://7xs23g.com1.z0.glb.clouddn.com/prototype.png)
``xiaoming``的原型指向``Student``的``prototype``对象，这个原型对象有个``constuctor``属性，指向``Student()``函数本身。

上面可以看到``xiaoming.hello() !== xiaohong.hello()``，各自的``hello()``函数实际上是两个不同的函数，如果我们要创建共享的``hello``函数，根据属性查找原则，只需要把函数定义在他们共同所指的原型对象上来就可以了。
```javascript
Student.prototype.hello = function () {
    alert('Hello, ' + this.name + '!');
};
```

那么假如我们想从``Student``扩展出``PrimaryStudent``，使得新的基于``PrimaryStudent``创建的对象不但能调用``PrimaryStudent.prototype``定义的方法，也可以调用``Student.prototype``定义的方法。也就是说原型链是这样的
```
new PrimaryStudent() ----> PrimaryStudent.prototype ----> Student.prototype ----> Object.prototype ----> null
```
那要怎么做呢？
我们可以定义一个空函数``F``，用于桥接原型链，并将其封装起来，隐藏``F``的定义，代码如下
```javascript
function inherits(Child, Parent) {
    let F = function () {};//空函数F
    // 把F的原型指向Student.prototype:
    F.prototype = Parent.prototype;
    // 把Child的原型指向一个新的F对象，该F对象的原型正好指向Parent.prototype
    Child.prototype = new F();
    // 把Child的原型的构造函数修复为Child
    Child.prototype.constructor = Child;
}
```
使用
```javascript
function PrimaryStudent(props) {
    Student.call(this, props);//调用Student的构造方法，并绑定this
    this.grade = props.grade || 1;
}
// 实现原型继承链:
inherits(PrimaryStudent, Student);

// 继续在PrimaryStudent原型（就是new F()对象）上定义方法： 
PrimaryStudent.prototype.getGrade = function () {
    return this.grade;
};
```
原型链图如下
![继承](http://7xs23g.com1.z0.glb.clouddn.com/prototype_extends.png)

## 类继承

说实话，你让我写原型继承，我的内心其实是拒绝的，这特么都是些什么啊乱七八糟的，继承要写这么多，而且很容易出错有没有！那么有没有类似``Java``这种类继承的方式呢，答案是当然有，ES6早就为我们准备好了。
ES6中增加了新的关键字``class``用于定义类。``extends``用于实现类的继承。
```javascript
class Student {
    constructor(name) {
        this.name = name;
    }
    // 相当于Student.prototype.hello = function () {...}
    hello() {
        alert('Hello, ' + this.name + '!');
    }
}
```
创建对象的方式跟原型继承一样，``new``就可以了。
继承：
```javascript
class PrimaryStudent extends Student {
    constructor(name, grade) {
        super(name); // 记得用super调用父类的构造方法!
        this.grade = grade;
    }

    myGrade() {
        alert('I am at grade ' + this.grade);
    }
}
```
是不是炒鸡简单，跟``Java``的类继承基本一模一样。这样写跟原型继承的写法在``JavaScript``引擎看来完全一样。

# 总结

这篇基本描述了``JavaScript``的一些注意容易踩坑的点和与``Java``面向对象实现不同的点。像一些基础的集合、字符串等都没写。当然还有一些前端用的比较多的比如``DOM``、``AJAX``、``jQuery``就不写了，大概浏览下就好了。
另外一点要说的就是，对于我们这些非前端工程师来说，ES6中定义了的一律用ES6的，而且像``Node.js``这种脱离了浏览器引擎的框架已经完全支持ES6的写法。
学完``JavaScript``就得学``Node.js``啊。下一篇带来``Node.js``基础和实战。

# 参考

《JavaScript面向对象编程指南(第二版)》
[阮一峰的网络日志][1]
[廖雪峰 JavaScript教程][2]
[JavaScript秘密花园][3]


  [1]: http://www.ruanyifeng.com/blog/javascript/
  [2]: http://www.liaoxuefeng.com/wiki/001434446689867b27157e896e74d51a89c25cc8b43bdb3000
  [3]: http://bonsaiden.github.io/JavaScript-Garden/zh/