---
title: nodejs学习
date: 2021-02-25 14:16:24
tags: nodejs
---

![1](/nodejs学习/1.jpg)

<!-- more -->

# day1

```js
var fs = require("fs");
var data = fs.readFileSync("input.txt");
console.log(data.toString());
console.log("program end!\n");


console.log("now, we are using callback");
fs.readFile("input.txt", function(err, data){
	if(err) return console.error(err);
	console.log("\n" + data.toString());
});
console.log("callback end!\n");

var events = require("events");
var eventEmitter = new events.EventEmitter();
var connectHandler = function connected() {
	console.log("连接成功");
	eventEmitter.emit("data_received");
}

eventEmitter.on("connection", connectHandler);
eventEmitter.on("data_received", function(){
	console.log("数据接收成功");
});

eventEmitter.emit("connection");
console.log("event end");


eventEmitter.on("some_event", function(arg){
	console.log("some_event 事件触发, 参数是:", arg);
});

setTimeout(function(){
	eventEmitter.emit("some_event", "arg 参数");
}, 1000);
```



# day 2

``` js
var events = require("events");
var eventEmitter = new events.EventEmitter();

var listener = function() {
	console.log("监视器1 执行成功");
}

var listener2 = function() {
	console.log("监视器2 执行成功");
}

eventEmitter.addListener("connection", listener);
eventEmitter.on("connection", listener2);

var cnt = eventEmitter.listenerCount("connection");
console.log("总共有" + cnt + "个监视器");

eventEmitter.emit("connection");

eventEmitter.removeListener("connection", listener);
console.log("移除监视器1 ");

eventEmitter.emit("connection");

cnt = eventEmitter.listenerCount("connection");
console.log("总共有" + cnt + "个监视器");

console.log("program end!");
```

* 貌似eventEmitter的on 和 addListener 功能没有什么差别。（待验证）  

# day3

```js
var http = require("http");
/*http.createServer(function(request, response) {
	response.writeHead(200, {'Content-type': 'text/plain'});
	response.end("Hello js\n");
}).listen(8888);*/

var server = http.createServer((req, res) => {
	res.writeHead(200, {
		'Content-type': 'text/html'
	});

	res.addListener("finish", () => {
		console.log("server response is finished");
	});

	res.end("<h1>HEllO JS</h1>");
}).listen(8080, () => {
	console.log("http server starts at 8080 port");
});

server.on("connection", () => {
	console.log("a client has connected to the server");
});
//console.log("server running at http://127.0.0.1:8888/");

```

* 函数还能这么写，有点意思

# day4

```js
/*for (var i = 0; i < 10; ++i) {
	setTimeout(() => {
		console.log(i);
	}, 0);
}*/

buf = Buffer.from("hello");
console.log(buf.toString());
buf.write("nihao",1);
console.log(buf.toString());

/*
	几个疑问：
	1. fs的ReadStream类绑定data, end之类的 。这些event为什么在api中查询不到？
	2. flag处，为什么取消注释后（即读执行后立马执行写操作），会导致读的内容无法打印
	注：写的操作我没有函数化的时候，读执行后立马执行写是可以读出来的。函数化之后就不行了。
*/
var fs = require("fs");
var data = "";
var readerStream = fs.createReadStream("input.txt");

readerStream.on("data", function(chunk) {
	data += chunk;
});

readerStream.on("end", function() {
	console.log(data);
});

readerStream.on("error", function(err) {
	console.log(err.stack);
});

console.log("read program end");

var write = function() {
	var data = "其实我是c先生阿~~~~";
	var writeStream = fs.createWriteStream("input.txt");
	writeStream.write(data, "UTF-8");
	writeStream.end();

	writeStream.on("finish", function() {
		console.log("write finish");
	});

	writeStream.on("error", function(err) {
		console.log(err.stack);
	});

	console.log("write program end");
}

//flag is here
//write();

setTimeout(write, 2000);

```

* 很奇怪。。。然而周边没有会nodejs的，看能不能解决掉

