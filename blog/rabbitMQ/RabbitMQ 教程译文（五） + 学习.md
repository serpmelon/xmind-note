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

