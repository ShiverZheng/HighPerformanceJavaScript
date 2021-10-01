## DOM编程 DOM Scripting 
浏览器中通常会把```DOM```和```JavaScript```独立实现。两个相互独立的功能只要通过接口彼此连接，就会产生消耗。访问DOM的次数越多，消耗就越高。

### DOM访问与修改 DOM Access and Modification

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


### 节点克隆 Cloning Nodes
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

### 遍历DOM Walking the DOM

#### 在DOM中爬行
`childNodes`是个元素集合，因此在循环中注意缓存`length`属性以避免在每次迭代中更新。

#### 元素节点
使用children替代childNodes会更快，因为集合项更少。

#### 选择器API
```js
// elements的值包含了一个引用列表，指向id = ”menu“ 的元素之中的所有a元素。
var elements = document.querySelectorAll('#menu a')
```
