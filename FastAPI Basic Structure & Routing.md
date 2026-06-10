# TIL: FastAPI Basic Structure & Routing

> 날짜: 2026-06-10  
> 태그: `#FastAPI` `#Python` `#Backend` `#REST`

---

## 📌 FastAPI란?

**FastAPI** 는 Python 기반의 현대적인 웹 프레임워크다.

```
빠른 속도    — Node.js, Go 수준의 성능 (ASGI 기반)
자동 문서화  — Swagger UI, ReDoc 자동 생성
타입 힌트    — Python 타입 힌트로 요청/응답 자동 검증 (Pydantic)
비동기 지원  — async/await 네이티브 지원
```

---

## 📌 프로젝트 기본 구조

```
my-project/
├── app/
│   ├── main.py          # FastAPI 앱 진입점
│   ├── routers/         # 라우터 모음
│   │   ├── users.py
│   │   └── items.py
│   ├── models/          # DB 모델 (SQLAlchemy)
│   ├── schemas/         # 요청/응답 스키마 (Pydantic)
│   ├── crud/            # DB CRUD 로직
│   └── database.py      # DB 연결 설정
├── requirements.txt
└── .env
```

---

## 📌 앱 기본 설정 (main.py)

```python
from fastapi import FastAPI
from app.routers import users, items

app = FastAPI(
    title="My API",
    description="FastAPI 기본 구조 예시",
    version="1.0.0"
)

# 라우터 등록
app.include_router(users.router, prefix="/users", tags=["users"])
app.include_router(items.router, prefix="/items", tags=["items"])

@app.get("/")
def root():
    return {"message": "Hello, FastAPI!"}
```

---

## 📌 라우팅 기본

```python
from fastapi import FastAPI

app = FastAPI()

# GET — 조회
@app.get("/users")
def get_users():
    return [{"id": 1, "name": "김철수"}, {"id": 2, "name": "이영희"}]

# Path Parameter — 경로 변수
@app.get("/users/{user_id}")
def get_user(user_id: int):   # 타입 힌트 → 자동 형변환 & 검증
    return {"id": user_id, "name": "김철수"}

# Query Parameter — 쿼리 파라미터
@app.get("/users")
def get_users(skip: int = 0, limit: int = 10):
    # GET /users?skip=0&limit=10
    return {"skip": skip, "limit": limit}

# POST — 생성
@app.post("/users", status_code=201)
def create_user(user: UserCreate):  # 요청 바디 (Pydantic 스키마)
    return {"id": 3, **user.dict()}

# PATCH — 부분 수정
@app.patch("/users/{user_id}")
def update_user(user_id: int, user: UserUpdate):
    return {"id": user_id, **user.dict()}

# DELETE — 삭제
@app.delete("/users/{user_id}", status_code=204)
def delete_user(user_id: int):
    return None
```

---

## 📌 Pydantic 스키마로 요청/응답 정의

```python
from pydantic import BaseModel, EmailStr
from typing import Optional

# 요청 스키마
class UserCreate(BaseModel):
    name: str
    email: EmailStr
    age: int

# 응답 스키마
class UserResponse(BaseModel):
    id: int
    name: str
    email: str

    class Config:
        from_attributes = True  # ORM 모델 → Pydantic 변환 허용

# 부분 수정용 스키마 (모든 필드 Optional)
class UserUpdate(BaseModel):
    name: Optional[str] = None
    email: Optional[EmailStr] = None
    age: Optional[int] = None
```

---

## 📌 APIRouter로 라우터 분리

규모가 커질수록 라우터를 파일로 분리하는 게 필수다.

```python
# routers/users.py
from fastapi import APIRouter
from app.schemas.user import UserCreate, UserResponse

router = APIRouter()

@router.get("/", response_model=list[UserResponse])
def get_users():
    return []

@router.post("/", response_model=UserResponse, status_code=201)
def create_user(user: UserCreate):
    return {"id": 1, **user.dict()}

@router.get("/{user_id}", response_model=UserResponse)
def get_user(user_id: int):
    return {"id": user_id, "name": "김철수", "email": "kim@test.com"}
```

```python
# main.py에서 등록
app.include_router(users.router, prefix="/users", tags=["users"])

# 결과 라우트:
# GET  /users
# POST /users
# GET  /users/{user_id}
```

---

## 📌 자동 문서화

FastAPI는 코드만 작성하면 문서가 자동 생성된다.

```
http://localhost:8000/docs      → Swagger UI (인터랙티브)
http://localhost:8000/redoc     → ReDoc (읽기 전용)
http://localhost:8000/openapi.json → OpenAPI 스펙 (JSON)
```

별도 설정 없이 라우터와 스키마를 정의하는 것만으로 자동으로 채워진다.

---

## 📌 앱 실행

```bash
# 설치
pip install fastapi uvicorn

# 개발 서버 실행 (코드 변경 시 자동 재시작)
uvicorn app.main:app --reload

# 포트 지정
uvicorn app.main:app --reload --port 8080
```

---

## 📌 Flask vs FastAPI 비교

| 구분 | Flask | FastAPI |
|------|-------|---------|
| 성능 | WSGI (동기) | ASGI (비동기) |
| 타입 검증 | 직접 구현 | Pydantic 자동 |
| 자동 문서화 | ❌ | ✅ Swagger 자동 |
| 비동기 지원 | 제한적 | 네이티브 |
| 학습 난이도 | 쉬움 | 보통 |
| AI 백엔드 적합성 | 보통 | 높음 |

---

## 💭 오늘 느낀 것

- `prefix`와 `tags`로 라우터를 깔끔하게 분리하는 구조가 Spring의 `@RequestMapping`과 비슷하면서도 훨씬 간결하다
- Pydantic 스키마 하나로 요청 검증 + 응답 직렬화 + 자동 문서화까지 한 번에 되는 게 인상적이다
- DocuChat 만들 때 구조를 제대로 잡지 않고 썼는데, 다음 프로젝트엔 이 구조대로 처음부터 설계해야겠다

---

## 📚 참고

- [FastAPI 공식 문서](https://fastapi.tiangolo.com/)
- [FastAPI 공식 문서 — Bigger Applications](https://fastapi.tiangolo.com/tutorial/bigger-applications/)
- [Pydantic 공식 문서](https://docs.pydantic.dev/)
