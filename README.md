# HTML5-WebSocket
## 1、概述
HTTP协议是一种无状态协议，服务端本身不具有识别客户端的能力，必须借助外部机制，比如session和cookie，才能与特定客户端保持对话。这多多少少带来一些不便，尤其在服务器端与客户端需要持续交换数据的场合（比如网络聊天），更是如此。为了解决这个问题，HTML5提出了浏览器的WebSocket API。

WebSocket的主要作用是，允许服务器端与客户端进行全双工（full-duplex）的通信。举例来说，HTTP协议有点像发电子邮件，发出后要等待对方回信；WebSocket则是像打电话，服务器端和客户端可以同时向对方发送数据，它们之间存在着一条持续打开的数据通道。

WebSocket协议完全可以取代Ajax方法，用来向服务器端发送文本和二进制数据，而且还没有“同域限制”。

WebSocket不使用HTTP协议，而是使用自己的协议。浏览器发出的WebSocket请求类似于下面的样子：
```
GET / HTTP/1.1
Connection: Upgrade
Upgrade: websocket
Host: example.com
Origin: null
Sec-WebSocket-Key: sN9cRrP/n9NdMgdcy2VJFQ==
Sec-WebSocket-Version: 13
```
上面的头信息显示，有一个HTTP头是Upgrade。HTTP1.1协议规定，Upgrade头信息表示将通信协议从HTTP1.1转向该项所指定的协议。“Connection: Upgrade”就表示浏览器通知服务器，如果可以，就升级到webSocket协议。Origin用于验证浏览器域名是否在服务器许可范围内。Sec-WebSocket-Key则是用于握手协议的密钥，是base64编码的16字节随机字符串。

服务端的WebSocket回应则是： 
```
HTTP/1.1 101 Switching Protocols
Connection: Upgrade
Upgrade: websocket
Sec-WebSocket-Accept: fFBooB7FAkLlXgRSz0BT3v4hq5s=
Sec-WebSocket-Origin: null
Sec-WebSocket-Location: ws://example.com
```
服务端同样用“Connection: Upgrade”通知浏览器，需要改协议。Sec-WebSocket-Accept是服务器在浏览器提供的Sec-WebSocket-Key字符串后面，添加“258EAFA5-E914-47DA-95CA-C5AB0DC85B11”字符串，然的再取sha-1的hash值。浏览器将对这个值进行验证，以证明确实是目标服务器回应了webSocket请求。Sec-WebSocket-Location表示通信的WebSocket网址。

注意：WebSocket协议用ws表示。此外，还有wss协议，表示加密的WebSocket协议，对应HTTPs协议。

完成握手以后，WebSocket协议就在TCP协议之上，开始传送数据。

WebSocket协议需要服务器支持，目前比较流行的实现是基于node.js的socket.io，更多实现可参阅Wikipedia。至于浏览器端，目前主流浏览器都支持WebSocket协议（包括IE10+），仅有的例外是手机端的Opera Mini和Android Browser。

## 2、客户端
浏览器端对WebSocket协议的处理，无非就是三件事：

建立连接和断开连接
发送数据和接收数据
处理错误
### 2.1 建立连接和断开连接

首先，客户端要检查浏览器是否支持WebSocket，使用的方法是查看window对像是否具有WebSocket属性。
```
if (window.WebSocket != undefined) {
    // 支持
}
```
然后，开始与服务器建立连接（这里假定服务器就是本机的1740端口，需要使用ws协议）。
```
if (window.WebSocket != undefined) {
    var connection = new WebSocket('ws://localhost:1740');
}
```
建立连接以后的WebSocket实例对象（即上面代码中的connection），有一个readyState属性，表示目前状态，可以取4个值：

0：正在连接
1：连接成功
2：正在关闭
3：连接关闭
握手协议成功以后，readyState就从0变成1，并触发open事件，这时就可以向服务器发送信息了。我们可以指定open事件的回调函数。
```
connection.onopen = wsOpen;

function wsOpen(event) {
    console.log('Connected');
}
//关闭WebSocket会触发close事件。

connection.onclose = wsClose;

function onClose(event) {
    var wasClean = event.wasClean; //bool值，连接是否关闭了
    var code = event.code; //连接状态（数字），由服务端提供
    var reason = event.reason; //关闭原因，由服务端提供
    console.log('Closed');
}

connection.close();
```
### 2.2 发送数据和接收数据

连接建立后，客户端通过send方法向服务器端发送数据。

connection.send(message);
除了发送字符串，也可以使用Blob或ArrayBuffer对象发送二进制数据。

//使用ArrayBuffer发送Canvas图像数据
```
var img = canvas_context.getImageData(0, 0, 400, 320);
var binary = new Unit8Array(img.data.length);
for(var i = 0; i < img.data.length; i++) {
    binary[i] = img.data[i]
}
connection.send(binary.buffer);

```
//使用Blob发送文件
```
var file = document.querySelector('input[type="file"]').files[0];
connection.send(file);
```
客户端收到服务器发送的数据，会触发message事件。可以通过定义message事件的回调函数，来处理服务端返回的数据。
```
connection.onmessage = wsMessage;

function wsMessage(event) {
    console.log(event.data);
}
```
上面代码的回调函数wsMessage的参数为事件对象evnet，该对象的data属性包含了服务器返回的数据。

### 2.3 处理错误

如果出现错误，浏览器会触发WebSocket实例对象的error事件。
```
connection.onerror = wsError;

function wsError(event) {
    console.log('Error: ' + event.data);
}
```
## 3、服务器端
服务器端需要单独部署处理WebSocket的代码。下面用node.js搭建一个服务器环境。
```
var http = require('http');
var server = http.createServer(function(request, response){});
//假设监听1740端口。

server.listen(1740, function() {
    console.log((new Date()) + ' Server is listening on port 1740');
});
接着启动WebSocket服务器。这需要加载websocket库，如果没有安装，可以先使用命令安装。

var WebSocketServer = require('websocket').server;
var wsServer = new WebSocketServer({
    httpServer: server
});
WebSocket服务器建立request事件的回调函数。

var connection;

wsServer.on('request', function(req){
    connect = req.accept('echo-protocol', req.origin);
});
```
上面代码的回调函数接受一个参数req，表示request请求对象。然后，在回调函数内部，建立WebSocket连接Connection。接着，就要对connection的message事件指定回调函数。
```
wsServer.on('request', function(r) {
    connection = req.accept('echo-protocol', req.origin);

    connection.on('message', function(message) {
        var msgString = message.utf8Data;
        connection.sendUTF(msgString);
    });
});
最后，监听用户的disconnect事件。

connection.on('close', function(reasonCode, description) {
    console.log(connection.remoteAddress + ' disconnected');
});
```
## 4、Socket.io简介
Socket.io是目前最流行的WebSocket实现，包括服务器和浏览器两个部分。它不仅简化了接口，使得操作更容易，而且对于那些不支持WebSocket的浏览器，会自动降为Ajax连接，最大限度地保证了兼容性。

* 第一步，在服务器端的项目根目录下，安装socket.io模块。

npm install socket.io
 

* 第二步，在根目录下建立app.js，并写入以下代码（使定使用了Express框架）。
```
var app = require('express')();
var server = require('http').createServer(app);
var io = require('socket.io').listen(server);

server.listen(80);

app.get('/', function (req, res) {
  res.sendfile(__dirname + '/index.html');
});
```
上面代码表示，先建立并运行HTTP服务器。Socket.io的运行建立在HTTP服务器之上。

 

* 第三步，将Socket.io插入客户端网页。

<script src="/socket.io/socket.io.js"></script>
然后，在客户端脚本中，建立WebSocket连接。

var socket = io.connect('http://localhost');
由于本例假定WebSocket主机与客户端是同一台机器，所以connect方法的参数是 http://localhost 接着,指定news事件（即服务器端发送news）的回调函数。

socket.on('news', function (data){
   console.log(data);
});
最后，用emit方法向服务器端发送信号，触发服务器端的anotherNews事件。

请注意：emit方法可以取代Ajax请求，而on方法指定的回调函数，也等同于Ajax的回调函数。

 

* 第四步，在服务器端的app.js中加入以下代码。
```
io.sockets.on('connection', function (socket) {
  socket.emit('news', { hello: 'world' });
  socket.on('anotherNews', function (data) {
    console.log(data);
  });
});
```
上面代码的io.sockets.on方法指定connect事件（WebSocket连接建立）的回调函数。在回调函数中，用emit方法向客户端发送数据，触发客户端的news事件。然后，再用on方法指定服务器端anotherNews事件的回调函数。
