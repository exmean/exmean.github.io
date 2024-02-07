---
layout: post
title: MySQL LOCK(잠금)의 개념과 종류
category: Database
permalink: /mysql/1
---

MySQL는 **동시성을 제어하기 위한 기능인 LOCK(잠금)**을 제공합니다. 동시성이란 외부에서 동시에 요청된 작업을 의미합니다. LOCK은 동시성에서 생기는 문제를 해결하기 위해 여러 커넥션에서 동시에 동일한 자원(레코드나 테이블을 의미)을 접근 및 변경을 요청할 경우 하나의 커넥션에서만 변경할 수 있게 해주는 역할을 합니다. **주로 INSERT나 UPDATE 같은 레코드를 변경하는 과정에서 일어납니다.** LOCK에 관한 이슈는 서비스 운영 중에 심심치 않게 마주칩니다. 저도 최근에 DEAD LOCK 이슈를 만난 적이 있는데요, 그렇다면 이런 이슈는 왜 발생하고 LOCK은 어떻게 움직이는지 자세히 살펴보도록 하겠습니다.

# LOCK 종류

## GLOBAL LOCK
```SQL
FLUSH TABLES WITH READ LOCK;    -- GLOBAL LOCK 획득
UNLOCK TABLES;                  -- LOCK 해제
```
    
**GLOBAL LOCK**은 MySQL에서 제공하는 잠금 중 가장 범위가 큰 잠금입니다. MySQL 내에 존재하는 <u>데이터베이스와 테이블 전체에 잠금</u>이 설정되기 때문에 **웹 서비스용으로 사용되는 MySQL 서버에서는 가급적 사용하지 않는 것이 좋습니다.** MySQL 8.0 버전부터는 **InnoDB**가 기본 스토리지 엔진으로 채택되면서 조금 더 가벼운 **BACKUP LOCK**이 도입되었습니다.   

### BACKUP LOCK
```SQL
LOCK INSTANCE FOR BACKUP;   -- BACKUP LOCK 획득
UNLOCK INSTANCE;            -- LOCK 해제
```
**BACKUP LOCK**에서는 **테이블의 데이터 변경은 허용**합니다. 즉, 운영중인 서비스의 중단없이 백업을 진행해야 하는 상황이라면 고려해 볼 수 있는 사항입니다. `XtraBackup`이나 `Enterprise Backup`과 같은 백업 툴을 이용해서 백업을 진행할 때도 BACKUP LOCK을 활용할 수 있습니다. 관련하여 [끔찍한 일화(?)](https://techblog.woowahan.com/2576)를 함께 공유합니다.    
    
## TABLE LOCK    
TABLE LOCK은 테이블 단위로 설정되는 잠금입니다. **명시적 또는 묵시적으로 특정 테이블의 LOCK을 획득할 수 있다.** MyISAM과 MEMORY 테이블에서 데이터를 변경하는 쿼리를 실행하면 묵시적으로 TABLE LOCK이 발생하고, 데이터가 변경이 되면 LOCK을 해제합니다. InnoDB 테이블의 경우 스토리지 엔진 차원에서 **레코드 잠금을 제공**하기 때문에 데이터를 변경하는 단순한 쿼리로 인해 묵시적인 테이블 락이 설정되지는 않는다. 정확히는 대부분의 데이터 변경(DML) 쿼리에서는 무시되고 스키마를 변경하는 쿼리(DDL)의 경우에만 영향을 미친다.   
    
## NAMED LOCK
테이블이나 레코드, 데이터베이스 객체가 아니라 특정 문자열에 대한 락이다. MySQL 8.0 버전부터는 네임드 락을 중첩해서 사용할 수 있게 되었고, 획득한 네임드 락을 한 번에 모두 해제하는 기능도 추가됐다. 어떤 것에 이용될 지는 아직 감이 잡히진 않는다.   
    

## METADATA LOCK    
데이터베이스 객체(대표적으로 테이블이나 뷰 등)의 이름이나 구조를 변경하는 경우에 획득하는 락이다. 명시적으로 획득하는 것이 아니고 변경하는 쿼리를 실행할 때 자동으로 획득된다.   
    
---

# MySQL InnoDB 스토리지 엔진의 잠금

MySQL InnoDB 스토리지 엔진은 엔진 내부에서 **레코드 기반의 잠금 방식을 탑재**하고 있다. 레코드 기반 잠금 방식 때문에 뛰어난 **동시성 처리를 제공**할 수 있다. 그리고 InnoDB 트랜잭션과 잠금에 대한 각종 모니터링 방법을 제공한다.

## RECORD LOCK
레코드 자체만을 잠근다. 다른 상용 DBMS의 레코드 락과의 차이점은 **인덱스의 레코드를 잠근다는 점이다.**  인덱스가 하나도 없는 테이블이더라도 내부적으로 자동 생성된 클러스터 인덱스를 이용해 잠금을 설정한다.   
    
## GAP LOCK
레코드 자체가 아니라 **레코드와 바로 인접한 레코드 사이의 간격을 잠그는 것을 의미한다. 레코드와 레코드 사이의 간격에 새로운 레코드가 생성되는 것을 제어**한다. 갭 락을 그 자체로 사용하기 보다는 넥스트 키 락의 일부로 자주 사용된다.   
    
## NEXT KEY LOCK
**레코드 락과 갭 락을 합쳐 놓은 형태의 잠금**을 말한다. 바이너리 로그에 기록되는 쿼리가 레플리카 서버에서 실행될 때 소스 서버에서 만들어 낸 결과와 동일한 결과를 만들어내도록 보장하는 것이 주목적이다(이 문장이 이해가 잘 되지 않는다). 그런데 의외로 **넥스트 키 락과 갭 락으로 인해 데드락이 발생하거나 다른 트랜잭션을 기다리게 만드는 일이 자주 발생**한다.   
    
## AUTO INCREMENT LOCK
AUTO_INCREMENT 락이라고 부른다. INSERT와 REPLACE 쿼리 문장과 같이 새로운 레코드를 저장하는 쿼리에서만 필요하다. 트랜잭션과 관계없이 AUTO_INCREMENT 값을 가져오는 순간만 락이 걸렸다가 즉시 해제된다. 테이블에 단 하나만 존재하고 명시적으로 획득하거나 해제하는 방법은 없다.   
    

## 인덱스와 잠금

- MySQL 서버가 기본 엔진으로 채택하고 있는 InnoDB는 인덱스 기반 잠금을 제공한다. **만약 인덱스의 설계가 잘못되어 있다면 수 많은 레코드가 동시에 잠금되는 현상이 일어날 수 있다.**   

---
마무리

# Reference
> Real MySQL 도서를 읽고 기록하였습니다.