# 01 Requirements and Scope

## 배경

현재 사내에 산재되어 있는 LLM 애플리케이션 개발, 프롬프트 엔지니어링, LLM 호출 비용 추적, 평가(Evaluation) 등의 관리를 일원화하기 위해 강력한 Observability 플랫폼이 필요해졌습니다. 이를 위해 오픈소스 플랫폼인 **Langfuse**를 도입하고, 내부 인프라 구조 및 사내 보안 규정에 맞게 커스터마이징하여 전사적인 사내 LLM Observability 플랫폼으로 활용하고자 합니다.

## 핵심 요구사항 (Goals)

Langfuse를 도입하고 운영하기 위한 주요 커스터마이징 요구사항은 다음과 같습니다.

### 1. 보안 및 인증 (Security & Authentication)
- **SSO 연동**: 기존 Langfuse의 NextAuth 기반 자체 가입/인증 대신, 사내 SSO(OIDC, SAML 등) 시스템을 연동하여 중앙 집중식 사용자 관리가 가능해야 합니다.
- **권한 관리 (RBAC)**: 사내 조직도와 연동되는 강력한 Role-Based Access Control을 구현하여 Organization 및 Project 수준의 접근을 엄격히 제어해야 합니다.

### 2. 비용 추적 및 커스텀 모델 지원
- **내부 모델 비용 산정**: 사내에 구축된 프라이빗 LLM이나 별도 계약된 전용 모델에 대한 사용량 및 비용을 추적할 수 있도록 커스텀 모델과 가격을 정의할 수 있어야 합니다.
- **Model Pricing 확장**: `worker/src/constants` 및 `packages/shared/src/server/llm/types.ts` 등의 가격 테이블을 유연하게 주입하거나 사내 토큰 비용 체계를 따르도록 수정해야 합니다.

### 3. 데이터 보존 및 아키텍처 스케일링
- **분석 성능 최적화**: 대규모 트래픽 발생 시 웹 노드에 부하를 주지 않도록 모든 Ingestion 작업은 비동기 큐(BullMQ)와 Worker로 분리되어야 합니다.
- **ClickHouse 기반의 Wide Event Data**: Observability 데이터의 고차원(High-cardinality) 조회를 위해 ClickHouse를 적극적으로 활용하며, 보존 주기(Retention Policy)를 사내 데이터 규정에 맞게 조정해야 합니다.
- **Prisma & PostgreSQL**: 유저 및 권한, 모델 메타데이터 등 트랜잭션 데이터의 정합성을 보장합니다.

## 범위 제외 (Non-Goals)

다음 사항은 본 커스터마이징 프로젝트의 범위에서 제외됩니다.
- Langfuse의 핵심 도메인 모델(Trace, Observation, Score 등)의 스키마 자체를 변경하는 것. (기존 스키마와 데이터 수집 파이프라인 구조는 최대한 오픈소스 표준을 유지합니다.)
- Python/JS SDK의 직접적인 수정. (기존에 제공되는 호환 SDK를 그대로 활용합니다.)
- 퍼블릭 클라우드를 위한 SaaS 멀티테넌시 과금 모듈 개발 (사내 도입용이므로 외부 결제 기능은 제외합니다.)

## 성공 기준
1. 사내 SSO 로그인을 통해 Organization 및 Project가 자동으로 프로비저닝되는가.
2. 초당 대규모 LLM Trace/Observation Ingestion 트래픽이 Worker 큐를 거쳐 ClickHouse에 유실 없이 저장되는가.
3. 사내 프라이빗 모델 사용 시 토큰당 비용이 대시보드에 정상적으로 합산되어 출력되는가.

---

| | |
|---|---|
| ➡️ 다음 | [03 Architecture](../10-architecture/03-architecture.md) |
| 🏠 색인 | [README](../README.md) |
