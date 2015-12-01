### _.first (array, [n]) 

返回array（数组）的第一个元素。传递 n参数将返回数组中从第一个元素开始的n个元素。

````js
_.first = _.head = _.take = function(array, n, guard) {
	// 处理特殊情况 void 修饰的 都返回 undefined
	if (array == null) return void 0;
	if ((n == null) || guard) return array[0];
	if (n < 0) return [];

	// 截取 0 - n 个元素
	return slice.call(array, 0, n);
}
````

示例

````js
_.first([5, 4, 3, 2, 1]);
=> 5
````

--- 
### _.initial (array, [n])

返回数组中除了最后一个元素外的其他全部元素。 在arguments对象上特别有用。传递 n参数将从结果中排除从最后一个开始的n个元素

````js
_.initial = function(array, n, guard) {
	return slice.call(array, 0, Math.max(0, array.length - (n == null || guard ? 1 : n)));
}
````

示例

````js
_.initial([5, 4, 3, 2, 1]);
=> [5, 4, 3, 2]
````

--- 
### _.last (array, [n]) 

返回array（数组）的最后一个元素。传递 n参数将返回数组中从最后一个元素开始的n个元素

````js
_.last = function(array, n, guard) {
	if (array == null) return void 0;
	if ((n == null) || guard) return array[array.length - 1];
	return slice.call(array, Math.max(array.length - n, 0));
}
````

示例

````js
_.last([5, 4, 3, 2, 1]);
=> 1
````

--- 
### _.rest (array, [index])  

返回数组中除了第一个元素外的其他全部元素。传递 index 参数将返回从index开始的剩余所有元素 。Especially useful on the arguments object

````js
_.rest = _.tail = _.drop = function(array, n, guard) {
	return slice.call(array, (n == null) || guard ? 1 : n);
}
````

示例

````js
_.rest([5, 4, 3, 2, 1]);
=> [4, 3, 2, 1]
````

--- 
### _.compact (array)  

返回一个除去所有false值的 array副本。 在javascript中, false, null, 0, "", undefined 和 NaN 都是false值.

````js
_.compact = function(array) {
	return _.filter(array, _.identity);
}
````

示例

````js
_.compact([0, 1, false, 2, '', 3]);
=> [1, 2, 3]
````