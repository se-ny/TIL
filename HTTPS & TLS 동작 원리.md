# TIL: HTTPS & TLS 동작 원리

> 날짜: 2026-06-05  
> 태그: `#Network` `#HTTPS` `#TLS` `#Security` `#Backend`

---

## 📌 HTTP vs HTTPS

| 구분 | HTTP | HTTPS |
|------|------|-------|
| 포트 | 80 | 443 |
| 암호화 | ❌ | ✅ (TLS) |
| 데이터 도청 | 가능 | 불가 |
| 데이터 변조 | 가능 | 불가 |
| 서버 신원 확인 | ❌ | ✅ (인증서) |

**HTTPS = HTTP + TLS(Transport Layer Security)**

TLS가 HTTP 아래에서 데이터를 암호화해주는 역할을 한다.

---

## 📌 대칭키 vs 비대칭키

TLS를 이해하려면 먼저 두 가지 암호화 방식을 알아야 한다.

### 🔑 대칭키 암호화

```
암호화 키 == 복호화 키 (같은 키 사용)

송신자: 데이터 + 키A → 암호화된 데이터
수신자: 암호화된 데이터 + 키A → 원본 데이터
```

- **장점:** 빠르다
- **단점:** 키를 어떻게 안전하게 전달하지? → **키 배송 문제**

---

### 🔑 비대칭키 암호화 (공개키 암호화)

```
공개키(Public Key)  — 누구에게나 공개
개인키(Private Key) — 서버만 보유

공개키로 암호화 → 개인키로만 복호화 가능
```

- **장점:** 키 배송 문제 해결 (공개키는 누가 가져가도 OK)
- **단점:** 느리다 → 대용량 데이터 암호화엔 부적합

---

## 📌 TLS Handshake 동작 흐름

TLS는 **비대칭키로 안전하게 대칭키를 교환**하고, 이후엔 **대칭키로 빠르게 통신**한다.

```
[클라이언트]                                    [서버]

1. Client Hello  ────────────────────────────→
   (지원하는 TLS 버전, 암호화 알고리즘 목록, 랜덤값A)

2.                ←────────────────────────────  Server Hello
                                                 (선택한 알고리즘, 랜덤값B)
                  ←────────────────────────────  Certificate
                                                 (서버 인증서 — 공개키 포함)

3. 인증서 검증
   (CA 서명 확인 → 신뢰할 수 있는 서버인지 확인)

4. Pre-master Secret 생성
   (랜덤값C 생성 → 서버 공개키로 암호화해서 전송)
   Client Key Exchange  ──────────────────────→

5.                                               개인키로 복호화
                                                 → Pre-master Secret 획득

6. 양쪽 모두 랜덤값A + 랜덤값B + Pre-master Secret
   → 동일한 대칭키(Session Key) 생성

7. Finished  ────────────────────────────────→  Finished
   (이제부터 대칭키로 암호화 통신 시작! 🎉)
```

---

## 📌 인증서 & CA (Certificate Authority)

TLS Handshake 3단계에서 **"이 서버가 진짜인지"** 어떻게 확인할까?

→ **CA(인증 기관)** 가 서명한 인증서를 통해 확인한다.

```
CA (DigiCert, Let's Encrypt 등)
  ↓ 서명
서버 인증서 (도메인, 공개키, 유효기간 포함)
  ↓ 제공
클라이언트 (브라우저에 CA 목록 내장)
  ↓ 검증
"이 인증서는 신뢰할 수 있는 CA가 서명했다 → 이 서버를 믿을 수 있다"
```

### 인증서 체인 (Chain of Trust)

```
Root CA (최상위, 브라우저에 내장)
  └→ Intermediate CA (중간 인증기관)
       └→ 서버 인증서 (실제 도메인)
```

Root CA를 직접 신뢰하기 때문에, 체인을 따라 서버 인증서까지 신뢰할 수 있다.

---

## 📌 TLS 1.2 vs TLS 1.3

| 구분 | TLS 1.2 | TLS 1.3 |
|------|---------|---------|
| Handshake 왕복 | 2-RTT | 1-RTT |
| 속도 | 느림 | 빠름 |
| 취약한 알고리즘 | 일부 허용 | 완전 제거 |
| 0-RTT | ❌ | ✅ (재연결 시) |

> **현재 표준은 TLS 1.3** — 가능하면 TLS 1.3 사용 권장

---

## 📌 실무 적용 포인트

```
✅ 모든 통신은 HTTPS 사용 (HTTP는 운영 환경에서 금지)
✅ TLS 1.2 이상 사용, 가능하면 TLS 1.3
✅ 인증서 만료일 관리 (Let's Encrypt는 90일마다 갱신)
✅ HSTS 설정 — HTTP로 접근해도 자동으로 HTTPS로 리다이렉트

❌ 자체 서명 인증서(Self-signed)는 운영 환경에서 사용 금지
❌ 인증서 만료 방치 → 사용자에게 보안 경고 노출
```

### Spring Boot HTTPS 설정 예시

```yaml
# application.yml
server:
  port: 443
  ssl:
    enabled: true
    key-store: classpath:keystore.p12
    key-store-password: ${SSL_PASSWORD}
    key-store-type: PKCS12
```

---

## 📌 앞서 배운 JWT와의 연결

```
HTTPS (TLS) → 전송 구간 암호화 (중간에서 못 훔침)
JWT          → 서버가 사용자를 인증하는 토큰

HTTPS 없이 JWT만 쓰면?
→ 중간자 공격(MITM)으로 토큰 탈취 가능 → 둘 다 필요!
```

---

## 💭 오늘 느낀 점

- TLS가 "비대칭키로 대칭키를 교환한다"는 핵심 아이디어를 이해하니 전체 흐름이 한 번에 잡혔다
- CA와 인증서 체인 개념 덕분에 브라우저 주소창 자물쇠 아이콘의 의미를 제대로 이해하게 됐다
- 지난번에 배운 JWT와 HTTPS가 서로 다른 레이어에서 보안을 담당한다는 것이 명확해졌다

---

## 📚 참고

- [MDN - TLS](https://developer.mozilla.org/ko/docs/Web/Security/Transport_Layer_Security)
- [Cloudflare - TLS Handshake 설명](https://www.cloudflare.com/learning/ssl/what-happens-in-a-tls-handshake/)
- [RFC 8446 - TLS 1.3](https://datatracker.ietf.org/doc/html/rfc8446)
