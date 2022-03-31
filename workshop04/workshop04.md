# Workshop04 시작
엔터프라이즈 모놀리틱 DB를 MSA 구조로 전환하기 세션의 Workshop04에 오신 것을 환영합니다.  
Workshop04 에서는 Oracle의 주문 조회용 데이터를 Amazon DynamoDB 로 마이그레이션해 보고, Gatling을 활용하여 부하 주입 및 성능 측정을 해보도록 하겠습니다.

이번 워크샵은 크게 3단계로 진행됩니다.
1. Oracle 데이터베이스에서 구매내역 조회 및 성능 확인
2. DMS를 활용하여 Oracle 데이터를 DynamoDB로 마이그레이션
3. DynamoDB 에서 구매내역 조회 및 성능 확인

실습 참가자들의 이해를 돕기 위해서 아래의 가상 시나리오를 생각하며 실습을 진행하도록 하겠습니다.
~~~
당신은 온라인 마켓 서비스를 담당하고 있습니다.
OLTP 서비스에서 가장 일반적으로 사용되는 Oracle Database를 데이터 저장소로 사용하고 있습니다.
데이터 모델링 규칙에 맞추어 정규화된 여러 테이블들에 데이터들이 저장되고, 각 테이블들은 참조키로 관계가 맺어져 있습니다.
각 요청에 필요한 데이터들은 여러 테이블들을 조인하여 만들어지고 어플리케이션으로 전달됩니다.
서비스가 점점 확장됨에 따라 비지니스 규칙에 따라 정규화된 여러 테이블들에 데이터를 인서트하거나,
여러 테이블들을 조인해서 데이터를 조회하는 부담이 점점 늘어나고 있습니다.
향후 서비스의 확장을 대비하여 유연한 확장성과 고가용성, 일관된 성능을 보장하는 AWS 완전관리형의 serverless nosql DB인
DynamoDB 로 데이터를 마이그레이션 하려고 합니다.

이번 실습을 통해 간단한 구매내역 조회 서비스를 대상으로 Oracle 과 DynamoDB 성능 비교를 해보고,
데이터 마이그레이션은 DMS를 활용해봄으로서 RDB 에서 NoSQL 로 마이그레이션 하는 기본적인 방법을 배워봅니다.
~~~
### 1. MobaXterm 세션들을 활성화 합니다.
Oracle, Legacy_server, MSA_server 이렇게 3개의 세션을 만들어 줍니다.
![sessions](./images/sessions.png)

### 1. 구매내역을 조회하는 Oracle 기반의 어플리케이션을 기동합니다.
MobaXterm의 Legacy_server에서 아래의 명령어를 수행합니다.
~~~
ec2-user@ip-10-100-1-101:/home/ec2-user> cd workshop04/legacy
ec2-user@ip-10-100-1-101:/home/ec2-user/workshop04/legacy> source bin/activate
(legacy) ec2-user@ip-10-100-1-101:/home/ec2-user/workshop04/legacy> flask run --host=0.0.0.0 --port=4000
 * Environment: production
   WARNING: This is a development server. Do not use it in a production deployment.
   Use a production WSGI server instead.
 * Debug mode: off
 * Running on all addresses.
   WARNING: This is a development server. Do not use it in a production deployment.
 * Running on http://10.100.1.101:4000/ (Press CTRL+C to quit)
~~~
### 2. Gatling을 사용하여 legacy 시스템(오라클)에 구매내역 조회 부하를 주입하고 어플리케이션 성능을 측정합니다.
아래 명령어는 Windows Server에서 cmd에서 수행합니다.
~~~
C:\Users\Administrator> CD C:\gatling\bin
C:\gatling\bin> gatling.bat
GATLING_HOME is set to "C:\gatling"
JAVA = "java"
Choose a simulation number:
     [0] SeoulSummit.Workshop02_legacy
     [1] SeoulSummit.Workshop02_msa
     [2] SeoulSummit.Workshop04_legacy
     [3] SeoulSummit.Workshop04_msa
2(2번 시나리오를 수행합니다.)
Select run description (optional)
(엔터를 한번 더 입력합니다.)
Simulation SeoulSummit.Workshop04_legacy started...

================================================================================
2022-03-06 20:42:11                                           5s elapsed
---- Requests ------------------------------------------------------------------
> Global                                                   (OK=114    KO=0     )
> selectPurchase                                           (OK=114    KO=0     )

---- Workshop04_legacy ---------------------------------------------------------
          active: 2      / done: 114
================================================================================
~~~
부하가 종료된 후 성능 통계정보를 확인합니다. p95의 평균 응답시간은 92ms 입니다.
~~~
================================================================================
---- Global Information --------------------------------------------------------
> request count                                       4395 (OK=4395   KO=0     )
> min response time                                     57 (OK=57     KO=-     )
> max response time                                    111 (OK=111    KO=-     )
> mean response time                                    81 (OK=81     KO=-     )
> std deviation                                          8 (OK=8      KO=-     )
> response time 50th percentile                         84 (OK=84     KO=-     )
> response time 75th percentile                         88 (OK=88     KO=-     )
> response time 95th percentile                         92 (OK=92     KO=-     )
> response time 99th percentile                         95 (OK=95     KO=-     )
> mean requests/sec                                 24.282 (OK=24.282 KO=-     )
---- Response Time Distribution ------------------------------------------------
> t < 800 ms                                          4395 (100%)
> 800 ms < t < 1200 ms                                   0 (  0%)
> t > 1200 ms                                            0 (  0%)
> failed                                                 0 (  0%)
================================================================================

Reports generated in 0s.
Please open the following file: C:\gatling\results\workshop04-legacy-20220306114205590\index.html
Press any key to continue . . .
~~~
위의 링크를 웹브라우저로 열어서 Web으로 제공되는 성능 보고서도 확인해 봅니다.
![image 3](./images/3.png)
![image 3-1](./images/3-1.png)
![image 4](./images/4.png)

MobaXterm Legacy_server로 이동 후 ctrl+C로 어플리케이션을 중지합니다.

### 3. DMS를 활용하여 Oracle data를 DynamoDB로 마이그레이션 합니다.
DynamoDB에 데이터를 마이그레션을 하기전에 DynamoDB에 효율적인 key-value 형태로 스테이징 테이블 및 데이터를 생성합니다.  
MobaXterm의 Oracle 세션으로 이동하여 오라클 서버에 접속합니다.
~~~
ec2-user@ip-10-100-1-101:/home/ec2-user> sudo su -
Last login: Sun Mar  6 08:06:00 UTC 2022 on pts/0
root@ip-10-100-1-101:/root# su - oracle
Last login: Sun Mar  6 08:06:13 UTC 2022 on pts/0
oracle@ip-10-100-1-101:/home/oracle> sqlplus oshop/oshop

SQL*Plus: Release 11.2.0.2.0 Production on Sun Mar 6 08:55:01 2022

Copyright (c) 1982, 2011, Oracle.  All rights reserved.


Connected to:
Oracle Database 11g Express Edition Release 11.2.0.2.0 - 64bit Production

SQL> 
~~~
오라클에 접속하였으면 아래 쿼리를 수행하여 테이블 및 데이터를 만듭니다.
~~~
CREATE TABLE "OSHOP"."PURCHASE_DYNAMODB_FORMAT" 
(
    "CUSTOMER_ID" VARCHAR2(20 BYTE),
    "PURCHASE_ID" VARCHAR2(20 BYTE), 
    "PURCHASE_DATE" VARCHAR2(10 BYTE),
    "CUSTOMER_NAME" VARCHAR2(10 BYTE),    
    "CUSTOMER_ADDRESS" VARCHAR2(100 BYTE),
    "PURCHASE_SEQ_1" VARCHAR2(20 BYTE), 
    "PRODUCT_NAME_1" VARCHAR2(100 BYTE), 
    "PRODUCT_PRICE_1" VARCHAR2(20 BYTE), 
    "QUANTITY_1" VARCHAR2(20 BYTE), 
    "SELLER_TEL_NUMBER_1" VARCHAR2(20 BYTE), 
    "PURCHASE_SEQ_2" VARCHAR2(20 BYTE), 
    "PRODUCT_NAME_2" VARCHAR2(100 BYTE), 
    "PRODUCT_PRICE_2" VARCHAR2(20 BYTE), 
    "QUANTITY_2" VARCHAR2(20 BYTE), 
    "SELLER_TEL_NUMBER_2" VARCHAR2(20 BYTE), 
    "PURCHASE_SEQ_3" VARCHAR2(20 BYTE), 
    "PRODUCT_NAME_3" VARCHAR2(100 BYTE), 
    "PRODUCT_PRICE_3" VARCHAR2(20 BYTE), 
    "QUANTITY_3" VARCHAR2(20 BYTE), 
    "SELLER_TEL_NUMBER_3" VARCHAR2(20 BYTE), 
    CONSTRAINT "PURCHASE_DYNAMODB_FT_PK" PRIMARY KEY ("CUSTOMER_ID", "PURCHASE_ID")
);

INSERT INTO PURCHASE_DYNAMODB_FORMAT
SELECT  c.customerid, pc.purchaseid, pc.purchasedate, c.name, c.address,
        MAX(case when purchaseseq = 1 then purchaseseq else 0 end) PURCHASE_SEQ_1,
        MAX(case when purchaseseq = 1 then pd.productname else '' end) PRODUCT_NAME_1,
        MAX(case when purchaseseq = 1 then pd.price else 0 end) PRODUCT_PRICE_1,
        MAX(case when purchaseseq = 1 then pc.quantity else 0 end) QUANTITY_1,
        MAX(case when purchaseseq = 1 then s.telnumber else '' end) SELLER_TEL_NUMBER_1,
        MAX(case when purchaseseq = 2 then purchaseseq else 0 end) PURCHASE_SEQ_2,
        MAX(case when purchaseseq = 2 then pd.productname else '' end) PRODUCT_NAME_2,
        MAX(case when purchaseseq = 2 then pd.price else 0 end) PRODUCT_PRICE_2,
        MAX(case when purchaseseq = 2 then pc.quantity else 0 end) QUANTITY_2,
        MAX(case when purchaseseq = 2 then s.telnumber else '' end) SELLER_TEL_NUMBER_2,
        MAX(case when purchaseseq = 3 then purchaseseq else 0 end) PURCHASE_SEQ_3,
        MAX(case when purchaseseq = 3 then pd.productname else '' end) PRODUCT_NAME_3,
        MAX(case when purchaseseq = 3 then pd.price else 0 end) PRODUCT_PRICE_3,
        MAX(case when purchaseseq = 3 then pc.quantity else 0 end) QUANTITY_3,
        MAX(case when purchaseseq = 3 then s.telnumber else '' end) SELLER_TEL_NUMBER_3
FROM purchase pc
	INNER JOIN customer c
	ON pc.customerid = c.customerid
	INNER JOIN product pd
	ON pc.productid = pd.productid
    INNER JOIN seller s
    ON pd.sellerid = s.sellerid
group by c.customerid, pc.purchaseid, pc.purchasedate, c.name, c.address;

commit;

~~~
스태이징 테이블을 구성하였으니 DMS를 통해서 DynamoDB로 데이터를 마이그레이션 합니다. DMS의 Replication Instance는 Workshop03에서 생성한 RI를 사용합니다. 만약 Workshop03을 수행하지 않았다면 Workshop03의 12.Replication Instance 생성 단계를 참고하여 Replicattion Instance를 생성합니다.  

Replication Instance를 생성하였다면 [DMS Console](https://ap-northeast-2.console.aws.amazon.com/dms/v2/home?region=ap-northeast-2#dashboard) 에서 Source와 Target endpoint를 생성합니다.  
왼쪽 메뉴에서 Endpoints로 이동 후 Create endpoint 버튼을 클릭합니다.
아래와 같이 Source endpoint에 대한 정보를 입력합니다.
* Endpoint identifier : s-seoulsummit-endpoint
* Source engine : Oracle
* Server name : 10.100.1.101
* Port : 1521
* User name : oshop
* Password : oshop
* SID/Service name : xe   
![image 8](./images/8.png)

VPC와 Replication instance 정보를 입력하고 Run test 버튼을 클릭하여 연결테스트를 수행합니다.
Test 가 성공하였다면 Create endpoint 버튼을 클릭합니다.  
![image 9](./images/9.png)

Target endpoint를 생성하기 위해서 Create endpoint 버튼을 클릭합니다.
아래와 같이 Target endpoint 에 대한 정보를 입력합니다.
* Endpoint identifier : t-seoulsummit-dynamodb1
* Target engine : Amazon DynamoDB
* Service access role ARN : arn:aws:iam::111111111111:role/EC2SSMRole2  
(Service access role ARN 은 CloudFormation의 seoul-summit 스택 Outputs 탭에서 확인할 수 있습니다.)  
![image 9-1](./images/9-1.png)
![image 10](./images/10.png)

Test endpoint connection을 수행합니다.
![image 11](./images/11.png)

다음으로 데이터 마이그레이션을 위한 Task를 생성합니다. 
[Task](https://ap-northeast-2.console.aws.amazon.com/dms/v2/home?region=ap-northeast-2#tasks) 페이지로 이동하여 Create task 버튼을 클릭합니다.

Task configuration 항목에 값을 설정합니다.
Replication instnace 는 CloudFormation outputs 항목의 ReplicationInstance를 참고합니다.
Source와 Target endpoint는 이전 단계에서 만든 endpoint 정보를 입력합니다.  
![image 16](./images/16.png)

Task settings를 아래와 같이 설정합니다.  
![image 17](./images/17.png)

Table mappings 정보를 설정합니다.
JSON editor 버튼을 선택하고 아래 editor에 JSON 을 입력합니다.
~~~json
{
    "rules": [
        {
            "rule-type": "selection",
            "rule-id": "1",
            "rule-name": "1",
            "object-locator": {
                "schema-name": "OSHOP",
                "table-name": "PURCHASE_DYNAMODB_FORMAT"
            },
            "rule-action": "include"
        },
        {
            "rule-type": "object-mapping",
            "rule-id": "2",
            "rule-name": "TransformToDDB",
            "rule-action": "map-record-to-record",
            "object-locator": {
                "schema-name": "OSHOP",
                "table-name": "PURCHASE_DYNAMODB_FORMAT"
            },
            "target-table-name": "purchase_t",
            "mapping-parameters": {
                "partition-key-name": "customer_id",
                "sort-key-name": "purchase_id",
                "exclude-columns": [
                    "CUSTOMER_ID",
                    "PURCHASE_DATE",
                    "PURCHASE_ID",
                    "CUSTOMER_NAME",
                    "CUSTOMER_ADDRESS",
                    "PURCHASE_SEQ_1",
                    "PRODUCT_NAME_1",
                    "PRODUCT_PRICE_1",
                    "QUANTITY_1",
                    "SELLER_TEL_NUMBER_1",
                    "PURCHASE_SEQ_2",
                    "PRODUCT_NAME_2",
                    "PRODUCT_PRICE_2",
                    "QUANTITY_2",
                    "SELLER_TEL_NUMBER_2",
                    "PURCHASE_SEQ_3",
                    "PRODUCT_NAME_3",
                    "PRODUCT_PRICE_3",
                    "QUANTITY_3",
                    "SELLER_TEL_NUMBER_3"
                ],
                "attribute-mappings": [
                    {
                        "target-attribute-name": "customer_id",
                        "attribute-type": "scalar",
                        "attribute-sub-type": "string",
                        "value": "${CUSTOMER_ID}"
                    },
                    {
                        "target-attribute-name": "purchase_id",
                        "attribute-type": "scalar",
                        "attribute-sub-type": "string",
                        "value": "${PURCHASE_ID}"
                    },
                    {
                        "target-attribute-name": "purchase_details",
                        "attribute-type": "document",
                        "attribute-sub-type": "dynamodb-map",
                        "value": {
                            "M": {
                                "CUSTOMER_NAME": {
                                    "S": "${CUSTOMER_NAME}"
                                },
                                "CUSTOMER_ADDRESS": {
                                    "S": "${CUSTOMER_ADDRESS}"
                                },
                                "PRODUCT_NAME_1": {
                                    "S": "${PRODUCT_NAME_1}"
                                },
                                "PRODUCT_PRICE_1": {
                                    "S": "${PRODUCT_PRICE_1}"
                                },
                                "QUANTITY_1": {
                                    "S": "${QUANTITY_1}"
                                },
                                "SELLER_TEL_NUMBER_1": {
                                    "S": "${SELLER_TEL_NUMBER_1}"
                                },
                                "PRODUCT_NAME_2": {
                                    "S": "${PRODUCT_NAME_2}"
                                },
                                "PRODUCT_PRICE_2": {
                                    "S": "${PRODUCT_PRICE_2}"
                                },
                                "QUANTITY_2": {
                                    "S": "${QUANTITY_2}"
                                },
                                "SELLER_TEL_NUMBER_2": {
                                    "S": "${SELLER_TEL_NUMBER_2}"
                                },
                                "PRODUCT_NAME_3": {
                                    "S": "${PRODUCT_NAME_3}"
                                },
                                "PRODUCT_PRICE_3": {
                                    "S": "${PRODUCT_PRICE_3}"
                                },
                                "QUANTITY_3": {
                                    "S": "${QUANTITY_3}"
                                },
                                "SELLER_TEL_NUMBER_3": {
                                    "S": "${SELLER_TEL_NUMBER_3}"
                                }
                            }
                        }
                    }
                ]
            }
        }
    ]
}
~~~

마지막으로 Manually later를 선택하고 Create task 버튼을 클릭합니다.  
![image 19](./images/19.png)

Task 생성이 완료되었으면 Task를 실행합니다.  
![image 20](./images/20.png)

[DynamoDB](https://ap-northeast-2.console.aws.amazon.com/dynamodbv2/home?region=ap-northeast-2#tables) 이동하여 purchase_t 를 클릭합니다.
![image 24](./images/24.png)

purchase_t 를 선택한 후 오른쪽 위에 "Explore table items"를 클릭합니다.  
![image 21](./images/21.png)

데이터 마이그레이션 된 것을 확인할 수 있습니다.
![image 22](./images/22.png)

하나의 아이템을 선택하면 해당 아이템의 상세 데이터를 볼 수 있습니다.
![image 23](./images/23.png)

### 4. Gatling을 이용하여 DynamoDB 기반의 구매 내역 조회 어플리케이션 성능을 확인합니다.
MobaXterm MSA_Server 세션으로 이동하여 어플리케이션을 실행합니다.
~~~
ec2-user@ip-10-100-1-101:/home/ec2-user> cd workshop04/msa
ec2-user@ip-10-100-1-101:/home/ec2-user/workshop04/msa> source bin/activate
(msa) ec2-user@ip-10-100-1-101:/home/ec2-user/workshop04/msa> flask run --host=0.0.0.0 --port=4000
 * Environment: production
   WARNING: This is a development server. Do not use it in a production deployment.
   Use a production WSGI server instead.
 * Debug mode: off
 * Running on all addresses.
   WARNING: This is a development server. Do not use it in a production deployment.
 * Running on http://10.100.1.101:4000/ (Press CTRL+C to quit)
~~~
Gatling으로 부하테스트를 수행합니다.
아래 명령어는 Windows Server에서 cmd에서 실행합니다.
~~~
C:\Users\Administrator> CD C:\gatling\bin
C:\gatling\bin> gatling.bat
GATLING_HOME is set to "C:\gatling"
JAVA = "java"
Choose a simulation number:
     [0] SeoulSummit.Workshop02_legacy
     [1] SeoulSummit.Workshop02_msa
     [2] SeoulSummit.Workshop04_legacy
     [3] SeoulSummit.Workshop04_msa
3(3번 시나리오를 수행합니다.)
Select run description (optional)
(엔터를 한번 더 입력합니다.)
Simulation SeoulSummit.Workshop04_msa started...

================================================================================
2022-03-06 20:51:06                                           5s elapsed
---- Requests ------------------------------------------------------------------
> Global                                                   (OK=219    KO=0     )
> selectPurchase                                           (OK=219    KO=0     )

---- Workshop04_msa ------------------------------------------------------------
          active: 2      / done: 219
================================================================================
~~~
부하가 종료된 후 성능 통계정보를 확인합니다. p95의 평균 응답시간은 56ms 입니다.
~~~
================================================================================
---- Global Information --------------------------------------------------------
> request count                                       9464 (OK=9464   KO=0     )
> min response time                                     20 (OK=20     KO=-     )
> max response time                                    103 (OK=103    KO=-     )
> mean response time                                    38 (OK=38     KO=-     )
> std deviation                                          9 (OK=9      KO=-     )
> response time 50th percentile                         36 (OK=36     KO=-     )
> response time 75th percentile                         41 (OK=41     KO=-     )
> response time 95th percentile                         56 (OK=56     KO=-     )
> response time 99th percentile                         68 (OK=68     KO=-     )
> mean requests/sec                                 52.287 (OK=52.287 KO=-     )
---- Response Time Distribution ------------------------------------------------
> t < 800 ms                                          9464 (100%)
> 800 ms < t < 1200 ms                                   0 (  0%)
> t > 1200 ms                                            0 (  0%)
> failed                                                 0 (  0%)
================================================================================

Reports generated in 0s.
Please open the following file: C:\gatling\results\workshop04-msa-20220306115100267\index.html
Press any key to continue . . .
~~~
위의 링크를 웹브라우저로 열어서 Web으로 제공되는 성능 보고서도 확인해 봅니다.
![image 25](./images/25.png)
![image 26](./images/26.png)
![image 27](./images/27.png)





