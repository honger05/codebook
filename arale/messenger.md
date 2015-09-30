
跨域 Iframe 通信解决方案，兼容主流和 IE 系列浏览器。

````js
// 初始化父页面的信使
var messenger = new Messenger('parent', 'MessengerProject');

// 绑定子页面 iframe
messenger.addTarget(iframe1.contentWindow, 'iframe1');
messenger.addTarget(iframe2.contentWindow, 'iframe2');

// 给子页面发消息
messenger.targets['iframe1'].send('发给子页面1的消息');

// 或者给所有子页面发消息
messenger.send('发给所有子页面的消息');

// 初始化子页面的信使
// 注意，第二个参数 `MessengerProject` 必须和父页面的信使保持一致，
var messenger = new Messenger('iframe1', 'MessengerProject');

// 绑定父页面
messenger.addTarget(window.parent, 'parent');

// 给父页面发消息
messenger.targets['parent'].send('发给父页面的消息');

/ iframe中 - 监听消息
// 回调函数按照监听的顺序执行
messenger.listen(function(msg){
    alert("收到消息: " + msg);
});
````

### 源码分析

````js
// 消息前缀, 建议使用自己的项目名, 避免多项目之间的冲突
var prefix = "arale-messenger";

// IE8+ 以及现代浏览器支持 postMessage
var supportPostMessage = 'postMessage' in window;
````

Target 类, 消息对象

````js
function Target(target, name){
  var errMsg = '';
  if(arguments.length < 2){
      errMsg = 'target error - target and name are both required';
  } else if (typeof target != 'object'){
      errMsg = 'target error - target itself must be window object';
  } else if (typeof name != 'string'){
      errMsg = 'target error - target name must be string type';
  }
  if(errMsg){
      throw new Error(errMsg);
  }
  this.target = target;
  this.name = name;
}
````

往 target 发送消息, 出于安全考虑, 发送消息会带上前缀

````js
if ( supportPostMessage ){
  // IE8+ 以及现代浏览器支持
  Target.prototype.send = function(msg){
      this.target.postMessage(prefix + msg, '*');
  };
} else {
  // 兼容IE 6/7
  Target.prototype.send = function(msg){
      var targetFunc = window.navigator[prefix + this.name];
      if ( typeof targetFunc == 'function' ) {
          targetFunc(prefix + msg, window);
      } else {
          throw new Error("target callback function is not defined");
      }
  };
}
````

信使类 创建Messenger实例时指定, 必须指定Messenger的名字, (可选)指定项目名, 以避免Mashup类应用中的冲突

!注意: 父子页面中projectName必须保持一致, 否则无法匹配

````js
function Messenger(messengerName, projectName){
  this.targets = {};
  this.name = messengerName;
  this.listenFunc = [];
  prefix = projectName || prefix;
  this.initListen();
}

// 添加一个消息对象
Messenger.prototype.addTarget = function(target, name){
  var targetObj = new Target(target, name);
  this.targets[name] = targetObj;
};

// 初始化消息监听
Messenger.prototype.initListen = function(){
  var self = this;
  var generalCallback = function(msg){
    if(typeof msg == 'object' && msg.data){
      msg = msg.data;
    }
    // 剥离消息前缀
    msg = msg.slice(prefix.length);
    for(var i = 0; i < self.listenFunc.length; i++){
      self.listenFunc[i](msg);
    }
  };

  if ( supportPostMessage ){
    if ( 'addEventListener' in document ) {
      window.addEventListener('message', generalCallback, false);
    } else if ( 'attachEvent' in document ) {
      window.attachEvent('onmessage', generalCallback);
    }
  } else {
    // 兼容IE 6/7
    window.navigator[prefix + this.name] = generalCallback;
  }
};

// 监听消息
Messenger.prototype.listen = function(callback){
  this.listenFunc.push(callback);
};
// 注销监听
Messenger.prototype.clear = function(){
  this.listenFunc = [];
};
// 广播消息
Messenger.prototype.send = function(msg){
  var targets = this.targets,
      target;
  for(target in targets){
    if(targets.hasOwnProperty(target)){
      targets[target].send(msg);
    }
  }
};

return Messenger;

````