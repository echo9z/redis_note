在此之前我们进行的操作都是通过 Redis 的命令行客户端 redis-cli 进行的，并没与介绍实际编程时如何操作 Redis。接下来主要会讲解如何通过具体的编程语言来操作 Redis。
  
## Redis 支持的程序客户端

参考官方支持列表：[https://redis.io/clients](https://redis.io/clients)。

## 在 Node.js 中操作 Redis

Node.js 中可以操作 Redis 的软件包推荐列表：
- [https://redis.io/clients#nodejs](https://redis.io/clients#nodejs)。
- [https://github.com/luin/ioredis](https://github.com/luin/ioredis)

下面主要以 ioredis 为例。
ioredis 特点：
- 功能齐全。它支持集群，前哨，流，流水线，当然还支持Lua脚本和发布/订阅（具有二进制消息的支持）。
- 高性能
- 令人愉快的 API。它的异步 API 支持回调函数与 Promise
- 命令参数和返回值的转换
- 透明键前缀
- Lua脚本的抽象，允许您定义自定义命令。
- 支持二进制数据
- 支持 TLS
- 支持脱机队列和就绪检查
- 支持ES6类型，例如 Map 和 Set
- 支持GEO命令（Redis 3.2不稳定）
- 复杂的错误处理策略
- 支持NAT映射
- 支持自动流水线
- API 参考文档：[https://github.com/luin/ioredis/blob/master/API.md](https://github.com/luin/ioredis/blob/master/API.md)

基本使用  

安装依赖：  
​```
```sh
npm install ioredis
```

```js
const Redis = require("ioredis");
const redis = new Redis(); // uses defaults unless given configuration object

// ioredis supports all Redis commands:
redis.set("foo", "bar"); // returns promise which resolves to string, "OK"

// the format is: redis[SOME_REDIS_COMMAND_IN_LOWERCASE](ARGUMENTS_ARE_JOINED_INTO_COMMAND_STRING)
// the js: ` redis.set("mykey", "Hello") ` is equivalent to the cli: ` redis> SET mykey "Hello" `

// ioredis supports the node.js callback style
redis.get("foo", function (err, result) {
  if (err) {
    console.error(err);
  } else {
    console.log(result); // Promise resolves to "bar"
  }
});

// Or ioredis returns a promise if the last argument isn't a function
redis.get("foo").then(function (result) {
  console.log(result); // Prints "bar"
});

// Most responses are strings, or arrays of strings
redis.zadd("sortedSet", 1, "one", 2, "dos", 4, "quatro", 3, "three");
redis.zrange("sortedSet", 0, 2, "WITHSCORES").then((res) => console.log(res)); // Promise resolves to ["one", "1", "dos", "2", "three", "3"] as if the command was ` redis> ZRANGE sortedSet 0 2 WITHSCORES `

// All arguments are passed directly to the redis server:
redis.set("key", 100, "EX", 10);
```

有关更多实例，可以参考这里：[https://github.com/luin/ioredis/tree/master/examples](https://github.com/luin/ioredis/tree/master/examples)。  
  
### Pipelining  

如果要发送一批命令（例如> 5），则可以使用流水线将命令在内存中排队，然后将它们一次全部发送到 Redis。这样，性能提高了50％〜300％（请参阅基准测试部分）。

`redis.pipeline()` 创建一个 Pipeline 实例。您可以像 Redis 实例一样在其上调用任何 Redis 命令。这些命令在内存中排队，并通过调用 `exec` 方法刷新到 Redis：

```js
const pipeline = redis.pipeline();
pipeline.set("foo", "bar");
pipeline.del("cc");
pipeline.exec((err, results) => {
  // `err` is always null, and `results` is an array of responses
  // corresponding to the sequence of queued commands.
  // Each response follows the format `[err, result]`.
});

// You can even chain the commands:
redis
  .pipeline()
  .set("foo", "bar")
  .del("cc")
  .exec((err, results) => {});

// `exec` also returns a Promise:
const promise = redis.pipeline().set("foo", "bar").get("foo").exec();
promise.then((result) => {
  // result === [[null, 'OK'], [null, 'bar']]
});
```
每个链接的命令还可以具有一个回调，该回调将在命令得到答复时被调用：  
```js
redis
  .pipeline()
  .set("foo", "bar")
  .get("foo", (err, result) => {
    // result === 'bar'
  })
  .exec((err, result) => {
    // result[1][1] === 'bar'
  });
```
除了将命令分别添加到管道队列之外，您还可以将命令和参数数组传递给构造函数：
```js
redis
  .pipeline([
    ["set", "foo", "bar"],
    ["get", "foo"],
  ])
  .exec(() => {
    /* ... */
  });
```

`#length` 属性显示管道中有多少个命令：
```js
const length = redis.pipeline().set("foo", "bar").get("foo").length;
// length === 2
```

### 事务

大多数时候，事务命令 `multi＆exec` 与管道一起使用。因此，在调用 `multi` 时，默认情况下会自动创建 `Pipeline` 实例，因此您可以像使用管道一样使用 `multi`：
```js
redis
  .multi()
  .set("foo", "bar")
  .get("foo")
  .exec((err, results) => {
    // results === [[null, 'OK'], [null, 'bar']]
  });
```
如果事务的命令链中存在语法错误（例如，错误的参数数量，错误的命令名称等），则不会执行任何命令，并返回错误：
```js
redis
  .multi()
  .set("foo")
  .set("foo", "new value")
  .exec((err, results) => {
    // err:
    //  { [ReplyError: EXECABORT Transaction discarded because of previous errors.]
    //    name: 'ReplyError',
    //    message: 'EXECABORT Transaction discarded because of previous errors.',
    //    command: { name: 'exec', args: [] },
    //    previousErrors:
    //     [ { [ReplyError: ERR wrong number of arguments for 'set' command]
    //         name: 'ReplyError',
    //         message: 'ERR wrong number of arguments for \'set\' command',
    //         command: [Object] } ] }
  });
```
就接口而言，`multi` 与管道的区别在于，当为每个链接的命令指定回调时，排队状态将传递给回调，而不是命令的结果：
```js
redis
  .multi()
  .set("foo", "bar", (err, result) => {
    // result === 'QUEUED'
  })
  .exec(/* ... */);
```
如果要使用不带管道的事务，请将 `{ pipeline: false }` 传递给 `multi`，每个命令将立即发送到 Redis，而无需等待 `exec` 调用：
```js
redis.multi({ pipeline: false });
redis.set("foo", "bar");
redis.get("foo");
redis.exec((err, result) => {
  // result === [[null, 'OK'], [null, 'bar']]
});
```
`multi` 的构造函数还接受一批命令：
```js
redis
  .multi([
    ["set", "foo", "bar"],
    ["get", "foo"],
  ])
  .exec(() => {
    /* ... */
  });
```
管道支持内联事务，这意味着您可以将管道中的命令子集分组为一个事务：
```js
redis
  .pipeline()
  .get("foo")
  .multi()
  .set("foo", "bar")
  .get("foo")
  .exec()
  .get("foo")
  .exec();
```

### 错误处理  
Redis服务器返回的所有错误都是 ReplyError 的实例，可以通过 Redis 进行访问：  

```js
const Redis = require("ioredis");
const redis = new Redis();
// This command causes a reply error since the SET command requires two arguments.
redis.set("foo", (err) => {
  err instanceof Redis.ReplyError;
});
```
这是 ReplyError 的错误堆栈：  

```txt
ReplyError: ERR wrong number of arguments for 'set' command
    at ReplyParser._parseResult (/app/node_modules/ioredis/lib/parsers/javascript.js:60:14)
    at ReplyParser.execute (/app/node_modules/ioredis/lib/parsers/javascript.js:178:20)
    at Socket.<anonymous> (/app/node_modules/ioredis/lib/redis/event_handler.js:99:22)
    at Socket.emit (events.js:97:17)
    at readableAddChunk (_stream_readable.js:143:16)
    at Socket.Readable.push (_stream_readable.js:106:10)
    at TCP.onread (net.js:509:20)
```

默认情况下，错误堆栈没有任何意义，因为整个堆栈都发生在 ioredis 模块本身而不是代码中。因此，要找出错误在代码中的位置并不容易。 ioredis 提供了一个选项 showFriendlyErrorStack 来解决该问题。启用 showFriendlyErrorStack 时，ioredis 将为您优化错误堆栈：  

```js
const Redis = require("ioredis");
const redis = new Redis({ showFriendlyErrorStack: true });
redis.set("foo");
```

输出将是：  

```js
ReplyError: ERR wrong number of arguments for 'set' command
at Object.<anonymous> (/app/index.js:3:7)
at Module._compile (module.js:446:26)
at Object.Module._extensions..js (module.js:464:10)
at Module.load (module.js:341:32)
at Function.Module._load (module.js:296:12)
at Function.Module.runMain (module.js:487:10)
at startup (node.js:111:16)
at node.js:799:3
```

这次，堆栈告诉您错误发生在代码的第三行。  

太好了！但是，优化错误堆栈会大大降低性能。因此，默认情况下，此选项是禁用的，只能用于调试目的。不建议在生产环境中使用此功能。  
