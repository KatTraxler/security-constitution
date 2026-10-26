# Discovery Signal Register

## Reusable Documentation And API Clues

> **Purpose**: This register stores static clues that help reviewers find promising insecure-by-design candidates. A discovery signal is not a vulnerability class and is not sufficient evidence for canon promotion. It routes a case toward one or more Design Shape Register entries, probes, or first-principles hypotheses that can test whether the suspicious documentation pattern becomes a real missing-control flaw.

> **Cardinality:** Signals are extracted from reusable Shapes, not used as substitutes for DFC instances. Shape-to-Signal mapping is many-to-many: one Shape may produce several Signals, and the same Signal may route to several Shapes. Upstream, one DFC records one reported insecure-by-design instance and may map to one or more Shapes.

---

## How This File Relates to the Others

- `feature-abuse-reference.md` stores negative calibration cases the model should reject as non-flaws.
- `discovery-signal-register.md` stores documentation/API clues that route investigation.
- `design-shape-register.md` stores reusable vulnerability structures and missing-control tests.

Artifact flow: confirmed cases and feature-abuse counterexamples → discovery signals + design shapes → Phase 2 Security Constitution artifacts. The discovery signal register is a **Phase 1 artifact**; Phase 2 turns it into pattern-recognition and routing guidance.

---

## Entry Schema

| Field | Meaning |
|-------|---------|
| **ID** | Stable signal identifier |
| **Name** | Short clue title |
| **Signal question** | The documentation or API clue to look for |
| **Why suspicious** | Why the clue deserves follow-up |
| **Routes to** | Vulnerability shapes, hypotheses, or probes this signal should trigger |
| **Not sufficient when** | Evidence showing the clue is controlled feature behavior or not security-relevant |
| **Promotion test** | What must be shown before treating the case as a vulnerability candidate |
| **Security Constitution lesson** | Prompt guidance distilled from the signal |

> Entries describe the *clue* and the *structure* it routes to, not a reproduction recipe. Hunting candidate-lists of specific provider services have been removed in favor of the abstract field and documentation cues that generalize across clouds.

---

## Signals

### SIGNAL-001: Managed Service Agent Acts On Referenced Resource

| Field | Detail |
|-------|--------|
| **Signal question** | Does documentation or API schema say a provider-managed service agent, or a "service account configured for your project," reads, writes, imports, exports, syncs, runs code, pulls images, uses secrets/connections, or otherwise operates on caller-supplied resources? |
| **Why suspicious** | A managed service agent may hold effective resource authority broader than the invoking caller's direct IAM permissions. |
| **Routes to** | Primary: `SHAPE-001` (provider-managed service-agent effective-permission invocation without caller authorization). Also the cross-boundary data-movement and audit-attribution shapes when the agent then moves data or obscures attribution. |
| **Not sufficient when** | The API performs caller-bound authorization on every referenced resource, requires explicit service-account delegation, or only operates on resources the caller already controls. |
| **Promotion test** | Show that a low-privilege caller with invoke permission but no direct resource permission can cause the managed service agent to access the referenced resource or produce downstream output. |
| **Signal cues (abstract)** | Request fields that name a caller-supplied source/sink/code/image/secret/connection/resource URI; docs phrases such as "service agent," "service account configured for your project," "default service account," "runs as," and "users with create permission can access data"; managed-agent roles carrying data-plane read/write permissions; audit chains where the caller creates a job and the managed agent performs the data-plane action. When the referenced resource is a **network attachment / PSC interface / VPC connector**, route to `SIGNAL-010` / `SHAPE-013` — the privileged action is then network presence inside the customer VPC, not a data read. |
| **Security Constitution lesson** | Treat service-agent documentation as a routing clue. Promote only after checking whether the invoking caller was authorized for the resource the service agent used. |

### SIGNAL-010: Managed Service Reaches Private Customer Resources Via Caller-Supplied Network Attachment / PSC / VPC Connector

| Field | Detail |
|-------|--------|
| **Signal question** | Does documentation or API schema say a managed service connects to a customer's **private / internal-only** database or host using a **caller-supplied network or connectivity reference** — a PSC network attachment, Private Service Connect interface, VPC connector, private endpoint, or hybrid connection — while the managed service itself runs in a **provider-owned / shared producer project**? Equivalently: does a PSC network attachment's endpoints show a **shared producer project** appearing across different customer tenants, or is the attachment set to automatically accept connections? |
| **Why suspicious** | The connectivity resource is a cross-boundary reference, and the deputy's privileged action is **establishing network presence inside the customer's VPC** rather than reading a storage/IAM resource. If the managed service does not bind the caller's authorization to that network resource and the producer identity is **shared across all tenants**, a caller with no rights to another tenant can drive the shared fleet into that tenant's private network — gated only by an owner-side accept-list that cannot distinguish the owner's own use from an attacker's. An automatic-accept default makes the exposed state the realistic one, and the shared producer is what lifts impact from within-tenant to cross-tenant. |
| **Routes to** | Primary: `SHAPE-013` (caller-supplied network-attachment deputy). Also `SHAPE-001` when the referent is a data resource, and the cross-boundary data-movement shape when the deputy then moves data over the network presence. Compose with the cross-service-handoff and shared-fleet-severity signals. |
| **Not sufficient when** | The service requires the caller to hold `compute.networkAttachments.use` (or ownership/grant) on the referenced attachment before the deputy acts; the deputy connects through a customer-managed connector identity gated by the attachment gate; the producer identity is **per-tenant** so owner-side allowlisting scopes admission to the owner's own use; or the deputy can only reach networks the caller already controls. Reaching a private resource inside the caller's own VPC is not the signal. |
| **Promotion test** | A low-privilege caller in tenant B (no `.use`, no compute permission, no access to victim tenant A) supplies A's network attachment plus a private internal host inside A's VPC and triggers the managed operation; show the shared producer attaches inside A's VPC and data from A's private source becomes observable to the caller, or the caller obtains an internal reachability/port oracle. |
| **Signal cues (abstract)** | Request fields a network attachment, PSC interface, VPC connector, private endpoint, or egress setting; docs phrases such as "network attachment," "Private Service Connect interface," "so the service can reach resources in your VPC / on private IP / on-premises," and "the connector runs in a provider-managed project"; an automatic-accept connection preference; a **shared producer project** observed across multiple tenants. |
| **Security Constitution lesson** | A caller-supplied network/connectivity reference is a routing clue for a network-presence confused deputy. Do not stop at "the service can reach my private DB" — ask (a) whether the managed deputy binds the caller's authorization to the network resource, and (b) whether the producer identity is shared across tenants. Two distinct impact axes, not to be conflated: **cross-tenant** impact requires a **shared fleet** (a per-tenant producer is cross-tenant-safe incidentally); **intra-project** impact is conditional and often blind/write-only — treat it as an authorization-model observation until concrete escalation is shown. |

### SIGNAL-015: Caller-Supplied Customer-Managed Run-As Service Account On A Binding/Activation Method

> **Scope:** scan both persistent create/update bindings and one-shot/lifecycle activation methods (restore, clone, validate, build, deploy, submit, launch, execute). A one-shot hit is only an admission signal until the selected customer-managed SA is shown to perform the protected effect; invoking an already-bound identity is the binding-vs-use negative.
>
> **Token-mint sibling — the delegation permission is not always the attachment gate.** A class of caller-supplied SA fields do not make the resource *run as* the SA; instead a service **mints the SA's token** and attaches it to an outbound request. The governing permission is then the token-issuance permission (`getAccessToken` for an OAuth token, `getOpenIdToken` for OIDC), not the attachment gate. Split on the token's replayability: a replayable OAuth token raises the ceiling (`IMPACT-001`); an audience-bound OIDC token is inert. Read the field's documentation first: a **caller-side** gate ("the caller must hold the token/act-as permission") means the gap is real if unenforced; a **deputy-side** gate ("granted to the impersonator service agent") is inert by default. Many such gates are enforced; treat an unstated gate as the live-worthy lead.

| Field | Detail |
|-------|--------|
| **Signal question** | Does a product API **binding or activation** method accept a caller-supplied field naming a **customer-managed** service account that the resource or operation will **run as** (e.g. `serviceAccount`, `serviceAccountEmail`, `runtimeServiceAccount`) — an identity-binding field, distinct from a provider-managed service agent — and fail to enforce the caller's attachment gate (`iam.serviceAccounts.actAs`)? |
| **Why suspicious** | Binding or activating a customer-managed SA is exactly the action the attachment gate governs. If a method does not enforce it, a low-privilege caller can select an arbitrary, more-privileged SA for a persistent resource or one-shot operation — a privilege escalation whose ceiling is the most-privileged SA in the project. The gate governs binding/activation, not later use, so ordinary invocation of an already-bound resource remains a negative. |
| **Routes to** | Primary: `SHAPE-016` (customer-managed run-as binding without the attachment gate). **If the run-as identity is a provider-managed service agent** instead, route to `SIGNAL-001` / `SHAPE-001` — the attachment gate is structurally inapplicable there and the control is a caller-bound check on the referenced resource. |
| **Not sufficient when** | The attachment gate is enforced (the no-gate caller is denied); the field is immutable / output-only / not in the update mask; the field supplies operational input that cannot change the execution identity; the named identity is a provider-managed service agent (→ `SIGNAL-001`); or the caller already holds the gate (no delta). A caller-bound check on a resource the SA later reads is the `SHAPE-001` control, not this one. |
| **Promotion test** | As a caller holding the product create/update permission but no attachment gate on a named target SA, call the binding method naming that SA; confirm acceptance (no gate denial) and that the **bound SA — not the caller —** performs the privileged action (via access logs). Calibrate against an enforcing sibling (a create method that denies the same caller+SA on the attachment gate). For a patch variant, also test whether the SA field is accepted in the update mask (mutable) or rejected (immutable). |
| **Signal cues (abstract)** | Request fields in run-as position (`serviceAccount`, `serviceAccountEmail`, `runtimeServiceAccount`, `*.serviceAccount`); doc phrases "the service account the resource runs as," "specify a service account," "run-as," "identity used by …"; a `create` or an `update`/`patch` whose update mask includes the SA field; a method-granting role that lacks the attachment gate. Distinguish from provider-managed service-agent references (→ `SIGNAL-001`) and from output-only / owner / created-by fields. Mature services generally enforce the gate; the signal concentrates in newer / off-list services, and a same-feature sibling that enforces is the best calibration. |
| **Security Constitution lesson** | A caller-supplied customer-managed run-as SA field routes to `SHAPE-016`. Promote only after confirming the named identity is customer-managed (not a provider agent — that's `SIGNAL-001`), the field actually changes the execution identity (not operational input), and the method does not enforce the caller's attachment gate. The service agent that physically impersonates the bound SA is normal run-as plumbing, never the locus. |
