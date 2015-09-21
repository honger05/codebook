使用 Aspect，可以允许你在指定方法执行的前后插入特定函数

---
### before `object.before(methodName, callback, [context])`

在 object[methodName] 方法执行前，先执行 callback 函数.

callback 函数在执行时，接收的参数和传给 object[methodName] 相同。

````js
dialog.before('show', function(a, b) {
  a; // 1
  b; // 2
});

dialog.show(1, 2)
````

### after `object.after(methodName, callback, [context])`

在 object[methodName] 方法执行后，再执行 callback 函数.

callback 函数在执行时，接收的参数第一个是 object[methodName] 的返回值，之后的参数和传给 object[methodName] 相同。

````js
dialog.after('show', function(returned, a, b) {
  returned; // show 的返回值
  a; // 1
  b; // 2
});

dialog.show(1, 2);
````

---
### 源码

定义两个出口

````js
exports.before = function(methodName, callback, context) {
  return weave.call(this, "before", methodName, callback, context);
};

exports.after = function(methodName, callback, context) {
  return weave.call(this, "after", methodName, callback, context);
};
````

定义一个可柯里化的函数 weave，柯里化成 after、before 函数

````js
function weave(when, methodName, callback, context) {
  var names = methodName.split(/\s+/);
  var name, method;
  while (name = names.shift()) {
    // this 指向基于 Base 生成的类的实例，如上例的 dialog
    method = getMethod(this, name);
    // 如果该函数（show）是第一次切面化绑定，则包装该函数。
    // 被包装的函数在执行前后都会触发下面注册的事件。
    if (!method.__isAspected) {
      wrap.call(this, name);
    }
    // 注册事件：例如 after:show 、 before:show .
    this.on(when + ":" + name, callback, context);
  }
  return this;
}
````

如果没有在实例中找到该方法（show），则抛出异常。

````js
// 在实例中查找该方法，若没有则抛出异常
function getMethod(host, methodName) {
  var method = host[methodName];
  if (!method) {
    throw new Error('Invalid method name: ' + methodName);
  }
  return method;
}
````

定义一个包装器函数，被包装的函数在执行前后（before、after）都会触发一个特定的函数

````js
// 包装器，使该函数（show）支持切面化
function wrap(methodName) {
  // 保存旧的方法，this 指向该对象（dialog）
  var old = this[methodName];
  // 定义新的方法，并在旧方法之前触发 before 绑定的函数，之后触发 after 绑定的函数
  this[methodName] = function() {
    var args = Array.prototype.slice.call(arguments);
    var beforeArgs = ["before:" + methodName].concat(args);
    // 触发 before 绑定的函数，如果返回 false 则阻止原函数 （show） 执行
    if (this.trigger.apply(this, beforeArgs) === false) return;
    // 执行旧的函数，并将返回值当作参数传递给 after 函数
    var ret = old.apply(this, arguments);
    var afterArgs = ["after:" + methodName, ret].concat(args);
    // 触发 after 绑定的函数，绑定的函数的第一个参数是旧函数的返回值
    this.trigger.apply(this, afterArgs);
    return ret;
  }
  // 包装之后打个标记，不用再重复包装了
  this[methodName].__isAspected = true;
}
````
