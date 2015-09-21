### 先谈谈基于原型的继承。

先看看 segementfault 上讨论的一道题。

````js
function F() {}
Object.prototype.a = function() {}
Function.prototype.b = function() {}
var f = new F()
// F.a F.b f.a
````

F 可以调用 a 和 b，因为 F 的原型链是这样的。(直观解释：F 是 Function 实例，F 继承自 Object)

````js
F ----> Function.prototype ----> Object.prototype ----> null

//既 F.__proto__ === Function.prototype
// Function.prototype.__proto__ === Object.prototype
// Object.prototype.__proto__ === null
````

而 f 只能调用 a ， f 的原型链是这样的。（直观解释：f 是 F 的实例，一切皆对象，f 继承自 Object）

````js
f ----> F.prototype ----> Object.prototype ----> null

//既 f.__proto__ === F.prototype
// F.prototype.__proto__ === Object.prototype
// Object.prototype.__proto__ === null
````

在 f 的原型链上并没有 Function.prototype . 因此访问不到 b 。

> 注意，访问原型对象 `__proto__` 是非标准方法，ES5 标准方法是 Object.getPrototypeOf();

到这里，基于原型链的继承已经很明显了，只需要

````js
function Animal() {}

function Dog() {}

Dog.prototype.__proto__ = Animal.prototype;

var dog = new Dog();

// dog.__proto__ --> Dog.prototype;
// dog.__proto__.__proto__ --> Animal.prototype
````

基于 ES5 标准写法是

```js
Dog.prototype = Object.create(Animal.prototype);
```

---
### 来看看 arale 的封装

````js
// 创建原型链
function Ctor() {};

var createProto = Object.__proto__ ? function(proto) {
  return {
    __proto__: proto
  }
} : function(proto) {
  Ctor.prototype = proto;
  return new Ctor();
}
````

有三种写法可以实现原型链继承，但是我测试 new Ctor 是最慢的啊，Object.creaete 其次，`__proto__` 是最快的。

```js
function Ctor() {}

function getProto1(proto, c) {
  Ctor.prototype = proto;
  var o = new Ctor();
  o.constructor = c;
  return o;
}

function getProto2(proto, c) {
  return Object.create(proto, {
    constructor: {
      value: c
    }
  })
}

function getProto3(proto, c) {
  return {
    __proto__: proto,
    constructor: c
  }
}
```

接着往下看。。。

````js
function Class(o) {
  if (!(this instanceof Class) && isFunction(o)) {
    return classify(o);
  }
}

function classify(cls) {
  cls.extend = Class.extend;
  cls.implement = implement;
  return cls;
}
````

这种写法是，当不使用 `new` 关键字调用时，将参数 `类化`，如：

> 修改：是不支持 `new` 的方式调用。

````js
function Animal() {}
Animal.prototype.talk = function() {}

//Class(Animal); Animal 拥有了 extend 和 implement 方法
var Dog = Class(Animal).extend({
  swim: function() {}
})
````

Class 的三个变种属性 `Extends` `Implements` `Statics`

这三个属性会做特殊处理

````js
Class.Mutators = {
  // 继承的方法，只支持单继承
  Extends: function(parent) {
    var existed = this.prototype;
    // 建立原型链来实现继承
    var proto = createProto(parent.prototype);
    mix(proto, existed);
    // 强制改变构造函数
    proto.constructor = this;
    this.prototype = proto;
    // 提供 superclass 语法糖，来调用父类属性
    this.superclass = parent.prototype;
  },
  // 混入属性，可以混入多个类的属性
  Implements: function(items) {
    // 将参数变成数组
    isArray(items) || (items = [ items ]);
    var proto = this.prototype, item;
    while (item = items.shift()) {
      // 无论参数是 类（Function），还是 对象（Object），都混入。
      mix(proto, item.prototype || item);
    }
  },
  Statics: function(staticProperties) {
    // 直接混入静态属性。
    mix(this, staticProperties);
  }
}
````

再来看看 `implement` 方法, 它是用来混入属性的。

三个变种属性将被执行

````js
function implement(properties) {
  var key, value;
  for (key in properties) {
    value = properties[key];
    if (Class.Mutators.hasOwnProperty(key)) {
      Class.Mutators[key].call(this, value);
    } else {
      this.prototype[key] = value;
    }
  }
}
````

好了，最关键的方法 `Class.create` 来了，它是用来创建类的，可以指定父类。

````js
Class.create = function(parent, properties) {
  // 如果没有指定父类，父类就是 Class
  if (!isFunction(parent)) {
    properties = parent;
    parent = null;
  }
  properties || (properties = {});
  // 如果指定了 Extends 属性， 父类就是它了
  parent || (parent = properties.Extends || Class);
  properties.Extends = parent;
  // 创建子类的构造函数
  function SubClass() {
    // 调用父类的构造函数
    parent.apply(this, arguments);
    // 仅调用自身的初始化方法，initialize
    if (this.constructor === SubClass && this.initialize) {
      this.initialize.apply(this, arguments);
    }
  }
  // 指定父类的情况下，继承父类的属性
  if (parent !== Class) {
    Mix(SubClass, parent, parent.StaticsWhiteList);
  }
  // 为子类添加实例属性，三个特殊属性，在这里被执行
  implement.call(SubClass, properties);
  // 返回可继续 继承 的子类
  return classify(SubClass);
}
````

最后来看看继承的方法 `Class.extend` ，被 classify 的类，都可以继续创建子类。

````js
Class.extend = function(properties) {
  properties || (properties = {});
  // 指定父类为调用者
  properties.Extends = this;
  return Class.create(properties);
}
````

最后的最后，简单介绍它的工具类，Helpers

````js
// 属性混合，增加白名单限制
function mix(r, s, wl) {
  for (var p in s) {
    // 最佳实践：任何 for in 循环都要带上 hasOwnProperty。除非你想遍历原型
    if (s.hasOwnProperty(p)) {
      if (wl && indexOf(wl, p) === -1) continue;
      if (p !== "prototype") {
        r[p] = s[p];
      }
    }
  }
}

// [].indexOf 是 ES5 加入的，并非所有浏览器都支持。
// 这里没有也不需要使用 polyfill 的方式。
var indexOf = Array.prototype.indexOf ? function(arr, item) {
  return arr.indexOf(item);
} : function(arr, item) {
  for (var i = 0, len = arr.length; i < len; i++) {
    if (arr[i] === item) {
      return i;
    }
  }
  return -1;
}

// 这个很简单，只有 Object.prototype.toString 才能知道它的 [[class]]
var toString = Object.prototype.toString;
var isArray = Array.isArray || function(val) {
  return toString.call(val) === "[object Array]";
}
var isFunction = function(val) {
  return toString.call(val) === "[object Function]";
}
````
