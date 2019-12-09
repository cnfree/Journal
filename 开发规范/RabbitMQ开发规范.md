# RabbitMQ开发规范

## 集群拆分
集群分两种：
* 本部门通讯集群：生产者和消费者为同一个部门所有
* 跨部门通讯集群：生产者和消费者不属于同一个部门

## 命名规范
**命名规范用于区分生产者和消费者分别属于哪个部门，采用何种方式进行路由**

* 本部门通讯集群命名规范：
  * exchange: 部门简称_exchangeName_路由方式_exchage
  * queue: sa_部门简称_queueName_路由方式_queue
  * sa: standalone 简写

* 跨部门通讯集群命名规范：
  * exchange: cluster_exchangeName_路由方式_exchage
  * queue: cluster_producer部门_consumer部门_queueName_路由方式_queue
  * 广播路由queue命名不包含consumer部门

## 广播路由
* **广播路由需要和各相关部门互相协调，确保各部门了解该消息同时会被其他部门处理**

## 消息过期
* 消息需要根据业务场景考虑是否需要设置TTL过期时间，比如短信验证码具有时效性，就需要设置消息的TTL

## 重试策略
* 如果是非业务异常，可以进行实时重试，超过重试失败次数，存储到其他地方（mongo），稍后再根据补偿策略进行重试
* 如果是业务异常，可以不重试，但最多重试1次，直接将错误消息存储到其它地方（mongo）, 例如订单号格式不正确，即使重试也是失败，直接进行人工干预
* 任何MQ均不能无限重试，必须设定重试次数

### 三种实现策略：
* 使用redis或者mongo等第三方存储当前重试次数。
* 在header中添加重试次数，并且使用channel.basicPublish() 方法重新将消息发送出去后将重试次数加1。
* 使用spring-rabbit中自带的retry功能。
    ```properties
    spring.rabbitmq.listener.simple.retry.max-attempts=5 最大重试次数
    spring.rabbitmq.listener.simple.retry.enabled=true 是否开启消费者重试（为false时关闭消费者重试，这时消费端代码异常会一直重复收到消息）
    spring.rabbitmq.listener.simple.retry.initial-interval=5000 重试间隔时间（单位毫秒）
    spring.rabbitmq.listener.simple.default-requeue-rejected=false 重试次数超过上面的设置之后是否丢弃（false不丢弃时需要写相应代码将该消息加入死信队列）
    ```

## 补偿策略
* 非业务异常需要考虑对消息进行补偿，比如定时读取mongo中处理失败的消息，补偿失败消息需要考虑业务的幂等性
* 业务异常消息，可以不重试，最多重试一次，再次失败后直接人工干预

## 补充资料

### 死信队列
当队列设置了死信队列后，出现dead letter的情况有：
* 消息或者队列的TTL过期
* 队列达到最大长度
* 消息被消费端拒绝（basic.reject or basic.nack）并且requeue=false