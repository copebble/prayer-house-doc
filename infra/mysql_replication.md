# MySQL Replication

MySQL Replicaton에는 크게 binary log position 방식과 GTID 방식이 있는데 GTID 방식을 선택하였다.

(이유 설명은 추후 추가, GTID & Log Pos 방식의 차이)

<br>

## 📌 구성

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

<br>

## 📌 Trouble shoot

복제 실패하는 경우가 발생

```mysql
# replica
show slave status\G
```

```mysql
# replica
select * from performance_schema.replication_applier_status_by_worker\G;
```
어느 지점에서 복제 실패나왔는지 상세 로그 확인

```mysql
# replica
set gtid_next='[문제 발생한 GTID]';
begin;
commit;
set gtid_next='AUTOMATIC';
```
항상 마무리는 AUTOMATIC으로 다시 원복
이렇게 해서 에러난 지점은 스킵하는 방식으로 처리 가능

<br>

## 📌 References

- [MySQL GTID 를 사용한 Replication(복제) 설정](https://hoing.io/archives/18445)

<br>

## 📌 (추가) Raspberry pi 서버 MySQL 설치

raspberry pi os 스펙은 다음과 같다.

- debian 계열 OS
- ARM, 64-bit

기본적으로 apt package 툴로 설치하려고 하면 `mariadb-server`만 존재한다.
`mysql-server`는 지원을 안하는 것 같아 보임

본인은 mysql 사용하고 싶었기에 mysql 공식 페이지에서 제공하는 **tar 파일**로 직접 설치하기로 결정

### 파일 다운로드

https://downloads.mysql.com/archives/community/

- 원하는 mysql 버전
- OS: Linux - Generic
- OS Version: Linux - Generic (glibc 2.28) (ARM, 64-bit)

다운 받은 파일을 그대로 scp 사용해서 원격 서버에 전송
