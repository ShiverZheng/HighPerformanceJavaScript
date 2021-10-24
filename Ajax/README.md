## 数据传输 Data Transmission

Ajax，从最基本的层面来说，是一种与服务器通信而无需重新加载页面的方法；

### 请求数据 Requesting Data

五种常用技术向服务器请求数据：
- XHMLHttpRequest (XHR)
- Dynamic script insertion 动态脚本注入
- Multipart XHR
- iframes `极端情况下使用，不作讨论`
- Comet `极端情况下使用，不作讨论`

#### XHMLHttpRequest (XHR) 

```js
var url = '/data.php'
var params = [
	'id=123',
	'limit=20'
]

var req = new XHMLHttpRequest()

req.onreadystatechange = function() {
	// 所有信息接收完毕
	if (req.readyState === 4) {
		// 获取相应头信息
		var responseHeaders = req.getAllResponseHeaders()
		// 获取数据
		var data = req.responseText
		// 数据处理 ...
	}

	// 接收到部分信息，但不是所有
	else if (req.readyState === 3) {
		var dataSoFar = req.responseText
	}
}

req.open('GET', url + '?' + params.join('&'), true)
// 设置请求头信息
req.setRequestHeader('X-Requested-With', 'XMLHttpRequest')
/// 发送一个请求
req.send(null)
```

通过监听 `readyState` 值等于 3，说明此时正在与服务器交互，响应信息还在传输过程中，可以与正在传输中的服务器响应进行交互。这就是所谓的”流”`streaming`，它是提升数据请求性能的强大工具。

#### XHR的 POST GET 对比

GET: 对于那些不会改变服务器状态，只会获取数据（这被称为“幂等行为”）的请求，应该使用 GET。经 GET 请求的数据会被缓存起来，如果需要多次请求同一数据的话，它会有助于提升性能。

POST: 只有当请求的 URL 加上参数的长度接近或超过**2048**个字符时，才应该用POST获取数据。这是因为 IE 限制 URL 长度，过长时将会导致请求的URL被截断。

#### 动态脚本注入

```js
var scriptElement = document.createElement('script')
scriptElement.src = 'http://any-domain.com/javascript/lib.js'
document.getElementsByTagName('head')[0].appendChild(scriptElement)

function jsonCallback(jsonString) {
	var data = eval('(' + jsonString +  ')')
	// 处理数据 ...
}

// lib.js文件需要把数据封装在jsonCallback函数里：
jsonCallback({
	"status": 1,
	"colors": ["#fff", "#000", "#ff0000"]
})

```

优点：克服了XHR的最大限制：跨域请求数据。
缺点：动态脚本注入提供的控制是有限的。
- 不能设置请求的头信息。
- 参数传递也只能使用 GET 方式，而不是 POST方式。
- 不能设置请求的超时处理或重试。
  > 事实上，就算失败了也不一定知道。你必须等待所有数据都已返回，才可以访问它们。你不能访问请求的头信息，也不能把整个响应消息作为字符串来处理。
- 不能使用纯XML、纯JSON或其他任何格式的数据，无论哪种格式，都必须封装在一个回调函数中
  > 响应消息作为脚本标签的源码，它必须是可执行的 JavaScript 代码。
  
**使用这种技术从那些无法直接控制的服务器上请求数据时需要小心。JavaScript 没有任何权限和访问控制的概念，因此使用动态脚本注入添加到页面中的任何代码都可以完全控制整个页面。包括修改任何内容，把用户重定向到其他网站，甚至跟踪用户在页面上的操作并发送数据到第三方服务器。引入外部来源的代码时请务必多加小心。**

#### Multipart XHR (MXHR)

允许客户端只用一个HTTP请求就可以从服务端传送多个资源，它通过在服务端将资源（CSS 文件、HTML 片段、JavaScript 代码或 base64 编码的图片）打包成一个由双方约定的字符串分割的长字符串并发送到客户端。然后用 JavaScript 代码处理这个长字符串，并根据它的 mime-type 类型和传入的其他“头信息”解析出每个资源。

```php
// 读取了三张图片，转换为base64编码的长字符串，之间用一个简单的Unicode编码的字符1连接，然后返回给客户端。
$images = array('kitten.jpg', 'sunset.jpg', 'baby.jpg')
foreach ($images as $image) {
	$image_fh = fopen($image, 'r')
	$image_data = fread($image_fh, filesize($image))
	fclose($image_fh)
		$payloads[] = base64_encode($image_data)
}

// 把字符串合并成一个长字符串，然后输出它
$newline = chr(1) // 该字符串不会出现在任何base64字符串中

echo implode($newline, $payloads)

```
客户端通过`splitImages`函数处理：
```js
function splitImages(imageString) {
	var imageData = iamgeString.split('\u0001')
	var imageElement

	for (var i = 0, len = imageData.length; i < len; i++) {
		imageElement = document.createElement('img')
		imageElement.src = 'data:image/jpeg;base64,' + imageData[i]
		document.getElmentById('container').appendChild(imageElement)
	}
}

var req = new XMLHttpRequest()

req.open('GET', 'rollup_images.php', true)
req.onreadystatechange = function() {
	if (req.readyState == 4) {
		splitImage(req.reponseText)
	}
}
req.send(null)

```

此函数将连接后的字符串分解成三段。每一段用来创建一个图片元素，然后将图片元素插入页面中。图片不是由base64字符串转换成二进制，而是使用`data: URL`的方式创建，并指定`mime-type`为`images/jpeg`。

最终的结果是：在一次 HTTP 请求中向浏览器传送了三张图片。你也可以传送 20 张或 100 张图片，这样的话响应消息会更大，但仍然只用了一次 HTTP 请求。当然，这种方式也可以扩展到其他资源的类型。JavaScript 文件、CSS 文件、HTML 片段以及多种类型的图片都能合并成一次响应。任何数据类型都可以被 JavaScript 作为字符串发送。

下面的函数用于将 JavaScript 代码、CSS 样式和图片转换为浏览器可用的资源：

```js
function handleImageData(data, mimeType) {
	var img = document.createElemet('img')
	img.src = 'data:' + mineType + ';base64,' + data
	return img
}
function handleCss(data) {
	var style = document.createElement('style')
	style.type = 'text/css'

	var ndoe = docuemnt.createTextNode(data)
	style.appendChild(node)
	document.getElementsByTagName('head')[0].appendChild(style)
}

function handleJavaScript(data) {
	eval(data)
}
```

由于 MXHR 响应消息的体积越来越大，因此有必要在每个资源收到时就立刻处理，而不是等到整个响应消息接收完成。这可以通过监听 readyState 为 3 的状态来实现：

```js
var req = new XMLHttpRequest()
var getLatestPacketInterval, lastLength = 0

req.open('GET', 'rollup_images.php', true)
req.onreadystatechange = readyStateHandler
req.send(null)

function readStateHandler() {
	if (req.readyState === 3 && getLatestPacketInterval === null) {
		// 开始轮询
		getLatestPacketInterval = window.setInterval(function() {
			getLatestPacket()
		})
	}

	if (req.readyState === 4) {
		// 停止轮询
		clearInterval(getLatestPacketInterval)

		// 获取最后一个数据包
		getLatestPacket()
	}
}

function getLatestPacket() {
	var length = req.responseText.length
	var packet = req.responseText.substring(lastLength, length)

	processPacket(packet)
	lastLength = length
}
```

当 `readyState` 为 3 的状态 第一次触发时，启动一个定时器，每隔 15 毫秒检查一次响应中的新数据。数据片段会被收集起来，直到发现一个分隔符，然后就把遇到分隔符之前收集的所有数据作为一个完整的资源进行处理。

> 编写健壮的 MXHR 代码很复杂，但值得深入研究。访问以下网址可获取完整的类库：http://techfoolery.com/mxhr/。

缺点
- 资源不能被浏览器缓存
> 如果你使用 MXHR 获取一个特定的 CSS 文件，然后在下一个页面中正常加载它，它将不会存在于缓存里。这是因为合并后的资源是作为字符串传输的，然后被 JavaScript 代码分解成片段。由于无法用编程的方式向浏览器缓存里注入文件，因此用这种方式获取的资源无法被缓存。
- 老版本的 IE 不支持 readyState 为 3 的状态 和 data: URL。
> IE 8 倒是两个都支持，但在 IE 6 和 7 中必须设法变通。


某些情况下 MXHR 依然能显著提升页面的整体性能：

- 页面包含了大量其他地方用不到的资源（因此也无需缓存），尤其是图片。
- 网站已经在每个页面中使用一个独立打包的 JavaScript 或 CSS 文件以减少 HTTP 请求。
> 因为对每个页面来说这些文件都是唯一的，所以并不需要从缓存中读取数据，除非重载页面。

由于 HTTP 请求是 Ajax 中最大的瓶颈之一，因此减少其需要的数量会对整个页面的性能有很大的影响。尤其是当你将100个图片请求转化为一个MXHR请求时。专门有一个这样的测试，它在各种主流浏览器上传送大量图片，结果显示此技术比各自独立请求的方式快 4 到 10 倍。你也可以自己运行该测试：http://techfoolery.com/mxhr/