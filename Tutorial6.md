# 教程6-远程过程调用（RPC）
[官方教程](http://www.rabbitmq.com/tutorials/tutorial-six-javascript.html)

**(使用 [amqp.node](http://www.squaremobius.net/amqp.node/) 客户端)**

在[第二篇教程](https://github.com/OwenLittleWhite/ChineseRabbitMqTutorial/blob/master/Tutorial2.md)中，
我们学习了如何使用 *工作队列* 在多个工作人员之间分配耗时的任务。

但是如果我们需要在远程计算机上运行一个函数并等待结果呢？ 嗯，这是一个不同的故事。 此模式通常称为 *远程过程调用* 或 *RPC* 。

在本教程中，我们将使用RabbitMQ构建RPC系统：客户端和可伸缩的RPC服务器。 
由于我们没有任何值得分发的耗时任务，我们将创建一个返回Fibonacci数字的虚拟RPC服务。

## Callback queue（回调队列）

一般来说，通过RabbitMQ进行RPC很容易。 客户端发送请求消息，服务器回复响应消息。
为了接收响应，我们需要发送带有请求的“回调”队列地址。 我们可以使用默认队列。 我们来试试吧：

```JavaScript
ch.assertQueue('', {exclusive: true});

ch.sendToQueue('rpc_queue',new Buffer('10'), { replyTo: queue_name });

# ... then code to read a response message from the callback queue ...
```

>消息属性<br><br>
AMQP 0-9-1协议预定义了一组带有消息的14个属性。 大多数属性很少使用，但以下情况除外：<br><br>
`persistent`：将消息标记为持久性（值为`true`）或瞬态（`false`）。
您可能会记住[第二个教程](https://github.com/OwenLittleWhite/ChineseRabbitMqTutorial/blob/master/Tutorial2.md)中的这个属性。<br><br>
`content_type`：用于描述编码的mime类型。 例如，对于经常使用的JSON编码，将此属性设置为：`application/json`是一种很好的做法。<br><br>
`reply_to`：通常用于命名回调队列。<br><br>
`correlation_id`：用于将RPC响应与请求相关联。

## Correlation Id(相关ID)

在上面介绍的方法中，我们建议为每个RPC请求创建一个回调队列。 这是非常低效的，但幸运的是有更好的方法 - 让我们为每个客户端创建一个回调队列。

这引发了一个新问题，在该队列中收到响应后，不清楚响应属于哪个请求。 
那是在使用`correlation_id`属性的时候。 我们将为每个请求将其设置为唯一值。 
稍后，当我们在回调队列中收到一条消息时，我们将查看此属性，并根据该属性，我们将能够将响应与请求进行匹配。
如果我们看到未知的`correlation_id`值，我们可以安全地丢弃该消息 - 它不属于我们的请求。

您可能会问，为什么我们应该忽略回调队列中的未知消息，而不是失败并出现错误？
这是由于服务器端可能存在竞争条件。 
虽然不太可能，但RPC服务器可能会在向我们发送答案之后，但在发送请求的确认消息之前死亡。 
如果发生这种情况，重新启动的RPC服务器将再次处理请求。 这就是为什么在客户端上我们必须优雅地处理重复的响应，理想情况下RPC应该是幂等的。

## 总结

![python-six.png](http://www.rabbitmq.com/img/tutorials/python-six.png)

我们的RPC将这样工作：

客户端启动时，会创建一个匿名的独占回调队列。
对于RPC请求，客户端发送带有两个属性的消息：`reply_to`（设置为回调队列）和`correlation_id`（设置为每个请求的唯一值）。
请求被发送到`rpc_queue`队列。

RPC worker（aka：server）正在等待该队列上的请求。 当出现请求时，它会执行该作业，并使用`reply_to`字段中的队列将带有结果的消息发送回客户端。

客户端等待回调队列上的数据。 出现消息时，它会检查`correlation_id`属性。 如果它与请求中的值匹配，则返回对应用程序的响应。

## 把它们放在一起

斐波那契函数：

```JavaScript
function fibonacci(n) {
  if (n == 0 || n == 1)
    return n;
  else
    return fibonacci(n - 1) + fibonacci(n - 2);
}

```

我们声明我们的斐波那契函数。 它假定只有有效的正整数输入。 （不要指望这个适用于大数字，它可能是最慢的递归实现）。

我们的RPC服务器[rpc_server.js](https://github.com/rabbitmq/rabbitmq-tutorials/blob/master/javascript-nodejs/src/rpc_server.js)
的代码如下所示：

```JavaScript
#!/usr/bin/env node

var amqp = require('amqplib/callback_api');

amqp.connect('amqp://localhost', function(err, conn) {
  conn.createChannel(function(err, ch) {
    var q = 'rpc_queue';

    ch.assertQueue(q, {durable: false});
    ch.prefetch(1);
    console.log(' [x] Awaiting RPC requests');
    ch.consume(q, function reply(msg) {
      var n = parseInt(msg.content.toString());

      console.log(" [.] fib(%d)", n);

      var r = fibonacci(n);

      ch.sendToQueue(msg.properties.replyTo,
        new Buffer(r.toString()),
        {correlationId: msg.properties.correlationId});

      ch.ack(msg);
    });
  });
});

function fibonacci(n) {
  if (n == 0 || n == 1)
    return n;
  else
    return fibonacci(n - 1) + fibonacci(n - 2);
}
```

服务器代码非常简单：

像往常一样，我们首先建立连接，频道和声明队列。
我们可能希望运行多个服务器进程。
为了在多个服务器上平均分配负载，我们需要在频道上设置`prefetch`设置。
我们使用`Channel.consume`来使用队列中的消息。 然后我们进入回调函数，在那里做工作并发回响应。

我们的RPC客户端[rpc_client.js](https://github.com/rabbitmq/rabbitmq-tutorials/blob/master/javascript-nodejs/src/rpc_client.js)的代码：

```JS
#!/usr/bin/env node

var amqp = require('amqplib/callback_api');

var args = process.argv.slice(2);

if (args.length == 0) {
  console.log("Usage: rpc_client.js num");
  process.exit(1);
}

amqp.connect('amqp://localhost', function(err, conn) {
  conn.createChannel(function(err, ch) {
    ch.assertQueue('', {exclusive: true}, function(err, q) {
      var corr = generateUuid();
      var num = parseInt(args[0]);

      console.log(' [x] Requesting fib(%d)', num);

      ch.consume(q.queue, function(msg) {
        if (msg.properties.correlationId == corr) {
          console.log(' [.] Got %s', msg.content.toString());
          setTimeout(function() { conn.close(); process.exit(0) }, 500);
        }
      }, {noAck: true});

      ch.sendToQueue('rpc_queue',
      new Buffer(num.toString()),
      { correlationId: corr, replyTo: q.queue });
    });
  });
});

function generateUuid() {
  return Math.random().toString() +
         Math.random().toString() +
         Math.random().toString();
}
```

现在是查看[rpc_client.js](https://github.com/rabbitmq/rabbitmq-tutorials/blob/master/javascript-nodejs/src/rpc_client.js)
和[rpc_server.js](https://github.com/rabbitmq/rabbitmq-tutorials/blob/master/javascript-nodejs/src/rpc_server.js)的完整示例源代码的好时机。

我们的RPC服务现已准备就绪。 我们可以启动服务器：

```bash
./rpc_server.js
# => [x] Awaiting RPC requests
```

要请求斐波纳契数，请运行客户端：

```bash
./rpc_client.js 30
# => [x] Requesting fib(30)
```

此处介绍的设计并不是RPC服务的唯一可能实现，但它具有一些重要优势：

* 如果RPC服务器太慢，您可以通过运行另一个服务器来扩展。 尝试在新控制台中运行第二个`rpc_server.js`。

* 在客户端，RPC只需要发送和接收一条消息。 因此，对于单个RPC请求，RPC客户端只需要一次网络往返。

我们的代码仍然相当简单，并不试图解决更复杂（但重要）的问题，例如：

* 如果没有运行服务器，客户应该如何反应？

* 客户端是否应该为RPC设置某种超时？

* 如果服务器出现故障并引发异常，是否应将其转发给客户端？

* 在处理之前防止无效的传入消息（例如检查边界，类型）。

