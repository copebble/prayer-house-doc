# MySQL installation

> raspberry pi 서버 MySQL 설치

raspberry pi os 스펙은 다음과 같다.
- debian 계열 OS
- ARM, 64-bit

기본적으로 apt package 툴로 설치하려고 하면 `mariadb-server`만 존재한다.
`mysql-server`는 지원을 안하는 것 같아 보임

본인은 mysql 사용하고 싶었기에 mysql 공식 페이지에서 제공하는 **tar 파일**로 직접 설치하기로 결정

<br>

## :pushpin: 파일 다운로드

https://downloads.mysql.com/archives/community/

- 원하는 mysql 버전
- OS: Linux - Generic
- OS Version: Linux - Generic (glibc 2.28) (ARM, 64-bit)

다운 받은 파일을 그대로 scp 사용해서 원격 서버에 전송
(본인은 `/usr/local/mysql` 경로에 설정)

> Raspberry pi OS version을 잘 확인해서 알맞는 mysql 설치해야 한다.

scp로 tar 파일 이동 후 압축 풀면 된다.

<br>

## :pushpin: MySQL 설정 및 실행

```shell
sudo useradd mysql
```
mysql 프로그램 실행 용도의 user를 생성한다.

```shell
sudo mkdir -p /var/lib/mysql/data

sudo chown -R mysql:mysql /var/lib/mysql
sudo chown -R mysql:mysql /var/lib/mysql/data
sudo chown -R mysql:mysql /var/log/mysql
sudo chown -R mysql:mysql /var/run/mysqld
```
mysql 실행시 사용되는 디렉토리에 대해 권한을 부여한다.

```shell
sudo /usr/local/mysql/bin/mysqld --initialize-insecure \
--user=mysql \
--basedir=/usr/local/mysql \
--datadir=/var/lib/mysql/data
```
`--basedir`: mysql tar 압축 해제했던 경로로 설정
`--datadir`: 따로 설정한 data용 디렉토리

위의 작업을 진행하면 기본적인 mysql 설정 완료

```shell
sudo vim /etc/my.cnf
```
`/etc/my.cnf` 파일을 대상으로 기본적인 mysql 설정을 진행하게 된다.
mysql 초기 실행시 설정하고 싶은 옵션이 있으면 해당 파일에 추가하면 된다.

```shell
sudo /usr/local/mysql/bin/mysqld_safe --user=mysql &

ps -ef | grep mysql
```
mysql을 실행하고 해당 프로세스가 잘 떴는지 확인

```shell
sudo ln -s /usr/local/mysql/bin/mysql /usr/bin/mysql
mysql --socket=/var/lib/mysql/mysql.sock
```
link 걸고 mysql 명령어를 통해 접속을 시도한다.
mysql 직접 설치시 default socket 파일에 내용이 없기 때문에 socket 파일을 직접 지정해야 한다.

```shell
# 위에가 불편하면
sudo ln -s /var/lib/mysql/mysql.sock /tmp/mysql.sock
mysql -uroot -p
```
매번 `--socket` 옵션을 추가해서 mysql 접속하는 것은 불편하기 때문에 default socket file에 link 걸면 mysql 명령어 만으로 접속 가능

<br>

## :pushpin: 그 외 기타
```shell
sudo ./bin/mysqladmin shutdown -uroot -p
```
혹여나 프로세스 종료하고 싶으면 mysqladmin 프로그램 통해 shutdown 하면 된다.