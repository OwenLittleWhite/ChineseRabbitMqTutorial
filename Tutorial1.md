# 教程1-“Hello World”
[官方教程](http://www.rabbitmq.com/tutorials/tutorial-one-javascript.html)
## 介绍
RabbitMQ是一个消息代理：它接受和转发消息。 
你可以把它想象成一个邮局：当你把邮件放在邮箱里时，你可以确定邮差先生最终会把邮件发送给你的收件人。 在这个比喻中，RabbitMQ是邮政信箱，邮局和邮递员。

RabbitMQ与邮局的主要区别是它不处理纸张，而是接受，存储和转发二进制数据块 *-messages*。

RabbitMQ和一般的消息传递使用了一些术语。

*Producing*只不过是发送而已。 一个发送消息的程序就是一个生产者（*producer*）：

![producer](http://www.rabbitmq.com/img/tutorials/producer.png)

队列是RabbitMQ内部的邮箱名称。
尽管消息流经RabbitMQ和您的应用程序，但它们只能存储在队列中。
队列只受主机的内存和磁盘限制，实质上是一个大的消息缓冲区。 
许多生产者可以发送消息到一个队列，许多消费者可以尝试从一个队列接收数据。 这就是我们代表队列的方式：

![queue](http://www.rabbitmq.com/img/tutorials/queue.png)

消费(*consuming*)与接受有类似的意义。 消费者(*consumer*)是一个主要等待接收消息的程序：

![consumer](http://www.rabbitmq.com/img/tutorials/consumer.png)

请注意，生产者，消费者和经纪人不必驻留在同一主机上; 事实上在大多数应用程序中，他们确实如此

## "Hello World"

**(使用 amqp.node 客户端)**

在本教程的这一部分，我们将用Javascript编写两个小程序;
发送单个消息的生产者，以及接收消息并将其打印出来的消费者。
我们将详细介绍[amqp.node](http://www.squaremobius.net/amqp.node/) API中的一些细节，
将注意力集中在这个非常简单的事情上，以便开始。 这是一个消息传递的“Hello World”。

在下图中，“P”是我们的生产者，“C”是我们的消费者。 中间的盒子是一个队列 -- 一个RabbitMQ代表消费者的消息缓冲区。

![python-one](http://www.rabbitmq.com/img/tutorials/python-one.png)

**amqp.node 客户端 library**

RabbitMQ提供多种协议。 本教程使用AMQP 0-9-1，这是一个开放，通用的消息传递协议。
RabbitMQ有许多[不同的语言](http://www.rabbitmq.com/devtools.html)客户端。
我们将在本教程中使用[amqp.node](http://www.squaremobius.net/amqp.node/)客户端。

首先，使用[npm](https://www.npmjs.com/)安装amqp.node：
`npm install amqplib`

现在我们安装了amqp.node，我们可以写一些代码。

### 发送

![sending](http://www.rabbitmq.com/img/tutorials/sending.png)

我们将调用我们的消息发布者（发送者）send.js和我们的消息使用者（接收者）receive.js。 发布者将连接到RabbitMQ，发送一条消息，然后退出。

在send.js中，我们需要引要求库：

```
#!/usr/bin/env node

var amqp = require('amqplib/callback_api');
```

然后连接到RabbitMQ服务器

```
amqp.connect('amqp://localhost', function(err, conn) {});
```
接下来我们创建一个频道，这是大部分完成任务的API驻留的地方：

```
amqp.connect('amqp://localhost', function(err, conn) {
  conn.createChannel(function(err, ch) {});
});
```

发送，我们必须声明队列给我们发送; 然后我们可以发布消息到队列中：

```
amqp.connect('amqp://localhost', function(err, conn) {
  conn.createChannel(function(err, ch) {
    var q = 'hello';

    ch.assertQueue(q, {durable: false});
    // Note: on Node 6 Buffer.from(msg) should be used
    ch.sendToQueue(q, new Buffer('Hello World!'));
    console.log(" [x] Sent 'Hello World!'");
  });
});
```

声明一个队列是[幂等](https://baike.baidu.com/item/%E5%B9%82%E7%AD%89/8600688?fr=aladdin)的--只有当它不存在时才会被创建。 消息内容是一个字节数组，所以你可以对其任意编码。

最后，我们关闭连接并退出;

```
setTimeout(function() { conn.close(); process.exit(0) }, 500);
```

[这是整个send.js文件。](https://github.com/rabbitmq/rabbitmq-tutorials/blob/master/javascript-nodejs/src/send.js)

**发送不成功！**

如果这是您第一次使用RabbitMQ，并且您没有看到“Sent”消息，
那么您可能会绞尽脑汁想知道哪里出了问题。 也许代理启动没有足够的可用磁盘空间（默认情况下，它至少需要200 MB空闲空间），因此拒绝接受消息。 检查代理日志文件以确认并在必要时减少限制。 [配置文件文档](http://www.rabbitmq.com/configure.html#config-items)
将告诉你如何设置`disk_free_limit`。

### 接收

那是我们的发布者。
与发布者不同，发布者发布一条消息，
而消费者是从RabbitMQ接收推送的消息。
我们会持续监听消息，并且把他们打印出来。

![receiving](http://www.rabbitmq.com/img/tutorials/receiving.png)

在`receive.js`代码中需要引进`send`中相同的库：

```
#!/usr/bin/env node

var amqp = require('amqplib/callback_api');
```

设置与发布者相同; 我们打开一个连接和一个通道，并声明我们将要使用的队列。 注意这与`sendToQueue`发布到的队列相匹配。

```
amqp.connect('amqp://localhost', function(err, conn) {
  conn.createChannel(function(err, ch) {
    var q = 'hello';

    ch.assertQueue(q, {durable: false});
  });
});
```

请注意，我们也在这里声明队列。 因为我们可能会在发布者之前启动消费者，所以我们希望确保队列存在，然后再尝试使用消息。

我们即将告诉服务器将队列中的消息传递给我们。 由于它会异步推送消息，因此我们提供了一个回调函数，
当RabbitMQ向消费者推送消息时，将执行回调函数。 这是`Channel.consume`所做的。

```
console.log(" [*] Waiting for messages in %s. To exit press CTRL+C", q);
ch.consume(q, function(msg) {
  console.log(" [x] Received %s", msg.content.toString());
}, {noAck: true});
```

[这是全部的receive.js文件](https://github.com/rabbitmq/rabbitmq-tutorials/blob/master/javascript-nodejs/src/receive.js)

### 全部放一起

现在我们可以运行这两个脚本。 在终端中，从rabbitmq-tutorials / javascript-nodejs / src /文件夹运行publisher：

```
./send.js
```

然后运行consumer

```
./receive.js
```

消费者将通过RabbitMQ打印从发布者处获得的消息。 消费者将继续运行，等待消息（使用Ctrl-C停止它），所以尝试从另一个终端运行发布者。

**列出队列**

你可能想看看RabbitMQ有什么队列，有多少条消息。 您可以使用rabbitmqctl工具（作为特权用户）执行此操作：

```
sudo rabbitmqctl list_queues
```

在Windows上，省略sudo：

```
rabbitmqctl.bat list_queues
```

教程2*work queue*.
