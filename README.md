# mysql-replication

차례
### 1. GTID란?
### 2. GTID를 활용한 Replica 승격 시나리오 (수동 Failover)



# 1. GTID (Global Transaction Identifier)

## 1. GTID란?

GTID는 각 트랜잭션에 전역적으로 유일한 ID를 부여하는 복제 방식이다.

기존처럼 **Binary Log의 파일 위치(File/Position)** 를 기준으로 복제를 추적하는 것이 아니라,
**트랜잭션 자체를 기준으로 복제를 추적**한다.

즉, GTID는 "로그 위치"가 아니라 "트랜잭션 단위"로 복제를 관리한다.

---

## 2. File/Position 방식과의 차이

### 2.1 기존 방식 (File/Position 기반)

Replica는 다음과 같이 복제 위치를 기억한다.

```
mysql-bin.000003, position 154
```

### 문제점

* Failover 시 로그 좌표를 직접 계산해야 함
* 중간에 로그가 삭제되면 복구가 어려움
* Primary 승격 시 수동 조정 필요
* Replica가 여러 개인 경우 관리 복잡

---

### 2.2 GTID 방식

Replica는 다음처럼 기억한다.

```
이 GTID까지 실행 완료
```

Replica는 Source 서버에 이렇게 요청한다.

```
내가 가진 GTID를 제외하고 보내줘
```

### 장점

* 자동으로 이어서 복제 가능
* File/Position을 맞출 필요 없음
* 복제 재구성 및 재연결이 단순함

---

## 3. Failover 상황에서의 차이

예시 상황:

> Binary Log에 기록은 되었지만 Replica로 전송되기 전에 서버 장애 발생

### File/Position 방식

* 어떤 Replica가 가장 최신인지 사람이 판단해야 함
* 로그 좌표 비교 필요
* 운영 리스크 높음

### GTID 방식

* 각 Replica가 보유한 GTID 집합 비교
* 가장 많은 GTID를 가진 Replica가 최신
* 자동 승격 도구 사용 가능 (예: Orchestrator, MHA)

---

## 4. GTID 핵심 개념

### 4.1 gtid_executed

해당 서버가 실행 완료한 트랜잭션 집합


### 4.2 gtid_purged

삭제된 Binary Log에 포함된 GTID 집합


### 4.3 auto_position = 1

File/Position 기반이 아닌
GTID 기반 복제를 사용하도록 설정하는 옵션


## 5. GTID 구조

```
server_uuid:transaction_id
```

예시:

```
3E11FA47-71CA-11E1-9E33-C80AA9429562:23
```

의미:

* 어떤 서버에서 발생한 트랜잭션인지 식별 가능
* 해당 서버에서 몇 번째 트랜잭션인지 식별 가능

## 6. GTID 설정 방법

* Primary / Replica 서버 모두 설정 필요
* 각 서버는 고유한 server-id 사용
* Binary Log 활성화 필수

my.cnf 설정

Primary / Replica 공통 설정:

```
[mysqld]
server-id = 1                # Replica는 2, 3 등 고유값
log-bin = mysql-bin
binlog_format = ROW

gtid_mode = ON
enforce_gtid_consistency = ON

log_slave_updates = ON       # Replica에서 반드시 필요
```

설정 후 MySQL 재시작
```
sudo systemctl restart mysql
```



# 2. GTID를 활용한 Replica 승격 시나리오 (수동 Failover)

상황:
Primary 서버 장애 발생
여러 Replica 중 하나를 새 Primary로 승격해야 함 - 판단 기준은?

### Failover 전체 흐름
Source 장애 발생

각 Replica의 GTID 비교

최신 Replica 선정

해당 Replica에서 STOP REPLICA + RESET

read_only 해제 → Source(Primary) 승격

나머지 Replica를 새 Primary에 재연결


### Step 1: 복제 구성 (Source 1개 + Replica 2개)

GTID 켜고 구성
<img width="1474" height="1740" alt="image" src="https://github.com/user-attachments/assets/d1770f9f-6501-4d4a-b770-b1e0aae827ca" />

### Step 2: 한 Replica만 일시 정지시키기

Replica B 복제 중지
```
STOP Replica;
```
* 이 상태에서는 Source에서 발생하는 새로운 트랜잭션을 받지 못한다.

### Step 3: Source에서 트랜잭션 여러 개 실행
```
INSERT INTO products VALUES (1);
INSERT INTO products VALUES (2);
INSERT INTO products VALUES (3);
```

<img width="1480" height="1738" alt="image" src="https://github.com/user-attachments/assets/2a3930ff-6499-48d9-aeb6-8bd2ce3a1b06" />

### Step 4: Source 강제 종료 (Source 서버 장애 발생)
```
docker kill mysql-primary
```

### Step 5: 승격시킬 최신 Replica 선정

각 Replica에서 실행:
```
SHOW MASTER STATUS;
SHOW GLOBAL VARIABLES LIKE 'gtid_executed';
```

<img width="1478" height="1740" alt="image" src="https://github.com/user-attachments/assets/0b729354-e648-4bb5-acee-2b282ff46324" />

판단 기준

가장 많은 GTID를 가진 Replica가 최신
GTID 집합 비교 후 최신 서버 선정

### Step 6: 복제 중지

선정된 Replica에서:
```
STOP REPLICA;
RESET REPLICA ALL;

SET GLOBAL read_only = OFF;
```
<img width="1478" height="574" alt="image" src="https://github.com/user-attachments/assets/b7e9e6a4-bdf0-483e-ac9d-a91622d566a8" />


→ 기존 Primary와의 복제 연결 제거
이제 이 서버가 새로운 Source 역할 수행

### Step 7: 다른 Replica를 새 Primary에 연결

기존 다른 Replica에서
```
STOP REPLICA;

CHANGE REPLICATION SOURCE TO
  SOURCE_HOST='new_primary_ip',
  SOURCE_USER='replica_user',
  SOURCE_PASSWORD='password',
  SOURCE_AUTO_POSITION=1;

START REPLICA;

```

💥 에러 발생
```
Got fatal error 1236:
Cannot replicate because the source purged required binary logs.
```

마스터가 필요한 binlog를 이미 삭제해버렸다는 뜻

> Source를 다시 살리고 다른 Replica를 연결 ..?
