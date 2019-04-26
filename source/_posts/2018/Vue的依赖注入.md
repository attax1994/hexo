---
title: Vue的依赖注入
date: 2018-04-02 21:02:27
categories: 
- 前端开发
tags:
- 设计模式
---

## 依赖注入
### 定义

在后端领域，依赖注入（DI），控制反转（IoC）等概念是老生常谈的设计模式，它们用于解决一系列的面向对象编程问题。

对于依赖注入而言，需要遵循依赖倒转原则(Dependence Inversion Priciple, DIP)：

- 高层模块不应该依赖低层模块。两个都应该依赖抽象（提取共性，描述特性，模块间解耦）
- 抽象不应该依赖细节，细节应该依赖抽象（抽象服务类置于底层，细节服务类继承抽象服务类）
- 针对接口编程，不要针对实现编程（遵循SOLID原则，多态、泛型编程等）



### 为什么需要依赖注入

前端工程化、模块化的时间相比于后端而言还是比较短的。在此之前，前端的设计方式主要是每个页面都有其对应的JS文件，分别负责这个页面上的交互操作。

然而随着项目的越来越大，这样的文件结构也显得越来越臃肿，特别是由于很多功能的共性没有提取出来，形成模块化，因而产生了很多重复性的代码。这样的结构存在很大的弊端：

- 一旦需要一些共性方面的修改，就需要逐个文件去检查，修改对应代码，费时费力
- 重复性的代码增大了项目的体积，代码却不能复用
- 每次引入一大堆JS文件，需要执行后存入内存方可使用，效率低下

因此，急需一种机制来负责管理这些可复用的**底层的操作，单例服务以及共有的特性**。



### 传统的依赖注入

传统的依赖注入采取`AMD`、`CMD`、`CommanJS`和`UMD`等模式。

一个AMD的依赖管理器可以大致理解为：

```JS
var ModuleManager = (function () {
  // 用于记录模块，作用类似于Map
  var _modules = {};

  // 根据模块名称，获取对应的模块（服务类实例）
  function getModule(name) {
    return _modules[name] || {};
  }

  // 定义一个模块
  function define(name, deps, fn) {
    if (typeof name !== 'string') {
      throw new TypeError('name should be string.');
    }
    if (_modules[name]) {
      throw new Error('This module has been already declared.');
    }
    if (!deps instanceof Array) {
      throw new TypeError('deps should be Array<String>.');
    }
    if (!fn instanceof Function) {
      throw new TypeError('fn should be function.');
    }

    // 导出它所依赖的模块
    var depsModules = [];
    for (var i = 0, len = deps.length; i < len; i++) {
      depsModules.push(getModule(deps[i]));
    }

    // 注入依赖，并得到返回的模块，写入Map
    _modules[name] = fn.apply(null, depsModules);
    return true;
  }

  // 获取一系列的依赖，从而使用它们进行操作
  function require(deps, fn) {
    if (!deps instanceof Array) {
      throw new TypeError('deps should be Array<String>.');
    }
    if (!fn instanceof Function) {
      throw new TypeError('fn should be function.');
    }

    // 导出它所依赖的模块
    var depsModules = [];
    for (var i = 0, len = deps.length; i < len; i++) {
      depsModules.push(getModule(deps[i]));
    }

    // 执行回调
    fn.apply(null, depsModules);
  }

  // 暴露define和require两个方法
  return {
    define: define,
    require: require,
  };
})();
```

通过这段代码可以看出，传统的依赖注入本质上是把依赖存在内存中，然后根据其名称导出，再注入到回调函数中供其运行。



### 更好的实践

`Angular2+`内置了依赖注入的机制，利用`@Injectable()`来修饰一个服务类。

```JS
@Injectable()
export class SomeService {
  constructor () {}
  
  dosth () {
    // ...
  }
}
```

然后在Component或Module的`providers`属性中去声明提供的服务类，从而在这一层生成一个实例。

```JS
@Component({
  selector: '...',
  template: ``,
  // 需要注意，这是一个数组，可以导入数个依赖
  providers: [SomeService],
})

// 在Module层导入也是可以的
@NgModule({
  imports: [],
  exports: [],
  declarations: [],
  providers: [SomeService],
})
```

最后，在Component的`constructor`中，就可以将其作为依赖导入。

```JS
class TemplateComponent {
  constructor(private _someService: SomeService) {
    this._someService.dosth();
  }
}

// 其实转译为
class TemplateComponent {
  // 在构造的时候传入对应的依赖实例
  constructor(SomeService) {
    this.SomeService = SomeService;
    this._someService = this.SomeService;
  }
}
```

对于一个Component中使用的依赖，其寻径过程遵循就近使用的原则： 

- 首先寻找自身Component修饰器中是否有对应`provider`
- 自身没有，向上层父Component寻找
- 父Component逐层向上，一直到其所在的Module层
- 自身Module层没有，寻找所在Module的上层Component或Module
- 一直到最高层的App.Component中都没有找到，说明该`provider`尚未注册，抛出错误

需要注意的是，这些依赖本身就是一个个实例，所以如果其中有一些不能复用的属性（虽然从设计上而言，应当避免这种情况），应当在所操作的Component上新注册一个该服务的实例，从而达到属性不共享。



### Vue中实现高复用的依赖注入

#### 1.简单粗暴法，Vue.use()

`Vue.use()`方法允许将具体的服务类实例挂载到全局对象的原型链上，调用时候直接去全局对象上寻找相应依赖。这种方法虽然不甚美观，而且随着项目越来越大，全局对象也会越来越臃肿，但确实是非常简单实现方式。

#### 2.利用Vuex

利用`Vuex.Store`的`Modules`属性，可以使用一个独特的`DependencyStore`来存放依赖。在此基础之上，可以实现模块的分类与分级，从而将其层级化，方便调用。

#### 3.第三方库InversifyJS

`InversifyJS`的使用方式类似于Angular的依赖注入模式：

创建声明

```JS
// 声明一系列的接口
export interface Warrior {
    fight(): string;
    sneak(): string;
}

export interface Weapon {
    hit(): string;
}

export interface ThrowableWeapon {
    throw(): string;
}
```

```JS
// 将其打包后导出为一个模块
const TYPES = {
    Warrior: Symbol.for("Warrior"),
    Weapon: Symbol.for("Weapon"),
    ThrowableWeapon: Symbol.for("ThrowableWeapon")
};

export { TYPES };
```

编写实现

```JS
// 针对先前的声明，创建具体的实现类
@injectable()
class Katana implements Weapon {
    public hit() {
        return "cut!";
    }
}

@injectable()
class Shuriken implements ThrowableWeapon {
    public throw() {
        return "hit!";
    }
}

@injectable()
class Ninja implements Warrior {

    private _katana: Weapon;
    private _shuriken: ThrowableWeapon;

    public constructor(
	    @inject(TYPES.Weapon) katana: Weapon,
	    @inject(TYPES.ThrowableWeapon) shuriken: ThrowableWeapon
    ) {
        this._katana = katana;
        this._shuriken = shuriken;
    }

    public fight() { return this._katana.hit(); }
    public sneak() { return this._shuriken.throw(); }

}

export { Ninja, Katana, Shuriken };
```

```JS
// 也可以在服务类中注入其他的服务类
@injectable()
class Ninja implements Warrior {
    @inject(TYPES.Weapon) private _katana: Weapon;
    @inject(TYPES.ThrowableWeapon) private _shuriken: ThrowableWeapon;
    public fight() { return this._katana.hit(); }
    public sneak() { return this._shuriken.throw(); }
}
```

创建一个容器，放置服务类

```JS
import { Container } from "inversify";
import { TYPES } from "./types";
import { Warrior, Weapon, ThrowableWeapon } from "./interfaces";
import { Ninja, Katana, Shuriken } from "./entities";

// 在容器中，将依赖进行绑定
const myContainer = new Container();
myContainer.bind<Warrior>(TYPES.Warrior).to(Ninja);
myContainer.bind<Weapon>(TYPES.Weapon).to(Katana);
myContainer.bind<ThrowableWeapon>(TYPES.ThrowableWeapon).to(Shuriken);

export { myContainer };
```

注入依赖

```JS
import { myContainer } from "./inversify.config";
import { TYPES } from "./types";
import { Warrior } from "./interfaces";

// 使用get方法来获取依赖，传入的参数为其键名（Symbol类型）
const ninja = myContainer.get<Warrior>(TYPES.Warrior);
```

#### 4. 利用TypeScript实现Ultimate Mode

首先声明一个ModuleManager模块来负责依赖管理

```typescript
const ModuleManager = (function () {
  // debug模式下可以放在window下进行检查
  // 实际生产环境不建议暴露在window下，而是使用内部变量
  (window as any).Modules = {};
  const modules = window.Modules;

  // 使用哈希值来作为键（这个设计可有可无）
  function BKDRHash(str: string): number {
    const seed: number = 31;
    let hash: number = 0,
      index: number = 0;
    while (str[index]) {
      hash = hash * seed + str.charCodeAt(index++);
      hash &= 0xFFFFFFFF;
    }
    return hash;
  }

  // 对对应模块生成一个单例，注册到modules中，
  function addModule(target: any): void {
    const hash = BKDRHash(target.toString().replace(/^function\s(.*)?\s.*/, '$1'));
    if (!modules[hash]) {
      modules[hash] = new target();
    }
  }

  // 从moudules里取出某个模块的单例
  function getModule(target: any): any {
    const hash = BKDRHash(target.toString().replace(/^function\s(.*)?\s.*/, '$1'));
    if (!modules[hash]) {
      modules[hash] = new target();
    }
    return modules[hash];
  }

  return {
    addModule,
    getModule,
  }
}());
```

然后需要利用TypeScript提供的Decorator修饰器，来对模块与注入的依赖进行管理

```JS
/**
 * 在需要注入的模块类上添加该修饰器，可以生成一个单例，纳入ModuleManager的管理
 * @returns {(target: any) => void}
 * @constructor
 */
const Injectable = (): ClassDecorator => (target: Function): any => {
  ModuleManager.addModule(target);
};

// 模块注册示例
@Injectable()
export class SomeModule {}

/**
 * 改造被注入的类中，某个属性的描述器，从而使其承载被注入的依赖
 * @param module
 * @returns {PropertyDecorator}
 * @constructor
 */
const Provided = (module: any): PropertyDecorator => (target: Object, name: string | symbol): any => {
  return {
    configurable: true,
    enumerable: false,
    // 不允许直接对其进行set，应当通过模块暴露的public方法去操作
    set: () => {},
    // getter设置为返回这个module的单例
    get: () => {
      return ModuleManager.getModule(module);
    },
  };
};

// 依赖注入示例
export class Example {
  @Provided(SomeModule) private _someModule: SomeModule;
}
```

