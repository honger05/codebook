
# arale 之 position 篇
----

简单实用的定位工具，将一个 DOM 节点相对于另一个 DOM 节点进行定位操作.

例如

````js
// element 缺省时，表示屏幕可见区域
// 将 DOM 节点 a  定位在 屏幕可见区域的左上角 10 px 位置
position.pin(a, {
	x: 10,
	y: 10
})

// 同上面效果一样
position.pin(
    { element: a, x: 'center', y: 'center' },
    { element: Position.VIEWPORT, x: 'center', y: 'center' }
);

// 将 a 定位到 b 下方 20px 的位置
position.pin(a, {
  element: b,
  x: 0,
  y: '100%+20px'
});
````

----
### 源码分析

#### 1. 封装定位对象 `{element: a, x: 0, y:0}`

````js
// 封装一个可视区域的伪元素
var VIEWPORT = {_id: 'VIEWPORT', nodeType: 1},

// 判断 ie6
ua = (window.navigator.userAgent || '').toLowerCase();
isIE6 = ua.indexOf('msie 6') !== -1;
````

将 DOM 对象封装成标准的定位对象 `{element: a, x: 0, y:0}`

同时，定位对象拥有 offset 和 size 两个方法，提供定位对象的偏移值和宽高

````js
function normalize(posObject) {
	posObject = toElement(posObject) || {}

  // 定位对象 如果是 DOM 对象的话 转为一般对象
	if (posObject.nodeType) {
		posObject = {element: posObject};
	}

	// 拿到定位对象的 DOM 对象，缺省值是 可视区域 VIEWPORT
	var element = toElement(posObject.element) || VIEWPORT;

  // 如果不是元素节点，则抛出异常
	if (element.nodeType !== 1) {
		throw new Error('posObject.element is invalid');
	}

	// 返回结果
	var result = {
		element: element,
		x: posObject.x || 0,
		y: posObject.y || 0
	}

  // 判断是否是可视区域
	var isVIEWPORT = (element === VIEWPORT || element._id === 'VIEWPORT');

  // 获取定位对象的 offset
  result.offset = function() {
    // 若定位 fixed 元素，则父元素的 offset 没有意义
    if (isPinFixed) {
    	return {
    		left: 0,
    		top: 0
    	};
    }
    else if (isVIEWPORT) {
    	return {
    		left: $(document).scrollLeft(),
    		top: $(document).scrollTop()
    	}
    }
    else {
    	return getOffset($(element)[0]);
    }
  }

  // 获取定位对象的 size, 含 padding 和 border
  result.size = function() {
    var el = isVIEWPORT ? $(window) : $(element);
    return {
    	width: el.outerWidth(),
    	height: el.outerHeight()
    }
  }

  return result;

}

// 返回 DOM 对象
function toElement(element) {
	return $(element)[0];
}
````

####　2. 参数转换 `left|center|right|%|px`

对 x, y 两个参数为 left|center|right|%|px 时的处理，全部处理为纯数字

````js
// 转换器
function posConverter(posObject) {
	posObject.x = xyConverter(posObject.x, posObject, 'width');
	posObject.y = xyConverter(posObject.x, posObject, 'height');
}

// 处理 x, y 值，都转化为数字
function xyConverter(x, posObject, type) {
	// 先转成字符串好处理
	x = x + '';

  // 处理 px
	x = x.replace(/px/gi, '');

  // 处理 alias 别名 转为 百分比
	if (/\D/.test(x)) {
		x = x.replace(/(?:top|left)/gi, '0%')
		     .replace(/center/gi, '50%')
		     .replace(/(?:bootom|right)/gi, '100%');
	}

	// 将百分比转为像素值
	if (x.indexOf('%') !== -1) {
		x = x.replace(/(\d+(?:\.\d))%/gi, function(m, d) {
      return posObject.size()[type] * (d / 100.0);
	  })
	}

	// 处理类似 100%+20px 的情况
  if (/[+\-*\/]/.test(x)) {
    try {
    	// eval 会影响压缩
      // new Function 方法效率高于 for 循环拆字符串的方法
      // 参照：http://jsperf.com/eval-newfunction-for
    	x = (new Function('return ' + x))();
    } catch (e) {
      throw new Error('Invalid position value: ' + x);
    }
  }

  return numberize(x);

}

function numberize(s) {
  return parseFloat(s, 10) || 0;
}
````

#### 3. 获取定位元素最近的祖先定位元素 offsetParent 

````js
function getParentOffset(element) {
	var parent = element.offsetParent();

  // IE7 下，body 子节点的 offsetParent 为 html 元素，其 offset 为
  // { top: 2, left: 2 }，会导致定位差 2 像素，所以这里将 parent
  // 转为 document.body
	if (parent[0] === document.documentElement) {
		parent = $(document.body)
	}

	// 修正 ie6 下 absolute 定位不准的 bug
	if (isIE6) {
		parent.css('zoom', 1);
	}

	// 获取 offsetParent 的 offset
	var offset;

	if (parent[0] === document.body && parent.css('position') === 'static') {
		offset = {top: 0, left: 0};
	}
	else {
		offset = getOffset(parent[0]);
	}

  // 根据基准元素 offsetParent 的 border 宽度，来修正 offsetParent 的基准位置
	offset.top += numberize(parent.css('border-top-width'));
	offset.left += numberize(parent.css('border-left-width'));

	return offset;
}

// fix jQuery 1.7.2 offset
// document.body 的 position 是 absolute 或 relative 时
// jQuery.offset 方法无法正确获取 body 的偏移值
//   -> http://jsfiddle.net/afc163/gMAcp/
// jQuery 1.9.1 已经修正了这个问题
//   -> http://jsfiddle.net/afc163/gMAcp/1/
// 这里先实现一份
// 参照 kissy 和 jquery 1.9.1
//   -> https://github.com/kissyteam/kissy/blob/master/src/dom/sub-modules/base/src/base/offset.js#L366
//   -> https://github.com/jquery/jquery/blob/1.9.1/src/offset.js#L28
function getOffset(element) {
	var box = element.getBoundingClientRect(),
	docElem = document.documentElement;

  // < ie8 不支持 win.pageXOffset, 则使用 docElem.scrollLeft
	return {
		left: box.left + (window.pageXOffset || docElem.scrollLeft) - (docElem.clientLeft || document.body.clientLeft || 0),
		top : box.top + (window.pageYOffset || docElem.scrollTop) - (docElem.clientTop || document.body.clientTop || 0);
	}
}
````

#### 4. 定位方法 `position.pin(pinObject, baseObject)`

将目标元素相对于基准元素进行定位,这是 Position 的基础方法，接收两个参数，分别描述了目标元素和基准元素的定位点

````js
Position.pin = function(pinObject, baseObject) {
	
  pinObject = normalize(pinObject);
  baseObject = normalize(baseObject);

  if (pinObject.element === VIEWPORT || pinObject.element._id === 'VIEWPORT') {
    return;
  }

  // 设定目标元素的 position 为绝对定位
  // 若元素的初始 position 不为 absolute，会影响元素的 display、宽高等属性
  var pinElement = $(pinObject.element);

  if (pinElement.css('position') !== 'fixed' || isIE6) {
    pinElement.css('position', 'absolute');
    isPinFixed = false;
  }
  else {
    // 定位 fixed 元素的标志位，下面有特殊处理
    isPinFixed = true;
  }

  // 将位置属性归一化为数值
  // 注：必须放在上面这句 `css('position', 'absolute')` 之后，
  // 否则获取的宽高有可能不对
  posConverter(pinObject);
  posConverter(baseObject);

  var parentOffset = getParentOffset(pinElement);
  var baseOffset = baseObject.offset();

  // 计算目标元素的位置
  var top = baseOffset.top + baseObject.y - pinObject.y - parentOffset.top;

  var left = baseOffset.left + baseObject.x - pinObject.x - parentOffset.left;

  // 定位目标元素
  pinElement.css({ left: left, top: top });

}

// 将目标元素相对于基准元素进行居中定位
// 接受两个参数，分别为目标元素和定位的基准元素，都是 DOM 节点类型
Position.center = function(pinElement, baseElement) {
    Position.pin({
        element: pinElement,
        x: '50%',
        y: '50%'
    }, {
        element: baseElement,
        x: '50%',
        y: '50%'
    });
};

// 这是当前可视区域的伪 DOM 节点
// 需要相对于当前可视区域定位时，可传入此对象作为 element 参数
Position.VIEWPORT = VIEWPORT;
````