# Chapter 7. 인증 아키텍처와 사내 SSO 연동

> *"16 built-in providers, all activated by environment variables alone."*

---

## 7.1 설계 질문

**다양한 IdP(OIDC, SAML, OAuth2)를 지원하면서, 첫 로그인 시 사용자를 올바른 프로젝트에 자동으로 배치하려면 어떻게 해야 하는가?**

## 7.2 인증 아키텍처 전체 흐름

🔗 [`web/src/server/auth.ts`](file:///Users/dhsshin/Documents/LLMOps/langfuse/web/src/server/auth.ts)

```mermaid
sequenceDiagram
    participant User as 사내 직원
    participant Browser as 브라우저
    participant LF as Langfuse (NextAuth)
    participant IdP as 사내 IdP (Keycloak 등)
    participant PG as PostgreSQL

    User->>Browser: langfuse.company.internal 접속
    Browser->>LF: GET /api/auth/signin
    LF-->>Browser: 로그인 페이지 (SSO 버튼 표시)

    Browser->>LF: SSO 로그인 클릭
    LF->>IdP: OIDC Authorization Request
    IdP-->>User: 사내 인증 화면
    User->>IdP: 아이디/비밀번호 입력
    IdP->>LF: Authorization Code + ID Token

    LF->>LF: Token 검증 + 프로필 추출

    alt 신규 사용자 (DB에 없음)
        LF->>PG: extendedPrismaAdapter.createUser()
        PG-->>LF: User 생성 완료
        LF->>PG: createProjectMembershipsOnSignup()
        Note over PG: 1. LANGFUSE_DEFAULT_ORG_ID 조직에 참여<br/>2. LANGFUSE_DEFAULT_PROJECT_ID 프로젝트에 참여<br/>3. 대기 중인 초대장(Invitation) 처리
    else 기존 사용자
        LF->>PG: extendedPrismaAdapter.linkAccount()
        LF->>PG: createProjectMembershipsOnSignup()
        Note over PG: 기존 멤버십 유지 (upsert, no-op)
    end

    LF-->>Browser: JWT 세션 토큰 (쿠키)
    Browser-->>User: 대시보드 표시
```

## 7.3 16개 빌트인 Provider — 환경변수 기반 활성화

🔗 [`auth.ts` L89-607](file:///Users/dhsshin/Documents/LLMOps/langfuse/web/src/server/auth.ts#L89-L607)

```mermaid
flowchart TD
    subgraph Generic["범용 OIDC (사내 추천)"]
        Custom["CustomSSOProvider<br/>AUTH_CUSTOM_*"]
        Keycloak["KeycloakProvider<br/>AUTH_KEYCLOAK_*"]
    end

    subgraph Enterprise["엔터프라이즈 IdP"]
        Okta["OktaProvider"]
        Auth0["Auth0Provider"]
        Azure["AzureADProvider"]
        Cognito["CognitoProvider"]
        OneLogin["OneLoginProvider"]
        JumpCloud["JumpCloudProvider"]
        Authentik["AuthentikProvider"]
        WorkOS["WorkOSProvider"]
    end

    subgraph DevPlatform["개발자 플랫폼"]
        GitHub["GitHubProvider"]
        GitHubE["GitHubEnterpriseProvider"]
        GitLab["GitLabProvider"]
        Google["GoogleProvider"]
        WordPress["WordPressProvider"]
    end

    subgraph Basic["기본 인증"]
        Cred["CredentialsProvider<br/>(이메일/비밀번호)"]
        Email["EmailProvider<br/>(OTP 비밀번호 재설정)"]
    end
```

### 사내 연동 — 최소 설정 가이드

**CustomSSOProvider** (가장 범용적):

```bash
# .env
AUTH_CUSTOM_CLIENT_ID=langfuse-internal
AUTH_CUSTOM_CLIENT_SECRET=$(vault kv get secret/langfuse/oidc)
AUTH_CUSTOM_ISSUER=https://sso.company.internal/realms/main
AUTH_CUSTOM_NAME="사내 SSO"

# 선택사항
AUTH_CUSTOM_SCOPE="openid email profile groups"
AUTH_CUSTOM_ALLOW_ACCOUNT_LINKING=true
AUTH_DISABLE_USERNAME_PASSWORD=true   # SSO 강제
```

**Keycloak 전용** (더 세밀한 제어):

```bash
AUTH_KEYCLOAK_CLIENT_ID=langfuse
AUTH_KEYCLOAK_CLIENT_SECRET=...
AUTH_KEYCLOAK_ISSUER=https://keycloak.company.internal/realms/main
AUTH_KEYCLOAK_SCOPE="openid email profile"
```

## 7.4 자동 프로비저닝 — 첫 로그인 시 멤버십 부여

🔗 [`createProjectMembershipsOnSignup.ts`](file:///Users/dhsshin/Documents/LLMOps/langfuse/web/src/features/auth/lib/createProjectMembershipsOnSignup.ts#L9-L239)

```mermaid
stateDiagram-v2
    [*] --> CheckDefaultOrg : createProjectMembershipsOnSignup()

    state CheckDefaultOrg {
        [*] --> FetchOrgs : LANGFUSE_DEFAULT_ORG_ID (쉼표 구분)
        FetchOrgs --> LoopOrgs : 각 조직에 대해
        LoopOrgs --> OrgUpsert : organizationMembership.upsert<br/>(role: LANGFUSE_DEFAULT_ORG_ROLE)
    }

    CheckDefaultOrg --> CheckDefaultProject

    state CheckDefaultProject {
        [*] --> FetchProjects : LANGFUSE_DEFAULT_PROJECT_ID (쉼표 구분)
        FetchProjects --> CheckEntitlement : rbac-project-roles 기능 확인
        CheckEntitlement --> ProjUpsert : projectMembership.upsert<br/>(role: LANGFUSE_DEFAULT_PROJECT_ROLE)
    }

    CheckDefaultProject --> ProcessInvitations

    state ProcessInvitations {
        [*] --> FindByEmail : membershipInvitation.findMany(email)
        state check_invites <<choice>>
        FindByEmail --> check_invites
        check_invites --> Transaction : 초대 있음
        Transaction --> CreateMembership : 멤버십 생성
        Transaction --> DeleteInvite : 초대장 삭제
        check_invites --> Done : 초대 없음
    }

    ProcessInvitations --> [*]
```

| 환경변수 | 기본값 | 동작 |
|---|---|---|
| `LANGFUSE_DEFAULT_ORG_ID` | — | 쉼표로 구분된 조직 ID 목록, 신규 사용자 자동 참여 |
| `LANGFUSE_DEFAULT_ORG_ROLE` | `VIEWER` | 자동 부여 역할 (ADMIN, MEMBER, VIEWER) |
| `LANGFUSE_DEFAULT_PROJECT_ID` | — | 쉼표로 구분된 프로젝트 ID 목록 |
| `LANGFUSE_DEFAULT_PROJECT_ROLE` | `VIEWER` | 자동 부여 역할 |

## 7.5 보안 강화 옵션

```mermaid
flowchart TD
    subgraph Lockdown["사내 보안 강화 설정"]
        D1["AUTH_DISABLE_USERNAME_PASSWORD=true<br/>→ 이메일/비밀번호 로그인 완전 차단"]
        D2["AUTH_DISABLE_SIGNUP=true<br/>→ 초대 전용 모드"]
        D3["LANGFUSE_ALLOWED_ORGANIZATION_CREATORS<br/>=admin@company.com<br/>→ 조직 생성 권한 제한"]
        D4["AUTH_SESSION_MAX_AGE=60<br/>→ 세션 1시간 제한"]
        D5["NODE_EXTRA_CA_CERTS=/etc/ssl/internal-ca.pem<br/>→ 사내 자체 서명 인증서"]
    end
```

### Keycloak 호환성 패치

🔗 [`auth.ts` L645-649](file:///Users/dhsshin/Documents/LLMOps/langfuse/web/src/server/auth.ts#L645-L649)

```typescript
if (data.provider.endsWith("keycloak")) {
  delete data["refresh_expires_in"];     // NextAuth 스키마에 없는 필드
  delete data["not-before-policy"];      // NextAuth 스키마에 없는 필드
}
```

> Keycloak이 반환하는 비표준 필드가 NextAuth의 Prisma Adapter와 충돌하는 알려진 이슈([#7655](https://github.com/nextauthjs/next-auth/issues/7655))를 Langfuse가 자체적으로 우회한다.

---

## 이 챕터의 핵심 인사이트

1. **환경변수만으로 16개 SSO Provider 활성화** — 코드 수정 불필요
2. **자동 프로비저닝이 "첫 로그인 = 즉시 대시보드"**를 가능하게 함
3. **Keycloak 호환성 패치가 이미 내장**되어 있으므로 별도 작업 불필요
4. **`AUTH_DISABLE_USERNAME_PASSWORD=true`로 SSO 강제**가 가장 중요한 보안 설정

---

| 방향 | 문서 |
|---|---|
| ⬅️ 이전 | [Ch.6 쿼리 경로](./ch06-query-path.md) |
| ➡️ 다음 | [Ch.8 비용 산정 엔진](./ch08-cost-engine.md) |
| 🏠 백서 색인 | [WHITEPAPER.md](../WHITEPAPER.md) |
