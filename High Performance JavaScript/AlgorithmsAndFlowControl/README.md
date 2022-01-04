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

## 条件语句 Conditionals

### if-else 对比 switch if-else Versus switch

通常来说，if-else适用于判断两个离散值或几个不同的值域。当判断多于两个离散值时，switch语句是更佳选择。

### 优化if-else Optimizing if-else 

优化if-else的目标是：最小化到达正确分支前所需判断的条件数量。最简单的优化方法是确保最可能出现的条件放在首位。考虑如下代码：
```js
	// 只有当value值经常小于5的时候才是最优的
	// 如果value大于5或者等于10，那么每次到达正确分支之前必须经过两个条件判断，最终增加了这个语句所消耗的平均时间。
	if (v < 5) {
		// do something
	} else if (v > 5 && v < 10) {
		// do something
	} else { 
		// do something
	}
```

> if-else中的条件语句应该总是按照从最大概率到最小概率的顺序排列，以确保运行速度最快。

```js
if (v < 6) {
	if (v < 3) {
		if (v == 0) {
			return result0
		} else if (value == 1) {
			return result1
		}
		return result2
	} else {
		if (v == 3) {
			return result3
		} else if (v == 4) {
			return result4
		}
		return result5
	}
} else {
	if (v < 8) {
		if (v == 6) {
			return result6
		} else {
			return result7
		}
	} else {
		if (v == 8) {
			return result8
		} else if (v == 9) {
			return result9
		} else {
			return result10
		}
	}
}
```
> 使用二分法把值域分成一系列的区间，然后逐步缩小范围。当值的范围均匀分布在0到10之间时，代码运行的平均时间大约是每个条件都判断的一半。这个方法非常适用于有多个值域需要测试的时候（如果是离散值，那么switch语句通常更为合适）。

### 查找表 Lookup Tables

有些时候优化条件语句的最佳方案是避免使用if-else和switch。

当有大量离散值需要测试时，if-else和switch都比使用查找表慢很多。JavaScript中可以用数组和普通对象来构造查找表，通过查找表访问数据比用if-else或switch快很多，特别是在条件语句数量很大的时候

当单个键和单个值之间存在逻辑映射时，查找表的优势就能体现出来。switch语句更适合于每个键都需要对应一个独特的动作或一系列动作的场合。

## 递归 Recursion

### 调用栈限制 Call Stack Limits

JavaScript引擎支持的递归数量与JavaScript调用栈大小直接相关。只有IE例外，它的调用栈与系统空闲内存有关，而其他所有浏览器都有固定数量的调用栈限制。大多数现代浏览器的调用栈数量比老版本浏览多出很多（比如Safari 2的调用栈大小只有100）。

### 递归模式 Recursion Patterns

常见的导致栈溢出的原因是不正确的终止条件，因此定位模式错误的第一步是验证终止条件。如果终止条件没问题，那么可能是算法中包含了太多层递归，为了能在浏览器中安全地工作，建议改用迭代或`Memoization`，或者结合两者使用。

### 迭代 Iteration

任何递归能实现的算法也可以同样用迭代来实现。迭代算法通常包含几个不同的循环，分别对应计算过程的不同方面，这也会引入它们自身的性能问题。然而，使用优化后的循环替代长时间运行的递归函数可以提升性能，因为运行一个循环比反复调用一个函数的开销要少得多。

```js
// 递归

// 对两个有序数组排列
function merge(left, right) {
	var result = []
	while (left.length > 0 && right.length > 0) {
		// 因为数组是有序的，所以每次只用对第一个元素进行比较
		// 始终把两个数组中最小的放入结果
		if (left[0] < right[0]) {
			result.push(left.shift())
		} else {
			result.push(right.shift())
		}
	}
	return result.concat(left).concat(right)
}

// 不断拆分成最小单位
function mergeSort(items) {
	if (items.length == 1) return items
	var middle = Math.floor(items.length / 2)
	var left = items.slice(0, middle)
	var right = items.slice(middle)
	return merge(mergeSort(left), mergeSort(right))
}
```
> 合并排序的代码相当简单直观，但是`mergeSort()`函数会导致很频繁的自调用。一个长度为n的数组最终会调用`mergeSort()` **2*n-1** 次，这意味着一个长度超过1500的数组会在Firefox上发生栈溢出错误。

```js
// 迭代
// 使用相同的merge函数
function mergeSort(items){
    var len = items.length
    if(len == 1) {
        return items
    }
    var result = []
    for(var i = 0; i < len; i++) {
        result.push([items[i]])
    }
    // 如果数组长度为奇数
    if(len % 2) {
        result.push([])
    }
    var lim = len / 2
    while(lim >= 1) {
        for(var j = 0, k = 0; j < lim; j++, k = k + 2) {
            result[j] = merge(result[k], result[k + 1])
        }
        lim = lim / 2;
    }
    return result[0];
}
```

### Memoization

Memoization正是一种避免重复工作的方法，它缓存前一个计算结果供后续计算使用，避免了重复工作。这使得它成为递归算法中有用的技术。

```js
function fatorial(n) {
	if (n == 0) return 1
	return n * fatorial(n - 1)
}

// better
function memfactorial(n) {
	if (!memfactorial.cache) {
		memfactorial.cache = {
			'0': 1,
			'1': 1
		}
	}

	if (!memfactorial.cache.hasOwnProperty(n)) {
		memfactorial.cache[n] = n * memfactorial(n -1)
	}

	return memfactorial.cache[n]
}
```

## 小结 Summary

如同其他编程语言，代码的写法和算法会影响JavaScript的运行时间。与其他语言不同的是，JavaScript可用资源有限，因此优化技术更为重要。
- for、while和do-while循环性能特性相似，所以没有一种循环类型明显快于或慢于其他类型。
- 避免使用for-in循环，除非你需要遍历一个属性数量未知的对象。
- 改善循环性能的最佳方式是减少每次迭代的运算量和减少循环迭代次数。
- 通常来说，switch总是比if-else快，但并不总是最佳解决方案。
- 在判断条件较多时，使用查找表比if-else和switch更快。
- 浏览器的调用栈大小限制了递归算法在JavaScript中的应用；栈溢出错误会导致其他代码中断运行。
- 如果你遇到栈溢出错误，可将方法改为迭代算法，或使用Memoization来避免重复计算。

运行的代码数量越大，使用这些策略所带来的性能提升也就越明显。