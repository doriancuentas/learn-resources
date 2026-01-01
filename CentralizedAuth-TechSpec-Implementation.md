# Centralized Authentication Infrastructure - Technical Specification
## Implementation-Ready Document

> **Status**: Ready for Implementation  
> **Version**: 1.0  
> **Last Updated**: January 2026  
> **Prerequisites**: [CentralizedAuthenticationInfra.md](./CentralizedAuthenticationInfra.md), [SSO-CrossCuttingOWASP](./SSO-CrossCuttingOWASP)

---

# Executive Summary

This document provides a comprehensive, implementation-ready technical specification for deploying a production-grade centralized authentication infrastructure serving multiple web properties across different top-level domains. It delivers detailed architecture diagrams, configuration templates, deployment procedures, security controls, and operational runbooks—everything needed to transition from per-application authentication silos to a unified [Single Sign-On (SSO)](#glossary) system.

The technical foundation leverages the [Ory](#glossary) open-source identity stack ([Kratos](#glossary) for identity management, [Hydra](#glossary) for [OIDC](#glossary) authorization, [Oathkeeper](#glossary) for access control) deployed via [Docker Compose](#glossary) on a single [VPS](#glossary) (4 [vCPU](#glossary), 16GB [RAM](#glossary)). [Infisical](#glossary) provides self-hosted secrets management, [Traefik](#glossary) handles ingress with automatic [TLS](#glossary) certificates, and [Cloudflare](#glossary) delivers edge security ([WAF](#glossary), [DDoS](#glossary) protection, [Zero Trust](#glossary) access). The [Postgres](#glossary) database stores identity data separately from application databases, while [Redis](#glossary) provides session caching for fast validation. This architecture supports 3+ domains (bumerange.com, rumnia.com, coyotee-renato.cloud) with true cross-domain [SSO](#glossary) via [OAuth2](#glossary)/[OIDC](#glossary) flows.

The implementation scope covers scaling from 100 to thousands of users while maintaining [OWASP](#glossary) security standards, including [PKCE](#glossary)-enforced authorization flows, [MFA](#glossary) support, session management best practices, and comprehensive rate limiting. The document includes five implementation phases spanning 4-5 weeks, from infrastructure setup through security hardening, with detailed configuration files, deployment commands, testing procedures, and disaster recovery protocols. Cost projections remain in the $20-50/month range leveraging free tiers and open-source components.

---

# Table of Contents

1. [Environment Context](#1-environment-context)
2. [Target Architecture](#2-target-architecture)
3. [Infrastructure Layout](#3-infrastructure-layout)
4. [Secret Management with Infisical](#4-secret-management-with-infisical)
5. [Component Deployment Specs](#5-component-deployment-specs)
6. [Cross-Domain SSO Implementation](#6-cross-domain-sso-implementation)
7. [OWASP Security Controls](#7-owasp-security-controls)
8. [Cloudflare Integration](#8-cloudflare-integration)
9. [Implementation Phases](#9-implementation-phases)
10. [Operational Runbook](#10-operational-runbook)
11. [Cost Analysis](#11-cost-analysis)
12. [Risk Mitigation](#12-risk-mitigation)

---

# 1. Environment Context

## 1.1 Available Resources

| Resource | Specs | Role |
|----------|-------|------|
| **Hostinger KVM [VPS](#glossary)** | 4 [vCPU](#glossary), 16GB [RAM](#glossary), 200GB [NVMe](#glossary) | Primary: Auth stack, [Infisical](#glossary) |
| **[Oracle](#glossary) Free Tier** | 4 [OCPU](#glossary), 24GB [RAM](#glossary) ([ARM](#glossary)) | Backup/DR or overflow services |
| **[Cloudflare](#glossary)** | [DNS](#glossary), [CDN](#glossary), Workers, [Zero Trust](#glossary) | Edge routing, [DDoS](#glossary) protection |
| **[Supabase](#glossary) Free Tier** | 500MB [Postgres](#glossary), 1GB storage | App databases (not auth DB) |

## 1.2 Domains

| Domain | Purpose | [SSO](#glossary) Role |
|--------|---------|----------|
| `bumerange.com` | Primary product | [OIDC](#glossary) Client |
| `rumnia.com` | Secondary product | [OIDC](#glossary) Client |
| `coyotee-renato.cloud` | Tools/Admin | [OIDC](#glossary) Client |
| `auth.bumerange.com` | Identity Provider | [OIDC](#glossary) Server ([Kratos](#glossary) + [Hydra](#glossary)) |

## 1.3 Constraints

- **Team**: Small, LLM-assisted operations
- **Budget**: Free-tier focused, minimal paid services
- **Compliance**: Not required yet, but architecture should support [GDPR](#glossary)/[SOC2](#glossary) path
- **Current State**: Per-app logins, [Cloudflare](#glossary) Workers, [Zero Trust](#glossary) for internal

---

# 2. Target Architecture

```mermaid
flowchart TB
    subgraph Internet ["Internet"]
        Users["👤 Users"]
    end

    subgraph Cloudflare ["Cloudflare Edge"]
        CF_DNS["DNS"]
        CF_Proxy["Proxy/CDN"]
        CF_WAF["WAF + Rate Limit"]
        CF_ZT["Zero Trust<br/>(Admin only)"]
    end

    subgraph Hostinger ["Hostinger VPS (16GB)"]
        subgraph Docker ["Docker Compose Stack"]
            Traefik["Traefik<br/>:443 :80"]
            
            subgraph Auth ["Auth Layer"]
                Kratos["Ory Kratos<br/>Identity"]
                Hydra["Ory Hydra<br/>OIDC Server"]
                Oathkeeper["Ory Oathkeeper<br/>Access Proxy"]
            end
            
            subgraph Secrets ["Secret Management"]
                Infisical["Infisical<br/>:8080"]
            end
            
            subgraph Data ["Data Layer"]
                Postgres[("Postgres<br/>:5432")]
                Redis[("Redis<br/>:6379")]
            end
        end
    end

    subgraph Apps ["Applications (Various Hosts)"]
        App1["bumerange.com<br/>(Cloudflare Workers)"]
        App2["rumnia.com<br/>(Supabase)"]
        App3["coyotee-renato.cloud<br/>(VPS)"]
    end

    Users --> CF_DNS
    CF_DNS --> CF_Proxy
    CF_Proxy --> CF_WAF
    CF_WAF --> Traefik
    CF_ZT --> Traefik

    Traefik --> Oathkeeper
    Oathkeeper --> Kratos
    Oathkeeper --> Hydra
    Oathkeeper --> App3

    Kratos --> Postgres
    Kratos --> Redis
    Hydra --> Postgres
    Infisical --> Postgres

    App1 -.->|"OIDC"| Hydra
    App2 -.->|"OIDC"| Hydra
    App3 -.->|"OIDC"| Hydra

    style Cloudflare fill:#f6821f,stroke:#333,color:#fff
    style Auth fill:#d4edda,stroke:#28a745
    style Secrets fill:#e2d5f1,stroke:#9370db
    style Data fill:#f5f5f5,stroke:#999
```

## 2.1 Component Interaction Diagram

```mermaid
sequenceDiagram
    autonumber
    participant User
    participant CF as Cloudflare Edge
    participant TR as Traefik
    participant OK as Oathkeeper
    participant KR as Kratos
    participant HY as Hydra
    participant PG as Postgres
    participant RD as Redis

    User->>CF: HTTPS Request
    CF->>CF: WAF Check + Rate Limit
    CF->>TR: Proxy to VPS
    TR->>TR: TLS Termination
    TR->>OK: ForwardAuth Request
    OK->>RD: Check Session Cache
    alt Session in Cache
        RD-->>OK: Valid Session
        OK->>OK: Inject User Headers
        OK-->>TR: Allow + Headers
    else No Cache Hit
        OK->>KR: Validate Session
        KR->>PG: Query Identity
        PG-->>KR: User Data
        KR->>RD: Cache Session
        KR-->>OK: Session Valid
        OK-->>TR: Allow + Headers
    end
    TR->>User: Forward to Application
```

## 2.2 Authentication Flow Architecture

```mermaid
flowchart TB
    subgraph UserActions ["User Actions"]
        Login["Login Request"]
        Register["Registration"]
        MFA["MFA Enrollment"]
        Recovery["Password Recovery"]
    end

    subgraph Kratos ["Kratos - Identity Management"]
        SelfService["Self-Service Flows"]
        SessionMgmt["Session Management"]
        Identity["Identity Storage"]
        Courier["Email Delivery"]
    end

    subgraph Hydra ["Hydra - OAuth2/OIDC"]
        AuthEndpoint["Authorization Endpoint"]
        TokenEndpoint["Token Endpoint"]
        ConsentFlow["Consent Management"]
        JWKs["Key Management"]
    end

    subgraph Apps ["Applications"]
        App1["bumerange.com"]
        App2["rumnia.com"]
        App3["coyotee-renato.cloud"]
    end

    Login --> SelfService
    Register --> SelfService
    MFA --> SelfService
    Recovery --> Courier

    SelfService --> SessionMgmt
    SessionMgmt --> Identity
    Identity --> PG[(Postgres)]

    Apps --> AuthEndpoint
    AuthEndpoint --> SelfService
    AuthEndpoint --> ConsentFlow
    ConsentFlow --> TokenEndpoint
    TokenEndpoint --> JWKs
    TokenEndpoint --> Apps

    style Kratos fill:#d4edda
    style Hydra fill:#cce5ff
    style Apps fill:#fff3cd
```

## 2.3 Security Layer Stack

```mermaid
flowchart LR
    subgraph Layer1 ["Layer 1: Edge Security"]
        CF_DDoS["DDoS Protection"]
        CF_WAF["WAF Rules"]
        CF_Rate["Rate Limiting"]
        CF_Bot["Bot Detection"]
    end

    subgraph Layer2 ["Layer 2: Ingress Security"]
        TR_TLS["TLS Termination"]
        TR_Headers["Security Headers"]
        TR_Middleware["Custom Middleware"]
    end

    subgraph Layer3 ["Layer 3: Identity Security"]
        OK_Auth["Request Authentication"]
        OK_Authz["Authorization Rules"]
        OK_Headers["Context Injection"]
    end

    subgraph Layer4 ["Layer 4: Application Security"]
        App_Validation["Input Validation"]
        App_BOLA["Object-Level Auth"]
        App_Audit["Audit Logging"]
    end

    Internet --> Layer1
    Layer1 --> Layer2
    Layer2 --> Layer3
    Layer3 --> Layer4

    style Layer1 fill:#ff6b6b
    style Layer2 fill:#ffa500
    style Layer3 fill:#ffd700
    style Layer4 fill:#90ee90
```

## 2.4 Design Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Orchestration | [Docker Compose](#glossary) | Lower ops burden for small team; [K3s](#glossary) migration path exists |
| Identity Provider | [Ory](#glossary) [Kratos](#glossary) | OSS, no per-user cost, self-hosted, [OIDC](#glossary)-ready |
| [OIDC](#glossary) Server | [Ory](#glossary) [Hydra](#glossary) | Required for cross-domain [SSO](#glossary) (different [TLDs](#glossary)) |
| Access [Proxy](#glossary) | [Ory](#glossary) [Oathkeeper](#glossary) | Unified auth enforcement, header injection |
| Ingress | [Traefik](#glossary) | [Let's Encrypt](#glossary) auto, ForwardAuth, [Docker](#glossary)-native |
| Secrets | [Infisical](#glossary) | Self-hosted, free, team UI, env sync |
| Session Cache | [Redis](#glossary) | Fast session validation, rate limit counters |
| Auth Database | [Postgres](#glossary) (dedicated) | Separate from app DBs for isolation |
| Edge | [Cloudflare](#glossary) | [WAF](#glossary), [DDoS](#glossary), [DNS](#glossary), existing [Zero Trust](#glossary) |

---

# 3. Infrastructure Layout

## 3.1 Hostinger VPS Allocation

```
Total: 16GB RAM, 4 vCPU, 200GB NVMe

┌─────────────────────────────────────────────────────────────┐
│ Service                  RAM      CPU     Disk    Port     │
├─────────────────────────────────────────────────────────────┤
│ Postgres                 2GB      0.5     20GB    5432     │
│ Redis                    512MB    0.25    1GB     6379     │
│ Ory Kratos               512MB    0.5     -       4433/4434│
│ Ory Hydra                512MB    0.5     -       4444/4445│
│ Ory Oathkeeper           256MB    0.25    -       4455/4456│
│ Traefik                  256MB    0.25    -       80/443   │
│ Infisical (all-in-one)   1GB      0.5     5GB     8080     │
│ Kratos UI (custom)       256MB    0.25    -       3000     │
├─────────────────────────────────────────────────────────────┤
│ Reserved for Apps        8GB      1.5     150GB   -        │
│ OS + Buffer              2GB      -       24GB    -        │
└─────────────────────────────────────────────────────────────┘
```

## 3.2 Network Architecture

```mermaid
flowchart LR
    subgraph External ["External (Internet)"]
        CF["Cloudflare<br/>*.bumerange.com<br/>*.rumnia.com<br/>*.coyotee-renato.cloud"]
    end

    subgraph VPS ["Hostinger VPS"]
        subgraph DockerNetwork ["Docker Network: auth-net"]
            Traefik["traefik<br/>:80 :443"]
            Kratos["kratos"]
            Hydra["hydra"]
            Oathkeeper["oathkeeper"]
            Infisical["infisical"]
            Postgres["postgres"]
            Redis["redis"]
        end
    end

    CF -->|":443 (proxied)"| Traefik
    Traefik --> Oathkeeper
    Oathkeeper --> Kratos
    Oathkeeper --> Hydra
    Kratos --> Postgres
    Kratos --> Redis
    Hydra --> Postgres
    Infisical --> Postgres
```

## 3.3 DNS Configuration (Cloudflare)

| Record | Type | Value | [Proxy](#glossary) |
|--------|------|-------|-------|
| `auth.bumerange.com` | A | `<VPS_IP>` | ✅ Proxied |
| `api.bumerange.com` | A | `<VPS_IP>` | ✅ Proxied |
| `secrets.bumerange.com` | A | `<VPS_IP>` | ✅ Proxied (+ [Zero Trust](#glossary)) |
| `bumerange.com` | A | App host | ✅ Proxied |
| `rumnia.com` | A | App host | ✅ Proxied |
| `coyotee-renato.cloud` | A | App host | ✅ Proxied |

---

# 4. Secret Management with Infisical

## 4.1 Architecture

```mermaid
flowchart TB
    subgraph Infisical ["Infisical (self-hosted)"]
        UI["Web UI<br/>secrets.bumerange.com"]
        API["API Server"]
        DB[("Encrypted Store<br/>(Postgres)")]
    end

    subgraph Consumers ["Secret Consumers"]
        Docker["Docker Compose<br/>(env injection)"]
        Workers["Cloudflare Workers<br/>(CLI sync)"]
        Apps["App Deployments<br/>(SDK/CLI)"]
        CI["CI/CD<br/>(GitHub Actions)"]
    end

    UI --> API
    API --> DB

    Docker -->|"infisical run"| API
    Workers -->|"wrangler secret"| API
    Apps -->|"SDK"| API
    CI -->|"Service Token"| API
```

## 4.2 Infisical Deployment

```yaml
# docker-compose.infisical.yml
version: '3.8'

services:
  infisical:
    image: infisical/infisical:latest
    container_name: infisical
    restart: unless-stopped
    ports:
      - "8080:8080"
    environment:
      - ENCRYPTION_KEY=${INFISICAL_ENCRYPTION_KEY}
      - AUTH_SECRET=${INFISICAL_AUTH_SECRET}
      - POSTGRES_HOST=postgres
      - POSTGRES_PORT=5432
      - POSTGRES_DB=infisical
      - POSTGRES_USER=infisical
      - POSTGRES_PASSWORD=${POSTGRES_INFISICAL_PASSWORD}
      - SITE_URL=https://secrets.bumerange.com
      - SMTP_HOST=${SMTP_HOST}
      - SMTP_PORT=587
      - SMTP_FROM=${SMTP_FROM}
      - SMTP_USERNAME=${SMTP_USERNAME}
      - SMTP_PASSWORD=${SMTP_PASSWORD}
    depends_on:
      - postgres
    networks:
      - auth-net
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.infisical.rule=Host(`secrets.bumerange.com`)"
      - "traefik.http.routers.infisical.entrypoints=websecure"
      - "traefik.http.routers.infisical.tls.certresolver=letsencrypt"
      # Require Cloudflare Zero Trust for access
      - "traefik.http.routers.infisical.middlewares=cf-access@docker"
```

## 4.3 Secret Organization

```
Infisical Projects Structure:
├── auth-infrastructure/
│   ├── production/
│   │   ├── POSTGRES_PASSWORD
│   │   ├── REDIS_PASSWORD
│   │   ├── KRATOS_SECRETS_COOKIE
│   │   ├── KRATOS_SECRETS_CIPHER
│   │   ├── HYDRA_SECRETS_SYSTEM
│   │   ├── HYDRA_SECRETS_COOKIE
│   │   ├── HYDRA_OIDC_SUBJECT_IDENTIFIERS_SALT
│   │   ├── SMTP_PASSWORD
│   │   └── OAUTH_GOOGLE_CLIENT_SECRET
│   └── staging/
│       └── (mirrors production)
├── bumerange-app/
│   ├── production/
│   │   ├── OIDC_CLIENT_SECRET
│   │   ├── DATABASE_URL
│   │   └── API_KEYS...
│   └── staging/
├── rumnia-app/
│   └── ...
└── coyotee-admin/
    └── ...
```

## 4.4 Secret Injection Patterns

### Docker Compose (direct injection)
```bash
# Start services with secrets from Infisical
infisical run --env=production -- docker-compose up -d
```

### Cloudflare Workers
```bash
# Sync secrets to Cloudflare
infisical export --env=production --format=dotenv | \
  xargs -I {} wrangler secret put {}
```

### Application SDK (Node.js example)
```javascript
import { InfisicalClient } from "@infisical/sdk";

const client = new InfisicalClient({
  siteUrl: "https://secrets.bumerange.com",
  serviceToken: process.env.INFISICAL_SERVICE_TOKEN
});

const secrets = await client.getAllSecrets({
  environment: "production",
  projectId: "bumerange-app"
});
```

## 4.5 Secret Injection Flow

```mermaid
sequenceDiagram
    autonumber
    participant Dev as Developer
    participant IF as Infisical UI
    participant IFAPI as Infisical API
    participant IFDB as Encrypted Store
    participant DC as Docker Compose
    participant Container as Service Container

    Dev->>IF: Add/Update Secret
    IF->>IFAPI: Store Secret Request
    IFAPI->>IFDB: Encrypt and Save
    IFDB-->>IFAPI: Confirm
    IFAPI-->>IF: Success

    Note over DC,Container: Deployment Time

    DC->>IFAPI: infisical run command
    IFAPI->>IFDB: Fetch Secrets
    IFDB-->>IFAPI: Encrypted Secrets
    IFAPI->>IFAPI: Decrypt
    IFAPI-->>DC: Environment Variables
    DC->>Container: Start with Injected Env
    Container->>Container: Use Secrets in Runtime
```

## 4.6 Multi-Environment Secret Organization

```mermaid
flowchart TB
    subgraph Infisical ["Infisical Projects"]
        subgraph AuthInfra ["auth-infrastructure"]
            AuthProd["production env"]
            AuthStag["staging env"]
            AuthDev["development env"]
        end

        subgraph BumerangeApp ["bumerange-app"]
            BumProd["production env"]
            BumStag["staging env"]
        end

        subgraph RumniaApp ["rumnia-app"]
            RumProd["production env"]
            RumStag["staging env"]
        end
    end

    subgraph Consumers ["Secret Consumers"]
        ProdVPS["Production VPS"]
        StagingVPS["Staging VPS"]
        DevLocal["Local Development"]
        CI["CI/CD Pipeline"]
    end

    AuthProd --> ProdVPS
    AuthStag --> StagingVPS
    AuthDev --> DevLocal

    BumProd --> ProdVPS
    RumProd --> ProdVPS

    BumStag --> StagingVPS
    RumStag --> StagingVPS

    AuthProd --> CI
    BumProd --> CI

    style AuthInfra fill:#e2d5f1
    style BumerangeApp fill:#d4edda
    style RumniaApp fill:#cce5ff
```

## 4.7 Infisical Ops Tasks

| Task | Frequency | Command/Action |
|------|-----------|----------------|
| Update container | Monthly | `docker pull infisical/infisical && docker-compose up -d` |
| Backup secrets DB | Daily (cron) | `pg_dump infisical > backup.sql` |
| Rotate encryption key | Annually | Generate new key, re-encrypt (documented process) |
| Audit access logs | Weekly | Check UI → Audit Logs |
| Revoke stale tokens | Monthly | UI → Service Tokens → Revoke unused |

---

# 5. Component Deployment Specs

## 5.1 Docker Compose Stack

```yaml
# docker-compose.auth.yml
version: '3.8'

networks:
  auth-net:
    driver: bridge

volumes:
  postgres_data:
  redis_data:
  traefik_certs:

services:
  # ===================
  # INGRESS
  # ===================
  traefik:
    image: traefik:v3.0
    container_name: traefik
    restart: unless-stopped
    command:
      - "--api.dashboard=true"
      - "--providers.docker=true"
      - "--providers.docker.exposedbydefault=false"
      - "--entrypoints.web.address=:80"
      - "--entrypoints.websecure.address=:443"
      - "--entrypoints.web.http.redirections.entrypoint.to=websecure"
      - "--certificatesresolvers.letsencrypt.acme.email=${ACME_EMAIL}"
      - "--certificatesresolvers.letsencrypt.acme.storage=/certs/acme.json"
      - "--certificatesresolvers.letsencrypt.acme.httpchallenge.entrypoint=web"
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - traefik_certs:/certs
    networks:
      - auth-net
    labels:
      # Dashboard (protected by Cloudflare Zero Trust)
      - "traefik.enable=true"
      - "traefik.http.routers.dashboard.rule=Host(`traefik.bumerange.com`)"
      - "traefik.http.routers.dashboard.service=api@internal"
      - "traefik.http.routers.dashboard.entrypoints=websecure"
      - "traefik.http.routers.dashboard.tls.certresolver=letsencrypt"

  # ===================
  # DATA LAYER
  # ===================
  postgres:
    image: postgres:16-alpine
    container_name: postgres
    restart: unless-stopped
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
      POSTGRES_MULTIPLE_DATABASES: kratos,hydra,infisical
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./init-db.sh:/docker-entrypoint-initdb.d/init-db.sh
    networks:
      - auth-net
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 10s
      timeout: 5s
      retries: 5

  redis:
    image: redis:7-alpine
    container_name: redis
    restart: unless-stopped
    command: redis-server --requirepass ${REDIS_PASSWORD}
    volumes:
      - redis_data:/data
    networks:
      - auth-net
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5

  # ===================
  # IDENTITY LAYER
  # ===================
  kratos:
    image: oryd/kratos:v1.1.0
    container_name: kratos
    restart: unless-stopped
    command: serve -c /etc/config/kratos.yml --watch-courier
    volumes:
      - ./config/kratos:/etc/config
    environment:
      DSN: postgres://kratos:${POSTGRES_KRATOS_PASSWORD}@postgres:5432/kratos?sslmode=disable
      SECRETS_COOKIE: ${KRATOS_SECRETS_COOKIE}
      SECRETS_CIPHER: ${KRATOS_SECRETS_CIPHER}
      COURIER_SMTP_CONNECTION_URI: smtp://${SMTP_USERNAME}:${SMTP_PASSWORD}@${SMTP_HOST}:587
    depends_on:
      postgres:
        condition: service_healthy
    networks:
      - auth-net
    labels:
      - "traefik.enable=true"
      # Public API (login, register, etc.)
      - "traefik.http.routers.kratos-public.rule=Host(`auth.bumerange.com`) && PathPrefix(`/self-service`, `/sessions`, `/.well-known`)"
      - "traefik.http.routers.kratos-public.entrypoints=websecure"
      - "traefik.http.routers.kratos-public.tls.certresolver=letsencrypt"
      - "traefik.http.routers.kratos-public.service=kratos-public"
      - "traefik.http.services.kratos-public.loadbalancer.server.port=4433"
      # Admin API (internal only)
      - "traefik.http.routers.kratos-admin.rule=Host(`auth.bumerange.com`) && PathPrefix(`/admin`)"
      - "traefik.http.routers.kratos-admin.middlewares=internal-only@docker"
      - "traefik.http.services.kratos-admin.loadbalancer.server.port=4434"

  hydra:
    image: oryd/hydra:v2.2.0
    container_name: hydra
    restart: unless-stopped
    command: serve all -c /etc/config/hydra.yml
    volumes:
      - ./config/hydra:/etc/config
    environment:
      DSN: postgres://hydra:${POSTGRES_HYDRA_PASSWORD}@postgres:5432/hydra?sslmode=disable
      SECRETS_SYSTEM: ${HYDRA_SECRETS_SYSTEM}
      SECRETS_COOKIE: ${HYDRA_SECRETS_COOKIE}
      OIDC_SUBJECT_IDENTIFIERS_PAIRWISE_SALT: ${HYDRA_OIDC_SALT}
    depends_on:
      postgres:
        condition: service_healthy
    networks:
      - auth-net
    labels:
      - "traefik.enable=true"
      # Public OIDC endpoints
      - "traefik.http.routers.hydra-public.rule=Host(`auth.bumerange.com`) && PathPrefix(`/oauth2`, `/.well-known/openid-configuration`, `/.well-known/jwks.json`)"
      - "traefik.http.routers.hydra-public.entrypoints=websecure"
      - "traefik.http.routers.hydra-public.tls.certresolver=letsencrypt"
      - "traefik.http.routers.hydra-public.service=hydra-public"
      - "traefik.http.services.hydra-public.loadbalancer.server.port=4444"
      # Admin API (internal only)
      - "traefik.http.services.hydra-admin.loadbalancer.server.port=4445"

  oathkeeper:
    image: oryd/oathkeeper:v0.40
    container_name: oathkeeper
    restart: unless-stopped
    command: serve -c /etc/config/oathkeeper.yml
    volumes:
      - ./config/oathkeeper:/etc/config
    depends_on:
      - kratos
      - hydra
    networks:
      - auth-net
    labels:
      - "traefik.enable=true"
      # Decision endpoint (ForwardAuth)
      - "traefik.http.services.oathkeeper.loadbalancer.server.port=4456"

  # ===================
  # AUTH UI
  # ===================
  kratos-ui:
    image: oryd/kratos-selfservice-ui-node:v1.1.0
    container_name: kratos-ui
    restart: unless-stopped
    environment:
      KRATOS_PUBLIC_URL: http://kratos:4433
      KRATOS_BROWSER_URL: https://auth.bumerange.com
      HYDRA_ADMIN_URL: http://hydra:4445
      COOKIE_SECRET: ${KRATOS_UI_COOKIE_SECRET}
      CSRF_COOKIE_SECRET: ${KRATOS_UI_CSRF_SECRET}
    networks:
      - auth-net
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.kratos-ui.rule=Host(`auth.bumerange.com`) && PathPrefix(`/login`, `/registration`, `/recovery`, `/verification`, `/settings`, `/consent`, `/logout`, `/error`)"
      - "traefik.http.routers.kratos-ui.entrypoints=websecure"
      - "traefik.http.routers.kratos-ui.tls.certresolver=letsencrypt"
      - "traefik.http.services.kratos-ui.loadbalancer.server.port=3000"

  # ===================
  # SECRET MANAGEMENT
  # ===================
  infisical:
    image: infisical/infisical:latest
    container_name: infisical
    restart: unless-stopped
    environment:
      ENCRYPTION_KEY: ${INFISICAL_ENCRYPTION_KEY}
      AUTH_SECRET: ${INFISICAL_AUTH_SECRET}
      POSTGRES_HOST: postgres
      POSTGRES_PORT: 5432
      POSTGRES_DB: infisical
      POSTGRES_USER: infisical
      POSTGRES_PASSWORD: ${POSTGRES_INFISICAL_PASSWORD}
      SITE_URL: https://secrets.bumerange.com
    depends_on:
      postgres:
        condition: service_healthy
    networks:
      - auth-net
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.infisical.rule=Host(`secrets.bumerange.com`)"
      - "traefik.http.routers.infisical.entrypoints=websecure"
      - "traefik.http.routers.infisical.tls.certresolver=letsencrypt"
      - "traefik.http.routers.infisical.middlewares=cf-access@docker"
      - "traefik.http.services.infisical.loadbalancer.server.port=8080"

  # ===================
  # MIDDLEWARES
  # ===================
  # Defined via Traefik dynamic config file
```

## 5.2 Kratos Configuration

```yaml
# config/kratos/kratos.yml
version: v1.1.0

dsn: postgres://kratos:PLACEHOLDER@postgres:5432/kratos?sslmode=disable

serve:
  public:
    base_url: https://auth.bumerange.com/
    cors:
      enabled: true
      allowed_origins:
        - https://bumerange.com
        - https://rumnia.com
        - https://coyotee-renato.cloud
      allowed_methods:
        - GET
        - POST
        - PUT
        - DELETE
        - OPTIONS
      allowed_headers:
        - Authorization
        - Content-Type
        - X-Requested-With
      allow_credentials: true
  admin:
    base_url: http://kratos:4434/

selfservice:
  default_browser_return_url: https://bumerange.com/
  
  flows:
    login:
      lifespan: 1h
      ui_url: https://auth.bumerange.com/login
      after:
        default_browser_return_url: https://bumerange.com/dashboard
        
    registration:
      lifespan: 1h
      ui_url: https://auth.bumerange.com/registration
      after:
        default_browser_return_url: https://bumerange.com/welcome
        
    verification:
      enabled: true
      lifespan: 24h
      ui_url: https://auth.bumerange.com/verification
      
    recovery:
      enabled: true
      lifespan: 1h
      ui_url: https://auth.bumerange.com/recovery
      
    settings:
      ui_url: https://auth.bumerange.com/settings
      required_aal: highest_available  # Require MFA if enrolled
      
    logout:
      after:
        default_browser_return_url: https://bumerange.com/

  methods:
    password:
      enabled: true
      config:
        haveibeenpwned_enabled: true
        min_password_length: 10
        identifier_similarity_check_enabled: true
        
    totp:
      enabled: true
      config:
        issuer: Bumerange
        
    lookup_secret:
      enabled: true  # Backup codes
      
    link:
      enabled: true  # Magic links
      
    oidc:
      enabled: true
      config:
        providers:
          - id: google
            provider: google
            client_id: PLACEHOLDER
            client_secret: PLACEHOLDER
            mapper_url: file:///etc/config/oidc-mappers/google.jsonnet
            scope:
              - email
              - profile

session:
  lifespan: 24h
  cookie:
    name: bumerange_session
    domain: .bumerange.com  # For subdomains only; cross-domain uses OIDC
    path: /
    same_site: Lax
    secure: true
    http_only: true
    persistent: true

courier:
  smtp:
    from_name: Bumerange Auth
    from_address: noreply@bumerange.com

identity:
  default_schema_id: user
  schemas:
    - id: user
      url: file:///etc/config/schemas/user.schema.json
```

## 5.3 Hydra Configuration

```yaml
# config/hydra/hydra.yml
version: v2.2.0

dsn: postgres://hydra:PLACEHOLDER@postgres:5432/hydra?sslmode=disable

serve:
  public:
    cors:
      enabled: true
      allowed_origins:
        - https://bumerange.com
        - https://rumnia.com
        - https://coyotee-renato.cloud
      allowed_methods:
        - GET
        - POST
        - PUT
        - DELETE
        - OPTIONS
      allow_credentials: true
      
urls:
  self:
    issuer: https://auth.bumerange.com
    public: https://auth.bumerange.com
  login: https://auth.bumerange.com/login
  consent: https://auth.bumerange.com/consent
  logout: https://auth.bumerange.com/logout

oauth2:
  expose_internal_errors: false
  
  # Token lifetimes
  access_token_lifespan: 15m
  refresh_token_lifespan: 720h  # 30 days
  id_token_lifespan: 1h
  auth_code_lifespan: 10m
  
  # Session settings
  session:
    cookie:
      same_site_mode: Lax
      
  # PKCE required for public clients
  pkce:
    enforced: true
    enforced_for_public_clients: true

strategies:
  access_token: jwt
  
ttl:
  login_consent_request: 1h
  access_token: 15m
  refresh_token: 720h
  id_token: 1h
  auth_code: 10m
```

## 5.4 Oathkeeper Configuration

```yaml
# config/oathkeeper/oathkeeper.yml
version: v0.40.0

serve:
  proxy:
    port: 4455
  api:
    port: 4456

access_rules:
  repositories:
    - file:///etc/config/rules.yml

authenticators:
  noop:
    enabled: true
    
  cookie_session:
    enabled: true
    config:
      check_session_url: http://kratos:4433/sessions/whoami
      preserve_path: true
      preserve_query: true
      extra_from: "@this"
      subject_from: "identity.id"
      
  bearer_token:
    enabled: true
    config:
      check_session_url: http://hydra:4445/admin/oauth2/introspect
      
  oauth2_introspection:
    enabled: true
    config:
      introspection_url: http://hydra:4445/admin/oauth2/introspect
      
authorizers:
  allow:
    enabled: true
    
  deny:
    enabled: true
    
mutators:
  noop:
    enabled: true
    
  header:
    enabled: true
    config:
      headers:
        X-User-ID: "{{ print .Subject }}"
        X-User-Email: "{{ print .Extra.identity.traits.email }}"
        X-User-Verified: "{{ print .Extra.identity.verifiable_addresses.0.verified }}"
        
  id_token:
    enabled: true
    config:
      issuer_url: https://auth.bumerange.com
      jwks_url: file:///etc/config/jwks.json
      
errors:
  handlers:
    redirect:
      enabled: true
      config:
        to: https://auth.bumerange.com/login?return_to={{ .URL | urlescape }}
        
    json:
      enabled: true
      config:
        verbose: false
```

---

# 6. Cross-Domain SSO Implementation

## 6.1 Why OIDC is Required

```
Problem: 3 different TLDs cannot share cookies
  ├── bumerange.com    ← Cookie domain: .bumerange.com
  ├── rumnia.com       ← Cookie domain: .rumnia.com
  └── coyotee-renato.cloud ← Cookie domain: .coyotee-renato.cloud

Solution: OIDC Authorization Code Flow with PKCE
  └── All apps redirect to auth.bumerange.com (Hydra)
  └── Hydra maintains single session via cookie
  └── Each app gets tokens, creates local session
```

## 6.2 OIDC Client Registration

```bash
# Register bumerange.com as OIDC client
hydra create oauth2-client \
  --endpoint http://hydra:4445 \
  --name "Bumerange App" \
  --grant-type authorization_code,refresh_token \
  --response-type code \
  --scope openid,email,profile,offline_access \
  --redirect-uri https://bumerange.com/auth/callback \
  --post-logout-redirect-uri https://bumerange.com/ \
  --token-endpoint-auth-method client_secret_post \
  --audience https://api.bumerange.com

# Output: client_id and client_secret → Store in Infisical
```

Repeat for `rumnia.com` and `coyotee-renato.cloud`.

## 6.3 SSO Flow Diagram

```mermaid
sequenceDiagram
    autonumber
    participant U as User
    participant B as bumerange.com
    participant H as auth.bumerange.com<br/>(Hydra)
    participant K as Kratos
    participant R as rumnia.com

    Note over U,R: User logs into Bumerange
    
    U->>B: Visit bumerange.com/dashboard
    B->>B: No local session
    B-->>U: "302 → auth.bumerange.com/oauth2/auth?<br/>client_id=bumerange&redirect_uri=.../callback&<br/>response_type=code&scope=openid+email&<br/>code_challenge=...&code_challenge_method=S256"
    
    U->>H: "GET /oauth2/auth"
    H->>H: No Hydra session cookie
    H-->>U: "302 → /login?login_challenge=xyz"

    U->>K: Login form
    U->>K: POST credentials
    K->>K: "Validate, create Kratos session"
    K-->>H: Login accepted for challenge xyz

    H->>H: "Consent (skip if trusted first-party)"
    H->>H: Issue authorization code
    H-->>U: "302 → bumerange.com/callback?code=abc<br/>Set-Cookie: hydra_session (domain: .bumerange.com)"

    U->>B: "GET /callback?code=abc"
    B->>H: "POST /oauth2/token (exchange code)"
    H-->>B: "{access_token, id_token, refresh_token}"
    B->>B: "Validate id_token, extract user"
    B->>B: Create local session
    B-->>U: "Set-Cookie: bumerange_session<br/>302 → /dashboard"

    Note over U,R: Later: User visits Rumnia

    U->>R: Visit rumnia.com/app
    R->>R: No local session
    R-->>U: "302 → auth.bumerange.com/oauth2/auth?<br/>client_id=rumnia&..."

    U->>H: "GET /oauth2/auth"
    H->>H: hydra_session cookie exists!
    H->>H: Session still valid
    H->>H: "Skip login, issue code immediately"
    H-->>U: "302 → rumnia.com/callback?code=def"
    
    U->>R: "GET /callback?code=def"
    R->>H: Exchange code for tokens
    R->>R: Create local session
    R-->>U: "Logged in (NO re-login!)"
```

## 6.4 Cross-Domain Logout

```mermaid
sequenceDiagram
    autonumber
    participant U as User
    participant B as bumerange.com
    participant H as auth.bumerange.com
    participant R as rumnia.com

    U->>B: Click Logout
    B->>B: Clear bumerange_session cookie
    B-->>U: "302 → auth.bumerange.com/oauth2/sessions/logout?<br/>id_token_hint=...&post_logout_redirect_uri=bumerange.com"

    U->>H: "GET /oauth2/sessions/logout"
    H->>H: Revoke Hydra session
    H->>H: "Revoke Kratos session (via hook)"
    H-->>U: "Clear hydra_session cookie<br/>302 → bumerange.com"

    Note over U,R: User visits Rumnia

    U->>R: Visit rumnia.com/app
    R->>R: "rumnia_session exists (stale!)"
    R->>H: "Validate access_token (or refresh)"
    H-->>R: "401 Unauthorized (session revoked)"
    R->>R: Clear local session
    R-->>U: "302 → Login required"
```

## 6.5 Client-Side Implementation (example: Next.js)

```typescript
// lib/auth.ts (for each app)
import { NextAuthOptions } from 'next-auth';
import OryProvider from 'next-auth/providers/ory';

export const authOptions: NextAuthOptions = {
  providers: [
    {
      id: 'bumerange-sso',
      name: 'Bumerange SSO',
      type: 'oauth',
      wellKnown: 'https://auth.bumerange.com/.well-known/openid-configuration',
      clientId: process.env.OIDC_CLIENT_ID,
      clientSecret: process.env.OIDC_CLIENT_SECRET,
      authorization: { params: { scope: 'openid email profile' } },
      idToken: true,
      checks: ['pkce', 'state'],
      profile(profile) {
        return {
          id: profile.sub,
          email: profile.email,
          name: profile.name,
        };
      },
    },
  ],
  callbacks: {
    async jwt({ token, account }) {
      if (account) {
        token.accessToken = account.access_token;
        token.refreshToken = account.refresh_token;
        token.expiresAt = account.expires_at;
      }
      return token;
    },
  },
};
```

---

# 7. OWASP Security Controls

> Reference: [SSO-CrossCuttingOWASP](./SSO-CrossCuttingOWASP)

## 7.1 OWASP API Top 10 Coverage

| [OWASP](#glossary) ID | Risk | This Architecture Mitigates Via |
|----------|------|--------------------------------|
| **API1** | Broken Object Level Authorization ([BOLA](#glossary)) | [Oathkeeper](#glossary) enforces auth on every route; apps must still do object-level checks |
| **API2** | Broken Authentication | [Kratos](#glossary) handles auth securely; rate limiting at [Cloudflare](#glossary) + [Traefik](#glossary) |
| **API3** | Broken Object Property Level Authorization | Apps responsible; headers provide user context |
| **API4** | Unrestricted Resource Consumption | [Cloudflare](#glossary) [WAF](#glossary) + rate limits; per-user limits at app layer |
| **API5** | Broken Function Level Authorization | [Oathkeeper](#glossary) rules define route access; apps enforce role checks |

## 7.2 Authentication Controls (per OWASP Cheat Sheets)

| Control | Implementation | Reference |
|---------|---------------|-----------|
| **Secure session IDs** | [Kratos](#glossary) generates cryptographically secure session tokens | Session Management CS |
| **Cookie attributes** | `Secure`, `HttpOnly`, `[SameSite](#glossary)=Lax` enforced in [Kratos](#glossary) config | Session Management CS |
| **Session timeout** | 24h lifespan, require re-auth for sensitive actions | Session Management CS |
| **Rate limiting on auth endpoints** | [Cloudflare](#glossary): 10 req/min per IP on `/login`, `/register` | Auth CS |
| **No credentials in URL** | [OIDC](#glossary) uses POST for token exchange; auth codes are one-time use | [OAuth2](#glossary) CS |
| **[PKCE](#glossary) required** | [Hydra](#glossary) enforces [PKCE](#glossary) for all clients | [OAuth2](#glossary) CS |

## 7.3 OAuth2/OIDC Security (per OWASP OAuth2 Cheat Sheet)

| Risk | Mitigation |
|------|-----------|
| **Open Redirects** | Strict redirect URI allowlist in [Hydra](#glossary) client registration |
| **[Authorization Code](#glossary) Interception** | [PKCE](#glossary) enforced for all clients (public + confidential) |
| **Token Leakage** | Short-lived access tokens (15 min); refresh tokens rotated on use |
| **State Parameter Bypass** | [Hydra](#glossary) validates `state` parameter; clients use `state` + `nonce` |
| **Insufficient Scope Validation** | Each app requests minimal scopes; [APIs](#glossary) validate scopes in headers |

## 7.4 Session Management Security

```yaml
# Kratos session hardening
session:
  cookie:
    same_site: Lax           # CSRF protection
    secure: true             # HTTPS only
    http_only: true          # No JS access
    persistent: true         # Survives browser restart (user preference)
  lifespan: 24h              # Max session duration
  
# Additional headers via Traefik
headers:
  Cache-Control: "no-store"  # Don't cache auth responses
  X-Content-Type-Options: "nosniff"
  X-Frame-Options: "DENY"
  Referrer-Policy: "strict-origin-when-cross-origin"
```

## 7.5 Required Abuse-Case Tests

| Test | What to Verify | Pass Criteria |
|------|---------------|---------------|
| **BOLA/IDOR** | Change resource IDs in requests | 403/404, not another user's data |
| **Session Fixation** | Use session ID before login, check after | New session ID issued post-login |
| **Open Redirect** | Modify `redirect_uri` in OAuth flow | 400 Bad Request |
| **Token Replay** | Reuse authorization code | Second attempt fails |
| **Brute Force** | 20+ failed logins in 1 minute | Rate limited (429) |
| **Logout Invalidation** | Use access token after logout | 401 Unauthorized |

## 7.6 OWASP Threat Mitigation Map

```mermaid
flowchart TB
    subgraph Threats ["OWASP API Top 10 Threats"]
        T1["API1: BOLA"]
        T2["API2: Broken Auth"]
        T3["API3: Excessive Data"]
        T4["API4: Resource Abuse"]
        T5["API5: Broken Function Auth"]
        T6["API7: Server Side Request Forgery"]
        T7["API8: Security Misconfiguration"]
        T8["API9: Improper Assets Management"]
    end

    subgraph Controls ["Implemented Controls"]
        C1["Oathkeeper<br/>Auth Enforcement"]
        C2["Kratos<br/>Secure Sessions"]
        C3["Rate Limiting<br/>CF + Traefik"]
        C4["PKCE<br/>Enforcement"]
        C5["Security Headers<br/>Traefik"]
        C6["Infisical<br/>Secret Management"]
        C7["Zero Trust<br/>Admin Access"]
    end

    T1 --> C1
    T2 --> C2
    T2 --> C4
    T4 --> C3
    T5 --> C1
    T6 --> C7
    T7 --> C5
    T7 --> C6
    T8 --> C7

    style T1 fill:#ffcdd2
    style T2 fill:#ffcdd2
    style T4 fill:#ffcdd2
    style C1 fill:#c8e6c9
    style C2 fill:#c8e6c9
    style C3 fill:#c8e6c9
```

## 7.7 Session Security Flow

```mermaid
sequenceDiagram
    autonumber
    participant User
    participant App as Application
    participant OK as Oathkeeper
    participant KR as Kratos
    participant RD as Redis

    Note over User,RD: Initial Login

    User->>App: Access Protected Resource
    App->>OK: Forward Request
    OK->>OK: No Session Cookie
    OK-->>App: 401 Unauthorized
    App-->>User: Redirect to Login

    User->>KR: POST /login (credentials)
    KR->>KR: Validate Password<br/>Check HaveIBeenPwned
    KR->>KR: Generate Session Token<br/>(Cryptographically Secure)
    KR->>RD: Cache Session (24h TTL)
    KR-->>User: Set-Cookie (Secure, HttpOnly, SameSite=Lax)

    Note over User,RD: Subsequent Requests

    User->>App: Request with Cookie
    App->>OK: Forward Request + Cookie
    OK->>RD: Validate Session
    RD-->>OK: Session Valid + User Data
    OK->>OK: Inject X-User-ID<br/>X-User-Email Headers
    OK-->>App: 200 OK + Headers
    App->>App: Use User Context
    App-->>User: Protected Resource

    Note over User,RD: Session Expiry

    User->>App: Request (25h later)
    App->>OK: Forward Request
    OK->>RD: Validate Session
    RD-->>OK: Session Expired
    OK-->>App: 401 Unauthorized
    App-->>User: Redirect to Login
```

## 7.8 MFA Enforcement

```yaml
# Step-up auth for sensitive operations
selfservice:
  flows:
    settings:
      required_aal: highest_available  # Require MFA if enrolled
      
# Oathkeeper rule for admin routes requiring MFA
- id: admin-requires-mfa
  match:
    url: "https://coyotee-renato.cloud/admin/**"
  authenticators:
    - handler: cookie_session
      config:
        check_session_url: http://kratos:4433/sessions/whoami
        only:
          - aal2  # AAL2 = MFA completed
  errors:
    - handler: redirect
      config:
        to: https://auth.bumerange.com/settings?return_to={{ .URL | urlescape }}
```

---

# 8. Cloudflare Integration

## 8.1 Cloudflare Configuration

### DNS Records

```
Zone: bumerange.com
  auth      A       <VPS_IP>    Proxied ✅
  secrets   A       <VPS_IP>    Proxied ✅
  traefik   A       <VPS_IP>    Proxied ✅
  api       A       <VPS_IP>    Proxied ✅

Zone: rumnia.com
  @         A       <APP_HOST>  Proxied ✅

Zone: coyotee-renato.cloud
  @         A       <APP_HOST>  Proxied ✅
```

### WAF Rules (Firewall Rules)

```javascript
// Block excessive auth attempts
(http.request.uri.path contains "/login" or 
 http.request.uri.path contains "/registration" or
 http.request.uri.path contains "/recovery")
and cf.threat_score > 10
→ Block

// Rate limit auth endpoints
(http.request.uri.path contains "/login" or 
 http.request.uri.path contains "/registration")
→ Rate Limit: 10 requests per minute per IP
```

### Zero Trust Access Policy

```yaml
# Infisical Admin Access
Application: secrets.bumerange.com
Policies:
  - Name: "Team Only"
    Decision: Allow
    Include:
      - Emails ending in: @bumerange.com
    Require:
      - Authentication method: One-time PIN (email)
      
# Traefik Dashboard
Application: traefik.bumerange.com
Policies:
  - Name: "Admin Only"
    Decision: Allow
    Include:
      - Email: admin@bumerange.com
    Require:
      - Authentication method: One-time PIN
```

## 8.2 Cloudflare Workers Integration

For apps running on Cloudflare Workers that need to validate auth:

```javascript
// worker.js - Validate OIDC token
export default {
  async fetch(request, env) {
    const authHeader = request.headers.get('Authorization');
    
    if (!authHeader || !authHeader.startsWith('Bearer ')) {
      // Redirect to OIDC login
      const loginUrl = new URL('https://auth.bumerange.com/oauth2/auth');
      loginUrl.searchParams.set('client_id', env.OIDC_CLIENT_ID);
      loginUrl.searchParams.set('redirect_uri', `${new URL(request.url).origin}/auth/callback`);
      loginUrl.searchParams.set('response_type', 'code');
      loginUrl.searchParams.set('scope', 'openid email profile');
      return Response.redirect(loginUrl.toString(), 302);
    }
    
    // Validate token via introspection
    const token = authHeader.split(' ')[1];
    const introspection = await fetch('https://auth.bumerange.com/oauth2/introspect', {
      method: 'POST',
      headers: { 'Content-Type': 'application/x-www-form-urlencoded' },
      body: `token=${token}&client_id=${env.OIDC_CLIENT_ID}&client_secret=${env.OIDC_CLIENT_SECRET}`,
    });
    
    const result = await introspection.json();
    if (!result.active) {
      return new Response('Unauthorized', { status: 401 });
    }
    
    // Proceed with request, user info in result.sub, result.email, etc.
    return handleAuthenticatedRequest(request, result);
  }
};
```

---

# 9. Implementation Phases

## 9.1 Phase Overview

```mermaid
gantt
    title Implementation Timeline
    dateFormat  YYYY-MM-DD
    section Phase 1
    VPS Setup + Docker           :p1a, 2026-01-06, 2d
    Infisical Deploy             :p1b, after p1a, 1d
    Initial Secrets              :p1c, after p1b, 1d
    section Phase 2
    Auth Stack Deploy            :p2a, after p1c, 2d
    Kratos + Hydra Config        :p2b, after p2a, 2d
    Auth UI + Testing            :p2c, after p2b, 2d
    section Phase 3
    First App Integration        :p3a, after p2c, 3d
    OIDC Client Setup            :p3b, after p3a, 1d
    SSO Testing                  :p3c, after p3b, 2d
    section Phase 4
    Remaining Apps               :p4a, after p3c, 5d
    Cross-domain Logout          :p4b, after p4a, 2d
    section Phase 5
    Security Hardening           :p5a, after p4b, 3d
    Documentation + Runbooks     :p5b, after p5a, 2d
```

## 9.2 Deployment Sequence Diagram

```mermaid
sequenceDiagram
    autonumber
    participant Ops as Operator
    participant VPS as VPS Server
    participant IF as Infisical
    participant PG as Postgres
    participant KR as Kratos
    participant HY as Hydra
    participant TR as Traefik
    participant CF as Cloudflare

    Note over Ops,CF: Phase 1: Infrastructure

    Ops->>VPS: Install Docker + Docker Compose
    Ops->>VPS: Configure Firewall
    Ops->>VPS: Create Project Structure
    Ops->>IF: Deploy Infisical Container
    IF->>PG: Initialize Infisical DB
    Ops->>CF: Configure DNS Records
    Ops->>CF: Configure Zero Trust for Infisical

    Note over Ops,CF: Phase 2: Auth Stack

    Ops->>IF: Add All Secrets via UI
    Ops->>KR: Deploy Kratos Container
    KR->>PG: Run Kratos Migrations
    Ops->>HY: Deploy Hydra Container
    HY->>PG: Run Hydra Migrations
    Ops->>TR: Deploy Traefik Container
    TR->>TR: Request Let's Encrypt Certs
    Ops->>VPS: Test All Endpoints

    Note over Ops,CF: Phase 3-4: Application Integration

    Ops->>HY: Register OIDC Clients
    HY-->>Ops: Client Credentials
    Ops->>IF: Store Client Secrets
    Ops->>VPS: Deploy Applications
    Ops->>Ops: Test SSO Flows

    Note over Ops,CF: Phase 5: Hardening

    Ops->>CF: Configure WAF Rules
    Ops->>CF: Configure Rate Limits
    Ops->>VPS: Enable Monitoring
    Ops->>VPS: Configure Backups
```

## 9.3 Phase Dependencies and Critical Path

```mermaid
flowchart LR
    subgraph Phase1 ["Phase 1: Infrastructure"]
        P1A["VPS Setup<br/>(Day 1-2)"]
        P1B["Infisical Deploy<br/>(Day 3)"]
        P1C["Initial Secrets<br/>(Day 4)"]
    end

    subgraph Phase2 ["Phase 2: Auth Stack"]
        P2A["Deploy Services<br/>(Day 5-6)"]
        P2B["Configure Stack<br/>(Day 7-8)"]
        P2C["Testing<br/>(Day 9-10)"]
    end

    subgraph Phase3 ["Phase 3: First App"]
        P3A["OIDC Setup<br/>(Day 11-13)"]
        P3B["Code Integration<br/>(Day 14-16)"]
    end

    subgraph Phase4 ["Phase 4: Remaining Apps"]
        P4A["App 2-3 Integration<br/>(Day 17-23)"]
    end

    subgraph Phase5 ["Phase 5: Production Ready"]
        P5A["Security Hardening<br/>(Day 24-26)"]
        P5B["Documentation<br/>(Day 27-28)"]
    end

    P1A --> P1B
    P1B --> P1C
    P1C --> P2A
    P2A --> P2B
    P2B --> P2C
    P2C --> P3A
    P3A --> P3B
    P3B --> P4A
    P4A --> P5A
    P5A --> P5B

    style P1A fill:#e3f2fd
    style P1B fill:#e3f2fd
    style P1C fill:#e3f2fd,stroke:#2196f3,stroke-width:3px
    style P2C fill:#fff9c4,stroke:#fbc02d,stroke-width:3px
    style P3B fill:#f3e5f5,stroke:#9c27b0,stroke-width:3px
    style P5B fill:#c8e6c9,stroke:#4caf50,stroke-width:3px
```

## 9.4 Resource Provisioning Timeline

```mermaid
gantt
    title Resource Allocation Over Implementation
    dateFormat  YYYY-MM-DD
    section VPS Resources
    Infisical Only (2GB)           :done, r1, 2026-01-06, 4d
    + Auth Stack (5GB)             :done, r2, 2026-01-10, 6d
    + First App (8GB)              :active, r3, 2026-01-16, 7d
    Full Stack (14GB Used)         :r4, 2026-01-23, 6d
    section Cloudflare Config
    DNS Setup                      :done, c1, 2026-01-06, 1d
    Zero Trust (Infisical)         :done, c2, 2026-01-07, 1d
    WAF Rules                      :c3, 2026-01-24, 2d
    Rate Limiting                  :c4, 2026-01-25, 1d
    section Testing
    Infisical Access Test          :done, t1, 2026-01-09, 1d
    Auth Flow Test                 :t2, 2026-01-15, 1d
    SSO Test (1 app)               :t3, 2026-01-20, 1d
    Cross-Domain SSO Test          :t4, 2026-01-26, 2d
    OWASP Security Tests           :t5, 2026-01-27, 2d
```

## 9.5 Phase 1: Infrastructure (Days 1-4)

### Day 1-2: VPS Setup

```bash
# On Hostinger VPS
# 1. Update system
sudo apt update && sudo apt upgrade -y

# 2. Install Docker
curl -fsSL https://get.docker.com | sh
sudo usermod -aG docker $USER

# 3. Install Docker Compose
sudo curl -L "https://github.com/docker/compose/releases/latest/download/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose

# 4. Install Infisical CLI
curl -1sLf 'https://dl.cloudsmith.io/public/infisical/infisical-cli/setup.deb.sh' | sudo -E bash
sudo apt install infisical

# 5. Configure firewall
sudo ufw allow 22/tcp
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw enable

# 6. Create project structure
mkdir -p ~/auth-stack/{config/{kratos,hydra,oathkeeper,traefik},data}
cd ~/auth-stack
```

### Day 3: Infisical Deploy

```bash
# Generate secrets for Infisical itself (bootstrap)
INFISICAL_ENCRYPTION_KEY=$(openssl rand -hex 32)
INFISICAL_AUTH_SECRET=$(openssl rand -hex 32)
POSTGRES_ROOT_PASSWORD=$(openssl rand -base64 24)

# Create initial .env (will migrate to Infisical after setup)
cat > .env <<EOF
INFISICAL_ENCRYPTION_KEY=$INFISICAL_ENCRYPTION_KEY
INFISICAL_AUTH_SECRET=$INFISICAL_AUTH_SECRET
POSTGRES_PASSWORD=$POSTGRES_ROOT_PASSWORD
EOF

# Deploy Infisical + Postgres
docker-compose -f docker-compose.infisical.yml up -d

# Configure DNS: secrets.bumerange.com → VPS IP
# Configure Cloudflare Zero Trust for secrets.bumerange.com
```

### Day 4: Migrate Secrets to Infisical

```bash
# Access Infisical UI at https://secrets.bumerange.com
# 1. Create account (first user = admin)
# 2. Create project: auth-infrastructure
# 3. Add all secrets via UI or CLI:

infisical secrets set POSTGRES_KRATOS_PASSWORD=$(openssl rand -base64 24) --env=production
infisical secrets set POSTGRES_HYDRA_PASSWORD=$(openssl rand -base64 24) --env=production
infisical secrets set KRATOS_SECRETS_COOKIE=$(openssl rand -hex 32) --env=production
infisical secrets set KRATOS_SECRETS_CIPHER=$(openssl rand -hex 32) --env=production
infisical secrets set HYDRA_SECRETS_SYSTEM=$(openssl rand -hex 32) --env=production
infisical secrets set HYDRA_SECRETS_COOKIE=$(openssl rand -hex 32) --env=production
infisical secrets set HYDRA_OIDC_SALT=$(openssl rand -hex 32) --env=production
# ... etc
```

**Checkpoint:** Infisical running, accessible only via Zero Trust, secrets stored.

## 9.3 Phase 2: Auth Stack (Days 5-10)

### Day 5-6: Deploy Kratos + Hydra

```bash
cd ~/auth-stack

# Create config files (copy from Section 5)
vim config/kratos/kratos.yml
vim config/hydra/hydra.yml
vim config/oathkeeper/oathkeeper.yml

# Run migrations
infisical run --env=production -- docker-compose run --rm kratos migrate sql -e --yes
infisical run --env=production -- docker-compose run --rm hydra migrate sql -e --yes

# Start stack
infisical run --env=production -- docker-compose -f docker-compose.auth.yml up -d
```

### Day 7-8: Auth UI + Configuration

```bash
# Verify endpoints
curl https://auth.bumerange.com/.well-known/openid-configuration
curl https://auth.bumerange.com/health/alive

# Test registration flow
# 1. Visit https://auth.bumerange.com/registration
# 2. Create test account
# 3. Check email verification works

# Configure social login (optional)
# Add Google OAuth credentials to Infisical
# Update kratos.yml with actual client_id/secret
```

### Day 9-10: Testing

```bash
# Test session management
# 1. Login at auth.bumerange.com
# 2. Verify session cookie set
# 3. Call /sessions/whoami
curl -b cookies.txt https://auth.bumerange.com/sessions/whoami

# Test MFA enrollment
# 1. Go to /settings
# 2. Enable TOTP
# 3. Verify AAL2 session
```

**Checkpoint:** Can register, login, logout, MFA works.

## 9.4 Phase 3: First App Integration (Days 11-16)

### Day 11-13: OIDC Client Setup

```bash
# Register first app as OIDC client
docker exec -it hydra hydra create oauth2-client \
  --endpoint http://localhost:4445 \
  --name "Bumerange Main App" \
  --grant-type authorization_code,refresh_token \
  --response-type code \
  --scope openid,email,profile,offline_access \
  --redirect-uri https://bumerange.com/auth/callback \
  --post-logout-redirect-uri https://bumerange.com/ \
  --token-endpoint-auth-method client_secret_post

# Store client_id and client_secret in Infisical
infisical secrets set BUMERANGE_OIDC_CLIENT_ID=<client_id> --env=production --project=bumerange-app
infisical secrets set BUMERANGE_OIDC_CLIENT_SECRET=<client_secret> --env=production --project=bumerange-app
```

### Day 14-16: App Code Integration + SSO Test

```typescript
// In bumerange.com app, add OIDC provider (example: NextAuth)
// See Section 6.5 for code

// Test SSO:
// 1. Visit bumerange.com (not logged in)
// 2. Redirected to auth.bumerange.com
// 3. Login
// 4. Redirected back to bumerange.com (logged in)
```

**Checkpoint:** First app uses centralized auth.

## 9.5 Phase 4: Remaining Apps (Days 17-23)

Repeat Phase 3 for:
- rumnia.com
- coyotee-renato.cloud

Test cross-domain SSO:
1. Login to bumerange.com
2. Visit rumnia.com → should be logged in (no re-auth)
3. Logout from rumnia.com
4. Visit bumerange.com → should be logged out

## 9.6 Phase 5: Hardening + Documentation (Days 24-28)

- Run OWASP abuse-case tests (Section 7.5)
- Configure monitoring (Section 10)
- Write runbooks
- Document recovery procedures

---

# 10. Operational Runbook

## 10.1 Monitoring Stack (Lightweight)

```yaml
# Add to docker-compose for monitoring
services:
  prometheus:
    image: prom/prometheus:latest
    volumes:
      - ./config/prometheus:/etc/prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
    networks:
      - auth-net

  grafana:
    image: grafana/grafana:latest
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=${GRAFANA_PASSWORD}
    volumes:
      - grafana_data:/var/lib/grafana
    networks:
      - auth-net
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.grafana.rule=Host(`grafana.bumerange.com`)"
      - "traefik.http.routers.grafana.middlewares=cf-access@docker"
```

## 10.2 Monitoring Architecture

```mermaid
flowchart LR
    subgraph Services ["Services (Metrics Sources)"]
        KR["Kratos<br/>:4434/metrics"]
        HY["Hydra<br/>:4445/metrics"]
        PG["Postgres<br/>Exporter"]
        RD["Redis<br/>Exporter"]
        TR["Traefik<br/>:8082/metrics"]
    end

    subgraph Monitoring ["Monitoring Stack"]
        PM["Prometheus<br/>(Scrape & Store)"]
        GR["Grafana<br/>(Visualize)"]
        Alert["Alert Manager<br/>(Optional)"]
    end

    subgraph Dashboards ["Dashboards"]
        D1["System Overview"]
        D2["Auth Performance"]
        D3["Security Events"]
        D4["Resource Usage"]
    end

    KR --> PM
    HY --> PM
    PG --> PM
    RD --> PM
    TR --> PM

    PM --> GR
    PM --> Alert

    GR --> D1
    GR --> D2
    GR --> D3
    GR --> D4

    Alert -.->|Email/Slack| Ops[Operators]

    style PM fill:#ff9800
    style GR fill:#2196f3
    style Alert fill:#f44336
```

## 10.3 Backup and Recovery Flow

```mermaid
flowchart TB
    subgraph Production ["Production VPS"]
        PG_Prod["Postgres<br/>(Auth Data)"]
        Redis_Prod["Redis<br/>(Session Cache)"]
        Configs["Config Files<br/>(Git)"]
    end

    subgraph Backup ["Backup Strategy"]
        Cron["Daily Cron Job"]
        Encrypt["Age Encryption"]
        Storage["Object Storage<br/>(B2/S3)"]
    end

    subgraph Recovery ["Recovery Targets"]
        Oracle_DR["Oracle Free Tier<br/>(Warm Standby)"]
        Local_Test["Local Testing<br/>(Docker Compose)"]
    end

    PG_Prod -->|pg_dumpall| Cron
    Cron --> Encrypt
    Encrypt --> Storage

    Storage -.->|Restore| Oracle_DR
    Storage -.->|Restore| Local_Test
    Configs -.->|Git Clone| Oracle_DR
    Configs -.->|Git Clone| Local_Test

    Redis_Prod -.->|Not Backed Up<br/>(Transient Data)| Note1[Sessions rebuild<br/>from Postgres]

    style PG_Prod fill:#4caf50
    style Storage fill:#2196f3
    style Oracle_DR fill:#ff9800
```

## 10.4 Incident Response Decision Tree

```mermaid
flowchart TD
    Alert["Alert Triggered"]

    Alert --> Check1{Service Down?}

    Check1 -->|Yes| Check2{Which Service?}
    Check1 -->|No| Check3{Performance Issue?}

    Check2 -->|Postgres| Action1["1. Check disk space<br/>2. Check connections<br/>3. Review logs<br/>4. Restart if needed"]
    Check2 -->|Kratos/Hydra| Action2["1. Check Postgres health<br/>2. Check Redis health<br/>3. Review container logs<br/>4. Restart container"]
    Check2 -->|Traefik| Action3["1. Check certificate expiry<br/>2. Check Docker socket<br/>3. Review logs<br/>4. Restart container"]

    Check3 -->|Yes| Check4{High Latency?}
    Check3 -->|No| Check5{Security Alert?}

    Check4 -->|Yes| Action4["1. Check Redis memory<br/>2. Check Postgres queries<br/>3. Review CloudFlare stats<br/>4. Scale if needed"]

    Check5 -->|Yes| Action5["1. Review access logs<br/>2. Check IP patterns<br/>3. Update WAF rules<br/>4. Block if attack"]
    Check5 -->|No| Action6["1. Review all logs<br/>2. Check metrics<br/>3. Escalate if unclear"]

    style Alert fill:#f44336,color:#fff
    style Check1 fill:#ff9800
    style Check2 fill:#ff9800
    style Action1 fill:#4caf50,color:#fff
    style Action2 fill:#4caf50,color:#fff
    style Action5 fill:#f44336,color:#fff
```

## 10.5 Key Metrics

| Metric | Source | Alert Threshold |
|--------|--------|-----------------|
| Kratos health | `/health/alive` | != 200 for 1 min |
| Hydra health | `/health/alive` | != 200 for 1 min |
| Login success rate | Kratos metrics | < 95% over 5 min |
| Session validation latency p99 | Oathkeeper | > 500ms |
| Postgres connections | pg_stat | > 80% of max |
| Redis memory | Redis INFO | > 80% of max |

## 10.3 Backup Procedures

```bash
# Daily backup script (cron job)
#!/bin/bash
DATE=$(date +%Y%m%d)
BACKUP_DIR=/root/backups

# Postgres
docker exec postgres pg_dumpall -U postgres > $BACKUP_DIR/postgres-$DATE.sql
gzip $BACKUP_DIR/postgres-$DATE.sql

# Encrypt and upload to Object Storage (optional)
age -r <your-public-key> $BACKUP_DIR/postgres-$DATE.sql.gz > $BACKUP_DIR/postgres-$DATE.sql.gz.age
# Upload to S3/B2/etc.

# Cleanup old backups (keep 7 days)
find $BACKUP_DIR -name "*.age" -mtime +7 -delete
```

## 10.4 Incident Response

### Auth Service Down

```
1. Check container status:
   docker-compose ps
   docker logs kratos --tail 100

2. Check Postgres:
   docker exec postgres pg_isready

3. Restart affected service:
   docker-compose restart kratos

4. If persistent, check resources:
   docker stats
   df -h

5. Rollback if recent deployment:
   git checkout <previous-commit>
   docker-compose up -d
```

### Session Validation Slow

```
1. Check Redis:
   docker exec redis redis-cli ping
   docker exec redis redis-cli info memory

2. Check Kratos/Oathkeeper logs for errors

3. Clear Redis cache if corrupted:
   docker exec redis redis-cli FLUSHALL

4. Scale up if resource-constrained:
   (Consider K3s migration if frequent)
```

---

# 11. Cost Analysis

## 11.1 Monthly Costs

| Item | Provider | Cost |
|------|----------|------|
| [VPS](#glossary) (4 [vCPU](#glossary), 16GB) | Hostinger | ~$15-25/mo |
| Domains (3) | [Cloudflare](#glossary) | ~$10-15/yr each |
| [Cloudflare](#glossary) (Pro optional) | [Cloudflare](#glossary) | $0 (Free tier) or $20/mo |
| Email ([SMTP](#glossary)) | Resend/Mailgun | Free tier |
| Backups (object storage) | Backblaze B2 | ~$1-2/mo |
| **Total** | | **~$20-50/mo** |

## 11.2 Cost vs Scale Projection

```mermaid
graph LR
    subgraph CurrentState ["Current: 100 Users"]
        C1["VPS: $20/mo<br/>(Hostinger)"]
        C2["Domains: $3/mo<br/>(3 domains)"]
        C3["Email: $0<br/>(Free tier)"]
        C4["Backups: $1/mo<br/>(B2)"]
        C5["CF: $0<br/>(Free tier)"]
    end

    subgraph Scale1K ["1,000 Users"]
        S1["VPS: $20/mo<br/>(Same)"]
        S2["Domains: $3/mo"]
        S3["Email: $10/mo<br/>(Paid tier)"]
        S4["Backups: $2/mo"]
        S5["CF: $0<br/>(Still free)"]
    end

    subgraph Scale10K ["10,000 Users"]
        L1["VPS: $50/mo<br/>(Upgraded)"]
        L2["Domains: $3/mo"]
        L3["Email: $20/mo"]
        L4["Backups: $5/mo"]
        L5["CF Pro: $20/mo<br/>(Advanced WAF)"]
        L6["Monitor: $10/mo<br/>(Managed)"]
    end

    CurrentState -->|"Total: $24/mo"| Scale1K
    Scale1K -->|"Total: $35/mo"| Scale10K
    Scale10K -->|"Total: $108/mo"| Future["50K+ Users:<br/>Consider K8s<br/>$300-500/mo"]

    style CurrentState fill:#c8e6c9
    style Scale1K fill:#fff9c4
    style Scale10K fill:#ffcc80
    style Future fill:#ffab91
```

## 11.3 Free Tier Limits

| Service | Free Tier | When You'd Upgrade |
|---------|-----------|-------------------|
| [Cloudflare](#glossary) | Unlimited [DNS](#glossary), 100K Workers/day | Need advanced [WAF](#glossary) rules |
| [Supabase](#glossary) | 500MB [Postgres](#glossary), 2GB bandwidth | 10K+ active users |
| [Infisical](#glossary) | Unlimited (self-hosted) | Want managed [SaaS](#glossary) |
| [Oracle](#glossary) Free Tier | 4 [OCPU](#glossary), 24GB [RAM](#glossary) ([ARM](#glossary)) | Never (keep as backup) |

---

# 12. Risk Mitigation

## 12.1 Risk Matrix

| Risk | Likelihood | Impact | Mitigation |
|------|------------|--------|-----------|
| [VPS](#glossary) goes down | Medium | High | [Oracle](#glossary) Free Tier as warm standby; regular backups |
| Secret leak | Low | Critical | [Infisical](#glossary) access via [Zero Trust](#glossary); no secrets in git |
| Credential stuffing | Medium | Medium | [Cloudflare](#glossary) rate limiting; HaveIBeenPwned integration |
| Session hijacking | Low | High | Secure cookies; short session lifespan; logout invalidates server-side |
| Key rotation needed | Certain | Medium | Document rotation procedure; schedule quarterly review |
| Scaling beyond [VPS](#glossary) | Medium | Medium | [K3s](#glossary) migration path documented; can move to managed [Kubernetes](#glossary) |

## 12.2 Disaster Recovery

| Scenario | RTO | RPO | Procedure |
|----------|-----|-----|-----------|
| [VPS](#glossary) failure | 4 hours | 24 hours | Restore from backup to [Oracle](#glossary) Free Tier |
| [Postgres](#glossary) corruption | 2 hours | 24 hours | Restore from daily backup |
| Compromised secrets | 1 hour | 0 | Rotate all secrets via [Infisical](#glossary); revoke all sessions |
| [DDoS](#glossary) attack | 0 ([Cloudflare](#glossary) absorbs) | 0 | Already mitigated by [Cloudflare](#glossary) |

---

# Appendix A: File Structure

```
~/auth-stack/
├── docker-compose.auth.yml
├── docker-compose.infisical.yml
├── docker-compose.monitoring.yml
├── .env.example                    # Template (no real secrets)
├── init-db.sh                      # Creates multiple Postgres databases
├── config/
│   ├── kratos/
│   │   ├── kratos.yml
│   │   ├── schemas/
│   │   │   └── user.schema.json
│   │   └── oidc-mappers/
│   │       └── google.jsonnet
│   ├── hydra/
│   │   └── hydra.yml
│   ├── oathkeeper/
│   │   ├── oathkeeper.yml
│   │   ├── rules.yml
│   │   └── jwks.json
│   ├── traefik/
│   │   ├── traefik.yml
│   │   └── dynamic/
│   │       └── middlewares.yml
│   └── prometheus/
│       └── prometheus.yml
├── scripts/
│   ├── backup.sh
│   ├── restore.sh
│   └── rotate-secrets.sh
└── docs/
    ├── incident-response.md
    └── oidc-client-setup.md
```

---

# Appendix B: Secret Checklist

Generate these secrets before deployment:

```bash
# Infisical bootstrap (store securely offline)
INFISICAL_ENCRYPTION_KEY=$(openssl rand -hex 32)
INFISICAL_AUTH_SECRET=$(openssl rand -hex 32)

# Database passwords
POSTGRES_PASSWORD=$(openssl rand -base64 24)
POSTGRES_KRATOS_PASSWORD=$(openssl rand -base64 24)
POSTGRES_HYDRA_PASSWORD=$(openssl rand -base64 24)
POSTGRES_INFISICAL_PASSWORD=$(openssl rand -base64 24)
REDIS_PASSWORD=$(openssl rand -base64 24)

# Kratos secrets
KRATOS_SECRETS_COOKIE=$(openssl rand -hex 32)
KRATOS_SECRETS_CIPHER=$(openssl rand -hex 32)
KRATOS_UI_COOKIE_SECRET=$(openssl rand -hex 32)
KRATOS_UI_CSRF_SECRET=$(openssl rand -hex 32)

# Hydra secrets
HYDRA_SECRETS_SYSTEM=$(openssl rand -hex 32)
HYDRA_SECRETS_COOKIE=$(openssl rand -hex 32)
HYDRA_OIDC_SALT=$(openssl rand -hex 32)

# External services
SMTP_PASSWORD=<from-email-provider>
GOOGLE_OAUTH_CLIENT_SECRET=<from-google-console>

# Monitoring
GRAFANA_PASSWORD=$(openssl rand -base64 16)
```

---

# Document Metadata

| Attribute | Value |
|-----------|-------|
| **Version** | 1.0 |
| **Status** | Ready for Implementation |
| **Last Updated** | January 2026 |
| **Prerequisites** | CentralizedAuthenticationInfra.md, SSO-CrossCuttingOWASP |
| **Estimated Implementation** | 4-5 weeks |
| **Review Cycle** | After each phase completion |

---

## Glossary

### AAL (Authentication Assurance Level)
- **Description**: A metric defined by [NIST](#glossary) indicating the strength of authentication (AAL1 = single factor, AAL2 = multi-factor, AAL3 = hardware-backed MFA)
- **Use this when**: Requiring step-up authentication for sensitive operations like admin panels or financial transactions
- **Like**: Security clearance levels—higher AAL = stricter proof of identity

### ACME (Automatic Certificate Management Environment)
- **Description**: A protocol for automating [TLS](#glossary)/[SSL](#glossary) certificate issuance and renewal, popularized by Let's Encrypt
- **Use this when**: Configuring [Traefik](#glossary) or other reverse proxies to automatically obtain HTTPS certificates without manual intervention
- **Like**: An automated locksmith that renews your building keys before they expire

### API (Application Programming Interface)
- **Description**: A set of defined methods and endpoints for communication between software applications, typically using HTTP requests
- **Use this when**: Building services that need to exchange data programmatically, like mobile apps calling backend servers
- **Like**: A restaurant menu—a defined list of what you can order and how to request it

### ARM (Advanced RISC Machine / Acorn RISC Machine)
- **Description**: A CPU architecture family known for power efficiency, commonly used in mobile devices and increasingly in cloud servers (e.g., [Oracle](#glossary) Free Tier Ampere processors)
- **Use this when**: Deploying to ARM-based servers; note that some Docker images may need multi-architecture builds
- **Like**: A fuel-efficient engine—does the same work as traditional CPUs but uses less power

### Authorization Code
- **Description**: A short-lived, one-time-use token issued by an [OAuth2](#glossary) server after user authentication, exchanged for access tokens by the client application
- **Use this when**: Implementing the [OAuth2](#glossary) Authorization Code flow, which is the most secure flow for web applications
- **Like**: A claim ticket at a coat check—you use it once to retrieve your actual belongings (tokens)

### Bearer Token
- **Description**: An access token type where simply possessing the token ("bearing" it) grants access to resources, typically sent in HTTP Authorization headers
- **Use this when**: Securing API requests after authentication; the token itself is proof of authorization
- **Like**: A movie ticket—whoever holds it can enter, no ID required

### BOLA (Broken Object Level Authorization)
- **Description**: An [OWASP](#glossary) API security vulnerability where attackers can access objects belonging to other users by manipulating IDs in requests
- **Use this when**: Auditing APIs to ensure users can only access their own resources (e.g., checking user_id in database queries)
- **Like**: Being able to open other people's gym lockers because the numbers aren't checked properly

### CDN (Content Delivery Network)
- **Description**: A geographically distributed network of servers that cache and serve content from locations closest to end users, reducing latency
- **Use this when**: Serving static assets (images, CSS, JS) or leveraging [Cloudflare](#glossary)'s edge network for faster global response times
- **Like**: Having franchise locations of a restaurant instead of one central kitchen—customers get served faster from nearby locations

### CLI (Command-Line Interface)
- **Description**: A text-based interface for interacting with software via typed commands in a terminal, as opposed to graphical user interfaces
- **Use this when**: Automating tasks, managing servers via SSH, or using developer tools like [Docker](#glossary), git, or [Infisical](#glossary) CLI
- **Like**: Typing specific instructions to a librarian instead of browsing shelves yourself—faster for experts

### Cloudflare
- **Description**: A cloud platform providing [CDN](#glossary), [DNS](#glossary), [DDoS](#glossary) protection, [WAF](#glossary), and [Zero Trust](#glossary) services; acts as a reverse proxy in front of origin servers
- **Use this when**: You need edge security, global content delivery, or want to hide your origin server IP from attackers
- **Like**: A combination security guard + valet service standing between visitors and your building

### CORS (Cross-Origin Resource Sharing)
- **Description**: A browser security mechanism that restricts web pages from making requests to domains different from the serving domain, unless explicitly allowed by HTTP headers
- **Use this when**: Your frontend (e.g., bumerange.com) needs to call APIs on different domains (e.g., api.bumerange.com or auth.bumerange.com)
- **Like**: A building security policy about which other buildings' employees can enter—you need explicit permission

### CSRF (Cross-Site Request Forgery)
- **Description**: An attack where a malicious site tricks a user's browser into making unwanted requests to a different site where the user is authenticated
- **Use this when**: Implementing forms and state-changing operations; mitigated by CSRF tokens or SameSite cookies
- **Like**: Someone forging your signature on checks while you're logged into your bank—prevented by requiring unique per-transaction codes

### DDoS (Distributed Denial of Service)
- **Description**: A cyberattack where multiple compromised systems flood a target with traffic, overwhelming it and making it unavailable to legitimate users
- **Use this when**: Assessing infrastructure resilience; [Cloudflare](#glossary) provides automatic DDoS mitigation at the edge
- **Like**: A flash mob blocking a store entrance so real customers can't get in

### DNS (Domain Name System)
- **Description**: The distributed system that translates human-readable domain names (like bumerange.com) into IP addresses that computers use to connect
- **Use this when**: Configuring domains to point to your servers, setting up subdomains, or managing [TLS](#glossary) verification
- **Like**: A phone book that converts names into phone numbers

### Docker
- **Description**: A containerization platform that packages applications and their dependencies into isolated, portable units called containers
- **Use this when**: Deploying consistent environments across development and production, or running multiple services on a single host
- **Like**: Shipping containers for software—everything needed is packaged in a standardized box that runs anywhere

### Docker Compose
- **Description**: A tool for defining and managing multi-container [Docker](#glossary) applications using YAML configuration files
- **Use this when**: Running complex stacks with multiple interconnected services (like [Kratos](#glossary), [Hydra](#glossary), [Postgres](#glossary), [Redis](#glossary)) as a coordinated system
- **Like**: A restaurant menu that orders multiple dishes at once and ensures they're all prepared together

### DSN (Data Source Name)
- **Description**: A connection string containing database credentials, host, port, and database name used by applications to connect to databases
- **Use this when**: Configuring [Kratos](#glossary), [Hydra](#glossary), or applications to connect to [Postgres](#glossary) or other databases
- **Like**: A full mailing address including street, city, zip code, and recipient name

### GDPR (General Data Protection Regulation)
- **Description**: European Union regulation governing personal data protection and privacy, requiring explicit user consent, data portability, and breach notification
- **Use this when**: Handling data of EU citizens; requires features like account deletion, data export, and explicit consent for data collection
- **Like**: A comprehensive set of rules about how businesses must handle customer information, with heavy penalties for violations

### Grafana
- **Description**: An open-source analytics and visualization platform that creates dashboards from metrics collected by systems like [Prometheus](#glossary)
- **Use this when**: Building operational dashboards to monitor system health, performance, and resource usage
- **Like**: A car's dashboard displaying speed, fuel, engine temperature—visual metrics for system health

### HTTPS (Hypertext Transfer Protocol Secure)
- **Description**: The encrypted version of HTTP, using [TLS](#glossary) to protect data in transit between browsers and servers
- **Use this when**: Serving any website (required for secure cookies, credential forms, and general security best practices)
- **Like**: Sending a letter in a locked box instead of a transparent envelope

### Hydra
- **Description**: [Ory](#glossary) Hydra is an open-source [OAuth2](#glossary) and [OpenID Connect](#glossary) server that issues tokens for cross-domain [SSO](#glossary)
- **Use this when**: You need [OIDC](#glossary)/[OAuth2](#glossary) authorization server capabilities for multi-domain SSO (different TLDs that can't share cookies)
- **Like**: A ticket booth that issues entry passes to multiple theme parks owned by the same company

### IDOR (Insecure Direct Object Reference)
- **Description**: A vulnerability where attackers can access unauthorized objects by manipulating reference IDs in requests; a specific type of [BOLA](#glossary)
- **Use this when**: Testing API security—can user A access user B's data by changing an ID parameter?
- **Like**: Being able to retrieve someone else's medical records by changing the patient number in the hospital system

### Infisical
- **Description**: An open-source secrets management platform for storing, managing, and injecting secrets (passwords, API keys, certificates) into applications
- **Use this when**: You need centralized secrets management with team access control, audit logs, and environment-specific secrets
- **Like**: A digital vault where teams can safely store and retrieve sensitive keys without exposing them in code

### JWT (JSON Web Token)
- **Description**: A compact, URL-safe token format that contains encoded JSON claims, typically used for authentication and information exchange
- **Use this when**: Implementing stateless authentication, passing user identity between services, or issuing access tokens in [OAuth2](#glossary)
- **Like**: A tamper-evident wristband at an event with encoded information about your ticket type

### K3s
- **Description**: A lightweight [Kubernetes](#glossary) distribution designed for resource-constrained environments, IoT, and edge computing
- **Use this when**: You outgrow [Docker Compose](#glossary) and need container orchestration but don't have the resources for full Kubernetes
- **Like**: A compact car with most features of a full-size vehicle but more efficient for city driving

### Kratos
- **Description**: [Ory](#glossary) Kratos is an open-source identity and user management server handling registration, login, MFA, account recovery, and user profiles
- **Use this when**: You need a self-hosted identity provider with modern security practices (no passwords stored, MFA built-in)
- **Like**: The user account management system of a company—handles who you are, your profile, and how you prove your identity

### Kubernetes
- **Description**: An open-source container orchestration platform for automating deployment, scaling, and management of containerized applications
- **Use this when**: Operating at scale beyond single-server deployments, requiring auto-scaling, self-healing, and complex service meshes
- **Like**: A factory automation system that manages assembly lines, worker distribution, and product flow

### Let's Encrypt
- **Description**: A free, automated certificate authority providing [TLS](#glossary)/[SSL](#glossary) certificates via the [ACME](#glossary) protocol
- **Use this when**: Obtaining HTTPS certificates for your domains; [Traefik](#glossary) can automatically integrate with Let's Encrypt
- **Like**: A government service that issues free ID cards with automatic renewal

### MFA (Multi-Factor Authentication)
- **Description**: An authentication method requiring two or more verification factors (something you know, have, or are) to gain access
- **Use this when**: Protecting high-value accounts or compliance-sensitive operations; [Kratos](#glossary) supports [TOTP](#glossary) and backup codes
- **Like**: Requiring both a key and a fingerprint scan to enter a secure facility

### NIST (National Institute of Standards and Technology)
- **Description**: A U.S. agency that publishes cybersecurity frameworks, standards, and guidelines like the Digital Identity Guidelines (SP 800-63)
- **Use this when**: Referencing authoritative security standards for authentication levels ([AAL](#glossary)), password policies, or cryptography
- **Like**: The organization that defines what makes a building structurally sound—authoritative standards everyone follows

### NVMe (Non-Volatile Memory Express)
- **Description**: A storage protocol designed for high-speed SSD storage, offering significantly faster performance than traditional SATA interfaces
- **Use this when**: Selecting [VPS](#glossary) storage options; NVMe provides faster database queries and application responsiveness
- **Like**: A high-speed train versus a regular train—gets you there much faster using optimized tracks

### Oathkeeper
- **Description**: [Ory](#glossary) Oathkeeper is an identity and access proxy that authenticates requests, enforces authorization rules, and enriches requests with user context headers
- **Use this when**: You need centralized authentication enforcement for multiple services without modifying each application's code
- **Like**: A gatekeeper who checks everyone's ID before letting them through and stamps their hand with access level information

### OAuth2
- **Description**: An authorization framework enabling applications to obtain limited access to user accounts via token-based flows, delegating authentication to identity providers
- **Use this when**: Implementing "Login with Google" or cross-domain [SSO](#glossary) where users authorize apps to access their data
- **Like**: Giving valet keys to a parking attendant—limited access for a specific purpose, not full ownership

### OCPU (Oracle Cloud Processing Unit)
- **Description**: Oracle Cloud's measure of CPU capacity, where 1 OCPU typically equals 2 [vCPUs](#glossary) with dedicated hyperthreading
- **Use this when**: Provisioning resources in Oracle Cloud; the free tier includes 4 OCPUs on [ARM](#glossary) architecture
- **Like**: A measurement unit like horsepower—describes compute capability but different manufacturers measure differently

### OIDC (OpenID Connect)
- **Description**: An identity layer built on top of [OAuth2](#glossary) that provides standardized authentication and user profile information via [ID tokens](#glossary)
- **Use this when**: Implementing true single sign-on across domains; apps use OIDC to verify user identity via a central provider
- **Like**: A standardized passport system—all countries (apps) recognize the same document format for identity verification

### OpenID Connect
- **Description**: See [OIDC](#glossary)—the full name for the protocol that extends [OAuth2](#glossary) with authentication capabilities
- **Use this when**: Referencing the formal specification or distinguishing from plain OAuth2 authorization flows
- **Like**: The official name of a program, while OIDC is its common abbreviation

### Oracle
- **Description**: Oracle Corporation, provider of Oracle Cloud Infrastructure including a generous free tier (4 [OCPU](#glossary), 24GB [RAM](#glossary) on [ARM](#glossary))
- **Use this when**: Referring to Oracle Cloud as backup infrastructure or overflow capacity beyond your primary [VPS](#glossary)
- **Like**: A company name, like saying "Amazon" when referring to AWS

### Ory
- **Description**: The company and open-source project providing a suite of identity infrastructure tools ([Kratos](#glossary), [Hydra](#glossary), [Oathkeeper](#glossary))
- **Use this when**: Referring to the ecosystem as a whole or the company behind the projects
- **Like**: The brand name for a suite of security tools, like "Microsoft Office" for productivity tools

### OWASP (Open Web Application Security Project)
- **Description**: A nonprofit foundation producing freely available resources on application security, including Top 10 vulnerability lists and security cheat sheets
- **Use this when**: Referencing industry-standard security best practices, vulnerability classifications, or threat modeling
- **Like**: Consumer Reports for web security—trusted, independent security research everyone references

### PKCE (Proof Key for Code Exchange)
- **Description**: An [OAuth2](#glossary) security extension that prevents authorization code interception attacks by requiring a dynamically generated secret
- **Use this when**: Implementing OAuth2 flows, especially for mobile/SPA clients; should be enforced for all clients per modern standards
- **Like**: Requiring the claim ticket stub to match the torn half at the coat check—proves the retriever is the original depositor

### Postgres
- **Description**: PostgreSQL, an open-source relational database known for reliability, feature robustness, and standards compliance
- **Use this when**: You need a production-grade SQL database for identity data, application state, or structured data storage
- **Like**: A highly organized filing system with strict rules about data relationships and integrity

### PostgreSQL
- **Description**: See [Postgres](#glossary)—the formal name for the database system
- **Use this when**: Referencing official documentation or formal technical specifications
- **Like**: The official name versus the common shorthand everyone uses

### Prometheus
- **Description**: An open-source monitoring and alerting toolkit that collects metrics via HTTP pulls and stores them as time-series data
- **Use this when**: Implementing infrastructure monitoring; scrapes metrics from services and stores them for [Grafana](#glossary) visualization
- **Like**: A data logger that regularly checks and records equipment readings over time

### Proxy
- **Description**: An intermediary server that forwards requests between clients and servers, often adding functionality like caching, security, or load balancing
- **Use this when**: Configuring [Cloudflare](#glossary) (reverse proxy), [Traefik](#glossary) (reverse proxy), or [Oathkeeper](#glossary) (identity proxy)
- **Like**: A receptionist who receives all visitors, checks authorization, and directs them to the right department

### RAM (Random Access Memory)
- **Description**: Volatile computer memory used for running processes and caching data; more RAM allows more simultaneous operations
- **Use this when**: Sizing servers—16GB RAM in this architecture accommodates the full [Ory](#glossary) stack plus database caching
- **Like**: A desk workspace—more space lets you work on more projects simultaneously

### Redis
- **Description**: An open-source, in-memory data structure store used as a database, cache, message broker, and session store
- **Use this when**: You need extremely fast session validation, rate limiting counters, or temporary data caching
- **Like**: A notepad attached to your desk—instant access to frequently needed information

### Reverse Proxy
- **Description**: A server that sits in front of backend servers, routing client requests to appropriate backends while potentially adding features
- **Use this when**: Configuring [Traefik](#glossary) or [Cloudflare](#glossary) to route traffic, handle [TLS](#glossary) termination, or enforce access policies
- **Like**: A hotel concierge who directs guests to the right rooms and services without guests needing to know the building layout

### RFC (Request for Comments)
- **Description**: A publication series from the Internet Engineering Task Force (IETF) that defines internet standards and protocols
- **Use this when**: Referencing authoritative protocol specifications like OAuth2 (RFC 6749) or HTTP (RFC 7231)
- **Like**: The official rulebook for how internet technologies should work

### SaaS (Software as a Service)
- **Description**: A software delivery model where applications are hosted by a provider and accessed via the internet, rather than installed locally
- **Use this when**: Comparing self-hosted solutions (like this architecture) to managed cloud services
- **Like**: Renting a furnished apartment vs owning and furnishing your own house—convenience vs control

### SameSite
- **Description**: A cookie attribute controlling whether cookies are sent with cross-site requests; values include Strict, Lax, and None
- **Use this when**: Configuring session cookies to prevent [CSRF](#glossary) attacks; Lax is recommended for auth cookies
- **Like**: A package delivery rule—"Strict" means only accept packages from the same building, "Lax" allows your own couriers

### SDK (Software Development Kit)
- **Description**: A collection of tools, libraries, documentation, and code samples for building applications for a specific platform or service
- **Use this when**: Integrating with services like [Infisical](#glossary) via their Node.js/Python/Go SDKs instead of raw API calls
- **Like**: A complete toolbox with instructions—everything you need to build something for a specific purpose

### SMTP (Simple Mail Transfer Protocol)
- **Description**: The standard protocol for sending email messages between servers, used by applications to send transactional emails
- **Use this when**: Configuring [Kratos](#glossary) to send verification emails, password reset links, or login notifications
- **Like**: The postal service protocol—standardized rules for how mail moves from sender to recipient

### SOC2 (Service Organization Control 2)
- **Description**: An auditing standard for service providers storing customer data, focusing on security, availability, processing integrity, confidentiality, and privacy
- **Use this when**: Planning for compliance as you scale; this architecture's audit logs and access controls support SOC2 requirements
- **Like**: A certification that you follow strict security practices, like a restaurant's health inspection grade

### SSH (Secure Shell)
- **Description**: A cryptographic network protocol for secure remote login and command execution on servers
- **Use this when**: Managing your [VPS](#glossary) remotely, deploying code, or accessing server logs
- **Like**: A secure telephone line to your server where conversations can't be eavesdropped

### SSL (Secure Sockets Layer)
- **Description**: The predecessor to [TLS](#glossary); term still commonly used to refer to TLS certificates and HTTPS encryption
- **Use this when**: Discussing certificates in general, though technically TLS is the modern protocol
- **Like**: An older brand name that became generic—people still say "SSL certificates" even though they're actually TLS

### SSO (Single Sign-On)
- **Description**: An authentication scheme allowing users to log in once and access multiple applications without re-authenticating
- **Use this when**: Users need seamless access across your suite of applications (bumerange.com, rumnia.com, coyotee-renato.cloud)
- **Like**: A master key that opens all doors in a building—one authentication grants access to everything

### Supabase
- **Description**: An open-source Firebase alternative providing hosted [Postgres](#glossary), authentication, storage, and real-time subscriptions
- **Use this when**: Building applications that need backend-as-a-service; free tier includes 500MB database (used for app data, not auth)
- **Like**: A pre-built backend infrastructure package—like buying a prefab office instead of constructing one from scratch

### TLD (Top-Level Domain)
- **Description**: The last segment of a domain name (like .com, .org, .cloud), determining the domain's hierarchical namespace
- **Use this when**: Explaining why cookie sharing works for subdomains but not across different TLDs (bumerange.com vs rumnia.com)
- **Like**: Different countries in the world—each has its own rules and you need special arrangements to move between them

### TLS (Transport Layer Security)
- **Description**: A cryptographic protocol providing secure communication over networks, successor to [SSL](#glossary); versions include 1.2 and 1.3
- **Use this when**: Configuring HTTPS, understanding certificate requirements, or enforcing encrypted connections
- **Like**: A secure courier service that ensures packages can't be opened or tampered with during transit

### TOTP (Time-Based One-Time Password)
- **Description**: An algorithm generating temporary codes based on current time, used by authenticator apps like Google Authenticator or Authy
- **Use this when**: Implementing [MFA](#glossary); [Kratos](#glossary) supports TOTP as a second factor for [AAL2](#glossary) authentication
- **Like**: A security token that generates a new password every 30 seconds, synchronized with the server

### Traefik
- **Description**: A modern HTTP reverse proxy and load balancer that integrates natively with [Docker](#glossary) and automatically manages [TLS](#glossary) certificates
- **Use this when**: You need ingress management with automatic service discovery, Let's Encrypt integration, and middleware support
- **Like**: An intelligent router that automatically learns about new services and directs traffic to them with encryption

### vCPU (Virtual Central Processing Unit)
- **Description**: A virtualized processor core allocated to a virtual machine; performance varies by hypervisor and host hardware
- **Use this when**: Sizing [VPS](#glossary) resources; 4 vCPUs in this architecture provide sufficient compute for the [Ory](#glossary) stack
- **Like**: A slice of a physical processor dedicated to your virtual machine—how much brain power your VM gets

### VPS (Virtual Private Server)
- **Description**: A virtualized server instance running on shared physical hardware, providing dedicated resources (CPU, RAM, storage) at lower cost than dedicated servers
- **Use this when**: Deploying applications that need persistent infrastructure with root access; this architecture uses Hostinger KVM VPS
- **Like**: An apartment in a building—you have your own dedicated space, but share the building infrastructure

### WAF (Web Application Firewall)
- **Description**: A security layer that filters and monitors HTTP traffic to/from web applications, blocking common attacks like SQL injection or XSS
- **Use this when**: Configuring [Cloudflare](#glossary) security rules to protect against automated attacks and malicious traffic
- **Like**: A security screener at an airport—inspects all incoming traffic and blocks items matching threat patterns

### YAML (YAML Ain't Markup Language)
- **Description**: A human-readable data serialization format commonly used for configuration files, particularly in [Docker Compose](#glossary) and Kubernetes
- **Use this when**: Writing configuration files for [Docker Compose](#glossary), [Kratos](#glossary), [Hydra](#glossary), or [Traefik](#glossary)
- **Like**: A structured way of writing down settings using indentation and colons—like a detailed outline format

### Zero Trust
- **Description**: A security model assuming no implicit trust based on network location, requiring verification for every access request
- **Use this when**: Configuring [Cloudflare](#glossary) Zero Trust to protect admin interfaces like Infisical and Traefik dashboards
- **Like**: A high-security facility where everyone shows ID every time they enter, regardless of whether they work there

---

*This tech spec is implementation-ready. Execute phases sequentially, verify checkpoints before proceeding. All secrets must be stored in Infisical before Phase 2.*

