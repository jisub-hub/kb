---
tags:
  - spring
  - security
  - checklist
  - deployment
  - hardening
created: 2026-06-16
---

# 보안 체크리스트 — Spring Boot · DB · K8s

> [!summary] 한 줄 요약
> 배포 전 놓치기 쉬운 보안 항목들. 코드 리뷰에서 잡히지 않고 프로덕션에서 발견되는 것들 위주로 정리.

---

## 1. Spring Boot 배포 체크리스트

### 설정·환경변수

```
□ 환경변수로 시크릿 분리
  - application.yml에 비밀번호, API 키 하드코딩 ❌
  - ${DB_PASSWORD}, ${JWT_SECRET} 형식으로 주입
  - .env 파일은 .gitignore에 포함

□ Spring Profiles 분리
  - application-prod.yml에 prod 전용 설정
  - spring.profiles.active=prod 환경변수로 활성화
  - dev 프로파일 설정이 prod에 적용되지 않는지 확인

□ Actuator 노출 범위 최소화
```
```yaml
management:
  endpoints:
    web:
      exposure:
        include: health,info,prometheus  # 필요한 것만
  endpoint:
    health:
      show-details: when-authorized      # 인증자만 상세 조회
  server:
    port: 9090     # 앱 포트와 분리 (내부망에서만 접근)
```

### JWT / 인증

```
□ JWT Secret은 256bit 이상 랜덤 값
  - echo "$(openssl rand -base64 32)" 으로 생성

□ Access Token 만료 짧게 (15분 ~ 1시간)
□ Refresh Token은 Redis에 저장 + 블랙리스트 관리
□ JWT payload에 민감 정보 넣지 않기 (base64 디코딩 가능)
□ algorithm: HS256 이상 (none 알고리즘 명시적 거부)

□ HTTPS 강제 (HTTP → HTTPS 리다이렉트)
```
```yaml
# Spring Security HTTPS 강제
server:
  ssl:
    enabled: true
http.requiresChannel().anyRequest().requiresSecure()
```

### 입력 검증

```
□ @Valid + @Validated 으로 컨트롤러 입력 검증
□ SQL Injection: JPA Parameterized Query 사용 (네이티브 쿼리도 :param 방식)
□ XSS: 출력 시 이스케이프, Content-Security-Policy 헤더 설정
□ 파일 업로드: 확장자 화이트리스트, Content-Type 검증, 파일명 서버 재생성
□ 파일 크기 제한:
```
```yaml
spring:
  servlet:
    multipart:
      max-file-size: 10MB
      max-request-size: 10MB
```

### 보안 헤더

```java
// Spring Security 보안 헤더 설정
http.headers(headers -> headers
    .contentSecurityPolicy(csp -> csp
        .policyDirectives("default-src 'self'; script-src 'self'"))
    .frameOptions(frame -> frame.deny())              // Clickjacking 방지
    .xssProtection(xss -> xss.enable(true))
    .httpStrictTransportSecurity(hsts -> hsts
        .includeSubDomains(true)
        .maxAgeInSeconds(31536000))
);
```

### 의존성

```bash
# 취약점 스캔 (CI에 포함)
./mvnw dependency-check:check    # OWASP Dependency-Check
./gradlew dependencyCheckAnalyze

# Snyk (별도 도구)
snyk test
```

---

## 2. 데이터베이스 보안 체크리스트

### 접근 제어

```sql
-- ✅ 앱 전용 DB 유저 생성 (최소 권한)
CREATE USER appuser WITH PASSWORD 'strongpassword';
GRANT CONNECT ON DATABASE mydb TO appuser;
GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA public TO appuser;
-- DDL 권한 없음 (ALTER, DROP, CREATE 금지)

-- ❌ 절대 금지: 앱에 superuser 또는 postgres 계정 사용

-- 읽기 전용 Replica용 유저
CREATE USER readonly WITH PASSWORD 'readonlypassword';
GRANT CONNECT ON DATABASE mydb TO readonly;
GRANT SELECT ON ALL TABLES IN SCHEMA public TO readonly;

-- Row Level Security (멀티테넌트)
ALTER TABLE orders ENABLE ROW LEVEL SECURITY;
CREATE POLICY tenant_isolation ON orders
    USING (tenant_id = current_setting('app.tenant_id')::uuid);
```

### 네트워크

```
□ DB 포트 외부 노출 금지 (VPC 내부 또는 보안 그룹으로 앱 서버만 허용)
□ pg_hba.conf: md5/scram-sha-256 인증, 특정 IP만 허용
□ SSL 연결 강제 (sslmode=require 또는 verify-full)
□ 관리자 접근은 SSH 터널 또는 VPN을 통해서만
```

```ini
# pg_hba.conf
# TYPE  DATABASE  USER      ADDRESS        METHOD
hostssl all       appuser   10.0.1.0/24   scram-sha-256
host    all       all       0.0.0.0/0     reject          # 그 외 차단
```

### 민감 데이터

```sql
-- 개인정보 컬럼 암호화 (pgcrypto)
CREATE EXTENSION pgcrypto;

-- 저장 시 암호화
INSERT INTO users (email, phone)
VALUES (pgp_sym_encrypt('user@example.com', current_setting('app.encrypt_key')),
        pgp_sym_encrypt('010-1234-5678', current_setting('app.encrypt_key')));

-- 조회 시 복호화
SELECT pgp_sym_decrypt(email::bytea, current_setting('app.encrypt_key')) AS email
FROM users;
```

### 감사 로그 (Audit)

```sql
-- 테이블 변경 이력 자동 기록
CREATE TABLE audit_log (
    id         BIGSERIAL PRIMARY KEY,
    table_name TEXT,
    operation  TEXT,
    old_data   JSONB,
    new_data   JSONB,
    changed_by TEXT DEFAULT current_user,
    changed_at TIMESTAMPTZ DEFAULT now()
);

CREATE OR REPLACE FUNCTION audit_trigger_func() RETURNS TRIGGER AS $$
BEGIN
    INSERT INTO audit_log(table_name, operation, old_data, new_data)
    VALUES (TG_TABLE_NAME, TG_OP,
            CASE WHEN TG_OP='DELETE' THEN row_to_json(OLD) ELSE NULL END,
            CASE WHEN TG_OP IN ('INSERT','UPDATE') THEN row_to_json(NEW) ELSE NULL END);
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER orders_audit
AFTER INSERT OR UPDATE OR DELETE ON orders
FOR EACH ROW EXECUTE FUNCTION audit_trigger_func();
```

---

## 3. K8s 보안 체크리스트

### Pod / Container

```yaml
# SecurityContext — 최소 권한 컨테이너
securityContext:
  runAsNonRoot: true          # root로 실행 금지
  runAsUser: 1000
  runAsGroup: 1000
  readOnlyRootFilesystem: true  # 파일시스템 읽기 전용
  allowPrivilegeEscalation: false
  capabilities:
    drop:
      - ALL                   # 모든 Linux capability 제거
    add:
      - NET_BIND_SERVICE      # 필요한 것만 추가
```

### Secret 관리

```
□ Secret을 YAML에 평문으로 커밋하지 않음
  (base64는 암호화가 아님 — 누구나 디코딩 가능)

□ 권장: External Secrets Operator + AWS Secrets Manager / Vault
□ 최소: Sealed Secrets (공개키로 암호화된 YAML)
□ Kubernetes Secret을 환경변수로 주입 (파일마운트가 더 안전)
```

```yaml
# External Secrets Operator 예시
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: db-secret
spec:
  secretStoreRef:
    name: aws-secretsmanager
    kind: ClusterSecretStore
  target:
    name: db-secret
  data:
    - secretKey: DB_PASSWORD
      remoteRef:
        key: prod/myapp/db
        property: password
```

### RBAC

```
□ 앱 ServiceAccount에 cluster-admin 절대 부여 금지
□ 필요한 리소스·동사만 명시 (최소 권한)
□ default ServiceAccount 사용 금지 → 전용 SA 생성
□ 분기별 권한 감사:
```
```bash
kubectl get rolebindings,clusterrolebindings -A \
  -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.roleRef.name}{"\n"}{end}'
```

### Network Policy

```
□ Default Deny All 적용 (namespace 단위)
□ 서비스 간 통신 화이트리스트
□ DB 접근은 앱 파드에서만 허용
□ 외부 Egress는 필요한 IP/도메인만 허용
→ 상세 설정: [[Kubernetes#7. Network Policy]]
```

### 이미지

```
□ 베이스 이미지: distroless 또는 alpine (최소 레이어)
□ latest 태그 금지 → sha256 digest 또는 버전 태그 명시
□ CI에서 이미지 취약점 스캔:
```
```bash
trivy image myorg/myapp:1.2.0
# 또는 CI에서
docker scan myorg/myapp:1.2.0
```

```
□ 프라이빗 레지스트리 → imagePullSecrets 설정
□ Pod의 imagePullPolicy: Always (최신 이미지 보장)
```

---

## 4. 인프라 공통

```
□ 시크릿 로테이션 계획 수립 (DB 비밀번호, API 키 주기적 변경)
□ 로그에 민감 정보 노출 여부 확인
  (비밀번호, 카드번호, 주민번호 등 마스킹)
□ 에러 응답에 스택트레이스 노출 금지 (prod 환경)
□ 의존성 업데이트 주기 설정 (Dependabot/Renovate 자동화)
□ 모든 외부 통신 TLS 1.2+ 강제
□ 핵심 시스템 침투 테스트 주기적 수행
```

```yaml
# Spring Boot — prod에서 에러 상세 숨기기
server:
  error:
    include-stacktrace: never
    include-message: never
    include-binding-errors: never
```

---

## 5. 관련
- [[security/_index]] — 암호화, 웹 공격(XSS·SQLi·CSRF) 심화
- [[RBAC]] — K8s RBAC 상세
- [[Network-Policy]] — K8s 네트워크 격리
- [[../spring/observability/AOP-Logging]] — 감사 로그로 접근 이력 추적
- [[DB-Deployment-Decision]] — DB 보안 설정 결정 트리
