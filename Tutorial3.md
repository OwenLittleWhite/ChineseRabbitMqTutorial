# 教程3-”Publish/Subscribe“

[官方教程](http://www.rabbitmq.com/tutorials/tutorial-three-javascript.html)

**(使用 [amqp.node](http://www.squaremobius.net/amqp.node/) 客户端)**

在[前面的教程](https://github.com/OwenLittleWhite/ChineseRabbitMqTutorial/blob/master/Tutorial2.md)中，
我们创建了一个Work Queue。 Work Queue背后的假设是每个任务实际上只被传递给一个worker。
在这一部分，我们将做一些完全不同的事情 - 我们会向多个consumer传递信息。 
这种模式被称为“发布/订阅”（*Publish/Subscribe*）。

为了说明这个模式，我们将建立一个简单的日志系统。 它将包含两个程序 - 第一个将发射日志消息，第二个将接收并打印它们。

在我们的日志记录系统中，接收程序的每个运行副本都会收到消息。 
这样我们就可以运行一个接收器并将日志指向磁盘; 同时我们将能够运行另一个接收器并在屏幕上查看日志。

本质上，发布的日志消息将被广播给所有的接收者。

## 交换机

在本教程的以前部分，我们发送和接收来自队列的消息。 现在是时候在Rabbit中引入完整的消息模型。

让我们快速回顾一下前面的教程中介绍的内容：

* *productor*是发送消息的用户应用程序。
* *queue*是存储消息的缓冲区。
* *consumer*是接收消息的用户应用程序。

RabbitMQ中消息传递模型的核心思想是生产者永远不会将任何消息直接发送到队列中。 实际上，生产者通常甚至不知道消息是否会被传送到任何队列中。

相反，生产者只能发送消息给交换机。 交换机很简单。 一方面它接收来自生产者的消息，另一方把他们推到队列中。 交换机必须知道如何处理收到的消息。 
是否应该附加到特定的队列？ 还是加到许多队列中？ 还是应该丢弃。 这些规则是由交换机的类型定义的。

![exchanges](http://www.rabbitmq.com/img/tutorials/exchanges.png)


有一些可用的交换机的类型：`direct`, `topic`, `headers`, `fanout`。
我们关注于最后一个 -- 扇出（fanout）。现在创建一个扇出类型的交换机，并叫它`logs`:

```
ch.assertExchange('logs', 'fanout', {durable: false})
```

扇出交换机非常简单。 正如你可能从名字中猜到的那样，它只是将所有收到的消息广播到它知道的所有队列中。 这正是我们logger所需要的。

**列出交换机**

你可以运行`rabbitmqctl`命令来列出服务器中所有的交换机：

```
sudo rabbitmqctl list_exchanges
```

在这个队列中有一些`amq.*`的交换机和默认的（未命名的）交换机。这些是默认创建的，现在你还用不到这些。

**默认交换机**

在前面的教程中，我们没有提到交换机，但是仍然可以发送消息到队列。
有可能是因为我们使用了一个默认的交换机，交换机的标识是一个空字符串`''`

回顾下之前我们是怎么发布消息

```
ch.sendToQueue('hello', new Buffer('Hello World!'));
```

这里我们使用了默认的或者是无名的交换机：
消息被路由到队列中，队列的名字是由第一个参数定义的，如果第一个参数存在的话。

现在我们可以把消息发布到命名的交换机上了：

```
ch.publish('logs', '', new Buffer('Hello World!'));
```

第二个参数空的字符串是说我们不想将消息发送到任何一个特定的队列中，而只是将它发布到`logs`交换机。

## 临时队列

正如你以前可能记得我们使用的是具有指定名称的队列（记得*hello*和*task_queue*吗？）。 
能够列出队列对我们来说至关重要 - 我们需要指定workers到同一队列。 当你想在生产者和消费者之间共享队列时，给队列一个名字是很重要的。

但是我们的logger并不是这样。 我们希望了解所有日志消息，而不仅仅是其中的一部分。 我们也只对目前流动的消息感兴趣，而不是旧的。 
要解决这个问题，我们需要两件事。

首先，每当我们连接到Rabbit，
我们需要一个新的，空的队列。 要做到这一点，我们可以创建一个随机名称的队列，或者，甚至更好 - 让服务器为我们选择一个随机的队列名称。

其次，一旦我们断开消费者，队列应该被自动删除。

在amqp.node客户端中，当我们将队列名称作为空字符串提供时，我们使用生成的名称创建一个非持久队列：

```
ch.assertQueue('', {exclusive: true});
```

当方法返回时，队列实例包含由RabbitMQ生成的随机队列名称。 例如，它可能看起来像`amq.gen-JzTY20BRgKO-HjmUJj0wLg`。

当声明它的连接关闭时，队列将被删除，因为它被声明为独占。

## 绑定

![bingdings](http://www.rabbitmq.com/img/tutorials/bindings.png)

我们已经创建了一个扇出交换机和一个队列。 现在我们需要告诉交换机将消息发送到我们的队列。 交换机和队列之间的关系被称为绑定。

```
ch.bindQueue(queue_name, 'logs', '');
```

从现在起，logger交换机将把消息附加到我们的队列中。

**列出绑定**

你可以列出所有的绑定，使用什么命令，你猜到了，

```
sudo rabbitmqctl list_bindings
```

## 全部放一起

![python-three-overall.png](http://www.rabbitmq.com/img/tutorials/python-three-overall.png)

发出日志消息的生产者程序与前面的教程没有什么不同。 最重要的变化是，
我们现在要发布消息到我们的logger交换机，而不是无名的。 发送时我们需要提供一个路由键，但是对于扇出交换机，它的值将被忽略。 这里是`emit_log.js`代码：

```
#!/usr/bin/env node

var amqp = require('amqplib/callback_api');

amqp.connect('amqp://localhost', function(err, conn) {
  conn.createChannel(function(err, ch) {
    var ex = 'logs';
    var msg = process.argv.slice(2).join(' ') || 'Hello World!';

    ch.assertExchange(ex, 'fanout', {durable: false});
    ch.publish(ex, '', new Buffer(msg));
    console.log(" [x] Sent %s", msg);
  });

  setTimeout(function() { conn.close(); process.exit(0) }, 500);
});
```

[(emit_log.js 源代码)](https://github.com/rabbitmq/rabbitmq-tutorials/blob/master/javascript-nodejs/src/emit_log.js)

如你所见，建立连接后，我们声明交换机。 这一步是必要的，因为不能发布到一个不存在的交易机。

如果没有队列绑定到这个交换机上消息就会丢失，但是没有关系。如果没有消费者在听，我们就可以安全的忽略这些消息。

`receive_logs.js`的代码:

```
#!/usr/bin/env node

var amqp = require('amqplib/callback_api');

amqp.connect('amqp://localhost', function(err, conn) {
  conn.createChannel(function(err, ch) {
    var ex = 'logs';

    ch.assertExchange(ex, 'fanout', {durable: false});

    ch.assertQueue('', {exclusive: true}, function(err, q) {
      console.log(" [*] Waiting for messages in %s. To exit press CTRL+C", q.queue);
      ch.bindQueue(q.queue, ex, '');

      ch.consume(q.queue, function(msg) {
        console.log(" [x] %s", msg.content.toString());
      }, {noAck: true});
    });
  });
});
```
[(receive_logs.js 源代码)](https://github.com/rabbitmq/rabbitmq-tutorials/blob/master/javascript-nodejs/src/receive_logs.js)

如果你想要将logs保存成文件的话，只需要打开控制台并且输入：

```
./receive_logs.js > logs_from_rabbit.log
```

如果你想看到屏幕上的logs，产生一个新的终端，并运行：

```
./receive_logs.js
```

发射log，输入

```
./emit_log.js
```

使用 `rabbitmqctl list_bindings`你可以判断代码里是不是像我们想要的那样创建绑定和队列
运行两个`receive_logs.js`项目你看到的可能是这样：

```
sudo rabbitmqctl list_bindings
# => Listing bindings ...
# => logs    exchange        amq.gen-JzTY20BRgKO-HjmUJj0wLg  queue           []
# => logs    exchange        amq.gen-vso0PVvyiRIL2WoV3i48Yg  queue           []
# => ...done.
```

结果的解释很简单：交换机logs中的数据转到两个带有服务器分配名称的队列中。 这正是我们的意图。

想知道我们如何监听消息的一部分，让我们到[教程4](https://github.com/OwenLittleWhite/ChineseRabbitMqTutorial/blob/master/Tutorial4.md)
