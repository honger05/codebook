### _.keys (object)             

检索object拥有的所有可枚举属性的名称

````js
_.keys = function(obj) {
	if (!_.isObject(obj)) return [];
	if (nativeKeys) return nativeKeys(obj);
	var keys = [];

	// 避免遍历到父类的 keys
	for (var key in obj) if (_.has(obj, key)) keys.push(key);
	return keys;
}
````

示例

````js
_.keys({one: 1, two: 2, three: 3});
=> ["one", "two", "three"]
````

--- 
### _.values (object)              

返回object对象所有的属性值。

````js
_.values = function(obj) {
	var keys = _.keys(obj);
	var length = keys.length;
	var values = new Array(length);
	for (var i = 0; i < length; i++) {
    values[i] = obj[keys[i]]
  }
  return values;
}
````

示例

````js
_.values({one: 1, two: 2, three: 3});
=> [1, 2, 3]
````

--- 
### _.pairs (object)                

把一个对象转变为一个[key, value]形式的数组。

````js
_.pairs = function(obj) {
  var keys = _.keys(obj);
  var length = keys.length;
  var pairs = new Array(length);
  for (var i = 0; i < length; i++) {
    pairs[i] = [keys[i], obj[keys[i]]];
  }
  return pairs;
};
````

示例

````js
_.pairs({one: 1, two: 2, three: 3});
=> [["one", 1], ["two", 2], ["three", 3]]
````