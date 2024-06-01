# Reactive Kafka 개발 기록

spring webflux(reactive) & kafka 개발 기록

## :pushpin: MAX_POLL_RECORDS 설정

>✅ reactive kafka consuming 추구하고자 했던 목표
> 
> - 비동기를 이용해 chunk size로 kafka에서 message를 가져와서 한 꺼번에 async & non-blocking 처리 이후 kafka batch commit 하는 비동기 consumer 처리 개발
> - 또 중요한 것은 db connection pool 고갈 등의 문제를 방지하고자 publisher, subscriber 간에 backpressure 설정 필요

```kotlin
val properties = kafkaProperties.buildConsumerProperties(null)
    .toMutableMap<String, Any>()
    .apply {
        putAll(
            mapOf(
                ConsumerConfig.MAX_POLL_RECORDS_CONFIG to 5,
                ConsumerConfig.GROUP_ID_CONFIG to consumerGroupId,
                "spring.json.value.default.type" to T::class.java
            )
        )
    }
```
`MAX_POLL_RECORDS_CONFIG`(max.poll.records): 한 번에 kafka에서 받아올 레코드 개수 설정

5개로 설정하면 5개씩 kafka topic에서 꺼내서 consuming

하지만 위 설정만으로 부족
reactor에서 publish, subsriber간에 처리 속소 간극 발생, 5개씩 꺼내서 사용하는 게 무의미할 정도로 producing된 모든 메시지들을 한꺼번에 꺼내 publishing 하는 상황도 발생 가능
**이렇게 되면 db connection pool에서 connection 고갈 등 다른 문제를 발생시킬 수 있음**

> publish, subscriber 간 backpressure 설정이 필요하게 됨

```kotlin
.subscribe(object : BaseSubscriber<PrayerLikeCancelEvent>() {
    override fun hookOnSubscribe(subscription: Subscription) {
        request(REQUEST_BATCH_SIZE)
    }

    override fun hookOnNext(value: PrayerLikeCancelEvent) {
        CoroutineScope(Dispatchers.Default).launch {
            try {
                cancelPrayerLikeUseCase.execute(value)
            } catch (e: NoRetryEventException) {
                logger.error { "NoRetryEventException: ${e.message}" }
            }

            logger.info { "hookOnNext >> $value" }
            bufferCount.incrementAndGet()

            if (bufferCount.get() >= REQUEST_BATCH_SIZE) {
                bufferCount.set(0)
                request(REQUEST_BATCH_SIZE)
            }
        }
    }
})
```
subscribe에 BaseSubscriber를 구현해서 publisher에 request 레코드 개수를 지정할 수 있다. 이렇게 되면 Coroutine 안에서 수행되는 suspend 함수의 내용이 끝나고 bufferCount가 batch size에 도달하는 경우에 새로 해당 개수만큼 reqeust를 publisher에 요구하게 된다.

```kotlin
private val bufferCount = AtomicInteger(0)
```
대신에 bufferCount는 atomic이 보장되어야 하기에 `AtomicInteger`를 사용

## :pushpin: References

- [Reactive Kafka Sender & Kafka Receiver](https://godekdls.github.io/Reactor%20Kafka/whatsnewinreactorkafka120release/)
  - reactive kafka 개발하는데 있어서 좋은 참고 자료
- https://github.com/reactor/reactor-kafka/issues/214
  - spring reactive kafka consumer 관련 설정 내용에 대한 논의(참고용)