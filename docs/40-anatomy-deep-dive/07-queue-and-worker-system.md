# 07 Queue and Worker System Anatomy

Langfuse는 웹 서버(Next.js API)가 모든 무거운 작업을 즉시 처리하지 않도록, 비동기 작업 브로커인 Redis 기반의 **BullMQ**를 뼈대로 한 Queue 및 Worker 시스템을 운용합니다. 이 문서는 그 구조를 해부합니다.

## 1. 큐 스키마 및 계약 (Queues Contract)

`web` (프로듀서)과 `worker` (컨슈머) 코드가 분리된 모노레포 구조에서 가장 중요한 것은 두 컴포넌트 간의 페이로드 계약(Payload Contract)입니다. 이 계약은 `@langfuse/shared` 패키지의 `packages/shared/src/server/queues.ts`에 정의되어 있습니다.

모든 큐와 잡(Job)은 Zod 스키마로 엄격하게 타이핑되어 있으며, 컴파일 타임 및 런타임에 유효성이 검증됩니다.

### 주요 Queue 종류

1. **Ingestion 관련 큐**
   - `IngestionQueue` / `OtelIngestionQueue`: 메인 이벤트 파이프라인.
   - `IngestionSecondaryQueue`: 트래픽을 과도하게 발생시키거나 S3 Slowdown 에러를 발생시킨 특정 프로젝트(High-throughput projects)의 트래픽을 격리하여 다른 프로젝트의 처리 속도에 영향을 주지 않도록 하는 백업 큐.
2. **Evaluation (평가) 큐**
   - `EvaluationExecution` / `LLMAsJudgeExecution`: LLM-as-a-judge 등 자동 평가 스크립트 실행. 외부 LLM 호출이 동반되므로 레이턴시가 긺.
3. **데이터 유지 및 내보내기 큐**
   - `DataRetentionQueue`: Retention 주기(TTL)가 끝난 데이터를 지우는 배치 작업.
   - `BatchExport`: S3로 대용량 데이터 내보내기 작업 생성.
4. **연동 및 알림 큐**
   - `WebhookQueue`, `PostHogIntegrationQueue`, `NotificationQueue`: 외부 서비스로의 이벤트 전파.

## 2. Worker 아키텍처

Worker는 `worker` 디렉토리 하위의 독립된 Node.js 애플리케이션으로 실행됩니다. 

- **수평 확장 (Horizontal Scaling)**: Worker 노드는 상태를 갖지 않으며(Stateless), Redis에만 의존하므로 트래픽이 증가하면 여러 컨테이너 인스턴스를 띄워 손쉽게 스케일 아웃할 수 있습니다.
- **격리 (Isolation)**: Worker가 OOM(Out of Memory)이나 네트워크 지연으로 인해 뻗더라도, 사용자 UI나 Ingestion을 수신하는 Next.js Web 노드는 전혀 타격을 받지 않습니다.

## 3. 에러 복구 및 Resilience 설계

비동기 작업은 네트워크 타임아웃, DB 락(Lock) 등으로 실패할 확률이 높습니다. Langfuse는 이러한 에러를 복구하기 위해 두 가지 메커니즘을 씁니다.

### A. Retry Baggage

`queues.ts`를 보면 일부 Job 타입에 `retryBaggage`라는 객체가 선택적으로 포함됩니다.

```typescript
export const RetryBaggage = z.object({
  originalJobTimestamp: z.date(),
  attempt: z.number(),
});
```
작업이 실패하여 큐에 재삽입(Re-queue)될 때, 최초 발생 시간과 재시도 횟수를 기록합니다. 이를 통해 무한 재시도 루프에 빠지는 것을 막고, 지수 백오프(Exponential Backoff)를 적용하거나 일정 횟수 초과 시 완전한 실패로 처리할 수 있습니다.

### B. Dead Letter Queue (DLQ)

`DeadLetterRetryQueue`는 반복해서 실패한(예: Max Attempts 도달) Job들이 버려지기 전에 모이는 큐입니다.
시스템 관리자는 DLQ에 모인 실패 작업들의 에러 로그를 분석하여 버그를 수정한 뒤, 수동 혹은 스크립트로 이 큐의 작업들을 원래 큐로 복원하여 데이터 유실 없이 작업을 마무리할 수 있습니다.
