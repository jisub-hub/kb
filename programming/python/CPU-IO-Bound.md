---
tags:
  - python
  - concurrency
  - cpu-bound
  - io-bound
  - performance
created: 2026-06-16
---

# CPU-bound vs I/O-bound 작업 처리 전략

> [!summary] 한 줄 요약
> **CPU-bound**는 계산이 병목(GIL로 스레딩 무의미 → 멀티프로세싱), **I/O-bound**는 대기가 병목(스레딩·asyncio로 충분). 잘못된 전략은 성능 저하 또는 오버헤드 폭증으로 이어진다.

---

## 1. 병목 유형 구분

```
[CPU-bound]
  CPU가 쉬지 못함 — 계산이 다 끝날 때까지 다음 작업 불가
  예: 이미지 처리, 영상 인코딩, 암호화, ML 추론(CPU), 수치 계산
  
  → Python GIL 때문에 스레드 추가해도 동시 실행 불가
  → 해결: multiprocessing (별도 프로세스 = 별도 GIL)

[I/O-bound (네트워크 I/O)]
  CPU는 유휴 — 외부 응답 대기 (DB, HTTP, 파일 시스템)
  예: REST API 호출, DB 쿼리, 파일 읽기, 메시지 브로커 대기
  
  → GIL은 I/O syscall에서 해제됨 → 스레딩 유효
  → 해결: threading / asyncio / async/await

[혼합]
  예: AI 추론(CPU) + 결과를 DB에 저장(I/O)
  → CPU 부분은 ProcessPool, I/O 부분은 asyncio
```

---

## 2. 병목 유형 진단

```python
import cProfile
import time

# 프로파일링으로 어디서 시간 쓰는지 확인
cProfile.run('your_function()')

# 간단한 측정
import psutil, os

pid = os.getpid()
proc = psutil.Process(pid)

start = time.perf_counter()
your_function()
elapsed = time.perf_counter() - start

cpu_percent = proc.cpu_percent()
# CPU 사용률이 높으면(>80%) → CPU-bound
# CPU 사용률이 낮으면(<30%) → I/O-bound
```

---

## 3. CPU-bound 처리

### 3-1. multiprocessing

```python
from concurrent.futures import ProcessPoolExecutor
import multiprocessing

def cpu_heavy(n: int) -> int:
    return sum(i * i for i in range(n))

# CPU 코어 수만큼 프로세스 생성
with ProcessPoolExecutor(max_workers=multiprocessing.cpu_count()) as pool:
    results = list(pool.map(cpu_heavy, [10_000_000] * 8))

print(results)
```

### 3-2. NumPy/SciPy (C 확장 — GIL 해제)

```python
import numpy as np

# GIL 해제: NumPy C 레이어가 직접 실행 → 스레드로도 병렬화 가능
data = np.random.rand(10_000, 10_000)

from concurrent.futures import ThreadPoolExecutor
def matrix_op(arr):
    return np.dot(arr, arr.T)   # NumPy는 GIL 해제

with ThreadPoolExecutor() as pool:
    future = pool.submit(matrix_op, data)
    result = future.result()
```

### 3-3. Gunicorn (웹 서버) — CPU-bound

```python
# gunicorn.conf.py
import multiprocessing

bind = "0.0.0.0:8000"
workers = multiprocessing.cpu_count()     # CPU 코어 수
worker_class = "sync"                      # 동기 워커 (계산 작업)
timeout = 120
max_requests = 1000
max_requests_jitter = 200
```

---

## 4. I/O-bound 처리

### 4-1. asyncio (네트워크 I/O 최적)

```python
import asyncio
import httpx   # async HTTP 클라이언트

async def fetch(client: httpx.AsyncClient, url: str) -> dict:
    resp = await client.get(url, timeout=10)
    return resp.json()

async def fetch_all(urls: list[str]) -> list[dict]:
    async with httpx.AsyncClient() as client:
        # 100개 URL 동시 요청 (이벤트 루프가 대기 중 다른 작업 처리)
        tasks = [fetch(client, url) for url in urls]
        return await asyncio.gather(*tasks)

results = asyncio.run(fetch_all(urls))
```

### 4-2. asyncio + DB (비동기 쿼리)

```python
import asyncpg

async def fetch_users():
    conn = await asyncpg.connect("postgresql://...")
    # await → I/O 대기 중 이벤트 루프가 다른 코루틴 처리
    rows = await conn.fetch("SELECT * FROM users WHERE active = true")
    await conn.close()
    return rows

# FastAPI에서
from sqlalchemy.ext.asyncio import AsyncSession

@app.get("/users")
async def get_users(db: AsyncSession = Depends(get_db)):
    result = await db.execute(select(User))
    return result.scalars().all()
```

### 4-3. ThreadPoolExecutor (동기 블로킹 라이브러리)

```python
import asyncio
from concurrent.futures import ThreadPoolExecutor

# 블로킹 라이브러리를 asyncio에서 사용해야 할 때
def blocking_db_call(query: str) -> list:
    # 동기 드라이버 (psycopg2 등)
    return cursor.execute(query)

async def async_db_call(query: str) -> list:
    loop = asyncio.get_event_loop()
    # run_in_executor: 블로킹 함수를 스레드 풀로 오프로드
    return await loop.run_in_executor(None, blocking_db_call, query)
```

### 4-4. Gunicorn — I/O-bound 웹 서버

```python
# gunicorn.conf.py
import multiprocessing

bind = "0.0.0.0:8000"
workers = multiprocessing.cpu_count() * 2 + 1  # I/O 대기 시간 활용
worker_class = "uvicorn.workers.UvicornWorker"  # FastAPI/ASGI
timeout = 60
keepalive = 5
```

---

## 5. 상황별 처리 전략 결정표

| 작업 유형 | 예시 | 전략 | 주의 |
|-----------|------|------|------|
| CPU-bound (Pure Python) | 소수 계산, JSON 파싱 대용량 | `ProcessPoolExecutor` | 프로세스 생성 오버헤드 |
| CPU-bound (C 확장) | NumPy, OpenCV | `ThreadPoolExecutor` | GIL 해제됨 |
| CPU-bound (ML 추론) | TensorFlow, PyTorch | GPU 또는 `ProcessPool` | GPU는 GIL 무관 |
| I/O-bound (HTTP) | API 호출, 크롤링 | `asyncio + httpx` | connection pool 설정 |
| I/O-bound (DB) | SELECT, INSERT | `asyncio + asyncpg` | 커넥션 풀 크기 주의 |
| I/O-bound (파일) | 로그 읽기, CSV | `asyncio + aiofiles` | SSD면 차이 미미 |
| 혼합 | AI 추론 후 DB 저장 | `ProcessPool(추론) + asyncio(DB)` | 설계 복잡도 증가 |

---

## 6. 실제 CPU-bound 실험

```python
import time
import threading
import multiprocessing
from concurrent.futures import ThreadPoolExecutor, ProcessPoolExecutor

def count(n):
    return sum(range(n))

N = 50_000_000
TASKS = 4

# 순차
start = time.time()
for _ in range(TASKS): count(N)
print(f"순차:         {time.time() - start:.2f}s")

# 스레딩 (GIL 때문에 느려질 수 있음)
start = time.time()
with ThreadPoolExecutor(max_workers=TASKS) as ex:
    list(ex.map(count, [N]*TASKS))
print(f"스레딩:       {time.time() - start:.2f}s")

# 멀티프로세싱
start = time.time()
with ProcessPoolExecutor(max_workers=TASKS) as ex:
    list(ex.map(count, [N]*TASKS))
print(f"멀티프로세싱: {time.time() - start:.2f}s")

# 예상 결과 (4코어 CPU):
# 순차:         8.0s
# 스레딩:       8.2s  ← GIL로 거의 동일
# 멀티프로세싱: 2.3s  ← 4배 빠름
```

---

## 7. 관련
- [[Python-GIL-Concurrency]] — GIL 원리, Python 3.13 free-threaded
- [[Web-Frameworks]] — FastAPI(asyncio), Gunicorn 워커 타입
- [[../../infra/container/Docker-Network-Volume]] — 컨테이너 내 Python 앱 CPU 제한
