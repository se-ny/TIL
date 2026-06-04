# TIL: HTTP 메서드 & 상태코드

> 날짜: 2026-06-04  
> 태그: `#Network` `#HTTP` `#Backend` `#REST`

---

## 📌 HTTP 메서드

HTTP 메서드는 **클라이언트가 서버에게 어떤 동작을 요청하는지** 나타낸다.

---

### 주요 메서드 정리

| 메서드 | 역할 | 멱등성 | 안전성 |
|--------|------|:------:|:------:|
| `GET` | 리소스 조회 | ✅ | ✅ |
| `POST` | 리소스 생성 | ❌ | ❌ |
| `PUT` | 리소스 전체 수정 | ✅ | ❌ |
| `PATCH` | 리소스 부분 수정 | ❌ | ❌ |
| `DELETE` | 리소스 삭제 | ✅ | ❌ |

> **멱등성(Idempotent):** 같은 요청을 여러 번 해도 결과가 동일  
> **안전성(Safe):** 서버 상태를 변경하지 않음

---

### 💡 PUT vs PATCH

```json
// 현재 유저 데이터
{ "name": "김철수", "email": "kim@test.com", "age": 25 }

// PUT 요청 — 전체 교체 (빠진 필드는 null 처리)
PUT /users/1
{ "name": "김영희" }
→ 결과: { "name": "김영희", "email": null, "age": null } ❌ 위험!

// PATCH 요청 — 부분 수정 (보낸 필드만 변경)
PATCH /users/1
{ "name": "김영희" }
→ 결과: { "name": "김영희", "email": "kim@test.com", "age": 25 } ✅
```

---

### 💡 POST vs PUT

```
POST /users       → 새 유저 생성 (서버가 ID 결정)
PUT  /users/1     → id=1 유저를 지정해서 수정 또는 생성 (클라이언트가 URI 결정)
```

---

## 📌 HTTP 상태코드

상태코드는 **서버가 요청을 어떻게 처리했는지** 클라이언트에게 알려주는 3자리 숫자다.

---

### 2xx — 성공

| 코드 | 이름 | 언제 쓰나 |
|------|------|----------|
| `200` | OK | 일반적인 성공 (GET, PUT, PATCH) |
| `201` | Created | 리소스 생성 성공 (POST) |
| `204` | No Content | 성공했지만 응답 바디 없음 (DELETE) |

```http
HTTP/1.1 201 Created
Location: /users/42
```

---

### 3xx — 리다이렉션

| 코드 | 이름 | 언제 쓰나 |
|------|------|----------|
| `301` | Moved Permanently | URL 영구 변경 |
| `302` | Found | URL 임시 변경 |
| `304` | Not Modified | 캐시된 데이터 그대로 사용 가능 |

---

### 4xx — 클라이언트 오류

| 코드 | 이름 | 언제 쓰나 |
|------|------|----------|
| `400` | Bad Request | 요청 형식 잘못됨 (유효성 검사 실패) |
| `401` | Unauthorized | 인증 안 됨 (로그인 필요) |
| `403` | Forbidden | 인증은 됐지만 권한 없음 |
| `404` | Not Found | 리소스 없음 |
| `405` | Method Not Allowed | 허용되지 않은 HTTP 메서드 |
| `409` | Conflict | 리소스 충돌 (이미 존재하는 이메일 등) |
| `422` | Unprocessable Entity | 요청 형식은 맞지만 내용이 처리 불가 |
| `429` | Too Many Requests | 요청 횟수 초과 (Rate Limiting) |

> ⚠️ **401 vs 403 헷갈리지 말기!**
> - `401` — "넌 누구야?" (비로그인 상태)
> - `403` — "넌 누군지 알지만 안 돼" (권한 없음)

---

### 5xx — 서버 오류

| 코드 | 이름 | 언제 쓰나 |
|------|------|----------|
| `500` | Internal Server Error | 서버 내부 오류 (예상치 못한 예외) |
| `502` | Bad Gateway | 게이트웨이/프록시 오류 |
| `503` | Service Unavailable | 서버 과부하 또는 점검 중 |
| `504` | Gateway Timeout | 게이트웨이 응답 시간 초과 |

---

## 📌 REST API 설계에서 올바른 메서드 & 상태코드 사용

```http
# 유저 목록 조회
GET /users
→ 200 OK

# 유저 단건 조회
GET /users/1
→ 200 OK / 404 Not Found

# 유저 생성
POST /users
→ 201 Created / 400 Bad Request / 409 Conflict

# 유저 수정
PATCH /users/1
→ 200 OK / 404 Not Found

# 유저 삭제
DELETE /users/1
→ 204 No Content / 404 Not Found

# 로그인
POST /auth/login
→ 200 OK / 401 Unauthorized

# 권한 없는 리소스 접근
GET /admin/users
→ 403 Forbidden
```

---

## 📌 Spring Boot에서 상태코드 반환하기

```java
// 방법 1: ResponseEntity 사용 (권장)
@PostMapping("/users")
public ResponseEntity<UserResponse> createUser(@RequestBody UserRequest request) {
    UserResponse response = userService.create(request);
    return ResponseEntity.status(HttpStatus.CREATED).body(response); // 201
}

@DeleteMapping("/users/{id}")
public ResponseEntity<Void> deleteUser(@PathVariable Long id) {
    userService.delete(id);
    return ResponseEntity.noContent().build(); // 204
}

// 방법 2: @ResponseStatus 어노테이션
@ResponseStatus(HttpStatus.CREATED)
@PostMapping("/users")
public UserResponse createUser(@RequestBody UserRequest request) {
    return userService.create(request);
}

// 예외 처리 — 상태코드 매핑
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(EntityNotFoundException.class)
    public ResponseEntity<ErrorResponse> handleNotFound(EntityNotFoundException e) {
        return ResponseEntity.status(HttpStatus.NOT_FOUND)
                             .body(new ErrorResponse(e.getMessage())); // 404
    }

    @ExceptionHandler(DuplicateEmailException.class)
    public ResponseEntity<ErrorResponse> handleConflict(DuplicateEmailException e) {
        return ResponseEntity.status(HttpStatus.CONFLICT)
                             .body(new ErrorResponse(e.getMessage())); // 409
    }
}
```

---

## 💭 오늘 느낀 점

- `PUT` 은 전체 교체라 실무에서 잘못 쓰면 데이터가 날아갈 수 있다 — `PATCH`를 더 자주 쓰게 될 것 같다
- `401 vs 403` 은 개념은 알았지만 실제로 구분해서 쓴 적이 없었는데, 앞으로 API 설계할 때 제대로 구분해야겠다
- `@RestControllerAdvice`로 예외별 상태코드를 한 곳에서 관리하는 패턴이 깔끔하다

---

## 📚 참고

- [MDN - HTTP 요청 메서드](https://developer.mozilla.org/ko/docs/Web/HTTP/Methods)
- [MDN - HTTP 상태 코드](https://developer.mozilla.org/ko/docs/Web/HTTP/Status)
- [RFC 7231 - HTTP/1.1 Semantics](https://datatracker.ietf.org/doc/html/rfc7231)
