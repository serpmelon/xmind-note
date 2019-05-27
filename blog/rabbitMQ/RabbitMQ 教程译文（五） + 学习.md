原文地址
[https://www.rabbitmq.com/tutorials/tutorial-five-java.html](https://www.rabbitmq.com/tutorials/tutorial-five-java.html "原文地址")

**Topics**

在之前的教程中，我们改进了日志系统，我们使用*direct* exchange代替*fanout* exchange，不使用广播的形式发送日志。而且我们可以挑选感兴趣的日志接收。

虽然使用*direct* exchange改进了我们的日志系统，但是还是有限制的，在多个条件下*direct*无法执行。

在我们的日志系统中，我们可能不只是根据日志严重级别来接收，可能还需要考虑日志的发送源。你可能从UNIX的工具*syslog*中知道这个概念，在这个工具中我们可以根据严重级别（info/warn/crit...）和设备（auth/cron/kern..）获取信息。

这会带给我们更多的灵活性，我们可能想接收*cron* 的严重错误日志和*kern*的全部日志。

为了在我们的日子系统中实现这个功能，我们需要学习一个更复杂的exchange *topic*。

**Topic exchange**

消息发送给*topic* exchange不能是一个任意的*routing_key*，而是一个单词列表，单词通过‘.’分割。单词可以随便写，但是一般会和信息相关。比如"stock.usd.nyse", "nyse.vmw", "quick.orange.rabbit"。*routing_key*的最大长度是255bytes，在范围内，单词数量没有限制。

*binding key*格式必须和*routing_key*一样。*topic* exchange的逻辑和*direct* exchange类似：一个带着特殊*routing_key*的信息将会发送给所有绑定了这个特殊*binding key*的队列。但是对于*binding key*有两个特殊的情况：

*（星号）可以代替一个完整的单词

\#（哈希）可以代替0个或多个单词

在下面的例子中会很容易解释这些
![](https://i.imgur.com/MIHa5v0.png)

在这个例子中我们会发送一些描述动物的信息。这些信息的*routing_key*有三个单词组成（两个分隔符'.'），第一个单词描述“速度”，第二个“颜色”，第三个“品种”： "<speed>.<colour>.<species>"。

我们创建三种绑定：队列Q1使用" *\*.orange.\** "绑定，队列Q2使用"*.*.rabbit" 和 "lazy.#"。

这些绑定关系可以解释为：

队列Q1对所有橘色的动物感兴趣；

队列Q2对所有的兔子和所有的懒得的动物感兴趣；

一个*routing key*为"quick.orange.rabbit"的信息将会发送给Q1和Q2。"lazy.orange.elephant"的信息也会发送这两个队列。另一方面，"quick.orange.fox"将会只发送给Q1，"lazy.brown.fox"只发送给Q2。"lazy.pink.rabbit" 将会发送给Q2，但是只发送一次，尽管它匹配了两个*binding key*。 "quick.brown.fox"没有匹配任何*binding key*，所以会被抛弃。

如果我们打破之前的约定，发送了一个或者四个单词的*routing key*的信息会怎么办？比如"orange" 或者 "quick.orange.male.rabbit"。这些*routing key*不会匹配任何*binding* 所以会被丢弃。

然而 "lazy.orange.male.rabbit"尽管包含四个单词，但是会匹配"lazy.#"，所以会发送给Q2。

**Topic Exchange**

*topic exchange*功能很强大，而且它可以表现的和其他*exchange*一样。当一个队列的*binding key*是"#"，它会接收所有的信息，而并不会关心*routing key*，就像*fanout exchange*一样。当特殊字符"#" "*"没有出现在*binding key*中，那么此时的*topic exchange*就像*direct exchange*一样。

**合体~**

我们将会在日志系统中使用*topic exchange*。开始之前，我们约定日志信息的*routing key*是两个单词： "<facility>.<severity>"。

EmitLogTopic.java代码如下
```
import com.rabbitmq.client.Channel;
import com.rabbitmq.client.Connection;
import com.rabbitmq.client.ConnectionFactory;

public class EmitLogTopic {

    private static final String EXCHANGE_NAME = "topic_logs";

    public static void main(String[] argv) throws Exception {
        ConnectionFactory factory = new ConnectionFactory();
        factory.setHost("localhost");
        try (Connection connection = factory.newConnection();
             Channel channel = connection.createChannel()) {

            channel.exchangeDeclare(EXCHANGE_NAME, "topic");

            String routingKey = getRouting(argv);
            String message = getMessage(argv);

            channel.basicPublish(EXCHANGE_NAME, routingKey, null, message.getBytes("UTF-8"));
            System.out.println(" [x] Sent '" + routingKey + "':'" + message + "'");
        }
    }

    private static String getRouting(String[] strings) {
        if (strings.length < 1)
            return "anonymous.info";
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
        if (length < startIndex) return "";
        StringBuilder words = new StringBuilder(strings[startIndex]);
        for (int i = startIndex + 1; i < length; i++) {
            words.append(delimiter).append(strings[i]);
        }
        return words.toString();
    }
}
```

ReceiveLogsTopic.java代码如下
```
import com.rabbitmq.client.Channel;
import com.rabbitmq.client.Connection;
import com.rabbitmq.client.ConnectionFactory;
import com.rabbitmq.client.DeliverCallback;

public class ReceiveLogsTopic {

    private static final String EXCHANGE_NAME = "topic_logs";

    public static void main(String[] argv) throws Exception {
        ConnectionFactory factory = new ConnectionFactory();
        factory.setHost("localhost");
        Connection connection = factory.newConnection();
        Channel channel = connection.createChannel();

        channel.exchangeDeclare(EXCHANGE_NAME, "topic");
        String queueName = channel.queueDeclare().getQueue();

        if (argv.length < 1) {
            System.err.println("Usage: ReceiveLogsTopic [binding_key]...");
            System.exit(1);
        }

        for (String bindingKey : argv) {
            channel.queueBind(queueName, EXCHANGE_NAME, bindingKey);
        }

        System.out.println(" [*] Waiting for messages. To exit press CTRL+C");

        DeliverCallback deliverCallback = (consumerTag, delivery) -> {
            String message = new String(delivery.getBody(), "UTF-8");
            System.out.println(" [x] Received '" + delivery.getEnvelope().getRoutingKey() + "':'" + message + "'");
        };
        channel.basicConsume(queueName, true, deliverCallback, consumerTag -> { });
    }
}
```
