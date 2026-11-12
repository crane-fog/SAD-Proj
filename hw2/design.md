---
toc:
    depth_to: 2
puppeteer:
    format: "A4"
    printBackground: true
    timeout: 3000
    margin:
        top: 10mm
        bottom: 10mm
        left: 14mm
        right: 14mm
---

<style>
.markdown-preview.markdown-preview {
    font-family: "Times New Roman", "宋体";
    font-size: 11pt;
}
.markdown-preview.markdown-preview svg,
.markdown-preview.markdown-preview img {
    max-width: 100% !important;
    height: auto !important;
    display: block;
    margin: 10px auto;
}
</style>

<center><h3 style="margin: 10px;">Design<br/>A Declarative Personal SysOps Agent with Adaptive Long-Term Memory</h3><b>2450313 徐佳咏、2351428 徐之涵、2352626 马小云、2451405 刘宸瑞</b></center>

## 0. Table of contents

[TOC]

---

## 1. Design Summary

Since Part I SRS, we have translated our black-box requirements into an implementation-ready architecture packaged within a single Go binary, an embedded SQLite datastore, and a Vue 3 web console. We decomposed the system into four cohesive subsystems—Web & API Gateway, State Reconciliation Engine, Agent & Memory Orchestrator, and SysOps SSH & Safety—formalized across C4 Logical, Deployment, and Technology views, UML behavioral/structural models, and Architecture Decision Records.

**What Changed and Why**

Translating high-level requirements into concrete interaction and state models exposed three operational gaps that required refining our Part I baseline:

1. **Transactional Remediation with Pre-Mutation Backups and Compensation (Refining `UC-05`)**:
   Part I assumed approved multi-step remediation plans would either wholly succeed or fail cleanly. In practice, a command failing midway leaves a target Linux host in a partially mutated state. Applying the Compensating Transaction Pattern, every mutating action in `propose_remediation_plan` now requires a pre-mutation backup (`backup_cmd` saving snapshots to `/var/tmp/sysops-backup/<incident_id>`) and a paired `rollback_cmd`. Execution runs sequentially under fail-fast semantics: if step $k$ fails, execution halts, the incident transitions to `Partially_Failed`, and the Web UI exposes per-step status alongside a one-click reverse rollback workflow.
2. **Per-Node Concurrency Locking and Incident Coalescing (Refining `UC-03`, `UC-04`, `UC-05`)**:
   Part I lacked mutual exclusion when multiple state drifts occur simultaneously on the same host. We introduced a Per-Node Lease Lock and Coordinator (`IDLE`, `PROBING`, `DIAGNOSING`, `PENDING_APPROVAL`, `REMEDIATING`). On a locked node, identical drift signals coalesce into the active incident, while distinct drifts enter a serialized queue (`QUEUED_ON_LOCK`) rather than spawning duplicate LLM loops. An exclusive `REMEDIATING` lock also suspends routine `UC-03` probes on that host until post-remediation verification finishes.
3. **Deterministic Tool Execution Interceptor (Refining `UC-04`, `UC-09`)**:
   To correct our Part I component diagram—where the Review Agent appeared to invoke SSH/Plan tools directly—we introduced a deterministic Go Tool Execution Interceptor. The Diagnostic Agent emits all tool calls; the Interceptor intercepts each call, queries the Review Agent strictly as a sidecar classifier (`READ_ONLY`, `MUTATING`, `DESTRUCTIVE`), and routes approved read-only commands to SSH or suspends mutating plans at the `UC-05` approval gate.

All architectural views, ADRs, relational schemas, UML models, and updated UI mockups are complete, establishing a concrete baseline for Part III PoC validation.

---

## 2. Architecture and Key Decisions

### 2.1 Logical architecture

The system adopts a modular monolith and Ports and Adapters architecture organized into four subsystems:

1. **Web & API Gateway**: Serves the embedded Vue 3 SPA, exposes REST endpoints for node/manifest management, HITL approval (`UC-05`), and memory curation (`UC-08`), and streams real-time diagnostic and step-execution states via Server-Sent Events (SSE).
2. **State Reconciliation Engine**: Parses declarative YAML manifests (`UC-02`), schedules deterministic SSH probes (`UC-03`), and enforces per-node mutual exclusion via the Node Lease Lock & Incident Coordinator.
3. **Agent & Memory Orchestrator**: Executes the bounded ReAct diagnostic loop (`UC-04`), interactive chat (`UC-06`), FTS5 memory retrieval, and post-incident memory distillation (`UC-07`), intercepting all agent tool calls via the Tool Execution Interceptor.
4. **SysOps SSH & Safety Subsystem**: Enforces `UC-09` guardrails via the Command Review Adjudicator (safety classification) and Secret Redaction Middleware (credential scrubbing), and executes probes, backups, repairs, and rollbacks through the Transactional SSH Executor.

```mermaid
%%{init: {'flowchart': {'nodeSpacing': 20, 'rankSpacing': 38, 'curve': 'basis'}}}%%
flowchart LR
    Operator(["Personal SysOps Operator<br/>(Web Browser)"])

    subgraph ControlPlane ["Single-Binary Control Plane (Go Daemon)"]
        WebGW["<b>1. Web & API Gateway</b><br/>Vue 3 SPA + Gin REST / SSE"]
        Recon["<b>2. State Reconciliation Engine</b><br/>YAML Scheduler + Per-Node Lease Lock"]
        AgentOrch["<b>3. Agent & Memory Orchestrator</b><br/>ReAct Loop + FTS5 Memory Service"]
        Interceptor["<b>Tool Execution Interceptor</b><br/>Deterministic Permission Gate"]
        SafetySSH["<b>4. SysOps SSH & Safety</b><br/>Review Adjudicator + SSH Pool + Redactor"]
        SQLite[("<b>Embedded SQLite</b><br/>WAL + FTS5 Store")]
    end

    LinuxNode[("Target Linux Nodes<br/>(OpenSSH Server)")]
    LLMProvider["OpenAI-Compatible LLM API<br/>(Planner & Reviewer Models)"]

    Operator -- "HTTPS / SSE" --> WebGW
    WebGW -- "Manage / Approve" --> Recon
    WebGW -- "Chat / Rollback" --> AgentOrch & SafetySSH

    Recon -- "Drift Event (DIAGNOSING)" --> AgentOrch
    Recon -- "Periodic Probes" --> SafetySSH

    AgentOrch -- "1. Tool Call" --> Interceptor
    Interceptor -- "2. Safety Verdict & SSH" --> SafetySSH
    Interceptor -- "3. Save Plan" --> Recon

    SafetySSH -- "SSH v2" --> LinuxNode
    SafetySSH -- "Scrubbed Logs" --> AgentOrch
    AgentOrch & SafetySSH -- "HTTPS JSON" --> LLMProvider
    Recon & AgentOrch & SafetySSH --> SQLite

    style ControlPlane fill:#F8FAFC,stroke:#475569,stroke-width:1.5px
    style WebGW fill:#EFF6FF,stroke:#3B82F6
    style Recon fill:#F0FDF4,stroke:#22C55E
    style AgentOrch fill:#FAF5FF,stroke:#A855F7
    style Interceptor fill:#FEF9C3,stroke:#CA8A04
    style SafetySSH fill:#FFFBEB,stroke:#F59E0B
    style SQLite fill:#F1F5F9,stroke:#475569
```

### 2.2 Deployment architecture

To meet the Single-Binary Zero-Config (`SRS 7.1`) and Agentless Control Plane (`SRS 6.7`) requirements, the runtime topology spans three nodes without external databases or remote monitoring agents:

1. **Control Plane Host**: Runs the `sysops-agent` Go executable (embedding compiled Vue 3 static assets via `go:embed`), paired with a local SQLite database (`sysops.db` in WAL mode) and an AES-256-GCM host key (`master.key`) for credential encryption. It also exposes port to provide the web console to the operator.
2. **Target Linux Nodes (1–20 Hosts)**: Standard Linux servers accessible via `sshd` (Port 22). During `UC-05` execution, pre-mutation file/state snapshots are staged in `/var/tmp/sysops-backup/<incident_id>/` on the target host.
3. **LLM Inference Endpoint**: Any cloud or self-hosted (Ollama/vLLM) OpenAI-compatible API endpoint accessed over HTTPS.

```mermaid
%%{init: {'flowchart': {'nodeSpacing': 25, 'rankSpacing': 40, 'curve': 'basis'}}}%%
flowchart LR
    Browser(["<b>Operator Browser</b><br/>Vue 3 SPA"])

    subgraph ControlHost ["Control Plane Host (Go Daemon)"]
        Binary["<b>sysops-agent Binary</b><br/>Gin + Reconciler + Agent + SSH Pool"]
        SQLiteFile[("<b>sysops.db + master.key</b><br/>SQLite WAL & AES-GCM Key")]
    end

    subgraph TargetFleet ["Managed Linux Nodes (1..20)"]
        SSHD["<b>OpenSSH (sshd :22)</b><br/>systemd · Docker · Filesystem"]
        BackupDir[("<b>/var/tmp/sysops-backup/</b><br/>Pre-Mutation Snapshots")]
    end

    LLMNode["<b>OpenAI-Compatible LLM API</b><br/>Planner & Reviewer Endpoints"]

    Browser -- "HTTPS / SSE (:8080)" --> Binary
    Binary --> SQLiteFile
    Binary -- "Multiplexed SSH v2" --> SSHD
    SSHD --> BackupDir
    Binary -- "HTTPS JSON (Redacted)" --> LLMNode

    style ControlHost fill:#EFF6FF,stroke:#2563EB,stroke-width:1.5px
    style TargetFleet fill:#F0FDF4,stroke:#16A34A,stroke-width:1.5px
    style LLMNode fill:#FAF5FF,stroke:#9333EA,stroke-width:1.5px
    style Browser fill:#F8FAFC,stroke:#64748B,stroke-width:1.5px
```

### 2.3 Technology architecture

| Layer                   | Selected Technology                   | Alternatives Considered       | Rationale & Trade-offs                                                                                                                                                             |
| :---------------------- | :------------------------------------ | :---------------------------- | :--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Backend Runtime**     | **Go**                                | Python (FastAPI), Rust        | Compiles backend, concurrency scheduler, SSH pool, and frontend into one zero-dependency binary, avoiding Python runtime packaging overhead.                                       |
| **Web & Streaming**     | **Gin + SSE (Server-Sent Events)**    | WebSockets, gRPC-Web          | Diagnostic traces and step statuses stream unidirectionally while approvals use REST `POST`; SSE provides lighter connection management and auto-reconnection than WebSockets.     |
| **Frontend UI**         | **Vue 3 + TypeScript + Tailwind CSS** | React / Next.js, HTMX         | Compiles to compact static assets while supporting reactive state needed for the YAML Policy Builder and step-by-step Rollback Console.                                            |
| **Datastore & Memory**  | **Embedded SQLite (WAL + FTS5**)      | PostgreSQL + pgvector, Milvus | External vector DBs break single-binary deployment. SysOps faults rely on exact lexical tokens (unit names, exit codes, error flags), where FTS5 BM25 ranking excels at zero cost. |
| **Node Transport**      | **Native Go SSH**                     | Ansible Runner, Custom Agent  | Custom agents violate zero-footprint rules; Ansible subprocesses add startup overhead.                                                                                             |
| **LLM Integration**     | **OpenAI-Compatible JSON Tool API**   | LangChain, Vendor SDKs        | Thin custom client avoids heavy framework abstraction and enables zero-code switching across cloud or local LLM providers (`SRS 7.3`).                                             |
| **Shell Safety Parser** | **Go `mvdan.cc/sh/v3` AST Parser**    | Regex Matching                | Parses shell pipelines, subshells, and redirections into an AST before Review Agent evaluation, preventing obfuscated command bypasses where simple regex filters fail.            |

### 2.4 Key decisions

【每个人都撰写，最后徐佳咏整理总结】

【徐之涵】【马小云】【刘宸瑞】

#### ADR-01: Single-Binary Agentless Control Plane with Per-Node Lease Locking

- **Context**: The system must deploy as a lightweight personal tool (`SRS 7.1`) while preventing concurrent drift probes (`UC-03`), agentic diagnostics (`UC-04`), and state-mutating repairs (`UC-05`) from colliding on the same target Linux host.
- **Options**: (1) Distributed microservices + Redis locks + remote agents; (2) Uncoordinated SSH goroutines; (3) Single-binary Go daemon with SQLite-backed Per-Node Lease Locks.
- **Decision**: Adopt Option 3, enforcing a per-node lock state machine (`IDLE`, `PROBING`, `DIAGNOSING`, `PENDING_APPROVAL`, `REMEDIATING`) with heartbeat-renewed TTLs (`lease_expires_at`).
- **Consequences**: Identical drift signals on a locked node coalesce into the active incident (`coalesced_count++`), distinct drifts queue serially (`QUEUED_ON_LOCK`), and `REMEDIATING` locks suppress routine `UC-03` probes until post-repair verification completes. Heartbeat TTLs prevent deadlocked nodes if an SSH or LLM session hangs.

#### ADR-02: Embedded SQLite FTS5 Memory Retrieval with Three-Branch Adaptive Evolution

- **Context:** The control plane must remain lightweight, agentless, and deployable as a single Go binary with a local SQLite database (`SRS 7.1`). It must also retrieve past verified incident solutions during `UC-04` without repeatedly asking the LLM to solve the same operational failure from scratch. Personal SysOps incidents often contain highly specific lexical identifiers (such as `nginx.service`, `OOMKilled`, an exit code, or a configuration error), while remembered fixes must retain provenance, distinguish stale versions, and never become autonomous execution instructions.

- **Options:** (1) Use local SQLite FTS5 with BM25 text ranking and explicit version/status columns; (2) deploy PostgreSQL with pgvector for combined relational and embedding-based search; or (3) maintain a separate vector database such as Milvus with embedding generation and synchronization. The alternatives can improve semantic recall when users describe equivalent problems using different wording, but add operational dependencies, model/embedding costs, data synchronization, and backup complexity that conflict with the small-scale deployment target.

- **Decision:** Select **SQLite WAL + FTS5** as the primary memory store. Index sanitized `symptom_keywords`, `root_cause`, `remediation_steps`, and `scope_tags` using an external-content `memories_fts` table synchronized with `memories`. Use parameterized queries, operational-token normalization, BM25 ranking, explicit host/service scope checks, and `status = ACTIVE`, returning a bounded top-k set (default `search_memory.count = 3`) within a fixed context-token budget. Retrieval returns historical evidence, not instructions. New one-off candidates require operator acceptance before becoming `ACTIVE`. Apply the deterministic `UC-07` **three-branch evolution rule** only after verified resolution: **unmatched** incidents create a staged candidate that becomes active only after the operator accepts it; **matched, unchanged** successful fixes increment `hit_count` and update `last_verified_at`; **matched, modified** fixes insert a successor version and mark the prior version `SUPERSEDED`. Persist selected memory IDs with incidents; preserve `source_incident_id`, verification timestamps, version lineage, and incident-scoped idempotency receipts; use a durable UC-07 job claimed by a worker after resolution, and allow explicit discard or archiving without destroying audit history.

- **Consequences and Trade-offs:** The decision avoids a separate search service, supports reproducible and inexpensive retrieval of exact system-error tokens, and integrates memory updates with other SQLite operations in atomic transactions. It also gives operators clear provenance and predictable version behavior. In exchange, FTS5 cannot guarantee semantic matching across paraphrases or changing log vocabulary, may require normalization for punctuation-rich SysOps identifiers, and does not independently determine whether an older fix is still safe in a new host context. Search hits are **advisory evidence only**: command execution remains subject to `CHG-03` interception and `UC-05` approval. Part III should validate retrieval precision/recall and stale-memory behavior on a small fault-injection corpus before considering an optional semantic reranker.

### 2.5 Quality attributes

【刘宸瑞】

### 2.6 Architectural Risks

【刘宸瑞】

### 2.7 Agentic Systems

#### 2.7.1 Agent Roles and Trust Boundaries

The Agent & Memory Orchestrator performs bounded, evidence-driven diagnosis after the State Reconciliation Engine detects drift (`UC-03`). It consists of a **Diagnostic Agent**, a **Memory Service**, a post-resolution **Memory Distillation** workflow, and a deterministic Go **Tool Execution Interceptor**. The Diagnostic Agent uses an OpenAI-compatible LLM to form hypotheses, request observations, and propose a remediation plan; it has no direct SSH, database-write, or approval authority. The Memory Service performs local SQLite FTS5 retrieval and persists versioned records. The distillation workflow transforms *verified* resolved incidents into reusable, sanitized troubleshooting knowledge (`UC-07`).

The independent **Command Review Agent** belongs to the SysOps SSH & Safety subsystem, not to the tool-execution control path. It returns a classification (`READ_ONLY`, `MUTATING`, or `DESTRUCTIVE`) for candidate shell commands, but never invokes the SSH executor or the remediation tools. Instead, the deterministic Go interceptor mediates every agent-originated tool call, applies lifecycle and permission checks, obtains safety verdicts when shell commands are involved, and routes requests to the appropriate service. This separation addresses `CHG-03` and prevents an LLM from granting itself execution privileges.

#### 2.7.2 Internal Orchestration and Tool-Call Architecture

The following diagram distinguishes agent reasoning, deterministic authorization, human approval, SSH execution, and memory operations. The inter-agent review call returns only a safety verdict; all actual dispatch remains with the interceptor and the execution subsystem.

```mermaid
flowchart TB
    Drift["UC-03: State Drift Event"] --> Orch["Agent Orchestrator"]
    Verified["UC-05: Verified Resolved Event"] --> Distill["Post-Incident Memory Distillation"]

    subgraph AgentMem["Agent & Memory Orchestrator"]
        Orch --> Diagnostic["Diagnostic Agent / Bounded ReAct Loop"]
        Diagnostic <-->|"Prompts / tool-call responses"| LLM["OpenAI-Compatible LLM Adapter"]
        Diagnostic -->|"Agent tool calls"| Interceptor["Deterministic Go Tool Execution Interceptor"]
        Distill -->|"commit_memory_record()"| Interceptor
        Interceptor -->|"Scoped search_memory() or verified memory commit"| Memory["Memory Service"]
    end

    Memory <--> DB[("SQLite + FTS5 Memory Store")]

    subgraph Safety["SysOps SSH & Safety"]
        Reviewer["Independent Command Review Agent<br/>Classifier Only"]
        SSH["Transactional SSH Executor"]
        Redactor["Local Secret Redaction Middleware"]
    end

    Interceptor -->|"Only credential-free, locally validated command"| Reviewer
    Reviewer -->|"READ_ONLY / MUTATING / DESTRUCTIVE"| Interceptor
    Interceptor -->|"Allowlisted, scope-checked READ_ONLY + current lease"| SSH
    Interceptor -->|"Invalid syntax, unsafe scope or inline secret"| Reject["Reject and return guidance"]
    Interceptor -->|"Validated non-destructive plan"| Gate["UC-05: Pending Approval"]
    Interceptor -->|"DESTRUCTIVE"| Block["Hard block and audit"]
    UI["Operator via Vue / Gin"] -->|"Approve, edit, or reject"| Gate
    Gate -->|"Approved and revalidated actions only"| SSH
    SSH <-->|"SSH"| Node["Target Linux Node"]
    SSH -->|"Raw command output"| Redactor
    Redactor -->|"Sanitized, bounded, scope-checked observations"| Diagnostic
```

**Tool contracts and execution boundaries**

| Tool | Main inputs | Behavior and authority |
| :--- | :--- | :--- |
| `search_memory` | `keywords: string[]`, `scope_context: {node_id, os_family, os_version?, service_name, service_version?}`, `count: int = 3` | Uses scoped, parameterized FTS5 search on `ACTIVE` memory versions; an incompatible OS/service scope is excluded even when keywords match. Results are historical evidence, never executable instructions; cap record text and the *aggregate* prompt budget. |
| `exec_readonly_shell` | `incident_id`, `node_id`, `command`, `scope_context` | The interceptor first parses the complete command AST, checks a narrow allowlist of diagnostic templates **and argument/path restrictions**, rejects inline secrets locally, then obtains a review verdict only for a safe-to-disclose command. A `READ_ONLY` verdict alone is insufficient: re-check current node lease generation and data-access scope immediately before SSH dispatch, cap output, and apply outbound content policy after local redaction. |
| `propose_remediation_plan` | `incident_id`, `node_id`, `root_cause`, `actions[]` with `command`, `backup_cmd`, `rollback_cmd` | Persists an immutable proposal for `UC-05`, never executes it. All three commands per step are locally checked for secrets and reviewed before approval/execution; `DESTRUCTIVE` and unknown commands are rejected. The SSH subsystem rechecks approval, plan revision and lease before dispatch. |
| `commit_memory_record` | `incident_id: string`, `symptom_keywords`, `root_cause`, `remediation_steps`, `scope_tags`, `matched_memory_id?: string`, `is_modified: bool` **required** | Internal UC-07 service tool (not an arbitrary LLM SQL write). Verifies `incident_id` is resolved and owns an idempotency receipt; derives/validates an `ACTIVE` matched memory and explicit `is_modified` from the *verified* fix before an atomic three-branch update. For unmatched one-off incidents the candidate may be discarded instead of promoted. The required flag refines the optional field in the Part I tool sketch. |

#### 2.7.3 Short-Term Context and Long-Term Memory Retrieval

For each incident, the orchestrator constructs a **short-term diagnostic context** from the active YAML manifest, observed drift, node and service identifiers, bounded and sanitized diagnostic output, relevant operator feedback, and at most three retrieved memories (`search_memory` default `count = 3`). SQLite FTS5 with BM25 ranking prioritizes exact operational tokens such as service names, exit codes, and log error signatures. Only `Active` memories matching the incident's applicable scope should enter retrieval; `Superseded` and `Archived` versions are excluded from the default search. Retrieved fixes are advisory evidence and must not bypass command review or human approval.

The orchestrator retains concise observations and tool outcomes rather than unbounded raw logs. SSH probes use bounded commands (for example, `journalctl -n 50` or `docker logs --tail 50`), and context can be summarized between turns while preserving key error signatures and causal evidence. The local Secret Redaction Middleware removes credentials, tokens, and private keys before diagnostic output is persisted in SQLite, sent to an external LLM endpoint, or streamed to the UI. The web console receives sanitized progress summaries, not private raw model reasoning. An OpenAI-compatible adapter permits local or cloud inference without changing the orchestration logic.

**Scope, input trust, and size limits.** The Memory Service receives `scope_context` from the *trusted* node/manifest record, not from a model-generated tag; it first filters for compatible OS/service identity and active status, then ranks by BM25. `count = 3` is not a size budget: bound each stored field and each returned record, cap the combined injected memory text (proposed maximum: 4,000 tokens), and reserve a separate token budget for the manifest and live observations. Treat retrieved repairs and SSH logs as untrusted data susceptible to prompt injection; instruction-like text in these sources never changes the command allowlist or approval policy. Large or unsupported data is truncated, summarized locally, or withheld instead of being sent wholesale to the model.

#### 2.7.4 Bounded ReAct Diagnosis (`UC-04`)

When `UC-03` detects drift and the per-node coordinator grants a `DIAGNOSING` lease (`CHG-02`), the orchestrator first searches historical memories and builds the initial context. The Diagnostic Agent then follows a bounded **Reason–Act–Observe** loop: (1) infer a likely cause from available evidence; (2) issue a diagnostic tool call; (3) let the interceptor and safety subsystem vet and execute an eligible read-only probe; and (4) incorporate the redacted observation into the next inference step. The configured default ceiling is `max_turns = 8` (`SRS UC-04`). The orchestrator also applies cancellation to every LLM HTTP request (proposed 30-second per-call deadline) and a 120-second end-to-end diagnosis budget from `SRS 7.7`. On timeout, it cancels the in-flight request, stops renewing the diagnosis lease once it is safe to do so, records a sanitized timeout outcome, and escalates; a hung provider cannot indefinitely hold the node or cause an unbounded retry. If a command is not read-only, the interceptor refuses it and instructs the agent to collect evidence through safe probes or defer the change to the remediation plan.

If sufficient evidence is obtained, the agent submits a structured root-cause summary and ordered remediation actions through `propose_remediation_plan`. The incident transitions to `PENDING_APPROVAL` and waits for explicit operator review (`UC-05`); the Diagnostic Agent never runs the proposed mutations itself. If the turn ceiling is reached or useful diagnosis cannot proceed, the incident is marked `Escalated` with the available sanitized evidence, rather than guessing or automatically repairing the host. If the LLM endpoint is unavailable, deterministic drift detection and alerting continue independently, while agent diagnosis is deferred or escalated (`SRS 7.5`).

#### 2.7.5 Safety, Human Handoff, and Post-Resolution Memory

The local interceptor provides the enforceable boundary between untrusted model output and privileged tools. A `READ_ONLY` classification constrains mutation risk but **does not authorize reading arbitrary files**: the trusted Go control plane permits only approved, parameter-bounded diagnostics (for example `df`, `free`, `systemctl status <approved-unit>`, `journalctl -u <approved-unit> -n 50 --no-pager` and `docker logs --tail 50 <approved-container>`). General file-reading, arbitrary path arguments, redirections, nested shells, `eval`, and unsupported pipelines fail closed. Both the requested data source and the sanitized output must remain within the node/service scope approved for that incident; redaction is defense in depth, not a substitute for access control. A deterministic local preflight rejects commands containing inline tokens/passwords/private keys before any shell, backup, or rollback command reaches a potentially remote Review Agent. Commands and logs with unrecognized sensitivity must not be sent to external LLM endpoints. Command parsing and review must fail closed on unsupported or obfuscated shell syntax; the Review Agent's result is input to a deterministic policy decision, not permission by itself. The interceptor checks ownership and the monotonically increasing `lease_generation` *again at the SSH dispatch boundary*; an expired or replaced generation cannot start a command. Node takeover stays blocked until any prior remote command has been positively stopped or recovered, rather than relying on TTL expiry alone. `DESTRUCTIVE` commands are hard-blocked and audited. `MUTATING` operations are prohibited during autonomous diagnosis and require a reviewed, versioned plan plus explicit human approval. After approval, the SysOps SSH & Safety subsystem performs the `backup_cmd`, then the change, and finally post-change verification; failures enter the compensating recovery path described in `CHG-01` and the SysOps SSH & Safety team design. The approval and recovery details belong to `UC-05` and the SSH subsystem, rather than the Diagnostic Agent.

Only after the desired-state probe confirms recovery and the incident reaches `Resolved` does `UC-07` begin memory distillation. The system extracts the observed symptoms, confirmed root cause, verified remediation, and any operator modifications, scrubs sensitive content, and invokes `commit_memory_record`. Its three-branch policy stages an unmatched one-off candidate for approval before an active memory is created, increments `hit_count` and refreshes `last_verified_at` for an unchanged successful reuse, or inserts a new version and marks the old entry `Superseded` when the fix was modified. Operators first review unmatched one-off candidates in a non-indexed `PENDING_REVIEW` distillation state: accepting promotes the candidate to `ACTIVE`, while discarding records a `DISCARDED` receipt without making the memory searchable. After promotion, operators may also edit or archive memories (`UC-08`). Selected `matched_memory_id` and environmental scope are persisted with the incident before diagnosis completes. The server compares the canonical verified remediation against the current active matched record (including operator modifications); a mere FTS similarity hit is not automatically considered the same pattern. All new or successor verified records set `last_verified_at` to the incident's actual verification time; manual curation never fabricates verification. This process separates temporary diagnostic context from reusable, provenance-linked long-term knowledge, while ensuring that unverified LLM suggestions are never treated as successful fixes.

**Design consistency:** This section implements the `UC-04`, `UC-07`, and `UC-09` behaviors in Part I SRS; the `CHG-01` transactional plan, `CHG-02` node lease coordination, and `CHG-03` deterministic interceptor refinements in Part II; and the REST/SSE and versioned memory interfaces proposed in the team's HW2 API design.

---

## 3. Design Models

### 3.1 Domain & Structural Class Diagrams

【刘宸瑞】

### 3.2 Relational Data Model

#### 3.2.1 Data Model and Persistence Boundaries

The control plane uses **embedded SQLite** with Write-Ahead Logging (WAL) for operational state and **FTS5** for keyword-based incident-memory retrieval. This choice preserves the project's single-binary, agentless deployment requirement (`ADR-01`) and avoids a separate database or vector-search service (`ADR-02`). The schema must support three connected lifecycles: (1) a managed node's declarative policy and detected drift (`UC-02/03`); (2) bounded, reviewed diagnosis and human-approved, recoverable remediation (`UC-04/05/09`); and (3) verified incident-to-memory distillation, retrieval, versioning, and curation (`UC-07/08`).

Database columns store **durable domain facts**, whereas the API presents versioned resource representations. For example, the API's `lock_state` is derived from the `node_leases` row; `manifest_version` resolves to an immutable `manifest_revisions` record; and the incident's current plan revision is represented by the latest `remediation_plans` row. Operational event delivery is supported by a durable `events` outbox, not by treating an SSE connection as persistent storage.

#### 3.2.2 Entity–Relationship Diagram

The following logical ER diagram shows primary and foreign keys (`PK`, `FK`). SQLite uses `TEXT` for opaque IDs and UTC timestamps, `INTEGER` for versions/counters, and JSON-valid `TEXT` for compact structured fields. Abbreviated attributes are expanded in the schema summary below; the separate executable SQL sketch is intentionally not part of this document.

```mermaid
erDiagram
    NODES ||--|| NODE_LEASES : coordinated_by
    NODES ||--o{ MANIFESTS : owns
    MANIFESTS ||--o{ MANIFEST_REVISIONS : has
    NODES ||--o{ INCIDENTS : reports
    MANIFEST_REVISIONS ||--o{ INCIDENTS : observed_under
    INCIDENTS ||--o{ DIAGNOSTIC_OBSERVATIONS : records
    INCIDENTS ||--o{ REMEDIATION_PLANS : revises
    REMEDIATION_PLANS ||--o{ REMEDIATION_ACTIONS : contains
    INCIDENTS ||--o{ OPERATIONS : schedules
    REMEDIATION_ACTIONS ||--o{ ACTION_RUNS : executed_as
    OPERATIONS ||--o{ ACTION_RUNS : contains
    INCIDENTS ||--o{ MEMORIES : produces
    MEMORIES o|--o{ MEMORIES : supersedes
    MEMORIES o|--o{ INCIDENTS : recalled_by

    NODES {
        string id PK
        string name
        string host
        string connection_status
        string health_status
        int version
    }
    NODE_LEASES {
        string node_id PK,FK
        string lock_state
        int lease_generation
        string owner_token
        string lease_expires_at
        int recovery_required
    }
    MANIFESTS {
        string id PK
        string node_id FK
        string name
        string active_revision_id FK
        int version
    }
    MANIFEST_REVISIONS {
        string id PK
        string manifest_id FK
        int revision_number
        string yaml
    }
    INCIDENTS {
        string id PK
        string node_id FK
        string manifest_revision_id FK
        string matched_memory_id FK
        string status
        int coalesced_count
        int plan_revision
        int version
    }
    DIAGNOSTIC_OBSERVATIONS {
        string id PK
        string incident_id FK
        int turn_index
        string reviewed_command
        string sanitized_output
    }
    REMEDIATION_PLANS {
        string id PK
        string incident_id FK
        int revision_number
        string review_status
    }
    REMEDIATION_ACTIONS {
        string id PK
        string plan_id FK
        int step_index
        string command
        string backup_cmd
        string rollback_cmd
    }
    OPERATIONS {
        string id PK
        string incident_id FK
        string node_id FK
        string status
        int lease_generation
    }
    ACTION_RUNS {
        string id PK
        string operation_id FK
        string action_id FK
        string backup_status
        string command_status
        string rollback_status
    }
    MEMORIES {
        string id PK
        string source_incident_id FK
        string supersedes_id FK
        string lineage_id
        int revision_number
        string status
        int hit_count
        string last_verified_at
        int version
    }
```

`MEMORIES.supersedes_id` points to an earlier version within the same lineage; a newly revised entry is the active successor of the old one. `INCIDENTS.matched_memory_id` records the principal verified, scoped historical memory selected **during diagnosis, before resolution**, so restart/retry does not lose the relationship. Persist it in the same transaction as the relevant diagnostic observation (or an incident–memory selection record), including the matched memory version and scope used for selection. An optional incident–memory match table can record the remaining retrieved candidates for provenance. A high BM25 score by itself never establishes an actual match. `EVENTS`, `IDEMPOTENCY`, and `MEMORY_DISTILLATIONS` are infrastructure/audit tables explained below and intentionally omitted from the main domain diagram for readability. The distillation table also acts as a **durable work queue**: the incident-resolution transaction inserts a `PENDING` work record atomically with `RESOLVED`, making a post-crash retry possible. A `PENDING_REVIEW` work state stores unmatched one-off candidates outside the searchable `memories` table.

#### 3.2.3 Core Tables, Keys, and Indexes

| Table | Important columns and constraints | Purpose / access pattern |
| :--- | :--- | :--- |
| `nodes` | `id` PK; `name`, `host`, `port`, `username`, `host_key_fingerprint`, `connection_status`, `health_status`, `last_probe_at`, `version`; encrypted credential blob never returned by API | Registered Linux nodes (`UC-01/03`). `connection_status` is not conflated with health or lock state. |
| `node_leases` | `node_id` PK/FK; `lock_state`, `owner_token`, monotonic `lease_generation`, `lease_expires_at`, `recovery_required` | SQLite-backed per-node coordinator (`CHG-02`); exclusive ownership and safety on uncertain remote execution. |
| `manifests` + `manifest_revisions` | `manifests.node_id` FK; active revision pointer; immutable `(manifest_id, revision_number)` unique key; YAML and validation state in revisions | Versioned declarative policies. A running incident stays bound to the revision observed when drift occurred, even when a policy is edited later. |
| `incidents` | `id` PK; `node_id`, `manifest_revision_id` FKs; `status`, `drift_signature`, `drift_summary`, `coalesced_count`, `root_cause_summary`, `matched_memory_id`, `matched_memory_version`, `plan_revision`, `version`, `resolved_at` | Traces a fault from `OPEN`/`DIAGNOSING` to `PENDING_APPROVAL`, `RESOLVED`, or `ESCALATED`, including `QUEUED_ON_LOCK` and `PARTIALLY_FAILED`. |
| `diagnostic_observations` | `id` PK; `incident_id` FK; `turn_index`, `tool_name`, `review_verdict`, sanitized command/observation, `created_at` | Bounded, redacted evidence for the ReAct trace; no raw secrets or private model reasoning are stored (`UC-04/09`). |
| `remediation_plans` + `remediation_actions` | `(incident_id, revision_number)` unique; ordered `(plan_id, step_index)` unique; each action has `command`, mandatory `backup_cmd` and `rollback_cmd`, risk and review metadata | Immutable reviewed plan revisions. A plan edit creates a new revision; previously approved steps cannot silently change (`CHG-01` and the team's SSH & Safety design). |
| `operations` + `action_runs` | `operations` records operation/worker/lease generation; `action_runs` records backup, command and compensation states for a specific approved action | Tracks execution and recovery outcomes without mutating the reviewed command definitions. Timeout/uncertain status is explicit. |
| `memories` | `id` PK; `lineage_id`, `revision_number`, `supersedes_id` FK; `source_incident_id` FK; `symptom_keywords`, `root_cause`, `remediation_steps`, `scope_tags`, `status`, `hit_count`, `last_verified_at`, `version` | Reusable, verified, redacted knowledge with provenance, optimistic concurrency, and audit-preserving version lineage (`UC-07/08`). |
| `events` | Autoincrement `id`; `resource_type`, `resource_id`, `resource_version`, `type`, sanitized JSON `payload`, `committed_at` | Durable SSE outbox. Keep up to **24 hours or 100,000 entries**, whichever threshold is reached first; reclaim old rows in short background batches. Expired replay cursors cause `stream.reset` and a fresh consistent `/sync` snapshot (per the proposed Web/API contract). |
| `idempotency` + `memory_distillations` | `(operator_id, key)` unique for operator commands; `memory_distillations.incident_id` unique, plus `state` (`PENDING`, `RUNNING`, `PENDING_REVIEW`, `COMPLETED`, `DISCARDED`, `FAILED`), `candidate_json`, `memory_id`, `claimed_at`, `retry_count` | Durable UC-07 work/receipt; resolution plus `PENDING` work is committed together, while one-off candidates wait for operator decision outside the active memory index. |

Indexes target the most common operations: `incidents(node_id, status, updated_at)`, `remediation_plans(incident_id, revision_number)`, `remediation_actions(plan_id, step_index)`, `memories(status, last_verified_at)`, and an FTS5 index on searchable memory text. A partial unique index on `memories(lineage_id) WHERE status = 'ACTIVE'` prevents two simultaneously active versions of one troubleshooting pattern; `UNIQUE(supersedes_id)` prevents a previous version from acquiring two immediate successors. Foreign keys are enabled on every SQLite connection; WAL supports concurrent readers while short transactions serialize writes.

#### 3.2.4 SQLite FTS5 Search and Context Retrieval

The `memories_fts` virtual table indexes four searchable fields in `memories`: `symptom_keywords`, `root_cause`, `remediation_steps`, and `scope_tags`. An **external-content** FTS5 index keyed by the `memories` rowid and synchronized through insert/update/delete triggers avoids a second independent source of truth. The Memory Service normalizes diagnostic terms, constructs a safely escaped FTS query, and uses bound parameters to prevent user-provided text from being interpreted as arbitrary SQL or untrusted FTS query syntax. `scope_context` is obtained from trusted current node/manifest metadata (`node_id`, OS distribution/version, service identity/version); incompatible records are excluded before any result is injected into the LLM. Each memory field has an enforced storage length limit; retrieved content is truncated/summarized to a total memory budget (proposed 4,000 tokens), not merely bounded by record count. It joins ranked FTS matches back to `memories`, filters `status = 'ACTIVE'` and applicable scope, and uses `bm25(...)` ranking with a configurable top-`k` limit (default `search_memory.count = 3` in Part I SRS).

For example, a future incident involving `nginx.service` and an address-in-use error can recover a prior verified memory containing those tokens; it does **not** automatically execute its stored remediation steps. Keyword retrieval is lightweight and particularly useful for exact SysOps identifiers, but it can miss semantically similar incidents with different wording. This limitation is a known trade-off of `ADR-02`, not a claim of semantic-equivalent search. `Superseded` and `Archived` records remain available for audit but are excluded from normal retrieval.

#### 3.2.5 Adaptive Memory Evolution (`UC-07`)

A memory write is authorized only after an incident has been **verified `RESOLVED`** (post-remediation desired-state check or verified manual resolution). The distillation pipeline receives the redacted symptom, confirmed cause, successful steps, scope tags, and a reference to any matched memory. The three outcomes below are serialized in short SQLite transactions. A newly unmatched candidate is staged first, and promotion or discard occurs in a separate transaction after the operator chooses; each committed resource change writes its matching durable event atomically:

| Verified outcome | Transactional update | Observable result |
| :--- | :--- | :--- |
| **New pattern** (`matched_memory_id IS NULL`) | Distiller stores a sanitized candidate in `memory_distillations` with `state = PENDING_REVIEW`. Operator may `DISCARD` it as a one-off, leaving a non-indexed receipt; on acceptance insert `ACTIVE` memory with new lineage, `revision_number = 1`, `source_incident_id`, `hit_count = 1`, and **`last_verified_at = incidents.resolved_at`**. | Unaccepted/discarded candidates never appear in FTS5; accepted candidate is searchable with incident provenance. |
| **Existing pattern, unchanged fix** (`is_modified = false`) | Verify the selected `matched_memory_id` is the current `ACTIVE` version with a compatible environment and verified matching remediation; atomically claim the incident receipt, then set `hit_count += 1`, `last_verified_at = incidents.resolved_at`, `version += 1`. | Existing verified knowledge gains another use without creating a new version. |
| **Existing pattern, modified fix** (`is_modified = true`) | Validate the current active matched lineage and operator-edited/verified differences; mark old record `SUPERSEDED` and insert a new `ACTIVE` row with the same lineage, `supersedes_id = old.id`, `revision_number + 1`, new incident provenance and **`last_verified_at = incidents.resolved_at`**. | Version history remains auditable; only new version is searchable by default. |

If the operator later edits a memory through `PATCH /memories/{id}`, the service follows the versioning path but **does not change `last_verified_at` merely because of the edit**. `POST /memories/{id}/archives` changes the current record to `ARCHIVED` rather than deleting evidence or incident links. All three branches use one unique `memory_distillations(incident_id)` work/receipt row, but pending review and the later operator decision are separate, crash-recoverable transactions. A repeated or retried distillation event must not create a duplicate memory or increment `hit_count` twice; all branches maintain the one-active-version-per-lineage invariant. **On resolution**, the control plane commits `RESOLVED`, `resolved_at` and a `PENDING` distillation job in a *single* SQLite transaction. The worker claims that job atomically; startup recovery reclaims interrupted jobs. It derives a tentative match from a persisted incident selection, validates its current scope/status/lineage, and compares a canonical normalized set of *verified* remediation steps with the selected memory. If the match is not genuinely the same fault pattern, it resets `matched_memory_id = NULL` and routes to a new candidate rather than superseding unrelated history. `is_modified` is **always supplied as an explicit boolean** after this deterministic validation; the Part I optional parameter is refined to required in the HW2 design. A `PENDING_REVIEW` candidate is decided by the operator before its first `ACTIVE` insertion. The operator acceptance/discard interaction is shown as a **conceptual UC-07 workflow only**: no additional public REST endpoint or API request/response shape is specified in this section. The Web/API owner will decide how this interaction maps to the existing API contract. Until that decision is agreed, unmatched candidates remain non-indexed in `PENDING_REVIEW`; the design must not silently promote them to `ACTIVE`. Any eventual operator decision must be authenticated, version-checked, idempotent, and durable so worker retries cannot duplicate a memory entry.

#### 3.2.6 Integrity, Safety, and API Alignment

The schema implements `CHG-01` by storing pre-mutation backups, compensation commands and individual execution outcomes; `CHG-02` by storing per-node lease ownership/generation and incident coalescing counts; and `CHG-03` by recording **reviewed tool outcomes** without granting database or SSH privileges to the LLM. All persisted diagnostic text, memory content and event payloads must pass local secret redaction before commit. SSH credentials are encrypted at rest and are never represented in responses or SSE payloads.

For the proposed team REST/SSE design, resource changes and their corresponding outbox events are committed atomically; the REST/SSE gateway can replay events by increasing event `id` after reconnection. Events are retained for up to 24 hours or 100,000 rows; a replay request older than retained history receives `stream.reset` and must fetch a fresh `/sync` snapshot. The UI's `GET /memories`, `GET /memories/{id}`, `PATCH /memories/{id}` and archive action map to `memories` and `memories_fts`, while node and incident DTO versions map to their corresponding `version` fields. A plan approval is bound to the current incident/plan revision and an idempotency key. This design keeps the Agent & Memory subsystem's persistence consistent with the separate SysOps SSH & Safety and Web/API designs without treating model output as an authoritative database mutation.

**Implementation note:** The ER diagram and table/index definitions above are the proposed HW2 design artifact. A separate executable SQL schema may be developed for Part III, but no companion `.sql` file is included with this PR. For Part III the new work-queue and review-state columns must also be reflected in the executable migration and tested against SQLite foreign-key/FTS5 behavior.

### 3.3 Agent & Memory Diagrams

This section specifies the interaction order for the Agent & Memory Orchestrator's two principal use cases. **Sequence 1** refines autonomous diagnosis (`UC-04`) with command-safety review and log sanitization (`UC-09`); **Sequence 2** refines adaptive memory distillation after a verified incident resolution (`UC-07`). The diagrams complement the component responsibilities defined in Section 2.7 and use the versioned SQLite persistence model from Section 3.2. All LLM-generated tool requests cross a **deterministic Go Tool Execution Interceptor**: the Command Review Agent is a classifier, never an SSH executor (`CHG-03`).

#### 3.3.1 Sequence 1: Autonomous Root-Cause Diagnosis, Tool Safety Review, and Log Redaction (`UC-04`, `UC-09`)

```mermaid
sequenceDiagram
    autonumber
    participant C as Node Lease Coordinator
    participant O as Agent Orchestrator
    participant M as Memory Service
    participant DB as SQLite
    participant D as Diagnostic Agent / LLM
    participant I as Go Tool Interceptor
    participant R as Command Review Agent
    participant E as SSH Executor
    participant S as Local Redaction & Output Scope Guard
    participant UI as Gin API / Web UI

    Note over C,O: UC-03 detects drift, coordinator owns DIAGNOSING generation
    C->>O: Start diagnosis(incident_id, manifest, drift, lease_generation)
    O->>M: search_memory(keywords, trusted scope_context, count=3)
    M->>DB: FTS5 ACTIVE + compatible OS/service scope + BM25
    DB-->>M: Ranked verified memories within size budget
    M-->>O: Up to 3 bounded advisory memories
    O->>DB: Persist selected matched_memory_id and version
    O->>O: Assemble bounded, sanitized context (logs are untrusted data)

    loop ReAct turns up to 8 AND before 120s deadline
        O->>D: Request next diagnosis step (per-call deadline)
        alt LLM request timeout or overall deadline
            O->>D: Cancel in-flight request
            O->>DB: Record timeout and ESCALATED status
            O->>C: Stop diagnosis lease renewal and start safe-release protocol
        else LLM responds
            D-->>O: Tool call exec_readonly_shell(incident_id,node_id,command,scope)
            O->>I: Intercept tool call with current lease generation
            I->>I: Parse AST, allowlist command/args/path, reject inline secrets
            alt Unsupported scope, inline secret or unsafe syntax
                I-->>O: Fail closed locally without transmitting to reviewer
            else Locally admissible and safe to disclose
                I->>R: Send credential-free command for risk classification
                R-->>I: READ_ONLY / MUTATING / DESTRUCTIVE
                alt READ_ONLY AND bounded data-access scope
                    I->>C: Recheck current owner/generation before dispatch
                    C-->>I: Current generation and no uncertain predecessor
                    I->>E: Dispatch only if lease is still current
                    E-->>S: Bounded SSH output (local only)
                    S->>S: Redact and verify outbound data scope
                    alt Safe bounded observation
                        S-->>O: Sanitized observation and exit/timeout result
                        O->>DB: Persist safe evidence and auditable tool outcome
                    else Potential private data or oversized result
                        S-->>O: Withhold payload and return a safe error summary
                        O->>DB: Persist sanitized denial
                    end
                else MUTATING or ambiguous
                    I-->>O: No autonomous execution and defer to human plan
                else DESTRUCTIVE
                    I-->>O: Hard block and audit denial
                end
            end
        end
    end

    alt Sufficient evidence and within time/turn limits
        D-->>O: propose_remediation_plan(incident_id,root_cause,actions)
        O->>I: Submit immutable proposal
        I->>I: Verify command/backup_cmd/rollback_cmd, reject inline secrets
        I->>R: Classify locally validated credential-free commands
        R-->>I: Safety verdicts (classifier only)
        alt Any invalid or destructive step
            I-->>O: Reject or escalate without execution
        else Reviewed non-destructive plan
            I->>DB: Save immutable plan revision and PENDING_APPROVAL
            DB-->>UI: Sanitized committed event available to SSE
            Note over UI,E: UC-05 human approval still required for mutation
        end
    else Turn limit or safe progress exhausted
        O->>DB: Save evidence and ESCALATED status without mutation
    end
```

**Interaction rationale.** Before issuing model calls, the orchestrator performs scope-constrained FTS5 retrieval using trusted OS/service metadata and persists the chosen matched memory ID/version with the incident. Historical fixes and service logs are treated as *untrusted evidence*, not instructions. Per-record and total context budgets limit injected text; `max_turns = 8` and explicit LLM/request deadlines bound time, not merely completed turns.

The deterministic interceptor validates the *whole* shell command against a narrow diagnostic allowlist and its path/argument scope, rejects embedded credentials before sending anything to a potentially external Review Agent, and consults that reviewer only as a classifier. The reviewer does not execute commands. Even a `READ_ONLY` verdict cannot authorize arbitrary file reads; the SSH dispatcher checks the current node lease owner/generation immediately before dispatch. The previous lease owner must have no unknown remote execution before another generation takes over. SSH output is redacted and checked for allowed data scope *before* entering prompts, storage or SSE; sensitive or unbounded results are withheld.

A valid diagnosis yields an immutable, versioned proposal with reviewed `command`, `backup_cmd`, and `rollback_cmd`. It enters `PENDING_APPROVAL`, not automatic execution. Uncertain, invalid, unsafe or timed-out diagnosis escalates with sanitized evidence. The actual remediation and rollback remain under the human-approved UC-05 execution subsystem, consistent with `CHG-01` to `CHG-03` and the pending team SSH design.

#### 3.3.2 Sequence 2: Verified Incident Distillation and Three-Branch Memory Evolution (`UC-07`)

```mermaid
sequenceDiagram
    autonumber
    participant C as Incident Coordinator / UC-05
    participant O as Agent & Memory Orchestrator
    participant S as Local Secret Redaction
    participant L as LLM Distiller
    participant M as Memory Service
    participant DB as SQLite + FTS5
    participant G as Gin API / SSE Dispatcher
    actor U as Operator / Memory Hub

    Note over C,DB: Desired-state verification passed and unique incident-scoped job required
    C->>DB: Atomically set RESOLVED and resolved_at and enqueue PENDING distillation job
    Note over C,O: Crash-safe work is durable before any asynchronous trigger
    O->>DB: Atomically claim PENDING job (or retry interrupted job)
    DB-->>O: Incident, persisted matched_memory_id/version, verified fix, operator edits
    O->>S: Local redaction and data-scope verification
    S-->>O: Bounded sanitized incident evidence
    O->>L: Distill structured candidate with request timeout
    L-->>O: keywords, root_cause, steps, scope_tags
    O->>S: Revalidate and re-redact candidate fields
    S-->>O: Sanitized structured memory candidate
    O->>M: Verify current ACTIVE match, environment scope and canonical fix
    M->>DB: Load matched version and current lineage (if any)
    DB-->>M: Active record or missing/incompatible match
    M->>M: Deterministically set matched_memory_id and required is_modified
    O->>M: commit_memory_record(incident_id,candidate,matched_memory_id,is_modified)
    M->>DB: Start IMMEDIATE transaction and recheck RESOLVED with unique job receipt

    alt Already COMPLETED or DISCARDED (idempotent retry)
        DB-->>M: Return prior outcome, do not double-count
    else No confirmed match - new pattern
        M->>DB: Store candidate_json and commit job state PENDING_REVIEW
        M-->>G: Publish safe candidate review notification
        G-->>U: Conceptual UC-07 candidate review (API binding TBD)
        alt Operator discards one-off
            U->>G: Discard candidate
            G->>DB: Commit DISCARDED receipt without creating an ACTIVE memory
        else Operator accepts
            U->>G: Accept candidate
            G->>DB: INSERT new ACTIVE memory with last_verified_at=resolved_at
            G->>DB: Commit job COMPLETED with memory.updated event
            DB->>DB: FTS5 indexes new accepted memory
        end
    else Matched ACTIVE memory, unchanged verified fix
        M->>DB: Increment hit_count and version and set last_verified_at to resolved_at
        M->>DB: Commit job COMPLETED with memory.updated event
    else Matched ACTIVE memory, verified modified fix
        M->>DB: Set previous status=SUPERSEDED
        M->>DB: INSERT ACTIVE successor with supersedes_id and last_verified_at=resolved_at
        M->>DB: Commit job COMPLETED with memory.updated event
        DB->>DB: FTS5 indexes only currently ACTIVE version
    end

    G->>DB: Read committed events in monotonically increasing ID
    DB-->>G: Sanitized memory.updated or candidate status
    G-->>U: SSE notification to refresh candidate review or GET /memories as appropriate
    Note over U,DB: Archived and superseded memories excluded from default search
```

**Crash-safe trigger and provenance.** A verified `RESOLVED` transition and one `PENDING` distillation job are inserted in the same SQLite transaction. A worker atomically claims the durable job and retries interrupted work after restart; the memory service uses the incident ID to verify resolution and enforce at-most-once materialization via a unique incident receipt. The prior matched memory ID and version were saved during UC-04, so a delayed worker can recover the correct historical context.

**Deterministic matching.** The LLM distiller proposes sanitized text but is not trusted to choose the database update branch. The Memory Service validates any selected memory against the current `ACTIVE` lineage, OS/service scope, and the incident's verified root cause/repair. It compares normalized verified steps (including operator edits) against the matched verified recipe, not merely FTS score. Unrelated or uncertain matches are routed to a new candidate; otherwise the service supplies the explicit required `is_modified` boolean. `matched_memory_id` may be null only for an unmatched candidate.

**New candidate review and three-branch policy.** The operator-facing accept/discard transport remains an open integration decision with the Web/API owner; this sequence does not define a new public route. For an unmatched incident, the structured candidate remains `PENDING_REVIEW` and is **not** indexed into active FTS5 until the operator accepts it. The operator may discard a one-off anomaly; a `DISCARDED` receipt remains for idempotency. Accepted new memories set `last_verified_at` from the incident's actual verification time. An unchanged matched fix increments `hit_count`, while a modified matched fix creates a new active successor and supersedes the prior version; both use the incident verification time. Edits later made only through UC-08 must not imply a newly verified repair. Any `COMPLETED` or `DISCARDED` job is idempotent, so repeated signals cannot create duplicate versions or double-count use.

**API and safety alignment.** All evidence and distillation outputs are locally redacted and size-bounded before model transmission or persistence. Resource changes and sanitized `memory.updated` events are committed atomically to the durable event log. The Web/UI dispatcher streams the event only after commit, and default retrieval excludes `ARCHIVED` or `SUPERSEDED` records. Distillation failure does not undo a verified incident resolution; the durable work record transitions to retriable `FAILED` with bounded backoff and explicit operator visibility.

### 3.4 State Machine Diagrams

【刘宸瑞】

### 3.5 Drift & Remediation Diagrams

【刘宸瑞】

### 3.6 API Interfaces

【马小云】

---

## 4. Updated Requirements

1. **CHG-01 — Transactional Remediation & Rollback (`SRS 5.2.3 UC-05` & `5.4 propose_remediation_plan`)**
    - **Change**: Replaced atomic batch execution with sequential fail-fast execution backed by mandatory pre-mutation snapshots (`backup_cmd` saving state to `/var/tmp/sysops-backup/<incident_id>/`, verified with `exit_code == 0` before running `command`) and paired compensation commands (`rollback_cmd`). If step $k$ fails, remaining steps $k+1..N$ are skipped, the incident enters `Partially_Failed`, and the Web UI exposes per-step statuses (`Succeeded`, `Failed`, `Skipped`) with one-click **Auto-Rollback** ($k-1 \dots 1$) or re-planning.
    - **Rationale**: Multi-step SSH mutations are non-atomic; midway failure leaves target nodes in an inconsistent state without pre-change backups and compensation logic.
2. **CHG-02 — Per-Node Concurrency Locking & Coalescing (`SRS 5.1 UC-03`, `5.2.2 UC-04` & `5.2.3 UC-05`)**
    - **Change**: Introduced a per-node lease lock (`IDLE`, `PROBING`, `DIAGNOSING`, `PENDING_APPROVAL`, `REMEDIATING`) in SQLite. On drift detection, if a node is locked by the same drift signature, `coalesced_count` increments without spawning a second LLM loop; if locked by a distinct drift signature, the new incident enters `Queued_On_Lock` and re-probes the node upon lock release. Active `REMEDIATING` locks suppress routine `UC-03` probe ticks.
    - **Rationale**: Prevents duplicate LLM loops, concurrent SSH write races, and false-positive drift alerts while services restart.
3. **CHG-03 — Tool Execution Interceptor Boundary (`SRS 5.2.2 UC-04`, `5.2.5 UC-09` & `5.4 Diagram`)**
    - **Change**: Replaced direct agent-to-agent tool routing in the Part I component diagram with a deterministic Go Tool Execution Interceptor. The Diagnostic Agent initiates all tool calls; the Interceptor queries the Review Agent strictly for a safety verdict (`READ_ONLY`, `MUTATING`, `DESTRUCTIVE`) and dispatches read-only probes or suspends mutating plans for `UC-05` approval.
    - **Rationale**: Aligns the structural model with `UC-04`/`UC-09` semantics, where the Diagnostic Agent calls tools and the Review Agent acts solely as a policy classifier.

---

## 5. UI and UX Design

### 5.1 Key Screens Mockups

【马小云】

### 5.2 User Flows

【马小云】

---

## 6. Open Issues and Risks

【每个人都撰写至少一条，最后徐佳咏整理总结】

【徐之涵】【马小云】【刘宸瑞】

1. **Lock Starvation & Cascade Resolution of Queued Incidents**:
    - Uncertainty: While an incident waits in `PENDING_APPROVAL` for operator review, distinct drifts on the same node wait in `Queued_On_Lock`. Delayed human approval could delay diagnosing secondary drifts, or the primary fix (e.g., freeing disk space) might implicitly resolve the queued drift.
    - Part III PoC Validation: Test a two-tier lock (allowing read-only `UC-04` diagnosis under a shared lock while `PENDING_APPROVAL` is idle, reserving exclusive locks for `REMEDIATING`) and enforce an automatic `UC-03` re-probe on dequeue to auto-close already healed incidents.


**R-XZH: Stale or Misleading Memory Retrieval across Host Versions.**

- **Uncertainty:** Keyword-based FTS5 can retrieve a previously successful fix that shares an error string with a new incident but was verified on a different Linux distribution, package version, service configuration, or host. Conversely, semantically similar issues with different wording may be missed. `hit_count` measures previously verified reuse, not present-day applicability. A historical repair must not become an automatic command merely because it ranks highly.

- **Proposed mitigation:** Store source-incident provenance, verification timestamps, scope tags, and immutable version lineage; exclude `ARCHIVED` and `SUPERSEDED` entries from default retrieval; expose retrieval evidence to the diagnostic agent as context, not as a privileged instruction. Require fresh read-only diagnostics and independent review of any proposed command, followed by explicit human approval for mutation. Consider an age warning and scope-based ranking rules where environment information is available.

- **Part III PoC validation:** Construct a labeled set of repeated and near-duplicate incidents (including mismatched OS/service versions and paraphrased log messages). Compare top-3 FTS5 retrieval results with expected relevant memories, record precision@3 and recall@3, and manually inspect false matches. Verify that archived or superseded records are not injected into default context, that a mismatched historical fix cannot bypass command review or HITL approval, and that an operator-edited replacement keeps the old memory in the audit trail. The exact acceptance thresholds should be agreed with the team before testing, rather than reported as measured results in Part II.

---

## 7. AI Tools Reflection

【每个人都撰写，最后徐佳咏整理总结】

### xjy

- Tool chain: Gemini 4 Argon in Google AI Studio and OpenCode
- Cost: covered by Google AI Pro & OpenCode Go subscription
- Pros / cons: communicate about the project concept, helpful in brain storming, English translations, and generating UML diagrams, but easily provides ideas that are too complex or too simple
- Verification: manually review & rewrite
- Enhancement: correct & rewrite AI output to align with our true design intent
- Artifacts: no specific artifacts

### xzh

- Tool chain: ChatGPT for technical writing, Mermaid diagrams, and relational data modeling
- Cost: covered by ChatGPT subscription
- Pros / cons: accelerated drafting and highlighted design dependencies, but could introduce incorrect details or overlook edge cases
- Verification: reviewed the drafts against the SRS, team design, and related pull requests; checked the diagrams and integration
- Enhancement: corrected and revised AI-generated text and diagrams to align with the SRS and team design
- Artifacts: 2.7, 3.2, 3.3, ADR-02, and related risk and references

### mxy

【马小云】

### lcr

【刘宸瑞】

---

## 8. Annotated References

【每个人按需撰写】

【徐之涵】【马小云】【刘宸瑞】

1. **B. Beyer, C. Jones, J. Petoff, and N. R. Murphy, Eds., _Site Reliability Engineering: How Google Runs Production Systems_. Sebastopol, CA, USA: O'Reilly Media, 2016.**
   This foundational work formalizes Site Reliability Engineering (SRE) principles including declarative state management, toil elimination through idempotent automation, alert deduplication, and safe change management. Chapters on Eliminating Toil, Practical Alerting, and Data Processing Pipelines demonstrate how uncoordinated automation and concurrent alert storms often worsen production incidents rather than resolving them.
   These principles directly shaped our State Reconciliation Engine and Per-Node Lease Lock Coordinator (`Section 2.1`, `ADR-01`). Rather than treating periodic SSH checks as stateless cron jobs that blindly invoke an LLM on every failure tick, we separated deterministic drift detection (`UC-03`, \$0 token cost) from bounded agentic diagnosis (`UC-04`). Specifically, the book's analysis of concurrent automation hazards motivated our per-node lock state machine (`IDLE`, `PROBING`, `DIAGNOSING`, `PENDING_APPROVAL`, `REMEDIATING`), which coalesces duplicate drift alerts, serializes distinct faults on the same host, and suppresses routine health probes during active `REMEDIATING` windows. Finally, its emphasis on post-change verification informed our mandatory post-remediation `UC-03` convergence check before marking any incident `Resolved`.


**S. Yao, J. Zhao, D. Yu, N. Du, I. Shafran, K. Narasimhan, and Y. Cao, “ReAct: Synergizing reasoning and acting in language models,” in *Proceedings of the International Conference on Learning Representations (ICLR)*, 2023.** https://arxiv.org/abs/2210.03629

Yao et al. introduce ReAct, a method that interleaves language-model reasoning with actions that gather information from tools or external environments. The core idea is that a system can revise its next action based on newly observed evidence rather than issuing an entire action plan without feedback. This is directly relevant to our `UC-04` diagnostic workflow: the Diagnostic Agent receives a drift report, proposes a bounded read-only inspection step, obtains a sanitized observation, and updates its diagnosis until it can justify a remediation proposal or reaches the configured turn ceiling. The paper informs the *reason–act–observe* structure, but it does not by itself provide security guarantees for a shell-capable agent. We therefore place a deterministic Go Tool Execution Interceptor between model-generated requests and privileged tools (`CHG-03`). The independent Command Review Agent supplies only a classification; the interceptor applies fail-closed policy, and mutation is reserved for the human-approved `UC-05` workflow. We also cap diagnosis at `max_turns = 8` and restrict logged output and memory retrieval to bounded context so an incident cannot produce an uncontrolled loop or context explosion. Thus, ReAct contributes the interaction pattern, while our own architecture adds explicit permissions, auditable handoff, safe degradation, and post-action verification appropriate to personal Linux operations. This separation is a central design decision because a convincing model-generated explanation is not equivalent to a safe command or a verified repair.

**SQLite Project, “SQLite FTS5 Extension,” SQLite Documentation.** https://www.sqlite.org/fts5.html

The SQLite FTS5 documentation describes an embedded full-text indexing facility, including `MATCH` queries, the built-in `bm25()` ranking function, external-content tables, and the triggers needed to keep an external-content index synchronized with its underlying relational data. It supplies the technical foundation for `ADR-02` and the retrieval design in Section 3.2. Our system stores canonical structured memory records in the ordinary `memories` table and maintains a `memories_fts` index for searchable symptoms, root causes, remediation steps, and environment scope. FTS5 allows exact operational terms such as service identifiers and error messages to be retrieved without deploying PostgreSQL or a separate embedding service; the returned top-k results then help initialize the Diagnostic Agent's context. The documentation also makes clear that external-content indexes must be actively synchronized, which motivates our insert/update/delete triggers and the planned checks for index consistency during Part III. We use BM25 only as a lexical relevance signal, then apply the application's own `ACTIVE` status and scope filters; a high search rank never authorizes execution or proves that a historical fix still applies. The design is intentionally modest: it accepts the limitations of keyword matching, retains auditable superseded versions, and treats all retrieved steps as untrusted historical evidence subject to current diagnosis and safety gates. The official reference therefore supports both our implementation choice and its stated trade-offs, rather than serving as an unsupported claim that FTS5 solves semantic retrieval.

---

## 9. Team member contributions

【最后徐佳咏撰写】
