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