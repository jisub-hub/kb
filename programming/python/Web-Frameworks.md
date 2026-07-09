---
tags:
  - python
  - flask
  - fastapi
  - django
  - web-framework
created: 2026-06-16
---

# Python 웹 프레임워크 — Flask · FastAPI · Django

> [!summary] 한 줄 요약
> **Flask**: 마이크로 프레임워크, 자유도 최고, 소규모 서비스. **FastAPI**: 비동기 + 자동 문서화 + 타입 힌트, API 서버/ML 서빙 최적. **Django**: 배터리 포함(ORM·Admin·Auth), 빠른 프로토타입·복잡한 백오피스.

---

## 1. 세 프레임워크 비교

| | Flask | FastAPI | Django |
|--|-------|---------|--------|
| **출시** | 2010 | 2018 | 2005 |
| **패러다임** | 동기 (비동기 부분 지원) | 비동기 (동기도 가능) | 동기 (3.1+부터 비동기) |
| **WSGI/ASGI** | WSGI (+ Quart로 ASGI) | ASGI | WSGI + ASGI |
| **ORM** | 없음 (SQLAlchemy 별도) | 없음 (SQLAlchemy/Tortoise) | 내장 ORM |
| **Admin** | 없음 (Flask-Admin 별도) | 없음 | ✅ 강력한 Admin 내장 |
| **인증** | 없음 (별도) | 없음 (별도) | ✅ 내장 |
| **자동 문서화** | ❌ (Swagger 별도) | ✅ OpenAPI 자동 생성 | ❌ (DRF+Swagger) |
| **성능** | 중간 | 가장 빠름 (비동기) | 중간 |
| **학습 곡선** | 낮음 | 중간 | 높음 |
| **번들 크기** | 최소 | 최소~중간 | 큼 |

---

## 2. Flask

### 특징

```
장점:
  ✅ 단순함 — 파일 하나로 API 서버 구축 가능
  ✅ 자유도 — ORM, 인증, 직렬화 전부 선택
  ✅ 가장 큰 서드파티 생태계 (오랜 역사)
  ✅ 학습 자료 풍부
  ✅ 마이크로서비스 단일 엔드포인트에 최적

단점:
  ❌ 기본 비동기 미지원 (Quart로 대체 가능)
  ❌ 검증·직렬화 모두 직접 구현 필요
  ❌ 규모 커지면 구조화 필요 (Blueprint, Application Factory)
  ❌ 자동 문서화 없음

적합:
  - 빠른 프로토타입
  - 단순 내부 API, 웹훅
  - 레거시 Python 2/3 코드베이스 통합
  - Flask 생태계 익숙한 팀
```

### 코드 예시

```python
from flask import Flask, request, jsonify, abort
from flask_sqlalchemy import SQLAlchemy

app = Flask(__name__)
app.config["SQLALCHEMY_DATABASE_URI"] = "postgresql://user:pass@localhost/db"
db = SQLAlchemy(app)

class User(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    email = db.Column(db.String(120), unique=True, nullable=False)
    name = db.Column(db.String(80), nullable=False)

@app.route("/users", methods=["GET"])
def list_users():
    users = User.query.all()
    return jsonify([{"id": u.id, "email": u.email} for u in users])

@app.route("/users/<int:user_id>", methods=["GET"])
def get_user(user_id):
    user = User.query.get_or_404(user_id)
    return jsonify({"id": user.id, "email": user.email, "name": user.name})

@app.route("/users", methods=["POST"])
def create_user():
    data = request.get_json()
    if not data or "email" not in data:
        abort(400, description="email required")

    user = User(email=data["email"], name=data.get("name", ""))
    db.session.add(user)
    db.session.commit()
    return jsonify({"id": user.id}), 201

# Blueprint로 구조화
from flask import Blueprint
users_bp = Blueprint("users", __name__, url_prefix="/api/users")

@users_bp.route("/me")
def me():
    ...

app.register_blueprint(users_bp)
```

### Flask + Gunicorn 운영

```bash
gunicorn "app:create_app()" \
  --workers 4 \
  --worker-class sync \
  --bind 0.0.0.0:8000
```

---

## 3. FastAPI

### 특징

```
장점:
  ✅ 타입 힌트 기반 자동 검증 (Pydantic)
  ✅ OpenAPI 문서 자동 생성 (/docs, /redoc)
  ✅ asyncio 네이티브 — 비동기 DB, HTTP, 파일 I/O
  ✅ 성능 최고 수준 (Uvicorn + asyncio, Node.js급)
  ✅ Dependency Injection 내장
  ✅ Python 타입 힌트와 자연스럽게 통합

단점:
  ❌ ORM·Admin·Auth 없음 → SQLAlchemy + 직접 구현
  ❌ 비동기 코드 이해 필요
  ❌ 생태계가 Flask보다 젊음

적합:
  - REST API 서버 (ML 모델 서빙, IoT 데이터 수집)
  - 높은 동시 I/O (수천 센서 연결, 실시간 스트리밍)
  - 타입 안전성이 중요한 팀
  - 마이크로서비스 API Gateway 뒤에 위치하는 서비스
  - FastAPI가 Python API 서버의 사실상 표준 (2024+)
```

### 코드 예시

```python
from fastapi import FastAPI, Depends, HTTPException, status
from pydantic import BaseModel, EmailStr
from sqlalchemy.ext.asyncio import AsyncSession, create_async_engine
from sqlalchemy.orm import sessionmaker, DeclarativeBase
from sqlalchemy import Column, Integer, String, select

app = FastAPI(title="My API", version="1.0.0")

# ── DB 설정 ────────────────────────────────────────────────────
engine = create_async_engine("postgresql+asyncpg://user:pass@localhost/db")
AsyncSessionLocal = sessionmaker(engine, class_=AsyncSession, expire_on_commit=False)

async def get_db():
    async with AsyncSessionLocal() as session:
        yield session

# ── 모델 ──────────────────────────────────────────────────────
class Base(DeclarativeBase): pass

class User(Base):
    __tablename__ = "users"
    id    = Column(Integer, primary_key=True)
    email = Column(String, unique=True, nullable=False)
    name  = Column(String, nullable=False)

# ── 스키마 (Pydantic) ─────────────────────────────────────────
class UserCreate(BaseModel):
    email: EmailStr
    name:  str

class UserResponse(BaseModel):
    id:    int
    email: str
    name:  str

    model_config = {"from_attributes": True}   # ORM 객체 자동 변환

# ── 엔드포인트 ─────────────────────────────────────────────────
@app.get("/users", response_model=list[UserResponse])
async def list_users(db: AsyncSession = Depends(get_db)):
    result = await db.execute(select(User))
    return result.scalars().all()

@app.get("/users/{user_id}", response_model=UserResponse)
async def get_user(user_id: int, db: AsyncSession = Depends(get_db)):
    user = await db.get(User, user_id)
    if not user:
        raise HTTPException(status_code=404, detail="User not found")
    return user

@app.post("/users", response_model=UserResponse, status_code=201)
async def create_user(body: UserCreate, db: AsyncSession = Depends(get_db)):
    user = User(email=body.email, name=body.name)
    db.add(user)
    await db.commit()
    await db.refresh(user)
    return user

# ── 의존성 주입 패턴 ───────────────────────────────────────────
from fastapi.security import OAuth2PasswordBearer
oauth2_scheme = OAuth2PasswordBearer(tokenUrl="token")

async def get_current_user(
    token: str = Depends(oauth2_scheme),
    db: AsyncSession = Depends(get_db)
) -> User:
    # JWT 파싱 + 사용자 조회
    ...
    return user

@app.get("/users/me", response_model=UserResponse)
async def read_me(current_user: User = Depends(get_current_user)):
    return current_user
```

### FastAPI + Gunicorn 운영

```bash
# 프로덕션: Gunicorn 멀티프로세스 + UvicornWorker
gunicorn app:app \
  --workers 4 \
  --worker-class uvicorn.workers.UvicornWorker \
  --bind 0.0.0.0:8000

# 개발: uvicorn 직접 (hot reload)
uvicorn app:app --reload --host 0.0.0.0 --port 8000
```

---

## 4. Django

### 특징

```
장점:
  ✅ 배터리 포함 — ORM, Admin, Auth, Forms, 캐시, 마이그레이션 내장
  ✅ Django Admin — 모델 정의만 해도 관리자 UI 자동 생성
  ✅ 강력한 ORM — 쿼리셋 체이닝, QuerySet API
  ✅ 보안 내장 — CSRF, XSS, SQL인젝션 기본 방어
  ✅ 성숙한 생태계 — Django REST Framework (DRF) 으로 API 구축
  ✅ 규모 있는 단일 애플리케이션에 강점

단점:
  ❌ 무거운 시작 오버헤드
  ❌ 모든 것이 Django 방식 (자유도 낮음)
  ❌ 비동기 지원이 후발 (3.1+, ORM 비동기는 4.1+)
  ❌ 마이크로서비스 분리 어색함
  ❌ 단순 API 서버에는 과도한 설정

적합:
  - 복잡한 도메인 모델 (ERP, CMS, 이커머스)
  - 관리자 UI가 필요한 서비스 (Django Admin 활용)
  - 빠른 MVP (많은 기능이 내장)
  - 백오피스 어드민 시스템
  - 팀이 Python 비숙련자 포함 (구조화된 패턴 강제)
```

### 코드 예시 (DRF)

```python
# models.py
from django.db import models

class User(models.Model):
    email = models.EmailField(unique=True)
    name  = models.CharField(max_length=80)
    created_at = models.DateTimeField(auto_now_add=True)

    class Meta:
        ordering = ["-created_at"]

# serializers.py
from rest_framework import serializers

class UserSerializer(serializers.ModelSerializer):
    class Meta:
        model = User
        fields = ["id", "email", "name", "created_at"]
        read_only_fields = ["id", "created_at"]

# views.py
from rest_framework import viewsets, permissions

class UserViewSet(viewsets.ModelViewSet):
    queryset = User.objects.all()
    serializer_class = UserSerializer
    permission_classes = [permissions.IsAuthenticated]

    def get_queryset(self):
        return super().get_queryset().filter(
            email__contains=self.request.query_params.get("q", "")
        )

# urls.py
from rest_framework.routers import DefaultRouter

router = DefaultRouter()
router.register("users", UserViewSet)
urlpatterns = router.urls

# admin.py — Admin 자동 UI
from django.contrib import admin

@admin.register(User)
class UserAdmin(admin.ModelAdmin):
    list_display = ["id", "email", "name", "created_at"]
    search_fields = ["email", "name"]
    list_filter = ["created_at"]
```

### Django + Gunicorn 운영

```bash
# 정적 파일 수집
python manage.py collectstatic --noinput

# 마이그레이션
python manage.py migrate

# Gunicorn 실행
gunicorn myproject.wsgi:application \
  --workers 4 \
  --bind 0.0.0.0:8000 \
  --timeout 60
```

---

## 5. 프레임워크 선택 기준

```
[새 프로젝트 의사결정 트리]

비동기 I/O가 핵심? (고동시성, IoT, ML 서빙)
  YES → FastAPI
  NO  ↓

관리자 UI / 복잡한 도메인 모델?
  YES → Django (+ DRF)
  NO  ↓

팀 크기 < 3명, 빠른 프로토타입?
  YES → Flask (단순함 우선)
  NO  → FastAPI (성능 + 타입 안전)

[상황별 선택]
  IoT 센서 데이터 수집 API     → FastAPI (비동기, 고동시성)
  ML 모델 REST API 서빙        → FastAPI (Pydantic 검증, 자동 문서)
  이커머스 백엔드               → Django (Admin, 복잡한 관계)
  내부 관리 대시보드            → Django (Admin UI)
  간단한 웹훅 수신              → Flask (경량)
  마이크로서비스 API            → FastAPI (빠른 시작, 타입 힌트)
  CRUD 어드민 도구              → Django (Admin 활용)
```

---

## 6. 공통 운영 패턴

### Nginx + Gunicorn

```nginx
# /etc/nginx/sites-available/myapi
upstream gunicorn {
    server 127.0.0.1:8000;
    keepalive 64;
}

server {
    listen 80;
    server_name api.example.com;

    location / {
        proxy_pass http://gunicorn;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_connect_timeout 5s;
        proxy_read_timeout 60s;
    }

    location /static/ {
        alias /var/www/static/;      # Django collectstatic 결과
        expires 1y;
        add_header Cache-Control "public, immutable";
    }
}
```

### requirements.txt 구성

```
# Flask 스택
Flask==3.0.3
Flask-SQLAlchemy==3.1.1
Flask-Migrate==4.0.7
gunicorn==22.0.0
psycopg2-binary==2.9.9

# FastAPI 스택
fastapi==0.111.0
uvicorn[standard]==0.30.1
gunicorn==22.0.0
sqlalchemy==2.0.30
asyncpg==0.29.0
pydantic[email]==2.7.4

# Django 스택
Django==5.0.6
djangorestframework==3.15.1
psycopg2-binary==2.9.9
gunicorn==22.0.0
```

---

## 7. 관련
- [[Python-GIL-Concurrency]] — GIL, Gunicorn 워커 수 결정, asyncio
- [[../../infra/container/Docker-Network-Volume]] — Docker 배포
- [[../../infra/observability/PLG-Stack]] — 로그·메트릭 수집
