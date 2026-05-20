# TIL: 락(Lock)과 데드락(Deadlock)

> 날짜: 2026-05-20
> 태그: `#Database` `#Backend` `#Lock` `#Deadlock` `#Transaction`

---

## 📌 왜 락(Lock)이 필요한가?

어제 배운 **ACID의 격리성(Isolation)** 은 내부적으로 **락(Lock)** 으로 구현된다.

여러 트랜잭션이 동시에 같은 데이터에 접근하면?

```
트랜잭션 A: 잔액 조회 (10만원 확인)
트랜잭션 B: 동시에 출금 (-10만원)
트랜잭션 A: 10만원인 줄 알고 송금 진행 → 잔액 -10만원 💥
```

→ 이런 **동시성 문제**를 막기 위해 락을 사용한다.

---

## 📌 락의 종류

### 🔒 공유 락 (Shared Lock, S-Lock)

> **"읽는 동안 다른 사람도 읽을 수 있지만, 쓰지는 못한다"**

- `SELECT ... LOCK IN SHARE MODE`
- 여러 트랜잭션이 동시에 획득 가능
- 공유 락이 걸린 상태에서 쓰기(배타 락) 불가

```sql
-- 트랜잭션 A
BEGIN;
SELECT balance FROM accounts WHERE id = 1 LOCK IN SHARE MODE;
-- 다른 트랜잭션도 읽기 가능, 쓰기는 대기
```

---

### 🔒 배타 락 (Exclusive Lock, X-Lock)

> **"내가 쓰는 동안 아무도 읽거나 쓸 수 없다"**

- `SELECT ... FOR UPDATE` 또는 `UPDATE`, `DELETE` 시 자동 획득
- 오직 하나의 트랜잭션만 획득 가능
- 다른 모든 락(공유/배타) 대기

```sql
-- 트랜잭션 A
BEGIN;
SELECT balance FROM accounts WHERE id = 1 FOR UPDATE;
-- 다른 트랜잭션은 읽기/쓰기 모두 대기
UPDATE accounts SET balance = balance - 10000 WHERE id = 1;
COMMIT;
```

---

### 락 호환성 표

|  | 공유 락(S) | 배타 락(X) |
|--|:----------:|:----------:|
| **공유 락(S)** | ✅ 호환 | ❌ 대기 |
| **배타 락(X)** | ❌ 대기 | ❌ 대기 |

---

## 📌 락의 범위

### 행 락 (Row Lock)
- 특정 행(row)에만 락을 건다
- MySQL InnoDB 기본 방식
- 동시성이 높아 성능이 좋다

### 테이블 락 (Table Lock)
- 테이블 전체에 락을 건다
- `ALTER TABLE`, MyISAM 엔진 등에서 사용
- 동시성이 낮아 성능이 떨어진다

### 갭 락 (Gap Lock)
- 존재하지 않는 **범위(gap)** 에 락을 건다
- REPEATABLE READ 격리 수준에서 Phantom Read 방지용
- InnoDB 특유의 락 방식

```sql
-- id가 5~10 사이의 행이 없어도
-- 그 범위에 INSERT 못 하도록 갭 락
SELECT * FROM accounts WHERE id BETWEEN 5 AND 10 FOR UPDATE;
```

---

## 📌 데드락 (Deadlock)

> **"두 트랜잭션이 서로 상대방의 락이 풀리길 기다리며 무한 대기하는 상태"**

### 💡 데드락 발생 시나리오

```
트랜잭션 A: accounts(id=1) 락 획득 → accounts(id=2) 락 요청 (대기)
트랜잭션 B: accounts(id=2) 락 획득 → accounts(id=1) 락 요청 (대기)

A는 B를 기다리고, B는 A를 기다린다 → 영원히 진행 불가 💀
```

```sql
-- 트랜잭션 A
BEGIN;
UPDATE accounts SET balance = balance - 10000 WHERE id = 1; -- id=1 락
UPDATE accounts SET balance = balance + 10000 WHERE id = 2; -- id=2 대기 중...

-- 트랜잭션 B (동시 실행)
BEGIN;
UPDATE accounts SET balance = balance - 10000 WHERE id = 2; -- id=2 락
UPDATE accounts SET balance = balance + 10000 WHERE id = 1; -- id=1 대기 중... → 데드락!
```

---

### 🛠 DB의 데드락 해결 방법

DB는 데드락을 **자동으로 감지**하고, 둘 중 하나를 **강제 롤백(victim)** 시킨다.

```
MySQL 에러 메시지:
ERROR 1213 (40001): Deadlock found when trying to get lock;
try restarting transaction
```

→ 애플리케이션에서 이 예외를 잡아 **재시도 로직** 을 구현해야 한다.

---

### ✅ 데드락 예방법

**1. 항상 같은 순서로 락을 획득하라**

```java
// ❌ BAD: A→B, B→A 순서가 달라 데드락 위험
void transferAtoB() { lock(1); lock(2); }
void transferBtoA() { lock(2); lock(1); }

// ✅ GOOD: 항상 id 오름차순으로 락 획득
void transfer(int from, int to) {
    int first = Math.min(from, to);
    int second = Math.max(from, to);
    lock(first); lock(second);
}
```

**2. 트랜잭션을 짧게 유지하라**
- 락을 오래 잡을수록 데드락 확률 증가
- 필요한 작업만 트랜잭션 안에 넣기

**3. 필요한 락을 한 번에 획득하라**
- 나눠서 획득하면 그 사이에 다른 트랜잭션이 끼어들 수 있음

**4. 락 타임아웃 설정**

```sql
-- MySQL: 락 대기 타임아웃 설정 (초)
SET innodb_lock_wait_timeout = 5;
```

---

## 📌 JPA에서 락 사용하기

```java
// 비관적 락 (Pessimistic Lock) - DB 락 직접 사용
@Lock(LockModeType.PESSIMISTIC_WRITE)
Optional<Account> findById(Long id);

// 낙관적 락 (Optimistic Lock) - 버전 번호로 충돌 감지
@Version
private Long version; // 엔티티에 버전 필드 추가
```

| 구분 | 비관적 락 | 낙관적 락 |
|------|----------|----------|
| 방식 | DB 락 사용 | 버전 번호 비교 |
| 충돌 가정 | 충돌이 자주 난다 | 충돌이 드물다 |
| 성능 | 낮음 (락 대기) | 높음 (락 없음) |
| 실패 시 | 대기 후 진행 | 예외 발생 → 재시도 |
| 적합한 상황 | 금융, 재고 차감 | 게시글 수정, 좋아요 |

---

## 💭 오늘 느낀 점

- 어제 배운 격리성(Isolation)이 락으로 구현된다는 걸 알고나니 ACID가 더 입체적으로 느껴졌다
- 데드락은 "내가 코드 잘못 짜서 생기는 버그"가 아니라 동시성 환경에서 구조적으로 발생할 수 있는 문제라는 걸 인식하는 게 중요한 것 같다
- JPA 낙관적 락 vs 비관적 락 선택 기준이 명확해졌다

---

## 📚 참고

- [MySQL 공식 문서 - InnoDB Locking](https://dev.mysql.com/doc/refman/8.0/en/innodb-locking.html)
- [MySQL 공식 문서 - Deadlocks](https://dev.mysql.com/doc/refman/8.0/en/innodb-deadlocks.html)
