# 11 Open Decisions

이 문서는 Langfuse 사내 도입 및 커스터마이징 과정에서 아직 확정되지 않은 정책과 기술적 결정 사항들을 추적합니다. 결정이 완료되면 이 문서에 기록된 항목을 갱신하고 관련 설계 문서(Architecture, Data Model 등)에 반영해야 합니다.

## 진행 중인 결정 사항 (Active Decisions)

### 1. ClickHouse 배포 아키텍처 (Clustered vs Unclustered)
- **배경**: Langfuse는 `unclustered` (단일 노드)와 `clustered` (다중 노드 클러스터) ClickHouse 마이그레이션을 모두 지원합니다.
- **고려 사항**:
  - 사내 LLM 트래픽 규모(일일 요청 건수)에 따라 단일 노드로 충분할지, 초기부터 Zookeeper/ClickHouse Keeper 기반의 클러스터링을 구성할지 결정 필요.
  - 클러스터링 적용 시 인프라 관리 복잡도 및 리소스 비용 상승.
- **상태**: 논의 중 (사내 트래픽 수요 조사 필요)

### 2. 비용 청구 시스템 (Chargeback) 정밀도
- **배경**: 사내 프라이빗 모델과 외부 API(OpenAI 등)를 혼용할 때, 부서별로 비용을 청구(Chargeback)하는 로직이 필요합니다.
- **고려 사항**:
  - 단순히 Token 수를 기반으로 비용을 계산할지, 시스템 인프라 유지 비용을 1/n로 나누어 과금할지 결정.
  - Langfuse 내부의 Price 테이블만 사용할지, 별도의 사내 ERP/비용 청구 시스템 API를 호출할지 결정.
- **상태**: 재무/운영팀과 협의 중

### 3. 데이터 보존 주기 (Retention Policy)
- **배경**: PII(개인식별정보)가 포함될 수 있는 프롬프트 데이터(Observation)의 저장 기한을 결정해야 합니다.
- **고려 사항**:
  - 컴플라이언스 규정에 따른 민감 데이터 삭제 의무 (예: 90일 보관).
  - 전체 삭제를 할 것인가, 프롬프트 페이로드만 삭제(마스킹)하고 메타데이터(토큰 수, 응답 시간 등)는 영구 보존할 것인가.
- **상태**: 사내 보안/컴플라이언스팀 확인 대기

### 4. 프론트엔드 Admin 권한 관리의 분리
- **배경**: 시스템 전체를 관리하는 Super Admin 권한과, 개별 프로젝트 내의 권한(Owner, Viewer 등) 관리가 필요합니다.
- **고려 사항**:
  - Langfuse의 `Organization` 단위 관리 기능을 확장하여 사내 그룹웨어 권한 체계와 동기화할지.
  - 별도의 Admin 전용 백오피스 UI를 Langfuse Web에 라우팅으로 추가할지, 분리된 외부 백오피스 툴을 구성할지 결정.
- **상태**: 논의 중

## 완료된 결정 사항 (Resolved Decisions)

> *현재 완료된 결정 사항 없음.*
> *결정이 완료될 경우 여기에 날짜, 결정 내용, 그리고 이유를 기록합니다.*

---

| | |
|---|---|
| ⬅️ 이전 | [13 Customization Source Breakdown](../50-source-analysis/13-customization-source-breakdown.md) |
| 🏠 색인 | [README](../README.md) |
