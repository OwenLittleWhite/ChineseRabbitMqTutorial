# 教程2-“Work Queues”

[官方教程](http://www.rabbitmq.com/tutorials/tutorial-two-javascript.html)

**(使用 [amqp.node](http://www.squaremobius.net/amqp.node/) 客户端)**

![python-two.png](http://www.rabbitmq.com/img/tutorials/python-two.png)

在[教程1](https://github.com/OwenLittleWhite/ChineseRabbitMqTutorial/blob/master/Tutorial1.md)中，
我们编写了用于从命名队列发送和接收消息的程序。 在这一个中，我们将创建一个*Work Queues*，用于在多个workers之间分配耗时的任务。

*Work Queues*（又名：任务队列）背后的主要思想是避免立即执行必须等待完成的，资源密集型任务。 而是安排稍后完成任务。
我们把一个任务封装成一个消息并发送给一个队列。 在后台运行的工作进程取出任务并最终执行作业。 当你运行workers时，任务将在他们之间共享。

这个概念在web应用程序中特别有用，在短的HTTP请求窗口中不可能处理复杂的任务。

## 准备

在本教程的前一部分，
我们发送了一个包含“Hello World！”的消息。 
现在我们将发送代表复杂任务的字符串。 我们没有现实世界的任务，比如图像被重新调整大小或者PDF文件被渲染，所以让我们假装我们忙 - 通过使用
`setTimeout`方法来伪装它。 我们将把字符串中的点数作为复杂度。 每一个点将占到“工作”的一秒钟。 例如，由`Hello ...`描述的假任务将需要三秒钟的时间。

我们稍微修改前面例子中的`send.js`代码，以允许从命令行发送任意消息。 这个程序将安排任务到我们的工作队列，所以让我们把它命名为`new_task.js`：

```
var q = 'task_queue';
var msg = process.argv.slice(2).join(' ') || "Hello World!";

ch.assertQueue(q, {durable: true});
ch.sendToQueue(q, new Buffer(msg), {persistent: true});
console.log(" [x] Sent '%s'", msg);
```

我们旧的`receive.js`脚本也需要进行一些更改：需要伪造工作，一个点代表一秒。 它将从队列中弹出消息并执行任务，所以我们称之为`worker.js`：

```
ch.consume(q, function(msg) {
  var secs = msg.content.toString().split('.').length - 1;

  console.log(" [x] Received %s", msg.content.toString());
  setTimeout(function() {
    console.log(" [x] Done");
  }, secs * 1000);
}, {noAck: true});
```

请注意，我们的假任务是在模拟执行时间。

像在教程1中那样运行这两个文件

```
# shell 1
./worker.js
```

    # shell 2
    ./new_task.js
    
## 循环调度

循环调度使用任务队列的一个优点是能够轻松地平行工作。 如果我们积压工作，我们可以增加更多的worker，这样可以轻松扩展。

首先，我们试着同时运行两个worker.js脚本。 他们都会从队列中得到消息，但究竟是如何？ 一起来看看。

您需要打开三个控制台。 两个将运行worker.js脚本。 这些控制台将是我们的两个consumer（worker） - C1和C2。

```
# shell 1
./worker.js
# => [*] Waiting for messages. To exit press CTRL+C
```

```
# shell 2
./worker.js
# => [*] Waiting for messages. To exit press CTRL+C
```

在第三个，我们将发布新的任务。 一旦你开始了consumer，你可以发布一些消息：

```
# shell 3
./new_task.js First message.
./new_task.js Second message..
./new_task.js Third message...
./new_task.js Fourth message....
./new_task.js Fifth message.....
```

一起来看看consumer接收到了什么：

```
# shell 1
./worker.js
# => [*] Waiting for messages. To exit press CTRL+C
# => [x] Received 'First message.'
# => [x] Received 'Third message...'
# => [x] Received 'Fifth message.....'
```

```
# shell 2
./worker.js
# => [*] Waiting for messages. To exit press CTRL+C
# => [x] Received 'Second message..'
# => [x] Received 'Fourth message....'
```

默认情况下，RabbitMQ将按顺序将每条消息
发送给下一个使用者。 平均而言，每个worker将获得相同数量的消息。 这种分发消息的方式称为循环法（round-robin）。 试试三个或更多的worker。

## 消息确认

做任务可能需要几秒钟的时间。 你可能会想知道如果其中一个consumer
做一个很耗时
的任务，并且只是部分完成就挂掉了会
怎么样。 使用目前的代码，一旦RabbitMQ向客户发送消息，立即将其标记为删除。 
在这种情况下，如果你关闭了一个worker，刚刚处理的信息会丢失。
所有派发给这个worker的尚未处理的消息也会丢失。

但我们不想失去任何任务。 如果worker挂掉，我们希望将任务交付给另一个worker去做。

为了确保消息永不丢失，RabbitMQ支持消息确认。 consumer发回确认（告知）告诉RabbitMQ已经收到，处理了一个特定的消息，并且RabbitMQ可以自由删除它。

如果一个consumer还没有发送确认就挂掉了（通道关闭，连接关闭，TCP连接丢失），RabbitMQ会明白消息没有完全执行完，然后重新把它放回到队列中。
如果同时有其他的consumers在线的话，很快这个消息会发送给一个consumer。这样的话你可以确保没有消息丢失，即使workers偶然挂掉了。

没有任何消息超时; 当consumer挂掉时，RabbitMQ将重新传递消息。 即使处理消息需要非常长的时间也没关系。

在前面的例子中，消息确认没有打开。 现在是时候使用`{noAck：false}`来打开它们（您也可以完全删除选项），并在完成任务后,worker会发送正确的确认。

```
ch.consume(q, function(msg) {
  var secs = msg.content.toString().split('.').length - 1;

  console.log(" [x] Received %s", msg.content.toString());
  setTimeout(function() {
    console.log(" [x] Done");
    ch.ack(msg);
  }, secs * 1000);
}, {noAck: false});
```

使用这段代码，我们可以确定，即使在处理消息的时候使用CTRL + C来杀死一个worker，也不会丢失任何东西。 worker关掉后不久，所有未确认的消息将被重新发送。

**忘记确认**
忘记`ack`是一个常见的错误。 很容易犯的错误，但后果是严重的。 当你的客户端
退出时（这可能看起来像随机的重新传递），消息将被重新传递，但是RabbitMQ将会消耗越来越多的内存，因为它将不能释放任何未被确认的消息。

为了调试这种错误，你可以使用`rabbitmqctl`打印`messages_unacknowledged`字段：

## 消息持久化

我们已经学会了如何确保即使consumer挂掉，任务也不会丢失。 但是如果RabbitMQ服务器停止，我们的任务仍然会丢失。

当RabbitMQ退出或崩溃时，它会忘记队列和消息，除非你告诉它不能忘。 需要做两件事来确保消息不会丢失：我们需要将队列和消息标记为持久。

首先，我们需要确保RabbitMQ永远不会失去队列。 为了做到这一点，我们需要声明它是*durable*：

```
ch.assertQueue('hello', {durable: true});
```

虽然这个命令本身是正确的，但是在我们目前的设置中不起作用。 这是因为我们已经定义了一个名为`hello`的队列。
RabbitMQ不允许你使用不同的参数重新定义一个已经存在
的队列，并且会向任何尝试这样做的程序返回一个错误。 但有一个快速的解决方法 - 让我们声明一个不同名称的队列，例如`task_queue`：

```
ch.assertQueue('task_queue', {durable: true});
```

producer和consumer都要设置`durable`选项

此时我们确信，即使RabbitMQ重新启动，task_queue队列也不会丢失。 现在我们需要将消息标记为持久的 - 通过使用Channel.sendToQueue的`persistent`选项。

```
ch.sendToQueue(q, new Buffer(msg), {persistent: true});
```

**注意消息持久性**

将消息标记为`persistent`并不能完全保证消息不会丢失。
尽管它告诉RabbitMQ将消息保存到磁盘，但是当RabbitMQ接收到消息并且还没有保存消息时，仍然有一个很短的时间窗口。
此外，RabbitMQ不会为每个消息执行fsync（2） - 它可能只是保存到缓存中，而不是写入磁盘。 持久性保证不强，但对于我们简单的任务队列已经足够了。
如果您需要更强大的保证，那么您可以使用[publisher confirms](https://www.rabbitmq.com/confirms.html)。

## 公平调度

您可能已经注意到调度仍然不能像我们想要的那样正常工作。 
例如在有两个worker的情况下，当所有奇数的信息都很复杂，偶数的信息很简单时，一个worker就会一直很忙，而另一个worker几乎不会做任何工作。
RabbitMQ是不知道这种情况，并将仍然均匀地发送消息。

发生这种情况是因为RabbitMQ只在消息进入队列时调度消息。 它没有考虑consumer未确认消息的数量。 它只是盲目地把第n条消息分发给第n个consumer。

![prefecth-count](http://www.rabbitmq.com/img/tutorials/prefetch-count.png)

为了解决这个问题，我们可以使用值为1的prefetch方法。这告诉RabbitMQ一次不能给一个worker多个消息。 或者换句话说，不要向worker
发送新消息，直到处理并确认了前一个消息。 相反，它将把它分派给下一个还不忙的worker。

```
ch.prefetch(1);
```

**关于队列大小的说明**

如果所有的worker都很忙，你的队列就
可能会填满。 你要关注一下这个问题，可以增加更多的worker，或者选择其他的策略。

## 把它放在一起

`new_task.js`的最终代码

```
#!/usr/bin/env node

var amqp = require('amqplib/callback_api');

amqp.connect('amqp://localhost', function(err, conn) {
  conn.createChannel(function(err, ch) {
    var q = 'task_queue';
    var msg = process.argv.slice(2).join(' ') || "Hello World!";

    ch.assertQueue(q, {durable: true});
    ch.sendToQueue(q, new Buffer(msg), {persistent: true});
    console.log(" [x] Sent '%s'", msg);
  });
  setTimeout(function() { conn.close(); process.exit(0) }, 500);
});
```

`worker.js`:

```
#!/usr/bin/env node

var amqp = require('amqplib/callback_api');

amqp.connect('amqp://localhost', function(err, conn) {
  conn.createChannel(function(err, ch) {
    var q = 'task_queue';

    ch.assertQueue(q, {durable: true});
    ch.prefetch(1);
    console.log(" [*] Waiting for messages in %s. To exit press CTRL+C", q);
    ch.consume(q, function(msg) {
      var secs = msg.content.toString().split('.').length - 1;

      console.log(" [x] Received %s", msg.content.toString());
      setTimeout(function() {
        console.log(" [x] Done");
        ch.ack(msg);
      }, secs * 1000);
    }, {noAck: false});
  });
});
```

使用ack和prefetch可以设置一个工作队列。 即使RabbitMQ重新启动，durable也能让任务继续存在。

有关`Channel`方法和消息属性的更多信息，可以浏览[amqplib文档](http://www.squaremobius.net/amqp.node/channel_api.html)。

现在我们可以继续阅读[教程3](https://github.com/OwenLittleWhite/ChineseRabbitMqTutorial/blob/master/Tutorial3.md)，并学习如何向许多consumer传递相同的消息。
