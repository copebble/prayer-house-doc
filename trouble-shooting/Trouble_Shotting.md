- 직면했던 문제의 상황과 해결 방안에 대한 기록

<br>

## 📌 기도 제목 좋아요와 좋아요 취소 consuming 순서 이슈

<br>

## 📌 Spring Kafka consuming retryable 전략

<br>

## 📌 외부 모듈과 Spring Transaction 동기화

### Redis

redis 캐싱 처리와 kafka produce 처리는 보통 Spring Transaction에 묶이는 경우가 많다.
로직 수행중 에러 발생시 Transaction rollback 진행되는데 redis, kafka는 이미 처리가 되어 롤백이 안되는 상황 발생

이 때문에 현재 진행중인 Transaction과 일종의 동기화 설정을 해줘야 한다.
(롤백되면 redis 캐싱, kafka produce 모두 수행 안되도록)

- Redis & Cache
```kotlin
return RedisCacheManager
    .builder(redisConnectionFactory)
    .cacheDefaults(redisCacheConfiguration)
    .transactionAware() // 진행중인 Transaction 동기화 (rollback 대비)
    .build()

return RedisTemplate<String, Any>().apply {
    this.keySerializer = StringRedisSerializer()
    this.valueSerializer = StringRedisSerializer()
    this.connectionFactory = redisConnectionFactory
    this.setEnableTransactionSupport(true)  // 진행중인 Transaction 동기화
}
```
Redis 관련 Config에서 cacheManager bean에는 `transactionAware` 속성 추가, RedisTemplate에는 `setEnableTransactionSupport` true 설정해야 진행중인 Transaction과 동기화 된다.

RedisTemplate 같은 경우 RedisCacheManager 설정과 별개로 동작하고,
`@Transactional(readOnly = true)` 같이 readOnly 속성이 부여된 트랜잭션의 경우 transaction 동기화 되지 않는다. (롤백되어도 캐싱 처리됨)

위와 같이 설정하면 Transaction 시작시 redis `MULTI`로 트랜잭션 시작(더 알아봐야 될 듯)

### Kafka

```kotlin

@TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
fun handleRegisterNewPrayerEvent(event: PrayerLapseEvent.RegisterNewPrayer) {
    prayerLapsePublishService.publishFirstRegistration(event)
}
```
위와 같이 @TransactionalEventListener 사용해서 진행중인 트랜잭션이 커밋된 이후에 이벤트가 발송되도록 처리  
위 방식이 아닌 kafkaProducer 자체에 transaction prefix 설정하는 것이 있는데 트랜잭션 롤백되어도 일단 카프카에 발송이 된다는 점에서 해당 방식 사용 X