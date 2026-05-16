# 11 Worker Source Breakdown

이 문서는 Ingestion 파이프라인의 종착지이자, ClickHouse로 데이터를 밀어넣는 핵심 백그라운드 워커의 소스코드를 시각화와 함께 분석합니다.

> **관련 상위 아키텍처 문서**
> - 📄 [07 Queue and Worker System Anatomy](../40-anatomy-deep-dive/07-queue-and-worker-system.md)

---

## 워커 내부 처리 플로우차트

하나의 `IngestionQueue` Job이 Worker Node에 할당되었을 때 벌어지는 동작의 흐름입니다.

```mermaid
flowchart TD
    Start((Job Dequeue)) --> RedisCheck{Redis에 최근<br/>처리 기록이 있는가?}
    
    RedisCheck -- Yes --> Skip[로직 즉시 종료<br/>(Skip)]
    
    RedisCheck -- No --> FlagCheck{S3 Slowdown<br/>마킹된 프로젝트인가?}
    
    FlagCheck -- Yes --> Redirect[Secondary Queue로<br/>Job 토스 후 종료]
    
    FlagCheck -- No --> S3Down[S3에서 JSON 다운로드<br/>(Chunk 병렬 처리)]
    
    S3Down --> SetRedis[Redis 처리 완료 캐시<br/>(EX 5분) 저장]
    
    SetRedis --> Merge[Prisma 메타데이터<br/>병합 및 갱신]
    
    Merge --> Buffer[ClickhouseWriter<br/>메모리 버퍼에 Push]
    
    Buffer --> Timer{타이머 초과 or<br/>배열 Max 도달?}
    
    Timer -- No --> Wait[대기]
    Timer -- Yes --> Flush[(ClickHouse 일괄<br/>Batch Insert)]
```

---

## 1. `ingestionQueue.ts` 워커 구현체

🔗 **Source File:** [`worker/src/queues/ingestionQueue.ts`](file:///Users/dhsshin/Documents/LLMOps/langfuse/worker/src/queues/ingestionQueue.ts)

이 파일은 BullMQ의 `Processor` 인터페이스 구현체를 제공합니다. 메인 함수인 `ingestionQueueProcessorBuilder` 내부를 들여다봅니다.

### A. 중복 방지 캐싱 구조 (Redis Seen Event Cache)
```typescript
if (env.LANGFUSE_ENABLE_REDIS_SEEN_EVENT_CACHE === "true" && redis && job.data.payload.data.fileKey) {
  const key = `langfuse:ingestion:recently-processed:${projectId}:${type}:${eventBodyId}:${fileKey}`;
  const exists = await redis.exists(key);
  if (exists) {
    recordIncrement("langfuse.ingestion.recently_processed_cache", ...);
    return; // 스킵
  }
}
```
- **왜 중복이 발생하는가?**: SDK가 네트워크 통신 실패로 인해 동일한 이벤트를 재시도하거나, BullMQ의 특성 상 At-least-once 전달을 보장하기 때문에 동일한 Job이 여러 번 Dequeue될 가능성이 있습니다.
- 워커는 S3에 올려진 파일 키(`fileKey`)를 기준으로 Redis에 조회하여, 최근 5분 이내에 처리된 이력(`exists`)이 있다면 S3에서 다운로드조차 하지 않고 조기에 종료(Return)합니다. 이를 통해 워커의 낭비되는 CPU 사이클과 S3 GET 요청 과금 비용을 크게 절약합니다.

### B. Secondary Queue 릴레이 분기 (우선순위 라우팅)
```typescript
const shouldRedirectSlowdown = await hasS3SlowdownFlag(projectId);
if (enableRedirectToSecondaryQueue && (shouldRedirectEnv || shouldRedirectSlowdown)) {
  const secondaryQueue = SecondaryIngestionQueue.getInstance({ shardingKey });
  await secondaryQueue.add(QueueName.IngestionSecondaryQueue, job.data);
  return;
}
```
- 앞서 `processEventBatch`에서 S3 Slowdown 에러가 마킹된 프로젝트(`shouldRedirectSlowdown === true`)의 잡이라면, 이 워커가 실행되자마자 스스로를 멈추고 **우선순위가 낮은 `SecondaryIngestionQueue`로 잡을 다시 밀어 넣습니다**.
- 이 구조 덕분에 소수의 프로젝트가 과도한 트래픽을 일으키더라도 다른 정상 프로젝트의 Ingestion은 지연 없이 처리될 수 있습니다(Noisy Neighbor 문제 완벽 방지).

### C. 병렬 S3 다운로드
```typescript
const S3_CONCURRENT_READS = env.LANGFUSE_S3_CONCURRENT_READS;
const batches = chunk(eventFiles, S3_CONCURRENT_READS);
for (const batch of batches) {
  const batchEvents = await Promise.all(batch.map(downloadAndParseFile));
  events.push(...batchEvents.flat());
}
```
- 이벤트 JSON 파일을 가져올 때, 파일 리스트를 `chunk()` 함수로 잘라 `LANGFUSE_S3_CONCURRENT_READS` 설정값(기본 10~50)만큼 한 번에 병렬로 다운로드(`Promise.all`)하여 I/O 대기 타임을 최소화합니다.

---

## 2. 메모리 버퍼링 및 Batch Insert (`ClickhouseWriter.ts`)

🔗 **Source File:** [`worker/src/services/ClickhouseWriter.ts`](file:///Users/dhsshin/Documents/LLMOps/langfuse/worker/src/services/ClickhouseWriter.ts)

S3에서 불러온 데이터는 즉시 쿼리되지 않고, 버퍼링 과정을 거쳐 ClickHouse의 성능을 극대화합니다.

### `IngestionService.mergeAndWrite`와 `ClickhouseWriter.addToQueue`
```typescript
// 내부적으로 addToQueue를 호출
clickhouseWriter.addToQueue(TableName.Traces, traceData);
```
- **`addToQueue`의 진실**: 이 메서드는 직접 DB 커넥션을 열어 Insert하는 것이 아니라, 워커 Node.js 프로세스의 로컬 메모리 버퍼(Array)에 JS 객체를 밀어 넣는(Push) 단순한 동작입니다.
- **자동 Flush 타이머**: 클래스 내부적으로 `setInterval` 루프가 돌면서, 주기적으로(예: 3초) 혹은 배열 크기가 `MAX_BUFFER_SIZE`에 도달하면 메모리에 있는 배열 전체를 꺼냅니다. 그 뒤 ClickHouse Node.js Client를 통해 1개의 쿼리로 압축하여 `INSERT INTO ... FORMAT JSONEachRow` 형태로 전송(Flush)합니다.
- 사내 인프라를 직접 호스팅하는 경우, 이 버퍼 주기와 크기를 커스텀 튜닝(Env 변수 조정)하여 ClickHouse 서버가 받는 Write 부하를 유연하게 조절할 수 있습니다.
