# 15 Source Code Architecture

이 문서는 Langfuse 모노레포(Monorepo)의 디렉토리 구조, 패키지 간의 의존성(Dependency Rule) 및 핵심 기술 스택을 설명합니다.

## 시스템 경계 및 기술 스택

Langfuse는 [Turborepo](https://turbo.build/repo)를 기반으로 하는 pnpm workspace 모노레포 구조를 가집니다.

| 영역 | 스택 | 설명 |
| --- | --- | --- |
| 패키지 관리 | pnpm, Turborepo | 빠른 의존성 설치 및 모노레포 태스크 체이닝(lint, build 등). |
| 프론트엔드 | React 18, Next.js (App/Pages Router 혼용), TailwindCSS | Admin UI 셸 및 뷰 컴포넌트 담당. |
| 백엔드 (Web) | Next.js API Routes, tRPC | 클라이언트-서버 간 타입 안전(Type-safe) 통신 및 데이터 제공. |
| 백엔드 (Worker)| Node.js, BullMQ, TypeScript | 메시지 큐 처리를 위한 독립 워커 실행 환경. |
| 데이터베이스 | Prisma, ClickHouse Node Client | Postgres ORM 및 ClickHouse 쿼리 실행. |
| API 문서 | Fern | `fern/` 디렉토리의 정의 파일을 통해 API 명세서 및 클라이언트 생성. |

## 프로젝트 디렉토리 구조

```text
langfuse/
├─ web/                     # Next.js 애플리케이션 (프론트엔드 UI + tRPC + 퍼블릭 REST API)
├─ worker/                  # BullMQ 기반의 백그라운드 큐 소비자 프로세스
├─ packages/
│  └─ shared/               # 공통 도메인 모델, DB 스키마, 큐 계약(Contracts), 리포지토리
├─ ee/                      # Enterprise 에디션 전용 기능 (SSO 연동, 고급 감사 로그 등)
├─ generated/               # Fern을 통해 자동 생성된 API 클라이언트 (직접 수정 금지)
├─ fern/                    # REST API 명세 정의 파일 소스
└─ scripts/                 # 개발 및 배포 환경 세팅 스크립트
```

## 의존성 규칙 (Dependency Rules)

의존성 흐름은 반드시 단방향을 유지해야 하며, 하위 계층(공통 패키지)이 상위 계층(애플리케이션)을 참조해서는 안 됩니다.

- **`web`**: `@langfuse/shared`와 `@langfuse/ee`에 의존합니다. 프론트엔드와 API 서버 역할을 동시 수행합니다.
- **`worker`**: `@langfuse/shared`에 의존합니다. `web`의 코드를 직접 임포트할 수 없습니다.
- **`@langfuse/ee`**: 엔터프라이즈 전용 패키지로, `@langfuse/shared`에 의존합니다.
- **`@langfuse/shared`**: 가장 기저에 있는 공통 모듈입니다. **`web`, `worker`, `ee`의 어떠한 코드도 참조해서는 안 됩니다.**

```mermaid
flowchart TD
    W[web] --> EE[@langfuse/ee]
    W --> S[@langfuse/shared]
    EE --> S
    WK[worker] --> S
```

## `packages/shared` 패키지 세부 구조

`packages/shared`는 전체 시스템에서 가장 중요한 코어 계약(Core Contracts)을 가지고 있습니다.

- `src/server/queues.ts`: `web`이 Redis에 Job을 넣고 `worker`가 Job을 빼갈 때 서로 동일한 타입과 큐 이름을 사용할 수 있도록 강제하는 페이로드 스키마가 정의되어 있습니다.
- `src/domain/{observations,traces,scores}.ts`: 시스템의 핵심 도메인 모델들이 위치합니다.
- `prisma/schema.prisma`: 전체 관계형 데이터베이스(Postgres) 스키마.
- `clickhouse/migrations/`: ClickHouse 데이터베이스 마이그레이션 SQL 스크립트.

## 빌드 및 검증 스크립트 가이드

모노레포 내에서 작업을 진행할 때 다음 스크립트들을 사용합니다.

- **의존성 설치**: `pnpm install`
- **로컬 서버 실행**: `pnpm run dev` (web과 worker가 동시에 실행됨)
- **린트 검사**: `pnpm run lint`
- **타입 검사**: `pnpm run typecheck` (`pnpm tc`)
- **DB 스키마 생성**: `pnpm run db:generate` (Prisma 스키마 변경 시 필수 실행)
- **API 클라이언트 생성**: Fern 정의 변경 시 자동화 파이프라인을 통해 생성되며 `generated/` 폴더는 직접 수정하지 않습니다.
