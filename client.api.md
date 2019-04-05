# client.api

## IO
客户端的io通常暴露为一个全局对象或作为模块输出为一个函数，通常也包含一些属性
```js
// import
const io = require('socket.io-client');
import io from 'socket.io-client';

// type
typeof io;  // function

// props
io.name;     // 'lookup' -> constructor
io.Manager;  // function Manager(url, opts)，Manager类
io.Socket;   // function Socket(io, nsp, opts)，Socket类
io.connect;  // function lookup(url, opts)，io的构造函数
io.managers; // {id:socket} 缓存所有Manager创建的socket对象
io.protocol; // 协议版本号 4
```

## 实例化
* 实例化一个客户端对象，实际上就是实例化一个Manager对象`new Manager(url, opts)`，Manager内部再调用原型方法`socket`，即`new Socket(io, nsp, opts)`返回一个具有namespace的Socket实例对象。
```js
/**
 * 通过Manager创建一个具有namespace的Socket对象
 * 
 * options中的`forceNew: true`,`'force new connection': true`,`multiplex: false`（来自于Manager配置选项），这三个选项作用一样，即创建一个新的连接，而不使用已有的连接，所有的选项都会被传给Manager的构造函数。
 * 
 * @param {string} [url=window.location] - 连接的URL地址，默认为window.location，如果有目录地址，会被解析为Socket对象的namespace，可以携带query查询参数，也可以在options中配置查询参数
 * @param {object} [options] - 参考`new Manager(url[,options])`的options
 * @return {object} - Socket对象，namespace默认为/，url的目录地址会被解析为namespace
 */
function io(url, options){}

// init 1: 创建多个连接
// 只有1个连接
const socket1 = io();
const socket2 = io('/admin');
// 2个连接
const socket1 = io();
const socket2 = io('/admin', {forceNew: true});
// 2个连接
const socket1 = io();
const socket2 = io();

// init 2: 自定义连接路径
// url: localhost/myownpath/?EIO=3&transport=polling&sid=<id>
// namespace: /
const socket = io('http://localhost', {
  path: '/myownpath'
});
// server-side
const io = require('socket.io')({
  path: '/myownpath'
});
// url: localhost/mypath/?EIO=3&transport=polling&sid=<id>
// namespace: /admin
const socket = io('http://localhost/admin', {
  path: '/mypath'
});

// init3: 携带query查询参数
const socket = io('http://localhost?token=abc');
const socket = io({
  query: {token: 'cde'}
});
socket.on('reconnect_attempt', () => {
  socket.io.opts.query = {token: 'fgh'}
});
// server-side
const io = require('socket.io')();
// middleware
io.use((socket, next) => {
  let token = socket.handshake.query.token;
  if (isValid(token)) {
    return next();
  }
  return next(new Error('authentication error'));
});
// then
io.on('connection', (socket) => {
  let token = socket.handshake.query.token;
  // ...
});

// init4: 携带自定义的extraHeaders
// 这个选项仅适用于长轮询`polling`传输方式（这也是默认的方式）
// 不适用于`WebSocket`传输方式，ws握手阶段不允许添加自定义的header信息
const socket = io({
  transportOptions: {
    polling: {
      extraHeaders: {'x-clientid': 'abc'}
    }
  }
});
// server-side
const io = require('socket.io')();
// middleware
io.use((socket, next) => {
  let clientId = socket.handshake.headers['x-clientid'];
  if (isValid(clientId)) {
    return next();
  }
  return next(new Error('authentication error'));
});

// init5: 自定义传输方式websocket
// 默认首先是`polling`，然后尝试升级到`websocket`
const socket = io({
  transports: ['websocket']
});
// 可在重新连接阶段重置传输方式
// 因为websocket方式可能会因代理，防火墙或浏览器等原因失败
socket.on('reconnect_attempt', () => {
  socket.io.opts.transports = ['polling', 'websocket'];
});

// init6: 使用自定义parser
// 默认parser为了兼容性（支持blob,file,binary check等），牺牲了些性能
// 自定义parser可满足应用的特殊需要
const parser = require('socket.io-msgpack-parser'); 
// or require('socket.io-json-parser')
const socket = io({
  parser: parser
});
// 服务端必须有同样的parser
const io = require('socket.io')({
  parser: parser
});

// init7: 带有自签名证书
// 客户端
const socket = io({
  // 选项1
  ca: fs.readFileSync('server-cert.pem'), // 签发证书
  // 选项2，警告：这可能会遭受中间人攻击(MITM)
  rejectUnauthorized: false
});
// 服务端
const fs = require('fs');
const server = require('https').createServer({
  key: fs.readFileSync('server-key.pem'), // 私钥
  cert: fs.readFileSync('server-cert.pem') // 签发证书
});
const io = require('socket.io')(server);
server.listen(3000);
```

## Manager类
### 构造函数
```js
/**
 * 创建一个Manager对象
 * @param {string} url 连接地址
 * @param {object} [options] 可配置选项
 * @return {object} Manager对象
 */
function Manager(url, options) {
  // default options: 
  var default = {
    path:'/socket.io', // 服务器端捕获的路径名称
    reconnection:true, // 是否自动重新连接
    reconnectionAttempts:Infinity, // 重新连接尝试次数
    reconnectionDelay:1000, // 在重新连接之前等待多长时间，受随机因子+-的影响，默认延迟在500~1500之间
    reconnectionDelayMax:5000, // 重新连接之间等待的最长时间，每次尝试连接将reconnectionDelay与randomizationFactor一起增加2倍
    randomizationFactor:0.5, // 随机因子：0 <= n <= 1
    timeout:20000, // 触发`connect_error`和`connect_timeout`事件之前的超时时间
    autoConnect:true, // 如果为false则需要手动调用`manager.open()`方法来自动连接
    query:{}, // 查询字符串，服务端可从socket.handshake.query获取
    parser: - // 自定义解析器，默认socket.io-parser
  }
  //  基于Engine.io-client可用options:
  var default = {
    upgrade: true, // 客户端是否应该尝试将传输从长轮询升级到更好的状态
    forceJSONP: false, // 是否强制使用JSONP进行轮询传输
    jsonp: true, // 确定是否在必要时使用JSONP进行轮询，避免“无传输可用”的错误发生
    forceBase64: false, // 强制长轮询使用base64编码，即使支持XHR2和WebSocket
    enablesXDR: false, // 是否为IE8开启DomainRequest支持,避免加载条闪烁和点击声音，但XDR不允许携带cookie
    timestampRequests: -, //是否为每个传输请求添加时间戳，长轮询默认总是添加，除非此项设置为false
    timestampParam: t, // 时间戳参数
    policyPort: 843, // 策略服务器侦听的端口
    transports: ['polling', 'websocket'], // 传输列表（按顺序尝试）
    transportOptions: {}, // 由传输名称索引，覆盖常规的传输方式选项
    rememberUpgrade: false, // 记住先前与服务端的连接方式，下次连接时默认采用这种传输方式，如果错误，则尝试正常的升级方式。建议仅在SSL连接时打开
    onlyBinaryUpgrades: false, // 传输升级是否被限制为仅在传输二进制数据时升级
    requestTimeout: 0, // xhr-polling请求的超时时间（毫秒）（仅适用于轮询传输）
    protocols: -, // [子协议](https://developer.mozilla.org/en-US/docs/Web/API/WebSockets_API/Writing_WebSocket_servers#Subprotocols)列表，仅适用于websocket传输
  }

  // 基于Engine.io-client的Node.js可用options:
  var default = {
    agent: false, // this.agent使用
    pfx: -, // SSL的证书，私钥和CA证书
    key: -, // SSL的私钥
    passphrase: -, // 私钥或pfx的密码短语
    cert: -, // 使用的公共x509证书
    ca: -, // 用于检查远程主机的授权证书或授权证书数组
    ciphers: -, // 描述要使用或排除的密码的字符串[cipher](http://www.openssl.org/docs/apps/ciphers.html#CIPHER_LIST_FORMAT)
    rejectUnauthorized: false, // 如果为true,则会在支持的CA证书列表中验证服务端凭证，验证失败会触发error事件，验证发生在连接阶段，即http请求之前。
    perMessageDeflate: true, // WebSocket permessage-deflate扩展的参数[ws](https://github.com/einaros/ws)
    extraHeaders: {}, // 自定义header参数给每个服务端的请求（xhr-polling or websocket），可在握手期间或特殊代理中使用这些值
    forceNode: false, // 对websockets强制使用NodeJS的实现，即使有可用的本机Browser-Websocket（如：nw.js，electron），默认情况下优先使用NodeJS的实现。
    localAddress: -, // 要连接的本地IP地址
  }
}

// 获取manager对象
const socket = io();
const manager = socket.io;
```
### 属性（存取器属性）
```js
/**
 * 设置或返回reconnection选项值
 * @param {Boolean} [value]
 * @return {Manager|Boolean}
 */
manager.reconnection([value])

/**
 * 设置或返回reconnectionAttempts选项值
 * @param {Number} [value]
 * @return {Manager|Number}
 */
manager.reconnectionAttempts([value])

/**
 * 设置或返回reconnectionDelay选项值
 * @param {Number} [value]
 * @return {Manager|Number}
 */
manager.reconnectionDelay([value])

/**
 * 设置或返回reconnectionDelayMax选项值
 * @param {Number} [value]
 * @return {Manager|Number}
 */
manager.reconnectionDelayMax([value])

/**
 * 设置或返回timeout选项值
 * @param {Number} [value]
 * @return {Manager|Number}
 */
manager.timeout([value])
```

### 方法
```js
/**
 * 如果初始化时autoConnect选项设置为false，调用此方法启动一个新的连接尝试，callback将在启动成功或失败是被调用
 * @param {Function} [callback]
 * @return {Manager} 
 */
manager.open([callback])

/**
 * @see manager.open([callback])
 */
manager.connect([callback])

/**
 * 创建一个具有nsp命名空间的Socket对象
 * @param {String} nsp
 * @param {Object} options
 * @return {Socket}
 */
manager.socket(nsp, options)
```

### 事件
```js
// 连接错误事件
manager.on('connect_error', (error) => {}); // error为错误对象
socket.on('connect_error', (error) => {}); // error为错误对象

// 连接超时事件
manager.on('connect_timeout', (timeout) => {}); // timeout为超时时间

// 重新连接事件
manager.on('reconnect', (attempt) => {}); // 成功时发生，attempt为尝试的次数
manager.on('reconnect_attempt', (attempt) => {}); // 重新连接时发生
manager.on('reconnecting', (attempt) => {}); // 重新连接时发生，attempt为尝试的次数
manager.on('reconnect_error', (error) => {}); // 错误时发生，error为错误对象
manager.on('reconnect_fail', (attempt) => {}); // 失败时发生，attempt为尝试的次数

// 乒乓事件
manager.on('ping', (data) => {}); // client ping server
manager.on('pong', (ms) => {}); // server response client, ms为`ping`以来花费的毫秒数
```

## Socket类
```js
// 创建Socket对象
const socket = io();
```

### 属性
```js
/**
 * 获取socket session id 
 * `connect`事件后才有值，`reconnect`事件后会更新值
 * @return {String}
 */
console.log(socket.id); // undefined
socket.on('connect', () => {
  console.log(socket.id); // 'G5p5...'
});

/**
 * 获取connected连接状态
 * @return {Boolean}
 */
socket.on('connect', () => {
  console.log(socket.connected); // true
});

/**
 * 获取disconnected断连状态
 * @return {Boolean}
 */
socket.on('connect', () => {
  console.log(socket.disconnected); // false
});

/**
 * 获取命名空间
 * @return {String}
 */
socket.on('connect', () => {
  console.log(socket.nsp); // '/'
});

/**
 * 获取Manager实例
 * @return {Manager}
 */
socket.on('connect', () => {
  console.log(socket.io); // Manager
});

/**
 * 获取Socket实例json
 * @return {Socket}
 */
socket.on('connect', () => {
  console.log(socket.json); // Socket
});
```

### 方法
```js
/**
 * 手动建立连接 open
 * @return {Socket}
 */
socket.open();

const socket = io({
  autoConnect: false
});
socket.open();
// or
socket.on('disconnect', () => {
  socket.open();
});

/**
 * 手动建立连接 connect
 * @see open()
 */
socket.connect();

/**
 * 发送一个`message`事件消息
 * @param {Any} [...args]
 * @param {Function} [ack]
 * @return {Socket}
 * @see emit()
 */
socket.send([...args][,ack]);

/**
 * 发送事件消息
 * @param {String} eventName 事件名称
 * @param {Any} [...args] 传输数据，支持所有可序列化的数据结构，包括`Buffer`
 * @param {Function} [ack] 消息确认回调
 * @return {Socket}
 */
socket.emit(eventName,[...args][,ack]);

socket.emit('with-binary', 1, '2', { 3: '4', 5: new Buffer(6) });
socket.emit('hello', 'world', (data) => {
  console.log(data); // 'world'
});
// server:
 io.on('connection', (socket) => {
   socket.on('hello', (name, fn) => {
     fn(name);
   });
 });

/**
 * 注册监听事件 
 * 
 * socket继承[Emitter](https://github.com/component/emitter)类，
 * 因此也具有`hasListeners`,`once`,`off`等方法
 * 
 * @param {String} eventName 事件名称
 * @param {Function} callback 事件触发时的回调
 * @return {Socket}
 */
socket.on(eventName, callback);

socket.on('news', (data) => {
  console.log(data);
});
// with multiple arguments
socket.on('news', (arg1, arg2, arg3, arg4) => {
  // ...
});
// with callback
socket.on('news', (cb) => {
  cb(0);
});

/**
 * 指定发出的数据是否压缩处理，默认为true
 * 
 * @param {Boolean} value 
 * @return {Socket}
 */
socket.compress(value)

socket.compress(false).emit('an event', { some: 'data' });

/**
 * 指定发出的数据是否包含二进制，指定时可提高性能
 * 
 * @param {Boolean} value
 * @return {Socket}
 */
socket.binary(value)

socket.binary(false).emit('an event', { some: 'data' });

/**
 * 关闭连接
 * 
 * @return {Socket}
 */
socket.close()

/**
 * 关闭连接
 * 
 * @see close()
 */
socket.disconnect()
```

### 事件
```js
// 连接成功事件
socket.on('connect', () => {});

// 注：应该在`connect`回调外部注册自定义事件
// 这样就无需在重连接`reconnection`事件后重新注册
socket.on('myevent', () => {
  // ...
});

// 连接错误事件
socket.on('connect_error', (error) => {});

// 连接超时事件
socket.on('connect_timeout', (timeout) => {});

// 错误事件
socket.on('error', (error) => {});

// 断连事件
// reason: `io server disconnect` or `io client disconnect`
socket.on('disconnect', (reason) => {});
socket.on('disconnect', (reason) => {
  if (reason === 'io server disconnect') {
    // 断连来自服务端的原因，客户端需要手动再次连接
    socket.connect();
  }
  // else socket会自动重连
});

// 重连事件
socket.on('reconnect', (attempt) => {}); // 重连成功
socket.on('reconnect_attempt', (attempt) => {}); // 重连尝试
socket.on('reconnecting', (attempt) => {}); // 重连尝试
socket.on('reconnect_error', (error) => {}); // 重连错误
socket.on('reconnect_failed', (attempt) => {}); // 重连失败

// 乒乓事件
socket.on('ping', () => {}); // client ping server
socket.on('pong', (ms) => {}); // server response client, ms为自ping以来花费的毫秒数
```