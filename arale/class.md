### 先谈谈基于原型的继承。

先看看 segementfault 上讨论的一道题。

````js
function F() {}
Object.prototype.a = function() {}
Function.prototype.b = function() {}
var f = new F()
// F.a F.b f.a
````


