# 10 Ingestion Source Breakdown

이 문서는 Langfuse의 Ingestion 파이프라인 진입점 소스코드를 라인 바이 라인(Line-by-Line) 수준에서 분석합니다. 개발자가 직접 API를 수정하거나 버그를 추적할 때 참고할 수 있습니다.

## 1. `web/src/pages/api/public/ingestion.ts`

이 파일은 SDK들이 데이터를 전송하는 메인 엔드포인트입니다.

### A. 인증 및 인가 (Lines 75-89)
```typescript
const authCheck = await new ApiAuthService(
  prisma,
  redis,
).verifyAuthHeaderAndReturnScope(req.headers.authorization);

if (!authCheck.validKey) {
  throw new UnauthorizedError(authCheck.error);
}
```
- `ApiAuthService`는 입력받은 Bearer 토큰(또는 Basic Auth)을 파싱하여 캐시(Redis)나 DB(Prisma)에서 API 키를 조회합니다.
- `scope.projectId` 유무를 체크하여 Organization 키인지 Project 키인지 구분합니다. (Ingestion은 Project 단위로만 가능)

### B. Rate Limit (Lines 103-116)
```typescript
const rateLimitCheck = await RateLimitService.getInstance().rateLimitRequest(
  authCheck.scope,
  "ingestion",
);
```
- `RateLimitService`를 통해 초당 수집 한도를 검사합니다.
- **Fail-open 로직**: Rate Limit 확인 중 에러가 발생해도 예외를 던지지 않고 `logger.error("Error while rate limiting", e)`만 남긴 뒤 그대로 진행합니다. 수집 가용성을 높이기 위함입니다.

## 2. `packages/shared/src/server/ingestion/processEventBatch.ts`

API 레이어를 통과한 데이터가 S3와 Redis로 전달되는 핵심 비즈니스 로직입니다.

### A. Zod 런타임 스키마 분해 및 검증 (Lines 154-187)
```typescript
const ingestionSchema = createIngestionEventSchema(isLangfuseInternal);
const batch: z.infer<typeof ingestionSchema>[] = input
  .flatMap((event) => {
    const parsed = ingestionSchema.safeParse(event);
...
```
- `createIngestionEventSchema` 팩토리를 통해 `Trace`, `Observation`, `Score` 등 각각의 이벤트 타입에 맞는 Zod 스키마를 가져옵니다.
- 만약 검증에 실패하면, 전체 배치를 거부(Reject)하는 대신 **실패한 개별 이벤트만 에러 배열(`validationErrors`)에 담고** 성공한 이벤트들은 다음 단계로 통과시킵니다. (부분 성공 지원)

### B. S3 Slowdown 에러 폴백 (Lines 227-264)
```typescript
const results = await Promise.allSettled(
  Object.keys(sortedBatchByEventBodyId).map(async (id) => {
    // S3 업로드 로직 (getS3StorageServiceClient)
  })
);

results.forEach((result) => {
  if (result.status === "rejected") {
    s3UploadErrored = true;
    if (isS3SlowDownError(result.reason)) {
      markProjectS3Slowdown(authCheck.scope.projectId!).catch(() => {});
    }
  }
});
```
- S3는 초당 PUT 요청 수가 3,500회(Prefix 당)를 초과하면 `503 Slow Down` 에러를 반환할 수 있습니다.
- `isS3SlowDownError`로 이 상황을 캐치하면, 해당 `projectId`를 레디스 캐시에 `Slowdown` 상태로 마킹합니다.
- 이렇게 마킹된 프로젝트의 후속 요청은 Ingestion 큐 안에서도 `SecondaryIngestionQueue`라는 지연 큐로 강제 라우팅되어 메인 큐의 병목을 막습니다.

### C. Sampling 전략 (Lines 300-312)
```typescript
const { isSampled, isSamplingConfigured } = isTraceIdInSample({
  projectId: authCheck.scope.projectId,
  event: eventData.data[0],
});

if (!isSampled) {
  recordIncrement("langfuse.ingestion.sampling", eventData.data.length, {
    projectId: authCheck.scope.projectId ?? "<not set>",
    sampling_decision: "out",
  });
  return;
}
```
- 시스템 설정이나 프로젝트 설정에 따라 확률 기반 샘플링이 구성되어 있다면, `isTraceIdInSample` 함수에서 `traceId`의 해시값을 기반으로 통과 여부(`isSampled`)를 결정합니다.
- 탈락된 이벤트는 Redis 큐(`IngestionQueue.add`)에 인큐잉되지 않고 바로 버려집니다.
