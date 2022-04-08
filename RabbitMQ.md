## RabbitMQ简介

MQ（Message Queue，消息队列）是一种应用系统之间的通信方法。是通过读写出入队列的消息来通信（RPC则是通过直接调用彼此来通信的）。    

AMQP，即Advanced Message Queuing Protocol，高级消息队列协议，是应用层协议的一个开放标准，为面向消息的中间件设计。消息中间件主要用于组件之间的解耦，消息的发送者无需知道消息使用者的存在，反之亦然。    

AMQP的主要特征是面向消息、队列、路由（包括点对点和发布/订阅）、可靠性、安全。     

RabbitMQ是一个开源的AMQP实现，服务器端用Erlang语言编写，支持多种客户端，如：Python、Ruby、.NET、Java、JMS、C、PHP、ActionScript、XMPP、STOMP等，支持AJAX。用于在分布式系统中存储转发消息，在易用性、扩展性、高可用性等方面表现不俗。
    
    
下面通过生产者代码来解释一下RabbitMQ中涉及到的概念。 

```java
public class MsgSender {
    private final static String QUEUE_NAME = "hello";

    public static void main(String[] args) throws IOException {
        /**
         * 创建连接连接到MabbitMQ
         */
        ConnectionFactory factory = new ConnectionFactory();
        // 设置MabbitMQ所在主机ip或者主机名
        factory.setHost("127.0.0.1");
        // 创建一个连接
        Connection connection = factory.newConnection();
        // 创建一个频道
        Channel channel = connection.createChannel();
        // 指定一个队列
        channel.queueDeclare(QUEUE_NAME, false, false, false, null);
        // 发送的消息
        String message = "hello world!龙轩";
        // 往队列中发出一条消息
        channel.basicPublish("", QUEUE_NAME, null, message.getBytes());
        System.out.println(" [x] Sent '" + message + "'");
        // 关闭频道和连接
        channel.close();
        connection.close();
    }
}
```

### ConnectionFactory、Connection、Channel    

 ConnectionFactory、Connection、Channel，这三个都是RabbitMQ对外提供的API中最基本的对象。不管是服务器端还是客户端都会首先创建这三类对象。
 
 ConnectionFactory为Connection的制造工厂。
 
 Connection是与RabbitMQ服务器的socket链接，它封装了socket协议及身份验证相关部分逻辑。
 
 Channel是我们与RabbitMQ打交道的最重要的一个接口，大部分的业务操作是在Channel这个接口中完成的，包括定义Queue、定义Exchange、绑定Queue与Exchange、发布消息等。

### Queue

Queue（队列）是RabbitMQ的内部对象，用于存储消息，用下图表示。

![123](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2017/2/21/49bb61c8b9e831308676035722329bcb~tplv-t2oaga2asx-zoom-in-crop-mark:1304:0:0:0.awebp)

RabbitMQ中的消息都只能存储在Queue中，生产者（下图中的P）生产消息并最终投递到Queue中，消费者（下图中的C）可以从Queue中获取消息并消费。

![123](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2017/2/21/3e12269d4cd28a48b43b363f99f36770~tplv-t2oaga2asx-zoom-in-crop-mark:1304:0:0:0.awebp)

队列是有Channel声明的，而且这个操作是幂等的。同名的队列多次声明也只会创建一次。我们发送消息就是想这个声明的队列里发送消息。

看一下消费者的代码：

```java
public class MsgReceiver {
    private final static String QUEUE_NAME = "hello";

    public static void main(String[] argv) throws IOException, InterruptedException {
        ConnectionFactory factory = new ConnectionFactory();
        factory.setHost("127.0.0.1");
        // 打开连接和创建频道，与发送端一样
        Connection connection = factory.newConnection();
        Channel channel = connection.createChannel();

        // 声明队列，主要为了防止消息接收者先运行此程序，队列还不存在时创建队列。
        channel.queueDeclare(QUEUE_NAME, false, false, false, null);
        // 创建队列消费者
        QueueingConsumer consumer = new QueueingConsumer(channel);
        System.out.println(" [*] Waiting for messages. To exit press CTRL+C");
        // 指定消费队列
        channel.basicConsume(QUEUE_NAME, true, consumer);
        while (true) {
            // nextDelivery是一个阻塞方法（内部实现其实是阻塞队列的take方法）
            QueueingConsumer.Delivery delivery = consumer.nextDelivery();
            String message = new String(delivery.getBody());
            System.out.println(" [x] Received '" + message + "'");
        }
    }
}
```

从上述代码中，我们可以看到ConnectionFactory、Connection、Channel这三个对象都还是会创建。而队列在消费者这里又声明了一遍。这是为了防止先启动消费者，当为消费者指定队列时，如果RabbitMQ服务器上未声明过队列，就会抛出IO异常。

### QueueingConsumer

队列消费者，用于监听队列中的消息。调用nextDelivery方法时，内部实现就是调用队列的take方法。该方法的作用：获取并移除此队列的头部，在元素变得可用之前一直等待（如果有必要）。说白了就是如果没有消息，就处于阻塞状态。

运行结果如下：（生产者、消费者谁先运行都可以)
![123](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2017/2/21/f9d4d99d2ebb1904ce9c6c8ebe51fd16~tplv-t2oaga2asx-zoom-in-crop-mark:1304:0:0:0.awebp)
