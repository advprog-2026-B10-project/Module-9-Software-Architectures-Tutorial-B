## Authentication Module

Below are diagrams and descriptions for the BidMart authentication module (controllers, services, repositories, DTOs, and security components).

### Component Diagram

```mermaid
graph TB
  subgraph external ["External Actors"]
    FE["🌐 Next.js Frontend\n:3000"]
    EmailAPI["📧 Email Provider\nSMTP/API"]
    OtherModules["📦 Other Modules\nAuction, Catalog, Order"]
  end

  subgraph authmodule ["Authentication Module"]
    direction TB

    subgraph controllers ["Controllers"]
      AC["🔐 AuthController\n/api/auth/**\nPOST /register, POST /login, POST /mfa/verify, POST /refresh, POST /logout"]
      UC["👤 UserController (profile)\nGET /profile, PATCH /profile"]
    end

    subgraph services ["Services"]
      AS["⚙️ AuthService\nregister(), login(), verifyMfa(), refreshTokens(), logout(), toggleMfa()"]
      JWT["🔑 JwtService\ngenerateToken(), generateRefreshToken(), isTokenValid()"]
      MFA["🔐 MfaTotpService\ngenerateSecret(), verifyCode(), buildOtpAuthUri()"]
      EMAIL["✉️ EmailService\nsendVerificationEmail()"]
    end

    subgraph security ["Security Layer"]
      FILTER["🛡️ JwtAuthenticationFilter\nextracts Bearer token, sets SecurityContext"]
      ENC["🔒 BCryptPasswordEncoder\npassword hashing/verifying"]
    end

    subgraph repository ["Repository"]
      UR["🗄️ UserRepository (JPA)\nfindByEmail(), findByVerificationToken(), existsByEmail()"]
      RR["🗄️ RefreshTokenRepository\nfindByToken(), deleteByEmail(), revokeActiveSessionsByEmail()"]
    end

    subgraph model ["Model / Entities"]
      USER["📋 User Entity\nemail, password, role, mfaSecret, verificationToken, enabled"]
      RT["📋 RefreshToken Entity\ntoken, email, expiresAt, revoked"]
    end
  end

  FE -->|"REST / WS"| AC
  FE -->|"REST (JWT)"| UC
  AC -->|"delegates"| AS
  UC -->|"delegates"| AS
  AS -->|"generate/validate"| JWT
  AS -->|"verify code"| MFA
  AS -->|"send verification"| EMAIL
  AS -->|"CRUD"| UR
  AS -->|"store refresh"| RR
  FILTER -->|"load user"| UR
  OtherModules -.->|"(currently) direct import / uses UserRepository"| UR
  UR -->|"JPA / JDBC"| DB[("🗄️ PostgreSQL")] 

  style authmodule fill:#1E293B,color:#E2E8F0,stroke:#475569
  style controllers fill:#312E81,color:#C7D2FE,stroke:#6366F1
  style services fill:#374151,color:#F8FAFC,stroke:#4B5563
  style repository fill:#0EA5E9,color:#fff,stroke:#0284C7
  style model fill:#14532D,color:#BBF7D0,stroke:#22C55E
```

### Ringkasan Interaksi (endpoints → internal calls)

| From | To | Interaksi | Protokol |
|---|---|---|---|
| Frontend | `AuthController` | `POST /api/auth/register` → `AuthService.register()` (send verification email) | REST/JSON |
| Frontend | `AuthController` | `POST /api/auth/login` → `AuthService.login()` (may return MFA challenge token) | REST/JSON |
| Frontend | `AuthController` | `POST /api/auth/mfa/verify` → `AuthService.verifyMfa()` (verify TOTP, then issue tokens) | REST/JSON |
| Frontend | `AuthController` | `POST /api/auth/refresh` → `AuthService.refreshTokens()` (validate persisted refresh token) | REST/JSON |
| Frontend | `AuthController` | `POST /api/auth/logout` → `AuthService.logout()` (delete refresh tokens) | REST/JSON |
| Admin Frontend | `AuthController` | `GET /api/auth/users` → `AuthService.getAdminUsers()` (includes active session counts via `RefreshTokenRepository`) | REST/JSON |
| Admin Frontend | `AuthController` | `POST /api/auth/admin/users/{id}/sessions/revoke` → `AuthService.revokeUserSessions()` | REST/JSON |
| Any Service | `JwtAuthenticationFilter` | Extract Bearer token → `JwtService.isTokenValid()` and `UserRepository.findByEmail()` | Servlet Filter |

### Class Diagram — core types

```mermaid
classDiagram
  class AuthService {
    +register(RegisterRequest request)
    +login(LoginRequest request) AuthResponse
    +verifyMfa(MfaVerifyRequest request) AuthResponse
    +refreshTokens(String refreshToken) AuthResponse
    +logout(String email)
    +toggleMfa(String email, boolean enabled) MfaStatusResponse
  }

  class JwtService {
    +generateToken(User user) String
    +generateRefreshToken(User user) String
    +isTokenValid(String token) boolean
    +extractEmail(String token) String
    +getAccessTokenExpiration() long
  }

  class User {
    -Long id
    -String email
    -String password
    -String displayName
    -Role role
    -boolean enabled
    -String verificationToken
    -boolean mfaEnabled
    -String mfaSecret
  }

  class RefreshToken {
    -String token
    -String email
    -Instant expiresAt
    -boolean revoked
  }

  AuthService --> User
  AuthService --> RefreshToken
  AuthService --> JwtService
```

### Sequence Diagram — Login + MFA flow

```mermaid
sequenceDiagram
  actor User as Pengguna
  participant FE as Frontend
  participant AC as AuthController
  participant AS as AuthService
  participant UR as UserRepository
  participant PE as BCryptPasswordEncoder
  participant MFA as MfaTotpService
  participant JWT as JwtService
  participant RR as RefreshTokenRepository

  FE->>AC: POST /api/auth/login {email, password}
  AC->>AS: login(request)
  AS->>UR: findByEmail(email)
  UR-->>AS: User
  AS->>PE: matches(password, user.password)
  alt password invalid
    AS-->>AC: throw AuthException(401)
  else password valid
    alt user.mfaEnabled == true
      AS-->>AC: return AuthResponse {mfaRequired: true, mfaChallengeToken}
      FE->>AC: POST /api/auth/mfa/verify {challengeToken, code}
      AC->>AS: verifyMfa(request)
      AS->>MFA: verifyCode(user.mfaSecret, code)
      MFA-->>AS: verified
      AS->>JWT: generateToken(user)
      AS->>JWT: generateRefreshToken(user)
      AS->>RR: save(refreshToken)
      AS-->>AC: AuthResponse {token, refreshToken}
    else
      AS->>JWT: generateToken(user)
      AS->>JWT: generateRefreshToken(user)
      AS->>RR: save(refreshToken)
      AS-->>AC: AuthResponse {token, refreshToken}
    end
  end
```

### Sequence Diagram — Refresh tokens

```mermaid
sequenceDiagram
  participant FE as Frontend
  participant AC as AuthController
  participant AS as AuthService
  participant RR as RefreshTokenRepository
  participant JWT as JwtService

  FE->>AC: POST /api/auth/refresh {refreshToken}
  AC->>AS: refreshTokens(refreshToken)
  AS->>RR: findByToken(refreshToken)
  RR-->>AS: RefreshToken entity
  AS->>JWT: isTokenValid(refreshToken)
  AS->>JWT: generateToken(user)
  AS-->>AC: AuthResponse {token, refreshToken}
```

### Notes & Migration guidance

- The current monolith stores `User` and `RefreshToken` in the same DB; when extracting Auth as a service, keep refresh tokens private to Auth's datastore.
- Replace direct repository imports from other modules with published events (e.g. `user.banned`) or a service API to avoid tight coupling.
- Continue to use short-lived JWTs for clients and persist refresh tokens server-side for session control.

---
