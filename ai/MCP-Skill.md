---
tags:
  - ai
  - mcp
  - agent
  - skill
  - tool-use
created: 2026-06-16
---

# MCP & Skill (에이전트 확장 표준)

> [!summary] 한 줄 요약
> **MCP(Model Context Protocol)**는 LLM 에이전트가 외부 시스템(GitHub, DB, Slack 등)에 접근하는 **표준 인터페이스**. **Skill**은 에이전트가 호출할 수 있는 **재사용 가능한 기능 단위**. 함께 사용하면 에이전트의 능력을 체계적으로 확장할 수 있다.

---

## 1. MCP (Model Context Protocol)

Anthropic이 2024년 공개한 오픈 표준. LLM이 외부 데이터·도구에 **일관된 방식**으로 접근하도록 규격화.

```
[기존 방식]                    [MCP 방식]
  에이전트 ─── 커스텀 API ──► GitHub
  에이전트 ─── 커스텀 API ──► Slack       에이전트
  에이전트 ─── 커스텀 API ──► DB              │
  (통합마다 새 코드 필요)           MCP 클라이언트
                                             │  MCP 표준 프로토콜
                                  ┌──────────┼──────────┐
                             MCP      MCP         MCP
                            Server   Server      Server
                            (GitHub) (Slack)     (DB)
```

### MCP 3대 구성요소

| 구성요소 | 설명 | 예시 |
|---|---|---|
| **Tools** | 에이전트가 호출하는 함수 | `create_issue`, `send_message` |
| **Resources** | 읽기 전용 데이터 소스 | 파일, DB 레코드, API 응답 |
| **Prompts** | 재사용 가능한 프롬프트 템플릿 | 코드 리뷰 템플릿, 번역 양식 |

### MCP 서버 구현 (Python)

```python
# pip install mcp
from mcp.server import Server
from mcp.server.stdio import stdio_server
from mcp.types import Tool, TextContent

app = Server("my-db-server")

@app.list_tools()
async def list_tools() -> list[Tool]:
    return [
        Tool(
            name="query_database",
            description="SQL 쿼리를 실행하고 결과를 반환. SELECT만 허용.",
            inputSchema={
                "type": "object",
                "properties": {
                    "sql": {"type": "string", "description": "실행할 SELECT 쿼리"}
                },
                "required": ["sql"]
            }
        )
    ]

@app.call_tool()
async def call_tool(name: str, arguments: dict) -> list[TextContent]:
    if name == "query_database":
        sql = arguments["sql"]
        if not sql.strip().upper().startswith("SELECT"):
            raise ValueError("SELECT만 허용됩니다")
        result = db.execute(sql)
        return [TextContent(type="text", text=str(result))]

async def main():
    async with stdio_server() as (read, write):
        await app.run(read, write, app.create_initialization_options())
```

### Claude Desktop에서 MCP 설정

```json
// ~/Library/Application Support/Claude/claude_desktop_config.json
{
  "mcpServers": {
    "my-db": {
      "command": "python",
      "args": ["/path/to/my_db_server.py"],
      "env": {
        "DB_URL": "postgresql://..."
      }
    },
    "filesystem": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-filesystem", "/allowed/path"]
    },
    "github": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"],
      "env": {"GITHUB_TOKEN": "ghp_..."}
    }
  }
}
```

### 공식 MCP 서버 목록 (주요)

| 서버 | 패키지 | 기능 |
|---|---|---|
| **filesystem** | `@modelcontextprotocol/server-filesystem` | 로컬 파일 읽기/쓰기 |
| **github** | `@modelcontextprotocol/server-github` | 이슈, PR, 코드 조회 |
| **postgres** | `@modelcontextprotocol/server-postgres` | SQL 쿼리 실행 |
| **slack** | `@modelcontextprotocol/server-slack` | 메시지 조회/전송 |
| **web-search** | Brave Search MCP | 웹 검색 |
| **memory** | `@modelcontextprotocol/server-memory` | KV 영속 메모리 |

---

## 2. Skill (에이전트 스킬)

에이전트가 반복적으로 수행하는 **캡슐화된 능력 단위**. MCP Tool보다 더 높은 추상화 수준.

```
MCP Tool: 단순 함수 호출 (create_github_issue)
Skill:    복합 목표 달성 (코드 리뷰 후 이슈 생성 + 슬랙 알림)
          여러 도구 조합 + 프롬프트 로직 포함
```

### Skill 구성 요소

```python
from dataclasses import dataclass
from typing import Callable

@dataclass
class Skill:
    name: str                    # 식별자
    description: str             # 언제 이 스킬을 쓰는가 (모델이 판단에 사용)
    trigger_conditions: list     # 어떤 상황에서 발동
    system_prompt: str           # 스킬 특화 시스템 프롬프트
    tools: list                  # 사용하는 MCP 툴/함수 목록
    handler: Callable            # 실행 로직
```

### Claude Code의 `/skill` 시스템

Claude Code에서 `/` 명령어로 호출되는 내장 스킬 시스템:

```
# 사용자가 슬래시 커맨드 입력
/commit-push
/review
/explain

# 내부 동작
1. skill 파일 로드 (마크다운 기반 instruction)
2. 해당 스킬의 시스템 프롬프트 + 툴 권한 적용
3. 에이전트 루프 실행
```

```markdown
<!-- ~/.claude/skills/commit-push.md -->
# Commit & Push Skill

## 트리거
사용자가 "commit push" 또는 "/commit-push" 입력 시

## 절차
1. git status 확인
2. 변경 내용 분석
3. Conventional Commits 스타일 메시지 작성
4. git add → commit → push
```

### 커스텀 Skill 패턴 (Spring Boot 에이전트)

```java
public class CodeReviewSkill implements AgentSkill {

    @Override
    public String getName() { return "code-review"; }

    @Override
    public String getDescription() {
        return "PR 코드를 리뷰하고 이슈를 GitHub에 등록한다. "
             + "PR URL이 주어지면 자동 실행.";
    }

    @Override
    public List<ToolDefinition> getTools() {
        return List.of(
            githubTool.getFileContent(),
            githubTool.createComment(),
            githubTool.createIssue(),
            slackTool.sendMessage()
        );
    }

    @Override
    public String getSystemPrompt() {
        return """
               코드 리뷰 전문가로서:
               1. 변경된 파일을 모두 읽어라
               2. 보안 취약점, 성능 이슈, 코드 스타일 순으로 검토
               3. 각 문제를 GitHub 코멘트로 남겨라
               4. 심각한 이슈는 GitHub 이슈로 등록
               5. 완료 후 Slack #dev-review에 요약 전송
               """;
    }
}
```

---

## 3. Agent + MCP + Skill 통합 아키텍처

```
[사용자]
    │ "PR #123 리뷰해줘"
    ▼
[에이전트 하니스]  ← [[Agent-Harness]] 참고
    │ Skill 매칭: "code-review" 스킬 선택
    ▼
[Skill: CodeReview]
    │ System Prompt + 허용 Tools 설정
    ▼
[LLM 루프]
    │ tool_use: get_file_content(PR #123의 파일들)
    ├──► [MCP: GitHub Server] ─► GitHub API
    │    결과 반환
    │ tool_use: create_comment(...)
    ├──► [MCP: GitHub Server] ─► GitHub API
    │ tool_use: send_message(#dev-review)
    └──► [MCP: Slack Server]  ─► Slack API
    │
    ▼
[완료: Slack에 요약 전송]
```

---

## 4. MCP 보안 고려사항

```python
# 1. 최소 권한 원칙 — 필요한 Tool만 노출
@app.list_tools()
async def list_tools():
    tools = []
    if user_has_permission("read"):
        tools.append(read_tool)
    if user_has_permission("write"):
        tools.append(write_tool)    # write는 명시적 권한 확인
    return tools

# 2. 입력 검증 — SQL Injection, Path Traversal 방지
def validate_path(path: str, allowed_root: str):
    resolved = os.path.realpath(path)
    if not resolved.startswith(allowed_root):
        raise SecurityError("허용된 경로 밖 접근 시도")

# 3. 비밀 관리 — 시크릿은 환경 변수로, 프롬프트에 노출 금지
DB_URL = os.environ["DB_URL"]   # config 파일/프롬프트에 직접 X

# 4. 읽기 전용 우선 — 상태 변경 툴은 별도 승인 게이트
@app.call_tool()
async def call_tool(name: str, arguments: dict):
    if name in DESTRUCTIVE_TOOLS:
        if not await request_user_approval(name, arguments):
            return [TextContent(type="text", text="사용자가 거부했습니다")]
```

---

## 5. Skill vs Tool vs Function 비교

| 개념 | 추상화 수준 | 정의 | 실행 주체 |
|---|---|---|---|
| **Function/API** | 낮음 | 단순 함수 호출 | 코드 |
| **MCP Tool** | 낮음 | 표준화된 함수 (입력 스키마 있음) | MCP 서버 |
| **Agent Tool** | 중간 | 에이전트가 선택해 호출하는 MCP Tool | 에이전트 하니스 |
| **Skill** | 높음 | 목표 달성을 위한 도구+프롬프트 묶음 | 에이전트 루프 |

---

## 6. 관련
- [[Agent-Harness]] · [[LLM]] · [[Prompt-Engineering]]
- [[Tool-Search]] — MCP 툴이 많아져 스키마가 컨텍스트를 잠식할 때 점진적 공개
- [[Hermes-Skills]] — Hermes의 skill 호출(로딩) 메커니즘·조건부 활성·절차 기억
- [[Hermes-Plugins]] — 확장 3종 중 plugin(인프로세스 코드) vs MCP(외부 서버) 비교
