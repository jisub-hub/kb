---
tags:
  - ai
  - rag
  - spring
  - llm
  - e2e
  - langchain
  - pgvector
created: 2026-06-16
---

# E2E 예시 — RAG 기반 도메인 챗봇 구축

> [!summary] 한 줄 요약
> "사내 문서·상품 매뉴얼을 학습한 챗봇" 을 RAG + Spring Boot + pgvector + LLMOps 모니터링까지 한 줄로 연결해 구현한다.

---

## 1. 전체 아키텍처

```
[문서 수집·임베딩 파이프라인]
  PDF/Markdown 파일
      │
      ▼
  DocumentLoader  (Spring AI / LangChain4j)
      │  text 청킹 (500 tokens, 50 overlap)
      ▼
  EmbeddingModel  (text-embedding-3-small)
      │  1536차원 벡터
      ▼
  pgvector        (PostgreSQL + HNSW 인덱스)

[사용자 쿼리 파이프라인]
  사용자 질문
      │
      ▼
  ① Query 임베딩
      │
      ▼
  ② pgvector 유사도 검색 (Top-K)  ←──── Hybrid: BM25 + 벡터 RRF 결합
      │  관련 문서 청크 3~5개
      ▼
  ③ Prompt 조립  (시스템 프롬프트 + 컨텍스트 + 질문)
      │
      ▼
  ④ LLM 호출  (Claude Sonnet / GPT-4o)
      │
      ▼
  ⑤ 응답 스트리밍  + 출처 인용
      │
      ▼
  ⑥ LLMOps 메트릭 기록
      (토큰수, 응답시간, 유사도점수, 사용자피드백)
```

---

## 2. 의존성

```groovy
// build.gradle
dependencies {
    // Spring AI (pgvector + OpenAI/Anthropic)
    implementation 'org.springframework.ai:spring-ai-pgvector-store-spring-boot-starter'
    implementation 'org.springframework.ai:spring-ai-anthropic-spring-boot-starter'
    implementation 'org.springframework.ai:spring-ai-openai-spring-boot-starter'

    // PDF 파싱
    implementation 'org.springframework.ai:spring-ai-pdf-document-reader'
    // Tika (docx, xlsx, html 등)
    implementation 'org.springframework.ai:spring-ai-tika-document-reader'

    // 모니터링
    implementation 'org.springframework.boot:spring-boot-starter-actuator'
    implementation 'io.micrometer:micrometer-registry-prometheus'
}
```

```yaml
# application.yml
spring:
  ai:
    anthropic:
      api-key: ${ANTHROPIC_API_KEY}
      chat:
        options:
          model: claude-sonnet-4-6
          max-tokens: 2048
    openai:
      api-key: ${OPENAI_API_KEY}
      embedding:
        options:
          model: text-embedding-3-small
    vectorstore:
      pgvector:
        index-type: HNSW
        distance-type: COSINE_DISTANCE
        dimensions: 1536
        initialize-schema: true
  datasource:
    url: jdbc:postgresql://localhost:5432/ragdb
```

---

## 3. 문서 수집 파이프라인 (Ingestion)

```java
@Service
@RequiredArgsConstructor
@Slf4j
public class DocumentIngestionService {

    private final VectorStore vectorStore;
    private final TokenTextSplitter splitter;

    // 500 tokens, 50 overlap — 실무 검증 값
    @Bean
    public TokenTextSplitter textSplitter() {
        return new TokenTextSplitter(500, 50, 5, 10000, true);
    }

    public void ingestPdf(Path filePath, String sourceLabel) {
        var reader = new PagePdfDocumentReader(
            new FileSystemResource(filePath),
            PdfDocumentReaderConfig.builder()
                .withPageTopMargin(0)
                .withPageBottomMargin(0)
                .build()
        );

        List<Document> docs = reader.get();

        // 메타데이터 추가 (출처 인용에 사용)
        docs.forEach(doc -> {
            doc.getMetadata().put("source", sourceLabel);
            doc.getMetadata().put("ingested_at", Instant.now().toString());
        });

        // 청킹
        List<Document> chunks = splitter.apply(docs);
        log.info("[INGEST] {} → {} chunks", sourceLabel, chunks.size());

        // pgvector 저장 (임베딩은 내부에서 자동 생성)
        vectorStore.add(chunks);
    }

    // 폴더 전체 일괄 수집
    public void ingestDirectory(Path dir) throws IOException {
        try (var stream = Files.walk(dir)) {
            stream.filter(p -> p.toString().endsWith(".pdf"))
                  .forEach(p -> ingestPdf(p, p.getFileName().toString()));
        }
    }
}
```

---

## 4. 검색 + 생성 서비스 (RAG Chain)

```java
@Service
@RequiredArgsConstructor
@Slf4j
public class RagChatService {

    private final ChatClient chatClient;
    private final VectorStore vectorStore;
    private final MeterRegistry meterRegistry;

    private static final String SYSTEM_PROMPT = """
        당신은 사내 문서를 기반으로 답변하는 전문 어시스턴트입니다.
        아래 제공된 컨텍스트 문서만을 근거로 정확하게 답변하세요.
        컨텍스트에 없는 내용은 "해당 내용은 문서에서 찾을 수 없습니다."라고 답하세요.
        답변 끝에 참고한 출처를 [출처: {source}] 형식으로 명시하세요.
        """;

    public ChatResponse ask(String userQuestion, String userId) {
        long startMs = System.currentTimeMillis();
        MDC.put("userId", userId);

        try {
            // ① 유사 문서 검색 (Top-5, 유사도 0.7 이상만)
            List<Document> relevantDocs = vectorStore.similaritySearch(
                SearchRequest.query(userQuestion)
                    .withTopK(5)
                    .withSimilarityThreshold(0.7f)
            );

            if (relevantDocs.isEmpty()) {
                return ChatResponse.noContext(userQuestion);
            }

            // ② 컨텍스트 조립
            String context = relevantDocs.stream()
                .map(doc -> String.format("[출처: %s]\n%s",
                    doc.getMetadata().get("source"),
                    doc.getContent()))
                .collect(Collectors.joining("\n\n---\n\n"));

            // ③ LLM 호출
            String response = chatClient.prompt()
                .system(SYSTEM_PROMPT)
                .user(u -> u.text("""
                    컨텍스트:
                    {context}

                    질문: {question}
                    """)
                    .param("context", context)
                    .param("question", userQuestion))
                .call()
                .content();

            // ④ 메트릭 기록
            long elapsed = System.currentTimeMillis() - startMs;
            recordMetrics(elapsed, relevantDocs.size(), true);

            return ChatResponse.of(response, extractSources(relevantDocs));

        } catch (Exception e) {
            recordMetrics(System.currentTimeMillis() - startMs, 0, false);
            throw e;
        } finally {
            MDC.remove("userId");
        }
    }

    // 스트리밍 응답 (SSE)
    public Flux<String> askStreaming(String userQuestion) {
        List<Document> docs = vectorStore.similaritySearch(
            SearchRequest.query(userQuestion).withTopK(5));
        String context = buildContext(docs);

        return chatClient.prompt()
            .system(SYSTEM_PROMPT)
            .user(u -> u.text("{context}\n\n질문: {question}")
                .param("context", context)
                .param("question", userQuestion))
            .stream()
            .content();
    }

    private void recordMetrics(long elapsedMs, int contextDocs, boolean success) {
        meterRegistry.timer("rag.query.duration",
            "success", String.valueOf(success))
            .record(elapsedMs, TimeUnit.MILLISECONDS);

        meterRegistry.gauge("rag.context.docs.count",
            Tags.of("query", "latest"), contextDocs);

        if (!success) {
            meterRegistry.counter("rag.query.error.total").increment();
        }
    }

    private String buildContext(List<Document> docs) {
        return docs.stream()
            .map(d -> "[출처: " + d.getMetadata().get("source") + "]\n" + d.getContent())
            .collect(Collectors.joining("\n\n---\n\n"));
    }

    private List<String> extractSources(List<Document> docs) {
        return docs.stream()
            .map(d -> (String) d.getMetadata().get("source"))
            .distinct().toList();
    }
}
```

---

## 5. REST API 엔드포인트

```java
@RestController
@RequestMapping("/api/v1/chat")
@RequiredArgsConstructor
public class ChatController {

    private final RagChatService chatService;

    // 단일 응답
    @PostMapping
    public ResponseEntity<ChatResponse> chat(@RequestBody ChatRequest request,
                                              @AuthenticationPrincipal UserPrincipal user) {
        return ResponseEntity.ok(chatService.ask(request.question(), user.getEmail()));
    }

    // 스트리밍 응답 (SSE)
    @GetMapping(value = "/stream", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
    public Flux<ServerSentEvent<String>> chatStream(@RequestParam String question) {
        return chatService.askStreaming(question)
            .map(chunk -> ServerSentEvent.<String>builder()
                .data(chunk)
                .build());
    }
}

record ChatRequest(String question) {}

record ChatResponse(String answer, List<String> sources) {
    static ChatResponse of(String answer, List<String> sources) {
        return new ChatResponse(answer, sources);
    }
    static ChatResponse noContext(String question) {
        return new ChatResponse("해당 내용은 문서에서 찾을 수 없습니다.", List.of());
    }
}
```

---

## 6. 문서 수집 스케줄러

```java
@Component
@RequiredArgsConstructor
@Slf4j
public class DocumentSyncScheduler {

    private final DocumentIngestionService ingestionService;

    // 매일 새벽 2시 — 신규 문서 동기화
    @Scheduled(cron = "0 0 2 * * *")
    public void syncDocuments() {
        log.info("[SYNC] 문서 동기화 시작");
        try {
            ingestionService.ingestDirectory(Path.of("/data/documents"));
            log.info("[SYNC] 완료");
        } catch (Exception e) {
            log.error("[SYNC] 실패: {}", e.getMessage());
        }
    }
}
```

---

## 7. LLMOps — 비용·품질 모니터링

```java
// 토큰 사용량 추적
@Component
public class LlmUsageTracker {

    private final MeterRegistry meterRegistry;

    public void record(String model, int inputTokens, int outputTokens) {
        // Prometheus Counter
        meterRegistry.counter("llm.tokens.total",
            "model", model, "type", "input").increment(inputTokens);
        meterRegistry.counter("llm.tokens.total",
            "model", model, "type", "output").increment(outputTokens);

        // 비용 추정 (Claude Sonnet 기준: input $3/1M, output $15/1M)
        double costUsd = (inputTokens / 1_000_000.0 * 3.0)
                       + (outputTokens / 1_000_000.0 * 15.0);
        meterRegistry.counter("llm.cost.usd.total", "model", model)
            .increment(costUsd);
    }
}
```

```yaml
# Grafana 대시보드 PromQL
# 1. 일별 LLM 비용
increase(llm_cost_usd_total[1d])

# 2. RAG 쿼리 응답시간 P95
histogram_quantile(0.95, rate(rag_query_duration_seconds_bucket[5m]))

# 3. 컨텍스트 없는 응답 비율 (품질 지표)
rate(rag_query_error_total[5m]) / rate(rag_query_duration_seconds_count[5m])

# 4. 평균 검색 문서 수 (너무 낮으면 임베딩 품질 문제)
rag_context_docs_count
```

---

## 8. pgvector 인덱스 최적화

```sql
-- 인덱스 생성 (데이터 적재 후, 10만건 이상일 때 효과적)
CREATE INDEX ON document_chunks
USING hnsw (embedding vector_cosine_ops)
WITH (m = 16, ef_construction = 64);

-- 검색 시 ef_search 조정 (정확도 vs 속도)
SET hnsw.ef_search = 100;   -- 높을수록 정확하지만 느림

-- 현재 청크 수 확인
SELECT COUNT(*) FROM document_chunks;
SELECT pg_size_pretty(pg_total_relation_size('document_chunks'));
```

---

## 9. 전체 흐름 요약

```
[최초 구축]
  1. PDF/문서 → DocumentIngestionService.ingestDirectory()
  2. pgvector에 청크·임베딩 저장

[런타임]
  사용자 → POST /api/v1/chat
    → RagChatService.ask()
      → pgvector 유사도 검색
      → Claude Sonnet 호출
      → 출처 포함 응답 반환
    → LlmUsageTracker.record() → Prometheus

[운영]
  - @Scheduled 매일 2시 신규 문서 동기화
  - Grafana 대시보드: 비용·P95·품질 지표 모니터링
  - 유사도 점수 0.7 미만 → "문서에서 찾을 수 없음" 반환 (Hallucination 방지)
```

---

## 10. 관련
- [[RAG]] — 청킹 전략, HNSW 파라미터, Contextual Retrieval 심화
- [[pgvector]] — 인덱스 종류(IVFFlat vs HNSW), 파라미터 튜닝
- [[LLMOps]] — 비용 추적, A/B 테스트, 프롬프트 버전 관리
- [[Agent-Harness]] — RAG를 Tool로 감싸 멀티스텝 에이전트로 확장
- [[RAG-Ingestion-Pipeline]] — §3 인입 코드의 운영 심화(증분·삭제 전파·파싱 품질)
- [[Conversational-RAG]] — 멀티턴 검색 로직 심화(후속 질문 재작성·이력 관리)
- [[../infra/database/pgvector]] — DB 레벨 설정
