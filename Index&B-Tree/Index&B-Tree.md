# TIL: 인덱스(Index)와 B-Tree 구조

> 날짜: 2026-05-21
> 태그: `#Database` `#Backend` `#Index` `#BTree` `#QueryOptimization`

---

## 📌 인덱스란?

**인덱스(Index)** 는 DB에서 **데이터를 빠르게 찾기 위한 자료구조**다.

책의 목차와 같은 개념 — 전체를 다 읽지 않고 원하는 페이지로 바로 이동할 수 있다.

### 인덱스가 없을 때 vs 있을 때

```
테이블에 100만 개의 행이 있을 때 WHERE id = 999999 조회

❌ 인덱스 없음 → Full Table Scan: 1번째 행부터 999999번째까지 전부 확인
✅ 인덱스 있음 → B-Tree 탐색: O(log N) 으로 바로 찾아감
```

---

## 📌 B-Tree 구조

MySQL InnoDB를 포함한 대부분의 DB가 인덱스 자료구조로 **B-Tree(Balanced Tree)** 를 사용한다.

### B-Tree의 특징

```
                  [ 30 | 70 ]              ← 루트 노드
                /      |      \
        [10|20]     [40|50|60]    [80|90]  ← 내부 노드
        /  |  \      ...           |  \
      [5] [15] [25]              [75] [85] ← 리프 노드 (실제 데이터)
```

- **균형 트리** — 모든 리프 노드의 깊이가 동일 → 항상 O(log N) 보장
- **범위 검색에 강함** — 리프 노드끼리 연결되어 있어 범위 스캔 효율적
- **정렬된 상태 유지** — ORDER BY, BETWEEN, >, < 쿼리에 유리

### B+Tree (실제 DB에서 사용)

실제로는 B-Tree의 변형인 **B+Tree**를 사용한다.

| 구분 | B-Tree | B+Tree |
|------|--------|--------|
| 데이터 저장 위치 | 모든 노드 | 리프 노드만 |
| 리프 노드 연결 | ❌ | ✅ (연결 리스트) |
| 범위 검색 | 느림 | 빠름 |
| 공간 효율 | 낮음 | 높음 |

---

## 📌 인덱스의 종류

### 클러스터형 인덱스 (Clustered Index)
- **테이블 데이터 자체가 인덱스 순서로 정렬**되어 저장됨
- MySQL InnoDB에서 **Primary Key = 클러스터형 인덱스**
- 테이블당 **1개만** 존재 가능
- PK로 조회 시 리프 노드 = 실제 데이터 → 매우 빠름

```sql
-- PK가 클러스터형 인덱스 → 가장 빠른 조회
SELECT * FROM users WHERE id = 1;
```

### 보조 인덱스 (Secondary Index)
- 클러스터형 인덱스 외에 추가로 생성하는 인덱스
- 리프 노드에 **실제 데이터 대신 PK 값**을 저장
- PK로 한 번 더 조회하는 **이중 탐색** 발생

```sql
-- email에 보조 인덱스가 있을 때
SELECT * FROM users WHERE email = 'test@test.com';
-- 1) 보조 인덱스에서 email로 PK(id) 찾기
-- 2) 클러스터형 인덱스에서 PK로 실제 데이터 찾기
```

---

## 📌 인덱스 생성과 확인

```sql
-- 인덱스 생성
CREATE INDEX idx_email ON users(email);

-- 복합 인덱스 (여러 컬럼)
CREATE INDEX idx_name_age ON users(name, age);

-- 인덱스 확인
SHOW INDEX FROM users;

-- 인덱스 삭제
DROP INDEX idx_email ON users;
```

---

## 📌 EXPLAIN — 실행 계획 확인

인덱스가 실제로 사용되는지 확인하려면 `EXPLAIN`을 사용한다.

```sql
EXPLAIN SELECT * FROM users WHERE email = 'test@test.com';
```

```
+----+-------------+-------+------+---------------+-----------+------+-------+
| id | select_type | table | type | possible_keys | key       | rows | Extra |
+----+-------------+-------+------+---------------+-----------+------+-------+
|  1 | SIMPLE      | users | ref  | idx_email     | idx_email |    1 |       |
+----+-------------+-------+------+---------------+-----------+------+-------+
```

| type 값 | 의미 | 성능 |
|---------|------|------|
| `const` | PK/Unique 1건 조회 | 🟢 최상 |
| `ref` | 인덱스로 조회 | 🟢 좋음 |
| `range` | 인덱스 범위 스캔 | 🟡 보통 |
| `index` | 인덱스 풀 스캔 | 🟠 나쁨 |
| `ALL` | 풀 테이블 스캔 | 🔴 최악 |

---

## 📌 인덱스가 안 타는 경우 (주의!)

```sql
-- ❌ 컬럼에 함수 적용 → 인덱스 무효화
SELECT * FROM users WHERE YEAR(created_at) = 2024;
-- ✅ 범위 조건으로 변경
SELECT * FROM users WHERE created_at BETWEEN '2024-01-01' AND '2024-12-31';

-- ❌ LIKE 앞에 와일드카드 → 인덱스 무효화
SELECT * FROM users WHERE name LIKE '%김';
-- ✅ 뒤쪽 와일드카드는 OK
SELECT * FROM users WHERE name LIKE '김%';

-- ❌ 묵시적 형변환 → 인덱스 무효화
SELECT * FROM users WHERE phone = 01012345678; -- phone이 VARCHAR인데 숫자로 비교
-- ✅ 타입 맞춰서 조회
SELECT * FROM users WHERE phone = '01012345678';

-- ❌ 복합 인덱스 (name, age) 에서 두 번째 컬럼만 조회 → 인덱스 무효화
SELECT * FROM users WHERE age = 25;
-- ✅ 첫 번째 컬럼부터 사용해야 함 (선두 컬럼 원칙)
SELECT * FROM users WHERE name = '김철수' AND age = 25;
```

---

## 📌 인덱스의 단점 — 무조건 많이 걸면 안 된다!

| 장점 | 단점 |
|------|------|
| SELECT 속도 향상 | INSERT/UPDATE/DELETE 속도 저하 |
| 정렬/범위 검색 빠름 | 추가 저장 공간 필요 |
| | 인덱스 관리 오버헤드 |

**인덱스를 걸면 좋은 컬럼:**
- WHERE 조건에 자주 사용되는 컬럼
- JOIN의 ON 조건에 사용되는 컬럼
- 카디널리티(중복도)가 낮은 컬럼 (ex. 성별 ❌, 이메일 ✅)

---

## 💭 오늘 느낀 점

- 트랜잭션(ACID) → 락(동시성 제어) → 인덱스(성능 최적화) 흐름으로 DB를 입체적으로 이해하게 됐다
- EXPLAIN은 쿼리 짤 때 습관적으로 확인해야겠다 — `ALL`이 뜨면 무조건 의심
- 복합 인덱스의 선두 컬럼 원칙은 실무에서 진짜 자주 실수할 것 같다

---

## 📚 참고

- [MySQL 공식 문서 - How MySQL Uses Indexes](https://dev.mysql.com/doc/refman/8.0/en/mysql-indexes.html)
- [MySQL 공식 문서 - EXPLAIN Output Format](https://dev.mysql.com/doc/refman/8.0/en/explain-output.html)
