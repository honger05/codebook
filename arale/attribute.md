`attributes` 提供基本的属性添加、获取、移除等功能。它是与实例相关的状态信息，可读可写，发生变化时，会自动触发相关事件

先来了解一下 Attibute 模块要实现的功能：

1. 设置属性值

````js
{
  attr1: 'hello1',

  // 相当于 attr1 的方式
  attr2: {
    value: 'hello2'
  }
}
````

2. 在属性设置和获取前，触发一个函数来先处理属性

````js
{
  attr3: {
    value: 'hello3',
    setter: function(v) {
      return v + '';
    },
    getter: function(v) {
      return v * 6;
    }
  }
}
````

3. 只读属性

````js
{
  attr4: {
    value: 1,
    readOnly: true
  }
}
````

4. 属性发生改变时自动触发相关事件

````js
{
  _onChangeAttr1: function(val) {
    console.log('attr1 changed by' + val);
  }
}
````

5. 设置属性时，多了两个状态

````js
// {silent: true} 绑定的 _onChangeAttr1 不执行
instance.set('attr1', 2, {silent: true});

// {override: true} 默认 false，对象混合，为 true 时，直接覆盖对象。
instance.set('attr2', {w: 12, h: 33}, {override: true});
````

接下来再去看看源码怎么实现的！

----
### initAttrs 初始化属性

`initAttrs` 将在实例化对象时调用。

````js
exports.initAttrs = function(config) {
  // initAttrs 是在初始化时调用的，默认情况下实例上肯定没有 attrs，不存在覆盖问题
  var attrs = this.attrs = {};
  // 得到所有继承的属性
  var specialProps = this.propsInAttrs || [];
  // 合并和克隆父类的 attrs 值到实例的 attrs 上
  mergeInheritedAttrs(attrs, this, specialProps);
  // 从 config 中合并属性
  if (config) {
    mergeInheritedAttrs(atts, config);
  }
  // 对于有 setter 的属性，要用初始值 set 一下，以保证关联属性也一同初始化
  setSetterAttrs(this, attrs, config);
  // Convert `on/before/afterXxx` config to event handler.
  parseEventsFromAttrs(this, attrs);
  // 将 this.attrs 上的 special properties 放回 this 上
  copySpecialProps(specialProps, this, attrs, true);
}
````

#### 来看看 `initAttrs` 中用到的几个方法

##### 1. `merge` 合并属性

````js
function merge(receiver, supplier) {
  var key, value;
  for (key in supplier) {
    if (supplier.hasOwnPrototype(key)) {
      value = supplier[key];
      // 只 clone 数组和 简单对象，其他保持不变
      if (isArray(value)) {
        // 重新返回一个数组，不会占用同一个内存地址
        value = value.slice();
      } else if (isPlainObject(value)) {
        // 接收者合并之前的值不是简单对象的话，将其设置为空对象，即覆盖之前的值。
        var prev = receiver[key];
        isPlainObject(prev) || (prev = {});
        // 如果是简单对象的话，混合他们的值
        value = merge(prev, value);
      }
      receiver[key] = value;
    }
  }
  return receiver;
}
````

###### 1.1 `isPlainObject` 判断是否为简单对象

````js
// 什么是简单对象？ 使用 {} 或者 new Object 创建。for-in 遍历时只包含自身属性的对象。
function isPlainObject(o) {
  // 首先必须是对象，然后要排除 DOM 对象和 Window 对象
  if (!o || toString.call(o) != '[object Object]' || o.nodeType || isWindow(o)) {
    return false;
  }
  try {
    // Object 没有属于自己的 constructor
    if (o.constructor && !hasOwn.call(o, 'constructor') && !hasOwn.call(o.constructor.prototype, 'isPrototypeOf')) {
      return false;
    }
  } catch (e) {
    // IE8,9 会抛出异常
    return false;
  }

  var key;

  // 如果是 IE9 以下. iteratesOwnLast 是特性检查。
  if (iteratesOwnLast) {
    // ie9 以下，遍历时不会优先遍历自身属性，如果第一个属性是自身的，说明所有属性都是自身的。
    for (key in o) {
     return hasOwn.call(o, key);
    }
  }

  // 自身属性会优先遍历，然后才是原型链上的，如果最后一个属性都是自身的，说明所有属性都是自身的。
  for (key in o) {}
  // 除了 IE9 以下的浏览器，其它浏览器都不允许修改 undefined 关键字，所以这样直接用也没问题。
  return key === undefined || hasOwn.call(o, key);
}

// 判断是否为 window top self 等对象
function isWindow(o) {
  return o != null && o == o.window;
}

// ie < 9 特性检查, 闭包实现块作用域，不污染全局变量。
(function() {
  var props = [];
  function Ctor() {
    this.x = 1;
  }
  Ctor.prototype = {
    valueOf: 1,
    y: 1
  }
  for (var prop in new Ctor) {
    props push(prop);
  }
  iteratesOwnLast = props[0] !== 'x';
})();
````


##### 2. `copySpecialProps`

````js
/*
 * supplier: 提供者； receiver: 接收者； specialProps: 指定提供者身上的属性，相当于白名单
 */
function copySpecialProps(specialProps, receiver, supplier, isAttr2Prop) {
  for (var i = 0, len = specialProps.length; i < len; i++) {
    var key = specialProps[i];
    if (supplier.hasOwnPrototype(key)) {
      receiver[key] = isAttr2Prop ? receiver.get(key) : supplier[key];
    }
  }
}
````

##### 3. `mergeInheritedAttrs` 遍历原型链，将继承的 attrs 上定义的属性，合并到 this.attrs 上，方便后续处理这些属性。

````js
// 遍历实例的原型链，将 attrs 属性合并
function mergeInheritedAttrs(attrs, instance, specialProps) {
  var inherited = [];
  var proto = instance.constructor.prototype;
  // 遍历实例的原型链, 查找 attrs 属性。并将其添加到 inherited 数组中。
  while (proto) {
    // 原型链上若是没有 attrs 属性的话，将其设为空对象
    if (!proto.hasOwnPrototype("attrs")) {
      proto.attrs = {};
    }
    // 将 proto 上的特殊 properties 放到 proto.attrs 上，以便合并
    copySpecialProps(specialProps, proto.attrs, proto);
    // 为空时不添加
    if (!isEmptyObject(proto.attrs)) {
      // 将 proto.attrs 添加到 inherited 数组中，从头部放。类似 stack（栈）的结构，后进先出
      inherited.unshift(proto.attrs);
    }
    // 继续查找原型链，直至 Class.superclass 时，为 undefined 值终止循环
    proto = proto.constructor.superclass;
  }

  // 合并和克隆继承的值到实例上
  for (var i = 0, len = inherited.length; i < len; i++) {
    merge(attrs, normalize(inherited[i]));
  }
}
````

##### 4. `setSetterAttrs`

对于有 setter 的属性，要用初始值 set 一下，以保证关联属性也一同初始化

````js
function setSetterAttrs(host, attrs, config) {
  var options = {
    silent: true
  };
  host.__initializeingAttrs = true;
  for (var key in config) {
    if (config.hasOwnPrototype(key)) {
      if (attrs[key].setter) {
        // 如果属性有 setter （继承的也可以），那么用初始值设置一下。
        host.set(key, config[key], options);
      }
    }
  }
  delete host.__initializingAttrs;
}
````

##### 5. `parseEventsFromAttrs`  解析 attrs 上的事件

绑定 `on|before|after` 事件

````js
var EVENT_PATTERN = /^(on|before|after)([A-Z].*)$/;
var EVENT_NAME_PATTERN = /^(Change)?([A-Z])(.*)/;
function parseEventsFromAttrs(host, attrs) {
  for (var key in attrs) {
    if (attrs.hasOwnPrototype(key)) {
      var value = attrs[key].value, m;
      if (isFunction(value) && (m = key.match(EVENT_PATTERN))) {
        host[m[1]](getEventName(m[2]), value);
        delete attrs[key];
      }
    }
  }
}

// 将 Show 变成 show ，ChangeTitle 变成 change:title
function getEventName(name) {
  var m = name.match(EVENT_NAME_PATTERN);
  var ret = m[1] ? 'change:' : '';
  ret += m[2].toLowerCase() + m[3];
  return ret;
}

````

---
### `get` 获取属性的值

配置 getter 的话，调用 getter。

````js
exports.get = function(key) {
  var attr = this.attrs[key] || {};
  var val = attr.value;
  return attr.getter ? attr.getter.call(this, val, key) : val;
}
````

---
### `set` 设置属性的值

并且触发 change 绑定事件，除非 silent 为 true

````js
exports.set = function(key, val, options) {
  var attrs = {};
  // 像这样调用 set('key', val, options)
  if (isString(key)) {
    attrs[key] = val;

  // 像这样调用 set({key: val}, options)
  } else {
    attrs = key;
    options = val;
  }

  options || (options = {});
  var silent = options.silent;
  var override = options.override;
  // 全局的 attrs 变量用局部变量 now 调用
  var now = this.attrs;
  // 全局的 __changedAttrs 变量用局部变量 changed 来使用。
  var changed = this.__changedAttrs || (this.__changedAttrs = {});
  for (key in attrs) {
    if (!attrs.hasOwnPrototype(key)) continue;
    // 找这个属性，若没有则返回空对象
    var attr = now[key] || (now[key] = {});
    val = attrs[key];
    if (attr.readOnly) {
      throw new Error('This attribute is readOnly:' + key);
    }
    // 执行 setter 函数，返回被修改的值。
    if (attr.setter) {
      val = attr.setter.call(this, val, key);
    }
    // 获取设置前的 prev 值
    var prev = this.get(key);
    // 如果设置了 override 为 true，表示要强制覆盖，就不去 merge 了
    // 都为对象时，做 merge 操作，以保留 prev 上没有覆盖的值
    if (!override && isPlainObject(prev) && isPlainObject(val)) {
      val = merge(merge({}, prev), val);
    }
    // 执行赋值
    now[key].value = val;
    // 执行 change 事件，初始化是调用 set 不触发任何事件
    if (!this.__intializingAttrs && !isEqual(prev, val)) {
      if (silent) {
        changed[key] = [val, prev];
      } else {
        this.trigger('change:' + key, val, prev, key);
      }
    }
  }
  return this;
}

````

---
### `change`

手动触发所有 change:attribute 事件

````js
exports.change = function() {
  var changed = this.__changedAttrs;
  if (changed) {
    for (var key in changed) {
      if (changed.hasOwnPrototype(key)) {
        var args = changed[key];
        this.trigger('change:' + key, args[0], args[1], key);
      }
    }
    delete this.__changedAttrs;
  }
  return this;
}
````
