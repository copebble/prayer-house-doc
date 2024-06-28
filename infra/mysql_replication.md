# MySQL Replication

MySQL Replicaton에는 크게 binary log position 방식과 GTID 방식이 있는데 GTID 방식을 선택하였다.

(이유 설명은 추후 추가, GTID & Log Pos 방식의 차이)

<br>

## :pushpin: 구성

GTID 방식은 log position 방식에 비해 비교적 단순

DB 인스턴스 시작 전에 `my.cnf` 설정파일에서 다음과 같은 설정 필요

```
# SOURCE(MASTER)
[mysqld]
log_bin                     = mysql-bin
# binary log 파일 format 설정
binlog_format               = ROW

# replica와 다르게 설정해야 함(중요)
server-id                   = 100

# GTID REPLICA
gtid_mode                   = ON
enforce-gtid-consistency    = ON

# save inner binlog
log_replica_updates         = ON

expire_logs_days            = 10

# REPLICA(SLAVE)
[mysqld]
log_bin                     = mysql-bin
# binary log 파일 format 설정
binlog_format               = ROW

# source와 다르게 설정해야 함(중요)
server-id                   = 200

# GTID REPLICA
gtid_mode                   = ON
enforce-gtid-consistency    = ON

# save inner binlog
log_replica_updates         = ON
# replica는 read 목적으로만 설정
read_only

expire_logs_days            = 10
```

mysql 구동 시작

```sql
-- source-db root 계정 접속
CREATE USER 'repl'@'%' IDENTIFIED BY 'repl';
GRANT REPLICATION SLAVE, REPLICATION CLIENT ON *.* TO 'repl'@'%';

FLUSH PRIVILEGES;
```
replication 전용 mysql user 생성

```sql
-- replica-db root 계정 접속
RESET MASTER;

-- GTID 방식
CHANGE REPLICATION SOURCE TO
       MASTER_HOST='source-db',
       MASTER_USER='repl',
       MASTER_PASSWORD='repl',
       SOURCE_AUTO_POSITION=1;

START SLAVE;
SHOW SLAVE STATUS\G
```
`MASTER_HOST`는 참고로 ip 주소가 될 수도 도메인 주소가 될 수도 있음
(docker compose 환경에서 구성했기에 docker 내부 DNS에 의존, container 이름으로 설정 가능)

`SOURCE_AUTO_POSITION=1` gtid 방식으로 position auto 설정 추가

`START SLAVE;` or `START REPLICA;` replication 시작

### References

- [MySQL GTID 를 사용한 Replication(복제) 설정](https://hoing.io/archives/18445)