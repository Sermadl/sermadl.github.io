---
title: "Fast API 구조 알아보기"
categories: [Fast API]
tags: [Fast API, Python, SKALA, SKALA1기, SK]
---

> 생성형 AI 미니 프로젝트를 진행하면서 알게된 Fast API에 대해서 기술합니다

Fast API는 Spring처럼 각 클래스의 역할이 정해져있지 않습니다.<br>
(컨트롤러, 서비스, 레포지토리, 엔티티와 같은 클래스처럼..!)<br>

그래서 저는 Spring과 구조를 똑같이 만들어서 개발을 했습니다.<br>
~~보기 편하더라고요~~

![fast-api-structure](/assets/img/fast-api-structure.png)

static 폴더는 웹소켓 테스트를 위한 Html이 있는 폴더입니닷,,<br>

이 구조가 정말 맞는 구조인지 헷갈려서 이번 글을 작성하면서 공부를 해보려고 합니다.<br>

<hr>

## Fast API란?

FastAPI는 현대적이고, 빠르며(고성능), 파이썬 표준 타입 힌트에 기초한 Python의 API를 빌드하기 위한 웹 프레임워크입니다.<br>
빠르고, 직관적이며, 쉽다는 것이 특징입니다.<br>

<hr>

## Fast API 프로젝트 기본 구조

FastAPI는 자유도가 높은 만큼, 구조를 잘 잡는 것이 중요합니다.<br>
블로그 전반적으로 다음과 같은 폴더 구조를 사용하고 계셨습니다.<br>

```bash
app/
├── api/
│   └── v1/
│       └── endpoints/
│           └── users.py
├── crud/
│   └── user.py
├── db/
│   └── session.py
├── models/
│   └── user.py
├── services/
├── schemas/
│   └── user.py
└── main.py
```

<hr>

### `api/` - 라우터 계층

```python
from fastapi import APIRouter
from app.schemas.user import UserCreate
from app.services.user_service import create_user

router = APIRouter()

@router.post("/")
def signup(user: UserCreate):
    return create_user(user)
```

RESTful API의 각 엔드포인트를 분리해서 관리합니다. <br>
보통 v1, v2 처럼 버전관리도 함께 합니다.

<hr>

### `crud/` - 데이터베이스 상호작용

```python
from sqlalchemy.orm import Session
from app.models.user import User
from app.schemas.user import UserCreate

def create_user(db: Session, user: User):
    db.add(user)
    db.commit()
    db.refresh(user)
    return user
```

데이터베이스와 상호작용하는 CRUD 함수들이 정의되어 있습니다.<br>

<hr>

### `db/` - 데이터베이스 연동

```python
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker

DATABASE_URL = "sqlite:///./test.db"
engine = create_engine(DATABASE_URL)
SessionLocal = sessionmaker(autocommit=False, autoflush=False, bind=engine)
```

SQLAlchemy를 사용해 DB 연결을 관리합니다. FastAPI의 Depends()와 함께 사용하면 의존성 주입도 간편합니다.

<hr>

### `models/` - 테이블 정의

```python
from sqlalchemy import Column, Integer, String
from app.db.session import Base

class User(Base):
    __tablename__ = "users"
    id = Column(Integer, primary_key=True, index=True)
    username = Column(String, unique=True, index=True)
    email = Column(String, unique=True, index=True)
    full_name = Column(String)
```

데이터베이스에 정의된, 정의될 테이블을 정의합니다.

<hr>

### `services/` - 비즈니스 로직

```python
from app.models import user

def create_user(user: UserCreate):
    db_user = User(
        username=user.username,
        email=user.email,
        full_name=user.full_name
    )

    return {"msg": f"{user.username}님 가입 완료"}
```

라우터에서 로직을 분리하여, 테스트 및 재사용성을 높입니다.<br>

<hr>

### `schemas/` - 데이터 유효성 검증

```python
from pydantic import BaseModel

class UserCreate(BaseModel):
    username: str
    email: str
    password: str
```

DTO와 같은 개념으로, 요청(Request), 응답(Response) 데이터의 스키마를 정의합니다.<br>

<hr>

### `main.py` - 앱 시작점

```python
from fastapi import FastAPI
from app.api.v1 import user

app = FastAPI()

app.include_router(user.router, prefix="/users", tags=["User"])
```

FastAPI 인스턴스를 생성하고, 라우터들을 포함시키는 중앙 허브 역할을 합니다.

<hr>

## 마치며

LLM을 사용해야 했기 때문에 서버를 2개 띄우는 것보다 Fast API 서버 하나로 빠르게 개발하기 위해서 처음 사용해보았는데<br>
생각보다 굉장히 간단하고, 쉽고, 빠르게 개발할 수 있어서 좋았던 것 같습니다!<br>

다음 글에서 봐용<br>

💪🤓🤳

<hr>
<br>

> 참고 자료

[하나](https://fastapi.tiangolo.com/ko/), [둘](https://wikidocs.net/175950?utm_source=chatgpt.com). [셋](https://devocean.sk.com/blog/techBoardDetail.do?ID=167023&boardType=techBlog)
