[原文地址](https://www.rabbitmq.com/tutorials/tutorial-four-java.html)
除了特殊声明，以下所有图片皆来自教程原文

**Routing路由**
在之前的章节中，我们创建了一个简单的日志系统。我们广播日志给多个接收者。

在本章节中，我们会给这个日志系统增加一个特色功能：我们将会实现日志信息的订阅功能。比如，我们可能会将错误日志直接存储到本地硬盘，但是还会在控制台打印显示所有的日志信息。

**Bindings绑定**
在之前的例子中，我们已经创建了一个*binding*，如下：

```
channel.queueBind(queueName, EXCHANGE_NAME, "");
```
*binding*维持了*exchange*和队列之间的关系。简单的说就是：队列对这个*exchange*的信息感兴趣。

*binding*会接受额外的一个参数*routingKey*。为了避免和*basic_publish*中的参数混淆，我们叫它*binding key*。下面是使用该参数构建*binding*的代码

```
channel.queueBind(queueName, EXCHANGE_NAME, "black");
```
这句话的意思就是*binding key*会依赖于*exchange*的类型，对于*fanout*类型的*exchange*来说，它会忽略掉这个参数。

**Direct exchange**
我们之前章节中的日志系统会广播所有日志信息给所有的消费者。我们将会扩展它的功能，允许通过日志的严重程度过滤信息。比如，我们可能想让一个程序将错误日志信息写到硬盘，但是不希望将警告或者普通信息写到硬盘来浪费空间。

我们之前使用的*fanout*类型的*exchange*只会无脑广播，缺少灵活性。

我们将会使用*direct*类型的*exchange*来代替。*direct*的路由算法非常简单：信息会发送到*binding key* 和信息的*routing key*匹配的队列。

下图展示了发送过程：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190420175056668.PNG)
如图所示，*exchange* “X”绑定了两个队列，第一个队列通过*binding key*“orange”绑定，第二个队列有两个*binding*，一个是“black”一个是“green”。

在这个系统中，*routing key*为“orange”的信息将会被发送到Q1队列，*routing key*为“black”和“green”的信息将会被发送到Q2队列，其他的信息将会被丢弃。

**Multiple bindings 多次绑定**
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190420202126830.PNG)
我们可以使用同一个*binding key*绑定多个队列。在我们的例子中，我们可以在“X”和Q1队列之间加一个*binding key* “black”，在这种情况下*direct*类型的*exchange*表现的就和*fanout*类型的一样了，也会广播所有的信息给所有的消费者Q1 ,Q2。

**发送日志**
我们将会在我们的日志系统中使用这个模型。替换之前的*fanout*类型的*exchange*为*direct*类型。我们会使用日志严重程度作为*binding key*。这样的话，消费者就会选择他们感兴趣的信息接收。现在我们先看看如何发送日志。
首先我们需要创建一个*exchange*。

```
channel.exchangeDeclare(EXCHANGE_NAME, "direct");
```
然后准备发送信息

```
channel.basicPublish(EXCHANGE_NAME, severity, null, message.getBytes());
```
为了简化操作，我们假设日志程度分为：错误、警告、普通信息。

**Subscribing订阅**
接收信息的程序和之前的教程基本一样，唯一的不同是我们需要根据我们的兴趣创建不同的*binding*。

```
String queueName = channel.queueDeclare().getQueue();

for(String severity : argv){
  channel.queueBind(queueName, EXCHANGE_NAME, severity);
}
```
**合体~~**
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190420205119253.PNG)
*EmitLogDirect.java*完整代码如下：

```
import com.rabbitmq.client.BuiltinExchangeType;
import com.rabbitmq.client.Channel;
import com.rabbitmq.client.Connection;
import com.rabbitmq.client.ConnectionFactory;

public class EmitLogDirect {

    private static final String EXCHANGE_NAME = "direct_logs";

    public static void main(String[] argv) throws Exception {
        ConnectionFactory factory = new ConnectionFactory();
        factory.setHost("localhost");
        try (Connection connection = factory.newConnection();
             Channel channel = connection.createChannel()) {
            channel.exchangeDeclare(EXCHANGE_NAME, BuiltinExchangeType.DIRECT);

            String severity = getSeverity(argv);
            String message = getMessage(argv);

            channel.basicPublish(EXCHANGE_NAME, severity, null, message.getBytes("UTF-8"));
            System.out.println(" [x] Sent '" + severity + "':'" + message + "'");
        }
    }

    private static String getSeverity(String[] strings) {
        if (strings.length < 1)
            return "info";
        return strings[0];
    }

    private static String getMessage(String[] strings) {
        if (strings.length < 2)
            return "Hello World!";
        return joinStrings(strings, " ", 1);
    }

    private static String joinStrings(String[] strings, String delimiter, int startIndex) {
        int length = strings.length;
        if (length == 0) return "";
        if (length <= startIndex) return "";
        StringBuilder words = new StringBuilder(strings[startIndex]);
        for (int i = startIndex + 1; i < length; i++) {
            words.append(delimiter).append(strings[i]);
        }
        return words.toString();
    }
}
```
*ReceiveLogsDirect.java*完整代码如下

```
import com.rabbitmq.client.*;

public class ReceiveLogsDirect {

  private static final String EXCHANGE_NAME = "direct_logs";

  public static void main(String[] argv) throws Exception {
    ConnectionFactory factory = new ConnectionFactory();
    factory.setHost("localhost");
    Connection connection = factory.newConnection();
    Channel channel = connection.createChannel();

    channel.exchangeDeclare(EXCHANGE_NAME, "direct");
    String queueName = channel.queueDeclare().getQueue();

    if (argv.length < 1) {
        System.err.println("Usage: ReceiveLogsDirect [info] [warning] [error]");
        System.exit(1);
    }

    for (String severity : argv) {
        channel.queueBind(queueName, EXCHANGE_NAME, severity);
    }
    System.out.println(" [*] Waiting for messages. To exit press CTRL+C");

    DeliverCallback deliverCallback = (consumerTag, delivery) -> {
        String message = new String(delivery.getBody(), "UTF-8");
        System.out.println(" [x] Received '" +
            delivery.getEnvelope().getRoutingKey() + "':'" + message + "'");
    };
    channel.basicConsume(queueName, true, deliverCallback, consumerTag -> { });
  }
}
```

收工~~
