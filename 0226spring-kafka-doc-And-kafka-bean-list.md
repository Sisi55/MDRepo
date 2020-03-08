##  3. Introduction

- Spring for Apache Kafka에 대한 개괄적인 개요와 
- 가능한 빨리 시작하고 실행하는 데 도움이되는 기본 개념과 일부 코드 스니펫

### 3.1. Quick Tour for the Impatient

- Spring Kafka를 시작하는 5분짜리 투어

```cmd
// build.gradle
compile('org.springframework.kafka:spring-kafka')
```

> Spring Boot를 사용할 때 버전을 생략하면 Boot 버전과 호환되는 올바른 버전을 자동으로 가져온다

#### 3.1.1. Compatibility

####  3.1.2. A Very, Very Quick Example

- **plain Java** to send and receive a message: (봐야할까? )

```java
@Test
public void testAutoCommit() throws Exception {
    logger.info("Start auto");
    ContainerProperties containerProps = 
        new ContainerProperties("topic1","topic2");
    final CountDownLatch latch = new CountDownLatch(4);
    containerProps.setMessageListener(
        	new MessageListener<Integer,String>(){
        @Override
        public void onMessage(ConsumerRecord<Integer,String> message){
            logger.info("received: "+ message);
            latch.countDown();
        }
    });
    
    KafkaMessageListenerContainer<Integer,String> container = 
        createContainer(containerProps); //
    container.setBeanName("testAuto");
    container.start();
    Thread.sleep(1000);
    KakfaTemplate<Integer,String> template = createTemplate(); //
    template.setDefaultTopic("topic1");
    template.sendDefault(0,"foo");
    template.sendDefault(2,"bar");
    template.sendDefault(0,"baz");
    template.sendDefault(2,"qux");
    template.flush();
    assertTrue(latch.await(60), TimeUnit.SECONDS);
    container.stop();
    logger.info("Stop auto");
}
```



```java
private KafkaMessageListenerContainer<Integer,String> createContainer(
		ContainerProperties containerProps
){
    Map<String,Object> props = consumerProps();
    DefaultKafkaConsumerFactory<Integer,String> cf = 
        new DefaultKafkaCosumerFactory<Integer,String>(props);
    KafkaMessageListenerContainer<Integer,String> container = 
        new KafkaMessageListenerContainer<>(cf, containerProps);
    return container;
}

private KafkaTemplate<Integer,String> createTemplate(){
    Map<String,Object> senderProps = senderProps();
    ProducerFactory<Integer,String> pf = 
        new DefaultKafkaProducerFactory<>(senderProps);
    KafkaTemplate<Integer,String> template = new KafkaTemplate<>(pf);
    return template;
}

private Map<String,Object> consumerProps(){
    Map<String,Object> props = new HashMap<>();
    props.put(ConsumerConfig.BOOTSTRAP_SERVER_CONFIG,"localhost:9092");
    props.put(ConsumerConfig.GROUP_ID_CONFIG,group);
    props.put(ConsumerConfig.ENABLE_AUTO_CONNIT_CONFIG,true);
    props.put(ConsumerConfig.SESSION_TIMEOUT_MS_CONFIG,"15000");
    props.put(ConsumerConfig.KEY_DESERIALIZER_CASS_CONFIG,
              IntegerDeserializer.class);
    props.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG,
              StringDeserializer.class);
    return props;
}

private Map<String,Object> senderProps(){
    ...
}
```



#### 3.1.3. With Java Configuration

```java
@Autowired
private Listener listener;

@Autowired
private KafkaTemplate<Integer,String> template;

@Test
public void testSimple() throws Exception {
    template.send("annotated1", 0,"foo");
    template.flush();
    assertTrue(this.listener.latch1.await(10,TimeUnit.SECONDS));
}

@Configuration
@EnableKafka
public class Config {
    
    @Bean
    ConcurrentKafkaListenerContainerFactory<Integer,String> 
        	kakfaListenerContainerFactory(){
        
        ConcurrentKafkaListenerContainerFactory<Integer,String> factory
            = new ConcurrentKafkaListenerContainerFactory<>();
        factory.setConsumerFactory(consumerFactory());
        return factory;
    }
        
    @Bean
    public ConsumerFactory<Integer,String> consumerFactory(){
        return new DefaultKafkaConsumerFactory<>(consumerConfigs());
    }
    
    @Bean
    public Map<String,Object> consumerConfigs(){
        Map<String,Object> props = new HashMap<>();
        props.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, 
                  embeddedKafka.getBrokersAsString());
        //...
        return props;
    }
    
    @Bean
    public Listener listener(){
        return new Listener();
    }
    
    @Bean
    public ProducerFactory<Integer,String> producerConfigs(){
        Map<String,Object> props = new HashMap<>();
        props.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, 
                 embeddedKafka.getBrokersAsString());
        //...
        return props;
    }
    
    @Bean
    public KafkaTemplate<Integer,String> kafkaTemplate(){
        return new KafkaTemplate<Integer,String>(producerFactory());
    }
}
```



```java
public class Listener {
    private final CountDownLatch latch1 = new CountDownLatch(1);
    
    @KafkaListener(id="foo",topics="annotated1")
    public void listen1(String foo){
        this.latch1.countDown();
    }
}
```



#### 3.1.4. Even Quicker, with Spring Boot

```java
@SpringApplication
public class Application implements CommandLineRunner {
    
    public static Logger logger = 
        LoggerFactory.getLogger(Application.class);
    
    public static void main(String[] args){
        SpringApplication.run(Application.class, args).close();
    }
    
    @Autowired
    private KafkaTemplate<String,String> template;
    
    private final CountDownLatch latch = new CountDownLatch(3);
    
    @Override
    public void run(String... args) throws Exception {
        this.template.send("myTopic", "foo1");
        this.template.send("myTopic", "foo2");
        this.template.send("myTopic", "foo3");
        latch.await(60, TimeUnit.SECONDS);
        logger.info("All received");
    }
    
    @KafkaListener(topics="myTopic")
    public void listen(ConsumerRecord<?,?> cr) throws Exception {
        logger.info(cr.toString());
        latch.countDown();
    }
}
```



**Example 1. application.properties**

- 로컬 브로커를 사용할 때 필요한 유일한 속성 다음과 같다.

```properties
spring.kafka.consumer.group-id=foo 
# 그룹 관리를 사용하여 주제 파티션을 소비자에게 할당하기 때문에 필요
spring.kafka.consumer.auto-offset-reset=earliest
# 새 소비자 그룹이 우리가 보낸 메시지를 받도록 보장한다
# 전송 후 컨테이너가 시작될 수 있기 때문에

```



## 4. Reference

###  4.1. Using Spring for Apache Kafka

#### 4.1.1. Configuring Topics

- `KafkaAdmin` 빈을 application context 에 정의하면
- 자동으로 브로커에 토픽들을 추가한다
- 이것을 하려면, application context 의 각 토픽에 @Bean NewTopic 을 추가해야 한다.
- TopicBuilder 클래스를 이용하면 된다.
- 다음 예시를 보자.

```java
@Bean
public KafkaAdmin admin(){
    Map<String,Object> configs = new HashMap<>();
    configs.put(AdminClientConfig.BOOTSTRAP_SERVERS_CONFIG, ...);
    return new KafkaAdmin(configs);
}

@Bean
public NewTopic topic1(){
    return TopicBuilder.name("thing1")
        .partitions(10)
        .replicas(3)
        .compact() //
        .build();
}

@Bean
public NewTopic topic2(){
    return TopicBuilder.name("thing2")
        .partitions(10)
        .replicas(3)
        .config(TopicConfig.COMPRESSION_TYPE_CONFIG, "zstd") //
        .build();
}

@Bean
public NewTopic topic3(){
    return TopicBuilder.name("things3")
        .assignReplicas(0, Arrays.asList(0,1))
        .assignReplicas(1, Arrays.asList(1,2))
        .assignReplicas(2, Arrays.asList(2,0))
        .build();
}
```

> Spring Boot를 사용하는 경우 KafkaAdmin Bean이 자동으로 등록되므로 NewTopic @Bean 만 필요하다
>
> (그래서 KafkaAdmin 빈을 등록하지 않았구나 예제에서)
>
> (근데 로컬로 동작해서 서버 설정같은건 필요할듯?)



- 만약 브로커가 사용가능하지 않으면, 메시지는 로그되지만 context 는 계속해서 로드된다.
- admin 의 initializer() 함수를 통해 나중에 다시 시도하도록 코드로 작성가능하다. (브로커 사용 가능하게 다시 시도 ?)
- 이 상태를 fatal 하게 간주하고 싶으면, admin 의 fatalIfBrokerNotAvailable 속성을 true 로 설정하자.
- context 는 초기화에 실패할 것이다.



- 보다 고급 기능을 사용하려면 AdminClient를 직접 사용할 수 있다
- 예시를 보자

```java
@Autowired
private KAfkaAdmin admin;
//...
AdminClient client = AdminClient.create(admin.getConfig());
//...
client.close();
```



####  4.1.2. Sending Messages

- how to send messages

##### Using `KafkaTemplate`

- how to use `KafkaTemplate` to send messages

Overview

- KafkaTemplate 은 producer 를 wrap 한다
- 그리고 데이터를 카프카 토픽에 전송하기 위한 편리한 메소드를 제공한다
- `sendDefault(..)`, `send()`, `metrics()`, `partitionsFor(Topic)`, 
- `execute()`, `flush()`, `ProducerCallback` 등



- `sendDefault(..)` 를 사용하려면 디폴트 토픽이 template 에 제공되어야 한다.
- `metrics()`, `partitionsFor(토픽)` 함수는 Producer 에게 위임된다
- `execute()` 함수는 Producer 에 직접적인 접근할 수 있게 한다



- `template` 을 사용하려면
- `producer factory` 를 설정하고 template 생성자에 제공해야 한다.
- 예시를 보자.

```java
@Bean
public ProducerFactory<Integer,String> producerFactory(){
    return new DefaultKafkaProducerFactory<>(producerConfigs());
}

@Bean
public Map<String,Object> producerConfigs(){
    Map<String,Object> props = new HashMap<>();
    props.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, 
              "localhost:9092");
    props.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG,
             StringSerializer.class);
    props.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG,
             StringSerializer.class);
    return props;
}

@Bean
public KafkaTemplate<Integer,String> kafkaTemplate(){
    return new KafkaTemplate<Integer,String>(producerFactory());
}
```

> - 표준 <bean /> 정의를 사용하여 템플릿을 구성 할 수도 있다
> - template 를 사용하기 위해선, 함수를 호출해야 한다
> - 함수에 Message<?> 매개변수를 사용할 때,
> - 메시지 헤더에 다음 아이템을 담은 토픽,파티션,키 정보가 제공되어야 한다. .
> - `KafkaHeaders.TOPIC`
> - `KafkaHeaders.PARTITION_ID`
> - `KafkaHeaders.MESSAGE_KEY`
> - `KafkaHeaders.TIMESTAMP`
> - 메시지 payload 는 data 이다.



- 선택적으로 KafkaTemplate 를 구성할 수 있다.
- 전송 결과(성공/실패) 비동기 콜백을 받아오는 ProducerListener 로
- Future 가 완료되기를 기다리는 대신
- 다음 예시는 `interface ProducerListener ` 이다

```java
public interface ProducerListener<K,V> {
    void onSuccess(ProducerRecord<K,V> producerRecord,
                  RecordMetadata recordMetadata);
    
    void onError(ProducerRecord<K,V> producerRecord,
                Exception exception);
}
```

> 기본적으로 template 은 LogginProducer
>
> ```
> 
> ```
>
> Listener 로 구성된다. 오류를 기록하고, 전송이 성공하면 아무것도 하지 않는.



- send 함수는 ListenableFuture<SendResult> 를 반환한다.
- 전송 결과를 비동기적으로 받는 콜백을 등록할 수 있다.
- 예시는 어떻게 하는지 보여준다

```java
ListenableFuture<SendResult<Integer,String>> future = 
    template.send("something");

future.addCallback(
    	new ListenableFutureCallback<SendResult<Integer,String>>(){
    
    @Override
	public void onSuccess(SendResult<Integer,String> result){
        //...
    }
            
    @Override
	public void onFailure(Throwable ex){
        //...
    }
});
```

> - SendResult에는 ProducerRecord 및 RecordMetadata의 두 가지 속성이 있다
> - 송신 스레드가 결과를 기다리도록(block:동기) 하려면 future 객체의 get() 함수를 호출하면 된다
> - template 이 각 전송마다 flush() 를 호출할 수 있도록 template 생성자에 autoFlush 매개를 전달할 수 있다
> - 하지만 성능에 영향이 있다

**Examples**

-  examples of sending messages to Kafka

**Example 2. Non Blocking (Async)**

```java
public void sendToKafka(final MyOutputData data){
    
    final ProducerRecord<String,String> record = 
        createRecord(data);
    
    ListenableFuture<SendResult<Integer,String>> future = 
        template.send(record);
    future.addCallback(
        	new ListenableFutureCallback<SendResult<Integer,String>>(){
        
		@Override
		public void onSuccess(SendResult<Integer,String> result){
            handleSuccess(data);
        }
                
		@Override
		public void onFailure(Throwable ex){
            handleFailure(data,record,ex);
        }
    });    
}
```



**Blocking (Sync) **  

// pass

#####  Using `DefaultKafkaProducerFactory`

- ProducerFactory 는 producer 를 생성하는데 사용된다.
- 문서에서 권장되듯, 기본적으로 트랜잭션을 사용하지 않을 때, DefaultKafkaProducerFactory 는 singleton producer 를 생성한다. 모든 clients 에서 쓰이는.
- 하지만 template 에 flush() 를 호출하면, 같은 producer 를 사용하는 다른 스레드에 지연을 야기한다
- DefaultKafkaProducerFactory 의 producerPerThread 속성을 true 로 설정하면, factory 는 이 이슈를 막기위해 각 스레드마다 분리된 producer 를 생성한다.

> DefaultKafkaProducerFactory 의 producerPerThread 속성을 true 로 설정하면, 
>
> producer 가 더이상 필요하지 않을때 closeThreadBoundProducer() 를 꼭 호출해야 한다.
>
> 이것은 물리적으로 producer 를 닫는다. 그리고 ThreadLocal 로부터 제거한다.
>
> reset(), destry() 를 호출하는 것만으로는 producer 들을 clean up 하지 못한다



/

#####  Using `ReplyingKafkaTemplate`

// pass

> - 예시있다
> - The following Spring Boot application shows an example of how to use the feature: 
> - In this case, the following `@KafkaListener` application responds:

##### Aggregating Multiple Replies

// pass

#### 4.1.3. Receiving Messages

- 메시지를 수신할 수 있다.
- MessageListenerContainer 를 구성하고 message listener 를 제공하는 방법
- @KafkaListener 를 이용하는 방법

##### Message Listeners // 1

- message listener container 를 사용할 때, 데이터를 받기 위해 listener 를 제공해야 한다.
- message listener 에 대해 지원되는 8가지 인터페이스가 있다. (생략)
- 8개 인터페이스 언제 사용하는지도 나와있다

> Consumer 객체는 스레드 안전하지 않다. listener 를 호출한 스레드에서만 사용해야 한다.



##### Message Listener Containers

- 두 가지 MessageListenerContainer 구현이 제공된다
  - KafkaMessageListenerContainer
  - ConcurrentMessageListenerContainer
- KafkaMessageListenerContainer :  단일 스레드의 모든 주제 또는 파티션에서 모든 메시지를 수신한다
- ConcurrentMessageListenerContainer :  다중 스레드 소비를 제공하기 위해 하나 이상의 KafkaMessageListenerContainer 인스턴스에 위임한다



- 리스너 컨테이너에 RecordInterceptor를 추가 할 수 있다
- 리스너를 호출하기 전에 호출된다. 레코드의 검사 또는 수정을 허용한다.
- 인터셉터가 널을 리턴하면 리스너가 호출되지 않는다
- 리스너가 배치 리스너 인 경우 인터셉터가 호출되지 않는다



- CompositeRecordInterceptor를 사용하여 여러 인터셉터를 호출 할 수 있다
- 기본적으로 트랜잭션을 사용할 때, 트랜잭션이 시작된 후에 인터셉터가 호출된다
- 리스너 컨테이너의 interceptBeforeTx 속성을 설정할 수 있다. 트랜잭션이 시작되기 전에 인터셉터를 호출할 수 있다.



- Kafka는 이미 ConsumerInterceptor를 제공하므로, 배치 리스너에 인터셉터를 제공하지 않는다

**Using `KafkaMessageListenerContainer`**

`KafkaMessageListenerContainer` 의 생성자는 기본적으로

`ConsumerFactory<K,V>` 와 `ContainerProperties` 매개를 받는다.



생성자는 `ConsumerFactory` 와 토픽,파티션 정보를 갖는다.

+`ContainerProperties` 객체의 구성



`TopicPartitionOffset` 매개를 갖는 생성자는

`ConcurrentMessageListenerContainer` 가 사용한다. consumer 객체들이 사용하는.



ContainerPropertes 생성자 :

- 1
- 컨테이너에 어느 파티션을 사용할지 명시한다. consumer 의 assign() 메소드 이용하여.
- 선택적으로 초기 offset 을 설정하여.

- 2
- 토픽 list을 매개로 받으면, 카프카는 파티션을 group.id 에 따라 할당한다
- group 에 파티션을 분산한다
- 3
- 정규식 패턴을 사용한다. 토픽 선택에



- MessageListener 를 컨테이너에 배정하기 위해, 
- ContainerProps.setMessageListener 함수를 사용한다.
- 컨테이너를 생성할때.
- 다음 예시는 어떻게 하는건지 보여준다

```java
ContainerProperties containerProps = 
    new ContainerProperties("topic1","topic2");
containerProps.setMessageListener(
    	new MessageListener<Integer,String>(){
    //...
});

DefaultKafkaConsumerFactory<Integer,String> cf = 
    new DefaultKafkaConsumerFactory<>(consumerProps);
KafkaMessageListenerContainer<Integer,String> container = 
    new KafkaMessageListenerContainer<>(cf);
return container;
```

> ConsumerFactory 를 생성하여 Container 생성자에 전달하네



- `ContainerProperties` for more information about the various properties



- `logContainerConfig` 속성: true 이면, 요약,구성 속성을 info 로깅한다
- 기본적으로 토픽 offset commit 은 debug 모드로 로딩된다.
- commitLogLevel 속성 이용하면 로그 레벨 특정할 수 있다

`containerProperties.setCommitLogLevel(LogIfLevelEnabled.Level.INFO);`



- `missingTopicsFatal` 속성: 기본값 false.
- 설정된 topic 중 하나라도 브로커에없는 경우 컨테이너가 시작되지 않습니다
- 컨테이너가 주제 패턴 (정규식)을 수신하도록 구성된 경우에는 적용되지 않는다
- 이전에는 `container` 스레드가 `consumer.poll()` 안에서 루프했다. topic 이 나타나기를. 많은 메시지를 로깅하면서.



- `authorizationExceptionRetryInterval` 속성: 
- 컨테이너가 메시지들을 fetching 다시 시도하도록 한다. AuthorizationException getting 후에. KafkaConsumer 로부터.
- 언제 일어날 수 있냐면, 설정된 사용자가 거부할 때, 특정 토픽 읽기를.
- authorizationExceptionRetryInterval 는 애플리케이션을 돕는다. 복구할 수 있도록. 적절한 권한들을 부여받자마자.



> authorization 에러는 fatal 로 간주된다. 그래서 container 를 stop 하게 만든다.
>
> 기본적으로 interval 는 설정되지 않는다.



**Using `ConcurrentMessageListenerContainer`**

// pass

**Committing Offsets**

// pass

**Listener Container Auto Startup**

// pass



##### `@KafkaListener` Annotation // 2

- @KafkaListener 어노테이션은 빈 메소드를 listener container 의 listener 로 지정하는데 사용된다.

- 빈은 wrapped 된다. MessageingMessageListenerAdapter 에. 다양한 특징들로 설정된다. converter : 필요하면 data를 convert 하는. 메소드 파라미터로. match 할 수 있다. 

Record Listeners

- @KafkaListener 어노테이션은 제공한다. 간단 POJO listeners 메커니즘.
- how to use it

```java
public class Listener {
    
    @KafkaListener(id="foo", topics="myTopic", 
                   clientIdPrefix="myClientId")
    public void listen(String data){
        //...
    }
}
```



- 이 메커니즘은 요구한다.` @EnableKafka` 어노테이션. 어디에? @Configuration 클래스,  중 하나에? listener container factory 이것은 설정에 사용된다. ConcurrentMessageListenerContainer
- 기본적으로 KafkaListenerContainerFactory 빈 이름.

- how to use `ConcurrentMessageListenerContainer`:

```java
@Configuration
@EnableKafka
public class KafkaConfig {
    
    @Bean
    KafkaListenerContainerFactory<
        	ConCurrentMessageListenerContainer<Integer,String>> 
        	KafkaListenerContainerFactory(){
        
        ConcurrentKafkaListenerContainerFactory<Integer,String> factory 
            = new ConcurrentKafkaListenerContainerFactory<>();
        factory.setConsumerFactory(consumerFactory());
        factory.setConcurrency(3);
        factory.getConatainerProperties().setPollTimeout(3000); //
        return factory;
    }
    
    @Bean
    public ConsumerFactory<Integer,String> consumerFactory(){
        return new DefaultKafkaConsumerFactory<>(consumerConfigs());
    }
    
    @Bean
    public Map<String,Object> consumerConfigs(){
        Map<String,Object> props = new HashMap<>();
        props.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, embeddedKafka.getBrokersAsString());
        //...
        return props;
    }
}
```

> - container 속성을 설정하기 위해
> - factory 에 getContainerProperties() 함수를 사용해야 한다.
> - (get 하고 여기에 set 하라는 이야기겠지?)
> - 이것은 container 에 실제로 주입될 속성들의 template 으로 사용된다.



- 어노테이션에 의해 생성되는 소비자들에 client.id 속성을 설정할 수 있다.
- clientIdPrefix 는 접두사이다. 이 정수는 container 숫자. 병렬적으로 사용될. 



- container factory 의 병렬성,자동시작 속성을 재정의할 수 있다. 그것의 어노테이션 속성으로.
- 이 속성은 단순한 값.
- how to do so

```java
@KafkaListener(id="myListener", topics="myTopic",
              autoStartup="${listen.auto.start:true}",
              concurrency="${listen.concurrency:3}")
public void listen(String data){
    //...
}
```



- POJO 리스너들로 설정할 수도 있다.
- 명시적 토픽,파티션과 함께
- how to do so

```java
//...
```



- how to use a different container factory

- > @KafkaListener 의 containerFactory 속성

```java
//...
```



- how to use the headers

- > @KafkaListenr 의 메소드 매개에 @Header(..) 

```java
//...
```



**Batch listeners**

- @KafkaListener 메소드를 설정할 수 있다.
- 전체 batch consumer 레코드들을 받기 위해.
- consumer poll 로부터.



- batch listener 를 생성하기 위해 listener container factory 를 설정하려면
- batchListener 속성을 설정할 수 있다.
- 다음과 같이.

```java
//... @Bean : batchFactory : KafkaListenerContainerFactory
factory.setBatchListener(true);
```



- how to receive a list of payloads

```java
@KafkaListener(id="list", topics="myTopic", 
               containerFactory=batchFactory)
public void listen(List<String> list){
    //...
}
```



-  in headers that parallel the payloads : 병렬로 한 갈래씩 뽑을 수 있다

- how to use the headers

```java
// 매개에 @Header(MESSAGE_KEY), @Header(PARTITION_ID), ...
```



- can receive a List of Message<?> objects
- how to do so

```Java
// List<Message<?>> 매개
// 수동적 커밋, Consumer<?,?> : Acknowledgment
```



+ `List<ConsumerRecord<Integer, String>>` + example



- can receive the complete `ConsumerRecords` object 
- returned by the `poll()` method
- listener 가 추가적인 메소드로 접슨할 수 있게 한다. 어디에?
- partitions() : TopicPartition, records(TopicPartition) : selective records
- how to do so

```java
@KafkaListener
public void pollResults(ConsumerRecords<?,?> records){
    //...
}    
```



- container factory 에 RecordFilterStrategy 설정이 있다면
- ConsumerRecords<?,?> 리스터를 무시한다. warn 로그 메시지와 함께.
- <List <? >> 형식의 리스너를 사용하는 경우 배치 리스너로만 레코드를 필터링 할 수 있습니다.



**Annotation Properties**

+ example : @KafkaListener 속성

+ ex:  @Bean : Listener 를 이렇게 사용할 수 있다

  ```java
  @KafkaListener(topics = "#{__listener.topic}",        
                 groupId = "#{__listener.topic}.group")
  //...
  ```

  

##### Obtaining the Consumer `group.id`

- 여러 컨테이너에서 동일한 리스너 코드를 실행하는 경우 
- ...

#####  Container Thread Naming

- 리스터 컨테이너는 2개 task executor 를 사용한다.
- 하나는 consumer, 하나는 리스너 invoke 하는데

##### `@KafkaListener` as a Meta Annotation

// ?

##### `@KafkaListener` on a Class

- Class 레벨로 @KafkaListener 사용하면,
- 메소드 레벨에 @KafkaHandler 를 명시해야 한다
- 변환된 메시지 payload 타입이 어떤 메소드가 호출될지 결정하는데 사용된다.
- (매개에 따라 결정된다고 ?)
- how to do so

```java
@KafkaListener(id="multi", topics="myTopic")
static class MultiListenerBean {
    
    @KafkaHandler
    public void listen(Stirng foo){//...
    }
    
    @KafkaHandler
    public void listen(Integer bar){
        //...
    }
    
    @KafkaHandler(isDefault=true)
    public void listenDefault(Object object){
        //...
    }
}
```



##### `@KafkaListener` Lifecycle Management

- 리스터 컨테이너 : @KafkaListener 어노테이션으로 생성되는.
- application context 의 빈이 아니다.
- 대신에 그것들은 KafkaListenerEndpointRegistry 빈 타입으로 등록된다.
- 이 빈은 컨테이너의 생명주기를 관리한다
- container factory 에 의해 생성되는 모든 컨테이너들도 같은 맥락.



- 우리는 registry 를 사용함으로써 프로그램 적으로 생명주기를 관리할 수 있다
- regisry 를 start/stop 하는 것은 모든 등록된 containers 를 start/stop 하는 것
- id 속성을 개별 컨테이너 참조를 가져올 수 있다
- how to do so

```java
@KafkaListener(id="myContainer", ...)

@Autowired
private KafkaListenerEndpointRegistry registry;
//...
this.registry.getListenerContainer("myContainer").start();
//...
```

- registry 는 컨테이너들의 생명주기만 관리한다
- 빈으로 선언된 컨테이너들은 registry 에가 관리하지 않는다
- application context 가 획득할 수 있다



- registry 의 getListenerContainers() 메소드로 컨테이너들 획득할 수 있다.

- getAllListenerContainers() 메소드로 모든 컨테이너 획득 : collection
- registry 가 관리하는, + 빈으로 선언된 것들도
- 리턴된 collection 은 임의 prototype 빈을 포함할 것이다. 초기화된.
- 그러나 lazy 빈을 초기화해주진 않을 것이다.





##### `@KafkaListener` `@Payload` Validation

- validator 을 넣기 쉬워졌다. @KafkaListener 의 @Payload 매개에
- how to do so

```java
@Configuration
@EnableKafka
public class Config implements KafkaListenerConfigurer {
    //...
    @Override
    public void configureKafkaListeners(
    		KafkaListenerEndpointRegistrar registrar
    ){
        registrar.setValidator(new MyValidator());
    }
}
```



- Spring Boot : `LocalValidatorFactoryBean` is auto-configured

```java
@Configuration
@EnableKafka
public class Config implements KafkaListenerConfigurer {
    
    @Autowired
    private LocalValidatorFactoryBean validator;
    //...
    
    @Override
    public void configureKafkaListeners(
    		KafkaListenerEndpointRegistrar registrar
    ){
        registrar.setValidator(this.validator);
    }
}
```



- how to validate

```java
@Getter
@Setter
public static class ValidatedClass {
    
    @Max(10)
    private int bar;
}

@KafkaListener(id="validated", topics="annotated35", 
              errorHandler="validationErrorHandler",
              containerFactory="kafkaJsonListenerContainerFactory")
public void validatedListener(@Payload @Valid ValidatedClass val){
    return (m,e) -> {
        //...
    };
}
```



##### Rebalancing Listeners

/



##### Forwarding Listener Results using `@SendTo`  // **

/



##### Filtering Messages

##### Retrying Deliveries

##### Stateful Retry

/









(4.1.8 은 보면 좋을듯)

#### 4.1.8. Serialization, Deserialization, and Message Conversion

##### Overview

// 없. 아는내용

#####  JSON //

- provides `JsonSerializer` and `JsonDeserializer` implementations 
- based on the` Jackson JSON` object mapper
- `Java object` as a `JSON byte[]`
- how to create a JsonDeserializer

```java
JsonDeserializer<Thing> thingDeserializer = 
    new JsonDeserializer<>(Thing.class);
```



- can customize both `JsonSerializer` and `JsonDeserializer `with an ObjectMapper



- When constructing the `serializer`/`deserializer `
- for use in the `producer/consumer factory`
- can use the fluent API, which simplifies configuration

(아 내가 클래스 등록 안해서 JsonDeserializer 가 먹히지 않았던건가 ?)

```java
@Bean
public DefaultKafkaProducerFactory pf(KafkaProperties properties){
    Map<String,Object> props = properties.buildProducerProperties();
    DefaultKafkaProducerFactory pf = 
        new DefaultKafkaProducerFactory(
        	props, 
        	new JsonSerializer<>(MyKeyType.class)
        		.forKeys().noTypeInfo(),
    		new JsonSerializer<>(MyValueType.class)
    			.noTypeInfo());
}

@Bean
public DefaultKafkaConsumerFactory pf(KafkaProperties properties){
    Map<String,Object> props = properties.buildConsumerProperties();
    DefaultKafkaConsumerFactory pf = new DefaultKafkaConsumerFactory(
    	props,
        new JsonDeserializer<>(MyKeyType.class)
        	.forKeys()
        	.ignoreTypeHeaders(),
        new JsonSerializer<>(MyValue.class)
        	.ignoreTypeHeaders()
    );
}
```



##### Mapping Types //

- Mappings consist of a comma-delimited list of token:className pairs
- creates a set of mappings // 생략
- 스프링부트에서는 application.properties 로 제공할 수 있다

##### Delegating Serializer and Deserializer

##### Retrying Deserializer

/

##### Spring Messaging Message Conversion //

/

**Using Spring Data Projection Interfaces** // ?

/

#####  Using `ErrorHandlingDeserializer`

/

##### Payload Conversion with Batch Listeners

/

##### `ConversionService` Customization

##### Adding custom `HandlerMethodArgumentResolver` to `@KafkaListener`

/



(4.2) -- 근데 이거 전에 예제 좀 더 보다와..

###  4.2. Kafka Streams Support

- kafka-streams 가 classpath 에 있어야 한다. build.gradle 에 의존성 추가하라는 의미일까?

#### 4.2.1. Basics

- kafka streams 문서에서는 다음 api 사용법을 권장한다

```java
// builder 를 사용하여 processing topology 를 정의한다
// topic 선택, stream operation:filter,map 등도 지정
StreamsBuilder builder = ...;

// configuration 을 이용하여
// kafka cluster 가 어디에 있는지
// serializer/deserializer 디폴트로 사용할거, 보안 등 설정하기
StreamsConfig config = ...;

KafkaStreams streams = new KafkaStreams(builder, config);

// start kafka streams instance
streams.start();

// stop kafka streams instance
streams.close();
```



- main component 2가지
- `StreamsBuilder` : `KStream` 과 `KTable` 인스턴스를 build 한다
- `KafkaStreams` : 이들 instance 의 생명주기를 관리한다



> - 모든 KStream 인스턴스들은 하나의 KafkaStream 인스턴스에 의해 노출된다.
> - 싱글 StreamsBuilder 에 의해
> - 동시에 시작되고 멈춘다. 다른 로직이라 해도.
> - 다시 말해, StreamsBuilder에 의해 정의된 모든 streams 는
> - 하나의 라이프사이클로 묶여 있다
> - .
> - streams.close() 로 닫힌 KafkaStreams 인스턴스는 다시 시작될 수 없다
> - 대신에, 스트림 처리를 시작하는 새로운 KafkaStreams 객체가 생성되어야 한다
> - (그니까 일회용이다 ?)

####  4.2.2. Spring Management

- 스프링 application context 관점에서 kafka streams 을 간단하게 사용하기 위해
- 그리고 컨테이너를 통해 생명주기 관리를 사용하기 위해
- `StreamsBuilderFactoryBean` 를 도입했다.
- 이것은 `AbstractFactoryBean` 의 구현체이며, 빈으로써 `StreamsBuilder` 싱글톤 객체를 노출한다.
- example creates such a bean

```java
@Bean
public FactoryBean<StreamsBuilder> myKStreamBuilder(
		KafkaStreamsConfiguration streamsConfig){
    
    return new StreamsBuilderFactoryBean(streamsConfig);
}
```

- 또한 `StreamsBuilderFactoryBean` 은 `SmartLifecycle` 을 구현한다.
- 내부 `KafkaStreams` 인스턴스의 생명주기를 관리한다
- `KafkaStreams` 를 시작하기 전에 `KStreams` 객체를 정의해야 한다.


- 그러므로 `StreamsBuilderFactoryBean` 에 기본 `autoStartup=true`을 사용할 경우,
- `StreamsBuilder` 에 `KStream` 객체를 선언해야 한다. application context 가 refresh 되기 전에
- 예를 들어, `KStreams` 는 regular bean 선언이 될 수 있다

- following example shows how to do so

```java
@Bean
public KStream<?,?> kStream(StreamsBuilder kStreamBuilder){
    
    KStream<Integer,String> stream = kStreamBuilder
        .stream(STREAMING_TOPIC1);
    
    return stream;
}
```



- 수명주기를 수동적으로 제어하고 싶다면
- 예를 들어, 특정 조건에 따라 시작하거나 멈춘다
- 참조할 수 있다. StreamsBuilderFactoryBean 빈을. $ 접두사를 사용함으로써.



- StreamsBuilderFactoryBean 은 그것의 내부 KafkaStreams 객체를 사용하므로,
- 안전하다. 멈추거나 다시 시작하는거.
- 새로운 KafkaStreams 는 각각의 start() 마다 생성된다.
- (start() 하면 새로운 KafkaStreams 가 생성된다 ?)



- 또한 각 별개로 다른 StreamsBuilderFactoryBean 객체를 사용할 수 있다.
- KStream 객체의 수명주기를 개별적으로 제어하고 싶을 때

#### 4.2.3. Streams JSON Serialization and Deserialization

- 데이터를 직렬화, 역직렬화 하기 위해
- 토픽/상태저장소 에 읽거나 쓸 때
- 스프링 카프카에서 JsonSerde 구현제를 제공한다
- Json 을 사용하여 JsonSerializer,JsonDeserializer 에 위임하는.



- JsonSerde 구현체는 같은 configuration 옵션을 제공한다.
- 생성자를 통해 (target type or objectMapper ?)



- 다음 예제는 JsonSerde 를 사용한다. Cat 을 직렬화/역직렬화 하기 위해
- 필요할 때 비슷한 방식으로 사용하면 된다.

```java
stream.through(Serdes.Integer(), new JsonSerde<>(Cat.class), "cats");
```



- 프로그램으로 직렬화/역직렬화 할 때, producer, consumer factory 에서
- 구성을 간소하게 할 수 있다.

```java
streams.through(new JsonSerde<>(MyKeyType.class)
                	.forKeys()
                	.noTypeInfo(),
                new JsonSerde<>(MyValueType.class)
                	.noTypeInfo(),
                "myTopics");
```



#### 4.2.4. Using `KafkaStreamsBrancher`

// 우와 이건 진짜 꼭 좋다

- KafkaStreamBrancher 클래스는 KStream 에 조건문을 만드는 편리한 방법 제공한다.
- 다음 예제는 KafkaStreamBrancher 를 사용하지 않는 예제

```java
// KStream<>[] 배열로 받아온 후, 배열에 접근하여 처리
```

- 다음 예제는 KafkaStreamBrancher 를 사용하는 예제

```java
// 람다식처럼 바로 처리가능
new KafkaStreamsBrancher<String,String>()
    .branch((key,value) -> value.contains("A"), ks -> ks.to("A"))
    .branch((key,value) -> value.contains("B"), ks -> ks.to("B"))
    .defaultBranch(ks -> ks.to("C")) // optional ?
    .onTopOf(builder.stream("source")); // method chain 더 붙일수있다
```

> *`onTopOf` method returns the provided stream so we can continue with method chaining*

####  4.2.5. Configuration

// pass

####  4.2.6. Header Enricher // no

// pass

####  4.2.7. `MessagingTransformer`

// pass

#### 4.2.8. Recovery from Deserialization Exceptions // no

// pass

####  4.2.9. Kafka Streams Example

```java
@Configuration
@EnableKafka
@EnableKafkaStreams
public static class KafkaStreamsConfig {
    
    @Bean(name=KafkaStreamsDefaultConfiguration
          .DEFAULT_STREAMS_CONFIG_BEAN_NAME) // ?
    public KafkaStreamsConfiguration kStreamsConfigs(){
        
        Map<String,Object> props = new HashMap<>();
        props.put(StreamsConfig.APPLICATION_ID_CONFIG, "testStreams");
        props.put(StreamsConfig.KEY_SERDE_CLASS_CONFIG, 
                  Serdes.Integer().getClass().getName());
        props.put(StreamsConfig.VALUE_SERDE_CLASS_CONFIG,
                 Serdes.String().getClass().getName());
        props.put(StreamsConfig.TIMESTAMP_EXTRACTOR_CLASS_CONFIG,
                 WallclockTimeStampExtractor.class.getName());
        
        return new KafkaStreamsConfiguration(props);
    }
    
    @Bean
    public StreamsBuilderFactoryBeanCustomizer customizer(){
        
        return fb -> fb.setStateListener(
        	(newState, oldState) -> {
                System.out.println("State transtion from "+
                                  oldState +" to "+newState);
            } 
        )
    }
    
    @Bean
    public Kstream<Integer,String> 
        	kStream(StreamsBuilder kStreamBuilder){
        
        KStream<Integer,String> stream = 
            KStreamBuilder.stream("streamingTopic1");
        
        stream.mapValues(String::toUpperCase)
            .groupByKey()
            .reduce((String value1,String value2) -> value1 + value2,
                   TimeWindows.of(1000),
                   "windowStore")
            .toStream()
            .map((windowedId,value) -> 
                 new KeyValue<>(windowedId.key(),value))
            .to("streamingTopic2");
        
        stream.print();
        
        return stream;
        	
    }
}
```

> - @Configuration 에 @EnableKafkaStream 붙여야 한다
> - KafkaStreamConfiguration 빈을 선언하기만 하면 된다. named : defaultKafkaStreamsConfig
> - 추가적인 커스터마이징을 할 수 있다. 빈의
> - StreamsBuilderFactoryBeanCusomizer 구현체 빈을 제공함으로써. 여기처럼
> - 꼭 하나만 주거나, @Pribary 붙여야 한다



# Kafka 사례조사

## 프로그램 추천 서비스 : 약간 우리랑 비슷할듯

- 주문형 TV 프로그램,영화를 볼 수 있는 플랫폼에서.
- user 가 resume 할지 알고싶다 ? (뭘 측정해서 알수 있을까)
- user profile 을 실시간으로 만들고 싶어
- 모든 데이터를 저장하고 싶은거. 분석 저장소에.
- 카프카를 사용하여 이것을 어떻게 구현할 수 있을까 ?
- 필요한 것은 무엇일까
- Producers, consumers, Kafka Connect, Kafka Streams, anything you think is useful
- 카프카가 중간에 있고, 첫번째 topic 흥미로워해야할? show position
- 사용자가 하나의 비디오 안에서 얼마나 소비해왔는지 알려주는 topic
- 우리가 생성해야 할 데이터는 show position
- 사용자 브라우저에 비디오 플레이어가 있다고 가정.
- 그리고 사용자들은 비디오를 재생하고 있다
- 비디오가 재생되는 동안, 우리는 데이터를 보낼 거야. video position service

- 카프카로 전송하기 전에 데이터 올바르게 조금 손봐야 한다

- resume 능력. topic 의 consumer 는 position 을 보여줘야 한다. 각각의 유저 얼마나 많이 소비해왔는지. (뭘 ?)
- 그냥 예전에 중단했던 지점을 알려주는 서비스 ?
- 우리는 show position 데이터를 갖고 있다.
- 각 유자가 어떤 쇼를 왔는지 얼마나 봤는지
- 여기에 추천 엔진을 붙일 수 있다.
- kafka stream 에 의해 구동될 수 있다.
- 정말 좋은 알고리즘으로.
- 실시간으로 추천 내역을 생성한다.
- 추천 기능이 언제 사용되냐면, 비디오 플레이어를 그만두고 되돌아갈 때마다
- (껐다 다시 켤 때마다 ?) 아니 포털 메일으로 돌아갈 때마다
- 이 프로그램은 다음으로 시청하기에 적합합니다. 당신에게.
- 당연하게 카프카에서만 머물면 안된다
- consumer 로서 분석기가 있어야 한다
- 카프카 connect 에 의해 연결되는 ?
- topic : show position 에 대해,
- 이 토픽은 producer 가 multiple 하다.
- 전 세계 사람들이 tv 프로그램을 볼거야. 그리고 데이터를 토픽에 넣겠지.
- 그래서 아주 많이 분산되어야 해. 파티션 많이
- 그러나 recommendation topic 에 대해서는 사용자 id 를 키로, 적은 파티션만 있어도 괜찮. 게다가 갱신도 적어. 트래픽 적어.

## IOT 택시

- 사람들이 원할 때 택시와 매칭되는 서비스
- 근접한 운전자와 매칭되어야 한다.
- 운전자 수가 적으면/사용자가 많으면 가격이 올라간다 
- (수요와 공급 관련 내용인듯)
- 시간 당 모든 위치 데이터가 저장된다
- 비용은 정확하게 계산한다.
- 분석기가 하겠지.
- user position 토픽. 사용자 위치를 담는 topic
- 사용자가 앱을 열거나/닫으면 유저가 어디에 있는지 담는다
- 앱과 휴대폰을 직접적으로 카프카에 연결하지 않는다 ?
- 항상 서비스를 proxy 로 사용한다
- 사용자 앱에서 사용자 위치를 받는 기능이 있어야 한다
- 그 서비스는 프로듀서가 될 것이다 user position 토픽에 대해.
- 그 다음에 택시 운전자 앱이 있어야 한다. 마찬가지로 producer 로써.
- taxi_positon 토픽의.
- 두 토픽 모두 양이 많을 것이다.
- 정확히 말하자면 사용자 보다 택시 데이터가 더 많을거야.
- 그래고 surge pricing 토픽이 있다. user position 과 taxi position 을 계산하는.
- 두 토픽을 계산하는 간단한/복잡한 룰이 있어야 한다
- 카프카 streams 나 spark 를 이용할 수 있겠다.
- user/taxi position 토픽에서 데이터를 읽어들여 input
- surge pricing 토픽으로 output 할거다
- kafka streams 의 input topic 개수는 제한이 없나보다

- surge pricing 토픽을 사용자 앱에 서비스해야 한다
- 예상 서비스 비용으로.
- 택시 비용 기능을 만든다.
- 데이터 저장소는 하둡/s3가 될 수 있겠다 : 전형적이다
- 카프카 connect 를 이용하겠다.
- 궁금한게 :
- kafka 로는 저장할 수 없나봐.. 그럼 nosql 붙여볼 생각도 해봐야겠다

## social media 서비스

- CQRS 를 이용할거야
- Command Query Responsibility Segration
- 알기에 흥미로운 모델이다.
- 서비스는 이미지를 등록할 수 있고, 거기에 반응할 수 있고 좋아요,댓글로.
- 그리고 작성자는 전체 좋아요,댓글 수를 실시간으로 볼 수 있어.
- 데이터 양도 많고 처리량도 상당할거야
- 사용자는 작성글 트랜드도 볼 수 있고 (실시간 피드 ?)
- CQRS 를 사용한 아키텍처 라면 구현할수있다
- 토픽 3개가 있다.
- posts, likes, comments
- 게시 기능이 있다.
- 게시글에는 text, link, hashtag 가 포함되어 있겠다
- 모든 게시 기능은 post 토픽에 전송할거야.
- 좋아요, 댓글도 producer 각 토픽에.
- 카프카 stream 를 사용할거야. 집계할 때.
- posts,likes,comments 토픽에서 데이터를 읽고,
- 얼마나 내 글에 좋아요 했는지 집계한다
- counts 토픽에 저장.
- 마찬가지로 trending 게시글을 원한다면
- 아주 좋아요 눌리고, 많이 댓글 달린.
- 지난 1시간으로 기간을 제한해볼 수도 있겠다

- posts 토픽은 확장 필요.
- user id 는 time 에 post id 를 만들었습니다. 데이터 저장 ?
- streams 강의에 비슷한 예제가 있었던듯! 이게 event 로 쓰인다고 한다

- 결론적으로 토픽은 5개
- post, like, comment, post_with_count, treding_post

# back service

- 아주 큰 거래가 발생한 경우 = 사기
- 사용자에게 경로를 주는 서비스
- producer 는 db 인 모양.
- 사용자가 앱을 통해 한계값을 설정할 수 있다
- 실시간이 생명인 기능
- 카프카 connect 를 사용할거야 producer 가 데베여서.
- strems 도 사용해볼 수 있어. 작은 소비자.

# 빅데이터

- 카프카에서 하둡,s3,엘라스틱 서치로 데이터를 trasfer 한다
- 카프카 기능 두 가지
- 실시간 -빠른 층
- 버퍼 기능 -느린층
- 수집 버퍼로 사용하는 것은 매우 일반적인 패턴



# 그리고 유데미 강의!!

> title : hands on project

## sec8. build springboot kafka producer 

- set up  base project for library event kafka producer
  - ~~library events producer~~
- build the library event domain
  - ~~library event domain~~  < 별거 없었음. 도메인만 있었다
- create post endpoint /libraryevent
  - ~~post library event~~
- introduction spring kafkaTemplate. produce messages
- configure kafkaTemplate using springboot profiles. application.yml
  - ~~library event producer autoconfigurer~~
- how spring boot autoConfiguration works ? - kafka producer
- autoCreate Topic sing kafkaAdmin
  - ~~auto create topic~~
- build libraryEvents producer using kafkaTemplate - approach1
  - ~~library events producer kafka template asynchronous part1~~
  - ~~library events producer kafka template async part2~~
- libraryEvents producer api - behine scenes
  - ~~library events producer api behind the scenes~~
- build libraryEvents producer using kafkaTemplate - approach2
  - ~~library events producer kafka template sync approach2~~
- build libraryEvents producer using kafkaTemplate - approach3
  - ~~library events producer kafka template asnyc approach3~~
- sending kafkarecord with headers using kafkaTemplate
  - ~~**kafka template with headers**~~
- add libraryEvent type - new,update

# sec11. build spring boot kafka producer - sending message with key

- create put endpoint - v1/libraryEvent
  - ~~**put library events producer**~~ 

# sec12. kafka producer : imortant configurations

- kafka producer - inportant configurations
- configure acks value as all
  - ~~**config producer override**~~ **// 4**

# sec13. build springboot kafka consumer (3)

- set up library events consumer base project
  - ~~**library events consumer set up**~~
- introduction spring kafka consumer
- configure kafka consumer using springboot profiles - application.yml
  - ~~**kafka consumer auto configure**~~
- build kafka consumer using @KafkaListener annotation
  - ~~**kafka consumer kafkalistener**~~
- how springboot autoconfiguration works? - kafka consumer

# sec14. consumer groups and consumer offset management (2)

- consumer groups and rebalance
- default consumer offset management in spring kafka
- manual consumer offset management
  - ~~**manual consumer offset management**~~
- concurrent consumer
  - ~~**concurrent consumers**~~

# sec15. persisting library events in db - using h2 inmemory database (4)

- configuring h2 - in memory db
  
  - ~~**configure in memory db**~~
- create libraryEvent and book entity
  
  - ~~**configure library event entity**~~ // producer,consumer 같이 있다!
- build service layer to process libraryEvent - add
  
  - ~~**library event service add**~~ // consuer 에서 받고 service 로 넘겨서 save 하는건 이게 처음인듯 아니네 처음부터 consumer 에서 save 를 했다
  
    // 그럼 같은 class 객체를 produce 하고 consuer 하는 흐름을 정리해보자
- build service layer to process libraryEvent - modify
  
  - ~~**library event service modify**~~



# SpringBoot Kafka Bean

### KafkaProperties

- kafkaProperties

### KafkaAutoConfiguration

- kafkaAdmin
- **kafkaProducerFactory**
- **kafkaTemplate**
- kafkaJaasInitializer
- kafkaProducerListener
- kafkaAutoConfiguration
- **kafkaConsumerFactory**
- kafkaTransactionManager

### KafkaStreamsAnnotationDrivenConfiguration

- kafkaStreamsAnnotationDrivenConfiguration
- defaultKafkaStreamConfig
- kafkaStreamsFactoryBeanConfigurer

### KafkaListenerConfigurationSelector

- kafkaListenerConfigurationSelector

### KafkaAnnotationDrivenConfiguration

- enableKafkaConfiguration
- kafkaAnnotationDrivenConfiguration
- **kafkaListenerContainerFactoryConfigurer**
- **kafkaListenerContainerFactory**

