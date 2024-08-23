# Spring Reactive & Database 연동

<br>

## 📌 Entity 작성 방법

```kotlin
@Table(schema = "prayerhouse", name = "follow_requests")
class FollowRequestR2Entity private constructor(
    askMemberId: Long,
    followeeId: Long,
    status: FollowRequestStatus
) : BaseUpdatableEntity() {
    @Id
    @Column("id")
    var id: Long = 0L
        protected set

    @Column("ask_member_id")
    val askMemberId: Long = askMemberId

    @Column("followee_id")
    val followeeId: Long = followeeId

    @Column("status")
    val status: FollowRequestStatus = status
}
```
- 식별자 

`@Id`에 대해서는 setter가 있어야 한다는 점 유의해야 한다.
최대한 capsulation을 위해 `protected`로 선언해 외부에서는 쉽게 변경하지 못하도록 했다.

- `@Column`

JPA에서 제공하는 `Column`과 비교했을 때 상당히 빈약한 옵션을 제공한다.
**기본적으로 column_name만 작성할 수 있다.** 그 외에 length, nullable, columnDefinition 등의 옵션을 제공하지 않아 entity에 db field 속성을 모두 명시할 수 없다는 점이 아쉽긴 하다.

