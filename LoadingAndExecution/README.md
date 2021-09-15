### 脚本位置 Script Positioning
- 由于脚本会阻塞页面的其他资源的下载，因此推荐将所有的```<script>```标签尽可能的放到```<body>```标签的底部，以尽量减少对整个页面下载的影响。

### 组织脚本 Grouping Script
- 减少外链数，将多个```<script>```合成一个，这样只需要引入一个```<script>```就可以减少性能消耗。

### 无阻塞的脚本 Nonblocking Scripts
- 延迟的脚本 Deferred Scripts
  - Defer 加载后续文档元素的过程将和js的加载 *并行* 进行（异步），但js的执行会在所有元素解析完成之后，DOMContentLoaded 事件触发之前完成。
  - Async 会在 HTML 文档解析时 *并行* 下载文件，并在下载完成后立即执行（暂停 HTML 解析）。

- 动态脚本元素 Dynamic Script Elements
  - 通常来说将```<script>```添加到```<head>```标签里比添加到```<body>```里更加保险。 
  - 动态创建```<script>```并插入到页面中，文件在该元素被插入时开始下载，无论何时启动，文件的下载和执行不会阻塞页面的其他进程。
  - 除IE外浏览器会在```<script>```元素接收完成时出发一个load事件。
  ```javascript
    var script = document.createElement('script')
    script.type = 'text/javascript'
    script.onload = function() {
        alert('script loaded!')
    }
    script.src = 'file.js'
    document.getElementByTagName('head')[0].appendChild(script)
  ```
  - 动态脚本加载凭借它跨浏览器兼容性和易用的优势，成为最通用的无阻塞加载解决方案。

- 脚本注入 XMLHttpRequest
    - 另一种无阻塞加载脚本的方式是使用```XMLHttpRequest(XHR)```对象获取脚本并注入页面中。
    ```javascript
        var xhr = new XHMLHttpRequest()
        xhr.open('get', 'file.js', true)
        xhr.onreadystatechange = function() {
            if (xhr.readyState == 4) {
                if (xhr.status >= 200 && xhr.status < 300 || xhr.status == 304) {
                    var script = document.createElement('script')
                    script.type = 'text/javascript'
                    script.text = xhr.responseText
                    document.body.appendChild(script)
                }
            }
        }
        xhr.send(null)
    ```
    - 如果上述代码收到了有效的响应，就会创建一个```<scirpt>```元素，设置该元素的```text```属性为接收到的```responseText```，这样实际相当于会创建一个带有内联脚本的```<script>```标签，一旦被添加到页面，代码就会立刻执行然后准备就绪。
    - 局限性为文件必须和页面处在相同的域，这意味着```JavaScript```文件不能从CDN下载，因此通常不会采用这种方式。
- 推荐的无阻塞模式 Recommended Nonblocking Pattern
  - 先添加动态加载所需的代码 （尽量精简，甚至只包含```loadScript()```）。
  - 加载初始化页面所需的剩下的代码 
    ```javascript
        function loadScript(url, callback) {
            var script = document.createElement('script')
            script.type = 'text/javascript'

            // IE
            if (script.readyState) {
                script.onreadystatechange = function() {
                    if (script.readyState == 'loaded' || script.readyState == 'complete') {
                        script.onreadystatechange = null
                        callback()
                    }
                }
            } else {
                script.onload = function() {
                    callback()
                }
            }

            script.src = url
            document.getElementByTagName('head')[0].appendChild(script)
        }

        loadScript('the-rest.js', function(){
            Application.init()
        })
    ```
    ```javascript
    <script type="text/javascript" src="loader.js" />
    <script>
        loadScript('the-rest.js', function() {
            Appliction.init()
        })
    </script>
    ```

  - 把这段代码放在```</body>```闭合之前，有如下好处：
    - 确保了```JavaScript```执行过程中不会阻碍页面其他内容的显示
    - 第二个```JavaScript```文件完成下载时，应用所需的DOM结构已经创建完毕，并做好了交互的准备，从而避免了需要另一个事件来检测页面是否准备好。

### 小结 Summary
- ```<body>```闭合标签之前，将所有的```<script>```标签放在页面底部，确保在脚本执行前页面已经完成渲染。
- 合并脚本，页面中的```<scirpt>```标签越少，加载也越快，响应更迅速，无论外链文件还是内嵌脚本都是如此。
- 有多种无阻塞下载```JavaScript```的方法：
  - 使用```<script>```标签的```defer```属性
  - 使用动态创建的```<script>```元素下载并执行代码
  - 使用```XHR```对象下载```JavaScript```代码并注入页面中