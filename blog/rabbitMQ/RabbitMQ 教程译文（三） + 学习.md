[原文地址](https://www.rabbitmq.com/tutorials/tutorial-three-java.html)
除了特殊声明，以下所有图片皆来自教程原文

**发布订阅**
在之前的教程中，我们创建了一个工作队列。在例子中我们假设，每一个任务会发送给特定的一个消费者。在本章节中，我们要做完全不同的事：我们将会发送信息给多个消费者，这就是发布订阅模式。

为了描述这个模式，我们将会创建一个简单的日志系统。这个系统包含两部分内容，一是发出日志信息，二是接收并打印日志信息。

在我们的系统中，每一个运行的接收日志的程序都会接收到日志信息。这种情况下我们可能使用一个接收者将日志直接存储到硬盘，使用另一个接收者用于屏幕展示。

总的来说，日志信息将会广播给所有的接收者。

**Exchanges**
在之前的教程中，我们通过一个队列来发送和接收信息。现在是时候介绍Rabbit中的完整信息模型了
我们先快速回忆下之前的教程内容
1、生产者就是一个发送信息的用户应用
2、队列就是存储信息的缓存
3、消费者就是一个接收信息的用户应用

在RabbitMQ中，消息模型的核心思想就是：生产者不会直接给队列发送信息。甚至大部分情况是生产者根本不知道队列的存在。

生产者只可以发送信息到一个*exchange*。*exchange*其实很简单，一方面接收生产者发送的信息，一方面把信息加入到队列中。*exchange*必须知道怎么处理它接收到的信息。是要把它发送到指定队列么？还是发送到很多队列？或者是不是应该丢弃。这些处理规则通过*exchange*类型来定义。
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190415091615488.png)
下面是一些有效的*exchange*类型：*direct*、*topic*、*headers*和*fanout*。我们重点看下最后一个类型，*fanout*。现在我们来创建这个类型的*exchange*，我们叫它“logs”。

```
channel.exchangeDeclare("logs", "fanout");
```
*fanout*类型的*exchange*非常简单，你可能已经根据名称猜出它的用途了，该类型的*exchange*会广播所有它接收到的信息给它知道的所有队列。这正是我们日志系统需要的。

*展示Exchanges*
你可以通过*rabbitmqctl*命令在服务端展示所有的*exchange*

```
sudo rabbitmqctl list_exchanges
```
在这个列表中，可能会有很多“amq.*”的*exchange*和一些默认的*exchange*，这些都是默认创建的*exchange*，目前你还不会用到。

*未命名的exchange*
在之前的教程中，我们一点都不知道*exchange*的存在，但是我们还是向队列发送了信息。这是因为我们之前使用了默认的*exchange*，我们是通过空字符串来定义的*exchange*。回忆下我们之前是怎么发送信息的

```
channel.basicPublish("", "hello", null, message.getBytes());
```
这个方法的第一个参数就是*exchange*的名称。空字符串表示这是一个默认或者无名的*exchange*。如果“routingKey”存在的话，信息会通过该值发送到指定队列。

现在我们使用一个命名的*exchange*替换之前的

```
channel.basicPublish( "logs", "", null, message.getBytes());
```

**Temporary Queue临时队列**
在我们之前的教程中，我们都会给队列一个名称（“hello”、“task_queue”）。对于我们来说给一个队列命名是非常重要的，这样我们就可以指定消费者消费哪个队列。所以当你需要在生产者和消费者见共享队列信息，给队列命名是非常重要的。

但是，这些对我们日志来说是不重要的。我们想获取所有的日志信息，而不是它的一部分。我们对于新的日志感兴趣而不是旧的日志。为了解决这个问题，我们需要做两件事。

首先，无论何时我们连接到Rabbit，我们需要一个全新的空的队列。为了实现这个我们需要创建一个有随机名称的队列，或者更好的情况是服务器给我们队列一个随机名称。

其次，当消费者断开连接，队列要自动删除。

在java实现中，我们提供了一个无参方法*queueDeclare()*，该方法会创建一个非持久，专一的（exclusive），自动删除的队列，该队列的名称是服务端生成的。

```
String queueName = channel.queueDeclare().getQueue();
```
你可以在[文档](https://www.rabbitmq.com/queues.html)中学习更多关于*exclusive*标识和其他参数的知识。

在上述代码中，“queueName” 是一个随机生成的名称，它可能像这样“amq.gen-JzTY20BRgKO-HjmUJj0wLg”。

**Bindings绑定**
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190415095030615.png)
我们已经创建了一个*fanout*类型的*exchange*和一个队列，现在我们要让*exchange*发送信息到队列中。*exchange*和队列中间的关系我们叫做绑定（Binding）。

```
channel.queueBind(queueName, "logs", "");
```
从现在开始，“logs”*exchange*将会发送信息到队列中。

**Putting it all together 合体~**
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190415095405122.png)
发送信息的生产者程序和之前教程中的看起来没什么不一样。最大的改变就是，现在我们发送信息到“logs”*exchange*，而不是之前的无名*exchange*。在发送过程中我们需要提供*routingKey*，但是它对于*fanout*类型的*exchange*是无效的。下面就是全部的生产者程序“EmitLog.java”

```
public class EmitLog {

  private static final String EXCHANGE_NAME = "logs";

  public static void main(String[] argv) throws Exception {
    ConnectionFactory factory = new ConnectionFactory();
    factory.setHost("localhost");
    try (Connection connection = factory.newConnection();
         Channel channel = connection.createChannel()) {
        channel.exchangeDeclare(EXCHANGE_NAME, "fanout");

        String message = argv.length < 1 ? "info: Hello World!" :
                            String.join(" ", argv);

        channel.basicPublish(EXCHANGE_NAME, "", null, message.getBytes("UTF-8"));
        System.out.println(" [x] Sent '" + message + "'");
    }
  }
}
```
正如你所见，我们在创建连接之后声明了一个*exchange*，这一步很有必要，它避免了发送信息到一个不存在的*exchange*中。

如果没有队列绑定到*exchange*，那么*exchange*中的信息将会丢失，但是这些对我们日志系统没什么影响；如果还没有消费者监听队列，那我们可以安全的丢弃这些信息。

“ReceiveLog.java”的代码

```
import com.rabbitmq.client.Channel;
import com.rabbitmq.client.Connection;
import com.rabbitmq.client.ConnectionFactory;
import com.rabbitmq.client.DeliverCallback;

public class ReceiveLogs {
  private static final String EXCHANGE_NAME = "logs";

  public static void main(String[] argv) throws Exception {
    ConnectionFactory factory = new ConnectionFactory();
    factory.setHost("localhost");
    Connection connection = factory.newConnection();
    Channel channel = connection.createChannel();

    channel.exchangeDeclare(EXCHANGE_NAME, "fanout");
    String queueName = channel.queueDeclare().getQueue();
    channel.queueBind(queueName, EXCHANGE_NAME, "");

    System.out.println(" [*] Waiting for messages. To exit press CTRL+C");

    DeliverCallback deliverCallback = (consumerTag, delivery) -> {
        String message = new String(delivery.getBody(), "UTF-8");
        System.out.println(" [x] Received '" + message + "'");
    };
    channel.basicConsume(queueName, true, deliverCallback, consumerTag -> { });
  }
}
```
后面就是运行查看结果

打完收工~~
