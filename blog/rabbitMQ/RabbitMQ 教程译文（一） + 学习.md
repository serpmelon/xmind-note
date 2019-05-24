[原文地址](https://www.rabbitmq.com/tutorials/tutorial-one-java.html)
以下图片除非特殊说明，均来自RabbitMQ官网教程。

**介绍**
RabbitMQ是一个信息代理工具：它可以用来接收和传递信息。你可以把它想象成一个邮局，当你需要邮寄信件的时候，你只需要将信件放到邮箱里，信件就会由邮递员交到目的地。在这里，RabbitMQ充当了邮局、邮箱和邮递员的角色。

RabbitMQ与邮局的最大区别就是，它不传递纸质信件，它传递二进制数据。

下面是RabbitMQ使用到的一些术语：
*Producing 生产*  相当于送信，一个发送信息的程序就是一个*Producer生产者*
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190411161128986.png)
队列（queue）就是RabbitMQ中的邮箱。虽然信息在应用程序和RabbitMQ之间穿跌，但是信息只能存储在队列中。队列本质上是一个大块的信息缓存，它只受主机的内存和硬盘限制。多个生产者可以发送信息到一个队列，多个消费者也可以从一个队列中获取信息。下图我们表示一个队列：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190411161654806.png)
*Consuming消费*就是接受消息。一个消费者就是一个等待接受信息的程序。
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190411161905920.png)
大家需要注意，上述提到的生产者、消费者和代理不一定要在一个主机上，通常情况下他们都在不同的主机上。一个应用既可以是生产者也可以是消费者。

下面是RabbitMQ的helloworld代码，以java实现。
在这部分，我们会写两个程序，一个生产者，发送一条信息；一个消费者接受这条信息并且打印出来。我们会忽略一些细节，先完成这个例子程序。

下图中“P”是生产者，“C”是消费者，中间的盒子就是队列
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190411163751829.png)
**发送信息**
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190411163930431.png)
我们会调用发送者“Send”和信息接受者“Recv”。发送者会连接RabbitMQ，并发送一条信息，然后退出。
在“Send.java”中，我们依赖一些类

```
import com.rabbitmq.client.Channel;
import com.rabbitmq.client.Connection;
import com.rabbitmq.client.ConnectionFactory;
```
创建类然后命名队列

```
public class Send {
  private final static String QUEUE_NAME = "hello";
  public static void main(String[] argv) throws Exception {
      ...
  }
}   
```
然后创建一个和服务的连接

```
ConnectionFactory factory = new ConnectionFactory();
factory.setHost("localhost");
try (Connection connection = factory.newConnection();
     Channel channel = connection.createChannel()) {
}
```
“Connection”是socket 连接的抽象，它帮我们处理协议、权限等细节。然后连接本地的信息代理，如果需要连接其他主机上的代理则需要特殊的名称或者IP地址。

接下来，我们创建一个“Channel”，大部分处理操作的API都在这里。因为“Connection”和“Channel”都实现了*java.io.Closeable*，所以我们可以使用*try-with-resources*表达式，这样我们就不用使用代码去关闭它们了。

为了发送信息，我们需要声明一个队列，然后我们发送一个信息到队列，所有这些操作都在*try-with-resources*表达式中。

```
channel.queueDeclare(QUEUE_NAME, false, false, false, null);
String message = "Hello World!";
channel.basicPublish("", QUEUE_NAME, null, message.getBytes());
System.out.println(" [x] Sent '" + message + "'");
```
队列的声明操作具有幂等性，只有在队列不存在的时候才会创建队列。代码中的信息是一个字节数组，你可以对它进行任何形式的编码。

下面是完整代码

```
import com.rabbitmq.client.Channel;
import com.rabbitmq.client.Connection;
import com.rabbitmq.client.ConnectionFactory;

public class Send {

    private final static String QUEUE_NAME = "hello";

    public static void main(String[] argv) throws Exception {
        ConnectionFactory factory = new ConnectionFactory();
        factory.setHost("localhost");
        try (Connection connection = factory.newConnection();
             Channel channel = connection.createChannel()) {
            channel.queueDeclare(QUEUE_NAME, false, false, false, null);
            String message = "Hello World!";
            channel.basicPublish("", QUEUE_NAME, null, message.getBytes("UTF-8"));
            System.out.println(" [x] Sent '" + message + "'");
        }
    }
}
```
**接收信息**
我们的消费者监听RabbitMQ接收信息。不同于上面说的发送者发送完一条信息后就退出了，这里的消费者会一直监听信息，然后打印信息。
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190411170443697.png)
“Recv.java”代码需要的引用和“Send.java”差不多

```
import com.rabbitmq.client.Channel;
import com.rabbitmq.client.Connection;
import com.rabbitmq.client.ConnectionFactory;
import com.rabbitmq.client.DeliverCallback;
```
额外的*DefaultConsumer*是实现了*Consumer*接口的实现类，我们将会使用它通过服务器缓存信息，然后推送给我们。（这里是什么意思？原文没有体现DefaultConsumer，难道说的是*DeliverCallback*么？）（原文*The extra DefaultConsumer is a class implementing the Consumer interface we'll use to buffer the messages pushed to us by the server.*）

与“Send.java”中一样创建“Recv.java”，创建“Connection”、“Channel”、声明队列。需要注意的是这里的队列要和发送者匹配。

```
public class Recv {

  private final static String QUEUE_NAME = "hello";

  public static void main(String[] argv) throws Exception {
    ConnectionFactory factory = new ConnectionFactory();
    factory.setHost("localhost");
    Connection connection = factory.newConnection();
    Channel channel = connection.createChannel();

    channel.queueDeclare(QUEUE_NAME, false, false, false, null);
    System.out.println(" [*] Waiting for messages. To exit press CTRL+C");

  }
}
```
你可能注意到这里我们声明了一个队列，这是因为，我们的消费者可能比生产者早创建，我们必须确保我们在消费一个队列的时候这个队列是存在的。

这里我们为什么没有使用*try-with-resources*表达式来自动关闭连接和通道？如果这样做了，我们会让程序简单的运行下去，关闭所有资源，然后退出。这就尴尬了，因为通常情况下，我们希望在消费者异步监听队列获取信息的时候程序不会结束。

接下来，我们将要告诉服务器从队列中发送信息给我们。因为信息的推送是异步的，所以我们以对象形式提供了一个回调方法，用来缓存信息，直到我们使用完。下面是我们的回调方法：

```
DeliverCallback deliverCallback = (consumerTag, delivery) -> {
    String message = new String(delivery.getBody(), "UTF-8");
    System.out.println(" [x] Received '" + message + "'");
};
channel.basicConsume(QUEUE_NAME, true, deliverCallback, consumerTag -> { });
```
下面是完整的代码

```
import com.rabbitmq.client.Channel;
import com.rabbitmq.client.Connection;
import com.rabbitmq.client.ConnectionFactory;
import com.rabbitmq.client.DeliverCallback;

public class Recv {

    private final static String QUEUE_NAME = "hello";

    public static void main(String[] argv) throws Exception {
        ConnectionFactory factory = new ConnectionFactory();
        factory.setHost("localhost");
        Connection connection = factory.newConnection();
        Channel channel = connection.createChannel();

        channel.queueDeclare(QUEUE_NAME, false, false, false, null);
        System.out.println(" [*] Waiting for messages. To exit press CTRL+C");

        DeliverCallback deliverCallback = (consumerTag, delivery) -> {
            String message = new String(delivery.getBody(), "UTF-8");
            System.out.println(" [x] Received '" + message + "'");
        };
        channel.basicConsume(QUEUE_NAME, true, deliverCallback, consumerTag -> { });
    }
}
```
**运行**
编译两个文件并运行（这里就不写具体的操作了）。


以上基本就是教程第一课的译文，大家有任何问题可以留言讨论；
