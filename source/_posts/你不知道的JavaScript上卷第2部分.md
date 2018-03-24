---
title: 你不知道的JavaScript上卷 第二部分
date: 2018-03-19 16:06:19
categories: 
- 读书笔记
tags: 
- books
- 你不知道的JavaScript
---

# 你不知道的JavaScript上卷 第二部分 this和对象原型
## 第一章 关于this
### 1.1 为什么要用this
this提供了一种更优雅的方式来隐式传递一个对象引用。

###1.2 误解
#### 1.2.1 指向自身
除了函数对象，还有更多**更适合**存储状态的地方。
**函数表达式需要引用自身时（比如使用递归，或者定义自己的属性和方法），应当要给函数表达式具名，而不是使用arguments.callee。**

#### 1.2.2 它的作用域
**this在任何情况下都不指向函数的词法作用域。**

### 1.3 this到底是什么
this是在运行时绑定的，而不是在编写时绑定，它的上下文取决于函数调用时的各种条件。this只取决于函数的调用方式。


## 第二章 this全面解析
### 2.1 调用位置
调用位置是函数在代码中被调用的位置，最重要的是要分析调用栈。
比如
```JavaScript
function baz(){
    // 当前调用栈是baz，调用位置是baz
    console.log(this); // window || global
    bar();
}

function bar(){
    // 当前调用栈是baz -> bar，调用位置在baz中
    // 可以拿到baz词法作用域的变量，但是和this无关
    console.log(this); // window || global
    foo();
}

function foo(){
    // 当前调用栈是baz -> bar -> foo，调用位置在bar中
    // 可以拿到bar和baz词法作用域的变量，但是和this无关
    console.log(this) // window || global
}
baz(); // baz的调用位置
```
### 2.2 绑定规则
#### 2.2.1 默认绑定
this默认指向全局对象。
严格模式下，如果this找不到具体对象，就是undefined。
但如果函数声明的区域不是严格模式的，即使在严格模式下调用，也不影响它将默认的this指向全局。

#### 2.2.2 隐式绑定
this隐式绑定的，是整个调用链上，处于方法上一层的对象。
```JavaScript
function foo(){
    console.log(this.a)
}

var obj = {
    a: 2,
    foo: foo,
};

obj.foo(); //2
//其实等效于，JS的函数调用其实更像是一种语法糖
obj.foo.call(obj);
```
需要特别注意的是，回调函数丢失this绑定是非常常见的，因为调用它的操作通常不包含隐式绑定和显式绑定。
这个时候可以选择：
- ES6的lambda表达式
- 从外部提供`var _this = this`变量来注入进去
- bind显式绑定

#### 2.2.3 显式绑定
call和apply的效果一样，传入参数的方式不一样。
call从第二个参数开始，逐个传入。
apply的第二个参数是一个Array，参数放在Array里。
bind的参数形式类似于call，但是返回的是显式绑定后的函数对象，需要后续去调用执行。
bind常用于柯里化一个函数对象。
```JavaScript
// ES6写法
Function.prototype._bind = function(thisArg, ...args){
    return (...newArgs) => this.call(thisArg, ...args, ...newArgs);
}

// ES5写法
Function.prototype._bind = function(thisArg){
    var args = [].slice.call(arguments, 1);
    var fn = this;
    return function(){
        var newArgs = [].slice.call(arguments, 0);
        return fn.apply(thisArg, args.concat(newArgs));
    };
}

//MDN提供的polyfill
if(!Function.prototype.bind) {
    Function.prototype.bind = function(oThis) {
        // 类型判断
        if(typeof this !== 'function'){
            throw new TypeError('Function.prototype.bind - what is trying to be found is not callable');
        }
        
        var aArgs = Array.prototype.slice.call(arguments, 1),
            fToBind = this,
            fNOP = function(){},
            fBound = function(){
                return fToBind.apply((
                    this instanceof fNOP && oThis? this: oThis
                ),
                aArgs.concat(Array.prototype.slice.call(arguments))
                );
            };
        
        // 继承原型链
        fNOP.prototype = this.prototype;
        fBound.prototype = new fNOP();

        return fBound;
    }
}
```
#### 2.2.4 new绑定
用new来调用函数，会自动执行下面的操作：
1. 创建（或者说构造）一个全新的对象。
2. 这个新对象会被执行[[Prototype]]链接。
3. 这个新对象会绑定到函数调用的this。
4. 如果函数没有返回其他对象，那么new表达式中的函数调用会自动返回这个新对象。

### 2.3 优先级
- new在调用构造函数的时候做初始绑定。
- 显式绑定或硬绑定优先级最高。
- 没有显式绑定，就使用隐式绑定。
- 如果都没有，就使用默认绑定（undefined，是不是全局看严格模式）。

### 2.4 绑定例外
#### 2.4.1 被忽略的this
显式绑定中，如果对this不敏感，可以传入`null`，但是可能有副作用。
更安全的做法是，传入一个空对象，即`var empty = Object.create(null)`，它连指向Object.prototype的__proto__都没有，比{}更空。

#### 2.4.2 间接引用
`(p.foo = o.foo)()`这里返回的引用是foo，而不是p.foo，它的this指向会指向undefined。

### 2.5 this词法
lambda表达式（箭头函数）的this是根据外层作用域来决定。
lambda表达式常用于回调函数，比如eventListener和setTimeout操作中。
```JavaScript
function foo() {
    // 返回一个箭头函数
    return (a) => {
        // 这里的this其实是foo中的this
        console.log(this.a);
    };
}

var obj1 = {a:2},
    obj2 = {a:3};

// bar的this指向foo的this，foo的this此时显式绑定了obj1
var bar = foo.call(obj1);
// 虽然显式绑定了bar的this，但是这种方法对箭头函数不起作用
// 它的this依然指向先前foo被显式绑定的obj1
bar.call(obj2); //2
```

## 第三章 对象
### 3.1 语法
```JavaScript
//文字语法（创建简单对象时推荐使用）
var myObj = {
    key: value,
};

//构造形式
var myObj = new Object();
myObj.key = value;
```

### 3.2 类型
JS一共有七种主要类型（语言类型）：
- string
- number
- boolean
- undefined
- null
- object
- symbol

内置对象（注意第一个字母大写，这些都是构造函数）：
- String
- Number
- Boolean
- Object
- Function
- Array
- Date
- RegExp
- Error

### 3.3 内容
对象的内容是由一些存储在特定命名位置的值组成的，我们称之为属性。
存储在对象容器内部的是这些属性的名称，它们就像指针一样，指向这些值真正的存储位置。
`.key`方式称为属性访问，`['key']`方式称为键访问。
属性访问更直接和直观，但是其属性名有一定的规范；键访问允许传入动态的字符串变量来访问，只要是字符串即可，更灵活。
#### 3.3.1 可计算属性名
ES6支持在键访问中传入一个表达式来当作属性名，比如`myObj[prefix + 'foo']`。

#### 3.3.2 属性与方法
属于对象的函数通常称为方法。

#### 3.3.3 数组
数组通过数字下标访问。
数组可以添加命名属性，但是不会改变其length值。

#### 3.3.4 复制对象
对于JSON安全的对象来说，可以使用`var cloned = JSON.parse(JSON.stringify(target))`的方式。
ES6中可以使用Object.assign方法
```JavaScript
const cloned = Object.assign(
        // 生成原型链
        Object.create(Object.getPrototypeOf(target)), 
        target
    );
```
需要注意的是，PropertyDescriptor不会被按原样复制，而是保持默认值。

#### 3.3.5 属性描述符
一个descriptor有6个可能的属性，同时只能拥有其中4个（分为基本类型和引用类型）：
- value，属性值（只能用于基本类型）
- writable，是否可以写入，false表示readonly，默认true（只能用于基本类型）
- get，getter方法（只能用于引用类型）
- set，setter方法（只能用于引用类型）
- configurable，descriptor是否可以修改（或者是删除），默认true
- enumerable，是否为可枚举属性，关系到能否出现于for...of、for...in等操作中，默认true

#### 3.3.6 不变性
1. 结合`writable: false`和`configurable: false`就可以创建一个真正的常量属性。
```JavaScript
var myObject = {};

Object.defineProperty( myObject, 'Favorite_Number', {
    value: 22,
    writable: false,
    configurable: false,
});
```
2. 禁止扩展，使用Object.preventExtensions，不能再添加新属性。
3. 密封，使用Object.seal，只能修改值，不能添加/删除或重新配置属性。
4. 冻结，使用Object.freeze，在seal基础上在将属性改为`writable: false`，只读。

#### 3.3.7 \[\[Get]]
当给对象进行一次属性访问时，其实是使用了\[\[Get]]操作。
对象默认的Get操作首先在对象中查找是否有名称相同的属性，如果找到就返回；没有找到，就会去原型链上寻找。

#### 3.3.8 \[\[Put]]
如果属性已经存在，Put会检查：
1. 属性是否是访问描述符，如果是，并且有setter，就调用setter。
2. 属性的数据描述符中writable是否为false？如果为false，非严格模式下静默失败，严格模式抛出错误。
3. 如果都不是，将该值设置为属性的值。

#### 3.3.9 Getter和Setter
定义方式
```JavaScript
// 直接式
var myObject = {
    // 此时再去访问a，只会获得2
    get a(){
        return 2;
    },
    // 不是单纯的setter，而会取两倍的值
    set a(val){
        this._a_ = val * 2;
    }
}

// descriptor式
Object.defineProperty(myObject, 'b', {
    get: function(){
        return this._b_;
    },
    set: function(val){
        this._b_ = val;
    },
    enumerable: true,
});
```

#### 3.3.10 存在性
`in`操作符会检查属性是否在对象及其原型链中。需要注意的是，`in`只检查键是否存在，比如在Array中，并不会检查Array的内容，而是检查下标。
hasOwnProperty只会检查属性是否在对象中，不会检查原型链。

#### 3.3.11 可枚举性
可枚举，相当于可以出现在对象属性的遍历中。
propertyIsEnumerable会检查给定的属性名是否直接存在于对象中，并且满足enumerable: true。

### 3.4 遍历
- for...in：遍历所有可遍历的属性名（不是值）。
- for...of：遍历所有可遍历属性的值。
- foreach，every，some，map，filter，reduce等委托的原型方法则是将遍历改为链式调用。

使用Symbol.iterator对象，可以用来定义对象的@@iterator内部属性：
```JavaScript
var myObject = {
    a: 2,
    b: 3,
};

Object.defineProperty(myObject, Symbol.iterator, {
    enumerable: false,
    writable: false,
    configurable: true,
    value: function() {
        var o = this;
        var idx = 0;
        var ks = Object.keys(o);
        return {
            next: function() {
                return {
                    value: o[ks[idx++]],
                    done: (idx > ks.length),
                };
            },
        };
    }
});

var it = myObject[Symbol.iterator]();
it.next(); // {value:2, done: false}
it.next(); // {value:3, done: false}
it.next(); // {value:undefined, done: true}
```


## 第四章 混合对象“类”
面向类的设计模式：实例化(instantiation)，继承(inheritance)，（相对）多态(polymorphism)。
### 4.1 类理论
类理论包括：
- 类
- 继承
- 实例化
- 多态，父类的通用行为可以被子类用更特殊的行为重写（JS中不推荐）。

#### 4.4.1 类设计模式
面向对象的设计模式有，**迭代器模式、观察者模式、工厂模式、单例模式**等。

#### 4.4.2 JavaScript中的类
JS中的类是构造函数与原型的结合，与其他语言的类不同。

### 4.2 类的机制
#### 4.2.1 建造
为了获得真正可以交互的对象，我们必须按照类来建造一个东西，这个东西通常被称为实例。有需要的话，我们可以直接在实例上调用方法并访问其所有公有数据属性。

#### 4.2.2 构造函数
类实例是由一个特殊的类方法构造的，这个方法名和类名相同，被称为构造函数。**这个方法的任务是初始化实例所需要的所有信息及属性。**
```JavaScript
function CoolGuy() {
    this.name = 'John';
}
```
构造函数需要用new来调用，这样引擎才会去构造一个新的类实例。

### 4.3 类的继承
子类需要继承父类的属性与原型方法。

#### 4.3.1 多态
子类可以重写父类方法。
子类重写的方法在子类的prototype上，不会改变父类的prototype，只是子类在寻找方法时，会跟随原型链，率先找到处于自身原型上的方法，从而调用。
ES6用super代指父类的构造函数，从而能让子类通过这种方式找到父类的原型及其方法。

#### 4.3.2 多重继承
JS的继承是单向的。
JS不支持多重继承，因此使用混入模式实现多重继承。

### 4.4 混入
#### 4.4.1 显式混入
```JavaScript
// 不覆盖属性
function mixinWithoutOverwrite(source, target) {
    for (var key in source){
        // 只在不存在的情况下复制
        if( !(key in target) ){
            target[key] = source[key];
        }
    }

    return target;
}


//覆盖属性
function mixinWithOverwrite(source, target) {
    for (var key in source){
        target[key] = source[key];    
    }

    return target;
}
```
另一种是寄生式继承，将父类的实例混入子类。
```JavaScript
function Super() {
    this.name = 'father';
}
Super.prototype.say = function() {
    console.log('Method from father');
}

function Sub() {
    // 父类实例
    var father = new Super();
    father.name = 'son';
    // 保存引用
    var extendedSay = father.say;
    // 多态重写
    father.say = function() {
        extendedSay.call(this);
        console.log('Method from son');
    }
    return father;
}
// 调用时无需用new
var son = Sub();
```

#### 4.4.2隐式混入
隐式混入其实就是把其他对象的方法拿过来使用，使用call、apply、或bind，将其this指针指向自身。
```JavaScript
var landLord = {
    name: 'himself',
    checkMoney: function() {
        console.log('Landlord is giving money to ' + this.name);
    },
}

var robinHood = {
    name: 'Robin Hood',
    rob: function() {
        // 把其他对象的方法拿过来使用
        landLord.checkMoney.call(this);
    },
}

robinHood.rob();
```


## 第五章 原型
### 5.1 \[\[Prototype]]
对于默认的Get操作来说，如果无法在对象本身找到需要的属性或方法，就会继续访问对象的Prototype链。
```JavaScript
var another = {
    a:2
};
var myObj = Object.create(another);
// 能找到，但是在myObj.__proto__上找到
myObj.a; //2
```
使用for...in遍历对象的原理和查找原型链相似。

#### 5.1.1 Object.prototype
**一般原型链都会最终指向Object.prototype**，而**Object.prototype的__proto__为null**。

#### 5.1.2 属性设置和屏蔽
如果原型链上找不到某个属性，该属性就会被直接添加到调用对象上。
如果属性既出现在对象中，也出现在其原型链中，那么对象中的属性就会屏蔽原型链上层所有的同名属性。
优先顺序为；**对象实例属性 > 下层原型 > 上层原型**。
Set的情况则更为复杂，比如myObj自身没有foo，但`myObj.foo = 'bar'`的情况：
1. 如果原型链上次存在名为foo的属性，且可以写入，那么就会直接在myObj中添加一个名为foo的新属性，并且赋值进去。
2. 如果原型链上层存在foo，但它是只读，那么就无法进行修改，并抛出错误。
3. 如果原型链上层存在foo，且是一个setter，那么就会调用这个setter。此时foo不会添加到myObj，也不会改变这个setter。

### 5.2 类
#### 5.2.1 类函数
所有函数默认都会拥有一个名为prototype的公有且不可枚举属性，它其实指向另一个对象。
同一个构造函数，及其构造的实例，它们对该函数prototype的引用其实指向同一段内存地址，修改这个prototype的属性及方法，会影响到这个构造函数及其所有实例。

#### 5.5.2 构造函数
prototype默认有一个公有且不可枚举的属性constructor，这个属性引用的是对象关联的函数。
```JavaScript
function foo() {
    // ...
}
foo.prototype.constructor === foo; // true
var a = new foo();
// 并不是a有这个属性，它是在原型链上找到的
a.constructor === foo; // true
```
需要注意的是，构造函数本身还是函数，new操作符只是将其调用方式变成“构造函数调用”，本质上还是需要去执行它。

#### 5.2.3 技术
prototype中的constructor引用是非常不可靠，且不安全的。

### 5.3 原型继承
注意`instanceof`操作符，其实就是检查左边对象的原型链上是否存在右边构造函数的原型。
反过来操作就是`Foo.prototype.isPrototypeOf(a)`。
```JavaScript
//ES5
function Super(){
    this.name = 'father';
}
Super.prototype.say = function() {
    console.log('I am ' + this.name);
};

function Sub(){
    // 执行父类构造函数，获得属性
    Super();
    // 添加或覆盖属性
    this.name = 'son';
}
// 形成原型链
Sub.prototype = Object.create(Super.prototype);
// 修复constructor指向
Sub.prototype.constructor = Sub;

// 或者更直接的原型链
Sub.prototype.__proto__ = Super.prototype;
//ES6提供一种新的操作方式
Object.setPrototypeOf(Sub.prototype, Super.prototype);


// ES6
class A {
    constructor(){
        this.name = 'father'
    }
    say() {
        console.log('Method from father: I am ' + this.name);
    }
}
class B extends A {
    constructor(){
        super();
        this.name = 'son'
    }
    say() {
        console.log(`Method from son: I am ${this.name}.`);
        super.say();
    }
}
```
另一个需要注意的是非标准的__proto__，它其实是一套getter/setter。
```JavaScript
Object.defineProperty(Object.prototype, "__proto__", {
    get: function() {
        return Object.getPrototypeOf(this);
    },
    set: function(o) {
        Object.setPrototypeOf(this, o);
        return o;
    },
});
```

### 5.4 对象关联
#### 5.4.1 创建关联
```JavaScript
//Object.create()的polyfill
if(!Object.create){
    Object.create = function(o) {
        function F(){}
        F.prototype = o;
        return new F();
    };
}
```
#### 5.4.2 关联关系是备用？
原型的实现应当遵循委托设计模式，API在原型链上出现的位置要参考现实中的情况。


## 第六章 行为委托
### 6.1 面向委托的设计
#### 6.1.1 类理论
类设计模式鼓励你在继承时使用方法重写和多态。许多行为可以先抽象到父类，然后再用子类进行特殊化。

#### 6.1.2 委托理论
首先定义对象，而不是类。通过Object.create()来创建委托对象，赋值给另一个对象，从而让该对象出现在另一个对象的原型链上。
比如`var bar = Object.create(foo);`，使得`bar.__proto__===foo`，从而bar可以通过原型链获得foo的所有属性和方法。
这种设计模式被称为对象关联。委托行为意味着某些对象在找不到属性或者方法引用时，会把这个请求委托给另一个对象。
需要注意的是，**禁止两个对象互相委托**，否则当引用一个两者都不存在的属性或方法，会产生无限递归的循环。

#### 6.1.3 比较思维模型
对象关联风格相对于类风格更为简洁，因为它只关注对象之间的关联关系。
```JavaScript
Foo = {
    init: function(who) {
        this.me = who;
    },
    identify: function() {
        return 'I am ' + this.me;
    },
}
Bar = Object.create(Foo);
Bar.speak = function() {
    console.log('Hello, ' + this.identify() + '.');
};

var b1 = Object.create(Bar);
var b2 = Object.create(Bar);
// 从b1.__proto__.__proto__上得到init方法
b1.init('b1');
b2.init('b2');

// 从b1.__proto__上得到Bar的spaak方法
// 从b1.__proto__.__proto__上得到identify方法
b1.speak(); // Hello, b1.
b2.speak(); // Hello, b2.
```

### 6.2 类与对象
#### 6.2.1 控件“类”
```JavaScript
// 使用jQuery
// 父类
function Widget(width, height) {
    this.width = width || 50,
    this.height = height || 50;
    this.$elem = null;
}
Widget.prototype.render = function($where) {
    if(this.$elem) {
        this.$elem.css({
            width: this.width + 'px',
            height: this.height + 'px',
        }).appendTo($where);
    }
};

//子类
function Button(width, height, label) {
    Widget.call(this, width, height);
    this.label = label || 'default';
    this.$elem = $('button').text(this.label);
}
// 对象关联法建立继承
Button.prototype = Object.create(Widget.prototype);
//重写render，但不完全替换，而是在原有基础上添加一点功能
Button.prototype.render = function($where) {
    Widget.prototype.render.call(this, $where);
    this.$elem.click(this.onclick.bind(this));
};
Button.prototype.onclick = function(ev) {
    console.log('Button' + this.label + ' clicked!');
};

$(document).ready(function() {
    var $body = $(document.body);
    var btn1 = new Button(125, 30, 'Hello');
    var btn2 = new Button(150, 40, 'World');

    btn1.render($body);
    btn2.render($body);
});
```

#### 6.2.2 委托控件对象
委托模式可以防止上一节中不太合理的伪多态形式调用`Widget.prototype.render.call(this, $where);`。
同时init和setup两个初始化函数更具语义表达能力，同时由于将构建和初始化分开，使得初始化的时机变得更灵活，允许异步调用。
对象关联可以更好地支持**关注分离原则，创建和初始化不需要合并为一个步骤**。
```JavaScript
var Widget = {
    init: function() {
        this.width = width || 50,
        this.height = height || 50;
        this.$elem = null;
    },
    insert: function($where){
        if(this.$elem) {
            this.$elem.css({
                width: this.width + 'px',
                height: this.height + 'px',
            }).appendTo($where);
        }
    },
};

// Widget加载到Button的__proto__上
var Button = Object.create(Widget);
Button.setup = function(width, height, label) {
    //委托调用
    this.init(width, height);
    this.label = label || 'default';
    this.$elem = $('button').text(this.label);
};
Button.build = function($where) {
    this.insert($where);
    this.$elem.click(this.onclick.bind(this));
}
Button.onclick = function(ev) {
    console.log('Button' + this.label + ' clicked!');
};

$(document).ready(function() {
    var $body = $(document.body);

    var btn1 = Object.create(Button);
    btn1.setup(125, 30, 'Hello');
    var btn2 = Object.create(Button);
    btn2.setup(150, 40, 'World');

    btn1.build($body);
    btn2.build($body);
});
```

### 6.3 更简洁的设计
对象关联除了能让代码看起来更简洁，更具扩展性，还可以通过委托模式简化代码结构。

### 6.4 更好的语法
ES6中使用`Object.setPrototypeOf(target, obj)`的方式来简化`target = Object.create(obj)`的写法。

### 6.5 内省
内省就是检查实例的类型。
`instanceof`实际上检查的是目标对象与构造器原型的关系，对于通过委托互相关联的对象（Foo与Bar互相关联），可以使用：
- `Foo.isPrototypeOf(Bar)`
- `Object.getPrototypeOf(Bar) === Foo`
