# Knowledge Base

AI/LLM 애플리케이션 개발, 인프라, 백엔드 프로그래밍 관련 기술 지식을 정리한 문서 모음입니다.

## 구성

```
kb/
├── ai/          # LLM, RAG, 에이전트, 프롬프트 엔지니어링, 추론 최적화
│   ├── hermes/  # Hermes 에이전트 런타임
│   └── openclaw/ # OpenClaw 자율 에이전트
├── hardware/    # GPU 워크스테이션, 엣지 AI, 서버
├── infra/       # Kubernetes, 데이터베이스, 네트워크, 클라우드, 모니터링
└── programming/ # Java/Spring, Python, IoT, 보안, 소프트웨어 공학
```

## 주요 문서

**RAG 시스템 구축**
- [RAG 기초](ai/RAG.md) · [청킹 전략](ai/Chunking-Strategies.md) · [리랭킹](ai/Reranking-Retrieval.md) · [쿼리 변환](ai/Query-Transformation.md)
- [RAG 실패 디버깅](ai/RAG-Failure-Debugging.md) · [Agentic RAG](ai/Agentic-RAG.md) · [평가(Evals)](ai/Evaluation.md)

**AI 에이전트**
- [하네스 엔지니어링](ai/Harness-Engineering.md) · [에이전트 보안](ai/Agent-Security.md) · [MCP & Skill](ai/MCP-Skill.md)

**프롬프트 & 운영**
- [프롬프트 엔지니어링](ai/Prompt-Engineering.md) · [프롬프트 캐싱](ai/Prompt-Caching.md) · [LLMOps](ai/LLMOps.md)

**LLM 인프라**
- [vLLM](ai/vLLM.md) · [로컬 LLM 배포](ai/Local-LLM-API-Deployment.md) · [GPU 의사결정 가이드](ai/GPU-Inference-Decision-Guide.md)

## CLI 사용법

repo를 클론하면 포함된 `kb` 스크립트로 로컬에서 질문할 수 있습니다.

```bash
git clone https://github.com/jisub-hub/kb.git
cd kb
./kb "RAG 청킹 전략 뭐가 좋아?"
./kb "프롬프트 캐싱 어떻게 적용해?"
```

> LLM 추론 서버가 필요합니다. 기본값은 `http://localhost:11434` (Ollama)입니다.

## 문서 형식

각 문서는 Obsidian 마크다운 형식으로 작성되어 있습니다.
- frontmatter에 `tags`, `created` 포함
- `[[wikilink]]`로 관련 문서 연결
- `_index.md`가 각 섹션의 목차(MOC) 역할
