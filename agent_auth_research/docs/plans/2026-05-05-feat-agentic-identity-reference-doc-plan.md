---
title: "feat: Agentic Identity Reference Document"
type: feat
date: 2026-05-05
deepened: 2026-05-06
tags: [agents, identity, auth, oauth, oidc, spiffe, mcp, a2a, jwt, mtls, wimse, oidca, scim, ciba, fapi]
---

# feat: Agentic Identity Reference Document

## Enhancement Summary

**Deepened on:** 2026-05-06
**Research agents used:** Security Sentinel, Architecture Strategist, Agent-Native Reviewer, Pattern Recognition Specialist, OAuth Deep Dive, Spec Flow Analyzer, OpenID Foundation paper (arxiv:2510.25819)
**Sections enhanced:** All 14 original + 7 new sections added
**Research sources:** 60+ cited sources across IETF drafts, OpenID specs, cloud provider docs, framework docs

### Key Improvements from Deepening

1. **Critical: Composition layer added** — new Part 0 (Scenario Recipes) closes the document's test-query gap (multi-hop e.g. GitHub Actions → MCP)
2. **Critical: OpenID paper gaps filled** — OIDC-A, SCIM for agents, Biscuits/Macaroons, CIBA, AuthZEN, SSF, FAPI 2.0, Web Bot Auth, intent-based auth, scope attenuation — none were in original plan
3. **Critical: Security anti-patterns section added** — 8 named anti-patterns with detection/remediation (including `alg:none`, RS256/HS256 confusion, PKCE `plain`, wildcard `aud`)
4. **Critical: Token lifecycle management section added** — proactive refresh, caching with TTL, thread-safe token stores, backoff/retry
5. **Critical: Agent-native architecture fixed** — mechanism subheadings renamed to include mechanism name for RAG; per-section YAML frontmatter blocks; `scenarios:` YAML block; eval harness
6. **Important: 4 foundational patterns named** — STS Token Exchange, Workload Attestation, Automatic Credential Rotation, No Secrets in Code/Config
7. **Important: Consumer-side validation added** — JWT validation algorithm specifics, introspection, failure mode diagnosis per mechanism
8. **Important: YAML schema fully specified** — JSON Schema as deliverable, 15+ new required fields, relationship graph, compliance tags, deprecation lifecycle

### New Considerations Discovered (from OpenID Foundation paper)

- **Agent autonomy spectrum** — Assistants vs Semi-Autonomous vs Fully Autonomous changes which identity model applies
- **Scope attenuation** — offline capability tokens (Biscuits/Macaroons) are the only low-latency solution for deep delegation chains
- **De-provisioning vs. revocation** — these are architecturally distinct; SCIM DELETE + SSF propagation is the pattern
- **GUI/browser agent auth bypass** — computer-use agents bypass all API-layer auth; no standard exists yet
- **Multi-user agent support** — no popular protocol supports shared agents; ABAC intersection model needed
- **Execution-count constraints** — novel alternative to time-based token expiry for bounding revocation delay impact
- **Intent-based authorization** — NL instruction → machine-readable policy; solves consent fatigue at scale

---

## Overview

Produce a single, agent-consumable Markdown reference document — `docs/agentic-identity-reference.md` — that catalogs every agentic identity and authentication mechanism in active use as of 2026. The document is structured for both human reading and LLM/agent consumption: consistent YAML frontmatter per entry, stable heading hierarchy, machine-scannable summary tables, and a Scenario Recipes section enabling multi-hop query resolution.

**Why "agent consumable"**: An agent retrieving this document should be able to determine, given a deployment context (e.g., "EKS pod calling an MCP server on behalf of a user"), which identity mechanisms apply, which claims to expect, and what headers to send — without reading the entire document.

**Primary audience**: AI agents retrieving this as context, developers/architects building agentic systems.

---

## Problem Statement

The agentic identity space has exploded in 2025–2026. A developer or agent building integrations must navigate:
- 14+ distinct identity/auth mechanisms (OAuth grants, SPIFFE, mTLS, cloud workload identity, CI/CD OIDC, capability tokens, etc.)
- 12+ AI agent frameworks each with their own auth patterns
- 5+ cloud provider workload identity systems with superficially similar but subtly different behaviors
- Several active IETF drafts (WIMSE, AAuth, AIMS, On-Behalf-Of User, WIMSE mutual TLS) that are not yet in any framework
- Emerging agent-specific proposals (OIDC-A, SCIM for agents, AuthZEN, Web Bot Auth) at varying maturity levels
- Novel authorization patterns unique to agents (scope attenuation, intent-based auth, execution-count constraints) with no prior documentation

No single authoritative reference exists. This document fills that gap.

---

## Proposed Solution

A structured reference at `docs/agentic-identity-reference.md` + `docs/agentic-identity-reference.yaml` + `docs/agentic-identity-reference.schema.json` + `docs/evals.yaml` with:

1. **Scenario Recipes** — end-to-end multi-hop walkthrough for the top 15 real deployment scenarios
2. **Foundational Patterns** — 4 named patterns that repeat across all mechanisms
3. **Quick Reference card** — single table at document top: mechanism | token type | holder-bound | requires user | primary use case | wire format prefix
4. **Per-mechanism deep dives** — consistent schema including both producer-side and consumer-side
5. **Agent Framework matrix** — OSS frameworks vs. cloud agent platforms (split)
6. **Protocol deep dives** — MCP, A2A, CIBA, OpenID Federation
7. **Emerging standards** — OIDC-A, SCIM for agents, AuthZEN, Web Bot Auth, FAPI 2.0, AP2/KYAPay, Biscuits/Macaroons
8. **Security anti-patterns** — 8 named anti-patterns with detection/remediation
9. **Token lifecycle management** — proactive refresh, caching, thread safety, backoff
10. **Failure mode diagnostics** — troubleshooting guide per mechanism
11. **YAML + JSON Schema** — machine-readable with full relationship graph, compliance tags, scenarios
12. **Eval harness** — 25+ query/answer pairs for CI validation

---

## Technical Approach

### Document Schema (per identity mechanism section)

Each mechanism section uses a consistent sub-structure for both human and agent consumption. **Critical: subheadings include the mechanism name** (e.g., `### How OAuth 2.0 Client Credentials works`) so RAG-extracted chunks self-identify.

Each section opens with a self-identifying YAML block:

```markdown
## OAuth 2.0 Client Credentials {#oauth2-client-credentials}

```yaml
id: mech:oauth2.client-credentials
status: stable
category: m2m
tldr: "Client authenticates with client_id + secret; receives Bearer token; no user involved."
```

> **TL;DR:** [one sentence]

| Property | Value |
|---|---|
| Token type | JWT / Opaque / X.509 / API Key |
| Lifetime | 60s / 1h / long-lived |
| Holder-bound? | Yes (mTLS/DPoP) / No (bearer) |
| Revocable? | Yes / Short-expiry only |
| Requires user interaction? | Yes / No |
| Attestation mechanism | Platform-IMDS / SPIRE-node / None |
| Rotation mechanism | Platform-automatic / SDK-managed / Manual / None |
| Security assurance | Low / Medium / High / Very High |
| Use case category | M2M / User-delegated / Workload / CI-CD |

### How [Mechanism] works
### [Mechanism] wire format / HTTP headers
### [Mechanism] claims reference
### [Mechanism] producer: how to obtain
### [Mechanism] consumer: how to validate
### [Mechanism] failure modes and diagnosis
### [Mechanism] where used (frameworks, runtimes, platforms)
### [Mechanism] when NOT to use
### [Mechanism] security characteristics and threat model
### [Mechanism] related RFCs / specs
### [Mechanism] decision criteria
```

### File Structure

```
docs/
  agentic-identity-reference.md        # Main reference document
  agentic-identity-reference.yaml      # Machine-readable companion (YAML, authoritative)
  agentic-identity-reference.schema.json  # JSON Schema for YAML validation
  evals.yaml                           # 25+ query/answer eval tuples for CI
```

**Architecture**: The YAML companion is the authoritative source. The Markdown property tables are generated from (or kept in sync with) the YAML via a CI check that diffs parsed YAML fields against Markdown tables. This prevents the drift risk called out in the risk register.

---

## Document Sections (Implementation Plan)

### Part 0: How to Use This Document (Agent Navigation Guide)

```
docs/agentic-identity-reference.md → Part 0
```

A meta-section for agents consuming this document as context:
- For a specific scenario → start at Part 1 Scenario Recipes or the `scenarios:` YAML block
- For a single mechanism lookup → use `mechanisms[id]` in the YAML or jump to section anchor
- For "what does runtime X support" → Part 1 decision tables
- For wire format → `mechanisms[id].wire_format_examples` (complete HTTP examples)
- For "is this mechanism stable enough to ship" → check `mechanisms[id].status` and `last_verified`
- YAML companion is authoritative; Markdown is the human-readable rendering

### Part A: Foundational Patterns

A named-patterns section before Part 1. Defines 4 patterns referenced by all subsequent sections:

- **Pattern P1: STS Token Exchange** — a workload presents a credential to a Security Token Service, which issues a new credential scoped to that environment. Used in: AWS IRSA, GCP WIF, SPIFFE OIDC federation, MCP resource indicators, RFC 8693. Callout `[Pattern: STS Token Exchange]` appears in each section using it.
- **Pattern P2: Workload Attestation** — a trusted system in the execution environment (hypervisor, node agent, orchestrator) performs out-of-band attestation and issues or proxies credentials. Appears in: AWS IMDS, GCP metadata server, SPIRE node attestation, Kubernetes projected SA tokens.
- **Pattern P3: Automatic Credential Rotation** — the workload never holds a long-lived secret; a local agent or platform rotates credentials before expiry and exposes them at a well-known path. Appears in: SPIFFE X.509-SVIDs, projected SA tokens, cloud workload tokens.
- **Pattern P4: No Secrets in Code or Config** — the guiding architectural principle. Anti-patterns explicitly labeled as violations. Referenced in acceptance criteria.

**Guiding Principles box** (precedes the flowchart): No Secrets in Code/Config, Prefer Short-Lived Credentials, Bind Tokens to the Holder When Possible, Validate All Claims. Cross-referenced throughout.

### Part 1: Decision Guide (Agent Entry Point)

```
docs/agentic-identity-reference.md → Part 1
```

Four lookup tables + a Mermaid decision flowchart + a parallel `decision_tree:` YAML block:

**Table 1: By deployment platform → identity mechanism**
Covers: GKE, EKS, AKS, Lambda, Cloud Run, Container Apps, Nomad, bare metal, GitHub Actions, GitLab CI, CircleCI, local/dev

**Table 2: By agent framework → native auth mechanism**
All 12 frameworks with token refresh handling column (Yes-automatic / Yes-SDK / No-manual / N/A-long-lived-key)

**Table 3: By use case ("I need to...") → mechanism(s)**
Examples:
- Delegate user context downstream → RFC 8693 Token Exchange + `act` claim
- Call external API machine-to-machine with no user → Client Credentials
- GitHub Actions job to call MCP server as machine identity → GHA OIDC + Token Exchange → OAuth 2.1 Bearer
- Cross-cloud call (AWS → GCP) → SPIFFE JWT-SVID federation or GCP WIF with AWS IdP
- CI pipeline to deploy without stored credentials → CI/CD OIDC + Cloud WIF
- Deep multi-agent delegation chain with low latency → Biscuits/Macaroons (offline attenuation)
- Migrate from API keys → workload identity path

**Table 4: By security assurance level → mechanisms**
Low (long-lived bearer API key) → Medium (short-lived JWT) → High (short-lived + holder-bound) → Very High (hardware-attested + holder-bound)

**Mermaid flowchart** (human-readable) + **`decision_tree:` YAML block** (agent-parseable):
```yaml
decision_tree:
  - id: rule-gha-mcp-machine
    if: {runtime: github_actions, target_type: mcp_server, identity_kind: machine}
    then: {scenario: github-actions-to-mcp-server}
  - id: rule-eks-pod-external-api
    if: {runtime: eks_pod, target_type: external_api, identity_kind: machine}
    then: {scenario: eks-pod-to-external-api}
```

**Trust Domain Taxonomy** (from OpenID Foundation paper):
- Single Trust Domain — centralized IdP, synchronous ops, shared infrastructure
- Cross-Domain Federation — multiple IdPs, async chains, RFC 8693 + Identity Assertion Grant
- Decentralized/Open Web — no central authority, CIBA, Web Bot Auth, A2A

**Agent Autonomy Spectrum** (classification for identity model selection):
- Assistants — human guides workflow; delegated user sub-identity
- Semi-autonomous — chain-of-thought with human checkpoints; enhanced service account
- Fully autonomous — open-ended task completion with delegation; sovereign agent identity

### Part 1b: Scenario Recipes

The composition layer — end-to-end multi-hop walkthroughs for the top 15 deployment scenarios.

Each recipe: which mechanism produces initial identity → what exchange happens → what credential reaches target → exact headers sent.

**Recipes to include:**
1. GitHub Actions workflow → MCP server (machine identity)
2. EKS pod → Anthropic API + private MCP server (parallel credentials)
3. Kubernetes pod → cross-cloud GCP service
4. Lambda function → DynamoDB + external API
5. LangGraph agent → multi-tool with different auth per tool
6. Agent A (user-delegated) → delegate to Agent B (RFC 8693 chain)
7. Three-hop delegation: User → Agent A → Agent B → Agent C with scope attenuation
8. CI/CD pipeline → cloud infra deployment (no stored secrets)
9. Browser-automation agent (cookie + session management)
10. Cross-organizational A2A call (partner agent verifying identity)
11. Agent migrating from API keys to workload identity (step-by-step)
12. Headless agent needing user consent (Device Authorization Grant)
13. Long-running async agent with CIBA mid-task re-authorization
14. High-value transaction with FAPI 2.0 (DPoP + mTLS)
15. On-premises agent calling cloud services (IAM Roles Anywhere / Workload Identity Federation)

**`scenarios:` YAML block** keyed by `(source_runtime, target_type, identity_kind)`.

### Part 2: OAuth 2.0 Grants for Agents

Mechanisms:
- **Client Credentials (RFC 6749 §4.4)** — with `private_key_jwt` client auth variant (RFC 7523); complete token exchange request/response pair
- **Token Exchange (RFC 8693)** — complete worked example of delegation chain; `act` claim structure; scope attenuation enforcement; chain depth limits; revocation in chains
- **Authorization Code + PKCE (RFC 7636 / OAuth 2.1)** — S256 mandatory; `plain` prohibited (named anti-pattern AP-7); state parameter for CSRF; PAR enhancement
- **Device Authorization Grant (RFC 8628)** — headless agents; DPoP draft extension for Device Grant
- **Rich Authorization Requests (RFC 9396)** — fine-grained scope for agents; `authorization_details` parameter
- **Pushed Authorization Requests (RFC 9126)** — exact flow, `request_uri`, `expires_in`; adoption status; when MCP clients should use PAR
- **Token Introspection (RFC 7662)** — opaque token validation; complete response format; caching strategies; RFC 9701 JWT-encoded introspection response

**Research insights added:**
- OAuth 2.1 vs 2.0 complete diff table (what's removed, what's tightened)
- Private key JWT (RFC 7523) complete claim spec: `iss`, `sub`, `aud`, `jti`, `exp`; how it differs from `client_secret_basic`
- Refresh token rotation (RFC 9700): token family invalidation on replay detection
- Multi-threaded agent refresh race condition patterns: double-check locking, file lock, distributed lock

### Part 3: OpenID Connect for Agents

- **Workload Identity Federation** — cloud providers trust external OIDC issuers; no secrets
- **ID Tokens as agent identity assertions** — portable, self-contained, short-lived
- **OIDC Discovery** — `/.well-known/openid-configuration` auto-discovery; mandatory in MCP spec
- **OIDC-A (OpenID Connect for Agents)** ← NEW from OpenID paper — proposed agent-specific extension; agent-specific claims: `agent_model`, `agent_provider`, `agent_version`; capability assertions; discovery mechanisms; action/output binding metadata; current status (draft/proposal)
- **OpenID Federation** ← NEW — decentralized trust fabric; HTTPS-based identifiers; multi-organization trust chains; cross-domain agent operation without shared IdP

### Part 4: SPIFFE / SPIRE

- **SPIFFE ID format** — `spiffe://trust-domain/path`; rules and encoding
- **X.509-SVID** — mTLS certificates; SAN-embedded SPIFFE ID; SPIRE auto-rotation; mTLS revocation assessment (OCSP unreliable; short certs are the real mitigation)
- **JWT-SVID** — complete claims spec: `sub`, `aud`, `exp`; algorithms (RS256/ES256/EdDSA); `alg` header validation required
- **OIDC Federation** — SPIRE JWKS trusted by cloud providers; STS exchange flow
- **IETF drafts** — `draft-schwenkschuster-oauth-spiffe-client-auth` (SPIFFE JWT as OAuth client auth)
- **Agent identity portability limitation** — SPIFFE is infrastructure-dependent and doesn't extend across organizations lacking shared infrastructure control; alternatives: OpenID Federation, X.509 chains, VCs, DIDs

Platforms: Kubernetes with Istio/Linkerd/Consul; Nomad; AWS EKS (SPIRE); GKE

### Part 5: Token Binding Mechanisms

Three distinct entries (not two):
- **mTLS Client Authentication (RFC 8705 §2)** — client cert at token endpoint; `private_key_jwt` alternative
- **Certificate-Bound Access Tokens (RFC 8705 §3)** — `cnf.x5t#S256` in token; resource server validates client holds private key; RFC 8705 also usable with Device Authorization Grant (no draft needed)
- **DPoP (RFC 9449)** — full validation requirements: `jti` uniqueness within replay window, `iat` clock skew, `htu`/`htm` binding, `ath` (access token hash), `DPoP-Nonce` challenge round-trip; support matrix (Keycloak 26.4, Spring Security 6.5, Auth0 GA March 2026); DPoP for Device Authorization Grant (draft)

Comparison table: all three on same axes — binding enforcement location (AS only vs AS+RS), what RS must verify, hardware key support, latency impact.

**NEW: Capability-Based Tokens (Biscuits / Macaroons)** ← from OpenID paper
- Offline attenuation without contacting issuer
- Object-Capability (OCap) security model
- Biscuits: hierarchical authority embedded cryptographically; holder creates restricted version
- Macaroons: contextual caveats added by delegation chain
- Use case: high-velocity agent networks where RFC 8693 STS latency is prohibitive
- Gap: no integration with existing web OAuth standards; offline revocation unsolved
- Cross-reference to Part A Pattern P1 (STS Token Exchange) — this is the offline alternative

### Part 6: Cloud Provider Workload Identity

All three follow Token Exchange (RFC 8693) under the hood:

**AWS:**
- IRSA (EKS), Lambda execution roles via IMDS, IAM Roles Anywhere (X.509 trust anchors), EKS Pod Identity (IRSAv2)
- **IMDSv1 SSRF attack** ← NEW SECURITY ADDITION — agents that fetch arbitrary URLs can be directed to `http://169.254.169.254/...` to steal the host cloud credential. IMDSv2 requirements: hop-limit=1, session token header mandatory. Critical threat for any agent with URL-fetching capability.

**GCP:**
- Workload Identity Federation; GKE Workload Identity; STS exchange endpoint; no key files
- `Metadata-Flavor: Google` header requirement

**Azure:**
- System-assigned vs user-assigned Managed Identity; Azure Federated Identity Credentials; EasyAuth sidecar
- `Metadata: true` IMDS header requirement

Per-provider: trust establishment flow (mirroring Part 8's CI/CD format), token exchange endpoint, resulting credential type, lifetime.

### Part 7: Kubernetes Service Account Tokens

- **Legacy tokens** (deprecated) — long-lived, no audience, auto-mounted, security liability; detection: `serviceAccountName: default` or absent; fix: `automountServiceAccountToken: false`
- **Projected service account tokens** — 1-hour default, audience-bound, auto-rotated, OIDC-compatible; full claim structure including `kubernetes.io` block
- **OIDC issuer configuration** — cluster as IdP; JWKS endpoint; cloud provider trust establishment
- **RBAC integration** — how RBAC binds to service account identity

### Part 8: CI/CD OIDC Tokens

Three providers with:
1. Full claim tables (all claims including recent additions)
2. How to request the token
3. How to validate
4. **Trust establishment flow for each cloud provider** ← added to match Part 6 format
5. **Secure vs insecure trust policy examples** ← named anti-pattern: wildcard `sub` conditions

**GitHub Actions OIDC:**
- Issuer, full claim table including `check_run_id` (2025-11) and new immutable `sub` format using numeric IDs (2026-04)
- Bad example (insecure): `sub: "repo:org/*"` — any repo in org can assume the role
- Good example (secure): `sub: "repo:org/specific-repo:ref:refs/heads/main"`

**GitLab CI OIDC:** full claim table, `id_tokens` config block, `sub` format

**CircleCI OIDC:** org-UUID issuer, UUID-based `sub`, custom V2 claims

### Part 9: HTTP Authorization Headers

Exact wire format reference for every header pattern (moved to appendix position but kept as named section):
- `Authorization: Bearer <jwt-or-opaque>` — RFC 6750; when opaque vs JWT
- `Authorization: DPoP <token>` + `DPoP: <proof-jwt>` — RFC 9449; full proof JWT structure
- `Authorization: Basic <base64(id:secret)>` — RFC 7617; only at token endpoint
- `Authorization: AWS4-HMAC-SHA256 ...` — AWS SigV4; request signing
- `X-API-Key: <key>` — defacto standard; SaaS APIs
- `Authorization: GNAP <token>` — RFC 9635; early adoption
- **`Signature: <sig>` / `Signature-Input: <params>`** ← NEW — HTTP Message Signatures (RFC 9421); used by Web Bot Auth for agent identity proofs

Includes: which RFCs define each, which frameworks emit each, **full validation requirements** per type, **never in URL query parameters** rule.

### Part 10a: OSS AI Agent Framework Auth

Per-framework detailed sections — **split from cloud platforms** (different integration model):

- **LangGraph** — `@auth.authenticate` + `@auth.on` pattern; `langgraph.json` config; Federated Token Retrieval for tool auth; token refresh: No-manual
- **AutoGen / Microsoft Agent Framework** — Entra Agent ID; "agent" subtype service principal; zero-trust JWT pattern; token refresh: Yes-automatic (Entra SDK)
- **CrewAI** — env var API keys; `SimpleTokenAuth` for A2A; JWS-signed AgentCards; token refresh: No-manual
- **Semantic Kernel** — Azure Managed Identity; `DefaultAzureCredential` chain; Foundry credential store; token refresh: Yes-automatic (Azure Identity SDK)
- **LlamaIndex** — env var API keys; no framework auth layer; token refresh: N/A-long-lived-key ← **flag as anti-pattern for production**
- **Haystack** — env var credentials; pipeline build-time configuration; token refresh: N/A-long-lived-key ← **flag as anti-pattern for production**

### Part 10b: Cloud Agent Platform Auth

- **Anthropic Claude / MCP** — `mcp_servers` headers config; OAuth 2.1 PKCE; resource indicators; token refresh: Yes-SDK
- **OpenAI Agents SDK** — `sk-proj-` scoped API keys; MCP OAuth 2.1 for tools; GPT Action auth (None/API Key/OAuth); token refresh: N/A for API keys
- **Google ADK** — `AuthScheme` + `AuthCredential` types; `API_KEY`, `OAUTH2`, `SERVICE_ACCOUNT`, `OPEN_ID_CONNECT`; 3LO pause-and-resume; token refresh: Yes-SDK
- **AWS Bedrock AgentCore** — AgentCore Identity; Token Vault (KMS-encrypted); inbound IAM SigV4 + OIDC; outbound 3LO vs 2LO; token refresh: Yes-automatic (Token Vault)
- **Azure AI Foundry** — Entra Agent ID Blueprint; Federated Managed Identity; `idtyp: "agent"` claim; `azpacr: "2"` assurance; token refresh: Yes-automatic (Managed Identity)
- **Vertex AI Agents** — ADC credential chain; Workload Identity Federation; Agent Engine service accounts; token refresh: Yes-automatic (google-auth library)

### Part 11: Model Context Protocol (MCP) Auth

- **2025-03-26 spec** — OAuth 2.1 + PKCE mandatory; Protected Resource Metadata (RFC 9728); complete response format with required/optional fields; Dynamic Client Registration
- **2025-06-18 spec** — Resource Indicators (RFC 8707) mandatory; explicit Resource Server role
- **2026 developments** — Role-based tool access; incremental scope consent; SD-JWT Agent Cards draft
- **`resource_metadata` SSRF risk** ← NEW SECURITY ADDITION — malicious MCP server can point 401 challenge at attacker-controlled URL; mitigation: issuer allowlist, no-redirect policy, `issuer` field validation
- **MCP downstream auth gap** ← from OpenID paper — how MCP servers authenticate to their own downstream platforms is not yet standardized

### Part 12: Agent-to-Agent (A2A) Auth

- **Google A2A Protocol** — AgentCard format; `/.well-known/agent.json`; JWS signing (JCS canonicalization, RS256/ES256/PS256)
- **A2A trust bootstrap gap** — on first contact, how to establish partner's key is trusted; DNS/TLS anchoring; key rotation without downtime
- **AgentCard revocation gap** ← named explicitly — no revocation mechanism defined; mitigation: time-bounded re-fetch (treat cards as short-lived)
- **Privilege escalation in delegation chains** — RFC 8693 must enforce strict scope subset; circular delegation detection (`act` chain cycle check)
- **CrewAI A2A** — `SimpleTokenAuth`; JWS-signed AgentCards in production
- **Trust model comparison** — DNS/TLS-anchored (A2A) vs PKI-anchored (SPIFFE) vs IdP-anchored (OAuth)

### Part 13: CIBA and Async Authorization ← NEW SECTION

**CIBA (Client Initiated Backchannel Authentication, CIBA Core 1.0 + FAPI CIBA Profile):**
- Use cases: long-running agent tasks that need mid-task re-authorization; high-risk operations; headless agents requiring user consent; asynchronous agent networks (Part 1 trust domain: Decentralized/Open Web)
- Three delivery modes: poll, ping, push
- `login_hint`, `binding_message` for user UX
- How the agent pauses workflow, requests user approval out-of-band on trusted device, resumes with new token
- MCP ecosystem integration status (partial; SEP-1036 URL mode as alternative)

### Part 14: Lifecycle Management ← NEW SECTION (formerly future consideration)

**Agent Provisioning:**
- **SCIM for agents** ← from OpenID paper — AgenticIdentity resource type; own attributes distinct from user attributes; owners/responsibility; provisioning state; de-provisioning via SCIM DELETE + SSF propagation
- Enterprise pattern: structured provisioning with SCIM; automated de-provisioning workflow
- Consumer pattern: user-initiated deletion; platform permission revocation

**De-provisioning vs. Revocation distinction** ← critical from OpenID paper:
- **Revocation** — invalidates current credential; identity registration persists
- **De-provisioning** — permanent and complete removal of agent identity from all ACLs and systems; cross-domain propagation via Shared Signals Framework (SSF)
- Response to compromise: de-provision (remove identity), not just revoke (invalidate credential)

**SSF (Shared Signals Framework)** ← NEW — OpenID protocol for security event propagation; near-real-time revocation events across federated domains; OpenID Provider Commands for direct session termination

### Part 15: Emerging Standards Tracker

**Extended from original Plan 13:**

| Draft/Standard | Status | Key Contribution |
|---|---|---|
| draft-ietf-wimse-arch-07 | Active (expires Sep 2026) | Multi-system workload identity architecture |
| draft-ietf-wimse-workload-creds-00 | Active | Workload credential format |
| draft-ietf-wimse-mutual-tls-00 | Active | mTLS for WIMSE workloads |
| draft-rosenberg-oauth-aauth-00 | Active (May 2025) | Agentic Authorization OAuth 2.1 extension |
| draft-klrc-aiagent-auth-00 | Published Mar 2026 | AIMS: WIMSE + SPIFFE + OAuth composite framework |
| draft-oauth-ai-agents-on-behalf-of-user-02 | Active | `requested_actor` / `actor_token` params |
| draft-ietf-oauth-identity-assertion-authz-grant-01 | Active | Identity assertion grant for agents |
| draft-parecki-oauth-dpop-device-flow | Active | DPoP for Device Authorization Grant |
| draft-schwenkschuster-oauth-spiffe-client-auth | Active | SPIFFE JWT-SVIDs as OAuth client auth |
| draft-nandakumar-agent-sd-jwt | Active | SD-JWT-encoded Agent Cards |
| RFC 9635 (GNAP) | Final (Aug 2024) | Next-gen auth protocol with delegation primitives |
| RFC 9700 | Final (Jan 2025) | Best Current Practice for OAuth 2.0 security |
| OID4VCI 1.0 | Final (Sep 2025) | Verifiable credential issuance for agents |
| W3C DID v1.1 | Implementation invitation (Mar 2026) | Self-sovereign agent identity |
| OIDC-A | Proposal (OpenID Foundation) | Agent-specific OIDC extension; `agent_model`, `agent_provider`, `agent_version` claims |
| SCIM Agentic Identity Schema | Proposal (OpenID Foundation) | First-class agent entity in IAM systems |
| AuthZEN | OpenID WG (active) | Standardized PEP-PDP communication API for agents |
| Web Bot Auth (IETF proposal) | Early | HTTP-layer agent identity; "passport for agents"; HTTP Message Signatures |
| SEP-1036 (MCP Enhancement) | Proposal | URL-mode elicitation for out-of-band interactions |
| IPSIE | OpenID WG (active) | Enterprise identity interoperability profiles with agent requirements |
| OpenID Federation | Final 1.0 | Decentralized trust fabric; HTTPS-based identifiers |
| FAPI 2.0 | Final | Financial-grade API security profile; sender-constrained tokens; strict consent |
| Biscuits / Macaroons | Implementations exist | Capability-based offline attenuation tokens; OCap model |
| AP2 (Agent Payments Protocol) | Emerging | Cryptographically signed intent/cart mandates for commerce agents |
| KYAPay | Emerging | Know Your Agent + payment authorization JWT |
| C2PA | Standard | Tamper-evident provenance metadata; agent output binding |
| NIST SP 800-162 | Final | Attribute-Based Access Control; PEP/PDP externalized authorization |

### Part 16: Security Anti-Patterns ← NEW SECTION

Eight named anti-patterns with: what it looks like, specific risk, detection method, replacement pattern.

**AP-1: `alg: none` acceptance** — critical; stripping signature bypasses verification; fix: explicit algorithm allowlist, reject `none` unconditionally

**AP-2: RS256/HS256 algorithm confusion** — attacker uses RSA public key as HMAC secret; fix: never derive algorithm from token header, always specify expected algorithm explicitly

**AP-3: PKCE `plain` method** — verifier is observable in authorization URL; fix: S256 only; servers must reject `plain` even when requested

**AP-4: Wildcard `aud` validation** — prefix/substring match instead of exact; fix: exact string match for every `aud` value

**AP-5: JWKS fetched from `jku`/`x5u` JWT headers** — attacker provides own JWKS; fix: always use pre-configured issuer JWKS endpoint, never from JWT header

**AP-6: Long-lived API keys in environment variables** — key exfiltration persists indefinitely; readable by any process; fix: workload identity for each platform; minimum rotation period

**AP-7: Wildcard CI/CD `sub` conditions** — any repo in org can assume the cloud role; fix: fully qualified `repo:org/repo:ref:refs/heads/main`; concrete bad/good IAM policy examples for AWS, GCP, Azure

**AP-8: Token forwarding without audience re-validation** — passing a token meant for service A to service B; confused deputy; fix: use RFC 8693 token exchange to obtain audience-appropriate token before forwarding

### Part 17: Token Lifecycle Management ← NEW SECTION

**Proactive Refresh:**
- Refresh at 80% of token lifetime (e.g., 48 minutes for a 60-minute token), not on 401
- Decision table by token type: which mechanisms support proactive refresh vs platform-managed vs on-demand only vs none (SPIFFE: poll SPIRE workload API continuously; projected SA tokens: read current file)

**In-Memory Token Caching with TTL:**
- Cache key: `hash(client_id + scope + resource + audience)`
- Cache TTL: `min(60s, (exp - now) * 0.5)` — never extend past `exp`
- Which tokens must NOT be cached at app layer (SPIFFE SVIDs, projected SA tokens — the file path IS the cache)
- Negative caching for revoked/invalid tokens: 5–10s max

**Thread-Safe Token Stores:**
- Double-check locking pattern (check expiry → acquire lock → check expiry again → refresh → release)
- File lock pattern for multi-process agents
- Distributed lock (Redis SETNX) for multi-host fleets
- AWS SDK `botocore` handles this; `google-auth` library handles this; most agent frameworks do not

**Backoff and Retry on Auth Failures:**
- Retriable: 429, 503, network timeout
- Non-retriable: 400 `invalid_grant`, 401 `invalid_client`
- Parameters: initial 500ms, factor 2x, max 30s, jitter
- Per-token-type failure mode when refresh endpoint unavailable

### Part 18: Failure Mode Diagnostics ← NEW SECTION

Troubleshooting guide per mechanism:

**How to interpret 401 WWW-Authenticate responses:**
- `error="invalid_token"` — expired, revoked, or malformed token
- `error="insufficient_scope"` — right token, missing scope
- `error="invalid_audience"` — token not issued for this service
- `error="use_dpop_nonce"` + `DPoP-Nonce:` header — server requires DPoP nonce; retry with nonce
- `resource_metadata="..."` — MCP-style; fetch metadata for discovery

**Decision tree for common 401 failures:**
1. Is the token expired? → check `exp` claim
2. Is the issuer correct? → check `iss` vs expected issuer
3. Is the audience correct? → check `aud` includes this service's identifier
4. Is the algorithm correct? → check `alg` header matches expected
5. Is the scope sufficient? → check `scp` or `scope` claim
6. Is the token opaque? → call introspection endpoint (RFC 7662)

**Per-mechanism failure modes:** (subsection per mechanism in Parts 2–9)

### Part 19: JWT Validation Reference ← NEW SECTION

Complete RFC 7519 §7.2 validation sequence:
1. `alg` header against explicit allowlist (reject `none`; reject RS256 when ES256 expected)
2. Signature verification using pre-configured JWKS endpoint (never `jku`/`x5u` from token)
3. `iss` exact match
4. `aud` exact match (the agent's own identifier)
5. `exp` with clock skew ≤60s
6. `nbf` with clock skew
7. `iat` within acceptable range
8. `jti` replay check (where applicable: DPoP proofs, CI/CD OIDC tokens for high-assurance)
9. Custom claims (scope, agent claims, delegation chain)

Algorithm notes: EdDSA (Ed25519) adoption; `"kty": "OKP"`, `"crv": "Ed25519"`, `"alg": "EdDSA"` in JWKS; not supported by all libraries.

### Part 20: YAML Companion Full Schema

The YAML schema is now a formal deliverable (`docs/agentic-identity-reference.schema.json`).

**Root-level required fields:**
```yaml
schema_version: "2.0"
data_version: "1.0.0"
last_verified: "2026-05-06"
id_registry: [...]  # All ever-published IDs with status: active|aliased|retired
```

**Per-mechanism required fields (extended):**
```yaml
mechanisms:
  - id: mech:oauth2.client-credentials
    name: "OAuth 2.0 Client Credentials"
    aliases: []
    status: stable  # stable | draft | deprecated | experimental
    rfc: "RFC 6749 §4.4"
    token_type: jwt_or_opaque
    lifetime: short
    holder_bound: false
    revocable: short_expiry
    requires_user: false
    attestation_mechanism: none  # platform-imds | spire-node | k8s-api | none
    rotation_mechanism: manual  # platform-automatic | sdk-managed | manual-refresh | none
    security_assurance: medium  # low | medium | high | very_high
    use_case: [m2m, backend_agents, scheduled_agents]
    frameworks: [langchain, crewai, openai_agents, google_adk, bedrock_agentcore, ...]
    runtimes: [kubernetes, lambda, cloud_run, container_apps]
    wire_format_examples:
      - description: "Token endpoint request"
        request: "POST /token\nContent-Type: application/x-www-form-urlencoded\n\ngrant_type=client_credentials&client_id=...&client_secret=..."
      - description: "API call with token"
        request: "GET /resource\nAuthorization: Bearer eyJhbGci..."
    claims:
      - name: sub
        required: true
        description: "Client identifier"
      - name: iss
        required: true
        description: "Authorization server identifier"
    pattern: [sts-token-exchange]
    relationships:
      extends: []
      composes_with: [mech:dpop, mech:mtls-client-auth]
      incompatible_with: []
    compliance_tags: [fips-140-2-compatible, fedramp-moderate]
    deprecation:
      status: active
      since: null
      sunset: null
      replacement: null
    production_readiness: ga
    implementation_complexity: 2  # 1-5
    last_verified: "2026-05-06"
    spec_version: "RFC 6749 + OAuth 2.1 draft-15"
    validation_steps:
      - "Verify alg header against allowlist"
      - "Verify signature using issuer JWKS"
      - "Verify iss exact match"
      - "Verify aud exact match"
      - "Verify exp with clock skew"
    common_errors:
      - error: "invalid_token"
        cause: "Token expired or revoked"
        fix: "Refresh token"
    anti_patterns:
      - "AP-6: Storing client_secret in env var without rotation"
```

**`scenarios:` block:**
```yaml
scenarios:
  - id: github-actions-to-mcp-server
    source_runtime: github_actions
    target: mcp_server
    identity_kind: machine
    steps:
      - mechanism: mech:github-actions-oidc
        produces: oidc_jwt
      - mechanism: mech:oauth2.token-exchange
        consumes: oidc_jwt
        produces: oauth2_bearer
      - mechanism: mech:mcp-oauth21
        wire_format: "Authorization: Bearer <oauth2_bearer>"
    references: [mech:github-actions-oidc, mech:oauth2.token-exchange, mech:mcp-oauth21]
```

**`glossary:` block** — IDs like `term:holder-bound`, `term:m2m`, `term:workload-identity`, `term:scope-attenuation` with definitions.

---

## Acceptance Criteria

### Functional Requirements

- [ ] Document covers all 20 sections with consistent section schema
- [ ] Every identity mechanism has: TL;DR, property table (including `attestation_mechanism`, `rotation_mechanism`, `security_assurance`), wire format, claims reference, producer flow, consumer validation, failure modes, where-used, when-NOT-to-use, decision criteria
- [ ] Part 0 includes decision tables covering all 4 dimensions (platform, framework, use-case, security-assurance)
- [ ] Part 1b includes all 15 scenario recipes with complete HTTP header examples
- [ ] All three CI/CD OIDC providers have complete claim tables AND cloud-side trust configuration examples (secure + insecure patterns)
- [ ] All 12 frameworks in Parts 10a/10b have dedicated subsections including token refresh handling column
- [ ] All 8 anti-patterns in Part 16 have: detection method, concrete example, and replacement pattern
- [ ] Token lifecycle management (Part 17) covers all 4 topics with per-mechanism strategy table
- [ ] Failure mode diagnostics (Part 18) covers all common 401/403 response types
- [ ] JWT validation reference (Part 19) includes complete 9-step sequence
- [ ] YAML companion (`docs/agentic-identity-reference.yaml`) has all required fields including new: `attestation_mechanism`, `rotation_mechanism`, `security_assurance`, `pattern[]`, `relationships{}`, `compliance_tags[]`, `validation_steps[]`, `common_errors[]`, `anti_patterns[]`, `scenarios[]`
- [ ] JSON Schema (`docs/agentic-identity-reference.schema.json`) validates all YAML entries; includes `enum` constraint on framework IDs and status values
- [ ] Eval harness (`docs/evals.yaml`) has 25+ query/answer tuples including all 15 scenario recipe queries
- [ ] All active IETF drafts in Part 15 have accurate status and expiry dates
- [ ] Mermaid decision flowchart AND parallel `decision_tree:` YAML block both present
- [ ] **OpenID Foundation paper concepts covered:** OIDC-A, SCIM for agents, Biscuits/Macaroons, CIBA, AuthZEN, SSF, FAPI 2.0, Web Bot Auth, intent-based auth, scope attenuation, de-provisioning vs. revocation, agent autonomy spectrum, execution-count constraints

### Non-Functional Requirements

- [ ] Markdown is valid CommonMark; passes `markdownlint`
- [ ] All external links verified (no 404s at time of writing)
- [ ] YAML companion validates against JSON Schema
- [ ] Agent readability: any section is self-contained (subheadings include mechanism name; per-section YAML frontmatter present)
- [ ] Each section has stable `id` anchor in namespaced URN format (`mech:`, `framework:`, `rfc:`, `claim:`, `header:`, `platform:`, `threat:`)
- [ ] **Single source of truth**: YAML fields and Markdown property tables are in sync; CI diff check enforces this

### Quality Gates

- [ ] Each mechanism section includes `### [Mechanism] consumer: how to validate` covering the complete validation sequence
- [ ] Each mechanism section includes `### [Mechanism] failure modes and diagnosis`
- [ ] Each JWT-based mechanism explicitly documents algorithm validation requirements
- [ ] All 8 anti-patterns explicitly labeled as insecure (not merely omitted)
- [ ] CI/CD OIDC sections include concrete secure vs. insecure trust policy condition examples
- [ ] MCP section documents `resource_metadata` SSRF risk with mitigations
- [ ] A2A section explicitly states AgentCard revocation is not defined in current spec
- [ ] Emerging standards section clearly distinguishes draft from final; each draft has version number and expiry date
- [ ] No security claim without a cited RFC, spec, or primary source

---

## Success Metrics

- An agent given "GitHub Actions workflow calling an MCP server as machine identity" can identify the full mechanism chain (GHA OIDC → Token Exchange → OAuth 2.1 Bearer) and the exact headers without reading the full document (via Part 1b recipe + YAML scenarios block)
- All 25+ eval harness queries return the correct mechanism(s) when tested with an LLM
- All claim structures match official documentation at time of publication
- Document remains accurate for at least 6 months (emerging standards tracker provides re-verification dates)

---

## Dependencies & Prerequisites

- No code dependencies — this is a documentation artifact
- Research completed as of 2026-05-06: comprehensive findings from security, architecture, agent-native, pattern, OAuth deep-dive, spec-flow agents + OpenID Foundation paper (arxiv:2510.25819)
- Key source documents to reference during writing:
  - OpenID Foundation paper: https://arxiv.org/html/2510.25819v1
  - MCP Authorization Spec (2025-03-26 and draft): https://modelcontextprotocol.io/specification/2025-03-26/basic/authorization
  - A2A Protocol Specification: https://a2a-protocol.org/latest/specification/
  - IETF WIMSE WG: https://datatracker.ietf.org/wg/wimse/about/
  - GitHub Actions OIDC reference: https://docs.github.com/actions/reference/openid-connect-reference
  - GitLab CI OIDC: https://docs.gitlab.com/ci/secrets/id_token_authentication/
  - CircleCI OIDC: https://circleci.com/docs/openid-connect-tokens/
  - AWS AgentCore Identity: https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/runtime-oauth.html
  - Azure Foundry Agent Identity: https://learn.microsoft.com/azure/foundry/agents/concepts/agent-identity
  - SPIFFE Concepts: https://spiffe.io/docs/latest/spiffe-about/spiffe-concepts/
  - Google ADK Auth: https://google.github.io/adk-docs/tools-custom/authentication/
  - RFC 9700: https://datatracker.ietf.org/doc/rfc9700/
  - RFC 9449 (DPoP): https://datatracker.ietf.org/doc/html/rfc9449
  - RFC 9728 (Protected Resource Metadata): https://datatracker.ietf.org/doc/html/rfc9728
  - RFC 7523 (JWT Bearer): https://datatracker.ietf.org/doc/html/rfc7523
  - AuthZEN WG: https://openid.net/wg/authzen/
  - OpenID Federation: https://openid.net/specs/openid-federation-1_0.html
  - CIBA Core: https://openid.net/specs/openid-client-initiated-backchannel-authentication-core-1_0.html
  - SSF: https://openid.net/wg/sharedsignals/
  - Biscuits: https://www.biscuitsec.org/

---

## Risk Analysis & Mitigation

| Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|
| IETF drafts change after writing | High | Low | Mark all drafts with version number and expiry date; add `last_verified` per entry |
| Cloud provider docs change | Medium | Medium | Link to official docs; `last_verified` date per section |
| MCP spec continues evolving rapidly | High | Medium | Version MCP section by spec date (2025-03-26, 2025-06-18, 2026 dev notes) |
| YAML companion drifts from Markdown | Medium | High | YAML is authoritative; Markdown tables generated from / diffed against YAML in CI |
| Decision tables become stale | Medium | High | Tables derived from per-mechanism sections; update mechanism first, table second |
| Eval harness queries become stale | Low | Medium | Evals keyed to mechanism IDs; failing eval = mechanism content changed |
| OIDC-A / SCIM for agents sections outdated | High | Low | Mark as proposal/not-final; link to OpenID CG repo for current status |

---

## Future Considerations

- Add `docs/agentic-identity-reference.json` variant for agents that prefer JSON
- Version the document (`v1.0.0`) per SemVer: major = breaking schema change, minor = new mechanism, patch = correction
- Per-language code examples (Python/TypeScript) for obtaining each token type
- MCP resource server exposing this document as a queryable API: `lookup_mechanism(id)`, `find_scenario(runtime, target, identity_kind)`, `get_wire_format(mechanism_id)`
- OID4VC and W3C DID full sections as adoption matures
- FAPI 2.0 financial agent deployment guide (full recipe)
- GUI/browser agent auth bypass section when Web Bot Auth reaches standardization

---

## Documentation Plan

This plan IS the research artifact. The deliverables are:
- `docs/agentic-identity-reference.md` — main document
- `docs/agentic-identity-reference.yaml` — machine-readable companion (authoritative)
- `docs/agentic-identity-reference.schema.json` — JSON Schema for validation
- `docs/evals.yaml` — eval harness with 25+ query/answer pairs

---

## References & Research

### Primary Sources — Specifications

- OpenID Foundation paper (arxiv): https://arxiv.org/html/2510.25819v1
- MCP Authorization (2025-03-26): https://modelcontextprotocol.io/specification/2025-03-26/basic/authorization
- MCP Authorization (draft): https://modelcontextprotocol.io/specification/draft/basic/authorization
- A2A Specification: https://a2a-protocol.org/latest/specification/
- IETF WIMSE WG: https://datatracker.ietf.org/wg/wimse/about/
- WIMSE Architecture (draft-07): https://datatracker.ietf.org/doc/draft-ietf-wimse-arch/
- AAuth draft: https://www.ietf.org/archive/id/draft-rosenberg-oauth-aauth-00.html
- AIMS draft: https://datatracker.ietf.org/doc/html/draft-klrc-aiagent-auth-00
- On-Behalf-Of User draft: https://datatracker.ietf.org/doc/html/draft-oauth-ai-agents-on-behalf-of-user-02
- Identity Assertion Grant draft: https://datatracker.ietf.org/doc/html/draft-ietf-oauth-identity-assertion-authz-grant-01
- RFC 9449 (DPoP): https://datatracker.ietf.org/doc/html/rfc9449
- RFC 9700 (OAuth security BCP): https://datatracker.ietf.org/doc/rfc9700/
- RFC 8693 (Token Exchange): https://datatracker.ietf.org/doc/html/rfc8693
- RFC 8707 (Resource Indicators): https://datatracker.ietf.org/doc/html/rfc8707
- RFC 9396 (RAR): https://datatracker.ietf.org/doc/html/rfc9396
- RFC 9728 (Protected Resource Metadata): https://datatracker.ietf.org/doc/html/rfc9728
- RFC 9126 (PAR): https://datatracker.ietf.org/doc/html/rfc9126
- RFC 7523 (JWT Bearer): https://datatracker.ietf.org/doc/html/rfc7523
- RFC 7662 (Introspection): https://datatracker.ietf.org/doc/html/rfc7662
- RFC 9635 (GNAP): https://datatracker.ietf.org/doc/html/rfc9635
- SPIFFE Concepts: https://spiffe.io/docs/latest/spiffe-about/spiffe-concepts/
- SPIFFE JWT-SVID Spec: https://spiffe.io/docs/latest/spiffe-specs/jwt-svid/
- SD-JWT Agent Cards draft: https://datatracker.ietf.org/doc/draft-nandakumar-agent-sd-jwt/
- OID4VCI 1.0: https://openid.net/specs/openid-4-verifiable-credential-issuance-1_0.html
- W3C DID v1.1: https://www.w3.org/news/2026/w3c-invites-implementations-of-decentralized-identifiers-dids-v1-1/
- OpenID Federation 1.0: https://openid.net/specs/openid-federation-1_0.html
- CIBA Core 1.0: https://openid.net/specs/openid-client-initiated-backchannel-authentication-core-1_0.html
- SSF Framework: https://openid.net/wg/sharedsignals/
- AuthZEN WG: https://openid.net/wg/authzen/
- IPSIE WG: https://openid.net/wg/ipsie/
- FAPI 2.0: https://openid.net/specs/fapi-2_0-security-profile.html
- Biscuits: https://www.biscuitsec.org/
- HTTP Message Signatures RFC 9421: https://datatracker.ietf.org/doc/rfc9421/
- C2PA Standard: https://c2pa.org/specifications/specifications/2.1/specs/C2PA_Specification.html
- NIST SP 800-162: https://csrc.nist.gov/publications/detail/sp/800-162/final

### Primary Sources — Framework / Platform Docs

- LangGraph Auth: https://docs.langchain.com/langgraph-platform/custom-auth
- Google ADK Auth: https://google.github.io/adk-docs/tools-custom/authentication/
- AWS AgentCore Identity: https://aws.amazon.com/blogs/machine-learning/introducing-amazon-bedrock-agentcore-identity-securing-agentic-ai-at-scale/
- Azure Foundry Agent Identity: https://learn.microsoft.com/azure/foundry/agents/concepts/agent-identity
- Entra Agent ID Token Claims: https://learn.microsoft.com/entra/agent-id/identity-platform/agent-token-claims
- GitHub Actions OIDC Reference: https://docs.github.com/actions/reference/openid-connect-reference
- GitLab CI OIDC: https://docs.gitlab.com/ci/secrets/id_token_authentication/
- CircleCI OIDC: https://circleci.com/docs/openid-connect-tokens/
- CrewAI A2A: https://docs.crewai.com/en/learn/a2a-agent-delegation
- Vertex AI Auth: https://docs.cloud.google.com/vertex-ai/docs/authentication

### Related Research

- Agentic JWT (A-JWT) research: https://arxiv.org/html/2509.13597v1
- AI Agents with DIDs and VCs: https://arxiv.org/abs/2511.02841
- Auth0 A2A Authentication: https://auth0.com/blog/auth0-google-a2a/
- Zero-Trust Agents (Microsoft): https://techcommunity.microsoft.com/blog/azure-ai-foundry-blog/zero-trust-agents-adding-identity-and-access-to-multi-agent-workflows/4427790
- JWT Algorithm Confusion (PortSwigger): https://portswigger.net/web-security/jwt/algorithm-confusion
- RFC 9728 explained (WorkOS): https://workos.com/blog/introducing-rfc-9728-say-hello-to-standardized-oauth-2-0-resource-metadata
- OAuth 2.1 vs 2.0 (FusionAuth): https://fusionauth.io/articles/oauth/differences-between-oauth-2-oauth-2-1
- OpenID Foundation AI whitepaper announcement: https://openid.net/new-whitepaper-tackles-ai-agent-identity-challenges/
