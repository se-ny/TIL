# Celery + asyncio 충돌 해결 패턴

## 문제 상황

Celery 태스크 안에서 `asyncio.run()`으로 async 함수를 실행할 때, SQLAlchemy asyncpg가 다음 에러를 발생시켰다:

```
AttributeError: 'NoneType' object has no attribute 'send'
```

## 원인

`asyncio.run()`은 **새로운 이벤트 루프**를 생성한다. 그런데 SQLAlchemy의 asyncpg 커넥션 풀은 **기존 이벤트 루프**에 묶여 있다.

결과적으로:
1. Celery 워커가 처음 import될 때 → 엔진/세션 생성 (루프 A에 묶임)
2. 태스크 실행 시 `asyncio.run()` → 루프 B 생성
3. 루프 B에서 루프 A에 묶인 asyncpg 소켓 접근 → **충돌**

## 해결책 — 태스크 실행 시 새 엔진 생성

전역 엔진을 재사용하지 않고, `asyncio.run()` 내부에서 **매번 새 엔진과 세션을 생성**한다.

```python
@celery_app.task(bind=True, name="analyze_jobs_task")
def analyze_jobs_task(self, job_post_ids: list[str], user_profile_id: int):
    async def _run():
        # ✅ asyncio.run() 내부에서 새 엔진 생성
        from sqlalchemy.ext.asyncio import create_async_engine, async_sessionmaker
        from app.core.config import settings

        engine = create_async_engine(settings.DATABASE_URL, echo=False)
        SessionLocal = async_sessionmaker(engine, expire_on_commit=False)

        async with SessionLocal() as session:
            # DB 작업 수행
            ...

        # ✅ 사용 후 엔진 정리
        await engine.dispose()
        return result

    return asyncio.run(_run())
```

핵심은 두 가지다:
- `create_async_engine`을 `_run()` 함수 **안에서** 호출
- 작업 완료 후 `engine.dispose()`로 커넥션 반환

---

## 왜 이렇게 해야 하나?

`asyncio.run()`은 새 루프를 만들고 완료 후 루프를 닫는다. 따라서 그 루프 안에서 생성된 엔진/커넥션은 해당 루프 생명주기와 일치한다.

전역 엔진은 처음 생성된 루프에 묶이므로, 다른 루프에서 사용하면 소켓이 이미 닫혔거나 다른 루프에 속해 있어 충돌이 발생한다.

---

## Celery에서 async를 다루는 방법들

| 방법 | 특징 | 적합한 경우 |
|------|------|------------|
| `asyncio.run()` + 새 엔진 | 간단, 매 태스크마다 커넥션 생성 | 태스크 실행 빈도가 낮을 때 |
| `gevent` / `eventlet` pool | Celery 자체를 비동기로 | 고성능이 필요할 때 |
| `celery-pool-asyncio` | async def 태스크 직접 지원 | 전체 코드베이스가 async일 때 |

job-agent 프로젝트에서는 `--pool=solo` + `asyncio.run()` + 새 엔진 방식을 사용했다. Windows 환경에서 가장 안정적으로 동작했다.

---

## 핵심 포인트 정리

- Celery 태스크는 기본적으로 동기 함수다
- `asyncio.run()`은 새 이벤트 루프를 만든다
- asyncpg 커넥션은 생성된 루프에 묶인다
- 해결책: `asyncio.run()` 내부에서 엔진을 새로 만들고, 끝나면 `dispose()`한다
- Windows에서는 `--pool=solo` 옵션이 필수다 (ProactorEventLoop 이슈)
