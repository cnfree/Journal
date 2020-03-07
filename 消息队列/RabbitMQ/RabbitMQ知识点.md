# RabbitMQ 知识点

## 基本概念
* **Broker**：简单来说就是消息队列服务器实体。
* **Exchange**：消息交换机，它指定消息按什么规则，路由到哪个队列。
* **Queue**：消息队列载体，每个消息都会被投入到一个或多个队列。
* **Binding**：绑定，它的作用就是把exchange和queue按照路由规则绑定起来。
* **Routing Key**：路由关键字，exchange根据这个关键字进行消息投递。
* **vhost**：虚拟主机，一个broker里可以开设多个vhost，用作不同用户的权限分离。
* **producer**：消息生产者，就是投递消息的程序。
* **consumer**：消息消费者，就是接受消息的程序。
* **channel**：消息通道，在客户端的每个连接里，可建立多个channel，每个channel代表一个会话任务。

消息队列的使用过程大概如下：

 1. 客户端连接到消息队列服务器，打开一个channel。
 2. 客户端声明一个exchange，并设置相关属性。
 3. 客户端声明一个queue，并设置相关属性。
 4. 客户端使用routing key，在exchange和queue之间建立好绑定关系。
 5. 客户端投递消息到exchange。

## Exchanges, queues, and bindings
![exchange]

* **direct exchange**  routing key完全匹配才转发
* **fanout exchange** 不理会routing key,消息直接**广播到所有绑定的queue**
* **topic exchange**  对routing key模式匹配
* header exchange       路由消息并不依赖routing key而是去匹配AMQP消息的header部分

![route]

![direct]
Direct类型，将消息中的RoutingKey与该Exchange关联的所有Binding中的BindingKey进行比较，如果相等，则发送到该Binding对应的Queue中。

![fanout]
Fanout类型，将消息发送给所有与该  Exchange  定义过  Binding  的所有  Queues  中去，其实是一种广播行为。

![topic]
Topic类型，按照正则表达式，对RoutingKey与BindingKey进行匹配，如果匹配成功，则发送到对应的Queue中。

## 持久化
 * 创建queue和exchange默认情况下都是没有持久化的,节点重启之后queue和exchange就会消失
 * 需要特别指定queue和exchange的durable属性
 * 消息持久化同时要求exchange和queue也是持久化的
 * 消息的持久化需要在消息投递的时候设置delivery mode值为2
 * 持久化的代价就是性能损失,磁盘IO远远慢于RAM(使用SSD会显著提高消息持久化的性能)
 * 持久化会大大降低RabbitMQ每秒可处理的消息.两者的性能差距可能在10倍以上

## Connection 和 Channel
![connection&channel]

* 无论是生产者还是消费者，都需要和 RabbitMQ Broker 建立连接，这个连接就是一条 TCP 连接，也就是 Connection
* 一旦 TCP 连接建立起来，客户端紧接着可以创建一个 AMQP  Channel，每个 Channel 都会被指派一个唯一的 ID。
* Channel 是建立在 Connection 之上的虚拟连接，RabbitMQ 处理的每条 AMQP 指令都是通过 Channel 完成的。
* RabbitMQ 采用类似 NIO（Non-blocking I/O）的做法，选择 TCP 连接复用，不仅可以减少性能开销，同时也便于管理。
* 每个线程持有一个Channel，复用了 Connection 的 TCP 连接。同时 RabbitMQ 可以确保每个线程的私密性，就像拥有独立的连接一样。
* 复用单一的 Connection 可以在产生性能瓶颈，可能需要开辟多个 Connection。

## NIO
* NIO，也称非阻塞 I/O，包含三大核心部分：Channel（ Channel ）、Buffer（缓冲区）和 Selector（选择器）。
* NIO 基于 Channel 和 Buffer 进行操作，数据总是从 Channel 读取数据到缓冲区中，或者从缓冲区写入到 Channel 中。
* Selector 用于监听多个 Channel 的时间（比如连接打开，数据到达等）。因此，单线程可以监听多个数据的 Channel 。

## 消息的基本行为
> 如果一个 Queue 没被任何消费者订阅，那么这个 Queue 中的消息会被 Cache（缓存），当有消费者订阅时则会立即发送，当 Message 被消费者正确接收时，就会被从 Queue 中移除

## 消息发送确认
* 当消息无法路由到队列时，确认消息路由失败。
* 消息成功路由时，当需要发送的队列都发送成功后，进行确认消息，对于持久化队列意味着写入磁盘，对于镜像队列意味着所有镜像接收成功。
#### ConfirmCallback
* 通过实现 ConfirmCallback 接口，消息发送到 Broker 后触发回调，确认消息是否到达 Broker 服务器，也就是只确认是否正确到达 Exchange 中
    ```java
    @Component
    public class RabbitTemplateConfig implements RabbitTemplate.ConfirmCallback{
    
        @Autowired
        private RabbitTemplate rabbitTemplate;
    
        @PostConstruct
        public void init(){
            rabbitTemplate.setConfirmCallback(this);            //指定 ConfirmCallback
        }
    
        @Override
        public void confirm(CorrelationData correlationData, boolean ack, String cause) {
            System.out.println("消息唯一标识："+correlationData);
            System.out.println("确认结果："+ack);
            System.out.println("失败原因："+cause);
        }
    }
    ```
    ```yaml
     spring:
       rabbitmq:
         publisher-confirms: true  
    ```
#### ReturnCallback  
* 通过实现 ReturnCallback 接口，启动消息失败返回，比如路由不到队列时触发回调
    ```java
    @Component
    public class RabbitTemplateConfig implements RabbitTemplate.ReturnCallback{
    
        @Autowired
        private RabbitTemplate rabbitTemplate;
    
        @PostConstruct
        public void init(){
            rabbitTemplate.setReturnCallback(this);             //指定 ReturnCallback
        }
    
        @Override
        public void returnedMessage(Message message, int replyCode, String replyText, String exchange, String routingKey) {
            System.out.println("消息主体 message : "+message);
            System.out.println("消息主体 message : "+replyCode);
            System.out.println("描述："+replyText);
            System.out.println("消息使用的交换器 exchange : "+exchange);
            System.out.println("消息使用的路由键 routing : "+routingKey);
        }
    }
    ```
    ```yaml
    spring:
      rabbitmq:
        publisher-returns: true 
    ```

## 消息接收确认(Acknowledgment)
* 消息通过 ACK 确认是否被正确接收，每个 Message 都要被确认（acknowledged），可以手动去 ACK 或自动 ACK
* 自动确认会在消息发送给消费者后立即确认，但存在丢失消息的可能，如果消费端消费逻辑抛出异常，也就是消费端没有处理成功这条消息，那么就相当于丢失了消息
* 如果消息已经被处理，但后续代码抛出异常，使用 Spring 进行管理的话消费端业务逻辑会进行回滚，这也同样造成了实际意义的消息丢失
* 如果手动确认则当消费者调用 ack、nack、reject 几种方法进行确认，手动确认可以在业务失败后进行一些操作，如果消息未被 ACK 则会发送到下一个消费者
* 如果某个服务忘记 ACK 了，则 RabbitMQ 不会再发送数据给它，因为 RabbitMQ 认为该服务的处理能力有限
* ACK 机制还可以起到限流作用，比如在接收到某条消息时休眠几秒钟
* 消息确认模式有：
  * AcknowledgeMode.NONE：自动确认
  * AcknowledgeMode.AUTO：根据情况确认
  * AcknowledgeMode.MANUAL：手动确认
* 如果不配置Ack，默认是no_ack=true的模式，此时MQ只要确认消息发送成功，无须等待应答就会丢弃消息。

#### 确认消息（局部方法处理消息）
* 默认情况下消息消费者是自动 ack （确认）消息的，如果要手动 ack（确认）则需要修改确认模式为 manual
    ```yaml
    spring:
      rabbitmq:
        listener:
          simple:
            acknowledge-mode: manual
    ```
* 或在 RabbitListenerContainerFactory 中进行开启手动 ack
    ```java
    @Bean
    public RabbitListenerContainerFactory<?> rabbitListenerContainerFactory(ConnectionFactory connectionFactory){
        SimpleRabbitListenerContainerFactory factory = new SimpleRabbitListenerContainerFactory();
        factory.setConnectionFactory(connectionFactory);
        factory.setMessageConverter(new Jackson2JsonMessageConverter());
        factory.setAcknowledgeMode(AcknowledgeMode.MANUAL);             //开启手动 ack
        return factory;
    }
    ``` 
* 确认消息
    ```java
    @RabbitHandler
    public void processMessage2(String message,Channel channel,@Header(AmqpHeaders.DELIVERY_TAG) long tag) {
        System.out.println(message);
        try {
            channel.basicAck(tag,false);            // 确认消息
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
    ```
* 需要注意的 basicAck 方法需要传递两个参数
  * **deliveryTag（唯一标识 ID）**：当一个消费者向 RabbitMQ 注册后，会建立起一个 Channel ，RabbitMQ 会用 basic.deliver 方法向消费者推送消息，这个方法携带了一个 **delivery tag**， 它代表了 RabbitMQ 向该 Channel 投递的这条消息的**唯一标识 ID**，是一个单调递增的正整数，**delivery tag 的范围仅限于 Channel**
  * **multiple**：为了减少网络流量，**手动确认可以被批处理**，当该参数为 **true** 时，则可以**一次性确认** delivery_tag **小于等于**传入值的所有消息       
* 消费者获取消息时检查到头部包含 error 则 nack 消息
    ```java
    @RabbitHandler
    public void processMessage2(String message, Channel channel,@Headers Map<String,Object> map) {
        System.out.println(message);
        if (map.get("error")!= null){
            System.out.println("错误的消息");
            try {
                channel.basicNack((Long)map.get(AmqpHeaders.DELIVERY_TAG),false,true);      //否认消息
                return;
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
        try {
            channel.basicAck((Long)map.get(AmqpHeaders.DELIVERY_TAG),false);            //确认消息
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
    ```
* 也可以拒绝该消息，消息会被丢弃，不会重回队列
    ```java
    channel.basicReject((Long)map.get(AmqpHeaders.DELIVERY_TAG),false);        //拒绝消息
    ```
### 确认消息（全局处理消息）
* 自动确认涉及到一个问题就是如果在处理消息的时候抛出异常，消息处理失败，但是因为自动确认而导致 Rabbit 将该消息删除了，造成消息丢失
    ```java
    @Bean
    public SimpleMessageListenerContainer messageListenerContainer(ConnectionFactory connectionFactory){
        SimpleMessageListenerContainer container = new SimpleMessageListenerContainer();
        container.setConnectionFactory(connectionFactory);
        container.setQueueNames("consumer_queue");                 // 监听的队列
        container.setAcknowledgeMode(AcknowledgeMode.NONE);     // NONE 代表自动确认
        container.setMessageListener((MessageListener) message -> {         //消息监听处理
            System.out.println("====接收到消息=====");
            System.out.println(new String(message.getBody()));
            //相当于自己的一些消费逻辑抛错误
            throw new NullPointerException("consumer fail");
        });
        return container;
    }
    ```
* 手动确认消息
    ```java
    @Bean
    public SimpleMessageListenerContainer messageListenerContainer(ConnectionFactory connectionFactory){
        SimpleMessageListenerContainer container = new SimpleMessageListenerContainer();
        container.setConnectionFactory(connectionFactory);
        container.setQueueNames("consumer_queue");              // 监听的队列
        container.setAcknowledgeMode(AcknowledgeMode.MANUAL);        // 手动确认
        container.setMessageListener((ChannelAwareMessageListener) (message, channel) -> {      //消息处理
            System.out.println("====接收到消息=====");
            System.out.println(new String(message.getBody()));
            if(message.getMessageProperties().getHeaders().get("error") == null){
            channel.basicAck(message.getMessageProperties().getDeliveryTag(),false);
                System.out.println("消息已经确认");
            }else {
                //channel.basicNack(message.getMessageProperties().getDeliveryTag(),false,false);
                channel.basicReject(message.getMessageProperties().getDeliveryTag(),false);
                System.out.println("消息拒绝");
            }
    
        });
        return container;
    }
    ```
* AcknowledgeMode 除了 NONE 和 MANUAL 之外还有 AUTO ，它会根据方法的执行情况来决定是否确认还是拒绝（是否重新入queue）
  * 如果消息成功被消费（成功的意思是在消费的过程中没有抛出异常），则自动确认
  * 当抛出 AmqpRejectAndDontRequeueException 异常的时候，则消息会被拒绝，且 requeue = false（不重新入队列）
  * 当抛出 ImmediateAcknowledgeAmqpException 异常，则消费者会被确认
  * 其他的异常，则消息会被拒绝，且 requeue = true（如果此时只有一个消费者监听该队列，则有发生死循环的风险，多消费端也会造成资源的极大浪费，这个在开发过程中一定要避免的）。可以通过 setDefaultRequeueRejected（默认是true）去设置
    ```java
    @Bean
    public SimpleMessageListenerContainer messageListenerContainer(ConnectionFactory connectionFactory){
        SimpleMessageListenerContainer container = new SimpleMessageListenerContainer();
        container.setConnectionFactory(connectionFactory);
        container.setQueueNames("consumer_queue");              // 监听的队列
        container.setAcknowledgeMode(AcknowledgeMode.AUTO);     // 根据情况确认消息
        container.setMessageListener((MessageListener) (message) -> {
            System.out.println("====接收到消息=====");
            System.out.println(new String(message.getBody()));
            //抛出NullPointerException异常则重新入队列
            //throw new NullPointerException("消息消费失败");
            //当抛出的异常是AmqpRejectAndDontRequeueException异常的时候，则消息会被拒绝，且requeue=false
            //throw new AmqpRejectAndDontRequeueException("消息消费失败");
            //当抛出ImmediateAcknowledgeAmqpException异常，则消费者会被确认
            throw new ImmediateAcknowledgeAmqpException("消息消费失败");
        });
        return container;
    }
    ```
    
## ACK可能遇到的问题
  1. Unacked消息堆积    
     * 如果MQ没得到ack响应，这些消息会堆积在Unacked消息里，不会抛弃，直至客户端断开重连时，才变回ready；
     * 如果Consumer客户端不断开连接，这些Unacked消息，永远不会变回ready状态，Unacked消息多了，占用内存越来越大，会导致MQ响应越来越慢，甚至崩溃的问题。
     * 解决办法：及时ack消息
  2. 自动ack机制会导致消息丢失
     * MQ只要确认消息发送成功，无须等待应答就会丢弃消息，
     * 这会导致客户端还未处理完时，出异常或断电了，导致消息丢失的后果
     * 解决办法：手动ack
  3. 启用nack机制导致死循环；
     * 出错了，发nack，会通知MQ把消息塞回的队列头部（不是尾部）   
     * 出异常后，把消息塞回队列头部，下一步又消费这条会出异常的消息，又出错，塞回队列……进入了死循环了，当然新的消息不会消费，导致堆积
  4. 启用Qos和ack机制后，没有及时ack导致的队列堵塞
     * 开启了QoS，当RabbitMQ的队列达到Unacked消息上限时，不会再推送消息给Consumer，无法继续处理后续的消息。
  5. 重复消费
     * 在channel断开连接后,rabbitmq服务器会把还未ack的该条消息发给其它消费者,防止数据丢失，并且rabbit不会再给该消费者丢消息
     * 消费者处理过程中挂掉，处理了消息，但是返回时挂掉了，此时mq以为没处理，会让另一个消费者处理，这就处理了两次，需要考虑幂等。
     
 ## Reject 
 * 如果是消费者认为该消息本身有问题而不需要处理。可以使用basic.reject命令
 * reject命令里requeue设置成true的话，rabbit会将该消息丢给其他消费者，设置成false的话则不会再发送。
 * basicReject一次只能拒绝接收一个消息，而basicNack方法可以支持一次0个或多个消息的拒收，并且也可以设置是否requeue。    


## API
 * SimpleMessageListenerContainer 
   * 基础监听，作用就是队列的总监听，可以为其配置ack模式，异常处理类等   
 * org.springframework.amqp.support.converter.SimpleMessageConverter
   * 解析队列中的 message 转 obj 
 * org.springframework.amqp.rabbit.retry.MessageRecoverer
   * 这个接口，需要开发者自定义实现，其中的一个方法recover(Message message, Throwable cause)，在监听出错，也就是没有抓取的异常而是抛出的异常会触发该方法，在这个接口的实现中，将消息重新入队列
 * org.springframework.util.ErrorHandler
   * 这个接口也是在出现异常时候，会触发该方法  
   
## 消息可靠
* 持久化
  * exchange要持久化
  * queue要持久化
  * message要持久化
* 消息确认
  * 启动消费返回（@ReturnList注解，生产者就可以知道哪些消息没有发出去）
  * 生产者和Server（broker）之间的消息确认
  * 消费者和Server（broker）之间的消息确认       

## 死信队列
* 死信队列：DLX，dead-letter-exchange
* 当消息在一个队列中变成死信 (dead message) 之后，它能被重新publish到另一个Exchange，这个Exchange就是DLX
* 死信队列和普通队列唯一的区别就是它的routingkey是"#"。也就是说：只要你路由到我这个死信队列，我都接收。
    ```java
    channel.queueBind("dlx.queue","dlx.exchange","#");
    ```
* 消息变成死信有以下几种情况
  * 消息被拒绝(basic.reject / basic.nack)，并且requeue = false
  * 消息TTL过期
  * 队列达到最大长度
* 死信处理过程
  * DLX也是一个正常的Exchange，和一般的Exchange没有区别，它能在任何的队列上被指定，实际上就是设置某个队列的属性。
  * 当这个队列中有死信时，RabbitMQ就会自动的将这个消息重新发布到设置的Exchange上去，进而被路由到另一个队列。
  * 可以监听这个队列中的消息做相应的处理。
* 死信队列设置
  * 首先需要设置死信队列的exchange和queue，然后进行绑定
  * 需要有一个监听，去监听这个队列进行处理
  * 正常声明交换机、队列、绑定，然后在队列加上一个参数即可
  ```java
    arguments.put("x-dead-letter-exchange"，"dlx.exchange"); 
  ```
  * 这样消息在过期、requeue、 队列在达到最大长度时，消息就可以直接路由到死信队列！

## 复合死信队列
![dlx]

## 参考文章
  * [RabbitMQ中 exchange、route、queue的关系]





  
[connection&channel]: img/connection&channel.png
[dlx]: img/dlx.png
[exchange]:img/exchange.png
[direct]:img/direct.png
[fanout]:img/fanout.png
[topic]:img/topic.png
[route]:img/route.png
[RabbitMQ中 exchange、route、queue的关系]: https://www.cnblogs.com/cnndevelop/p/9879288.html