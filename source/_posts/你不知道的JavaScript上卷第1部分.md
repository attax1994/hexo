---
title: 你不知道的JavaScript上卷 第一部分
date: 2018-03-19 16:06:19
categories: 
- books
tags: 
- books
- 你不知道的JavaScript
---

# 你不知道的JavaScript上卷 第一部分
## 第一章 作用域是什么
### 1.1 编译原理
编译的三个基本步骤：
1.	**分词/词法分析（Tokenizing/Lexing）**
将代码字符串分解成有意义的代码块，这些代码块被称为词法单元（token）。如var a = 2;分解成var、a、=、2、;。空格在某些语言中也是词法单元。
2.	**解析/语法分析（Parsing）**
将词法单元流（Array）转换成一个由元素逐级嵌套所组成的代表程序语法结构的树，即抽象语法树（Abstract Syntax Tree, AST）。
其中，顶级节点为VariableDeclaration（var），其子节点有Identifier（a），以及AssignmentExpression（赋值表达式=），后者还有子节点NumericLiteral（2）。
3. **代码生成**
将AST转换为可执行代码的过程。

### 1.2 理解作用域
- **引擎**：负责程序的编译和执行过程。
- **编译器**：负责语法分析和代码生成。
- **作用域**：收集并维护由所有声明的标识符（变量）组成的一系列查询，并确定当前执行的代码对这些标识符的访问权限。
- **LSH查询**：变量出现在赋值操作左侧时进行，试图找到变量的容器本身。
- **RSH查询**（理解为非LSH）：变量出现在赋值操作右侧时进行，查找变量的值。

### 1.3 作用域嵌套
当前作用域无法找到某个变量时，引擎会在外层嵌套的作用域中查找，直至抵达最外层（全局）时仍未找到，抛出错误。

### 1.4 异常
变量无法找到时，抛出ReferenceError，与作用域判别失败相关。
对变量的值进行不合理操作，抛出TypeError，与操作是否合法相关。


## 第二章 词法作用域
### 2.1 词法阶段
词法作用域即定义在词法阶段的作用域。
**作用域查找会在找到第一个匹配的标识符时停止（内层外层均有，则使用内层的）。称为“遮蔽效应”，内部标识符会遮蔽外部同名标识符。**

### 2.2 欺骗词法
***欺骗词法作用域会导致性能下降。***
原因主要是无法对代码的词法进行静态分析，预先确认变量和函数的位置，从而快速寻找。

#### 2.2.1 eval
eval()可以接收一个字符串作为参数，将其中的内容视为在书写时就存在于程序中这个位置的代码。
**通常被用来执行动态创建的代码。**
**严格模式中，eval有其自身的词法作用域，其中的声明无法修改所在的作用域。**
与之相似的有setTimeout，setInterval，new Function()，都可以接收字符串。

#### 2.2.2 with
with通常被当作重复引用同一个对象中的多个属性的快捷方式，可以不需要重复引用对象本身。with会创建一个全新的词法作用域。
**在严格模式下，with被禁止。**


## 第三章 函数作用于和块作用域
### 3.1 函数中的作用域
JS具有基于函数的作用域。内部能向外访问，外部不能向内访问。

### 3.2 隐藏内部实现
**最小授权或最小暴露原则**：在软件设计中，应该最小限度地暴露必要的内容，而将其他内容都隐藏起来。
隐藏作用域能够避免同名标识符之间的冲突。

### 3.3 函数作用域
外部作用域不能访问包装函数内部的任何内容。
如果function是声明中第一个词（前面没有其他词，甚至是括号），那么就是一个函数声明，否则就是一个函数表达式。

#### 3.3.1 匿名和具名
**函数表达式可以匿名，函数声明必须具名。**
匿名函数也有缺点：
1.	没有函数名，栈追踪调试困难。
2.	递归自身时，只能使用arguments.callee。或者是事件解绑时候找不到。
3.	没有函数名，降低了可读性。

选择性地给函数表达式具名，可以解决以上问题。

#### 3.3.2 立即执行表达式(IIFE)
用圆括号()将函数包裹，然后紧跟圆括号调用。
另一种可以将括号放在内部。
`(function ( ){ …… } ( ))`
UMD标准中，IIFE也被广泛运用，比如：
```JavaScript
var a = 2;
(function IIFE( def ) {
	def( window )
})(function def ( global ) {
	var a = 3;
	console.log( a ); //3
	console.log( global.a ); //2
});
```

### 3.4 块作用域
变量声明应该距离使用的地方越近越好，并且最大限度地本地化。（因多次多处访问而提前做缓存除外）
块级作用域在ES6中得到广泛应用。在此之前需要注意，var声明的变量并不属于块级作用域。
可以生成块级作用域的有：
-	with
-	try/catch
- let、const（ES6新特性），可以形成暂时性锁区。

块级作用域的用处：
1.	有利于垃圾收集。程序块在执行后，其中的变量如果不被后续需要（闭包等），就可以将内存回收
2.	减少变量空间污染。


## 第四章 提升
### 4.1 先有鸡还是先有蛋
function和var声明，会被提升到顶部。
其他的操作不会提升，处于自身原本的位置。

### 4.2 编译器再度来袭
每个作用域都会进行提升操作，其顶部为其自身作用域的顶部（注意不会跨越作用域）。
**函数表达式不会提升，而是处于正常的位置。**
**具名的函数表达式也不存在提升**
```JavaScript
foo(); // TypeError，因为此时foo被初始化为undefined，还没有赋值为函数表达式
bar(); // ReferenceError，因为函数表达式不提升，此时bar还没有声明
var foo = function bar(){
	// do sth
}
```
等效于
```JavaScript
var foo;
foo(); // TypeError，因为此时foo被初始化为undefined，还没有赋值为函数表达式
bar(); // ReferenceError，因为函数表达式不提升，此时bar还没有声明
foo = function (){
	var bar = self;
	// do sth
}
```

### 4.3 函数优先
函数首先被提升，然后是变量。函数与变量同名时，变量的声明会被省略，但是依旧可以进行赋值操作。
**后出现的函数声明会覆盖前面的同名函数，所以在同一作用域内，千万不要声明同名的函数**
**在普通块内部的声明，也会提升到作用域顶部，而不是在块内。**
```JavaScript
foo() // 'b'

var a = true;
if(a){
	function foo(){ console.log('a') } // 先声明，被后面覆盖了
} else {
	function foo(){ console.log('b') } // 后声明，覆盖前面了
}
```


## 第五章 作用域闭包
### 5.1 启示
JS中闭包无处不在。
闭包是基于词法作用域书写代码时产生的自然结果。
闭包特征就是将函数表达式，连带其词法作用域，进行传值。

### 5.2 实质问题
一个闭包的例子：
```JavaScript
function foo(){
	var a = 2;
	
	function bar(){
		console.log(a);
	}

	return bar; //也可以直接返回函数表达式，function(){ console.log(a); }
}

var baz = foo();
baz(); // 2，读取到了foo中定义的变量a
```
本质上是，函数bar的词法作用域能够访问foo的内部作用域，因为它的作用域嵌套在了foo的作用域内。
也就是说，最终return的结果是带有scope（作用域）信息的，这个是能够找到需要调用变量的根本依据。
当bar在使用时，闭包会阻止垃圾回收器回收foo的作用域（有好有坏）。

### 5.3 现在我懂了
引擎在调用函数的同时，其词法作用域会保持完整。
如果将（访问他们各自词法作用域的）函数当作第一级的值类型并到处传递，你就会看到闭包在这些函数中的应用。
**也就是说，只要使用了回调函数，实际上就是在使用闭包。**

### 5.4 循环和闭包
异步时候，函数进行回调时，调用变量值的是回调发生时候的值，所以要考虑运行时的状态。
```JavaScript
for(var i=0; i<5; i++){
	setTimeout( function(){
		console.log(i); //输出都是5
	}, 1000*i);
}
```
解决方法有IIFE
```JavaScript
for(var i=0; i<5; i++){
	// 创建一个新的词法作用域，把它的值在这个作用域里面记录下来
	(function(j){
		setTimeout(function(){
			console.log(j); //这样输出就是0,1,2,3,4了
		}, 1000*j);
	})(i);
}
```
或者利用作用块
```JavaScript
// let可以作用于作用块
for(let i=0; i<5; i++){
	setTimeout( function(){
		console.log(i); //这样输出就是0,1,2,3,4了
	}, 1000*i);
}
```
这里面本质就是
```JavaScript
for(var i=0; i<5; i++){
	// 使用仅存在于该作用块的变量
	let _i = i;
	setTimeout( function(){
		console.log(_i); //这样输出就是0,1,2,3,4了
	}, 1000*_i);
}
```

### 5.5 模块
实现模块的模式称为模块暴露。
```JavaScript
function MyModule(){
	var something = 'cool';
	var another = [1,2,3];

	function doSomething(){
		console.log(something);
	}

	function doAnother(){
		console.log(another.join(','));
	}

	//ES5的写法
	return {
		doSomething: doSomething,
		doAnother: doAnother,
	};

	// ES6的写法
	/* return {
		doSomething,
		doAnother,
	}; */
}

// 得到了这个闭包
var foo = MyModule();

foo.doSomething(); // cool
foo.doAnother(); // 1,2,3
```
- 只需要使用一次的时候，记得改成IIFE。
- 需要参数的时候，记得自己去定义参数表。

#### 5.5.1 现代的模块机制
一个典型的模块管理器可以定义为（应该是参考了Require.js）
```JavaScript
var MyModules = (function Manager(){
	var modules = {};

	function define(name, deps, impl) {
		// 抽取依赖
		for(var i=0; i<deps.length; i++){
			deps[i] = modules[deps[i]];
		}
		// 绑定依赖，获得模块
		modules[name] = impl.apply(impl,deps);
	}

	function get(name){
		return modules[name];
	}

	// 把方法暴露出来
	return {
		define: define,
		get: get,
	};
});
```
其使用方式为
```JavaScript
// 名称为bar的模块，没有依赖
MyModules.define('bar', [], function(){
	function hello(who){
		return 'Let me introduce: ' + who;
	}

	return {
		hello: hello,
	};
});

// 名称为foo，依赖了bar。
// 注意这里函数表达式能拿到bar，是因为define方法去modules中抽取了bar，然后传入给它。
MyModules.define('foo', ['bar'], function(bar){
	var hungry = 'hippo';

	function awesome(){
		// 因为外层参数已经传入了bar，所以这里也就能拿到bar的hello方法
		console.log(bar.hello(hungry).toUpperCase());
	}

	return {
		awesome: awesome,
	};
});

var foo = MyModules.get('foo');
foo.awesome(); // LET ME INTRODUCE HIPPO
```

#### 5.5.2 未来的模块机制
ES6中使用import和export来进行模块管理。
ES6模块没有行内模式，必须被定义在独立的文件中。
`bar.js`
```JavaScript
export function hello(who){
		return 'Let me introduce: ' + who;
}
```
`foo.js`
```JavaScript
// 直接获得了hello
import {hello} from 'bar';

var hungry = 'hippo';

export function awesome(){
	console.log(hello(hungry).toUpperCase());
}
```
`baz.js`
```JavaScript
// 导入完整的foo模块
module foo from 'bar';

foo.awesome(); // LET ME INTRODUCE HIPPO
```

### 5.6 小结
当函数可以记住并访问所在的词法作用域，即使函数是在当前词法作用域之外执行，这时就产生了闭包。
模块有两个特征：
1. 为创建内部作用域而调用了一个包装函数。
2. 包装函数的返回值至少包括一个对内部函数的引用，这样就会创建涵盖整个包装函数内部作用域的闭包。

