# Workshop02 시작
_엔터프라이즈 모놀리틱 DB를 MSA 구조로 전환하기_ 세션의 Workshop2에 오신 것을 환영합니다.
Workshop2 에서는 Oracle 데이터를 Elasticache Redis로 마이그레이션하여 실시간 리더보드를 구성해보는 실습을 해보도록 하겠습니다.
이번 워크샵은 크게 3단계로 진행됩니다.
1. Oracle 데이터베이스에서 데이터 변경이 지속적으로 발생하는 동안 Rank 함수를 사용하여 Ranking 데이터 생성하고 영향도 확인하기
2. Ranking 데이터 CSV 파일을 Elasticache Redis의 Sorted Set으로 마이그레이션 하기
3. Elasticache Redis에 데이터 변경이 지속적으로 발생하는 동안 Ranking 조회 후 영향도 확인하기

실습 참가자들의 이해를 돕기 위해서 아래의 가상 시나리오를 생각하며 실습을 진행하도록 하겠습니다.
```
당신은 사용자의 레벨과 경험치를 기준으로 Leaderboard 제공하는 서비스를 담당하고 있습니다.
Leaderboard 서비스의 데이터 저장소로 Oracle을 사용하고 있고, Leaderboard 생성으로 인한 시스템 자원 사용률 증가가 
대고객 서비스의 지연을 발생시킨다는 것을 알고 있습니다.
그래서 별도의 DB서버를 구성하여 그곳에서 Leaderboard 데이터를 생성하고, 그 결과를 다시 서비스 DB서버로 복사하는 아키텍처를 구성하였습니다.
서비스의 사용자수가 점점 증가함에 따라 Leaderboard 데이터 생성과 데이터 복사에 점점 많은 시간이 소요되고 있습니다.
또 고객의 실시간 Leaderboard에 대한 요구사항도 점점 증가하고 있습니다.

이런 문제를 해결하기 위해서 Leaderboard 데이터 저장소를 In-Memory DB로 변경하려고 합니다.
```
1. 사전 준비
실습을 진행하기 위해서 '사전준비' 단계를 먼저 수행해주시기 바랍니다.
Cloudformation ~ Windows Server 까지 접속하기

2. MobaXterm을 실행하고 세션들을 활성화 합니다.  
Oracle, Redis, Legacy_server, MSA_server 이렇게 4개의 세션을 만듭니다.
![sessions](./images/sessions.png)

3. Oracle database를 데이터 저장소로 사용하고 있는 Leaderboard Application을 구동합니다.  
MobaXterm에서 Legacy_server tab으로 이동하여 아래 명령어를 수행합니다.
```
ec2-user@ip-10-100-1-101:/home/ec2-user> cd workshop02/legacy
ec2-user@ip-10-100-1-101:/home/ec2-user/workshop02/legacy> source bin/activate
(legacy) ec2-user@ip-10-100-1-101:/home/ec2-user/workshop02/legacy> flask run --host=0.0.0.0 --port=4000
 * Environment: production
   WARNING: This is a development server. Do not use it in a production deployment.
   Use a production WSGI server instead.
 * Debug mode: off
 * Running on all addresses.
   WARNING: This is a development server. Do not use it in a production deployment.
 * Running on http://10.100.1.101:4000/ (Press CTRL+C to quit)
```
서버 어플리케이션이 구동되었습니다.

4. Oracle database의 Real time leaderboard 구성에 대한 영향도를 확인해 봅니다.  
[Gatling](https://gatling.io/)을 사용하여 여러 유저들의 Level을 변경하는 요청을 보내고, 요청이 수행되는 동안 Oracle rank함수를 사용하여 ranking 데이터를 조회해 봅니다.
부하 주입은 Windows 서버에서 실행하게 됩니다.  
윈도우즈 서버에서 Command Prompt 윈도우를 실행합니다.  
Command Prompt에서 아래 명령를 차례로 입력합니다.  
부하 주입 시간은 3분으로 설정되어 있습니다.
```
C:\Users\Administrator> CD C:\gatling\bin
C:\gatling\bin> gatling.bat
GATLING_HOME is set to "C:\gatling"
JAVA = "java"
Choose a simulation number:
     [0] SeoulSummit.Workshop02_legacy
     [1] SeoulSummit.Workshop02_msa
     [2] SeoulSummit.Workshop04_legacy
     [3] SeoulSummit.Workshop04_msa
0(0번 시나리오를 수행합니다.)
Select run description (optional)
(엔터를 한번 더 입력합니다.)
Simulation SeoulSummit.Workshop2_legacy started...

================================================================================
2022-03-06 15:51:14                                           5s elapsed
---- Requests ------------------------------------------------------------------
> Global                                                   (OK=601    KO=0     )
> UpdateLevel                                              (OK=601    KO=0     )

---- Workshop02_legacy ---------------------------------------------------------
          active: 9      / done: 601
================================================================================
```
Level update 요청이 수행되는 동안 Oracle에 접속하여 rank 함수를 호출하여 ranking data를 조회합니다.
MobaXterm의 Oracle session에서 아래 명령을 수행합니다.
```
ec2-user@ip-10-100-1-101:/home/ec2-user> sudo su -
Last login: Tue Feb  8 06:07:37 UTC 2022 on pts/0
root@ip-10-100-1-101:/root# su - oracle
Last login: Tue Feb  8 06:07:44 UTC 2022 on pts/0
oracle@ip-10-100-1-101:/home/oracle> sqlplus oshop/oshop

SQL*Plus: Release 11.2.0.2.0 Production on Tue Feb 8 06:12:37 2022

Copyright (c) 1982, 2011, Oracle.  All rights reserved.


Connected to:
Oracle Database 11g Express Edition Release 11.2.0.2.0 - 64bit Production

SQL> SELECT userid, rank() OVER (ORDER BY USERLEVEL DESC, EXPOINT DESC) AS rank FROM USER_SCORE;

```
![image](./images/1.png)
부하가 끝나면 아래와 같이 통계정보를 확인할 수 있습니다.
```
================================================================================
---- Global Information --------------------------------------------------------
> request count                                      22320 (OK=22320  KO=0     )
> min response time                                     23 (OK=23     KO=-     )
> max response time                                    175 (OK=175    KO=-     )
> mean response time                                    80 (OK=80     KO=-     )
> std deviation                                         16 (OK=16     KO=-     )
> response time 50th percentile                         79 (OK=79     KO=-     )
> response time 75th percentile                         90 (OK=90     KO=-     )
> response time 95th percentile                        107 (OK=107    KO=-     )
> response time 99th percentile                        123 (OK=123    KO=-     )
> mean requests/sec                                123.315 (OK=123.315 KO=-     )
---- Response Time Distribution ------------------------------------------------
> t < 800 ms                                         22320 (100%)
> 800 ms < t < 1200 ms                                   0 (  0%)
> t > 1200 ms                                            0 (  0%)
> failed                                                 0 (  0%)
================================================================================

Reports generated in 0s.
Please open the following file: C:\gatling\results\workshop02-legacy-20220306065107937\index.html
Press any key to continue . . .
```
Gatling은 웹기반의 리포팅을 제공합니다. 부하가 끝난 후 제공되는 html 링크를 웹브라우저를 사용하여 열어봅니다.
Ranking 조회 쿼리가 수행된 구간에서 p95의 Response Time이 상승하고 요청 처리량이 감소하는 그래프를 볼 수 있습니다.
![image2](./images/2.png)
![image2-1](./images/2-1.png)
![image3](./images/3.png)

MobaXterm의 Legacy_server 세션으로 이동하여 CTRL+C 를 눌러 어플리케이션을 종료합니다.
```
10.100.1.103 - - [08/Feb/2022 06:28:07] "GET /legacy/updateuserlevel HTTP/1.1" 200 -
^C(legacy) ec2-user@ip-10-100-1-101:/home/ec2-user/workshop2/legacy>
```

5. Oracle의 Leaderboard 데이터를 Elasticache Redis로 마이그레이션 합니다.  
MobaXterm에서 Oracle 세션으로 이동하여 Redis로 데이터를 마이그레이션 하기 위한 스테이징 테이블과 데이터를 만듭니다.
~~~ SQL
CREATE TABLE "OSHOP"."USER_SCORE_REDIS_FORMAT" 
   (	"key" VARCHAR2(15), 
	"score" NUMBER, 
	"member" NUMBER, 
	 CONSTRAINT "USER_SCORE_REDIS_FORMAT_PK" PRIMARY KEY ("member"));

INSERT INTO USER_SCORE_REDIS_FORMAT
SELECT 'leaderboard' AS key,  
	userlevel||LPAD(expoint, 13, '0') AS score,
	USERID AS MEMBER
FROM USER_SCORE us;

commit;
~~~
스테이징 테이블을 CSV파일로 저장합니다.
MobaXterm에서 MSA_Server 세션으로 이동하여 아래 명령어를 수행합니다.
~~~
ec2-user@ip-10-100-1-101:/home/ec2-user> cd workshop02/msa
ec2-user@ip-10-100-1-101:/home/ec2-user/workshop02/msa> sudo su -
Last login: Sun Mar  6 03:28:55 UTC 2022 on pts/3
root@ip-10-100-1-101:/root# su - oracle
Last login: Sun Mar  6 03:28:58 UTC 2022 on pts/3
oracle@ip-10-100-1-101:/home/oracle> cd workshop02
oracle@ip-10-100-1-101:/home/oracle/workshop02> sh 01-unload-data.sh
oracle@ip-10-100-1-101:/home/oracle/workshop02> ls
01-unload-data.sh  unload_user_score.sql  user_score.csv
oracle@ip-10-100-1-101:/home/oracle/workshop02> tail user_score.csv
leaderboard,     941645071728030,                              296488
leaderboard,     931645071728030,                              296489
leaderboard,     251645071728030,                              296490
leaderboard,     121645071728030,                              296491
leaderboard,     151645071728030,                              296492
leaderboard,     961645071728030,                              296493
leaderboard,     771645071728030,                              296494
leaderboard,     601645071728030,                              296495
leaderboard,     91645071728030,                               296496
leaderboard,     211645071728030,                              296497
~~~
데이터가 CSV 파일로 저장되었습니다.
이 파일을 Redis서버로 마이그레이션 합니다.
~~~
oracle@ip-10-100-1-101:/home/oracle/workshop02> exit
logout
root@ip-10-100-1-101:/root# exit
logout
ec2-user@ip-10-100-1-101:/home/ec2-user/workshop02/msa> sh 01-copy-score.sh
ec2-user@ip-10-100-1-101:/home/ec2-user/workshop02/msa> sh 02-transform.sh
ec2-user@ip-10-100-1-101:/home/ec2-user/workshop02/msa> sh 03-load-data-to-redis.sh
Warning: Using a password with '-a' or '-u' option on the command line interface may not be safe.
All data transferred. Waiting for the last reply...
Last reply received from server.
errors: 0, replies: 300000
~~~
Redis에 접속하여 마이그레이션된 데이터를 확인합니다.
~~~
ec2-user@ip-10-100-1-101:/home/ec2-user/workshop02/msa> redis-cli -a Welcome1234
Warning: Using a password with '-a' or '-u' option on the command line interface may not be safe.
127.0.0.1:6379> zcard leaderboard
(integer) 300000
127.0.0.1:6379> zrevrange leaderboard 0 9
 1) "299830"
 2) "299827"
 3) "299472"
 4) "299265"
 5) "299209"
 6) "298755"
 7) "298525"
 8) "298502"
 9) "298233"
10) "298162"
127.0.0.1:6379> zrevrank leaderboard 299830
(integer) 0
127.0.0.1:6379> exit
ec2-user@ip-10-100-1-101:/home/ec2-user/workshop02/msa>
~~~
6. Redis에서 실시간 Leaderboard 서비스를 테스트 해봅니다.
MobaXterm의 MSA_server 세션으로 이동하여 Redis를 데이터 스토어로 사용하는 Leaderboard 서비스를 실행합니다.
```
ec2-user@ip-10-100-1-101:/home/ec2-user/workshop02/msa> source bin/activate
(msa) ec2-user@ip-10-100-1-101:/home/ec2-user/workshop02/msa> flask run --host=0.0.0.0 --port=4000
 * Environment: production
   WARNING: This is a development server. Do not use it in a production deployment.
   Use a production WSGI server instead.
 * Debug mode: off
 * Running on all addresses.
   WARNING: This is a development server. Do not use it in a production deployment.
 * Running on http://10.100.1.101:4000/ (Press CTRL+C to quit)
 ```
Legacy와 동일하게 Gatling을 활용하여 Leaderboard의 Level을 업데이트하면서 ranking을 조회해보고 영향도를 확인합니다.
Windows 서버의 cmd 를 열고 아래 명령을 실행합니다.
```
C:\Users\Administrator> CD C:\gatling\bin
C:\gatling\bin> gatling.bat
GATLING_HOME is set to "C:\gatling"
JAVA = "java"
Choose a simulation number:
     [0] SeoulSummit.Workshop02_legacy
     [1] SeoulSummit.Workshop02_msa
     [2] SeoulSummit.Workshop04_legacy
     [3] SeoulSummit.Workshop04_msa
1(1번 시나리오를 수행합니다.)
Select run description (optional)
(엔터를 한번 더 입력합니다.)
Simulation SeoulSummit.Workshop2_legacy started...

================================================================================
2022-03-06 14:22:22                                           5s elapsed
---- Requests ------------------------------------------------------------------
> Global                                                   (OK=3034   KO=0     )
> UpdateLevel                                              (OK=3034   KO=0     )

---- Workshop02_msa ------------------------------------------------------------
          active: 10     / done: 3034
================================================================================
```
Leaderboard 데이터 변경 요청을 주입하는 동안 Elasticache에서 ranking 조회를 해봅니다.
MobaXterm 에서 Redis 탭으로 이동하여 아래 명령을 수행합니다.
데이터들이 업데이트 되면서 ranking이 실시간으로 계속 바뀌는 것을 확인할 수 있습니다.
```
ec2-user@ip-10-100-1-101:/home/ec2-user> cd workshop02/msa
ec2-user@ip-10-100-1-101:/home/ec2-user/workshop02/msa> redis-cli -a Welcome1234
127.0.0.1:6379> zrevrange leaderboard 10000 10010
 1) "25408"
 2) "144669"
 3) "91035"
 4) "43661"
 5) "158444"
 6) "92597"
 7) "144895"
 8) "142565"
 9) "962"
10) "56587"
11) "18594"
127.0.0.1:6379> zrevrange leaderboard 10000 10010
 1) "78525"
 2) "36589"
 3) "74403"
 4) "184208"
 5) "155603"
 6) "65338"
 7) "277657"
 8) "209997"
 9) "29455"
10) "20594"
11) "81012"
```
부하 종료 후 통계 데이터를 확인합니다.
데이터 변경 처리량이 Oracle보다 높은 것을 확인할 수 있습니다.
```
================================================================================
---- Global Information --------------------------------------------------------
> request count                                     114373 (OK=114373 KO=0     )
> min response time                                      2 (OK=2      KO=-     )
> max response time                                     41 (OK=41     KO=-     )
> mean response time                                    16 (OK=16     KO=-     )
> std deviation                                          3 (OK=3      KO=-     )
> response time 50th percentile                         15 (OK=15     KO=-     )
> response time 75th percentile                         17 (OK=17     KO=-     )
> response time 95th percentile                         20 (OK=20     KO=-     )
> response time 99th percentile                         22 (OK=22     KO=-     )
> mean requests/sec                                631.895 (OK=631.895 KO=-     )
---- Response Time Distribution ------------------------------------------------
> t < 800 ms                                        114373 (100%)
> 800 ms < t < 1200 ms                                   0 (  0%)
> t > 1200 ms                                            0 (  0%)
> failed                                                 0 (  0%)
================================================================================

Reports generated in 0s.
Please open the following file: C:\gatling\results\workshop02-msa-20220306052215938\index.html
Press any key to continue . . .
```
웹기반의 보고서를 확인하기 위해 위의 링크를 웹브라우저로 열어봅니다.
평균 응답속도가 95%의 평균 응답속도가 20ms을 유지하며 ranking을 조회하더라도 초당 처리량은 변화가 없는 것을 확인할 수 있습니다.
![image](./images/5.png)
![image](./images/5-1.png)
![image](./images/6.png)