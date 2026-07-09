---
tags:
  - ai
  - llm
  - anthropic
  - claude
created: 2026-06-15
---

# LLM (Large Language Model)

> [!summary] 한 줄 요약
> 방대한 텍스트로 학습해 **다음 토큰을 확률적으로 예측**하는 모델. 분류·요약·추출·대화·코드 생성 등 자연어 작업을 범용으로 처리한다. 애플리케이션에서는 **API 호출**로 사용하며, 비결정적·확률적이라는 점이 전통적 코드와 근본적으로 다르다.

---

## 1. 핵심 개념
| 개념 | 설명 |
|------|------|
| **토큰(token)** | 텍스트의 최소 단위(단어 조각). 입력/출력 비용·길이의 기준. 한국어/코드는 영어보다 토큰이 많이 든다 |
| **컨텍스트 윈도우** | 한 번에 모델이 볼 수 있는 토큰 수 (예: Claude Opus 4.8 = 1M) |
| **max_tokens** | 응답으로 생성할 최대 토큰 (하드 상한) |
| **temperature/effort** | 출력의 다양성·사고 깊이 조절 (모델별 상이) |
| **stop reason** | 생성 종료 이유: `end_turn`, `max_tokens`, `tool_use`, `refusal` 등 |
| **stateless** | API는 상태가 없다 → **매 요청에 전체 대화 히스토리를 다시 보낸다** |

> [!warning] 비결정성
> 같은 입력도 출력이 달라질 수 있다. 검증/파싱/재시도를 전제로 설계하고, 구조화 출력으로 형식을 강제하라.

---

## 2. 모델 선택 (Claude 기준, 2026-06)
| 모델 | ID | 컨텍스트 | 특징 |
|------|-----|----------|------|
| **Opus 4.8** ⭐ | `claude-opus-4-8` | 1M | 가장 강력·자율적. 장기 에이전트/지식작업 기본값 |
| Sonnet 4.6 | `claude-sonnet-4-6` | 1M | 속도·지능 균형. 고볼륨 프로덕션 |
| Haiku 4.5 | `claude-haiku-4-5` | 200K | 가장 빠르고 저렴. 단순/속도 민감 작업 |

> [!tip]
> 기본은 **Opus 4.8**. 비용/지연이 중요하고 작업이 단순하면 Sonnet/Haiku로 내린다. 모델 ID·가격·능력은 빠르게 바뀌니 `/claude-api`로 최신 확인.

### Adaptive Thinking & Effort
- `thinking: {type: "adaptive"}` — 모델이 **스스로** 사고량 결정 (고정 budget_tokens는 폐기됨)
- `output_config: {effort: "..."}` — `low | medium | high | xhigh | max`. 코딩/에이전트는 `high`~`xhigh` 권장

---

## 3. 호출 패턴 (Spring Boot + Anthropic Java SDK)
```groovy
implementation("com.anthropic:anthropic-java:2.34.0")
```
```java
@Service
public class LlmService {
    private final AnthropicClient client = AnthropicOkHttpClient.fromEnv(); // ANTHROPIC_API_KEY

    public String ask(String userText) {
        MessageCreateParams params = MessageCreateParams.builder()
            .model("claude-opus-4-8")                 // 문자열로 모델 지정
            .maxTokens(16000L)
            .system("You are a helpful assistant. Answer concisely in Korean.")
            .addUserMessage(userText)
            .build();

        Message res = client.messages().create(params);
        // content는 블록 리스트 — text 블록만 추출
        return res.content().stream()
                  .flatMap(b -> b.text().stream())
                  .map(TextBlock::text)
                  .collect(Collectors.joining());
    }
}
```
> ⚠️ 긴 입력/출력이 예상되면 **스트리밍**(`createStreaming`)을 써서 HTTP 타임아웃을 피한다.

### 구조화 출력 (형식 보장)
```java
// output_config.format 으로 JSON 스키마를 강제 → 파싱 안전
// (Java SDK는 POJO 기반 .outputConfig(MyDto.class) 오버로드 제공)
```
프롬프트로 "JSON으로 답해" 하기보다 **구조화 출력**으로 스키마를 강제하는 것이 안정적이다. 프리필(마지막 assistant 턴 강제)은 최신 모델에서 막혀 있으니 쓰지 말 것.

---

## 4. 비용 / 성능 레버
- **토큰을 줄여라**: 불필요한 컨텍스트 제거, 요약, 청킹 ([[RAG]]).
- **프롬프트 캐싱**: 반복되는 큰 프리픽스(시스템 프롬프트·문서)를 캐시 → 최대 ~90% 절감. → [[Prompt-Engineering]]
- **모델 다운시프트**: 단순 작업은 Haiku/Sonnet.
- **배치 API**: 지연에 둔감한 대량 작업은 50% 저렴.
- **토큰 카운팅**: 추정은 `tiktoken` 금지(부정확) → 제공자 토큰 카운트 API 사용.

---

## 5. 실패/주의 모드
- **환각(hallucination)**: 그럴듯한 거짓. 근거가 필요하면 [[RAG]] + 인용.
- **거부(refusal)**: `stop_reason == "refusal"` 분기 처리.
- **max_tokens 절단**: 응답이 잘리면 한도 상향 또는 스트리밍.
- **프롬프트 인젝션**: 외부/사용자 텍스트가 지시를 덮어쓸 수 있음 → 신뢰 경계 분리, 운영 지시는 system 채널로.

## 6. 관련
- [[RAG]] · [[Agent-Harness]] · [[Prompt-Engineering]] · [[REST-API]] · [[Redis]]
