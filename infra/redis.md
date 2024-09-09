# Redis

프로젝트에 redis 적용하며 알게된 내용 정리

<br>

## 📌 Redis Transaction

```
MULTI

...redis operation 수행

EXEC [or DISCARD]
```
사용방법은 간단하다. `MULTI`로 시작해서 `EXEC`(혹은 `DISCARD`)로 끝내면 된다.

### RDB Transaction과 다른 점

MySQL 기준으로 RDB의 Transaction과 Redis Transaction은 차이점이 존재한다.

- 조회 null 처리
multi로 시작한 상황에서 레디스 Key를 가지고 데이터 조회하려고 하면 nil을 반환한다. (null 처리)
RDB는 tx안에서 조회를 할 수 있다. 
(물론 동시에 실행되고 있는 다른 트랜잭션에서 commit 되기 이전의 데이터를 조회)

- rollback 미지원
RDB의 tx는 중간 에러에 대해서 전체 rollback을 하지만 redis tx는 중간에 에러가 발생해도 이전에 실행되었던 operation들에 대해서 rollback을 하지 않는다.

어플리케이션 개발할 때 이부분을 고려해서 개발해야 한다. 예를 들어 redis caching 처리하는 로직에서 만약 중간에 실패해서 해당 캐시에 대해 데이터 불일치를 고민해야 한다면 cache evict을 고려해볼 수 있다.

## 📌 TLS 및 user 인증 설정

redis를 외부에 오픈을 할 것이기 때문에 보안에 신경을 써야 한다. 안그러면 해커 프로그램이 나의 레디스 자원을 마음대로 사용할 수 있다.

### user 설정

redis는 기본적으로 프로그램이 실행되면 default 유저(일종의 root user)가 설정되는데 해당 유저에 password를 필히 설정해야 한다.

```conf
# redis.conf
```

### TLS 설정