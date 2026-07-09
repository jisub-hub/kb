---
tags:
  - cloud
  - serverless
  - lambda
  - aws
  - api-gateway
  - eventbridge
  - faas
created: 2026-06-16
---

# Serverless & Lambda — 서버리스 컴퓨팅

> [!summary] 한 줄 요약
> **Serverless(FaaS)**는 서버 관리 없이 함수 단위로 실행되는 클라우드 모델. **AWS Lambda**가 대표적. API Gateway로 HTTP 요청을 받거나 EventBridge로 이벤트·스케줄을 트리거한다. 단, **15분 실행 제한**이 있어 장시간 배치 작업에는 부적합.

---

## 1. Serverless의 유래

```
2006 — Amazon EC2: 가상 서버를 분 단위 사용
2012 — AWS OpsWorks: 서버 설정 자동화 (여전히 서버 관리 필요)

2014 — AWS Lambda 출시 (re:Invent 2014)
       핵심 아이디어: "함수 하나" 단위로 배포, 요청 올 때만 실행
       이전까지의 고민 해결:
         - 서버 프로비저닝 불필요
         - 트래픽 0이면 비용 0 (idle 서버 비용 없음)
         - 자동 스케일링 (동시 요청 수만큼 인스턴스 생성)

2016 — Google Cloud Functions, Azure Functions 출시
2018 — Serverless Framework, AWS SAM 등 도구 생태계 성숙

[FaaS(Function as a Service) = Serverless의 실행 모델]
  IaaS (EC2): 서버 전체 관리
  PaaS (Heroku): 앱 관리
  FaaS (Lambda): 함수 실행만 관리
  SaaS: 소비만
```

---

## 2. Lambda 장단점

```
장점:
  ✅ 서버 관리 불필요 — OS 패치, 용량 계획 없음
  ✅ 사용한 만큼만 비용 — 요청 100만 건/월 무료 (Free Tier)
  ✅ 자동 스케일링 — 동시 요청 수천 건 자동 처리
  ✅ 빠른 배포 — zip/컨테이너 이미지 업로드로 즉시 반영
  ✅ 관리형 인프라 — 가용성, 내결함성 AWS가 보장
  ✅ 이벤트 중심 — API Gateway, S3, SQS, EventBridge 등 130+ 트리거

단점 (제약):
  ❌ 실행 시간 최대 15분
       → 장시간 배치, ETL, ML 학습 불가
       → 동영상 트랜스코딩 (ffmpeg 처리) 불가
       → 대용량 파일 처리 불가

  ❌ Cold Start (콜드 스타트)
       → 호출이 없다가 첫 요청 시 컨테이너 초기화 시간 (수백ms ~ 수초)
       → JVM 기반(Java) 특히 느림 (SnapStart로 완화 가능)
       → 해결: Provisioned Concurrency (미리 워밍, 추가 비용)

  ❌ 상태 없음 (Stateless)
       → 함수 인스턴스 간 메모리 공유 불가
       → 상태는 DynamoDB, S3, ElastiCache에 외부 저장

  ❌ 로컬 스토리지 제한
       → /tmp: 최대 10GB (기본 512MB), 임시만 가능
       → 큰 파일 처리: S3 통해야 함

  ❌ 네트워크 제한
       → VPC Lambda: ENI 생성 지연 (개선됨, 약 수 ms)
       → 동시성 제한: 리전당 기본 1,000 (증설 신청 가능)

  ❌ 베더 종속성
       → AWS 특화 트리거/SDK (이식성 낮음)
```

---

## 3. Lambda 기본 구조

```python
# handler.py
import json

def handler(event: dict, context) -> dict:
    """
    event:   트리거 소스별 다른 구조 (API GW, S3, SQS 등)
    context: Lambda 런타임 정보 (function_name, memory_limit, etc.)
    """
    # API Gateway 프록시 통합 시 event 구조
    body = json.loads(event.get('body', '{}'))
    path = event.get('path', '/')
    method = event.get('httpMethod', 'GET')
    headers = event.get('headers', {})

    # 비즈니스 로직
    result = process(body)

    # API Gateway 응답 형식
    return {
        'statusCode': 200,
        'headers': {
            'Content-Type': 'application/json',
            'Access-Control-Allow-Origin': '*',
        },
        'body': json.dumps(result)
    }
```

### 배포 (AWS SAM)

```yaml
# template.yaml
AWSTemplateFormatVersion: '2010-09-09'
Transform: AWS::Serverless-2016-10-31

Globals:
  Function:
    Runtime: python3.12
    MemorySize: 512
    Timeout: 30
    Environment:
      Variables:
        DB_URL: !Ref DbUrl

Resources:
  SensorProcessorFunction:
    Type: AWS::Serverless::Function
    Properties:
      Handler: handler.handler
      CodeUri: src/
      Events:
        ApiPost:
          Type: Api
          Properties:
            Path: /sensors
            Method: post
```

---

## 4. API Gateway + Lambda

```
[HTTP API (권장, 저렴)]
  클라이언트 → API Gateway (HTTP API) → Lambda
  
  특징:
  - REST API보다 70% 저렴
  - JWT 인증 내장
  - CORS 설정 간단
  - 지연: ~1ms 오버헤드

[REST API (기능 풍부)]
  클라이언트 → API Gateway (REST API) → Lambda
  
  특징:
  - API Key 관리, Usage Plan, Throttling
  - Request/Response 변환 (Mapping Templates)
  - WAF 통합
```

```yaml
# SAM — HTTP API 정의
Resources:
  SensorApi:
    Type: AWS::Serverless::HttpApi
    Properties:
      StageName: prod
      CorsConfiguration:
        AllowOrigins: ["https://app.example.com"]
        AllowMethods: ["GET", "POST"]

  GetSensorFunction:
    Type: AWS::Serverless::Function
    Properties:
      Handler: sensors.get_handler
      Events:
        GetSensor:
          Type: HttpApi
          Properties:
            ApiId: !Ref SensorApi
            Path: /sensors/{deviceId}
            Method: GET

  PostSensorFunction:
    Type: AWS::Serverless::Function
    Properties:
      Handler: sensors.post_handler
      Events:
        PostSensor:
          Type: HttpApi
          Properties:
            ApiId: !Ref SensorApi
            Path: /sensors
            Method: POST
```

```python
# sensors.py — 경로별 핸들러
def get_handler(event, context):
    device_id = event['pathParameters']['deviceId']
    # DynamoDB에서 조회
    data = dynamodb.get_item(...)
    return {'statusCode': 200, 'body': json.dumps(data)}

def post_handler(event, context):
    body = json.loads(event['body'])
    # 유효성 검사 + 저장
    ...
    return {'statusCode': 201, 'body': json.dumps({'id': new_id})}
```

---

## 5. EventBridge + Lambda (배치·스케줄)

```
[EventBridge = AWS의 이벤트 버스]
  - 규칙(Rule): 이벤트 패턴 매칭 또는 cron 스케줄
  - 타겟(Target): Lambda, SQS, Step Functions, ECS 등
  - 15분 제한 우회: Step Functions + Lambda 체인 또는 ECS Fargate로 위임
```

```yaml
# SAM — 스케줄 트리거
Resources:
  DailySummaryFunction:
    Type: AWS::Serverless::Function
    Properties:
      Handler: batch.daily_summary
      Timeout: 900                      # 최대 15분
      MemorySize: 1024
      Events:
        DailySchedule:
          Type: ScheduleV2             # EventBridge Scheduler
          Properties:
            ScheduleExpression: "cron(0 1 * * ? *)"   # 매일 01:00 UTC
            RetryPolicy:
              MaximumRetryAttempts: 2
```

```python
# batch.py
import boto3
from datetime import datetime, timedelta

def daily_summary(event, context):
    """
    매일 1시: 전날 센서 데이터 집계 → S3 리포트 저장
    """
    yesterday = (datetime.utcnow() - timedelta(days=1)).date()

    # TimescaleDB 쿼리 (VPC Lambda + RDS Proxy 또는 직접 접속)
    stats = query_daily_stats(str(yesterday))

    # S3에 리포트 저장
    s3 = boto3.client('s3')
    s3.put_object(
        Bucket='reports-bucket',
        Key=f'daily/{yesterday}.json',
        Body=json.dumps(stats),
        ContentType='application/json',
    )
    print(f"[일일 집계] {yesterday} 완료, {len(stats)} 디바이스")
```

### 15분 제한 우회 — Step Functions + ECS Fargate

```
[장시간 작업 처리 패턴]

  EventBridge (cron)
      ↓
  Step Functions (워크플로우 조율)
      ├── Lambda: 작업 분할 (chunk 목록 생성)
      ├── Map State: 각 chunk를 ECS Fargate Task로 병렬 실행
      └── Lambda: 결과 합산 + 알림

  ECS Fargate: 제한 없는 실행 시간, 컨테이너 기반
               Lambda 15분 제한 없음
```

---

## 6. Lambda 성능 최적화

```python
# 핸들러 바깥에서 초기화 (Cold Start 시 1회만 실행)
import boto3
import psycopg2

# 모듈 레벨 초기화: 재사용됨 (Warm 호출 시 스킵)
dynamodb = boto3.resource('dynamodb')
table = dynamodb.Table('sensors')

# DB 연결: RDS Proxy를 통해 커넥션 풀 관리
conn = psycopg2.connect(os.environ['DB_URL'])

def handler(event, context):
    # 이미 초기화된 conn 재사용 → Cold Start 이후는 빠름
    cursor = conn.cursor()
    ...
```

```yaml
# Provisioned Concurrency (콜드 스타트 완전 제거, 추가 비용)
AutoScalingTarget:
  Type: AWS::ApplicationAutoScaling::ScalableTarget
  Properties:
    MinCapacity: 5
    MaxCapacity: 100
    ResourceId: !Sub function:${SensorProcessorFunction}:prod
    ScalableDimension: lambda:function:ProvisionedConcurrency
```

---

## 7. 비용 계산

```
[Lambda 가격 (2026 기준, 서울 리전 ap-northeast-2)]
  요청:   $0.20 / 100만 건
  실행:   $0.0000166667 / GB-초 (512MB × 1s = $0.0000083)
  무료:   100만 요청/월 + 400,000 GB-초/월

[예시]
  센서 데이터 수신 API:
  - 100만 요청/일 × 30일 = 3천만 요청/월
  - 평균 실행 시간: 100ms, 메모리: 512MB
  
  요청 비용: (3,000 - 1) × $0.20 = $599.8/월 (무료 100만 제외)
  실행 비용: 3,000만 × 0.1s × 0.5GB × $0.0000166667 ≈ $25/월
  합계:     약 $624/월
  
  ← 이 규모면 ECS Fargate나 EC2 Auto Scaling이 더 저렴할 수 있음

[Lambda 적합 규모]
  ✅ 간헐적 트래픽 (하루 10만 건 이하)
  ✅ 트래픽 예측 불가 (이벤트 기반)
  ❌ 지속적 고트래픽 (ECS/EC2가 유리)
```

---

## 8. 관련
- [[Object-Storage]] — Lambda + S3 presigned URL 패턴
- [[../data-pipeline/Apache-Airflow]] — 장시간 ETL (Lambda 15분 제한 우회)
- [[../container/Zero-Downtime-Deployment]] — 컨테이너 기반 배포와 비교
