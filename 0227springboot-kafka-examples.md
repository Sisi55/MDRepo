# library-events-producer

```cmd
# build.gradle
springBootVersion : 2.2.2.RELEASE
sourceCompatibility='11'
implementation 'org.springframework.kafka:spring-kafka'
testImplementation 'org.springframework.kafka:spring-kafka-test'
```

```yml
# aplication.yml

spring:
	profiles:

		active: local
---

spring:
	profiles: local
	kafka:
		template:
			default-topic: library-events
		producer:
			bootstrap-servers: localhost:9092,localhost:9093,localhost:9094
			key-serializer: org.apache.kafka.common.serialization.IntegerSerializer
			value-serializer: org.apache.kafka.common.serialization.StringSerializer
			properties:
				acks: all
				retries: 10
				retry.backoff.ms: 1000
		admin:
			properties:

				bootstrap.servers: localhost:9092,localhost:9093,localhost:9094
---

spring:
	profiles: dev
	kafka:
		producer:
			bootstrap-servers: dev:9092
			key-serializer: ...IntegerSerializer

			value-serializer: ...StringSerializer
---

spring:
	profiles: prod
	kafka:
		producer:
			bootstrap-servers: prod:9092
			key-serializer: ...IntegerSerializer
			value-serializer: ...StringSerializer
```



- `producer/LibraryEventProducer.java`

- > send 함수는 ListenableFuture<SendResult> 를 반환한다.
  >
  > 전송 결과를 비동기적으로 받는 콜백을 등록할 수 있다.

```java
@Component
@Slf4j
public class LibraryEventProducer {
    
    @Autowired
    KafkaTemplate<Integer,String> kafkaTemplate;
    
    String topic = "library-events";
    
    @Autowired
    ObjectMapper objectMapper;
    
    public void sendLibraryEvent(LibraryEvent libraryEvent)
        	throws JsonProcessingException {
        
        Integer key = libraryEvent.getLibraryEventId();
        String value = objectMapper.writeValueAsString(libraryEvent);
        
        ListenableFuture<SendResult<Integer,String>> listenableFuture =
            kafkaTemplate.sendDefault(key,value);
        listenableFuture.addCallback(
            	new ListenableFutureCallback<
            		SendResult<Integer,String>>(){
            
			@Override
			public void onFailure(Throwable ex){
    			handleFailure(key,value,ex);
			}
                        
			@Override
			public void onSuccess(SendResult<Integer,String> result){
    			handleSuccess(key,value,result);
			}
        });
    }
    
    public ListenableFuture<SendResult<Integer,String>> 
        	sendLibraryEvent_Approach2(LibraryEvent libraryEvent)
        	throws JsonProcessingException {
        
        Integer key = libraryEvent.getLibraryEventId();
        String value = objectMapper.writeValueAsString(libraryEvent);
         
        ProducerRecord<Integer,String> producerRecord = 
            buildProducerRecord(key,value,topic);
        
        ListenableFuture<SendResult<Integer,String>> listenableFuture =
            kafkaTemplate.send(producerRecord);
        listenableFuture.addCallback(
        	new ListenableFutureCallback<SendResult<Integer,String>>(){
                @Override
                public void onFailure(Throwable ex){
                    handleFailure(key,value,ex);
                }
                
                @Override
                public void onSuccess(
                    	SendResult<Integer,String> result){
                    handleSuccess(key,value,result);
                }
            });
        
        return listenableFuture;
    }
    
    private ProducerRecord<Integer,String>
        	buildProducerRecord(Integer key,String value,String topic){
        
        List<Header> recordHeaders = List.of( // ?
        	new RecordHeader("event-source","scanner".getBytes())
        );
        
        return new ProducerRecord<>(
            topic,null,key,value,recordHeaders);
    }
    
    public SendResult<Integer,String> 
        	sendLibraryEventsSynchronous(LibraryEvent libraryEvent)
        	throws JsonProcessingException,ExecutionException,
    				InterruptedException,TimeoutException{
		
		Integer key = libraryEvent.getLibraryEventId();
		String value = objectMapper.writeValueAsStirng(libraryEvent);
		SendResult<Integer,String> sendResult = null;
                        
		try{
    		sendResult = kafkaTemplate.sendDefault(key,value)
            	.get(1,TimeUnit.SECONDS);
		}catch(ExecutionException | InterruptedException e){
    		log.error("ExecutionException/InterrupedException Sending the Message and the exception is {}", e.getMessage());
      	  	throw e;
		}catch(Exception e){
   		 	log.error("Exception Sending the Message and the exception is {}", e.getMessage());
      		throw e;
		}
                        
		return sendResult;
	}
    
    private void handleFilure(Integer key,String value,Throwable ex){
        log.error("Error Sending the Message and th exception is {}",
                 ex.getMessage());
        try{
            throw ex;
        }catch(Throwable throwable){ //
            log.error("Error in OnFailure: {}", 
                      throwable.getMessage());
        }
    }
    
    private void handleSuccess(Integer key,String value,
                               SendResult<Integer,String> result){
        log.info("Message Sent SuccessFully for the key : {} and the value is {}, partition is {}", 
                 key,value,result.getRecordMetadata().partition());
    }
}
```



- `domain/LibraryEvent.java`

```java
@AllArgsConstructor
@NoArgsConstructor
@Data // ?
@Builder
public class LibraryEvent {
    
    private Integer libraryEventId; // ?
    private LibraryEventType libaryEventType; //
    
    @NotNull
    @Valid // ?
    private Book book; //
}
```

- domain/LibraryEventType.java

```java
public enum LibraryEventType {
    NEW,
    UPDATE
}
```

- domain/Book.java

```java
@AllArgsConstructor
@NoArgsConsturctor
@Data
@Builder
public class Book {
    @NotNull
    private Integer bookId;
    
    @NotBlank
    private String bookName;
    
    @NotBlank
    private String bookAuthor;
}
```

- controller/LibraryEventsController.java

```java
@RestController
@Slf4j
public class LibraryEventsController {
    
    @Autowired
    LibraryEventProducer libraryEventProducer;
    
    @PostMapping("/v1/libraryevent")
    public ResponseEntity<LibraryEvent> postLibraryEvent(
    		@RequestBody @Valid LibraryEvent libraryEvent)
        	throws JsonProcessingException, ExecutionException,
    				InterruptedException {
                        
		libraryEvent.setLibraryEventType(LibraryEventType.NEW);  
		libraryEventProducer.sendLibraryEvent_Approach2(libraryEvent);
                        
		return ResponseEntity.status(HttpStatus.CREATED)
            .body(libraryEvent);
	}
    
    @PutMapping("/v1/libraryevent")
    public ResponseEntity<?> putLibraryEvent(
    		@RequestBody @Valid LibraryEvent libraryEvent)
        	throws JsonProcessingException, ExecutionException,
    				InterruptedException {
                        
    	if(libraryEvent.getLibraryEventId() == null){
        	return ResponseEntity.status(HttpStatus.BAD_REQUEST)
            	.body("Please pass the LibraryEventId");
    	}
    
    	libraryEvent.setLibraryEventType(LibraryEventType.UPDATE);
    	libraryEventProducer.sendLibraryEvent_Approach2(libraryEvent);
    
    	return ResponseEntity.status(HttpStatus.OK)
        	.body(libraryEvent);        
	}
}
```

- controller/LibraryEventControllerAdvice.java

```java
@ControllerAdvice
@Slf4j
public class LibraryEventControllerAdvice {
    
    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<?> handleRequestBody(
    		MethodArgumentNotValidException ex){
        
        List<FieldError> errorList = ex.getBindingResult()
            .getFieldErrors();
        String errorMessage = errorList.stream()
            .map(fieldError -> fieldError.getField()+" - "+
                fieldError.getDefaultMessage())
            .sorted()
            .collect(Collectors.joining(", "));
        
        log.info("errorMessage : {} ", errorMessage);
        
        return new ResponseEntity<>(errorMessage, 
                                    HttpStatus.BAD_REQUEST);
    }
}
```

- config/AutoCreateConfig.java

```java
@Configuration
@Profile("local")
public class AutoCreateConfig {
    
    @Bean
    public NewTopic libraryEvents(){
        
        return TopicBuilder.name("library-events")
            .partitions(3)
            .replicas(3)
            .build();
    }
}
```

# library-events-consumer

- config/LibraryEventsConsumerConfig

```java
@Configuration
@EnableKafka
@Slf4j
public class LibraryEventsConsumerConfig {
    
    @Autowired
    LibraryEventsService libraryEventsService;
    
    @Bean // container factory : container 어디서 쓰이지 ? 자동 주입?
    ConcurrentKafkaListenerContainerFactory<?,?>
        	kafkaListenerContainerFactory(
    		ConcurrentKafkaListenerContainerFactoryConfigurer 		
        												configurer,
        	ConsumerFactory<Object,Object> kafkaConsumerFactory
    ){
        ConcurrentKafkaListenerContainerFactory<Object,Object> factory
            = new ConcurrentKafkaListenerContainerFactory<>();
        configurer.configure(factory, kafkaConsumerFactory);
        // container config 를 container 에 set 하는게 아니라
        // container config 에 container/consumer factory 를 전달 ?
        factory.setConcurrency(3);
        // factory.getContainerProperties().setAckMode(
        //		ContainerProperties.AckMode.MANUAL);
        factory.setErrorHandler(((thrownException,data) -> {
            log.info("Exception in consumerConfig is {} and the record is {}", thrownException.getMessage(), data);
        }));
        factory.setRetryTemplate(retryTemplate());
        factory.setRecoveryCallback((context -> {
            
            if(context.getLastThrowable().getCause()
              instanceof RecoverableDataAccessException){
                // invoke recovery logic
                log.info("Inside the recoverable logic");
                
                //Arrays.asList(context.attributeNames())
                //    .forEach(attributeName -> {
                //        log.info("Attribute name is:{}",attributeName);
                //        log.info("Attribute Value is:{}",
                //                 context.getAttribute(attributeName));
                //    });
                
                ConsumerRecord<Integer,String> consumerRecord = 
                    (ConsumerRecord<Integer,String>)
                    context.getAttribute("record");
                libraryEventsService.handleRecovery(consumerRecord);
            }else{
                log.info("Inside the non recoverable logic");
                throw new RuntimeException(context.getLastThrowable()
                                          .getMessage());
            }
            
            return null;
        }));
        
        return factory;
    }
    
    private RetryTemplate retryTemplate(){
        FixedBackOffPolicy fixedBackOffPolicy = 
            new FixedBackOffPolicy();
        fixedBackOffPolicy.setBackOffPeriod(1000);
        
        RetryTemplate retryTemplate = new RetryTemplate();
        retryTemplate.setRetryPolicy(simpleRetryPolicy());
        retryTemplate.setBackOffPolicy(fixedBackOffPolicy);
        
        return retryTemplate;
    }
    
    private RetryPolicy simpleRetryPolicy(){
        // SimpleRetryPolicy simpleRetryPolicy = new SimpleRetryPolicy();
        // simpleRetryPolicy.setMaxAttempts(3);
        
        Map<Class<? extends Throwable>,Boolean> exceptionsMap = 
            new HashMap<>();
        exceptionsMap.put(IllegalArgumentException.class,false);
        exceptionsMap.put(RecoverableDataAccessException.class,true);
        
        SimpleRetryPolicy simpleRetryPolicy = new SimpleRetryPolicy(
            3, exceptionsMap, true);
        
        return simpleRetryPolicy;
    }
}
```



- service/LibraryEventsService

```java
@Service
@Slf4j
public class LibraryEventsService {
    
    @Autowired
    ObjectMapper objectMapper;
    
    @Autowired
    KafkaTemplate<Integer,String> kafkaTemplate;
    
    @Autowired
    private LibraryEventsRepository libraryEventsRepository;
    
    public void processLibraryEvent(
        	ConsumerRecord<Integer,String> consumerRecord) 
        	throws JsonProcessingException {
        
        LibraryEvent libraryEvent = objectMapper.readValue(
            consumerRecord.value(), LibraryEvent.class);
        log.info("{}", libraryEvent);
        
        if(libraryEvent.getLibraryEventId() != null &&
          libraryEvent.getLibraryEventId() == 000){
            throw new RecoverableDataAccessException(
                "Temporary network issue");
        }
        
        switch(libraryEvent.getLibraryEventType()){
            case NEW:
                save(libraryEvent);
                break;
            case UPDATE:
                validate(libraryEvent);
                save(libraryEvent);
                break;
            default:
                log.info("Invalid Library Event Type");
        }
    }
    
    private void validate(LibraryEvent libraryEvent){
        if(libraryEvent.getLibraryEventId() == null){
            throw new IllegalArgumentException("missing id");
        }
        
        Optional<LibraryEvent> libraryEventOptional
            = libraryEventsRepository.findById(
        		libraryEvent.getLibraryEventId());
        if(!libraryEventOptional.isPresend()){
            throw new IllegalArgumentException("not valid object");
        }
        
        log.info("{}", libraryEventOptional.get());
    }
    
    private void save(LibraryEvent libraryEvent){
        libraryEvent.getBook().setLibraryEvent(libraryEvent);
        libraryEventsRepository.save(libraryEvent);
        
        log.info("{}", libraryEvent);
    }
    
    public void handleRecovery(ConsumerRecord<Integer,String> record){
        Integer key = record.key();
        String message = record.value();
        
        ListenableFuture<SendResult<Integer,String>> listenableFuture
            = kafkaTemplate.sendDefault(key,message);
        listenableFuture.addCallback(
        	new ListenableFutureCallback<SendResult<Integer,String>>(){
                @Override
                public void onFailure(Throwable ex){
                    handleFailure(key,message,ex);
                }
                @Override
                public void onSuccess(
                    	SendResult<Integer,String> result){
                    handleSuccess(key,message,result);
                }
            })
    }
    
    private void handlFailure(Integer key,Stirng value,Throwable ex){
        log.error("{}", ex.getMessage());
        try{
            throw ex;
        }catch(Throwable throwable){
            log.error("{}", throwable.getMessage());
        }
    }
    
    private void handleSuccess(Integer key,Stirng value,
                               SendResult<Integer,String> result){
        log.info("{},{},{}",
                 key,value,result.getRecordMetadata().partition());
    }    
}
```



- jpa/LibraryEventsRepository

```java
public interface LibraryEventsRepository
    extends CrudRepository<LibraryEvent,Integer>{
    
}
```



- entity/LibraryEvent

```java
@AllArgsConsturctor
@NoArgsConstructor
@Data
@Builder
@Entity
public class LibraryEvent{
    
    @Id
    @GeneratedValue
    private Integer libraryEventId;
    
    @Enumerated(EnumType.STRING)
    private LibraryEventType libraryEventType;
    
    @OneToOne(mappedBy="libraryEvent", cascade={CascadeType.ALL})
    @ToString.Exclude // 무한 순회 방지 ?
	private Book book;        
}
```



- entity/Book

```java
@AllArgsConstructor
@NoArgsConstructor
@Data
@Builder
@Entity
public class Book{
    
    @Id
    private Integer bookId;
    
    private String bookName;
    private String bookAuthor;
    
    @OneToOne
    @JoinColumn(name="libraryEventId")
    private LibraryEvent libraryEvent;
}
```



- consumer/LibraryEventsConsumerManualOffset

```java
//@Component
@Slf4j
public class LibraryEventsConsumerManualOffset
    implements AcknowledgingMessageListener<Integer,String>{
    
    @Override
    @KafkaListener(topic={"library-events"})
    public void onMessage(
        	ConsumerRecord<Integer,String> consumerRecord,
    		Acknowledgment acknowledgment){
        
        log.info("{}", consumerRecord);
        acknowledgment.acknowledge();
    }
}
```



- consumerLibraryEventsConsumer

```java
@Component
@Slf4j
public class LibraryEventsConsumer {
    
    @Autowired
    private LibraryEventsService libraryEventsService;
    
    @KafkaListener(topic={"library-events"})
    public void onMessage(
        	ConsumerRecord<Integer,String> consumerRecord)
        	throws JsonProcessingException {
        
        log.info("{}", consumerRecord);
        libraryEventsService.processLibraryEvent(consumerRecord);
    }
}
```

# post-library-event

- controller/LibraryEventsController

```java
@RestController
public class LibraryEventsController {
    
    @PostMapping("/v1/libraryevent")
    public ResponseEntity<LibraryEvent> postLibraryEvent(
    		@RequestBody LibraryEvent libraryEvent){
        
        return ResponseEntity.status(HttpStatus.CREATED)
            .body(libraryEvent);
    }
}
```

# auto-create-topic

- config/AutoCreateConfig

```java
@Configuration
@Profile("local")
pblic class AutoCreateConfig {
    
    @Bean
    public NewTopic libraryEvents(){
        return TopicBuilder.name("library-events")
            .partitions(3)
            .replicas(3)
            .build();
    }
}
```

# library-events-producer-kafka-template-asynchronous-part1

- producer/LibraryeventProducer

```java
@Component
@Slf4j
public class LibraryEventProducer {
    
    @Autowired
    KafkaTemplate<Integer,String> kafkaTemplate;
    
    @Autowired
    ObjectMapper objectMapper;
    
    public void sendLibraryEvent(LibraryEvent libraryEvent)
        	throws JsonProcessingException {
        
        Integer key = libraryEvent.getLibraryEventId();
        String value = objectMapper.writeValueAsString(libraryEvent);
        
        ListenableFuture<SendResult<Integer,String>> listenableFuture =
            kafkaTemplate.sendDefault(key,value);
        listenableFuture.addCallback(
        	new ListenableFutureCallback<SendResult<Integer,String>>(){
                @Override
                public void onFailure(Throwable ex){
                    handleFailure(key,value,ex);
                }
                @Override
                public void onSuccess(
                    	SendResult<Integer,String> result){
                    handleSuccess(key,value,result);
                }
            });
    }
    
    private void handleFailure(Integer key,String value,Throwable ex){
        log.error("{}", ex.getMessage());
        try{
            throw ex;
        }catch(Throwable throwable){
            log.error("{}", throwable.getMessage());
        }
    }
    
    private void handleSuccess(Integer key,String value,
                               SendResult<Integer,String> result){
        log.info("{},{},{}", key,value,
                 result.getRecordMetadata().partition());
    }
}
```

# library-events-producer-kafka-template-async-part2

- controller/LibraryEventsController

```java
@RestController
public class LibraryEventsController {
    
    @Autowired
    LibraryEventProducer libraryEventProducer;
    
    @PostMapping("/v1/libraryevent")
    public ResponseEntity<LibraryEvent> postLibraryEvent(
    		@RequestBody LibraryEvent libraryEvent)
        	throws JsonProcessingException {
        
        libraryEventProducer.sendLibraryEvent(libraryEvent);
        
        return ResponseEntity.status(HttpStatus.CREATED)
            .body(libraryEvent);
    }
}
```



- producer/LibraryEventProducer

```java
@Component
@Slf4j
public class LibraryEventProducer {
    
    @Autowired
    KafkaTemplate<Integer,String> kafkaTemplate;
    
    @Autowired
    ObjectMapper objectMapper;
    
    public void sendLibraryEvent(LibraryEvent libraryEvent)
        	throws JsonProcessingException {
        
        Integer key = libraryEvent.getLibraryEventId();
        String value = objectMapper.wirteValueAsString(libraryEvent);
        
        ListenableFuture<SendResult(Integer,String)> listenableFuture
            = kafkaTemplate.sendDefault(key,value);
        listenableFuture.addCallback(new ListenableFutureCallback<SendResult<Integer,Stirng>>(){
            ..
        });
    }
    
    private void handleFailure(..){..}
    private void handleSuccess(..){..}
}
```

# library-events-producer-kafka-template-async-approach3

- producer/LibraryEventProducer

```java
@Component
@Slf4j
public class LibraryEventProducer {
    
    @Autowired
    KafkaTemaplte<Integer,String> kafkaTemplate;
    
    string topic = "library-events";
    
    @Autowired
    ObjectMapper objectMapper;
    
    //...
    
    public void sendLibraryEvent_approach2(LibraryEvent libraryEvent)
        	throws JsonProcessingException {
        Integer key = libraryEvent.getLibraryEventId();
        Stirng value = objectMapper.writeValueAsString(libraryEvent);
        
        ProducerRecord<Integer,String> producerRecord = 
            buildProducerRecord(key,value,topic); //
        
        ListenableFuture<SendResult<Integer,String>> listenableFuture
            = kafkaTemplate.send(producerRecord);
        listenableFuture.addCallback(
            new ListenableFutureCallback<SendResult<Integer,String>>(){
                ..});
    }
    
    private ProducerRecord<Integer,String> buildProducerRecord(
    		Integer key,String value, Stirng topic){
        
        return new ProducerRecord<>(topic,null,key,value,null);
    }
    
    //...
}
```

# library-events-producer

- producer/LibraryEventProducer

```java
//...
private ProducerRecord<Integer,String> buildProducerRecord(
		Integer key,String value,String topic){
    
    List<Header> recordHeaders = List.of(
    		new RecordHeader("event-source","scanner".getBytes()));
    
    return new ProducerRecord<>(topic,null,value,recordHeaders);
}
//...
```

# kafka-consumer-kafkalistener

- consumer/LibraryEventsConsumer

```java
@Component
@Slf4j
public class LibraryEventsConsumer {
    
    @KafkaListener(topics={"library-events"})
    public void onMessage(
        	ConsumerRecord<Integer,String> consumerRecord){
        
        log.info("ConsumerRecord : {}", consumerRecord);
    }
}
```

- config/LibraryEventsConsumerConfig

```java
@Configuration
@EnableKafka
public class LibraryEventsConsumerConfig{
    
}
```

# kafka-consumer-elasticsearch

- build/gradle

```cmd
compile('org.elasticsearch.client:elasticsearch-rest-high-level-client:7.5.1')
compile('org.apache.kafka:kafka-clients:2.0.0')
compile('com.google.code.gson:gson:2.8.5')
```



- consumer

```java
public class ElasticSearchConsumer {
    
    // return: elastic search client
    public static RestHighLevelClient createClient(){
        
        // local
        // String hostname = "local";
        // RestClientBuilder builder = RestClient.builder(
        //		new HttpHost(hostname,9200,"http"));
        
        // bonsai / hosted
        Stirng hostname = "";
        Stirng username = "";
        String password = "";
        
        // credentials provider
        final CredentialsProvider credentialsProvider = 
            new BasicCredentialsProvider();
        
        credentialsProvider.setCredentials(
            AuthScope.ANY,
            new UsernamePasswordCredentials(username,password));
        
        
        RestClientBuilder.HttpClientConfigCallback callback =
            new RestClientBuilder.HttpClientConfigCallback(){
            	@Override
            	public HttpAsyncClientBuilder customizeHttpClient(
                		HttpAsyncClientBuilder httpClientBuilder){
                	return httpClientBuilder
                    	.setDefaultCredentialsProvider(
                    		credentialsProvider);
            	}
        	};
        
        RestClientBuilder builder = RestClient.builder(
        	new HttpHost(hostname,443,"https"))
            	.setHttpClientConfigCallback(callback);
        
        RestHighLevelClient client = new RestHighLevelClient(builder);
        return client;
    }
    
    // sb autoconfig ?
    public static KafkaConsumer<String,String> 
        	createConsumer(String topic){
        
        String bootstrapServers = "127.0.0.1:9092";
        Stirng groupId = "kafka-demo-elasticsearch";
        
        Properties proterties = new Properties();
        //... put
        
        KafkaConsumer<String,Stirng> consumer =
            new KafkaConsumer<>(properties);
        consumer.subscribe(Arrays.asList(topic)); // param
        
        return consumer;
    }
    
    private static JsonParser jsonParser = new JsonParser();
    
    private static String extractIdFromTweet(String tweetJson){
        // gson lib
        return jsonParser.parse(tweetJson)
            .getAsJsonObject()
            .get("id_str") // where set ?
            .getAsStirng(); // id_str 만 string 으로 가져온다 ?
        // tweet id
    }
    
    public static void main(Stirng[] args) throws IOException {
        Logger logger = LoggerFactory.getLogger(
        	ElasticSearchConsumer.class.getName());
        RestHighLevelClient client = createClient();
        
        KafkaConsumer<String,String> consumer = 
            createConsumer("twitter_tweets");
        
        while(true){
            
            ConsumerRecords<String,String> records = 
                consumer.poll(Duration.ofMillis(100));
            
            Integer recordCount = records.count();
            logger.info("received: "+recordCount);
            
            BulkRequest bulkRequest = new BulkRequest();
            
            for(ConsumerRecord<String,String> record: records){
                
                // kafka generic id
                //Stirng id = 
                //  record.topic()+record.partition()+record.offset();
                
                // twitter feed specific id
                try {
                    String id = extractIdFromTweet(record.value());
                    
                    // insert data into elasticsearch
                    IndexRequest indexRequest = 
                        new IndexRequest("tweets")
                        	.wource(record.value(), XContentType.JSON)
                        	.id(id);
                    
                    bulkRequest.add(indexRequest);                    
                    
                }catch(NullPointerException e){
                    logger.warn("skip bad data: "+record.value());
                }
            }
            
            if(recordCount > 0){
                
                BulkResponse bulkItemResponses =
                    client.bulk(bulkRequest, RequestOptions.DEFAULT);
                
                logger.info("commit offsets..");
                consumer.commitSync();
                logger.info("offsets commited");
                
                try{
                    Thread.sleep(1000);
                }catch(InterruptedExcption e){
                    e.printStackTrace();
                }
            }
        }
    }
}
```

> 단순히 consumer 로 받고,
>
> 엘라스틱서치에 client.bulk 하는 코드!
>
> 분석 알고리즘은 나의 몫

# kafka-producer-twitter

- build.gradle

```java
compile('org.apache.kafka:kafka-clients:2.0.0')
compile('com.twitter:hbc-core:2.2.0')   
```



- producer

```java
public class TwitterProducer {
    
    Logger logger = LoggerFactory.getLogger(
    		TwitterProducer.class.getName());
    
    // credentials
    String consumerKey = "";
    String consumerSecret = "";
    String token = "";
    String secret = "";
    
    List<String> terms = 
       Lists.newArrayList("bitcoin","usa","politics","sport","soccer");
    
    public TwitterProducer(){}
    
    public static void main(Stirng[] args){
        new TwitterProducer().run();
    }
    
    public void run(){
        
        logger.info("setup");
        
        BlockingQueue<String> msgQueue =
            new LinkedBlockingQueue<String>(1000);
        
        Client client = createTwitterClient(msgQueue);
        client.connect();
        
        KafkaProducer<String,String> producer = createKafkaProducer();
        
        Runtime.getRuntime().addShutdownHook(new Thread(() -> {
            logger.info("stopping application..");
            logger.info("shutting down client from twitter..");
            
            client.stop();
            producer.close();
            
            logger.info("done");
        }));
        
        // loop to send tweets to kafka
        while(!client.isDone()){
            
            String msg = null;
            try{
                msg = msgQueue.poll(5, TimeUnit.SECONDS);
            }catch(InterruptedException e){
                e.printStackTrace();
                client.stop();
            }
            
            if(msg != null){
                logger.info(msg);
                
                producer.send(new ProducerRecord<>(
                    	"twitter_tweets",null,msg),
                              new Callback(){
                    
                    @Override
                    public void onCompletion(
                        	RecordMetaData recordMetadata,
                            Exception e){
                        if(e != null){
                            logger.error("Something bad happen", e);
                        }
                    }        
                });
            }
        }
    }
    
    public Client createTwitterClient(BlockingQueue<String> msgQueue){
        
        Hosts hosebirdHosts = 
            new HttpHosts(Constants.STREAM_HOST);
        
        StatusesFilterEndpoint hosebirdEndpoint = 
            new StatusesFilterEndpoint();
        
        hosebirdEndpoint.tractTerms(terms); // 구독 키워드 list
        
        // secrets should be read from config file
        Authentication hosebirdAuth =
            new OAuth1(consumerKey,consumerSecret,
                      token,secret);
        
        ClientBuilder builder = new ClientBuilder()
            .name("Hosebird-Client-01")
            .hosts(hosebirdHosts)
            .authentication(hosebirdAuth)
            .endpoint(hosebirdEndpoint)
            .processor(new StringDelimitedProcessor(msgQueue));//param
        
        Client hosebirdClient = builder.build();
        return hosebirdClient;
    }
    
    public KafkaProducer<String,String> createKafkaProducer(){
        String bootstrapServers = "127.0.0.1:9092";
        
        Properties properties = new Properties();
        //... put
        
        KafkaProducer<String,String> producer = 
            new KafkaProducer<>(properties);
        
        return producer;
    }
}
```

# kafka-streams-tweets

- build/gradle

```cmd
compile('org.apache.kafka:kafka-streams:2.0.0')
compile('com.google.code.gson:gson:2.8.5')
```



- streams

```java
public class StreamsFilterTweets {
    
    public static void main(String[] args){
        
        // create properties
        Properties properties = new Properties();
        properties.setProperty(StreamsConfig.BOOTSTRAP_SERVERS_CONFIG,
                              "127.0.0.1:9092");
        properties.setProperty(StreamsConfig.APPLICATION_ID_CONFIG,
                              "demo-kafka-streams");
        properties.setProperty(
            StreamsConfig.DEFAULT_KEY_SERDE_CLASS_CONFIG,
        	Serdes.StringSerde.class.getName());
        properties.setProperty(
        	StreamsConfig.DEFAULT_VALUE_SERDE_CLASS_CONFIG,
        	Serdes.StringSerde.class.getName());
        
        // create topology
        StreamsBuilder streamsBuilder = new StreamsBuilder();
        
        // input topic
        KStream<String,String> inputTopic = 
            streamsBuilder.streams("twitter_tweets");
        KStream<String,String> filteredStream = inputTopic.filter(
        	(k,jsonTweet) -> 
            extractUserFollowersInTweet(jsonTweet)>10000
        );
        filteredStream.to("important_tweets");
        
        // build topology
        KafkaStreams kafkaStreams = new KafkaStreams(
        	streamsBuilder.build(), properties);
        
        // start streams application
        KafkaStreams.start();
    }
    
    private static JsonParser jsonParser = new JsonParser();
    
    private static Integer 
        	extractUserFollowersInTweet(String tweetJson){
        	// tweet 에서 user 를 추출하겠지 ?
        try{
            
            return jsonParser.parse(tweetJson)
                .getAsJsonObject()
                .get("user")
                .getAsJsonObject()
                .get("followers_count")
                .getAsInt();
            
        }catch(NullPointerException e){
            return 0;
        }
    }
}
```

# manual-consumer-offset-management

- config/LibraryEventsConsumerConfig

```java
@Configuration
@EnableKafka
public class LibraryEventsConsumerConfig {
    
    @Bean
    ConcurrentKafkaListenerContainerFactory<?,?> 
        	kafkaListenerContainerFactory(
        ConcurrentKafkaListenerContainerFactoryConfigurer configurer, 
        ConsumerFactory<Object,Object> kafkaConsumerFactory){
        
        ConcurrentKafkaListenerContainerFactory<Object,Object> factory
            = new ConcurrentKafkaListenerContainerFactory<>();
        configurer.configure(factory, kafkaConsumerFactory);        
        //factory.getContainerProperties()
        //    .setAckMode(ContainerProperties.AckMode.MANUAL);
        
        return factory;
    }
}
```



- consumer

```java
@Component
@Slf4j
public class LibraryEventsConsumer {
    
    @KafkaListener(topics={"library-events"})
    public void onMessage(
        	ConsumerRecord<Integer,String> consumerRecord){
        
        log.info("ConsumerRecord: {}", consumerRecord);
    }
}
```



- consumer/manualOffset

```java
//@Component
@Slf4j
public class LibraryEventsConsumerManualOffset
    	implements AcknowledgingMessageListener<Integer,String>{
    
    @Override
    @KafkaListener(topics={"library-events"})
    public void onMessage(
        	ConsumerRecord<Integer,String> consumerRecord,
    		Acknowledgment acknowledgment){
        
        log.info("{}", consumerRecord);
        acknowledgment.acknowledge();   
    }
}
```



# consumer & producer 모아본다면 ?

- config
  - xxConsumerConfig
  - AutoCreateConfig < 여러개면 따로, 하나이면 application 에.
- consumer // consumer + producer = kafka 로 묶어도 괜찮을듯
  - xxConsumer
- producer
  - xxProducer
- entity
  - xx
- jpa (domain)
  - xxRepository
- controller
  - xxController
- service
  - xxService

