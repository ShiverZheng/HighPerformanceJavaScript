# DOM编程 DOM Scripting 
浏览器中通常会把```DOM```和```JavaScript```独立实现。两个相互独立的功能只要通过接口彼此连接，就会产生消耗。访问DOM的次数越多，消耗就越高。

## DOM访问与修改 DOM Access and Modification

把运算留给```ECMAScript```这一端处理的一个好处是：```innerHTMLLoop2```比```innerHTMLLoop```快上百倍。

```js
    // Bad
    function innerHTMLLoop() {
        for (var count = 0; count < 1500; count++) {
            document.getElementById('here').innerHTML += 'a'
        }
    }

    // Pretty
    function innerHTMLLoop2() {
        var content = ''
        for (var count = 0; count < 1500; count++) {
            content += 'a'
        }
        document.getElementById('here').innerHTML += content
    }
```


## 节点克隆 Cloning Nodes
使用```element.cloneNode()```(```element```表示已有节点)替代```document.createElement()```，在大多数浏览器中，节点克隆都更有效率，但不是特别明显。

```js
    // 将HTML集合拷贝到普通数组
    function toArray(coll) {
        for (var i = 0, a = [], len = coll.length; i < len; i++) {
            a[i] = coll[i]    
        }
        return a
    }
```

```js
    // 在相同的内容和数量下，遍历一个数组的速度明显快于遍历一个HTML合集
    var coll = document.getElementsByTagName('div')
    var arr = toArray(coll)

    // slower
    function loopCollection() {
        for (var count = 0; count < coll.length; count++) {
            /* do nothing */
        }
    }

    // faster
    function loopCollectionArray() {
        for (var count = 0; count < arr.length; count++) {
            /* do nothing */
        }
    }
```
在每次迭代过程中，读取元素集合的 `length` 属性会引发集合进行更新，这在所有浏览器中都有明显的性能问题。优化方法很简单，把集合的长度缓存到一个局部变量中，然后在循环的条件退出语句中使用该变量：

```js
// 此函数运行速度和loopCopiedArray（）一样快。

function loopCacheLengthCollection() {
    var coll = document.getElementsByTagName('div')
    len = coll.length
    for (var count = 0; count < len; count++) {
        /* do nothing */
    }
}
```

```js
// 在循环使用局部变量存储集合引用和集合元素带来速度提升
// slow
function collectionGlobal0) {
    var coll = document.getElementsByTagName('div')
    var len = coll.length
    var name = ''
    for (var count = 0; count < len; count++) {
        name = document.getElementsByTagName('div')[count].nodeName
        name = document.getElementsByTagName('div')[count].nodeType
        name = document.getElementsByTagName('div')[count].tagName
    }
    return name
}
// faster
function collectionLocal() {
    var coll = document.getElementsByTagName('div')
    var len = coll.length
    var name = ''
    for (var count = 0; count < len; count++) {
        name = coll[count].nodeName
        name = coll[count].nodeType
        name = coll[count].tagName
    }
    return name
}

// fastest
function collectionNodesLocal() {
    var coll = document.getElementsByTagName('div')
    var len = coll.length
    var name = ''
    var el = null
    for (var count = 0; count < len; count++) {
        el = coll[count];
        name = el.nodeName;
        name = el.nodeType;
        name = el.tagName;
    }
    return name;
}
```

## 遍历DOM Walking the DOM

### 在DOM中爬行
`childNodes`是个元素集合，因此在循环中注意缓存`length`属性以避免在每次迭代中更新。

### 元素节点
使用children替代childNodes会更快，因为集合项更少。

### 选择器API
```js
// NodeList
document.querySelectorAll('body')

// HTMLCollection
document.getElementsByTagName('body')
```
- `NodeList`是一个**静态的节点集合**（DOM树、节点数量、类型的快照），其不受DOM树元素变化的影响。对节点增删，不影响`NodeList`。但是对节点内部内容修改（如`innerHTML`）是有影响的。
- `HTMLCollection`是一个的**动态的元素集合**，DOM树发生变化，`HTMLCollection`也会随之变化，节点的增删是敏感的。
只有`NodeList`对象有包含属性节点和文本节点。
- `HTMLCollection`元素可以通过`name`，`id`或`index`索引来获取。`NodeList`只能通过`index`索引来获取。
- `HTMLCollection`和`NodeList`本身无法使用数组的方法，除非转为一个数组。

## 重绘与重排 Repaints and Reflows

浏览器下载完页面中的所有组件——HTML 标记、JavaScript、CSS、图片——之后会解析并生成两个内部数据结构：
- DOM树 —— 表示页面结构
- 渲染树 —— 表示DOM节点如何显示

DOM树中的每一个需要显示的节点在渲染树中至少存在一个对应的节点（隐藏的DOM元素在渲染树中没有对应的节点）。
> 渲染树中的节点被称为“帧`frames`”或“盒`boxes`”，符合CSS模型的定义，理解页面元素为一个具有填充`padding`，边距`margins`，边框`borders`和位置`position`的盒子。
一旦DOM和渲染树构建完成，浏览器就开始显示（绘制`paint`）页面元素。

当 DOM 的变化影响了元素的几何属性（宽和高）——比如改变边框宽度或给段落增加文字，导致行数增加——浏览器需要重新计算元素的几何属性，同样其他元素的几何属性和位置也会因此受到影响。浏览器会使渲染树中受到影响的部分失效，并重新构造渲染树。这个过程称为**重排`reflow`**。

完成重排后，浏览器会重新绘制受影响的部分到屏幕中，该过程称为**重绘`repaint`**。

并不是所有的 DOM 变化都会影响几何属性。例如，改变一个元素的背景色并不会影响它的宽和高。在这种情况下，只会发生一次重绘（不需要重排），因为元素的布局并没有改变。
> 重绘和重排操作都是代价昂贵的操作，它们会导致Web应用程序的UI反应迟钝。所以，应当尽可能减少这类过程的发生。

### 重排何时发生？ When Does a Reflow Happen ?

当页面布局和几何属性改变时就需要“重排”。下述情况中会发生重排：

- 添加或删除可见的DOM元素
- 元素位置改变
- 元素尺寸改变（包括：外边距、内边距、边框厚度、宽度、高度等属性改变）
- 内容改变，例如：文本改变或图片被另一个不同尺寸的图片替代
- 页面渲染器初始化
- 浏览器窗口尺寸改变
  
根据改变的范围和程度，渲染树中或大或小的对应的部分也需要重新计算。有些改变会触发整个页面的重排：例如，当滚动条出现时。

### 渲染树变化的排队与刷新  Queuing and Flushing Render Tree Changes

获取布局信息的操作会导致列队刷新，比如以下方法：

- offsetTop,offsetLeft,offsetWidth,offsetHeight
- scrollTop,scrollLeft,scrollWidth,scrollHeight
- clientTop,clientLeft,clientWidth,clientHeight
- getComputedStyle（）（currentStyle in IE）

> 以上属性和方法需要返回最新的布局信息，因此浏览器不得不执行渲染列队中的“待处理变化”并触发重排以返回正确的值。

### 最小化重绘和重排 Minimizing Repaints and Reflows

重绘和重排可能代价非常昂贵，因此一个好的提高程序响应速度的策略就是减少此类操作的发生。为了减少发生次数，应该合并多次对DOM和样式的修改，然后一次处理掉。

```js
    // bad
    var el = document.getElementById('mydiv')
    el.style.borderLeft = '1px'
    el.style.boderRight = '2px'
    el.style.padding = '5px'

    // good
    var el = document.getElementById('mydiv')
    el.style.cssText = 'boder-left: 1px; boder-right: 2px;padding: 5px'
    
    // also good
    var el = document.getElementById('mydiv')
    el.className = 'active'
```

#### 批量修改DOM

你需要对DOM元素进行一系列操作时，可以通过以下步骤来减少重绘和重排的次数：

- 使元素脱离文档流
- 对其应用多重改变
- 把元素带回文档中

该过程里会触发两次重排——第一步和第三步。如果你忽略这两个步骤，那么在第二步所产生的任何修改都会触发一次重排。

有三种基本方法可以使DOM脱离文档：

- 隐藏元素，修改应用，重新显示。
- 使用文档片断（docuement fragment）在当前DOM之外构建一个子树，再把它拷贝回文档。
- 将原始元素拷贝到一个脱离文档的节点中，修改副本，完成后再替换原始元素。为了演示脱离文档的操作，考虑下面的链接列表，它必须更新更多信息：

```html
<ul id="mylist">
    <li><a href="http://phpied.com">Stoyan</a></li>
    <li><a href="http://julienlecomte.com">Julien</a></li>
</ul>
```
假设附加数据已经存储在一个对象中，并要插入列表。这些数据定义如下：
```js
var data = [
    {
        "name": "Nicholas",
        "url": "http://nczonline.net"
    },
    {
        "name": "Ross",
        "url": "http://techfoolery.com"
    }
]
```
下面是一个用来更新指定节点数据的通用函数：
```js
function appendDataToElement(appendToElement, data) {
    var a, li
    for (var i = 0, max = data.length; i < max; i++) {
        a = document.createElement('a')
        a.href = data[i].url
        a.appendChild(document.createTextNode(data[i].name))
        li = document.createElement('li')
        li.appendChild(a)
        appendToElement.appendChild(li)
    }
}
```

```js
    // bad
    // data数组的每一个条目被附加到当前DOM时都会发生重排
    var ul = docuement.getElementById('myList')
    appendDataToElment(ul, data)

    // good
    // 一个减少重排的方法是通过改变display属性，临时从文档中移除<ul>元素，然后再恢复它
    var ul = docuement.getElementById('myList')
    ul.style.display = 'none'
    appendDataToElment(ul, data)
    ul.style.dispaly = 'block'

    // also good
    // 需要修改的节点创建一个备份，然后对副本进行操作，一旦操作完成，就用新的节点替代旧的节点
    var old = document.getElementById('myList') 
    var clone = old.cloneNode(true)
    appendDataToElement(clone, data)
    old.parentNode.replaceChild(clone, old)

    // best
    // 文档片断是个轻量级的 document 对象，它的设计初衷就是为了完成这类任务——更新和移动节点
    var fragment = document.createDocumentFragment()
    appendDataToElement(fragment, data)
    document.getElmentById('myList').appendChild(fragment)
```


### 缓存布局信息 Caching Layout Information

浏览器尝试通过队列化修改和批量执行的方式最小化重排次数。当你查询布局信息时，比如获取偏移量（offsets）、滚动位置（scroll values）或计算出的样式值（computedsytle values）时，浏览器为了返回最新值，会刷新队列并应用所有变更。

最好的做法是尽量减少布局信息的获取次数，获取后把它赋值给局部变量，然后再操作局部变量。

考虑一个例子，把myElement元素沿对角线移动，每次移动一个像素，从`100px * 100px`的位置开始，到`500px * 500px`的位置结束。在`timeout`循环体中你可以使用下面的方法：

```js
// bad 
myElement.style.left = 1 + myElement.offsetleft + 'px'
myElement.style.top = 1 + myElement.offsetTop + 'px'
if (myElement.offsetLeft >= 500) {
    stopAnimation()
}

// best
// 获取一次起始位置的值，然后将其赋值给一个变量，比如 var current=myElement.offsetLeft。然后，在动画循环中，直接使用 current变量而不再查询偏移量：
current++
myElement.style.left = current + 'px'
myElement.style.top = current + 'px'
if (myElement.offsetLeft >= 500) {
    stopAnimation()
}
```

### 让元素脱离动画流 Take Elements Out of the Flow for Animations

用展开/折叠的方式来显示和隐藏部分页面是一种常见的交互模式。它通常包括展开区域的几何动画，并将页面其他部分推向下方。

一般来说，重排只影响渲染树中的一小部分，但也可能影响很大的部分，甚至整个渲染树。浏览器所需要重排的次数越少，应用程序的响应速度就越快。因此当页面顶部的一个动画推移页面整个余下的部分时，会导致一次代价昂贵的大规模重排，让用户感到页面一顿一顿的。渲染树中需要重新计算的节点越多，情况就会越糟。

使用以下步骤可以避免页面中的大部分重排：
- 使用绝对位置定位页面上的动画元素，将其脱离文档流。
- 让元素动起来。当它扩大时，会临时覆盖部分页面。但这只是页面一个小区域的重绘过程，不会产生重排并重绘页面的大部分内容。
- 当动画结束时恢复定位，从而只会下移一次文档的其他元素。

## 事件委托

每绑定一个事件处理器都是有代价的，它要么是加重了页面负担（更多的标签或 JavaScript 代码），要么是增加了运行期的执行时间。需要访问和修改的 DOM 元素越多，应用程序也就越慢，特别是事件绑定通常发生在`onload`或`DOMContentReady`时，此时对每一个富交互应用的网页来说都是一个拥堵的时刻。

事件绑定占用了处理时间，而且，浏览器需要跟踪每个事件处理器，这也会占用更多的内存。当这些工作结束时，这些事件处理器中的绝大部分都不再需要（因为并不是100%的按钮或链接会被用户点击），因此有很多工作是没必要的。

一个简单而优雅的处理 DOM 事件的技术是事件委托。它是基于这样一个事实：事件逐层冒泡并能被父级元素捕获。使用事件代理，只需给外层元素绑定一个处理器，就可以处理在其子元素上触发的所有事件。

根据DOM标准，每个事件都要经历三个阶段：
- 捕获
- 到达目标
- 冒泡

事件委托兼容性代码：
- 访问事件对象，并判断事件源
- 取消文档树中的冒泡（可选）
- 阻止默认动作（可选）


## 小结 Summary
访问和操作DOM是现代web应用的重要部分。但每次穿越连接ECMAScript和DOM两个岛屿之间的桥梁，都会被收取“过桥费”。为了减少DOM编程带来的性能损失，请记住以下几点：
- 最小化DOM访问次数，尽可能在JavaScript端处理
- 如果需要多次访问某个DOM节点，请使用局部变量存储它的引用
- 小心处理HTML集合，因为它实时连系着底层文档。把集合的长度缓存到一个变量中，并在迭代中使用它。如果需要经常操作集合，建议把它拷贝到一个数组中。
- 如果可能的话，使用速度更快的API，比如`querySelectorAll()`和`firstElementChild`。
- 要留意重绘和重排；批量修改样式时，“离线”操作 DOM 树，使用缓存，并减少访问布局信息的次数。
- 动画中使用绝对定位，使用拖放代理。
- 使用事件委托来减少事件处理器的数量。