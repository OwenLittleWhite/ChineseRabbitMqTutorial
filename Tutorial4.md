# 教程4-”Routing“

[官方教程](http://www.rabbitmq.com/tutorials/tutorial-four-javascript.html)

**(使用 [amqp.node](http://www.squaremobius.net/amqp.node/) 客户端)**

在[之前的教程中](https://github.com/OwenLittleWhite/ChineseRabbitMqTutorial/blob/master/Tutorial3.md)，
我们构建了一个简单的日志系统 我们能够将日志消息广播给许多接收者。

在这篇教程中，
我们将添加一个功能 —— 只订阅一部分消息。 例如，只将重要的错误消息记录到日志文件（以节省磁盘空间），同时仍然能够在控制台上打印所有的日志消息。

## Bindings（绑定）

在前面的例子中，我们已经创建了绑定。 你可能会记得像这样的代码:

```
ch.bindQueue(q.queue, ex, '');
```

绑定是交换机和队列之间的关系。 这可以简单地理解为：队列对来自这个交换机的消息感兴趣。

绑定可以采用额外的绑定键参数（上面代码中的空字符串）。 下面是我们如何用键创建绑定：

```
ch.bindQueue(queue_name, exchange_name, 'black');
```

绑定键的含义取决于交换机的类型。 我们之前使用的扇出交换机，忽略了它。

## Direct exchange(指定交换机)


我们之前教程的日志记录系统将所有消息广播给所有消费者。 我们希望扩展这个功能，以便根据消息的严重性来过滤消息。
例如，我们可能希望将日志消息写入磁盘的脚本仅接收严重错误，而不会在警告或信息日志消息上浪费磁盘空间。

我们之前使用的`fanout`扇出交换机，并不是很灵活 - 它只能够无脑地广播。

我们将使用`direct`交换机。 指定交换机背后的路由算法很简单 - 消息进入`bingding key`(绑定密钥)与消息的`routing key`(路由密钥)完全匹配的队列。

为了说明这一点，请考虑以下设置：

![direct-exchange](http://www.rabbitmq.com/img/tutorials/direct-exchange.png)

在这个设置中，我们可以看到两个队列绑定的`direct`交换机X. 第一个队列用绑定键橙色绑定，第二个队列有两个绑定，一个绑定键为黑色，另一个为绿色。

## 多个绑定

![direct-exchange-multiple](http://www.rabbitmq.com/img/tutorials/direct-exchange-multiple.png)

用相同的绑定键绑定多个队列是完全合法的。在我们的例子中，我们可以在X和Q1之间用绑定键`black`添加绑定，这样子的话，这个`direct`交换机表现得像`fanout`
，将消息广播给所有匹配的队列，路由键为`black`的消息将会发送给Q1和Q2。

## 发射日志

我们将这个模型用在日志系统中。我们不再将消息发送到`fanout`交换机而是`direct`交换机。我们将日志的严重性作为路由键。这样，接收的脚本要选择它想要接收哪种严重程度的日志
。首先，我们关注如何发射日志。

像之前一样，我们需要先创建一个交换机：

```
var ex = 'direct_logs';

ch.assertExchange(ex, 'direct', {durable: false});
```

然后我们准备发送一条消息：

```
var ex = 'direct_logs';

ch.assertExchange(ex, 'direct', {durable: false});
ch.publish(ex, severity, new Buffer(msg));
```
简单点2偶们假设严重性是'info','warning','error'其中之一。

## 订阅

接收消息的脚本跟上一篇教程一样。有一点不同 —— 我们为每一个我们感兴趣的严重性创建一个新的绑定。

```
args.forEach(function(severity) {
  ch.bindQueue(q.queue, ex, severity);
});

```

## 所有的放一起

![python-four](http://www.rabbitmq.com/img/tutorials/python-four.png)

` emit_log_direct.js`脚本的代码：

```
#!/usr/bin/env node

var amqp = require('amqplib/callback_api');

amqp.connect('amqp://localhost', function(err, conn) {
  conn.createChannel(function(err, ch) {
    var ex = 'direct_logs';
    var args = process.argv.slice(2);
    var msg = args.slice(1).join(' ') || 'Hello World!';
    var severity = (args.length > 0) ? args[0] : 'info';

    ch.assertExchange(ex, 'direct', {durable: false});
    ch.publish(ex, severity, new Buffer(msg));
    console.log(" [x] Sent %s: '%s'", severity, msg);
  });

  setTimeout(function() { conn.close(); process.exit(0) }, 500);
});
```

`receive_logs_direct.js`的代码：
 
 ```
 #!/usr/bin/env node

var amqp = require('amqplib/callback_api');

var args = process.argv.slice(2);

if (args.length == 0) {
  console.log("Usage: receive_logs_direct.js [info] [warning] [error]");
  process.exit(1);
}

amqp.connect('amqp://localhost', function(err, conn) {
  conn.createChannel(function(err, ch) {
    var ex = 'direct_logs';

    ch.assertExchange(ex, 'direct', {durable: false});

    ch.assertQueue('', {exclusive: true}, function(err, q) {
      console.log(' [*] Waiting for logs. To exit press CTRL+C');

      args.forEach(function(severity) {
        ch.bindQueue(q.queue, ex, severity);
      });

      ch.consume(q.queue, function(msg) {
        console.log(" [x] %s: '%s'", msg.fields.routingKey, msg.content.toString());
      }, {noAck: true});
    });
  });
});
 ```
 
 如果你只是想保存‘warning’和‘error’的日志不包括‘info’到文件中，打开控制台并输入：
 
 ```
 ./receive_logs_direct.js warning error > logs_from_rabbit.log
 ```

如果你想在屏幕上看到所有的消息，打开一个新的终端，然后输入：
 
```
./receive_logs_direct.js info warning error
# => [*] Waiting for logs. To exit press CTRL+C
```

然后，举个例子，发出一个`error`的日志消息，只需要输入：
  
 ```
 ./emit_log_direct.js error "Run. Run. Or it will explode."
# => [x] Sent 'error':'Run. Run. Or it will explode.'
 ```

[发射日志的源代码](https://github.com/rabbitmq/rabbitmq-tutorials/blob/master/javascript-nodejs/src/emit_log_direct.js) 以及 [接收日志的源代码](https://github.com/rabbitmq/rabbitmq-tutorials/blob/master/javascript-nodejs/src/receive_logs_direct.js)


想知道如何接收特定模式的消息吗？接下来移步到[Tutorial5](https://github.com/OwenLittleWhite/ChineseRabbitMqTutorial/blob/master/Tutorial5.md)
