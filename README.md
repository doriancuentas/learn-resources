# Learning Resources Repository

A curated collection of technical documentation covering data engineering, system design, infrastructure, authentication security, and software architecture.

---

## Document Index

### [DataCatalogsAndFormatsV1.md](./DataCatalogsAndFormatsV1.md)
Modern table formats and data catalog systems for lakehouse architectures with Iceberg, Delta Lake, and Hudi.
`Tags: Iceberg, Delta Lake, Hudi, Data Catalog, ACID, MVCC, Time Travel`

### [DataPipelineSystemDesign.md](./DataPipelineSystemDesign.md)
Production pipeline orchestration comparing Airflow, Prefect, Dagster, and Temporal with failure handling and multi-tenancy.
`Tags: Airflow, Prefect, Dagster, Temporal, DAG, Orchestration, State Management`

### [DistributedQuerySystems.md](./DistributedQuerySystems.md)
50-year evolution of distributed query engines from MapReduce to modern lakehouse architectures.
`Tags: MPP, Columnar Storage, Query Optimization, Snowflake, BigQuery, Trino`

### [HiveIcebergAutodiscovery.md](./HiveIcebergAutodiscovery.md)
Partition discovery in Hive Metastore and modern Iceberg architectures with performance analysis.
`Tags: Hive Metastore, Partition Discovery, Iceberg, AWS Glue, S3 LIST, Metadata`

### [SnowflakeStorageIntegration.md](./SnowflakeStorageIntegration.md)
Production Snowflake integration with AWS S3 using IAM roles and storage integrations.
`Tags: Snowflake, S3, IAM, Storage Integration, STS Tokens, Virtual Warehouse`

### [MessageQueueVsStreamProcessing.md](./MessageQueueVsStreamProcessing.md)
Performance comparison of messaging infrastructure: SQS, RabbitMQ, Kafka, Pulsar, and Redis Streams.
`Tags: SQS, RabbitMQ, Kafka, Pulsar, Redis, Event Sourcing, CQRS`

### [PipelineOrchestrationFrameworks.md](./PipelineOrchestrationFrameworks.md)
Open-source orchestration frameworks for event-driven systems by latency requirements.
`Tags: Benthos, Camel, Kestra, Temporal, Prefect, Argo, EIP Patterns`

### [SupabaseB2CSystemDesign-ClaudeOpus4.5.md](./SupabaseB2CSystemDesign-ClaudeOpus4.5.md)
Startup B2C authentication architecture with Supabase including SSO, RLS, MFA, and OAuth2.1.
`Tags: Supabase, OAuth2.1, OIDC, PKCE, SSO, RLS, JWT, MFA`

### [SupabaseB2CSystemDesign-GPT5.2.md](./SupabaseB2CSystemDesign-GPT5.2.md)
Enterprise Supabase architecture with multi-app SSO, fine-grained authorization, and credential lifecycle management.
`Tags: Supabase, RLS, ReBAC, RBAC, ABAC, Policy Decision Points, OpenFGA`

### [ServiceMeshCommunication.md](./ServiceMeshCommunication.md)
Service mesh architecture with Envoy, SPIFFE/SPIRE, mTLS, and observability for microservices.
`Tags: Envoy, SPIFFE, SPIRE, mTLS, Service Mesh, Sidecar, Observability`

### [FlutterAndSupabaseLearning.md](./FlutterAndSupabaseLearning.md)
9-week guide to building Flutter mobile apps with Supabase backend and AI integration.
`Tags: Flutter, Supabase, OAuth, RLS, Real-time, Edge Functions, OpenAI`

### [CentralizedAuthenticationInfra.md](./CentralizedAuthenticationInfra.md)
Centralized authentication infrastructure with Ory stack, SSO, PAT/API keys, and MFA enforcement.
`Tags: Ory Kratos, Oathkeeper, Hydra, SSO, OAuth2, OIDC, PAT, API Keys, MFA`

### [CentralizedAuth-TechSpec-Implementation.md](./CentralizedAuth-TechSpec-Implementation.md)
Implementation-ready technical spec for centralized auth infrastructure with Docker Compose and VPS deployment.
`Tags: Ory Stack, Docker Compose, VPS, Traefik, Infisical, PKCE, Cloudflare, OWASP`

### [SSO-CrossCuttingOWASP.md](./SSO-CrossCuttingOWASP.md)
OWASP security patterns for SSO ecosystems preventing BOLA, IDOR, session hijacking, and OAuth vulnerabilities.
`Tags: OWASP, BOLA, IDOR, CSRF, XSS, JWT Security, OAuth Security, Rate Limiting`

### [BMADToolsCheatsheet.md](./BMADToolsCheatsheet.md)
BMAD Method tools quick reference for AI-assisted development with agent commands and workflows.
`Tags: BMAD Method, AI Agents, Cursor IDE, Gemini, ChatGPT, v0.dev, Lovable`

### [BMADMethod.md](./BMADMethod.md)
Technical deep-dive into BMAD-METHOD agent orchestration framework with provider-agnostic LLM interface.
`Tags: BMAD Method, AI Agents, LLM Orchestration, RAG, OpenAI, Anthropic, Vertex`

### [BumerangeToTerraform.md](./BumerangeToTerraform.md)
Production Terraform infrastructure with three-layer separation, AWS provisioning, and migration safety.
`Tags: Terraform, IaC, AWS, EC2, IAM, RDS, PostgreSQL, CloudTrail`

### [ClaudeEcoplugs.md](./ClaudeEcoplugs.md)
Critical analysis of Claude Code architecture covering 9 core components and enterprise comparisons.
`Tags: Claude Code, AI Development, Workflows, Telemetry, RAG, OpenTelemetry, Temporal`

---

## 👨‍💻 Developer Profile

Based on the comprehensive research and documentation in this repository, this collection represents the knowledge base of a **Senior Staff Engineer / Principal Engineer** with deep expertise spanning multiple domains:

**Core Competencies:**
- **Data Engineering Architecture** - Expert in modern lakehouse architectures (Apache Iceberg, Delta Lake, Hudi), distributed query engines (Trino, Presto, Spark), and data catalog systems. Deep understanding of metadata management, partition discovery, and ACID transactions on object storage.
- **Cloud-Native Infrastructure** - Extensive experience with AWS services (S3, Glue, Athena, Lambda, SQS), Snowflake data warehousing, and infrastructure-as-code using Terraform. Proven ability to design secure, scalable, and cost-effective cloud architectures.
- **Distributed Systems & Microservices** - Advanced knowledge of service mesh architectures (Envoy, mTLS, SPIFFE/SPIRE), message queuing vs. streaming platforms (Kafka, Pulsar, RabbitMQ), and pipeline orchestration frameworks (Airflow, Temporal, Prefect).
- **Authentication & Identity Security** - Specialized expertise in centralized authentication infrastructure, SSO ecosystems (same-domain and cross-domain), OAuth2/OIDC flows with PKCE, JWT-based session management, and MFA enforcement. Proficient in Ory stack (Kratos, Oathkeeper, Hydra), Traefik, Infisical secrets management, and OWASP-aligned security patterns for preventing BOLA, IDOR, session hijacking, token replay, and OAuth vulnerabilities. Implementation experience includes multi-domain SSO, PAT/API key systems, distributed authentication with sidecars and reverse proxies, and production-grade deployment with Docker Compose on VPS infrastructure.
- **Full-Stack Development** - Proficient in modern application architectures using Supabase (PostgreSQL, RLS, Auth), Flutter for mobile development, and B2C authentication patterns including OAuth2.1, OIDC, and multi-factor authentication.
- **AI-Assisted Development** - Deep experience with AI agent frameworks (BMAD-METHOD, Claude Code), LLM orchestration, and RAG patterns. Understands how to leverage AI for software development workflows from planning through deployment.

**Engineering Philosophy:**
- Emphasizes **separation of concerns** and **modularity** in system design
- Values **security by default** with defense-in-depth strategies and OWASP compliance
- Advocates for **observable, debuggable systems** with comprehensive telemetry
- Champions **data-driven decision making** with clear performance benchmarks
- Believes in **documentation as code** with visual diagrams and comprehensive glossaries
- Prioritizes **vendor-agnostic, open-source solutions** to avoid lock-in

**Technical Depth:**
This repository demonstrates mastery of 50+ years of distributed systems evolution, from relational databases through NoSQL to modern lakehouse architectures, combined with practical expertise in cloud-native development, security patterns, centralized authentication infrastructure, and AI-powered workflows. The breadth spans from low-level infrastructure (networking, certificates, IAM, OAuth flows) to high-level application architecture (authentication flows, state management, user experience, multi-tenant security).

**Target Role:** Senior Staff Engineer / Principal Engineer / Solutions Architect specializing in data platforms, distributed systems, cloud-native architectures, and authentication/identity infrastructure.

---

## 📝 Excluded from Current Review
- ⏸️ **DietaryPlanningProtocol.md** - Comprehensive dietary planning with equations and sources (non-technical content, skipped per instructions)

---

## Repository Summary

### Overall Statistics
- **Total Documents**: 19 technical guides
- **Total Mermaid Diagrams**: 290+ professional diagrams
- **Total Glossary Terms**: 850+ comprehensive definitions
- **Total Glossary Links**: 1,700+ cross-references
- **Total Lines of Content**: 25,000+ lines

### Document Categories

**Data Engineering & Pipelines** (7 documents)
- 117+ diagrams covering data architectures, pipelines, and query systems
- 400+ glossary terms for data engineering concepts
- Topics: Iceberg, Delta Lake, Hive, Kafka, streaming, orchestration, Snowflake

**System Design & Architecture** (4 documents)
- 67+ diagrams for system architecture and auth flows
- 200+ glossary terms for distributed systems and authentication
- Topics: Supabase, service mesh, Flutter, B2C systems, microservices

**Authentication & Security** (3 documents)
- 65+ diagrams for authentication flows, security architecture, OWASP threats
- 185+ glossary terms for auth, OAuth, OIDC, security patterns
- Topics: Ory stack, SSO, OAuth2/OIDC, PKCE, MFA, PAT/API keys, OWASP controls, Traefik, Infisical

**Development & Tools** (5 documents)
- 56+ diagrams for development workflows and tooling
- 150+ glossary terms for AI agents, infrastructure, and DevOps
- Topics: BMAD Method, Claude Code, Terraform, infrastructure as code, UTM/iOS virtualization

### Quality Standards Achieved

All documents now include:
- ✅ **Executive Summary** - Scaled appropriately to document length
- ✅ **Valid Mermaid Diagrams** - All labels properly quoted, consistent styling
- ✅ **Comprehensive Glossary** - Clear definitions with real-world analogies
- ✅ **Complete Cross-Referencing** - Every term occurrence linked to glossary
- ✅ **Professional Structure** - TOC, sections, references, and metadata

---

## Mermaid Diagram Guidelines

All documents in this repository follow these Mermaid diagram standards:

### Critical Rules

1. **Always wrap node labels with special characters in double quotes**
   ```mermaid
   flowchart TB
       A["Auth Server (OIDC/OAuth2)"]
       B["API Gateway: /api/v1"]
       C["Database (PostgreSQL)"]
   ```
   Special characters include: `() / : , -`

2. **Prefer `flowchart TB` (Top-Bottom) for readability**
   - Use `TB` for hierarchical flows
   - Use `LR` (Left-Right) for subgraphs to avoid wide charts

3. **Include environment boundaries**
   ```mermaid
   flowchart TB
       subgraph Development
           DevAPI["Dev API"]
           DevDB["Dev Database"]
       end
       subgraph Production
           ProdAPI["Prod API"]
           ProdDB["Prod Database"]
       end
   ```

4. **Clear component grouping**
   - Separate external services from internal components
   - Group related services in subgraphs
   - Add visual distinction with styling

---

## How to Use This Repository

### For Readers
1. Check the document index to find topics of interest
2. Click document links to read full content
3. Each document is self-contained with glossary and diagrams
4. Use CLAUDE.md for contributing guidelines

### For Contributors
1. Follow the Mermaid diagram guidelines strictly
2. Include executive summary and glossary in all new documents
3. Use PascalCase for filenames (e.g., `NewDocumentName.md`)
4. Add new documents to this README index
5. See CLAUDE.md for complete documentation standards

### Extracting Executive Summaries for Developer Profile Updates

To update the Developer Profile based on all document content:

```bash
# Extract all executive summaries
for file in *.md; do
  if [[ "$file" != "README.md" && "$file" != "CLAUDE.md" && "$file" != "dietary"* ]]; then
    echo "=== $file ==="
    grep -A 20 "## Executive Summary" "$file" 2>/dev/null | grep -B 20 "^##" | head -n -1
  fi
done
```

Then analyze the summaries to identify:
1. Technical domains covered (data engineering, cloud, auth, etc.)
2. Specific technologies mentioned (Kafka, Terraform, OAuth2, etc.)
3. Architectural patterns emphasized
4. Technical depth based on complexity
5. Update the Developer Profile to reflect consolidated expertise

---

## Repository Structure

```
learn-resources/
├── README.md                                    # This file
├── CLAUDE.md                                    # Documentation standards
├── DataCatalogsAndFormatsV1.md
├── DataPipelineSystemDesign.md
├── DistributedQuerySystems.md
├── HiveIcebergAutodiscovery.md
├── SnowflakeStorageIntegration.md
├── MessageQueueVsStreamProcessing.md
├── PipelineOrchestrationFrameworks.md
├── SupabaseB2CSystemDesign-ClaudeOpus4.5.md
├── SupabaseB2CSystemDesign-GPT5.2.md
├── ServiceMeshCommunication.md
├── FlutterAndSupabaseLearning.md
├── CentralizedAuthenticationInfra.md
├── CentralizedAuth-TechSpec-Implementation.md
├── SSO-CrossCuttingOWASP.md
├── BMADToolsCheatsheet.md
├── BMADMethod.md
├── BumerangeToTerraform.md
├── ClaudeEcoplugs.md
├── UTMoniOSFindings.md
└── DietaryPlanningProtocol.md
```

---

## Update Log

| Date | Action | Documents |
|------|--------|-----------|
| 2026-01-01 | Repository initialized | All documents |
| 2026-01-01 | File standardization - renamed to PascalCase | All 18 documents |
| 2026-01-01 | Comprehensive review and enhancement completed | All 15 original documents |
| 2026-01-01 | Added 230+ Mermaid diagrams, 750+ glossary terms | All 15 original documents |
| 2026-01-01 | Removed ephemeral progress tracking | README.md |
| 2026-01-01 | Added centralized authentication infrastructure guides | CentralizedAuthenticationInfra.md, CentralizedAuth-TechSpec-Implementation.md, SSO-CrossCuttingOWASP.md |
| 2026-01-01 | Enhanced with 50+ diagrams, 185+ auth/security glossary terms | Authentication & Security category |
| 2026-01-01 | Updated developer profile with authentication/security expertise | README.md |
| 2026-01-26 | Added UTM on iOS findings - ARM64 virtualization, bootable image creation | UTMoniOSFindings.md |
| 2026-01-26 | Restructured README with concise index format and tags | README.md |

---

## Contributing

When adding or updating documents:
1. Ensure all Mermaid diagrams follow the guidelines above
2. Include executive summary appropriate to document size
3. Create comprehensive glossary with all key terms
4. Link ALL occurrences of glossary terms throughout document
5. Test all Mermaid diagrams render correctly in GitHub
6. Update this README with new document entries (concise format with tags)
7. Follow PascalCase naming convention
8. See CLAUDE.md for complete documentation standards

---

**Note**: This is a living repository continuously updated with new technical research and documentation. For complete documentation standards and contribution guidelines, see [CLAUDE.md](./CLAUDE.md).
