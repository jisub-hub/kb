---
tags:
  - database
  - migration
  - flyway
  - liquibase
  - devops
created: 2026-06-17
---

# DB 마이그레이션 (Flyway / Liquibase)

> [!summary] 한 줄 요약
> 스키마 변경을 **코드처럼 버전 관리**하는 도구. 애플리케이션 배포와 함께 DB를 자동·재현 가능하게 진화시킨다. 대표 도구는 **Flyway**(SQL 중심, 단순)와 **Liquibase**(추상화·롤백 강력).

---

## 1. 왜 필요한가
- **스키마 버전 관리**: 누가·언제·왜 컬럼을 추가했는지 git 이력으로 추적. DB도 코드와 함께 진화한다.
- **팀 협업**: 각자 로컬에서 수동 `ALTER`를 치면 환경마다 스키마가 어긋남(drift). 마이그레이션 파일을 공유하면 모두 동일한 상태.
- **재현성**: 빈 DB에 마이그레이션을 순서대로 적용하면 운영과 **동일한 스키마**를 어디서나 재생성(로컬·CI·신규 리전).
- `ddl-auto: update`(Hibernate 자동 DDL)는 **운영 금지** — 변경을 추적 못 하고 데이터 손실 위험. 마이그레이션 도구가 정답.

## 2. Flyway — SQL 중심
파일명 컨벤션이 곧 버전이다.

```
src/main/resources/db/migration/
├── V1__init_schema.sql          # V + 버전 + __ + 설명
├── V2__add_orders_table.sql
├── V2.1__add_order_index.sql    # 점 표기로 마이너 버전
├── R__user_summary_view.sql     # Repeatable: 체크섬 변경 시마다 재실행 (뷰/프로시저)
└── U2__undo_orders_table.sql    # Undo (유료 Teams 기능, 비권장)
```

| 접두사 | 의미 | 실행 시점 |
|--------|------|-----------|
| `V` | Versioned | 한 번만, 버전 순서대로 |
| `R` | Repeatable | 체크섬이 바뀌면 매번 (마지막에) |
| `U` | Undo | 명시적 롤백 시 (Teams 전용) |

```sql
-- V2__add_orders_table.sql
CREATE TABLE orders (
    id          BIGINT PRIMARY KEY AUTO_INCREMENT,
    member_id   BIGINT NOT NULL,
    status      VARCHAR(20) NOT NULL,
    created_at  TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);
```

- 적용 이력은 **`flyway_schema_history`** 테이블에 체크섬과 함께 저장. 이미 적용된 파일을 수정하면 체크섬 불일치로 실패(의도된 안전장치).
- **baseline**: 이미 운영 중인 기존 DB에 Flyway를 도입할 때, 현재 상태를 `V1`로 간주하는 기준점. `flyway baseline` 또는 `baseline-on-migrate: true`.

## 3. Liquibase — 추상화 + 롤백
변경을 **changeset** 단위로 기술. XML/YAML/JSON/SQL 지원, DB 벤더 독립적.

```yaml
# db/changelog/db.changelog-master.yaml
databaseChangeLog:
  - include: { file: db/changelog/001-create-orders.yaml }
  - include: { file: db/changelog/002-add-index.yaml }
```

```yaml
# 001-create-orders.yaml
databaseChangeLog:
  - changeSet:
      id: 001-create-orders
      author: hchbae
      changes:
        - createTable:
            tableName: orders
            columns:
              - column: { name: id, type: BIGINT, autoIncrement: true,
                          constraints: { primaryKey: true } }
              - column: { name: status, type: VARCHAR(20),
                          constraints: { nullable: false } }
      rollback:                       # 명시적 롤백 정의
        - dropTable: { tableName: orders }
```

- changeset은 `id + author + 파일경로`로 식별, **`DATABASECHANGELOG`** 테이블에 기록.
- 핵심 강점은 **rollback** 1급 지원: `liquibase rollbackCount 1`, `rollbackToDate`, 태그 기반 롤백.

## 4. Flyway vs Liquibase 비교

| 항목 | Flyway | Liquibase |
|------|--------|-----------|
| 변경 기술 | **순수 SQL** (직관적) | XML/YAML/JSON/SQL (추상화) |
| 롤백 | 약함 (Undo 유료) | ⭐ 강력 (무료, 1급 기능) |
| DB 독립성 | 낮음 (SQL 직접) | ⭐ 높음 (벤더 중립 DSL) |
| 학습 곡선 | 낮음 | 다소 높음 |
| 가독성 | SQL 그대로라 명확 | XML 장황할 수 있음 |
| 적합 | SQL에 익숙·단일 DB·단순함 선호 | 멀티 DB·강한 롤백·엔터프라이즈 |

> [!tip] 한 줄 가이드
> **대부분의 백엔드 팀은 Flyway로 충분**(SQL이 곧 진실). 멀티 벤더 지원이나 자동 롤백이 필수면 Liquibase.

## 5. Spring Boot 통합
의존성만 추가하면 **애플리케이션 기동 시 자동 실행**(JPA `EntityManager` 생성보다 먼저).

```yaml
# Flyway
spring:
  flyway:
    enabled: true
    locations: classpath:db/migration
    baseline-on-migrate: true     # 기존 DB 도입 시
    validate-on-migrate: true     # 체크섬 검증
# Liquibase
  liquibase:
    change-log: classpath:db/changelog/db.changelog-master.yaml
    enabled: true
```

- Flyway/Liquibase가 클래스패스에 있으면 Spring Boot가 자동 구성. [[JPA-Hibernate]]의 `ddl-auto`는 `validate` 또는 `none`으로 두고, 스키마 변경은 전적으로 마이그레이션 도구에 맡긴다.

## 6. 운영 원칙
### 6-1. 롤백 전략: forward-only vs rollback
- **Forward-only(권장)**: 잘못된 변경도 "되돌리는 새 마이그레이션"(예: `V11__revert_v10.sql`)으로 앞으로만 전진. 운영 DB에서 자동 롤백은 데이터 유실 위험이 커 위험하다.
- Liquibase의 rollback도 **개발/스테이징 정리용**으로 주로 쓰고, 운영 데이터가 쌓인 뒤에는 신중히.

### 6-2. 무중단 배포 — Expand-Contract 패턴
구버전·신버전 앱이 **동시에 떠 있는 시간**(롤링 배포, [[../deploy/Kubernetes]])이 존재하므로, 스키마와 코드를 한 번에 바꾸면 깨진다. 변경을 단계로 쪼갠다.

```
1) Expand   : 새 컬럼/테이블을 nullable로 추가 (구버전 앱도 무시하고 동작)
2) Migrate  : 신버전 앱 배포 — 양쪽(구/신 컬럼) 모두 쓰기 (백필 포함)
3) Contract : 구버전 앱 전부 내려간 뒤, 구 컬럼 제거 + NOT NULL 제약 추가
```

- 컬럼 rename은 단번에 하지 말 것 → "새 컬럼 추가 → 양쪽 쓰기 → 백필 → 읽기 전환 → 구 컬럼 삭제"로 분해. 데이터 동기화 관점은 [[Eventual-Consistency]] 사고와 맞닿는다.

### 6-3. 큰 테이블 마이그레이션 — Lock 주의
- MySQL에서 대형 테이블 `ALTER`/인덱스 추가는 **테이블 잠금 + 장시간**(수십 분) → 서비스 정지. `ALGORITHM=INPLACE, LOCK=NONE` 가능 여부 확인, 또는 **pt-online-schema-change / gh-ost** 같은 온라인 DDL 도구 사용.
- PostgreSQL: `CREATE INDEX CONCURRENTLY`(테이블 락 회피), `ADD COLUMN`에 기본값은 11+에서 빠르지만 대량 `UPDATE` 백필은 배치로 쪼개 락·복제 지연 회피.
- `lock_timeout`/`statement_timeout`을 설정해 마이그레이션이 운영 트래픽을 무한정 막지 않게 한다.

## 7. CI/CD 연계
- 마이그레이션 파일은 **애플리케이션 코드와 같은 PR**에 포함 → 리뷰 대상으로.
- CI 파이프라인([[CICD]])에서: 빈 DB에 전체 마이그레이션 적용 → 테스트 통과 검증, `flyway validate`로 체크섬 확인.
- 배포 순서: 일반적으로 **마이그레이션 먼저(Expand) → 앱 배포**. init 컨테이너나 별도 Job으로 분리하면 여러 파드가 동시에 마이그레이션 실행하는 경쟁을 방지(Flyway는 락을 잡지만 Job 분리가 더 안전).

## 8. 안티패턴
| 안티패턴 | 왜 나쁜가 |
|----------|-----------|
| 운영 DB 수동 DDL | 이력 추적 불가, drift 발생, 재현 불가 |
| 적용된 마이그레이션 파일 수정 | 체크섬 불일치 → 환경마다 다른 스키마 |
| `ddl-auto: update` 운영 사용 | 통제 불가능한 자동 변경, 데이터 손실 |
| 하나의 거대 changeset | 부분 실패 시 롤백 어려움 → 작게 쪼갤 것 |
| 큰 테이블 즉시 `ALTER` | 락으로 인한 서비스 중단 |

## 9. 관련
- [[JPA-Hibernate]] · [[CICD]] · [[Eventual-Consistency]] · [[../deploy/Kubernetes]] · [[Connection-Pool]]
