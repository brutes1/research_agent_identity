# Agentic Identity Reference

**Version:** 1.0.0 | **Last Verified:** 2026-05-07 | **Schema:** `docs/agentic-identity-reference.yaml`

> **Authoritative source:** `docs/agentic-identity-reference.yaml`. This Markdown is the human-readable rendering. Property tables are kept in sync with the YAML companion.

---

## Table of Contents

- [Part 0: How to Use This Document](#part-0-how-to-use-this-document)
- [Part A: Foundational Patterns](#part-a-foundational-patterns)
- [Part 1: Decision Guide](#part-1-decision-guide)
- [Part 1b: Scenario Recipes](#part-1b-scenario-recipes)
- [Part 2: OAuth 2.0 Grants for Agents](#part-2-oauth-20-grants-for-agents)
- [Part 3: OpenID Connect for Agents](#part-3-openid-connect-for-agents)
- [Part 4: SPIFFE / SPIRE](#part-4-spiffe--spire)
- [Part 5: Token Binding Mechanisms](#part-5-token-binding-mechanisms)
- [Part 6: Cloud Provider Workload Identity](#part-6-cloud-provider-workload-identity)
- [Part 7: Kubernetes Service Account Tokens](#part-7-kubernetes-service-account-tokens)
- [Part 8: CI/CD OIDC Tokens](#part-8-cicd-oidc-tokens)
- [Part 9: HTTP Authorization Headers](#part-9-http-authorization-headers)
- [Part 10a: OSS AI Agent Framework Auth](#part-10a-oss-ai-agent-framework-auth)
- [Part 10b: Cloud Agent Platform Auth](#part-10b-cloud-agent-platform-auth)
- [Part 11: Model Context Protocol (MCP) Auth](#part-11-model-context-protocol-mcp-auth)
- [Part 12: Agent-to-Agent (A2A) Auth](#part-12-agent-to-agent-a2a-auth)
- [Part 13: CIBA and Async Authorization](#part-13-ciba-and-async-authorization)
- [Part 14: Agent Lifecycle Management](#part-14-agent-lifecycle-management)
- [Part 15: Emerging Standards Tracker](#part-15-emerging-standards-tracker)
- [Part 16: Security Anti-Patterns](#part-16-security-anti-patterns)
- [Part 17: Token Lifecycle Management](#part-17-token-lifecycle-management)
- [Part 18: Failure Mode Diagnostics](#part-18-failure-mode-diagnostics)
- [Part 19: JWT Validation Reference](#part-19-jwt-validation-reference)
- [Glossary](#glossary)

---

## Part 0: How to Use This Document

This document is structured for both human reading and LLM/agent consumption. Each mechanism section is self-contained: subheadings include the mechanism name so RAG-extracted chunks self-identify, and every section opens with a YAML frontmatter block containing a stable machine-readable ID.

**Navigation guide for agents:**

| I need to… | Go to |
|---|---|
| Solve a specific end-to-end deployment scenario | [Part 1b Scenario Recipes](#part-1b-scenario-recipes) or `scenarios:` YAML block |
| Look up a single mechanism by ID | `mechanisms[id]` in YAML or jump to section anchor |
| Find what runtime X supports | [Part 1 Decision Tables](#part-1-decision-guide) |
| Get the exact wire format / HTTP headers | `mechanisms[id].wire_format_examples` in YAML or [Part 9](#part-9-http-authorization-headers) |
| Check if a mechanism is stable enough to ship | `mechanisms[id].status` + `last_verified` in YAML |
| Understand security risks | [Part 16 Anti-Patterns](#part-16-security-anti-patterns) |
| Debug a 401 error | [Part 18 Failure Mode Diagnostics](#part-18-failure-mode-diagnostics) |
| Validate a JWT correctly | [Part 19 JWT Validation Reference](#part-19-jwt-validation-reference) |

**ID namespace conventions used throughout this document:**

| Prefix | Refers to |
|---|---|
| `mech:` | An identity/auth mechanism (e.g., `mech:oauth2.client-credentials`) |
| `framework:` | An agent framework (e.g., `framework:langgraph`) |
| `platform:` | A cloud/runtime platform (e.g., `platform:eks`) |
| `rfc:` | An RFC or specification (e.g., `rfc:8693`) |
| `claim:` | A JWT/token claim (e.g., `claim:aud`) |
| `header:` | An HTTP header (e.g., `header:authorization-bearer`) |
| `threat:` | A threat or anti-pattern (e.g., `threat:alg-none`) |
| `term:` | A glossary term (e.g., `term:holder-bound`) |
| `scenario:` | A scenario recipe (e.g., `scenario:gha-to-mcp`) |

---

## Part A: Foundational Patterns

Before diving into mechanisms, four patterns repeat across the entire agentic identity space. Each mechanism section cross-references these with a `[Pattern: Px]` callout.

### Pattern P1: STS Token Exchange

A workload presents a credential it already holds to a Security Token Service (STS). The STS validates the incoming credential and issues a new credential scoped to the target environment, resource, or audience.

**Where used:** AWS IRSA, GCP Workload Identity Federation, Azure Federated Identity, SPIFFE OIDC federation, MCP resource indicators, RFC 8693 Token Exchange.

**Key principle:** The workload never receives a long-lived secret. It presents a short-lived platform credential and exchanges it for a short-lived service credential. The STS enforces the trust mapping.

### Pattern P2: Workload Attestation

A trusted system co-located with the workload (hypervisor, node agent, orchestrator, kernel) performs out-of-band verification of the workload's identity and issues or proxies credentials on its behalf. The workload cannot forge its own attestation.

**Where used:** AWS IMDS (instance identity document), GCP metadata server, SPIRE node attestation, Kubernetes projected service account tokens (kubelet signs tokens), Azure IMDS.

**Key principle:** Identity flows from the execution environment to the workload, not the other way around. The workload proves it IS running in a given environment; it does not claim an identity.

### Pattern P3: Automatic Credential Rotation

The workload never holds a long-lived secret. A local agent or platform subsystem rotates credentials before expiry and exposes the current credential at a well-known path or API. The workload reads the current credential at use time.

**Where used:** SPIFFE X.509-SVIDs (rotated by SPIRE every 1 hour by default), Kubernetes projected SA tokens (kubelet rotates; mounted as file), cloud IMDS tokens (refreshed before expiry), AWS SDK credential provider chain.

**Key principle:** Rotation is automatic and transparent. If a credential is compromised, the blast radius is bounded by lifetime, not by detection time.

### Pattern P4: No Secrets in Code or Config

Long-lived credentials (passwords, API keys, client secrets) must never appear in source code, container images, CI/CD pipeline definitions, or environment variables baked into deployments.

**Violation indicators:** `export API_KEY=sk-...` in Dockerfile or `.env`; `client_secret` in `config.yaml` committed to git; credentials in GitHub Actions `env:` block without OIDC alternative.

**Remediation path for each credential type:**
- API keys → workload identity (cloud) or client credentials with `private_key_jwt` (self-hosted)
- Shared secrets → SPIFFE mTLS or DPoP
- CI/CD secrets → CI/CD OIDC token exchange
- Database passwords → cloud secret manager with automatic rotation

---

## Part 1: Decision Guide {#part-1-decision-guide}

> **Guiding principles:** No Secrets in Code/Config (P4) · Prefer Short-Lived Credentials · Bind Tokens to the Holder When Possible · Validate All Claims

### Agent Autonomy Spectrum

The appropriate identity model depends on agent autonomy level:

| Level | Description | Identity Model |
|---|---|---|
| **Assistant** | Human guides every step; agent executes one action at a time | Delegated user sub-identity; `act` claim; RFC 8693 |
| **Semi-autonomous** | Chain-of-thought with human checkpoints; bounded task scope | Enhanced service account; short-lived tokens with restricted scope |
| **Fully autonomous** | Open-ended task completion; may spawn sub-agents | Sovereign agent identity; SPIFFE or OIDC-A; scope attenuation per hop |

### Trust Domain Taxonomy

| Domain Type | Characteristics | Recommended Mechanism |
|---|---|---|
| **Single Trust Domain** | Centralized IdP; shared infrastructure; synchronous ops | Client Credentials / Workload Identity |
| **Cross-Domain Federation** | Multiple IdPs; async delegation chains | RFC 8693 Token Exchange + Identity Assertion Grant |
| **Decentralized / Open Web** | No central authority; unknown partners | CIBA; Web Bot Auth; A2A JWS; OpenID Federation |

### Table 1: By Deployment Platform → Recommended Identity Mechanism

| Platform | Primary Mechanism | Notes |
|---|---|---|
| GKE (Google Kubernetes Engine) | GKE Workload Identity → GCP IAM | Annotate SA; token auto-mounted |
| EKS (Amazon Elastic Kubernetes Service) | IRSA or EKS Pod Identity | Pod Identity preferred (no annotation needed) |
| AKS (Azure Kubernetes Service) | Azure Managed Identity (pod-level) | Use Azure Workload Identity chart |
| AWS Lambda | Lambda execution role via IMDS | IMDSv2 only; hop-limit=1 |
| GCP Cloud Run | GCP metadata server token | `Metadata-Flavor: Google` header required |
| Azure Container Apps | Managed Identity | System-assigned or user-assigned |
| Nomad | SPIFFE/SPIRE or Vault Agent | SPIRE Nomad plugin or Vault Agent sidecar |
| Bare metal / VMs | SPIFFE SPIRE or IAM Roles Anywhere | X.509 trust anchors for on-prem |
| GitHub Actions | GitHub Actions OIDC → cloud WIF | `permissions: id-token: write` required |
| GitLab CI | GitLab CI OIDC → cloud WIF | `id_tokens:` block in `.gitlab-ci.yml` |
| CircleCI | CircleCI OIDC → cloud WIF | `CIRCLE_OIDC_TOKEN` env var |
| Local / Dev | `gcloud auth application-default login` / AWS CLI SSO | Never use prod credentials locally |

### Table 2: By Agent Framework → Native Auth Mechanism

| Framework | Native Auth | Token Refresh Handling |
|---|---|---|
| LangGraph | `@auth.authenticate` decorator; custom handler | No — manual implementation required |
| AutoGen / MAF | Entra Agent ID; `agent` subtype service principal | Yes — Entra SDK handles automatically |
| CrewAI | Env var API keys; `SimpleTokenAuth` for A2A | No — manual |
| Semantic Kernel | Azure Managed Identity; `DefaultAzureCredential` | Yes — Azure Identity SDK |
| LlamaIndex | Env var API keys | N/A — long-lived keys only (anti-pattern for prod) |
| Haystack | Env var pipeline credentials | N/A — long-lived keys only (anti-pattern for prod) |
| Claude / MCP | OAuth 2.1 PKCE + resource indicators | Yes — SDK managed |
| OpenAI Agents SDK | `sk-proj-` scoped API keys; MCP OAuth 2.1 for tools | N/A for API keys |
| Google ADK | `AuthScheme` + `AuthCredential`; OAUTH2, SERVICE_ACCOUNT, OIDC | Yes — SDK managed |
| AWS Bedrock AgentCore | AgentCore Identity Token Vault (KMS-encrypted) | Yes — automatic (Token Vault) |
| Azure AI Foundry | Entra Agent ID Blueprint; Federated Managed Identity | Yes — Managed Identity auto-rotates |
| Vertex AI Agents | Application Default Credentials; Workload Identity Federation | Yes — google-auth library |

### Table 3: By Use Case → Recommended Mechanism(s)

| I need to… | Recommended mechanism(s) |
|---|---|
| Agent calls external API, no user involved (M2M) | OAuth 2.0 Client Credentials (`mech:oauth2.client-credentials`) |
| Delegate user context downstream to sub-agent | RFC 8693 Token Exchange + `act` claim (`mech:oauth2.token-exchange`) |
| GitHub Actions job calls MCP server as machine identity | GHA OIDC → Token Exchange → OAuth 2.1 Bearer (`scenario:gha-to-mcp`) |
| Cross-cloud call (AWS EKS → GCP service) | SPIFFE JWT-SVID federation OR GCP WIF with AWS as IdP |
| CI/CD pipeline deploys to cloud without stored secrets | CI/CD OIDC + Cloud Workload Identity Federation |
| Deep multi-agent delegation chain, low latency | Biscuits / Macaroons (offline attenuation) |
| Headless agent needs user consent | Device Authorization Grant (`mech:oauth2.device-auth`) |
| Long-running async agent needs mid-task re-auth | CIBA (`mech:ciba`) |
| High-value financial transaction | FAPI 2.0: DPoP + mTLS + PAR + RAR |
| Agent migrating from API keys to workload identity | See `scenario:api-key-migration` |
| On-premises agent calling cloud services | IAM Roles Anywhere (AWS) or Workload Identity Federation with X.509 |
| Agent identity verified by unknown partner | A2A JWS-signed AgentCard or OpenID Federation |

### Table 4: By Security Assurance Level → Mechanisms

| Assurance | Characteristics | Example Mechanisms |
|---|---|---|
| **Low** | Long-lived bearer token or API key; no binding | Static API keys; long-lived OAuth tokens |
| **Medium** | Short-lived JWT; bearer (no holder binding) | Client Credentials; CI/CD OIDC tokens; projected SA tokens |
| **High** | Short-lived + holder-bound (DPoP or mTLS) | DPoP-bound access tokens; certificate-bound tokens |
| **Very High** | Hardware-attested + holder-bound; short-lived | SPIFFE X.509-SVID via HSM; FAPI 2.0 mTLS + DPoP |

### Decision Flowchart (Mermaid)

```mermaid
flowchart TD
    Start([Agent needs to call a service]) --> Q1{Does a human user<br/>need to authorize?}
    Q1 -->|Yes| Q2{Is the agent<br/>headless / no browser?}
    Q1 -->|No - M2M| Q3{What runtime<br/>environment?}
    Q2 -->|Yes| DeviceGrant[Device Authorization Grant]
    Q2 -->|No| AuthCode[Auth Code + PKCE]
    Q3 -->|Kubernetes| Q4{Cloud-managed cluster?}
    Q3 -->|Cloud serverless| CloudIMDS[Cloud IMDS / Metadata Server]
    Q3 -->|CI/CD pipeline| CIDOIDC[CI/CD OIDC → WIF]
    Q3 -->|Bare metal / multi-cloud| SPIFFE[SPIFFE / SPIRE]
    Q4 -->|Yes| WorkloadID[Cloud Workload Identity<br/>GKE/EKS/AKS]
    Q4 -->|No| Q5{SPIRE available?}
    Q5 -->|Yes| SPIFFE
    Q5 -->|No| ClientCreds[Client Credentials<br/>+ private_key_jwt]
```

### `decision_tree:` YAML block

```yaml
decision_tree:
  - id: rule-gha-mcp-machine
    if: {runtime: github_actions, target_type: mcp_server, identity_kind: machine}
    then: {scenario: gha-to-mcp}
  - id: rule-eks-external-api-m2m
    if: {runtime: eks_pod, target_type: external_api, identity_kind: machine}
    then: {mechanism: mech:aws-irsa, then: mech:oauth2.client-credentials}
  - id: rule-user-delegation
    if: {identity_kind: user_delegated, target_type: any}
    then: {mechanism: mech:oauth2.token-exchange}
  - id: rule-cicd-cloud
    if: {runtime: github_actions, target_type: cloud_iam, identity_kind: machine}
    then: {mechanism: mech:github-actions-oidc, then: mech:cloud-wif}
  - id: rule-deep-delegation-low-latency
    if: {identity_kind: delegated_chain, latency_constraint: low}
    then: {mechanism: mech:biscuits}
  - id: rule-high-assurance
    if: {security_assurance_required: very_high}
    then: {mechanism: mech:fapi2-dpop-mtls}
```

---

## Part 1b: Scenario Recipes {#part-1b-scenario-recipes}

End-to-end multi-hop walkthroughs. Each recipe shows: initial identity source → exchange(s) → credential at target → exact HTTP headers.

---

### Scenario 1: GitHub Actions → MCP Server (Machine Identity) {#scenario-gha-to-mcp}

```yaml
id: scenario:gha-to-mcp
source_runtime: github_actions
target: mcp_server
identity_kind: machine
```

**Steps:**

1. **Request GHA OIDC token** — job requests OIDC token with `permissions: id-token: write`
2. **Exchange at cloud STS** — POST to cloud WIF token endpoint with GHA OIDC as `subject_token`
3. **Obtain OAuth 2.1 bearer** — use resulting cloud credential to call MCP server's `/authorize` endpoint
4. **Call MCP server** — attach bearer in `Authorization` header

```http
# Step 1: GitHub injects OIDC token (automatic)
# ACTIONS_ID_TOKEN_REQUEST_URL + ACTIONS_ID_TOKEN_REQUEST_TOKEN

# Step 2: Exchange at STS (e.g., GCP)
POST https://sts.googleapis.com/v1/token
Content-Type: application/x-www-form-urlencoded

grant_type=urn:ietf:params:oauth:grant-type:token-exchange
&subject_token=<gha_oidc_jwt>
&subject_token_type=urn:ietf:params:oauth:token-type:id_token
&audience=//iam.googleapis.com/projects/123/locations/global/workloadIdentityPools/pool/providers/gha

# Step 3: Call MCP tool
GET /tools/search
Authorization: Bearer <oauth2_bearer>
```

---

### Scenario 2: EKS Pod → Anthropic API + Private MCP Server {#scenario-eks-parallel-creds}

```yaml
id: scenario:eks-parallel-creds
source_runtime: eks_pod
targets: [anthropic_api, private_mcp_server]
identity_kind: machine
```

The pod holds two credentials simultaneously — an Anthropic API key (scoped to the project, rotated weekly via AWS Secrets Manager) and a workload identity token for the internal MCP server.

```http
# Anthropic API call
POST https://api.anthropic.com/v1/messages
x-api-key: <rotated_key_from_secrets_manager>
anthropic-version: 2023-06-01

# Internal MCP server call (SPIFFE mTLS)
# Client presents X.509-SVID certificate — no Authorization header needed
# Server verifies spiffe://cluster.local/ns/default/sa/agent
```

---

### Scenario 3: Kubernetes Pod → Cross-Cloud GCP Service {#scenario-k8s-to-gcp}

```yaml
id: scenario:k8s-to-gcp
source_runtime: eks_pod
target: gcp_service
identity_kind: machine
```

```http
# Step 1: Read projected SA token
# /var/run/secrets/tokens/gcp-token (audience: gcp-pool-provider)

# Step 2: Exchange at GCP STS
POST https://sts.googleapis.com/v1/token
grant_type=urn:ietf:params:oauth:grant-type:token-exchange
&subject_token=<k8s_projected_token>
&subject_token_type=urn:ietf:params:oauth:token-type:jwt
&audience=//iam.googleapis.com/projects/PROJECT/locations/global/workloadIdentityPools/POOL/providers/PROVIDER
&requested_token_type=urn:ietf:params:oauth:token-type:access_token

# Step 3: Call GCP service
GET https://storage.googleapis.com/storage/v1/b/my-bucket/o
Authorization: Bearer <gcp_access_token>
```

---

### Scenario 4: Lambda Function → DynamoDB + External API {#scenario-lambda-dual}

Lambda execution role provides IAM SigV4 for DynamoDB (via SDK, transparent). For external API, the function retrieves a rotated key from AWS Secrets Manager or uses IRSA-exchanged OAuth token.

```http
# DynamoDB — AWS SDK handles SigV4 automatically using IMDS credentials
# IMDSv2 required: PUT http://169.254.169.254/latest/api/token first

# External API — use Secrets Manager rotated key
GET /external-api/resource
Authorization: Bearer <secret_from_secrets_manager>
```

---

### Scenario 5: LangGraph Agent → Multi-Tool, Different Auth per Tool {#scenario-langgraph-multi-tool}

```yaml
id: scenario:langgraph-multi-tool
framework: langgraph
identity_kind: mixed
```

LangGraph's `@auth.authenticate` handler is called once per request. The handler resolves per-tool credentials from the token store, using short-lived OAuth tokens for each tool's required scope.

```python
# In LangGraph auth handler
@auth.authenticate
async def authenticate(request: Request) -> AuthContext:
    agent_token = request.headers["Authorization"].split(" ")[1]
    # validate agent_token, resolve tool credentials
    return AuthContext(
        user=agent_identity,
        credentials={"tool_a": tool_a_token, "tool_b": tool_b_token}
    )
```

---

### Scenario 6: User-Delegated Agent → Sub-Agent (RFC 8693 Chain) {#scenario-delegation-chain}

```yaml
id: scenario:delegation-chain
mechanism: mech:oauth2.token-exchange
```

```http
# Agent A holds user-delegated token (has act claim from initial delegation)
# Agent A requests token for Agent B with narrowed scope

POST /token HTTP/1.1
Content-Type: application/x-www-form-urlencoded

grant_type=urn:ietf:params:oauth:grant-type:token-exchange
&subject_token=<agent_a_token>
&subject_token_type=urn:ietf:params:oauth:token-type:access_token
&actor_token=<agent_b_identity_token>
&actor_token_type=urn:ietf:params:oauth:token-type:id_token
&scope=read:documents
&resource=https://docs.example.com

# Resulting token has act claim:
# { "sub": "user@example.com", "act": { "sub": "agent-b", "act": { "sub": "agent-a" } } }
```

---

### Scenario 7: Three-Hop Delegation with Scope Attenuation {#scenario-three-hop}

```yaml
id: scenario:three-hop-attenuation
```

Each hop narrows scope. The STS enforces that `requested_scope ⊆ subject_token_scope`.

```
User (read:write:delete) →
  Agent A (read:write, act=A) →
    Agent B (read, act=B, act.act=A) →
      Agent C (read:specific-resource, act=C, ...)
```

No agent can escalate to permissions the upstream token does not contain.

---

### Scenario 8: CI/CD Pipeline → Cloud Infra (No Stored Secrets) {#scenario-cicd-no-secrets}

```yaml
id: scenario:cicd-no-secrets
source_runtime: github_actions
```

```yaml
# .github/workflows/deploy.yml
permissions:
  id-token: write
  contents: read

jobs:
  deploy:
    steps:
      - uses: google-github-actions/auth@v2
        with:
          workload_identity_provider: projects/123/locations/global/workloadIdentityPools/pool/providers/gha
          service_account: deploy-sa@project.iam.gserviceaccount.com
```

No `GCP_SA_KEY` secret ever stored in GitHub. GitHub token is short-lived; trust is enforced by GCP WIF policy scoped to the exact repo+branch `sub` claim.

---

### Scenario 9: Browser-Automation Agent (Cookie + Session) {#scenario-browser-agent}

No standard exists for browser agents to prove their AI identity (Web Bot Auth is in early proposal). Current pattern: the agent operates under the human user's session. Mitigation: use a dedicated service account with minimum required OAuth scopes; MFA protected; session monitored for anomalous access patterns.

---

### Scenario 10: Cross-Organizational A2A Call {#scenario-cross-org-a2a}

```yaml
id: scenario:cross-org-a2a
mechanism: mech:a2a-jws
```

```http
# Calling partner agent fetches target's AgentCard
GET https://partner.example.com/.well-known/agent.json

# Sends request signed with JWS (JCS canonicalization)
POST https://partner.example.com/agent/task
Content-Type: application/json
Authorization: Bearer <oauth2_bearer>

{
  "message": { ... },
  "signature": "<jws_compact_serialization>"
}
```

Trust bootstrap: verify AgentCard over TLS (DNS anchoring). Cards must be re-fetched periodically (no revocation mechanism defined in current spec — treat as short-lived, max 1 hour cache).

---

### Scenario 11: Migrate from API Keys to Workload Identity {#scenario-api-key-migration}

```yaml
id: scenario:api-key-migration
```

**Step-by-step migration path:**

1. **Audit** — identify all long-lived API keys using `git log -S 'API_KEY' --all`; grep for `export.*KEY=` in CI
2. **Add workload identity** — enable IRSA/GKE WI/AKS WI on the cluster or function
3. **Deploy dual-credential** — code reads from workload identity first, falls back to key while migrating
4. **Rotate key to short TTL** — reduce key lifetime to 7 days via key management system
5. **Remove key** — once all workloads use WI, revoke the API key

---

### Scenario 12: Headless Agent Needs User Consent {#scenario-device-grant}

```yaml
id: scenario:device-grant
mechanism: mech:oauth2.device-auth
```

```http
# Step 1: Start device flow
POST /device/code
client_id=agent-client&scope=read:calendar

# Response
{
  "device_code": "...",
  "user_code": "XKCD-7842",
  "verification_uri": "https://example.com/device",
  "expires_in": 300
}

# Step 2: Agent displays user_code to user (email, Slack, etc.)
# Step 3: Poll for token
POST /token
grant_type=urn:ietf:params:oauth:grant-type:device_code&device_code=...&client_id=...

# Returns access_token once user completes authorization
```

---

### Scenario 13: Long-Running Async Agent with CIBA Re-Authorization {#scenario-ciba-reauth}

```yaml
id: scenario:ciba-reauth
mechanism: mech:ciba
```

Agent starts a multi-hour task. When a high-risk action is reached mid-task, agent pauses and initiates CIBA:

```http
POST /bc-authorize
Content-Type: application/x-www-form-urlencoded

client_id=agent-client
&scope=delete:production-data
&login_hint=user@example.com
&binding_message=Agent is about to delete 500 records. Approve?
&user_code=...

# Response: { "auth_req_id": "...", "expires_in": 300, "interval": 5 }
# Agent polls /token every 5s until user approves on their phone
```

---

### Scenario 14: High-Value Transaction (FAPI 2.0) {#scenario-fapi2}

```yaml
id: scenario:fapi2-high-value
mechanism: mech:fapi2
```

Financial/healthcare transactions require DPoP + mTLS + PAR + RAR:

```http
# Step 1: PAR with authorization_details
POST /par
DPoP: <proof_jwt>
Content-Type: application/x-www-form-urlencoded

client_id=...&code_challenge=...&code_challenge_method=S256
&authorization_details=[{"type":"payment","amount":1000,"currency":"USD"}]

# Step 2: Complete auth code flow (redirect)
# Step 3: API call with DPoP + mTLS
POST /payments
Authorization: DPoP <access_token>
DPoP: <proof_jwt>
# Client cert presented at TLS layer
```

---

### Scenario 15: On-Premises Agent → Cloud Services {#scenario-on-prem-to-cloud}

```yaml
id: scenario:on-prem-to-cloud
```

**AWS IAM Roles Anywhere:** On-prem server presents X.509 certificate from on-prem PKI. AWS validates cert against configured Trust Anchor, issues temporary IAM credentials.

```bash
aws_signing_helper credential-process \
  --certificate /etc/agent/cert.pem \
  --private-key /etc/agent/key.pem \
  --trust-anchor-arn arn:aws:rolesanywhere:... \
  --profile-arn arn:aws:rolesanywhere:... \
  --role-arn arn:aws:iam::...
```

**GCP Workload Identity Federation:** Upload on-prem PKI root cert to GCP as X.509 trust anchor. Workload presents cert to GCP STS to obtain GCP access token.

---

## Part 2: OAuth 2.0 Grants for Agents {#part-2-oauth-20-grants-for-agents}

### OAuth 2.0 Client Credentials {#oauth2-client-credentials}

```yaml
id: mech:oauth2.client-credentials
status: stable
category: m2m
tldr: "Client authenticates with client_id + secret (or private key JWT); receives Bearer token; no user involved."
rfc: "RFC 6749 §4.4; OAuth 2.1 draft-15"
```

> **TL;DR:** The workhorse of M2M agent auth. Use when no user is involved. Prefer `private_key_jwt` over `client_secret` for production.

| Property | Value |
|---|---|
| Token type | JWT or Opaque |
| Lifetime | Typically 1 hour |
| Holder-bound? | Optional (add DPoP or mTLS) |
| Revocable? | Short-expiry only (unless with introspection) |
| Requires user interaction? | No |
| Attestation mechanism | None (client is responsible for securing its own key) |
| Rotation mechanism | Manual-refresh or SDK-managed |
| Security assurance | Medium (bearer); High (with DPoP/mTLS) |
| Use case category | M2M, backend agents, scheduled agents |

#### How OAuth 2.0 Client Credentials works

The client presents its identity and authenticates to the authorization server's token endpoint. The AS verifies the client, then issues an access token bounded to the client's registered scope. No resource owner interaction is required.

Two client authentication methods:

**`client_secret_basic`** (avoid in production for sensitive scopes):
```http
POST /token HTTP/1.1
Authorization: Basic base64(client_id:client_secret)
Content-Type: application/x-www-form-urlencoded

grant_type=client_credentials&scope=read:data
```

**`private_key_jwt`** (RFC 7523 — recommended):
```http
POST /token HTTP/1.1
Content-Type: application/x-www-form-urlencoded

grant_type=client_credentials
&client_assertion_type=urn:ietf:params:oauth:client-assertion-type:jwt-bearer
&client_assertion=<jwt>
&scope=read:data
```

The JWT for `private_key_jwt` must contain:
- `iss` = `client_id`
- `sub` = `client_id`
- `aud` = token endpoint URL (exact)
- `jti` = unique per request (prevent replay)
- `exp` = at most 300 seconds from `iat`

#### OAuth 2.0 Client Credentials wire format

```http
# Token request response
HTTP/1.1 200 OK
Content-Type: application/json

{
  "access_token": "eyJhbGci...",
  "token_type": "Bearer",
  "expires_in": 3600,
  "scope": "read:data"
}

# Using the token
GET /resource HTTP/1.1
Authorization: Bearer eyJhbGci...
```

#### OAuth 2.0 Client Credentials producer: how to obtain

1. Register client with AS; obtain `client_id` and configure `private_key_jwt` or secret
2. POST to token endpoint with `grant_type=client_credentials`
3. Cache token until `exp - 60s`; refresh proactively

#### OAuth 2.0 Client Credentials consumer: how to validate

1. Verify `alg` is in allowlist (e.g., `RS256`, `ES256`, `EdDSA`) — reject `none`
2. Verify signature using issuer's JWKS endpoint (pre-configured, not from token `jku`)
3. Verify `iss` exact match
4. Verify `aud` contains this service's identifier (exact string match)
5. Verify `exp` > now (allow ≤60s clock skew)
6. Verify `scope` contains required permission
7. For opaque tokens: call RFC 7662 introspection endpoint

#### OAuth 2.0 Client Credentials failure modes and diagnosis

| Error | Cause | Fix |
|---|---|---|
| `invalid_client` | Wrong `client_id` or failed auth | Check `client_id`; verify JWT `iss`=`sub`=`client_id` |
| `invalid_scope` | Requested scope not registered | Register scope with AS; check scope spelling |
| `unauthorized_client` | Client not authorized for client_credentials | Enable grant type on client registration |
| 401 on API call | Token expired or wrong audience | Check `exp`; verify `aud` includes API identifier |

#### OAuth 2.0 Client Credentials where used

Frameworks: LangGraph, CrewAI, LlamaIndex, Haystack (all via env-var pattern — upgrade to `private_key_jwt`)
Cloud platforms: All (AWS STS, GCP IAM, Azure AD all issue CC tokens)
Runtimes: Kubernetes, Lambda, Cloud Run, Container Apps, bare metal

#### OAuth 2.0 Client Credentials when NOT to use

- When user context is required — use Auth Code + PKCE + Token Exchange
- When the deployment runtime supports workload identity (GKE, EKS, Lambda) — prefer cloud WI over managing a client secret
- For high-assurance transactions — add DPoP or mTLS binding

---

### OAuth 2.0 Token Exchange {#oauth2-token-exchange}

```yaml
id: mech:oauth2.token-exchange
status: stable
category: delegation
tldr: "Workload presents a credential to STS; receives new credential scoped to a different audience, resource, or identity. Enables agent delegation chains."
rfc: "RFC 8693"
```

> **TL;DR:** The foundation of multi-agent delegation. [Pattern: P1 — STS Token Exchange]

| Property | Value |
|---|---|
| Token type | JWT (new token from STS) |
| Lifetime | Short (STS-configured, typically 1h) |
| Holder-bound? | Optional (STS can issue DPoP or mTLS-bound token) |
| Revocable? | Short-expiry |
| Requires user interaction? | No (for machine delegation) |
| Attestation mechanism | Inherited from subject token |
| Rotation mechanism | Refresh subject token, re-exchange |
| Security assurance | Medium–High |
| Use case category | M2M delegation, user delegation, cross-cloud, CI/CD |

#### How OAuth 2.0 Token Exchange works

The client presents a `subject_token` (the credential it currently holds) and optionally an `actor_token` (the credential of who is acting). The STS validates both, enforces scope subset rules, and issues a new access token.

Key parameters:
- `grant_type`: `urn:ietf:params:oauth:grant-type:token-exchange`
- `subject_token`: credential being exchanged (user token, OIDC token, SPIFFE JWT)
- `subject_token_type`: type URI for subject_token
- `actor_token`: identity of the acting party (agent)
- `actor_token_type`: type URI for actor_token
- `resource`: target resource URI (RFC 8707)
- `scope`: requested scope (must be ⊆ subject_token scope)

#### OAuth 2.0 Token Exchange wire format

```http
POST /token HTTP/1.1
Content-Type: application/x-www-form-urlencoded

grant_type=urn:ietf:params:oauth:grant-type:token-exchange
&subject_token=<user_access_token>
&subject_token_type=urn:ietf:params:oauth:token-type:access_token
&actor_token=<agent_id_token>
&actor_token_type=urn:ietf:params:oauth:token-type:id_token
&resource=https://api.example.com
&scope=read:documents

# Response
{
  "access_token": "eyJ...",
  "token_type": "Bearer",
  "issued_token_type": "urn:ietf:params:oauth:token-type:access_token",
  "expires_in": 3600
}
```

The resulting token's `act` claim identifies the delegation chain:
```json
{
  "sub": "user@example.com",
  "act": {
    "sub": "agent-b-client-id",
    "act": {
      "sub": "agent-a-client-id"
    }
  }
}
```

#### OAuth 2.0 Token Exchange consumer: how to validate

All standard JWT checks, plus:
- Verify `act` chain depth does not exceed policy limit (recommend max 3 hops)
- Verify no circular references in `act` chain
- Verify scope of token is ⊆ scope of original subject_token (check via introspection if STS supports)
- Verify `aud` matches this service (do not accept forwarded tokens)

#### OAuth 2.0 Token Exchange failure modes and diagnosis

| Error | Cause | Fix |
|---|---|---|
| `invalid_request` + `scope exceeds subject_token` | Requested scope > granted scope | Narrow requested scope |
| `invalid_token` on subject_token | Subject token expired | Refresh subject_token before exchange |
| Circular `act` chain | Agent B delegates back to Agent A | Check `act` chain before calling exchange |

---

### OAuth 2.0 Authorization Code + PKCE {#oauth2-auth-code-pkce}

```yaml
id: mech:oauth2.auth-code-pkce
status: stable
category: user-delegated
tldr: "User authorizes agent via browser redirect; S256 PKCE mandatory in OAuth 2.1; produces short-lived access + refresh token pair."
rfc: "RFC 6749 §4.1; RFC 7636; OAuth 2.1 draft-15"
```

| Property | Value |
|---|---|
| Token type | JWT or Opaque |
| Lifetime | Access: 1h; Refresh: days–weeks |
| Holder-bound? | Optional (DPoP or mTLS) |
| Revocable? | Yes (refresh token revocation) |
| Requires user interaction? | Yes |
| Security assurance | Medium–High |

#### How OAuth 2.0 Authorization Code + PKCE works

1. Agent generates `code_verifier` (43–128 chars, high entropy) and `code_challenge = BASE64URL(SHA256(code_verifier))`
2. Agent sends user to authorization endpoint with `code_challenge` and `code_challenge_method=S256`
3. User authenticates and consents
4. AS returns `code` to redirect URI
5. Agent exchanges `code` + `code_verifier` at token endpoint
6. AS verifies `SHA256(code_verifier) == code_challenge` stored at step 2

**Critical:** `plain` PKCE method is **forbidden** (named anti-pattern AP-3). The `code_challenge` is in the URL and would be equivalent to the verifier.

**PAR enhancement (RFC 9126):** For MCP clients and high-assurance scenarios, send all authorization parameters via a POST to `/par` first, receive a `request_uri`, use only the URI in the redirect. Prevents parameter tampering in browser.

```http
# PAR request
POST /par HTTP/1.1
Content-Type: application/x-www-form-urlencoded

client_id=...&code_challenge=...&code_challenge_method=S256
&scope=read:data&redirect_uri=...

# Response
{ "request_uri": "urn:ietf:params:oauth:request_uri:...", "expires_in": 60 }

# Authorization redirect uses only request_uri
GET /authorize?client_id=...&request_uri=urn:ietf:params:oauth:request_uri:...
```

---

### OAuth 2.0 Device Authorization Grant {#oauth2-device-auth}

```yaml
id: mech:oauth2.device-auth
status: stable
category: user-delegated-headless
tldr: "For headless/browserless agents that need user consent. Agent displays a user code; user completes auth on a separate device."
rfc: "RFC 8628"
```

| Property | Value |
|---|---|
| Token type | JWT or Opaque |
| Lifetime | Access: 1h; Refresh: configurable |
| Requires user interaction? | Yes (on separate device) |
| Security assurance | Medium |

#### How OAuth 2.0 Device Authorization Grant works

See [Scenario 12](#scenario-device-grant) for complete wire format. The agent polls the token endpoint with `device_code` at the interval specified in the device authorization response. Polling with `authorization_pending` response is normal until user completes auth.

**DPoP extension for Device Grant:** An active IETF draft (`draft-parecki-oauth-dpop-device-flow`) adds DPoP binding to device grant tokens. Check draft status before implementing.

---

### OAuth 2.0 Rich Authorization Requests (RAR) {#oauth2-rar}

```yaml
id: mech:oauth2.rar
status: stable
category: fine-grained-auth
tldr: "Replaces scope strings with structured authorization_details objects for fine-grained per-request authorization."
rfc: "RFC 9396"
```

RAR is especially useful for agents that need to express specific actions or resource identifiers beyond what a scope string can convey.

```http
POST /token
Content-Type: application/x-www-form-urlencoded

grant_type=client_credentials
&authorization_details=[
  {
    "type": "file_access",
    "locations": ["https://storage.example.com/bucket/file.pdf"],
    "actions": ["read"],
    "datatypes": ["application/pdf"]
  }
]
```

---

### OAuth 2.0 Token Introspection {#oauth2-introspection}

```yaml
id: mech:oauth2.introspection
status: stable
category: validation
tldr: "Resource servers validate opaque tokens by querying the AS. JWT-encoded response available (RFC 9701)."
rfc: "RFC 7662; RFC 9701"
```

```http
POST /introspect HTTP/1.1
Authorization: Basic base64(rs_client_id:rs_client_secret)
Content-Type: application/x-www-form-urlencoded

token=<opaque_token>

# Response (active token)
HTTP/1.1 200 OK
{
  "active": true,
  "sub": "agent-client-id",
  "scope": "read:data",
  "exp": 1234567890,
  "iss": "https://as.example.com",
  "aud": "https://api.example.com"
}
```

**Caching strategy:** Cache active introspection responses for `min(60s, (exp - now) * 0.5)`. Cache inactive (revoked) responses for 5–10s max. Key: `hash(token_value)`.

---

## Part 3: OpenID Connect for Agents {#part-3-openid-connect-for-agents}

### OIDC Workload Identity Federation {#oidc-workload-identity-federation}

```yaml
id: mech:oidc-wif
status: stable
category: workload-identity
tldr: "Cloud providers trust external OIDC issuers. Workload presents OIDC token; cloud STS exchanges it for cloud-native credential. No secrets."
```

[Pattern: P1 — STS Token Exchange]

Each cloud provider implements its own WIF endpoint but the pattern is identical: register the external OIDC issuer's JWKS URL with the cloud provider, define trust conditions (typically filtering on `sub` or `aud` claims), then have workloads exchange their OIDC tokens for cloud-native tokens.

Trust establishment steps (GCP example):
1. Create Workload Identity Pool
2. Add OIDC provider with issuer URI + JWKS endpoint
3. Set attribute mapping (`google.subject = assertion.sub`)
4. Set attribute condition (`assertion.repository == "org/repo"`)
5. Grant pool identity permission to GCP service

---

### OIDC-A: OpenID Connect for Agents {#oidc-a}

```yaml
id: mech:oidc-a
status: experimental
category: agent-identity
tldr: "Proposed OIDC extension adding agent-specific claims. Not yet standardized. Track OpenID Foundation AI CG for status."
```

OIDC-A extends standard OIDC ID tokens with agent-specific claims to enable verifying the model, provider, and version of an AI agent at auth time.

**Proposed agent-specific claims:**
- `agent_model` — model identifier (e.g., `claude-opus-4-7`)
- `agent_provider` — provider (e.g., `anthropic`)
- `agent_version` — version string
- Capability assertions — what the agent is authorized to do
- Action/output binding metadata — links token to specific action

**Current status:** Proposal from OpenID Foundation AI CG (arxiv:2510.25819). Not adopted as formal spec. Implementations exist in Azure AI Foundry (`idtyp: "agent"`) and AWS Bedrock AgentCore as proprietary precursors. Monitor: https://openid.net/wg/

---

### OpenID Federation {#openid-federation}

```yaml
id: mech:openid-federation
status: stable
category: cross-domain-trust
tldr: "Decentralized trust fabric. Organizations publish HTTPS-based Trust Anchors. Agents can operate across org boundaries without a shared IdP."
rfc: "OpenID Federation 1.0"
```

OpenID Federation enables multi-organization trust chains. Each entity publishes an Entity Statement at its HTTPS identifier. Trust chains are resolved from Trust Anchor → Intermediate → Leaf, with each statement signed by the superior.

Useful for: agent-to-agent calls across organizational boundaries; partner integrations without shared identity infrastructure.

---

## Part 4: SPIFFE / SPIRE {#part-4-spiffe--spire}

### SPIFFE Identity Format {#spiffe-id-format}

```yaml
id: mech:spiffe-id
status: stable
```

A SPIFFE ID is a URI: `spiffe://trust-domain/path/to/workload`

Rules:
- Scheme: `spiffe` (lowercase)
- Trust domain: DNS-like label; max 255 chars; no port
- Path: hierarchical workload identifier; must start with `/`
- No query parameters or fragments
- Example: `spiffe://cluster.local/ns/production/sa/payment-agent`

---

### SPIFFE X.509-SVID (mTLS) {#spiffe-x509-svid}

```yaml
id: mech:spiffe-x509-svid
status: stable
category: workload-identity
tldr: "X.509 certificate with SPIFFE ID in SAN URI field. Used for mTLS. Auto-rotated by SPIRE every ~1h."
```

[Pattern: P2 — Workload Attestation] [Pattern: P3 — Automatic Credential Rotation]

| Property | Value |
|---|---|
| Token type | X.509 certificate |
| Lifetime | ~1 hour (SPIRE default; configurable) |
| Holder-bound? | Yes (private key never leaves workload) |
| Revocable? | Effective via short lifetime (OCSP unreliable at scale) |
| Requires user interaction? | No |
| Attestation mechanism | SPIRE node agent (k8s-sat, aws-iid, gcp-iit) |
| Rotation mechanism | Platform-automatic (SPIRE agent rotates and re-mounts) |
| Security assurance | Very High |

#### How SPIFFE X.509-SVID works

The SPIRE agent runs as a DaemonSet (Kubernetes) or system service. It attests the node using platform-specific methods (Kubernetes SAT, AWS instance identity document, GCP instance identity). For each workload, SPIRE generates a key pair, issues an X.509 certificate with the workload's SPIFFE ID in the SAN URI extension, and delivers it via the Workload API (Unix socket).

The workload receives its identity via the Workload API; it never manages keys directly.

#### SPIFFE X.509-SVID wire format

```
# The workload presents its certificate at the TLS handshake
# No Authorization header — identity is in the TLS client certificate's SAN
# Server reads: certificate.SubjectAltName.URI = "spiffe://cluster.local/ns/app/sa/agent"
```

#### SPIFFE X.509-SVID consumer: how to validate

1. Verify certificate is signed by a trusted CA in the trust bundle (SPIFFE Trust Domain)
2. Verify certificate is not expired
3. Extract SPIFFE ID from SAN URI field
4. Apply workload authorization policy based on SPIFFE ID path

#### SPIFFE X.509-SVID where used

Platforms: Kubernetes (Istio, Linkerd, Consul, standalone SPIRE), Nomad, AWS EKS, GKE
Frameworks: Envoy (TLS origination/termination), gRPC (peer authentication)

#### SPIFFE X.509-SVID when NOT to use

- Across organizations without shared SPIRE federation — use OpenID Federation or X.509 chains instead
- For user-delegated identity — SPIFFE is workload-only

---

### SPIFFE JWT-SVID {#spiffe-jwt-svid}

```yaml
id: mech:spiffe-jwt-svid
status: stable
category: workload-identity
tldr: "JWT issued by SPIRE with SPIFFE ID as sub. Used when mTLS is impractical (HTTP, gRPC without TLS termination in proxy)."
```

| Property | Value |
|---|---|
| Token type | JWT |
| Lifetime | Short (SPIRE default: 1h, configurable as low as 60s) |
| Holder-bound? | No (bearer JWT) |
| Revocable? | Short-expiry |
| Security assurance | High |

#### SPIFFE JWT-SVID claims reference

```json
{
  "sub": "spiffe://trust-domain/path/to/workload",
  "aud": ["https://api.example.com"],
  "exp": 1234567890,
  "iat": 1234564290
}
```

**Algorithm note:** RS256, ES256, or EdDSA. The `alg` header in the JWT **must** be validated against an explicit allowlist — AP-2 (RS256/HS256 confusion) is particularly dangerous here.

#### SPIFFE JWT-SVID consumer: how to validate

1. Verify `alg` against allowlist (reject `none`)
2. Fetch JWKS from SPIRE OIDC Discovery endpoint (`/.well-known/openid-configuration`)
3. Verify signature
4. Verify `sub` starts with `spiffe://expected-trust-domain/`
5. Verify `aud` contains this service's identifier
6. Verify `exp`

---

### SPIFFE OIDC Federation {#spiffe-oidc-federation}

SPIRE exposes an OIDC Discovery endpoint, making the SPIRE trust domain an OIDC issuer. Cloud providers can be configured to trust the SPIRE JWKS endpoint, enabling workloads to exchange JWT-SVIDs for cloud-native tokens via [Pattern: P1 — STS Token Exchange].

---

## Part 5: Token Binding Mechanisms {#part-5-token-binding-mechanisms}

### mTLS Client Authentication (RFC 8705 §2) {#mtls-client-auth}

```yaml
id: mech:mtls-client-auth
status: stable
category: client-authentication
tldr: "Client authenticates at token endpoint by presenting a TLS client certificate. The cert proves client identity without a shared secret."
rfc: "RFC 8705 §2"
```

This is **client authentication at the AS**. The AS records `cnf.x5t#S256` in the token to bind it to the certificate.

Distinct from: [RFC 8705 §3 Certificate-Bound Tokens](#mtls-certificate-bound) which binds tokens for use at resource servers.

---

### Certificate-Bound Access Tokens (RFC 8705 §3) {#mtls-certificate-bound}

```yaml
id: mech:mtls-certificate-bound
status: stable
category: token-binding
tldr: "Access token contains cnf.x5t#S256 (SHA-256 of client cert). Resource server verifies client still holds the private key."
rfc: "RFC 8705 §3"
```

| Property | Value |
|---|---|
| Holder-bound? | Yes |
| Binding enforcement | Resource server verifies client cert fingerprint matches `cnf.x5t#S256` in token |
| Hardware key support | Yes (key stored in HSM; cert is public) |
| Latency impact | TLS handshake overhead only |

```json
{
  "sub": "agent-client",
  "aud": "https://api.example.com",
  "cnf": {
    "x5t#S256": "bwcK0esc3ACC3DB2Y5_lESsXE8o9ltc05O89jdN-dg2"
  }
}
```

---

### DPoP (Demonstration of Proof-of-Possession) {#dpop}

```yaml
id: mech:dpop
status: stable
category: token-binding
tldr: "Client generates an ephemeral key pair per-session. Proves possession at each API call via a signed proof JWT. No TLS client cert required."
rfc: "RFC 9449"
```

| Property | Value |
|---|---|
| Token type | DPoP-bound JWT (Bearer-like but proof required) |
| Lifetime | Access token: 1h; DPoP proof: 60s (single-use) |
| Holder-bound? | Yes |
| Binding enforcement | AS binds token to public key; RS verifies proof at each call |
| Latency impact | One extra JWT signature per API call |
| Security assurance | High |

#### How DPoP works

```http
# Step 1: Token request includes DPoP header
POST /token HTTP/1.1
DPoP: <proof_jwt>
Content-Type: application/x-www-form-urlencoded

grant_type=client_credentials&...

# Step 2: AS issues DPoP-bound token (token_type: DPoP, not Bearer)
{ "token_type": "DPoP", "access_token": "eyJ..." }

# Step 3: API call requires both token and fresh proof
GET /resource HTTP/1.1
Authorization: DPoP <access_token>
DPoP: <fresh_proof_jwt>
```

#### DPoP proof JWT structure

```json
{
  "typ": "dpop+jwt",
  "alg": "ES256",
  "jwk": { "kty": "EC", "crv": "P-256", "x": "...", "y": "..." }
}
{
  "jti": "unique-per-request",
  "htm": "GET",
  "htu": "https://api.example.com/resource",
  "iat": 1234567890,
  "ath": "hash_of_access_token"
}
```

#### DPoP consumer: how to validate

1. Verify `typ` is `dpop+jwt`
2. Verify `alg` is not `none` and is in allowlist
3. Verify signature using `jwk` from DPoP proof header
4. Verify public key `jwk` matches `cnf.jkt` in access token
5. Verify `htm` matches HTTP method of request
6. Verify `htu` matches HTTP URI (without query/fragment)
7. Verify `ath` = `BASE64URL(SHA256(access_token))`
8. Verify `jti` not replayed within server-configured replay window (recommend 60s)
9. Verify `iat` within clock skew (recommend ≤60s)
10. Check for `DPoP-Nonce` challenge: if server issued a nonce, verify `nonce` claim

#### DPoP DPoP-Nonce challenge flow

```http
# Server challenges for nonce
HTTP/1.1 401 Unauthorized
WWW-Authenticate: DPoP error="use_dpop_nonce" error_description="..."
DPoP-Nonce: <server_generated_nonce>

# Client retries with nonce in DPoP proof
DPoP: <proof_jwt_with_nonce_claim>
```

#### DPoP support matrix

| Platform | Status |
|---|---|
| Keycloak | GA (v26.4+) |
| Spring Security | GA (v6.5+) |
| Auth0 | GA (March 2026) |
| Azure AD | GA |
| AWS Cognito | Not yet supported |

---

### Biscuits / Macaroons (Capability Tokens) {#biscuits-macaroons}

```yaml
id: mech:capability-tokens
status: experimental
category: capability-based
tldr: "Offline-attenuatable capability tokens. Holder can create restricted versions without contacting issuer. Ideal for high-velocity agent delegation chains."
```

| Property | Value |
|---|---|
| Token type | Biscuit (cryptographic) / Macaroon (HMAC chain) |
| Holder-bound? | Yes (attenuation proves possession) |
| Revocable? | No (offline; no revocation mechanism) |
| Requires user interaction? | No |
| Security assurance | High (online issuance) → degrades over delegation chain |

**Biscuits** embed cryptographic authority hierarchically. The holder can create a restricted version by adding caveats (e.g., "only read", "only until 2026-05-08T12:00:00Z"). The recipient cannot expand permissions.

**Macaroons** use contextual HMAC caveats. Third-party caveats allow discharging caveats via external services.

**Use case for agents:** When Agent A delegates to Agent B to Agent C, RFC 8693 token exchange requires three round-trips to an STS. Biscuits allow Agent A to attenuate once offline; Agents B and C further attenuate without network calls. Critical for sub-100ms latency delegation chains.

**Gap:** No integration with standard OAuth token validation. No cross-ecosystem revocation. Offline nature means compromised tokens cannot be invalidated before expiry. Compensate with short lifetimes.

---

## Part 6: Cloud Provider Workload Identity {#part-6-cloud-provider-workload-identity}

All three major cloud providers implement [Pattern: P1 (STS Token Exchange)] and [Pattern: P2 (Workload Attestation)].

### AWS Workload Identity {#aws-workload-identity}

#### AWS IRSA (IAM Roles for Service Accounts) {#aws-irsa}

```yaml
id: mech:aws-irsa
status: stable
tldr: "EKS pod annotated with IAM role ARN. kubelet mounts projected SA token; AWS SDK exchanges it for temporary IAM credentials."
```

| Property | Value |
|---|---|
| Attestation mechanism | Kubernetes projected SA token (OIDC) |
| Token lifetime | 1 hour (SA token); 1–12 hours (IAM credentials, configurable) |
| Rotation mechanism | Platform-automatic |

**Setup:** Annotate the Kubernetes service account:
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::ACCOUNT:role/my-agent-role
```

The AWS SDK (boto3, AWS SDK v2) automatically reads the projected token and exchanges it for STS credentials. [Pattern: P1 — STS Token Exchange]

#### AWS EKS Pod Identity {#aws-eks-pod-identity}

Successor to IRSA (IRSAv2). No annotation on service account; IAM role association configured at EKS cluster level via Pod Identity Agent. Simpler to manage at scale.

#### AWS Lambda Execution Roles {#aws-lambda-execution-role}

Lambda functions receive IAM credentials via IMDSv2. **IMDSv1 SSRF attack:** agents that fetch arbitrary URLs can be directed to `http://169.254.169.254/...` to steal the instance's IAM credentials.

**Mitigation:**
- Enforce IMDSv2 (PUT request required to obtain session token; hop-limit=1)
- Block requests to `169.254.169.254` from agent URL-fetch tool
- Use VPC endpoint with IMDSv2 hop-limit=1

```bash
# Enforce IMDSv2 at launch
aws ec2 modify-instance-metadata-options \
  --instance-id i-1234567890 \
  --http-put-response-hop-limit 1 \
  --http-endpoint enabled \
  --http-tokens required
```

#### AWS IAM Roles Anywhere {#aws-iam-roles-anywhere}

On-premises agents present X.509 certificates from on-prem PKI. AWS validates against a configured Trust Anchor (certificate CA), then issues temporary IAM credentials. See [Scenario 15](#scenario-on-prem-to-cloud).

---

### GCP Workload Identity {#gcp-workload-identity}

```yaml
id: mech:gcp-workload-identity
status: stable
tldr: "Kubernetes SA token or external OIDC token exchanged at GCP STS for GCP access token. GKE WI automates this transparently."
```

| Property | Value |
|---|---|
| Attestation mechanism | GKE metadata server or external OIDC |
| Token lifetime | 1 hour |
| Rotation mechanism | Platform-automatic |

**GKE Workload Identity:** annotate K8s SA + bind to GCP service account. GKE metadata server issues GCP tokens directly; no projected token exchange required.

**Critical header:** All requests to GCP IMDS must include `Metadata-Flavor: Google`. Requests without this header are rejected (SSRF protection).

**Trust establishment flow:**
1. Create WIF Pool + OIDC provider
2. Set attribute mapping and conditions
3. Grant WIF pool member access to GCP SA
4. Pod receives GCP token automatically via GKE metadata server

---

### Azure Managed Identity {#azure-managed-identity}

```yaml
id: mech:azure-managed-identity
status: stable
tldr: "Azure-managed identity for workloads. No credentials to manage. System-assigned or user-assigned."
```

| Property | Value |
|---|---|
| Attestation mechanism | Azure IMDS |
| Token lifetime | Typically 1 hour; auto-refreshed |
| Rotation mechanism | Platform-automatic |

**Critical header:** `Metadata: true` required on all IMDS requests.

**System-assigned:** bound to specific resource; deleted with resource.
**User-assigned:** standalone resource; can be bound to multiple workloads; survives resource deletion.

**Azure Federated Identity Credentials:** allows external OIDC tokens (GitHub Actions, K8s) to federate with Managed Identity. Enables cross-cloud workload identity.

```http
GET http://169.254.169.254/metadata/identity/oauth2/token
  ?api-version=2018-02-01
  &resource=https://management.azure.com/
Metadata: true
```

---

## Part 7: Kubernetes Service Account Tokens {#part-7-kubernetes-service-account-tokens}

### Legacy Kubernetes Tokens (Deprecated) {#k8s-legacy-tokens}

```yaml
id: mech:k8s-legacy-sa-token
status: deprecated
tldr: "Long-lived, no audience binding, auto-mounted as /var/run/secrets/kubernetes.io/serviceaccount/token. Security liability — disable."
```

**Detection:** `serviceAccountName: default` without `automountServiceAccountToken: false`, or absence of `automountServiceAccountToken: false` in pod spec.

**Fix:**
```yaml
spec:
  automountServiceAccountToken: false  # Disable globally
  # Or use projected tokens (below)
```

---

### Kubernetes Projected Service Account Tokens {#k8s-projected-sa-tokens}

```yaml
id: mech:k8s-projected-sa-token
status: stable
category: workload-identity
tldr: "1-hour audience-bound OIDC-compatible token. Auto-rotated by kubelet. The foundation of EKS IRSA, GKE WI, AKS WI."
```

[Pattern: P2 — Workload Attestation] [Pattern: P3 — Automatic Credential Rotation]

| Property | Value |
|---|---|
| Token type | JWT (OIDC-compatible) |
| Lifetime | 1 hour (default; configurable) |
| Holder-bound? | No |
| Revocable? | Short-expiry |
| Attestation mechanism | kubelet (k8s-api) |
| Rotation mechanism | Platform-automatic (kubelet rotates file) |

#### Kubernetes Projected Service Account Token claims

```json
{
  "iss": "https://oidc.eks.us-east-1.amazonaws.com/id/CLUSTER_ID",
  "sub": "system:serviceaccount:namespace:serviceaccount-name",
  "aud": ["sts.amazonaws.com"],
  "exp": 1234567890,
  "iat": 1234564290,
  "kubernetes.io": {
    "namespace": "production",
    "pod": { "name": "agent-7d9f4c", "uid": "abc-123" },
    "serviceaccount": { "name": "agent-sa", "uid": "def-456" }
  }
}
```

#### Kubernetes Projected Service Account Token configuration

```yaml
# In Pod spec
volumes:
  - name: token
    projected:
      sources:
        - serviceAccountToken:
            audience: sts.amazonaws.com   # Or GCP/Azure equivalent
            expirationSeconds: 3600
            path: token
```

**Do not cache at app layer** — the file path is the cache. Read the token at use time; kubelet has already rotated it if needed.

---

## Part 8: CI/CD OIDC Tokens {#part-8-cicd-oidc-tokens}

### GitHub Actions OIDC {#github-actions-oidc}

```yaml
id: mech:github-actions-oidc
status: stable
category: cicd-identity
tldr: "GitHub-issued OIDC token for each Actions job. Short-lived. Exchange for cloud credentials via WIF. No stored secrets needed."
```

| Property | Value |
|---|---|
| Issuer | `https://token.actions.githubusercontent.com` |
| Lifetime | ~10 minutes |
| Requires | `permissions: id-token: write` in workflow |

#### GitHub Actions OIDC claims

| Claim | Example | Notes |
|---|---|---|
| `sub` | `repo:org/repo:ref:refs/heads/main` | **Immutable format (numeric IDs) from 2026-04** |
| `iss` | `https://token.actions.githubusercontent.com` | |
| `aud` | `sts.amazonaws.com` (configurable) | |
| `repository` | `org/repo` | |
| `repository_owner` | `org` | |
| `workflow` | `deploy.yml` | |
| `ref` | `refs/heads/main` | |
| `sha` | `abc123...` | |
| `event_name` | `push` | |
| `job_workflow_ref` | `org/repo/.github/workflows/deploy.yml@refs/heads/main` | Reusable workflow identity |
| `runner_environment` | `github-hosted` | |
| `check_run_id` | `12345678` | Added 2025-11 |

#### GitHub Actions OIDC secure vs insecure trust policy

**Insecure (AP-7):** Any repo in the org can assume this role:
```json
{
  "Condition": {
    "StringLike": {
      "token.actions.githubusercontent.com:sub": "repo:myorg/*"
    }
  }
}
```

**Secure:** Only the specific repo + branch:
```json
{
  "Condition": {
    "StringEquals": {
      "token.actions.githubusercontent.com:sub": "repo:myorg/myrepo:ref:refs/heads/main",
      "token.actions.githubusercontent.com:aud": "sts.amazonaws.com"
    }
  }
}
```

#### GitHub Actions OIDC trust establishment for GCP

```yaml
# GCP Attribute Condition (not a wildcard)
attribute.repository == "myorg/myrepo"
# AND
attribute.ref == "refs/heads/main"
```

---

### GitLab CI OIDC {#gitlab-ci-oidc}

```yaml
id: mech:gitlab-ci-oidc
status: stable
category: cicd-identity
```

| Property | Value |
|---|---|
| Issuer | `https://gitlab.com` (or self-hosted URL) |
| Lifetime | ~5 minutes |
| Configuration | `id_tokens:` block in `.gitlab-ci.yml` |

#### GitLab CI OIDC configuration

```yaml
job:
  id_tokens:
    GCP_TOKEN:
      aud: https://iam.googleapis.com/projects/...
  script:
    - gcloud auth login --cred-file=$GOOGLE_APPLICATION_CREDENTIALS
```

#### GitLab CI OIDC claims

| Claim | Example |
|---|---|
| `sub` | `project_path:mygroup/myproject:ref_type:branch:ref:main` |
| `iss` | `https://gitlab.com` |
| `namespace_id` | `12345` |
| `project_id` | `67890` |
| `pipeline_id` | `987654` |
| `job_id` | `123456` |
| `ref` | `main` |
| `ref_type` | `branch` |

---

### CircleCI OIDC {#circleci-oidc}

```yaml
id: mech:circleci-oidc
status: stable
category: cicd-identity
```

| Property | Value |
|---|---|
| Issuer | `https://oidc.circleci.com/org/<org-uuid>` |
| `sub` | `org/<org-uuid>/project/<project-uuid>/user/<user-uuid>` |
| Token | `CIRCLE_OIDC_TOKEN` env var |

CircleCI uses org-UUID and project-UUID based identifiers rather than human-readable names. Trust policies must use UUIDs.

---

## Part 9: HTTP Authorization Headers {#part-9-http-authorization-headers}

Complete wire format reference. **Never place credentials in URL query parameters** — they appear in server logs, browser history, and HTTP referrer headers.

### `Authorization: Bearer` {#header-authorization-bearer}

```http
Authorization: Bearer eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9...
```

- RFC 6750
- `Bearer` keyword is case-insensitive per spec; use `Bearer` by convention
- Value may be JWT or opaque token
- Full header value after `Bearer ` must be base64url-safe characters only

---

### `Authorization: DPoP` + `DPoP:` {#header-authorization-dpop}

```http
Authorization: DPoP eyJhbGci...access_token...
DPoP: eyJhbGci...proof_jwt...
```

- RFC 9449
- Two headers required: `Authorization` carries the token; `DPoP` carries the proof
- `DPoP` header value is a compact JWS (three base64url segments separated by `.`)
- Proof JWT `typ` must be `dpop+jwt`
- `Authorization` keyword is `DPoP` (not `Bearer`) when token is DPoP-bound

---

### `Authorization: Basic` {#header-authorization-basic}

```http
Authorization: Basic base64(client_id:client_secret)
```

- RFC 7617
- **Only acceptable at the OAuth token endpoint** (never at resource servers)
- Value is `BASE64(username:password)` — colon-separated, NOT URL-encoded before base64
- **Do not use for agent-to-agent calls**; use `private_key_jwt` instead

---

### `Authorization: AWS4-HMAC-SHA256` (SigV4) {#header-sigv4}

```http
Authorization: AWS4-HMAC-SHA256 Credential=AKIAIOSFODNN7EXAMPLE/20130524/us-east-1/s3/aws4_request, SignedHeaders=host;x-amz-date, Signature=fe5f80f77d5fa3beca038a248ff027d0445342fe2855ddc963176630326f1024
```

- AWS-specific request signing
- AWS SDK handles this automatically using IMDS credentials
- Do not implement manually; use AWS SDK credential provider chain

---

### `X-API-Key:` {#header-x-api-key}

```http
X-API-Key: sk-proj-abc123...
```

- No RFC; de-facto standard for SaaS APIs
- Long-lived bearer credential — Anti-Pattern AP-6 risk
- Where unavoidable: store in secrets manager, rotate at least every 90 days, monitor usage

---

### `Signature:` / `Signature-Input:` (HTTP Message Signatures) {#header-http-message-sig}

```http
Signature-Input: sig1=("@method" "@path" "@authority" "content-type");created=1618884479;keyid="agent-key-2026"
Signature: sig1=:base64_encoded_signature:
```

- RFC 9421
- Signs a subset of HTTP fields; supports multiple signatures
- Used by Web Bot Auth for agent identity proofs
- Enables non-repudiation at the HTTP layer

---

### `Authorization: GNAP` {#header-gnap}

```http
Authorization: GNAP eyJhbGci...
```

- RFC 9635 (final, Aug 2024)
- Next-generation auth protocol with first-class delegation primitives
- Early adoption; monitor ecosystem before adopting

---

## Part 10a: OSS AI Agent Framework Auth {#part-10a-oss-ai-agent-framework-auth}

### LangGraph Auth {#framework-langgraph}

```yaml
id: framework:langgraph
token_refresh: manual
```

LangGraph provides auth hooks via decorator pattern. Developers implement custom authentication and authorization logic.

```python
from langgraph.auth import Auth

auth = Auth()

@auth.authenticate
async def authenticate(request: Request) -> str:
    """Validate bearer token; return user identity."""
    token = request.headers.get("Authorization", "").removeprefix("Bearer ")
    return validate_token(token)  # Developer implements

@auth.on
async def authorize_all(ctx: AuthContext, value: Any) -> None:
    """Authorization policy; raise PermissionError to deny."""
    if not has_permission(ctx.user, ctx.permissions):
        raise PermissionError("Insufficient permissions")
```

**LangGraph token refresh handling:** No — developer must implement token refresh. Use the [Part 17](#part-17-token-lifecycle-management) double-check locking pattern.

**For tool auth:** Federated Token Retrieval pattern — tool requests token from central store scoped to tool's audience.

---

### AutoGen / Microsoft Agent Framework Auth {#framework-autogen}

```yaml
id: framework:autogen
token_refresh: automatic
```

Microsoft Agent Framework (MAF) uses **Entra Agent ID** — the `agent` subtype of Entra service principal. The agent authenticates with Entra using a managed identity or federated credential; Entra issues a JWT with `idtyp: "agent"` and `azpacr: "2"` (high assurance).

**Zero-trust JWT pattern:**
```python
from azure.identity import ManagedIdentityCredential
from azure.core.credentials import TokenRequestContext

credential = ManagedIdentityCredential()
token = credential.get_token("https://graph.microsoft.com/.default")
# Token auto-refreshed by Azure Identity SDK
```

**Token refresh:** Yes — Azure Identity SDK handles automatically using Entra's refresh token rotation.

---

### CrewAI Auth {#framework-crewai}

```yaml
id: framework:crewai
token_refresh: manual
```

CrewAI uses environment variable API keys for LLM provider calls. For A2A (agent-to-agent), CrewAI implements Google's A2A protocol with `SimpleTokenAuth` for same-org calls or JWS-signed AgentCards for production cross-org.

```python
from crewai import Agent, Task, Crew
# LLM provider auth via env var (anti-pattern for production — use secrets manager)
# os.environ["ANTHROPIC_API_KEY"] = "..." ← Use Vault or Secrets Manager instead
```

**Token refresh:** No — manual. All credentials are long-lived env vars or API keys.

---

### Semantic Kernel Auth {#framework-semantic-kernel}

```yaml
id: framework:semantic-kernel
token_refresh: automatic
```

Semantic Kernel integrates natively with Azure Managed Identity via `DefaultAzureCredential`. Credential chain: Managed Identity → Azure CLI → VS Code → Environment variables.

```python
from semantic_kernel.connectors.ai.open_ai import AzureChatCompletion
from azure.identity import DefaultAzureCredential

kernel.add_service(
    AzureChatCompletion(
        deployment_name="gpt-4o",
        endpoint="https://myendpoint.openai.azure.com/",
        credential=DefaultAzureCredential()  # Auto-refreshed
    )
)
```

**Token refresh:** Yes — Azure Identity SDK automatic refresh.

---

### LlamaIndex Auth {#framework-llamaindex}

```yaml
id: framework:llamaindex
token_refresh: na_long_lived
```

LlamaIndex has no framework-level auth layer. All credentials are environment variables (API keys). **Flag as anti-pattern for production.** Wrap LlamaIndex agents in a credentials management layer that pulls from AWS Secrets Manager or HashiCorp Vault.

---

### Haystack Auth {#framework-haystack}

```yaml
id: framework:haystack
token_refresh: na_long_lived
```

Haystack uses pipeline build-time credential configuration via environment variables. Same production concern as LlamaIndex. Integrate with a secrets management layer.

---

## Part 10b: Cloud Agent Platform Auth {#part-10b-cloud-agent-platform-auth}

### Anthropic Claude / MCP Auth {#platform-anthropic-mcp}

```yaml
id: platform:anthropic-mcp
token_refresh: yes_sdk
```

Claude uses the MCP protocol for tool connections. Auth is OAuth 2.1 + PKCE with Resource Indicators (RFC 8707). See [Part 11](#part-11-model-context-protocol-mcp-auth) for full MCP auth details.

```json
// Claude Desktop / Claude Code mcp_servers config
{
  "mcp_servers": {
    "my-tool": {
      "url": "https://api.mytool.com/mcp",
      "headers": {
        "Authorization": "Bearer ${MY_TOOL_TOKEN}"
      }
    }
  }
}
```

---

### OpenAI Agents SDK Auth {#platform-openai-agents}

```yaml
id: platform:openai-agents
token_refresh: na_for_api_keys
```

The OpenAI Agents SDK uses `sk-proj-` scoped project API keys for direct API access. For tool connections via MCP, OAuth 2.1 is used. GPT Actions support three auth schemes: None, API Key, OAuth.

**Token refresh:** N/A for API keys (long-lived). For MCP OAuth: Yes-SDK.

---

### Google Agent Development Kit (ADK) {#platform-google-adk}

```yaml
id: platform:google-adk
token_refresh: yes_sdk
```

ADK uses structured `AuthScheme` + `AuthCredential` type system:

| `AuthScheme` type | Use case |
|---|---|
| `API_KEY` | Static API keys (avoid in prod) |
| `OAUTH2` | OAuth 2.0 flows (2LO or 3LO) |
| `SERVICE_ACCOUNT` | GCP service account JSON (avoid — use WI instead) |
| `OPEN_ID_CONNECT` | OIDC-based auth |

3LO (three-legged OAuth) with pause-and-resume: ADK pauses the agent, requests user authorization, resumes with new token after consent.

```python
from google.adk.auth import AuthConfig, OAuth2
config = AuthConfig(
    auth_scheme=OAuth2(
        authorization_endpoint="https://...",
        token_endpoint="https://...",
        scopes=["read:data"]
    )
)
# ADK manages token refresh automatically
```

---

### AWS Bedrock AgentCore {#platform-bedrock-agentcore}

```yaml
id: platform:bedrock-agentcore
token_refresh: yes_automatic
```

AgentCore Identity provides a Token Vault — KMS-encrypted credential store that manages token lifecycle automatically.

- **Inbound auth:** IAM SigV4 (machine-to-machine) or OIDC JWT (user-delegated)
- **Outbound auth:** 3LO (user-delegated) or 2LO (M2M) credentials stored in Token Vault
- **Token storage:** KMS-encrypted; per-agent namespaced; IAM-controlled access

```python
import boto3

bedrock_agentcore = boto3.client("bedrock-agentcore")
# AgentCore automatically manages token refresh via Token Vault
# No manual credential management required
```

---

### Azure AI Foundry / Entra Agent ID {#platform-azure-foundry}

```yaml
id: platform:azure-foundry
token_refresh: yes_automatic
```

Azure AI Foundry implements the **Entra Agent ID Blueprint** — agents are registered as `agent` subtype service principals in Microsoft Entra.

Key claims in Entra Agent tokens:
- `idtyp: "agent"` — identifies token as agent (not user or service principal)
- `azpacr: "2"` — assurance level 2 (highest; indicates managed identity or certificate auth)
- `oid` — agent's object ID in Entra
- `appid` — client app ID

**Federated Managed Identity:** agents use federated identity to authenticate to external systems without credentials.

---

### Vertex AI Agents {#platform-vertex-ai}

```yaml
id: platform:vertex-ai
token_refresh: yes_automatic
```

Vertex AI Agents use Application Default Credentials (ADC) credential chain:
1. `GOOGLE_APPLICATION_CREDENTIALS` env var (if set)
2. gcloud user credentials
3. GKE/GCE Workload Identity (preferred in cloud)
4. Google Cloud metadata server

Agent Engine (Vertex) creates dedicated service accounts per agent. Use Workload Identity Federation to bind these to external identity sources.

```python
from google.auth import default
credentials, project = default(scopes=["https://www.googleapis.com/auth/cloud-platform"])
# google-auth library handles refresh automatically
```

---

## Part 11: Model Context Protocol (MCP) Auth {#part-11-model-context-protocol-mcp-auth}

### MCP Auth Spec 2025-03-26 {#mcp-auth-2025}

```yaml
id: mech:mcp-oauth21
status: stable
tldr: "MCP mandates OAuth 2.1 + PKCE S256. Server advertises authorization server via Protected Resource Metadata (RFC 9728). Dynamic Client Registration optional."
```

**Core requirements (2025-03-26 spec):**
- OAuth 2.1 with PKCE (S256 mandatory)
- Protected Resource Metadata (RFC 9728) — server discovery via `/.well-known/oauth-protected-resource` or `WWW-Authenticate: Bearer resource_metadata="..."`
- Authorization Server advertised via `authorization_servers` field in resource metadata

**Protected Resource Metadata response:**
```json
{
  "resource": "https://api.mytool.com/mcp",
  "authorization_servers": ["https://auth.mytool.com"],
  "bearer_methods_supported": ["header"],
  "resource_documentation": "https://api.mytool.com/docs"
}
```

#### MCP `resource_metadata` SSRF risk

A malicious MCP server can point the `resource_metadata` URL or the `WWW-Authenticate` header's `resource_metadata` parameter at an attacker-controlled endpoint. This can:
1. Redirect the MCP client to a phishing authorization server
2. Cause the client to send tokens to the attacker

**Mitigations:**
- Maintain an allowlist of trusted authorization server issuers
- Validate that the discovered `issuer` matches a pre-configured trusted issuer
- Never follow redirects from resource metadata fetch
- Validate `issuer` field in authorization server metadata matches the URL fetched

---

### MCP Auth Spec 2025-06-18 {#mcp-auth-2025-06}

Resource Indicators (RFC 8707) become mandatory. The `resource` parameter must be sent in all authorization requests, scoping tokens to the specific MCP server.

```http
POST /authorize
...
resource=https://api.mytool.com/mcp&scope=tools:execute
```

---

### MCP Auth 2026 Developments {#mcp-auth-2026}

- **Role-based tool access** — OAuth scopes mapped to tool categories (e.g., `tools:read`, `tools:write`, `tools:execute`)
- **Incremental scope consent** — tools request additional scopes only when invoked
- **SD-JWT Agent Cards** — `draft-nandakumar-agent-sd-jwt`; selective disclosure of agent capabilities

---

### MCP Downstream Auth Gap

The MCP spec defines how clients authenticate to MCP servers. It does not define how MCP servers authenticate to their downstream platforms (databases, SaaS APIs, etc.). Each MCP server operator must implement this separately — typically using the patterns in Parts 2, 6, and 7.

---

## Part 12: Agent-to-Agent (A2A) Auth {#part-12-agent-to-agent-a2a-auth}

### Google A2A Protocol {#a2a-protocol}

```yaml
id: mech:a2a-jws
status: stable
category: agent-to-agent
tldr: "Agents publish AgentCards at /.well-known/agent.json. A2A calls are authenticated with OAuth 2.1 Bearer + optional JWS request signing."
```

#### A2A AgentCard format

```json
{
  "name": "My Research Agent",
  "description": "Retrieves and summarizes documents",
  "url": "https://agent.example.com/a2a",
  "version": "1.0.0",
  "capabilities": {
    "streaming": true,
    "pushNotifications": false
  },
  "authentication": {
    "schemes": ["Bearer"],
    "credentials": null
  },
  "skills": [
    {
      "id": "retrieve-document",
      "name": "Retrieve Document",
      "description": "...",
      "inputModes": ["text/plain"],
      "outputModes": ["application/json"]
    }
  ]
}
```

#### A2A JWS request signing

For cross-organizational calls, requests can be signed with JWS:
- Canonicalize request body using JSON Canonicalization Scheme (JCS, RFC 8785)
- Sign with RS256, ES256, or PS256
- Include signature in request

#### A2A trust bootstrap gap

On first contact with a new agent, there is no standardized way to verify the AgentCard's authenticity beyond DNS/TLS. **Mitigation:** Treat AgentCards as short-lived (max 1-hour cache TTL); re-fetch and re-verify on each session.

#### A2A AgentCard revocation gap

**The A2A spec (as of 2026) defines no revocation mechanism for AgentCards.** A compromised agent's card cannot be invalidated. **Mitigation:** Short TTL re-fetch policy (see above); out-of-band revocation lists; monitor for anomalous agent behavior.

#### A2A trust model comparison

| Model | Trust Anchor | Cross-org Support | Revocation |
|---|---|---|---|
| A2A (DNS/TLS) | DNS + TLS certificate | Yes (public DNS) | None (gap) |
| SPIFFE | SPIRE PKI | No (shared infra required) | Short lifetime |
| OAuth (IdP-anchored) | OAuth Authorization Server | Yes (federation) | Yes (token revocation) |
| OpenID Federation | HTTPS Trust Anchor | Yes | Entity statements can be updated |

---

## Part 13: CIBA and Async Authorization {#part-13-ciba-and-async-authorization}

### CIBA (Client Initiated Backchannel Authentication) {#ciba}

```yaml
id: mech:ciba
status: stable
category: async-user-auth
tldr: "Async user authorization. Agent submits backchannel auth request; user approves on a separate trusted device; agent polls/receives token."
rfc: "CIBA Core 1.0; FAPI CIBA Profile"
```

| Property | Value |
|---|---|
| Token type | JWT access token |
| Requires user interaction? | Yes (out-of-band, async) |
| Delivery modes | Poll, Ping, Push |
| Use case | Long-running agents; high-risk operations; async workflows |

#### How CIBA works

1. Agent submits backchannel auth request with `login_hint` and `binding_message`
2. AS sends auth request to user's registered device (push notification, SMS, email)
3. User reviews `binding_message` and approves/denies on trusted device
4. Agent receives token via poll (agent polls), ping (AS POSTs to agent callback), or push (AS delivers token directly)

```http
POST /bc-authorize HTTP/1.1
Content-Type: application/x-www-form-urlencoded

client_id=agent-client
&scope=read:medical-records
&login_hint=user@example.com
&binding_message=Agent requests access to lab results from Dr. Smith visit. Approve?

# Response
{
  "auth_req_id": "1c266114-a1be-4252-8ad1-04986c5b9ac9",
  "expires_in": 120,
  "interval": 5
}

# Poll (poll mode)
POST /token
grant_type=urn:openid:params:grant-type:ciba&auth_req_id=1c266114-...

# Authorization pending
{ "error": "authorization_pending" }
# After approval:
{ "access_token": "...", "token_type": "Bearer", "expires_in": 3600 }
```

#### CIBA binding_message guidance

The `binding_message` must be short (max 128 chars), human-readable, and describe the specific action being authorized. Do not use generic messages like "Agent needs access" — this leads to consent fatigue and approval without understanding.

---

## Part 14: Agent Lifecycle Management {#part-14-agent-lifecycle-management}

### Agent Provisioning with SCIM {#scim-agent-provisioning}

```yaml
id: mech:scim-agent
status: experimental
category: lifecycle
tldr: "OpenID Foundation proposal: first-class AgenticIdentity SCIM resource type. Enables enterprise provisioning/deprovisioning of agents via standard HR/IAM workflows."
```

The OpenID Foundation AI whitepaper proposes extending SCIM with an `AgenticIdentity` resource type. Unlike `User` or `Group`, `AgenticIdentity` includes:
- `agent_model` — model identifier
- `agent_provider` — provider
- `owners` — human/service accounts responsible for the agent
- `provisioning_state` — active / suspended / deprovisioned
- Capability assertions
- Associated identity credentials

**Enterprise provisioning pattern:**
1. HR/IAM triggers SCIM POST to create `AgenticIdentity` resource
2. SCIM server provisions agent in all registered systems
3. Agent receives workload identity (OIDC-A claims or Entra Agent ID)
4. On agent retirement: SCIM DELETE + SSF propagation

---

### De-provisioning vs. Revocation {#deprovisioning-vs-revocation}

These are architecturally distinct operations — confusion between them is a security risk.

| Operation | What it does | When to use |
|---|---|---|
| **Revocation** | Invalidates the current credential. Identity registration persists. Agent can obtain new credentials. | Credential rotation; suspected leakage of specific token |
| **De-provisioning** | Permanently removes the agent identity from all ACLs, databases, and systems. Cannot obtain new credentials. | Agent retirement; confirmed compromise; policy violation |

**Response to compromise:** De-provision (remove identity), not just revoke (invalidate credential). A revoked token can be re-issued to the same agent identity.

---

### Shared Signals Framework (SSF) for Agent De-provisioning {#ssf}

```yaml
id: mech:ssf
status: stable
category: lifecycle-propagation
tldr: "OpenID protocol for real-time security event propagation across federated domains. Enables near-real-time de-provisioning propagation."
rfc: "OpenID Shared Signals Framework"
```

SSF transmitters emit security events (agent suspended, agent deprovisioned, credential compromised). SSF receivers act on events within their domain.

**De-provisioning flow with SSF:**
1. Primary IAM issues SCIM DELETE
2. Primary IAM emits SSF event: `agent-deprovisioned` with agent ID
3. All subscribed SSF receivers (partner systems, MCP servers, A2A registries) receive event
4. Each receiver removes agent from its ACLs and invalidates active sessions

**OpenID Provider Commands:** SSF extension that allows direct session termination commands across federated domains.

---

## Part 15: Emerging Standards Tracker {#part-15-emerging-standards-tracker}

*Last verified: 2026-05-07. All drafts are subject to change. Check IETF datatracker for current status.*

| Standard | Status | Key Contribution | Last Verified |
|---|---|---|---|
| `draft-ietf-wimse-arch-07` | Active (expires Sep 2026) | Multi-system workload identity architecture | 2026-05-07 |
| `draft-ietf-wimse-workload-creds-00` | Active | Workload credential format | 2026-05-07 |
| `draft-ietf-wimse-mutual-tls-00` | Active | mTLS profile for WIMSE workloads | 2026-05-07 |
| `draft-rosenberg-oauth-aauth-00` | Active (May 2025) | Agentic Authorization OAuth 2.1 extension | 2026-05-07 |
| `draft-klrc-aiagent-auth-00` | Published Mar 2026 | AIMS: WIMSE + SPIFFE + OAuth composite | 2026-05-07 |
| `draft-oauth-ai-agents-on-behalf-of-user-02` | Active | `requested_actor` / `actor_token` params for agents | 2026-05-07 |
| `draft-ietf-oauth-identity-assertion-authz-grant-01` | Active | Identity assertion grant for agents | 2026-05-07 |
| `draft-parecki-oauth-dpop-device-flow` | Active | DPoP for Device Authorization Grant | 2026-05-07 |
| `draft-schwenkschuster-oauth-spiffe-client-auth` | Active | SPIFFE JWT-SVIDs as OAuth client auth | 2026-05-07 |
| `draft-nandakumar-agent-sd-jwt` | Active | SD-JWT-encoded Agent Cards | 2026-05-07 |
| RFC 9635 (GNAP) | **Final** (Aug 2024) | Next-gen auth protocol; delegation primitives | — |
| RFC 9700 | **Final** (Jan 2025) | OAuth 2.0 security BCP; refresh token rotation mandatory for public clients | — |
| OID4VCI 1.0 | **Final** (Sep 2025) | Verifiable credential issuance | — |
| W3C DID v1.1 | Implementation invitation (Mar 2026) | Self-sovereign agent identity | — |
| OIDC-A | Proposal (OpenID Foundation AI CG) | Agent-specific OIDC extension; `agent_model`, `agent_provider`, `agent_version` | 2026-05-07 |
| SCIM Agentic Identity Schema | Proposal (OpenID Foundation) | First-class agent entity in SCIM | 2026-05-07 |
| AuthZEN | OpenID WG (active) | Standardized PEP-PDP communication API | 2026-05-07 |
| Web Bot Auth | Early IETF proposal | HTTP-layer agent identity; HTTP Message Signatures | 2026-05-07 |
| IPSIE | OpenID WG (active) | Enterprise identity interoperability with agent requirements | 2026-05-07 |
| OpenID Federation 1.0 | **Final** | Decentralized trust; HTTPS-based identifiers; multi-org trust chains | — |
| FAPI 2.0 | **Final** | Financial-grade API security; sender-constrained tokens; strict consent | — |
| Biscuits | Implementations exist | Capability-based offline attenuation; OCap model | — |
| AP2 (Agent Payments Protocol) | Emerging | Cryptographically signed intent/cart mandates for commerce agents | 2026-05-07 |
| KYAPay | Emerging | Know Your Agent + payment authorization JWT | 2026-05-07 |
| C2PA | Standard | Tamper-evident provenance metadata; agent output binding | — |
| NIST SP 800-162 | **Final** | Attribute-Based Access Control; externalized authorization (PEP/PDP model) | — |

---

## Part 16: Security Anti-Patterns {#part-16-security-anti-patterns}

Eight named anti-patterns. Each appears in system-level threat models as `threat:AP-N`.

### AP-1: `alg: none` Acceptance {#ap-1}

```yaml
id: threat:alg-none
severity: critical
```

**What it looks like:** JWT library configured to accept `alg: none`; algorithm derived from token header without validation.

**Specific risk:** Attacker strips the signature from any JWT, sets `alg: none`, and the server accepts it as valid without verification.

**Detection:** Audit all JWT validation code for `verify=False`, `algorithms=None`, or equivalent. Test with a JWT where `alg: none` and signature is empty.

**Fix:** Explicitly specify the allowed algorithm list in every JWT validation call:
```python
# WRONG
jwt.decode(token, key)

# CORRECT
jwt.decode(token, key, algorithms=["RS256", "ES256"])
```

---

### AP-2: RS256 / HS256 Algorithm Confusion {#ap-2}

```yaml
id: threat:alg-confusion
severity: critical
```

**What it looks like:** Server accepts both RS256 and HS256 for the same key. Server derives algorithm from token header.

**Specific risk:** Attacker takes the server's RSA public key (publicly available), sets `alg: HS256`, and signs the token using the public key as an HMAC secret. Server verifies the HMAC using its own public key — verification passes.

**Detection:** Check whether your validation code uses the same key for both RS256 and HS256 paths.

**Fix:** Never derive algorithm from the token header. Hard-code the expected algorithm per key or per issuer:
```python
# WRONG — algorithm from token header
alg = jwt.get_unverified_header(token)["alg"]
jwt.decode(token, public_key, algorithms=[alg])

# CORRECT — algorithm from configuration
jwt.decode(token, public_key, algorithms=["RS256"])  # Never HS256 for asymmetric keys
```

---

### AP-3: PKCE `plain` Method {#ap-3}

```yaml
id: threat:pkce-plain
severity: high
```

**What it looks like:** `code_challenge_method=plain` in authorization request.

**Specific risk:** With `plain`, the `code_challenge` equals the `code_verifier`. The verifier is transmitted in the authorization URL. Any party observing the URL (proxy, log) can extract it and use it to exchange the code.

**Fix:** S256 only. Authorization servers must reject `plain` even when clients request it:
```python
# WRONG
code_challenge_method = "plain"
code_challenge = code_verifier

# CORRECT
code_challenge_method = "S256"
code_challenge = base64url(sha256(code_verifier))
```

---

### AP-4: Wildcard `aud` Validation {#ap-4}

```yaml
id: threat:wildcard-aud
severity: high
```

**What it looks like:** `aud` validation uses prefix match, substring match, or regex rather than exact string match.

**Specific risk:** Token issued for `https://api.example.com` is accepted by `https://api.example.com.evil.com`. Enables confused deputy attacks where a token meant for Service A is accepted by Service B.

**Fix:** Exact string match. If `aud` is an array, every value must be checked and at least one must match this service's identifier exactly:
```python
# WRONG
if expected_aud in token_aud:  # substring check

# CORRECT
if expected_aud in token_aud and isinstance(token_aud, list):
    # token_aud is ["https://api.example.com"]
    # expected_aud is "https://api.example.com"
    pass  # exact match
```

---

### AP-5: JWKS Fetched from `jku` / `x5u` JWT Headers {#ap-5}

```yaml
id: threat:jku-header
severity: critical
```

**What it looks like:** JWT validation code fetches JWKS from the `jku` (JWK Set URL) or `x5u` (X.509 Certificate URL) field in the JWT header.

**Specific risk:** Attacker generates their own key pair, publishes their own JWKS at any URL, sets `jku` in the token header to that URL, and signs the token with their private key. Vulnerable validator fetches the attacker's JWKS and successfully "validates" the forged token.

**Fix:** Always use a pre-configured JWKS endpoint from your trusted issuer registry. Never use JWKS URLs from the token itself:
```python
# WRONG
header = jwt.get_unverified_header(token)
jwks = requests.get(header["jku"]).json()  # Attacker-controlled!

# CORRECT
jwks = requests.get(TRUSTED_ISSUER_JWKS_ENDPOINT).json()  # Pre-configured
```

---

### AP-6: Long-Lived API Keys in Environment Variables {#ap-6}

```yaml
id: threat:long-lived-apikey-env
severity: high
```

**What it looks like:** `API_KEY=sk-...` in `.env`, Dockerfile `ENV`, or Kubernetes `env:` block; API keys in GitHub Actions `secrets:` as a substitute for OIDC.

**Specific risk:** Key exfiltration (e.g., via IMDSv1 SSRF, container breakout, or log leakage) results in indefinite compromise. Key is readable by any process in the container with the same UID.

**Fix:** Replace with workload identity where possible (Parts 6, 7, 8). Where an API key is unavoidable:
1. Store in secrets manager (AWS Secrets Manager, GCP Secret Manager, Azure Key Vault)
2. Rotate at least every 90 days; 30 days for high-sensitivity APIs
3. Monitor usage for anomalies
4. Use the minimum scope (no `*` permissions)

---

### AP-7: Wildcard CI/CD `sub` Conditions {#ap-7}

```yaml
id: threat:wildcard-cicd-sub
severity: high
```

**What it looks like:** IAM trust policy with `sub: "repo:org/*"` or `sub: StringLike: "repo:myorg/*"`.

**Specific risk:** Any repository in the organization can assume this cloud role. If an attacker forks any repo in the org or gains push access to any repo, they can access production cloud resources.

**Fix (AWS):**
```json
// INSECURE
{"StringLike": {"token.actions.githubusercontent.com:sub": "repo:myorg/*"}}

// SECURE
{
  "StringEquals": {
    "token.actions.githubusercontent.com:sub": "repo:myorg/myrepo:ref:refs/heads/main",
    "token.actions.githubusercontent.com:aud": "sts.amazonaws.com"
  }
}
```

---

### AP-8: Token Forwarding Without Audience Re-Validation {#ap-8}

```yaml
id: threat:token-forwarding
severity: high
```

**What it looks like:** Service A receives a token intended for it and forwards that same token when calling Service B. Service B accepts it.

**Specific risk:** Confused deputy — Service B should only accept tokens scoped to itself. If Service B accepts any valid token from the same AS, an attacker who obtains a token for any service in the AS's domain can call Service B.

**Fix:** Use RFC 8693 Token Exchange to obtain an audience-appropriate token for each downstream service:
```python
# WRONG — forwarding Service A's token to Service B
requests.get("https://service-b.example.com/api", headers={"Authorization": f"Bearer {service_a_token}"})

# CORRECT — exchange for a token scoped to Service B
service_b_token = sts.exchange(
    subject_token=service_a_token,
    resource="https://service-b.example.com",
    scope="service-b:read"
)
requests.get("https://service-b.example.com/api", headers={"Authorization": f"Bearer {service_b_token}"})
```

---

## Part 17: Token Lifecycle Management {#part-17-token-lifecycle-management}

### Proactive Refresh Strategy {#proactive-refresh}

**Rule:** Refresh at 80% of token lifetime, not on 401.

| Token type | Refresh approach | Notes |
|---|---|---|
| OAuth access token (short-lived) | Proactive at 80% lifetime | e.g., 48 min for 60-min token |
| OAuth refresh token | On access token refresh failure | Rotate on use (RFC 9700) |
| SPIFFE X.509-SVID | Not required — poll SPIRE Workload API continuously | SPIRE rotates automatically |
| K8s projected SA token | Not required — read file at use time | kubelet rotates the file |
| Cloud IMDS token | Read via SDK; SDK handles refresh | boto3, google-auth, Azure Identity |
| DPoP proof JWT | New proof per request (no caching) | `jti` must be unique |

---

### In-Memory Token Caching {#token-caching}

```python
# Cache key
cache_key = hashlib.sha256(
    f"{client_id}:{scope}:{resource}:{audience}".encode()
).hexdigest()

# Cache TTL
now = time.time()
remaining = token["exp"] - now
cache_ttl = min(60, remaining * 0.5)  # Never extend past exp

# Do NOT cache at app layer:
# - SPIFFE SVIDs (the file IS the cache)
# - Projected SA tokens (the file IS the cache)
# - DPoP proofs (single-use by design)

# Negative caching for revoked/invalid tokens
negative_ttl = min(10, 5)  # 5-10 seconds max
```

---

### Thread-Safe Token Stores {#thread-safe-tokens}

**Double-check locking pattern (single process, multi-thread):**
```python
import threading
import time

_token_cache = {}
_lock = threading.Lock()

def get_token(cache_key: str, fetch_fn) -> str:
    # Fast path: check without lock
    cached = _token_cache.get(cache_key)
    if cached and cached["exp"] - time.time() > 60:
        return cached["token"]

    # Slow path: acquire lock and re-check
    with _lock:
        cached = _token_cache.get(cache_key)
        if cached and cached["exp"] - time.time() > 60:
            return cached["token"]
        # Fetch new token
        new_token = fetch_fn()
        _token_cache[cache_key] = {"token": new_token["access_token"], "exp": new_token["exp"]}
        return new_token["access_token"]
```

**Multi-host fleet — distributed lock (Redis):**
```python
import redis

r = redis.Redis()
lock_key = f"token_refresh_lock:{cache_key}"
token_key = f"token:{cache_key}"

def get_token_distributed(cache_key: str, fetch_fn) -> str:
    cached = r.get(token_key)
    if cached:
        return cached.decode()
    # Try to acquire distributed lock (SETNX + TTL)
    if r.set(lock_key, "1", nx=True, ex=30):
        try:
            new_token = fetch_fn()
            r.setex(token_key, int(new_token["expires_in"] * 0.8), new_token["access_token"])
            return new_token["access_token"]
        finally:
            r.delete(lock_key)
    else:
        # Another host is refreshing; wait and retry
        time.sleep(0.5)
        return get_token_distributed(cache_key, fetch_fn)
```

**SDK-managed (preferred):** `boto3`, `google-auth`, `azure-identity` all implement thread-safe token caching internally. Do not reimplement token caching when using these SDKs.

---

### Backoff and Retry on Auth Failures {#auth-retry}

| HTTP Status | Error | Retriable? | Action |
|---|---|---|---|
| 429 Too Many Requests | Rate limited | Yes | Respect `Retry-After` header |
| 503 Service Unavailable | AS unavailable | Yes | Exponential backoff |
| Network timeout | | Yes | Exponential backoff |
| 400 `invalid_grant` | Refresh token expired/revoked | No | Re-initiate auth flow |
| 401 `invalid_client` | Client auth failed | No | Check credentials |
| 400 `invalid_scope` | Scope not registered | No | Fix scope |

**Backoff parameters:**
```python
initial_delay = 0.5  # seconds
multiplier = 2.0
max_delay = 30.0  # seconds
jitter = random.uniform(0, 0.1) * current_delay  # 10% jitter
```

---

## Part 18: Failure Mode Diagnostics {#part-18-failure-mode-diagnostics}

### Interpreting 401 WWW-Authenticate Responses {#wwwauthenticate-parsing}

```http
# Standard bearer error
WWW-Authenticate: Bearer realm="example", error="invalid_token", error_description="Token expired"

# Insufficient scope
WWW-Authenticate: Bearer error="insufficient_scope", scope="read:data write:data"

# Wrong audience
WWW-Authenticate: Bearer error="invalid_token", error_description="Token audience mismatch"

# DPoP nonce challenge
WWW-Authenticate: DPoP error="use_dpop_nonce"
DPoP-Nonce: <server_generated_nonce>

# MCP resource metadata discovery
WWW-Authenticate: Bearer resource_metadata="https://api.example.com/.well-known/oauth-protected-resource"
```

### 401 Diagnosis Decision Tree {#401-diagnosis}

```
1. Is token present?
   No → Add Authorization header
   Yes → Continue

2. Is token expired?
   Check: exp claim < current time (allow 60s skew)
   Yes → Refresh token and retry
   No → Continue

3. Is issuer correct?
   Check: iss claim == configured issuer (exact match)
   Wrong → Using wrong token for this service
   OK → Continue

4. Is audience correct?
   Check: aud claim contains this service's identifier (exact match)
   Wrong → Use RFC 8693 Token Exchange to get audience-appropriate token (AP-8)
   OK → Continue

5. Is algorithm correct?
   Check: alg header is in service's allowlist
   Wrong → AP-1 or AP-2 risk; check token issuance
   OK → Continue

6. Is scope sufficient?
   Check: scope/scp claim contains required permission
   No → Request additional scope or use different grant
   Yes → Continue

7. Is token opaque?
   Call RFC 7662 introspection endpoint
   active=false → Token revoked; re-obtain
   active=true → Check other claims

8. Is DPoP required?
   Error: use_dpop_nonce or missing DPoP header
   → Implement DPoP; include fresh proof JWT with each call
```

### Per-Mechanism Common Errors {#per-mechanism-errors}

| Mechanism | Symptom | Likely Cause |
|---|---|---|
| Client Credentials | `invalid_client` | Wrong `client_id`; `private_key_jwt` key mismatch |
| Client Credentials | `invalid_scope` | Scope not registered on client |
| Token Exchange | `invalid_request` + scope error | Requested scope exceeds subject token scope |
| PKCE flow | `invalid_grant` on code exchange | `code_verifier` doesn't match stored `code_challenge` |
| SPIFFE JWT-SVID | 401 with `invalid audience` | `aud` in JWT-SVID doesn't match service URL |
| GHA OIDC | AWS `InvalidIdentityToken` | OIDC issuer not configured in IAM OIDC provider |
| GHA OIDC | AWS `AccessDenied` | Trust policy `sub` condition doesn't match workflow |
| K8s projected token | `TokenReviewStatus: false` | Audience mismatch between projected token and API server |
| DPoP | `use_dpop_nonce` on first call | Server requires nonce; include `nonce` claim in DPoP proof |
| DPoP | `invalid_dpop_proof` | `htu` doesn't match request URL; `htm` wrong; `ath` mismatch |
| MCP OAuth | Redirect loop | `resource_metadata` SSRF; validate issuer allowlist |

---

## Part 19: JWT Validation Reference {#part-19-jwt-validation-reference}

### Complete RFC 7519 §7.2 Validation Sequence {#jwt-validation-sequence}

Execute in this order. Fail on first violation.

**Step 1: Algorithm validation (before signature check)**
```python
header = jwt.get_unverified_header(token)
if header["alg"] not in ALLOWED_ALGORITHMS:
    raise InvalidToken("Algorithm not in allowlist")
if header["alg"] == "none":
    raise InvalidToken("alg:none rejected unconditionally")
# Never accept alg from token if the caller specifies expected_alg
```

**Step 2: Signature verification**
```python
# Use pre-configured JWKS URL — NEVER from jku/x5u in token header
jwks = load_jwks_from_trusted_cache(issuer)
verified = jwt.decode(token, jwks, algorithms=ALLOWED_ALGORITHMS, options={"verify_signature": True})
```

**Step 3: `iss` exact match**
```python
if verified["iss"] != EXPECTED_ISSUER:
    raise InvalidToken(f"Issuer mismatch: {verified['iss']}")
```

**Step 4: `aud` exact match**
```python
aud = verified.get("aud", [])
if isinstance(aud, str):
    aud = [aud]
if THIS_SERVICE_IDENTIFIER not in aud:
    raise InvalidToken(f"Audience mismatch: {aud}")
```

**Step 5: `exp` with clock skew**
```python
now = time.time()
if verified["exp"] < now - CLOCK_SKEW_SECONDS:  # CLOCK_SKEW_SECONDS = 60
    raise InvalidToken("Token expired")
```

**Step 6: `nbf` with clock skew (if present)**
```python
if "nbf" in verified and verified["nbf"] > now + CLOCK_SKEW_SECONDS:
    raise InvalidToken("Token not yet valid")
```

**Step 7: `iat` within acceptable range (if present)**
```python
if "iat" in verified and verified["iat"] > now + CLOCK_SKEW_SECONDS:
    raise InvalidToken("Token issued in the future")
```

**Step 8: `jti` replay check (for DPoP proofs and high-assurance tokens)**
```python
if token_requires_jti_check(context):
    if jti_seen_recently(verified["jti"], replay_window_seconds=60):
        raise InvalidToken("Token replay detected")
    mark_jti_seen(verified["jti"])
```

**Step 9: Custom claims**
```python
# Scope check
required_scope = "read:data"
token_scope = verified.get("scp", verified.get("scope", ""))
if isinstance(token_scope, str):
    token_scope = token_scope.split(" ")
if required_scope not in token_scope:
    raise InsufficientScope(f"Required: {required_scope}")

# Agent claims (OIDC-A, Entra Agent ID)
if requires_agent_verification(context):
    if verified.get("idtyp") != "agent":
        raise InvalidToken("Expected agent token, got user token")
```

---

### Algorithm Reference {#jwt-algorithm-reference}

| Algorithm | Key type | Status | Notes |
|---|---|---|---|
| `RS256` | RSA (2048+ bit) | Common, stable | Vulnerable to confusion with HS256 (AP-2) |
| `RS384` | RSA (2048+ bit) | Common, stable | |
| `RS512` | RSA (2048+ bit) | Common, stable | |
| `ES256` | P-256 ECDSA | Recommended | Smaller keys; widely supported |
| `ES384` | P-384 ECDSA | High assurance | |
| `EdDSA` (Ed25519) | OKP | Recommended (2026+) | Not supported by all libraries; JWKS: `"kty":"OKP","crv":"Ed25519"` |
| `PS256` | RSA-PSS | A2A signing | |
| `HS256` | Symmetric | Only for symmetric setups | Never use with asymmetric keys (AP-2) |
| `none` | None | **REJECTED** | AP-1; reject unconditionally |

---

## Glossary {#glossary}

| Term | Definition |
|---|---|
| `term:holder-bound` | A token is holder-bound when the recipient must prove possession of a private key to use it (DPoP, mTLS certificate-bound). Contrast: bearer token (possession = authorization). |
| `term:m2m` | Machine-to-machine: service calling another service without a user in the loop. |
| `term:workload-identity` | An identity assigned to a running compute workload (pod, function, container) by its execution environment, rather than a human or application managing credentials. |
| `term:scope-attenuation` | Progressively narrowing the set of permissions at each delegation hop. A delegated token can never grant more permissions than the delegating token held. |
| `term:trust-domain` | A namespace in which identity assertions are trusted. SPIFFE trust domains are `spiffe://trust-domain/...`. OIDC issuers define trust domains by their `iss` value. |
| `term:sts` | Security Token Service. A service that validates one credential and issues a new one, potentially with different scope, audience, or format. |
| `term:wif` | Workload Identity Federation. A cloud provider feature that trusts external OIDC issuers, enabling workloads to exchange OIDC tokens for cloud-native credentials. |
| `term:svid` | SPIFFE Verifiable Identity Document. Either an X.509 certificate (X.509-SVID) or a JWT (JWT-SVID) carrying a SPIFFE ID. |
| `term:dpop` | Demonstration of Proof-of-Possession. RFC 9449. HTTP-layer token binding via ephemeral key pair and per-request proof JWT. |
| `term:act-claim` | The `act` claim in a JWT identifies the actor (typically an agent) currently acting on behalf of the subject (`sub`). Nested `act` objects represent delegation chains. |
| `term:cnf-claim` | The `cnf` (confirmation) claim in a JWT binds the token to a public key (`cnf.jkt` for DPoP, `cnf.x5t#S256` for mTLS certificate-bound). |
| `term:pkce` | Proof Key for Code Exchange (RFC 7636). S256 method mandatory. Prevents auth code interception attacks in public clients. |
| `term:par` | Pushed Authorization Requests (RFC 9126). Send auth parameters via back-channel POST; use resulting `request_uri` in redirect. Prevents parameter tampering. |
| `term:rar` | Rich Authorization Requests (RFC 9396). Structured `authorization_details` objects replace flat scope strings for fine-grained authorization. |
| `term:ciba` | Client Initiated Backchannel Authentication. Async user authorization flow. Agent submits auth request; user approves on separate device. |
| `term:ssf` | Shared Signals Framework. OpenID protocol for real-time security event propagation across federated domains. |
| `term:authzen` | OpenID WG standardizing PEP-PDP communication API for externalized authorization decisions. |
| `term:deprovisioning` | Permanent, complete removal of an agent identity from all systems. Distinct from revocation (credential invalidation while identity persists). |
| `term:irsa` | IAM Roles for Service Accounts. AWS EKS feature binding K8s service accounts to IAM roles via OIDC token exchange. |
| `term:ocap` | Object-Capability security model. Access controlled by unforgeable token representing the permission itself, rather than identity + ACL lookup. Biscuits and Macaroons implement OCap. |
| `term:fapi` | Financial-grade API (FAPI 2.0). OpenID profile requiring sender-constrained tokens (DPoP or mTLS), PAR, and strict consent for high-value transactions. |

---

*Document end. Schema version: 2.0 | Data version: 1.0.0 | Last verified: 2026-05-07*

*YAML companion: `docs/agentic-identity-reference.yaml` (authoritative)*
*JSON Schema: `docs/agentic-identity-reference.schema.json`*
*Eval harness: `docs/evals.yaml`*
