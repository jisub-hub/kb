---
tags:
  - ai
  - agent
  - hermes
  - index
  - moc
  - orchestration
created: 2026-06-24
---

# Hermes 에이전트 — MOC

> [!summary] 이 폴더
> **툴 장착 자율 에이전트 런타임 Hermes**(`~/.hermes` 기준) 관련 지식을 한곳에 모은다 — 개요·만드는 법·핵심 엔진·확장(스킬·플러그인)·메모리/세션·오케스트레이션(서브에이전트 위임)·배포·설정 레퍼런스·실사용 예시. 모델(지능)과 런타임(실행 루프)을 분리하는 관점이 전제. 일반 에이전트 비교·보안 등 폴더 밖 노트는 §관련 참조.

## 🧭 시작점
- [[Hermes-Overview]] — **전체 그림**: 모델 vs 런타임 계층, 생애주기 4단계(개발→위임→자율→배포), 핵심 서브시스템 종합 ⭐
- [[Hermes-Building-Agents]] — **Agent의 용도 + 만드는 법**: 언제·왜 만드나 + 정체성·모델·툴·스킬·플러그인·메모리·배포 단계별 ⭐

## ⚙️ 핵심 엔진
- [[Hermes-Agent]] — function calling(`<tool_call>` XML)·에이전트 루프·프레임워크, 모델 vs 프레임워크 구분
- [[Hermes-Runtime-Internals]] — 자기관리 런타임: delegate_task 서브에이전트·컨텍스트 압축·tool-loop 가드레일·터미널 샌드박스

## 🧩 확장 (스킬·플러그인·툴)
- [[Hermes-Skills]] — 스킬 로딩(progressive disclosure: skills_list→skill_view)·조건부 활성·skill_manage
- [[Hermes-Plugins]] — 플러그인(커스텀 툴·훅): `~/.hermes/plugins/`·register(ctx)·확장 3종(plugin/skill/MCP)
- [[Agent-Event-Hooks]] — 생명주기 훅 3계층(Gateway/Plugin/Shell): pre_tool_call 차단·자동포맷·컨텍스트 주입
- [[Tool-Search]] — 툴 과부하 대응 점진 공개(ai/, 폴더 밖)

## 🧠 메모리 · 세션
- [[Hermes-Memory-Management]] — **메모리 관리 종합**: MEMORY.md/USER.md·세션 모델·컨텍스트 압축·멀티테넌트·poisoning ⭐
- [[Agent-Persistent-Memory]] — 영속 메모리(MEMORY.md/USER.md): frozen snapshot·prefix cache·auto-compact 안 함·큐레이션
- [[Multi-Tenant-Memory-Agent]] — 인증→Hermes→개인/팀 KB 누적: RLS 격리, memory poisoning

## 🔗 오케스트레이션 · 자율 실행
- [[Orchestrating-Claude-Code]] — 오케스트레이터(로컬 gemma)→Claude Code 위임: CLI 헤드리스(`claude -p`)/ACP/MCP, 권한·타임아웃 함정
- [[Claude-Code-Headless]] — `claude -p` 헤드리스 옵션·구독 재사용 무과금(ai/, 폴더 밖)
- [[Agent-Scheduled-Execution]] — 예약 자율 실행(cron): cronjob 툴·no-agent 모드·runaway 방지
- [[Agent-Kanban]] — durable 멀티 에이전트 보드: named 프로필 분업·dispatcher·heartbeat/reclaim

## 🚀 배포
- [[Hermes-Deployment-Surface]] — 배포 표면: OpenAI 호환 API 서버(:8642)·멀티플랫폼 게이트웨이(telegram/discord/slack/whatsapp)·세션·멀티 프로필

## 📖 레퍼런스 · 예시
- [[Hermes-Config-Reference]] — `~/.hermes` 설정·환경변수·런타임 파라미터 통합 표(출처 표기) ⭐
- [[Hermes-Usage-Examples]] — 실사용 예시: 위임·cron·게이트웨이·Kanban·**Obsidian RAG 라우팅(실운용)** ⭐

## 🔭 관련 (폴더 밖 ai/)
- [[Autonomous-Agent-Runtimes]] — 자율 런타임 비교(OpenClaw vs Hermes), 모델 vs 런타임 계층
- [[Multi-Agent-Coding-Team]] — 역할별 모델(기획/Coder/QA) 코딩 팀, 오케스트레이터=검증자
- [[Agent-Security]] — 에이전트 보안·샌드박싱·무인 실행 통제
- [[Local-LLM-API-Deployment]] — 로컬 모델 OpenAI 호환 배포(오케스트레이터 백엔드)
- [[Obsidian-RAG-System]] · [[RAG-Routing-Engine-Comparison]] — `claude -p`를 RAG 라우팅 엔진으로 쓰는 실운용
