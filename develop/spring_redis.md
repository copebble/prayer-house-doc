# Spring에서 Redis 적용

<br>

## 📌 Spring Transaction 동기화

### configuration

redis 캐싱 처리와 kafka produce 처리는 보통 Spring Transaction에 묶이는 경우가 많다.
로직 수행중 에러 발생시 Transaction rollback 진행되는데 redis는 이미 처리가 되어 롤백이 안되는 상황 발생

이 때문에 현재 진행중인 Transaction과 일종의 동기화 설정을 해줘야 한다.

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
(참고로 RedisTemplate 같은 경우 RedisCacheManager 설정과 별개로 동작)

위와 같이 설정하면 Spring Transaction 시작시 redis `MULTI`로 트랜잭션 시작

### readOnly

`@Transactional(readOnly = true)` 같이 readOnly 속성이 부여된 트랜잭션의 경우 transaction 동기화 되지 않는다. (롤백되어도 캐싱 처리됨)
**즉, redis의 multi를 수행하지 않는다.**

### Spring @Transactional 분리

이미 상위 메소드에서 `@Transactional`로 감싸져있는 상태에서 **하위 메소드에 redis multi 기능을 사용하고 싶지 않으면 관련 설정을 해야 한다.**

참고로 하위 메소드를 `@Transactional(readOnly = true)`로 감싸도 소용이 없다. 상위 메소드에 설정되어 있는 `@Transactional`의 트랜잭션에 묶여서 처리되기 때문이다.

```kotlin
@AppTransactional(readOnly = true, propagation = Propagation.NOT_SUPPORTED)
```
`Propagation.NOT_SUPPORTED` 통해 하위 메소드에서는 상위 메소드와 별개로 Transactional을 사용하지 않겠다는 옵션을 추가 설정해야 한다.