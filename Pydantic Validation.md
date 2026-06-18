# TIL: Pydantic Validation

> 날짜: 2026-06-18  
> 태그: `#FastAPI` `#Python` `#Pydantic` `#Validation` `#Backend`

---

## 📌 Pydantic이란?

**Pydantic** 은 Python 타입 힌트를 기반으로 **데이터 유효성 검사와 직렬화**를 담당하는 라이브러리다.

FastAPI는 내부적으로 Pydantic을 사용해서 요청/응답 데이터를 자동으로 검증한다.

```python
# Pydantic 없이 직접 검증 (번거롭고 실수하기 쉬움)
def create_user(data: dict):
    if "name" not in data:
        raise ValueError("name은 필수입니다")
    if not isinstance(data["age"], int):
        raise ValueError("age는 정수여야 합니다")
    if data["age"] < 0:
        raise ValueError("age는 0 이상이어야 합니다")
    ...

# Pydantic 사용 (간결하고 명확)
class UserCreate(BaseModel):
    name: str
    age: int = Field(ge=0)  # 자동으로 위 검증을 전부 처리!
```

---

## 📌 기본 모델 정의

```python
from pydantic import BaseModel, Field, EmailStr
from typing import Optional, List
from datetime import datetime

class UserCreate(BaseModel):
    name: str
    email: EmailStr                    # 이메일 형식 자동 검증
    age: int
    bio: Optional[str] = None          # 선택 필드 (기본값 None)
    tags: List[str] = []               # 리스트 타입
    created_at: datetime = Field(      # 기본값 함수
        default_factory=datetime.now
    )
```

---

## 📌 Field로 세부 제약 조건 설정

```python
from pydantic import BaseModel, Field

class ProductCreate(BaseModel):
    name: str = Field(
        min_length=2,        # 최소 길이
        max_length=100,      # 최대 길이
        description="상품명"  # Swagger 문서에 표시
    )
    price: float = Field(
        gt=0,                # greater than 0 (0 초과)
        le=1_000_000,        # less than or equal (100만 이하)
        description="가격"
    )
    stock: int = Field(
        ge=0,                # greater than or equal (0 이상)
        description="재고 수량"
    )
    discount_rate: float = Field(
        default=0.0,
        ge=0.0,
        le=1.0,              # 0.0 ~ 1.0 사이
        description="할인율 (0.0 ~ 1.0)"
    )
```

| Field 옵션 | 의미 |
|-----------|------|
| `gt` | 초과 (greater than) |
| `ge` | 이상 (greater than or equal) |
| `lt` | 미만 (less than) |
| `le` | 이하 (less than or equal) |
| `min_length` | 최소 문자 길이 |
| `max_length` | 최대 문자 길이 |
| `pattern` | 정규식 패턴 |

---

## 📌 커스텀 validator

Field만으로 부족한 복잡한 검증 로직은 `@field_validator`로 직접 정의한다.

```python
from pydantic import BaseModel, field_validator, model_validator

class UserCreate(BaseModel):
    name: str
    password: str
    password_confirm: str
    age: int

    # 단일 필드 검증
    @field_validator("name")
    @classmethod
    def name_must_not_contain_special_chars(cls, v: str) -> str:
        if not v.replace(" ", "").isalnum():
            raise ValueError("이름에 특수문자를 포함할 수 없습니다")
        return v.strip()  # 앞뒤 공백 제거 후 반환

    @field_validator("age")
    @classmethod
    def age_must_be_adult(cls, v: int) -> int:
        if v < 18:
            raise ValueError("18세 이상만 가입할 수 있습니다")
        return v

    # 여러 필드를 함께 검증 (model_validator)
    @model_validator(mode="after")
    def passwords_must_match(self) -> "UserCreate":
        if self.password != self.password_confirm:
            raise ValueError("비밀번호가 일치하지 않습니다")
        return self
```

---

## 📌 중첩 모델 (Nested Model)

```python
from pydantic import BaseModel
from typing import List

class Address(BaseModel):
    city: str
    district: str
    detail: str

class OrderItem(BaseModel):
    product_id: int
    quantity: int = Field(ge=1)
    price: float

class OrderCreate(BaseModel):
    user_id: int
    address: Address            # 중첩 모델
    items: List[OrderItem]      # 모델 리스트
    memo: str = ""

# 사용 예시
order = OrderCreate(
    user_id=1,
    address={"city": "서울", "district": "강남구", "detail": "테헤란로 123"},
    items=[
        {"product_id": 1, "quantity": 2, "price": 15000},
        {"product_id": 3, "quantity": 1, "price": 32000},
    ]
)
print(order.address.city)  # "서울"
print(order.items[0].price)  # 15000.0
```

---

## 📌 요청/응답 스키마 분리 패턴

실무에서는 목적에 따라 스키마를 분리하는 게 좋다.

```python
from pydantic import BaseModel, EmailStr
from typing import Optional
from datetime import datetime

# 생성 요청 (비밀번호 포함)
class UserCreate(BaseModel):
    name: str
    email: EmailStr
    password: str

# 수정 요청 (모든 필드 선택적)
class UserUpdate(BaseModel):
    name: Optional[str] = None
    email: Optional[EmailStr] = None

# 응답 (비밀번호 제외, id/created_at 포함)
class UserResponse(BaseModel):
    id: int
    name: str
    email: str
    created_at: datetime

    model_config = {"from_attributes": True}  # ORM 객체 → Pydantic 변환 허용
```

---

## 📌 FastAPI에서 Pydantic 활용

```python
from fastapi import FastAPI, HTTPException
from pydantic import ValidationError

app = FastAPI()

@app.post("/users", response_model=UserResponse, status_code=201)
def create_user(user: UserCreate):
    # FastAPI가 자동으로 요청 바디를 UserCreate로 파싱 & 검증
    # 검증 실패 시 자동으로 422 Unprocessable Entity 반환

    # DB 저장 로직 (예시)
    db_user = save_user_to_db(user)

    # response_model=UserResponse → 자동으로 UserResponse 형태로 직렬화
    return db_user


# 수동 유효성 검사가 필요한 경우
@app.post("/validate")
def validate_manually(data: dict):
    try:
        user = UserCreate(**data)
        return {"valid": True, "data": user.model_dump()}
    except ValidationError as e:
        return {"valid": False, "errors": e.errors()}
```

---

## 📌 Pydantic v1 vs v2 주요 변경점

FastAPI 최신 버전은 Pydantic v2를 사용한다.

| 구분 | Pydantic v1 | Pydantic v2 |
|------|------------|------------|
| validator | `@validator` | `@field_validator` |
| 전체 모델 검증 | `@root_validator` | `@model_validator` |
| ORM 모드 설정 | `class Config: orm_mode = True` | `model_config = {"from_attributes": True}` |
| dict 변환 | `.dict()` | `.model_dump()` |
| json 변환 | `.json()` | `.model_dump_json()` |
| 성능 | 기본 | v1 대비 5~50배 빠름 (Rust 기반) |

---

## 💭 오늘 느낀 점

- Pydantic의 `@field_validator`와 `@model_validator`를 조합하면 복잡한 비즈니스 규칙도 모델 안에서 깔끔하게 처리할 수 있다
- 요청/응답 스키마를 분리하는 패턴이 처음엔 번거로워 보였는데, 비밀번호 같은 민감한 정보가 응답에 노출되는 걸 자연스럽게 방지해준다
- DocuChat이나 채용공고 에이전트에서 Pydantic 스키마를 좀 더 꼼꼼하게 설계했으면 좋았겠다는 생각이 든다

---

## 📚 참고

- [Pydantic 공식 문서](https://docs.pydantic.dev/latest/)
- [FastAPI - Request Body 공식 문서](https://fastapi.tiangolo.com/tutorial/body/)
- [FastAPI - Response Model 공식 문서](https://fastapi.tiangolo.com/tutorial/response-model/)
