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

<br>

## 📌 Redis Lock 관련 내용

애플리케이션에 redis lock 적용할 때 
성능상 spin lock은 계속해서 redis 조회를 해서 redis에 부하를 줄 수 있다는 내용을 보았고 Pub-Sub 형식의 **Redisson lock**을 사용하기로 결정

### 주의할 점

Redisson Lock에서 getLock 이후 try lock 하는 과정이 있다.

```kotlin
val rLock = redissonClient.getLock(key)

rLock.tryLock(ttl, ttl, TimeUnit.MILLISECONDS)    
```
`tryLock` 메소드에는 두 번째 인자로 leaseTime을 받게 되는데 설정된 시간 만큼 lock 소유권을 보장해주는 옵션이다. 
즉 3000 millisec 설정했다면 해당 스레드에서 3초 동안은 락을 보장해준다는 것이다.

여기서 중요하게 고려해야하는 부분이 바로 여기서 발생하게 된다.

락을 획득한 이후 수행되는 비즈니스 처리 과정이 LeaseTime보다 더 오래 걸린다면,
race condition에 놓일 수 있다. (동시성 이슈 발생 가능)

```
Tx A. Lock 획득(3초) ----------- A 로직 수행 (5초 소요) >> 같은 자원 *Account* 접근 ----------- 
Tx B. Lock wait --- 3초 뒤 (Lock 풀림) Lock 획득 --- B 로직 수행 >> 같은 자원 *Account* 접근 ---
```
위의 과정 처럼 본래 동시성 이슈를 해결하기 위해 lock을 걸었는데 결국 동시성 이슈가 발생할 수 있게 된 것이다. **왜냐하면 Tx A에서 leaseTime을 3초 설정해도 lock만 풀리는 것이지 해당 로직을 그대로 수행하기 때문이다.**

**이래서 설정한 leaseTime 보다 로직 소요시간이 더 적게 걸리도록 개발을 하는 것에 유의해야 한다.**
(특히 DB, 외부 애플리케이션 통신 등 외부 통신 과정에서 지연에 대해 timeout 설정을 잘해야 될 듯)

### AOP 관련

보통 Redis Lock을 Spring AOP 이용해서 Aspect 클래스에다가 구현을 할 것이다.

```kotlin
@Aspect
@Component
@Order(Ordered.HIGHEST_PRECEDENCE)  // @Transactional 보다 앞서 실행되기 위함
class DistributedLockAspect(
    //...
) {...}
```
`@Transactional`도 AOP 기반으로 동작하기 때문에 우선순위를 @Transactional보다 앞서 설정해야 한다.
왜냐하면 불필요한 Tx 잡기 전에 Redis Lock부터 검증하려고 하기 위해서다.
(Redis Lock 획득할 때만 본 로직이 수행되기 때문에 이후에 Tx 잡도록 설정하기 위함)