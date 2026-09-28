# Snowflake MCP v2 OAuth: the ideal flow, the tricky parts, and the dev problem

This document explains, from high level to wire level, how a [Helix](#g-helix) user gets Snowflake results that run as their own Snowflake user, what is subtle about that [OAuth](#g-oauth) flow even in production, and why our two-[devapp](#g-devapp) setup cannot reproduce the first-time link without a trick.

Sources: [`token_broker`](#g-token-broker) and [`snowflake_v2`](#g-mcp) code as inspected at PR head `40e012e8a8b1`, plus the recorded implementation findings from trace items 29-47 and 77-85. The HTML companion captures the referenced modules, methods, actors, and evidence links; no source checkout is required to navigate this document.

How to read it:

1. Sections go from the big picture to wire detail. Read section 1 (with 1.1-1.6) for the pieces, where they run, the network boundaries, and prod versus dev; then sections 2-3 for the idea, section 4 for the HTTP payloads, section 5 for the traps that exist even in production, sections 6-7 for the dev problem and the candidate fix, and section 8 for when one devapp is enough and when you need two.
2. Placeholders in braces are never real values: `{jwt}` is a Pinterest [JWT](#g-jwt), `{code}` an [OAuth](#g-oauth) [authorization code](#g-authorization-code), `{state}` an [OAuth](#g-oauth) [state](#g-state), `{sf_client_id}` and `{sf_client_secret}` the Snowflake [integration](#g-security-integration) [credentials](#g-confidential-client), `{access_token}` and `{refresh_token}` Snowflake [access](#g-access-token) and [refresh](#g-refresh-token) tokens.
3. [SPIFFE IDs](#g-spiffe) are shortened in diagrams:
    - **a.** `ingress SPIFFE` = `spiffe://pin220.com/teletraan/ingress-pinadmin/prod-use1`
    - **b.** `devapp SPIFFE` = `spiffe://pin220.com/devapp/restricted/dmachicao`
    - **c.** `MCP SPIFFE` = the production [workload identity](#g-spiffe) of the v2 [MCP](#g-mcp) (not provisioned yet)
4. In Mermaid, `#59;` renders as a semicolon. It is used in [`X-Forwarded-Client-Cert`](#g-xfcc) values, which separate fields with semicolons.
5. Linked terms jump to their entry in [section 10](#10-terms-and-glossary), which gives a general definition, the meaning in this document, an example, and internal source-map references. Terms are not linked inside headings, diagrams, or code.
6. Code links:
    - **a.** Each diagram has a "Code references" list below it, keyed to node IDs (flowcharts) or step numbers (the numbers `autonumber` draws on sequence diagram arrows).
    - **b.** In the text, a `(code)` or `(config)` link opens the exact actor, module, class, method, route, model, or setting in `OAuthFlowAndDevProblemOffline.html`.
    - **c.** The HTML offers an expandable hierarchy and a relationship map. The active view controls whether a reference reveals the exact nested element or highlights its owning actor and relationship.

## 1. The cast

Five parties take part. Two of them, [Envoy](#g-envoy) and the [ingress](#g-ingress), are infrastructure that nobody in this project writes code for, but they decide who the [broker](#g-token-broker) thinks you are. That is the root of the dev problem.

| Party | Owner | Role |
| --- | --- | --- |
| [Helix](#g-helix) | Helix team | Chat UI. Calls [MCP](#g-mcp) tools and forwards the user's Pinterest [JWT](#g-jwt) (`forward_jwt: true`). |
| Snowflake [MCP](#g-mcp) v2 | us ([`snowflake_v2`](#g-mcp)) | Validates SQL, asks the [broker](#g-token-broker) for the user's Snowflake token, runs the query as the user. |
| [`token_broker`](#g-token-broker) | it-swe | Runs the [OAuth](#g-oauth) dance with Snowflake, stores encrypted tokens per user, [refreshes](#g-refresh-token) them. |
| [Envoy](#g-envoy) / [ingress](#g-ingress) | Traffic / platform | Terminates [TLS](#g-tls) and [mTLS](#g-mtls), enforces Pinterest [SSO](#g-sso) on some hosts, and injects identity headers. |
| Snowflake | Snowflake admins | [OAuth](#g-oauth) [authorization server](#g-authorization-server) ([security integration](#g-security-integration)) and the database. |

```mermaid
flowchart TB
    U(["User in a browser"])
    H["Helix"]
    M["Snowflake MCP v2"]
    B["token_broker"]
    DB[("Token store<br/>encrypted rows")]
    SF["Snowflake<br/>OAuth server and warehouse"]
    E{{"Envoy / ingress<br/>SSO and identity headers"}}

    U -- "chat" --> H
    H -- "tool call + user JWT" --> E --> M
    M -- "GET /api/token + user JWT" --> E
    E -- "adds caller SPIFFE (XFCC)" --> B
    U -- "clicks connect link" --> E
    E -- "adds X-Forwarded-User after SSO" --> B
    B <--> DB
    B -- "authorize, token exchange, refresh" --> SF
    M -- "SQL with the user's access token" --> SF
```

Code references:

1. `H`: Helix sets `Authorization: Bearer {jwt}` for MCP servers with `forward_jwt` on ([prompthub `backend/app/utils/mcp/server.py` L139-147](./OAuthFlowAndDevProblemOffline.html#actor-helix)).
2. `M`: the MCP server and its three tools ([`server.py` L21-37](./OAuthFlowAndDevProblemOffline.html#module-mcp-server)).
3. `M` to `E` (`GET /api/token`): ([`token_broker_client.py` L49-64](./OAuthFlowAndDevProblemOffline.html#method-mcp-get-snowflake-access-token)).
4. `E` to `B` (XFCC): the broker reads the caller from the header ([`spiffe.py` L23-58](./OAuthFlowAndDevProblemOffline.html#method-broker-parse-spiffe)).
5. `E` to `B` (`X-Forwarded-User` after SSO): the broker's Envoy config turns on `internal_oauth` and exempts `/oauth/callback` and `/health` ([pinconf `token-broker/prod/main/http.yaml` L8-21](./OAuthFlowAndDevProblemOffline.html#element-ingress-internal-oauth)); the broker reads the header ([`routes/oauth.py` L32-45](./OAuthFlowAndDevProblemOffline.html#method-broker-oauth-start)).
6. `DB`: tables `mcp_tokens`, `oauth_states`, and `audit_logs` ([`models.py` L13-90](./OAuthFlowAndDevProblemOffline.html#model-mcp-token)).
7. `B` to `SF`: Snowflake endpoints in the provider entry of draft PR 197114 ([`mcp_servers.production.yaml` L27-37](./OAuthFlowAndDevProblemOffline.html#class-broker-server-config)).
8. `M` to `SF`: ([`snowflake_client.py` L33-54](./OAuthFlowAndDevProblemOffline.html#method-mcp-snowflake-init)).

The diagram above shows who talks to whom. Sections 1.1-1.6 show where each party runs, which network boundaries each hop crosses, and how that differs between production and our devapps. Every party is a separate program: the [MCP](#g-mcp) and the [broker](#g-token-broker) never share a process or a [pod](#g-pod), and on our devapps they run on different machines.

### 1.1 Building blocks

These are the pieces the rest of the document assumes. Linked names have a full glossary entry in [section 10](#10-terms-and-glossary).

1. **Machine.** A computer, usually a virtual one in AWS (an EC2 instance). A [devapp](#g-devapp) is one EC2 machine per engineer, and you run programs on it directly (`bazel run`).
2. **Container.** A packaged program with its own files, started from an image. In production, `token_broker` runs from the `token_broker` image.
3. **[Pod](#g-pod).** The unit Kubernetes schedules: one or more containers that share a network address, so they reach each other on `localhost`. A pod holds one app container plus its [sidecars](#g-sidecar).
4. **[PinCompute](#g-pincompute).** Pinterest's Kubernetes platform. A service describes its pods in a `PinApp` file: image, replica count, sidecars, and service identity. MCP servers and `token_broker` run on PinCompute.
5. **[Sidecar](#g-sidecar).** A helper container in the same pod as the app. `token_broker` has two: [Envoy](#g-envoy), which handles all network traffic in and out, and [Knox](#g-knox), which delivers secrets.
6. **Service identity ([SPIFFE](#g-spiffe)).** Every production service gets a name like `spiffe://svc.pin220.com/<service>/<env>/main`, carried in the certificate its Envoy presents. People are identified by [JWTs](#g-jwt) and [SSO](#g-sso); services are identified by SPIFFE.
7. **[Service mesh](#g-mesh).** All the Envoy sidecars together. When pod A calls pod B, A's Envoy opens an [mTLS](#g-mtls) connection to B's Envoy. B's Envoy then knows A's SPIFFE ID, writes it into [XFCC](#g-xfcc), and checks a [Pastis](#g-pastis) policy before passing the request to B's app.
8. **[Pastis](#g-pastis).** Pinterest's authorization policies, written in Rego and enforced by the sidecar. For example, the broker's policy says which caller SPIFFE IDs may use `/api/*`.
9. **[Ingress](#g-ingress).** `ingress-pinadmin`, the Envoy fleet at the edge of the Pinterest network. It receives browser traffic for `*.pinadmin.com` and devapp `*.pinterdev.com` hosts, runs [SSO](#g-sso) on hosts configured for it, and forwards into the network. A request that comes through it carries the ingress's SPIFFE ID, not the original caller's.
10. **External SaaS.** Snowflake runs outside Pinterest, on the internet. Pinterest's Envoy, SPIFFE IDs and [Pastis](#g-pastis) policies mean nothing there; Snowflake trusts only its own [OAuth](#g-oauth) tokens and client credentials.

Anatomy of one production pod, using `token_broker` as the example:

```mermaid
flowchart LR
    subgraph POD["Pod: token-broker-prod (PinCompute, namespace token-broker)"]
        direction TB
        ENV["Envoy sidecar<br/>mTLS, XFCC, Pastis, SSO filter"]
        APP["token_broker container<br/>FastAPI app, MODE=production"]
        KNOX["Knox sidecar<br/>secrets and Tink keys"]
        ENV -- "plain HTTP on localhost" --> APP
        KNOX -- "files on a shared volume" --> APP
    end
    IN["Callers: other pods (mesh)<br/>or the ingress (browser)"] -- "mTLS" --> ENV
    APP -- "SPIFFE-authenticated MySQL" --> DB[("MySQL")]
    APP -- "HTTPS (egress path not verified)" --> SF["Snowflake (internet)"]
```

Code references:

1. `POD`: cluster `cr-m001-prod-use1`, namespace `token-broker` ([`token-broker-prodResourceSpec.yaml` L1-8](./OAuthFlowAndDevProblemOffline.html#element-runtime-pincompute)).
2. `ENV`, `KNOX`, `APP`: service identity, the two sidecars, the image, and the environment ([`token-broker-prod.yaml` L8-39](./OAuthFlowAndDevProblemOffline.html#element-runtime-pincompute)).
3. `DB`: MySQL through SPIFFE auth in production, SQLite in development and tests ([`engine.py` L36-66](./OAuthFlowAndDevProblemOffline.html#class-broker-engine)).
4. The Knox-to-app transport ("files on a shared volume") is how the Knox sidecar generally works at Pinterest. It was not verified for this service.
5. [Pastis](#g-pastis) on [PinCompute](#g-pincompute) MCP servers: ([`policy.md` L5](./OAuthFlowAndDevProblemOffline.html#element-mesh-pastis)).

### 1.2 Production: where each piece runs

Each box is a separate deployment, and each arrow is a network call. The label on an arrow says which boundary it crosses and what identity it carries. Snowflake MCP v2 has no production deployment yet (there is no `pindeploy/` folder at PR head `40e012e8a8b1`); its box follows the standard MCP server layout.

```mermaid
flowchart TB
    subgraph NET["Internet"]
        BR(["User's browser"])
        SF["Snowflake<br/>OAuth server + warehouse<br/>account pinterestit"]
    end
    subgraph PIN["Pinterest network"]
        ING{{"ingress-pinadmin<br/>Envoy fleet<br/>*.pinadmin.com, SSO"}}
        subgraph K8S["PinCompute (Kubernetes)"]
            HX["Helix pod<br/>spiffe://pin220.com/k8s/bex-tools/helix-prod"]
            MCP["MCP v2 pod (planned)<br/>app + Envoy<br/>spiffe://svc.pin220.com/mcp-server-snowflake-v2/prod/main"]
            BRK["token-broker pod<br/>app + Envoy + Knox<br/>spiffe://svc.pin220.com/token-broker/prod/main"]
        end
        DB[("MySQL<br/>encrypted token rows")]
        KX[("Knox<br/>client secret, Tink key")]
    end

    BR -- "1: chat UI, HTTPS + SSO" --> ING --> HX
    HX -- "2: tools/call + user JWT<br/>mesh mTLS, XFCC = Helix" --> MCP
    MCP -- "3: GET /api/token + user JWT<br/>mesh mTLS, XFCC = MCP" --> BRK
    BR -- "4: connect link, callback<br/>HTTPS, SSO sets X-Forwarded-User" --> ING
    ING -- "mesh mTLS, XFCC = ingress" --> BRK
    BRK --- DB
    BRK --- KX
    BR -- "5: Okta login + consent" --> SF
    BRK -- "6: code exchange, refresh<br/>HTTPS + client secret" --> SF
    MCP -- "7: SQL<br/>HTTPS + user access token" --> SF
```

How to read it:

1. Hops 2 and 3 stay inside the mesh. The receiving Envoy learns the caller's SPIFFE ID from the mTLS certificate, and [Pastis](#g-pastis) decides whether that caller may call. The user rides along as a JWT in the `Authorization` header.
2. Hop 4 is the only way a browser reaches the broker. The browser has no certificate, so the ingress handles it: it runs SSO, sets [`X-Forwarded-User`](#g-x-forwarded-user) to the user's [LDAP username](#g-ldap), and forwards into the mesh under its own SPIFFE ID.
3. Hops 5-7 leave Pinterest. Snowflake authenticates the user by Okta (5), the broker by the integration's client secret (6), and the MCP by the user's access token (7).

Code references:

1. `HX` SPIFFE, and the naming pattern for MCP servers: ([`oauth-mcp-server-pastis-traffic-pinconf.md` L20-31](./OAuthFlowAndDevProblemOffline.html#actor-ingress)).
2. `MCP` layout: MCP servers are a [`PinApp`](#g-pincompute) with a service identity and an Envoy sidecar ([gong `mcp-server-gong-staging.yaml` L7-14](./OAuthFlowAndDevProblemOffline.html#element-runtime-pincompute)). v2 has no `pindeploy/` yet; its dev config already names `spiffe://svc.pin220.com/mcp-server-snowflake-v2/dev/main` ([`runtime.dev.yaml` L216-220](./OAuthFlowAndDevProblemOffline.html#module-mcp-runtime-config)).
3. `BRK`: ([`token-broker-prod.yaml` L8-39](./OAuthFlowAndDevProblemOffline.html#element-runtime-pincompute)). The SPIFFE value shown follows the naming pattern above and was not read from the broker's [pinconf](#g-pinconf).
4. `ING` and hop 4: SSO on the broker's public host ([pinconf `token-broker/prod/main/http.yaml` L8-21](./OAuthFlowAndDevProblemOffline.html#element-ingress-internal-oauth)); the ingress config file ([`oauth-mcp-server-pastis-traffic-pinconf.md` L64-66](./OAuthFlowAndDevProblemOffline.html#actor-ingress)).
5. Hop 3 authorization: today the broker's [Pastis](#g-pastis) policy allows `/api/*` only from the helix-slackbot SPIFFE IDs ([pinconf `token-broker/pastis/base.rego` L8-22](./OAuthFlowAndDevProblemOffline.html#element-mesh-pastis)), so the MCP v2 SPIFFE must be added (section 5.6).
6. Hop 1 (the Helix UI through the ingress) and the route pods use to reach the internet (hops 6-7) were not verified in this project.

### 1.3 Network boundaries

A boundary is a point where the receiver cannot simply trust the sender and has to authenticate it. The flow crosses five kinds:

| Boundary | Hops (section 1.2) | Transport | How the receiver knows the caller |
| --- | --- | --- | --- |
| Internet to Pinterest | 1, 4 | HTTPS to the ingress | [SSO](#g-sso) session cookie from `auth.pinadmin.com`; the ingress writes [`X-Forwarded-User`](#g-x-forwarded-user) |
| Pod to pod, inside the mesh | 2, 3 | [mTLS](#g-mtls) between Envoy sidecars | Caller SPIFFE ID from the certificate, written to [XFCC](#g-xfcc) and checked by [Pastis](#g-pastis); the user is the [JWT](#g-jwt) `sub` |
| Ingress to pod | after 1 and 4 | mTLS | XFCC names the ingress, so the original caller is known only from `X-Forwarded-User` |
| Inside one pod | every inbound call | plain HTTP on `localhost` | Nothing: the app trusts whatever its own Envoy sends. That is why `token_broker` trusts XFCC and `X-Forwarded-User` without checking them |
| Pinterest to Snowflake, browser to Snowflake | 5, 6, 7 | HTTPS over the internet | Okta login (browser), client secret (broker), OAuth [access token](#g-access-token) (MCP) |

The "inside one pod" row is the root of the dev problem. In production only Envoy can reach the app, so trusting its headers is safe. On a devapp, anything that reaches the port gets the same trust.

### 1.4 Dev: where each piece runs today

Our devapps have no pods, no sidecars and no mesh. Each service is a plain process started with `bazel run` on its own EC2 machine. The only Envoy involved is the shared ingress, which reaches devapp ports through fixed hostnames. This is the current layout (trace 88-100); section 6.1 shows the original one that failed.

```mermaid
flowchart LR
    subgraph NET["Internet"]
        BR(["Your browser (laptop)"])
        SF["Snowflake dev account<br/>xla76048"]
    end
    subgraph PIN["Pinterest network"]
        ING{{"ingress-pinadmin<br/>mcp-* hosts: SSO, port 8000"}}
        HX["helix-test pod (PinCompute)<br/>spiffe://pin220.com/k8s/bex-tools/helix-test"]
        subgraph D2["devapp 2: EC2 machine, no sidecar"]
            M["MCP v2 process<br/>bazel run, :8000"]
        end
        subgraph D1["devapp 1: EC2 machine, no sidecar"]
            B["token_broker process<br/>bazel run, :8000<br/>SQLite, in-memory key"]
        end
    end

    BR -- "chat" --> HX
    HX -- "1: POST /mcp + user JWT<br/>via mcp-devrestricted-dmachicao-1" --> ING --> M
    M -- "2: GET /api/token + user JWT<br/>out to the public host, not the mesh" --> ING
    BR -- "3: connect link, callback<br/>via mcp-devrestricted-dmachicao, SSO" --> ING
    ING -- "XFCC = ingress SPIFFE<br/>X-Forwarded-User = user (after SSO)" --> B
    BR -- "Okta login + consent" --> SF
    B -- "code exchange, refresh" --> SF
    M -- "SQL + user access token" --> SF
```

Code references:

1. The devapp host-to-port routing (`mcp-*` to 8000 with SSO): section 8.1 and ([`oauth-mcp-server-devapp.md`](./OAuthFlowAndDevProblemOffline.html#element-runtime-devapps)).
2. `HX` SPIFFE: ([`oauth-mcp-server-pastis-traffic-pinconf.md` L31](./OAuthFlowAndDevProblemOffline.html#actor-ingress)).
3. `B`: SQLite in development ([`engine.py` L58-60](./OAuthFlowAndDevProblemOffline.html#class-broker-engine)); in-memory key ([`encryption.py` L46-72](./OAuthFlowAndDevProblemOffline.html#method-broker-get-dev-primitive)).
4. Hop 2 and the broker's view of it: the MCP's `token_broker.url` is the broker's SSO host and `client_id` is the ingress SPIFFE (trace 89); the headers the broker saw were probed in trace 88.
5. Devapps are EC2 machines: ([`devapp-tool-index.md` L94](./OAuthFlowAndDevProblemOffline.html#element-runtime-devapps)).

### 1.5 Production and dev side by side

| Aspect | Production | Dev (our devapps) |
| --- | --- | --- |
| Where it runs | [PinCompute](#g-pincompute) pods, one deployment per service | One EC2 machine per service, a `bazel run` process |
| Sidecars | Envoy (and Knox for the broker) in every pod | None |
| MCP to broker | Mesh mTLS; XFCC = MCP SPIFFE, the [partition](#g-partition) is per MCP | Public host through the ingress; XFCC = ingress SPIFFE, a partition shared by every ingress caller |
| Browser to broker | `token-broker.pinadmin.com` through the ingress with SSO | `mcp-devrestricted-dmachicao.pinterdev.com` through the ingress with SSO |
| Who may call `/api/*` | [Pastis](#g-pastis) policy by caller SPIFFE | Anyone who can reach the host with a valid JWT |
| Broker database | MySQL | SQLite file `/tmp/token_broker_dev.db` |
| Encryption key | [Tink](#g-tink) key from Knox | Created in memory at startup; a restart makes stored tokens unreadable |
| Secrets | Knox sidecar | Broker process environment, streamed in at start |
| Snowflake | Account `pinterestit`, production integration | Account `xla76048`, integration `SNOWFLAKE_MCP_V2_DMACHICAO_DEV` |
| Helix | `helix-prod` | `helix-test` |

Dev and production run the same code. Only the network path and the identities the broker sees differ, and that is why dev needs config changes rather than code changes (sections 6-7).

### 1.6 Inside each service

Modules of the two programs, and the one network call between them. Arrows inside a box are Python imports and function calls.

```mermaid
flowchart LR
    subgraph MCP["Snowflake MCP v2 process (snowflake_v2, ours)"]
        direction TB
        SRV["server.py<br/>PinterestMCP, streamable HTTP /mcp"]
        TOOLS["tools/<br/>execute_snowflake_query<br/>describe_table_view<br/>search_tables_views"]
        VAL["tools/validators.py<br/>SQL and allow-list checks"]
        AUD["tools/audit.py<br/>AUDIT: log lines"]
        TBC["clients/token_broker_client.py"]
        SFC["clients/snowflake_client.py"]
        CFG["runtime_config.py<br/>YAML from SNOWFLAKE_V2_CONFIG_PATH"]
        SRV --> TOOLS
        TOOLS --> VAL
        TOOLS --> AUD
        TOOLS --> SFC
        SFC --> TBC
        TBC -.-> CFG
        SFC -.-> CFG
    end
    subgraph BRK["token_broker process (it-swe)"]
        direction TB
        MAIN["main.py<br/>FastAPI app"]
        RAPI["api/routes/api.py<br/>/api/token, /api/status, /api/unlink"]
        ROA["api/routes/oauth.py<br/>/oauth/start, /oauth/callback"]
        AUTH["auth/jwt.py, auth/spiffe.py<br/>user from JWT, partition from XFCC"]
        SVC["services/oauth.py, services/encryption.py<br/>Snowflake calls, Tink or in-memory AEAD"]
        CRUD["crud/<br/>tokens, oauth states, audit log"]
        DBM["db/engine.py, db/models.py<br/>MySQL or SQLite"]
        BCFG["core/config.py<br/>mcp_servers.env.yaml, Knox secrets"]
        MAIN --> RAPI
        MAIN --> ROA
        RAPI --> AUTH
        RAPI --> SVC
        ROA --> SVC
        SVC --> CRUD
        CRUD --> DBM
        SVC -.-> BCFG
    end
    TBC ==>|"HTTP GET /api/token<br/>(network call, section 1.2 hop 3)"| RAPI
    SFC ==>|"snowflake-connector, OAuth token"| SNOW["Snowflake"]
    SVC ==>|"authorize URL, token endpoint"| SNOW
```

Code references:

1. `SRV`: ([`server.py` L21-41](./OAuthFlowAndDevProblemOffline.html#module-mcp-server)).
2. `TOOLS`, `VAL`, `AUD`: ([`execute_snowflake_query.py` L39-97](./OAuthFlowAndDevProblemOffline.html#method-mcp-execute-snowflake-query)), ([`validators.py` L110](./OAuthFlowAndDevProblemOffline.html#method-mcp-validate-query)), ([`audit.py`](./OAuthFlowAndDevProblemOffline.html#method-mcp-emit-audit-log)).
3. `TBC`, `SFC`, `CFG`: ([`token_broker_client.py` L49-86](./OAuthFlowAndDevProblemOffline.html#method-mcp-get-snowflake-access-token)), ([`snowflake_client.py` L33-67](./OAuthFlowAndDevProblemOffline.html#method-mcp-snowflake-init)), ([`runtime_config.py` L10, L177](./OAuthFlowAndDevProblemOffline.html#module-mcp-runtime-config)).
4. Broker modules: ([`main.py`](./OAuthFlowAndDevProblemOffline.html#module-broker-main)), ([`routes/api.py`](./OAuthFlowAndDevProblemOffline.html#method-broker-get-token)), ([`routes/oauth.py`](./OAuthFlowAndDevProblemOffline.html#method-broker-oauth-start)), ([`jwt.py`](./OAuthFlowAndDevProblemOffline.html#method-broker-get-current-user)), ([`spiffe.py`](./OAuthFlowAndDevProblemOffline.html#method-broker-get-caller-spiffe)), ([`services/oauth.py`](./OAuthFlowAndDevProblemOffline.html#module-broker-oauth-service)), ([`services/encryption.py`](./OAuthFlowAndDevProblemOffline.html#method-broker-encrypt-decrypt)), ([`mcp_token_crud.py`](./OAuthFlowAndDevProblemOffline.html#module-broker-token-crud)), ([`engine.py`](./OAuthFlowAndDevProblemOffline.html#class-broker-engine)), ([`config.py`](./OAuthFlowAndDevProblemOffline.html#module-broker-config)).

## 2. The one idea that explains everything: the token row key

The [broker](#g-token-broker) stores each Snowflake token in a row keyed by three values: `(user_id, client_id, mcp_server_id)`. Two different endpoints build that key from two different sources:

1. `/oauth/start` writes the key when you link. The `user_id` comes from the [`X-Forwarded-User`](#g-x-forwarded-user) header (or a `user_id` query parameter only if that header is absent) ([code](./OAuthFlowAndDevProblemOffline.html#method-broker-oauth-start)). The `client_id` comes from the link's query string, taken as is with no validation ([code](./OAuthFlowAndDevProblemOffline.html#method-broker-oauth-start)).
2. `/api/token` reads the key on every [tool call](#g-mcp). The `user_id` comes from the [JWT](#g-jwt) [`sub` claim](#g-claims) ([code](./OAuthFlowAndDevProblemOffline.html#method-broker-get-current-user)). The `client_id` comes from the leftmost `URI=` in the [`X-Forwarded-Client-Cert` (XFCC)](#g-xfcc) header, which the [mesh](#g-mesh) sets to the calling workload's [SPIFFE](#g-spiffe) ([code](./OAuthFlowAndDevProblemOffline.html#method-broker-parse-spiffe)).

If the written key and the read key are not equal, the [broker](#g-token-broker) answers `auth_required` forever, even though a token exists ([code](./OAuthFlowAndDevProblemOffline.html#method-broker-get-token)). In production both sides line up because [Envoy](#g-envoy) sets [`X-Forwarded-User`](#g-x-forwarded-user) to your [LDAP username](#g-ldap) after [SSO](#g-sso), and the [MCP](#g-mcp)'s connect link carries the [MCP](#g-mcp)'s own [SPIFFE](#g-spiffe). In our [devapps](#g-devapp) neither lines up.

```mermaid
flowchart TB
    subgraph W["Written by GET /oauth/start (browser)"]
        W1["user_id = X-Forwarded-User header<br/>(fallback: ?user_id= only if header absent)"]
        W2["client_id = ?client_id= from the link<br/>(not validated)"]
        W3["mcp_server_id = ?mcp_server_id=snowflake"]
    end
    subgraph R["Read by GET /api/token (MCP, server to server)"]
        R1["user_id = JWT sub claim"]
        R2["client_id = leftmost URI= in X-Forwarded-Client-Cert"]
        R3["mcp_server_id = ?mcp_server_id=snowflake"]
    end
    W1 -. "must be equal" .- R1
    W2 -. "must be equal" .- R2
    W3 -. "equal by config" .- R3
```

Code references:

1. `W1`: header first, query parameter as fallback ([`routes/oauth.py` L32-50](./OAuthFlowAndDevProblemOffline.html#method-broker-oauth-start)).
2. `W2`, `W3`: query parameters passed through unchanged ([`routes/oauth.py` L25-60](./OAuthFlowAndDevProblemOffline.html#method-broker-oauth-start)) and saved with the state ([`services/oauth.py` L72-81](./OAuthFlowAndDevProblemOffline.html#method-broker-build-authorize-url)).
3. `R1`: ([`jwt.py` L65-103](./OAuthFlowAndDevProblemOffline.html#method-broker-get-current-user)).
4. `R2`: ([`spiffe.py` L23-58](./OAuthFlowAndDevProblemOffline.html#method-broker-parse-spiffe)).
5. `R3` and the lookup: ([`api.py` L21-29](./OAuthFlowAndDevProblemOffline.html#method-broker-get-token)), ([`mcp_token_crud.py` L26-38](./OAuthFlowAndDevProblemOffline.html#method-broker-crud-get-token)).
6. The key itself, as a unique constraint: ([`models.py` L13-23](./OAuthFlowAndDevProblemOffline.html#model-mcp-token)).
7. The callback writes the row under the key saved at start time: ([`services/oauth.py` L147-157](./OAuthFlowAndDevProblemOffline.html#method-broker-exchange-code)).

## 3. Ideal workflow, high level (production)

Each participant is a separate service with its own process. The boxes show the network zone each one runs in (section 1.2), and each arrow says which network it crosses. This is what a non-developer does and sees. There are three phases: the first question fails with a link, the user links once, and every later question works. A [refresh](#g-refresh-token) happens silently when the 10-minute [access token](#g-access-token) expires ([code](./OAuthFlowAndDevProblemOffline.html#method-broker-get-token)). [Helix](#g-helix) today shows the link as plain text; [PromptHub](#g-prompthub) PR 1689 would render it as a Connect card, but the flow is the same.

```mermaid
sequenceDiagram
    autonumber
    box rgb(60, 60, 60) User device (internet)
        actor U as User (browser)
    end
    box rgb(40, 40, 90) Pinterest mesh, separate pods, mTLS between them
        participant H as Helix pod
        participant M as MCP v2 pod (ours)
        participant B as token_broker pod (it-swe, own DB)
    end
    box rgb(90, 60, 20) External SaaS (internet)
        participant S as Snowflake
    end

    rect rgb(110, 42, 42)
        Note over U,S: Phase A: first question, not linked yet
        U->>H: "Who am I in Snowflake?" (HTTPS via ingress, SSO)
        H->>M: tools/call execute_snowflake_query + user JWT (mesh mTLS)
        M->>B: GET /api/token + user JWT (mesh mTLS, XFCC = MCP)
        B-->>M: status auth_required
        M-->>H: tool result with auth_start_url
        H-->>U: "Connect Snowflake: {link}"
    end

    rect rgb(32, 52, 96)
        Note over U,S: Phase B: one-time link in the browser
        U->>B: open link (HTTPS via ingress, SSO identifies the user)
        B-->>U: redirect to Snowflake authorize
        U->>S: sign in (Okta) and consent (internet HTTPS)
        S-->>U: redirect to broker callback with code
        U->>B: GET /oauth/callback?code&state (via ingress)
        B->>S: exchange code for tokens (internet HTTPS + client secret)
        S-->>B: access token (10 min) and refresh token (7 days)
        B-->>U: "Connected, close this window"
    end

    rect rgb(28, 72, 46)
        Note over U,S: Phase C: retry and every later question
        U->>H: retry the question
        H->>M: tools/call + user JWT (mesh mTLS)
        M->>B: GET /api/token + user JWT (mesh mTLS)
        B-->>M: status ok + access token
        M->>S: connect as the user, run SQL (internet HTTPS)
        S-->>M: rows + query id
        M-->>H: results
        H-->>U: answer computed with the user's role grants
    end
```

Code references:

1. Steps 2 and 16, Helix forwards the user JWT: ([prompthub `server.py` L139-147](./OAuthFlowAndDevProblemOffline.html#actor-helix)).
2. Steps 3 and 17, the MCP asks the broker: ([`token_broker_client.py` L49-64](./OAuthFlowAndDevProblemOffline.html#method-mcp-get-snowflake-access-token)).
3. Step 4, `auth_required`: ([`api.py` L31-35](./OAuthFlowAndDevProblemOffline.html#method-broker-get-token)).
4. Steps 5-6, the MCP builds the link and returns it as `auth_start_url`: ([`token_broker_client.py` L74-85](./OAuthFlowAndDevProblemOffline.html#method-mcp-get-snowflake-access-token)), ([`execute_snowflake_query.py` L63-69](./OAuthFlowAndDevProblemOffline.html#method-mcp-execute-snowflake-query)).
5. Steps 7-8, `/oauth/start` and the redirect to Snowflake: ([`routes/oauth.py` L25-61](./OAuthFlowAndDevProblemOffline.html#method-broker-oauth-start)), ([`services/oauth.py` L48-98](./OAuthFlowAndDevProblemOffline.html#method-broker-build-authorize-url)).
6. Steps 11-13, callback and token exchange: ([`routes/oauth.py` L75-103](./OAuthFlowAndDevProblemOffline.html#method-broker-oauth-callback)), ([`services/oauth.py` L101-174](./OAuthFlowAndDevProblemOffline.html#method-broker-exchange-code)).
7. Step 14, the "Connected" page: ([`success.html`](./OAuthFlowAndDevProblemOffline.html#method-broker-oauth-callback)).
8. Step 18, the token returned: ([`api.py` L73-95](./OAuthFlowAndDevProblemOffline.html#method-broker-get-token)).
9. Steps 19-20, connect as the user and run SQL: ([`snowflake_client.py` L33-67](./OAuthFlowAndDevProblemOffline.html#method-mcp-snowflake-init)).
10. Step 21, the tool result: ([`execute_snowflake_query.py` L83-97](./OAuthFlowAndDevProblemOffline.html#method-mcp-execute-snowflake-query)).

## 4. Ideal workflow, wire level

Each phase below shows the actual HTTP requests, the headers that matter, and the JSON bodies. [Envoy](#g-envoy) appears explicitly because it adds or replaces the identity headers the [broker](#g-token-broker) trusts. [Helix](#g-helix)'s own call path to the [MCP](#g-mcp) is simplified to "Envoy"; the header that matters there is [`Authorization`](#g-bearer).

### 4.1 Phase A: the tool call hits "not linked"

The [MCP](#g-mcp) forwards only the user's [`Authorization`](#g-bearer) header to the [broker](#g-token-broker); it sends no user or client parameter ([code](./OAuthFlowAndDevProblemOffline.html#method-mcp-get-snowflake-access-token)). The [broker](#g-token-broker) learns the user from the [JWT](#g-jwt) and the caller from [mTLS](#g-mtls). The [JWT](#g-jwt) signature is checked against `auth.pinadmin.com`'s public key; the [audience](#g-claims) is not checked (`verify_aud: False`) ([code](./OAuthFlowAndDevProblemOffline.html#method-broker-jwt-decode)). The `auth_required` response carries no link, so the [MCP](#g-mcp) builds it itself from its config (`token_broker.url` and `token_broker.client_id`) ([code](./OAuthFlowAndDevProblemOffline.html#method-mcp-get-snowflake-access-token)). That config value is the only thing that makes the link's `client_id` match the [partition](#g-partition) `/api/token` sees.

```mermaid
sequenceDiagram
    autonumber
    participant H as Helix
    participant EM as Envoy (MCP side)
    participant M as MCP v2
    participant EB as Envoy (broker sidecar)
    participant B as token_broker
    participant DB as Token store

    H->>EM: POST /mcp<br/>Authorization: Bearer {jwt}<br/>{"jsonrpc":"2.0","method":"tools/call",<br/>"params":{"name":"execute_snowflake_query",<br/>"arguments":{"query":"SELECT CURRENT_USER()"}}}
    EM->>M: same request (JWT verified by mesh)
    Note over M: validate_query() passes<br/>SnowflakeClient() needs a token first
    M->>EB: GET /api/token?mcp_server_id=snowflake<br/>Authorization: Bearer {jwt}<br/>(mTLS from the MCP workload)
    EB->>B: + X-Forwarded-Client-Cert:<br/>By=spiffe://.../token-broker#59;URI={MCP SPIFFE}
    Note over B: user_id = JWT sub = "dmachicao"<br/>client_id = leftmost URI = {MCP SPIFFE}
    B->>DB: SELECT mcp_tokens WHERE (dmachicao, {MCP SPIFFE}, snowflake)
    DB-->>B: no row
    B-->>M: 200 {"status":"auth_required",<br/>"message":"User has not linked their account"}
    Note over M: connect_url = token_broker.url + /oauth/start?<br/>client_id={config client_id}&mcp_server_id=snowflake
    M-->>H: tool result {"results":null,"status":"auth_required",<br/>"message":"User has not linked their account",<br/>"auth_start_url":"https://{broker host}/oauth/start?client_id=...&mcp_server_id=snowflake"}
```

Code references:

1. Step 1, Helix adds the JWT header: ([prompthub `server.py` L139-147](./OAuthFlowAndDevProblemOffline.html#actor-helix)).
2. Step 2, "JWT verified by mesh": the recorded repository search found no Envoy or [Pastis](#g-pastis) config for `mcp-server-snowflake-v2` yet. The server declares its future policy URL at ([`server.py` L30](./OAuthFlowAndDevProblemOffline.html#module-mcp-server)).
3. Note after step 2: `validate_query()` ([`validators.py` L110-128](./OAuthFlowAndDevProblemOffline.html#method-mcp-validate-query)), called from ([`execute_snowflake_query.py` L49-61](./OAuthFlowAndDevProblemOffline.html#method-mcp-execute-snowflake-query)); `SnowflakeClient()` fetches the token before connecting ([`snowflake_client.py` L33-41](./OAuthFlowAndDevProblemOffline.html#method-mcp-snowflake-init)).
4. Step 3: request headers come from ([`header_utils.py` L30-36](./OAuthFlowAndDevProblemOffline.html#method-mcp-authorization)); the broker request is built in ([`token_broker_client.py` L39-64](./OAuthFlowAndDevProblemOffline.html#method-mcp-get-snowflake-access-token)).
5. Step 4 and the note after it: ([`spiffe.py` L23-58](./OAuthFlowAndDevProblemOffline.html#method-broker-parse-spiffe)), ([`jwt.py` L65-103](./OAuthFlowAndDevProblemOffline.html#method-broker-get-current-user)), both wired in ([`api.py` L21-26](./OAuthFlowAndDevProblemOffline.html#method-broker-get-token)).
6. Steps 5-6: ([`mcp_token_crud.py` L26-38](./OAuthFlowAndDevProblemOffline.html#method-broker-crud-get-token)).
7. Step 7: ([`api.py` L31-35](./OAuthFlowAndDevProblemOffline.html#method-broker-get-token)).
8. Note `connect_url` and step 8: ([`token_broker_client.py` L74-85](./OAuthFlowAndDevProblemOffline.html#method-mcp-get-snowflake-access-token)), `client_id` from ([`runtime.dev.yaml` L216-220](./OAuthFlowAndDevProblemOffline.html#module-mcp-runtime-config)), returned by ([`execute_snowflake_query.py` L63-69](./OAuthFlowAndDevProblemOffline.html#method-mcp-execute-snowflake-query)).

### 4.2 Phase B: the browser link, SSO, PKCE, and the callback

This is the part that has to identify the human, and it can only do that through the browser's [SSO](#g-sso) session. `/oauth/start` has no [JWT](#g-jwt) and no [mTLS](#g-mtls) client; it trusts [`X-Forwarded-User`](#g-x-forwarded-user) ([code](./OAuthFlowAndDevProblemOffline.html#method-broker-oauth-start)), which only [Envoy](#g-envoy)'s [`internal_oauth` filter](#g-internal-oauth) should set ([config](./OAuthFlowAndDevProblemOffline.html#element-ingress-internal-oauth)). The [broker](#g-token-broker) saves the key and a [PKCE](#g-pkce) verifier in an [`oauth_states`](#g-state) row that lives 600 seconds ([code](./OAuthFlowAndDevProblemOffline.html#method-broker-build-authorize-url)), then [redirects](#g-http-redirect) to Snowflake. The [callback](#g-redirect-uri) carries no identity at all: it looks up the [state](#g-state) row and stores the token under the key saved at start time ([code](./OAuthFlowAndDevProblemOffline.html#method-broker-exchange-code)). That is why the start request is the only moment that decides whose token it is.

```mermaid
sequenceDiagram
    autonumber
    actor U as Browser
    participant EI as Ingress Envoy<br/>(token-broker.pinadmin.com)
    participant A as auth.pinadmin.com
    participant B as token_broker
    participant DB as Token store
    participant S as Snowflake OAuth<br/>(account URL)

    U->>EI: GET /oauth/start?client_id={MCP SPIFFE}&mcp_server_id=snowflake
    EI-->>U: 302 Location: https://auth.pinadmin.com/oauth/authorize/?...<br/>(no SSO session cookie yet)
    U->>A: SSO (Okta)
    A-->>U: 302 back to /oauth/start with session cookie
    U->>EI: GET /oauth/start?client_id={MCP SPIFFE}&mcp_server_id=snowflake
    EI->>B: + X-Forwarded-User: dmachicao
    Note over B: user_id = X-Forwarded-User = dmachicao<br/>client_id = query = {MCP SPIFFE}<br/>state = random, code_verifier = random<br/>code_challenge = BASE64URL(SHA256(code_verifier))
    B->>DB: INSERT oauth_states (state, dmachicao, {MCP SPIFFE},<br/>snowflake, code_verifier, redirect_uri, expires in 600 s)
    B-->>U: 307 Location: https://{account}.snowflakecomputing.com/oauth/authorize?<br/>response_type=code&client_id={sf_client_id}<br/>&redirect_uri=https://token-broker.pinadmin.com/oauth/callback<br/>&state={state}&scope=refresh_token<br/>&code_challenge={challenge}&code_challenge_method=S256
    U->>S: GET /oauth/authorize?...
    S-->>U: Okta sign-in, then consent page for the integration
    U->>S: approve
    S-->>U: 302 Location: https://token-broker.pinadmin.com/oauth/callback?code={code}&state={state}
    U->>EI: GET /oauth/callback?code={code}&state={state}
    EI->>B: forwarded (identity headers ignored here)
    B->>DB: consume oauth_states WHERE state={state}<br/>returns (dmachicao, {MCP SPIFFE}, snowflake, code_verifier)
    B->>S: POST /oauth/token-request<br/>Content-Type: application/x-www-form-urlencoded<br/>grant_type=authorization_code&code={code}<br/>&redirect_uri=https://token-broker.pinadmin.com/oauth/callback<br/>&client_id={sf_client_id}&client_secret={sf_client_secret}<br/>&code_verifier={code_verifier}
    S-->>B: 200 {"access_token":"{access_token}","refresh_token":"{refresh_token}",<br/>"token_type":"Bearer","expires_in":600, ...}
    Note over B: encrypt with Tink (Knox key per provider)
    B->>DB: UPSERT mcp_tokens (dmachicao, {MCP SPIFFE}, snowflake) = encrypted tokens
    B->>DB: INSERT audit_logs action=link
    B-->>U: 200 success.html ("Connected, you can close this window")
```

Code references:

1. Steps 1-5, SSO on the broker host: ([pinconf `token-broker/prod/main/http.yaml` L8-11](./OAuthFlowAndDevProblemOffline.html#element-ingress-internal-oauth)); [Pastis](#g-pastis) lets any caller reach `/oauth/*` ([pinconf `token-broker/pastis/base.rego` L29-32](./OAuthFlowAndDevProblemOffline.html#element-mesh-pastis)).
2. Step 6 and the note after it: ([`routes/oauth.py` L32-50](./OAuthFlowAndDevProblemOffline.html#method-broker-oauth-start)); state, verifier, and challenge ([`services/oauth.py` L20-30](./OAuthFlowAndDevProblemOffline.html#method-broker-generate-state)), ([`services/oauth.py` L61-67](./OAuthFlowAndDevProblemOffline.html#method-broker-build-authorize-url)).
3. Step 7: ([`services/oauth.py` L69-81](./OAuthFlowAndDevProblemOffline.html#method-broker-build-authorize-url)), ([`oauth_state_crud.py` L29-50](./OAuthFlowAndDevProblemOffline.html#method-broker-create-state)) with the verifier encrypted ([`oauth_state_crud.py` L13-16](./OAuthFlowAndDevProblemOffline.html#method-broker-create-state)), TTL ([`settings.py` L53](./OAuthFlowAndDevProblemOffline.html#module-broker-config)).
4. Step 8: ([`services/oauth.py` L83-98](./OAuthFlowAndDevProblemOffline.html#method-broker-build-authorize-url)); the 307 comes from ([`routes/oauth.py` L61](./OAuthFlowAndDevProblemOffline.html#method-broker-oauth-start)).
5. Steps 13-14, SSO is off for the callback path ([pinconf `http.yaml` L15-18](./OAuthFlowAndDevProblemOffline.html#element-ingress-internal-oauth)); handler ([`routes/oauth.py` L75-103](./OAuthFlowAndDevProblemOffline.html#method-broker-oauth-callback)).
6. Step 15, single-use state: ([`oauth_state_crud.py` L52-75](./OAuthFlowAndDevProblemOffline.html#method-broker-get-validate-state)).
7. Steps 16-17: ([`services/oauth.py` L115-145](./OAuthFlowAndDevProblemOffline.html#method-broker-exchange-code)).
8. Note (Tink, Knox) and step 18: ([`mcp_token_crud.py` L40-101](./OAuthFlowAndDevProblemOffline.html#method-broker-store-token)), ([`encryption.py` L58-90](./OAuthFlowAndDevProblemOffline.html#method-broker-get-aead-primitive)).
9. Step 19: ([`services/oauth.py` L159-165](./OAuthFlowAndDevProblemOffline.html#method-broker-exchange-code)).
10. Step 20: ([`routes/oauth.py` L99-103](./OAuthFlowAndDevProblemOffline.html#method-broker-oauth-callback)), ([`success.html`](./OAuthFlowAndDevProblemOffline.html#method-broker-oauth-callback)).

### 4.3 Phase C: the retry runs SQL as the user

Now the read key matches the written key. The [broker](#g-token-broker) decrypts the [access token](#g-access-token) and returns it with [`Cache-Control: no-store`](#g-no-store) ([code](./OAuthFlowAndDevProblemOffline.html#method-broker-get-token)). The [MCP](#g-mcp) opens a Snowflake connection with `authenticator=oauth` ([code](./OAuthFlowAndDevProblemOffline.html#method-mcp-snowflake-init)), so Snowflake sees the human's user, [default role](#g-roles), and [implicit secondary roles](#g-secondary-roles) (the [integration](#g-security-integration) uses `OAUTH_USE_SECONDARY_ROLES = IMPLICIT`). The Snowflake connector's login and query calls are shown simplified; the [MCP](#g-mcp) never logs result values, only one `AUDIT:` line with the Snowflake query and session IDs ([code](./OAuthFlowAndDevProblemOffline.html#method-mcp-emit-audit-log)).

```mermaid
sequenceDiagram
    autonumber
    participant H as Helix
    participant M as MCP v2
    participant EB as Envoy (broker sidecar)
    participant B as token_broker
    participant DB as Token store
    participant S as Snowflake

    H->>M: POST /mcp tools/call (Authorization: Bearer {jwt})
    M->>EB: GET /api/token?mcp_server_id=snowflake<br/>Authorization: Bearer {jwt}
    EB->>B: + X-Forwarded-Client-Cert: ...URI={MCP SPIFFE}
    B->>DB: SELECT (dmachicao, {MCP SPIFFE}, snowflake)
    DB-->>B: row, not expired
    B->>DB: INSERT audit_logs action=retrieve
    B-->>M: 200 Cache-Control: no-store<br/>{"status":"ok","access_token":"{access_token}","token_type":"Bearer"}
    M->>S: connector login (simplified)<br/>POST /session/v1/login-request<br/>{"data":{"AUTHENTICATOR":"OAUTH","TOKEN":"{access_token}",<br/>"ACCOUNT_NAME":"{account}"}}<br/>database, schema from runtime config
    S-->>M: session for user DMACHICAO, default role + implicit secondary roles
    M->>S: POST /queries/v1/query-request {"sqlText":"SELECT CURRENT_USER(), ..."}
    S-->>M: rows, queryId, sessionId
    Note over M: log AUDIT: {"event":"execute_snowflake_query","status":"success",<br/>"snowflake_query_id":"...","snowflake_session_id":"...","row_count":1}
    M-->>H: {"results":[...],"row_count":1,"status":"success","query_id":"..."}
```

Code references:

1. Step 2: ([`token_broker_client.py` L49-73](./OAuthFlowAndDevProblemOffline.html#method-mcp-get-snowflake-access-token)).
2. Step 3: ([`spiffe.py` L23-58](./OAuthFlowAndDevProblemOffline.html#method-broker-parse-spiffe)).
3. Steps 4-5, lookup and expiry check: ([`api.py` L29-37](./OAuthFlowAndDevProblemOffline.html#method-broker-get-token)), ([`mcp_token_crud.py` L121-125](./OAuthFlowAndDevProblemOffline.html#method-broker-is-expired)).
4. Step 6: ([`api.py` L73-86](./OAuthFlowAndDevProblemOffline.html#method-broker-get-token)).
5. Step 7: ([`api.py` L88-95](./OAuthFlowAndDevProblemOffline.html#method-broker-get-token)), headers at ([`api.py` L18](./OAuthFlowAndDevProblemOffline.html#method-broker-get-token)).
6. Steps 8-9: ([`snowflake_client.py` L33-54](./OAuthFlowAndDevProblemOffline.html#method-mcp-snowflake-init)).
7. Steps 10-11: ([`snowflake_client.py` L56-67](./OAuthFlowAndDevProblemOffline.html#method-mcp-execute-query-sync)).
8. Note `AUDIT:`: ([`audit.py` L8-9](./OAuthFlowAndDevProblemOffline.html#method-mcp-emit-audit-log)), ([`execute_snowflake_query.py` L83-90](./OAuthFlowAndDevProblemOffline.html#method-mcp-execute-snowflake-query)).
9. Step 12: ([`execute_snowflake_query.py` L91-97](./OAuthFlowAndDevProblemOffline.html#method-mcp-execute-snowflake-query)).

### 4.4 Silent refresh, re-auth, and unlink

The [access token](#g-access-token) lives 600 seconds and the [refresh token](#g-refresh-token) 7 days ([integration](#g-security-integration) settings). The [broker](#g-token-broker) [refreshes](#g-refresh-token) only when `/api/token` finds an expired row, so the first call after a quiet period pays the [refresh](#g-refresh-token) time ([code](./OAuthFlowAndDevProblemOffline.html#method-broker-get-token)). In dev that is about 15 seconds, because [Knox](#g-knox) has no [Tink](#g-tink) key for the Snowflake provider and each encryption step waits 5 seconds before falling back to an in-memory key ([code](./OAuthFlowAndDevProblemOffline.html#method-broker-get-aead-primitive)); the [MCP](#g-mcp) timeout is 30 seconds for that reason ([config](./OAuthFlowAndDevProblemOffline.html#module-mcp-runtime-config)). [Unlink](#g-revocation) [revokes](#g-revocation) at Snowflake first and deletes the row only on success ([code](./OAuthFlowAndDevProblemOffline.html#method-broker-unlink)).

```mermaid
sequenceDiagram
    autonumber
    participant M as MCP v2
    participant B as token_broker
    participant DB as Token store
    participant S as Snowflake OAuth

    M->>B: GET /api/token?mcp_server_id=snowflake (JWT + XFCC)
    B->>DB: row found, expires_at in the past
    alt refresh_token present
        B->>S: POST /oauth/token-request<br/>grant_type=refresh_token&refresh_token={refresh_token}<br/>&client_id={sf_client_id}&client_secret={sf_client_secret}
        alt Snowflake accepts
            S-->>B: 200 {"access_token":"...","expires_in":600, ...}
            B->>DB: UPSERT row, INSERT audit_logs action=refresh
            B-->>M: {"status":"ok","access_token":"..."}
        else refresh rejected (revoked, 7 days passed, key lost)
            S-->>B: 400
            B-->>M: {"status":"reauth_required",<br/>"message":"Token expired and refresh failed. Please re-authenticate."}
            Note over M: same auth_start_url as Phase A, user relinks
        end
    else no refresh_token
        B-->>M: {"status":"reauth_required","message":"Token expired. No refresh token available."}
    end

    Note over M,S: Unlink (user action, own JWT)
    M->>B: POST /api/unlink?client_id={partition}&mcp_server_id=snowflake (Authorization: Bearer {jwt})
    B->>S: POST /oauth/revoke (token)
    S-->>B: 200
    B->>DB: DELETE row, INSERT audit_logs action=unlink
```

Code references:

1. Steps 1-2: ([`api.py` L21-37](./OAuthFlowAndDevProblemOffline.html#method-broker-get-token)), ([`mcp_token_crud.py` L121-125](./OAuthFlowAndDevProblemOffline.html#method-broker-is-expired)).
2. Steps 3-5: ([`services/oauth.py` L177-238](./OAuthFlowAndDevProblemOffline.html#method-broker-refresh-access-token)), audit at ([`api.py` L46-52](./OAuthFlowAndDevProblemOffline.html#method-broker-get-token)).
3. Step 6: ([`api.py` L53-60](./OAuthFlowAndDevProblemOffline.html#method-broker-get-token)).
4. Steps 7-8: ([`api.py` L61-66](./OAuthFlowAndDevProblemOffline.html#method-broker-get-token)).
5. Step 9: ([`api.py` L67-71](./OAuthFlowAndDevProblemOffline.html#method-broker-get-token)).
6. Note "same `auth_start_url`": the MCP treats `auth_required` and `reauth_required` the same ([`token_broker_client.py` L11](./OAuthFlowAndDevProblemOffline.html#method-mcp-get-snowflake-access-token)), ([`token_broker_client.py` L74-85](./OAuthFlowAndDevProblemOffline.html#method-mcp-get-snowflake-access-token)).
7. Steps 10-13: ([`api.py` L128-170](./OAuthFlowAndDevProblemOffline.html#method-broker-unlink)), ([`services/oauth.py` L241-292](./OAuthFlowAndDevProblemOffline.html#method-broker-revoke-token)), ([`mcp_token_crud.py` L127-137](./OAuthFlowAndDevProblemOffline.html#method-broker-delete-token)).

The link status as the [broker](#g-token-broker) sees it:

```mermaid
stateDiagram-v2
    [*] --> not_linked
    not_linked --> linked: callback stores token (Phase B)
    linked --> expired: 600 s pass
    expired --> linked: refresh ok on next /api/token
    expired --> reauth_required: refresh fails or no refresh token
    reauth_required --> linked: user opens the link again
    linked --> not_linked: unlink (revoke, then delete)
    linked --> undecryptable: dev broker restart (in-memory key lost)
    undecryptable --> linked: user relinks (row overwritten)
```

Code references:

1. `not_linked` to `linked`: ([`services/oauth.py` L147-165](./OAuthFlowAndDevProblemOffline.html#method-broker-exchange-code)).
2. `linked` to `expired`: counted 300 seconds before `expires_at` ([`mcp_token_crud.py` L121-125](./OAuthFlowAndDevProblemOffline.html#method-broker-is-expired)).
3. `expired` to `linked`: ([`api.py` L38-60](./OAuthFlowAndDevProblemOffline.html#method-broker-get-token)).
4. `expired` to `reauth_required`: ([`api.py` L61-71](./OAuthFlowAndDevProblemOffline.html#method-broker-get-token)).
5. `reauth_required` to `linked`: same code as item 1.
6. `linked` to `not_linked`: ([`api.py` L128-170](./OAuthFlowAndDevProblemOffline.html#method-broker-unlink)).
7. `linked` to `undecryptable`: the dev key exists only in process memory ([`encryption.py` L46-55](./OAuthFlowAndDevProblemOffline.html#method-broker-get-dev-primitive)), used when Knox fails ([`encryption.py` L70-72](./OAuthFlowAndDevProblemOffline.html#method-broker-get-aead-primitive)).
8. `undecryptable` to `linked`: `store_token` overwrites the existing row ([`mcp_token_crud.py` L40-73](./OAuthFlowAndDevProblemOffline.html#method-broker-store-token)).

## 5. The tricky parts that exist even in production

These are not dev bugs. They are properties of the design that make it easy to get wrong, and they explain why the dev setup breaks in the specific way it does.

### 5.1 Two identities, two trust sources

The user is trusted from two different places: the [JWT](#g-jwt) [`sub`](#g-claims) at `/api/token` and [`X-Forwarded-User`](#g-x-forwarded-user) at `/oauth/start`. The [partition](#g-partition) is also trusted from two places: [mTLS](#g-mtls) ([XFCC](#g-xfcc)) at `/api/token` and a plain query parameter at `/oauth/start`. Production works only because [Envoy](#g-envoy) makes the header equal to the [JWT](#g-jwt) [subject](#g-claims) and because the [MCP](#g-mcp)'s config puts its own [SPIFFE](#g-spiffe) in the link. Nothing in the [broker](#g-token-broker) checks that the two sides agree; a mismatch is silent and shows up only as an endless `auth_required`.

```mermaid
flowchart LR
    subgraph P["Production"]
        PJ["JWT sub<br/>dmachicao"] --- PX["X-Forwarded-User<br/>dmachicao (Envoy SSO)"]
        PC["XFCC URI<br/>MCP SPIFFE (mesh mTLS)"] --- PL["link ?client_id=<br/>MCP SPIFFE (MCP config)"]
    end
    subgraph D["Our devapps today"]
        DJ["JWT sub<br/>dmachicao"] -. "differs" .- DX["X-Forwarded-User<br/>ingress SPIFFE"]
        DC["XFCC URI<br/>ingress SPIFFE"] -. "differs" .- DL["link ?client_id=<br/>devapp SPIFFE"]
    end
```

Code references:

1. `PJ`, `DJ`: ([`jwt.py` L96-103](./OAuthFlowAndDevProblemOffline.html#method-broker-get-current-user)).
2. `PX`, `DX`: the broker takes the header as is ([`routes/oauth.py` L32-45](./OAuthFlowAndDevProblemOffline.html#method-broker-oauth-start)); in production Envoy sets it after SSO ([pinconf `token-broker/prod/main/http.yaml` L8-11](./OAuthFlowAndDevProblemOffline.html#element-ingress-internal-oauth)).
3. `PC`, `DC`: ([`spiffe.py` L23-58](./OAuthFlowAndDevProblemOffline.html#method-broker-parse-spiffe)).
4. `PL`, `DL`: the link uses `token_broker.client_id` from config ([`runtime.dev.yaml` L218](./OAuthFlowAndDevProblemOffline.html#module-mcp-runtime-config)), ([`token_broker_client.py` L74-85](./OAuthFlowAndDevProblemOffline.html#method-mcp-get-snowflake-access-token)).
5. No check that the two sides agree: `/oauth/start` never reads a JWT or XFCC ([`routes/oauth.py` L25-61](./OAuthFlowAndDevProblemOffline.html#method-broker-oauth-start)).

### 5.2 Why `/oauth/start` must identify the browser user through SSO

It is tempting to "fix" dev by passing `user_id` in the link. That would be unsafe in production. The link is a plain URL; if the [broker](#g-token-broker) trusted a user named in it, an attacker could send a victim a link naming the victim, or complete a link naming someone else with their own Snowflake login. The victim's later [Helix](#g-helix) queries would then run with the attacker's Snowflake identity. This is the classic [OAuth](#g-oauth) [login-CSRF and token-planting](#g-login-csrf) problem. The [broker](#g-token-broker) avoids it in production because [Envoy](#g-envoy) overwrites [`X-Forwarded-User`](#g-x-forwarded-user) with the [SSO](#g-sso) user, so a `user_id` in the URL is ignored ([code](./OAuthFlowAndDevProblemOffline.html#method-broker-oauth-start)).

```mermaid
sequenceDiagram
    autonumber
    actor X as Attacker
    actor V as Victim
    participant B as Broker that trusted ?user_id=
    participant S as Snowflake

    X->>B: GET /oauth/start?user_id=victim&client_id={MCP SPIFFE}
    B-->>X: redirect to Snowflake authorize
    X->>S: signs in as ATTACKER and consents
    S-->>B: callback with attacker's code
    Note over B: stores ATTACKER's Snowflake token<br/>under key (victim, MCP SPIFFE, snowflake)
    V->>B: later Helix query as victim (JWT sub = victim)
    B-->>V: returns ATTACKER's token
    Note over V,S: victim's queries run as the attacker in Snowflake<br/>(data exfiltration or poisoning)
```

Code references:

1. Step 1: the real broker uses `?user_id=` only when `X-Forwarded-User` is absent ([`routes/oauth.py` L45](./OAuthFlowAndDevProblemOffline.html#method-broker-oauth-start)); on a host where nothing sets the header, it behaves like this diagram.
2. Step 4 and the note after it: the callback stores the token under the key saved in the state row, whoever signed in at Snowflake ([`services/oauth.py` L107-157](./OAuthFlowAndDevProblemOffline.html#method-broker-exchange-code)).
3. Steps 5-6: ([`api.py` L21-95](./OAuthFlowAndDevProblemOffline.html#method-broker-get-token)).

### 5.3 State, PKCE, and the confidential client

The [integration](#g-security-integration) is a [confidential client](#g-confidential-client) with enforced [PKCE](#g-pkce) (`OAUTH_ENFORCE_PKCE = TRUE`), so the [token exchange](#g-token-exchange) needs three things only the [broker](#g-token-broker) has: the [client secret](#g-confidential-client), the [`code_verifier`](#g-pkce) saved in the [state](#g-state) row, and the exact registered [`redirect_uri`](#g-redirect-uri) ([code](./OAuthFlowAndDevProblemOffline.html#method-broker-exchange-code)). The [state](#g-state) row expires 600 seconds after the user opens the link, not when the link is created: the link itself is a static URL and never expires ([code](./OAuthFlowAndDevProblemOffline.html#method-broker-build-authorize-url)). The [callback](#g-redirect-uri) must reach the same [broker](#g-token-broker) process and database that created the [state](#g-state) row.

### 5.4 Encryption keys and restarts

Tokens are encrypted with a [Tink](#g-tink) key from [Knox](#g-knox). The [devapp](#g-devapp) has no [Knox](#g-knox) key for the Snowflake provider, so the dev [broker](#g-token-broker) generates an in-memory key at startup ([code](./OAuthFlowAndDevProblemOffline.html#method-broker-get-dev-primitive)). Restarting the dev [broker](#g-token-broker) makes every stored row unreadable: `/api/token` returns `reauth_required`, and `/api/unlink` returns 502 because [revocation](#g-revocation) needs to decrypt first ([code](./OAuthFlowAndDevProblemOffline.html#method-broker-unlink)). A row stored under the [ingress](#g-ingress) [SPIFFE](#g-spiffe) as the user cannot be [unlinked](#g-revocation) with a user [JWT](#g-jwt) at all, because unlink takes the user from the [JWT](#g-jwt) ([code](./OAuthFlowAndDevProblemOffline.html#method-broker-unlink)).

### 5.5 Scope and roles

The dev [broker](#g-token-broker) requests [scope](#g-scope) `refresh_token` only, and the production entry in PR 197114 does the same ([config](./OAuthFlowAndDevProblemOffline.html#class-broker-server-config)). Snowflake then gives the session the user's [default role](#g-roles) plus [implicit secondary roles](#g-secondary-roles). The design wants `session:role-any refresh_token`, which requires [`OAUTH_ANY_ROLE_MODE`](#g-scope) on the [integration](#g-security-integration). `session:role:all`, used by the previous engineer, is not a documented Snowflake [scope](#g-scope).

### 5.6 One broker URL cannot serve both hops in production

[snowflake_v2](#g-mcp) has one `token_broker.url` for the server call and for the browser link ([code](./OAuthFlowAndDevProblemOffline.html#method-mcp-get-snowflake-access-token)). The two hops need different routes in production. The server call must stay inside the [mesh](#g-mesh) so [XFCC](#g-xfcc) carries the [MCP](#g-mcp) [SPIFFE](#g-spiffe). The browser link must use the public [SSO](#g-sso) host so [`X-Forwarded-User`](#g-x-forwarded-user) is the user. If the [MCP](#g-mcp) [pod](#g-pod) called the public host, the [broker](#g-token-broker) would [partition](#g-partition) by the [ingress](#g-ingress) [SPIFFE](#g-spiffe), which every [ingress](#g-ingress) caller shares. This is likely a small [snowflake_v2](#g-mcp) change (a separate public link URL) before production.

```mermaid
flowchart LR
    M["MCP pod"] -- "GET /api/token<br/>must go over the mesh" --> MB["token-broker mesh address<br/>XFCC = MCP SPIFFE"]
    M -- "builds link" --> L["https://token-broker.pinadmin.com/oauth/start?...<br/>browser, SSO, X-Forwarded-User = user"]
    M -. "if it called the public host" .-> BAD["XFCC = ingress SPIFFE<br/>shared partition for all ingress callers"]
```

Code references:

1. `M`: one URL for both hops ([`runtime_config.py` L33-38](./OAuthFlowAndDevProblemOffline.html#module-mcp-runtime-config)), used by the server call ([`token_broker_client.py` L25-58](./OAuthFlowAndDevProblemOffline.html#method-mcp-get-snowflake-access-token)) and by the link ([`token_broker_client.py` L74-85](./OAuthFlowAndDevProblemOffline.html#method-mcp-get-snowflake-access-token)).
2. `MB`: the broker's [Pastis](#g-pastis) policy allows `/api/*` only from the helix-slackbot SPIFFE IDs today, so the MCP SPIFFE must be added before production ([pinconf `token-broker/pastis/base.rego` L8-22](./OAuthFlowAndDevProblemOffline.html#element-mesh-pastis)).
3. `L`: SSO on the public host ([pinconf `token-broker/prod/main/http.yaml` L8-11](./OAuthFlowAndDevProblemOffline.html#element-ingress-internal-oauth)).
4. `BAD`: the partition is whatever the leftmost `URI=` says ([`spiffe.py` L23-35](./OAuthFlowAndDevProblemOffline.html#method-broker-parse-spiffe)).

## 6. The dev problem

### 6.1 Our dev topology

[Helix-test](#g-helix) reaches the [MCP](#g-mcp) on [devapp](#g-devapp) 2 through its `mcp-*` host, which requires Pinterest [SSO](#g-sso) (anonymous requests get a [302](#g-http-redirect) to `auth.pinadmin.com`) and accepts [Helix](#g-helix)'s forwarded [JWT](#g-jwt). The [broker](#g-token-broker) on [devapp](#g-devapp) 1 sits behind the `www-*` host, which has no [SSO](#g-sso) step: anonymous requests are passed through. The [MCP](#g-mcp) reaches the [broker](#g-token-broker) through that same public `www-*` host, not through the [mesh](#g-mesh). Every request that reaches the [broker](#g-token-broker) therefore comes from the [ingress](#g-ingress), so the [broker](#g-token-broker) sees the [ingress](#g-ingress) identity in both places where it expects a real identity.

```mermaid
flowchart LR
    subgraph Laptop
        BR(["Browser"])
    end
    HX["helix-test.pinadmin.com"]
    subgraph ING["ingress-pinadmin (Envoy)"]
        MH["mcp-devrestricted-dmachicao-1.pinterdev.com<br/>SSO required"]
        WH["www-devrestricted-dmachicao.pinterdev.com<br/>no SSO"]
    end
    subgraph D2["devapp 2 (devrestricted-dmachicao-1)"]
        M["MCP v2 :8000"]
    end
    subgraph D1["devapp 1 (devrestricted-dmachicao)"]
        B["token_broker :10001<br/>SQLite /tmp/token_broker_dev.db<br/>in-memory encryption key"]
    end
    SF["Snowflake dev account xla76048"]

    HX -- "POST /mcp + Helix JWT" --> MH --> M
    M -- "GET /api/token + JWT via public URL" --> WH
    BR -- "GET /oauth/start (connect link)" --> WH
    WH -- "X-Forwarded-User = ingress SPIFFE<br/>XFCC URI = ingress SPIFFE" --> B
    SF -- "callback via browser" --> WH
    M -- "SQL with OAuth token" --> SF
```

Code references:

1. `M :8000`: ([`runtime.dev.yaml` L1-4](./OAuthFlowAndDevProblemOffline.html#module-mcp-runtime-config)).
2. `B :10001`: the port comes from the `PORT` environment variable; the default is 8080 ([`settings.py` L40-41](./OAuthFlowAndDevProblemOffline.html#module-broker-config)), ([`main.py` L103](./OAuthFlowAndDevProblemOffline.html#module-broker-main)).
3. `B` SQLite: ([`engine.py` L53-60](./OAuthFlowAndDevProblemOffline.html#class-broker-engine)).
4. `B` in-memory key: ([`encryption.py` L46-72](./OAuthFlowAndDevProblemOffline.html#method-broker-get-dev-primitive)).
5. `SF` account: ([`runtime.dev.yaml` L6-9](./OAuthFlowAndDevProblemOffline.html#module-mcp-runtime-config)).
6. `M` to `WH`: the MCP's broker URL on devapp 2 is a local override of ([`runtime.dev.yaml` L217](./OAuthFlowAndDevProblemOffline.html#module-mcp-runtime-config)); the committed default is loopback.
7. `WH` to `B`: the broker reads both headers without checking who set them ([`routes/oauth.py` L32-45](./OAuthFlowAndDevProblemOffline.html#method-broker-oauth-start)), ([`spiffe.py` L23-58](./OAuthFlowAndDevProblemOffline.html#method-broker-parse-spiffe)).

### 6.2 What happens when you follow the real user workflow

This is the loop recorded in trace 41 and confirmed from the [broker](#g-token-broker) database in trace 80. The link completes successfully at Snowflake, the [broker](#g-token-broker) stores a valid token, and [Helix](#g-helix) still says "auth required", because the row was written under a different key than the one `/api/token` reads.

```mermaid
sequenceDiagram
    autonumber
    actor U as You (browser)
    participant H as Helix-test
    participant M as MCP v2 (devapp 2)
    participant I as Ingress (www-* host, no SSO)
    participant B as Broker (devapp 1)
    participant S as Snowflake

    H->>M: POST /mcp tools/call (Authorization: Bearer {jwt})
    M->>I: GET https://www-devrestricted-dmachicao.pinterdev.com/api/token?mcp_server_id=snowflake<br/>Authorization: Bearer {jwt}
    I->>B: + X-Forwarded-Client-Cert: ...URI={ingress SPIFFE}
    Note over B: reads key (dmachicao, ingress SPIFFE, snowflake): none
    B-->>M: {"status":"auth_required"}
    M-->>H: auth_start_url = https://www-devrestricted-dmachicao.pinterdev.com/oauth/start?<br/>client_id={devapp SPIFFE}&mcp_server_id=snowflake

    U->>I: GET /oauth/start?client_id={devapp SPIFFE}&mcp_server_id=snowflake<br/>(no SSO on this host)
    I->>B: + X-Forwarded-User: {ingress SPIFFE}
    Note over B: writes state for (ingress SPIFFE, devapp SPIFFE, snowflake)
    B-->>U: 307 to Snowflake authorize
    U->>S: sign in and consent
    S-->>U: 302 to /oauth/callback?code&state
    U->>I: GET /oauth/callback?code={code}&state={state}
    I->>B: forwarded
    B->>S: POST /oauth/token-request (code, verifier, secret)
    S-->>B: tokens
    Note over B: stores row (ingress SPIFFE, devapp SPIFFE, snowflake)

    H->>M: retry
    M->>I: GET /api/token (same JWT)
    I->>B: XFCC URI = ingress SPIFFE
    Note over B: reads (dmachicao, ingress SPIFFE, snowflake): still none
    B-->>M: {"status":"auth_required"}
    M-->>H: same link again (loop)
```

Code references:

1. Steps 3 and 17 and the notes after them, partition from the ingress certificate: ([`spiffe.py` L23-35](./OAuthFlowAndDevProblemOffline.html#method-broker-parse-spiffe)); read key: ([`api.py` L21-35](./OAuthFlowAndDevProblemOffline.html#method-broker-get-token)).
2. Step 5: `client_id={devapp SPIFFE}` comes from ([`runtime.dev.yaml` L218](./OAuthFlowAndDevProblemOffline.html#module-mcp-runtime-config)).
3. Step 7 and the note after it: the ingress SPIFFE is taken as the user because the header is present ([`routes/oauth.py` L45](./OAuthFlowAndDevProblemOffline.html#method-broker-oauth-start)).
4. Steps 11-14 and the note after them: ([`routes/oauth.py` L75-103](./OAuthFlowAndDevProblemOffline.html#method-broker-oauth-callback)), ([`services/oauth.py` L101-157](./OAuthFlowAndDevProblemOffline.html#method-broker-exchange-code)).

The keys side by side:

| Key part | Written by `/oauth/start` | Read by `/api/token` | Match |
| --- | --- | --- | --- |
| `user_id` | [`X-Forwarded-User`](#g-x-forwarded-user) = [ingress](#g-ingress) [SPIFFE](#g-spiffe) | [JWT](#g-jwt) [`sub`](#g-claims) = `dmachicao` | no |
| `client_id` | link `?client_id=` = [devapp](#g-devapp) [SPIFFE](#g-spiffe) | [XFCC](#g-xfcc) `URI=` = [ingress](#g-ingress) [SPIFFE](#g-spiffe) | no |
| `mcp_server_id` | `snowflake` | `snowflake` | yes |

### 6.3 The trick that made the first run pass

To prove the rest of the pipeline, the agent built its own start request on [devapp](#g-devapp) 1 over [loopback](#g-loopback), where no [ingress](#g-ingress) sets [`X-Forwarded-User`](#g-x-forwarded-user), so the [broker](#g-token-broker) fell back to the `user_id` query parameter ([code](./OAuthFlowAndDevProblemOffline.html#method-broker-oauth-start)). It also put the [ingress](#g-ingress) [SPIFFE](#g-spiffe) in `client_id` so the [partition](#g-partition) matched. You then signed in on that hand-made link, not on the link [Helix](#g-helix) showed you. The retry worked. That proved the returning-user path (token retrieval, [refresh](#g-refresh-token), querying as your user) but not the first-time link a real user performs. It is the section 5.2 pattern performed by a trusted operator, which is why it is ruled out for the real test.

```mermaid
sequenceDiagram
    autonumber
    participant A as Agent shell on devapp 1
    participant B as Broker (127.0.0.1:10001)
    actor U as You (browser)
    participant S as Snowflake
    participant H as Helix-test
    participant M as MCP v2

    A->>B: GET http://127.0.0.1:10001/oauth/start?user_id=dmachicao<br/>&client_id={ingress SPIFFE}&mcp_server_id=snowflake<br/>(loopback: no X-Forwarded-User)
    Note over B: user_id falls back to the query parameter<br/>state saved for (dmachicao, ingress SPIFFE, snowflake)
    B-->>A: 307 Location: Snowflake authorize URL
    A-->>U: hand-made authorize link (not the Helix link)
    U->>S: sign in and consent
    S-->>B: callback via www-* host, code and state
    Note over B: row stored under (dmachicao, ingress SPIFFE, snowflake)
    H->>M: retry in Helix
    M->>B: GET /api/token via www-* host (JWT sub dmachicao, XFCC ingress SPIFFE)
    Note over B: key matches, refresh took about 15 s
    B-->>M: {"status":"ok","access_token":"..."}
    M-->>H: SELECT CURRENT_USER() ran as your user (query id recorded, trace 47)
```

Code references:

1. Step 1 and the note after it: ([`routes/oauth.py` L29-50](./OAuthFlowAndDevProblemOffline.html#method-broker-oauth-start)).
2. Step 5 and the note after it: ([`services/oauth.py` L107-157](./OAuthFlowAndDevProblemOffline.html#method-broker-exchange-code)).
3. Step 7 and the note "refresh took about 15 s": refresh path ([`api.py` L37-60](./OAuthFlowAndDevProblemOffline.html#method-broker-get-token)); each Knox attempt falls back to the dev key only after it fails ([`encryption.py` L58-72](./OAuthFlowAndDevProblemOffline.html#method-broker-get-aead-primitive)).
4. Step 9: ([`execute_snowflake_query.py` L83-97](./OAuthFlowAndDevProblemOffline.html#method-mcp-execute-snowflake-query)).

### 6.4 Why a config-only change on the current host does not help

Setting the [MCP](#g-mcp)'s `token_broker.client_id` ([config](./OAuthFlowAndDevProblemOffline.html#module-mcp-runtime-config)) to the [ingress](#g-ingress) [SPIFFE](#g-spiffe) fixes the [partition](#g-partition) column only. The browser still reaches the [broker](#g-token-broker) without [SSO](#g-sso), so the user column stays the [ingress](#g-ingress) [SPIFFE](#g-spiffe) while `/api/token` reads `dmachicao` (trace 80).

| Key part | Written | Read | Match |
| --- | --- | --- | --- |
| `user_id` | [ingress](#g-ingress) [SPIFFE](#g-spiffe) | `dmachicao` | no |
| `client_id` | [ingress](#g-ingress) [SPIFFE](#g-spiffe) (new config) | [ingress](#g-ingress) [SPIFFE](#g-spiffe) | yes |

## 7. Candidate dev fix: put the broker behind the signed-in host

Each [devapp](#g-devapp) has one `mcp-*` host that routes to port 8000 and requires Pinterest [SSO](#g-sso). [Devapp](#g-devapp) 1's port 8000 is free. Running the [broker](#g-token-broker) there with `PORT=8000` puts it behind [SSO](#g-sso) like production, with no code change to either service. This is also the pattern the shared [MCP](#g-mcp) [OAuth](#g-oauth) library documents for [devapps](#g-devapp): its `base_url` (browser start link and provider [callback](#g-redirect-uri)) is `https://mcp-<devapp>.pinterdev.com` ([`docs/oauth/oauth-mcp-server-devapp.md`](./OAuthFlowAndDevProblemOffline.html#element-runtime-devapps)). Config changes needed:

1. [Broker](#g-token-broker): `PORT=8000` ([code](./OAuthFlowAndDevProblemOffline.html#module-broker-config)); provider [`redirect_uri`](#g-redirect-uri) and `allowed_redirect_uris` set to `https://mcp-devrestricted-dmachicao.pinterdev.com/oauth/callback` ([code](./OAuthFlowAndDevProblemOffline.html#method-broker-validate-redirect)).
2. Snowflake [integration](#g-security-integration): [`OAUTH_REDIRECT_URI`](#g-redirect-uri) set to the same [callback](#g-redirect-uri).
3. [MCP](#g-mcp): `token_broker.url` set to `https://mcp-devrestricted-dmachicao.pinterdev.com`, and `token_broker.client_id` set to the [partition](#g-partition) that host produces ([config](./OAuthFlowAndDevProblemOffline.html#module-mcp-runtime-config)).

Three facts are unverified, marked with `?` in the diagram:

1. [`X-Forwarded-User`](#g-x-forwarded-user) holds `dmachicao` on that host. The shared [MCP](#g-mcp) helper also reads [`x-pinterest-forwarded-user`](#g-x-forwarded-user) because the [LDAP username](#g-ldap) sometimes arrives only there ([code](./OAuthFlowAndDevProblemOffline.html#method-mcp-authorization)); [`token_broker`](#g-token-broker) reads only [`X-Forwarded-User`](#g-x-forwarded-user) ([code](./OAuthFlowAndDevProblemOffline.html#method-broker-oauth-start)).
2. The [ingress](#g-ingress) accepts the forwarded [Helix](#g-helix) [JWT](#g-jwt) on the [MCP](#g-mcp)'s server call to that host. It accepts [Helix](#g-helix)'s [JWT](#g-jwt) on [devapp](#g-devapp) 2 today, and it rejects a [devapp](#g-devapp) `PIN_AUTH_TOKEN` ([`DISCOVERY.md:155-158`](./OAuthFlowAndDevProblemOffline.html#actor-ingress)).
3. The [XFCC](#g-xfcc) `URI=` value on that path, which becomes the [partition](#g-partition) ([code](./OAuthFlowAndDevProblemOffline.html#method-broker-parse-spiffe)).

```mermaid
sequenceDiagram
    autonumber
    actor U as You (browser)
    participant H as Helix-test
    participant M as MCP v2 (devapp 2)
    participant I as Ingress (mcp-devrestricted-dmachicao, SSO)
    participant A as auth.pinadmin.com
    participant B as Broker (devapp 1 :8000)
    participant S as Snowflake

    H->>M: POST /mcp tools/call (Bearer {jwt})
    M->>I: GET /api/token?mcp_server_id=snowflake (Bearer {jwt})
    Note over I: accepts Helix JWT? (unverified 2)
    I->>B: XFCC URI = ? (probably ingress SPIFFE, unverified 3)
    B-->>M: {"status":"auth_required"}
    M-->>H: link https://mcp-devrestricted-dmachicao.pinterdev.com/oauth/start?client_id={that partition}&...

    U->>I: GET /oauth/start?client_id={that partition}&mcp_server_id=snowflake
    I-->>U: 302 to auth.pinadmin.com (no session yet)
    U->>A: SSO
    A-->>U: back with session cookie
    U->>I: GET /oauth/start?...
    I->>B: X-Forwarded-User: dmachicao ? (unverified 1)
    Note over B: state for (dmachicao, that partition, snowflake)
    B-->>U: 307 to Snowflake authorize (redirect_uri = mcp-devrestricted-dmachicao/oauth/callback)
    U->>S: sign in and consent
    S-->>U: 302 to https://mcp-devrestricted-dmachicao.pinterdev.com/oauth/callback?code&state
    U->>I: GET /oauth/callback (session cookie already present)
    I->>B: forwarded
    B->>S: token exchange
    Note over B: row (dmachicao, that partition, snowflake)
    H->>M: retry
    M->>I: GET /api/token (Bearer {jwt})
    I->>B: XFCC URI = that partition
    Note over B: key matches
    B-->>M: {"status":"ok","access_token":"..."}
```

Code references:

1. Steps 2-3 and 19-20: the broker call and its partition ([`token_broker_client.py` L49-64](./OAuthFlowAndDevProblemOffline.html#method-mcp-get-snowflake-access-token)), ([`spiffe.py` L23-58](./OAuthFlowAndDevProblemOffline.html#method-broker-parse-spiffe)).
2. Step 5: link from `token_broker.url` and `token_broker.client_id` ([`token_broker_client.py` L74-85](./OAuthFlowAndDevProblemOffline.html#method-mcp-get-snowflake-access-token)).
3. Step 11 and the note after it: ([`routes/oauth.py` L32-50](./OAuthFlowAndDevProblemOffline.html#method-broker-oauth-start)).
4. Step 12, `redirect_uri` checked against the provider config: ([`services/oauth.py` L33-46](./OAuthFlowAndDevProblemOffline.html#method-broker-validate-redirect)), ([`services/oauth.py` L69-70](./OAuthFlowAndDevProblemOffline.html#method-broker-build-authorize-url)).
5. Steps 15-17 and the note after them: ([`routes/oauth.py` L75-103](./OAuthFlowAndDevProblemOffline.html#method-broker-oauth-callback)), ([`services/oauth.py` L101-157](./OAuthFlowAndDevProblemOffline.html#method-broker-exchange-code)).
6. Step 21: ([`api.py` L73-95](./OAuthFlowAndDevProblemOffline.html#method-broker-get-token)).

The probe that settles unverified facts 1 and 3 before any [Helix](#g-helix) step: a temporary logger on [devapp](#g-devapp) 1 port 8000 that you hit once from your browser. It logs only the two forwarded-user header values, the [XFCC](#g-xfcc) `URI=` values, and header names; never cookies or [`Authorization`](#g-bearer).

```mermaid
sequenceDiagram
    autonumber
    actor U as You (browser)
    participant I as Ingress (mcp-devrestricted-dmachicao, SSO)
    participant P as Header probe (devapp 1 :8000)

    U->>I: GET https://mcp-devrestricted-dmachicao.pinterdev.com/whoami
    I-->>U: 302 to auth.pinadmin.com if no session, then back
    I->>P: GET /whoami + X-Forwarded-User + X-Pinterest-Forwarded-User + XFCC
    Note over P: log: x-forwarded-user, x-pinterest-forwarded-user,<br/>XFCC URI values, header names only
    P-->>U: 200 "probe ok"
    Note over P: decision:<br/>X-Forwarded-User = dmachicao -> candidate fix works by config<br/>only x-pinterest-forwarded-user = dmachicao -> needs a token_broker change (it-swe)<br/>neither -> first-time test belongs in staging
```

Code references:

1. The probe script is not in any repo; it only mirrors what the broker reads.
2. Step 3, the headers logged: the broker reads `X-Forwarded-User` ([`routes/oauth.py` L32](./OAuthFlowAndDevProblemOffline.html#method-broker-oauth-start)) and XFCC ([`spiffe.py` L38-58](./OAuthFlowAndDevProblemOffline.html#method-broker-get-caller-spiffe)); the shared MCP library also reads `x-pinterest-forwarded-user` ([`authorizer.py` L32-60](./OAuthFlowAndDevProblemOffline.html#method-mcp-authorization)).
3. Helix sets `x-pinterest-forwarded-user` itself only on calls without a JWT (scheduled chats) ([prompthub `server.py` L139-147](./OAuthFlowAndDevProblemOffline.html#actor-helix)); a browser request gets it only if the ingress adds it (unverified).

## 8. One devapp or two

The number of [devapps](#g-devapp) is not about CPU or memory. It follows from how the [devapp](#g-devapp) [ingress](#g-ingress) routes hostnames to ports, and from which of our two processes needs Pinterest [SSO](#g-sso) in front of it.

### 8.1 The routing rule that decides it

Each [devapp](#g-devapp) gets a fixed set of public hostnames, and each hostname goes to one fixed port. Only one of them enforces Pinterest [SSO](#g-sso). There is no per-path routing at the [ingress](#g-ingress) and no way to add a second [SSO](#g-sso) host to the same [devapp](#g-devapp).

| Public host on [devapp](#g-devapp) `<host>` | Port on the [devapp](#g-devapp) | [SSO](#g-sso) at the [ingress](#g-ingress) | Evidence |
| --- | --- | --- | --- |
| `https://mcp-<host>.pinterdev.com` | 8000 | yes: anonymous requests get [302](#g-http-redirect) to `auth.pinadmin.com`; [Helix](#g-helix)'s forwarded [JWT](#g-jwt) is accepted | probes in trace 84; [Helix](#g-helix) reaches the [devapp](#g-devapp) 2 [MCP](#g-mcp) today |
| `https://www-<host>.pinterdev.com` | 10001 | no: requests pass through | probe returns 503 from [Envoy](#g-envoy) with no [redirect](#g-http-redirect) (trace 84); [broker](#g-token-broker) rows stored the [ingress](#g-ingress) [SPIFFE](#g-spiffe) as the user (trace 80) |
| `https://api-<host>.pinterdev.com` | 10002 | no | same probe result as `www` |

Other prefixes seen in repo docs (`oabs-`, `sterling-`, `langfuse-`) are service-specific routes, not something we can claim for our processes.

### 8.2 The requirements each process puts on the topology

Five requirements come from the flow in sections 2-4. Together they decide where each process can run.

1. **[Helix](#g-helix) must reach the [MCP](#g-mcp) on a public host.** [Helix](#g-helix) runs in the cluster; it cannot call `localhost` on a [devapp](#g-devapp). The documented and verified host is the `mcp-*` host (port 8000).
2. **The browser must reach the [broker](#g-token-broker)'s `/oauth/start` through [SSO](#g-sso).** Only then does [`X-Forwarded-User`](#g-x-forwarded-user) carry the human (section 5.2). On a [devapp](#g-devapp) that means the [broker](#g-token-broker) must listen on port 8000 behind the `mcp-*` host.
3. **The Snowflake [callback](#g-redirect-uri) must reach the [broker](#g-token-broker).** Any host works, because the [callback](#g-redirect-uri) carries no identity (section 4.2). It must match the [integration](#g-security-integration)'s [`OAUTH_REDIRECT_URI`](#g-redirect-uri) exactly.
4. **The [MCP](#g-mcp)'s `/api/token` call must pass through an [Envoy](#g-envoy).** The [broker](#g-token-broker) rejects a request without [`X-Forwarded-Client-Cert`](#g-xfcc) ([code](./OAuthFlowAndDevProblemOffline.html#method-broker-get-caller-spiffe)), so a [loopback](#g-loopback) call from the [MCP](#g-mcp) to the [broker](#g-token-broker) on the same [devapp](#g-devapp) fails even when both run there. Sending a hand-written [XFCC](#g-xfcc) header over [loopback](#g-loopback) would be spoofing the [mesh](#g-mesh), which is a trick, not a test.
5. **The link's `client_id` must equal the [partition](#g-partition) from requirement 4** (section 2).

Requirements 1 and 2 both want the single [SSO](#g-sso) host of a [devapp](#g-devapp), which maps to one port. That collision is the whole reason for a second [devapp](#g-devapp).

### 8.3 Two devapps: the straightforward layout

The [broker](#g-token-broker) takes [devapp](#g-devapp) 1's [SSO](#g-sso) host and the [MCP](#g-mcp) takes [devapp](#g-devapp) 2's [SSO](#g-sso) host. Every requirement is met with config only, and each process keeps its documented host. This is the candidate in section 7. Use two [devapps](#g-devapp) when you want no extra moving parts, or when you also run a local [Helix](#g-helix) or [PromptHub](#g-prompthub) (it also wants port 8000, which is why the previous engineer put [PromptHub](#g-prompthub) on his second [devapp](#g-devapp)).

```mermaid
flowchart LR
    HX["Helix-test"]
    BR(["Browser"])
    subgraph D2["devapp 2"]
        M["MCP v2 :8000"]
    end
    subgraph D1["devapp 1"]
        B["token_broker :8000"]
    end
    MH2["mcp-devrestricted-dmachicao-1<br/>SSO"]
    MH1["mcp-devrestricted-dmachicao<br/>SSO"]

    HX -- "POST /mcp + JWT (req 1)" --> MH2 --> M
    M -- "GET /api/token + JWT (req 4)" --> MH1
    BR -- "GET /oauth/start (req 2)" --> MH1
    BR -- "GET /oauth/callback (req 3)" --> MH1
    MH1 -- "X-Forwarded-User, XFCC" --> B
```

Code references:

1. `B :8000`: `PORT` setting ([`settings.py` L40-41](./OAuthFlowAndDevProblemOffline.html#module-broker-config)).
2. `M :8000`: ([`runtime.dev.yaml` L1-4](./OAuthFlowAndDevProblemOffline.html#module-mcp-runtime-config)).
3. req 1: ([prompthub `server.py` L139-147](./OAuthFlowAndDevProblemOffline.html#actor-helix)).
4. req 2: ([`routes/oauth.py` L32-50](./OAuthFlowAndDevProblemOffline.html#method-broker-oauth-start)).
5. req 3: ([`routes/oauth.py` L75-116](./OAuthFlowAndDevProblemOffline.html#method-broker-oauth-callback)).
6. req 4: ([`spiffe.py` L38-58](./OAuthFlowAndDevProblemOffline.html#method-broker-get-caller-spiffe)), ([`token_broker_client.py` L49-64](./OAuthFlowAndDevProblemOffline.html#method-mcp-get-snowflake-access-token)).

### 8.4 One devapp, option A: MCP on the unauthenticated host

Put the [broker](#g-token-broker) on port 8000 behind the [SSO](#g-sso) host, and the [MCP](#g-mcp) on port 10001 behind `www-*`. [Helix](#g-helix) would register `https://www-devrestricted-dmachicao.pinterdev.com/mcp`. Everything else is identical to section 8.3, still config only.

Two unverified points decide whether it works:

1. Whether [Helix-test](#g-helix) can reach a `www-*` [devapp](#g-devapp) host at all. Only `mcp-*` hosts are documented for [Helix](#g-helix).
2. Whether the [ingress](#g-ingress) passes [Helix](#g-helix)'s [`Authorization`](#g-bearer) header through unchanged on a host without [SSO](#g-sso). The [MCP](#g-mcp) needs it to forward to the [broker](#g-token-broker).

The trade-off: the [MCP](#g-mcp) endpoint has no [SSO](#g-sso) in front of it, so anyone on the corporate network can call its tools. Each call still needs a valid Pinterest [JWT](#g-jwt), and the [broker](#g-token-broker) returns only the token linked to that [JWT](#g-jwt)'s user, so the exposure is limited to people using their own identity. Acceptable for a short dev test, not a pattern to copy.

```mermaid
flowchart LR
    HX["Helix-test"]
    BR(["Browser"])
    subgraph D1["devapp 1 (single devapp)"]
        M["MCP v2 :10001"]
        B["token_broker :8000"]
    end
    WH["www-devrestricted-dmachicao<br/>no SSO"]
    MH["mcp-devrestricted-dmachicao<br/>SSO"]

    HX -- "POST /mcp + JWT<br/>(Helix can reach www-*? unverified)" --> WH --> M
    M -- "GET /api/token + JWT via public host<br/>(not loopback: needs XFCC)" --> MH
    BR -- "/oauth/start and /oauth/callback" --> MH
    MH -- "X-Forwarded-User, XFCC" --> B
```

Code references:

1. `M :10001`: an override of the committed port ([`runtime.dev.yaml` L4](./OAuthFlowAndDevProblemOffline.html#module-mcp-runtime-config)).
2. `M` to `MH` "not loopback: needs XFCC": the MCP client would accept a loopback URL ([`token_broker_client.py` L25-36](./OAuthFlowAndDevProblemOffline.html#method-mcp-get-snowflake-access-token)), but the broker returns 401 without XFCC ([`spiffe.py` L38-58](./OAuthFlowAndDevProblemOffline.html#method-broker-get-caller-spiffe)).
3. `MH` to `B`: same as section 8.3 items 4-6.

### 8.5 One devapp, option B: one SSO host shared by path

Run a small [reverse proxy](#g-reverse-proxy) on port 8000 behind the [SSO](#g-sso) host and route by path: `/mcp` goes to the [MCP](#g-mcp), and `/oauth/*`, `/api/*`, and `/health*` go to the [broker](#g-token-broker). The two services have no overlapping paths. Both are then behind [SSO](#g-sso) on one [devapp](#g-devapp), and the [proxy](#g-reverse-proxy) must pass [`X-Forwarded-User`](#g-x-forwarded-user) and [`X-Forwarded-Client-Cert`](#g-xfcc) through unchanged. The backends should listen on [`127.0.0.1`](#g-loopback) so only the [proxy](#g-reverse-proxy) reaches them.

The cost is an extra component that exists only in dev. [Devapp](#g-devapp) 1 has no nginx, Caddy, HAProxy, or [Envoy](#g-envoy) binary, and `socat` cannot route by path, so the [proxy](#g-reverse-proxy) would be a small script kept outside the repo. It does not change either service's code, and production has the equivalent separation through two distinct hostnames.

```mermaid
flowchart LR
    HX["Helix-test"]
    BR(["Browser"])
    MH["mcp-devrestricted-dmachicao<br/>SSO"]
    subgraph D1["devapp 1 (single devapp)"]
        P["path proxy :8000"]
        M["MCP v2 127.0.0.1:8001"]
        B["token_broker 127.0.0.1:10001"]
    end

    HX -- "POST /mcp + JWT" --> MH
    BR -- "/oauth/start, /oauth/callback" --> MH
    MH --> P
    P -- "/mcp" --> M
    P -- "/oauth/*, /api/*, /health*<br/>headers passed through" --> B
    M -- "GET /api/token + JWT via public host" --> MH
```

Code references:

1. `P`: dev-only; no code in any repo.
2. `M 127.0.0.1:8001`: host and port overrides of ([`runtime.dev.yaml` L1-4](./OAuthFlowAndDevProblemOffline.html#module-mcp-runtime-config)).
3. `B 127.0.0.1:10001`: `HOST` defaults to `0.0.0.0`, so it must be set ([`settings.py` L40-41](./OAuthFlowAndDevProblemOffline.html#module-broker-config)).
4. Broker paths the proxy must route: `/oauth/*` ([`routes/oauth.py` L25-116](./OAuthFlowAndDevProblemOffline.html#method-broker-oauth-start)), `/api/*` ([`api.py` L21-170](./OAuthFlowAndDevProblemOffline.html#method-broker-get-token)).
5. "headers passed through": ([`routes/oauth.py` L32](./OAuthFlowAndDevProblemOffline.html#method-broker-oauth-start)), ([`spiffe.py` L38-58](./OAuthFlowAndDevProblemOffline.html#method-broker-get-caller-spiffe)).

### 8.6 One devapp the way the previous engineer did it, and why it fails now

The previous engineer ran the [broker](#g-token-broker) and the [MCP](#g-mcp) on one [devapp](#g-devapp): the [MCP](#g-mcp) on port 8000 behind the [SSO](#g-sso) host, and the [broker](#g-token-broker) on port 10001 behind `www-*`. The [MCP](#g-mcp) called the [broker](#g-token-broker) over [loopback](#g-loopback). That worked with his [broker](#g-token-broker) ([PR 180341](./OAuthFlowAndDevProblemOffline.html#actor-runtime)) because it did not require [XFCC](#g-xfcc) on `/api/token` ([code](./OAuthFlowAndDevProblemOffline.html#actor-runtime)), and it bound the user to an opaque transaction created during the `/api/token` call ([code](./OAuthFlowAndDevProblemOffline.html#actor-runtime)), so it did not matter who the browser looked like at `/oauth/start`. Master [`token_broker`](#g-token-broker) has neither feature. With it, this layout fails both requirement 2 (the [broker](#g-token-broker) has no [SSO](#g-sso)) and requirement 4 ([loopback](#g-loopback) has no [XFCC](#g-xfcc)).

```mermaid
flowchart LR
    HX["Helix-test"]
    BR(["Browser"])
    subgraph D1["devapp (previous engineer)"]
        M["Snowflake MCP :8000"]
        B["broker :10001"]
    end
    MH["mcp-* host, SSO"]
    WH["www-* host, no SSO"]

    HX --> MH --> M
    M -- "loopback /api/token<br/>PR 180341: OK<br/>master token_broker: rejected, no XFCC" --> B
    BR -- "/oauth/start/{transaction_id}<br/>PR 180341: user bound to transaction<br/>master: user = ingress SPIFFE" --> WH --> B
```

Code references:

1. `M` to `B`, PR 180341: `client_id` came from the JWT `aud` or a query parameter, with no XFCC ([`identity.py` L121-137](./OAuthFlowAndDevProblemOffline.html#actor-runtime)).
2. `M` to `B`, master: ([`spiffe.py` L38-58](./OAuthFlowAndDevProblemOffline.html#method-broker-get-caller-spiffe)).
3. `BR` to `B`, PR 180341: `/api/token` created the transaction and the link ([`api.py` L32-57](./OAuthFlowAndDevProblemOffline.html#actor-runtime)); `/oauth/start/{transaction_id}` read the user from it ([`oauth.py` L60-81](./OAuthFlowAndDevProblemOffline.html#actor-runtime)). It also mapped a devapp SPIFFE in `X-Forwarded-User` back to the LDAP name ([`identity.py` L9-39](./OAuthFlowAndDevProblemOffline.html#actor-runtime)).
4. `BR` to `B`, master: ([`routes/oauth.py` L32-45](./OAuthFlowAndDevProblemOffline.html#method-broker-oauth-start)).

### 8.7 How to choose

```mermaid
flowchart TB
    Q0{"Does the header probe show<br/>X-Forwarded-User = your LDAP<br/>on the mcp-* host?"}
    Q0 -- "no" --> ST["No honest dev test of the first-time link<br/>ask it-swe about x-pinterest-forwarded-user<br/>or test in staging"]
    Q0 -- "yes" --> Q1{"Also running a local<br/>Helix or PromptHub?"}
    Q1 -- "yes" --> TWO["Two devapps (8.3)<br/>plus a third host for PromptHub, or run it elsewhere"]
    Q1 -- "no" --> Q2{"Want zero extra<br/>dev-only components?"}
    Q2 -- "yes" --> Q3{"Can Helix-test reach<br/>a www-* host?"}
    Q3 -- "unknown or no" --> TWO2["Two devapps (8.3)"]
    Q3 -- "yes" --> ONEA["One devapp, option A (8.4)"]
    Q2 -- "no" --> ONEB["One devapp, option B (8.5)<br/>path proxy on :8000"]
```

Code references:

1. `Q0`: the broker reads only `X-Forwarded-User` ([`routes/oauth.py` L32-45](./OAuthFlowAndDevProblemOffline.html#method-broker-oauth-start)).
2. `ST`: the header an it-swe change would add, as the shared MCP library reads it ([`authorizer.py` L32-60](./OAuthFlowAndDevProblemOffline.html#method-mcp-authorization)).

In short: two [devapps](#g-devapp) are needed when both the [broker](#g-token-broker) and the [MCP](#g-mcp) must sit behind [SSO](#g-sso) and you add nothing else, because each [devapp](#g-devapp) has exactly one [SSO](#g-sso) host. One [devapp](#g-devapp) works if the [MCP](#g-mcp) can live on a host without [SSO](#g-sso) (option A, unverified), or if a small dev-only [proxy](#g-reverse-proxy) shares the one [SSO](#g-sso) host by path (option B).

## 9. Options and what each proves

| Option | [Devapps](#g-devapp) | Code change | Real user steps in [Helix](#g-helix) | Proves the first-time link | Notes |
| --- | --- | --- | --- | --- | --- |
| [Broker](#g-token-broker) behind [devapp](#g-devapp) 1 `mcp-*` host, [MCP](#g-mcp) behind [devapp](#g-devapp) 2 `mcp-*` host (8.3) | 2 | none (config only) | yes | yes, if the probe shows `X-Forwarded-User = dmachicao` | Matches the [MCP](#g-mcp) [OAuth](#g-oauth) library's [devapp](#g-devapp) pattern |
| [Broker](#g-token-broker) behind `mcp-*`, [MCP](#g-mcp) behind `www-*` (8.4) | 1 | none | yes | yes, same probe condition | Unverified that [Helix](#g-helix) can reach `www-*`; [MCP](#g-mcp) has no [SSO](#g-sso) |
| [Broker](#g-token-broker) and [MCP](#g-mcp) behind one `mcp-*` host with a path [proxy](#g-reverse-proxy) (8.5) | 1 | none in either service; a dev-only [proxy](#g-reverse-proxy) script | yes | yes, same probe condition | Extra component only dev has |
| Ask it-swe to accept [`x-pinterest-forwarded-user`](#g-x-forwarded-user) | 1 or 2 | [`token_broker`](#g-token-broker) (it-swe) | yes | yes | Only if the probe shows the [LDAP](#g-ldap) in that header; correct in production too |
| [Loopback](#g-loopback) start with `user_id` (trace 42-47) | 1 or 2 | none | no, hand-made link | no | Proves the returning-user path only |
| [MCP](#g-mcp)-side dev "connect" route | 1 | [snowflake_v2](#g-mcp), dev-only path | yes, but through a path production lacks | no | Breaks dev/prod parity; [token-planting](#g-login-csrf) risk if it ever reached production |
| Staging [broker](#g-token-broker) (`token-broker-staging.pinadmin.com`) | 0 or 1 | none in v2 | yes | yes | Needs it-swe provider deploy, [Knox](#g-knox) keys, and a production-account [integration](#g-security-integration) |

## 10. Terms and glossary

Each entry gives a general definition, what the term means in this document, an example where one helps, and internal source-map references. When the term is an exchange between parties, the general part has the canonical sequence if a standard defines one, and the in-this-doc part has the Pinterest sequence. A message lists every field the recorded source shows for that request or response. A field this document's sources do not specify is left out. Every source reference opens the matching element in `OAuthFlowAndDevProblemOffline.html`; no Optimus checkout or Pinterest network link is required. Inside the glossary, only the first mention of another term in each entry is linked.

### 10.1 OAuth

1. <a id="g-oauth"></a>**OAuth** (OAuth 2.0)
    - **a.** General: an open standard ([RFC 6749](./OAuthFlowAndDevProblemOffline.html#g-oauth)) that lets one application act for a user at another service without ever seeing the user's password. The authorization-code grant (§4.1) is the canonical exchange. The sample values are the RFC's own (`client_id=s6BhdRkqt3`, `code=SplxlOBeZQQYbYS6WxSbIA`, `Authorization: Basic czZCaGRSa3F0MzpnWDFmQmF0M2JW`). [PKCE](#g-pkce) is not part of this diagram.

      ```mermaid
      sequenceDiagram
          actor UA as User-agent
          participant C as Client
          participant AS as Authorization server
          C->>UA: 302 Location: https://server.example.com/authorize?response_type=code&client_id=s6BhdRkqt3&state=xyz&redirect_uri=https://client.example.com/cb
          UA->>AS: GET /authorize?response_type=code&client_id=s6BhdRkqt3&state=xyz&redirect_uri=https://client.example.com/cb
          Note over AS: authenticate the resource owner and obtain consent
          AS-->>UA: 302 Location: https://client.example.com/cb?code=SplxlOBeZQQYbYS6WxSbIA&state=xyz
          UA->>C: GET /cb?code=SplxlOBeZQQYbYS6WxSbIA&state=xyz
          C->>AS: POST /token<br/>Authorization: Basic czZCaGRSa3F0MzpnWDFmQmF0M2JW<br/>Content-Type: application/x-www-form-urlencoded<br/>grant_type=authorization_code&code=SplxlOBeZQQYbYS6WxSbIA&redirect_uri=https://client.example.com/cb
          AS-->>C: 200 Content-Type: application/json;charset=UTF-8<br/>Cache-Control: no-store<br/>Pragma: no-cache<br/>{"access_token":"2YotnFZFEjr1zCsicMWpAA","token_type":"example","expires_in":3600,"refresh_token":"tGzv3JOkF0XG5Qx2TlKWIA"}
      ```

    - **b.** In this doc: the [broker](#g-token-broker) is the client and Snowflake is the authorization server. The client secret is a form field, not HTTP Basic, and the authorize request also carries [PKCE](#g-pkce). How the browser obtains `X-Forwarded-User` is the [SSO](#g-sso) sequence. The resource owner's consent POST to Snowflake is not in the cited sources, so that step stays the user action "approve".

      ```mermaid
      sequenceDiagram
          actor U as Browser
          participant B as token_broker
          participant DB as Token store
          participant S as Snowflake
          U->>B: GET /oauth/start?client_id={MCP SPIFFE}&mcp_server_id=snowflake<br/>X-Forwarded-User: dmachicao
          B->>DB: INSERT oauth_states (state={state}, user_id=dmachicao, client_id={MCP SPIFFE}, mcp_server_id=snowflake, code_verifier=encrypt(code_verifier), redirect_uri=https://token-broker.pinadmin.com/oauth/callback, expires_at=now+600s)
          B-->>U: 307 Location: https://{account}.snowflakecomputing.com/oauth/authorize?response_type=code&client_id={sf_client_id}&redirect_uri=https://token-broker.pinadmin.com/oauth/callback&state={state}&scope=refresh_token&code_challenge={code_challenge}&code_challenge_method=S256
          U->>S: GET /oauth/authorize?response_type=code&client_id={sf_client_id}&redirect_uri=https://token-broker.pinadmin.com/oauth/callback&state={state}&scope=refresh_token&code_challenge={code_challenge}&code_challenge_method=S256
          U->>S: approve
          S-->>U: 302 Location: https://token-broker.pinadmin.com/oauth/callback?code={code}&state={state}
          U->>B: GET /oauth/callback?code={code}&state={state}
          B->>S: POST /oauth/token-request<br/>Content-Type: application/x-www-form-urlencoded<br/>grant_type=authorization_code&code={code}&redirect_uri=https://token-broker.pinadmin.com/oauth/callback&client_id={sf_client_id}&client_secret={sf_client_secret}&code_verifier={code_verifier}
          S-->>B: 200 {"access_token":"{access_token}","token_type":"Bearer","expires_in":600,"refresh_token":"{refresh_token}","scope":"{scope}"}
          B->>DB: UPSERT mcp_tokens (user_id=dmachicao, client_id={MCP SPIFFE}, mcp_server_id=snowflake, access_token=encrypt({access_token}), refresh_token=encrypt({refresh_token}), token_type, expires_at, scopes)
          B->>DB: INSERT audit_logs (user_id=dmachicao, client_id={MCP SPIFFE}, mcp_server_id=snowflake, action=link)
          B-->>U: 200 success.html
      ```

    - **c.** The Pinterest login at `auth.pinadmin.com` also has "oauth" in names such as [`internal_oauth`](#g-internal-oauth), but that is [SSO](#g-sso), a separate step in front of the host.
    - **d.** Source: [`routes/oauth.py` L25-L116](./OAuthFlowAndDevProblemOffline.html#method-broker-oauth-start), [`services/oauth.py` L48-L174](./OAuthFlowAndDevProblemOffline.html#method-broker-build-authorize-url). Canonical messages: [RFC 6749 §4.1 and §5.1](./OAuthFlowAndDevProblemOffline.html#g-oauth).
2. <a id="g-authorization-server"></a>**Authorization server**
    - **a.** General: the OAuth server that authenticates the user, shows the consent screen, and issues authorization codes and tokens.
    - **b.** In this doc: Snowflake, configured by its [security integration](#g-security-integration). Endpoints: `/oauth/authorize` (browser), `/oauth/token-request` ([token exchange](#g-token-exchange) and [refresh](#g-refresh-token)), and `/oauth/revoke` ([revocation](#g-revocation)).
    - **c.** Source: Snowflake provider entry in [`mcp_servers.production.yaml` L27-L37](./OAuthFlowAndDevProblemOffline.html#class-broker-server-config).
3. <a id="g-authorization-code"></a>**Authorization code** (`{code}`)
    - **a.** General: a short-lived, single-use value the authorization server sends back through the user-agent after the user approves ([RFC 6749 §4.1.2](./OAuthFlowAndDevProblemOffline.html#g-authorization-code)). It is not a token. The only message that carries it is the redirect to the client's [redirect URI](#g-redirect-uri).

      ```mermaid
      sequenceDiagram
          actor UA as User-agent
          participant C as Client
          participant AS as Authorization server
          AS-->>UA: 302 Location: https://client.example.com/cb?code=SplxlOBeZQQYbYS6WxSbIA&state=xyz
          UA->>C: GET /cb?code=SplxlOBeZQQYbYS6WxSbIA&state=xyz
      ```

    - **b.** In this doc: Snowflake redirects the browser to the [broker](#g-token-broker). The callback query is `code` and `state` on success. The handler also accepts `error` and `error_description` when Snowflake denies the request; those two are absent on success.

      ```mermaid
      sequenceDiagram
          actor U as Browser
          participant S as Snowflake
          participant B as token_broker
          S-->>U: 302 Location: https://token-broker.pinadmin.com/oauth/callback?code={code}&state={state}
          U->>B: GET /oauth/callback?code={code}&state={state}
      ```

    - **c.** Source: [`exchange_code_for_tokens`, `services/oauth.py` L101-L174](./OAuthFlowAndDevProblemOffline.html#method-broker-exchange-code), callback query parameters [`routes/oauth.py` L75-L81](./OAuthFlowAndDevProblemOffline.html#method-broker-oauth-callback).
4. <a id="g-token-exchange"></a>**Token exchange** (token endpoint, `grant_type`)
    - **a.** General: a direct client-to-authorization-server POST, with no user-agent involved ([RFC 6749 §4.1.3](./OAuthFlowAndDevProblemOffline.html#g-token-exchange) for the code, [§6](./OAuthFlowAndDevProblemOffline.html#g-refresh-token) for refresh). `grant_type` says which. The canonical confidential client authenticates with HTTP Basic. The success body is [§5.1](./OAuthFlowAndDevProblemOffline.html#g-token-exchange).

      ```mermaid
      sequenceDiagram
          participant C as Client
          participant AS as Authorization server
          C->>AS: POST /token<br/>Authorization: Basic czZCaGRSa3F0MzpnWDFmQmF0M2JW<br/>Content-Type: application/x-www-form-urlencoded<br/>grant_type=authorization_code&code=SplxlOBeZQQYbYS6WxSbIA&redirect_uri=https://client.example.com/cb
          AS-->>C: 200 Content-Type: application/json;charset=UTF-8<br/>Cache-Control: no-store<br/>Pragma: no-cache<br/>{"access_token":"2YotnFZFEjr1zCsicMWpAA","token_type":"example","expires_in":3600,"refresh_token":"tGzv3JOkF0XG5Qx2TlKWIA"}
          C->>AS: POST /token<br/>Authorization: Basic czZCaGRSa3F0MzpnWDFmQmF0M2JW<br/>Content-Type: application/x-www-form-urlencoded<br/>grant_type=refresh_token&refresh_token=tGzv3JOkF0XG5Qx2TlKWIA
          AS-->>C: 200 Content-Type: application/json;charset=UTF-8<br/>Cache-Control: no-store<br/>Pragma: no-cache<br/>{"access_token":"2YotnFZFEjr1zCsicMWpAA","token_type":"example","expires_in":3600,"refresh_token":"tGzv3JOkF0XG5Qx2TlKWIA"}
      ```

    - **b.** In this doc: `POST /oauth/token-request` from the [broker](#g-token-broker) to Snowflake. Authentication is `client_id` and `client_secret` in the body. The code grant also sends the [PKCE](#g-pkce) verifier. The refresh grant does not. The broker reads `access_token`, `refresh_token`, `token_type`, `expires_in`, and `scope` from the JSON.

      ```mermaid
      sequenceDiagram
          participant B as token_broker
          participant S as Snowflake
          B->>S: POST /oauth/token-request<br/>Content-Type: application/x-www-form-urlencoded<br/>grant_type=authorization_code&code={code}&redirect_uri=https://token-broker.pinadmin.com/oauth/callback&client_id={sf_client_id}&client_secret={sf_client_secret}&code_verifier={code_verifier}
          S-->>B: 200 {"access_token":"{access_token}","token_type":"Bearer","expires_in":600,"refresh_token":"{refresh_token}","scope":"{scope}"}
          B->>S: POST /oauth/token-request<br/>Content-Type: application/x-www-form-urlencoded<br/>grant_type=refresh_token&refresh_token={refresh_token}&client_id={sf_client_id}&client_secret={sf_client_secret}
          S-->>B: 200 {"access_token":"{access_token}","token_type":"Bearer","expires_in":600,"refresh_token":"{refresh_token}","scope":"{scope}"}
      ```

    - **c.** Source: [`services/oauth.py` L115-L130](./OAuthFlowAndDevProblemOffline.html#method-broker-exchange-code) (code) and [L197-L208](./OAuthFlowAndDevProblemOffline.html#method-broker-refresh-access-token) (refresh). Canonical messages: [RFC 6749 §4.1.3, §5.1, and §6](./OAuthFlowAndDevProblemOffline.html#g-token-exchange).
5. <a id="g-state"></a>**OAuth state** (`{state}`, `oauth_states` row)
    - **a.** General: a random value the client adds to the authorize request; the authorization server returns it unchanged on the callback ([RFC 6749 §4.1.1](./OAuthFlowAndDevProblemOffline.html#g-oauth) and [§4.1.2](./OAuthFlowAndDevProblemOffline.html#g-authorization-code), example `state=xyz`). The client accepts only a callback whose state it issued.

      ```mermaid
      sequenceDiagram
          actor UA as User-agent
          participant C as Client
          participant AS as Authorization server
          C->>UA: 302 Location: https://server.example.com/authorize?response_type=code&client_id=s6BhdRkqt3&state=xyz&redirect_uri=https://client.example.com/cb
          UA->>AS: GET /authorize?response_type=code&client_id=s6BhdRkqt3&state=xyz&redirect_uri=https://client.example.com/cb
          AS-->>UA: 302 Location: https://client.example.com/cb?code=SplxlOBeZQQYbYS6WxSbIA&state=xyz
          UA->>C: GET /cb?code=SplxlOBeZQQYbYS6WxSbIA&state=xyz
          Note over C: accept the callback only if state is one this client issued
      ```

    - **b.** In this doc: `/oauth/start` generates `state` with `secrets.token_urlsafe(32)` and saves the row that binds the callback to a user. `/oauth/callback` consumes that row and stores the token under the saved key. The row is the only link between who started the flow and whose token gets stored. Columns are the `oauth_states` model.

      ```mermaid
      sequenceDiagram
          actor U as Browser
          participant B as token_broker
          participant DB as Token store
          participant S as Snowflake
          Note over B: state = secrets.token_urlsafe(32)
          B->>DB: INSERT oauth_states (state={state}, user_id=dmachicao, client_id={MCP SPIFFE}, mcp_server_id=snowflake, code_verifier=encrypt(code_verifier), redirect_uri=https://token-broker.pinadmin.com/oauth/callback, expires_at=now+600s)
          B-->>U: 307 Location: https://{account}.snowflakecomputing.com/oauth/authorize?response_type=code&client_id={sf_client_id}&redirect_uri=https://token-broker.pinadmin.com/oauth/callback&state={state}&scope=refresh_token&code_challenge={code_challenge}&code_challenge_method=S256
          S-->>U: 302 Location: https://token-broker.pinadmin.com/oauth/callback?code={code}&state={state}
          U->>B: GET /oauth/callback?code={code}&state={state}
          B->>DB: consume oauth_states WHERE state={state}<br/>returns user_id=dmachicao, client_id={MCP SPIFFE}, mcp_server_id=snowflake, decrypt(code_verifier), redirect_uri
      ```

    - **c.** Source: [`generate_state` L20-L21](./OAuthFlowAndDevProblemOffline.html#method-broker-generate-state), [`OAuthState` model L49-L68](./OAuthFlowAndDevProblemOffline.html#model-oauth-state), [`STATE_EXPIRY_SECONDS` L53](./OAuthFlowAndDevProblemOffline.html#module-broker-config), [`oauth_state_crud.py` L29-L75](./OAuthFlowAndDevProblemOffline.html#method-broker-create-state).
6. <a id="g-access-token"></a>**Access token** (`{access_token}`)
    - **a.** General: the credential sent with each API call to prove the caller may act for the user ([RFC 6750 §2.1](./OAuthFlowAndDevProblemOffline.html#g-bearer)). It is short-lived, and whoever holds it can use it (see [bearer](#g-bearer)). The sample token is the RFC's.

      ```mermaid
      sequenceDiagram
          participant C as Client
          participant RS as Resource server
          C->>RS: GET /resource<br/>Authorization: Bearer mF_9.B5f-4.1JqM
          RS-->>C: 200
      ```

    - **b.** In this doc: a Snowflake access token valid 600 seconds. The [MCP](#g-mcp) gets it from `/api/token` and passes it to the Snowflake connector as `authenticator=oauth` and `token`. The connector's own HTTP login body is not in this repo; the arguments below are what `SnowflakeClient` passes to `snowflake.connector.connect`. The broker response also sets `Cache-Control: no-store` and `Pragma: no-cache`. `token_type` is the value stored on the row.

      ```mermaid
      sequenceDiagram
          participant H as Helix
          participant M as MCP v2
          participant B as token_broker
          participant S as Snowflake connector
          H->>M: POST /mcp<br/>Authorization: Bearer {jwt}<br/>{"jsonrpc":"2.0","method":"tools/call","params":{"name":"execute_snowflake_query","arguments":{"query":"SELECT CURRENT_USER()"}}}
          M->>B: GET /api/token?mcp_server_id=snowflake<br/>Authorization: Bearer {jwt}
          B-->>M: 200 Cache-Control: no-store<br/>Pragma: no-cache<br/>{"status":"ok","access_token":"{access_token}","token_type":"{token_type}"}
          M->>S: connect(account={account}, database={database}, schema={schema}, authenticator=oauth, token={access_token})
      ```

    - **c.** Source: [`get_token`, `api.py` L21-L95](./OAuthFlowAndDevProblemOffline.html#method-broker-get-token), [`snowflake_client.py` L33-L41](./OAuthFlowAndDevProblemOffline.html#method-mcp-snowflake-init), [`token_broker_client.py` L49-L58](./OAuthFlowAndDevProblemOffline.html#method-mcp-get-snowflake-access-token). Canonical message: [RFC 6750 §2.1](./OAuthFlowAndDevProblemOffline.html#g-bearer).
7. <a id="g-refresh-token"></a>**Refresh token** (`{refresh_token}`, refresh)
    - **a.** General: a longer-lived credential used only at the token endpoint to get a new access token, without asking the user to sign in again ([RFC 6749 §6](./OAuthFlowAndDevProblemOffline.html#g-refresh-token)). It is never sent to the resource server. The sample values are the RFC's.

      ```mermaid
      sequenceDiagram
          participant C as Client
          participant AS as Authorization server
          C->>AS: POST /token<br/>Authorization: Basic czZCaGRSa3F0MzpnWDFmQmF0M2JW<br/>Content-Type: application/x-www-form-urlencoded<br/>grant_type=refresh_token&refresh_token=tGzv3JOkF0XG5Qx2TlKWIA
          AS-->>C: 200 Content-Type: application/json;charset=UTF-8<br/>Cache-Control: no-store<br/>Pragma: no-cache<br/>{"access_token":"2YotnFZFEjr1zCsicMWpAA","token_type":"example","expires_in":3600,"refresh_token":"tGzv3JOkF0XG5Qx2TlKWIA"}
      ```

    - **b.** In this doc: Snowflake issues it because the [scope](#g-scope) includes `refresh_token`, and it lives 7 days. `/api/token` refreshes only when the row is expired. The broker treats the row as expired 300 seconds before `expires_at`. This grant has no `code_verifier`. If Snowflake rejects the refresh, or the row has no refresh token, the broker returns `reauth_required` and no token.

      ```mermaid
      sequenceDiagram
          participant M as MCP v2
          participant B as token_broker
          participant DB as Token store
          participant S as Snowflake
          M->>B: GET /api/token?mcp_server_id=snowflake<br/>Authorization: Bearer {jwt}
          B->>DB: SELECT mcp_tokens WHERE user_id={jwt sub} AND client_id={XFCC URI} AND mcp_server_id=snowflake
          B->>S: POST /oauth/token-request<br/>Content-Type: application/x-www-form-urlencoded<br/>grant_type=refresh_token&refresh_token={refresh_token}&client_id={sf_client_id}&client_secret={sf_client_secret}
          S-->>B: 200 {"access_token":"{access_token}","token_type":"Bearer","expires_in":600,"refresh_token":"{refresh_token}","scope":"{scope}"}
          B->>DB: UPSERT mcp_tokens (access_token=encrypt({access_token}), refresh_token=encrypt({refresh_token}), token_type, expires_at, scopes)<br/>INSERT audit_logs action=refresh
          B-->>M: 200 Cache-Control: no-store<br/>Pragma: no-cache<br/>{"status":"ok","access_token":"{access_token}","token_type":"{token_type}"}
      ```

    - **c.** Source: [`api.py` L37-L71](./OAuthFlowAndDevProblemOffline.html#method-broker-get-token), [`refresh_access_token` L177-L238](./OAuthFlowAndDevProblemOffline.html#method-broker-refresh-access-token), [`is_expired` L121-L125](./OAuthFlowAndDevProblemOffline.html#method-broker-is-expired). Canonical message: [RFC 6749 §6](./OAuthFlowAndDevProblemOffline.html#g-refresh-token).
8. <a id="g-scope"></a>**Scope** (`OAUTH_ANY_ROLE_MODE`)
    - **a.** General: a space-separated list of permissions the application asks for in the authorize URL; the authorization server may grant less.
    - **b.** In this doc: Snowflake scopes. `refresh_token` asks for a [refresh token](#g-refresh-token). `session:role:<ROLE>` asks for a session in one named role. Section 5.5's target `session:role-any` would let the session use any role the user holds, and needs `OAUTH_ANY_ROLE_MODE` on the [integration](#g-security-integration). With no role scope, the session uses the [default role](#g-roles).
    - **c.** Example: `scope=refresh_token`, which is what the broker sends today.
    - **d.** Source: [`default_scopes`, `mcp_servers.production.yaml` L32](./OAuthFlowAndDevProblemOffline.html#class-broker-server-config), [`build_authorize_url` L88](./OAuthFlowAndDevProblemOffline.html#method-broker-build-authorize-url). External: [Snowflake custom OAuth clients](./OAuthFlowAndDevProblemOffline.html#g-security-integration).
9. <a id="g-redirect-uri"></a>**Redirect URI and callback** (`redirect_uri`, `OAUTH_REDIRECT_URI`)
    - **a.** General: the client URL where the authorization server sends the user-agent after consent ([RFC 6749 §3.1.2](./OAuthFlowAndDevProblemOffline.html#g-redirect-uri) and [§4.1.2](./OAuthFlowAndDevProblemOffline.html#g-authorization-code)). It must exactly equal a pre-registered value. On success the query is `code` and `state`. On denial the RFC's sample query is `error=access_denied` and `state` ([§4.1.2.1](./OAuthFlowAndDevProblemOffline.html#g-authorization-code)). `error_description` and `error_uri` are optional and are not in that sample.

      ```mermaid
      sequenceDiagram
          actor UA as User-agent
          participant C as Client
          participant AS as Authorization server
          AS-->>UA: 302 Location: https://client.example.com/cb?code=SplxlOBeZQQYbYS6WxSbIA&state=xyz
          UA->>C: GET /cb?code=SplxlOBeZQQYbYS6WxSbIA&state=xyz
          AS-->>UA: 302 Location: https://client.example.com/cb?error=access_denied&state=xyz
          UA->>C: GET /cb?error=access_denied&state=xyz
      ```

    - **b.** In this doc: `https://token-broker.pinadmin.com/oauth/callback` in production; the dev candidate in section 7 uses `https://mcp-devrestricted-dmachicao.pinterdev.com/oauth/callback`. The [broker](#g-token-broker) config and the Snowflake [integration](#g-security-integration)'s `OAUTH_REDIRECT_URI` must match. The callback carries no user identity. The handler's query parameters are `code`, `state`, `error`, and `error_description`.

      ```mermaid
      sequenceDiagram
          actor U as Browser
          participant S as Snowflake
          participant B as token_broker
          S-->>U: 302 Location: https://token-broker.pinadmin.com/oauth/callback?code={code}&state={state}
          U->>B: GET /oauth/callback?code={code}&state={state}
          S-->>U: 302 Location: https://token-broker.pinadmin.com/oauth/callback?error={error}&error_description={error_description}&state={state}
          U->>B: GET /oauth/callback?error={error}&error_description={error_description}&state={state}
      ```

    - **c.** Source: [`redirect_uri`, `mcp_servers.production.yaml` L34](./OAuthFlowAndDevProblemOffline.html#class-broker-server-config), [`_validate_redirect_uri` L33-L46](./OAuthFlowAndDevProblemOffline.html#method-broker-validate-redirect), [`oauth_callback` L75-L96](./OAuthFlowAndDevProblemOffline.html#method-broker-oauth-callback). Canonical error query: [RFC 6749 §4.1.2.1](./OAuthFlowAndDevProblemOffline.html#g-authorization-code). Snowflake's `error` values are not listed in the cited sources, so the Pinterest diagram keeps them as placeholders.
10. <a id="g-pkce"></a>**PKCE** (`code_verifier`, `code_challenge`, `S256`)
    - **a.** General: Proof Key for Code Exchange ([RFC 7636](./OAuthFlowAndDevProblemOffline.html#g-pkce)). The client creates a random `code_verifier`, sends only `BASE64URL(SHA256(code_verifier))` as `code_challenge` with `code_challenge_method=S256` on the authorize request, and sends the verifier itself on the token request. The authorization server accepts the code only when the verifier hashes to the stored challenge, so a stolen code is useless without the verifier. The values below are the ones in RFC 7636 Appendix B and the authorization-code example in RFC 6749.

      ```mermaid
      sequenceDiagram
          actor UA as User-agent
          participant C as Client
          participant AS as Authorization server
          Note over C: code_verifier = dBjftJeZ4CVP-mB92K27uhbUJU1p1r_wW1gFWFOEjXk<br/>code_challenge = BASE64URL(SHA256(ASCII(code_verifier)))<br/>= E9Melhoa2OwvFrEMTJguCHaoeK1t8URWbuGJSstw-cM<br/>code_challenge_method = S256
          C->>UA: 302 Location: https://server.example.com/authorize?response_type=code&client_id=s6BhdRkqt3&state=xyz&redirect_uri=https://client.example.com/cb&code_challenge=E9Melhoa2OwvFrEMTJguCHaoeK1t8URWbuGJSstw-cM&code_challenge_method=S256
          UA->>AS: GET /authorize?response_type=code&client_id=s6BhdRkqt3&state=xyz&redirect_uri=https://client.example.com/cb&code_challenge=E9Melhoa2OwvFrEMTJguCHaoeK1t8URWbuGJSstw-cM&code_challenge_method=S256
          Note over AS: authenticate the resource owner, obtain consent,<br/>and store code_challenge with the issued code
          AS-->>UA: 302 Location: https://client.example.com/cb?code=SplxlOBeZQQYbYS6WxSbIA&state=xyz
          UA->>C: GET /cb?code=SplxlOBeZQQYbYS6WxSbIA&state=xyz
          C->>AS: POST /token<br/>Content-Type: application/x-www-form-urlencoded<br/>grant_type=authorization_code&code=SplxlOBeZQQYbYS6WxSbIA&redirect_uri=https://client.example.com/cb&client_id=s6BhdRkqt3&code_verifier=dBjftJeZ4CVP-mB92K27uhbUJU1p1r_wW1gFWFOEjXk
          Note over AS: accept only if BASE64URL(SHA256(code_verifier))<br/>equals the stored code_challenge
          AS-->>C: 200 Content-Type: application/json<br/>Cache-Control: no-store<br/>Pragma: no-cache<br/>{"access_token":"2YotnFZFEjr1zCsicMWpAA","token_type":"example","expires_in":3600,"refresh_token":"tGzv3JOkF0XG5Qx2TlKWIA"}
      ```

    - **b.** In this doc: the Snowflake [integration](#g-security-integration) enforces it (`OAUTH_ENFORCE_PKCE = TRUE`). The [broker](#g-token-broker) is the client. It creates `code_verifier` with `secrets.token_urlsafe(64)`, stores it encrypted on the [state](#g-state) row, and sends `code_challenge_method=S256`. The token request also sends the confidential-client secret. The start query is only the two parameters the [MCP](#g-mcp) puts on the link.

      ```mermaid
      sequenceDiagram
          actor U as Browser
          participant EI as Ingress Envoy
          participant B as token_broker
          participant DB as Token store
          participant S as Snowflake
          U->>EI: GET /oauth/start?client_id={MCP SPIFFE}&mcp_server_id=snowflake
          EI->>B: GET /oauth/start?client_id={MCP SPIFFE}&mcp_server_id=snowflake<br/>X-Forwarded-User: dmachicao
          Note over B: code_verifier = secrets.token_urlsafe(64)<br/>code_challenge = BASE64URL(SHA256(code_verifier)) with padding stripped<br/>state = secrets.token_urlsafe(32)
          B->>DB: INSERT oauth_states (state={state}, user_id=dmachicao, client_id={MCP SPIFFE}, mcp_server_id=snowflake, code_verifier=encrypt(code_verifier), redirect_uri=https://token-broker.pinadmin.com/oauth/callback, expires_at=now+600s)
          B-->>U: 307 Location: https://{account}.snowflakecomputing.com/oauth/authorize?response_type=code&client_id={sf_client_id}&redirect_uri=https://token-broker.pinadmin.com/oauth/callback&state={state}&scope=refresh_token&code_challenge={code_challenge}&code_challenge_method=S256
          U->>S: GET /oauth/authorize?response_type=code&client_id={sf_client_id}&redirect_uri=https://token-broker.pinadmin.com/oauth/callback&state={state}&scope=refresh_token&code_challenge={code_challenge}&code_challenge_method=S256
          S-->>U: 302 Location: https://token-broker.pinadmin.com/oauth/callback?code={code}&state={state}
          U->>B: GET /oauth/callback?code={code}&state={state}
          B->>DB: consume oauth_states WHERE state={state}<br/>returns user_id, client_id, mcp_server_id, decrypt(code_verifier), redirect_uri
          B->>S: POST /oauth/token-request<br/>Content-Type: application/x-www-form-urlencoded<br/>grant_type=authorization_code&code={code}&redirect_uri=https://token-broker.pinadmin.com/oauth/callback&client_id={sf_client_id}&client_secret={sf_client_secret}&code_verifier={code_verifier}
          S-->>B: 200 {"access_token":"{access_token}","token_type":"Bearer","expires_in":600,"refresh_token":"{refresh_token}","scope":"{scope}"}
      ```

    - **c.** Source: [`services/oauth.py` L24-L30](./OAuthFlowAndDevProblemOffline.html#method-broker-generate-state), [L62-L67](./OAuthFlowAndDevProblemOffline.html#method-broker-build-authorize-url), [L83-L98](./OAuthFlowAndDevProblemOffline.html#method-broker-build-authorize-url), [L115-L130](./OAuthFlowAndDevProblemOffline.html#method-broker-exchange-code); verifier encryption in [`oauth_state_crud.py` L13-L22](./OAuthFlowAndDevProblemOffline.html#method-broker-create-state). Canonical messages: [RFC 7636](./OAuthFlowAndDevProblemOffline.html#g-pkce) Appendix B and §4, token success body from [RFC 6749 §5.1](./OAuthFlowAndDevProblemOffline.html#g-token-exchange). The Snowflake response keys are the ones the broker reads (`access_token`, `refresh_token`, `token_type`, `expires_in`, `scope`); a key Snowflake omits is stored as empty.
11. <a id="g-confidential-client"></a>**Confidential client, client ID, client secret** (`{sf_client_id}`, `{sf_client_secret}`)
    - **a.** General: an OAuth client that runs on a server and can keep a secret ([RFC 6749 §2.1](./OAuthFlowAndDevProblemOffline.html#g-confidential-client)). The canonical way to present the secret at the token endpoint is HTTP Basic, `base64(client_id:client_secret)` ([§2.3.1](./OAuthFlowAndDevProblemOffline.html#g-confidential-client)). A public client cannot keep a secret and sends only `client_id`.

      ```mermaid
      sequenceDiagram
          participant C as Confidential client
          participant AS as Authorization server
          C->>AS: POST /token<br/>Authorization: Basic czZCaGRSa3F0MzpnWDFmQmF0M2JW<br/>Content-Type: application/x-www-form-urlencoded<br/>grant_type=authorization_code&code=SplxlOBeZQQYbYS6WxSbIA&redirect_uri=https://client.example.com/cb
          AS-->>C: 200 Content-Type: application/json;charset=UTF-8<br/>Cache-Control: no-store<br/>Pragma: no-cache<br/>{"access_token":"2YotnFZFEjr1zCsicMWpAA","token_type":"example","expires_in":3600,"refresh_token":"tGzv3JOkF0XG5Qx2TlKWIA"}
      ```

    - **b.** In this doc: the [broker](#g-token-broker) is the confidential client registered in the Snowflake [security integration](#g-security-integration). It sends `{sf_client_id}` and `{sf_client_secret}` as form fields, not as HTTP Basic. In production those values come from the [Knox](#g-knox) key `token_broker:snowflake:production:client_credentials`. Do not confuse `{sf_client_id}` with the broker's own `client_id` parameter, which is the [partition](#g-partition) (a [SPIFFE ID](#g-spiffe)).

      ```mermaid
      sequenceDiagram
          participant B as token_broker
          participant S as Snowflake
          B->>S: POST /oauth/token-request<br/>Content-Type: application/x-www-form-urlencoded<br/>grant_type=authorization_code&code={code}&redirect_uri=https://token-broker.pinadmin.com/oauth/callback&client_id={sf_client_id}&client_secret={sf_client_secret}&code_verifier={code_verifier}
          S-->>B: 200 {"access_token":"{access_token}","token_type":"Bearer","expires_in":600,"refresh_token":"{refresh_token}","scope":"{scope}"}
      ```

    - **c.** Source: [`_fetch_knox_credentials`, `config.py` L38-L54](./OAuthFlowAndDevProblemOffline.html#method-broker-fetch-knox-credentials), [`secret_key`, `mcp_servers.production.yaml` L37](./OAuthFlowAndDevProblemOffline.html#class-broker-server-config), body fields [`services/oauth.py` L115-L124](./OAuthFlowAndDevProblemOffline.html#method-broker-exchange-code). Canonical header: [RFC 6749 §2.3.1](./OAuthFlowAndDevProblemOffline.html#g-confidential-client).
12. <a id="g-revocation"></a>**Revocation and unlink**
    - **a.** General: the client asks the authorization server to invalidate a token before it expires ([RFC 7009](./OAuthFlowAndDevProblemOffline.html#g-revocation)). The body is `token` and an optional `token_type_hint`. A successful response is 200 with an empty body. The sample values are the RFC's.

      ```mermaid
      sequenceDiagram
          participant C as Client
          participant AS as Authorization server
          C->>AS: POST /revoke<br/>Authorization: Basic czZCaGRSa3F0MzpnWDFmQmF0M2JW<br/>Content-Type: application/x-www-form-urlencoded<br/>token=45ghiukldjahdnhzdauz&token_type_hint=refresh_token
          AS-->>C: 200
      ```

    - **b.** In this doc: `POST /api/unlink` takes `client_id` and `mcp_server_id` from the query and the user from the [JWT](#g-jwt). The broker decrypts the stored access token, POSTs it to Snowflake `/oauth/revoke` with `token_type_hint=access_token` plus the client id and secret, and deletes the row only when Snowflake returns 200 or 204. After a dev broker restart the token cannot be decrypted, so unlink fails with 502. The unlink response body is `status`, `user_id`, `client_id`, and `mcp_server_id`.

      ```mermaid
      sequenceDiagram
          participant M as MCP v2
          participant B as token_broker
          participant S as Snowflake
          participant DB as Token store
          M->>B: POST /api/unlink?client_id={partition}&mcp_server_id=snowflake<br/>Authorization: Bearer {jwt}
          B->>S: POST /oauth/revoke<br/>Content-Type: application/x-www-form-urlencoded<br/>token={access_token}&token_type_hint=access_token&client_id={sf_client_id}&client_secret={sf_client_secret}
          S-->>B: 200
          B->>DB: DELETE mcp_tokens WHERE user_id={jwt sub} AND client_id={partition} AND mcp_server_id=snowflake<br/>INSERT audit_logs action=unlink
          B-->>M: 200 {"status":"unlinked","user_id":"{jwt sub}","client_id":"{partition}","mcp_server_id":"snowflake"}
      ```

    - **c.** Source: [`unlink`, `api.py` L128-L170](./OAuthFlowAndDevProblemOffline.html#method-broker-unlink), [`revoke_token_at_server` L270-L284](./OAuthFlowAndDevProblemOffline.html#method-broker-revoke-token). Canonical message: [RFC 7009 §2.1](./OAuthFlowAndDevProblemOffline.html#g-revocation).
### 10.2 Identity and tokens

13. <a id="g-jwt"></a>**JWT** (JSON Web Token)
    - **a.** General: a signed token format ([RFC 7519](./OAuthFlowAndDevProblemOffline.html#g-jwt)) made of three base64url parts joined by dots, `header.payload.signature`. The payload is JSON [claims](#g-claims) about a subject. The issuer signs it with a private key, and anyone with the matching public key can check that the claims were not changed. It is signed, not encrypted: anyone holding it can read the claims. RFC 7519 defines the token, not a request sequence.
    - **b.** In this doc: the Pinterest user JWT issued by `auth.pinadmin.com` and signed with RS256. [Helix](#g-helix) forwards it to the [MCP](#g-mcp) (`forward_jwt: true`). The MCP forwards only that header to the [broker](#g-token-broker). The broker checks the signature and uses `sub` as `user_id`. The decoded payload in the old example was illustrative; the fields the broker reads are `sub` and the signature. It does not check `aud`.

      ```mermaid
      sequenceDiagram
          participant H as Helix
          participant M as MCP v2
          participant B as token_broker
          H->>M: POST /mcp<br/>Authorization: Bearer {jwt}<br/>{"jsonrpc":"2.0","method":"tools/call","params":{"name":"execute_snowflake_query","arguments":{"query":"SELECT CURRENT_USER()"}}}
          M->>B: GET /api/token?mcp_server_id=snowflake<br/>Authorization: Bearer {jwt}
          Note over B: verify RS256 signature with auth.pinadmin.com public key<br/>user_id = payload.sub<br/>verify_aud = false
      ```

    - **c.** Source: [`jwt.py` L17-L62](./OAuthFlowAndDevProblemOffline.html#method-broker-jwt-decode) (public key fetch and signature check), [L65-L103](./OAuthFlowAndDevProblemOffline.html#method-broker-get-current-user) (`get_current_user`), forwarded header [`token_broker_client.py` L39-L57](./OAuthFlowAndDevProblemOffline.html#method-mcp-get-snowflake-access-token).
14. <a id="g-claims"></a>**Claim, `sub`, audience (`aud`)**
    - **a.** General: a claim is one key/value pair in a [JWT](#g-jwt) payload. `sub` (subject) names who the token is about, `aud` (audience) names the service the token was issued for, and `exp` is its expiry time. A service that checks `aud` refuses tokens minted for other services.
    - **b.** In this doc: the [broker](#g-token-broker) reads `sub` as the user (`dmachicao`) and does not check `aud` (`verify_aud: False`), so any valid Pinterest user JWT is accepted.
    - **c.** Source: [`jwt.py` L53](./OAuthFlowAndDevProblemOffline.html#method-broker-jwt-decode), [L96-L103](./OAuthFlowAndDevProblemOffline.html#method-broker-get-current-user).
15. <a id="g-bearer"></a>**Bearer token, `Authorization` header**
    - **a.** General: the HTTP header `Authorization: Bearer <token>` ([RFC 6750 §2.1](./OAuthFlowAndDevProblemOffline.html#g-bearer)). "Bearer" means possession is proof: whoever sends the token is treated as its owner. The sample token is the RFC's.

      ```mermaid
      sequenceDiagram
          participant C as Client
          participant RS as Resource server
          C->>RS: GET /resource<br/>Authorization: Bearer mF_9.B5f-4.1JqM
      ```

    - **b.** In this doc: carries the Pinterest [JWT](#g-jwt) from [Helix](#g-helix) to the [MCP](#g-mcp) and from the MCP to the [broker](#g-token-broker). The MCP copies that one header and sends no other header on `/api/token`. The header probe in section 7 must never log it.

      ```mermaid
      sequenceDiagram
          participant H as Helix
          participant M as MCP v2
          participant B as token_broker
          H->>M: POST /mcp<br/>Authorization: Bearer {jwt}
          M->>B: GET /api/token?mcp_server_id=snowflake<br/>Authorization: Bearer {jwt}
      ```

    - **c.** Source: [`token_broker_client.py` L39-L58](./OAuthFlowAndDevProblemOffline.html#method-mcp-get-snowflake-access-token). Canonical header: [RFC 6750 §2.1](./OAuthFlowAndDevProblemOffline.html#g-bearer).
16. <a id="g-sso"></a>**SSO** (single sign-on)
    - **a.** General: one login session reused across many sites. There is no single canonical message sequence; SAML, OIDC, and vendor filters differ. The shape this document relies on is: no session, the site redirects the browser to a login page; after login, the browser returns with a session cookie and the site accepts the original request.

      ```mermaid
      sequenceDiagram
          actor U as Browser
          participant App as Site
          participant IdP as Login page
          U->>App: GET /resource
          App-->>U: 302 Location: {login page}
          U->>IdP: authenticate
          IdP-->>U: 302 Location: /resource<br/>Set-Cookie: {session}
          U->>App: GET /resource<br/>Cookie: {session}
          App-->>U: 200
      ```

    - **b.** In this doc: Pinterest SSO at `auth.pinadmin.com`, backed by [Okta](#g-okta). [Envoy](#g-envoy)'s [`internal_oauth` filter](#g-internal-oauth) enforces it on some hosts (`token-broker.pinadmin.com`, devapp `mcp-*` hosts) and then sets [`X-Forwarded-User`](#g-x-forwarded-user). Devapp `www-*` and `api-*` hosts do not enforce it. The query string on `auth.pinadmin.com/oauth/authorize/` is not in the sources this document cites, so the 302 below keeps the `?` the Phase B diagram uses and does not invent parameters. The session cookie name is likewise not in those sources.

      ```mermaid
      sequenceDiagram
          actor U as Browser
          participant EI as Ingress Envoy
          participant A as auth.pinadmin.com
          participant B as token_broker
          U->>EI: GET /oauth/start?client_id={MCP SPIFFE}&mcp_server_id=snowflake
          EI-->>U: 302 Location: https://auth.pinadmin.com/oauth/authorize/?...
          U->>A: authenticate
          A-->>U: 302 Location: /oauth/start?client_id={MCP SPIFFE}&mcp_server_id=snowflake
          U->>EI: GET /oauth/start?client_id={MCP SPIFFE}&mcp_server_id=snowflake<br/>Cookie: {session}
          EI->>B: GET /oauth/start?client_id={MCP SPIFFE}&mcp_server_id=snowflake<br/>X-Forwarded-User: dmachicao
      ```

    - **c.** Source: the broker's production Envoy config, SSO on and off for `/oauth/callback` and `/health` ([pinconf `token-broker/prod/main/http.yaml` L8-21](./OAuthFlowAndDevProblemOffline.html#element-ingress-internal-oauth)); the 302 target is the one drawn in section 4.2.
17. <a id="g-okta"></a>**Okta**
    - **a.** General: a commercial identity provider that stores user accounts and handles passwords and multi-factor sign-in.
    - **b.** In this doc: the login behind Pinterest [SSO](#g-sso) and behind the Snowflake sign-in page in Phase B.
18. <a id="g-ldap"></a>**LDAP username**
    - **a.** General: LDAP is a protocol for corporate user directories; an LDAP username is an employee's login name in that directory.
    - **b.** In this doc: the value that must appear both as [JWT](#g-jwt) `sub` and as [`X-Forwarded-User`](#g-x-forwarded-user) for the two halves of the token row key to match.
    - **c.** Example: `dmachicao`.
19. <a id="g-spiffe"></a>**SPIFFE ID, workload identity**
    - **a.** General: SPIFFE ([spiffe.io](./OAuthFlowAndDevProblemOffline.html#g-spiffe)) is an open standard for identifying software rather than people. A SPIFFE ID is a URI, `spiffe://<trust-domain>/<path>`, placed in the certificate a workload (a service, job, or machine) presents during [mTLS](#g-mtls).
    - **b.** In this doc: identifies which service called the [broker](#g-token-broker). The broker takes the caller's SPIFFE ID from [XFCC](#g-xfcc) and uses it as `client_id`, the [partition](#g-partition). The three IDs that matter are listed in "How to read it" item 3; the `ingress SPIFFE` is what the broker sees whenever a request arrives through the public [ingress](#g-ingress).
    - **c.** Example: `spiffe://pin220.com/devapp/restricted/dmachicao` identifies a devapp; `spiffe://svc.pin220.com/mcp-server-snowflake-v2/dev/main` is the `client_id` in the MCP dev config.
    - **d.** Source: [`spiffe.py` L16-L58](./OAuthFlowAndDevProblemOffline.html#method-broker-parse-spiffe), [`token_broker` block, `runtime.dev.yaml` L216-L220](./OAuthFlowAndDevProblemOffline.html#module-mcp-runtime-config).
20. <a id="g-partition"></a>**Partition**
    - **a.** General: a slice of stored data kept apart from other slices by part of its key.
    - **b.** In this doc: the `client_id` part of the [broker](#g-token-broker)'s token row key `(user_id, client_id, mcp_server_id)`. It keeps tokens separate per calling agent, so one agent cannot fetch tokens linked for another. `/api/token` reads it from [XFCC](#g-xfcc); `/oauth/start` takes it from the link's `?client_id=`. See [section 2](#2-the-one-idea-that-explains-everything-the-token-row-key).
    - **c.** Source: [`MCPToken` unique key, `models.py` L13-L23](./OAuthFlowAndDevProblemOffline.html#model-mcp-token), [`get_token` inputs, `api.py` L21-L29](./OAuthFlowAndDevProblemOffline.html#method-broker-get-token).

### 10.3 Network and infrastructure

21. <a id="g-tls"></a>**TLS**
    - **a.** General: the encryption protocol under HTTPS. The server proves its identity with a certificate and the traffic is encrypted. "Terminating TLS" means a proxy decrypts the traffic and passes plain HTTP to the service behind it.
    - **b.** In this doc: [Envoy](#g-envoy) terminates TLS for the public `*.pinadmin.com` and `*.pinterdev.com` hosts.
    - **c.** Example: any `https://` URL.
22. <a id="g-mtls"></a>**mTLS** (mutual TLS)
    - **a.** General: [TLS](#g-tls) in which the client also presents a certificate, so the server learns cryptographically which client connected. The handshake records themselves are not drawn here; the message this document depends on is the certificate the client presents and the HTTP request that follows.

      ```mermaid
      sequenceDiagram
          participant C as Client
          participant S as Server
          C->>S: TLS client certificate, SAN URI = spiffe://example.com/client
          C->>S: GET /resource
          Note over S: the peer identity is the URI in that certificate
      ```

    - **b.** In this doc: service-to-service calls inside the [mesh](#g-mesh) use mTLS. The client certificate carries the caller's [SPIFFE ID](#g-spiffe), and the [broker](#g-token-broker)'s [sidecar](#g-sidecar) turns it into the [XFCC](#g-xfcc) header. A browser has no client certificate, so `/oauth/start` has no mTLS identity. Semicolons in the header are written `#59;` so Mermaid keeps them.

      ```mermaid
      sequenceDiagram
          participant ME as MCP Envoy
          participant BE as Broker Envoy
          participant B as token_broker
          ME->>BE: mTLS, client certificate URI={MCP SPIFFE}
          ME->>BE: GET /api/token?mcp_server_id=snowflake<br/>Authorization: Bearer {jwt}
          BE->>B: GET /api/token?mcp_server_id=snowflake<br/>Authorization: Bearer {jwt}<br/>X-Forwarded-Client-Cert: By={broker SPIFFE}#59;URI={MCP SPIFFE}
      ```

23. <a id="g-envoy"></a>**Envoy**
    - **a.** General: an open-source network proxy ([envoyproxy.io](./OAuthFlowAndDevProblemOffline.html#g-envoy)) placed in front of services to route requests, handle TLS, enforce authentication, and add or remove headers.
    - **b.** In this doc: runs in two places. The [ingress](#g-ingress) Envoy receives public hostnames and enforces [SSO](#g-sso) on some of them; the [sidecar](#g-sidecar) Envoy next to the [broker](#g-token-broker) handles [mTLS](#g-mtls) and sets [XFCC](#g-xfcc). The broker trusts [`X-Forwarded-User`](#g-x-forwarded-user) and XFCC because only Envoy should set them.
    - **c.** Source: the broker's assumptions about Envoy in [`spiffe.py` L1-L7](./OAuthFlowAndDevProblemOffline.html#method-broker-parse-spiffe) and [`routes/oauth.py` L35-L44](./OAuthFlowAndDevProblemOffline.html#method-broker-oauth-start); the broker's own Envoy config in [pinconf](#g-pinconf) ([`token-broker/prod/main/http.yaml`](./OAuthFlowAndDevProblemOffline.html#element-ingress-internal-oauth)).
24. <a id="g-ingress"></a>**Ingress**
    - **a.** General: the entry point that accepts traffic from outside a cluster or network and routes it to internal services, usually by hostname.
    - **b.** In this doc: `ingress-pinadmin`, an [Envoy](#g-envoy) fleet that serves `*.pinadmin.com` and devapp `*.pinterdev.com` hosts. It has its own [SPIFFE ID](#g-spiffe) (`ingress SPIFFE`). The fleet is a [Teletraan](#g-teletraan) service, which is why that SPIFFE contains `teletraan` rather than `svc.pin220.com`. When a request reaches the [broker](#g-token-broker) through it, [XFCC](#g-xfcc) names the ingress instead of the original caller, and on hosts without [SSO](#g-sso) the broker also receives the ingress SPIFFE ID as [`X-Forwarded-User`](#g-x-forwarded-user) (trace 80).
    - **c.** Example: `https://www-devrestricted-dmachicao.pinterdev.com` goes through the ingress to port 10001 on devapp 1.
25. <a id="g-teletraan"></a>**Teletraan**
    - **a.** General: Pinterest's deployment system. A service deployed there carries `teletraan` in its [SPIFFE ID](#g-spiffe).
    - **b.** In this doc: only the [ingress](#g-ingress). Its SPIFFE is `spiffe://pin220.com/teletraan/ingress-pinadmin/prod-use1`. Services on [PinCompute](#g-pincompute) use `spiffe://svc.pin220.com/<service>/<env>/main` instead.
    - **c.** Source: ingress config path in [`oauth-mcp-server-pastis-traffic-pinconf.md` L67](./OAuthFlowAndDevProblemOffline.html#actor-ingress).
26. <a id="g-sidecar"></a>**Sidecar**
    - **a.** General: a helper process deployed next to each instance of a service, in the same [pod](#g-pod), that handles networking so the service code does not have to.
    - **b.** In this doc: "Envoy (broker sidecar)" is the [Envoy](#g-envoy) that accepts [mTLS](#g-mtls) calls for the [broker](#g-token-broker) and adds [XFCC](#g-xfcc).
27. <a id="g-mesh"></a>**Service mesh**
    - **a.** General: the set of [sidecar](#g-sidecar) proxies and their control plane that give every service an identity, encrypt service-to-service traffic with [mTLS](#g-mtls), and route it.
    - **b.** In this doc: "over the mesh" means the [MCP](#g-mcp) calls the [broker](#g-token-broker)'s internal address directly through sidecars, so [XFCC](#g-xfcc) carries the MCP's [SPIFFE ID](#g-spiffe). On that path the receiving sidecar checks a [Pastis](#g-pastis) policy before the request reaches the app. The alternative, calling a public host, goes through the [ingress](#g-ingress), which replaces the caller identity with its own.
28. <a id="g-pastis"></a>**Pastis** (Rego)
    - **a.** General: Pinterest's authorization system. Policies are written in [Rego](./OAuthFlowAndDevProblemOffline.html#g-pastis), the policy language of Open Policy Agent. A policy allows or denies a request from the caller's identity and the URL path.
    - **b.** In this doc: the receiving [Envoy](#g-envoy) checks the caller's [SPIFFE ID](#g-spiffe) against the service's Pastis policy before the request reaches the app. The [broker](#g-token-broker) policy allows `/api/*` only from the helix-slackbot SPIFFE IDs and lets any caller reach `/oauth/*`. [Devapps](#g-devapp) have no [sidecar](#g-sidecar), so nothing enforces Pastis there. Snowflake MCP v2 on [PinCompute](#g-pincompute) would need a policy; [pinconf](#g-pinconf) has none for it yet, though the server declares the URL it will use.
    - **c.** Source: broker policy ([pinconf `token-broker/pastis/base.rego` L8-32](./OAuthFlowAndDevProblemOffline.html#element-mesh-pastis)); MCP servers on PinCompute ([`policy.md` L5](./OAuthFlowAndDevProblemOffline.html#element-mesh-pastis)); the URL v2 declares ([`server.py` L30](./OAuthFlowAndDevProblemOffline.html#module-mcp-server)).
29. <a id="g-xfcc"></a>**XFCC** (`X-Forwarded-Client-Cert`)
    - **a.** General: an HTTP header [Envoy](#g-envoy) adds to describe the client certificate of an [mTLS](#g-mtls) connection ([Envoy docs](./OAuthFlowAndDevProblemOffline.html#g-xfcc)). Each proxy hop adds one element, separated by commas; fields inside an element are separated by semicolons. `By=` is the receiving proxy's identity and `URI=` the client's [SPIFFE ID](#g-spiffe). The leftmost element is the original caller. The broker reads only `URI=`. Other fields Envoy may add (`Hash`, `Cert`, `Subject`, `DNS`, `Chain`) are not in the sources this document uses, so they are not drawn. Semicolons are `#59;` so Mermaid keeps them.

      ```mermaid
      sequenceDiagram
          participant C as Calling proxy
          participant P as Receiving proxy
          participant App as App
          C->>P: mTLS, client certificate URI=spiffe://example.com/client
          P->>App: GET /resource<br/>X-Forwarded-Client-Cert: By=spiffe://example.com/receiver#59;URI=spiffe://example.com/client
          Note over App: leftmost element, field URI=
      ```

    - **b.** In this doc: `/api/token` takes the leftmost `URI=` as `client_id` (the [partition](#g-partition)) and rejects requests without the header. Writing the header by hand over [loopback](#g-loopback) would fake the mesh ("spoofing").

      ```mermaid
      sequenceDiagram
          participant ME as MCP Envoy
          participant BE as Broker Envoy
          participant B as token_broker
          ME->>BE: GET /api/token?mcp_server_id=snowflake<br/>Authorization: Bearer {jwt}
          BE->>B: GET /api/token?mcp_server_id=snowflake<br/>Authorization: Bearer {jwt}<br/>X-Forwarded-Client-Cert: By={broker SPIFFE}#59;URI={MCP SPIFFE}
          Note over B: client_id = leftmost URI= = {MCP SPIFFE}
      ```

    - **c.** Source: [`spiffe.py` L23-L58](./OAuthFlowAndDevProblemOffline.html#method-broker-parse-spiffe), used by [`api.py` L24](./OAuthFlowAndDevProblemOffline.html#method-broker-get-token).
30. <a id="g-x-forwarded-user"></a>**`X-Forwarded-User`, `x-pinterest-forwarded-user`**
    - **a.** General: `X-Forwarded-*` headers are added by proxies to tell the service behind them about the original request. `X-Forwarded-User` conventionally holds the authenticated user. It is safe only when the proxy removes any value the client sent. There is no standard message sequence beyond that.

      ```mermaid
      sequenceDiagram
          actor U as Browser
          participant P as Proxy
          participant App as App
          U->>P: GET /resource<br/>X-Forwarded-User: attacker
          P->>App: GET /resource<br/>X-Forwarded-User: alice
          Note over P: the proxy overwrites the client-supplied value
      ```

    - **b.** In this doc: [`internal_oauth`](#g-internal-oauth) sets it to the [LDAP username](#g-ldap) after [SSO](#g-sso). `/oauth/start` uses it as `user_id` and ignores `?user_id=` whenever the header is present. On devapp hosts without SSO it holds the [ingress](#g-ingress) [SPIFFE ID](#g-spiffe). The shared MCP OAuth library also accepts `x-pinterest-forwarded-user`, which [Helix](#g-helix) sets itself on calls without a user JWT (scheduled chats), and ignores any value that starts with `spiffe://`; `token_broker` reads only `X-Forwarded-User`.

      ```mermaid
      sequenceDiagram
          actor U as Browser
          participant EI as Ingress Envoy
          participant B as token_broker
          U->>EI: GET /oauth/start?user_id=someone-else&client_id={MCP SPIFFE}&mcp_server_id=snowflake
          EI->>B: GET /oauth/start?user_id=someone-else&client_id={MCP SPIFFE}&mcp_server_id=snowflake<br/>X-Forwarded-User: dmachicao
          Note over B: user_id = X-Forwarded-User = dmachicao<br/>the query user_id is ignored
      ```

    - **c.** Source: [`routes/oauth.py` L32](./OAuthFlowAndDevProblemOffline.html#method-broker-oauth-start) and [L45](./OAuthFlowAndDevProblemOffline.html#method-broker-oauth-start); shared library [`authorizer.py` L32-L60](./OAuthFlowAndDevProblemOffline.html#method-mcp-authorization); Helix [prompthub `server.py` L139-147](./OAuthFlowAndDevProblemOffline.html#actor-helix); who sets `x-forwarded-user` [pinconf `docs/adding-internal-web-service.md` L225-236](./OAuthFlowAndDevProblemOffline.html#actor-ingress).
31. <a id="g-internal-oauth"></a>**`internal_oauth` filter**
    - **a.** General: an Envoy filter is a plug-in step that every request passes through inside [Envoy](#g-envoy). There is no canonical sequence outside Pinterest.
    - **b.** In this doc: Pinterest's Envoy filter that enforces Pinterest [SSO](#g-sso) on hosts configured for it and sets [`X-Forwarded-User`](#g-x-forwarded-user) for browser requests. Despite the name, it is SSO in front of a host, not the Snowflake [OAuth](#g-oauth) flow. It is configured per service in that service's [pinconf](#g-pinconf) Envoy config, not in `token_broker`. The broker's config names its own client secret in [Knox](#g-knox) and turns the filter off for `/oauth/callback` and `/health`. The `auth.pinadmin.com` query string is not in the cited sources.

      ```mermaid
      sequenceDiagram
          actor U as Browser
          participant EI as Ingress Envoy
          participant B as token_broker
          U->>EI: GET /oauth/start?client_id={MCP SPIFFE}&mcp_server_id=snowflake
          EI-->>U: 302 Location: https://auth.pinadmin.com/oauth/authorize/?...
          U->>EI: GET /oauth/start?client_id={MCP SPIFFE}&mcp_server_id=snowflake<br/>Cookie: {session}
          EI->>B: GET /oauth/start?client_id={MCP SPIFFE}&mcp_server_id=snowflake<br/>X-Forwarded-User: dmachicao
          U->>EI: GET /oauth/callback?code={code}&state={state}
          EI->>B: GET /oauth/callback?code={code}&state={state}
          Note over EI: /oauth/callback and /health skip the filter
      ```

    - **c.** Source: [pinconf `token-broker/prod/main/http.yaml` L8-21](./OAuthFlowAndDevProblemOffline.html#element-ingress-internal-oauth), [pinconf `docs/adding-internal-web-service.md` L225-236](./OAuthFlowAndDevProblemOffline.html#actor-ingress), and the broker's description in [`routes/oauth.py` L37-L43](./OAuthFlowAndDevProblemOffline.html#method-broker-oauth-start).
32. <a id="g-http-redirect"></a>**HTTP redirect (302, 307)**
    - **a.** General: a response telling the browser to go to the URL in its `Location` header ([RFC 9110](./OAuthFlowAndDevProblemOffline.html#g-http-redirect)). 302 ("Found") and 307 ("Temporary Redirect") are both temporary. 307 keeps the original HTTP method; 302 historically lets the client change a POST into a GET.

      ```mermaid
      sequenceDiagram
          actor U as Browser
          participant S as Server
          S-->>U: 302 Location: https://example.com/next
          U->>S: GET /next
          S-->>U: 307 Location: https://example.com/next
          U->>S: GET /next
      ```

    - **b.** In this doc: the ingress answers 302 when there is no [SSO](#g-sso) session. `/oauth/start` answers 307 to Snowflake's authorize URL, and that 307 carries the full authorize query from [PKCE](#g-pkce). FastAPI's `RedirectResponse` default status is 307.

      ```mermaid
      sequenceDiagram
          actor U as Browser
          participant EI as Ingress Envoy
          participant B as token_broker
          U->>EI: GET /oauth/start?client_id={MCP SPIFFE}&mcp_server_id=snowflake
          EI-->>U: 302 Location: https://auth.pinadmin.com/oauth/authorize/?...
          B-->>U: 307 Location: https://{account}.snowflakecomputing.com/oauth/authorize?response_type=code&client_id={sf_client_id}&redirect_uri=https://token-broker.pinadmin.com/oauth/callback&state={state}&scope=refresh_token&code_challenge={code_challenge}&code_challenge_method=S256
      ```

    - **c.** Source: [`RedirectResponse`, `routes/oauth.py` L61](./OAuthFlowAndDevProblemOffline.html#method-broker-oauth-start); authorize query [`services/oauth.py` L83-L93](./OAuthFlowAndDevProblemOffline.html#method-broker-build-authorize-url).
33. <a id="g-no-store"></a>**`Cache-Control: no-store`**
    - **a.** General: an HTTP response header that forbids browsers and proxies from saving the response.
    - **b.** In this doc: the [broker](#g-token-broker) sets it, with `Pragma: no-cache`, on every response that contains an [access token](#g-access-token).
    - **c.** Source: [`api.py` L18](./OAuthFlowAndDevProblemOffline.html#method-broker-get-token).
34. <a id="g-loopback"></a>**Loopback** (`127.0.0.1`, `localhost`)
    - **a.** General: the network address that means "this same machine". Traffic to it never leaves the host, so it passes through no proxy.
    - **b.** In this doc: a call to `127.0.0.1:10001` on a devapp skips [Envoy](#g-envoy), so it carries no [`X-Forwarded-User`](#g-x-forwarded-user) and no [XFCC](#g-xfcc). That is why the section 6.3 trick worked (the user fell back to `?user_id=`) and why an MCP-to-broker loopback call fails (`/api/token` requires XFCC).
    - **c.** Example: `http://127.0.0.1:10001/oauth/start?user_id=dmachicao&...`.
35. <a id="g-reverse-proxy"></a>**Reverse proxy**
    - **a.** General: a server that receives requests and forwards each one to one of several backend services, for example by URL path. nginx, Caddy, HAProxy, and [Envoy](#g-envoy) are common ones.
    - **b.** In this doc: option B in section 8.5, a small dev-only script on port 8000 that sends `/mcp` to the [MCP](#g-mcp) and `/oauth/*`, `/api/*`, `/health*` to the [broker](#g-token-broker).
36. <a id="g-pod"></a>**Pod**
    - **a.** General: the smallest unit Kubernetes runs, one or more containers that share a network address.
    - **b.** In this doc: "MCP pod" is a production instance of Snowflake [MCP](#g-mcp) v2 on [PinCompute](#g-pincompute); its [Envoy](#g-envoy) [sidecar](#g-sidecar) runs in the same pod.
37. <a id="g-pincompute"></a>**PinCompute, `PinApp`**
    - **a.** General: Pinterest's Kubernetes platform. A `PinApp` is the file that describes one service's [pods](#g-pod): container image, replica count, [sidecars](#g-sidecar), and service identity. Those files live under `pindeploy/`.
    - **b.** In this doc: production [MCP](#g-mcp) servers and the [broker](#g-token-broker) run as PinCompute pods. A [devapp](#g-devapp) is an EC2 machine with no PinCompute, so it has no pod and no [mesh](#g-mesh). Snowflake MCP v2 has no `pindeploy/` directory yet, so it has no production PinApp; the gong server's PinApp is the layout it would follow.
    - **c.** Example: `kind: PinApp`, name `token-broker-prod`, namespace `token-broker`, with [Envoy](#g-envoy) and [Knox](#g-knox) sidecars.
    - **d.** Source: [`token-broker-prod.yaml` L1-L16](./OAuthFlowAndDevProblemOffline.html#element-runtime-pincompute), gong PinApp ([`mcp-server-gong-staging.yaml` L7-L14](./OAuthFlowAndDevProblemOffline.html#element-runtime-pincompute)).

### 10.4 Pinterest services and tools

38. <a id="g-mcp"></a>**MCP, Snowflake MCP v2 (`snowflake_v2`), tool call**
    - **a.** General: the Model Context Protocol ([modelcontextprotocol.io](./OAuthFlowAndDevProblemOffline.html#g-mcp)) is an open protocol that lets AI assistants call external "tools" hosted by an MCP server. Each tool call is a [JSON-RPC](#g-json-rpc) request. The canonical JSON-RPC request includes `jsonrpc`, `method`, `params`, and `id`.

      ```mermaid
      sequenceDiagram
          participant Client as MCP client
          participant Server as MCP server
          Client->>Server: POST /mcp<br/>{"jsonrpc":"2.0","id":1,"method":"tools/call","params":{"name":"execute_snowflake_query","arguments":{"query":"SELECT CURRENT_USER()"}}}
          Server-->>Client: {"jsonrpc":"2.0","id":1,"result":{}}
      ```

    - **b.** In this doc: Snowflake MCP v2, code in `snowflake_v2`, is our MCP server. It exposes `execute_snowflake_query`, `describe_table_view`, and `search_tables_views`. Snowflake MCP v1 (`servers/snowflake/`) is separate and not changed. Helix's request in the cited example has no `id`. The not-linked tool result is the object `execute_snowflake_query` returns: `results`, `status`, `message`, and `auth_start_url`. The link query is `client_id` and `mcp_server_id` only.

      ```mermaid
      sequenceDiagram
          participant H as Helix
          participant M as MCP v2
          H->>M: POST /mcp<br/>Authorization: Bearer {jwt}<br/>{"jsonrpc":"2.0","method":"tools/call","params":{"name":"execute_snowflake_query","arguments":{"query":"SELECT CURRENT_USER()"}}}
          M-->>H: {"results":null,"status":"auth_required","message":"User has not linked their account","auth_start_url":"https://{broker host}/oauth/start?client_id={config client_id}&mcp_server_id=snowflake"}
      ```

    - **c.** Source: [`manifest.toml` L18-L37](./OAuthFlowAndDevProblemOffline.html#module-mcp-server), [`execute_snowflake_query.py` L39-L97](./OAuthFlowAndDevProblemOffline.html#method-mcp-execute-snowflake-query), [`token_broker_client.py` L74-L85](./OAuthFlowAndDevProblemOffline.html#method-mcp-get-snowflake-access-token), [`runtime.dev.yaml` L216-L220](./OAuthFlowAndDevProblemOffline.html#module-mcp-runtime-config).
39. <a id="g-json-rpc"></a>**JSON-RPC, `tools/call`**
    - **a.** General: a remote-call format ([JSON-RPC 2.0](./OAuthFlowAndDevProblemOffline.html#g-json-rpc)). A request object has `jsonrpc`, `method`, `params`, and `id`. A success response has `jsonrpc`, `result`, and the same `id`. The sample below is the specification's own request shape with this document's method name.

      ```mermaid
      sequenceDiagram
          participant C as Client
          participant S as Server
          C->>S: POST /<br/>Content-Type: application/json<br/>{"jsonrpc":"2.0","method":"tools/call","params":{"name":"execute_snowflake_query","arguments":{"query":"SELECT CURRENT_USER()"}},"id":1}
          S-->>C: {"jsonrpc":"2.0","result":{},"id":1}
      ```

    - **b.** In this doc: [Helix](#g-helix) sends `POST /mcp` with `"method":"tools/call"` and `params` naming the tool and its arguments. The cited Helix example has no `id`. The success tool result from `execute_snowflake_query` is `results`, `row_count`, `status`, and `query_id`.

      ```mermaid
      sequenceDiagram
          participant H as Helix
          participant M as MCP v2
          H->>M: POST /mcp<br/>Authorization: Bearer {jwt}<br/>Content-Type: application/json<br/>{"jsonrpc":"2.0","method":"tools/call","params":{"name":"execute_snowflake_query","arguments":{"query":"SELECT CURRENT_USER()"}}}
          M-->>H: {"results":[{...}],"row_count":1,"status":"success","message":"Query executed successfully, returned 1 rows","query_id":"{query_id}"}
      ```

    - **c.** Source: request headers [prompthub `server.py` L139-147](./OAuthFlowAndDevProblemOffline.html#actor-helix); success body [`execute_snowflake_query.py` L91-L97](./OAuthFlowAndDevProblemOffline.html#method-mcp-execute-snowflake-query). The request JSON is this document's example of that call: `jsonrpc`, `method`, and `params`, with no `id`.
40. <a id="g-helix"></a>**Helix, Helix-test**
    - **a.** General: Pinterest's internal AI chat assistant; it is the MCP client in this flow.
    - **b.** In this doc: Helix calls [MCP](#g-mcp) tools and, with the `forward_jwt: true` setting, forwards the user's [JWT](#g-jwt). `helix-test.pinadmin.com` is the test environment used for dev runs. Today it shows `auth_start_url` as plain text. Its code lives in [PromptHub](#g-prompthub).
    - **c.** Source: the MCP call headers ([prompthub `backend/app/utils/mcp/server.py` L139-147](./OAuthFlowAndDevProblemOffline.html#actor-helix)).
41. <a id="g-prompthub"></a>**PromptHub**
    - **a.** General: the `pinternal/prompthub` monorepo, which holds Prompt Hub, Helix Chat, and Write.
    - **b.** In this doc: the repo where [Helix](#g-helix)'s MCP client lives. PR 1689 there would render `auth_start_url` as a Connect card, and a local copy also binds port 8000, which competes for the devapp [SSO](#g-sso) host.
    - **c.** Source: [prompthub `README.md` L1-4](./OAuthFlowAndDevProblemOffline.html#actor-helix).
42. <a id="g-token-broker"></a>**`token_broker` (the broker)**
    - **a.** General: a token broker is a central service that runs [OAuth](#g-oauth) for users and stores their tokens, so each client application does not have to.
    - **b.** In this doc: it-swe's service in `optimus/token_broker`, production host `token-broker.pinadmin.com`. Browser endpoints: `/oauth/start` and `/oauth/callback`. Service endpoints: `/api/token`, `/api/status`, and `/api/unlink`. It stores encrypted tokens in `mcp_tokens` and writes `audit_logs`. The dev copy runs on devapp 1 with SQLite and an in-memory key.
    - **c.** Source: [`README.md`](./OAuthFlowAndDevProblemOffline.html#module-broker-config), [`routes/oauth.py`](./OAuthFlowAndDevProblemOffline.html#method-broker-oauth-start), [`routes/api.py`](./OAuthFlowAndDevProblemOffline.html#method-broker-get-token), [`models.py`](./OAuthFlowAndDevProblemOffline.html#model-mcp-token); [Pastis](#g-pastis) policy that decides which callers may reach `/api/*` ([pinconf `token-broker/pastis/base.rego` L8-32](./OAuthFlowAndDevProblemOffline.html#element-mesh-pastis)).
43. <a id="g-devapp"></a>**Devapp**
    - **a.** General: a Pinterest per-engineer cloud development machine.
    - **b.** In this doc: devapp 1 is `devrestricted-dmachicao` (broker) and devapp 2 is `devrestricted-dmachicao-1` (MCP). Each has fixed public hosts: `mcp-<host>.pinterdev.com` to port 8000 with [SSO](#g-sso), and `www-<host>` to 10001 and `api-<host>` to 10002 without SSO (section 8.1).
    - **c.** Example: `https://mcp-devrestricted-dmachicao-1.pinterdev.com` reaches the MCP on devapp 2.
    - **d.** Source: shared devapp pattern in [`oauth-mcp-server-devapp.md`](./OAuthFlowAndDevProblemOffline.html#element-runtime-devapps).
44. <a id="g-knox"></a>**Knox**
    - **a.** General: Pinterest's secret store ([open source](./OAuthFlowAndDevProblemOffline.html#g-knox)). Services fetch keys and credentials from it at runtime instead of keeping them in code or config files.
    - **b.** In this doc: holds the Snowflake [client credentials](#g-confidential-client) and the [Tink](#g-tink) keyset the [broker](#g-token-broker) encrypts with. The devapp has no Knox key for the Snowflake provider, so each lookup fails after about 5 seconds and the broker falls back to an in-memory key.
    - **c.** Source: [`config.py` L38-L54](./OAuthFlowAndDevProblemOffline.html#method-broker-fetch-knox-credentials), [`encryption.py` L58-L72](./OAuthFlowAndDevProblemOffline.html#method-broker-get-aead-primitive).
45. <a id="g-tink"></a>**Tink**
    - **a.** General: Google's open-source cryptography library ([developers.google.com/tink](./OAuthFlowAndDevProblemOffline.html#g-tink)) with safe defaults. A keyset holds the key material; AEAD is its encrypt-and-authenticate primitive.
    - **b.** In this doc: the [broker](#g-token-broker) encrypts access tokens, refresh tokens, and [PKCE](#g-pkce) verifiers with a per-provider AEAD key named `tink:aead:itswe:token_broker:snowflake[:env]`. In dev the fallback is an AES-256-GCM key created in memory at startup, so a restart makes every stored row unreadable.
    - **c.** Source: [`encryption.py` L13-L55](./OAuthFlowAndDevProblemOffline.html#method-broker-get-dev-primitive), [`encryption_key_suffix`, `mcp_servers.production.yaml` L36](./OAuthFlowAndDevProblemOffline.html#class-broker-server-config).
46. <a id="g-pinconf"></a>**pinconf**
    - **a.** General: the `pinternal/pinconf` repository. Traffic and authorization config lives there, separate from application code.
    - **b.** In this doc: [Envoy](#g-envoy) config (`http.yaml`, including [`internal_oauth`](#g-internal-oauth)) and [Pastis](#g-pastis) policies (`base.rego`) are pinconf files. Their references open the corresponding ingress or policy element in the offline HTML. Snowflake MCP v2 has no Envoy or Pastis files there yet; the server only declares the policy URL it will use.
    - **c.** Source: [pinconf `token-broker/prod/main/http.yaml` L8-21](./OAuthFlowAndDevProblemOffline.html#element-ingress-internal-oauth), [pinconf `token-broker/pastis/base.rego` L8-32](./OAuthFlowAndDevProblemOffline.html#element-mesh-pastis), declared URL ([`server.py` L30](./OAuthFlowAndDevProblemOffline.html#module-mcp-server)).

### 10.5 Snowflake

47. <a id="g-security-integration"></a>**Security integration** (Snowflake OAuth integration)
    - **a.** General: a Snowflake account object that registers an OAuth client and its rules ([Snowflake docs](./OAuthFlowAndDevProblemOffline.html#g-security-integration)).
    - **b.** In this doc: the integration issues `{sf_client_id}` and `{sf_client_secret}` and sets `OAUTH_REDIRECT_URI`, `OAUTH_ENFORCE_PKCE = TRUE`, `OAUTH_USE_SECONDARY_ROLES = IMPLICIT`, and the 7-day [refresh token](#g-refresh-token) validity. Snowflake [access tokens](#g-access-token) last 600 seconds.
    - **c.** Example: `CREATE SECURITY INTEGRATION ... TYPE = OAUTH OAUTH_CLIENT = CUSTOM OAUTH_CLIENT_TYPE = 'CONFIDENTIAL' ...`.
48. <a id="g-roles"></a>**Role, default role**
    - **a.** General: in Snowflake, privileges are granted to roles and roles are granted to users. A session has one primary role; unless another is requested, it is the user's default role.
    - **b.** In this doc: the [OAuth](#g-oauth) session starts in the user's default role, so query results reflect that user's grants ("role grants" in section 3).
49. <a id="g-secondary-roles"></a>**Implicit secondary roles**
    - **a.** General: secondary roles let a Snowflake session use privileges from all of a user's granted roles, not only the primary one.
    - **b.** In this doc: `OAUTH_USE_SECONDARY_ROLES = IMPLICIT` on the [integration](#g-security-integration) makes Snowflake activate all granted roles as secondary roles in every OAuth session, in addition to the [default role](#g-roles).
    - **c.** Source: [Snowflake `CREATE SECURITY INTEGRATION` (Snowflake OAuth)](./OAuthFlowAndDevProblemOffline.html#g-security-integration).

### 10.6 Attacks

50. <a id="g-login-csrf"></a>**Login CSRF and token planting**
    - **a.** General: cross-site request forgery (CSRF) makes a victim's browser send a request the victim did not intend. In login CSRF, the victim ends up signed in, or linked, as the attacker. Token planting is the outcome: the attacker's credential is stored under the victim's identity. There is no single standard message sequence for it.
    - **b.** In this doc: what would happen if `/oauth/start` trusted `?user_id=` (section 5.2). The victim's [Helix](#g-helix) queries would run as the attacker in Snowflake. [Envoy](#g-envoy) overwriting [`X-Forwarded-User`](#g-x-forwarded-user) after [SSO](#g-sso) prevents it, because the query `user_id` is then ignored. The messages below are the real broker parameters, under the assumption the header is absent.

      ```mermaid
      sequenceDiagram
          actor X as Attacker
          actor V as Victim
          participant B as token_broker
          participant DB as Token store
          participant S as Snowflake
          X->>B: GET /oauth/start?user_id=victim&client_id={MCP SPIFFE}&mcp_server_id=snowflake
          B->>DB: INSERT oauth_states (state={state}, user_id=victim, client_id={MCP SPIFFE}, mcp_server_id=snowflake, code_verifier=encrypt(code_verifier), redirect_uri=https://token-broker.pinadmin.com/oauth/callback, expires_at=now+600s)
          B-->>X: 307 Location: https://{account}.snowflakecomputing.com/oauth/authorize?response_type=code&client_id={sf_client_id}&redirect_uri=https://token-broker.pinadmin.com/oauth/callback&state={state}&scope=refresh_token&code_challenge={code_challenge}&code_challenge_method=S256
          X->>S: GET /oauth/authorize?response_type=code&client_id={sf_client_id}&redirect_uri=https://token-broker.pinadmin.com/oauth/callback&state={state}&scope=refresh_token&code_challenge={code_challenge}&code_challenge_method=S256
          X->>S: approve as the attacker
          S-->>X: 302 Location: https://token-broker.pinadmin.com/oauth/callback?code={code}&state={state}
          X->>B: GET /oauth/callback?code={code}&state={state}
          B->>S: POST /oauth/token-request<br/>Content-Type: application/x-www-form-urlencoded<br/>grant_type=authorization_code&code={code}&redirect_uri=https://token-broker.pinadmin.com/oauth/callback&client_id={sf_client_id}&client_secret={sf_client_secret}&code_verifier={code_verifier}
          B->>DB: UPSERT mcp_tokens (user_id=victim, client_id={MCP SPIFFE}, mcp_server_id=snowflake) = attacker's tokens
          V->>B: GET /api/token?mcp_server_id=snowflake<br/>Authorization: Bearer {victim jwt}<br/>X-Forwarded-Client-Cert: By={broker SPIFFE}#59;URI={MCP SPIFFE}
          B-->>V: 200 {"status":"ok","access_token":"{attacker access_token}","token_type":"{token_type}"}
      ```

    - **c.** Source: [`routes/oauth.py` L45](./OAuthFlowAndDevProblemOffline.html#method-broker-oauth-start) (`?user_id=` is used only when `X-Forwarded-User` is absent), callback stores the key saved at start [`services/oauth.py` L147-L157](./OAuthFlowAndDevProblemOffline.html#method-broker-exchange-code), later fetch [`api.py` L21-L95](./OAuthFlowAndDevProblemOffline.html#method-broker-get-token).
