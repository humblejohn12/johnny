# wisetracker 솔루션 설치 스크립트          

>이 문서는 wisetracker 솔루션 설치 문서입니다. 테스트 서버에 설치한다는 가정으로 스크립트가 작성이 되어 있습니다.  
>실제 설치 시에는 설치 환경에 맞게 내용을 수정하시기 바랍니다. 

## 목차
1. 사전작업 및 기타설정
1. 설치
1. 테스트
1. 별도 적용 쿼리 및 기타 서버 설정
1. 기타사항(설치 이외 부가적인 서버 관리 내용) 


             
# 1. 사전작업 및 기타설정

### 사전작업
	
##### 1) 테이블들( INS_IDA ,INS_IDR1 ,INS_IDU )을 확보
테스트 서버에 설치하는 경우 전달 받은 SolutionInstall 폴더 안에 InstallFiles라는 폴더가 있고, 
그 안에 **ida.sql, idr.sql, idu.sql** 이라는 3가지 sql 파일을 가지고 있는 sql.zip 파일이 있다. 
이 파일들 안의 테이블을 확보하라는 이야기이다. **테스트 서버에 설치 시에는 전달 받은 파일 확인하는 것으로 이 단계 완료**  

##### 2) xManager 에서 받은 profile.sql파일에서 DB명과 수집도메인 또는 아이피를 변경  
profile은 고객에게 제공하는 서비스 이름을 의미하고 profile.sql은 해당 서비스 DB 데이터를 가지고 있는 sql 파일을 지칭한다.
테스트 서버에 설치시에는 전달 받은 105_WT_SaaS.sql 파일(sql.zip 안에 포함되어 있음)을 profile.sql로서 사용한다.  

   * **TA_DB**: TA_DB 테이블 안의 DB_NM을 사용하는 DB 이름으로 수정해준다. 
   비활성화(IS_ACTIVE = ACT002)되어 있는 DB가 TA_PROFILE_DB 테이블에 있지 않은지 확인 후 삭제한다. 
   일반적으로 **DB_NO가 1,2,11인 것만 활성화** 되어 TA_PROFILE_DB 테이블에 등록이 되어 있어야 한다.              

   * **TA_SERVER**: TA_SERVER 테이블에 있는 SERVER_NO가 1인 SERVER_HOST를 현재 작업하는 서버의 도메인으로 바꾸어 준다. 

##### 3) xManager 에서 받은 profile.sql 파일에서 키값이 걸려있는 테이블을 먼저 지워도 키로 인해 비울 수 없다는 메세지가 나온다.  
이 때 이하 쿼리문 실행하면 된다.

```
SET FOREIGN_KEY_CHECKS=0;  <--추가
TRUNCATE TABLE TA_CODE_DTL; 
TRUNCATE TABLE TA_CODE;
SET FOREIGN_KEY_CHECKS=1; <--추가
```

> sql 파일은 UTF-8 언어셋으로 되어 있어야 한다.  
> 만약 sql 파일이 UTF-8 언어셋이 아니라면 sublime같은 에디터를 통해 UTF-8로 인코딩하면 된다. 


### Server Time Sync

```
crontab -e

+---------------------------------------------------------------+
1 0 * * * /usr/sbin/ntpdate time.bora.net > /dev/null 2> /dev/null
+---------------------------------------------------------------+
```


### 서버언어셋

```
vi /etc/profile

---[Top-----------------------------------------
####### Wisetracker Solution Setup #######
export JAVA_HOME=/home/wisetracker/java
export LANG=ko_KR.UTF-8
export MYSQL_HOME=/home/wisetracker/mysql
export PHANTOMJS_HOME=/home/wisetracker/phantomjs
export PATH=$JAVA_HOME/bin:$PATH:$MYSQL_HOME/bin:$PHANTOMJS_HOME/bin

alias vi='vim'
alias grep='grep --colour=auto'
alias debugt='tail -fn200 /home/wisetracker/tomcat/logs/catalina.out'
alias wisetracker='sh /home/wisetracker/server/wisetracker_UTIL'
----------------------------------------Bottom]-

source /etc/profile
```

### 계정 및 설치파일 업로드

```
useradd wisetracker
passwd wisetracker
```

> 원래 /home 밑에 wisetracker라는 디렉토리가 없었는데 계정 생성 과정에 wisetracker라는 디렉토리가 생긴다.



```
su - wisetracker
mkdir -p /home/wisetracker/work
mkdir -p /home/wisetracker/server
cd /home/wisetracker/work
tar xvfpz InstallFile.tar.gz
```

> InstallFile.tar.gz은 전달 받은 파일 중에 있으며 scp 명령어를 통해서 로컬 컴퓨터에서 서버로 파일을 가지고 온다.  
> ex) scp InstallFile.tar.gz root@175.158.15.226:/home/wisetracker/work





# 2. 설치          


### 설치 순서

1) MySQL 설치 
2) JDK7 설치 
3) Tomcat 설치 
4) war(TRACKER, infographic) 배포 
5) phantomjs 설치 
6) 크론 배포


### 1) MySQL 
설치중 에러시 고려  


```
yum install perl-DBI*
yum -y install libtermcap-devel ncurese-devel
yum -y install bison* 

(64bit일때만) : yum -y install gcc.x86_64 gcc-c++.x86_64 wget.x86_64 bzip2-devel.x86_64 pkgconfig.x86_64 openssl-devel.x86_64 make.x86_64 man.x86_64 nasm.x86_64 gmp.x86_64 gdbm-devel.x86_64 readline-devel.x86_64 compat-readline43.x86_64 ncurses-devel.x86_64 db4-devel.x86_64 automake* autoconf* 

yum -y install cmake* 
또는(64bit일때) rpm -Uvh /home/wisetracker/work/source/cmake-2.6.4-5.el6.x86_64.rpm
```



본격 설치 과정  


```
userdel mysql
useradd -M mysql    
mkdir -p /data55
chown -R mysql:mysql /data55


mv /etc/my.cnf /etc/my.cnf.bak
cp /home/wisetracker/work/config/my.cnf.5.5 /etc/my.cnf   
*vi /etc/my.cnf으로 설정값 확인(최소 datadir = /data55을 확인해줘야 한다. 이게 안되어 있으면 프로그램이 제대로 작동하지 않는다.) 

cd /home/wisetracker/work/source
tar xvfpz mysql-5.5.36.tar.gz
cd mysql-5.5.36

cmake -DCMAKE_INSTALL_PREFIX=/home/wisetracker/mysql-5.5.36 -DMYSQL_DATADIR=/data55 -DSYSCONFDIR=/etc -DDEFAULT_CHARSET=utf8 -DDEFAULT_COLLATION=utf8_general_ci -DWITH_EXTRA_CHARSETS=euckr -DWITH_INNOBASE_STORAGE_ENGINE=1 -DWITH_MYISAM_STORAGE_ENGINE=1 -DWITH_HEAP_STORAGE_ENGINE=1 -DWITH_PARTITION_STORAGE_ENGINE=1 -DWITH_FEDERATED_STORAGE_ENGINE=1 -DMYSQL_TCP_PORT=3306 -DMYSQL_UNIX_ADDR=/tmp/mysql.sock -DENABLED_LOCAL_INFILE=ON  

make && make install

ln -s /home/wisetracker/mysql-5.5.36 /home/wisetracker/mysql
chown -R mysql:mysql /home/wisetracker/mysql-5.5.36
chown -R mysql:mysql /data55

cd /home/wisetracker/mysql/scripts
./mysql_install_db --user=mysql --basedir=/home/wisetracker/mysql --datadir=/data55
chown -R mysql:mysql /data55

cp /home/wisetracker/work/source/mysql-5.5.36/support-files/mysql.server /etc/init.d/mysql
chmod 755 /etc/init.d/mysql
chkconfig --add mysql
chkconfig --level 345 mysql on

vi +/datadir= /etc/init.d/mysql
+----------------------------------+
basedir=/home/wisetracker/mysql
datadir=/data55
+----------------------------------+

ln -s /home/wisetracker/mysql/bin/mysql.server /usr/bin/
ln -s /home/wisetracker/mysql/bin/mysql /usr/bin/
ln -s /home/wisetracker/mysql/bin/mysqldump /usr/bin/
ln -s /home/wisetracker/mysql/bin/mysqladmin /usr/bin/

service mysql start 

mysql
use mysql
GRANT ALL PRIVILEGES ON *.* TO '2n9soft'@'localhost' IDENTIFIED BY '2n9soft' WITH GRANT OPTION;
GRANT ALL PRIVILEGES ON *.* TO 'wisetracker'@'localhost' IDENTIFIED BY 'wisetracker' WITH GRANT OPTION;
GRANT ALL PRIVILEGES ON *.* TO '2n9soft'@'%' IDENTIFIED BY '2n9soft' WITH GRANT OPTION;
GRANT ALL PRIVILEGES ON *.* TO 'wisetracker'@'%' IDENTIFIED BY 'wisetracker' WITH GRANT OPTION;
GRANT ALL PRIVILEGES ON *.* TO '2n9soft'@'211.239.123.%' IDENTIFIED BY '2n9soft' WITH GRANT OPTION;
GRANT ALL PRIVILEGES ON *.* TO 'wisetracker'@'211.239.123.%' IDENTIFIED BY 'wisetracker' WITH GRANT OPTION;
UPDATE user SET password=old_password('2n9soft');
flush privileges;

select Host,User,Password from user;
+---------------+-------------+------------------+
| Host          | User        | Password         |
+---------------+-------------+------------------+
| localhost     | root        | 2b1d95471a99f22d |
| webserver02   | root        | 2b1d95471a99f22d |
| 127.0.0.1     | root        | 2b1d95471a99f22d |
| ::1           | root        | 2b1d95471a99f22d |
| localhost     |             | 2b1d95471a99f22d |
| webserver02   |             | 2b1d95471a99f22d |
| localhost     | 2n9soft     | 2b1d95471a99f22d |
| localhost     | wisetracker | 2b1d95471a99f22d |
| %             | 2n9soft     | 2b1d95471a99f22d |
| %             | wisetracker | 2b1d95471a99f22d |
| 211.239.123.% | 2n9soft     | 2b1d95471a99f22d |
| 211.239.123.% | wisetracker | 2b1d95471a99f22d |
+---------------+-------------+------------------+
```  
  
  
DB명은 PROFILE 에서 정했던 DB이름으로 CREATE 한다. 테스트 서버에 설치할 때는 INS_로 시작하도록 하면 된다.


```
mysql -u2n9soft -p2n9soft
+-----------------------------------------------------------------+
CREATE DATABASE INS_IDR1 /*!40100 DEFAULT CHARACTER SET utf8 */;
CREATE DATABASE INS_IDA /*!40100 DEFAULT CHARACTER SET utf8 */;
CREATE DATABASE INS_IDU /*!40100 DEFAULT CHARACTER SET utf8 */;
+-----------------------------------------------------------------+
```
  
  
전달 받은 파일 중 sql.zip 을 scp 명령어를 이용해서 로컬 컴퓨터에서 서버로 옮겨서 압축을 푼 후 이하 코드 실행.
ex) scp sql.zip root@175.158.15.226:~wisetracker/work


```
mysql -u2n9soft -p2n9soft INS_IDR1 < /home/wisetracker/work/sql/idr.sql
mysql -u2n9soft -p2n9soft INS_IDA < /home/wisetracker/work/sql/ida.sql
mysql -u2n9soft -p2n9soft INS_IDU < /home/wisetracker/work/sql/idu.sql
 
mysql -u2n9soft -p2n9soft INS_IDR1 < /home/wisetracker/work/sql/function.sql
mysql -u2n9soft -p2n9soft INS_IDA < /home/wisetracker/work/sql/function.sql
mysql -u2n9soft -p2n9soft INS_IDU < /home/wisetracker/work/sql/function.sql
```


밑에 4줄은 테스트 서버에 설치할 때는 실행하지 않아도 된다.



```
mysql -u2n9soft -p2n9soft INS_IDA < /home/wisetracker/work/sql/TA_IPTABLE_LOCAL.sql 
mysql -u2n9soft -p2n9soft INS_IDA < /home/wisetracker/work/sql/TA_IPTABLE_GLOBAL.sql  
mysql -u2n9soft -p2n9soft INS_IDA < /home/wisetracker/work/sql/TA_IPTABLE_GLOBAL_INFO.sql
mysql -u2n9soft -p2n9soft INS_IDA < /home/wisetracker/work/sql/TA_IPTABLE_EXT.sql  
```

  
profile.sql을 DB에 넣는다.(테스트 서버 설치시 profile.sql은 105_WT_SaaS.sql)



```
mysql -u2n9soft -p2n9soft INS_IDA < /home/wisetracker/work/sql/profile/105_WT_SaaS.sql
```

>경고
>PROFILE.sql 넣을때  (ERROR 1054 (42S22) at line 22033: Unknown column 'WEBVIEW_CLASS_NM' in 'field list' )
>--ALTER TABLE INS_IDA.TA_PROFILE ADD WEBVIEW_CLASS_NM varchar(128) DEFAULT '_N_' COMMENT 'WEBVIEW CLASS 이름' AFTER INTERRUPTION_TRK;

#### 추가작업 1) 
```
vi /etc/init.d/mysql

+-----------------------------------------------------------------+
... 생략...
$bindir/mysqld_safe --datadir=$datadir --pid-file=$server_pid_file --language=korean --default-character-set=euckr --skip-character-set-client-handshake  --skip-name-resolv $other_args >/dev/null 2>&1 &
wait_for_pid created $!; return_value=$?

## wisetracker HEAP TABLE LOAD TA_IPTABLE ##
LOGGER_DB=INS_IDA
$bindir/mysql -u2n9soft -p2n9soft $LOGGER_DB -e "INSERT INTO $LOGGER_DB.TA_IPTABLE SELECT * FROM $LOGGER_DB.TA_IPTABLE_LOCAL"
...생략 ...
+-----------------------------------------------------------------+
```

wait_for_pid created $!; return_value=$? 밑에 ##wisetracker HEAP TABLE LOAD TA_IPTABLE## 이하를 추가하면 된다.
wait_for_pid created $!; return_value=$? 이상은 건드리지 않는다.

```
service mysql restart
```


#### 추가작업 2) 글로벌수집시 리포트에서 글로벌 IPTABLE 를 한글로 표현
mysql 접속 후 아래 쿼리문 실행


```
USE INS_IDA
SELECT * FROM TA_REPORT_OPTION WHERE RPT_NO='11050001'; 
+-----+----------+------------+-------------------+---------------+---------+--------------+
| SEQ | RPT_NO   | PROFILE_NO | USE_PAGEDIV       | USE_DIVTEXT   | FLEX_TP | REPORT_WHERE |
+-----+----------+------------+-------------------+---------------+---------+--------------+
|   5 | 11050001 |        105 | PAGEDIV1,PAGEDIV2 | 지역,국가       | 201024  | _N_          |
+-----+----------+------------+-------------------+---------------+---------+--------------+

UPDATE TA_REPORT_OPTION SET FLEX_TP='201026' WHERE RPT_NO='11050001';

SELECT * FROM TA_REPORT_OPTION WHERE RPT_NO='11050001';              
+-----+----------+------------+-------------------+---------------+---------+--------------+
| SEQ | RPT_NO   | PROFILE_NO | USE_PAGEDIV       | USE_DIVTEXT   | FLEX_TP | REPORT_WHERE |
+-----+----------+------------+-------------------+---------------+---------+--------------+
|   5 | 11050001 |        105 | PAGEDIV1,PAGEDIV2 | 지역,국가       | 201026  | _N_          |
+-----+----------+------------+-------------------+---------------+---------+--------------+
```

#### 추가작업 3)
```
SELECT JOIN_OPTION FROM TA_DIMENSION WHERE DIMENSION_NO = 1105;
+-------------+
| JOIN_OPTION |
+-------------+
|             |
+-------------+
UPDATE TA_DIMENSION SET JOIN_OPTION ='LEFT' WHERE DIMENSION_NO = 1105;
SELECT JOIN_OPTION FROM TA_DIMENSION WHERE DIMENSION_NO = 1105;
+-------------+
| JOIN_OPTION |
+-------------+
| LEFT        |
+-------------+
```

#### 추가작업 4) 혹시 메뉴에서 아래 메뉴 글자가 깨질경우
```
UPDATE TA_LANGUAGE_STORAGE  SET DATA_NM='회원 vs 비회원' WHERE DATA_NO = '11010001';
UPDATE TA_LANGUAGE_STORAGE  SET DATA_NM='회원 성별' WHERE DATA_NO = '11020001';
UPDATE TA_LANGUAGE_STORAGE  SET DATA_NM='회원 연령대' WHERE DATA_NO = '11030001';
UPDATE TA_LANGUAGE_STORAGE  SET DATA_NM='방문자 사용언어'  WHERE DATA_NO = '11040001';
UPDATE TA_LANGUAGE_STORAGE  SET DATA_NM='방문 지역' WHERE DATA_NO = '11050001';
```

#### 추가확인 1)
```
SELECT count(*) FROM INS_IDA.TA_IPTABLE;
+----------+
| count(*) |
+----------+
|    73099 |
+----------+
```

테스트 서버에서 105_WT_SaaS.sql로 작업할 때는


```
+----------+
| COUNT(*) |
+----------+
|        0 |
+----------+
```


#### 추가확인 2) 프로파일 확인
```
SELECT PROFILE_NO,PROFILE_TP,PROFILE_NM,SLOT_NO,TRK_SLOT_NO,SUB_SLOT_CNT,ACTIVE_ST,INTERNAL_URL,EXCLUDE_URL
FROM TA_PROFILE WHERE PROFILE_NO < 991001;

+------------+------------+------------+---------+-------------+--------------+-----------+--------------+-------------+
| PROFILE_NO | PROFILE_TP | PROFILE_NM | SLOT_NO | TRK_SLOT_NO | SUB_SLOT_CNT | ACTIVE_ST | INTERNAL_URL | EXCLUDE_URL |
+------------+------------+------------+---------+-------------+--------------+-----------+--------------+-------------+
|        105 | 991003     | WT_SaaS    |     100 |           1 |            4 | ACT001    | .            |             |
+------------+------------+------------+---------+-------------+--------------+-----------+--------------+-------------+
```

INTERNAL_URL 사전추가할시


```
UPDATE TA_PROFILE SET INTERNAL_URL='WT_SaaS.com|logger.co.kr' WHERE PROFILE_NO='105';
```

* INTERNAL_URL 사전 추가는 웹분석을 할 때 필요했던 것이다. 모바일 분석에서는 신경쓰지 않는다.


  




추가로 밑에 쿼리들을 실행시킨다.

```
DELETE FROM TA_CODE_DEFINE_PI_SLOT_100 WHERE PROFILE_NO = 0;
DELETE FROM TA_CODE_DEFINE_PI_SLOT_101 WHERE PROFILE_NO = 0;
DELETE FROM TA_CODE_DEFINE_PI_SLOT_102 WHERE PROFILE_NO = 0;
DELETE FROM TA_CODE_DEFINE_PI_SLOT_103 WHERE PROFILE_NO = 0;
DELETE FROM TA_CODE_DEFINE_PI_SLOT_104 WHERE PROFILE_NO = 0; 

INSERT INTO TA_CODE_DEFINE_PI_SLOT_100 (PROFILE_NO , PI_STEP_CD , PI_NM , PAGE_URL ) VALUES
('0', 'LIF', '로그인양식', '_N_'), 
('0', 'LIR', '로그인', '_N_'), 
('0', 'OCV', '장바구니보기', '_N_'), 
('0', 'ODF', '주문정보입력', '_N_'), 
('0', 'ODR', '주문완료', '_N_'), 
('0', 'PDV', '상품상세보기', '_N_'), 
('0', 'PLV', '상품리스트', '_N_'), 
('0', 'RGF', '회원등록양식', '_N_'), 
('0', 'RGI', '회원등록안내', '_N_'), 
('0', 'RGR', '회원등록', '_N_'), 
('0', 'VST', '전체방문수', '_N_');
INSERT INTO TA_CODE_DEFINE_PI_SLOT_101 (PROFILE_NO , PI_STEP_CD , PI_NM , PAGE_URL ) VALUES
('0', 'LIF', '로그인양식', '_N_'), 
('0', 'LIR', '로그인', '_N_'), 
('0', 'OCV', '장바구니보기', '_N_'), 
('0', 'ODF', '주문정보입력', '_N_'), 
('0', 'ODR', '주문완료', '_N_'), 
('0', 'PDV', '상품상세보기', '_N_'), 
('0', 'PLV', '상품리스트', '_N_'), 
('0', 'RGF', '회원등록양식', '_N_'), 
('0', 'RGI', '회원등록안내', '_N_'), 
('0', 'RGR', '회원등록', '_N_'), 
('0', 'VST', '전체방문수', '_N_');
INSERT INTO TA_CODE_DEFINE_PI_SLOT_102 (PROFILE_NO , PI_STEP_CD , PI_NM , PAGE_URL ) VALUES
('0', 'LIF', '로그인양식', '_N_'), 
('0', 'LIR', '로그인', '_N_'), 
('0', 'OCV', '장바구니보기', '_N_'), 
('0', 'ODF', '주문정보입력', '_N_'), 
('0', 'ODR', '주문완료', '_N_'), 
('0', 'PDV', '상품상세보기', '_N_'), 
('0', 'PLV', '상품리스트', '_N_'), 
('0', 'RGF', '회원등록양식', '_N_'), 
('0', 'RGI', '회원등록안내', '_N_'), 
('0', 'RGR', '회원등록', '_N_'), 
('0', 'VST', '전체방문수', '_N_');
INSERT INTO TA_CODE_DEFINE_PI_SLOT_103 (PROFILE_NO , PI_STEP_CD , PI_NM , PAGE_URL ) VALUES
('0', 'LIF', '로그인양식', '_N_'), 
('0', 'LIR', '로그인', '_N_'), 
('0', 'OCV', '장바구니보기', '_N_'), 
('0', 'ODF', '주문정보입력', '_N_'), 
('0', 'ODR', '주문완료', '_N_'), 
('0', 'PDV', '상품상세보기', '_N_'), 
('0', 'PLV', '상품리스트', '_N_'), 
('0', 'RGF', '회원등록양식', '_N_'), 
('0', 'RGI', '회원등록안내', '_N_'), 
('0', 'RGR', '회원등록', '_N_'), 
('0', 'VST', '전체방문수', '_N_');
INSERT INTO TA_CODE_DEFINE_PI_SLOT_104 (PROFILE_NO , PI_STEP_CD , PI_NM , PAGE_URL ) VALUES
('0', 'LIF', '로그인양식', '_N_'), 
('0', 'LIR', '로그인', '_N_'), 
('0', 'OCV', '장바구니보기', '_N_'), 
('0', 'ODF', '주문정보입력', '_N_'), 
('0', 'ODR', '주문완료', '_N_'), 
('0', 'PDV', '상품상세보기', '_N_'), 
('0', 'PLV', '상품리스트', '_N_'), 
('0', 'RGF', '회원등록양식', '_N_'), 
('0', 'RGI', '회원등록안내', '_N_'), 
('0', 'RGR', '회원등록', '_N_'), 
('0', 'VST', '전체방문수', '_N_');
```





### 2) JDK7

##### 32 Bit OS 일 경우
```
cd /home/wisetracker
cp /home/wisetracker/work/source/jdk-7u45-linux-i586.gz /home/wisetracker/
tar xvfpz jdk-7u45-linux-i586.gz
```

##### 64 Bit OS 일 경우
```
cd /home/wisetracker
cp /home/wisetracker/work/source/jdk-7u45-linux-x64.gz /home/wisetracker/
tar xvfpz jdk-7u45-linux-x64.gz
```

위 실행 후

```
ln -s /home/wisetracker/jdk1.7.0_45 /home/wisetracker/java
```


### 3) Apache-Tomcat 7

```
cp /home/wisetracker/work/source/apache-tomcat-7.0.47.tar.gz /home/wisetracker/
cd /home/wisetracker/
tar xvzf apache-tomcat-7.0.47.tar.gz
chown -R wisetracker:wisetracker ./apache-tomcat-7.0.47
ln -s /home/wisetracker/apache-tomcat-7.0.47  /home/wisetracker/tomcat
```

#### web.xml 갱신

```
mv /home/wisetracker/tomcat/conf/web.xml /home/wisetracker/tomcat/conf/web.xml.OLD
cat /home/wisetracker/work/config/TOMCAT7/web.xml > /home/wisetracker/tomcat/conf/web.xml
```

#### server.xml 갱신

```
mv /home/wisetracker/tomcat/conf/server.xml /home/wisetracker/tomcat/conf/server.xml.OLD
cat /home/wisetracker/work/config/TOMCAT7/server.xml > /home/wisetracker/tomcat/conf/server.xml
```

#### Tomcat/conf/logging.properties 파일 설정

```
mv /home/wisetracker/tomcat/conf/logging.properties /home/wisetracker/tomcat/conf/logging.properties.OLD
cp /home/wisetracker/work/config/logging.properties /home/wisetracker/tomcat/conf/logging.properties
chown wisetracker:wisetracker /home/wisetracker/tomcat/conf/logging.properties
```


#### Tomcat 로그로테이션 시키기
tomcat에서 기록하는 catalina.out 파일 로테이트 처리 (이 파일을 crontab에 등록하여 파일을 기록한다. )


```
cd /home/wisetracker
mkdir -p util

vi /home/work/util/logRotate.sh


+----------------------------------------------------------------+
#!/bin/sh
source /etc/profile

export LANG=en_US
CUR_DATE=`date +%Y-%m-%d-%k`
LOG_PATH=/home/wisetracker/tomcat/logs

cd ${LOG_PATH}
tar cvfz catalina.out.${CUR_DATE}.tar.gz catalina.out
cat /dev/null > catalina.out
chown -R wisetracker:wisetracker /home/wisetracker/tomcat/logs
+----------------------------------------------------------------+

cp -p /home/wisetracker/work/util/logRotate.sh /home/wisetracker/util/


chown -R wisetracker:wisetracker /home/wisetracker/util
chmod u+x /home/wisetracker/util/logRotate.sh

crontab -u wisetracker -e 
+-------------------------------------+
#wisetrackerTomcatlogRotate
59 * * * * /home/wisetracker/util/logRotate.sh
+-------------------------------------+
```

#### TOMCAT service로 구동방법

* CASE 1 ) root계정으로 구동하여 80포트를 기본 포트로 할경우

```
vi /home/wisetracker/work/config/tomcat
+--------------------------------------------------------------------+
[ su - wisetracker -C "$CATALINA_HOME/bin/catalina.sh start" ]과 같은 2곳을
[ $CATALINA_HOME/bin/catalina.sh start                   ]로 변경한다.
+--------------------------------------------------------------------+
```

```
cp /home/wisetracker/work/config/tomcat /etc/init.d/
chmod +x /etc/init.d/tomcat
chkconfig --add tomcat
chkconfig --level 345 tomcat on
```

* CASE 2 ) 고객사에서 보안 정책 상 root 계정을 우리가 사용할 수 없다고 하여 wisetracker 계정으로 구동해야하는 경우

```
cp /home/wisetracker/work/config/tomcat /etc/init.d/
chmod +x /etc/init.d/tomcat
chkconfig --add tomcat
chkconfig --level 345 tomcat on
```


#### 실행시 메모리 설정

```
vi /home/wisetracker/tomcat/bin/catalina.sh (8GB 기준) : 주석끝나고 첫줄.
+--------------------------------------------------------------------------+
export JAVA_OPTS="-server  -XX:+UseConcMarkSweepGC -Xms1024m -Xmx2048m -XX:PermSize=256m -XX:MaxPermSize=256m"
+--------------------------------------------------------------------------+
```
but 트레픽이 작으면
```
vi /home/wisetracker/tomcat/bin/catalina.sh (8GB 기준) : 주석끝나고 첫줄.
+--------------------------------------------------------------------------+
export JAVA_OPTS="-server  -XX:+UseConcMarkSweepGC -Xms512m -Xmx1024m -XX:PermSize=128m -XX:MaxPermSize=256m"
+--------------------------------------------------------------------------+
```

* JAVA_OPTS 변수가 실제로 사용이 되는 곳 위쪽의 어느 곳이든지 이 설정을 추가해주면 된다. 
* 하지만 굳이 주석 끝나고 첫줄이라고 명시하는 것은 팀내 개발자들 사이에서 해당 설정 수정 시 쉽게 찾을 수 있도록 하기 위함이다.



#### 톰켓 MAX 스레드풀 확인 Msanager 페이지설정(보안상 패스할수도)
```
vi /home/wisetracker/tomcat/conf/tomcat-users.xml
+----------------------------------------------------------+
<?xml version='1.0' encoding='utf-8'?>
<tomcat-users>
<role rolename="manager-gui"/>m
<user username="bizspring" password="tmakxm00" roles="manager-gui" />
</tomcat-users>
+----------------------------------------------------------+
```

* (*접속URL http://고객사 솔루션 DOMAIN 주소/manager/html -> Id,Pass )


>여기까지 기본 APACHE-TOMCAT 설치 완료가 되었다.  
>설치가 잘 되었는지 확인하기 위해 tomcat을 실행시키고 기본페이지에 접속해보자.
>고양이페이지 잘뜨면 tomcat/webapps 아래 불필요한 디렉토리는 압축하여 보관하자.

```
cd /home/wisetracker/tomcat/webapps
tar cvfpz docs.tar.gz docs
tar cvfpz examples.tar.gz examples
tar cvfpz host-manager.tar.gz host-manager
chown wisetracker:wisetracker *.tar.gz
rm -rf docs examples host-manager
ls -al
```


### 4) WAR 배포

```
mkdir -p ~wisetracker/work/wisetracker 
chown -R wisetracker:wisetracker /home/wisetracker/work/wisetracker 
```

이하 6줄의 코드와 같이 전달 받은 SolutionSource.zip을 scp를 이용해서 /home/wisetracker/work/wisetracker 밑으로 옮기고 압축을 풀고 Tomcat을 중지시킨다.


```
scp SolutionSource.zip root@175.158.15.226:/home/wisetracker/work/wisetracker/
ssh root@175.158.15.226
cd /home/wisetracker/work/wisetracker
unzip SolutionSource.zip

service tomcat stop
ps -ef | grep java
```


#### - TRACKER War 배포

* war 파일을 tomcat/webapps 밑에 넣어 놓으면 tomcat이 구동될 때 자동으로 압축을 풀고 배포한다. 

```
cp -pR /home/wisetracker/work/wisetracker/InsightTrk.war /home/wisetracker/tomcat/webapps/

service tomcat start
ps -ef | grep java

vi /home/wisetracker/tomcat/webapps/InsightTrk/WEB-INF/classes/resources/props/jdbc.properties
+----------------------------------------------------------------------------------------------+
############################################################################# 
# Database Local
#############################################################################
db.local.jdbcUrl=jdbc:mysql://localhost:3306/INS_IDA?useUnicode=true&amp;characterEncoding=utf8&amp;autoReconnect=true
db.local.minPoolSize=10
db.local.maxPoolSize=20
#############################################################################  
# Database Admin 
#############################################################################
db.admin.jdbcUrl=jdbc:mysql://localhost:3306/INS_IDA?useUnicode=true&amp;characterEncoding=utf8&amp;autoReconnect=true
db.admin.minPoolSize=1
db.admin.maxPoolSize=1
#############################################################################
# Profile Raw Db Connection Set 1 with 2 Item 
#############################################################################
## IDR  
db.server1.RDB.jdbcUrl=jdbc:mysql://localhost:3306/INS_IDR1?useUnicode=true&amp;characterEncoding=utf8&amp;autoReconnect=true
db.server1.RDB.minPoolSize=10
db.server1.RDB.maxPoolSize=20
----------------------------------------------------------------------------------------------+
```

* 위 수정사항 이 외에도 파일 내 서버 관련 url을 전부 적절한 url(테스트 서버에서는 localhost)로 바꾸어주어야 한다. 
* 그래야 이 후 구동 시 문제 없이 tomcat과 mysql 정상적으로 연동이 된다.




##### 글로벌용 IPTABLE를 사용시(아래 tracker소스 배포할때 수정)

```
vi +/tracker.iptable.name /home/wisetracker/tomcat/webapps/InsightTrk/WEB-INF/classes/resources/props/initParameter.properties
+-----------------------------------------------------------------+
tracker.iptable.name=TA_IPTABLE
tracker.iptable.name=TA_IPTABLE_GLOBAL
+-----------------------------------------------------------------+

service tomcat stop
ps -ef | grep java
cd
```

#### - infographic War 배포

```
cp -pR /home/wisetracker/work/wisetracker/infographic.war /home/wisetracker/tomcat/webapps/


service tomcat start
ps -ef | grep java


vi /home/wisetracker/tomcat/webapps/infographic/WEB-INF/classes/resources/props/jdbc.properties

+----------------------------------------------------------------------------------------------+
#############################################################################
# Database Distribution ( Main DB ) 
#############################################################################
infographic.url=jdbc:mysql://localhost:3306/INS_IDA?useUnicode=true&amp;characterEncoding=utf8&amp;autoReconnect=true
infographic.minIdle=5
infographic.maxIdle=10
#############################################################################
# Database Distribution ( Raw DB 1 )
#############################################################################
infographic.raw.server1.url=jdbc:mysql://localhost:3306/INS_IDR1?useUnicode=true&amp;characterEncoding=utf8&amp;autoReconnect=true
infographic.raw.server1.minIdle=5
infographic.raw.server1.maxIdle=10                 
+----------------------------------------------------------------------------------------------+
```

* 위 수정사항 이 외에도 파일 내 서버 관련 url을 전부 적절한 url(테스트 서버에서는 localhost)로 바꾸어주어야 한다.  
* 그래야 이 후 구동 시 문제 없이 tomcat과 mysql 연동이 된다.


```
vi /home/wisetracker/tomcat/webapps/infographic/WEB-INF/classes/resources/wafi/wafi.jdbc.properties

+----------------------------------------------------------------------------------------------+
#############################################################################
# Database Distribution ( PushMessage DB )
#############################################################################

wafi.driverClassName=com.mysql.jdbc.Driver
wafi.url=jdbc:mysql://localhost:3306/INS_IDA?useUnicode=true&amp;characterEncoding=utf8&amp;autoReconnect=true
+----------------------------------------------------------------------------------------------+



vi /home/wisetracker/tomcat/webapps/infographic/WEB-INF/classes/resources/props/config.properties
+----------------------------------------------------------------------------------------------+
context.root.dir=/home/wisetracker/tomcat/webapps/infographic
infograhpic.realUploadPath=/home/wisetracker/tomcat/webapps/infographic/upload
infograhpic.bulkFileDir=/home/wisetracker/tomcat/webapps/infographic/bulkFile
+----------------------------------------------------------------------------------------------+
```


#### - 리포트 연결 리다이렉션
```
vi /home/wisetracker/tomcat/webapps/ROOT/index.jsp
+----------------------------------------------------------------------------------------------+
<meta http-equiv="REFRESH" content="0;url=/infographic/login.do?locale=ko_KR">
+----------------------------------------------------------------------------------------------+

cd /home/wisetracker
chown -R wisetracker:wisetracker apache-tomcat-7.0.47
```

기본 설치가 끝났으니 리포트 로그인이 되는지 확인하자.

```
mysql -u2n9soft -p2n9soft INS_IDA -e "SELECT ACCOUNT_ID,ACCOUNT_PWD FROM TA_ACCOUNT"      
+------------+-------------------------------------------+
| ACCOUNT_ID | ACCOUNT_PWD                               |
+------------+-------------------------------------------+
| bizspring  | *066E53AA9DE901C311B5905210ED2C24BAE0A8FD | -> tmakxm00
| admin      | *C20A1B0C3EBD8A015B71B44FC5F7B0842B4A542B | -> adm@PWD
| reporter   | *5EF65FCB73FA51A8A47544ACF6A34FF363F13CCD | -> rpt@PWD
+------------+-------------------------------------------+
```

#### - 고객이 DB접속정보를 암호화 해달라고 할때,<예외 상황>

* 단, [ENC] 가 들어간 뒷문장부터 그 라인만 대상이 된다.

```
java BizspringPropertyPlaceholderConfigurer 암호화대상파일 encode
java BizspringPropertyPlaceholderConfigurer 암호화대상파일 decode
```

### 5) phantomjs 

-- 64bit
```
cd /home/wisetracker
cp -pR /home/wisetracker/work/source/phantomjs.1.9.7_64bit.tar.gz /home/wisetracker/
tar xvfpz phantomjs.1.9.7_64bit.tar.gz
ln -s /home/wisetracker/phantomjs-1.9.7-linux-x86_64 /home/wisetracker/phantomjs
```
--32bit
```
cd /home/wisetracker
cp -pR /home/wisetracker/work/source/phantomjs.1.9.7_32bit.tar.gz /home/wisetracker/
tar xvfpz phantomjs.1.9.7_32bit.tar.gz
ln -s /home/wisetracker/phantomjs-1.9.7-linux-i686 /home/wisetracker/phantomjs
```
phantomjs를 실행하려면 /home/wisetracker/phantomjs/bin/phantomjs를 실행시키면 된다.


scp 명령어를 통해 전달받은 파일 중 highchart-export-web.war를 서버의 /home/wisetracker/work/wisetracker/ 폴더에 옮겨 놓은 후에 밑에 코드 실행
ex) scp highchart-export-web.war root@175.158.15.226:/home/wisetracker/work/wisetracker/


```
service tomcat stop
cp /home/wisetracker/work/wisetracker/highcharts-export-web.war /home/wisetracker/tomcat/webapps
service tomcat start
chown -R wisetracker:wisetracker highcharts-export-web
chown -R wisetracker:wisetracker highcharts-export-web.war
service tomcat stop
service tomcat start
```

TEST : http://211.239.123.87/highcharts-export-web/ ( 이미지 다운로드시 "한글"이 안깨지는 확인)  
- 테스트서버인 경우 : http://175.158.15.226/highcharts-export-web/




### 6)Cron 배포
```
cp -pR /home/wisetracker/work/wisetracker/InsightSch.tar.gz /home/wisetracker/
cd /home/wisetracker/
tar xvfzp InsightSch.tar.gz
cd InsightSch
chmod 755 logs

yum -y install dos2unix*
dos2unix /home/wisetracker/InsightSch/bin/*
chmod u+x /home/wisetracker/InsightSch/bin/*
chown -R wisetracker:wisetracker /home/wisetracker/InsightSch   
```

**체크)** 정제로직이 변경되어 raw database 는 IDR 서버의 IDA DB를 main database 는 IDA서버의 IDA DB를 바라봐야 한다.  
한 대일 때는 다 localhost로 하면 된다. ex) 테스트 서버에 설치할 때

```
vi /home/wisetracker/InsightSch/classes/resources/props/jdbc.properties
+----------------------------------------------------------+
#############################################################################
# Local Database IDA - raw database 
#############################################################################
db.server1.IDR1.driverClassName=com.mysql.jdbc.Driver
db.server1.IDR1.name=MYSQL
db.server1.IDR1.jdbcUrl=jdbc:mysql://localhost:3306/INS_IDA?useUnicode=true&amp;characterEncoding=utf8&amp;autoReconnect=true
db.server1.IDR1.username=2n9soft
db.server1.IDR1.password=2n9soft
db.server1.IDR1.acquireRetryAttempts=1
db.server1.IDR1.preferredTestQuery=SELECT 1
db.server1.IDR1.testConnectionOnCheckin=true
db.server1.IDR1.testConnectionOnCheckout=false
db.server1.IDR1.maxIdleTime=300
db.server1.IDR1.minPoolSize=1
db.server1.IDR1.maxPoolSize=5
db.server1.IDR1.maxStatementsPerConnection=5
db.server1.IDR1.idleConnectionTestPeriod=25

#############################################################################
# Database SDB IDA - main database 
#############################################################################
db.sdb.driverClassName=com.mysql.jdbc.Driver
db.sdb.name=MYSQL
db.sdb.jdbcUrl=jdbc:mysql://localhost:3306/INS_IDA?useUnicode=true&amp;characterEncoding=utf8&amp;autoReconnect=true
db.sdb.username=2n9soft
db.sdb.password=2n9soft
db.sdb.acquireRetryAttempts=1
db.sdb.preferredTestQuery=SELECT 1
db.sdb.testConnectionOnCheckin=true
db.sdb.testConnectionOnCheckout=false
db.sdb.maxIdleTime=300
db.sdb.minPoolSize=1
db.sdb.maxPoolSize=5
db.sdb.maxStatementsPerConnection=5
db.sdb.idleConnectionTestPeriod=25
+----------------------------------------------------------+
```

설정확인
```
vi +/'scheduler.home' /home/wisetracker/InsightSch/classes/resources/props/initParameter.properties
+----------------------------------------------------------+
scheduler.home=/home/wisetracker/InsightSch
+----------------------------------------------------------+

crontab -u wisetracker -e
+------------------------------------------------------------------------------------+
###############################################################################
# wisetracker Scheduler
###############################################################################
# basic Scheduler
#!!#*/5 * * * *     /bin/bash /home/wisetracker/InsightSch/bin/basicSchedulerPart1.sh
#!!#*/6 * * * *     /bin/bash /home/wisetracker/InsightSchSch/bin/basicSchedulerPart2.sh
#!!#*/7 * * * *     /bin/bash /home/wisetracker/InsightSch/bin/basicSchedulerPart3.sh

#!!#*/10              * * * * /bin/bash /home/wisetracker/InsightSch/bin/ckZoneScheduler.sh
####6,36              * * * * /bin/bash /home/wisetracker/InsightSch/bin/czImageScheduler.sh
#!!#25,55             * * * * /bin/bash /home/wisetracker/InsightSch/bin/pathFinderStep1Scheduler.sh
#!!#35,45,55          * * * * /bin/bash /home/wisetracker/InsightSch/bin/deleteScheduler.sh
#!!#38,43,48,53       4 * * * /bin/bash /home/wisetracker/InsightSch/bin/pathFinderStep2Scheduler.sh <-- (LCD 전날)

### Profile Group Scheduler 통합분석있을시
####38,43,48,53       6 * * * /bin/bash /home/wisetracker/InsightSch/bin/groupScheduler.sh <-- (LCD 전날)

# Kpi Scheduler
#!!#4                 4 * * * /bin/bash /home/wisetracker/InsightSch/bin/kpiDataScheduler.sh <-- (LCD 전날)
#!!#10                7 * * * /bin/bash /home/wisetracker/InsightSch/bin/kpiMailScheduler.sh  <---- no state(102) 일단 묵시한뒤 요청이 있으면 활성화
#!!#2                 8 * * * /bin/bash /home/wisetracker/InsightSch/bin/mailSendScheduler.sh <---- no state(102) 일단 묵시한뒤 요청이 있으면 활성화

# Dw Scheduler
####15,30,35,40       * * * * /bin/bash /home/wisetracker/InsightSch/bin/userIdDataScheduler.sh
####45                * * * * /bin/bash /home/wisetracker/InsightSch/bin/userIdFactDataScheduler.sh
####20               16 * * * /bin/bash /home/wisetracker/InsightSch/bin/userIdDataDeleteScheduler.sh

# STPS Scheduler
####55                2 * * * /bin/bash /home/wisetracker/InsightSch/bin/queryGenScheduler.sh
####4,12,22,32,42,52  * * * * /bin/bash /home/wisetracker/InsightSch/bin/deleteFile.sh
###############################################################################
+------------------------------------------------------------------------------------+

chmod 755 /proc
```
chmod 755 /proc을 rc.local에 넣어 부팅될때마다 실행되게 해 둔다.

```
vi /etc/rc.local
+-----------------+
chmod 755 /proc
+-----------------+
```




# 3. 테스트(수집, 정제)


### TRACKER TEST 

##### STEP 1)  
- 테스트할 프로파일을 스크립트를 받아 적당한 더미사이트에 삽입한다.  
- 공용도메인이 추가되어있어야 하며 없다면 테스트 PC에서 hosts을 수정하고 스크립트 적용한 더미사이트의 httpd.conf를 수정하여 분석도메인과 똑같은 가상환경에서 테스트한다.

##### STEP 2) 등록된 프로파일의 기본 정보를 참고하여 실제 DB에 쌓이는지 확인(TRK_SLOT_NO를 확인)

```
USE INS_IDA
SELECT PROFILE_NO,PROFILE_NM,BASE_URl,ACTIVE_ST,SLOT_NO,TRK_SLOT_NO,INTERNAL_URL,EXCLUDE_URL 
FROM TA_PROFILE 
WHERE ACTIVE_ST='ACT001' AND PROFILE_NO < 991001;
+------------+------------+-------------+-----------+---------+-------------+--------------------------+-------------+
| PROFILE_NO | PROFILE_NM | BASE_URl    | ACTIVE_ST | SLOT_NO | TRK_SLOT_NO | INTERNAL_URL             | EXCLUDE_URL |
+------------+------------+-------------+-----------+---------+-------------+--------------------------+-------------+
|        105 | WT_SaaS    | .           | ACT001    |     100 |           1 | WT_SaaS.com|logger.co.kr |             |
+------------+------------+-------------+-----------+---------+-------------+--------------------------+-------------+
```

105 프로파일은 1번슬롯에 저장하며 정제시 100번슬롯에 저장한다. 이는 TRK_SLOT_NO와 SLOT_NO를 보고 알 수 있다.  
공용도메인은 정규표현식으로 WT_SaaS.com|logger.co.kr 일때 만족한다.


수집데이터 확인 쿼리 (위 예시에서 수집데이터(Raws)는 슬롯 1번에 쌓이므로)

```
USE INS_IDR1;
SELECT TM,MIN(VT),MAX(VT),MIN(VT_TZ),MAX(VT_TZ),COUNT(*) FROM TR_PAGES_SLOT_1_0 GROUP BY TM;
SELECT TM,MIN(VT),MAX(VT),MIN(VT_TZ),MAX(VT_TZ),COUNT(*) FROM TR_PAGES_SLOT_1_1 GROUP BY TM;
SELECT TM,MIN(VT),MAX(VT),MIN(VT_TZ),MAX(VT_TZ),COUNT(*) FROM TR_PAGES_SLOT_1_2 GROUP BY TM;
SELECT TM,MIN(VT),MAX(VT),MIN(VT_TZ),MAX(VT_TZ),COUNT(*) FROM TR_PAGES_SLOT_1_3 GROUP BY TM;
```




### CRON TEST


##### STEP 1) 쿼리젠 

IDR서버의 IDA DB에서(테스트서버는 그냥 IDA DB에서) 이하 테스트 실행.  
평소 쿼리젠은 활성화 되어 있지 않다. 동작을 위해 활성화 하고 동작 후에는 다시 비활성화 조치.  
ACTIVE_ST를 수정해줌으로서 활성화/비활성화 전환. 'ACT001'은 활성화 상태

```
USE INS_IDA
UPDATE TA_SCHEDULE SET ACTIVE_ST='ACT001' WHERE SCHE_NO IN ('92');
```

mysql을 빠져 나와서 밑에 코드 실행

```
su - wisetracker
cd /home/wisetracker/InsightSch/logs/
/home/wisetracker/InsightSch/bin/queryGenScheduler.sh
```




(1개의 프로파일일때 아래 결과)
```
USE INS_IDA
SELECT SCHE_NO, LCD, COUNT(*) FROM TA_SCHEDULE_DETAIL GROUP BY SCHE_NO, LCD;
+---------+---------------------+----------+
| SCHE_NO | LCD                 | COUNT(*) |
+----m-----+---------------------+----------+
|       1 | 2014-08-29 22:59:59 |      100 |
|       2 | 2014-08-29 22:59:59 |       92 |
|       3 | 2014-08-29 22:59:59 |       72 |
|      91 | 2014-08-29 22:59:59 |        4 |
|     104 | 2014-08-29 22:59:59 |        4 |
+---------+---------------------+----------+

SELECT COUNT(*) FROM TA_SCHEDULE_QUERY_DML ;
+----------+
| COUNT(*) |
+----------+
|      236 |
+----------+

UPDATE TA_SCHEDULE SET ACTIVE_ST='ACT002' WHERE SCHE_NO IN ('92');
```




#### STEP 2) 기본 2년치 파티션닝 추가

>-- 파티션닝 상태 확인  
>`SHOW CREATE TABLE TS_STAT_CONTS_SLOT_100;`  
>맨 밑에 (PARTITION p201301 VALUES IN (201301) ENGINE = MyISAM) 이렇게 기본 한달만 있다.

* 기본 한달을 넣어주는 것은 처음 테이블을 생성할 때 파티션에 대한 설정이 있어야만 나중에 파티션 관련 설정을 추가할 수 있기 때문이다.

```
su - wisetracker
vi /home/wisetracker/InsightSch/bin/addPartitionQueryGen.sh
+--------------------------------------------------------------------------------+
java $ARG_PROPS -classpath $CLASSPATH AddPartitionQueryGen INS_IDA 2018 2020
+--------------------------------------------------------------------------------+
```


`sh /home/wisetracker/InsightSch/bin/addPartitionQueryGen.sh`


파일 생성 확인 후 디비에 넣는다.

```
cd /home/wisetracker/InsightSch
ls -al | grep createPartition
mysql -u2n9soft -p2n9soft INS_IDA < 2018_2020_INS_IDA_createPartition.sql

``` 


#### STEP 3) LCD시간과 활성화 할 SCHE_NO 설정

테스트시 LCD 부분을 미리 적당히 맞추어 놓고 시작( 실행하는 시간 4~5시간 전시간으로)

```
USE INS_IDA

-- LCD
UPDATE TA_SCHEDULE SET LCD ='2018-04-17 10:59:59';
UPDATE TA_SCHEDULE_DETAIL SET LCD ='2018-04-17 10:59:59';

-- ACTIVE_ST
UPDATE TA_SCHEDULE SET ACTIVE_ST='ACT002';
UPDATE TA_SCHEDULE SET ACTIVE_ST='ACT001' 
WHERE SCHE_NO IN ('1','2','3','4','5','6','7','8','9','10','90','91','98','104','105');
```

```
UPDATE TA_SCHEDULE set LCD='2018-04-15 23:59:59' WHERE SCHE_NO IN ('98','105','101');
(이것은 전날 데이터를 당일 한번 수행하므로 테스트시에는 이틀전으로 돌려야 테스트 가능, 101은 통합프로파일 존재할때)
```

* 이제 정제 테스트 하기전에 LCD 확인

#### - 모든정제조회 (서버나 나누어질경우 BASE는 IDR서버의 IDA DB에 LCD, ING_YN등을 갱신한다.)
```
SELECT 
A.SCHE_TP,A.SCHE_NO,A.SCHE_NM, MIN( IFNULL( B.LCD, A.LCD ) )  AS LCD ,A.ING_YN,A.ACTIVE_ST 
FROM TA_SCHEDULE A   JOIN  TA_SCHEDULE_DETAIL B
ON A.SCHE_NO = B.SCHE_NO 
WHERE A.ACTIVE_ST = 'ACT001' 
GROUP BY A.SCHE_TP , A.SCHE_NM,A.ACTIVE_ST 
UNION ALL
SELECT 
A.SCHE_TP,A.SCHE_NO,A.SCHE_NM, MIN( IFNULL( B.LCD, A.LCD ) )  AS LCD,A.ING_YN, A.ACTIVE_ST
FROM TA_SCHEDULE A   LEFT JOIN  TA_SCHEDULE_DETAIL B
ON A.SCHE_NO = B.SCHE_NO 
WHERE A.ACTIVE_ST = 'ACT001'  
AND     A.SCHE_TP != 'STPR01' 
GROUP BY A.SCHE_TP , A.SCHE_NM,A.ACTIVE_ST ;
+---------+---------+-------------------+---------------------+--------+-----------+
| SCHE_TP | SCHE_NO | SCHE_NM           | LCD                 | ING_YN | ACTIVE_ST |
+---------+---------+-------------------+---------------------+--------+-----------+
| STPR01  |       1 | BASE_CRON1        | 2018-04-17 10:59:59 | N      | ACT001    |
| STPR01  |       2 | BASE_CRON2        | 2018-04-17 10:59:59 | N      | ACT001    |
| STPR01  |       3 | BASE_CRON3        | 2018-04-17 10:59:59 | N      | ACT001    |
| STPR02  |      91 | CLICKZONE         | 2018-04-17 10:59:59 | N      | ACT001    |
| STPR07  |     104 | PATH_FINDER       | 2018-04-17 10:59:59 | N      | ACT001    |
| STPR02  |      91 | CLICKZONE         | 2018-04-17 10:59:59 | N      | ACT001    |
| STPR04  |      90 | BASE_DELETE       | 2018-04-17 10:59:59 | N      | ACT001    |
| STPR07  |     104 | PATH_FINDER       | 2018-04-17 10:59:59 | N      | ACT001    |
| STPS05  |      98 | KPI_DATA          | 2018-04-15 23:59:59 | N      | ACT001    |<-- 테스트를 위해 이틀전 시간
| STPS10  |     105 | PATH_FINDER_STEP2 | 2018-04-15 23:59:59 | N      | ACT001    |<-- 테스트를 위해 이틀전 시간
+---------+---------+-------------------+---------------------+--------+-----------+
```

##### - BASE_CRON만 조회
```
SELECT SCHE_NO,LCD,ING_YN,COUNT(*) FROM TA_SCHEDULE_DETAIL GROUP BY SCHE_NO,LCD,ING_YN;
+---------+---------------------+--------+----------+
| SCHE_NO | LCD                 | ING_YN | COUNT(*) |
+---------+---------------------+--------+----------+
|       1 | 2018-04-17 10:59:59 | N      |      100 |
|       2 | 2018-04-17 10:59:59 | N      |       92 |
|       3 | 2018-04-17 10:59:59 | N      |       72 |
|      91 | 2018-04-17 10:59:59 | N      |        4 |
|     104 | 2018-04-17 10:59:59 | N      |        4 |
+---------+---------------------+--------+----------+
```







#### STEP 4) #!!# 로 막아둔 크론들을 하나씩 실행/로그파일 확인 

크론 로그중 [ no states ] 메세지가 있을경우 스케줄러에 비활성화 되어있는것

예) /home/wisetracker/InsightSch/bin/ckZoneScheduler.sh 파일에 " STPS06 99 "
이런 부분이 있다. 여기서 91번을 참고하여 아래와 같이 활성화 UPDATE

EX. UPDATE TA_SCHEDULE SET ACTIVE_ST='ACT001' WHERE SCHE_NO IN ('99','102','103');

크론 로그중 [ scheduleDetailBeans or no jobs yet ] 메세지가 있을경우 report 메뉴에서 빠져 크론이 돌때 Detail에서 빠지기때문이다. 
이때는 빠진 리포트를 추가하거나, 빠진 리포트를 처음부터 추가해서 프로파일을 생성하여야 다시 적용







# 4. 별도 적용 쿼리 및 기타 서버 설정

### 별도 적용 쿼리

#### 리플리케이션 및 처리데이터 삭제결정 유무 옵션

IDR서버에서
```
-- 우선확인
USE INS_IDR1
SELECT * FROM TA_CODE;
SELECT * FROM TA_CODE_DTL;

-- MASTER (테스트 서버에서는 이것만 하면 된다. SLAVE 확인 못함.)
insert into TA_CODE(CD_NO,CD_NM,CD_DESC,CD_ALIAS,REG_DT,UPD_DT) values ('RPN','Replication Server Name','Replication Server Name','RPN','2012-01-04 05:02:54','2012-01-04 05:02:54'); 
insert into TA_CODE_DTL(CD_DTL_NO,CD_NO,CD_DTL_NM,CD_DTL_DESC,CD_DTL_ALIAS,REG_DT,UPD_DT) values ('RPN001','RPN','MASTER','MASTER','RPN_NAME','2012-10-12 11:54:41','2012-10-12 11:54:41');

-- SLAVE
insert into TA_CODE(CD_NO,CD_NM,CD_DESC,CD_ALIAS,REG_DT,UPD_DT) values ('RPN','Replication Server Name','Replication Server Name','RPN','2012-01-04 05:02:54','2012-01-04 05:02:54'); 
insert into TA_CODE_DTL(CD_DTL_NO,CD_NO,CD_DTL_NM,CD_DTL_DESC,CD_DTL_ALIAS,REG_DT,UPD_DT) values ('RPN001','RPN','SLAVE','SLAVE','RPN_NAME','2012-10-12 11:54:41','2012-10-12 11:54:41');
```


```
-- 유입검색엔진, cpc매체별키워드,국가/지역에 대한 리포트 항목 순서를 업데이트하는 쿼리
USE INS_IDA
UPDATE TA_REPORT_OPTION SET USE_PAGEDIV = 'PAGEDIV2,PAGEDIV1', USE_DIVTEXT = '검색엔진,검색어' WHERE RPT_NO = 21020001;
UPDATE TA_REPORT_OPTION SET USE_PAGEDIV = 'PAGEDIV2,PAGEDIV1', USE_DIVTEXT = 'CPC매체,CPC키워드' WHERE RPT_NO = 23020001;
UPDATE TA_REPORT_OPTION SET USE_PAGEDIV = 'PAGEDIV2,PAGEDIV1', USE_DIVTEXT = '국가,지역' WHERE RPT_NO = 11050001;

-- kpi 메일 발송 기능을 위해서, 참조하는 템플릿 파일 경로 설정하는 부분 추가. 
UPDATE TA_CODE_DTL SET CD_DTL_DESC ='/home/wisetracker/InsightSch/classes/resources/kpiStyle' WHERE CD_DTL_ALIAS ='MESSAGE_VAR_DIR_KPI_TEMPLATE';
UPDATE TA_CODE_DTL SET CD_DTL_DESC ='/home/wisetracker/tomcat/webapps/wisetrackerRpt/dataDownload' WHERE CD_DTL_ALIAS ='MESSAGE_VAR_DIR_ATTACH';
UPDATE TA_CODE_DTL SET CD_DTL_DESC ='solution@bizspring.co.kr' WHERE CD_DTL_ALIAS ='MESSAGE_VAR_SENDER_EMAIL';

-- 회원 ID 데이터뷰어 비활성화 
DELETE FROM TA_DESKTOP_AUTHORITY_ACCOUNT WHERE MENU_SEQ = 9 AND ACCOUNT_NO != 1;

-- 메뉴 자르기
UPDATE TA_LANGUAGE_STORAGE SET DATA_NM=replace(DATA_NM,' / Traffic','') WHERE DATA_NO=1001 AND DATA_TP=151003;
UPDATE TA_LANGUAGE_STORAGE SET DATA_NM=replace(DATA_NM,' / Entries','') WHERE DATA_NO=1002 AND DATA_TP=151003;
UPDATE TA_LANGUAGE_STORAGE SET DATA_NM=replace(DATA_NM,' / Contents','') WHERE DATA_NO=1003 AND DATA_TP=151003;
UPDATE TA_LANGUAGE_STORAGE SET DATA_NM=replace(DATA_NM,' / Marketing','') WHERE DATA_NO=1004 AND DATA_TP=151003;
UPDATE TA_LANGUAGE_STORAGE SET DATA_NM=replace(DATA_NM,' / Visitor','') WHERE DATA_NO=1005 AND DATA_TP=151003;
UPDATE TA_LANGUAGE_STORAGE SET DATA_NM=replace(DATA_NM,' / Navigation','') WHERE DATA_NO=1006 AND DATA_TP=151003;
UPDATE TA_LANGUAGE_STORAGE SET DATA_NM=replace(DATA_NM,' / System','') WHERE DATA_NO=1007 AND DATA_TP=151003;
UPDATE TA_LANGUAGE_STORAGE SET DATA_NM=replace(DATA_NM,' / Commerce','') WHERE DATA_NO=1008 AND DATA_TP=151003;
```


-- REPORT SET 정렬

```
SELECT PROFILE_NO,PROFILE_TP, PROFILE_NM FROM TA_PROFILE WHERE PROFILE_NO < 991001;
+------------+------------+-------------+
| PROFILE_NO | PROFILE_TP | PROFILE_NM  |
+------------+------------+-------------+
|       5901 | 991001     | bizsmart.jp |
+------------+------------+-------------+
SELECT 
CONCAT('UPDATE TA_REPORT_CAT_ATTR SET LIST_ORD =  ',A.LIST_ORD ,' WHERE  RPT_MENU_NO = ',  P.RPT_MENU_NO, ' AND RPT_NO = ', A.RPT_NO,';' ) AS UPQ
FROM TA_REPORT A JOIN TA_PROFILE P
ON A.PROFILE_NO = P.PROFILE_NO
WHERE A.PROFILE_NO = '5901';

+----------------------------------------------------------------------------------------------+
| UPQ                                                                                          |
+----------------------------------------------------------------------------------------------+
  UPDATE TA_REPORT_CAT_ATTR SET LIST_ORD =  14 WHERE  RPT_MENU_NO = 105 AND RPT_NO = 11010001;
  UPDATE TA_REPORT_CAT_ATTR SET LIST_ORD =  15 WHERE  RPT_MENU_NO = 105 AND RPT_NO = 11020001; 
  UPDATE TA_REPORT_CAT_ATTR SET LIST_ORD =  16 WHERE  RPT_MENU_NO = 105 AND RPT_NO = 11030001; 
  UPDATE TA_REPORT_CAT_ATTR SET LIST_ORD =  13 WHERE  RPT_MENU_NO = 105 AND RPT_NO = 11040001; 
  UPDATE TA_REPORT_CAT_ATTR SET LIST_ORD =  12 WHERE  RPT_MENU_NO = 105 AND RPT_NO = 11050001; 
  UPDATE TA_REPORT_CAT_ATTR SET LIST_ORD =  25 WHERE  RPT_MENU_NO = 105 AND RPT_NO = 12010001; 
  UPDATE TA_REPORT_CAT_ATTR SET LIST_ORD =  27 WHERE  RPT_MENU_NO = 105 AND RPT_NO = 12030001; 
  UPDATE TA_REPORT_CAT_ATTR SET LIST_ORD =  23 WHERE  RPT_MENU_NO = 105 AND RPT_NO = 12110001; 
  UPDATE TA_REPORT_CAT_ATTR SET LIST_ORD =  26 WHERE  RPT_MENU_NO = 105 AND RPT_NO = 12120001; 
  UPDATE TA_REPORT_CAT_ATTR SET LIST_ORD =  28 WHERE  RPT_MENU_NO = 105 AND RPT_NO = 12200001; 
  UPDATE TA_REPORT_CAT_ATTR SET LIST_ORD =  22 WHERE  RPT_MENU_NO = 105 AND RPT_NO = 12210001; 
  UPDATE TA_REPORT_CAT_ATTR SET LIST_ORD =  24 WHERE  RPT_MENU_NO = 105 AND RPT_NO = 12220001; 
  UPDATE TA_REPORT_CAT_ATTR SET LIST_ORD =  11 WHERE  RPT_MENU_NO = 105 AND RPT_NO = 13010001; 
  UPDATE TA_REPORT_CAT_ATTR SET LIST_ORD =  43 WHERE  RPT_MENU_NO = 105 AND RPT_NO = 13030001; 
  UPDATE TA_REPORT_CAT_ATTR SET LIST_ORD =  42 WHERE  RPT_MENU_NO = 105 AND RPT_NO = 13040001; 
  UPDATE TA_REPORT_CAT_ATTR SET LIST_ORD =  46 WHERE  RPT_MENU_NO = 105 AND RPT_NO = 14050001; 
  UPDATE TA_REPORT_CAT_ATTR SET LIST_ORD =  5 WHERE  RPT_MENU_NO = 105 AND RPT_NO = 14510001;  
  UPDATE TA_REPORT_CAT_ATTR SET LIST_ORD =  44 WHERE  RPT_MENU_NO = 105 AND RPT_NO = 14520001; 
  UPDATE TA_REPORT_CAT_ATTR SET LIST_ORD =  6 WHERE  RPT_MENU_NO = 105 AND RPT_NO = 14530001;  
  UPDATE TA_REPORT_CAT_ATTR SET LIST_ORD =  45 WHERE  RPT_MENU_NO = 105 AND RPT_NO = 14540001; 
  UPDATE TA_REPORT_CAT_ATTR SET LIST_ORD =  31 WHERE  RPT_MENU_NO = 105 AND RPT_NO = 23070001; 
  UPDATE TA_REPORT_CAT_ATTR SET LIST_ORD =  33 WHERE  RPT_MENU_NO = 105 AND RPT_NO = 23080001; 
  UPDATE TA_REPORT_CAT_ATTR SET LIST_ORD =  32 WHERE  RPT_MENU_NO = 105 AND RPT_NO = 23090001; 
  UPDATE TA_REPORT_CAT_ATTR SET LIST_ORD =  34 WHERE  RPT_MENU_NO = 105 AND RPT_NO = 23100001; 
  UPDATE TA_REPORT_CAT_ATTR SET LIST_ORD =  32 WHERE  RPT_MENU_NO = 105 AND RPT_NO = 23220001; 
  UPDATE TA_REPORT_CAT_ATTR SET LIST_ORD =  32 WHERE  RPT_MENU_NO = 105 AND RPT_NO = 23230001; 
  UPDATE TA_REPORT_CAT_ATTR SET LIST_ORD =  8 WHERE  RPT_MENU_NO = 105 AND RPT_NO = 32270001;  
  UPDATE TA_REPORT_CAT_ATTR SET LIST_ORD =  1 WHERE  RPT_MENU_NO = 105 AND RPT_NO = 41010001;  
  UPDATE TA_REPORT_CAT_ATTR SET LIST_ORD =  2 WHERE  RPT_MENU_NO = 105 AND RPT_NO = 41010002;  
  UPDATE TA_REPORT_CAT_ATTR SET LIST_ORD =  3 WHERE  RPT_MENU_NO = 105 AND RPT_NO = 41010005;  
  UPDATE TA_REPORT_CAT_ATTR SET LIST_ORD =  4 WHERE  RPT_MENU_NO = 105 AND RPT_NO = 41010008;  
  UPDATE TA_REPORT_CAT_ATTR SET LIST_ORD =  41 WHERE  RPT_MENU_NO = 105 AND RPT_NO = 41010010;
  UPDATE TA_REPORT_CAT_ATTR SET LIST_ORD =  47 WHERE  RPT_MENU_NO = 105 AND RPT_NO = 61010001;
+----------------------------------------------------------------------------------------------+

```
맨 밑의 SELECT문 결과 안에 있는 UPDATE 문들을 실행



-- REPORT DATA Card 생성

```
USE INS_IDA;
SELECT PROFILE_NO,PROFILE_TP, PROFILE_NM FROM TA_PROFILE WHERE PROFILE_NO < 991001;
+------------+------------+------------+
| PROFILE_NO | PROFILE_TP | PROFILE_NM |
+------------+------------+------------+
|        105 | 991003     | WT_SaaS    |
+------------+------------+------------+

mysql> CALL INFOGRAPHIC_DEFAULT_CARD_CREATE(105); <- 위 SELECT문 결과의 PROFILE_NO을 인자값으로 수행 
```

이하 SELECT문으로 실행 결과 확인
```
SELECT CARD_NO,ACCOUNT_NO,CARD_NM,REG_DT FROM TG_DATACARD;
+---------+------------+---------------------------------+---------------------+
| CARD_NO | ACCOUNT_NO | CARD_NM                         | REG_DT              |
+---------+------------+---------------------------------+---------------------+
|       1 |          1 | 페이지뷰 & 방문수               | 2013-04-10 17:16:26 |
|       2 |          1 | 일 순수 방문자수                | 2013-04-10 17:16:26 |
...... 생략 ......
|      43 |          1 | CPC매체                         | 2013-04-10 17:16:26 |
|      44 |          1 | CPC키워드                       | 2013-04-10 17:16:26 |
+---------+------------+---------------------------------+---------------------+
```





#### 기타 서버 설정

**필수**
-- ulimit 수정 

```
vi /etc/security/limits.conf
+-----------------------------------------------+
root            hard    nofile          565535
root            soft    nofile          565535
mysql           hard    nofile          565535
mysql           soft    nofile          565535
wisetracker         hard    nofile          565535
wisetracker         soft    nofile          565535
+-----------------------------------------------+
```
쉘 재접속하여 적용

```
ulimit -a
+------------------------------------------------+
...... 생략 ......
open files                      (-n) 565535
...... 생략 ......
+------------------------------------------------+

service tomcat restart
```



# 5. 기타사항(설치 이외 부가적인 서버 관리 내용)

### UTIL SCRIPT INSTALL   

```
cd /home/wisetracker/server/
cp -pR /home/wisetracker/work/util/INSIGHT_UTIL /home/wisetracker/server/wisetracker_UTIL
vi wisetracker_UTIL
```


이하 코드는 wisetracker_UTIL의 내용아다. configure 값 수정하여 사용

```
+---------------------------------------------------------------------------------------------------------------+
#!/bin/bash

######################### Configure ##########################

MYSQL_PATH=/usr/local/mysql/bin/mysql
MYSQL_USER=2n9soft
MYSQL_PW=2n9soft

MYSQLDB_IDA=TEST_IDA
MYSQLDB_IDR=TEST_IDR1

MYSQL_HOST_IDA=122.99.192.146
MYSQL_HOST_IDR=122.99.192.146

MYSQL_DATA_DIR=/var/local/mysql/data
MYSQL_DATA_ERROR_FILE=$MYSQL_DATA_DIR/`hostname`.err

CATALINAOUT=/home/wisetracker/tomcat/logs/catalina.out

#############################################################


while ( true ) ; do

   echo "

▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒
▒ BIZSPRING wisetracker CHECK UTIL
▒                       By JSZZANG
▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒▒

    1.  서버 부하 
    2.  디스크사용률
    3.  수집확인
    4.  TOMCAT LOG(debugt)
    5.  MySQL LOG(vi at root)
    6.  TOMCAT 프로세스 Alive
    7.  MYSQL 프로세스 Alive
    8.  DB 프로세스 리스트(IDA)
    9.  DB 정제진행상황
    10. DB 상세정제진행상황
    11. DB PROFILE 간단 정보
    12. IPTABLE 로딩확인
    13. DB MYSQL 접속(IDA)
    q.  프로그램 종료
"

   echo -n "번호 선택 : "
   read no

PROCESS_3_QUERY="
SELECT TM,MIN(VT),MAX(VT),COUNT(*) FROM TR_PAGES_SLOT_1_0 GROUP BY TM
"

PROCESS_9_QUERY="
SELECT 
A.SCHE_TP, A.SCHE_NM, MIN( IFNULL( B.LCD, A.LCD ) )  AS LCD ,A.ING_YN,A.ACTIVE_ST 
FROM TA_SCHEDULE A   JOIN  TA_SCHEDULE_DETAIL B
ON A.SCHE_NO = B.SCHE_NO 
WHERE A.ACTIVE_ST = 'ACT001'  
GROUP BY A.SCHE_TP , A.SCHE_NM,A.ACTIVE_ST 
UNION ALL
SELECT 
A.SCHE_TP, A.SCHE_NM, MIN( IFNULL( B.LCD, A.LCD ) )  AS LCD,A.ING_YN, A.ACTIVE_ST
FROM TA_SCHEDULE A   LEFT JOIN  TA_SCHEDULE_DETAIL B
ON A.SCHE_NO = B.SCHE_NO 
WHERE A.ACTIVE_ST = 'ACT001'  
AND     A.SCHE_TP != 'STPR01' 
GROUP BY A.SCHE_TP , A.SCHE_NM,A.ACTIVE_ST
"
PROCESS_10_QUERY="
SELECT SCHE_NO,LCD,ING_YN,COUNT(*) 
FROM TA_SCHEDULE_DETAIL 
WHERE ACTIVE_ST='ACT001' GROUP BY SCHE_NO,ING_YN  
"
PROCESS_11_QUERY="
SELECT PROFILE_NO,PROFILE_NM,BASE_URL,INTERNAL_URL,SLOT_NO,TRK_SLOT_NO,SUB_SLOT_CNT 
FROM TA_PROFILE 
WHERE PROFILE_NO < 991001 AND ACTIVE_ST='ACT001'
"

PROCESS_12_QUERY="
SELECT COUNT(*) FROM TA_IPTABLE
"

   case $no in
     "1" )
          top ;;
     "2" )
          df -h ;;
     "3" )
          $MYSQL_PATH -h$MYSQL_HOST_IDR -u$MYSQL_USER -p$MYSQL_PW $MYSQLDB_IDR -e "$PROCESS_3_QUERY" ;;
     "4" )
          tail -fn200 $CATALINAOUT ;;
     "5" )
          vi $MYSQL_DATA_ERROR_FILE ;;
     "6" )
          ps -ef | grep java ;;
     "7" )
          ps -ef | grep mysql ;;
     "8" )
          $MYSQL_PATH -h$MYSQL_HOST_IDA -u$MYSQL_USER -p$MYSQL_PW $MYSQLDB_NAME -e "SHOW PROCESSLIST" ;;
     "9" )
          $MYSQL_PATH -h$MYSQL_HOST_IDA -u$MYSQL_USER -p$MYSQL_PW $MYSQLDB_IDA -e "$PROCESS_9_QUERY" ;;
     "10" )
          $MYSQL_PATH -h$MYSQL_HOST_IDA -u$MYSQL_USER -p$MYSQL_PW $MYSQLDB_IDA -e "$PROCESS_10_QUERY" ;;
     "11" )
          $MYSQL_PATH -h$MYSQL_HOST_IDA -u$MYSQL_USER -p$MYSQL_PW $MYSQLDB_IDA -e "$PROCESS_11_QUERY" ;;
     "12" )
          $MYSQL_PATH -h$MYSQL_HOST_IDA -u$MYSQL_USER -p$MYSQL_PW $MYSQLDB_IDA -e "$PROCESS_12_QUERY" ;;
     "13" )
          $MYSQL_PATH -h$MYSQL_HOST_IDA -u$MYSQL_USER -p$MYSQL_PW ;;
     "q" )
          exit 0 ;;
   esac
done
+---------------------------------------------------------------------------------------------------------------+


vi /etc/profile

alias wisetracker='sh /home/wisetracker/server/wisetracker_UTIL'

source /etc/profile

```

### BACKUP SCRIPT INSTALL 

#### 1. CRON 셋팅(운영시 #!!# 주석을 풀고 운영)

```
crontab -uwisetracker -e
+------------------------------------------------------------------------------+
######################## wisetracker BACKUP : SCHEMA ######################
## ※백업위치 IDA(TA,TS) 
## /home/wisetracker/server/backup/SQL_$DATABASE_NAME/CREATE/IDA/TA (267개)
## /home/wisetracker/server/backup/SQL_$DATABASE_NAME/CREATE/IDA/TS (472개)
## ※백업위치 IDR(TA,TS)
## /home/wisetracker/server/backup/SQL_$DATABASE_NAME/CREATE/IDR/TA (2개)
## /home/wisetracker/server/backup/SQL_$DATABASE_NAME/CREATE/IDR/TR (682개)
## mkdir /home/wisetracker/server/backup/logs 디렉토리가 있어야함
##################################################################

###### Day (매일 00,01시 1분) ######
#!!#01 00 * * * /bin/bash /home/wisetracker/wisetrackerDatabaseBackup_For_SCHEMA_IDA.sh
#!!#01 01 * * * /bin/bash /home/wisetracker/wisetrackerDatabaseBackup_For_SCHEMA_IDR.sh


######################## wisetracker BACKUP : DATA ########################
## ※백업위치
## /home/wisetracker/server/backup/SQL_$DATABASE_NAME/DATA/1DAY
## /home/wisetracker/server/backup/SQL_$DATABASE_NAME/DATA/1MONTH
## /home/wisetracker/server/backup/SQL_$DATABASE_NAME/DATA/6MONTH
## 총IDA(656) = TA(265개) + TS(375개) 둘다빠지는(14(형식이틀리고 스키마만있어도되어서 + 뷰2)
################################################################
###### Day (매일 02시 01분) ######
#!!#01 02 * * * /home/wisetracker/wisetrackerDatabaseBackup_For_DATA.sh 1
###### Month (매월 1일 03시 01분) ######
#!!#01 03 01 * * /home/wisetracker/wisetrackerDatabaseBackup_For_DATA.sh 2
###### Half Year(1,6월 2일 03시 01분) ######
#!!#01 03 02 01,06 * /home/wisetracker/wisetrackerDatabaseBackup_For_DATA.sh 3
+------------------------------------------------------------------------------+
```

#### 2. 백업 디렉토리 생성

```
mkdir -p /home/wisetracker/server/backup/log
chown -R wisetracker:wisetracker /home/wisetracker/server/backup

cp /home/wisetracker/work/util/wisetrackerDatabaseBackup_For_* /home/wisetracker/
chown -R wisetracker:wisetracker /home/wisetracker/wisetrackerDatabaseBackup_For*
```

#### 3. BACKUP SOURCE 체크 Or 수정 (/home/wisetracker/work/util 원본소스이용)

3-1) 
```
/home/wisetracker/wisetrackerDatabaseBackup_For_SCHEMA_IDA.sh
+------------------------------------------------------------------------------+
MYSQL_HOST=localhost
DATABASE_NAME=고객사별칭_IDA
MYSQL_HOME=/home/wisetracker/mysql
+------------------------------------------------------------------------------+
```

3-2) 
```
/home/wisetracker/wisetrackerDatabaseBackup_For_SCHEMA_IDR.sh
+------------------------------------------------------------------------------+
MYSQL_HOST=localhost
DATABASE_NAME=고객사별칭_IDR1
MYSQL_HOME=/home/wisetracker/mysql
+------------------------------------------------------------------------------+
```


3-3) 
```
/home/wisetracker/wisetrackerDatabaseBackup_For_DATA.sh
+------------------------------------------------------------------------------+
MYSQL_HOST=localhost
DATABASE_NAME=고객사별칭_IDA
MYSQL_HOME=/home/wisetracker/mysql
USER_HOME=/home/wisetracker
+------------------------------------------------------------------------------+
```










