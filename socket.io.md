# socket.io

## overview
1. Socket.IO是一个库，可以在浏览器和服务器之间实现实时，双向和基于事件的通信，包括Nodejs服务端API和浏览器JS客户端API，具有以下特点：
  * 可靠：
    * 首先建立一个长轮询连接，然后尝试升级到更好的传输，比如WebSocket
    * 即使存在代理、负载均衡器或个人防火墙和防病毒软件也可以建立连接
  * 自动重连：
    * 断开连接的客户端将尝试永久重新连接，直到服务器再次可用
  * 断线检测：
    * 通过在服务器和客户端上设置定时器来实现心跳机制，即允许服务器和客户端知道另一端何时不响应
    * 在连接握手期间共享超时值（pingInterval和pingTimeout参数），计时器需要将任何后续客户端调用定向到同一服务器，因此在使用多个节点时需要粘性会话的支持，比如Nginx的ip_hash
  * 支持二进制：
    * 可以发出任何可序列化的数据结构
    * 浏览器中的`ArrayBuffer`和`Blob`
    * Node.js中的`ArrayBuffer`和`Buffer`
  * 支持多路复用：
    * 允许创建多个命名空间，这些命名空间将充当单独的通信通道，但将共享相同的底层连接
  * 支持房间：
    * 在每个命名空间中，可以定义socket可以加入和离开的任意通道，称为房间
    * 这可以方便的实现群组的功能

2. Socket.IO 不是 WebSocket实现，它会为每个数据包添加一些元数据：数据包类型，命名空间和需要消息确认时的确认ID等。因此，WebSocket客户端无法连接到Socket.IO服务器，Socket.IO客户端也无法连接到WebSocket服务器。
```js
//警告：客户端将无法连接！
const client = io('ws：//echo.websocket.org');
```

3. 安装
```bash
# server
npm install --save socket.io

# client
npm install --save socket.io-client
```

4. 使用
```js
//=====server
const app = require('express')(); 
const server = require('http').Server(app); 
const io = require('socket.io')(server);
server.listen(80); 
app.get('/', (req，res) => { 
  res.sendFile(__dirname + '/ index.html'); 
});
// 默认命名空间 `/`
io.on('connection', (socket) => { 
  // 广播消息，包括发送者
  io.emit('boardcast', {hello: 'everyone receive'});
  // 广播消息，不包括发送者
  socket.broadcast.emit（'user connected'）;
  // 私信
  socket.emit('news'，{ hello: 'world' }); 
  // 带消息确认
  socket.on('message', (data， fn) => {
    console.log(data);   
    fn('i am received: ' + data);
  }); 
  // 发送易失性消息
  const tweets = setTnterval(() => {
    getBieberTweet((tweet) => {
      socket.volatile.emit('bieber tweet', tweet);
    });
  }, 100);
  socket.on('disconnect', () => {
    // 清除定时器
    clearInterval(tweets);
    io.emit('user disconnected');
  });
});
// 自定义命名空间`news`
const news = io.of('/news').on('connection', (socket) =>{
  socket.emit('message', 'only that will get');
  news.emit('message', 'everyone will get');
});

//=====client: 
import io from 'socket.io-client';
const socket = io.connect('http://localhost');
socket.on('news', (data) => {
  console.log(data);
  socket.emit('message', 'hello', (data) => {
    console.log(data); // 'i am received: hello'
  });
});
// 命名空间
const news = io.connect('http://localhost/news');
news.on('connect', (data) => {
  news.emit('message', 'hello');
});
// 只使用websocket语法
news.send('hello');
```

## rooms and namespaces
1. 通过定义命名空间，最大限度地减少资源数量（TCP连接），同时通过在通信通道之间引入分离来分离应用程序中的问题。

* 默认命名空间 `/`
每个命名空间都会发出一个connection事件，将每个Socket实例作为参数接收
```js
io.on('connection', (socket) => {
  socket.on('disconnect', () => {});
});

io.sockets.emit('hi', 'everyone');
io.emit('hi', 'everyone'); // 上面语句的简写
```

* 自定义命名空间
设置自定义命名空间，用`of`在服务器端调用函数
```js
// server
const chat = io.of('/chat');
chat.on('connection', (socket) => {
   socket.on('disconnect', () => {});
});
chat.emit('hi', 'everyone in chat namespace');
// client
const chat = io('/chat');
```

2. 房间
在每个命名空间内，可以定义socket加入join或离开leave某个房间
```js
io.on('connection', (socket) => { 
  socket.join('chat room'); 
  socket.leave('chat room');
});
// 向房间发送消息
io.to('chat room').emit('message', 'hi')
```
默认房间：
每个socket自动加入到一个以其socket#id标识的房间
```js
socket.on('say to someone', (id, msg) => {
  socket.broadcast.to(id).emit('my message', msg);
});
```

3. 从socket.io进程外部向命名空间/房间发送事件
  * socket.io-redis
  * socket.io-emmiter
```js
// redis adapter
var io = require('socket.io')(3000);
var redis = require('socket.io-redis');
io.adapter(redis({ host: 'localhost', port: 6379 }));
// emitter
var io = require('socket.io-emitter')({ host: '127.0.0.1', port: 6379 });
setInterval(function(){
  io.emit('time', new Date);
}, 5000);
```

## migrating from 0.9
1. 使用中间件进行身份验证
```js
const app = require('express')(); 
const server = require('http').Server(app); 
const io = require('socket.io')(server);
// use 方法 代替set方法
io.use(function(socket, next) {
  var handshakeData = socket.request;
  // make sure the handshake data looks good as before
  // if error do this:
  // next(new Error('not authorized'));
  // else just call next
  next();
});
// namespace
io.of('/namespace').use(function(socket, next) {
  var handshakeData = socket.request;
  // authorization
  next();
});
```

2. 日志
日志现在基于Debug
```bash
# server
DEBUG= node index.js
DEBUG=socket.io:* node index.js # 打印socket相关
DEBUG=socket.io:socket node index.js # 打印socket对象

# client
DEBUG=socket.io:socket node index.js # 记录日志到localStorage
localStorage.debug = '' # 清除日志
```

3. 快捷方式
```js
// 消息都会到达连接到默认“/”命名空间的所有客户端
// 而不是其他命名空间中的客户端
// old api
io.sockets.emit（'eventname'，'eventdata'）;
// new api
io.emit（'eventname'，'eventdata'）;

// lift server
// old api
var io = require('socket.io');
var socket = io.listen(80, { /* options */ });
// new api
var io = require('socket.io');
var socket = io({ /* options */ });
```

4. config差异
```js
var io = require('socket.io');
var socket = io({
  // options
  'resource': 'path/to/socket.io',
  'path': '/path/to/socket.io' // 等同resource配置， 注意路径前面多一个 /
});
// 保持向后兼容
io.set('transports')
io.set('heartbeat interval')
io.set('heartbeat timeout')
io.set('resource')
```

5. parser/protocol 差异
此差异仅针对通过其它语言实现socket.io或socket.io client。

## using multiple nodes
* 如果计划在不同进程或计算机之间分配连接负载，则必须确保与特定会话ID关联的请求连接到发起它们的进程。
* 由于某些传输，如XHR轮询或JSONP轮询会在`socket`的生命周期内触发多个请求，未能启用粘性负载平衡将会导致`400`错误。
```bash
Error during WebSocket handshake: Unexpected response code: 400
```
* `WebSocket`方式传输不存在粘性会话的限制，因为服务端和客户端之间的底层TCP链接是长连接
```js
const client = io('http://localhost', {
  // WARNING: in that case, there is no fallback to long-polling
  transports: [ 'websocket' ] 
  // or [ 'websocket', 'polling' ], which is the same thing
})
```
* 开启粘性会话（sticky-session）的2种解决方案：
  * 基于客户端的起始地址路由客户端
  * 基于cookie路由客户端

* Nginx配置 [example](https://github.com/socketio/socket.io/tree/master/examples/cluster-nginx)
```bash
# nginx.conf
worker_processes 4;
events {
  worker_connections 1024;
}
http {
  upstream nodes {
    # enable sticky session based on IP
    ip_hash;
    server app01:3000;
    server app02:3000;
    server app03:3000;
  }
  server {
    listen 3000;
    server_name io.yourhost.com;
    location / {
      proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
      proxy_set_header Host $host;
      proxy_pass http://nodes;
      # enable WebSockets
      proxy_http_version 1.1;
      proxy_set_header Upgrade $http_upgrade;
      proxy_set_header Connection "upgrade";
    }
  }
}
```

* Httpd配置 [example](https://github.com/socketio/socket.io/tree/master/examples/cluster-httpd)

* HAProxy 配置 [example](https://github.com/socketio/socket.io/tree/master/examples/cluster-haproxy)

* 使用Node.js Cluster集群
  * node.js内置cluster模块支持集群
  * [sticky_session](https://github.com/indutny/sticky-session)模块可以确保请求基于起始ip地址`remoteAddress`，注意这种方式有可能导致非均衡的路由，它依赖于hash方法。
  * 也可以为每个工作进程开启不同的端口，通过代理服务器的配置（Nginx）来实现负载均衡。

* 在集群节点中传递消息
 * 通过实现路由消息的接口[Adapter](https://github.com/socketio/socket.io-adapter)，来实现消息的订阅`sub`和发布`pub`，如[socket.io-redis](https://github.com/socketio/socket.io-redis) Adapter。
  ```js
  var io = require('socket.io')(3000);
  var redis = require('socket.io-redis');
  io.adapter(redis({ host: 'localhost', port: 6379 }));
  io.emit('hi', 'all sockets');
  ```

## logging and debugging
* log和debug现在完全通过TJ开发的[DEBUG](https://github.com/visionmedia/debug)来管理。
* 基本思想是Socket.IO使用的每个模块都提供不同的调试范围
* 默认情况下，所有输出都被禁止，您可以通过设置`DEBUG`env变量（Node.JS）或`localStorage.debug`属性（浏览器）来选择查看消息
```bash
# 查看所有
DEBUG=* node youfile.js
# for browser
# localStorage.debug = '*'

# 仅查看socket.io的client的信息
DEBUG=socket.io:client* node yourfile.js
# 仅查看engine和socket.io的信息
DEBUG=engine,socket.io* node yourfile.js
```

## emit cheatsheet
```js
io.on('connect', (socket) => {

  // 发送给客户端（发送者）
  socket.emit('hello', 'can you hear me?', 1, 2, 'abc');

  // 发送给所有客户端，除了发送者
  socket.broadcast.emit('broadcast', 'hello friends!');

  // 发送给所有在'game'房间的客户端，除了发送者
  socket.to('game').emit('nice game', "let's play a game");

  // 发送给所有在'game1'房间或'game2'房间的客户端，除了发送者
  socket.to('game1').to('game2').emit('nice game', "let's play a game (too)");

  // 发送给所有在'game'房间的客户端，包括发送者
  io.in('game').emit('big-announcement', 'the game will start soon');

  // 发送给所有在'myNamespace'命名空间的客户端，包括发送者
  io.of('myNamespace').emit('bigger-announcement', 'the tournament will start soon');

  // 发送给所有在'myNamespace'命名空间的'room'房间的客户端，包括发送者
  io.of('myNamespace').to('room').emit('event', 'message');

  // 发送给某个客户端（私信）
  io.to(`${socketId}`).emit('hey', 'I just met you');

  // 警告：通过 `socket.to(socket.id).emit()`发送给某个人不起作用，它将会发给所有在 `socket.id`房间的人，除了发送者，使用 `socket.emit()`代替

  // 发送并带确认消息（回调）
  socket.emit('question', 'do you think so?', function (answer) {});

  // 发送并取消压缩
  socket.compress(false).emit('uncompressed', "that's rough");

  // 发送的消息可能会被丢弃，如果客户端没有准备好去接收消息的话
  socket.volatile.emit('maybe', 'do you really need it?');

  // 设置发送的数据是否携带二进制数据
  socket.binary(false).emit('what', 'I have no binaries!');

  // 发送给节点所有客户端（当使用多节点集群时）
  io.local.emit('hi', 'my lovely babies');

  // 发送给所有连接的客户端
  io.emit('an event sent to all connected clients');
});
```
注：保留事件，应用中不可使用:
* error
* connect
* disconnect
* disconnecting
* newListener
* removeListener
* ping
* pong

## internal overview
0. 依赖图[graph](https://socket.io/images/dependencies.jpg)
1. engine.io-parser
  * `engine.io` [protocol](https://github.com/socketio/engine.io-protocol)编码的JavaScript parser
  * 被`engine.io`和`engine.io-client`所依赖

2. engine.io
  * 为`Socket.IO`实现基于传输的跨浏览器/跨设备双向通信层
  * 主要特点是即时交换传输，由`engine.io-client`以XHR轮询开始，如果有可能的话，切换到`WebSocket`
  * 使用`engine.io-parser`对数据包进行编码/解码

3. engine.io-client
  * `engine.io`的客户端，为`Socket.IO`实现基于传输的跨浏览器/跨设备双向通信层
  * 可在浏览器（包括H5 WebWorker）和Node.js中运行
  * 使用`engine.io-parser`对数据包进行编码/解码

4. socket.io-adapter
  * 默认`Socket.IO`的内存adapter，不可直接使用
  * 可作为接口继承扩展使用，如`socket.io-redis`

5. socket.io-redis
  * 只是一个实现了socket.io-adapter的适配器
  * 利用`redis`的[`Pub/Sub`](https://redis.io/topics/pubsub)机制来在多个节点(集群)之间广播消息

6. socket.io-parser
  * 一个`socket.io`的编解码器，使用JavaScript编写
  * 遵照[`socket.io-protocol`](https://socket.io/docs/internals/)3.0版本协议规范

7. socket.io
  * 基于`engine.io`原始API的一些语法糖
  * 2个新概念：`Rooms`和`Namespaces`，用来在通信通道中进行关注点分离
  * 使用`socket.io-parser`编解码数据包

8. socket.io-client
  * `socket.io`的客户端
  * 依赖`engine.io-client`来管理传输交换和断连检测
  * 自动处理重新连接，防止底层连接被切断
  * 使用`socket.io-parser`编解码数据包

### under the hood
1. 建立连接：
  * `engine.io-client`建立客户端连接
  ```js
    // 在客户端创建一个`engine.io-client`的实例
    // 实例尝试去建立一个长轮询（polling）传输
    const client = io('http://localhost');
    // GET http://localhost/socket.io/?EIO=3&transport=polling&t=ML4jUwU&b64=1
    // "EIO=3"               # 当前Engine.IO protocol版本号
    // "transport=polling"   # 建立的传输类型
    // "t=ML4jUwU&b64=1"     # 用于更新缓存的hash时间戳
  ```
  * `engine.io`进行服务端响应
  ```bash
    {
      "type": "open",
      "data": {
        "sid": "36Yib8-rSutGQYLfAAAD",  // 唯一的session id
        "upgrades": ["websocket"],      // 可升级的传输类型列表
        "pingInterval": 25000,          // 心跳机制的第一个参数
        "pingTimeout": 5000             // 心跳机制的第二个参数
      }
    }
  ```
  * `engine.io-parser`进行内容编码
  ```bash
    '96:0{"sid":"hLOEJXN07AE0GQCNAAAB","upgrades":["websocket"],"pingInterval":25000,"pingTimeout":5000}2:40'
    "96"  # 首条消息的长度
    ":"   # 长度和内容之间的分隔符
    "0"   # "open"消息类型
    '{"sid":"hLOEJXN07AE0GQCNAAAB","upgrades":["websocket"],"pingInterval":25000,"pingTimeout":5000}' # JSON编码的握手（handshake）数据
    "2"   # 第二条消息的长度
    ":"   # 长度和内容之间的分隔符
    "4"   # "message"消息的类型
    "0"   # "open" 在Socket.IO protocol中的消息类型
  ```
  * `engine.io-parser`进行内容解码
  * `engine.io-client`触发`open`事件
  * `engine.io-client`触发`connect`事件

2. 连接升级
  * 一旦现有的传输（XHR轮询）缓存被刷新，将会发送一个升级测试请求：
  ```bash
    GET wss://localhost/socket.io/?EIO=3&transport=websocket&sid=36Yib8-rSutGQYLfAAAD
    "EIO=3"                     # 当前Engine.IO protocol版本号
    "transport=websocket"       # 新的传输类型
    "sid=36Yib8-rSutGQYLfAAAD"  # 唯一的session id
  ```
  * 客户端将会发送一个`ping`数据包，编码为`2prob`(通过`engine.io-parser`)，`2`为`ping`消息类型
  * 客户端将会发送一个`ping`数据包，编码为`2prob`(通过`engine.io-parser`)，`3`为`pong`消息类型
  * 一旦收到`pong`数据包，升级即视为已完成，所有后续消息将采用新的方式传输

## FAQ
* 使用通配符事件：[plugin](https://github.com/hden/socketio-wildcard)
* 和Cordova配合使用：[tutorial](https://socket.io/socket-io-with-apache-cordova/)
* 在iOS上使用：[SIOSocket](https://github.com/MegaBits/SIOSocket)
* 在Android上使用：[socket.io-client.java](https://github.com/nkzawa/socket.io-client.java)