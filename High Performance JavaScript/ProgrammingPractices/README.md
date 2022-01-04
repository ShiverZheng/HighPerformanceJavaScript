## 避免双重求值 Avoid Double Evaluation

JavaScript 像很多脚本语言一样，允许在程序中提取一个包含代码的字符串，然后动态执行它。
有四种标准方法可以实现：
1. `eval()`
2. `Function()`构造函数
3. `setTimeout()`
4. `setInterval()`

每个方法都允许传入一个JavaScript代码字符串并执行它，如下：
```js
var num1 = 5
var num2 = 6
//eval() evaluating a string of code
result = eval('num1 + num2'),

//Function() evaluating strings of code
sum = new Function('arg1', 'arg2', 'return arg1 + arg2')

//setTimeout() evaluating a string of code
setTimeout('sum = num1 + num2', 100)

//setInterval() evaluating a string of code
setInterval('sum = num1 + num2', 100)
```
当你在JavaScript代码中执行另一端JavaScript代码时，都会导致双重求值的性能消耗。此代码首先会以正常方式求值，然后执行过程中对于包含于字符串中的代码发起另一个求值运算。这是一项非常昂贵的操作，比直接包含代码执行速度慢很多。

## 使用Object/Array直接量 Use Object/Array Literals

使用对象和数组直接量是最快的创建对象和数组的方式：
```js
var obj = {
    name: 'sivan',
    age: 20
}

var array = ['sivan', '20']
```

## 不要重复工作 Don't Report Work
```js
function addHandler(target, eventType, handler) {
    // DOM2 Events
    if (target.addEventListener) {
        target.addEventListener(target, eventType, handler)
    // IE
    } else {
        target.attachEvent('on' + eventType, handler)
    }
}
```
代码的性能问题在于每次函数调用时都执行重复工作。每一次都进行同样的检查，看看某种方法是否存在。如果假设 target 唯一的值就是 DOM 对象，而且用户不可能在页面加载时魔术般地改变浏览器，那么这种判断就是重复的。如果`addHandler()`一上来 就调用`addEventListener()`那么每个后续调用都要出现这句代码。在每次调用中重复同样的工作是一种浪费，有多种办法避免这一点。

### 延迟加载 Lazy Loading

在上述例子中，在函数调用前，不需要判断使用哪种方法绑定事件处理器。使用延迟加载的函数如下:
```js
function addHandler(target, eventType, handler) {
    // overwrite the existing function
    if (target.addEventListener){ 
        addHandler = function(target, eventType, handler) {
            target.addEventListener(eventType, handler, false)
        }
    } else { 
        addHandler = function(target, eventType, handler) {
            target.attachEvent('on' + eventType, handler)
        }
    }
    //call the new function
    addHandler(target, eventType, handler)
}
```
这个函数按照延迟加载模式实现，它第一次被调用时，检查一次并决定使用哪种方法去绑定事件处理器。然后原始函数就被包含正确操作的新函数覆盖了，最后调用新函数并将原始参数传给它。

以后再调用 `addHandler()`时不会再次检测，因为检测代码已经被新函数覆盖了。

### 条件预加载 Conditional Advance Loading

它会在脚本加载期间提前检测，而不会等到函数调用。检测的操作依然只有一次，只是它在过程中来的更早，例如：
```js
var addHandler = document.body.addEventListener
? function(target, eventType, handler) {
    target.addEventListener(eventType, handler, false)
}
: function(target, eventType, handler) {
    target.attachEvent('on' + eventType, handler)
}
```
这个例子检查`addEventListener()`是否存在，然后根据此信息指定最合适的函数。如果存在，三元操作符返回 `DOM Level 2`的函数，否则返回 IE 特有的函数。这样做的结果是对`addHandler()`的调用十分快，因为检测条件提前发生了。

条件预加载确保所有函数调用时间相同。其代价是在脚本加载时进行检测，而不是加载后，预加载适用于一个函数马上就会被用到，而且在整个页面生命周期中经常使用的场合。

## 使用速度快的部分 Use the Fast Parts



### 位操作 Bitwise Operators

JavaScript 中的数字按照 IEEE-754 标准 64 位格式存储。在位运算中，数字被转换为有符号 32 位格式。 每次运算符会直接操作该 32 位数以得到结果。尽管需要转换，这个过程与 JavaScript 中其他数学运算和布尔操作相比还是快很多。

```js
var num1 = 25
var num2 = 3

alert(num1.toString(2)) // '11001'
alert(num2.toString(2)) // '11'
```
> 该表达式消隐了数字高位的零

- AND 按位与
> 两个操作数的对应位都是1时，则在该位返回1
- OR 按位或
> 两个操作数的对应位只要一个为1时，则在该位返回1
- XOR 按位异或
> 两个操作数的对应位只有一个为1时，则在该位返回1
- NOT 按位取反
> 遇0则返回1，反之亦然。


```js
//bitwise AND
var result1 = 25 & 3        // 1
alert(result.toString(2))   // '1'

//bitwise OR
var result2 = 25 | 3        // 27
alert(resul2.toString(2));  // '11011'

//bitwise XOR
var result3 = 25 ^ 3        // 26
alert(resul3.toString(2));  // '11000'

//bitwise NOT
var result = ~25            // -26
alert(resul2.toString(2));  // '-11010'
```

```js
for (var i = 0, len = rows.length; i < len; i++) {
    if (i % 2) {
        className = 'even';
    } else {
        className = 'odd';
    }
    //apply class
}
// 计算对 2 取模，需要用这个数除以 2 然后查看余数。
// 如果你看到 32 位数字的底层(二进制)表示法，你会发现偶数的最低位是0，奇数的最低位是1。
// 如果此数为偶数，那么它和1进行位与操作的结果就是0; 
// 如果此数为奇数，那么它和 1 进行位与操作的结果就是 1。
// 也就是说上面的代码可以重写如下:
for (var i = 0, len = rows.length; i < len; i++) {
    if (i & 1) {
        className = 'even';
    } else {
        className = 'odd';
    }
    //apply class
}
```

虽然代码改动不大，但位与版本比原始版本快了 50%(取决于浏览器)。

第二种使用位操作的技术称作位掩码。位掩码在计算机科学中是一种常用的技术，可同时判断多个布尔选项，快速地将数字转换为布尔标志数组。掩码中每个选项的值都等于 2 的幂。例如:

```js
var OPTION_A = 1
var OPTION_B = 2
var OPTION_C = 4
var OPTION_D = 8
var OPTION_E = 16
```

通过定义这些选项，你可以用位或操作创建一个数字来包含多个选项:

```js
var options = OPTION_A | OPTION_C | OPTION_D

// is option A in the list?
if (options & OPTION_A) {
    // do something
}

// is option B in the list?
if (options & OPTION_B) {
    // do something
}
```

### 原生方法 Native Methods

无论你的JavaScript代码如何优化，都永远不会比JavaScript引擎提供的原生方法更快。其原因十分简单: JavaScript 的原生部分在你写代码之前它们已经存在于浏览器之中了，都是用低级语言写的，诸如 C++。这意味着这些方法被编译成机器码，作为浏览器的一部分，不像你的 JavaScript 代码那样有那么多限制。

## 总结 Summary

JavaScript 提出了一些独特的性能挑战，关系到你组织代码的方法。网页应用变得越来越高级，包含的 JavaScript 代码越来越多，出现了一些模式和反模式。请牢记以下编程经验:
1. 通过避免使用 `eval()`和 `Function()`构造器避免二次评估。此外，给`setTimeout()`和`setInterval()`传递函数参数而不是字符串参数。
2. 创建新对象和数组时使用对象直接量和数组直接量。它们比非直接量形式创建和初始化更快。
3. 避免重复进行相同工作。当需要检测浏览器时，使用延迟加载或条件预加载。
4. 当执行数学远算时，考虑使用位操作，它直接在数字底层进行操作。
5. 原生方法总是比 JavaScript 写的东西要快。尽量使用原生方法。