# CRD Schema Drafts (v0.1)

This document defines the initial Kubernetes CRDs for the Secure-MAS control plane. These are **draft** schemas meant to be used with `kubebuilder` in a future implementation, but they include the core fields, status/conditions, and example YAMLs to keep the system consistent across controllers and services.

## Conventions

- `metadata.name` should be stable and unique per Task/Order.
- All resources include `spec` and `status` fields.
- `status.conditions` uses the standard Kubernetes `type/status/reason/message/lastTransitionTime` shape.
- Timestamps are RFC3339.
- IDs are UUIDv4 unless otherwise noted.

## Repository layout

- `crds/definitions/`: installable CRD definitions (apiextensions.k8s.io/v1).
- `crds/examples/`: example custom resources for quick testing.

---

## 1) Task CRD

### Purpose
Represents the lifecycle of a single orchestrated task. The Task transitions through a controlled state machine and triggers graph execution.

### Group/Version/Kind

- `securemas.io/v1alpha1`
- Kind: `Task`

### Spec (draft)

```yaml
spec:
  taskId: string
  tenantId: string
  graphRef: string            # reference to langgraph definition
  input:
    params:                   # arbitrary input params (string->string)
      key: value
  policyBundleRef: string     # versioned policy bundle
  securityProfileRef: string  # reference to TaskSecurityProfile
  checkpoint:
    uri: string               # checkpoint storage location
    mode: "readwrite"|"readonly"
  scheduling:
    preferredAgents: ["agent-f", "agent-g"]
    allowFallback: true
  timeouts:
    executionSeconds: 900
  annotations:
    riskLevel: "low"|"medium"|"high"
    dataClassification: "public"|"internal"|"restricted"|"secret"
```

### Status (draft)

```yaml
status:
  phase: "INIT"|"COMPLIANCE_REVIEW"|"READY"|"RUNNING"|"DEGRADED"|"SUSPENDED"|"COMPLETED"|"FAILED"
  currentAgentId: string
  lastCheckpoint:
    uri: string
    time: string
  attempts: 1
  conditions:
    - type: "Ready"
      status: "True"|"False"|"Unknown"
      reason: string
      message: string
      lastTransitionTime: string
  audit:
    logRef: string
```

### State machine (initial)

```
INIT -> COMPLIANCE_REVIEW -> READY -> RUNNING -> COMPLETED
RUNNING -> DEGRADED -> RUNNING (after remediation) | SUSPENDED | FAILED
```

### Example YAML

```yaml
apiVersion: securemas.io/v1alpha1
kind: Task
metadata:
  name: demo-task-001
spec:
  taskId: "d9f9a9b4-7b6a-4d64-9b62-4d7f30b5d9f3"
  tenantId: "tenant-001"
  graphRef: "graphs/demo-graph:v1"
  input:
    params:
      query: "find anomalies"
  policyBundleRef: "policy-bundle:v1"
  securityProfileRef: "tsp-basic"
  checkpoint:
    uri: "s3://securemas/checkpoints/demo-task-001"
    mode: "readwrite"
  scheduling:
    preferredAgents: ["agent-f", "agent-g"]
    allowFallback: true
  timeouts:
    executionSeconds: 900
  annotations:
    riskLevel: "medium"
    dataClassification: "restricted"
```

---

## 2) TaskSecurityProfile (TSP) CRD

### Purpose
Defines a security profile applied to a Task: allowlists, tool scopes, and runtime constraints.

### Group/Version/Kind

- `securemas.io/v1alpha1`
- Kind: `TaskSecurityProfile`

### Spec (draft)

```yaml
spec:
  profileId: string
  datasets:
    allow: ["dataset-001", "dataset-002"]
  tools:
    allow: ["tool:search", "tool:analyze"]
  egress:
    allowHosts: ["api.internal.local", "storage.internal.local"]
  output:
    redactPatterns: ["\b\d{16}\b"]
    maxTokens: 2048
  isolation:
    mode: "standard"|"tee"
    attestationRequired: true|false
```

### Status (draft)

```yaml
status:
  conditions:
    - type: "Validated"
      status: "True"|"False"|"Unknown"
      reason: string
      message: string
      lastTransitionTime: string
```

### Example YAML

```yaml
apiVersion: securemas.io/v1alpha1
kind: TaskSecurityProfile
metadata:
  name: tsp-basic
spec:
  profileId: "tsp-basic"
  datasets:
    allow: ["dataset-001"]
  tools:
    allow: ["tool:search", "tool:analyze"]
  egress:
    allowHosts: ["api.internal.local"]
  output:
    redactPatterns: ["\\b\\d{16}\\b"]
    maxTokens: 2048
  isolation:
    mode: "standard"
    attestationRequired: false
```

---

## 3) SecurityEvent CRD

### Purpose
Represents a security anomaly detected by MIB or other telemetry processors.

### Group/Version/Kind

- `securemas.io/v1alpha1`
- Kind: `SecurityEvent`

### Spec (draft)

```yaml
spec:
  eventId: string
  taskId: string
  agentId: string
  type: "compute"|"network"|"policy"|"behavior"
  severity: "low"|"medium"|"high"
  confidence: 0.0
  summary: string
  evidenceRefs: ["s3://.../evidence-bundle.json"]
  recommendedActions: ["RESTRICT_EGRESS", "SUSPEND_AGENT"]
  detectedAt: string
```

### Status (draft)

```yaml
status:
  phase: "NEW"|"INVESTIGATING"|"INVESTIGATED"|"ADJUDICATED"|"EXECUTED"|"DISMISSED"
  adjudicationOrderRef: string
  executionReceiptRef: string
  conditions:
    - type: "ReadyForAdjudication"
      status: "True"|"False"|"Unknown"
      reason: string
      message: string
      lastTransitionTime: string
```

### Example YAML

```yaml
apiVersion: securemas.io/v1alpha1
kind: SecurityEvent
metadata:
  name: securityevent-001
spec:
  eventId: "0c6b2c6a-0b2e-4d9a-a38b-4a6f4a7bcb5f"
  taskId: "d9f9a9b4-7b6a-4d64-9b62-4d7f30b5d9f3"
  agentId: "agent-f"
  type: "network"
  severity: "high"
  confidence: 0.85
  summary: "Unauthorized egress attempt to external host"
  evidenceRefs: ["s3://securemas/evidence/securityevent-001.json"]
  recommendedActions: ["RESTRICT_EGRESS", "REVOKE_TOKEN"]
  detectedAt: "2025-01-01T00:00:00Z"
```

---

## 4) AdjudicationOrder CRD

### Purpose
Represents a structured decision from AJB that EB can execute.

### Group/Version/Kind

- `securemas.io/v1alpha1`
- Kind: `AdjudicationOrder`

### Spec (draft)

```yaml
spec:
  orderId: string
  eventId: string
  taskId: string
  agentId: string
  actions:
    - type: "RESTRICT_EGRESS"|"SUSPEND_AGENT"|"REVOKE_TOKEN"|"QUARANTINE_POD"
      params:
        key: value
  reason: string
  issuedAt: string
  signature:
    algorithm: "hmac-sha256"|"ed25519"
    value: string
```

### Status (draft)

```yaml
status:
  phase: "PENDING"|"VERIFIED"|"EXECUTING"|"EXECUTED"|"REJECTED"
  executionReceiptRef: string
  conditions:
    - type: "SignatureVerified"
      status: "True"|"False"|"Unknown"
      reason: string
      message: string
      lastTransitionTime: string
```

### Example YAML

```yaml
apiVersion: securemas.io/v1alpha1
kind: AdjudicationOrder
metadata:
  name: adjudicationorder-001
spec:
  orderId: "c3e90d08-48c1-4d2b-9f3c-8f1f95a0b6b0"
  eventId: "0c6b2c6a-0b2e-4d9a-a38b-4a6f4a7bcb5f"
  taskId: "d9f9a9b4-7b6a-4d64-9b62-4d7f30b5d9f3"
  agentId: "agent-f"
  actions:
    - type: "RESTRICT_EGRESS"
      params:
        allowHosts: ["api.internal.local"]
    - type: "REVOKE_TOKEN"
      params:
        jti: "token-jti-001"
  reason: "Policy violation confirmed"
  issuedAt: "2025-01-01T00:01:00Z"
  signature:
    algorithm: "hmac-sha256"
    value: "<base64-signature>"
```

---

## 5) ExecutionReceipt CRD (optional in v0.1)

### Purpose
Records the execution result of an AdjudicationOrder.

### Group/Version/Kind

- `securemas.io/v1alpha1`
- Kind: `ExecutionReceipt`

### Spec (draft)

```yaml
spec:
  receiptId: string
  orderId: string
  eventId: string
  taskId: string
  agentId: string
  executedAt: string
  results:
    - action: "RESTRICT_EGRESS"
      status: "SUCCESS"|"FAIL"
      message: string
  auditLogRef: string
```

### Status (draft)

```yaml
status:
  conditions:
    - type: "Completed"
      status: "True"|"False"|"Unknown"
      reason: string
      message: string
      lastTransitionTime: string
```

### Example YAML

```yaml
apiVersion: securemas.io/v1alpha1
kind: ExecutionReceipt
metadata:
  name: executionreceipt-001
spec:
  receiptId: "28c3f41b-8a3f-4c9d-9e4f-9a8a4a2e4d50"
  orderId: "c3e90d08-48c1-4d2b-9f3c-8f1f95a0b6b0"
  eventId: "0c6b2c6a-0b2e-4d9a-a38b-4a6f4a7bcb5f"
  taskId: "d9f9a9b4-7b6a-4d64-9b62-4d7f30b5d9f3"
  agentId: "agent-f"
  executedAt: "2025-01-01T00:01:30Z"
  results:
    - action: "RESTRICT_EGRESS"
      status: "SUCCESS"
      message: "egress restricted to allowlist"
  auditLogRef: "s3://securemas/audit/receipt-001.log"
```
```
