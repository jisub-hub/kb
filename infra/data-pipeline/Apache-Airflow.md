---
tags:
  - data-pipeline
  - airflow
  - etl
  - dag
  - scheduler
created: 2026-06-16
---

# Apache Airflow — DAG 기반 ETL 파이프라인

> [!summary] 한 줄 요약
> Airflow는 **DAG(Directed Acyclic Graph)로 데이터 파이프라인을 코드로 정의**하는 워크플로우 오케스트레이터. Scheduler가 cron 주기로 DAG를 실행하고, 태스크 의존성·재시도·모니터링을 내장한다.

---

## 1. 핵심 아키텍처

```
[컴포넌트]

  DAG 파일 (Python) ←── 개발자가 작성
       │
       ▼
  Scheduler ── 주기 체크, 태스크 큐에 등록
       │
       ▼
  Executor (Celery/K8s) ── 태스크 실제 실행
       │
  Worker × N
       │
       ▼
  Metadata DB (PostgreSQL) ── 실행 이력, 상태 저장
       │
       ▼
  Web Server (Flask) ── DAG 모니터링 UI (localhost:8080)

[DAG란?]
  - Directed: 실행 방향이 있음 (A → B → C)
  - Acyclic:  순환 없음 (B가 다시 A를 호출하면 DAG 파일 로드 오류)
  - Graph:    태스크(노드) + 의존성(엣지)
```

---

## 2. 설치 (Docker Compose)

```yaml
# docker-compose.yml (공식 Airflow 구성)
version: '3'

x-airflow-common: &airflow-common
  image: apache/airflow:2.9.0
  environment:
    AIRFLOW__DATABASE__SQL_ALCHEMY_CONN: postgresql+psycopg2://airflow:airflow@postgres/airflow
    AIRFLOW__CELERY__RESULT_BACKEND: db+postgresql://airflow:airflow@postgres/airflow
    AIRFLOW__CELERY__BROKER_URL: redis://redis:6379/0
    AIRFLOW__CORE__FERNET_KEY: ''
    AIRFLOW__CORE__DAGS_ARE_PAUSED_AT_CREATION: 'true'
    AIRFLOW__CORE__LOAD_EXAMPLES: 'false'
    AIRFLOW__API__AUTH_BACKENDS: 'airflow.api.auth.backend.basic_auth'
  volumes:
    - ./dags:/opt/airflow/dags          # DAG 파일 마운트
    - ./logs:/opt/airflow/logs
    - ./plugins:/opt/airflow/plugins

services:
  postgres:
    image: postgres:15
    environment:
      POSTGRES_USER: airflow
      POSTGRES_PASSWORD: airflow
      POSTGRES_DB: airflow

  redis:
    image: redis:7-alpine

  airflow-webserver:
    <<: *airflow-common
    ports:
      - "8080:8080"
    command: webserver

  airflow-scheduler:
    <<: *airflow-common
    command: scheduler

  airflow-worker:
    <<: *airflow-common
    command: celery worker

  airflow-init:
    <<: *airflow-common
    command: version
    environment:
      _AIRFLOW_DB_UPGRADE: 'true'
      _AIRFLOW_WWW_USER_CREATE: 'true'
      _AIRFLOW_WWW_USER_USERNAME: admin
      _AIRFLOW_WWW_USER_PASSWORD: admin
```

---

## 3. DAG 기본 작성법

```python
# dags/sensor_etl.py
from datetime import datetime, timedelta
from airflow import DAG
from airflow.operators.python import PythonOperator
from airflow.operators.bash import BashOperator
from airflow.providers.postgres.hooks.postgres import PostgresHook

# ── DAG 기본 설정 ─────────────────────────────────────────────
default_args = {
    'owner': 'data-team',
    'depends_on_past': False,         # 이전 실행 실패해도 오늘 실행
    'retries': 3,
    'retry_delay': timedelta(minutes=5),
    'email_on_failure': True,
    'email': ['data-team@company.com'],
}

with DAG(
    dag_id='sensor_data_etl',
    default_args=default_args,
    description='IoT 센서 데이터 ETL 파이프라인',
    schedule='0 * * * *',            # 매시 정각 실행 (cron 표현식)
    start_date=datetime(2026, 1, 1),
    catchup=False,                    # 과거 미실행 구간 건너뜀
    tags=['iot', 'etl'],
    max_active_runs=1,                # 동시 실행 최대 1개
) as dag:

    # ── Task 1: 원천 API에서 데이터 수집 ──────────────────────
    def extract_sensor_data(**context):
        import httpx
        execution_dt = context['execution_date']

        resp = httpx.get(
            "http://iot-api/sensors/data",
            params={"from": execution_dt.isoformat()}
        )
        data = resp.json()

        # XCom으로 다음 태스크에 데이터 전달
        context['ti'].xcom_push(key='raw_data', value=data)
        print(f"[추출] {len(data)} 건 수집")
        return len(data)

    extract = PythonOperator(
        task_id='extract_sensor_data',
        python_callable=extract_sensor_data,
    )

    # ── Task 2: 데이터 변환 ───────────────────────────────────
    def transform_data(**context):
        raw = context['ti'].xcom_pull(task_ids='extract_sensor_data', key='raw_data')

        transformed = []
        for record in raw:
            # 정규화: 섭씨→켈빈, 결측값 처리, 이상값 필터링
            temp_c = record.get('temperature')
            if temp_c is None or temp_c < -50 or temp_c > 100:
                continue                          # 이상값 제거
            transformed.append({
                'device_id':  record['device_id'],
                'temp_kelvin': temp_c + 273.15,
                'humidity':    record.get('humidity', 0),
                'ts':          record['timestamp'],
            })

        context['ti'].xcom_push(key='transformed', value=transformed)
        print(f"[변환] {len(transformed)}/{len(raw)} 건 통과")
        return len(transformed)

    transform = PythonOperator(
        task_id='transform_data',
        python_callable=transform_data,
    )

    # ── Task 3: DB에 적재 ─────────────────────────────────────
    def load_to_db(**context):
        data = context['ti'].xcom_pull(task_ids='transform_data', key='transformed')
        if not data:
            print("[적재] 적재할 데이터 없음, 건너뜀")
            return

        # Airflow Connection 사용 (Admin > Connections에서 설정)
        hook = PostgresHook(postgres_conn_id='timescaledb_conn')
        conn = hook.get_conn()
        cursor = conn.cursor()

        cursor.executemany("""
            INSERT INTO sensor_readings (device_id, temp_kelvin, humidity, ts)
            VALUES (%(device_id)s, %(temp_kelvin)s, %(humidity)s, %(ts)s)
            ON CONFLICT (device_id, ts) DO UPDATE
              SET temp_kelvin = EXCLUDED.temp_kelvin,
                  humidity    = EXCLUDED.humidity
        """, data)

        conn.commit()
        print(f"[적재] {len(data)} 건 DB 저장 완료")

    load = PythonOperator(
        task_id='load_to_db',
        python_callable=load_to_db,
    )

    # ── 태스크 의존성 정의 ────────────────────────────────────
    extract >> transform >> load
```

---

## 4. ETL 패턴

### Sensor (외부 상태 폴링)

```python
from airflow.sensors.python import PythonSensor

def check_file_ready(**context):
    """파일이 생성될 때까지 대기"""
    import os
    return os.path.exists("/data/import/today.csv")

wait_for_file = PythonSensor(
    task_id='wait_for_file',
    python_callable=check_file_ready,
    poke_interval=60,        # 60초마다 체크
    timeout=3600,            # 1시간 대기 후 실패
    mode='reschedule',       # 워커 슬롯 점유 없이 폴링 (권장)
)
```

### 분기 (BranchOperator)

```python
from airflow.operators.python import BranchPythonOperator

def decide_path(**context):
    row_count = context['ti'].xcom_pull(task_ids='extract', key='count')
    if row_count > 0:
        return 'transform_and_load'
    return 'skip_task'

branch = BranchPythonOperator(
    task_id='check_data',
    python_callable=decide_path,
)

branch >> [transform_and_load, skip_task]
```

### 대용량 배치 — TaskGroup으로 병렬화

```python
from airflow.utils.task_group import TaskGroup

with TaskGroup("process_devices", tooltip="디바이스별 병렬 처리") as tg:
    for device_id in ['DEV-001', 'DEV-002', 'DEV-003']:
        PythonOperator(
            task_id=f'process_{device_id}',
            python_callable=process_device,
            op_kwargs={'device_id': device_id},
        )

extract >> tg >> load_summary
```

---

## 5. 주기적 데이터 수집 패턴

```python
# 스케줄 표현식
schedule='@hourly'          # = '0 * * * *'
schedule='@daily'           # = '0 0 * * *'
schedule='@weekly'          # = '0 0 * * 0'
schedule='0 9 * * 1-5'      # 평일 오전 9시
schedule='*/15 * * * *'     # 15분마다
schedule=None               # 수동 트리거만

# 실행 날짜 기반 파티셔닝 (incremental 수집)
def extract_incremental(**context):
    ds = context['ds']                          # '2026-06-16' 형식
    next_ds = context['next_ds']                # 다음 실행 날짜

    query = f"""
        SELECT * FROM raw_events
        WHERE event_time >= '{ds}' AND event_time < '{next_ds}'
    """
    ...
```

---

## 6. 모니터링 & 운영

```
[Web UI 주요 기능]
  Grid View:   각 날짜×태스크의 성공/실패 색상 매트릭스
  Graph View:  DAG 의존성 시각화
  Gantt:       태스크 실행 시간 비교
  Logs:        태스크별 실시간 로그 조회
  SLA Miss:    지정 시간 내 미완료 알림

[운영 명령]
  # 특정 날짜 재실행
  airflow dags backfill sensor_data_etl -s 2026-06-01 -e 2026-06-15

  # 실패한 태스크만 재실행 (Web UI에서 "Clear" 클릭으로도 가능)
  airflow tasks clear sensor_data_etl -t transform_data -s 2026-06-16

  # DAG 수동 트리거
  airflow dags trigger sensor_data_etl
```

---

## 7. n8n vs Airflow 선택

| 기준 | Apache Airflow | n8n |
|------|---------------|-----|
| 대상 | 데이터 엔지니어 | 개발자 + 비개발자 |
| 코딩 | Python DAG 필수 | 노코드 GUI + 코드 노드 |
| 복잡한 의존성 | ✅ (태스크 그래프) | 제한적 |
| 대용량 배치 | ✅ (CeleryExecutor, K8s) | ❌ |
| 비즈니스 자동화 | ❌ (비적합) | ✅ (Slack, Gmail 등) |
| 모니터링 | ✅ (내장 UI, SLA) | 기본적 |

> **Airflow**: 데이터 파이프라인·ETL·ML 피처 생성 등 **데이터 엔지니어링** 작업
> **n8n**: 비즈니스 자동화, 알림, SaaS 연동 등 **운영 자동화** 작업

---

## 8. 관련
- [[../database/TimescaleDB]] — Airflow ETL 적재 타겟 (시계열)
- [[../database/MongoDB]] — NoSQL 적재 대상
- [[../../ai/N8N-Automation]] — n8n + LangChain + LangGraph
