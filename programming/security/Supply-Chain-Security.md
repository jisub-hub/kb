---
tags:
  - security
  - supply-chain
  - sbom
  - container
  - devsecops
created: 2026-06-16
---

# 공급망 보안 (Supply Chain Security)

> [!summary] 한 줄 요약
> **소프트웨어 공급망 공격**(의존성·컨테이너 이미지·CI/CD 파이프라인 탈취)은 2021년 이후 급증하는 공격 벡터. SBOM 생성, 이미지 서명, 의존성 자동 스캔으로 대응한다.

---

## 1. 공급망 공격 사례

| 사고 | 연도 | 방식 |
|------|------|------|
| SolarWinds | 2020 | 빌드 파이프라인 침투 → 서명된 악성 업데이트 배포 |
| Log4Shell (CVE-2021-44228) | 2021 | 오픈소스 의존성 취약점 |
| XZ Utils (CVE-2024-3094) | 2024 | 오픈소스 메인테이너 사회공학적 탈취 |
| npm event-stream | 2018 | 인기 패키지 탈취 후 악성코드 삽입 |

---

## 2. SBOM (Software Bill of Materials)

**SBOM**: 소프트웨어에 포함된 모든 컴포넌트와 의존성의 목록. 취약점 영향 범위 파악에 필수.

### Syft로 SBOM 생성

```bash
# 설치
brew install syft   # 또는 curl -sSfL https://raw.githubusercontent.com/anchore/syft/main/install.sh | sh

# 컨테이너 이미지에서 SBOM 생성
syft myapp:1.2.3 -o spdx-json > sbom.spdx.json     # SPDX 형식
syft myapp:1.2.3 -o cyclonedx-json > sbom.cdx.json  # CycloneDX 형식

# Maven/Gradle 프로젝트
syft dir:/path/to/project -o cyclonedx-json > sbom.cdx.json

# SBOM 내용 확인
cat sbom.spdx.json | jq '.packages[] | {name: .name, version: .versionInfo}'
```

### Maven SBOM 플러그인

```xml
<!-- pom.xml -->
<plugin>
    <groupId>org.cyclonedx</groupId>
    <artifactId>cyclonedx-maven-plugin</artifactId>
    <version>2.8.0</version>
    <executions>
        <execution>
            <phase>package</phase>
            <goals><goal>makeAggregateBom</goal></goals>
        </execution>
    </executions>
</plugin>
```

```bash
mvn cyclonedx:makeAggregateBom   # target/bom.json 생성
```

---

## 3. Trivy — 취약점 스캔

```bash
# 설치
brew install trivy

# 컨테이너 이미지 스캔
trivy image myapp:1.2.3
trivy image --severity HIGH,CRITICAL myapp:1.2.3   # 고위험 이상만
trivy image --exit-code 1 --severity CRITICAL myapp:1.2.3  # CRITICAL 있으면 CI 실패

# 파일시스템 스캔 (소스코드 의존성)
trivy fs .
trivy fs --scanners vuln,secret .   # 취약점 + 하드코딩된 비밀값 탐지

# SBOM 기반 스캔
trivy sbom sbom.cdx.json

# K8s 클러스터 전체 스캔
trivy k8s --report summary cluster
trivy k8s --namespace production all

# GitLab CI 연동
trivy image --format gitlab myapp:1.2.3 > gl-container-scanning-report.json
```

### CI/CD 통합 (GitLab)

```yaml
# .gitlab-ci.yml
container-scan:
  stage: test
  image: aquasec/trivy:latest
  script:
    - trivy image --exit-code 0 --severity HIGH,CRITICAL
        --format json -o trivy-report.json $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA
    - trivy image --exit-code 1 --severity CRITICAL
        $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA   # CRITICAL 있으면 실패
  artifacts:
    reports:
      container_scanning: trivy-report.json
  allow_failure: false
```

---

## 4. 컨테이너 이미지 서명 — Cosign / Sigstore

```bash
# 설치
brew install cosign

# 키 쌍 생성
cosign generate-key-pair              # cosign.key + cosign.pub 생성

# 이미지 서명 (빌드 후 즉시)
cosign sign --key cosign.key myregistry.io/myapp:1.2.3

# 서명 검증 (배포 전)
cosign verify --key cosign.pub myregistry.io/myapp:1.2.3

# Keyless 서명 (OIDC — GitHub Actions에서 권장)
cosign sign --oidc-issuer https://token.actions.githubusercontent.com \
  myregistry.io/myapp:1.2.3

# SBOM을 이미지에 첨부
cosign attach sbom --sbom sbom.spdx.json myregistry.io/myapp:1.2.3
cosign verify-attestation --key cosign.pub myregistry.io/myapp:1.2.3
```

### GitHub Actions 키리스 서명

```yaml
# .github/workflows/build.yml
- name: Sign container image
  uses: sigstore/cosign-installer@v3
- run: |
    cosign sign --yes ${{ env.IMAGE }}:${{ github.sha }}
  env:
    COSIGN_EXPERIMENTAL: 1   # Rekor 투명성 로그에 기록
```

---

## 5. 의존성 자동 업데이트

### Dependabot (GitHub)

```yaml
# .github/dependabot.yml
version: 2
updates:
  - package-ecosystem: "maven"
    directory: "/"
    schedule:
      interval: "weekly"
    labels: ["dependencies", "java"]
    open-pull-requests-limit: 10
    ignore:
      - dependency-name: "com.example:legacy-lib"  # 특정 의존성 제외

  - package-ecosystem: "docker"
    directory: "/"
    schedule:
      interval: "daily"    # 베이스 이미지는 더 자주

  - package-ecosystem: "github-actions"
    directory: "/"
    schedule:
      interval: "weekly"
```

### Renovate (더 강력한 대안)

```json
// renovate.json
{
  "extends": ["config:base"],
  "packageRules": [
    {
      "matchUpdateTypes": ["minor", "patch"],
      "automerge": true           // 마이너/패치는 자동 머지
    },
    {
      "matchPackagePatterns": ["^org.springframework"],
      "groupName": "spring packages",  // Spring 관련 PR 묶음
      "schedule": ["every weekend"]
    }
  ],
  "vulnerabilityAlerts": {
    "enabled": true,
    "labels": ["security"]
  }
}
```

---

## 6. CI/CD 파이프라인 보안 체크리스트

```yaml
# 빌드 파이프라인 최소 권한 원칙
pipeline:
  # ✅ 해야 할 것
  - 빌드 환경은 읽기 전용 파일시스템
  - 비밀값은 환경변수가 아닌 Vault/Secrets Manager에서
  - 모든 외부 액션/이미지 버전 pinning (해시로 고정)
  - 아티팩트 무결성 검증 (sha256 체크섬)
  - OIDC 기반 클라우드 인증 (장기 자격증명 금지)

  # ❌ 하지 말 것
  - "latest" 태그 이미지 사용
  - 빌드 스크립트에서 curl | bash 패턴
  - 불필요한 네트워크 아웃바운드 허용
  - 빌드 캐시에 비밀값 저장
```

```yaml
# GitHub Actions: 해시로 버전 고정
- uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683  # v4.2.2
# 태그가 아닌 커밋 해시 → 태그 이동 공격 방지
```

---

## 7. 관련
- [[Web-Attacks]] · [[Network-Attacks]] · [[Cryptography]]
- [[../../infra/k8s/Security-Context]] · [[../../infra/k8s/RBAC]]
- [[../../infra/ssh-cicd/GitLab-CICD]]
