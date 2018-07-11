# Run Demo

1. 安装软件：确保安装了 JDK 8+ 和 Maven 3.2.5+
2. 编写配置：按照本页下面的接入说明配置`src/main/resource/kafka.properties`
3. 发送消息：sh run_producer.sh
4. 消费消息：sh run_consumer.sh


# Java SDK接入说明


#### 1、Maven 依赖配置

```java
//消息队列 Kafka服务端版本是0.10.0.0，客户端建议使用该版本
<dependency>
       <groupId>org.apache.kafka</groupId>
       <artifactId>kafka-clients</artifactId>
       <version>0.10.0.0</version>
</dependency>
```


#### 2.示例代码

4.1 准备配置文件kafka.properties，可以参考Demo中的进行修改

```
## 接入点，通过控制台获取
## 您在控制台获取的接入点
bootstrap.servers=kafka-cn-internet.aliyun.com:8080

## Topic，通过控制台创建
## 您在控制台创建的Topic
topic=alikafka-topic-demo

## Consumer Grouo，通过控制台创建
## 您在控制台创建的 Consumer Group
group.id=CID-consumer-group-demo
```
4.2 加载配置文件

```
private static Properties properties;

public synchronized static Properties getKafkaProperties() {
   if (null != properties) {
       return properties;
   }
   //获取配置文件kafka.properties的内容
   Properties kafkaProperties = new Properties();
   try {
       kafkaProperties.load(KafkaProducerDemo.class.getClassLoader().getResourceAsStream("kafka.properties"));
   } catch (Exception e) {
       //没加载到文件，程序要考虑退出
       e.printStackTrace();
   }
   properties = kafkaProperties;
   return kafkaProperties;
}
```

4.3 发送消息

```

//加载kafka.properties
Properties kafkaProperties =  JavaKafkaConfigurer.getKafkaProperties();

Properties props = new Properties();
//设置接入点，请通过控制台获取对应Topic的接入点
props.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, kafkaProperties.getProperty("bootstrap.servers"));
//Kafka消息的序列化方式
props.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, "org.apache.kafka.common.serialization.StringSerializer");
props.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, "org.apache.kafka.common.serialization.StringSerializer");
//请求的最长等待时间
props.put(ProducerConfig.MAX_BLOCK_MS_CONFIG, 30 * 1000);

//构造Producer对象，注意，该对象是线程安全的，一般来说，一个进程内一个Producer对象即可；
//如果想提高性能，可以多构造几个对象，但不要太多，最好不要超过5个
KafkaProducer<String, String> producer = new KafkaProducer<String, String>(props);

//构造一个Kafka消息
String topic = kafkaProperties.getProperty("topic"); //消息所属的Topic，请在控制台申请之后，填写在这里
String value = "this is the message's value"; //消息的内容

ProducerRecord<String, String>  kafkaMessage =  new ProducerRecord<String, String>(topic, value);

try {
  //发送消息，并获得一个Future对象
  Future<RecordMetadata> metadataFuture = producer.send(kafkaMessage);
  //同步获得Future对象的结果
  RecordMetadata recordMetadata = metadataFuture.get();
  System.out.println("Produce ok:" + recordMetadata.toString());
} catch (Exception e) {
  System.out.println("error occurred");
  e.printStackTrace();
}
```

4.4 消费消息

```java
//加载kafka.properties
Properties kafkaProperties =  JavaKafkaConfigurer.getKafkaProperties();

Properties props = new Properties();
//设置接入点，请通过控制台获取对应Topic的接入点
props.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, kafkaProperties.getProperty("bootstrap.servers"));
//两次poll之间的最大允许间隔
//请不要改得太大，服务器会掐掉空闲连接，不要超过30000
props.put(ConsumerConfig.SESSION_TIMEOUT_MS_CONFIG, 25000);
//每次poll的最大数量
//注意该值不要改得太大，如果poll太多数据，而不能在下次poll之前消费完，则会触发一次负载均衡，产生卡顿
props.put(ConsumerConfig.MAX_POLL_RECORDS_CONFIG, 30);
//消息的反序列化方式
props.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, "org.apache.kafka.common.serialization.StringDeserializer");
props.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, "org.apache.kafka.common.serialization.StringDeserializer");
//当前消费实例所属的消费组，请在控制台申请之后填写
//属于同一个组的消费实例，会负载消费消息
props.put(ConsumerConfig.GROUP_ID_CONFIG, kafkaProperties.getProperty("group.id"));
//构造消息对象，也即生成一个消费实例
KafkaConsumer<String, String> consumer = new org.apache.kafka.clients.consumer.KafkaConsumer<String, String>(props);
//设置消费组订阅的Topic，可以订阅多个
//如果GROUP_ID_CONFIG是一样，则订阅的Topic也建议设置成一样
List<String> subscribedTopics =  new ArrayList<String>();
//如果需要订阅多个Topic，则在这里add进去即可
//每个Topic需要先在控制台进行创建
subscribedTopics.add(kafkaProperties.getProperty("topic"));
consumer.subscribe(subscribedTopics);

//循环消费消息
while (true){
  try {
      ConsumerRecords<String, String> records = consumer.poll(1000);
      //必须在下次poll之前消费完这些数据, 且总耗时不得超过SESSION_TIMEOUT_MS_CONFIG
      //建议开一个单独的线程池来消费消息，然后异步返回结果
      for (ConsumerRecord<String, String> record : records) {
          System.out.println(String.format("Consume partition:%d offset:%d", record.partition(), record.offset()));
      }
  } catch (Exception e) {
      try {
          Thread.sleep(1000);
      } catch (Throwable ignore) {

      }
      e.printStackTrace();
  }
}
```


