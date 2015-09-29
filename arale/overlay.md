
基础浮层组件，提供浮层显示隐藏、定位和 IE6 下 select 遮挡、窗口 resize 时重新定位、点击页面空白处浮层消失等等特性，是所有浮层类组件的基类。

如果要开发一个浮层类的 UI 组件，可以基于它进行扩展，dialog、autocomplete、popup、select、calendar 等模块均继承了 Overlay 。

<style>
	.parent {
	    position:relative;
	    float:right;
	}
	.example {
	    color: #fff;
	    background-color: #89c237;
	    display:inline-block;
	}
</style>

<div id="c"><div class='overlay'>目标元素1</div></div>
<div id="a" class="example">基准元素</div>

### 源码分析

Overlay 继承自 Widget，先定义一些基本属性

````js
attrs: {

  // 基本属性
	width: null,
	height: null,
	zIndex: 99,
	visible: false,

  // 定位配置
	align: {
		// element 的定位点，默认为左上角
		selfXY: [0, 0],
		// 基准定位元素，默认为当前可视区域
		baseElement: Position.VIEWPORT,
		// 基准定位元素的定位点，默认为左上角
		baseXY: [0, 0]
	},

	parentNode: document.body
}
````

再定义 显示 和 隐藏 的方法

````js
show: function() {
  // 若从未渲染，则调用 render
	if (!this.rendered) {
    this.render();
  }

  // 触发 _onRenderVisible
  this.set('visible', true);
  return this;
},

hide: function () {
  this.set('visible', false);
  return this;
},
````

定义初始化前的 `setup` 方法

````js
setup: function() {
	var that = this;

	// 加载 iframe 遮罩层并与 overlay 保持同步
	this._setupShim();

	// 窗口resize时，重新定位浮层
	this._setupResize();

	// 将目标元素的 position 设置成 absolute
	this.after('render', function() {
	  var _pos = this.element.css('position');
	  if (_pos === 'static' || _pos === 'relative') {
      this.element.css({
        position: 'absolute',
        left: '-9999px',
        top: '-9999px'
      })
	  }
	});

	// 统一在显示之后重新设定位置
  this.after('show', function() {
    that._setPosition();
  })
}
````

生命周期的最后 -- 销毁 释放内存

````js
destroy: function() {
	erase(this, Overlay.allOverlays);
	erase(this, Overlay.blurOverlays);
	return Overlay.superclass.destroy.call(this);
}
````

一些私有方法

````js
// 进行定位
_setPosition: function(align) {

  // 不在文档流中，定位无效
	if (!isInDocument(this.element[0])) return;

  align || (align = this.get('align'));

  // 如果align为空，表示不需要使用js对齐
  if (!align) return;

  var isHidden = this.element.css('display') === 'none';

  // 在定位时，为避免元素高度不定，先显示出来
  if (isHidden) {
    this.element.css({
      visibility: 'hidden',
      disply: 'block'
    });
  }

  Position.pin({
    element: this.element,
    x: align.selfXY[0],
    y: align.selfXY[1]
  }, {
    element: align.baseElement,
    x: align.baseXY[0],
    y: align.baseXY[1]
  });

  // 定位完成后，还原
  if (isHidden) {
    this.element.css({
      visibility: '',
      display: 'none'
    })
  }

  return this;
}

// 判断元素是否在文档流中
function isInDocument(element) {
  return $.contains(document.documentElement, element);
}
````

加载 iframe 遮罩层并与 overlay 保持同步

````js
_setupShim: function() {
	var shim = new Shim(this.element);

	// 在隐藏和设置位置后，要重新定位
	// 显示后会设置位置，所以不用绑定 shim.sync
	this.after('hide _setPosition', shim.sync, shim);

  // 除了 parentNode 之外的其他属性发生变化时，都触发 shim 同步
  var attrs = ['width', 'height'];
  for (var attr in attrs) {
    if (attrs.hasOwnProperty(attr)) {
    	this.on('change:' + attr, shim.sync, shim);
    }
  }

  // 在销魂自身前要销毁 shim
  this.before('destroy', shim.destroy, shim);
},
````

resize窗口时重新定位浮层，用这个方法收集所有浮层实例

````js
// 收集浮层实例
_setupResize: function() {
	Overlay.allOverlays.push(this);
},

// 降频处理
var timeout;
var winWidth = $(window).width();
var winHeight = $(window).height();
Overlay.allOverlays = [];

$(window).resize(function() {
	timeout && clearTimeout(timeout);
	timeout = setTimeout(function() {
	  var winNewWidth = $(window).width();
	  var winNewHeight = $(window).height();
	
		// IE678 莫名其妙触发 resize
	  if (winWidth !== winNewWidth || winHeight !== winNewHeight) {
	    $(Overlay.allOverlays).each(function(i, item) {
	      // 当实例为空或隐藏时，不处理
	      if (!item || !item.get('visible')) {
	      	return;
	      }
	      item._setPosition();
	    })
	  }

	  winWidth = winNewWidth;
	  winHeight = winNewHeight;

  }, 80);

})
````

除了 element 和 relativeElements，点击 body 后都会隐藏 element

````js
// 收集
_blurHide: function(arr) {
	arr = $.makeArray(arr);
	arr.push(this.element);
	this._relativeElements = arr;
	Overlay.blurOverlays.push(this);
}

// 
Overlay.blurOverlays = [];
$(document).on('click', function(e) {
	hideBlurOverlays(e.target);
})

function hideBlurOverlays(target) {
	$(Overlay.blurOverlays).each(functioin(index, item) {
	  
	  // 当实例为空或隐藏时，不处理
	  if (!item || !item.get('visible')) {
      return;
	  }

    // 遍历 _relativeElements ，当点击的元素落在这些元素上时，不处理
    for (var i = 0; i < item._relativeElements.length; i++) {
    	var el = $(item._relativeElements[i])[0];
    	if (el === target || $.contains(el, target)) {
    		return;
    	}
    }

    // 到这里，判断触发了元素的 blur 事件，隐藏元素
    item.hide();

	})
}
````

绑定一些渲染属性, 用于 set 属性后的界面更新

````js
_onRenderWidth: function (val) {
  this.element.css('width', val);
},

_onRenderHeight: function (val) {
  this.element.css('height', val);
},

_onRenderZIndex: function (val) {
  this.element.css('zIndex', val);
},

_onRenderAlign: function (val) {
  this._setPosition(val);
},

_onRenderVisible: function (val) {
  this.element[val ? 'show' : 'hide']();
}
````