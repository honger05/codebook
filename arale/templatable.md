
处理组件的模板渲染，混入到 Widget 中使用。

Templatable 在渲染的时候会读取 this.get('model') 和 this.get('template')，这两个是实例化的时候传入的，最终生成 this.element。

this.get('template') 支持多种格式：

1. html 的字符串

2. id 选择器，最常用而且判断简单

3. 函数，通过 handlerbars 编译过的模板

### 源码分析

提供 Template 模板支持，默认引擎是 Handlebars

````js
// Handlebars 的 helpers
templateHelpers: null,

// Handlebars 的 partials
templatePartials: null,

// template 对应的 DOM-like object
templateObject: null,

// 根据配置的模板和传入的数据，构建 this.element 和 templateElement
parseElementFromTemplate: function() {
	// template 支持 id 选择器
	var t, template = this.get('template');
	if (/^#/.test(template) && (t = document.getElementById(template.substring(1)))) {
	  template = t.innerHTML;
	  this.set('template', template);
  }
  this.templateObject = convertTemplateToObject(template);
  this.element = $(this.compile());
}
````

将 template 字符串转换成对应的 DOM-like object

````js
function convertTemplateToObject(template) {
	return isFunction(template) ? null : $(encode(template));
}
`````

根据 selector 得到 DOM-like template object，并转换为 template 字符串

````js
function converObjectToTemplate(templateObject, selector) {
	if (!templateObject) return;

	var element;
	if (selector) {
		element = templateObject.find(selector);
		if (element.length === 0) {
			throw new Error('Invalid template selector: ' + selector);
		}
	}
	else {
		element = templateObject;
	}
  
  return decode(element.html());

}
````

一些工具类

````js
var _compile = Handlebars.compile;

// 增加编译的参数是 function 的情况
Handlebars.compile = function(template) {
	return isFunction(template) ? template : _compile.call(Handlebars, template);
}

function encode(template) {
	return template
	// 替换 {{xxx}} 为 <!-- {{xxx}} -->
	.replace(/({[^}]+}})/g, '<!--$1-->')
	// 替换 src="{{xxx}}" 为 data-TEMPLATABLE-src="{{xxx}}"
  .replace(/\s(src|href)\s*=\s*(['"])(.*?\{.+?)\2/g, ' data-templatable-$1=$2$3$2');
}

function decode(template) {
  return template.replace(/(?:<|&lt;)!--({{[^}]+}})--(?:>|&gt;)/g, '$1').replace(/data-templatable-/ig, '');
}

// 预编译
function precompile(partials) {
  if (!partials) return {};

  var result = {};
  for (var name in partials) {
    var partial = partials[name];
    result[name] = isFunction(partial) ? partial : Handlebars.compile(partial);
  }
  return result;
};
````

编译模板，混入数据，返回 html 结果

````js
compile: function(template, model) {
  template || (template = this.get('template'));

  model || (model = this.get('model')) || (model = {});
  if (model.toJSON) {
    model = model.toJSON();
  } 

  // handlebars runtime，注意 partials 也需要预编译
  if (isFunction(template)) {
    return template(model, {
      helpers: this.templateHelpers,
      partials: precompile(this.templatePartials)
    });
  } else {
    var helpers = this.templateHelpers;
    var partials = this.templatePartials;
    var helper, partial;

    // 注册 helpers
    if (helpers) {
      for (helper in helpers) {
        if (helpers.hasOwnProperty(helper)) {
          Handlebars.registerHelper(helper, helpers[helper]);
        }
      }
    }
    // 注册 partials
    if (partials) {
      for (partial in partials) {
        if (partials.hasOwnProperty(partial)) {
          Handlebars.registerPartial(partial, partials[partial]);
        }
      }
    }

    var compiledTemplate = compiledTemplates[template];
    if (!compiledTemplate) {
      compiledTemplate = compiledTemplates[template] = Handlebars.compile(template);
    }

    // 生成 html
    var html = compiledTemplate(model);

    // 卸载 helpers
    if (helpers) {
      for (helper in helpers) {
        if (helpers.hasOwnProperty(helper)) {
          delete Handlebars.helpers[helper];
        }
      }
    }
    // 卸载 partials
    if (partials) {
      for (partial in partials) {
        if (partials.hasOwnProperty(partial)) {
          delete Handlebars.partials[partial];
        }
      }
    }
    return html;
  }
},
````

局部刷新 selector 指定区域

````js
renderPartial: function (selector) {
  if (this.templateObject) {
    var template = convertObjectToTemplate(this.templateObject, selector);

    if (template) {
      if (selector) {
        this.$(selector).html(this.compile(template));
      } else {
        this.element.html(this.compile(template));
      }
    } else {
      this.element.html(this.compile());
    }
  }

  // 如果 template 已经编译过了，templateObject 不存在
  else {
    var all = $(this.compile());
    var selected = all.find(selector);
    if (selected.length) {
      this.$(selector).html(selected.html());
    } else {
      this.element.html(all.html());
    }
  }

  return this;
}
````