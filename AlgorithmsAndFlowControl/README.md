## 循环 Loops

### 循环的类型 Types of Loops
- **标准for循环**:

它由四部分组成：初始化、前测条件、后执行体、循环体。当代码运行中遇到for循环时，先运行初始化代码，然后进入前测条件。如果前测条件的结果为true，则运行循环体。循环体执行完后，后执行代码开始运行。

  > 请注意，在for循环初始化中的var语句会创建一个函数级变量，而不是循环级。由于JavaScript只有函数级作用域，因此在for循环中定义一个新变量等同于在循环体外定义一个新变量。
```js
	for (var i = 0; i < 10; i++) {
		// loop body
	}
```
- **while循环**:

第二种循环类型是 while循环。while循环是最简单的前测循环，由一个前测条件和一个循环体构成：
```js
	var i = 0
	while(i < 10) {
		// loop body
		i++
	}
```
在循环体运行前，先计算前测条件。如果计算结果为true，就运行循环体；否则，循环体会被跳过。任何for循环都能改写成while循环，反之亦然。

- **do-while循环**:

它是唯一一种后测循环，它由两部分组成，循环体和后测条件：
```js
	var i = 0
	do {
		// loop body
	} while (i++ < 10)
```
> 在do-while循环中，循环体会至少运行一次，而后再由后测条件决定是否再次运行。

- **for-in循环**:
  
可以枚举任何对象的属性名：
```js
	for (var prop in object) {
		// loop body
	}
```
循环体每次运行时，prop变量被赋值为object的一个属性名（字符串），直到所有属性遍历完成才返回。所返回的属性包括对象实例属性以及从原型链中继承而来的属性。

### 循环性能 Loop Performance

在JavaScript提供的四种循环类型中，只有for-in循环比其他几种明显要慢。

**减少迭代的工作量**
```js
	for (var i = 0; i < items.length; i++) {
		process(items[i])
	}

	var j = 0
	while (j < items.length) {
		process(items[j++])
	}

	var k = 0
	do {
		process(items[k++])
	} while (k < items.length)
```
上面的每个循环中，每次运行循环体时都会产生如下操作：
1. 一次控制条件中的属性查找`items.length`
1. 一次控制条件中的数值大小比较`i < items.length`
1. 一次控制条件结果是否为true的比较`i < items.length == true`
1. 一次自增操作`i++`
1. 一次数组查找`items[i]`
1. 一次函数调用`process(items[i])`

在这些简单循环中，尽管代码不多，但每次迭代都要进行许多操作。代码运行速度很大程度上取决于函数`process()`对每个数组项的操作，尽管如此，减少每次迭代中的操作总数能大幅提高循环的总体性能。

例子中每次循环都要查找`items.length`。这样做很耗时，由于该值在循环运行过程中从未改变，因此产生了不必要的性能损失。提高这个循环的性能很简单，只查找一次属性，并把值存储到一个局部变量，然后在控制条件中使用这个变量：

```js
	// minimizing property lookups
	for (var i = 0, len = items.length; i < len; i++) {
		process(items[i])
	}
	var j = 0
	var count = items.length
	
	while (j < count) {
		process(items[j++])
	}
	var k = 0
	var num = items.length
	do {
		process(items[k++])
	} while (k < num)
```
重写后的循环只在循环运行前对数组长度进行一次属性查找。这使得控制条件可直接读取局部变量，所以速度更快。根据数组的长度，在大多数浏览器中能节省大概25%的运行时间（IE中甚至可以节省50%）。

**减少属性查找并反转**
```js
	// 本例中使用了倒序循环，并把减法操作整合在控制条件中。
	// 现在每个控制条件只是简单地与零比较。
	// 控制条件与true值比较，任何非零数会自动转换为true，而零值等同于false。
	for (var i = items.length; i--) {
		process(items[i])
	}
	var j = items.length
	while (j--) {
		process(items[j])
	}
	var k = items.length - 1
	do {
		process(items[k])
	} while (k--)
```

控制条件从两次比较 **迭代数少于总数吗？它是否为true?** 减少到一次比较 **它是true吗？**。

每次迭代从两次比较减少到一次，进一步提高了循环速度。通过倒序循环和减少属性查找，你可以看到运行速度比原始版本快了50%～60%。
对比原始版本，每次迭代中只有如下操作：
1. 一次控制条件中的比较`i==true`
1. 一次减法操作`i--`
1. 一次数组查找`items[i]`
1. 一次函数调用`process(items[i])`

新的循环代码每次迭代中减少了两次操作，随着迭代次数增加，性能的提升会更趋明显。


**减少迭代次数**

达夫设备(Duff's Device)
```js
// 优化版
// creadit: Jeff Greenberg
var i = items.length % 8
while (i) {
	process(items[i--])
}
i = Math.floor(items.length / 8)

while (i) {
	process(item[i--])
	process(item[i--])
	process(item[i--])
	process(item[i--])
	process(item[i--])
	process(item[i--])
	process(item[i--])
	process(item[i--])
}
```

如果迭代次数大于1000， 那么达夫设备的执行效率将明显提升，例如在500000次迭代中，其运行时间比常规循环少70%。


### 基于函数的迭代

```js
	items.forEach(function(value, index, array) {
		process(value)
	})
```
尽管基于函数的迭代提供了一个更为便利的迭代方法，但它仍然比基于循环的迭代要慢一些。

对每个数组项调用外部方法所带来的开销是速度慢的主要原因。

在所有情况下，基于循环的迭代比基于函数的迭代快8倍，因此在运行速度要求严格时，基于函数的迭代不是合适的选择。