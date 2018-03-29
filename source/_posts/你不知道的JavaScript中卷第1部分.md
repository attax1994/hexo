---
title: 你不知道的JavaScript中卷 第一部分
date: 2018-03-19 16:06:19
categories: 
- 读书笔记
tags: 
- 你不知道的JavaScript
---

# 你不知道的JavaScript中卷 第一部分
## 第一章 类型
### 1.2 内置类型
使用typeof检测对象类型并不是非常安全的行为：
```JavaScript
// 安全的
typeof undefined    // 'undefined'
typeof true         // 'boolean'
typeof 42           // 'number'
typeof '42'         // 'string'
typeof {}           // 'object'
typeof Symbol()     // 'symbol'
typeof (function(){}) // 'function'
// 不安全的
typeof null         // 'object'
typeof []           // 'object'
```
函数对象可以拥有属性。函数的length属性为其声明的参数个数。

### 1.3 值和类型
### 1.3.1 undefined和undeclared
声明但还没有赋值的变量是undefined。
还没有在作用域声明过的变量是undeclared（严格模式下访问会抛出ReferenceError）。
使用`typeof`去检测undeclared的变量，可以防止报错，返回`'undefined'`。或者当其为全局变量的时候，用window的点操作符去访问。


## 第二章 值
### 2.1 数组
在稀疏数组中，中间未赋值的内容会初始化为undefined
```JavaScript
var a = [];
a[0] = 1;
a[2] = 3;

a[1]; // undefined
a.length; // 3
```
也可以用键访问（输入不能转换为数字的字符串）去设置数组的属性，但他们不会纳入数组的值和length，而是作为独立的属性存在。
类数组（Array Like）对象表示其类型类似于数组，但是原型链上没有Array对象，自然也无法直接使用Array的原型方法。
转换类数组使用ES5的`Array.prototype.slice.call(arrayLikeObj)`，或者ES6的`Array.from(arrayLikeObj)`。

### 2.2 字符串
JS中字符串的单个字符不可变，不能用键访问去改变字符串的某个字符。
字符串也可以借用数组的一些原型方法来进行处理。一个常用场景是字符串的倒置：
```JavaScript
var c = a
        // 先转为Array
        .splite('')
        // 再倒置Array
        .reverse()
        // 最后转回字符串
        .join('');
```

### 2.3 数字
直接对数字字面量调用原型方法，有时候需要两个点（.），这是因为第一个点会被理解为小数点。
Number.prototype上有这些方法：
- toExponential()，转为指数格式
- toFixed()，指定小数位数
- toPrecision()，指定有效位数

其他进制的问题：
- 0xf3，16进制
- 0b11110011，2进制
- 0o363，8进制

#### 2.3.2 较小的数值
误差范围值，即机器精度（machine epsilon）。
使用机器精度，可以判断两个浮点值的运算结果是否等于另一个数字。
`Math.abs(n1 - n2) < Number.EPSILON`

#### 2.3.3 整数安全范围
安全的最大整数是`2^53 - 1`，即9007199254740991，ES6中为Number.MAX_SAFE_INTEGER。同理，最小整数为其负数`-(2^53 - 1)`，ES6中为Number.MIN_SAFE_INTEGER。

#### 2.3.4 整数检测
- Number.isInteger，检测是否为整数
- Number.isSafeInteger，检测是否为安全的整数

#### 2.3.5 32位有符号整数
数位运算符只适用于32位的整数，其他数位将会被忽略。

### 2.4 特殊数值
#### 2.4.1 不是值的值
空值：
- null，指空值，曾经赋值，现在没有值empty value
- undefined，指没有值，从未被赋值，missing value

#### 2.4.2 undefined
通过void运算符可以返回undefined。

#### 2.4.3 特殊的数字
NaN是一个警戒值，表示数学运算没有成功，失败后返回的结果。
Infinity表示无穷数，数学运算结果溢出，
-0表示第一位正负位为负，保留了该信息，后续处理的时候会携带它。

#### 2.4.4 特殊等式
- Object.is(NaN, NaN) === true
- Object.is(+0, -0) === false

### 2.5 值和引用
简单值（即基本类型值），通过**值复制**的方式来赋值/传递。
复合值（对象和函数），总是通过**引用复制**的方式来赋值/传递。
减少引用复制副作用的方法是，传递时候采用复本创建的方式，比如a.slice()。


## 第三章 原生函数
原生函数有：
-  String()
-  Number()
-  Boolean()
-  Array()
-  Object()
-  Function()
-  RegExp()
-  Date()
-  Error()
-  Symbol()

### 3.1 内部属性\[\[Class]]
使用`Object.prototype.toString.call(target)`能精确地返回该目标的类型。

### 3.2 封装对象包装
JS会自动为基本类型值包装一个封装对象。
封装对象可以用来进行强制类型转换，但是一般不推荐直接生成实例。

### 3.3 拆封
使用封装对象的`valueOf`方法可以可以进行拆封。

### 3.4 原生函数作为构造函数
#### 3.4.1 Array
`Array(1,2,3)`与`new Array(1,2,3)`效果一样，它不需要new关键字。
只带一个参数时候，它会被理解为数组长度（length）。
一般不常用构造函数去创建数组，除非进行一个较长数组的初始化`Array(100).fill(true)`，或者`Array.apply(null, {length:3})`。

#### 3.4.2 Object、Function和RegExp
尽量减少对这三个构造函数的使用。
RegExp有时候能在构造复杂表达式时候提供一些便利，但尽可能使用字面表达式创建。

#### 3.4.3 Date和Error
它们只能使用构造函数来创建。

#### 3.4.4 Symbol
符号可以用作属性名。注意**它不能使用new关键字去创建**。
Symbol有一些预定义符号，如Symbol.create，Symbol.iterator等。

#### 3.4.5 原生原型
一些特殊的原型：
- Function.prototype，一个空函数
- RegExp.prototype，一个正则表达式
- Array.prototype，一个数组

原生的原型对于未赋值的变量来说，是很好的默认值（ES6中，默认值可以直接在参数表中设置）。但需要注意后面要重新指定，不能对prototype进行修改。


## 第四章 强制类型转换
### 4.1 值类型转换
显式值类型转换称为类型转换，发生在静态语言的编译阶段。隐式的情况称为强制类型转换，发生在动态语言的运行时。
JS中只存在强制类型转换（运行时中转换），只会返回标量基本类型值，比如字符串、数字和布尔值，不会返回对象或函数。

### 4.2 抽象值操作
#### 4.2.1 toString
- null -> 'null'
- undefined -> 'undefined'
- true/false -> 'true'/'false'
- 极大数，极小数 -> 指数形式表达
- Array -> arr.join(',')
- Object -> obj.toPrimitive

对于JSON安全：包含undefined、function、symbol和循环引用的对象不符合JSON的结构标准，它们在stringify操作中会被忽略，数组中转为null。
**当一个对象JSON不安全时，可以定义其toJSON()方法来返回一个JSON安全的对象**，它在JSON.stringify的时候会隐式调用。
`JSON.stringify(target, replacer?, space?)`的replacer可以是一个字符串数组或函数，用于指定哪些属性需要纳入处理。space指定缩进格式。

#### 4.2.2 ToNumber
- true/false -> 1/0
- undefined -> NaN
- null -> 0
- 对象：调用toPrimitive，首先检查该值有没有valueOf()方法，如有则执行，如没有就使用toString()方法，对最终结果进行强制类型转换。如果两者均不返回基本类型值，抛出TypeError错误。

#### 4.2.3 toBoolean
JS中的假值通常为基本类型：
- undefined
- null
- false
- +0, -0, NaN
- ''（空字符串）

### 4.3 显式强制类型转换
#### 4.3.1 字符串和数字之间的显式转换
Number转String：
- String(a)
- a.toString()

String转Number：
- Number(b)
- Number.parseInt(b) / Number.parseFloat(b)
- +b：注意不要两个+号连用变成自增，必要时中间加空格。

其他应用：
- 日期转数字（毫秒级时间戳）：`var d = new Date(); +d;`，但是更建议使用其getTime()方法。
- ~：JS中的取反操作类似于转换为数字，然后取`-(x+1)`，**常用于将-1转为0**
- |：后面跟一个toInt32后为0的值（0，NaN等），可以起到ToNumber的作用。

#### 4.3.2 显式解析数字字符串
**Number.parseInt()和Number.parseFloat()方法允许字符串中含有非数字字符**，遇到非数字字符就停止解析，返回结果。
对于非字符串，会隐式调用其toString()方法。
```JavaScript
// 得到Infinity，解析为'Infinity',第一个字符I的ASCII码为18，所以返回结果是18
// 十进制下返回NaN
Number.parseInt(Infinity, 19);

// toString()解析成了8e-7，返回8
Number.parseInt(0.0000008) 
```

#### 4.3.3 显式转换为布尔值
Boolean()可以显式转换为布尔值，另一种方法是用!和!!。
`if(...) {}`的判断语句中，如果没有特别声明布尔值，就会调用toBoolean()方法。

### 4.4 隐式强制类型转换
#### 4.4.2 字符串和数字之间的隐式强制类型转换
`+`运算符能用于数字加法，也能用于字符串拼接。它一旦遇到两边有一者不是Number的情况，就会对双方调用toPrimitive()或toString()方法，然后进行字符串拼接。
`-`运算符仅用于数字减法，一旦遇到两边有一者表示Number的情况，就会对双方调用ToNumber()行为，然后计算减法。

#### 4.4.5 ||和&&
它们的返回值是两个操作数中的一个。
`||`：
- 如果前者判断为true，就返回前者，否则返回后者。
- 类似于`a? a:b`。
- 常用作空值合并运算符。

`&&`：
- 如果前者判断为true，就返回后者，否则返回前者。
- 类似于`a? b:a`。
- 常用作守护运算符，前面的表达式为后面把关。比如`if(a){ foo(); }`可以改成`a && foo();`。

#### 4.4.6 Symbol的强制类型转换
Symbol只支持显式强制类型转换。
```JavaScript
var s1 = new Symbol('cool');
String(s1); // 'Symbol(cool)'

s1+ ''; // TypeError
```

### 4.5 宽松相等和严格相等
#### 4.5.2 抽象相等
`==`允许再相等比较中进行强制类型转换，`===`不允许。
特别注意，以下情况用Object.is()修复：
1. NaN不等于NaN。
2. +0等于-0

ES5规范11.9.3.4-5定义：
1. 如果Type(x)是数字，Type(y)是字符串，则返回 x == ToNumber(y) 的结果。
2. 如果Type(x)是字符串，Type(y)是数字，则返回 ToNumber(x) == y 的结果。

ES5规范11.9.3.6-7定义：
1. 如果Type(x)是布尔类型，则返回 ToNumber(x) == y 的结果。
2. 如果Type(y)是布尔类型，则返回 x == ToNumber(y) 的结果。

**可以使用`a == null`来替代`a === undefined || a === null`。**
ES5规范11.9.3.2-3定义：
1. 如果x为null，y为undefined，返回true。
2. 如果x为undefined，y为null，返回true。

ES5规范11.9.3.8-9定义：
1. 如果Type(x)是字符串或数字，Type(y)是对象，则返回 x == ToPrimitive(y)的结果。
2. 如果Type(x)是对象，Type(y)是字符串或数字，则返回 ToPrimitive(x) == y的结果。

特殊情况说明：
```JavaScript
'0' == false; // true, Number('0') == Number(false)
false == 0;   // true, Number(false) == 0
false == '';  // true, Number(false) == Number('')
false == [];  // true, Number(false) == ToPrimitive([])
'' == 0;      // true, Number('') == 0
'' == [];     // true, Number('') == ToPrimitive([])
0 == [];      // true, 0 == ToPrimitive([])
```

### 4.6 抽象关系比较
比较双方首先调用ToPrimitive，如果出现非字符串，就根据ToNumber规则将双方转换为数字来比较。如果比较双方都是字符串，就按字母顺序来逐位比较。
需要注意的是，`a <= b`在JS中被处理成`!(b < a)`。


## 第五章 语法
### 5.1 语句和表达式
JS中，表达式可以返回一个结果值。

#### 5.1.1 语句的结果值
语句都有一个结果值（undefined也算）。
对于代码块，其结果值就如同一个隐式的返回，即返回最后一个语句的结果值。

#### 5.1.2 表达式的副作用
语句系列逗号运算符：在()中写入多个语句，用逗号','分隔，返回最后一个语句的结果值。
比如`b = (a++, a);`可以返回a++之后的a。

### 5.2 运算符优先级
`&&`高于`||`。

#### 5.2.1 短路

















