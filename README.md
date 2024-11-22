
## 开心一刻


上午一好哥们微信我哥们：哥们在干嘛，晚上出来吃饭我：就我俩吗哥们：对啊我：那多没意思，我叫俩女的出来哥们：好啊，哈哈哈晚上吃完饭到家后，我给哥们发消息我：今天吃的真开心，下次继续哥们：开心尼玛呢！下次再叫你老婆和你女儿来，我特么踢死你


![开心一刻](https://img2024.cnblogs.com/blog/747662/202411/747662-20241121193236681-1502497880.gif)
## 写在前面


正文开始之前了，我们先来正确审视下文章标题：`不依赖 Spring，你会如何自实现 RabbitMQ 消息的消费`，主要从两点来进行审视


1. 不依赖 Spring


作为一个 `Javaer`，关于 `Spring` 的重要性，我相信你们都非常清楚；回头看看你们开发的项目，是不是都是基于 Spring 的？如果不依赖 Spring，你们还能继续开发吗？不过话说回来，既然 Spring 能带来诸多便利，该用还得用，不要头铁，不要造低效轮子！



> 如果能造出比 Spring 优秀的轮子，那你应该造！


你们可能会说：不依赖 Spring 就不依赖嘛，我可以依赖 `Spring Boot` 噻；你们要是这么聊天，那就没法聊了


![这还咋聊](https://img2024.cnblogs.com/blog/747662/202411/747662-20241121194615412-748310975.gif)
Spring Boot 是不是基于 Spring 的？没有 Spring，Spring Boot 也是跑不起来的；`不依赖 Spring` 的言外之意就是不依赖 Spring 生态，当然也包括 Spring Boot


关于 `不依赖 Spring`，我就当你们审视清楚了哦
2. 依赖 RabbitMQ Java Client


与 RabbitMQ 服务端的交互，咱们就不要逞强去自实现了，老实点用官方提供的 Java Client 就好



```
<dependency>
    <groupId>com.rabbitmqgroupId>
    <artifactId>amqp-clientartifactId>
    <version>5.7.3version>
dependency>

```

注意 client 版本要与 RabbitMQ 版本兼容


所以文章标题就可以转换成



> 只依赖 RabbitMQ Java Client，不依赖 Spring，如何自实现 RabbitMQ 消息的消费


另外，我再带你们回顾下 RabbitMQ 的 Connection 和 Channel


1. Connection


`Connection` 是客户端与 RabbitMQ 服务器之间的一个 TCP 连接，它是进行通信的基础，允许客户端发送命令到 RabbitMQ 服务器并接收响应；Connection 是比较重的资源，不能随意创建与关闭，一般会以 `池` 的方式进行管理。每个 Connection 可以包含多个 Channel
2. Channel


`Channel` 是 `多路复用` 连接（Connection）中的一条独立的双向数据流通道。客户端与 RabbitMQ 服务端之间的大多数操作都是在 Channel 上进行的，而不是在 Connection 上直接进行。Channel 比 Connection 更轻量级，可以在同一连接中创建多个 Channel 以实现并发处理


Channel 与 Consumer 之间的关系是一对多的，具体来说，一个 Channel 可以绑定多个 Consumer，但每个 Consumer 只能绑定到一个


## 自实现


我们采取主干到枝叶的实现方式，逐步实现并完善 RabbitMQ 消息的消费


### 主流程


依赖 RabbitMQ Java Client 来消费 RabbitMQ 的消息，代码实现非常简单，网上一搜一大把



```
/**
 * @author: 青石路
 */
public class RabbitTest1 {

    private static final String QUEUE_NAME = "qsl.queue";

    public static void main(String[] args) throws Exception {
        ConnectionFactory factory = initConnectionFactory();
        Connection connection = factory.newConnection();
        Channel channel = connection.createChannel();
        channel.basicConsume(QUEUE_NAME, false, new QslConsumer(channel));
        System.out.println(Thread.currentThread().getName() + " 线程执行完毕，消费者：" + consumerTag + "已经就绪");
    }

    public static ConnectionFactory initConnectionFactory() {
        ConnectionFactory factory = new ConnectionFactory();
        factory.setHost("10.5.108.226");
        factory.setPort(5672);
        factory.setUsername("admin");
        factory.setPassword("admin");
        factory.setVirtualHost("/");
        factory.setConnectionTimeout(30000);
        return factory;
    }

    static class QslConsumer extends DefaultConsumer {

        QslConsumer(Channel channel) {
            super(channel);
        }

        @Override
        public void handleDelivery(String consumerTag, Envelope envelope, AMQP.BasicProperties properties, byte[] body) throws IOException {
            String message = new String(body, StandardCharsets.UTF_8);
            System.out.println(Thread.currentThread().getName()+ " 收到消息：" + message);
            this.getChannel().basicAck(envelope.getDeliveryTag(), false);
        }
    }
}

```

是不是很简单？



> 这里我得补充下，`exchange`、`queue` 没有在代码中声明，绑定关系也没有声明，是为了简化代码，因为文章标题是 `消费`；实际 `exchange`、`queue`、`binding` 这些已经存在，如下图所示
> 
> 
> ![exchange_queue声明](https://img2024.cnblogs.com/blog/747662/202411/747662-20241121193235708-868780884.png)


上述代码，我相信你们都能看懂，主要强调下 2 点


1. 消息是否自动 Ack


对应代码



```
channel.basicConsume(QUEUE_NAME, false, new QslConsumer(channel));

```

`basicConsume` 的第二个参数，其注释如下


![basicConsume_消息确认方式](https://img2024.cnblogs.com/blog/747662/202411/747662-20241121193236038-1441840800.png)
`autoAck` 为 `true` 表示消息在送达到 Consumer 后被 RabbitMQ 服务端确认，消息就会从队列中剔除了；`autoAck` 为 `false` 表示 Consumer 需要显式的向 RabbitMQ 服务端进行消息确认


因为我们将 `autoAck` 设置成了 `true`，所以 main 线程存活的时间内，5 个消息被送达到 main 线程后就被 RabbitMQ 服务端确认了，也就从队列中删除了
2. 手动确认


如果 Consumer 的 `autoAck` 设置的是 `false`，那么需要显示的进行消息确认



```
this.getChannel().basicAck(envelope.getDeliveryTag(), false);

```

否则 RabbitMQ 服务端会将消息一直保留在队列中，反复投递


执行 main 方法，控制台输出如下


![消费者就绪](https://img2024.cnblogs.com/blog/747662/202411/747662-20241121193235922-856734063.png)
我们去 RabbitMQ 控制台看下队列 `qsl.queue` 的消费者


![RabbitMQ消费者](https://img2024.cnblogs.com/blog/747662/202411/747662-20241121193235720-704884463.png)
`Consumer tag` 值是：`amq.ctag-PxjqYiujeCvyYlgtvMz9EQ`，与控制台的输出一致；我们手动往队列中发送一条消息


![发送消息](https://img2024.cnblogs.com/blog/747662/202411/747662-20241121193236148-1233373231.png)
控制台输出如下


![手动发送一条消息_控制台输出](https://img2024.cnblogs.com/blog/747662/202411/747662-20241121193236080-109054293.png)
自此，主流程就通了，此时已经实现 RabbitMQ 消息的消费


### 多消费者


单消费者肯定存在性能瓶颈，所以我们需要支持多消费者，并且是同个队列的多消费者；实现方式也很简单，只需要调整下 main 方法即可



```
public static void main(String[] args) throws Exception {
    ConnectionFactory factory = initConnectionFactory();
    Connection connection = factory.newConnection();
    Channel channel = connection.createChannel();
    QslConsumer qslConsumer = new QslConsumer(channel);
    String consumerTag1 = channel.basicConsume(QUEUE_NAME, false, qslConsumer);
    String consumerTag2 = channel.basicConsume(QUEUE_NAME, false, qslConsumer);
    String consumerTag3 = channel.basicConsume(QUEUE_NAME, false, qslConsumer);
    System.out.println(Thread.currentThread().getName() + " 线程执行完毕，消费者["
            + Arrays.asList(consumerTag1, consumerTag2, consumerTag3) + "]已经就绪");
}

```

执行main 方法后 Channel 与 Consumer 关系如下


![同个Channel_3个Consumer](https://img2024.cnblogs.com/blog/747662/202411/747662-20241121193236273-1416775155.png)
此时是同个 Channel 绑定了 3 个不同的 Consumer；当然也可以一对一绑定，main 方法调整如下



```
public static void main(String[] args) throws Exception {
    ConnectionFactory factory = initConnectionFactory();
    Connection connection = factory.newConnection();
    Channel channel1 = connection.createChannel();
    Channel channel2 = connection.createChannel();
    Channel channel3 = connection.createChannel();
    QslConsumer qslConsumer1 = new QslConsumer(channel1);
    QslConsumer qslConsumer2 = new QslConsumer(channel2);
    QslConsumer qslConsumer3 = new QslConsumer(channel3);
    String consumerTag1 = channel1.basicConsume(QUEUE_NAME, false, qslConsumer1);
    String consumerTag2 = channel2.basicConsume(QUEUE_NAME, false, qslConsumer2);
    String consumerTag3 = channel3.basicConsume(QUEUE_NAME, false, qslConsumer3);
    System.out.println(Thread.currentThread().getName() + " 线程执行完毕，消费者["
            + Arrays.asList(consumerTag1, consumerTag2, consumerTag3) + "]已经就绪");
}

```

执行 main 方法后 Channel 与 Consumer 关系如下


![3个Channel_3个Consumer](https://img2024.cnblogs.com/blog/747662/202411/747662-20241121193236091-193302285.png)
既然两种方式都可以实现多消费者，哪那种方式更好呢



> Channel 与 Consumer 一对一绑定更好！
> 
> 
> Channel 之间是线程安全的，同个 Channel 内非线程安全，所以同个 Channel 上同时处理多个消费者存在并发问题；另外 RabbitMQ 的消息确认机制是基于Channel 的，如果一个 Channel 上绑定多个消费者，那么消息确认会变得复杂，非常容易导致消息重复消费或丢失


也许你们会觉得 `一对一` 的绑定相较于 `一对多` 的绑定，存在资源浪费问题；确实是有这个问题，但我们要知道，Channel 是 Connection 中的一条独立的双向数据流通道，非常轻量级，相较于并发带来的一系列问题而言，这点小小的资源浪费可以忽略不计了


消费者数量能不能配置化呢，当然可以，调整非常简单



```
private static final int concurrency = 3;

public static void main(String[] args) throws Exception {
    ConnectionFactory factory = initConnectionFactory();
    Connection connection = factory.newConnection();
    for (int i = 0; i < concurrency; i++) {
        Channel channel = connection.createChannel();
        QslConsumer qslConsumer = new QslConsumer(channel);
        String consumerTag = channel.basicConsume(QUEUE_NAME, false, qslConsumer);
        System.out.println("消费者：" + consumerTag + " 已经就绪");
    }
}

```

`concurrency` 的值是从数据库读取，还是从配置文件中获取，就可以发挥你们的想象呢；如果依赖 Spring 的话，往往会用配置文件的方式注入进来


### 消费者预取数


队列 `qsl.queue` 没有消费者的情况下，我们往队列中添加 5 条消息：我是消息1 \~ 我是消息5，然后调整下 `handleDelivery`



```
@Override
public void handleDelivery(String consumerTag, Envelope envelope, AMQP.BasicProperties properties, byte[] body) throws IOException {
    String message = new String(body, StandardCharsets.UTF_8);
    System.out.println(consumerTag + " 收到消息：" + message);
    this.getChannel().basicAck(envelope.getDeliveryTag(), false);
    try {
        TimeUnit.SECONDS.sleep(1);
    } catch (InterruptedException e) {
        throw new RuntimeException(e);
    }
}

```

最后执行 main，控制台输出如下


![5条消息同个消费者消费](https://img2024.cnblogs.com/blog/747662/202411/747662-20241121193236197-1191589687.png)
大家注意看框住的那部分，5 条消息被同一个消费者给消费了！5 条消息为什么不是负载均衡到 3 个消费者呢？这是因为消费者的 `prefetch count`（即 `预取数`）没有设置



> prefetch count 是消费者在接收消息时，告诉 RabbitMQ 一次最多可以发送多少条消息给该消费者。默认情况下，这个值是 0，这意味着 RabbitMQ 会尽可能快地将消息分发给消费者，而不考虑消费者当前的处理能力


再回过头去看控制台的输出，是不是就能理解了？一旦某个消费者就绪，队列中的 5 条消息全部推给它了，后面就绪的 2 个消费者就没有消息可消费了；所以我们需要配置 `prefetch count` 以实现负载均衡，调整很简单



```
private static final String QUEUE_NAME = "qsl.queue";
private static final int concurrency = 3;
private static final int prefetchCount = 1;

public static void main(String[] args) throws Exception {
    ConnectionFactory factory = initConnectionFactory();
    Connection connection = factory.newConnection();
    for (int i = 0; i < concurrency; i++) {
        Channel channel = connection.createChannel();
        channel.basicQos(prefetchCount);
        QslConsumer qslConsumer = new QslConsumer(channel);
        String consumerTag = channel.basicConsume(QUEUE_NAME, false, qslConsumer);
        System.out.println("消费者：" + consumerTag + " 已经就绪");
    }
}

```

然后重复如上的测试，控制台输出如下


![负载均衡消费](https://img2024.cnblogs.com/blog/747662/202411/747662-20241121193236365-252644796.png)
是不是实现了我们想要的 `负载均衡` ？



> prefetch count 的设置需要根据实际的业务需求和消费者的处理能力进行调整；如果设置得太高，可能会导致内存占用过多；如果设置得太低，则可能无法充分利用消费者的处理能力


### 其他完善


限于篇幅，我就只列举几个还待完善的点



> 1. 目前只支持单个队列，需要支持多个队列
> 2. 目录消费逻辑单一固定，需要支持动态指定逻辑，不同的队列对应不同的消费逻辑
> 3. 消费者支持停止和重启
> 4. ...


关于这些点，我们下篇不见不散


## 总结


1. Connection、Channel、Consumer 之间的关系需要理清楚


Connection 是 TCP 连接；Channel 是 Connection 中的双向数据流通道；Channel 可以绑定多个 Consumer，但推荐一个 Channel 只绑定一个 Consumer


`IO 多路复用` 是网络编程中常用的技术，建议大家掌握
2. 基于 RabbitMQ Java Client 提供的 API，实现了消息消费、多消费者以及负载均衡


没有 Spring，我们照样可以很优雅的消费 RabbitMQ 的消息


 本博客参考[楚门加速器p](https://tianchuang88.com)。转载请注明出处！
