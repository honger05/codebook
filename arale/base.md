`Base` 是一个基础类，提供 `Class、Events、Attribute` 和 `Aspect` 支持。

`Base` 只有一个方法 `extend` 还是继承自 `Class` 的。

````js
var Dog = Base.extend({
  attrs: {
    name: 'ww'
  },
  talk: function() {
    console.log('我是' +  this.get('name'));
  }
});

var dog = new Dog();
dog.talk();
````

---
### 源码解析

Base 是由 Class 继承，并与 Events, Aspect, Attribute 等模块组合而来的。

Base 的生命周期很简单，创建 --> 销毁 。

销毁是为了通知 GC 立即释放内存。提高性能，防止内存溢出。

````js
module.exports = Class.create({

  // 混合了 Events, Aspect, Attribute
  Implements: [ Events, Aspect, Attribute ],

  // 初始化
  initialize: function(config) {

    // 初始化 Attribute 模块的 initAttrs 方法
    this.initAttrs(config);

    // 解析实例上的事件
    parseEventsFromInstance(this, this.attrs);
  },

  // 销毁
  destroy: function() {

    // 销毁所有事件
    this.off();

    // 销毁所有自身属性
    for (var p in this) {
      if (this.hasOwnProperty(p)) {
        delete this[p];
      }
    }

    // 销毁 销毁函数
    this.destroy = function() {};
  }
})
````

解析实例上 _onChangeX 配置的事件，等到 X 变化时执行。

````js
// 查找 _onChangeX 事件，并将其注册，等待执行。
function parseEventsFromInstance(host, attrs) {
  for (var attr in attrs) {
    if (attrs.hasOwnproperty(attr)) {
      var m = '_onChange' + ucfirst(attr);
      if (host[m]) {
        host.on('change:' + attr, host[m]);
      }
    }
  }
}

// 将首字母大写
function ucfirst(str) {
  return str.charAt(0).toUpperCase() + str.substring(1);
}
````
