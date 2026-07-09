---
tags:
  - ai
  - claude-code
  - privacy
  - data-governance
  - security
  - llm
  - compliance
created: 2026-06-24
---

# Claude Code 데이터 안전·프라이버시 (공식 정책 정리)

> [!summary] 한 줄 요약
> Claude Code(`claude -p` 포함)는 **로컬 추론이 아니다** — 프롬프트·코드·출력이 **Anthropic으로 전송**된다(TLS). 학습 사용·보관기간은 **계정 유형**이 결정한다: **상업용(Team/Enterprise/API)은 기본 학습 안 함·30일**, **개인 구독(Free/Pro/Max)은 "모델 개선" 설정 ON 시 학습·5년**. 진짜 민감 데이터는 **로컬 모델(ollama 등)** 로. 출처: [code.claude.com/docs/en/data-usage](https://code.claude.com/docs/en/data-usage), [privacy.claude.com](https://privacy.claude.com). (2026-06 확인) · 적용 사례: [[RAG-Claude-Code-Inference]]

---

## 1. 전송 — 로컬 vs 클라우드
- Claude Code는 클라이언트는 로컬에서 돌지만, **LLM 호출 시 모든 프롬프트·출력을 네트워크로 전송**(TLS 1.2+). 즉 `claude -p`로 보낸 내용은 **외부(Anthropic)로 나간다.**
- 반대로 **ollama/mlx 등 로컬 모델은 LAN 밖으로 아무것도 보내지 않는다.** ([[Obsidian-RAG-System]])
- `--tools ""`(도구 비활성화)는 *들어오는* 경로(웹검색·파일읽기)만 막는다 — *나가는* 프롬프트 전송은 못 막는다. (혼동 주의)

## 2. 모델 학습 사용 여부 (계정 유형이 결정)
| 계정 | 학습 사용 |
|---|---|
| 소비자(Free/Pro/Max) · "모델 개선" **ON** | **사용됨** — "Claude Code 사용분 포함" 명시 |
| 소비자 · "모델 개선" **OFF** | 사용 안 함 |
| 상업용(Team/Enterprise/API/3rd-party/Gov) | **기본 사용 안 함** (Dev Partner Program 옵트인 시 예외) |

- 소비자는 **선택권(옵트인 성격)** — 설정에서 끌 수 있음: [claude.ai/settings/data-privacy-controls](https://claude.ai/settings/data-privacy-controls).
- Dev Partner Program은 **1st-party API에서만**(Bedrock/Vertex 불가), 조직 admin이 명시 옵트인.

> [!info] 이 RAG 배포의 계정
> 4090 서버의 Claude Code는 **Team 구독(상업용)** — 위 표의 상업용 규칙 적용: **학습 미사용·30일 보관**. (개인 Pro/Max였다면 "모델 개선" 설정에 따라 학습·5년이 될 수 있으나 해당 없음.)

## 3. 데이터 보관기간
| 계정/설정 | 보관 |
|---|---|
| 소비자 · 모델개선 ON | **5년** |
| 소비자 · 모델개선 OFF | **30일** |
| 상업용 표준 | **30일** |
| 상업용 + ZDR | 서버측 무저장 |

## 4. Zero Data Retention (ZDR)
- **Claude for Enterprise** 자격 계정에 한해 per-org로 활성(표준 Enterprise엔 미포함, 계정팀이 승인). API도 ZDR 가능.
- ZDR 시 Anthropic API는 서버측 영구 저장 없음(인프라 디스크 암호화 AES-256).

## 5. 텔레메트리·오류·피드백 (외부 트래픽) + 차단법
| 트래픽 | 내용 | 기본(Claude API auth) | 차단 |
|---|---|---|---|
| 지표(metrics) | 지연·신뢰성·사용패턴 — **코드/파일경로 미포함** | ON | `DISABLE_TELEMETRY=1` |
| 오류(Sentry) | 운영 오류 | ON | `DISABLE_ERROR_REPORTING=1` |
| `/feedback` | **대화+코드** 전송(명시 실행 시만) | 명령 사용 시 | `DISABLE_FEEDBACK_COMMAND=1` |
| 세션 품질 설문 | 평점만(전사본은 별도 동의 시) | ON | `CLAUDE_CODE_DISABLE_FEEDBACK_SURVEY=1` |
| WebFetch 도메인 체크 | **hostname만** api.anthropic.com에 조회(경로·내용 미전송) | ON(모든 provider) | `skipWebFetchPreflight: true` |

- **한 번에 비필수 차단**: `CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC=1` (단 WebFetch 체크는 별도).
- Bedrock/Vertex/Foundry는 지표·오류·feedback이 **기본 OFF**.

## 6. 로컬 평문 전사본
- Claude Code는 세션 전사본을 `~/.claude/projects/`에 **평문으로 30일 저장**(세션 재개용). `cleanupPeriodDays`로 단축.
- 서버에서 RAG로 돌리면 **프롬프트(=노트 내용)가 그 서버 디스크에도 평문으로 남는다** → 서버 접근통제 중요.

## 7. 실무 체크리스트 (민감도별 운용)
1. **계정 확인**: `claude` → `/status`로 플랜 확인. 개인 구독이면 **"모델 개선" OFF**(학습 차단 + 30일).
2. **외부 트래픽 최소화**: 서비스 기동 env에
   `export DISABLE_TELEMETRY=1 DISABLE_ERROR_REPORTING=1 CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC=1`
3. **민감/비공개 데이터 → 로컬 모델(ollama)** 로 처리. 외부 전송 0.
4. **상업적 보장 필요** → Team/Enterprise/API(기본 무학습·30일), 더 강하게는 **Enterprise + ZDR**.
5. 서버 `~/.claude/projects` 평문 전사본 접근통제 + `cleanupPeriodDays` 단축.
6. `/feedback`은 **대화+코드를 보낸다** — 민감 환경에선 `DISABLE_FEEDBACK_COMMAND=1`.

> [!warning] 흔한 오해
> "claude -p니까 로컬이라 안전" ❌ — Claude는 **호스티드 모델**이라 내용이 Anthropic으로 간다. "로컬·무유출"이 필요하면 **ollama/mlx 로컬 모델**이 답. 단 상업용 계정이면 **학습엔 안 쓰이고 30일 후 삭제**가 기본이라, 위협모델에 따라 충분할 수 있다.

## 8. 관련 노트
- [[RAG-Claude-Code-Inference]] — 이 정책을 적용한 RAG 추론 운영(엔진 선택·grounding·워밍 풀)
- [[Obsidian-RAG-System]] — 로컬 RAG 시스템(전송 0 옵션)
- [[RAG-Security]] — RAG 보안(권한·RLS) 일반
- [[Local-LLM-API-Deployment]] — 로컬 서빙(외부 전송 없는 대안)
