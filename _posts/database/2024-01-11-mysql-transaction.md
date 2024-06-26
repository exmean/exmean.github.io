---
layout: post
title: MySQL의 트랜잭션과 격리수준
category: MySQL
permalink: /mysql/1
---

우리에게 친숙한 **트랜잭션**은 작업의 완전성을 보장해줍니다! 쉽게 설명하자면 2개 이상의 작업이 있을 때 **단 1개의 작업이라도 오류로 인해 정상적으로 처리되지 못할 경우 정상 처리된 작업까지도 원 상태로 복구하는 기능**입니다. 예시로는 2개 이상의 작업을 이야기 했지만 실제로는 **단 한개의 작업이라도 트랜잭션을 통해 작업의 완전성을 보장**합니다. 현재 보편적으로 사용하는 데이터베이스 툴들은 AUTO COMMIT을 지원하고 있어서 트랜잭션을 인지하기가 어려운 것 같습니다. 하지만 데이터베이스를 다루는 개발자라면 내가 사용하는 데이터베이스는 어떻게 트랜잭션을 움직이고 있는지 알아야 상황에 맞는 대처를 할 수 있겠습니다.

# 트랜잭션 주의사항
먼저 트랜잭션을 사용할 때 주의할 사항들이 있습니다.

- 트랜잭션의 범위를 최소화해라.
- 불필요한 과정을 트랜잭션에 포함하지 말아라.
- 단순 조희는 트랜잭션의 범위에서 제거하자.
- 외부 작업 및 네트워크 관련 작업은 트랜잭션에서 제외해라. 외부적인 요인으로 인해 RDBMS 서버가 위험해질 수 있다.

이 부분을 읽으면서 Spring의 `@Transactional`과 연관이 있을지 궁금해졌습니다.   
해당 내용은 추후에 따로 정리해보겠습니다!

# 트랜잭션의 격리수준
격리수준이라는 단어를 검색해보면 **'트랜잭션 간의 고립 수준'**이라는 키워드가 가장 많이 보입니다. 따라서 트랜잭션의 격리수준이라 함은 지금 **내가 사용하는 트랜잭션이 다른 트랜잭션에 영향을 얼만큼 받는가**이고, 그렇다면 트랜잭션이 다른 트랜잭션에 의해 어떠한 영향을 받을 수 있겠구나 정도를 유추해볼 수 있습니다. 사용하는 유저가 많은 서비스를 생각해보면 수 많은 유저들이 동시에 데이터베이스를 사용할텐데 데이터베이스는 어떠한 기준으로 데이터를 반환하는 것일까, 그리고 반환된 데이터는 믿을 수 있는 데이터인가를 고민해보면 트랜잭션의 격리수준은 반드시 필요한 개념일 것 같습니다.

## READ UNCOMMITTED
**READ UNCOMMITTED** 격리수준에서는 어떤 트랜잭션에서 처리한 작업이 완료(COMMIT)되지 않았는데도 다른 트랜잭션에서 볼 수 있습니다. 이렇게 다른 트랜잭션이 완료되지 않아도 다른 트랜잭션에서 확인할 수 있는 현상을 **DIRTY READ**라고도 부릅니다. 
    
## READ COMMITTED
**READ COMMITTED** 격리수준은 오라클에서 기본으로 사용되는 격리 수준이고 **온라인 서비스에서 가장 많이 선택되는 격리 수준**입니다. 트랜잭션에서 레코드를 변경하는 경우 **<u>기존 레코드를 언두 영역으로 백업</u>**합니다. 여기서 말하는 언두 영역은 이전 레코드(데이터)를 저장하기 위한 임시 공간이라고 생각하면 되겠습니다. 만약 완료(COMMIT) 되지 않은 상태에서 **다른 트랜잭션이 접근하면 언두 영역에 있는 레코드를 조회**하게 됩니다. 이렇게 **READ COMMITTED** 격리수준에서는 언두 영역을 이용해 다른 트랜잭션에서 커밋하지 않은 결과를 조회할 수 없도록 합니다.   

> 하지만 READ COMMITTED 격리수준에서도 **REPEATABLE READ 정합성**에 어긋나는 결과가 나타날 수 있습니다. **<u>REPEATABLE READ 정합성이란 한 트랜잭션 내에서 같은 조회 쿼리가 실행될 경우 항상 같은 결과를 반환해야 한다는 규칙</u>**입니다.    
    
## REPEATABLE READ
**REPEATABLE READ** 격리수준은 MySQL의 InnoDB 스토리지 엔진에서 기본으로 사용되는 격리수준입니다. 바이너리 로그를 가진 MySQL에서는 최소 REPEATABLE READ 격리수준 이상을 사용해야 합니다. 바이너리 로그가 주로 복제나 복구 시에 사용하기 때문입니다. 당연히 복제나 복구 시에 사용되는 로그는 정확성이 중요하겠죠? REPEATABLE READ 격리 수준에서는 MVCC 변경 방식을 이용하기 때문에 REPEATABLE READ 부정합이 발생하지 않습니다. 여기서 말한 **MVCC 변경 방식**은 READ COMMITTED 격리수준에서 사용하는 **이전 레코드를 언두 영역에 백업하는 방식**을 말합니다. 그렇다면 **READ COMMITTED 격리수준에서는 발생하는 부정합 문제가 동일한 MVCC 변경 방식을 사용하는 REPEATABLE READ 격리수준에서는 왜 발생하지 않는 것일까요?** 그 이유는, REPEATABLE READ 격리수준에서는 언두 영역에 백업된 레코드의 여러 버전 중 몇번째 이전 버전까지 찾아 들어가야 하는지에 대한 정보를 가지고 있기 때문입니다.   

> 모든 InnoDB의 트랜잭션은 고유한 트랜잭션 번호(순차적으로 증가하는 값)을 가지는데, **<u>언두 영역에 백업된 모든 레코드에는 변경을 발생시킨 트랜잭션의 고유한 번호가 포함</u>**되어 있습니다. 언두 영역에 백업된 데이터들은 InnoDB 스토리지 엔진이 불필요하다고 판단하는 시점에 주기적으로 삭제합니다. 이렇게 REPEATABLE READ 격리수준에서는 MVCC 변경 방식을 이용하여 데이터의 정합성을 보장하기 위해 특정 트랜잭션 번호의 구간 내에서 백업된 언두 데이터를 보존합니다. 하나의 레코드가 많은 백업을 가질 수 있기 때문에 만약 장시간 트랜잭션을 종료하지 않으면 언두 영역이 백업된 데이터로 무한정 커질 수도 있습니다. **언두 영역에 백업된 레코드가 많아지면 MySQL의 전체적인 처리 성능이 떨어질 수도 있습니다.**   

하지만 REPEATABLE READ 격리수준에서도 부정합이 발생할 수 있는 케이스가 존재합니다.   

```sql
-- ID 500000까지의 데이터만 있다고 가정

SELECT * FROM tab WHERE ID >= 500000 FOR UPDATE;
-- 결과 1건
-- INSERT ID 5000001 COMMIT

SELECT * FROM tab WHERE ID >= 500000 FOR UPDATE;
-- 결과 2건
```

이처럼 다른 트랜잭션에서 수행한 변경 작업에 의해 레코드가 보였다 안보였다 하는 현상을 **<u>PHANTOM READ</u>**라고 합니다. **SELECT FOR UPDATE(혹은 SELECT LOCK IN SHARE MODE)** 쿼리는 레코드에 쓰기 잠금을 걸어야 하는데 언두 영역의 레코드에는 잠금을 걸 수 없기 때문에 이런 PHANTOM READ 현상이 발생합니다. **<u>하지만 갭 락과 넥스트 키 락으로 REPEATABLE READ 격리수준에서의 PHANTOM READ 현상은 해결 되었습니다.</u>**
    
## SERIALIZABLE
**SERIALIZABLE** 격리수준은 가장 단순한 격리 수준이면서 동시에 가장 엄격한 격리 수준입니다. 동시 처리 성능도 다른 트랜잭션 격리 수준보다 떨어집니다. 읽기 작업도 공유 잠금(읽기 잠금)을 획득해야만 하기 때문에 동시에 다른 트랜잭션은 읽기 트랜잭션이 종료되기 전까진 레코드를 변경하지 못하게 됩니다. 이 격리 수준에서는 일반적인 DBMS에서 일어나는 PHAMTOM READ 문제는 발생하지 않지만, 위에서 언급한 것처럼 이미 **InnoDB 스토리지 엔진에서는 갭 락과 넥스트 키 락 덕분에 REPEATABLE READ 격리 수준에서도 PHAMTOM READ 현상은 발생하지 않습니다.** 그래서 굳이 SERIALIZABLE 격리수준을 사용할 이유는 없습니다.   

---
이렇게 트랜잭션의 4가지 격리수준에 대해 정리해봤습니다. 모든 특징을 외울 순 없겠지만, 메인으로 사용하는 데이터베이스의 격리수준 정도는 숙지한다면 관련 이슈가 생길 때 빠르게 원인을 파악할 수 있을 것 같습니다.

# Reference
> Real MySQL 도서를 읽고 기록하였습니다.