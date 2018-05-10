# 教程5-`Topics`

[官方教程](http://www.rabbitmq.com/tutorials/tutorial-five-javascript.html)

**(使用 [amqp.node](http://www.squaremobius.net/amqp.node/) 客户端)**

在[之前的教程](https://github.com/OwenLittleWhite/ChineseRabbitMqTutorial/blob/master/Tutorial4.md)中，我们改进了日志记录系统。 我们没有使用只有虚拟广播的扇出(fanout)交换机，而是使用了直接(direct)交换机，并获得了选择性接收日志的可能性。

尽管使用直接交换改进了我们的系统，但它仍然有局限性 - 它不能根据多个标准进行路由。

在我们的日志系统中，我们可能不仅需要根据严重性来订阅日志，还要根据发布日志的来源进行订阅。 您可能从syslog unix工具知道这个概念，该工具根据严重性（信息/警告/严重...）和工具（auth / cron / kern ...）来路由日志。

这会给我们很大的灵活性 - 我们可能想听取来自'cron'的严重错误，而且还听取来自'kern'的所有日志。

为了在我们的日志系统中实现这一点，我们需要了解更复杂的话题(topic)交换机。

## topic交换机

发送到话题交换的消息不能有任意的routing_key - 它必须是由点分隔的单词列表。 单词可以是任何东西，但通常它们指定了与该消息相关的一些功能。 一些有效的路由键例子：`stock.usd.nyse`，`nyse.vmw`，`quick.orange.rabbit`。 只要您愿意，路由键中可以有多少个字，最多255个字节。

绑定键也必须是相同的形式。 topic交换机背后的逻辑类似于direct交换机 - 使用特定路由键发送的消息将被传递到与匹配绑定键绑定的所有队列。 但是绑定键有两个重要的特殊情况：

  *（星号）可以代替一个单词。

  ＃（井号）可以替代零个或多个单词。

用一个例子可以简单的解释：

![python-five](http://www.rabbitmq.com/img/tutorials/python-five.png)

在这个例子中，我们将发送所有描述动物的消息。 消息将使用由三个字（两个点）组成的路由键发送。 路由关键字中的第一个单词将描述速度，第二个颜色和第三个种类：“<speed>.<color>.<species>”。

我们创建了三个绑定：Q1绑定了绑定键 `*.orange.*` ，Q2绑定了 `*.*.rabbit` 和 `lazy.＃` 。

这些绑定可以概括为：

Q1对所有的橙色动物都感兴趣。

Q2希望听到关于兔子的一切，以及关于懒惰动物的一切。

将路由键设置为“quick.orange.rabbit”的消息将传递到两个队列。 消息“lazy.orange.elephant”也会去他们两个。 另一方面，“quick.orange.fox”只会进入第一个队列，而“lazy.brown.fox”只会进入第二个队列。 “lazy.pink.rabbit”只会传递到第二个队列一次，即使它匹配了两个绑定。 “quick.brown.fox”不匹配任何绑定，因此将被丢弃。

如果我们违反我们的合同并发送带有一个或四个单词的消息，如“orange”或“quick.orange.male.rabbit”，会发生什么情况？ 那么，这些消息将不匹配任何绑定，并会丢失。

另一方面，“lazy.orange.male.rabbit”即使有四个单词，也会匹配最后一个绑定，并将传递到第二个队列。

topic 交换机很强大，并且可以表现为其他类型的交换机

一个队列绑定带"#"(井号)的路由器，这个队列会收到所有的消息，忽视路由键， 就像是一个扇出交换机。

特殊字符“*” 和 “#” 绑定中都没有用上，那么这个topic交换机就像是一个直接交换机。

## 全部放一起

我们将在我们的日志系统中使用topic交换机。 我们首先假定日志的路由键有两个词："\<facility\>.\<severity\>"。

代码几乎与[前一个教程中](https://github.com/OwenLittleWhite/ChineseRabbitMqTutorial/blob/master/Tutorial4.md)的代码相同。

`emit_log_topic.js` 的代码:

```
#!/usr/bin/env node

var amqp = require('amqplib/callback_api');

amqp.connect('amqp://localhost', function(err, conn) {
  conn.createChannel(function(err, ch) {
    var ex = 'topic_logs';
    var args = process.argv.slice(2);
    var key = (args.length > 0) ? args[0] : 'anonymous.info';
    var msg = args.slice(1).join(' ') || 'Hello World!';

    ch.assertExchange(ex, 'topic', {durable: false});
    ch.publish(ex, key, new Buffer(msg));
    console.log(" [x] Sent %s:'%s'", key, msg);
  });

  setTimeout(function() { conn.close(); process.exit(0) }, 500);
});
```

`receive_logs_topic.js` 的代码：

```
#!/usr/bin/env node

var amqp = require('amqplib/callback_api');

var args = process.argv.slice(2);

if (args.length == 0) {
  console.log("Usage: receive_logs_topic.js <facility>.<severity>");
  process.exit(1);
}

amqp.connect('amqp://localhost', function(err, conn) {
  conn.createChannel(function(err, ch) {
    var ex = 'topic_logs';

    ch.assertExchange(ex, 'topic', {durable: false});

    ch.assertQueue('', {exclusive: true}, function(err, q) {
      console.log(' [*] Waiting for logs. To exit press CTRL+C');

      args.forEach(function(key) {
        ch.bindQueue(q.queue, ex, key);
      });

      ch.consume(q.queue, function(msg) {
        console.log(" [x] %s:'%s'", msg.fields.routingKey, msg.content.toString());
      }, {noAck: true});
    });
  });
});
```
接收所有日志:

```
./receive_logs_topic.js "#"
```

要从设施“kern”接收所有日志：

```
./receive_logs_topic.js "kern.*"
```

或者，如果您只想听到关于"critical"日志的信息：

```
./receive_logs_topic.js "*.critical"
```

你可以创建多个绑定：

```
./receive_logs_topic.js "kern.*" "*.critical"
```

发布带有路由键“kern.critical”类型的日志：

```
./emit_log_topic.js "kern.critical" "A critical kernel error"
```

玩这些程序玩得开心。 请注意，代码没有对路由或绑定键作任何假设，您可能需要使用两个以上的路由键参数。

(Full source code for emit_log_topic.js and receive_logs_topic.js)

(全部的源代码: [emit_log_topic.js](https://github.com/rabbitmq/rabbitmq-tutorials/blob/master/javascript-nodejs/src/emit_log_topic.js), [receive_logs_topic.js](https://github.com/rabbitmq/rabbitmq-tutorials/blob/master/javascript-nodejs/src/receive_logs_topic.js))

接下来，在[教程6](https://github.com/OwenLittleWhite/ChineseRabbitMqTutorial/blob/master/Tutorial6.md)了解如何用将远程过程调用来做往返消息

