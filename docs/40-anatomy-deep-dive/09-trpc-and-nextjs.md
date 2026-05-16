# 09 tRPC and Next.js Anatomy

Langfuse의 Web UI 셸(프론트엔드)과 내부 API 통신은 **Next.js** 프레임워크와 **tRPC (TypeScript Remote Procedure Call)** 를 통해 이루어집니다. 이 문서에서는 프론트엔드에서 데이터를 요청하여 받아오기까지의 해부학적 구조를 분석합니다.

## 1. tRPC 라우터 구조 (`web/src/server/api/routers`)

Langfuse는 타입 안정성(Type-Safety)을 극대화하기 위해 REST API 대신 tRPC를 주력으로 사용합니다. 클라이언트(React)와 서버(Node.js) 간에 타입을 공유하여 빌드 타임에 에러를 잡습니다.

- **위치**: `web/src/server/api/routers/` 하위에 도메인별 라우터(`traces.ts`, `projects.ts`, `models.ts` 등)가 분리되어 있습니다.
- **프로시저(Procedure) 체인**: 모든 API 요청은 tRPC 미들웨어를 통과하며 인증 및 인가(Authorization)를 확인합니다.

```typescript
// 예시: tRPC 라우터 정의 구조
export const traceRouter = createTRPCRouter({
  all: protectedProjectProcedure
    .input(z.object({ projectId: z.string(), page: z.number() }))
    .query(async ({ input, ctx }) => {
      // 1. ctx.session에서 유저 정보 확인 완료 (protectedProjectProcedure가 처리)
      // 2. ClickHouse에서 Trace 데이터 조회
      const data = await clickhouseClient.query(...);
      return data;
    }),
});
```

### `protectedProjectProcedure`의 역할
가장 많이 사용되는 미들웨어로, 다음 작업을 수행합니다.
1. 사용자가 로그인한 상태인지 확인 (NextAuth Session).
2. 요청한 `projectId`에 대해 해당 사용자가 접근 권한(Viewer, Admin 등)이 있는지 Prisma (PostgreSQL)의 `ProjectMembership` 테이블을 조회하여 인가(Authorization).
3. 권한이 확인된 사용자 정보와 프로젝트 식별자를 컨텍스트(`ctx`)에 담아 비즈니스 로직으로 넘김.

## 2. Prisma와 ClickHouse 병렬 데이터 패칭

대시보드를 렌더링할 때, 유저의 프로젝트 설정이나 메타데이터는 **PostgreSQL(Prisma)**에서, 실제 Trace나 메트릭 데이터는 **ClickHouse**에서 읽어야 합니다. tRPC 리졸버 내부에서는 속도 최적화를 위해 이 두 작업을 병렬(Parallel)로 처리하는 경우가 많습니다.

```typescript
// 병렬 호출 패턴
const [projectMeta, traceData] = await Promise.all([
  ctx.prisma.project.findUnique({ where: { id: input.projectId } }),
  ctx.clickhouse.query({
    query: `SELECT * FROM traces_7d_amt_mv WHERE project_id = {projectId: String}`,
    format: 'JSONEachRow',
    query_params: { projectId: input.projectId }
  })
]);
```

## 3. Server Components vs Client Components

Langfuse는 Next.js App Router의 기능을 적극 활용합니다.
- **Server Components**: 초기 페이지 로딩 시 검색 엔진 최적화(SEO)가 필요 없는 대시보드 구조라도, 번들 사이즈를 줄이고 서버에서 데이터를 직결로 가져오기 위해 레이아웃이나 정적 테이블을 서버 사이드에서 렌더링합니다.
- **Client Components**: 사용자와 상호작용이 많은 필터 조건 변경, 차트 렌더링, 폼 입력 등의 영역은 `useQuery`(tRPC React Hooks)를 사용하여 브라우저에서 동적으로 데이터를 fetching하고 상태를 업데이트합니다.

이러한 구조 덕분에 사내 커스텀 UI(예: 사내 결재 연동, 부서별 커스텀 뷰)를 추가할 때에도, 백엔드의 `routers` 폴더에 프로시저만 하나 추가하면 프론트엔드에서 타입이 즉시 자동 완성되어 매우 빠른 개발 사이클을 유지할 수 있습니다.
