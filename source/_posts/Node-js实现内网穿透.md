---
title: Node.js实现内网穿透
date: 2019-05-25 15:15:17
tags:
---
如何在办公室外调用办公室内一台服务器上部署的服务？

如果办公室有固定的IP地址，而且可以映射到内网的机器，那么这个问题很简单。如果无法通过这种方式访问内网的机器，我们还可以通过内网穿透的办法来解决。


基本的思路是，需要有一台公网的服务器，服务器上运行一个程序，这个程序监听两个端口，分别供外部客户端和内网客户端连接，然后这个程序负责转发双方的通信给对方。

![](https://wx3.sinaimg.cn/mw690/66ae68a1ly1g3cj95381qj20oc0gsjsj.jpg)

server.js
```
const net = require('net');

let inSocket, outSocket;
const inServer = net.createServer(function(socket) {
  inSocket = socket;
});

inServer.listen(4001);

const outServer = net.createServer(function(socket) {
  outSocket = socket;
  if (inSocket && outSocket) {
    inSocket.pipe(outSocket);
    outSocket.pipe(inSocket);
  }
});

outServer.listen(4002);
```

in_client.js 内网的客户端
```
const net = require('net');

const socket = net.connect({ port: 4001 }, () => {
  
});

socket.on('data', (data) => {
  console.log(data.toString());
  socket.write('Data from in office');
});

socket.on('end', () => {
  console.log('on end');
});
```

out_client.js 外网的客户端
```
const net = require('net');

const socket = net.connect({ port: 4002 }, () => {
  socket.write('Data from outside');
});

socket.on('data', (data) => {
  console.log(data.toString());
});

socket.on('end', () => {
  console.log('on end');
});
```

先运行server.js，然后运行in_client.js，再运行out_client.js

然后我们可以看到in_client.js和out_client.js互相可以通过server.js通信

这里in_client.js需要先运行，然后再运行out_client.js。类似于我们需要先在内网机器上运行teamviewer，然后再在外部运行teamviewer。

然后我们就可以扩展in_client和out_client的功能，例如当out_client发送特定的命令时，in_client就传输本地的屏幕和键盘事件给对方，这样就能实现类似teamviewer的远程桌面。

回到我们开始的问题，如果内网机器上部署了一个service，如何通过out_client访问到呢？

我们可以在in_client启动时就连接要请求的服务，然后等待外部的访问
```
const net = require('net');

let serviceSocket, socket;

socket = net.connect({ port: 4001 }); // 连接公网服务器

serviceSocket = net.connect({ port: 8080 }); // 假设在http://localhost:8080有一个服务

socket.pipe(serviceSocket);  // 从服务器转发过来的数据都会发送给service
serviceSocket.pipe(socket); // service返回的数据都会转发给socket，进而转发到服务器
```

server.js
```
const net = require('net');

let inSocket, outSocket;
const inServer = net.createServer(function(socket) {
  inSocket = socket;
});

inServer.listen(4001);

const outServer = net.createServer(function(socket) {
  outSocket = socket;
  if (inSocket && outSocket) {
    inSocket.pipe(outSocket);
    outSocket.pipe(inSocket);
  }
});

outServer.listen(4002);
```

这时我们访问http://localhost:4002，我们的请求就会被转发到http://localhost:8080

仍然有一个问题是，http请求完成以后，客户端过一段时间以后会把底层的TCP连接关掉，那么我们在server上建立的，inSocket和outSocket之间的pipe就断掉了。

解决这个问题的思路是，在server上再添加一个listen的socket，这个socket专门用来和in_client通信，告诉in_client有新的连接请求，in_client发现有新的连接请求以后，再和server建立一个用于转发tcp的连接，这个连接可以在通信完就关闭掉。

我们通过这个新建立的socket发送控制命令给in_client，告诉它有新的连接了，所以我们把这个socket叫controlSocket

server.js
```
const net = require('net');

let inSocket, outSocket, controlSocket;

// 这里我们添加了controlServer
const controlServer = net.createServer();
controlServer.on('connection', (socket) => {
  controlSocket = socket;
});
controlServer.listen(4003);

const inServer = net.createServer();
inServer.on('connection', function(socket) {
  inSocket = socket;
  // 这里我们先建立了外部的socket，然后才通过控制socket
  // 告诉in_client建立内部的socket，所以把pipe的代码放在这里
  if (inSocket && outSocket) {
    inSocket.pipe(outSocket);
    outSocket.pipe(inSocket);
  }
});
inServer.listen(4001);

const outServer = net.createServer();
outServer.on('connection', function(socket) {
  outSocket = socket;
  controlSocket.write('new_conn_cmd');
});
outServer.listen(4002);
```

in_client.js
```
const net = require('net');

let serviceSocket, outSocket, controlSocket;

controlSocket = net.connect({ port: 4003 });

controlSocket.on('data', (data) => {
  // 这里没有处理命令的长度
  data = data.toString();
  if (data === 'new_conn_cmd') {
    outSocket = net.connect({ port: 4001 });
    serviceSocket = net.connect({ port: 8080 });
    outSocket.pipe(serviceSocket);
    serviceSocket.pipe(outSocket);
  } else {
    console.log('Invalid commadn');
  }
});
```

这时我们再请求http://127.0.0.1:4002/，等一段时间，连接会被断掉，下一次请求的时候会建立新的连接。因为http协议会复用底层的连接，短时间内连续的请求不会建立新的连接 。
