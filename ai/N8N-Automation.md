---
tags:
  - ai
  - n8n
  - langchain
  - langgraph
  - workflow-automation
  - agent
created: 2026-06-16
---

# n8n · LangChain · LangGraph — AI 워크플로우 자동화

> [!summary] 한 줄 요약
> **n8n**은 노코드·비주얼 워크플로우 자동화 (Node-RED의 기업용 버전). **LangChain**은 LLM 파이프라인 조합 프레임워크. **LangGraph**는 LangChain 위에서 **상태 기계(State Machine) 기반 멀티 에이전트 워크플로우**를 구현한다.

---

## 1. 도구 비교

```
[n8n]
  대상: 개발자 + 비개발자
  방식: GUI 노드 드래그 + 코드 노드 혼합
  강점: 300+ 서비스 커넥터 (Slack, Gmail, GitHub, DB, HTTP...)
        self-hosted (데이터 자체 보관)
  적합: 비즈니스 자동화, 알림, 데이터 파이프라인

[LangChain]
  대상: 개발자 (Python/JS)
  방식: 컴포넌트 체인 (Chain, Agent, Tool, Memory)
  강점: LLM 추상화, 프롬프트 템플릿, 벡터스토어 통합
  적합: LLM 앱 빌딩 블록, RAG 파이프라인, 단순 에이전트

[LangGraph]
  대상: 개발자 (Python)
  방식: 방향성 그래프 (노드=함수, 엣지=조건 분기)
  강점: 상태 유지, 루프, 병렬 처리, 사람-인-루프(HITL)
  적합: 복잡한 멀티스텝 에이전트, 워크플로우 제어 필요
```

---

## 2. n8n

### 특징

```
- Self-hosted: Docker로 5분 설치, 데이터 외부 전송 없음
- 코드 노드: JavaScript/Python 인라인 실행
- AI 에이전트 노드: OpenAI, Claude, 로컬 LLM 연동
- 트리거: Webhook, Cron, 이벤트 (Kafka, DB 변경 감지)
- 에러 처리: 재시도, 오류 브랜치 내장
- 버전 관리: 워크플로우 변경 이력 관리
```

### 설치 (Docker)

```bash
docker run -d \
  --name n8n \
  -p 5678:5678 \
  -e N8N_BASIC_AUTH_ACTIVE=true \
  -e N8N_BASIC_AUTH_USER=admin \
  -e N8N_BASIC_AUTH_PASSWORD=changeme \
  -v n8n_data:/home/node/.n8n \
  n8nio/n8n

# http://localhost:5678
```

### 대표 워크플로우 패턴

```
[IoT 알림 자동화]
  Webhook 수신 (센서 데이터)
    → IF: temperature > 35
      → Telegram 메시지 발송
      → TimescaleDB INSERT
      → Slack 알림

[CCTV 이벤트 처리]
  HTTP POST (MediaMTX 이벤트 훅)
    → AI 분석 API 호출 (YOLO 감지 결과)
    → IF: 사람 감지
      → 녹화 저장 경로 기록
      → 담당자 이메일 발송
      → 사건 DB 기록

[LLM 파이프라인]
  새 지원서 이메일 수신 (Gmail 트리거)
    → 이력서 파일 다운로드
    → AI Agent 노드 (Claude API로 요약)
    → Notion 데이터베이스에 요약 저장
    → HR Slack 채널 알림
```

### AI Agent 노드 설정

```json
{
  "type": "n8n-nodes-langchain.agent",
  "parameters": {
    "model": "claude-opus-4-8",
    "systemPrompt": "당신은 IoT 데이터 분석 전문가입니다.",
    "tools": [
      "httpRequest",     // 외부 API 호출
      "code",            // Python/JS 코드 실행
      "calculator"       // 수식 계산
    ]
  }
}
```

---

## 3. LangChain

### 핵심 컴포넌트

```python
from langchain_anthropic import ChatAnthropic
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.output_parsers import StrOutputParser

# ── 기본 체인 ──────────────────────────────────────────────────
llm = ChatAnthropic(model="claude-sonnet-4-6")

prompt = ChatPromptTemplate.from_messages([
    ("system", "당신은 IoT 데이터 분석가입니다."),
    ("human", "다음 센서 데이터를 분석해주세요: {data}")
])

chain = prompt | llm | StrOutputParser()

result = chain.invoke({"data": "온도: 38.5°C, 습도: 80%"})
```

### RAG 파이프라인 (LangChain)

```python
from langchain_community.vectorstores import PGVector
from langchain_anthropic import AnthropicEmbeddings
from langchain.chains import RetrievalQA

# 벡터 스토어 연결
vectorstore = PGVector(
    connection_string="postgresql://...",
    embedding_function=AnthropicEmbeddings(),
    collection_name="documents"
)

# RAG 체인 구성
qa_chain = RetrievalQA.from_chain_type(
    llm=ChatAnthropic(model="claude-sonnet-4-6"),
    chain_type="stuff",
    retriever=vectorstore.as_retriever(search_kwargs={"k": 5}),
    return_source_documents=True
)

result = qa_chain.invoke({"query": "센서 이상값 기준은?"})
```

### Tool 사용 에이전트

```python
from langchain.agents import create_tool_calling_agent, AgentExecutor
from langchain.tools import tool

@tool
def get_sensor_data(device_id: str) -> str:
    """TimescaleDB에서 최근 센서 데이터 조회"""
    # DB 쿼리 ...
    return f"device {device_id}: temp=25.3, humidity=60"

@tool
def send_alert(message: str, channel: str = "slack") -> str:
    """운영팀에 알림 발송"""
    # Slack API 호출 ...
    return "알림 발송 완료"

tools = [get_sensor_data, send_alert]

agent = create_tool_calling_agent(
    llm=ChatAnthropic(model="claude-sonnet-4-6"),
    tools=tools,
    prompt=ChatPromptTemplate.from_messages([
        ("system", "IoT 모니터링 에이전트입니다. 필요시 도구를 사용하세요."),
        ("human", "{input}"),
        ("placeholder", "{agent_scratchpad}")
    ])
)

executor = AgentExecutor(agent=agent, tools=tools, verbose=True)
executor.invoke({"input": "디바이스 DEV-001의 최근 데이터 확인 후 이상 있으면 알림"})
```

---

## 4. LangGraph

### 특징

```
LangChain의 한계:
  - 선형 체인만 가능 (if/else 분기, 루프 어려움)
  - 상태 관리 복잡
  - 에이전트 실행 흐름 제어 어려움

LangGraph가 해결:
  - 노드 = Python 함수 (LLM 호출, 도구 실행, 조건 판단)
  - 엣지 = 노드 간 연결 (조건부 엣지로 분기)
  - State = TypedDict (노드 간 공유 상태)
  - 루프: 그래프 안에서 반복 실행 가능
  - HITL: interrupt_before로 사람 승인 대기
```

### 기본 그래프

```python
from langgraph.graph import StateGraph, END
from typing import TypedDict, Annotated
import operator

# ── 상태 정의 ──────────────────────────────────────────────────
class AgentState(TypedDict):
    messages: Annotated[list, operator.add]  # 메시지 누적
    device_id: str
    sensor_data: dict
    analysis_result: str
    alert_sent: bool

# ── 노드 함수 ─────────────────────────────────────────────────
def fetch_data(state: AgentState) -> AgentState:
    """센서 데이터 조회"""
    data = query_timescaledb(state["device_id"])
    return {"sensor_data": data}

def analyze(state: AgentState) -> AgentState:
    """LLM으로 이상 분석"""
    llm = ChatAnthropic(model="claude-sonnet-4-6")
    result = llm.invoke(f"데이터 분석: {state['sensor_data']}")
    return {"analysis_result": result.content}

def should_alert(state: AgentState) -> str:
    """조건부 엣지: 이상 여부에 따라 분기"""
    if "이상" in state["analysis_result"] or "위험" in state["analysis_result"]:
        return "send_alert"
    return "end"

def send_alert_node(state: AgentState) -> AgentState:
    """알림 발송"""
    send_slack(state["analysis_result"])
    return {"alert_sent": True}

# ── 그래프 구성 ───────────────────────────────────────────────
graph = StateGraph(AgentState)

graph.add_node("fetch_data", fetch_data)
graph.add_node("analyze", analyze)
graph.add_node("send_alert", send_alert_node)

graph.set_entry_point("fetch_data")
graph.add_edge("fetch_data", "analyze")

# 조건부 엣지 (분기)
graph.add_conditional_edges(
    "analyze",
    should_alert,
    {
        "send_alert": "send_alert",
        "end": END
    }
)
graph.add_edge("send_alert", END)

app = graph.compile()

# 실행
result = app.invoke({
    "device_id": "DEV-001",
    "messages": [],
    "sensor_data": {},
    "analysis_result": "",
    "alert_sent": False
})
```

### Human-in-the-Loop (HITL)

```python
from langgraph.checkpoint.memory import MemorySaver

checkpointer = MemorySaver()
app = graph.compile(
    checkpointer=checkpointer,
    interrupt_before=["send_alert"]   # 알림 발송 전 사람 승인 대기
)

# 1. 실행 시작
config = {"configurable": {"thread_id": "alert-001"}}
result = app.invoke(initial_state, config=config)
# → "send_alert" 노드 직전에서 중단됨

# 2. 현재 상태 확인
snapshot = app.get_state(config)
print("분석 결과:", snapshot.values["analysis_result"])

# 3. 승인 후 계속 실행
app.invoke(None, config=config)  # None = 상태 유지하고 재개
```

### 멀티 에이전트 (병렬)

```python
from langgraph.graph import StateGraph
from langgraph.prebuilt import ToolNode

# 병렬 처리: 여러 카메라를 동시 분석
def parallel_camera_analysis(state):
    with ThreadPoolExecutor() as executor:
        futures = {
            cam_id: executor.submit(analyze_camera, cam_id)
            for cam_id in state["camera_ids"]
        }
        results = {cam_id: f.result() for cam_id, f in futures.items()}
    return {"camera_results": results}
```

---

## 5. 도구 선택 기준

```
[n8n 선택]
  → 비개발자 포함 팀
  → 300+ 서비스 연동이 빠르게 필요
  → 비즈니스 자동화 (이메일, CRM, 슬랙 알림)
  → 코딩 없이 워크플로우 구성

[LangChain 선택]
  → Python 개발자
  → RAG 파이프라인 구성
  → 단순 LLM 체인 (프롬프트 → LLM → 파서)
  → 빠른 프로토타입

[LangGraph 선택]
  → 복잡한 에이전트 로직 (루프, 분기, 조건)
  → 상태 유지가 필요한 멀티스텝 워크플로우
  → 사람 개입(HITL) 필요 (승인 게이트)
  → 멀티 에이전트 협업

[Node-RED vs n8n]
  Node-RED: IoT/하드웨어 중심, MQTT/TCP/Serial 강함
  n8n:      비즈니스/SaaS 연동 중심, AI 에이전트 노드 강함
```

---

## 6. 관련
- [[Agent-Harness]] — 에이전트 루프, 툴 호출 상세
- [[RAG]] — LangChain 기반 RAG 파이프라인
- [[LLMOps]] — 모델 게이트웨이, 비용 추적
- [[../programming/iot/Node-RED]] — IoT 특화 시각적 플로우 (Node-RED 비교)
