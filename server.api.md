# server.api

## Server

### 实例化
```js
// 创建io服务端
const httpServer = require('http').createServer();
// create socket io server...
httpServer.listen(3000);

/**
 * 基于已有httpServer创建io服务端
 * 
 * @param {http.Server} httpServer
 * @param {Object} [options]
 * @return {Server}
 */
new Server(httpServer[, options])

const io = require('socket.io')(httpServer, {});
// or
const Server = require('socket.io');
const io = new Server(httpServer, {});

/**
 * 指定端口，即创建一个新的http.Server
 * 
 * @param {Number} port - http port
 * @param {Object} [options]
 * @return {Server}
 */
new Server(port[, options])

const io = require('socket.io')(3000, {});
// or
const Server = require('socket.io');
const io = new Server(3000, {});

/**
 * 创建一个io服务端，然后挂载`attach`到http.server上
 * 
 * @param {Object} options
 */
new Server(options)

const io = require('socket.io')({});
io.attach(httpServer, {});
httpServer.listen(3000);
// or
io.attach(3000, {});

/**
 * 创建io服务端的配置选项options 
 */
const opts = {
 path: '/socket.io', // 捕捉的连接路径
 serveClient: true, // 是否服务客户端静态文件`opts.path+/socket.io.js`
 adapter: -, // 自定义适配器，默认socket.io-adapter
 origins: '*', // 允许的跨域地址 
 parser: -, // 自定义解析器，默认socket.io-parser

 // 底层 Engine.IO server的可用选项：
 pingTimeout: 5000, // 多少ms没有收到pong包，认为连接已关闭
 pingInterval: 25000, // 发送一个新的ping包之前需要等待多少ms
 upgradeTimeout: 10000, // 传输升级请求必须在多少ms内完成，否则会被取消
 maxHttpBufferSize: 10e7, // 一次消息最多可包含多少个字节或字符（避免DoS）
 allowRequest: function(request,(err,result)=>{}){}, // request为handshade或upgrade请求，err：错误code，result：Boolean是否允许请求继续，false表示拒绝请求
 transports: ['polling', 'websocket'], // 允许的传输方式列表
 allowUpgrades: true, // 是否允许传输方式升级
 perMessageDeflate: true, // 是否允许websocket的[permessage-deflate](https://github.com/einaros/ws)扩展
 httpCompression: true, // 是否允许zlib,deflate等方式压缩
 cookie: io, // 在cookie中设置io:socket.id，设置false取消设置
 cookiePath: '/', // 设置上面cookie选项的作用路径，设置false则所有请求中不会包含io cookie设置
 cookieHttpOnly: true, // 不允许客户端脚本(js)访问，不受cookie和cookiePath选项的影响
 wsEngine: ws // websocket服务器的实现，必须符合[ws接口](https://github.com/websockets/ws/blob/master/doc/ws.md)
}
// 注1：（pingTimeout + pingInterval）参数将影响客户端获取disconnect事件（即服务端不可用）之前需要等待最多多少ms
// 注2：transports选项非常重要，通常是长轮询开始，然后尝试升级到websocket，如果只指定['websocket']则在websocket无法连接时没有了后备。
```

### 属性
```js
/**
 * 默认命名空间`/`的别名
 * 
 * @return {Namespace} 等同于 === io.of('/') === io.nsps['/'] 
 */
io.sockets;

io.sockets.emit('hi', 'everyone');
// 相当于
io.of('/').emit('hi', 'everyone');
```

### 方法
```js
/**
 * 设置或返回是否服务客户端静态文件`io.path()+/socket.io.js`
 * 存取属性
 * 
 * @param {Boolean} value 默认：true
 * @return {Server|Boolean}
 */
io.serveClient([value])

const io = require('socket.io')(httpServer, { serveClient: false });
// or ，但在attach方法后面设置没有效果
const io = require('socket.io')();
io.serveClient(false);
io.attach(httpServer);

/**
 * 设置或获取服务端捕捉的连接路径，也是服务客户端静态文件的路径
 * 存取属性
 * 
 * @param {String} [value] 默认：'/socket.io'
 * @return {Server|String}
 */
io.path([value])

/**
 * 设置或获取服务端的适配器实例，默认socket.io-adapter
 * 存取属性
 * 
 * @param {Adapter} [value]
 * @return {Server|Adapter}
 */
io.adapter([value])

const io = require('socket.io')(3000);
const redis = require('socket.io-redis');
io.adapter(redis({ host: 'localhost', port: 6379 }));

/**
 * 设置或返回服务端的跨域设置
 * 存取属性
 * 
 * @param {String|String[]} [value]
 * @return {Server|String}
 */
server.origins([value])

io.origins(['https://foo.example.com:443']);

/**
 * 自定义服务端的跨域设置
 * 
 * 缺点：
 * 在某些条件下，当无法确定origin时，它的值可能是*
 * 每次请求回调都会被执行，所以应使它尽可能快的执行
 * 如果和Express配合使用，CORS headers仅仅对socket.io请求起作用，对于Express则可以使用cors中间件。 
 * 
 * @param {Function} fn(error, Boolean)
 * @return {Server}
 */
io.origins(fn)

io.origins((origin, callback) => {
  if (origin !== 'https://foo.example.com') {
    return callback('origin not allowed', false); // failure
  }
  callback(null, true); // success
});

/**
 * 将io实例挂载到http.server
 * 
 * @param {http.server} httpServer
 * @param {Object} [options]
 * @return {Server}
 */
io.attach(httpServer[, options])
io.listen(httpServer[, options])

/**
 * 将io实例挂载到一个新的http.server
 * 
 * @param {String} port - a new httpServer
 * @param {Object} [options]
 * @return {Server}
 */
io.attach(port[, options])
io.listen(port[, options])

/**
 * 重新绑定底层engine.io引擎，高级
 * 
 * @param {engine.Server} engine.io
 * @return {Server} 
 */
io.bind(engine)

/**
 * 创建新的socket.io客户端，高级
 * 
 * @param {engine.Socket} socket.io
 * @return {Server} 
 */
io.onconnection(socket)

/**
 * 初始化并获取指定命名空间的sockets连接池
 * 
 * @param {String|RegExp|Function} nsp
 * @return {Namespace}
 */
io.of(nsp)

// string
const adminNamespace = io.of('/admin');

// regexp
const dynamicNsp = io.of(/^\/dynamic-\d+$/).on('connect', (socket) => {
  const newNamespace = socket.nsp; // newNamespace.name === '/dynamic-101'
  // 广播消息给所有在sub-namespace的客户端
  newNamespace.emit('hello');
});
// client-side
const socket = io('/dynamic-101');
// 广播消息给每个sub-namespace的所有客户端
dynamicNsp.emit('hello');
// 使用一个中间件，针对每个sub-namespace
dynamicNsp.use((socket, next) => { /* ... */ });

// function
io.of((name, query, next) => {
  next(null, checkToken(query.token));
}).on('connect', (socket) => { /* ... */ });

/**
 * 关闭socket.io服务端，当所有连接关闭后调用callback
 * 
 * @param {Function} callback
 */
io.close([callback])

const Server = require('socket.io');
const httpServer = require('http').Server();
const port = 3030;
const io = Server(port);
io.close(); // 关闭io服务器，释放端口占用
httpServer.listen(port); // 端口可用
io = Server(httpServer); // 再次启动

/**
 * 自定义引擎生成的socket.id，必须保证唯一
 * 
 * @return {String} uuid
 */
io.engine.generateId

// req: http.IncomingMessage
io.engine.generateId = (req) => {
  return "custom:id:" + custom_id++; // custom id must be unique
}
```

## Namespace
通过路径标识的sockets连接池，默认命名空间路径为'/'，不同的命名空间使用相同的底层连接。
```js
const nsp = io.sockets; // default '/' namespace
const nsp = io.of('/');
const nsp = io.nsps['/'];
io.on('connect', (socket) => {
  const nsp = socket.nsp;
});
```

### 属性
```js
/**
 * 获取命名空间的标识路径，默认为 '/'
 * 
 * @return {String}
 */
nsp.name;

/**
 * 获取命名空间连接池内的sockets客户端实例
 * 
 * @return {Object} {socket.id: Socket}
 */
nsp.connected;

/**
 * 获取命名空间使用的adapter实例
 * 对于使用其它适配器，如socket.io-redis很有用
 * 因为它公开了管理集群中的sockets和rooms的方法
 * 
 * @return {Adapter} socket.io-adapter
 */
nsp.adapter;

io.of('/').adapter; // 默认命名空间的适配器

/**
 * 获取命名空间下的所有房间
 * 
 * @return {String[]}
 */
nsp.rooms;
```

### 方法
```js
/**
 * 设置向指定房间发送广播消息，包括发送者
 * 
 * @param {String} room
 * @return {Namespace}
 */
nsp.to(room)
nsp.in(room)

nsp.to('room1').to('room2').emit('event', { some: 'data' });
nsp.in('room1').emit('event', { some: 'data' });

/**
 * 发送消息给命名空间下所有连接的客户端
 * 注意：命名空间消息不支持消息确认
 * 
 * @param {String} eventName
 * @param {String[]} [...args]
 */
nsp.emit(eventName[, …args])

io.emit('an event '); // 默认命名空间
io.of('/chat').emit('an event'); // chat命名空间

/**
 * 获取连接到此命名空间（或下面某个房间）的所有客户端ID列表
 * 如果适用，则跨所有节点
 * 
 * @param {Function} callback(error, clients) - clients: [socket.id,....] 
 */
nsp.clients(callback)

// default namespace === io.of('/').clients(callback)
io.clients((error, clients) => {
  if (error) throw error;
  console.log(clients); // => [PZDoMH..., Anw2La...]
});
// chat namespace room1
io.of('/chat').in('room1').clients((error, clients) => {
  if (error) throw error;
  console.log(clients); // => [Anw2L..., PDFXss...]
});

/**
 * 给命名空间注册一个中间件函数，处理每个socket连接
 * 
 * @param {Function} fn(socket,next)
 */
nsp.use(fn);

io.use((socket, next) => {
  if (socket.request.headers.cookie) return next();
  next(new Error('Authentication error'));
});
```

### 事件
```js
// 客户端连接事件 connect or connection
io.on('connect', (socket) => {});
io.on('connection', (socket) => {});
io.of('/admin').on('connect', (socket) => {});
io.of('/admin').on('connection', (socket) => {});
```

### Flag
```js
// Flag: volatile
// 发送易失消息，如果客户端未准备好，因不可知的网络等原因
// 客户端可能收到，也可能收不到
io.volatile.emit('an event', { some: 'data' }); 

// Flag: binary
// 设置后续发送的消息是否带有二进制数据，提升执行效率
io.binary(false).emit('an event', { some: 'data' });

// Flag: local
// 给当前节点的客户端发送广播消息，当使用多节点(集群)时使用（如redis adapter）
io.local.emit('an event', { some: 'data' });
```

## Socket
获取连接的客户端socket信息
```js
const io = require('socket.io')(http);
io.on('connection', (socket) => {
  console.log(socket);
});
```

### 属性
```js
/**
 * 获取会话id，来自底层`Client`
 * 
 * @return {String}
 */
socket.id

/**
 * 获取socket加入的房间Object，默认包含自身socket.id的key:value对
 * 
 * @return {Object} {<socket.id>:socket.id}
 */
socket.rooms

io.on('connection', (socket) => {
  socket.join('room 237', () => {
    let rooms = Object.keys(socket.rooms);
    console.log(rooms); // [ <socket.id>, 'room 237' ]
  });
});

/**
 * 获取底层的Client对象，即engine.io-client
 * 
 * @return {Client}
 */
socket.client;

/**
 * 获取底层的连接信息，等于 ==== socket.client.conn
 * 
 * @return {engine.Socket}
 */
socket.conn;

/**
 * 获取底层的连接请求信息，等于 ==== socket.client.request
 * headers: {Obejct} 请求头对象
 *  - accept:{String} 接受的Mime类型 '*\/*'
 *  - accept-encoding: {String} 编码方式 "gzip, deflate, br"
 *  - accept-language:{String} 语言及权重 "zh-CN,zh;q=0.9,en-US;q=0.8,en;q=0.7"
 *  - connection:{String} 连接方式 "keep-alive"
 *  - cookie:{String} cookie "io=KA8ni6rCIdMY6KITAAAG"
 *  - host: {String} 请求的服务器地址 localhost:3000
 *  - referer: {String} 页面来自 "http://localhost:3000/"
 *  - user-agent: {String} 用户代理 "Mozilla/5.0 ..."
 * method: {String} 请求方法 'GET'
 * httpVersion: {String} http版本 1.1
 * aborted: {Boolean} 是否取消
 * complete: {Boolean} 是否完成
 * url: {String} 请求url地址
 * res: {ServerResponse} 响应对象
 * socket: {Socket} socket对象
 * 
 * @return {Request}
 */
socket.request;

/**
 * 访问握手信息
 * 
 * headers: {Object} 请求头对象
 * query: {Object} 查询字符串对象
 * secure: {Boolean} 连接是否安全
 * xdomain: {Boolean} 连接是否跨域
 * time: {String} 创建的时间
 * issued: {Number} 创建的时间（unix时间戳）
 * address: {String} 客户端ip地址
 * url: {String} 请求的URL地址
 * 
 * @return {Object}
 */
socket.handshake;

io.use((socket, next) => {
  let handshake = socket.handshake;
});
// or
io.on('connection', (socket) => {
  let handshake = socket.handshake;
});

/**
 * 返回socket所在的命名空间连接池
 * 
 * @return {Namespace}
 */
socket.nsp;
```

### 方法
```js
/**
 * 注册一个中间件函数，每次客户端发送消息时都会经过中间件函数
 * 连接阶段`connection`不会经过中间件函数
 *
 * fn的参数packet为一个数组: [eventName, eventData]
 * @param {Function} fn(packet,next)
 */
socket.use(fn);

io.on('connection', (socket) => {
  socket.use((packet, next) => {
    if (packet.doge === true) return next();
    next(new Error('Not a doge error'));
  });
});

/**
 * 发送一个`message`事件给客户端
 * 
 * @see emit()
 */
socket.send([…args][, ack])

/**
 * 发送事件给客户端
 * 
 * @param {String} eventName 事件名称
 * @param {Any} [...args] 支持所有可序列号数据，包括`Buffer`
 * @param {Function} [ack] 消息确认回调
 * @return {Socket}
 */
socket.emit(eventName[, …args][, ack])

io.on('connection', (socket) => {
  socket.emit('with-binary', 1, '2', { 3: '4', 5: new Buffer(6) });
  socket.emit('hello', 'world', (data) => {
    console.log(data); // 'world'
  });
});
// client code
client.on('hello', (name, fn) => {
  fn(name);
});


/**
 * 注册监听客户端事件，触发回调，并可带消息确认
 * 
 * @param {String} eventName
 * @param {Function} callback
 * @return {Socket}
 */
socket.on(eventName, callback)

socket.on('news', (data) => {
  console.log(data);
});
// 带多个参数
socket.on('news', (arg1, arg2, arg3) => {
  // ...
});
// 带消息确认
socket.on('news', (data, callback) => {
  callback(0);
});

/**
 * socket继承`EventEmitter`
 * 参考Node.js的`events`模块
 */
socket.once(eventName, listener)
socket.removeListener(eventName, listener)
socket.removeAllListeners([eventName])
socket.eventNames()

/**
 * 将客户端加入某个房间
 * 默认客户端会被加入以自身id为key的房间，这很方便私聊
 * 加入房间的机制由Adapter负责处理，
 * 
 * @param {String} room 房间名称
 * @param {Function} callback 回调，携带err参数
 */
socket.join(room[, callback])

io.on('connection', (socket) => {
  socket.join('room 237', (err) => {
    let rooms = Object.keys(socket.rooms);
    console.log(rooms); // [ <socket.id>, 'room 237' ]
    io.to('room 237').emit('a new user has joined the room'); // broadcast to everyone in the room
  });
  // 私聊
  socket.on('say to someone', (id, msg) => {
    // 同id客户端私聊
    socket.to(id).emit('my message', msg);
  });
});

/**
 * 将客户端加入多个房间
 * 
 * @param {Array} [rooms] 房间数组
 * @param {Function} callback 回调，携带err参数
 */
socket.join(rooms[, callback])

/**
 * 将客户端退出某个房间
 * 
 * 注意：断开连接后会自动离开房间
 * 
 * @param {String} room 房间名称
 * @param {Function} callback 回调，携带err参数
 * @return {Socket}
 */
socket.leave(room[, callback]);

/**
 * 向房间内的所有客户端发送broadcast消息，除了发送者
 * 注意：broadcast消息不支持消息确认
 * 
 * @param {String} room 房间名称
 * @return {Socket}
 */
socket.to(room);

io.on('connection', (socket) => {
  // to one room
  socket.to('others').emit('an event', { some: 'data' });
  // to multiple rooms
  socket.to('room1').to('room2').emit('hello');
  // 私聊
  socket.to(/* another socket id */').emit('hey');
  // 警告：使用`socket.to(socket.id).emit()`会将消息发送给在名称为`socket.id`的房间内的所有客户端，不会回复给发送者，要回复发送者，使用 `socket.emit()`即可
});

/**
 * 向房间内的所有客户端发送broadcast消息，除了发送者
 * 
 * @see to()
 */
socket.in(room)

/**
 * 发送消息时是否压缩传输
 * 
 * @param {Boolean} value 默认: true
 * @return {Socket}
 * 
 */
socket.compress(value)

io.on('connection', (socket) => {
  socket.compress(false).emit('uncompressed', "that's rough");
});

/**
 * 断开客户端或断开namespace
 * value: true 断开客户端，否则断开namespace
 * 
 * @param {Boolean} value 
 * @return {Socket}
 */
socket.disconnect(close)

io.on('connection', (socket) => {
  setTimeout(() => socket.disconnect(true), 5000);
});
```

### Flag
```js
// Flag: 'broadcast' 标记后面发送的消息是广播消息
// Flag: 'volatile' 标记后面发送的消息是易失消息
// Flag: 'binary' 标记后面发送的消息是否带有二进制内容
io.on('connection', (socket) => {
  socket.broadcast.emit('an event', { some: 'data' }); // 发送给所有客户端，除了发送者
  socket.volatile.emit('an event', { some: 'data' }); // 客户端可能收到也可能收不到消息
  socket.binary(false).emit('an event', { some: 'data' }); // 消息数据不含二进制内容
});
```

### 事件
```js
/**
 * 客户端断开时触发，可能是服务端的原因，也可能是客户端的原因
 * 
 * Server reason: 
 *  - transport error
 *  - server namespace disconnect // socket.disconnect()
 * 
 * Client reason:
 *  - client namespace disconnect
 *  - ping timeout // pingTimeout config
 *  - transport close
 * 
 * @param {String} reason
 */
io.on('connection', (socket) => {
  socket.on('disconnect', (reason) => {
    // ...
  });
});

/**
 * 当发生错误时触发
 * 
 * @param {Object} error
 */
io.on('connection', (socket) => {
  socket.on('error', (error) => {
    // ...
  });
});

/**
 * 当客户端将要断开尚未离开房间时触发
 * 
 * @param {Object} reason - client or server reason
 */
io.on('connection', (socket) => {
  socket.on('disconnecting', (reason) => {
    let rooms = Object.keys(socket.rooms);
    // ...
  });
});

/**
 *  上述事件名称与`connect`, `newListener` 和 `removeListener`一起为保留名称
 *  自定义名称时不可使用
 */
```

## Client
socket底层的`Client`来自`engine.io`的连接信息，一个`Client`可能会关联多个不同命名空间的socket
```js
const io = require('socket.io')(http);
io.on('connection', (socket) => {
  const client = socket.client;
});
```

### 属性
```js
// 连接信息 === socket.conn
const conn = client.conn;

// 请求信息 === socket.request
const req = client.request;
const headers = req.headers; // {host,cookie,user-agent,referer,...}
```