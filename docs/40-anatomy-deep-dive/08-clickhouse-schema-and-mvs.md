# 08 ClickHouse Schema and MVs Anatomy

Langfuse가 대규모 LLM 이벤트 데이터를 실시간으로 집계(Aggregation)하고 대시보드에 빠르게 렌더링할 수 있는 이유는 ClickHouse의 **AggregatingMergeTree (AMT)** 와 **Materialized View (MV)** 를 활용한 뷰 롤업(Roll-up) 최적화 덕분입니다.

이 문서에서는 `packages/shared/clickhouse/migrations`에 정의된 핵심 스키마 아키텍처를 해부합니다.

## 1. Null 테이블을 활용한 트리거 (Trigger) 패턴

ClickHouse에 데이터를 직접 Insert할 때 일반적인 테이블에 바로 넣지 않고 `Null` 엔진을 사용하는 테이블을 거칩니다. (예: `traces_null`)

```sql
CREATE TABLE traces_null (
    project_id String,
    id String,
    start_time DateTime64(3),
    -- 기타 필드들...
) Engine = Null();
```

- **Null Engine의 역할**: 데이터를 스토리지에 물리적으로 저장하지 않고 통과만 시키는 `/dev/null`과 같은 역할을 합니다.
- **왜 쓰는가?**: 하나의 원본 데이터(Insert)로 여러 개의 목적지 테이블(Materialized Views)에 분기하여 데이터를 가공 및 적재하기 위한 깔때기(Trigger) 용도로 사용됩니다. 중간 데이터를 남기지 않아 스토리지 비용을 아낍니다.

## 2. AggregatingMergeTree (AMT) 와 뷰 롤업

Langfuse는 실시간 지표(토큰 사용량, 비용, 평균 지연시간 등)를 쿼리할 때마다 원본 `observations` 테이블을 Full Scan하지 않습니다. 대신 `AggregatingMergeTree` 엔진을 사용한 테이블을 구축합니다.

### 데이터 흐름
`traces_null`에 데이터가 Insert되면, 백그라운드에서 동작하는 Materialized View(`traces_all_amt_mv`, `traces_7d_amt_mv` 등)가 데이터를 잡아서 그룹화(`GROUP BY project_id, id`)하고 상태(State)를 업데이트합니다.

```sql
CREATE MATERIALIZED VIEW IF NOT EXISTS traces_all_amt_mv TO traces_all_amt AS
SELECT
    tn.project_id as project_id,
    tn.id as id,
    min(tn.start_time) as start_time,
    maxMap(tn.metadata) as metadata,
    sumMap(tn.cost_details) as cost_details,
    argMaxState(tn.input, if(tn.input <> '', tn.event_ts, toDateTime64(0, 3))) as input
FROM traces_null tn
GROUP BY project_id, id;
```

### Aggregation State 함수들
ClickHouse는 이전에 집계된 데이터에 새로운 데이터가 들어왔을 때 전체를 재계산하지 않고 상태만 병합(Merge)합니다.
- `sumMap(tn.cost_details)`: 이전 Trace의 비용 Map 데이터에 새로운 비용을 더해 실시간으로 총합을 유지합니다.
- `argMaxState(tn.input, ...)`: 동일한 ID의 업데이트 이벤트가 들어올 때, 최신 이벤트 타임스탬프(`event_ts`)를 가진 레코드의 데이터로 덮어쓰도록(State) 유지합니다. UPDATE 쿼리를 쓰지 않고도 데이터의 최신화를 보장하는 핵심 기법입니다.

## 3. TTL(Time-to-Live) 기반의 파티셔닝 뷰

Langfuse는 대시보드의 특정 기간(예: 최근 7일, 최근 30일) 조회 속도를 극대화하기 위해 기간별로 분리된 AMT 테이블을 가집니다.

- `traces_7d_amt`: `TTL toDate(start_time) + INTERVAL 7 DAY` 속성을 가져서 7일이 지난 데이터는 ClickHouse가 백그라운드에서 자동 삭제합니다.
- `traces_30d_amt`: 30일 보존 테이블.
- `traces_all_amt`: 전체 보존 테이블.

이러한 분리 구조 덕분에, 사용자가 대시보드에서 "최근 7일"을 필터링하면 백엔드는 무거운 `traces_all_amt` 대신 매우 가벼운 `traces_7d_amt`에서 데이터를 읽어와 응답 속도를 ms 단위로 유지할 수 있습니다. 사내 데이터 보존 정책을 적용할 때, 이 TTL 설정을 커스터마이징하여 프롬프트 원문 보존 주기를 조절할 수 있습니다.
