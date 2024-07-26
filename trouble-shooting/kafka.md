# Kafka 관련 trouble shooting

<br>

## 📌 Spring Transaction 동기화 문제

```kotlin

@TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
fun handleRegisterNewPrayerEvent(event: PrayerLapseEvent.RegisterNewPrayer) {
    prayerLapsePublishService.publishFirstRegistration(event)
}
```
위와 같이 @TransactionalEventListener 사용해서 진행중인 트랜잭션이 커밋된 이후에 이벤트가 발송되도록 처리  
위 방식이 아닌 kafkaProducer 자체에 transaction prefix 설정하는 것이 있는데 트랜잭션 롤백되어도 일단 카프카에 발송이 된다는 점에서 해당 방식 사용 X

<br>

## 📌 Spring Reactive Kafka error handling

reactive kafka error handling은 일반적인 kafka consumer에서 사용하는 error handling과 상당히 다르다.

### Deserialize exception

기존 consumer 설정

```yaml
# application.yaml
kafka:
  consumer:
    auto-offset-reset: earliest
    key-deserializer: org.apache.kafka.common.serialization.StringDeserializer
    value-deserializer: org.springframework.kafka.support.serializer.JsonDeserializer
    properties:
      spring.json.use.type.headers: false
      spring.json.trusted.packages: "*"
```

```kotlin
val properties = kafkaProperties.buildConsumerProperties(null)
  .toMutableMap<String, Any>()
  .apply {
    this@apply.putAll(
      mapOf(
        // deserializer 설정하는데 클래스 이름이 필요('::class.java.name' 설정 필요)
        ConsumerConfig.MAX_POLL_RECORDS_CONFIG to pollSize,
        ConsumerConfig.GROUP_ID_CONFIG to getConsumerGroup(retryIndex),
        JsonDeserializer.VALUE_DEFAULT_TYPE to eventClazz   // 필수
      )
    )
  }
```
위와 같이 `spring.json.value.default.type` 속성에 event class type 설정

만약 kafka topic에 해당 event type에 맞지 않는 데이터가 들어왔을 때 다음과 같은 에러가 발생된다.

```
org.apache.kafka.common.errors.RecordDeserializationException: 
Error deserializing key/value for partition prayerhouse-prayerlike-like-0 at offset 2726. 
If needed, please seek past the record to continue consumption.
```
문제는 kafka consumer를 reactor 방식으로 구현했기 때문에
```kotlin
consumerTemplate
	.receiveAutoAck()
	//...
```
위와 같은 방식으로 Flux publisher를 사용했다면 `receiveAutoAck` 아래에 event가 더이상 전달되지 않고 그대로 consumer가 중단되는 사태가 발생 (**상당히 치명적인 오류**)
심지어 kafka topic에 새로운 이벤트가 들어가도 이후 진행이 안된다.
(**애플리케이션 재실행해도 오류난 offset이 commit 된 상태가 아니기 때문에 다시 해당 offset부터 레코드를 꺼낼 것이고 또 다시 진행이 안되는 문제 발생**)

즉 `receiveAutoAck` 하기 이전에 deserialize 단계에서 발생한 오류라서 해당 오류를 사전에 잡는 것이 중요. 

```kotlin
mapOf(  
    ConsumerConfig.MAX_POLL_RECORDS_CONFIG to pollSize,  
    ConsumerConfig.GROUP_ID_CONFIG to getConsumerGroup(retryIndex),  
    ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG to ErrorHandlingDeserializer::class.java.name,  
    ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG to ErrorHandlingDeserializer::class.java.name,  
    ErrorHandlingDeserializer.KEY_DESERIALIZER_CLASS to StringDeserializer::class.java.name,  
    ErrorHandlingDeserializer.VALUE_DESERIALIZER_CLASS to JsonDeserializer::class.java.name,  
    JsonDeserializer.VALUE_DEFAULT_TYPE to eventClazz   // 필수  
)
```
ErrorHandlingDeserializer를 사용해 위임 deserializer 선언해준다. 
중요한 것은 `ErrorHandlingDeserializer::class` 이렇게 하면 설정 단계에서 인식을 못하기 때문에 클래스 이름 자체를 설정해주는 것이 좋다.

위와 같이 설정하면 `JsonDeserializer`가 먼저 수행되고 만약 deserialize 단계에서 오류가 발생하면 `null`로 내보내게 된다. 

즉 consumer reactor 단계에서 Record에 대한 null 처리만 제대로 해주면 된다.

```kotlin
publisher  
    .receiveAutoAck()  
    .flatMap { record ->  
        record.value()?.let {  
            Mono.just(it)  
        } ?: Mono.error(ConsumerRecordException("record null exception"))  
    }  
    .onErrorContinue { throwable, value ->  
        val record = value as ConsumerRecord<*, *>  
        logger.error { ">> Error: ${record.topic()}, ${record.offset()}, ${record.key()} ${throwable.message}" }  
        // 말도 안되는 상황이기에 바로 DLT topic 전송  
        // ...
    }
```
flatMap을 통해 정상적인 record에 대해서 `Mono.just`로 아래에 데이터를 흘려보내주고 null이면 `Mono.error`로 에러 생성해야 한다.
