---
title: 手写EventEmitter
date: 2018-03-20 15:11:28
categories:
- JavaScript
tags:
- JavaScript
- Design Pattern
---
<hr>

# 手写EventEmitter
## 准备工作
### 1. 事件的创建
JS中，最简单的创建事件方法，是使用Event构造器：
```JavaScript
var myEvent = new Event('event_name');
```
但是为了能够传递数据，就需要使用 CustomEvent 构造器：
```JavaScript
var myEvent = new CustomEvent('event_name', {
	detail:{
		// 将需要传递的数据写在detail中，以便在EventListener中获取
		// 数据将会在event.detail中得到
	},
});
```

### 2. 事件的监听
JS的EventListener是根据事件的名称来进行监听的，比如我们在上文中已经创建了一个名称为**'event_name'** 的事件，那么当某个元素需要监听它的时候，就需要创建相应的监听器：
```JavaScript
//假设listener注册在window对象上
window.addEventListener('event_name', function(event){
	// 如果是CustomEvent，传入的数据在event.detail中
	console.log('得到数据为：', event.detail);
	
	// ...后续相关操作
});
```
至此，window对象上就有了对**'event_name'** 这个事件的监听器，当window上触发这个事件的时候，相关的callback就会执行。

### 3. 事件的触发
对于一些内置（built-in）的事件，通常都是有一些操作去做触发，比如鼠标单击对应MouseEvent的click事件，利用鼠标（ctrl+滚轮上下）去放大缩小页面对应WheelEvent的resize事件。
然而，自定义的事件由于不是JS内置的事件，所以我们需要在JS代码中去显式地触发它。方法是使用 **dispatchEvent** 去触发（IE8低版本兼容，使用fireEvent）：
```JavaScript
// 首先需要提前定义好事件，并且注册相关的EventListener
var myEvent = new CustomEvent('event_name', { 
	detail: { title: 'This is title!'},
});
window.addEventListener('event_name', function(event){
	console.log('得到标题为：', event.detail.title);
});
// 随后在对应的元素上触发该事件
if(window.dispatchEvent) {	
	window.dispatchEvent(myEvent);
} else {
	window.fireEvent(myEvent);
}
// 根据listener中的callback函数定义，应当会在console中输出 "得到标题为： This is title!"
```
需要特别注意的是，当一个事件触发的时候，如果相应的element及其上级元素没有对应的EventListener，就不会有任何回调操作。
对于子元素的监听，可以对父元素添加事件托管，让事件在事件冒泡阶段被监听器捕获并执行。这时候，使用event.target就可以获取到具体触发事件的元素。
<hr>

## 事件触发器
### 1.基于浏览器的CustomEvent
上述过程是在JS中触发一个事件的原理，但是相对于实际的运用不算方便，所以需要对这个事件发生器进行一下封装。
这里可以选择将事件注册在document元素上，这样在页面销毁之前，这个事件会一直保存。
```JS
/**
 * 类似于Vue、Angular等MVVM的事件发生器，使用CustomEvent
 */
// ES6
class _Event {
    static on(type, handler) {
        return document.addEventListener(type, handler)
    }
    static emit(type, data) {
        return document.dispatchEvent(new CustomEvent(type, {
            detail: data
        }))
    }
}

// ES5
function myEvent() {}
myEvent.on = function (type, handler) {
    return document.addEventListener(type, handler);
};
myEvent.emit = function (type, data) {
    return document.dispatchEvent(new CustomEvent(type, {
        detail: data,
    }));
};

_Event.on('search', e => {
    console.log(e.detail);
});
_Event.emit('search', 'dosth ');
```

### 2.实现一个可以手动触发的Event管理器
对于一些MVVM项目来说，状态的管理不能局限于单一一个页面，页面之间的共享需要使用Shared Service Worker来实现，这时候依赖于document上的CustomEvent就比较乏力了。
于是，我们需要去设计一个Event的管理器，用它来记录各个Event的hanlders，然后用emit方法触发。
```JS
var Event = (function () {
    // private的_events记录每种event的handlers
    var _events = {};

    /**
     * 注册一个Event的handler
     * @param {String} evt 
     * @param {*} handler 
     */
    function on(evt, handler) {
        _events[evt] = _events[evt] || [];
        // 如果是没有名称的话就无法删除
        _events[evt].push({
            name: handler.name || 'anonymous',
            handler: handler,
        });
    }

    /**
     * 触发一种Event，并传入数据
     * @param {String} evt 
     * @param {*} args 
     */
    function emit(evt, args) {
        if (!_events[evt]) {
            return;
        }
        for (var i = 0, ii = _events[evt].length; i < ii; i++) {
            _events[evt][i].handler(args);
        }
    }

    /**
     * 用来移除非匿名的handler
     * @param {String} evt 
     * @param {String} name 
     */
    function remove(evt, name) {
        if (!_events[evt]) {
            return;
        }
        _events[evt] = _events[evt].filter(value => {
            return value.name !== name;
        });
    }

    return {
        on: on,
        emit: emit,
        remove: remove,
    };
})();

Event.on('search', function dosth(data) {
    console.log(data);
});

Event.emit('search', '123');

Event.remove('search', 'dosth');

Event.emit('search', '123');
```

