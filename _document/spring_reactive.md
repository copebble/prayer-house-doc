# Spring Reactive

event consuming application을 Spring Reactive(Webflux) 환경에서 개발하면서 느낀 지극히 개인적인 생각을 담은 노트


## :pushpin: Reactive Redis

### Transaction Support 부재

Spring MVC 환경에서 사용하던 `RedisTemplate`은 설정할 때 transactionSupport 옵션을 통해 Spring Transaction과 동기화할 수 있다.

```kotlin
return RedisTemplate<String, Any>().apply {  
    this.keySerializer = stringSerializer  
    this.valueSerializer = stringSerializer  
    // hash 타입 대응  
    this.hashKeySerializer = stringSerializer  
    this.hashValueSerializer = stringSerializer  
    this.connectionFactory = redisConnectionFactory  
    this.setEnableTransactionSupport(true)  // 진행중인 Transaction 동기화  
}
```
즉 하나의 Spring Transaction 안에서 commit되면 redis 캐싱되고 rollback 되면 같이 캐시내용이 롤백되는 트랜잭션 동기화 기능이라 true로 설정하면 데이터 불일치를 어느정도 예방할 수 있어 개인적으로 필수 설정 옵션이라 생각했다.

하지만 Spring Reactive 환경에서는 이러한 설정을 지원해주지 않는 것 같다. 어떻게 보면 비동기 방식으로 이루어지기 때문에 하나의 트랜잭션 안에서 관리하는 것이 어렵게 보일 수 있으나 Spring Transaction을 통한 db도 비동기 환경에서 하나의 tx로 관리가 되는 것처럼 redis도 같이 관리되면 좋을 것 같다는 아쉬운 마음이 컸다.

현재 이를 보완하기 적용한 내용은 try ~ catch로 잡는 방법이고, 또 하나의 방법은 Redis 고유의 transaction을 사용하는 방법이 있을 것 같다.

```kotlin
try {  
    // 기도 제목 좋아요 없는 상태인지 체크  
    checkPrayerLikeExistence(  
        prayerLike = prayerLike,  
        mustBeExisted = false  
    )  
  
    return prayerLikeStorePort.store(prayerLike).also {  
        // 캐시화  
        prayerLikeCachePort.cachePrayerLiked(  
            memberId = it.memberId,  
            alreadyLikedMap = mapOf(it.prayerLapseId to true)  
        )    }  
} catch (e: Exception) {  
    if (e !is NoRetryEventException) {  
        // 에러 발생시 cache evict  
        prayerLikeCachePort.deletePrayerLiked(  
            memberId = prayerLike.memberId,  
            prayerLapseId = prayerLike.prayerLapseId  
        )  
    }
  
    throw e  
}
```
하지만 위의 방식도 완전하지 않다. cache 처리는 됐는데 그 이후에 실패가 되어 delete 처리 진행되어야 하는데 그 과정에서 에러가 발생했다면 caching만 되고 delete는 되지 못한 그런 상황 발생 가능하다고 생각

이부분까지 고려한다면 사실 다른 방식을 고려해야될 것 같다.

결론은 Spring Reactive 방식에서 Redis 사용하기에 지원하지 않은 여러 편의 기능이 꽤 존재해서 고민하며 직접 구현해야하는 불편함이 상당수 존재하는 것 같다.