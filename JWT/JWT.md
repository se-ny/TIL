# TIL: JWT(JSON Web Token) 구조 & 동작 원리

> 날짜: 2026-05-22  
> 태그: `#Backend` `#Security` `#JWT` `#Authentication`

---

## 📌 JWT란?

**JWT(JSON Web Token)** 는 인증 정보를 **JSON 형태로 담아 서명한 토큰**이다.

서버가 별도로 상태를 저장하지 않아도 토큰 자체만으로 사용자를 식별할 수 있다 → **Stateless 인증**

---

## 📌 JWT 구조

JWT는 `.`으로 구분된 **3개의 파트**로 이루어진다.

```
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6Iuq5mOustOyWtCIsImlhdCI6MTUxNjIzOTAyMn0.SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c
         Header (헤더)                 .                   Payload (페이로드)                         .        Signature (서명)
```

---

### 1️⃣ Header (헤더)

```json
{
  "alg": "HS256",   // 서명 알고리즘 (HMAC SHA-256)
  "typ": "JWT"      // 토큰 타입
}
```
→ Base64Url로 인코딩

---

### 2️⃣ Payload (페이로드)

실제 담고 싶은 데이터(클레임, Claim)를 담는다.

```json
{
  "sub": "1234567890",     // 표준 클레임: Subject (사용자 식별자)
  "iat": 1516239022,       // 표준 클레임: Issued At (발급 시간)
  "exp": 1516242622,       // 표준 클레임: Expiration (만료 시간)
  "name": "김철수",         // 커스텀 클레임
  "role": "ADMIN"          // 커스텀 클레임
}
```

→ Base64Url로 인코딩

> ⚠️ **Payload는 암호화가 아닌 인코딩** — 누구나 디코딩해서 볼 수 있다!
> 비밀번호, 카드번호 등 민감한 정보는 절대 넣으면 안 된다.

---

### 3️⃣ Signature (서명)

```
HMACSHA256(
  base64UrlEncode(header) + "." + base64UrlEncode(payload),
  secret_key   ← 서버만 알고 있는 비밀키
)
```

→ 토큰이 **위변조되지 않았음**을 보장한다.
→ secret_key 없이는 유효한 서명을 만들 수 없다.

---

## 📌 JWT 인증 동작 흐름

```
[클라이언트]                          [서버]

1. 로그인 요청 (id/pw)  ────────────→  id/pw 검증
                        ←────────────  JWT 발급 (Access Token + Refresh Token)

2. API 요청
   Header: Authorization: Bearer <JWT>  ────→  서명 검증 (secret_key로)
                                        ←────  응답 (DB 조회 없이 토큰만으로 인증!)

3. Access Token 만료 시
   Refresh Token 전송  ────────────→  Refresh Token 검증
                       ←────────────  새 Access Token 발급
```

---

## 📌 Access Token vs Refresh Token

| 구분 | Access Token | Refresh Token |
|------|-------------|---------------|
| 역할 | API 요청 인증 | Access Token 재발급 |
| 유효기간 | 짧게 (15분 ~ 1시간) | 길게 (7일 ~ 30일) |
| 저장 위치 | 메모리 or 로컬스토리지 | HttpOnly 쿠키 (권장) |
| 노출 위험 | 높음 (매 요청마다 전송) | 낮음 (재발급 시에만 전송) |

**Access Token을 짧게 유지하는 이유:**
탈취당해도 금방 만료되어 피해를 최소화하기 위함

---

## 📌 JWT의 장단점

### ✅ 장점
- **Stateless** — 서버가 토큰 상태를 저장하지 않아도 됨 → 서버 확장(Scale-out)에 유리
- **Self-contained** — 토큰 자체에 사용자 정보 포함 → DB 조회 없이 인증 가능
- **다양한 플랫폼 지원** — 모바일, 웹 모두 동일하게 사용 가능

### ❌ 단점
- **토큰 탈취 시 대처 어려움** — 서버에서 강제 무효화 불가 (만료 전까지 유효)
- **Payload 노출** — 민감한 정보 담으면 안 됨
- **토큰 크기** — 세션 ID보다 훨씬 큰 크기 → 매 요청마다 네트워크 오버헤드

---

## 📌 Spring Boot에서 JWT 사용하기

```java
// JWT 생성
public String generateToken(UserDetails userDetails) {
    return Jwts.builder()
        .setSubject(userDetails.getUsername())
        .setIssuedAt(new Date())
        .setExpiration(new Date(System.currentTimeMillis() + 1000 * 60 * 60)) // 1시간
        .signWith(SignatureAlgorithm.HS256, SECRET_KEY)
        .compact();
}

// JWT 검증
public boolean validateToken(String token, UserDetails userDetails) {
    final String username = extractUsername(token);
    return (username.equals(userDetails.getUsername()) && !isTokenExpired(token));
}

// JWT 필터 — 매 요청마다 실행
@Component
public class JwtAuthFilter extends OncePerRequestFilter {
    @Override
    protected void doFilterInternal(HttpServletRequest request, ...) {
        String authHeader = request.getHeader("Authorization");
        if (authHeader != null && authHeader.startsWith("Bearer ")) {
            String token = authHeader.substring(7);
            // 토큰 검증 후 SecurityContext에 인증 정보 등록
        }
        filterChain.doFilter(request, response);
    }
}
```

---

## 📌 보안 주의사항

```
✅ Access Token은 짧은 만료시간 설정 (15분 ~ 1시간)
✅ Refresh Token은 HttpOnly 쿠키에 저장 (XSS 방어)
✅ HTTPS 필수 사용 (토큰 탈취 방지)
✅ Payload에 민감한 정보 절대 금지
❌ 로컬스토리지에 Refresh Token 저장 금지 (XSS 취약)
❌ secret_key를 코드에 하드코딩 금지 → 환경변수로 관리
```

---

## 💭 오늘 느낀 점

- Payload가 암호화가 아닌 단순 인코딩이라는 점이 충격적이었다 — Base64 디코딩하면 바로 보임
- Stateless 구조가 서버 확장에 유리한 이유를 이제 명확히 이해했다
- Access/Refresh Token을 나누는 이유가 단순히 "편의"가 아니라 보안 전략임을 알게 됐다
- 다음엔 Session 방식과 비교해서 언제 뭘 써야 하는지 정리해보고 싶다

---

## 📚 참고

- [JWT 공식 사이트](https://jwt.io/)
- [RFC 7519 - JSON Web Token](https://datatracker.ietf.org/doc/html/rfc7519)
- [Spring Security 공식 문서](https://docs.spring.io/spring-security/reference/index.html)
