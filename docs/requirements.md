# Spacedock Operator: Project Requirements

**Version:** 1.0 (v1alpha1 API)
**Audience:** Implementing coding agent and platform reviewers
**Status:** Ready for implementation

---

## 1. Overview

Spacedock is a Kubernetes operator that gives platform teams a two-level, self-service abstraction:

| Kind | Scope | Purpose |
|------|-------|---------|
| `Space` | Cluster | A governed tenant environment. It creates and manages a namespace with quotas, default limits, network isolation, pod security, access bindings and tenant policy. |
| `App` | Namespaced (inside a Space) | A deployable stateless workload. It creates and manages the Deployment, Service, Ingress, ConfigMap, Secret wiring, HPA, PDB and ServiceAccount. |

**API group:** `spacedock.io`
**Version:** `v1alpha1`
**Short names:** `space` / `spc` and `sdapp`

> Naming note: the kind is `App` rather than `Application` so it doesn't collide with Argo CD's `applications.argoproj.io` when someone runs `kubectl get application`. The short name `sdapp` avoids collisions with other `app` CRDs.

### 1.1 Goals

- A platform admin creates a `Space` and gets a fully governed namespace in one object.
- A developer creates an `App` with image, replicas, service, ingress, config and secrets, and doesn't have to write raw Deployment, Service or Ingress YAML.
- The operator enforces Space policy on Apps: allowed registries, allowed ingress domains, and replica and app-count limits.
- Drift correction. Operator-managed resources are reconciled back to the desired state.
- Clear status, conditions and events on every CR.

### 1.2 Non-goals (v1)

- Stateful workloads (StatefulSets, PVC templates). A later phase can add this.
- CI/CD, image building, or GitOps sync. These CRs are designed to be applied *by* GitOps tools.
- Multi-cluster placement.
- Storing plaintext secret values in CRs. This is never allowed.
- Gateway API HTTPRoute. This goes on the roadmap, and the API should leave room for it.

---

## 2. Technology Stack

| Item | Requirement |
|------|-------------|
| Language | Go (latest stable, ≥ 1.24) |
| Framework | Kubebuilder v4 + controller-runtime (latest stable) |
| Target Kubernetes | 1.30+. Tested on the latest 3 minor versions, plus EKS. |
| Validation | CRD OpenAPI schema + CEL (`x-kubernetes-validations`) + admission webhooks |
| Webhook certs | cert-manager |
| Apply strategy | Server-Side Apply with field manager `spacedock-operator` |
| Tests | Go unit tests, envtest (Ginkgo/Gomega), and e2e on kind |
| Packaging | Container image (distroless, non-root, multi-arch amd64/arm64), Helm chart, Kustomize manifests |
| Lint | golangci-lint |

---

## 3. Space API

### 3.1 Example

```yaml
apiVersion: spacedock.io/v1alpha1
kind: Space
metadata:
  name: team-payments
spec:
  displayName: "Payments Team"
  namespace: team-payments          # optional; defaults to metadata.name; immutable
  namespaceMetadata:
    labels:
      cost-center: "cc-1042"
    annotations: {}

  access:
    - kind: Group                   # User | Group | ServiceAccount
      name: payments-admins
      role: admin                   # admin | edit | view
    - kind: Group
      name: payments-devs
      role: edit
    - kind: ServiceAccount
      name: gitlab-deployer
      namespace: ci                 # required when kind=ServiceAccount
      role: edit

  quota:
    hard:                           # passthrough to ResourceQuota.spec.hard
      requests.cpu: "8"
      requests.memory: 16Gi
      limits.cpu: "16"
      limits.memory: 32Gi
      pods: "50"
      services: "20"
      services.loadbalancers: "0"
      services.nodeports: "0"
      persistentvolumeclaims: "10"
      requests.storage: 100Gi
      secrets: "50"
      configmaps: "50"

  limits:                           # generates a LimitRange of type Container
    defaultRequest: { cpu: 100m, memory: 128Mi }
    default:        { cpu: 500m, memory: 512Mi }
    min:            { cpu: 10m,  memory: 16Mi }
    max:            { cpu: "4",  memory: 8Gi }

  podSecurity: restricted           # privileged | baseline | restricted (default: restricted)

  network:
    isolation: Namespace            # None | Namespace | Strict (default: Namespace)
    allowIngressFromNamespaces:     # namespaces allowed in (for example the ingress controller, monitoring)
      - kube-system
      - ingress-nginx
      - monitoring
    allowEgress:
      dns: true                     # allow kube-dns (default true)
      internet: true                # allow 0.0.0.0/0 except cluster CIDRs (default true)
      cidrs: []                     # additional allowed egress CIDRs

  policy:
    allowedRegistries:              # image prefix allowlist; empty = allow all
      - 123456789012.dkr.ecr.ap-south-1.amazonaws.com/
    allowedIngressDomains:          # host allowlist with wildcard support; empty = allow all
      - "*.payments.example.io"
    allowedIngressClasses: [alb, nginx]   # empty = allow all
    allowedServiceTypes: [ClusterIP]      # default: [ClusterIP]
    maxApps: 20
    maxReplicasPerApp: 10

  deletionPolicy: Delete            # Delete | Retain (default Delete)
```

### 3.2 Field rules

| Field | Rule |
|-------|------|
| `spec.namespace` | DNS-1123 label, ≤ 63 chars, immutable (CEL `self == oldSelf`). It must not be a reserved namespace: `default`, `kube-*`, `spacedock-system`. |
| `spec.access[].role` | Maps to ClusterRoles `admin`, `edit` and `view`. The Helm chart can override the ClusterRole names. |
| `spec.access[]` | `namespace` is required only when `kind=ServiceAccount` (CEL). |
| `spec.quota.hard` | Type `corev1.ResourceList`. Optional. If it's omitted, the operator applies the cluster default quota from operator config (if one is configured). |
| `spec.limits` | Optional. CEL: for each resource, `min ≤ defaultRequest ≤ default ≤ max`. |
| `spec.policy.maxReplicasPerApp` | ≥ 1. |
| `spec.deletionPolicy` | `Retain` orphans the namespace and everything in it. |

### 3.3 Managed resources

The Space controller creates and owns these, labeled `app.kubernetes.io/managed-by=spacedock` and `spacedock.io/space=<name>`:

1. **Namespace** with:
   - labels `spacedock.io/space=<name>`
   - `pod-security.kubernetes.io/enforce=<level>`, plus `audit` and `warn` at the same level
   - the user labels and annotations from `namespaceMetadata`
2. **ResourceQuota** `spacedock-quota`.
3. **LimitRange** `spacedock-limits`.
4. **NetworkPolicies**:
   - `None`: no policies.
   - `Namespace`: default deny ingress. Ingress is allowed from the same namespace and from `allowIngressFromNamespaces`. Egress follows `allowEgress`.
   - `Strict`: default deny ingress and egress. Only DNS plus explicitly listed CIDRs and namespaces are allowed.
5. **RoleBindings**, one per `access` entry, named `spacedock-<role>-<hash>`.

### 3.4 Status

```yaml
status:
  observedGeneration: 3
  namespace: team-payments
  phase: Ready                     # Pending | Ready | Degraded | Terminating
  appCount: 4
  quotaUsage:                      # mirrored from ResourceQuota.status
    hard: { requests.cpu: "8", ... }
    used: { requests.cpu: "2500m", ... }
  conditions:
    - type: Ready
    - type: NamespaceReady
    - type: QuotaReady
    - type: NetworkPolicyReady
    - type: AccessReady
    - type: QuotaExceeded          # True when used > hard (for example, after a quota was reduced)
```

### 3.5 Behavior

- **Existing namespace.** If the target namespace exists and isn't labeled for this Space, set `NamespaceReady=False` with reason `NamespaceConflict` and don't adopt it. The annotation `spacedock.io/adopt: "true"` on the Space permits adoption.
- **Drift.** Manual edits to managed objects (quota, limits, policies, bindings) are reverted on the next reconcile. The controller watches all owned kinds.
- **Removed entries.** When an `access` entry or a network rule is removed, the operator deletes the corresponding object. It prunes by label selector.
- **Quota reduced below usage.** Kubernetes allows this. The operator surfaces it with the `QuotaExceeded` condition and a Warning event, and does not evict anything.
- **Deletion.** The finalizer is `spacedock.io/space-cleanup`.
  - `Delete`: delete the namespace, wait for it to be gone, then remove the finalizer.
  - `Retain`: remove owner refs and managed-by labels from the namespace, keep its contents, then remove the finalizer.
- **Policy changes.** When `spec.policy` changes, enqueue all Apps in the Space so they are re-evaluated.

---

## 4. App API

### 4.1 Example

```yaml
apiVersion: spacedock.io/v1alpha1
kind: App
metadata:
  name: checkout-api
  namespace: team-payments          # must be a Space-managed namespace
spec:
  suspend: false                    # true scales to 0 and keeps the config

  image:
    repository: 123456789012.dkr.ecr.ap-south-1.amazonaws.com/checkout-api
    tag: "1.4.2"                    # tag or digest required (exactly one)
    # digest: sha256:...
    pullPolicy: IfNotPresent
    pullSecrets: [ecr-pull]

  command: []
  args: []
  workingDir: ""

  replicas: 3                       # ignored when autoscaling.enabled=true

  ports:
    - name: http
      containerPort: 8080
      protocol: TCP

  service:
    enabled: true                   # default true if ports are defined
    type: ClusterIP
    annotations: {}
    ports:
      - name: http
        port: 80
        targetPort: http            # port name or number

  ingress:
    enabled: true
    className: alb
    annotations:
      alb.ingress.kubernetes.io/scheme: internal
    rules:
      - host: checkout.payments.example.io
        paths:
          - path: /
            pathType: Prefix
            servicePort: http       # a service port name
    tls:
      - hosts: [checkout.payments.example.io]
        secretName: checkout-tls    # optional (for example, with ALB + ACM)

  env:                              # corev1.EnvVar passthrough
    - name: APP_ENV
      value: production

  config:                           # operator-generated ConfigMap
    data:
      LOG_LEVEL: info
      FEATURE_X: "true"
    mountPath: ""                   # empty = envFrom; set = mount as files

  secrets:                          # references to EXISTING Secrets only
    - name: checkout-db
      mode: env                     # env | file
    - name: checkout-certs
      mode: file
      mountPath: /etc/checkout/certs
      items: []                     # optional key→path projection

  resources:
    requests: { cpu: 250m, memory: 256Mi }
    limits:   { cpu: "1",  memory: 512Mi }

  probes:
    liveness:  { httpGet: { path: /healthz, port: http }, periodSeconds: 10 }
    readiness: { httpGet: { path: /ready,   port: http }, periodSeconds: 5 }
    startup:   null

  autoscaling:
    enabled: true
    minReplicas: 2
    maxReplicas: 6
    targetCPUUtilizationPercentage: 70
    targetMemoryUtilizationPercentage: null

  strategy:
    maxSurge: 25%
    maxUnavailable: 0

  disruptionBudget:
    enabled: true                   # defaults true when effective replicas ≥ 2
    minAvailable: 1                 # or maxUnavailable (exactly one)

  serviceAccount:
    create: true
    name: ""                        # defaults to the App name
    annotations:
      eks.amazonaws.com/role-arn: arn:aws:iam::123456789012:role/checkout-api

  podMetadata:
    labels: {}
    annotations: {}

  securityContext: {}               # container securityContext overrides (within the Space's PSS level)
  podSecurityContext: {}
  nodeSelector: {}
  tolerations: []
  affinity: {}
  topologySpread: true              # default: spread across zones and hosts
  terminationGracePeriodSeconds: 30
```

### 4.2 Field rules (CEL and webhook)

| Rule | Mechanism |
|------|-----------|
| Exactly one of `image.tag` / `image.digest` | CEL |
| `replicas` 0–1000. It's ignored (with a warning) when autoscaling is enabled. | CEL + webhook warning |
| `autoscaling.minReplicas ≤ maxReplicas`, and at least one target metric set | CEL |
| `ports[].name` unique; `service.ports[].targetPort` must reference a defined port name or number | CEL |
| `ingress.enabled` requires `service.enabled` | CEL |
| `ingress.rules[].paths[].servicePort` must match a service port name | CEL |
| `disruptionBudget`: exactly one of `minAvailable` / `maxUnavailable` | CEL |
| `secrets[].mountPath` required when `mode=file` | CEL |
| `config.mountPath` and `secrets[].mountPath` must be absolute and must not collide | CEL |
| App namespace must be a Space namespace | Webhook (look up namespace label) |
| Image matches `space.policy.allowedRegistries` | Webhook + controller |
| Ingress hosts match `allowedIngressDomains`; class in `allowedIngressClasses` | Webhook + controller |
| Service type in `allowedServiceTypes` | Webhook + controller |
| `replicas` / `autoscaling.maxReplicas` ≤ `maxReplicasPerApp` | Webhook + controller |
| App count in the namespace < `maxApps` (on create) | Webhook |
| Ingress host not already claimed by another App in the cluster | Webhook (field index on host) |

**Defense in depth.** Every policy rule the webhook enforces is also checked by the controller. On a violation, the controller sets `PolicyCompliant=False`, stops applying changes, and **leaves the last good workload running**.

### 4.3 Defaulting (mutating webhook)

- `image.pullPolicy`: `IfNotPresent`, or `Always` if the tag is `latest`.
- `service.enabled`: true if `ports` is non-empty.
- `service.ports`: if omitted, mirror `ports` (port = containerPort).
- `strategy`: `maxSurge: 25%`, `maxUnavailable: 0`.
- `disruptionBudget.enabled`: true when effective min replicas ≥ 2.
- `securityContext` (restricted-compliant): `runAsNonRoot: true`, `allowPrivilegeEscalation: false`, `readOnlyRootFilesystem: true`, `capabilities.drop: [ALL]`, `seccompProfile.type: RuntimeDefault`. A user can relax only as far as the Space's PSS level allows. The API server rejects anything beyond that.
- `resources`: if omitted, leave empty and let the Space LimitRange apply the defaults.

### 4.4 Managed resources

All of these are named `<app-name>` unless noted. Each has an ownerReference to the App (controller=true), plus standard labels:

```
app.kubernetes.io/name: <app>
app.kubernetes.io/instance: <app>
app.kubernetes.io/managed-by: spacedock
app.kubernetes.io/version: <tag or short digest>
spacedock.io/space: <space>
```

| Resource | When created |
|----------|--------------|
| Deployment | Always. Replicas are set to 0 when `suspend=true`. |
| Service | `service.enabled` |
| Ingress | `ingress.enabled` |
| ConfigMap `<app>-config` | `config.data` non-empty |
| ServiceAccount | `serviceAccount.create` |
| HorizontalPodAutoscaler (autoscaling/v2) | `autoscaling.enabled` |
| PodDisruptionBudget | `disruptionBudget.enabled` |

When a feature is disabled or removed from the spec, its resource is deleted.

### 4.5 Behavior

- **Rollout on config and secret change.** The pod template carries the annotation `spacedock.io/config-hash`, a SHA-256 over the ConfigMap data plus the `resourceVersion` of each referenced Secret. Any change triggers a rolling update.
- **Secret watching.** Watch Secrets in Space namespaces, and use a field indexer (`spec.secrets[].name`) to enqueue the Apps that reference a changed Secret.
- **Missing Secret.** Set `SecretsReady=False` with reason `SecretNotFound` and don't create or update the Deployment. Requeue with backoff and resolve via the watch.
- **HPA ownership.** When autoscaling is enabled, the operator omits `spec.replicas` from its SSA apply so the HPA owns that field.
- **Space missing or terminating.** Set `Ready=False` with reason `SpaceNotReady` and stop mutating.
- **Status derived from the Deployment:** `readyReplicas`, `updatedReplicas`, `availableReplicas`, and a rollout state of `Progressing`, `Available` or `Failed`. `Failed` is set when `ProgressDeadlineExceeded` occurs.

### 4.6 Status

```yaml
status:
  observedGeneration: 7
  phase: Running                   # Pending | Progressing | Running | Degraded | Suspended | Failed
  image: 1234...ecr.../checkout-api:1.4.2
  replicas: 3
  readyReplicas: 3
  updatedReplicas: 3
  urls:
    - https://checkout.payments.example.io/
  serviceName: checkout-api
  configHash: 9f2c...
  conditions:
    - type: Ready
    - type: PolicyCompliant
    - type: SecretsReady
    - type: DeploymentReady
    - type: ServiceReady
    - type: IngressReady           # True once the Ingress has a load balancer address
    - type: AutoscalingReady
```

Printer columns for `kubectl get sdapp`: `IMAGE`, `READY` (ready/desired), `PHASE`, `URL`, `AGE`.
Printer columns for `kubectl get space`: `NAMESPACE`, `APPS`, `CPU-USED`, `PHASE`, `AGE`.

---

## 5. Controller Requirements (both controllers)

1. **Idempotent reconcile.** Use Server-Side Apply for all owned objects with `ForceOwnership` under field manager `spacedock-operator`.
2. **Watches.** `Owns()` all child kinds. The App controller also watches `Space` (mapping to all Apps in its namespace) and `Secret` (via the index).
3. **Status updates** go through the status subresource and only happen when the status has changed. Always set `observedGeneration`.
4. **Conditions** use `metav1.Condition` with the standard `Ready` aggregate.
5. **Events.** Emit Normal and Warning events for create, update, delete, policy violations and conflicts.
6. **Errors.** Return errors for transient failures (exponential backoff). Terminal validation failures set a condition and don't requeue hot.
7. **Predicates.** Use `GenerationChangedPredicate` on the primary CRs so status-only updates don't loop.
8. **Leader election** is enabled by default. The Deployment defaults to 2 replicas.
9. **Concurrency.** `MaxConcurrentReconciles` is configurable (default 5 per controller).
10. **Naming overflow.** Generated names that would exceed 63 characters are truncated and suffixed with a short hash.

---

## 6. Operator RBAC and Security

- ClusterRole with the minimum verbs on: namespaces, resourcequotas, limitranges, networkpolicies, rolebindings, serviceaccounts, configmaps, secrets (get/list/watch only), services, deployments, ingresses, horizontalpodautoscalers, poddisruptionbudgets, events, leases, and the Spacedock CRDs and their status and finalizers.
- The operator needs `bind` on ClusterRoles `admin`, `edit` and `view` (restricted with `resourceNames`) so it can create tenant RoleBindings without holding those permissions everywhere.
- The operator **never reads Secret data**. For the config hash it uses `resourceVersion` only. Configure the cache so Secrets are metadata-only (`metav1.PartialObjectMetadata`).
- Operator pod: non-root, read-only root filesystem, all capabilities dropped, seccomp RuntimeDefault.
- Tenants with `admin` or `edit` in a Space must not be able to change the operator-managed ResourceQuota, LimitRange or NetworkPolicies. The default `admin` role can't write quotas, and the operator reverts drift on the rest.
- Provide ClusterRoles for users: `spacedock-space-admin` (manage Spaces, for the platform team) and aggregate App permissions into the `admin`, `edit` and `view` roles via aggregation labels, so Space members can manage Apps.

---

## 7. Operator Configuration

Configuration comes from flags or a ConfigMap `spacedock-config` in `spacedock-system`:

| Key | Default | Purpose |
|-----|---------|---------|
| `defaultQuota` | none | Applied when a Space omits `quota` |
| `defaultLimits` | none | Applied when a Space omits `limits` |
| `defaultIngressClass` | none | Used when an App omits `ingress.className` |
| `reservedNamespaces` | `default,kube-*,spacedock-system` | Blocked as Space namespaces |
| `clusterRoleMapping` | `admin/edit/view` | Maps Space roles to ClusterRoles |
| `maxConcurrentReconciles` | 5 | |
| `webhook.failurePolicy` | `Fail` | |

---

## 8. Observability

- Default controller-runtime metrics on `:8443` (secure, authn/authz filtered), plus custom metrics:
  - `spacedock_spaces_total{phase}`
  - `spacedock_apps_total{space,phase}`
  - `spacedock_policy_violations_total{space,rule}`
  - `spacedock_space_quota_usage_ratio{space,resource}`
- An optional ServiceMonitor in the Helm chart.
- Structured JSON logs (zap) with `space`, `app`, `namespace` and `reconcileID` keys. The log level is configurable.
- Health probes `/healthz` and `/readyz`.

---

## 9. Repository Layout

```
spacedock/
├── api/v1alpha1/            # space_types.go, app_types.go, webhooks
├── internal/
│   ├── controller/          # space_controller.go, app_controller.go
│   ├── builder/             # pure functions: spec → desired objects (unit-testable)
│   ├── policy/              # registry/domain/replica policy evaluation (shared by webhook + controller)
│   └── hash/                # config hash
├── config/                  # kubebuilder kustomize (crd, rbac, webhook, manager, samples)
├── charts/spacedock/        # Helm chart
├── test/e2e/                # kind-based e2e
├── docs/                    # api-reference.md (generated), user-guide.md
├── Makefile
└── Dockerfile
```

The desired-state builders in `internal/builder` must be pure functions that take the CR and config and return objects, with no client calls. That keeps them fully unit-testable.

---

## 10. Testing Requirements

| Level | Coverage |
|-------|----------|
| Unit | All builders (golden-file tests for generated YAML), policy evaluation, hash, name truncation, defaulting. Target ≥ 80% for `internal/builder` and `internal/policy`. |
| envtest | Space lifecycle (create, update quota, remove access entry, delete with Delete and Retain), App lifecycle (create, image update, toggle ingress/HPA/PDB off and on, secret rotation triggers rollout, missing secret, policy violation keeps the old workload), drift revert. |
| Webhook | Every CEL and webhook rule in §3.2 and §4.2, with positive and negative cases. |
| e2e (kind) | Install via Helm with cert-manager and ingress-nginx. Create a Space and an App, curl through the ingress, verify the quota blocks over-allocation, and verify the NetworkPolicy blocks cross-Space traffic. |

---

## 11. Packaging and Delivery

- `make` targets: `generate`, `manifests`, `test`, `test-e2e`, `lint`, `docker-build`, `docker-push`, `helm-package`, `deploy`, `undeploy`.
- Helm chart values cover: image, replicas, resources, operator config (§7), webhook enable and failure policy, cert-manager issuer, ServiceMonitor toggle, and CRD install toggle.
- Generated CRD API reference in `docs/api-reference.md` (use `crd-ref-docs`).
- Sample CRs in `config/samples/`: a minimal Space, a full Space, a minimal App (image + port only), and a full App.
- CI pipeline (GitLab CI): lint, unit and envtest, build a multi-arch image, e2e on kind, then publish the chart and image on tag.

---

## 12. Implementation Milestones

Build in this order. Each milestone must pass its tests before starting the next.

1. **M1: Scaffold and Space core.** Kubebuilder project, both CRD types with CEL, and the Space controller: Namespace, ResourceQuota, LimitRange, PSS labels, status and finalizer.
2. **M2: Space access and network.** RoleBindings, NetworkPolicies (all three modes), pruning, adoption, and the conflict path.
3. **M3: App core.** Deployment, Service, ServiceAccount, status and conditions, `suspend`.
4. **M4: App extensions.** Ingress, ConfigMap, Secret references, config hash and Secret watch, HPA, PDB, topology spread.
5. **M5: Policy and webhooks.** Defaulting and validating webhooks, the shared policy package, controller-side enforcement, and Space→App re-enqueue.
6. **M6: Production readiness.** Metrics, Helm chart, RBAC hardening, e2e suite, docs, CI.

---

## 13. Acceptance Criteria

- [ ] Applying the full Space sample produces a namespace with the correct quota, limits, PSS labels, NetworkPolicies and RoleBindings within 10 seconds. The Space then reports `Ready=True`.
- [ ] Manually editing the ResourceQuota is reverted within one reconcile.
- [ ] An App with only `image` + `ports` produces a working Deployment and Service using Space LimitRange defaults and restricted security context defaults.
- [ ] The full App sample is reachable via its Ingress host in e2e. `status.urls` is populated.
- [ ] Changing `config.data`, or rotating a referenced Secret, triggers exactly one rolling update.
- [ ] An App with a disallowed registry or ingress host is rejected by the webhook. With the webhook disabled, the controller sets `PolicyCompliant=False` and the existing Deployment is left unchanged.
- [ ] Enabling autoscaling hands `replicas` to the HPA with no fight between the operator and the HPA (no replica flapping over 5 minutes).
- [ ] Deleting a Space with `Delete` removes the namespace. With `Retain`, the namespace and its workloads remain and are unlabeled.
- [ ] Pods in Space A cannot reach pods in Space B when isolation is `Namespace` or `Strict`.
- [ ] The operator never logs or caches Secret data.
- [ ] All tests in §10 pass in CI. `golangci-lint` is clean.

---

## 14. Roadmap (out of scope for v1)

- `externalSecrets` on App, generating ExternalSecret CRs (External Secrets Operator, for example AWS Secrets Manager) when the CRD is present.
- Gateway API `HTTPRoute` as an alternative to Ingress.
- `ServiceMonitor` per App.
- Sidecars and init containers.
- Persistent volumes / StatefulSet mode.
- Space templates (`SpaceClass`) for standard t-shirt sizes (S, M, L quotas).
- A `v1beta1` API with a conversion webhook.

---

## 15. Instructions for the Implementing Agent

- Follow Kubebuilder conventions. Regenerate CRDs and RBAC markers (`make manifests generate`) after every API change.
- Put all validation that can be expressed in CEL into CEL first. Use webhooks only for rules that need cluster lookups.
- Don't use `client.Update` for owned children. Use SSA patches.
- Don't store or log Secret values, and don't add a field that accepts plaintext secret data.
- Keep builders pure. Controllers orchestrate: fetch → evaluate policy → build desired state → apply → prune → update status.
- Every new spec field requires: CEL or webhook validation, defaulting (if any), a builder unit test, an envtest case, and a docs update.
- If a requirement here is ambiguous, pick the more restrictive or safer behavior and record the decision in `docs/decisions.md`.
