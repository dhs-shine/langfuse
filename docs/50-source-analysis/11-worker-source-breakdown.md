# 11 Worker Source Breakdown

이 문서는 Ingestion 파이프라인의 종착지이자, ClickHouse로 데이터를 밀어넣는 핵심 백그라운드 워커 소스코드를 분석합니다.

## 1. `worker/src/queues/ingestionQueue.ts`

이 파일은 BullMQ의 `Processor` 인터페이스 구현체를 제공합니다. 

### A. 중복 방지 캐싱 구조 (Redis Seen Event Cache) (Lines 83-106)
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
- **왜 중복이 발생하는가?**: SDK가 네트워크 통신 실패로 인해 동일한 이벤트를 재시도하거나, BullMQ의 특성 상 At-least-once 전달을 보장하기 때문에 동일한 Job이 두 번 실행될 가능성이 있습니다.
- 워커는 S3에 올려진 파일 키(`fileKey`)를 기준으로 Redis에 조회하여, 최근 5분 이내에 처리된 이력(`exists`)이 있다면 S3에서 다운로드조차 하지 않고 조기에 종료(Return)합니다. 이를 통해 워커의 낭비되는 CPU 사이클과 S3 GET 비용을 크게 절약합니다.

### B. Secondary Queue 릴레이 분기 (Lines 108-133)
```typescript
const shouldRedirectSlowdown = await hasS3SlowdownFlag(projectId);
if (enableRedirectToSecondaryQueue && (shouldRedirectEnv || shouldRedirectSlowdown)) {
  const secondaryQueue = SecondaryIngestionQueue.getInstance({ shardingKey });
  await secondaryQueue.add(QueueName.IngestionSecondaryQueue, job.data);
  return;
}
```
- 이전 파이프라인(`processEventBatch`)에서 S3 Slowdown 에러를 발생시킨 프로젝트의 잡이라면, 이 워커가 실행되자마자 스스로를 멈추고 **우선순위가 낮은 `SecondaryIngestionQueue`로 잡을 토스**합니다.
- 이 구조 덕분에 한 프로젝트가 과도한 트래픽을 일으키더라도 다른 정상 프로젝트의 Ingestion은 지연 없이 처리될 수 있습니다(Noisy Neighbor 문제 방지).

### C. 병렬 S3 다운로드 (Lines 181-206)
- S3에서 JSON 파일을 가져올 때, 파일 리스트를 `chunk()` 함수로 잘라 `LANGFUSE_S3_CONCURRENT_READS` (기본값 보통 10~50) 설정값만큼 병렬로 다운로드(`Promise.all`)하여 I/O 대기 시간을 최소화합니다.

## 2. ClickHouse 병합 및 일괄 저장 (Batch Insert)

`worker/src/services/ClickhouseWriter.ts`와 `IngestionService.mergeAndWrite`는 실제 데이터를 DB로 넘기는 역할을 합니다.

### `IngestionService.mergeAndWrite`
- 다운로드한 이벤트 파일(`events`)들을 메모리에 올리고, 동일한 ID의 이벤트가 여러 개 있다면 병합(Merge)합니다.
- Postgres(Prisma)에 메타데이터 갱신이 필요한 경우 갱신 후, `clickhouseWriter.addToQueue`를 호출합니다.

### `ClickhouseWriter.ts` (메모리 버퍼링 및 자동 Flush)
- **`addToQueue`의 진실**: 이 메서드는 직접 DB 커넥션을 열어 Insert하는 것이 아니라, 워커 컨테이너의 로컬 메모리 버퍼(배열)에 객체를 밀어 넣는(Push) 동작입니다.
- **자동 Flush 타이머**: 클래스 내부적으로 `setInterval`이 돌아가며 주기적으로(예: 3초마다) 혹은 배열의 크기가 한계치(예: 10,000건)에 도달하면 메모리의 배열 전체를 ClickHouse Node.js Client를 통해 1개의 쿼리로 압축하여 `INSERT INTO ... FORMAT JSONEachRow` 형태로 전송합니다.
- 사내 인프라 환경(예: ClickHouse 스펙이 다소 낮음)에서는 이 버퍼 주기와 크기를 커스텀 튜닝하여 DB 부하를 직접 조절할 수 있습니다.
