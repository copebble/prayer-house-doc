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

## :pushpin: migration new replica

how to apply GTID replication setup when adding new replica db server or replace old one with new one.

> **precondition**
> The new replica database have to be in first setting. If any query executed exits, the process below might be under error.

```shell
# source db server
source:~ $ mysqldump -u[user] -p -v --all-databases \
 --quick --single-transaction --routines --set-gtid-purged=ON \
 --triggers --extended-insert --master-data=2 \
 | gzip > /home/source/backup.sql.gz
```
- `--databases [database]`: dump specific database you want
- `--all-databases`: dump all databases

```shell
source:~ $ scp -P [port] /home/source/backup.sql.gz [replica_user]@[replica_host]:/home/replica
```
transmitting back up file to replica server

```shell
replica:~ $ gunzip backup.sql.gz
replica:~ $ mysql -uroot -p
mysql> source backup.sql;
```
run back up query before setting replication.

```shell
mysql> CHANGE REPLICATION SOURCE TO...
mysql> START SLAVE;
mysql> SHOW SLAVE STATUS\G;
```
Input replication setting information, and start replica mode.

```
...
Slave_IO_Running: Yes
Slave_SQL_Running: Yes
...
```
If you see the replica status, the replication successfully is applied.

<br>

## 📌 References

- [MySQL GTID 를 사용한 Replication(복제) 설정](https://hoing.io/archives/18445)
