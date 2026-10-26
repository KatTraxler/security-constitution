# Feature Abuse Reference Catalog

## True Negatives for Security Constitution Calibration

> **Purpose**: This catalog lists legitimate but abuse-prone cloud features that retain meaningful controls and should not be classified as insecure-by-design. It is the counter-corpus to the Design Shape Register.

> **What this register is for.** The constitution hunts insecure-by-design flaws, not feature abuse. Each entry here is a flow that looks like one of the shapes — a caller makes a platform identity act with that identity's permissions — but is **not** a design flaw because a specific permission or mitigation guards it. With that guard in place, abusing the flow is feature abuse: misuse of a granted capability, not a reportable bug. Remove the guard, and the same flow is a design flaw the shapes describe. The register exists so the model learns that boundary and stops flagging guarded flows.

---

## How This File Relates to the Others

- `feature-abuse-reference.md` stores negative calibration cases: powerful features that can be abused, but are not themselves design flaws.
- `design-shape-register.md` stores reusable shapes and the negative signals that distinguish those shapes from legitimate controlled features.

---

## Feature Abuse vs. Design Flaw

| Category | Feature Abuse | Insecure-by-Design |
| ------------- | ----------------------------------------------------------------- | ------------------------------------------------------- |
| Feature | Works as designed with appropriate or generally accepted controls | Missing expected controls or violates security norms |
| Flaw | Actor intent, over-permissioning, or operator choice | The design itself, regardless of actor intent |
| Authorization | Present and enforced | Absent, partial, or bypassable at the relevant boundary |
| Logging | Actions are auditable | Actions are invisible or materially incomplete |
| Mitigations | Guardrails, least privilege, and detection exist | None, severely limited, or only added after remediation |

### Grey-Zone Calibration: Managed Identity and Service-Role Runtime Authority

Managed identities, service roles, code-execution surfaces, and automation workflows remain feature abuse when the customer deliberately attaches or overgrants a powerful identity and the relevant attach, impersonation, assignment, or runtime action is explicitly authorized, visible, and governable. Escalate to design-flaw candidacy only when the platform collapses two authorities customers reasonably expect to be distinct: permission to operate the service, and permission to cause an attached identity to act. The signal is a non-obvious authority expansion at the runtime handoff — especially where no separate attachment gate (`actAs`, `PassRole`, assignment, or equivalent) exists, downstream authority is not visible at grant time, initiator attribution is lost, or least privilege requires graph reasoning rather than a local IAM binding.

---

## Entry Schema

| Field | Meaning |
| ---------------------------------------------- | ---------------------------------------------------- |
| **ID** | Stable feature-abuse identifier |
| **Name** | Short descriptive title |
| **Provider** | Cloud or SaaS provider |
| **Service** | Relevant service or feature |
| **The guard (why it's feature abuse, not a flaw)** | The specific permission or mitigation that makes this a negative example |
| **Summary** | What attackers abuse |
| **Privilege Escalation Path** | Optional concise chain showing how authority expands |
| **Observed Controls** | Control scoring for the feature |
| **Detection Available** | How defenders can see the behavior |
| **Mitigations Available** | Preventive or detective controls |
| **Confusion Risk** | Why a model or reviewer might incorrectly flag it |
| **Key Differentiator from Design Flaws** | The reason it belongs in this catalog |

---

## Entries

### FA-003: Azure Managed Identity Runtime Authority Through Services

| Field | Detail |
| ---------------------------------------------- | ------ |
| **ID** | FA-003 |
| **Name** | Azure Managed Identity Runtime Authority Through Services |
| **Provider** | Azure |
| **Service** | Azure compute/automation services with attached managed identities |
| **The guard (why it's feature abuse, not a flaw)** | Attaching a user-assigned managed identity is gated by the `Microsoft.ManagedIdentity/userAssignedIdentities/*/assign/action` permission, and the identity's downstream RBAC is customer-granted and customer-scoped. With that attachment gate and customer-scoped downstream grants in place, driving a service to act as its attached identity is misuse of a granted capability, not a missing control. Service operation is governed by Azure RBAC, and edits, runs, identity sign-ins, and downstream actions all have logging paths (even when correlation is required). |
| **Summary** | Azure services can run automation, workflows, or code under attached managed identities. A user with sufficient service-specific edit, run, deploy, or job authority may cause the service to use whatever downstream permissions the managed identity has. |
| **Privilege Escalation Path** | 1. A managed identity is attached to a compute/automation service. 2. The customer grants the managed identity downstream access (vault, resource-manager, storage, or application permissions). 3. The actor gains service-specific edit/run/deploy/job authority. 4. The actor modifies or runs the service so it requests tokens or acts as the managed identity. 5. The actor reaches resources allowed by the managed identity's customer-granted permissions. |
| **Observed Controls** | **Authorization**: Present/Partial. **Logging**: Partial. **Administrative Control**: Present. **Credential Lifecycle**: Present. **Blast Radius**: Partial. **Tenant Isolation**: Present. |
| **Detection Available** | Activity logs, automation/job logs, workflow run history, deployment and configuration events, managed-identity sign-in logs, downstream resource logs, and graph/path review can expose the chain. |
| **Mitigations Available** | Scope managed-identity permissions, avoid attaching high-privilege identities to user-editable execution surfaces, restrict the service-operation roles, use the attachment gate for user-assigned identity attachment, monitor runbook/workflow/app changes, and use JIT/PIM for sensitive operations. |
| **Confusion Risk** | May resemble a missing-control flaw because an attachment-time gate does not re-authorize every later runtime action. The difference is that the reviewed Azure examples show no automatically assigned overly privileged managed policy or provider-created broad default (contrast a provider-created overprivileged default execution role, which would be a design flaw). |
| **Key Differentiator from Design Flaws** | The customer controls the managed identity's downstream permissions and the roles that can operate the execution surface. Absent an automatic broad grant, missing administrative control, or a demonstrated severe provider-created default, this is feature abuse / misconfiguration, not insecure-by-design. |

### FA-006: GCP Service Account Impersonation Lateral Movement

| Field | Detail |
| ---------------------------------------------- | ------ |
| **ID** | FA-006 |
| **Name** | GCP Service Account Impersonation Lateral Movement |
| **Provider** | GCP |
| **Service** | Cloud IAM / Service Accounts |
| **The guard (why it's feature abuse, not a flaw)** | Impersonation is gated by the token-issuance permission (`iam.serviceAccounts.getAccessToken` / the Token Creator role), which is admin-granted, scoped per service account, and audited; the credentials it issues are short-lived. With that guard in place, impersonating a service account you have been granted Token Creator on is use of a granted capability, not a missing control. |
| **Summary** | Users with token-creator permission can impersonate service accounts and generate short-lived tokens. |
| **Privilege Escalation Path** | 1. A principal holds impersonation permission (token creation) on a target service account. 2. The principal requests a short-lived access token. 3. GCP authorizes and logs the request. 4. The principal uses the temporary credential as the service account within its granted scope. |
| **Observed Controls** | **Authorization**: Present. **Logging**: Present. **Administrative Control**: Present. **Credential Lifecycle**: Present. **Blast Radius**: Partial. **Tenant Isolation**: N/A. |
| **Detection Available** | Cloud Audit Logs for token generation; posture tooling for excessive impersonation. |
| **Mitigations Available** | Grant impersonation at individual service-account scope, audit Token Creator grants, enable Data Access logs, and use IAM Conditions. |
| **Confusion Risk** | May be confused with insecure impersonation because both involve acting as another identity. |
| **Key Differentiator from Design Flaws** | Every impersonation request is authenticated, authorized, logged, and short-lived. |

### FA-010: OCI Dynamic Groups and Instance Principal Privilege Inheritance

| Field | Detail |
| ---------------------------------------------- | ------ |
| **ID** | FA-010 |
| **Name** | OCI Dynamic Groups and Instance Principal Privilege Inheritance |
| **Provider** | Oracle Cloud Infrastructure |
| **Service** | OCI IAM, Dynamic Groups, Identity Domains, Compute instances |
| **The guard (why it's feature abuse, not a flaw)** | Every link is customer-authored and customer-gated: the customer authors the dynamic-group matching rule, authors the IAM policy granted to the group, and governs which users may create or modify instances in the relevant compartment. With those three gates in place, causing an instance to match a privileged rule is misuse of customer-authored configuration, not a missing platform control. |
| **Summary** | OCI Dynamic Groups grant permissions to compute instances based on customer-authored matching rules. A user who can create, modify, or access an instance that matches a privileged dynamic group's rule effectively acts with that group's instance-principal permissions, following the documented mechanics of Dynamic Groups and natural-language policies. |
| **Privilege Escalation Path** | 1. An administrator authors a dynamic group with a matching rule and grants the group a policy. 2. A user has compartment-level authority to create or modify compute instances. 3. The user causes an instance to match the rule. 4. The instance principal acts with the granted permissions; the user gains the same authority by operating the instance. |
| **Observed Controls** | **Authorization**: Present. **Logging**: Partial. **Administrative Control**: Present. **Credential Lifecycle**: Present (short-lived instance-principal credentials). **Blast Radius**: Partial. **Tenant Isolation**: Present. |
| **Detection Available** | Audit logs cover instance creation/modification and IAM policy changes; dynamic-group membership is enumerable; posture tooling can flag risky compartment configurations. |
| **Mitigations Available** | Tighten matching rules (compartment- or tag-scoped); pair group permissions with least-privilege policies; restrict compute create/modify authority in compartments containing privileged rules; review memberships and policies; build graph-based entitlement review. |
| **Confusion Risk** | May resemble a confused-deputy flaw because compute ends up acting with a delegated identity, but every link is customer-authored and customer-governable. The required graph reasoning is a property of rule-based access, not a missing platform control. |
| **Key Differentiator from Design Flaws** | All meaningful authorization decisions — who can create instances, which instances match, and which permissions the group receives — are customer-authored. Do not flag rule-based instance-principal inheritance as a design flaw unless a specific case shows a provider-created default, missing visibility control, or unavoidable guardrail gap beyond customer misconfiguration. |


