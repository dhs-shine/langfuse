# 08 ClickHouse Schema and MVs Anatomy

> **선행 문서**: [04 Data Model and Storage](../20-core-domain/04-data-model-and-storage.md)
> **소스 딥다이브**: [12 Query Source Breakdown](../50-source-analysis/12-query-source-breakdown.md)

---

Langfuse가 대규모 LLM 이벤트 데이터를 실시간으로 집계하고 대시보드에 ms 단위로 렌더링할 수 있는 이유는 ClickHouse의 **AggregatingMergeTree(AMT)** 와 **Materialized View(MV)** 를 활용한 뷰 롤업 최적화 덕분입니다.

## 전체 ClickHouse 테이블 계층 구조

```mermaid
flowchart TD
    subgraph Write["쓰기 경로 (Worker)"]
        CW["ClickhouseWriter"]
        CW -->|addToQueue Traces| T["traces<br/>(ReplacingMergeTree)"]
        CW -->|addToQueue TracesNull| TN["traces_null<br/>(Null Engine)"]
        CW -->|addToQueue Observations| O["observations<br/>(ReplacingMergeTree)"]
        CW -->|addToQueue Scores| S["scores<br/>(ReplacingMergeTree)"]
        CW -->|addToQueue ObsBatchStaging| BS["observations_batch_staging"]
        CW -->|addToQueue EventsFull| EF["events_full<br/>(V4 전용 통합 이벤트)"]
        CW -->|addToQueue DatasetRunItems| DRI["dataset_run_items"]
        CW -->|addToQueue BlobStorageFileLog| BSF["blob_storage_file_log"]
    end

    subgraph MVPipeline["MV 자동 집계"]
        TN -->|traces_all_amt_mv| AMT_ALL["traces_all_amt<br/>(AMT, 영구)"]
        TN -->|traces_7d_amt_mv| AMT_7D["traces_7d_amt<br/>(AMT, TTL 7일)"]
        TN -->|traces_30d_amt_mv| AMT_30D["traces_30d_amt<br/>(AMT, TTL 30일)"]
    end

    subgraph Read["읽기 경로 (Web tRPC)"]
        Dashboard["대시보드 쿼리"]
        Dashboard -->|최근 7일 필터| AMT_7D
        Dashboard -->|최근 30일 필터| AMT_30D
        Dashboard -->|전체 기간| AMT_ALL
        Detail["상세 페이지 쿼리"]
        Detail --> T
        Detail --> O
        Detail --> S
    end
```

## Null 테이블을 활용한 트리거 패턴

🔗 [`0023...sql` L3-39](file:///Users/dhsshin/Documents/LLMOps/langfuse/packages/shared/clickhouse/migrations/unclustered/0023_traces_aggregating_merge_trees.up.sql#L3-L39)

```mermaid
flowchart LR
    Insert["INSERT INTO traces_null"] --> NullEngine["Null Engine<br/>(저장 안 함)"]
    NullEngine -->|MV 1| ALL["traces_all_amt"]
    NullEngine -->|MV 2| D7["traces_7d_amt"]
    NullEngine -->|MV 3| D30["traces_30d_amt"]
```

- **Null Engine**: 데이터를 물리적으로 저장하지 않고 통과만 시키는 `/dev/null` 역할
- **왜 쓰는가?**: 하나의 Insert로 여러 개의 목적지 테이블(MV)에 분기하여 데이터를 가공·적재하기 위한 트리거 용도. 중간 데이터를 남기지 않아 스토리지 비용을 절약

## AggregatingMergeTree와 집계 함수

🔗 [`0023...sql` L42-87](file:///Users/dhsshin/Documents/LLMOps/langfuse/packages/shared/clickhouse/migrations/unclustered/0023_traces_aggregating_merge_trees.up.sql#L42-L87)

```mermaid
flowchart TD
    subgraph 집계함수["ClickHouse 집계 함수 동작 원리"]
        direction TB

        MIN["min(start_time)"]
        MIN_DESC["여러 이벤트 중 가장<br/>이른 시간만 유지"]
        MIN --- MIN_DESC

        SUMMAP["sumMap(cost_details)"]
        SUMMAP_DESC["Map의 동일 키끼리<br/>값을 누적 합산"]
        SUMMAP --- SUMMAP_DESC

        ARGMAX["argMax(input, event_ts)"]
        ARGMAX_DESC["가장 최신 타임스탬프를<br/>가진 값 하나만 유지<br/>(UPDATE 대체)"]
        ARGMAX --- ARGMAX_DESC

        GROUPUNIQ["groupUniqArrayArray(tags)"]
        GROUPUNIQ_DESC["여러 배열을 합치되<br/>중복 원소 제거"]
        GROUPUNIQ --- GROUPUNIQ_DESC
    end
```

| 집계 함수 | 적용 컬럼 | 동작 | UPDATE 대체 |
|---|---|---|---|
| `min` | `start_time`, `created_at` | 가장 이른(오래된) 값 유지 | ✅ |
| `max` | `end_time`, `updated_at` | 가장 늦은(최신) 값 유지 | ✅ |
| `sumMap` | `cost_details`, `usage_details` | Map 키별 숫자 합산 | ✅ |
| `maxMap` | `metadata` | Map 키별 최신값 유지 | ✅ |
| `groupUniqArrayArray` | `tags`, `observation_ids`, `score_ids` | 중복 없는 유니크 배열 병합 | ✅ |
| `argMax(value, ts)` | `input`, `output`, `bookmarked` | 최신 타임스탬프의 값만 보존 | ✅ |

> **핵심 설계 원칙**: ClickHouse는 `UPDATE`/`DELETE`가 매우 비싼 작업입니다. Langfuse는 이를 완전히 회피하고, 새 이벤트를 Insert만 하면 MV가 백그라운드에서 상태를 자동 병합하는 패턴을 사용합니다.

## TTL 기반 파티셔닝 뷰

🔗 [`0023...sql` L129-174](file:///Users/dhsshin/Documents/LLMOps/langfuse/packages/shared/clickhouse/migrations/unclustered/0023_traces_aggregating_merge_trees.up.sql#L129-L174)

```mermaid
flowchart LR
    User["대시보드 사용자"] --> Filter{"기간 필터 선택"}
    Filter -- "최근 7일" --> Q7["traces_7d_amt<br/>⚡ 가장 작은 테이블"]
    Filter -- "최근 30일" --> Q30["traces_30d_amt<br/>🔹 중간 크기"]
    Filter -- "전체 기간" --> QALL["traces_all_amt<br/>🔶 전체 데이터"]
```

| 테이블 | TTL | 사내 커스터마이징 |
|---|---|---|
| `traces_all_amt` | 없음 (영구) | 전체 보존 (규정 준수 감사용) |
| `traces_7d_amt` | `toDate(start_time) + INTERVAL 7 DAY` | TTL 값을 사내 정책(예: 14일)으로 변경 가능 |
| `traces_30d_amt` | `toDate(start_time) + INTERVAL 30 DAY` | TTL 값을 사내 정책(예: 90일)으로 변경 가능 |

## 블룸 필터 인덱스

```sql
INDEX idx_trace_id id TYPE bloom_filter(0.001) GRANULARITY 1
INDEX idx_user_id user_id TYPE bloom_filter(0.001) GRANULARITY 1
INDEX idx_session_id session_id TYPE bloom_filter(0.001) GRANULARITY 1
```

ClickHouse는 컬럼 스캔에 특화되어 있지만, 대시보드에서 특정 `trace_id` 단건 조회 시에는 해시 기반의 블룸 필터가 불필요한 스토리지 블록 접근을 차단하여 응답 속도를 크게 향상시킵니다.

---

## 관련 문서

| | |
|---|---|
| ⬇️ 소스 딥다이브 | [12 Query Source Breakdown](../50-source-analysis/12-query-source-breakdown.md) |
| ⬅️ 이전 | [07 Queue and Worker System](./07-queue-and-worker-system.md) |
| ➡️ 다음 | [09 tRPC and Next.js](./09-trpc-and-nextjs.md) |
| 🏠 색인 | [README](../README.md) |
