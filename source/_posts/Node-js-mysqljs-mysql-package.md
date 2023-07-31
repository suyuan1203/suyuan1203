---
title: Node.js mysqljs/mysql package
date: 2019-06-20 15:19:48
tags:
---
https://github.com/mysqljs/mysql是一个用Node.js写的mysql驱动，代码简洁，明白。

一个mysql驱动要完成的功能是，建立到mysql服务器的tcp连接，然后按照mysql的protocol把数据发送给服务器，服务器收到数据以后，解析出要执行的命令和sql，执行sql，把结果按照protocol写回到客户端。

mysqljs/mysql的对象主要有
* Connection 管理到服务器的连接
* Protocol 处理发送到服务器的数据和服务器返回的数据
* Sequence/Query 代表一个query
* ComQueryPacket和其他Packet  代表一个发送给服务器的消息
* PacketWriter 把Packet转换为buffer交给Protocol
* Parser 解析Protocol从服务器获得的buffer


## 建立连接
建立连接的代码
```
var mysql = require('mysql');
var connection = mysql.createConnection({});
connection.connect();
```

lib/Connection.js
```
function Connection(options) {
    Events.EventEmitter.call(this); // Connection是一个EventEmmiter
    this._socket = options.socket;
    this._protocol = new Protocol({config: this.config, connection: this});
}
```

Connection对象的_socket是一个到mysql的tcp链接，代码在
lib/Connection.js
```
Connection.prototype.connect = function connect(options, callback) {
    this._socket = (this.config.socketPath)
      ? Net.createConnection(this.config.socketPath)
      : Net.createConnection(this.config.port, this.config.host);
}
```

上面的this._protocol是一个Protocol对象，Protocol对象是一个stream

lib/protocol/Protocol.js
```
Util.inherits(Protocol, Stream);
function Protocol(options) {
    Stream.call(this);
}
```

## 执行query
connection.query('select * from users')实际创建了一个query，然后调用Protocol._enqueue
```
Connection.prototype.query = function query(sql, values, cb) {
    var query = Connection.createQuery(sql, values, cb);
    return this._protocol._enqueue(query);
}
```

Connection.createQuery返回了一个Query对象，Query对象继承自Sequence对象
lib/protocol/sequences/Query.js
```
Util.inherits(Query, Sequence);
function Query(options, callback) {
    Sequence.call(this, options, callback);
}
```

Sequence是一个EventEmmiter
lib/protocol/sequences/Sequence.js
```
Util.inherits(Sequence, EventEmitter);
function Sequence(options, callback) {
}
```

Connection.createQuery创建的query被添加到Protocol对象的一个内部queue中，然后调用query.start

lib/protocol/Protocol.js
```
Protocol.prototype._enqueue = function(sequence) {
    if (this._queue.length === 1) {
        this._parser.resetPacketNumber();
        this._startSequence(sequence);
      }
}
```

Protocol有一个enqueue事件，在有新的query被enqueue时会触发

lib/protocol/Protocol.js
```
Protocol.prototype._enqueue = function(sequence) {
    this.emit('enqueue', sequence);
}   
```

在建立的连接的时候，Connection对象会给这个事件添加监听函数

lib/Connection.js
```
Connection.prototype.connect = function connect(options, callback) {
    this._protocol.on('enqueue', this._handleProtocolEnqueue.bind(this));
}
```
this._handleProtocolEnqueue函数仅仅是把这个事件在Connection对象上再emit一次，这样就能允许Connection对象的调用者监听Connection的enqueue事件
```
Connection.prototype._handleProtocolEnqueue = function _handleProtocolEnqueue(sequence) {
  this.emit('enqueue', sequence);
};
```

在Protocol._enqueue方法中，监听了Query对象(Sequence对象)的很多事件

lib/protocol/Protocol.js
```
Protocol.prototype._enqueue = function(sequence) {
    sequence
    .on('error', function(err) {
      self._delegateError(err, sequence);
    })
    .on('packet', function(packet) {
      sequence._timer.active();
      self._emitPacket(packet);
    })
    .on('timeout', function() {
      var err = new Error(sequence.constructor.name + ' inactivity timeout');

      err.code    = 'PROTOCOL_SEQUENCE_TIMEOUT';
      err.fatal   = true;
      err.timeout = sequence._timeout;

      self._delegateError(err, sequence);
    });
    sequence.on('end', function () {
      self._dequeue(sequence);
    });
}
```

Protocol在_enqueue方法中调用了Protocol.prototype._startSequence，Protocol.prototype._startSequence的定义

lib/protocol/Protocol.js
```
Protocol.prototype._startSequence = function(sequence) {
  if (sequence._timeout > 0 && isFinite(sequence._timeout)) {
    sequence._timer.start(sequence._timeout);
  }

  if (sequence.constructor === Sequences.ChangeUser) {
    sequence.start(this._handshakeInitializationPacket);
  } else {
    sequence.start();
  }
};
```

Sequeuce.start一般由子类实现，所以可以看Query.start

## query按照protocol转换为buffer
lib/protocol/sequences/Query.js
```
Query.prototype.start = function() {
  this.emit('packet', new Packets.ComQueryPacket(this.sql));
};
```
这个方法创建了一个ComQueryPacket，然后通过packet事件通知其他组件。Query是一个EventEmitter，所以可以通过事件和Query对象通信。

lib/protocol/packets/ComQueryPacket.js
```
module.exports = ComQueryPacket;
function ComQueryPacket(sql) {
  this.command = 0x03;
  this.sql     = sql;
}

ComQueryPacket.prototype.write = function(writer) {
  writer.writeUnsignedNumber(1, this.command);
  writer.writeString(this.sql);
};

ComQueryPacket.prototype.parse = function(parser) {
  this.command = parser.parseUnsignedNumber(1);
  this.sql     = parser.parsePacketTerminatedString();
};
```
ComQueryPacket类仅仅有两个成员，command和sql

根据mysql文档
> A COM_QUERY is used to send the server a text-based query that is executed immediately.

Query.prototype.start方法emit的packet事件，会触发在Protocol.prototype._enqueue中设置的handler
```
Protocol.prototype._enqueue = function(sequence) {
    sequence
    .on('error', function(err) {
      self._delegateError(err, sequence);
    })
    .on('packet', function(packet) {
      sequence._timer.active();
      self._emitPacket(packet);
    })
    .on('timeout', function() {
      var err = new Error(sequence.constructor.name + ' inactivity timeout');

      err.code    = 'PROTOCOL_SEQUENCE_TIMEOUT';
      err.fatal   = true;
      err.timeout = sequence._timeout;

      self._delegateError(err, sequence);
    });
    sequence.on('end', function () {
      self._dequeue(sequence);
    });
}
```
我们看到这里handler调用了self._emitPacket(packet);

```
Protocol.prototype._emitPacket = function(packet) {
  var packetWriter = new PacketWriter();
  packet.write(packetWriter);
  this.emit('data', packetWriter.toBuffer(this._parser));

  if (this._config.debug) {
    this._debugPacket(false, packet);
  }
};
```

_emitPacket调用了packet.write
```
ComQueryPacket.prototype.write = function(writer) {
  writer.writeUnsignedNumber(1, this.command);
  writer.writeString(this.sql);
};
```
packet.write调用了PacketWriter的writeUnsignedNumber和writeString方法

PacketWriter定义
lib/protocol/PacketWriter.js
```
module.exports = PacketWriter;
function PacketWriter() {
  this._buffer = null;
  this._offset = 0;
}
```

writeUnsignedNumber方法把一个number写到内部的一个buffer，this._buffer
```
PacketWriter.prototype.writeUnsignedNumber = function(bytes, value) {
  this._allocate(bytes);

  for (var i = 0; i < bytes; i++) {
    this._buffer[this._offset++] = (value >> (i * 8)) & 0xff;
  }
};
```
writeString方法把一个string写到this._buffer

接下来Protocol emit了data事件
```
this.emit('data', packetWriter.toBuffer(this._parser));
```
这里调用了packetWriter.toBuffer

```
PacketWriter.prototype.toBuffer = function toBuffer(parser) {
  if (!this._buffer) {
    this._buffer = Buffer.alloc(0);
    this._offset = 0;
  }

  var buffer  = this._buffer;
  var length  = this._offset;
  var packets = Math.floor(length / MAX_PACKET_LENGTH) + 1;

  this._buffer = Buffer.allocUnsafe(length + packets * 4);
  this._offset = 0;

  for (var packet = 0; packet < packets; packet++) {
    var isLast = (packet + 1 === packets);
    var packetLength = (isLast)
      ? length % MAX_PACKET_LENGTH
      : MAX_PACKET_LENGTH;

    var packetNumber = parser.incrementPacketNumber();

    this.writeUnsignedNumber(3, packetLength);
    this.writeUnsignedNumber(1, packetNumber);

    var start = packet * MAX_PACKET_LENGTH;
    var end   = start + packetLength;

    this.writeBuffer(buffer.slice(start, end));
  }

  return this._buffer;
};
```
这个方法调整this._buffer的内容，具体步骤是，如果this._buffer的长度超过了MAX_PACKET_LENGTH，那么把this._buffer分成几个packet，每个packet的最大长度是MAX_PACKET_LENGTH。同时在每个packet前添加4个字节的内容，前三个字节代表packet的长度，第四个字节代表packet的序号。然后返回调整后的this._buffer


Protocol emit的data事件会触发在Connection中设置的handler
```
this._protocol.on('data', function(data) {
  connection._socket.write(data);
});
```
这里把packet写到connection._socket中，服务器就会收到packet。

当mysql服务器收到客户端的数据的时候，先读取4个字节，获得packet的长度和序号，然后读取第一个packet，接下来又读取4个字节，获得下一个packet的长度和序号，依次类推，最终读取完客户端发送的数据。


MAX_PACKET_LENGTH的定义是
```
var MAX_PACKET_LENGTH            = Math.pow(2, 24) - 1; // 大概16M
```
明显一般的packet都不会超过16M，所以服务器收到的大部分packet都类似于
```
三个字节长度 - 1个字节序号 - packet内容
```
服务器根据长度读取到packet的内容，然后从内容中解析出，要执行的command的类型，比如0x03，和对应的sql。

## 服务器返回数据
服务器返回数据的时候，会往Connection._socket中写数据，因此会触发Connection的data事件

在Connection的connect方法中，设置了Connection.socket的data事件的handler
```
this._socket.on('data', wrapToDomain(connection, function (data) {
  connection._protocol.write(data);
}));
```

服务器返回的数据交给Parser，Parser解析出服务器返回的查询结果
```
Protocol.prototype.write = function(buffer) {
  this._parser.write(buffer);
  return true;
};
```

