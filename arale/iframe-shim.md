组件会根据传入的目标元素生成一个实例，实例化后会生成一个 iframe 插入到目标元素前，这样 iframe 的层级永远低于目标元素。

开发者需要手动调用 sync 方法来同步 iframe 和目标元素。

如果是 ie6 以外的浏览器只会返回一个空实例，什么都不执行。

````js
var Shim = require('iframe-shim');

// 参数 target 是需要被遮挡的目标元素，可以传 DOM Element 或 Selector
var shim = new Shim('#target');
shim.sync();
````

### 源码分析

`Position` 工具 和 `iframe-shim` 工具 实现方式的区别：

position 是用对象字面量的方式定义的

而 iframe-shim 是用构造函数的方式定义的，因为存在多实例情况，需要保存各自的状态信息(target)。

> 延伸：如果是单例模式的话，它跟对象字面量的区别在于，它可以延迟初始化。

````js
// target 是需要添加垫片的目标元素，可以传 `DOM Element` 或 `Selector`
function Shim(target) {
  // 如果选择器选了多个 DOM，则只取第一个
  this.target = $(target).eq(0);
}

// 根据目标元素计算 iframe 的显隐、宽高、定位
Shim.prototype.sync = function() {
    var target = this.target;
    var iframe = this.iframe;

    // 如果未传 target 则不处理
    if (!target.length) return this;

    var height = target.outerHeight();
    var width = target.outerWidth();

    // 如果目标元素隐藏，则 iframe 也隐藏
    // jquery 判断宽高同时为 0 才算隐藏，这里判断宽高其中一个为 0 就隐藏
    // http://api.jquery.com/hidden-selector/
    if (!height || !width || target.is(':hidden')) {
        iframe && iframe.hide();
    } else {
        // 第一次显示时才创建：as lazy as possible
        iframe || (iframe = this.iframe = createIframe(target));

        iframe.css({
            'height': height,
            'width': width
        });

        Position.pin(iframe[0], target[0]);
        iframe.show();
    }

    return this;
};

// 销毁 iframe 等
Shim.prototype.destroy = function() {
    if (this.iframe) {
        this.iframe.remove();
        delete this.iframe;
    }
    delete this.target;
};

if (isIE6) {
    module.exports = Shim;
} else {
    // 除了 IE6 都返回空函数
    function Noop() {}

    Noop.prototype.sync = function() {return this};
    Noop.prototype.destroy = Noop;

    module.exports = Noop;
}

// Helpers

// 在 target 之前创建 iframe，这样就没有 z-index 问题
// iframe 永远在 target 下方
function createIframe(target) {
    var css = {
        display: 'none',
        border: 'none',
        opacity: 0,
        position: 'absolute'
    };

    // 如果 target 存在 zIndex 则设置
    var zIndex = target.css('zIndex');
    if (zIndex && zIndex > 0) {
        css.zIndex = zIndex - 1;
    }

    return $('<iframe>', {
        src: 'javascript:\'\'', // 不加的话，https 下会弹警告
        frameborder: 0,
        css: css
    }).insertBefore(target);
}
````