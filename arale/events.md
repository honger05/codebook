Events 提供了基本的事件添加、移除和触发功能。

仅提供非常纯粹的事件驱动机制。同类产品以下两种

- 重点借鉴 Backbone.Events .
- NodeJS 的 EventEmitter 功能点较多，主要考虑服务器端。

### on  `object.on(event, callback, [context])`

````js
// 注册事件，全保存在 this.__events 中，可以指定回调函数执行时上下文 [ context ]
Events.prototype.on = function(events, callback, context) {
  var cache, event, list;
  if (!callback) return this;
  cache = this.__events || (this.__events = {});
  // 可以接收多个参数，如 obj.on('x y', fn) 等同于 obj.on('x').on('y')
  events = events.split(/\s+/);
  while (event = events.shift()) {
    // 同一种事件放在同一个集合中 callback，context 两两保存的方式
    // Backbone 的内部结构是链表，改为数组后，性能大幅提升。Backbone 已合并 Arale 代码。
    list = cache[event] || (cache[event] = []);
    list.push(callback, context);
  }
  // 链式操作 返回 this
  return this;
}
````

----
### off `object.off([event], [callback], [context])`

从对象上移除事件回调函数。三个参数都是可选的，当省略某个参数时，表示取该参数的所有值

````js
Events.prototype.off = function(events, callback, context) {
  // 变量无块作用域，所以都定义在最顶部
  var cache, event, list, i;
  if (!(cache = this.__events)) return this;
  // 三个参数都不存在时，移除 this.__events 所有事件
  if (!(events || callback || context)) {
    delete this.__events;
    return this;
  }
  // 第一个事件参数不存在时，遍历所有 cache 中的事件。 keys 的用法在最下面
  events = events ? events.split(/\s+/) : keys(cache);
  // 遍历事件集合
  while (event = events.shift()) {
    list = cache[event];
    if (!list) continue;
    // 后面两个参数都不存在时，移除所有 event 事件
    if (!(callback || context)) {
      delete cache[event];
      continue;
    }
    for (i = list.length - 2; i >= 0; i -= 2) {
      // 当 callback 或 context 匹配时，删除之。。
      if (!(callback && list[i] !== callback || context && list[i + 1] !== context)) {
        list.splice(i, 2);
      }
    }
  }
  return this;
}
````

----
### trigger

````js
Events.prototype.trigger = function(events) {
  var cache, event, all, list, i, len, rest = [], args, returned = {status: true};

  if (!(cache = this.__events)) return this;
  events = events.split(/\s+/);
  // 循环要比 [].slice() 复制数组 快！
  for (i = 1, len = arguments.length; i < len; i++) {
    rest[i-1] = arguments[i];
  }
  while (event = events.shift()) {
    if (all = cache.all) all = all.slice();
    if (list = cache[event]) list = list.slice();
    // 执行事件的绑定
    callEach(list, rest, this, returned);
    // 执行 all 的绑定
    callEach(all, [ event ].concat(rest), this, returned);
  }
  return returned.status;
}

function callEach(list, args, context, returned) {
  var r;
  if (list) {
    for(var i = 0, len = list.length; i < len, i++) {
      // 执行绑定事件
      r = list[i].apply(list[i+1] || context, args);
      // trigger 的返回值是一个布尔值，只要有一个 callback 返回 false，trigger 就会返回 false。
      r === false && returned.status && (returned.status = false);
    }
  }
}
````

---
### mixTo `Events.mixTo(receiver)`

````js
Events.mixTo = function(receiver) {
  // 接收者若是函数，就放在它的原型上，若是对象，就放在对象的身上。
  receiver = receiver.prototype || receiver;
  var proto = Events.prototype;
  for (var p in proto) {
    if (proto.hasOwnprototype(p)) {
      receiver[p] = proto[p];
    }
  }
}
````

例如

````js
define(function(require) {
    var Events = require('events');

    function Dog() {
    }

    //在 Dog.prototype 上增加 Events 的三个方法。
    Events.mixTo(Dog);

    Dog.prototype.sleep = function() {
        this.trigger('sleep');
    };

    var dog = new Dog();
    dog.on('sleep', function() {
        alert('狗狗睡得好香呀');
    });

    dog.sleep();
});
````

Events 的另一种用法是

````js
define(function(require) {
    var Events = require('events');

    var object = new Events();
    object.on('expand', function() {
        alert('expanded');
    });

    object.trigger('expand');
});
````


---
### Object.keys

最后介绍下 ES5 的 `Object.keys` , 遍历对象中所有的 key 值

````js
var keys = Object.keys;
if (!keys) {
  keys = function(o) {
    var result = [];
    for (var name in o) {
      if (o.hasOwnprototype(name)) {
        result.push(name);
      }
    }
    return result;
  }
}
````
