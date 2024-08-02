# MySQL 관련 트러블 슈팅

<br>

## 📌 Encoding

mysql 서버 내부에서 데이터 조회시 한글 깨짐 문제

```sql
mysql> SHOW VARIABLES LIKE 'c%';
+----------------------------------------------+--------------------------------+
| Variable_name                                | Value                          |
+----------------------------------------------+--------------------------------+
| caching_sha2_password_auto_generate_rsa_keys | ON                             |
| caching_sha2_password_digest_rounds          | 5000                           |
| caching_sha2_password_private_key_path       | private_key.pem                |
| caching_sha2_password_public_key_path        | public_key.pem                 |
| character_set_client                         | latin1                         |
| character_set_connection                     | latin1                         |
| character_set_database                       | utf8mb4                        |
| character_set_filesystem                     | binary                         |
| character_set_results                        | latin1                         |
| character_set_server                         | utf8mb4                        |
| character_set_system                         | utf8mb3                        |
| character_sets_dir                           | /usr/share/mysql-8.0/charsets/ |
| check_proxy_users                            | OFF                            |
| collation_connection                         | latin1_swedish_ci              |
| collation_database                           | utf8mb4_unicode_ci             |
| collation_server                             | utf8mb4_0900_ai_ci             |
| completion_type                              | NO_CHAIN                       |
| concurrent_insert                            | AUTO                           |
| connect_timeout                              | 10                             |
| connection_memory_chunk_size                 | 8912                           |
| connection_memory_limit                      | 18446744073709551615           |
| core_file                                    | OFF                            |
| create_admin_listener_thread                 | OFF                            |
| cte_max_recursion_depth                      | 1000                           |
+----------------------------------------------+--------------------------------+
```

my.cnf 파일에 다음 내용 추가(source, replica 둘 다)
```
[mysqld]
collation-server            = utf8_unicode_ci
init-connect                = 'SET NAMES utf8'
character-set-server        = utf8

[mysql]
default-character-set       = utf8

[client]
default-character-set       = utf8
```

<br>

## 📌 MySQL Replication 관련 트러블 슈팅

replication 환경에서 개발하면서 마주했던 트러블에 대한 기록

### MySQL Replication 중단

replica 기능 잘 사용하다가 어느 순간 replica 연결이 끊기는 상황 종종 발생
replica database 접속해 로그 확인
```
Coordinator stopped because there were error(s) in the worker(s). 
The most recent failure being: Worker 1 failed executing transaction 'NOT_YET_DETERMINED' at source log mysql-bin.000061, 
end_log_pos 71303318. See error log and/or performance_schema.
replication_applier_status_by_worker table for more details about this failure or others, if any.
```

(작성 필요)

### References
- [MySQL Replication(복제) 구성 및 설정 - Async - Semi Async](https://hoing.io/archives/3111)
- [MySQL GTID 를 사용한 Replication(복제) 설정](https://hoing.io/archives/18445)