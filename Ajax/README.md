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


### 发送数据 Sending Data

当数据只需要发送到服务器时，有两种广泛使用的技术：XHR 和信标（beacons）。

#### XMLHttpRequest

当要传回的数据超出浏览器的最大 URL 尺寸限制时 XHR 特别有用。这种情况下，可以用 POST 方式回传数据：
```js
var url = '/data.php'
var params = [
	'id=123',
	'limit=20'
]

var req = new XMLHttpRequest()
req.onerror = function() {
	// error	
}

req.onreadstatechange = function() {
	if (req.readyState == 4) {
		// success
	}
}

req.open('POST', url, true)
req.setRequestHeader('Content-Type', 'application/x-www-form-urlencoded')
req.send(params.json('&'))


// 失败重试

function xhrPost(url, params, callback) {
	var req = new XHMHttpRequest()
	req.onerror = function() {
		setTimeout(function() {
			xhrPost(url, params, callback)
		}, 1000)
	}

	req.onreadystatechange = function() {
		if (req.readyState == 4) {
			if (callback && typeof callback === 'function') {
				callback()
			}
		}
	}
	req.open('POST', url, true)
	req.setRequestHeader('Content-Type', 'application/x-www-form-urlencoded')
	req.setRequestHeader('Content-length', params.length)
	req.send(params.json('&'))
}
```

> 当使用 XHR 发送数据到服务器时，GET 方式会更快。对于少量数据而言，一个 GET 请求往服务器只发送一个数据包。而一个 POST 请求，至少要发送两个数据包，一个装载头信息，另一个装载 POST 正文。POST 更适合发送大量数据到服务器，因为它不关心额外数据包的数量，另一个原因是 IE 对 URL 长度有限制，它不可能使用过长的GET 请求。

#### Beacons

这项技术非常类似动态脚本注入。使用JavaScript 创建一个新的 Image 对象，并把 src 属性设置为服务器上脚本的 URL。该 URL 包含了我们要通过 GET 传回的键值对数据。请注意并没有创建 img 元素或把它插入 DOM。

```js
var url = '/status_tracker.php'
var params = [
	'step=2',
	'time=123'
]

(new Image()).src = url + '?' + params.join('&')

```

服务器会接收到数据并保存下来，它无需向客户端发送任何回馈信息，因此没有图片会实际显示出来。这是给服务器回传信息最有效的方式。它的性能消耗很小，而且服务端的错误完全不会影响到客户端。

##### 局限

- 无法发送 POST 数据，而 URL 的长度有最大值，所以可以发送的数据的长度被限制得相当小。

- 可以接收服务器返回的数据，但只局限于非常少的几种方式。
  > 一种方式是监听 Image 对象的 load 事件，它会告诉你服务器是否成功接收数据。你还可以检查服务器返回的图片的宽度和高度（如果返回的是图片的话）并使用这些数字通知你服务器的状态。举个例子，宽度为 1 表示“成功”，为 2 表示“重试”。
  

如果你不需要在响应中返回数据，就应该发送一个不带消息正文的 204 No Content 状态码，它将阻止客户端继续等待永远不会到来的消息正文:
```js
var url = '/status_tracker.php'
var params = [
	'step=2',
	'time=1234'
]

var beacon = new Image()
beacon.src = url + '?' + params.join('&')

beacon.onload = function() {
	if (this.width === 1) {
		// success
	} else if (this.width === 2) {
		// faild, retry another beacon
	}
}


beacon.onerror = function() {
	// faild, retry another beacon
}
```

> 信标是向服务器回传数据最快且最有效的方式。服务器根本不需要发送任何响应正文，因此你也无须担心客户端下载数据。唯一的缺点是你能接收到的响应类型是有限的。如果你需要返回大量数据给客户端，那么请使用 XHR。如果你只关心发送数据到服务器（可能需要极少的返回信息），那么请使用图片信标。


## 数据格式 Data Formats

### XML

当 Ajax 最先开始流行时，它选择了 XML 作为数据格式。当时它有很多的优势：极佳的通用性（服务端和客户端都能完美支持）、格式严格，且易于验证。

相比其他格式，XML 极其冗长。每个单独的数据片段都依赖大量结构，所以有效数据的比例非常低。而且 XML 的语法有些模糊。

### JSON

JSON是一种使用 JavaScript 对象和数组直接量编写的轻量级且易于解析的数据格式，它由Douglas Crockford创立并推广开来。
```json
[
	{
		"id": 1,
		"username": "alice",
		"realname": "Alice Smith",
		"email": "alice@e.com"
	}
]
```

用户表示为一个对象，用户列表表示为一个数组，就像 JavaScript 中其他数组和对象的写法一样。这意味着当它被求值或封装在一个回调函数中时，JSON 数据就是一段可执行的 JavaScript 代码。在 JavaScript 中可以简单地使用`eval()`来解析 JSON字符串：
```js
function parseJSON(responseText) {
	return eval('(' + responseText + ')')
}
```
> 警告：关于 JSON 和 `eval` 需要注意的是：在代码中使用 `eval` 是很危险的，特别是用它执行第三方的 JSON 数据（其中可能包含恶意代码）时。尽可能使用 `JSON.parse()`方法解析字符串本身。该方法可以捕获 JSON 中的词法错误，并允许你传入一个函数，用来过滤或转换解析结果。

可以通过简化属性名，或者去掉属性名，直接使用数组的形式，但需要保持数据的顺序。

#### JSON-P

事实上，JSON 可以被本地执行会导致几个重要的性能影响。当使用 XHR 时，JSON数据被当成字符串返回。该字符串紧接着被 `eval()` 转换成原生对象。然而，在使用动态脚本注入时，JSON 数据被当成另一个 JavaScript 文件并作为原生代码执行。为实现这一点，这些数据必须封装在一个回调函数里。这就是`JSON填充（JSON with padding）`或JSON-P。

JSON-P因为回调包装的原因略微增大了文件尺寸，但与其解析性能的提升相比这点增加显得微不足道。由于数据是当作原生的 JavaScript，因此解析速度跟原生 JavaScript 一样快。

文件大小和下载耗时与 XHR 测试基本相同，而解析速度几乎快了 10 倍。标准JSON-P 的解析耗时为 0，因为它已经是原生格式，所以无需解析。简化版 JSON-P 和数组 JSON-P 同样如此，但是每种都必须通过迭代转换为标准 JSON-P 的格式。

最快的JSON格式是使用数组形式的JSON-P。尽管这只比使用XHR的JSON稍微快一点，但是这个差异随着列表的增大而增大。如果你的项目需要处理10 000或100 000个元素的列表，那么JSON-P比JSON会好很多。

有一种情形要避免使用 JSON-P（与性能无关），因为 JSON-P 必须是可执行的JavaScript，它可能被任何人调用并使用动态脚本注入技术插入到任何网站。而另一方面，JSON 在eval前是无效的JavaScript，使用 XHR 时它只是被当做字符串获取。所以不要把任何敏感数据编码在 JSON-P 中，因为你无法确认它是否保持着私有调用状态，即便是带有甚至随机 URL 或 做了cookie判断。

#### 应该使用JSON吗？

与 XML 相比，JSON 有着许多优点。它有着体积更小的格式，在响应信息中结构所占的比例更小，数据占用得更多。特别是当数据包含数组而不是对象时。JSON有着极好的通用性，大多数服务端编程语言都提供了编码和解码的类库。它在客户端的解析工作微不足道，使得你可以花更多的时间来编写代码处理真正有用的数据。对 Web 开发人员来说最重要的是，它是性能表现最好的数据格式，因为它不仅传输尺寸小，而且解析速度快。JSON是高性能 Ajax 的基础，尤其在使用动态脚本注入时。

### HTML

通常请求的数据需要被转换成 HTML 以显示到页面上。JavaScript 可以比较快地把一个较大的数据结构转换为简单的 HTML，但在服务器处理会快得多。一种可考虑的技术是在服务器上构建好整个HTML再传回客户端，JavaScript可以很方便地通过innerHTML属性把它插入页面相应的位置。

HTML 格式可能比实际数据占用更多空间，尽管可以通过尽量少用标签和属性来缓解这一问题。正因为如此，应当在客户端的瓶颈是 CPU 而不是带宽时才使用此技术。

### 7.2.5 数据格式总结 Data Format Conclusions

通常来说数据格式越轻量级越好，JSON和字符分隔的自定义格式是最好的。如果数据集很大并且对解析时间有要求，那么就从如下两种格式中做出选择：
- JSON-P数据，使用动态脚本注入获取。它把数据当作可执行 JavaScript 而不是字符串，解析速度极快。它能跨域使用，但涉及敏感数据时不应该使用它。
- 字符分隔的自定义格式，使用 XHR 或动态脚本注入获取，用`split()`解析。这项技术解析大数据集比JSON-P略快，而且通常文件尺寸更小。

## Ajax性能指南 Ajax Performance Guidelines

一旦选择了最适合的数据传输技术和数据格式，那么你就可以开始考虑其他优化技术了。这些技术要根据具体情况来使用，因此在考虑使用它们之前你要确认它们适用于你的应用程序。

### 缓存数据 Cache Data

最快的Ajax请求就是没有请求。有两种主要的方法可避免发送不必要的请求：

- 在服务端，设置HTTP头信息以确保你的响应会被浏览器缓存。
> 最简单而且好维护
- 在客户端，把获取到的信息存储到本地，从而避免再次请求。
> 最大的控制权

#### 设置HTTP头信息
如果希望 Ajax 响应能够被浏览器缓存，那么必须使用 GET 方式发出请求。还必须在响应中发送正确的 HTTP 头信息。

Expires 头信息会告诉浏览器应该缓存响应多久。它的值是一个日期，过期之后，对该 URL 的任何请求都不再从缓存中获取，而是会重新访问服务器。一个 Expires 头信息格式如下：

`Expires: Mon,28 Jul 2014 23:30:00 GMT`
> 这种特殊的 Expires 头信息告诉浏览器缓存此响应到2014年7月。这种 Expires头信息被称为“遥远的未来”，它对那些从不改变的内容非常有用，比如图片和其他静态数据集。
> 

#### 本地数据存储
除了依赖浏览器处理缓存外，还可以用更手工的方式来实现，即直接把从服务器接收到的数据储存起来。

可以把响应文本保存到一个对象中，以 URL 为键值作为索引。下面的例子对 XHR 作了封装，它首先会检查 URL 是否获取过：

```js
var localCahce = {}
function xhrRequest(url, callback) {
	// 检查此url的本地缓存
	if (lcoalCache[url]) {
		callback.success(localCache[url])
		return
	}

	// 没有找到缓存，则发送请求
	// ... in readState === 4
	localCache[url] = req.responseText
}

```
总的来说，设置一个 Expires 头信息是更好的方案。它实现起来比较容易，而且其缓存内容能跨页面和跨 会话（session）。而手工管理的缓存在你希望编程废止缓存内容并获取更新数据的时候会很有用。设想这样一种情况，你每次请求都会用到缓存数据，但用户执行了某些动作可能导致一个或多个已缓存的响应失效。这种情况下从缓存中删除那些响应十分简单：

`delete localCache[＇/user/friendlist/＇];`

`delete localCache[＇/user/contactlist/＇];`

##  小结 Summary

高性能的 Ajax 包括以下方面：了解你项目的具体需求，选择正确的数据格式和与之匹配的传输技术。

作为数据格式:
- 纯文本和 HTML 只适用于特定场合，但它们可以节省客户端的 CPU周期。
- XML 被广泛应用而且支持良好，但是它十分笨重且解析缓慢。
- JSON 是轻量级的，解析速度快（被视为原生代码而不是字符串），通用性与 XML 相当。
- 字符分隔的自定义格式十分轻量，在解析大量数据集时非常快，但需要编写额外的服务端构造程序，并在客户端解析。


当从页面当前所处的域下请求数据时，XHR 提供了最完善的控制和灵活性，尽管它会把接收到的所有数据当成一个字符串，且这有可能降低解析速度。

另一方面，动态脚本注入允许跨域请求和本地执行 JavaScript 和 JSON 但是它的接口不那么安全，而且还不能读取头信息或响应代码。

Multipart XHR 可以用来减少请求数，并处理一个响应中的各种文件类型，但是它不能缓存接收到的响应。

当需要发送数据时，图片信标是一种简单而有效的方法。

XHR 还可以用 POST 方法发送大量数据。

除了这些格式和传输技术，还有一些准则有助于加速你的 Ajax：

- 减少请求数，可通过合并 JavaScript 和 CSS 文件，或使用 MXHR。
- 缩短页面的加载时间，页面主要内容加载完成后，用 Ajax 获取那些次要的文件。
- 确保你的代码错误不会输出给用户，并在服务端处理错误。
- 知道何时使用成熟的 Ajax 类库，以及何时编写自己的底层 Ajax 代码。
  