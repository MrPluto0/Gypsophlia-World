> 由于已经接触过不少`B/S结构`项目，于是希望以`C/S`结构来构建一个项目，而`socket`通信就多用于此。本文章中会主要介绍客户端与服务端建立`WebSocket`连接的方式，消息的封装，消息的加密，及一些常见问题。

# 概念介绍

## WebSocket

传统的`Http`请求是无状态的，每次请求只能由一方发起，另一方接收消息后响应即结束。对于需要频繁交互的，如聊天室，对于需要实时更新的项目，以往的实现采用`轮询`的方式执行，每隔一段极短时间向服务器发起请求，从而更新新的内容，每次建立握手连接与关闭是非常耗费时间与资源的。

`WebSocket`协议是在`Http 1.1`协议基础上产生，解决了`Http`的上述问题，能够进行全双工的`TCP`通信。只需要建立一次握手连接，之后便可以维持一个通信信道，借助监听消息事件来发送消息。

需要注意的是，`WebSocket`的消息传输以`Frame`帧的形式分段发送，获取的结果一般为字符串，因此若想传输对象，数组等结构，需要进行序列化与反序列化。

## 目标

个人实践将构建一个客户端与服务端的机器人通信项目。

服务端解析定制的`DSL`脚本，进行分词/解析`AST`；根据客户端的不同响应来进行不同的通信。主要借助后端`ws`模块来通信。

> 领域特定语言（Domain Specific Language，DSL）可以提供一种相对简单的文法，用于特定领域的业务流程定制。

客户端即为人`机`交互，用户与机器人进行交流。主要借助`Html5`原生的`Websocket`来进行通信，与`ws`模块的使用有所不同。

![socket.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/b6cf6e6ed92e440c96bd604e7b0fcc68~tplv-k3u1fbpfcp-watermark.image?)

# 用法介绍

`Html5 WebSocket`和`ws`模块的用法有所不同，前者仅可在网页中使用，后者仅可在服务端使用；且对于获取到的数据也可能有所差异。

二者对于状态的划分与使用是相同的，通过`ws.readyState`获取状态码：

```js
ws.CONNECTING === 0;
ws.OPEN === 1;
ws.CLOSING === 2;
ws.CLOSED === 3;
```

## Html5 WebSocket

### 建立连接

字符串为`ws`协议的地址：`ws://...`

```js
let ws = new WebSocket('ws://localhost:8080');
```

### 接收消息

```js
ws.onmessage = function(event){
    let msg = event.data;
    // ...
}
```

这里需要强调对于数据的获取，需要间接通过`event`对象，且`event.data`可能有两种数据格式，一种为`String`，另一种为`Blob`。对于`String`的处理不必赘述，需要额外说明对于`Blob`的处理：借助`FileReader`来回调出其信息。

```js
const reader = new FileReader();

// reader.readAsArrayBuffer(event.data);
reader.readAsText(event.data, 'utf8');
reader.onload = () => {
  const message = reader.result;
  this.receiveMsg(message);
};
```

### 发送消息

```js
// typeof data === 'string'
ws.send(data);
```

### 关闭

```js
ws.onclose = funtion(){
    // ...
}
```

## ws模块

### 建立服务

这里的`WebSocket`属于`ws`模块中的对象。

```js
// 创建websocket服务端
let wss = new WebSocket.Server({ port:8080 });

// 创建一个新的连接
let ws = new WebSocket("ws://localhost:8080");
```

### 监听连接

每次有新的客户端建立连接后，服务端便可收到消息。

```js
wss.on('connection', async (ws, req) => {
    const { host } = req.headers;
    
    // close event
    ws.on("close", (code, reason) => {
        console.log("ID:%d Server closed: %d %s", ClientId, code, reason);
    });
    
    // message event
    ws.on("message", (message) => {
        stdout.info("ID:%d Server receive: %s",message.toString());
    }
})
```

回调函数中的`ws`便是对于客户端通信来说是一对一的，`req`附带响应`TCP`连接的信息，如上面是`host`是连接方的主机名。

### 心跳

对于每个`ws`连接，可以通过以下方式来进行心跳测试，每隔`waitTime`的时间内，监测是否收到回复。长时间未收到回复即会断开连接，以节省资源。

```js
setTimeout(() => {
  if (!reply) {
    ws.close();
  }
}, waitTime);
```

关于心跳的应用，在我的项目中有另外一处应用，即`限时断线`，作为机器人来说，是长时间保持在空闲状态的，否则会消耗资源，但是对于消息的监听是异步的，限时在某种意义上代表需要计算时间，需要阻塞，因此这里推荐一种`时间片`的做法：

时间片设置为`500ms`，计算循环轮数，每轮检测是否收到消息即可。这里的阻塞是必须的，否则在异步的情况下无法计算时间。

```js
let INTERVAL = 500;
let loops = Math.ceil((seconds * 1000) / INTERVAL);

while (loops > 0) {
  if (reply) {
    console.log("Receive");
    return;
  }
  await sleep(INTERVAL);
  loops--;
}

if (!reply) {
  console.log("Closed.");
  return;
}
```

### ws监听事件

> 需要说明，`ws.onmessage`等只能够绑定一个回调函数，而`ws.on("message",()=>{})`能够绑定多个回调函数，不会进行覆盖。这点与网页中点击事件的绑定类似，即`onclick`和`addListener('click',()=>{})`。

接收消息，有以下两种方法，这里的回调参数`message`与前面的又不太一致，存在`String`和`WebSocket.RawData`类型，因此可统一使用`toString`来转换。

```js
ws.onmessage = (message) => {}
ws.on("message", (message) => {
    console.log("Receive %s", message.toString());
})
```

监听关闭：

```js
ws.onclose = () => {}
ws.on("close",() => {})
```

# 通信封装

用到`WebSocket`的过程中势必会碰到一个问题，那是我们的需要不仅仅是收发消息，还需要对消息划分类型，传输类似于`Http`消息的数据，因此这里就可以用到序列化的知识。

## 封装

在我的项目中，构造了如下结构：

```json
{
    type: "init",  // "message" or "close"
    data: {
        name: "xxx",
        avatar: "xxx",
        account: 100
    }
}
```

`type`可以定义不同的消息类型，`data`中即为返回的数据，通过`JSON.stringify()`来序列化，接收消息后通过`JSON.parse()`来反序列化。

## 加密

在一些消息传递过程中，同时需要做到对信息的保密，如初始化时传递`密码`等，这时就需要用到`RSA`非对称加密（通过`openssl`生成公钥和私钥，详见相关中的另一篇文章），需要用到`jsencrypt`模块。

```js
// const { JSEncrypt } = require('nodejs-encrypt');
const { JSEncrypt } = require('jsencrypt');

// 公钥加密
publicEncrypt(str) {
    const encrypt = new JSEncrypt();
    encrypt.setPublicKey(pubKey);
    return encrypt.encryptLong(str);
},

// 私钥解密
privateDecrypt(str) {
    const encrypt = new JSEncrypt();
    encrypt.setPrivateKey(privKey);
    return encrypt.decryptLong(str);
},
```

> 在使用上述模块加密过程中可能会碰到加密失败问题，这是由于字符串过长，可以借助`encryptlong`模块来进行。另外，个人的另外一种思路是采用`md5`加密缩短字符串后再进行`RSA`加密。

# 最后

这篇文章的内容以`WebSocket`为主，讲述了它的一些基本使用和做项目过程中所碰到的一些问题。

这里推荐一下个人的项目，分为`Electron`客户端和`Nodejs`服务端，包含了`DSL`解析模块（`Tokenize`,`AST`），日志模块，测试模块等：[MrPluto0/dslBot (github.com)](https://github.com/MrPluto0/dslBot)

# 相关

[MrPluto0/dslBot解释器(人机交互)](https://github.com/MrPluto0/dslBot)

[实践：使用jsencrypt配合axios实现非对称加密传输数据 - 掘金 (juejin.cn)](https://juejin.cn/post/6971447144423096351)

[阮一峰的个人日志:WebSocket](https://www.ruanyifeng.com/blog/2017/05/websocket.html)

[ws模块英文文档(github.com)](https://github.com/websockets/ws/blob/master/doc/ws.md)