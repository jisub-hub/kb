---
tags:
  - python
  - gil
  - concurrency
  - gunicorn
  - multiprocessing
  - asyncio
created: 2026-06-16
---

# Python GIL과 동시성 — Gunicorn으로 웹 서버 확장

> [!summary] 한 줄 요약
> **GIL(Global Interpreter Lock)**은 CPython이 한 번에 하나의 스레드만 바이트코드를 실행하도록 막는 뮤텍스. CPU 바운드 작업에서 멀티스레드가 병렬 실행되지 않는 근본 원인. **Gunicorn 멀티프로세스**로 GIL을 우회하고, I/O 바운드는 **asyncio**로 처리한다.

---

## 1. GIL이란

```
[CPython 메모리 관리]
  Python 객체는 참조 카운트(reference count)로 메모리 관리
  두 스레드가 동시에 같은 객체의 refcount 수정 → race condition → 메모리 오염
  → GIL: CPython은 한 번에 하나의 스레드만 Python 바이트코드 실행 허용

[GIL 타임라인]
  Thread-1: ───[실행]───┐          ┌──[실행]──
  Thread-2:             └──[실행]──┘
            ↑GIL 획득   ↑GIL 양보  ↑GIL 재획득
  → 논리적으로 병렬이지만 실제로는 번갈아 가며 실행

[영향]
  CPU 바운드 (계산, 이미지 처리): 멀티스레드 = 싱글스레드와 성능 동일 또는 더 느림
  I/O 바운드 (HTTP, DB, 파일):   I/O 대기 중 GIL 해제 → 멀티스레드 효과 있음

[GIL 해제 시점]
  - I/O 시스템 콜 (socket read, file write)
  - time.sleep()
  - C 확장 (NumPy, OpenCV 내부 연산)
  - sys.getswitchinterval() 기본 5ms마다 강제 양보
```

### GIL 영향 실험

```python
import threading, time, math

def cpu_bound(n):
    """CPU만 사용하는 작업"""
    total = 0
    for i in range(n):
        total += math.sqrt(i)
    return total

n = 10_000_000

# 싱글스레드
start = time.perf_counter()
cpu_bound(n)
cpu_bound(n)
print(f"Single: {time.perf_counter() - start:.2f}s")

# 멀티스레드 2개 (GIL 때문에 더 느릴 수 있음)
start = time.perf_counter()
t1 = threading.Thread(target=cpu_bound, args=(n,))
t2 = threading.Thread(target=cpu_bound, args=(n,))
t1.start(); t2.start()
t1.join(); t2.join()
print(f"Multi-thread: {time.perf_counter() - start:.2f}s")
# 결과: 싱글 ≈ 멀티스레드 (GIL 때문에 병렬 없음, 오히려 컨텍스트 스위칭 오버헤드)

# 멀티프로세스 (각 프로세스는 독립 GIL)
from multiprocessing import Pool
start = time.perf_counter()
with Pool(2) as pool:
    pool.map(cpu_bound, [n, n])
print(f"Multi-process: {time.perf_counter() - start:.2f}s")
# 결과: 멀티프로세스 ≈ 싱글 × 0.5 (코어 2개 병렬)
```

---

## 2. GIL 우회 전략

### 전략 1 — 멀티프로세스 (CPU 바운드)

```python
from concurrent.futures import ProcessPoolExecutor
import os

def analyze_frame(frame_data: bytes) -> dict:
    """CCTV 프레임 분석 (CPU 집약적 AI 추론)"""
    # 각 프로세스는 독립 GIL → 진짜 병렬
    ...

# CPU 코어 수만큼 프로세스 생성
with ProcessPoolExecutor(max_workers=os.cpu_count()) as executor:
    futures = [executor.submit(analyze_frame, frame) for frame in frames]
    results = [f.result() for f in futures]
```

### 전략 2 — asyncio (I/O 바운드)

```python
import asyncio, aiohttp

async def fetch_sensor(url: str) -> dict:
    """HTTP I/O: GIL이 I/O 대기 중 해제됨"""
    async with aiohttp.ClientSession() as session:
        async with session.get(url) as resp:
            return await resp.json()

async def main():
    # 1000개 요청을 동시에 (실제로는 이벤트 루프가 번갈아 처리)
    tasks = [fetch_sensor(f"http://sensor-api/device/{i}") for i in range(1000)]
    results = await asyncio.gather(*tasks)

asyncio.run(main())
```

### 전략 3 — C 확장 라이브러리 (NumPy, OpenCV)

```python
import numpy as np

# NumPy 내부 C 코드 실행 시 GIL 해제 → 멀티스레드 효과 있음
arr = np.random.rand(10_000_000)

# 여러 스레드에서 NumPy 연산은 병렬로 실행됨
# (GIL 해제 구간)
```

### Python 3.13 — GIL 비활성화 옵션 (Free-Threaded)

```bash
# Python 3.13+: GIL 비활성화 빌드 (실험적)
python3.13t --disable-gil my_script.py

# 또는 환경 변수
PYTHON_GIL=0 python3.13 my_script.py
# → 주의: 스레드 안전하지 않은 C 확장은 크래시 가능
```

---

## 3. Gunicorn — 멀티프로세스 WSGI 서버

### Gunicorn 구조

```
[외부 요청]
     │
[Gunicorn Master Process]
  설정 관리, 워커 프로세스 생성/관리, 시그널 처리
     │
     ├── [Worker 1] — 독립 Python 인터프리터 + 독립 GIL
     ├── [Worker 2] — 독립 Python 인터프리터 + 독립 GIL
     ├── [Worker 3] — 독립 Python 인터프리터 + 독립 GIL
     └── [Worker N] — 독립 Python 인터프리터 + 독립 GIL

효과: N개 요청 진짜 병렬 처리 (CPU 코어 활용)
```

### 워커 타입 비교

| 워커 | 동시성 모델 | 적합 | 비고 |
|------|-----------|------|------|
| `sync` | 동기 스레드 없음 | 간단한 CPU 앱 | 기본값, 1요청씩 처리 |
| `gevent` | 코루틴 (greenlet) | I/O 바운드 대량 연결 | monkey-patching 필요 |
| `eventlet` | 코루틴 | 동일 | |
| `gthread` | 스레드 풀 | 블로킹 I/O (DB 등) | `--threads` 조합 |
| `uvicorn.workers.UvicornWorker` | asyncio | FastAPI/Starlette | **FastAPI 권장** |

### 기본 실행

```bash
# Flask / Django (동기 WSGI)
gunicorn app:app \
  --workers 4 \                  # CPU 코어 수 × 2 + 1 공식
  --worker-class sync \
  --bind 0.0.0.0:8000 \
  --timeout 30 \
  --keepalive 5 \
  --log-level info \
  --access-logfile /var/log/gunicorn-access.log \
  --error-logfile /var/log/gunicorn-error.log

# FastAPI (비동기 ASGI)
gunicorn app:app \
  --workers 4 \
  --worker-class uvicorn.workers.UvicornWorker \
  --bind 0.0.0.0:8000 \
  --timeout 60

# 또는 uvicorn 직접 (단일 프로세스)
uvicorn app:app --host 0.0.0.0 --port 8000 --workers 4
```

### Gunicorn 설정 파일

```python
# gunicorn.conf.py
import multiprocessing

# 워커 수: CPU 코어 × 2 + 1 (I/O 바운드), CPU 코어 수 (CPU 바운드)
workers = multiprocessing.cpu_count() * 2 + 1
worker_class = "uvicorn.workers.UvicornWorker"   # FastAPI
# worker_class = "sync"   # Flask/Django

bind = "0.0.0.0:8000"
timeout = 60                # 요청 처리 타임아웃
graceful_timeout = 30       # 종료 시 요청 완료 대기
keepalive = 5               # Keep-Alive 커넥션 유지 시간
max_requests = 1000         # 워커당 최대 처리 요청 수 (메모리 누수 방지)
max_requests_jitter = 100   # 모든 워커 동시 재시작 방지 랜덤성

# 로깅
accesslog = "/var/log/gunicorn/access.log"
errorlog = "/var/log/gunicorn/error.log"
loglevel = "info"
access_log_format = '%(h)s %(l)s %(u)s %(t)s "%(r)s" %(s)s %(b)s "%(f)s" %(D)sμs'

# 프리포크 (워커 메모리 공유 최적화)
preload_app = True    # 마스터에서 앱 로드 → 워커는 fork → 메모리 절약 (CoW)
                      # 주의: 소켓/DB 연결은 fork 후 재초기화 필요

# 리소스 제한
worker_connections = 1000   # gevent 사용 시 코루틴당 연결 수
```

### 워커 수 결정

```
[CPU 바운드 (AI 추론, 이미지 처리)]
  workers = CPU 코어 수
  → GIL 우회가 목적, 코어보다 많으면 컨텍스트 스위칭 손해

[I/O 바운드 (DB 쿼리, 외부 API)]
  workers = CPU 코어 수 × 2 + 1
  → I/O 대기 시간에 다른 워커 실행 가능

[비동기 (FastAPI + asyncio)]
  workers = CPU 코어 수
  worker_class = UvicornWorker
  → 워커 내부에서 asyncio로 I/O 동시성 처리 (스레드 불필요)

[실제 운영 최적화]
  → k6/locust로 부하 테스트 후 워커 수 조정
  → CPU 사용률 60~70% 유지가 목표
  → workers × threads 조합 (gthread): 소수 워커 + 스레드로 I/O 처리
     workers = 4, threads = 4 → 동시 처리 16
```

### Docker + Gunicorn

```dockerfile
FROM python:3.12-slim

WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

# 컨테이너 CPU 코어에 맞춰 워커 수 자동 설정
CMD gunicorn app:app \
    --workers $(($(nproc) * 2 + 1)) \
    --worker-class uvicorn.workers.UvicornWorker \
    --bind 0.0.0.0:8000 \
    --config gunicorn.conf.py
```

---

## 4. 관련
- [[Web-Frameworks]] — Flask / FastAPI / Django 비교
- [[../../infra/container/Docker-Network-Volume]] — Docker 배포
- [[../../infra/k8s/_index]] — K8s 배포 시 HPA로 Pod 자동 확장
