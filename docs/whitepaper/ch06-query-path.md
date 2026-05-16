# Chapter 6. 쿼리 경로와 타입 안전성

> *"6개의 독립 ClickHouse 쿼리를 한 번에 실행하여 가장 느린 쿼리의 응답 시간만큼만 대기한다."*

---

## 6.1 설계 질문

**프론트엔드(React)와 백엔드(Next.js) 사이에서 타입 안전성을 보장하면서, PostgreSQL과 ClickHouse를 동시에 조회하는 복잡한 쿼리를 어떻게 관리하는가?**

## 6.2 Langfuse의 답: tRPC — End-to-End 타입 안전성

```mermaid
flowchart LR
    subgraph FE["프론트엔드"]
        Hook["trpc.trace.all.useQuery()"]
    end

    subgraph Transport["전송 계층"]
        Batch["HTTP Batch Link<br/>POST /api/trpc/trace.all"]
    end

    subgraph BE["백엔드"]
        MW["미들웨어 체인"]
        Router["traceRouter.all resolver"]
    end

    Hook -->|"TypeScript 타입 추론"| Batch
    Batch --> MW --> Router
    Router -->|"동일 타입으로 응답"| Hook
```

**왜 REST가 아닌 tRPC인가?**

| 기준 | REST API | tRPC |
|---|---|---|
| 타입 안전성 | 수동 (OpenAPI → codegen) | **자동** (서버 타입이 클라이언트에 전파) |
| 스키마 변경 시 | 클라이언트가 깨져도 컴파일 타임에 모름 | **컴파일 에러로 즉시 감지** |
| Batch 요청 | 직접 구현 필요 | 빌트인 HTTP Batch Link |
| 외부 클라이언트 지원 | ✅ 표준 HTTP | ❌ (내부 전용) |

> Langfuse의 판단: **대시보드 UI는 내부 전용**이므로 tRPC로 타입 안전성을 극대화하고, **SDK용 Public API는 REST로** 유지하여 언어 중립성을 보장한다.

## 6.3 미들웨어 체인 — 보안 계층

🔗 [`web/src/server/api/trpc.ts`](file:///Users/dhsshin/Documents/LLMOps/langfuse/web/src/server/api/trpc.ts)

```mermaid
flowchart TD
    Request["HTTP 요청"] --> Public["publicProcedure<br/>세션 확인만 (Guest OK)"]
    Public --> Protected["protectedProcedure<br/>로그인 필수"]
    Protected --> Project["protectedProjectProcedure<br/>프로젝트 RBAC 확인"]
    Project --> Trace["protectedGetTraceProcedure<br/>특정 트레이스 접근 권한"]

    Public -.->|"세션 없음"| Auth401["401 Unauthorized"]
    Protected -.->|"미로그인"| Auth401
    Project -.->|"권한 없음"| Auth403["403 Forbidden"]
```

| 계층 | 체크 대상 | 실패 시 |
|---|---|---|
| `publicProcedure` | ctx.session 존재 여부 | Guest 허용 (일부 읽기 전용 뷰) |
| `protectedProcedure` | `ctx.session.user` 확인 | 401 반환 |
| `protectedProjectProcedure` | `ProjectMembership` + Role | 403 반환 |
| `protectedGetTraceProcedure` | ClickHouse에서 Trace 존재 확인 | 404 반환 |

## 6.4 병렬 쿼리 패턴 — PostgreSQL + ClickHouse 합성

🔗 [`traces.ts` L125-152](file:///Users/dhsshin/Documents/LLMOps/langfuse/web/src/server/api/routers/traces.ts#L125-L152)

```mermaid
sequenceDiagram
    participant UI as React DataTable
    participant R as traceRouter.all
    participant PG as PostgreSQL
    participant CH as ClickHouse

    UI->>R: useQuery({ filter, page, limit })
    R->>PG: applyCommentFilters(filterState)
    Note over PG: 댓글 필터가 있으면<br/>매칭 trace ID 조회

    alt 댓글 필터 매치 없음
        R-->>UI: { traces: [] }
    end

    R->>CH: getTracesTable({ filter, limit, page })
    Note over CH: AMT 뷰에서 필터링 + 페이지네이션
    CH-->>R: traces[]
    R-->>UI: { traces }
```

### `filterOptions` — 6개 동시 쿼리의 위력

🔗 [`traces.ts` L262-331](file:///Users/dhsshin/Documents/LLMOps/langfuse/web/src/server/api/routers/traces.ts#L262-L331)

```mermaid
flowchart LR
    subgraph PA["Promise.all (6개 동시)"]
        Q1["getNumericScoresGroupedByName"]
        Q2["getCategoricalScoresGroupedByName"]
        Q3["getTracesGroupedByName"]
        Q4["getTracesGroupedByTags"]
        Q5["getTracesGroupedByUsers"]
        Q6["getTracesGroupedBySessionId"]
    end

    subgraph Time["응답 시간"]
        T_SEQ["순차 실행: ~600ms<br/>(100ms × 6)"]
        T_PAR["병렬 실행: ~100ms<br/>(가장 느린 쿼리만)"]
    end

    PA --> T_PAR
```

> 성능 이득: **6배 빠른 필터 옵션 로딩**

## 6.5 ClickHouse SQL Injection 방어 — Parameter Binding

```mermaid
flowchart LR
    subgraph Unsafe["❌ 문자열 보간 (사용하지 않음)"]
        U1["query = `SELECT * FROM traces<br/>WHERE project_id = '${input}'`"]
        U2["→ SQL Injection 가능"]
    end

    subgraph Safe["✅ Named Parameter (Langfuse 전체 적용)"]
        S1["query = `SELECT * FROM traces<br/>WHERE project_id = {projectId: String}`"]
        S2["query_params: { projectId: input }"]
        S3["→ 자동 이스케이프"]
    end
```

---

## 이 챕터의 핵심 인사이트

1. **tRPC = 내부 UI용, REST = 외부 SDK용** — 각각의 장점을 취함
2. **미들웨어 체인이 4단계 보안 계층**을 형성
3. **Promise.all로 6배 빠른 필터 옵션 로딩**
4. **Named Parameter Binding으로 SQL Injection 원천 차단**

---

| 방향 | 문서 |
|---|---|
| ⬅️ 이전 | [Ch.5 ClickHouse 엔지니어링](./ch05-clickhouse-engineering.md) |
| ➡️ 다음 | [Ch.7 인증 아키텍처](./ch07-authentication.md) |
| 🏠 백서 색인 | [WHITEPAPER.md](../WHITEPAPER.md) |
