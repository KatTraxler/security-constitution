# Security Constitution

## Operational Lessons and First Principles for AI-Assisted Design-Flaw Detection

---

## 1. Preamble

### What This Document Is

The Security Constitution is the core intellectual contribution of this research. It encodes the researcher's design-flaw reasoning procedure into a format an AI model can operationalize — a set of first principles, anchors of trust, operational lessons, and calibration guidance that together teach a model to distinguish insecure-by-design flaws from feature abuse, classic bugs, and legitimate powerful features.

This document captures the **high-level operational lessons** distilled from validated findings — the principles the Constitution must teach a model to apply. These lessons are grounded in observed behavior, not speculation. Every principle statement traces to a specific validated finding with a cited DFC, SHAPE, or SIGNAL identifier.

### What This Document Is Not

This is not a checklist. It is not a vulnerability scanner. It does not replace the Phase 1 taxonomy artifacts — the Discovery Signal Register, Design Shape Register, and Feature Abuse Reference Catalog remain the canonical corpus. This document synthesizes those artifacts into promptable guidance.

### How It Relates to Other Artifacts

This document has five parts: the **First Principles** of secure design; the **Anchors of Trust** the principles decompose into; the **Operational Lessons** distilled from real design-flaw review; **Calibration Guidance**, including the reasonable-customer test that separates a design flaw from feature abuse; and the guiding maxim, *hypothesize statically, validate by execution*.

### Evidence Standard

The principles and lessons here are distilled from findings validated by execution across multiple cloud providers; the specific findings are presented in the talk and companion write-ups, not in this edition. A judgment reasoned from the Constitution is documentation signal until it is corroborated by execution — *hypothesize statically, validate by execution; anything else is documentation signal, not a finding.*

---

## 2. First Principles of Secure Design

These principles define the security guarantees a cloud service must provide. The first eight are foundational; Principles 9 through 12 were added from design-flaw review across the major clouds.

### Principle 1: Authorization at Every Trust Boundary

Authentication and authorization must be verified at every trust boundary crossing. A trust boundary exists wherever the acting principal, the authorization domain, the tenant, or the resource scope changes.

### Principle 2: Auditable Security-Relevant Actions

Actions with security impact must be logged and auditable. The audit trail must attribute the action to the initiating principal, not only to the platform intermediary that executed it.

### Principle 3: Least Privilege for both Humans and NHIs alike

The principle of least privilege applies equally to human principals (users, operators, administrators) and to non-human identities (service agents, managed identities, platform-level identities, and workload identities) — not just one or the other. A human principal should hold no more standing authority than their role requires, and a non-human identity that acts on behalf of a caller should hold no more authority than the specific operation requires. In both cases that authority should be bound to the acting principal's own authorization where the operation crosses a resource boundary — a human should not inherit a deputy's reach, and a deputy should not exceed the human caller's.

### Principle 4: Customer Control Over Risky Functionality

Customers must have the ability to restrict, disable, scope, or govern risky functionality. The absence of an opt-out, guardrail, or governance control for a capability with security impact is itself a design flaw.

### Principle 5: Tenant Isolation

Tenant isolation must be maintained for all operations. Normal use of a feature must not permit effective access, influence, or data movement across customer, account, organization, or tenant boundaries without explicit, auditable authorization from both sides.

### Principle 6: Internal Protocols Held to Public API Standards

Internal or legacy protocols exposed to customer-accessible infrastructure must be held to the same security standards as public APIs. A protocol that was acceptable in an internal-only context does not inherit that safety when exposed through a customer-facing surface.

### Principle 7: Credential Lifecycle Governance

Long-lived credentials must have administrator visibility, lifecycle controls (creation limits, expiration), and clear revocation semantics. Direct credential disablement should invalidate the credential. Ordinary account remediation must not leave separate durable credentials undiscoverable or unrevoked.

### Principle 8: Bounded Blast Radius for Delegation

The blast radius of any impersonation or delegation mechanism must be bounded to a single tenant. A platform identity, delegation chain, or trust relationship that can cross tenant boundaries without explicit authorization from the target tenant is a design flaw.

### Principle 9: Cross-Domain Authority Verification (New — Validated)

**When a platform operation bridges between two independently-governed authorization domains, the platform must verify the caller's authority in the *target* domain, not just the *source* domain.**

Authorization checked in the wrong domain is authorization not checked at all for the purpose of the boundary crossing. Two individually correct authorization checks — one in the source domain, one applied to a platform intermediary in the target domain — can fail to compose securely when no component verifies the *caller's* standing in the target domain.

This principle refines Principle 1 (authorization at every trust boundary) for the specific case of domain transitions: control-plane to data-plane, management to runtime, reference to content. A review must therefore ask, for each such transition: "Is the caller authorized in the target domain, or only in the source domain?"


### Principle 10: Secret Custody Follows Secret Use (New — Validated)

**A secret is held by whoever can use it. When a platform places a secret, or an identity that can exercise one, into an execution environment, it must authorize that delivery against every principal who can run code there, and treat each of them as holding the secret. Export restrictions limit how the secret travels; they do not limit who can act with it.**

An attacker who controls the execution environment can use the secret to sign, decrypt, authenticate, or connect without ever extracting it. The impact assessment must account for *use* of the secret, not just *extraction*. This informs the blast-radius assessment (Principle 8): the question is not "can the attacker export the secret?" but "what can the attacker do with access to its operations?" Non-exportability, hardware backing, and sealed storage reduce exfiltration risk; they do not change who can act with the secret. "Secret" here covers any credential the platform delivers — private keys and certificates, tokens, API keys, connection strings, mounted secrets — and any identity whose credential the environment can exercise.


### Principle 11: Network and Connectivity Resources Are Protected Resources (New — Validated)

**Establishing presence inside a network is a privileged act. A caller-supplied reference to a network or connectivity resource must be authorized against the invoking caller exactly as a reference to a storage object, table, or secret is.**

A managed deputy's privileged action is not always a data read. When a caller-controlled field names a connectivity resource (a network attachment, a VPC or serverless connector, a private endpoint, a hybrid connection) and a platform identity acts on it, the deputy can be made to originate traffic from inside a network the caller has no right to enter, reaching whatever is internally reachable there. Confused-deputy review must therefore treat "where does this deputy end up standing?" as a question of equal weight to "what does this deputy read?"

An owner-side admission control is not a substitute for caller-bound authorization. An accept-list on the target network answers "which producer may connect," not "is this caller authorized to use this attachment," and when the producer is a fleet shared across tenants it cannot distinguish the owner's own use from an attacker's. Authorization must bind the *caller* to the *specific* network resource before the deputy establishes presence; the owner's consent to the deputy is a separate and insufficient check.

This principle extends Principle 1 (authorization at every trust boundary) to the network domain and grounds anchor A5 (network source position). The structured review must check, for every caller-supplied connectivity reference, whether the caller holds use or ownership authority on that resource, and must apply Principle 5 and Lesson 11 when the deputy is shared across tenants, since that is the condition under which the pivot becomes cross-tenant.


### Principle 12: Cross-Service Authorization Completeness (New — Inferred)

**An authorization decision needs two inputs: the caller's authority and control over the target resource. When a request crosses between services, those inputs sit in different hands, and the platform must explicitly assign the decision to a party that holds both. A boundary with no assigned enforcer fails open.**

A confused deputy of this class is not a forgotten check. The service the caller invoked knows who the caller is but does not own the target; the service that owns the target sees only a trusted first-party deputy and assumes the caller was already vetted. Each side reasonably expects the other to decide, and neither can. Two mechanisms reunite the inputs: the calling service propagates the caller's verified identity to the resource service, which then authorizes the real caller against the target, or the resource service delegates an authorization query that carries the caller's identity back. Absent one of these, the boundary is a seam with no owner.

Authenticating a peer service is not the same as authorizing that service's claims about a third party's resource. Service trust and caller authority are independent questions, and a confused deputy is what happens when the first is accepted as an answer to the second.

This principle names the enforcement locus that Principle 1 leaves implicit. Principle 1 requires a check at every trust boundary and assumes someone is positioned to make it; Principle 9 asks whether that check is made in the right domain; this principle asks whether any party can make it at all. Anchor A13 marks the point in the request path where the caller's identity collapses into the deputy's, and therefore which party can still enforce. When the resource side admits only the deputy, through a shared provider fleet (A4) or an owner-side accept-list (A6), the failure goes cross-tenant, and Principle 5 and Lesson 11 apply.

---

## The Anchors of Trust (Generative Primitive Set)

The First Principles above say *authorization must hold at every trust boundary*. This section
enumerates the **anchors of trust** — the small set of primitives a cloud platform actually *roots a
grant or an action in*. A boundary is crossed safely when the anchor the platform trusts genuinely binds
the caller; a design flaw exists wherever the platform roots a decision in an anchor the attacker can
occupy, supply, induce, or that carries no principal to bind at all.

This set is **generative, not descriptive**: the Design Shape Register catalogues *specific*
trust-boundary structures; the anchors below are the *atoms those shapes are made of*. You discover a new
shape by taking an anchor and asking, for a service that has never been reviewed, *"where is trust rooted
here without binding the caller?"* — and, most productively, *"which two anchors compound?"* A candidate
that violates an anchor's expected binding but matches no existing shape is a **new-shape lead**, not an
unroutable miss.

For each anchor: what the platform roots trust in · the characteristic violation · the generative probe.

- **A1 — Caller-identity provenance (which domain the caller is authorized in).** Trust rooted in the
  authenticated principal's authority *in the domain where the call arrives*. **Violation:** the action
  lands in a different domain (control→data plane, management→runtime, one tenant→another, operation→named
  resource) where the caller's authority was never re-checked. *Probe:* for each domain transition in the
  code path, is the caller re-authorized in the **target** domain, or only the source?

- **A2 — Reference resolution (a caller-named resource the platform resolves and acts on).** Trust rooted
  in the *operation* permission, treated as if it also conferred authority over every resource the
  operation can name. **Violation:** operation-authority is substituted for resource-authority — no
  caller-bound check on the *referenced* resource. *Probe:* does the method check the caller's data-plane
  permission on the **referenced** object (URI, table, secret, vault), or only the method permission?

  **Sharpening — orchestration-wrapper charter.** The caller lacking resource-authority on the referent is
  *necessary but not sufficient*. A managed **orchestration wrapper** is *designed* so its service identity
  stands in for the underlying resource IAM — requiring the caller to hold **both** the wrapper role **and**
  the resource `.use` would defeat the abstraction. So this anchor graduates only when the reference lets
  the caller cross a boundary the orchestrator is **not chartered to let them cross**. Discriminating
  question: does the caller-missing `.use` gate a **trust / reachability boundary** the orchestrator was not
  chartered to bypass (→ finding), or a **resource the orchestrator exists to operate** on the caller's
  behalf (→ working-as-intended)? A permission that gates *who may bridge a workload into a network segment
  a network team fenced* (for example `compute.networkAttachments.use`) is a boundary the wrapper is not
  chartered to erase → finding; a permission that gates *reading a disk the wrapper's job is to attach and
  boot* is the operation the wrapper exists to perform → working-as-intended, unless the reference overruns
  the project/org boundary or makes the deputy assume a different-tenant identity.

  **Sharpening — caller-supplied referent vs hierarchy-resolved / provisioning target.** This anchor
  presupposes a referent the *caller supplies*; there is nothing to substitute operation-authority *for*
  when the target is **resolved from the resource hierarchy** (an ancestor folder or org config) rather than
  a caller field, or when it is a **freshly-provisioned** resource created on the caller's behalf rather than
  an *existing* resource the caller is denied. Two gates before drafting a candidate: (1) is the referenced
  target a **caller-supplied field**, or **hierarchy-resolved**? (2) is it an **existing** resource, or a
  **freshly-provisioned** one? Only *caller-supplied + existing-resource* referents are confused-deputy-read
  candidates; hierarchy-resolved or provisioning targets are hierarchy-gated or out of scope.

- **A3 — Delegation binding (a caller-named run-as / acting identity).** Trust rooted in the assumption
  that naming an identity implies authority to assume it. **Violation:** no caller-bound delegation gate
  (`actAs` / `PassRole` / an RBAC assign action) before a platform identity acts as the named identity; the
  deputy's authority is *independent* of the caller's. *Probe:* is a caller-bound delegation check enforced
  before the platform acts as the identity — and where that gate cannot apply (a provider-owned agent), is a
  backend caller-bound check present instead?

- **A4 — Deputy tenancy (who owns the acting identity).** The *severity* anchor. Trust rooted in a
  platform identity whose ownership may be per-tenant or shared across tenants. **Violation:** a shared,
  fleet, or provider-owned deputy cannot distinguish one tenant's use from another's, so nothing per-tenant
  binds its cross-boundary action → cross-tenant reach. *Probe:* is the acting identity the same producer /
  fleet / agent across two tenants? If yes, the impact ceiling is cross-tenant.

- **A5 — Network source position (where traffic originates).** Trust rooted in network locus — VPC
  membership, a private-connectivity producer, or a source IP range. **Two violation modes:** (a) a
  caller-supplied connectivity reference makes a deputy originate traffic from *inside* a network the caller
  has no right to enter; (b) an **identity-free variant** — trust rooted in a *shared source range that
  carries no per-request identity* (for example a provider's documented mandatory-allow health-check
  ranges): there is no principal to authorize, so the trust failure is entirely **victim-side** (a service
  that firewall-trusts the range). *Probe:* does any decision trust a network position the attacker can
  occupy or induce traffic from — and if the anchor carries no identity, is the whole trust misplaced onto
  the victim side?

- **A6 — Consent / allowlist configuration (an owner-authored accept-list).** Trust rooted in an owner
  opt-in: an automatic-accept connection preference, an "allow trusted provider services" flag, a resource
  policy without a source condition. **Violation:** the config answers *"which producer or service may
  connect,"* not *"is this caller authorized"* — an owner opting into automatic acceptance does not
  authorize an arbitrary low-privilege caller; the risk is sharpest when provider documentation *recommends*
  the broad configuration as a feature precondition. *Probe:* is a caller-scoped authorization being
  substituted by an owner-scoped accept-list? Does provider-recommended config manufacture the precondition?

- **A7 — Token / credential authority (the power a bearer actually wields).** Trust rooted in a token's
  audience/scope or in stored-credential provenance. **Violation:** (a) audience-binding or
  non-exportability is assumed to bound replay, when the real power is the identity's *bound IAM*, not its
  scope; (b) a low-privilege caller redirects a high-privilege *stored* credential against an
  attacker-controlled target. *Probe:* what is the *bound IAM* of the identity behind the token (not its
  scope)? Can a caller point a stored high-privilege credential at an attacker target?

- **A8 — Resource-name / namespace provenance (a globally-unique name resolved to "the resource I expect").**
  Trust rooted in the assumption that a first-come, globally-unique name (a storage bucket, an artifact
  repository, a service's staging bucket, a service-account email, a project ID) is owned by the party the
  consumer intends. **Violation:** a low-privilege attacker *pre-creates or squats* the name before the
  victim's service resolves it, so the platform hands the victim's trust — a read, a dependency fetch, an
  image pull, a deploy — to an *attacker-owned* resource. No identity is shared and no reference is
  caller-supplied at the boundary; **the name itself is the shared object.** *Probe:* does this service
  resolve any globally-unique name an attacker can register first? Is name **ownership** (not merely name
  existence) verified before the platform trusts what the name points to?

- **A9 — Hierarchy authority scope (a parent-scope principal acting on child-scope resources).** Trust
  rooted in the assumption that authority granted at organization / folder / billing-account / group scope
  is *bounded to that scope* — when in fact it cascades **down** the resource hierarchy into resources that
  never authorized it. **Violation:** a principal with a parent-scope permission (an org or folder policy, a
  hierarchical firewall policy, an org-level log sink, a tag binding used in a child's IAM condition, a
  shared-VPC host authority, a group membership) produces an effect *inside a resource that never authorized
  it* — with **no shared identity and no caller-supplied reference**; the carrier is the hierarchy itself.

  **A9 splits in two, and the distinction is load-bearing:**
  - **A9-intra — cascade *within one organization*.** A parent-scope principal in org X produces an effect
    in a child resource also in org X. This is *not* the cross-tenant case: the org boundary is the tenant
    boundary, and an org/folder-scope grant is a legitimate insider. A9-intra is an *intra-tenant
    separation-of-duties* concern, structurally capped below cross-tenant severity, and prone to being
    working-as-intended (see the discriminator).
  - **A9-cross — cascade *across the organization boundary*.** The only A9 path to a true cross-tenant
    result: a hierarchy construct that spans *organizations*, where a parent-scope principal in org A
    produces an effect in a resource owned by org B. The obvious carriers (billing linkage, cross-org group
    membership, cross-org project/folder moves) are typically governed by bilateral consent or
    domain-restriction controls; a real A9-cross needs a genuinely non-obvious cross-org construct.
  - **Working-as-intended discriminator (apply to both):** is the parent-scope permission a *documented
    security-governance* role whose stated purpose **is** to govern descendants (org-policy admin, firewall
    admin, tag admin)? If so → intended delegation → working-as-intended. A9 graduates only when the
    cascading permission is a **benign, operational role that cascades incidentally** (a cost-tagging,
    billing, metadata, or grouping role nobody treats as security-sensitive), **or** the cascade **exceeds
    its documented scope** (folder-scope authority reaching outside that folder; parent authority silently
    overriding a child's explicitly-more-restrictive posture with no consent or notification).

  *Probe:* does a parent-scope write produce an effect in a resource that did not authorize it — and (1)
  does that resource cross the **org** boundary (A9-cross) or stay intra-org (A9-intra, separation-of-duties);
  (2) is the cascading permission an *incidental* operational role, or a *governance* role whose job is to
  govern descendants (working-as-intended)?

- **A10 — End-user identity assertion provenance (who binds a human to a user selector).** Trust rooted in
  an asserted user, subject, actor, account, or session identifier that selects per-user authority.
  **Violation:** a provider-owned authenticated surface promises, or reasonably implies, that the selector
  belongs to the signed-in human, but accepts a different user's selector and causes a protected cross-user
  effect. **Customer-boundary discriminator:** many enterprise products intentionally leave browser sign-in,
  session management, and the authenticated-subject-to-selector mapping to the customer application. A
  free-form selector in that application is a serious enterprise risk, but it is **not** a cloud-platform
  confused deputy unless the provider owns or promises the binding that failed. **Control-provenance rule:**
  do not infer enforcement from an association field; ablate it against the named permission in both
  directions before calling it a gate. *Probe:* identify who authenticated user B; retain that authenticated
  session; change only the selector to user A; require an A-only harmless effect and an acting-identity audit
  trail. If an unsafe customer frontend is the only mismatched path, record a hardening/documentation result,
  not a provider flaw.

- **A11 — Derived-state authorization provenance (a materialized derivative served under coarser authority
  than its source).** Trust rooted in the assumption that a **materialized derivative** — a search or vector
  index, an embedding, a cache, a consolidated "memory," a feature, a summary, or a fine-tuned model —
  carries the authorization of the **source** it was built from. **Violation:** the derivative is served
  under a *coarser or different* authorization than the source's per-item ACL, so *write-side* authorization
  (who may **ingest**) is decoupled from *read-side* authorization (who may **retrieve**), and a reader
  authorized on the derivative but not on the source obtains the source content. Sharpest when the
  derivative crosses an **authorization domain** or a tenant boundary at ingest. **Distinct from A2:** the
  deputy does not read a *live* caller-named resource at request time — the read already happened at
  **ingest**, and the flaw is the derivative **shedding the source ACL** between ingest and serve.
  **Graduation discriminator:** this anchor does **not** graduate on a *customer-configured* sharing or
  data-collaboration surface where the owner affirmatively grants every link in the derivative's chain and
  the consumer sees only curated *output* — that is customer-consented composition (Lesson 9), the inverse
  of the A6 provider-recommended-broad-config amplifier. It graduates where the *provider* builds the
  derivative and drops the source ACL, or where the derivative exposes the *source directly*, beyond a
  curated output, across a boundary the owner did not grant. *Probe:* for every service that ingests source
  data into a queryable store, is retrieval re-authorized against the **source object's** ACL (or an ACL
  faithfully copied into the derivative), or only against IAM on the **derivative store**? Where the provider
  ships a per-item control, test it for **bypass** on every serve path (an ACL-less query, a derived
  answer/summary/citation, a stale index entry).

- **A12 — Temporal binding of an authorization decision (a check at bind time trusted at effect time).**
  Trust rooted in an authorization **checked once** — at bind, schedule, cache, or preflight time — and
  assumed to still hold at **effect / serve** time. **Violation:** the referent's ownership or content, or
  the *caller's own standing*, changes between the check and the use, and no component re-authorizes at the
  point of effect; the stale decision is executed. **A8 is the spatial analog** (occupy a globally-unique
  name before it is resolved); A12 is the **temporal** analog (the referent, or the caller's authority,
  mutates *after* the decision). It also refines Principle 7 (revocation): a durable schedule that keeps
  acting as a service account after the creator loses `actAs`, or a cached artifact re-served after the
  caller's access is revoked, is a revocation that did not revoke. *Probe:* for every store-now-act-later,
  schedule, cache, or preflight path, is the authorization **re-evaluated at execution/serve**, or **frozen
  at bind**? Revoke the caller's authority *after* the bind and test the next effect.

- **A13 — Authorization-subject survival across a service hop (where the caller identity collapses to a
  deputy).** Trust rooted in the assumption that the *caller's own* authority is still present at the point
  the resource decision is made. **Violation:** on the calling-service → resource-service hop the request's
  effective subject collapses from the original caller to a deputy / service identity **before** the
  target-resource authorization, so the resource service is structurally *blind* — it can only check the
  deputy it sees, not the caller whose authority it cannot. Distinct from A1 (is the caller re-authorized in
  the target *domain* — which presupposes the caller's identity is *present*), A2 (does the caller-named
  *object* survive), and A3 (is the *acting* identity bound): A13 is the prior question — whether the
  *subject* survives the hop at all. **Cross-tenant escalation is governed by the target-side admission
  control:** the crossing goes cross-tenant exactly when the target's admission fails to bind the *original
  caller* — either the collapse target is a **shared provider fleet** (A4) or it is gated by an **owner-side
  accept-list** (A6) that admits the deputy without re-binding the caller. When instead the target requires a
  deputy IAM grant on the *specific* resource, the grant *is* the authorization and the crossing is bounded
  to intra-org. *Probe:* trace the identity-collapse point and the resource-authorization point in the
  two-service path — the flaw is present when the former precedes the latter with no caller-context
  propagation, and it is *cross-tenant* when the target's admission binds only the deputy, not the caller.

> **Why A8–A13 exist (a caution against the set's own bias).** A1–A7 are deputy-, reference-, network-, and
> token-centric because confirmed cases cluster there — but that lopsidedness is an availability artifact,
> not the true shape of cross-tenancy. Cross-tenancy is *any effect authorized in tenant A that lands in
> tenant B without B's authorization*; the shared **fleet identity** (A4) is only one *carrier*. A shared
> **namespace** (A8), a shared **hierarchy** (A9), a **materialized derivative that shed its source's ACL**
> (A11), a **decision frozen before its referent changed** (A12), and a **caller identity collapsed into a
> deputy before the resource check** (A13) are the other carriers a purely deputy-centric set omits. The test
> for a cross-tenant candidate is not "is the deputy a shared fleet?" but "does an action authorized in A
> produce an effect in B, and is B's **ownership of the affected resource** verified at the point of effect —
> or substituted by presence, a global name, a hierarchy position, a stale decision, a bare reference, a
> shared identity, or a derivative that dropped its source ACL?"

> **How to use the anchors.** The anchors are a synthesis layer over the Principles and Lessons, not an
> independent source of truth. A candidate that violates an anchor is documentation signal until it is
> confirmed by execution against the six-condition framework (Lesson 6). Several anchors — notably the
> derived-state (A11), temporal (A12), and subject-collapse (A13) carriers — are offered as **generative
> primitives** to point a review at under-examined surfaces, not as anchors with a settled, worked instance.

---

## 3. Operational Lessons from Validated Findings

This section is the heart of the Constitution. Each lesson is a self-contained principle distilled from validated findings, presented with the operational test it implies and its generalization beyond the specific case.

---

### Lesson 1: Cross-Domain Authority Translation

#### Principle

When a platform operation bridges between two independently-governed authorization domains (e.g., ARM control-plane to the managed secret/certificate store data-plane), the platform must verify the caller's authority in the *target* domain, not just the *source* domain. Authorization checked in the wrong domain is authorization not checked at all for the purpose of the boundary crossing.

#### Generalization

This pattern applies wherever two authorization systems meet through a platform intermediary:

- **Azure**: ARM control-plane authorization vs. the managed secret/certificate store / Storage / SQL data-plane authorization, bridged by an RP's PLMI.
- **GCP**: IAM `invoke` permission vs. Cloud Storage / a managed data-warehouse service data-plane authorization, bridged by a Google-managed service agent.
- **AWS**: IAM `Create*` permission vs. S3 / KMS data-plane authorization, bridged by a service-linked role or service principal .

The generalized test: when a platform operation involves a domain transition, does the authorization check at the boundary validate the caller's standing in the target domain, or only in the source domain?

---

### Lesson 2: The Linked-Resource Deputy Test

#### Principle

When an API accepts a caller-supplied resource identifier and a platform identity acts on the named resource: (a) does the platform check whether the *caller* is authorized on the *target resource's content*, or only whether the caller can *reference* the target? (b) Is the platform identity's authority on the target derived from the caller's authorization, or does it come from an independent grant?

If the answer to (a) is "reference only" and (b) is "independent grant," the path is a confused deputy candidate.

#### Generalization

The linked-resource deputy test is provider-agnostic:

- **GCP**: Does a managed document-processing service's batch method check the caller's `storage.objects.get` on the referenced storage URI, or only the service's own batch-process permission? (a reference-only input with an independent service-agent grant is the negative pattern.)
- **AWS**: Does a a managed ML notebook service notebook launch check the caller's `s3:GetObject` on the referenced bucket, or only a notebook-create permission? (A reference-only input plus an independent execution-role grant is the negative pattern.)
- **Azure**: Does a managed platform compute service cert import check the caller's the managed secret/certificate store DataActions on the referenced vault, or only a control-plane deploy permission? (: reference only + independent PLMI grant.)

The test discriminates between two structurally different authorization models: one where the platform's authority is derived from and bounded by the caller's authority (safe), and one where the platform has independent authority the caller can steer toward a target of their choice (confused deputy).


---

### Lesson 3: Authorization Present but Insufficient

#### Principle

A model must distinguish "authorization exists at a boundary" from "the *correct* authorization exists at a boundary." Two independently correct authorization checks can fail to compose securely when a platform bridges between their domains without verifying the caller's standing in both.

The critical failure mode the lesson catches: a model that pattern-matches on "authorization check exists at this boundary" and concludes the path is safe, when the authorization check validates the wrong principal (e.g., the PLMI, not the caller) in the target domain.

#### Generalization

This lesson applies to any multi-component authorization chain where:

- Component A verifies principal X's authority in domain A.
- Component B verifies principal Y's authority in domain B.
- The operation requires principal X to have authority in domain B, but no component checks this.

The pattern is not specific to a cloud, domain or to control-plane/data-plane splits. It occurs wherever two authorization systems are composed through an intermediary, and the composition is assumed to be transitive when it is not.


---

### Lesson 4: Provider-Recommended Configuration as Attack Precondition

#### Principle

When provider documentation instructs customers to grant a platform-owned identity data-plane access on a customer resource to enable a control-plane feature, route to the confused deputy shape. The documented configuration may itself assemble the preconditions for a boundary violation.

A finding that requires exotic, unusual, or explicitly-warned-against configuration is less impactful than one the vendor tells you to set up. Provider-recommended configurations that create vulnerability preconditions are a **materiality amplifier**, not a mitigating factor.

#### Generalization

The principle applies across providers:

- **AWS**: Does an AWS Quick Start, an infrastructure-as-code service template, or service documentation instruct the customer to create a resource policy trusting a service principal without `aws:SourceArn` / `aws:SourceAccount` conditions? 
- **GCP**: Does a GCP feature's documentation instruct the customer to grant a Google-managed service agent a broad IAM role on a customer resource as a prerequisite? 
- **Azure**: Does Microsoft documentation instruct granting RBAC to a Microsoft RP service principal or enabling "Allow trusted Microsoft services" as a prerequisite for a feature?

When the vendor's own documentation assembles the vulnerable configuration, the finding's materiality is amplified: every customer who follows the documented setup guide is affected.


---

### Lesson 5: The Reasonable Customer Test as Complexity Cutter

#### Principle

The reasonable customer test resolves cases where technical authorization analysis is complex but customer expectations are simple. Formulation:

> *"Would a the managed secret/certificate store administrator who was told a control-plane management role excludes data-plane access be alarmed to learn a side channel exists through a certificate-import path on a managed platform compute service?"*

When the reasonable customer statement can be formulated as a clear contradiction between documented role scope and actual achievable access, the case is almost certainly a design flaw regardless of the technical complexity of the authorization chain.

The reasonable customer test is not a substitute for technical analysis. It is a complexity cutter: when the technical analysis is ambiguous or the authorization chain is convoluted, the reasonable customer test provides a clear decision criterion grounded in the customer's legitimate security expectations.

#### Generalization

The reasonable customer test applies wherever a provider documents a security boundary and the observed behavior violates it:

- "Would an S3 bucket owner who configured CloudTrail for all management events be alarmed to learn some API calls bypass CloudTrail?" 
- "Would a GCP project admin who denied a user `storage.objects.get` be alarmed to learn the user can read objects through a managed document-processing service?" 
- "Would a Kubernetes admin who rotated a service account's token be alarmed to learn previously-issued tokens remain valid indefinitely?" 

---

### Lesson 6: The Six-Condition Deputy Graduation Framework

#### Principle

A confused deputy path graduates from signal to shape instance when **all six** conditions hold for a specific platform code path:

| # | Condition | Type | Role in framework |
|---|---|---|---|
| 0 | **Deputy is provider-owned** — the deputy is a PLMI / Google-managed service agent / AWS-owned service principal or service-linked role, not a customer-owned SAMI/UAMI/customer-managed service account/customer-created role | Precondition | Eliminates `FA-003` (customer-consented authority composition) before evaluating downstream conditions |
| 1 | **Deputy engaged** — the platform identity acts on the code path | Core | Confirms the deputy is on the path |
| 2 | **Deputy has reach** — the deputy has accumulated authorization on a candidate target resource | Core | Confirms the deputy can complete the action |
| 3 | **Caller-controlled target selection** — the caller supplies a parameter naming the target resource | Core (discriminator) | Distinguishes "deputy engages" (signal) from "deputy acts on caller-chosen target" (shape candidate) |
| 4 | **No delegation gate** — no mechanism binds the deputy's action to the caller's authorization on the named target | Core | Confirms the bridge is not caller-bound |
| 5 | **Exploit primitive demonstrable** — an attacker-invocable surface consumes the deputy's output against attacker-controlled inputs and produces a capability the attacker did not have at baseline | Post-condition | Eliminates byte-staging / state-staging paths with no operational impact (see Lesson 8) |

Conditions 0 and 5 are framework-wide and apply to every shape candidate; they are the cross-shape preconditions and post-conditions. Conditions 1–4 are the shape-specific core that varies in implementation detail per provider and per shape.

**Condition 3 remains the core discriminator** between a signal-level observation (the platform identity engages) and a shape-candidate (the platform identity acts on a caller-chosen target). **Condition 0 is the framework's first guard against false positives** — without it, any FA-003 candidate matches Conditions 1–4 and produces a spurious promotion. **Condition 5 is the framework's last guard against false positives** — without it, any byte-staging path that satisfies Conditions 0–4 produces a finding with no demonstrable attacker capability.

#### Generalization

The six-condition framework is provider-agnostic. Conditions 0, 1, 2, 3, 5 are structurally identical across providers; only the implementation of Condition 4 (delegation gate) varies:

| Provider | Condition 4 mechanism | Default |
|----------|----------------------|---------|
| **Azure** | Per-RP delegation-gate analog; ARM `LinkedAuthorizationFailed` for cross-subscription references | No class-wide default. Each RP must implement its own gate. |
| **GCP** | Customer-managed deputy → `iam.serviceAccounts.actAs`. Google-managed P4SA → backend platform-enforced caller-bound check. | `actAs` is **structurally unavailable** for P4SAs — the P4SA resource lives in a Google-owned tenant project that customers cannot grant IAM on — so the backend caller-bound check is the only architecturally available Condition 4 mechanism for the P4SA class. No class-wide default; observed per-API after remediation. |
| **AWS** | FAS/caller-bound forwarding, or, for the AWS shared-service-principal case, the service-internal final-target account/relationship bind; `aws:SourceArn` / `aws:SourceAccount` / `aws:SourceOrgID` are separate target-policy controls | FAS is caller-bound by construction; the shared-service-principal case requires the AWS service itself to reject an unrelated canonical target or verify an explicit relationship. Source keys are customer-authored and not enforced by default in all AWS-generated policies. |

The framework prevents three common failure modes: (1) **overflagging FA-003**: treating every customer-MI-with-customer-RBAC chain as a confused deputy because Conditions 1–4 surface-match (Condition 0 catches this); (2) **overflagging byte-staging**: treating every deputy that produces an intermediate effect as a vulnerability because the caller cannot directly produce that effect (Condition 5 catches this); (3) **underflagging genuine confused deputies**: dismissing the candidate because "authorization exists" (Condition 4 is the core discriminator, not "presence of any authorization").


---

### Lesson 7: Non-Exportable Does Not Mean Non-Exploitable

#### Principle

When a platform deputy delivers a secret to an attacker-controlled execution environment, marking the secret "non-exportable" reduces but does not eliminate the security impact. An attacker who controls the execution environment can use the secret for signing, decryption, authentication, and connection even without extracting the raw material.

The impact assessment must account for *use* of the secret, not just *extraction*. This is the operational test behind Principle 10 (Secret Custody Follows Secret Use).

#### Generalization

This principle applies to any delivery of a secret to an attacker-influenced environment:

- VM-level key or credential delivery where the attacker has SSH or agent access.
- Container-level secret injection where the attacker has exec access.
- Serverless function environment variables or mounted secrets where the attacker controls the function code.
- Hardware Security Module (HSM) keys that are marked non-exportable but accessible for cryptographic operations through the HSM's API from the compromised environment.
- Tokens, API keys, and connection strings staged for a workload where the attacker can run code — the workload's ability to present them is the attacker's ability to present them.

The general test: "Does the attacker need to *export* the secret to achieve the harmful outcome, or is *use* sufficient?" In most real-world attack scenarios — impersonation, authentication, signing, connecting — use is sufficient.


---

### Lesson 8: The Exploit-Primitive Requirement

#### Principle

A surface-pattern match — a control-plane operation that causes a platform deputy to perform a privileged action the caller cannot perform directly — is **not** a finding on its own. A finding requires a **demonstrable exploit primitive**: an attacker-invocable surface that consumes the deputy's output against attacker-controlled inputs in a way that produces a capability the attacker did not already have.

Without an exploit primitive, the path produces a deputy-side effect with no capability transfer to the attacker — interesting documentation signal, not a vulnerability. "It could become exploitable if some other component consumes the deputy's output" is a story, not a primitive. "The deputy successfully performed the privileged action" is intermediate state, not attacker capability.

#### Generalization

The exploit-primitive requirement is the discipline that separates design analysis from security findings, and it applies across providers and shapes. In every case where a deputy delivers an artifact — data, resource, configuration, credential, action — to a platform component the attacker does not directly read, the finding's evidence ceiling is determined by what attacker-invocable surface consumes the artifact and what capability that consumption transfers. When no such surface can be named and demonstrated against the artifact, the path is signal-level evidence, recorded for cross-RP analysis but not promoted to a shape instance.

This principle is operationalized as **Condition 5** of the six-condition graduation framework (Lesson 6). It applies to every shape that involves a deputy delivering an artifact, regardless of provider or output type.


---

### Lesson 9: Customer-Consented Authority Composition Is Not A Confused Deputy

#### Principle

When the bridging chain consists of a customer-owned deputy holding customer-granted target authority that is enumerable and revocable from the customer side, and caller authority that is itself customer-granted, the chain is **customer-consented authority composition** — the documented design — and is not a confused deputy regardless of any permission-class mismatches the bridge exposes.

The confused-deputy framing depends on the deputy's authority being something the attacker exploits *because the customer cannot see it, did not grant it, and cannot revoke it*. When visibility, consent, and revocability all hold in the customer's favor, the customer chose to permit the composition; the appropriate framing is "documented role composition" (`FA-003`), not "confused deputy."

#### Generalization

This lesson is the negative-side mirror of the linked-resource deputy test (Lesson 2) and the six-condition graduation framework (Lesson 6). It applies across providers wherever a deputy-bridge candidate is identified: when the customer is the grantor of every authority in the bridging chain, the chain is the documented design, and the shape applies only to chains the customer did not assemble. The shape that matched the surface pattern was the wrong shape for this candidate; the appropriate classification is `FA-003` (controlled feature behavior).

This principle is operationalized as **Condition 0** of the six-condition graduation framework (Lesson 6). It is the framework's first guard against false-positive promotions on customer-owned identity chains that surface-match Conditions 1–4.


---

### Lesson 10: Speculative Impact Chains Are Documentation Signal

#### Principle

A finding's impact must be demonstrable in a single end-to-end runtime test under preconditions the researcher controls and reproduces. Impact claims with the structure "if X is true, then if Y is true, then if Z is true, then attacker gains W" multiply unverified conditions, and each unverified link strictly dilutes the evidentiary value of the chain. A finding's claim ceiling is the floor of its propositions' evidence levels: a single `inferred/untested` link demotes the entire chain to documentation signal, not finding.

This lesson is the recursive application of the Constitution's guiding maxim — *hypothesize statically, validate by execution; anything else is documentation signal, not a finding* — to impact reasoning. The maxim governs not only the boundary observation but every proposition the impact narrative depends on.

#### Generalization

This lesson applies to every finding's impact section across providers and shapes. The general principle: every "if" in an impact narrative is a research deficit until it is closed by a runtime test. Compound conditionals in impact claims should be decomposed, each proposition labeled with its evidence level, and the claim downgraded — or the gap closed — before promotion. The discipline is provider-agnostic, deputy-agnostic, and shape-agnostic.

---

### Lesson 11: The Shared Cross-Tenant Deputy Is the Highest-Severity Confused-Deputy Class

#### Principle

A confused deputy's severity ceiling is set by **whether the deputy identity is per-tenant or shared across all tenants of the service**. The same missing-caller-authorization flaw produces a within-tenant escalation when the deputy is per-tenant (a GCP per-project P4SA, a customer-managed identity) and a **cross-tenant** escalation when the deputy is a single identity or fleet shared across every customer of the service (a GCP managed producer fleet, an AWS service principal, an Azure Platform-Level Managed Identity). In the shared case, a caller with **no rights to the target organization** drives the shared deputy into it — the Google Cloud VRP **S0** class ("cross-tenant impact with no rights to the target organization"), the most severe blast-radius/tenant-isolation outcome.

Count the shared-deputy property as an **active severity multiplier only when the path actually crosses the tenant boundary because the deputy is shared**. A shared platform identity that is structurally present but neutralized by a linked-authorization or relationship gate describes an unrealized ceiling, not a realized one.

The recurring non-control is a **provider-side authorization boundary that does not bind the shared deputy's final target to the initiating tenant or to an explicit, verified cross-tenant relationship**. The concrete boundary differs by provider and service: it may be a missing caller-authorization check, an owner-side network admission gate that cannot distinguish tenants, or an AWS service-internal same-account/associated-account restriction that is absent or bypassed. A customer resource policy that names a shared AWS service identity without a source condition can make a particular target exploitable, but that target-side posture is not the provider control the shared-service-principal case calibrates. A review must therefore ask: (1) is the deputy shared across tenants; (2) what provider-enforced invariant is supposed to restrict the target tenant; and (3) can a caller drive the deputy outside that invariant? Customer defenses are recorded separately from that calibration.

#### Triage: distinguish a caller-bound check from incidental P4SA permission-scoping

A subtle but load-bearing point: **"cross-tenant blocked" almost never means "the service checked the caller."** For a family of per-project service-agent-fronted managed services (data-migration, pipeline, build, data-integration, and database services), cross-tenant is blocked **incidentally** — the foreign project's per-project P4SA (`service-{N}@gcp-sa-*`) simply lacks the relevant permission on the foreign resource, and granting a *foreign* tenant's P4SA a permission is not a routine or documented operation (a P4SA is an internal Google identity unique to one project; no admin workflow grants another tenant's P4SA access). That denial is therefore robust, but it is a **service-agent permission-scoping** boundary, NOT a caller-bound check. The genuinely strong control — a caller-bound check on the *specific* referenced resource (e.g., `compute.networkAttachments.use` on the named attachment) — is implemented by **no GCP service** observed in this program. Do not mistake the robust incidental boundary for the strong control; and when assessing severity, recognize that the corollary **intra-project caller-bound gap is therefore universal** across the per-project-P4SA family (confirmed absent for one such service via a minimal-caller test; indicated-absent for the rest). That intra-project gap, however, currently has **no demonstrated meaningful escalation** — it is conditional, typically blind/write-only, and leans toward working-as-intended delegation (the parallel: the referenced resource exists precisely to enable the service's own action). Treat the intra-project gap as an authorization-MODEL observation, not a finding, until a concrete escalation (read-exfil, scope-mismatch privilege jump) is shown.

#### Generalization

This lesson unifies the provider-specific shared-deputy shapes — the AWS shared service principal, the Azure shared PLMI, and the GCP shared managed connector fleet — under one rule: the shared-deputy property is the multiplier that lifts a within-tenant confused deputy (`SHAPE-001`, S1-class) to a cross-tenant one (S0-class). It composes with the linked-resource deputy test (Lesson 2) and Condition 0 of the six-condition framework (Lesson 6, deputy-ownership): provider-owned **and shared-across-tenants** is the worst case; provider-owned-but-per-tenant is bounded; customer-owned is `FA-003`. The provider remediation pattern is to bind the deputy's action to the authorized target relationship: a caller-bound `compute.networkAttachments.use`/ownership check for GCP, a service-internal same-account or explicitly verified cross-account relationship check on the final canonical target for AWS, and a per-RP caller-bound check for Azure. AWS `aws:SourceArn`/`aws:SourceAccount`/`aws:SourceOrgID` conditions remain valuable target-side defense-in-depth, but they do not replace or calibrate the AWS service's internal tenant bind. Where the deputy must remain shared, a per-tenant producer identity can also let an owner scope admission to its own use.


---

## 4. Calibration Guidance

### Distinguishing Design Flaws from Feature Abuse

Not every powerful feature is a design flaw. The Constitution must teach a model to distinguish:

- **Design flaw**: The absence of expected controls represents a structural vulnerability. The feature works as designed, but the design is missing a security control that a reasonable customer would expect.
- **Feature abuse**: A legitimate feature is used maliciously, but authorization checks, logging, administrative controls, and guardrails exist and function. The risk is recognized and mitigated.
- **Classic bug**: An implementation error (injection, missing validation, deserialization) enables exploitation. The design is sound; the code is not.

### The Reasonable Customer Test as a Decision Boundary

When the technical authorization analysis is ambiguous — when multiple authorization checks exist but their composition is complex — the reasonable customer test (Lesson 5) provides the decision boundary:

1. Formulate the customer's documented expectation as a single statement.
2. Formulate the actual achievable access as a single statement.
3. If these two statements contradict each other, the case is a design flaw.

**Example**: "The customer was told a control-plane management role has no data-plane access. Through a certificate-import path on a managed platform compute service, a a control-plane management role can achieve effective data-plane access to private keys." These statements contradict. The case is a design flaw.

### Provider-Recommended Configuration as a Materiality Signal

When evaluating whether a finding is impactful enough to warrant a design-flaw classification, consider whether the vulnerable configuration is:

| Configuration source | Materiality weight |
|---------------------|-------------------|
| Provider documentation instructs this configuration | **Highest** — every customer following the docs is affected (Lesson 4) |
| Provider-generated template or an infrastructure-as-code service/Terraform module creates this configuration | **High** — customers using standard tooling are affected |
| Configuration is common in practice but not provider-recommended | **Medium** — affected population depends on adoption |
| Configuration is unusual, non-standard, or explicitly warned against | **Lower** — but still a design flaw if the controls are missing |

Provider-recommended configurations that create vulnerability preconditions are a materiality amplifier. Do not treat "the customer configured it this way" as a mitigating factor when "this way" is what the vendor told them to do.

### Negative Calibration Signals

The following patterns are negative signals that reduce the likelihood of a design flaw. They do not definitively rule out a finding, but they should increase the evidentiary bar:

- **Explicit delegation control**: The platform requires `iam.serviceAccounts.actAs`, `iam:PassRole`, `assign/action`, or an equivalent caller-bound delegation check before the platform identity acts on a caller-referenced resource. (FA-003 pattern.)
- **Derived authority**: The platform identity's authority on the target is derived from the caller's own grant, not from an independent assignment. The platform uses the caller's managed identity, not a globally shared platform identity.
- **Enumerable and revocable**: The platform identity's grants are fully enumerable through standard admin APIs, individually revocable, and visible in the customer's IAM inventory.
- **Scope-limited by design**: The platform identity can only operate on resources within the caller's own subscription, project, or account — it cannot reach resources in other tenants or accounts.

---

## 5. Hypothesize Statically, Validate by Execution

The guiding maxim: **hypothesize statically, validate by execution; anything else is documentation signal, not a finding.**

This governs the boundary between what a model can conclude from documentation review and what requires live testing. Reasoning from the Constitution can surface candidates and render preliminary judgments — valuable for triage, but documentation-signal-class evidence. Validation by execution, against infrastructure you are authorized to test, is what confirms or refutes each candidate. Never conflate a documentation-derived hypothesis with a validated finding: keep the evidence label explicit, and record which claims are validated and which remain assumptions.
