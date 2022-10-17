---
title: 使用nodejs创建websocket
date: 2022-10-13 14:12:14
tags: [源码解析]
category: node
description: 本文将通过对nodejs-websocket的源码进行解析，从而深入的搞懂websocket是如何工作的
keywords: [node, nodejs, websocket, nodejs-websocket, 源码解析] 
---

## 前言
本文主题是阅读`nodejs-websocket`库的源码，进而搞懂`websocket`的实现方式，并通过自己手写搭建一个简单的`websocket`服务

`Web Scoket`是一个基于`TCP`协议的全双工的通信方式，当建立链接后，其服务端可以直接发起对客户端的通信

## 工作方式
要建立一个`Web Socket`链接，首先需要通过`HTTP`进行握手，在握手成功后，切换协议方式（`HTTP`切换成 `Web Socket`协议），链接建立成功，就可以开始通信了。

在握手时，客户端需要在请求报文中注明自己需要切换协议为`Web Socket`协议（在`Web Socket`中，我们不需要关注请求报文，因为当我们通过 `new WebSocket(ws://example.com/test)`，创建实例时，浏览器会在内部为我们处理请求的报文）

> 同源策略不适用于`Web Socket`

```
  GET /test HTTP/1.1
  Connection: Upgrade
  Upgrade: websocket
  Sec-WebSocket-Version: 13
  Sec-WebSocket-Key: udGGqBU9BqSiThHTKc0B6g==
```

当服务端收到新的`Web Scoket`链接时，判断当前发起的是`Web Socket`链接后，然后需要返回一个响应报文，来告诉客户端切换到`Web Socket`协议
```
HTTP/1.1 101 Switching Protocols
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Accept: 4UCGeYLdUGEUaei+36wTyP60Mcg=
```

> - `HTTP 101` 状态码英文名称是`Switching Protocols`，表示切换协议。服务器根据客户端的请求切换协议。只能切换到更高级的协议
> - `Sec-WebSocket-Accept` 用以告知服务器愿发起一个 `websocket` 连接。`Sec-WebSocket-Accept`的计算方法：`base64(hsa1(sec-websocket-key + 258EAFA5-E914-47DA-95CA-C5AB0DC85B11))`。参考：https://www.zhihu.com/question/67784701


## 源码解读
> 以下只列出了关键性代码

`Server`类，用于创建一个`server`示例，同时在构造函数中做一些前置处理

```javascript
function Server(secure, options, callback) {
  // 内部server，收到一个"connection"事件后调用
  // 这个事件是nodejs 服务的事件，与下面触发的"connection"事件不是一个
	var onConnection = function (socket) {
    // 创建一个连接后的实例，用于收发数据，错误处理等
    // 看下面的介绍
		var conn = new Connection(socket, that, function () {
			that.connections.push(conn)
			conn.removeListener('error', nop)
			that.emit('connection', conn)
		})
    // 连接关闭后移除当前链接
		conn.on('close', function () {
			var pos = that.connections.indexOf(conn)
			if (pos !== -1) {
				that.connections.splice(pos, 1)
			}
		})

		// Ignore errors before the connection is established
		conn.on('error', nop)
	}
  // 创建一个内部的server实例
  if (secure) {
    // 创建wss服务
		this.socket = tls.createServer(options, onConnection)
	} else {
    // 创建ws服务
		this.socket = net.createServer(options, onConnection)
	}
  // 保存已连接的web socket
	this.connections = []

	// 继承时this绑定
	events.EventEmitter.call(this)
	if (callback) {
    // 当web socket 链接后调用
    // connection事件是自己实现的
		this.on('connection', callback)
	}
}
// 继承events.EventEmitter
util.inherits(Server, events.EventEmitter)
```
`server.listen()` 用于开启`Web Socket`服务
```javascript
Server.prototype.listen = function (port, host, callback) {
	if (callback) {
		this.on('listening', callback)
	}
  // 开启服务，此时可以通过连接访问了
	this.socket.listen(port, host, function () {
		that.emit('listening')
	})

	return this
}

```

`Connection`类，是整个模块的核心，用于处理`Web Socket`的“握手”，以及“握手”成功后对数据的处理和异常的处理
> 只看被作为服务端来使用时的处理

`Connection`构造函数，数据初始化，绑定`net.Socket`的`readable`事件，用于对进来的请求做处理

```javascript
function Connection(socket, parentOrOptions, callback) {
	var that = this,
		connectEvent
  // 初始化
  // Server-side connection
  // Server 实例
  this.server = parentOrOptions
  // websocket 的url
  this.path = null
  // websocket 的host
  this.host = null
  this.extraHeaders = null
  this.protocols = []

	this.protocol = undefined
  // net.Socket 的实例
  // https://nodejs.org/dist/latest-v18.x/docs/api/net.html#event-connection
	this.socket = socket
  // 连接状态
	this.readyState = this.CONNECTING
  // 已接收数据
	this.buffer = Buffer.alloc(0)
	this.frameBuffer = null // string for text frames and InStream for binary frames
	this.outStream = null // current allocated OutStream object for sending binary frames
	this.key = null // the Sec-WebSocket-Key header
	this.headers = {} // read only map of header names and values. Header names are lower-cased

	// 入口，在这里读数据，包括“握手”和后面接收数据
	socket.on('readable', function () {
		that.doRead()
	})
  // 错误处理
	socket.on('error', function (err) {
		that.emit('error', err)
	})

	// Close listeners
  // 连接关闭，状态重置
	var onclose = function () {
		if (that.readyState === that.CONNECTING || that.readyState === that.OPEN) {
			that.emit('close', 1006, '')
		}
		that.readyState = this.CLOSED
		if (that.frameBuffer instanceof InStream) {
			that.frameBuffer.end()
			that.frameBuffer = null
		}
		if (that.outStream instanceof OutStream) {
			that.outStream.end()
			that.outStream = null
		}
	}
	socket.once('close', onclose)
	socket.once('finish', onclose)

	// 继承时this绑定
	events.EventEmitter.call(this)
	if (callback) {
		this.once('connect', callback)
	}
}
// 继承events.EventEmitter
util.inherits(Connection, events.EventEmitter)

```

`Connection.prototype.doRead()`是所有请求的入口函数，用于处理“握手”，以及数据读取

```javascript
Connection.prototype.doRead = function () {
	var buffer, temp

	// Fetches the data
  // 从内部缓存区读取数据
	buffer = this.socket.read()
	if (!buffer) {
		// Waits for more data
		return
	}

	// Save to the internal buffer
  // 保存当前进来的数据
	this.buffer = Buffer.concat([this.buffer, buffer], this.buffer.length + buffer.length)
  // 如果连接状态为正在连接
	if (this.readyState === this.CONNECTING) {
    // 握手处理
		if (!this.readHandshake()) {
			// May have failed or we're waiting for more data
			return
		}
	}

	if (this.readyState !== this.CLOSED) {
		// 读取数据
		while ((temp = this.extractFrame()) === true) {}
		if (temp === false) {
			// Protocol error
			this.close(1002)
			// 读取数据超过最大长度限制处理
		} else if (this.buffer.length > Connection.maxBufferLength) {
			// Frame too big
			this.close(1009)
		}
	}
}
```

`Connection.prototype.readHandshake()`用于对“握手”的报文进行提取解析处理

```javascript
Connection.prototype.readHandshake = function () {
	var found = false,

// Search for '\r\n\r\n'
// 这里不是很理解，可能是搜索到这几个字符，则说明当前请求接收完成
	for (i = 0; i < this.buffer.length - 3; i++) {
		if (this.buffer[i] === 13 && this.buffer[i + 2] === 13 &&
			this.buffer[i + 1] === 10 && this.buffer[i + 3] === 10) {
			found = true
			break
		}
	}
	if (!found) {
		// Wait for more data
		return false
	}
	// 拆分请求报文
	data = this.buffer.slice(0, i + 4).toString().split('\r\n')
	// 切换Web Socket协议处理
	if (this.answerHandshake(data)) {
		this.buffer = this.buffer.slice(i + 4)
		// 更改链接状态为open
		this.readyState = this.OPEN
		this.emit('connect')
		return true
	} else {
		// 对于Web Socket的错误处理都是返回这个响应信息
		this.socket.end('HTTP/1.1 400 Bad Request\r\n\r\n')
		return false
	}
}
```

`Connection.prototype.answerHandshake()`将`HTTP`协议切换成`Web Socket`协议。

```javascript
Connection.prototype.answerHandshake = function (lines) {
	var path, key, sha1, headers

	// 请求报文，必定包含上面（`工作方式所列出的内容`）提到的内容
	if (lines.length < 6) {
		return false
	}
	// 解析请求url
	path = lines[0].match(/^GET (.+) HTTP\/\d\.\d$/i)
	if (!path) {
		return false
	}
	this.path = path[1]

	// 解析为正确的headers格式，代码在下面
	this.readHeaders(lines)

	// 验证必要的header
	if (!('host' in this.headers) ||
		!('sec-websocket-key' in this.headers) ||
		!('upgrade' in this.headers) ||
		!('connection' in this.headers)) {
		return false
	}
	// 'Upgrade' 的值必须是'websocket'
	// 'Connection' 的值必须存在'Upgrade'
	if (this.headers.upgrade.toLowerCase() !== 'websocket' ||
		this.headers.connection.toLowerCase().split(/\s*,\s*/).indexOf('upgrade') === -1) {
		return false
	}
	// 决定当前链接的Web Scoket协议的版本
	// 这里对不符合的版本，在响应报文中，没有做提示，而是直接当成错误来处理
	if (this.headers['sec-websocket-version'] !== '13') {
		return false
	}

	this.key = this.headers['sec-websocket-key']

	// Agree on a protocol
	if ('sec-websocket-protocol' in this.headers) {
		// Parse
		this.protocols = this.headers['sec-websocket-protocol'].split(',').map(function (each) {
			return each.trim()
		})

		// Select protocol
		if (this.server._selectProtocol) {
			this.protocol = this.server._selectProtocol(this, this.protocols)
		}
	}

	// Build and send the response
	// 生成一个`Sec-WebSocket-Accept` key
	sha1 = crypto.createHash('sha1')
	sha1.end(this.key + '258EAFA5-E914-47DA-95CA-C5AB0DC85B11')
	key = sha1.read().toString('base64')

	// 响应头
	headers = {
		Upgrade: 'websocket',
		Connection: 'Upgrade',
		'Sec-WebSocket-Accept': key
	}
	if (this.protocol) {
		headers['Sec-WebSocket-Protocol'] = this.protocol
	}

	// 切换为Web Socket协议的应答
	this.socket.write(this.buildRequest('HTTP/1.1 101 Switching Protocols', headers))
	return true
}

// 将请求头解析成对象形式的headers
Connection.prototype.readHeaders = function (lines) {
	var i, match

	// Extract all headers
	// Ignore bad-formed lines and ignore the first line (HTTP header)
	for (i = 1; i < lines.length; i++) {
		if ((match = lines[i].match(/^([a-z-]+): (.+)$/i))) {
			this.headers[match[1].toLowerCase()] = match[2]
		}
	}
}
```
以上就完成了连接的握手处理

## 数据解析

`WebSocket` 以 `frame` 为单位传输数据, `frame` 是客户端和服务端数据传输的最小单元, 当一条消息过长时, 通信方可以将该消息拆分成多个 `frame` 发送, 接收方收到以后重新拼接、解码从而还原出完整的消息, 在 `WebSocket` 中, `frame` 有多种类型, `frame` 的类型由 `frame` 头部的 `Opcode` 字段指示

`Connection.prototype.extractFrame()` 从缓冲区中提取帧(`frame`)的内容

```javascript
Connection.prototype.extractFrame = function () {
	var fin, opcode, B, HB, mask, len, payload, start, i, hasMask

	if (this.buffer.length < 2) {
		return
	}

	// Is this the last frame in a sequence?
	B = this.buffer[0]
	HB = B >> 4
	// 判断RSV1, RSV2, RSV3位组成的自定义协议
	if (HB % 8) {
		// RSV1, RSV2 and RSV3 must be clear
		return false
	}
	// 用于指示当前的 frame 是消息的最后一个分段，1为最后一个frame
	fin = HB === 8
	// 操作码
	// *  %x0 表示连续消息片断
	// *  %x1 表示文本消息片断
	// *  %x2 表未二进制消息片断
	// *  %x3-7 为将来的非控制消息片断保留的操作码
	// *  %x8 表示连接关闭
	// *  %x9 表示心跳检查的ping
	// *  %xA 表示心跳检查的pong
	// *  %xB-F 为将来的控制消息片断的保留操作码
	opcode = B % 16
	// 判断是否是有效操作码，0~F,除了这里列出的，其他都为保留的操作码
	if (opcode !== 0 && opcode !== 1 && opcode !== 2 &&
		opcode !== 8 && opcode !== 9 && opcode !== 10) {
		// Invalid opcode
		return false
	}
	if (opcode >= 8 && !fin) {
		// Control frames must not be fragmented
		return false
	}

	B = this.buffer[1]
	// 取出Mask位，判断传输的数据是否有加掩码，设置为1，掩码键必须放在Masking-key区域
	// 客户端发送给服务端的所有消息，此位的值都是1；
	hasMask = B >> 7
	if ((this.server && !hasMask) || (!this.server && hasMask)) {
		// Frames sent by clients must be masked
		return false
	}

	// Payload length处理
	// 获取 Payload length：传输数据的长度
	// 如果这个值以字节表示是0-125这个范围，那这个值就表示传输数据的长度；
	// 如果这个值是126，则随后的两个字节表示的是一个16进制无符号数，用来表示传输数据的长度；
	// 如果这个值是127,则随后的是8个字节表示的一个64位无符合数，这个数用来表示传输数据的长度
	len = B % 128
	start = hasMask ? 6 : 2
	if (this.buffer.length < start + len) {
		// Not enough data in the buffer
		return
	}
	// 获取实际的 payload length
	if (len === 126) {
		len = this.buffer.readUInt16BE(2)
		start += 2
	} else if (len === 127) {
		// Warning: JS can only store up to 2^53 in its number format
		len = this.buffer.readUInt32BE(2) * Math.pow(2, 32) + this.buffer.readUInt32BE(6)
		start += 8
	}
	if (this.buffer.length < start + len) {
		return
	}

	// 提取有效数据
	payload = this.buffer.slice(start, start + len)
	// 如果有Mask位，使用给定掩码解码
	if (hasMask) {
		// Decode with the given mask
		mask = this.buffer.slice(start - 4, start)
		for (i = 0; i < payload.length; i++) {
			payload[i] ^= mask[i % 4]
		}
	}
	this.buffer = this.buffer.slice(start + len)

	// Proceeds to frame processing
	return this.processFrame(fin, opcode, payload)
}
```

`Connection.prototype.processFrame()` 处理接收到的给定帧(`frame`)

```javascript
Connection.prototype.processFrame = function (fin, opcode, payload) {
	// 链接关闭处理
	if (opcode === 8) {
		// Close frame
		if (this.readyState === this.CLOSING) {
			this.socket.end()
		} else if (this.readyState === this.OPEN) {
			this.processCloseFrame(payload)
		}
		return true
	} 
	// 心跳检查的ping 处理
	else if (opcode === 9) {
		// Ping frame
		if (this.readyState === this.OPEN) {
			this.socket.write(frame.createPongFrame(payload.toString(), !this.server))
		}
		return true
	} 
	// 心跳检查的pong 处理
	else if (opcode === 10) {
		// Pong frame
		this.emit('pong', payload.toString())
		return true
	}

	if (this.readyState !== this.OPEN) {
		// Ignores if the connection isn't opened anymore
		return true
	}
	// 连续消息帧处理
	if (opcode === 0 && this.frameBuffer === null) {
		// Unexpected continuation frame
		return false
	} else if (opcode !== 0 && this.frameBuffer !== null) {
		// Last sequence didn't finished correctly
		return false
	}

	if (!opcode) {
		// Get the current opcode for fragmented frames
		opcode = typeof this.frameBuffer === 'string' ? 1 : 2
	}
	// 文本消息帧处理
	if (opcode === 1) {
		// Save text frame
		payload = payload.toString()
		this.frameBuffer = this.frameBuffer ? this.frameBuffer + payload : payload

		if (fin) {
			// Emits 'text' event
			this.emit('text', this.frameBuffer)
			this.frameBuffer = null
		}
	} 
	// 二进制消息帧处理
	else {
		// Sends the buffer for InStream object
		if (!this.frameBuffer) {
			// Emits the 'binary' event
			this.frameBuffer = new InStream
			this.emit('binary', this.frameBuffer)
		}
		this.frameBuffer.addData(payload)

		if (fin) {
			// Emits 'end' event
			this.frameBuffer.end()
			this.frameBuffer = null
		}
	}

	return true
}
```

## 数据发送

发送数据，需要创建与接收时一样的数据结构，包括`fin, opcode, mask, payload-length, mask-key`

```javascript
Connection.prototype.sendText = function (str, callback) {
	if (this.readyState === this.OPEN) {
		return this.socket.write(frame.createTextFrame(str, !this.server), callback)
	}
}
// 创建一个文本frame
function createTextFrame(data, masked) {
	var payload, meta

	payload = Buffer.from(data)
	meta = generateMetaData(true, 1, masked === undefined ? false : masked, payload)

	return Buffer.concat([meta, payload], meta.length + payload.length)
}
// 创建frame的元数据部分
// 如果frame被屏蔽了，则payload会相应地改变
function generateMetaData(fin, opcode, masked, payload) {
	var len, meta, start, mask, i

	len = payload.length

	// 为元数据创建缓冲区
	meta = Buffer.alloc(2 + (len < 126 ? 0 : (len < 65536 ? 2 : 8)) + (masked ? 4 : 0))

	// 设置 fin 和 opcode
	meta[0] = (fin ? 128 : 0) + opcode

	// 设置 Mask 和 length
	meta[1] = masked ? 128 : 0
	start = 2
	if (len < 126) {
		meta[1] += len
	} else if (len < 65536) {
		meta[1] += 126
		meta.writeUInt16BE(len, 2)
		start += 2
	} else {
		// Warning: JS doesn't support integers greater than 2^53
		meta[1] += 127
		meta.writeUInt32BE(Math.floor(len / Math.pow(2, 32)), 2)
		meta.writeUInt32BE(len % Math.pow(2, 32), 6)
		start += 8
	}

	// 设置mask-key
	if (masked) {
		mask = Buffer.alloc(4)
		for (i = 0; i < 4; i++) {
			meta[start + i] = mask[i] = Math.floor(Math.random() * 256)
		}
		for (i = 0; i < payload.length; i++) {
			payload[i] ^= mask[i % 4]
		}
		start += 4
	}

	return meta
}
```


## 代码实现

接下来我们就可以根据上面的逻辑，实现一个简单版的websocket服务，包括握手和数据收发功能

```javascript
const net = require("net")
const crypto = require('crypto')

// 链接状态类别
const CONNECTING = 0;
const OPEN = 1;

// 默认状态
let readyState = CONNECTING

// websocket 入口
const myWebsocket = ({port, callback}) => {
  const server = createServer(callback)
  server.listen(port)
}

// 创建server
const createServer = (callback) => {
  let buffer
  const server = net.createServer(function (socket) {
		// 监听 "readable" 事件，从而获取请求的数据包
    socket.on("readable", function () {
      console.log('readable....');
      doRead(socket, callback)
    })
    const close = () => {
      readyState = CONNECTING
    }
    socket.on('close', close)
    socket.on('error', close)
  })

  return server
}

// 数据解析
const doRead = (socket, callback) => {
  let buffer, headers
	// 从缓冲区获取请求的数据
  buffer = socket.read()

	// 握手处理
  if (buffer && readyState === CONNECTING) {
    const lines = buffer.toString().split('\r\n')
    const path = lines[0].match(/^GET (.*) HTTP\/1.1$/)
    if (path) {
      // console.log(lines);
      headers = readHeaders(lines.slice(1))
      if (!checkHeaders(headers)) {
        return socket.end("HTTP/1.1 400 Switching Protocols \r\n\r\n");
      }

      readyState = OPEN;
      socket.write("HTTP/1.1 101 Switching Protocols\r\n" + returnHeaders(headers))
    } else {
      
      return;
    }
  } else {
		// websocket 连接后解析数据
    if (readyState === OPEN) {
      const payload = parseData(buffer)
			// 用户传递的回调
      callback&&callback(payload.toString(), sendText(socket))
    }
  }
}
// 获取headers
const readHeaders = (lines) => {
  return Object.fromEntries(lines.filter(line => !!line).map(line => {
    const v = line.match(/^([a-zA-Z-]+): (.+)$/)
    return [v[1].toLowerCase(), v[2]]
  }));
}
// 检查headers中是否符合websocket的请求头
const checkHeaders = (headers) => {
  if ('connection' in headers && 'upgrade' in headers && 'sec-websocket-version' in headers && 'sec-websocket-key' in headers) {

    return true
  }

  return false
}
// 切换协议时需要返回的请求头
const returnHeaders = (headers) => {
	// 需要使用sha1生成一个 Sec-WebSocket-Accept
  const sha1 = crypto.createHash('sha1')
  const key = sha1.end(headers['sec-websocket-key'] + '258EAFA5-E914-47DA-95CA-C5AB0DC85B11').read().toString('base64')

  return buildRequest({
    Upgrade: 'websocket',
		Connection: 'Upgrade',
		'Sec-WebSocket-Accept': key
  })
}
// 生成返回的请求头格式
const buildRequest = (headers) => {
  return Object.keys(headers).reduce((collect, key) => {
    collect += `${key}: ${headers[key]}\r\n`

    return collect;
  }, '') + "\r\n"
}
// 发送文本数据
const sendText = socket => data => {
	const payload = Buffer.from(data);
	const meta = generateMetaData(true, 1, false, payload)
	
	socket.write(Buffer.concat([meta, payload], meta.length + payload.length))
}

// 这部分逻辑是直接拿过来用的，还是别人的更香，注释就看上面
const parseData = function (buffer) {
	var fin, opcode, B, HB, mask, len, payload, start, i, hasMask

	if (buffer.length < 2) {
		return
	}

	// Is this the last frame in a sequence?
	B = buffer[0]
	HB = B >> 4
	if (HB % 8) {
		// RSV1, RSV2 and RSV3 must be clear
		return false
	}
	fin = HB === 8
	opcode = B % 16

	if (opcode !== 0 && opcode !== 1 && opcode !== 2 &&
		opcode !== 8 && opcode !== 9 && opcode !== 10) {
		// Invalid opcode
		return false
	}
	if (opcode >= 8 && !fin) {
		// Control frames must not be fragmented
		return false
	}

	B = buffer[1]
	hasMask = B >> 7
	if (!hasMask) {
		// Frames sent by clients must be masked
		return false
	}
	len = B % 128
	start = hasMask ? 6 : 2

	if (buffer.length < start + len) {
		// Not enough data in the buffer
		return
	}

	// Get the actual payload length
	if (len === 126) {
		len = buffer.readUInt16BE(2)
		start += 2
	} else if (len === 127) {
		// Warning: JS can only store up to 2^53 in its number format
		len = buffer.readUInt32BE(2) * Math.pow(2, 32) + buffer.readUInt32BE(6)
		start += 8
	}
	if (buffer.length < start + len) {
		return
	}

	// Extract the payload
	payload = buffer.slice(start, start + len)
	if (hasMask) {
		// Decode with the given mask
		mask = buffer.slice(start - 4, start)
		for (i = 0; i < payload.length; i++) {
			payload[i] ^= mask[i % 4]
		}
	}
	buffer = buffer.slice(start + len)

	// Proceeds to frame processing
	return payload
}
// 这部分逻辑是直接拿过来用的，还是别人的更香，注释就看上面
function generateMetaData(fin, opcode, masked, payload) {
	var len, meta, start, mask, i

	len = payload.length

	// Creates the buffer for meta-data
	meta = Buffer.alloc(2 + (len < 126 ? 0 : (len < 65536 ? 2 : 8)) + (masked ? 4 : 0))

	// Sets fin and opcode
	meta[0] = (fin ? 128 : 0) + opcode

	// Sets the mask and length
	meta[1] = masked ? 128 : 0
	start = 2
	if (len < 126) {
		meta[1] += len
	} else if (len < 65536) {
		meta[1] += 126
		meta.writeUInt16BE(len, 2)
		start += 2
	} else {
		// Warning: JS doesn't support integers greater than 2^53
		meta[1] += 127
		meta.writeUInt32BE(Math.floor(len / Math.pow(2, 32)), 2)
		meta.writeUInt32BE(len % Math.pow(2, 32), 6)
		start += 8
	}

	// Set the mask-key
	if (masked) {
		mask = Buffer.alloc(4)
		for (i = 0; i < 4; i++) {
			meta[start + i] = mask[i] = Math.floor(Math.random() * 256)
		}
		for (i = 0; i < payload.length; i++) {
			payload[i] ^= mask[i % 4]
		}
		start += 4
	}

	return meta
}

module.exports = myWebsocket
```
好的，现在我们已经封装完成了，就可以引用我们的代码来试用了
```javascript
myWebsocket({port: 3000, callback(data, sendText) {
  console.log('Received Data: ' + data);
  sendText(data.toUpperCase() + "!!!!");
}})
```

## 总结
经过对`nodejs-websocket`源码的阅读，我们从深层次了解了`websocket`是如何进行握手，并且如何从HTTP协议切换到`Web Socket`协议，以及在对数据进行解析时，需要通过哪些标志位(`FIN, Opcode, Mask`等)来告诉我们当前数据的状态是怎么样的，我们该如何处理。然后在代码的实现中，我们可以更好的去学习`Nodejs`是如何搭建一个服务，并加深我们对`nodejs`的开发能力

笔者能力有限，倘若在阅读时，有哪些不是很懂，欢迎在[这里](https://github.com/yxlazy/blog/issues)评论留言，大家一起讨论，共同进步

## 参考
https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Headers/Sec-WebSocket-Accept

https://developer.mozilla.org/zh-CN/docs/Web/API/WebSocket

https://juejin.cn/post/6844903463202062343

https://www.php.cn/http/http-http101.html

https://www.jianshu.com/p/3444ea70b6cb

https://zhuanlan.zhihu.com/p/407711596

https://github.com/sitegui/nodejs-websocket
