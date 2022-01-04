## 浏览器UI线程 The Browser UI Thread

### 浏览器限制 Browser Limits

浏览器在 JavaScript 运行时间上采取了限制。这是一个有必要的限制，确保恶意代码编写者不能通过无 尽的密集操作锁定用户浏览器或计算机。

此类限制有两个:
1. 调用栈尺寸限制
    >第四章讨论过
1. 长运行脚本限制
    > 有时被称作长运行脚本定时器或者失控脚本定时器，但其基本思想是浏览器记录 一个脚本的运行时间，一旦到达一定限度时就终止它。

有两种方法测量脚本的运行时间。
1. 统计自脚本开始运行以来执行过多少语句。
   > 此方法意味着脚本在不同的机器上可能会运行不同的时间长度，可用内存和CPU速度可以影响一条独立语句运行所花费的时间。

1. 统计脚本运行的总时间。
   > 在特定时间内可运行的脚本数量也因用户机器性能差异而不同，但脚本总是停在固定的时间上。毫不奇怪，每个浏览器对长运行脚本检查方法上略有不同。


### 多久才算“太久”？How Long Is Too Long?

即使一秒钟对脚本 运行来说也太长了。一个单一的 JavaScript 操作应当使用的总时间(最大)是 100 毫秒。这个数字根据 Robert Miller 在 1968 年的研究。有趣的是，可用性专家 Jakob Nielsen 在他的著作《可用性工程》(Morgan Kaufmann， 1944)上注释说这一数字并没有因时间的推移而改变，而且事实上在 1991 年被 Xerox-PARC(施乐公司的 帕洛阿尔托研究中心)的研究中重申。

Nielsen 指出如果该接口在 100 毫秒内响应用户输入，用户认为自己是“直接操作用户界面中的对象。” 超过 100 毫秒意味着用户认为自己与接口断开了。由于 UI 在 JavaScript 运行时无法更新，如果运行时间长 于 100 毫秒，用户就不能感受到对接口的控制。

### 用定时器让出时间片 Yielding with Timers

#### 定时器基础 Timers Basics

定时器与 UI 线程交互的方式有助于分解长运行脚本成为较短的片断。调用 setTimeout()或 setInterval() 告诉 JavaScript 引擎等待一定时间然后将 JavaScript 任务添加到 UI 队列中。例如:

```js
var button = document.getElementById("my-button")
button.onclick = function() {
    oneMethod()
    setTimeout(
        function() {
            document.getElementById("notice").style.color = "red"
        },
        250,
    )
}
```
以上例子中调用一个方法然后设置一个定时器。用于修改`notice`元素颜色的代码被包含在一个定时器设备中，将在`250ms`之后添加到队列。`250ms`从调用`setTimeout()`时开始计算，**而不是从整个函数运行结束时**开始计算。如果`setTimeout()`在时间点`n`上被调用，那么运行定时器代码的`JavaScript `任务将在`n+250`的时刻加入UI队列。

> 一次事件循环中，先执行宏任务队列里的一个任务，再把微任务队列里的所有任务执行完毕，再去宏任务队列取下一个宏任务执行。在当前的微任务没有执行完成时，是不会执行下一个宏任务的。

setTimeout 和 setInterval的运行机制是将指定的代码移出本次执行，等到下一轮 Event Loop 时，再检查是否到了指定时间。如果到了，就执行对应的代码；如果不到，就等到再下一轮 Event Loop 时重新判断。
这意味着，setTimeout指定的代码，必须等到本次执行的所有同步代码都执行完，才会执行。


```js
// 只会输出A
// 同步队列输出A之后，陷入while(true){}的死循环中，异步任务不会被执行。
console.log('A');
setTimeout(function () {
    console.log('B');
}, 0);
while (true) {}
```
#### 定时器精度 Timer Precision 

HTML5 标准规定了setTimeout()的第二个参数的最小值，即最短间隔，不得低于4毫秒。如果低于这个值，就会自动增加。在此之前，老版本的浏览器都将最短间隔设为10毫秒。

### 分解任务 Splitting Up Tasks

我们通常将一个任务分解成一系列子任务。如果一个函数运行时间太长，那么查看它是否可以分解成一 系列能够短时间完成的较小的函数。

此处理过程必须是同步处理吗?
数据必须按顺序处理吗?
如果这两个回答都是“否”，那么代码将适于使用定时器分解工作。

可将一行代码简单地看作一个原子任务，多行代码组合在一起构成一个独立任务。某些函数可基于函数调用进行拆分。例如:
```js
function saveDocument(id){
    //save the document
    openDocument(id)
    writeText(id)
    closeDocument(id)
    //update the UI to indicate success
    updateUI(id)
}
```
如果函数运行时间太长，它可以拆分成一系列更小的步骤，把独立方法放在定时器中调用。你可以将每 个函数都放入一个数组，然后使用前一节中提到的数组处理模式:
```js
function saveDocument(id){
    var tasks = [openDocument, writeText, closeDocument, updateUI]
    setTimeout(
        function() {
            //execute the next task
            var task = tasks.shift()
            task(id)
            //determine if there's more
            if (tasks.length > 0) setTimeout(arguments.callee, 25)
        },
        25,
    )
}
```
可以进行封装复用：
```js
function multiStep(steps, args, callback) {
    // clone the array
    var tasks = steps.concat()
    setTimeout(
        function() {
            // execute the next task
            var task = task.shift()
            task.apply(null, args || [])
            // determine if there is more
            if (tasks.length > 0) {
                setTimeout(arguments.callee, 25)
            } else {
                callback()
            }
        },
        25
    )
}
```

### 限时运行代码 Timed Code
```js
function timedProcessArray(items, process, callback) {
    var todo = items.concat()
    //create a clone of the original
    setTimeout(
        function() {
            var start = +new Date()
            do {
                process(todo.shift())
            } while (todo.length > 0 && (+new Date() - start < 50))

            if (todo.length > 0) {
                setTimeout(arguments.callee, 25)
            } else {
                callback(items)
            }
        },
        25
    )
}
```
此函数中添加了一个`do-while`循环，它在每个数组项处理之后检测时间。定时器函数运行时数组中存放了至少一个项，所以后测试循环比前测试更合理。在`Firefox 3`中，如果`process()`是一个空函数，处理一个 1000 个项的数组需要 38 - 34 毫秒;
原始的`processArray()`函数处理同一个数组需要超过 25000 毫秒。这就 是定时任务的作用，避免将任务分解成过于碎小的片断。

### 定时器与性能 Timers and Performance

网页应用中限制高频率重复定时器的数量。同时，建议创建一个单独的重复定时器，每次执行多个操作。

## 网页工人线程 Web Workers

网页工人线程修改 `DOM` 将导致用户界面出错，但每个网页工人线程都有自己的全局运行环境，只有 `JavaScript` 特性的一个子集可用。工人线程的运行环境由下列部分组成:
- 一个浏览器对象，只包含四个属性:`appName`, `appVersion`, `userAgent`, 和 `platform`
- 一个 `location` 对象(和 `window` 里的一样，只是所有属性都是只读的)
- 一个 `self` 对象指向全局工人线程对象
- 一个 `importScripts()`方法，使工人线程可以加载外部 `JavaScript` 文件
- 所有 `ECMAScript` 对象，诸如 `Object`，`Array`，`Data`，等等。
- `XMLHttpRequest` 构造器
- `setTimeout()`和 `setInterval()`方法
- `close()`方法可立即停止工人线程

因为网页工人线程有不同的全局运行环境，你不能在 `JavaScript` 代码中创建。事实上，你需要创建一个 **完全独立的 `JavaScript` 文件**，包含那些在工人线程中运行的代码。要创建网页工人线程，你必须传入这个 `JavaScript` 文件的 `URL`:
```js 
var worker = new Worker("code.js");
```
此代码一旦执行，将为指定文件创建一个新线程和一个新的工人线程运行环境。此文件被**异步下载**，直到下载并运行完之后才启动工人线程。

### 工人线程交互 Worker Communication

网页代码可通过 `postMessage()`方法向工人线程传递数据，它接收单个递给工人线程的数据。
在工人线程中还有 `onmessage` 事件句柄用于接收信息。 例如:

```js
var worker = new Worker("code.js")
worker.onmessage = function(event) {
    alert(event.data)
}
worker.postMessage("Nicholas")
```

工人线程从 message 事件中接收数据。这里定义了一个 `onmessage` 事件句柄，事件对象具有一个 `data` 属 性存放传入的数据。工人线程可通过它自己的 `postMessage()` 方法将信息返回给页面。
```js
//inside code.js
self.onmessage = function(event) {
    self.postMessage("Hello, " + event.data + "!")
}
```

只有某些类型的数据可以使用 `postMessage()`传递，可以传递原始值(string，number，boolean，null 和 undefined)，也可以传递 Object 和 Array 的实例。

### 加载外部文件 Loading External Files

工人线程通过 `importScripts()` 方法加载外部 `JavaScript` 文件，它接收一个或多个 `URL` 参数（`JavaScript` 文件网址）。工人线程以阻塞方式调用 `importScripts()`，直到所有文件加载完成并执行之后， 脚本才继续运行。

```js
//inside code.js
importScripts("file1.js", "file2.js")
self.onmessage = function(event) {
    self.postMessage("Hello, " + event.data + "!")
}
```

### 实际用途 Practical Uses

网页工人线程适合于那些纯数据的，或者与浏览器 UI 没关系的长运行脚本。它看起来用处不大，而网 页应用程序中通常有一些数据处理功能将受益于工人线程，而不是定时器。

```js
var worker = new Worker("jsonparser.js")

//when the data is available, this event handler is called
worker.onmessage = function(event) {
    //the JSON structure is passed back
    var jsonData = event.data
    //the JSON structure is used
    evaluateData(jsonData)
}

//pass in the large JSON string to parse
worker.postMessage(jsonText)
```

```js
//inside of jsonparser.js
//this event handler is called when JSON data is available
self.onmessage = function(event) {
    //the JSON string comes in as event.data
    var jsonText = event.data
    //parse the structure
    var jsonData = JSON.parse(jsonText)
    //send back to the results
    self.postMessage(jsonData)
}
```

解析一个大字符串只是许多受益于网页工人线程的任务之一。其它可能受益的任务如下:
- 编/解码一个大字符串
- 复杂数学运算(包括图像或视频处理)
- 给一个大数组排序
任何超过 100 毫秒的处理，都应当考虑工人线程方案是不是比基于定时器的方案更合适。当然，还要基于浏览器是否支持工人线程。

## 总结 Summary

- `JavaScript` 运行时间不应该超过 100 毫秒。过长的运行时间导致 `UI` 更新出现可察觉的延迟，从而对整体用户体验产生负面影响。
- `JavaScript` 运行期间，浏览器响应用户交互的行为存在差异。无论如何，`JavaScript` 长时间运行将导致用户体验混乱和脱节。
- 定时器可用于安排代码推迟执行，它使得你可以将长运行脚本分解成一系列较小的任务。
- 网页工人线程是新式浏览器才支持的特性，它允许你在 `UI` 线程之外运行 `JavaScript` 代码而避免锁定 `UI`。
- 网页应用程序越复杂，积极主动地管理 `UI` 线程就越显得重要。没有什么 `JavaScript` 代码可以重要到允许影响用户体验的程度。